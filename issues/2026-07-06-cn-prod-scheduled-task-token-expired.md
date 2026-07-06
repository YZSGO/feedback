# cn-prod 定时任务「早上9点没发成功」— 无人值守 run 复用过期 JWT，沙箱鉴权全挂

**问题反馈时间**: 2026-07-06 10:00 (北京时间)
**影响用户**: 17852862749 (userId: 1abd6a58-3424-4769-a71d-be92e9ff80a6)
**Session ID**: `ses_di0Aquj0px86Kv4uGko7Y` (08:00 定时 run, conversationId `oneshot-4551ee1b-...`)
**严重等级**: P1 (核心功能失效：用户离线时的定时任务几乎必然失败，无 workaround)

---

## 问题描述

用户反馈「早上 9 点的定时任务没发成功」。追踪 cn-prod (阿里云 SLS `optima-cn-prod-1911493506120573`) 后发现：

1. 该用户的定时/无人值守 run 今天在 **08:00:02 CST** 被平台批量唤醒（以 OneShot 一次性会话执行），**不是 09:00**（时间偏差见下方「其他发现」）。
2. 这次 run 期间 agent 在沙箱里跑的 CLI（`sentinel-cli` 等）**连续 6 次 bash 工具失败**，根因是 **agent 用的 JWT 已过期**（`Invalid token: Signature has expired.`）。
3. 任务因鉴权失败干不了活，也就没有结果可通过 `SendUserMessage` 发给用户 → 用户侧「9 点什么都没收到」。

用户在容器侧确认：`~/.optima/token.json` 里的 access token 约在 **2026-07-05 17:00 CST 过期**，而本次 run 在 **2026-07-06 08:00** 执行，**已过期约 15 小时**。

---

## 根因分析

### 1. 沙箱失败的直接原因 = JWT 过期（非网络/非 bash 规则）

08:00 session 的 6 次 `tool result failed` 的 `error_preview`：

```
08:01:38 tool=bash  {"code":"INVALID_REQUEST","message":"Body cannot be empty ..."}
08:01:40 tool=bash  {"code":"NOT_FOUND","message":"Route not found"}
08:01:43 tool=bash  Invalid token: Signature has expired.        ← 真因
08:01:53 tool=bash  {"code":"NOT_FOUND","message":"Route not found"}
08:01:53 tool=bash  {"code":"NOT_FOUND","message":"Route not found"}
08:02:25 tool=bash  [stderr] ... @optima-chat/sentinel-cli ... SentinelApiError ...
```

`Signature has expired` 是 PyJWT `ExpiredSignatureError`（`exp` 已过），token 本身过期，**不是**签名密钥不匹配。其余 `Route not found` / `SentinelApiError` 是鉴权失败后的连锁症状。

> ⚠️ 与 #1397（JWT 签名密钥稳定性）**不是同一个问题**：#1397 是「跨 pod 验签失败 / Signature verification failed」，本 issue 是「token 过期 exp + 无人值守 run 没有续期」。#1397 的修复不会自动修好本问题。

### 2. token 是 1h 短效 access token，且只由「在线客户端」刷新

- `user-auth/app/core/config.py:36` → `ACCESS_TOKEN_EXPIRE = 3600`（1 小时），refresh token 14 天。
- 写进 `~/.optima/token.json` 的就是这个用户 access token；`expires_at` 直接取 JWT 的 `exp`（`optima-gateway/packages/session-manager/src/token-efs-writer.ts:66 deriveExpiresAtMs`、`agent-runtime/src/ws-client.ts:164`）。gateway/agent **只透传、不重签**。
- 刷新是**纯客户端驱动**：前端用 refresh token 调 `/api/refresh-token`（`gateway-core/src/routes/auth.ts:19` → user-auth `grant_type=refresh_token`）换新 token，再发 `update_token` 回推 → `session-manager.ts:5887 updateSessionToken()` 写 EFS + 转发容器。代码注释实锤（`connection-handler.ts:1602`）：「前端已有 handler（forceRefreshToken + update_token 回推），无感续」。gateway **没有**服务端刷新循环。

### 3. 定时/OneShot 路径根本不给 token（核心缺口）

- `gateway-core/src/routes/one-shot.ts:86` → `sessionManager.createSession(userId)` —— **不带 options.token**。
- `createSession` 里 token 全靠 `options?.token`（`session-manager.ts:1364 / 1451(OPTIMA_TOKEN) / 1477 / 1551 currentToken`）。缺省即 `currentToken=undefined`、init_user 无 token。
- → 容器 `ws-client.ts:543 if(token)` 为假 → **不写新 token** → 只能吃 EFS 上**上一次交互 session 遗留**的 1h token → 过期即废。

**日志对照（同一天）**：

| Session | gateway token 事件 | init 写 token |
|---|---|---|
| 08:00 OneShot 定时 (`ses_di0Aquj0px86Kv4uGko7Y`) | **0 条** | **无 `Token written`** |
| 交互 session (`ses_lqWaAxXrC4tIKQH0NMs1i`) | 13 条，含 `Token updated via WS`→`written to EFS`→`forwarded to agent-runtime`（09:28、10:12 两次） | — |

