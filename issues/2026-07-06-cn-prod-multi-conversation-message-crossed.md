# cn-prod 多会话「消息串了」— restore_conversations 走 out-of-band lane，injectHistory 与在途 turn 竞态，回复基于错误会话上下文生成

**问题反馈时间**: 2026-07-06 (北京时间)
**影响用户**: 17852862749 (userId: 1abd6a58-3424-4769-a71d-be92e9ff80a6)
**Session ID**: `ses_lqWaAxXrC4tIKQH0NMs1i`（单 session 内并发多会话）
**涉及会话**: `cmr8mnjyy01q70jf8bibxhaic`（用户当时在聊、被投错的会话）、`cmqvxzgkw085g0jf50htck2dr`（被"串"进来的会话）
**严重等级**: P1（数据正确性：多会话并发 + 长 turn 时，某个会话的回复可能**基于另一个会话的上下文**生成/投递；同用户内，无隐私泄漏，但结果错乱且无用户侧 workaround）

---

## 问题描述

用户反馈「消息串了」。追踪 cn-prod（阿里云 SLS `optima-cn-prod-1911493506120573`）的 gateway-core + agent-runtime 后确认：该用户在**同一个 session 内同时开着多个会话**，其中一次 turn 的 **LLM 请求跑到了另一个会话的上下文上**。

Smoking gun —— 全窗口逐 turn 比对，只有 `turnId = turn-...-850z5icx` 这一个 turn 的 conversationId 在生命周期内发生了**翻转**：

```
03:43:20.995 UTC  conv=cmr8mnjyy01q   L2 Snip compaction          ← 压缩的是 cmr8mnjyy（240→128 msgs）
03:43:20.999 UTC  conv=cmqvxzgkw085   LLM request sending (102 msgs)   ← 却把 cmqvxzgkw 的历史发给了 LLM ❌
03:43:21.380 UTC  conv=cmqvxzgkw085   LLM response headers received
03:43:43.187 UTC  conv=cmqvxzgkw085   LLM stream completed (end_turn, 1297 tok)
03:43:43.188 UTC  conv=cmr8mnjyy01q   Optima session turn completed
```

同一个 turn：压缩与 turn-completed 都记在 `cmr8mnjyy`，但真正发给 LLM 的是 `cmqvxzgkw` 的 102 条历史。→ 用户在 `cmr8mnjyy` 会话里，收到的是**基于 `cmqvxzgkw` 会话内容**生成的回答。这就是"串"。

> **判定**：涉及会话全部属于该用户自己（cache 路径均在 `/home/aiuser/.optima/users/1abd6a58-.../conversations/`）。是**同用户跨会话**串，不是跨用户泄漏——**无隐私安全问题**，但属回复正确性缺陷。

---

## 根因分析

### 1. agent-runtime 单容器/单 session 靠"换上下文"服务多个会话

`ws-bridge` 全程只持有**一个** Pi Agent session + **一个** `currentConversationId`（`ws-bridge.ts:181`）。会话切换 = 销毁旧 session + 新建绑定新 conversationId 的 session + `injectHistory` 灌入该会话历史（切换判据 `ws-bridge.ts:1239`，`currentConversationId` 在 `ws-bridge.ts:1345` 赋值，注入在 `ws-bridge.ts:1364-1392`）。这套"共享单 session + 换上下文"本身就是竞态温床。

### 2. `restore_conversations` 走 **out-of-band lane**，不与 `message` 串行（核心缺口）

入站帧闸门只把两类事件放进 **ordered lane（单队列串行）**：

```ts
// inbound-frame-gate.ts:36
export const ORDERED_EVENT_TYPES: ReadonlySet<string> = new Set(['message', 'switch_config']);
```

其余帧类型（含 `restore_conversations`）一律 **out-of-band lane —— 立即异步处理、不进队列**（`inbound-frame-gate.ts:18-22`、`ws-client.ts:841/1628`）。

out-of-band lane 的**设计初衷是对的**：`answer_question` / `approval_response` / `abort` / `ping` 是 in-flight message 的 **unblock 信号**，进队列会和 message handler 互等死锁。**但 `restore_conversations` 与它们性质不同**：它不是 unblock 信号，而是会**改写共享 session 状态**的写操作。

