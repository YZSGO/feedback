# cn-prod tk 视频下载失败 - yt-dlp 未安装 + bash security 误杀

**问题反馈时间**: 2026-06-27 14:10  
**影响用户**: 17852862749 (userId: 1abd6a58-3424-4769-a71d-be92e9ff80a6)  
**Session ID**: `ses_08813d40-a40d-4879-886b-453880d12b41`  
**严重等级**: P2 (功能不可用，无 workaround)

---

## 问题描述

用户尝试让 agent 下载 tk 视频链接 `https://video.twimg.com/ext_tw_video/...`，遇到连续失败：

1. **14:10:18** - agent 执行 `yt-dlp -F "URL"` 被 bash security 拦截：
   ```
   Error: command rejected (soft-deny): Command contains quoted characters in flag names
   ```

2. **14:10:27** - agent 回复"CDN 视频已过期，无法使用"（实际是误判，链接 14:29 时仍可访问）

3. **14:12-14:13** - agent 尝试安装 yt-dlp（pip/brew/apt），全部失败（不在 allowlist）

4. **14:29** - 用户手动验证 tk 链接有效，可观看可下载

---

## 根因分析

### 1. cn-prod agent 镜像未安装 yt-dlp

`docker/agent.Dockerfile` 和 `docker/agent-isolated.Dockerfile` 中只通过 faster-whisper 依赖安装了 ffmpeg，**没有安装 yt-dlp**。

### 2. bash security 误杀合法命令

`packages/agent-runtime/src/tools/bash/utils/bashSecurity.ts:1537-1546` 的 `checkObfuscatedFlags` 函数将 `yt-dlp -F "URL"` 误判为 flag 混淆攻击：

```typescript
// 检测逻辑：flag 名中包含引号字符
if (flagContent.includes('"') || flagContent.includes("'")) {
  return {
    behavior: 'ask',
    message: 'Command contains quoted characters in flag names',
  }
}
```

**误伤场景**：`-F "URL"` 是 yt-dlp 的标准用法（`-F` 列出可用画质，URL 用引号包裹是标准做法），但被识别为 flag 名包含引号。

### 3. 可用工具不足

cn-prod agent 只加载了 **ecommerce** skill pack（12 个工具），缺少：
- 视频下载工具
- browser 工具（即使有，Playwright 也不支持视频下载）

LLM 只能尝试 yt-dlp，但 yt-dlp 未安装 + bash security 拦截。

### 4. LLM 误判 CDN 过期

14:10:27 的 `<think>` 块中，LLM 推断"CDN 视频已过期"，这是推理结论而非代码检测。实际 tk CDN 链接几小时内有效，用户 14:29 验证仍可访问。

---

## 日志证据

**cn-prod agent-runtime 日志**（13:00 后，session `ses_08813d40-a40d-4879-886b-453880d12b41`）：

```
14:10:18 - [tool_call_error] Error: command rejected (soft-deny): Command contains quoted characters in flag names
           Command: yt-dlp -F "https://video.twimg.com/..."

14:10:27 - [assistant_message] CDN 视频已过期，无法使用

14:12:14 - [tool_call] which yt-dlp
14:12:14 - [tool_result] yt-dlp not found

14:12:37 - [tool_call] pip install yt-dlp
14:12:37 - [tool_call_error] command rejected (not in allowlist)
```

**Skill pack 加载日志**：
```
14:10:00 - Skill pack loaded: ecommerce (12 tools)
```

---

## 解决方案

### 短期修复（立即）

**cn-prod agent Dockerfile 安装 yt-dlp**：

```dockerfile
# docker/agent.Dockerfile 或 docker/agent-isolated.Dockerfile
RUN pip install --no-cache-dir yt-dlp
```

**验证**：重新构建 cn-prod agent 镜像，测试 `yt-dlp -F "URL"` 是否可执行。

### 中期修复（1-2 周）

1. **bash security 规则优化**：
   - 对 `-F "URL"`、`-o "filename"` 等合法模式做例外
   - 区分 flag 名包含引号（混淆）vs flag 后跟引号参数（合法）
   - 添加 yt-dlp、ffmpeg 等媒体工具的白名单

2. **新增 video-downloader skill pack**：
   - 封装 yt-dlp 调用（支持 tk/YouTube/B站等）
   - 自动下载 + 上传到文件存储
   - 处理转码、画质选择

### 长期优化

- 评估是否需要 browser 工具的视频下载能力（Playwright + ffmpeg 组合）
- 完善 cn-prod 的工具链 allowlist（pip/brew 等包管理器）

---

## 影响范围

- **当前**：所有 cn-prod 用户无法下载视频（tk/YouTube 等）
- ** workaround**：用户需手动复制链接到本地下载工具

---

## 复现步骤

1. 在 cn-prod 环境发起新对话
2. 发送 tk 视频链接：`https://video.twimg.com/ext_tw_video/...`
3. 要求"下载这个视频"
4. 观察 agent 尝试调用 yt-dlp 失败 + 误判 CDN 过期

---

## 相关文件

- `docker/agent.Dockerfile` - cn-prod agent 镜像构建
- `packages/agent-runtime/src/tools/bash/utils/bashSecurity.ts:1537-1546` - 误杀规则
- `packages/agent-runtime/src/tools/bash/index.ts:128` - tool call error 格式化

---

## 验证清单

- [ ] cn-prod agent 镜像包含 yt-dlp
- [ ] `yt-dlp -F "URL"` 命令可执行（不被 bash security 拦截）
- [ ] 用户可成功下载 tk 视频
- [ ] bash security 不误杀合法的 flag + 引号参数模式