> 一句话：定时任务在当前代码里**没有独立的 token 供给**，复用用户上次在线时留在 EFS 的浏览器 access token，谁也不给它续。用户离线超过 1h（早上 9 点定时任务几乎必然如此）→ token 过期 → 沙箱鉴权全挂。

---

## 定时任务 token「应该多久 / 怎么修」

- 单次 run 时长上限：OneShot 同步 `DEFAULT_TIMEOUT_MS = 600_000`（10 分钟）、异步 `≤ 3600_000`（1 小时）（`one-shot.ts:22/192`）。
- 所以 **token 只需覆盖「本次执行」即可**——现有 1h TTL 只要**在触发那一刻现签**就绰绰有余。**不要把 TTL 调长**（削弱安全 + 治标不治本：定时间隔可跨天/跨周，任何固定 TTL 都盖不住离线场景）。
- **正解 = 把签发时机搬到触发时**：`executeOneShot` 触发时用 refresh token / service token 现签一个新的短效 access token，经 `createSession(userId, { token })` 注入容器。间隔续期交给 refresh token（14 天）/ service token。

现成可复用构件（当前 OneShot 路径没用）：
- `gateway-core/src/routes/auth.ts:19` `/api/refresh-token`（`grant_type=refresh_token`）
- `gateway-core/src/services/registry-m2m-token.ts:77`（`grant_type=client_credentials` M2M）
- `user-auth/app/services/impersonation.py:100/205 create_access_token`（系统代表用户签发）

---

## 解决方案

### 短期（立即，止血）
- `executeOneShot`（`one-shot.ts`）触发时，先为该 user 现签一个短效 access token 并经 `createSession(userId, { token })` 注入；避免复用 EFS 旧 token。
- 启动前校验 `token.json.expires_at`，已过期则先刷新/拒跑并告警，杜绝「静默跑一趟全失败还占额度」。

### 中期（1–2 周）
- 为无人值守 run 建立服务端 token 供给：优先用 refresh token / impersonation 签发；在 run 生命周期内需要时续签。
- 补日志：`SendUserMessage`/`brief` 发送路径当前不落 SLS（`proactive`/`brief`/`SendUserMessage` 全舰队 0 命中，`brief-tool.ts:97` 是可选链 `deps.log?.`）→ 无法定位「用户没收到消息」。补必达日志。

### 长期
- 明确「后台任务凭证」与「用户交互凭证」分离，不再拿 1h 交互 access token 当后台凭证。

---

## 其他发现（建议单独跟进）

1. **触发时间 08:00 vs 用户预期 09:00**：本用户连续三天（07-04/05/06）都在 **08:00:02** 被唤醒；对照的 9 点用户（如 `196d66e1-...`）在 **09:00:01** 唤醒且干净完成。疑似调度侧时区/时点 off-by-one（差整 1 小时，中国 UTC+8 无夏令时）。触发方在 SLS 服务外（阿里云 EventBridge/SAE 定时或外部 cron），需另查配置。**待确认**。
2. **可观测性盲区**：定时任务的「发送」动作不落日志，见「中期」第 2 条。

---

## 复现步骤

1. 用户在 cn-prod 创建一个定时任务（如「每天 9 点…」）。
2. 让用户在触发时间前 > 1 小时保持离线（不刷新 token）。
3. 到点后定时 run 唤醒 → 容器 `~/.optima/token.json` 为过期旧 token。
4. 观察沙箱内带鉴权的 CLI 全部 `Invalid token: Signature has expired.` → 任务无产出、用户收不到消息。

---

## 相关文件

- `user-auth/app/core/config.py:36` — `ACCESS_TOKEN_EXPIRE = 3600`
- `optima-gateway/packages/gateway-core/src/routes/one-shot.ts:86` — `createSession(userId)` 未带 token（核心缺口）
- `optima-gateway/packages/session-manager/src/session-manager.ts:1364/1451/1477/1551` — token 进 session/init_user 的入口
- `optima-gateway/packages/session-manager/src/session-manager.ts:5887/5926` — `updateSessionToken` / `update_token` 转发（仅客户端驱动）
- `optima-gateway/packages/session-manager/src/token-efs-writer.ts:66` — `deriveExpiresAtMs`（透传 exp，不重签）
- `optima-gateway/packages/agent-runtime/src/ws-client.ts:154/543` — 写 `~/.optima/token.json`
- `optima-gateway/packages/gateway-core/src/routes/auth.ts:19` — `/api/refresh-token`（可复用）

---

## 验证清单

- [ ] 定时/OneShot run 触发时注入现签的有效 token（不再复用 EFS 旧 token）
- [ ] 用户离线 > 1h 后，定时任务仍能正常调用带鉴权的 CLI（无 `Signature has expired`）
- [ ] token 过期时启动前拦截 + 告警，不再静默跑失败
- [ ] `SendUserMessage`/定时任务发送路径落 SLS 日志，可端到端追踪
- [ ] （单独）核对本用户定时任务的配置时间（08:00 vs 09:00）与时区