### 3. `restore_conversations` 在共享 session 上直接 `injectHistory`，与在途 turn 抢同一个 session

```ts
// restore-handler.ts:150
const agentSession = bridge.getSession();   // ← 就是在途 turn 正在用的那个 this.session
...
// restore-handler.ts:237
agentSession!.injectHistory!(finalEntry);   // ← 把「另一个会话」的历史灌进这个 session
```

于是当 conv A（cmr8mnjyy）的 turn 在途时，一条针对 conv B（cmqvxzgkw）的 `restore_conversations` 走 out-of-band lane 并发执行，在**同一个 session** 上 `injectHistory(B 的历史)` → 把会话上下文从 A 换成了 B → 在途 turn 的下一轮 LLM 请求读到 B 的上下文 → **串**。

> ⚠️ **本类 bug 有前科**：`restore-handler.ts:226-232` 的注释（#748，sibada-stage 实证：一次 reconnect 的 49 条 advertise 把 24 条活动会话上下文 `messages.length=0` 清空）已记录过**同一竞态的兄弟形态**。当时只窄修了「advertise-only（无 history）跳过 injectHistory」这一子分支；**带 history 的 restore（cacheHint 命中）在长 turn 期间 injectHistory 覆盖活动会话的通用竞态仍然存在**——本 issue 就是它的下一次显形。

### 4. 触发条件（本次为何命中）

三件事叠加把窗口撑到必然命中：

1. **超长 turn**：`turn-356ek4ch`（conv cmr8mnjyy，deepseek-v4-pro）`durationMs 1025649`（**~17 分钟**），多轮 tool_use + 反复 L2 压缩，长时间独占该 session。
2. **restore 洪水**：客户端不停 advertise/切会话，gateway 反复下发 `restore_conversations`（sync/count:1，刷了几十次）。每次 restore 要"修复乱序 tool 消息 + 注入 100 条历史"，**单次耗时 ~12 秒**（`durationMs` 12253/11837/12136/11814/11552…），持续与在途 turn 交错——其中 **03:41:40 一次针对 cmqvxzgkw 的 restore** 把上下文换走。
3. **gateway 侧护栏失守**：`message processing delay exceeded threshold`（`delayMs 590053`≈9.8 分钟，阈值 30s），且长 turn 期间多次 `renewActiveConvLock re-acquired (lock had expired)`——gateway 的活动会话锁在长 turn 内过期，本该串行化多会话的护栏没兜住。超时还触发 gateway 重投消息（agent 侧 `Duplicate delivery, acked but skipping re-processing`，幂等挡住了重投这一条，但**幂等挡不住会话指针被换**）。

---

## 解决方案

### 短期（立即，止血）
- **把 `restore_conversations` 从 out-of-band lane 挪进 ordered lane**（或单独给它一条"写操作"串行队列），使其**不与在途 `message` turn 并发**在同一 session 上 `injectHistory`。注意：仍要与 `answer_question`/`abort`/`ping` 等真·unblock 信号区分开，别把 restore 和它们混为一谈。
- **injectHistory 前校验 conversationId**：若目标会话 ≠ 当前在途 turn 的 conversationId，且该会话正有 in-flight turn，则**推迟/拒绝** injectHistory，别覆盖活动上下文（延续 #748 的防御思路，从"advertise-only 跳过"升级为"活动会话一律不被 restore 覆盖"）。

### 中期（1–2 周）
- **turn 冻结 conversationId**：turn 开始即绑定其 conversationId 与所需上下文，LLM 请求组装时不再读一个会被 restore 改脏的共享指针（`ws-bridge.ts` 的 `this.currentConversationId` / `this.session`）。
- **gateway 侧串行化**：长 turn 期间活动会话锁续期，覆盖整个 turn 时长（观察到 `renewActiveConvLock ... lock had expired`）；restore 与 active turn 在 gateway 侧也应串行，别在 turn 未完成时下发跨会话 restore 洪水。
- **抑制 restore 洪水**：客户端每次 reconnect/切会话都 advertise 全量会话，单次 injectHistory ~12s，几十次叠加放大竞态窗口。评估去抖/合并 advertise。

