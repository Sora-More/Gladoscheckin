# Cookie 刷新手册（`认证失败 (code -2)`）

作业变红、日志里是 `认证失败 (code -2, message: 没有权限)` 时看这一份就够了：
症状怎么确认、cookie 从哪取、怎么验证、为什么"自动续期"做不到，都有实测结论。
目标是**不用重新排查**，照着第 0~3 节做完就恢复。

## TL;DR（赶时间就照这个执行）

1. 红的原因几乎总是 cookie 过期，**先别怀疑代码**（第 0 节两个信号可确认）。
2. 从本机 Chrome 用 CDP 取新 cookie（第 1 节）。
3. `/api/user/points` 只读验证，必须回 `code:0`（第 2 节）。
4. 写 secret，**别 echo 值**（第 3 节）。
5. 触发一次 `workflow_dispatch`，日志出现 `✅ 签到成功` 即恢复（第 3 节）。

## 0. 30 秒定位：是不是 cookie 的问题

两个信号同时出现 = cookie 过期：

- `Running checkin` 步骤只有一行失败：
  ```
  checkin 认证失败 (code -2, message: 没有权限): Cookie 无效、已过期或不完整
  签到失败 (退出码 1)
  ```
- **同一份日志里没有** `Cookie 缺少会话字段 gld:sess.sig` 这类告警
  → 说明 `gld:sess` 与 `gld:sess.sig` 都在，结构没问题，只是服务端不认了。

> 别被下面这个误导：不带任何 cookie 请求，响应体**一模一样**。所以
> "接口回 code -2" 不能用来判断"站点挂了/接口改了"，它对任何未认证请求都这么回；
> 要区分"过期"还是"漏字段"，只能看有没有上面那个告警。
>
> ```bash
> curl -s -X POST https://glados.cloud/api/user/checkin \
>   -H 'Content-Type: application/json' -d '{}'
> # {"code":-2,"message":"没有权限"}
> ```

顺手确认工作流接线没坏（30 秒）：`gh run view <id> -R BreakFree003/Gladoscheckin` 里
`今天是否已经签到成功` 正常打印了 UTC 日界点和成功数、`Keep alive` 步骤是绿的，
就说明问题只在凭证，不在 workflow。

## 1. 从自己的浏览器取 cookie（macOS + Chrome）

前提：Chrome 里已登录 `glados.cloud`，且它开着远程调试端口（本机常态 `127.0.0.1:9222`）。

三个坑，先记住，能省掉两小时：

- **坑 1**：Chrome 155 上 `/json/version`、`/json/list`、`/json` 全部返回 **404**
  （`Content-Length: 0`）。别再用 HTTP 发现接口去找 target，直接连浏览器级 WebSocket。
- **坑 2**：`document.cookie` 读不到 `gld:sess` / `gld:sess.sig` —— 两个字段都是 `httpOnly`。
  必须走 CDP 的 `Storage.getCookies`（浏览器级，无需 target / contextId）。
- **坑 3**：端口与 ws 路径以 `DevToolsActivePort` 为准，别硬编码。

```bash
cat ~/Library/Application\ Support/Google/Chrome/DevToolsActivePort
# 9222
# /devtools/browser/9347257d-2f11-4e23-980e-adfe2f6e2e4f     ← 这段就是 ws 路径
```

Node ≥ 22 自带 `WebSocket`，不用装任何包。把下面存成 `/tmp/glados_cdp.mjs`：

```js
import fs from 'node:fs';
const [WS_BASE, WS_PATH, OUT] = ['ws://127.0.0.1:9222', process.argv[2], process.argv[3]];
const ws = new WebSocket(WS_BASE + WS_PATH);
let nextId = 0;
const pending = new Map();
const call = (method, params = {}) => {
  const id = ++nextId;
  ws.send(JSON.stringify({ id, method, params }));
  return new Promise((res, rej) => pending.set(id, { res, rej }));
};
ws.addEventListener('message', (ev) => {
  const m = JSON.parse(ev.data);
  const p = pending.get(m.id);
  if (!p) return;
  pending.delete(m.id);
  m.error ? p.rej(new Error(JSON.stringify(m.error))) : p.res(m.result);
});
ws.addEventListener('error', (e) => { console.error('WS error', e.message ?? e); process.exit(1); });
ws.addEventListener('open', async () => {
  const { cookies } = await call('Storage.getCookies');
  const gl = cookies.filter((c) => /(^|\.)glados\.cloud$/.test(c.domain));
  const now = Date.now() / 1000;
  for (const c of gl) {
    const exp = c.expires < 0 ? 'session' : new Date(c.expires * 1000).toISOString();
    const left = c.expires < 0 ? '' : ` (${((c.expires - now) / 86400).toFixed(1)}d left)`;
    console.log(`${c.name.padEnd(14)} exp=${exp}${left} httpOnly=${c.httpOnly}`);
  }
  if (!gl.length) { console.log('NO COOKIES: 先在浏览器登录 glados.cloud'); process.exit(2); }
  fs.writeFileSync(OUT, gl.map((c) => `${c.name}=${c.value}`).join('; '), { mode: 0o600 });
  console.log('written:', OUT);
  process.exit(0);
});
setTimeout(() => { console.error('timeout'); process.exit(3); }, 15000);
```

```bash
node /tmp/glados_cdp.mjs /devtools/browser/<上面那段 uuid> /tmp/glados_cookie.txt
```

正常输出（`gld:sess` 的剩余天数就是要不要马上动手的信号）：

```
gld:sess       exp=2026-11-09T04:43:55.663Z (13.4d left) httpOnly=true
gld:sess.sig   exp=2026-11-09T04:43:55.663Z (13.4d left) httpOnly=true
written: /tmp/glados_cookie.txt
```

6 个 cookie（还有 `koa:sess`、`__stripe_mid`、`eu_cookie`）全部拼进去发没问题，
脚本只认 `gld:*`，多余的忽略——不用手工挑字段。

## 2. 先验证再用（只读，不消耗签到）

```bash
curl -s -H "Cookie: $(cat /tmp/glados_cookie.txt)" \
  -H "Accept: application/json, text/plain, */*" \
  -H "Origin: https://glados.cloud" \
  https://glados.cloud/api/user/points | head -c 120
# {"code":0,"points":"326.0000000000000000","history":[...
```

`code:0` 才往下走。若这里也是 `-2`，说明浏览器里那份同样失效 →
先去浏览器重新登录（登录要邮箱验证码，见第 4 节），别急着写 secret。

## 3. 写入 secret + 触发验证

```bash
gh secret set GLADOS_COOKIES -R BreakFree003/Gladoscheckin < /tmp/glados_cookie.txt
gh secret list -R BreakFree003/Gladoscheckin          # updated_at 应为刚刚

gh workflow run "auto check" -R BreakFree003/Gladoscheckin
gh run view <run_id> -R BreakFree003/Gladoscheckin --log | grep -E "签到成功|积分|退出码"
# ✅ 签到成功, 获得 12 积分, 总 338 积分, 未到 500 兑换门槛
```

- **坑 4**：`gh run view --log` 不带 `-R`、且当前目录不是 git 仓库时会报
  `failed to determine base repo: fatal: not a git repository` —— 在 `/tmp` 里跑必然踩到。
- **坑 5**：`workflow_dispatch` 是"手动点就跑"的语义，**不会**被"今天已签到"跳过。
  今天已签过也无所谓，服务端回 `code 1 重复签到` 同样退出 0（绿）。
- **坑 6**：这次手动跑成功后，当天剩余槽位会自动 `skip`，属正常，不用管。

## 4. 已知事实：为什么不能自动续期

| 问题 | 结论 | 证据 |
|---|---|---|
| 服务端会滚动续期吗 | **不会** | 用有效会话请求 `/api/user/points`，响应头里**没有任何 `Set-Cookie`** |
| 浏览器里被续写过吗 | **没有** | Chrome cookie 库：`gld:sess` 的 `last_update_utc == creation_utc` |
| "天天用就不会过期"成立吗 | **不成立** | 2026-09-25 贴进 secret 的那份天天签到在用，第 31 天照样失效 |
| 能让 action 自己回写 secret 吗 | **不能** | `GITHUB_TOKEN` 无权写仓库 secrets（需 `secrets: write` 的 PAT / App token） |
| 能自动重新登录吗 | **不现实** | 登录要**邮箱验证码**，还可能有 Turnstile，见下 |

复现"服务端不续期"的证据：

```bash
# 只看响应头，不打印 cookie 值
curl -s -D - -o /dev/null -H "Cookie: $(cat /tmp/glados_cookie.txt)" \
  -H "Origin: https://glados.cloud" https://glados.cloud/api/user/points \
  | sed -E 's/(=)[^;]{8,}/\1<MASKED>/g' | grep -i -E "set-cookie|^HTTP"

# Chrome 自己记录的时间戳（1601 epoch，微秒）
cp ~/Library/Application\ Support/Google/Chrome/Default/Cookies /tmp/Cookies.copy
sqlite3 /tmp/Cookies.copy "select name,\
  datetime(creation_utc/1000000-11644473600,'unixepoch','localtime'),\
  datetime(last_update_utc/1000000-11644473600,'unixepoch','localtime'),\
  datetime(expires_utc/1000000-11644473600,'unixepoch','localtime')\
  from cookies where host_key like '%glados%';"
```