### 长期
- 重新审视"单容器单 session 换上下文服务多会话"模型：要么**每会话独立 session/隔离上下文**，要么把所有改 session 状态的入站帧统一纳入一条串行写通道，从架构上消除会话上下文串扰。

---

## 其他发现（建议单独跟进）

1. **超长 turn（17 分钟 / 1025649ms）本身值得查**：多轮 tool_use 循环 + 反复 L2 压缩，是它撑开了竞态窗口、也触发了 gateway 的 `MSG_PROCESSING_DELAY_SLOW`。是否有 tool loop 打转 / 压缩抖动，可另查。
2. **restore 单次 ~12s 偏慢**：每次 `restore: repaired out-of-order tool messages` + 注入 100 条历史耗时 11–12s，高频重复。乱序 tool 消息为何需要反复修复，值得看 cache 侧。

---

## 复现步骤

1. 同一 session 内开 ≥2 个会话（A、B）。
2. 在会话 A 发起一个**长 turn**（多轮 tool_use，持续数分钟，长时间占用 session）。
3. 在该 turn 未结束时，让客户端触发对会话 B 的 `restore_conversations`（reconnect / 切到 B / advertise），且 B 的历史在 local cache 命中（走 `injectHistory` 分支）。
4. 观察：A 的下一轮 LLM 请求 `conversationId` 变成 B、`messageCount` 变成 B 的历史条数 → A 收到基于 B 上下文生成的回复。
5. 日志判据：同一 `turnId` 内 `L2 Snip compaction` / `turn completed` 的 conversationId 与 `LLM request sending` 的 conversationId 不一致。

---

## 相关文件

- `optima-gateway/packages/agent-runtime/src/inbound-frame-gate.ts:36` — `ORDERED_EVENT_TYPES` 仅含 `message`/`switch_config`，**restore_conversations 落 out-of-band lane（核心缺口）**
- `optima-gateway/packages/agent-runtime/src/inbound-frame-gate.ts:18-22` — 双 lane 设计说明（out-of-band 初衷是 unblock 信号防死锁）
- `optima-gateway/packages/agent-runtime/src/ws-client.ts:1628` — `message / switch_config 保序，其余 out-of-band`
- `optima-gateway/packages/agent-runtime/src/ws-client.ts:913` — `handleRestoreConversations` dispatch（out-of-band 立即异步）
- `optima-gateway/packages/agent-runtime/src/restore-handler.ts:150` — `bridge.getSession()`（取到的是在途 turn 正用的同一 session）
- `optima-gateway/packages/agent-runtime/src/restore-handler.ts:237` — `agentSession.injectHistory(finalEntry)`（在共享 session 上灌别的会话历史）
- `optima-gateway/packages/agent-runtime/src/restore-handler.ts:226-232` — **#748 兄弟竞态注释**（advertise 清空活动会话，窄修）
- `optima-gateway/packages/agent-runtime/src/ws-bridge.ts:181` — `currentConversationId`（单个共享指针）
- `optima-gateway/packages/agent-runtime/src/ws-bridge.ts:1239/1345/1364-1392` — 会话切换判据 / 赋值 / injectHistory
- gateway-core（日志级观测，未逐行核源）：`MSG_PROCESSING_DELAY_SLOW`（delayMs 590053）、`renewActiveConvLock re-acquired (lock had expired)`、`Duplicate delivery`

---

## 验证清单

- [ ] `restore_conversations` 不再与在途 `message` turn 并发在同一 session 上 `injectHistory`（挪进 ordered lane 或独立写队列）
- [ ] 活动会话（有 in-flight turn）的上下文不会被任何 restore（含带 history 的 cacheHint 命中）覆盖
- [ ] 同一 `turnId` 内 compaction / LLM request / turn-completed 的 conversationId **始终一致**（加断言/日志校验）
- [ ] 复现场景（长 turn + 并发跨会话 restore）下，A 会话回复不再基于 B 上下文
- [ ] （中期）gateway 活动会话锁续期覆盖整个 turn 时长，长 turn 期间不再 `lock had expired`
- [ ] （可选）advertise/restore 去抖，降低竞态窗口与 12s×N 的重复注入开销