登录链路的实际形状（从 `https://glados.cloud/app.bundle.js` 里读出来的）：

```
POST /api/authorization  {"address": <email>, "site": ..., "captcha_token"?}
     → 服务端可能回 captcha_required + sitekey
     → 前端此时才加载 challenges.cloudflare.com/turnstile/v0/api.js?render=explicit
POST /api/login          {"method":"email","site":"glados.network","email":...,"mailcode":...}
```

`mailcode` 就是**邮箱收到的验证码**。所以"CI 自己续期"= 给 Actions 邮箱收件能力
（IMAP 凭据）+ 可能还要过 Turnstile，见第 5 节。

**寿命实测（重要：不是固定 30 天，越往后越短）**：

| 会话来源 | 创建 | 失效 | 寿命 |
|---|---|---|---|
| 2026-09-25 贴进 secret 的那份 | 2026-09-25 12:14 UTC | 2026-10-26 16:46 UTC 被拒 | 约 **30 天** |
| 2026-10-26 从浏览器取到的那份 | 2026-10-26 12:43:55（北京） | 2026-11-09 12:43:55 | **14 天**（cookie 上的 `Expires`；服务端 TTL 可能更长但不可知） |

→ **保守按 14 天维护**：拿到新 cookie 后两周内该再刷一次，别照 30 天等。

## 5. 想做全自动的话（结论：不做，留个路线）

唯一自洽的方案是"过期时自动重新登录 + 把新 cookie 存到 Actions 能读写的地方"：

1. 触发条件：签到回 `code -2` 时进入登录分支，正常路径不变。
2. 登录：`/api/authorization` →（可能）Turnstile → 收邮件取 `mailcode` → `/api/login`。
   代价：邮箱 IMAP 凭据进 secrets；Turnstile 一旦触发，纯 HTTP 就断了，得上无头浏览器且不保证过。
3. 持久化：`permissions: contents: write` + 用 `GITHUB_TOKEN` 把**对称加密后**的 cookie
   提交进仓库（`GITHUB_TOKEN` 的提交不会触发新工作流，不会死循环；密钥放 secret）。
   注意本仓库是 **public**，落盘的必须是密文。

收益是每两周省 30 秒；代价是一份邮箱凭据 + 一个加密密钥 + 一条随时可能被风控掐断的登录自动化。
**当前判断：不做。**

## 6. 安全约束（别踩）

- cookie 是**账号凭证**：不要 `echo`、不要 `cat`、不要写进 commit / issue / 日志。
- 落盘用 `chmod 600` 的临时文件，用完 `rm`。
- `gh secret set` 走 `< 文件` 重定向，不要 `-b "$(cat ...)"`（会进 shell history 与进程参数）。
- 打印响应头先脱敏：`sed -E 's/(=)[^;]{8,}/\1<MASKED>/g'`。
- 别忘了删：`/tmp/glados_cookie.txt`、`/tmp/Cookies.copy`、`/tmp/glados_cdp.mjs`。

## 7. 下次直接说这句话

> "GitHub Action 又红了，按 docs/cookie-refresh.md 处理。"

给 agent 的最小序列：

```bash
# 1) 确认症状（看是否是 code -2，且没有「缺少会话字段」告警）
gh run list --limit 3 -R BreakFree003/Gladoscheckin
gh run view <id> -R BreakFree003/Gladoscheckin --log | grep -E "code -2|缺少会话字段|签到"
# 2) 取 cookie（第 1 节）→ 3) 只读验证（第 2 节）
# 4) 写 secret + 触发（第 3 节）→ 5) 清理临时文件（第 6 节）
```

---

## 变更记录

- **2026-10-27**：首次凭证过期故障。旧会话（2026-09-25 写入）第 31 天失效，当天
  `16:46 UTC` 的 schedule 与 `17:53 UTC` 的手动重试各红一次。从本机 Chrome 取新 cookie
  → 写 secret（`18:47:38Z`）→ `workflow_dispatch` 补签成功（`+12` 分，总 `338`）。
  同次确认了"服务端不续期"与"登录需邮箱验证码"两件事，并新增本文档。
