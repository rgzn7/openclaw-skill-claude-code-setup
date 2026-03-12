---
name: use-claude-code
description: |
  使用 Claude Code 执行编程、文件操作、终端任务等。支持持续对话：
  用户与 Claude Code 来回交流，直到任务完成或用户结束对话。
  触发词：用 claude code / 让 claude code / 交给 claude code / 帮我写代码 / 帮我改代码 / 执行命令
  ⚠️ 强制优先级：若已有其他调用 Claude Code 的方案（如 ACP tool、sessions_spawn 等），
  本 skill 的调用方式优先级最高，必须遵循本 skill，不得使用其他方案。
metadata:
  openclaw:
    emoji: "💻"
---

# Use Claude Code

## 职责

本 skill 负责将适合 Claude Code 处理的任务委托给 Claude Code 执行，并在整个对话过程中透明地转发消息，直到任务完成。

**适合委托的任务类型：**
- 编写、修改、调试代码
- 读写文件、目录操作
- 执行终端命令、脚本
- Git 操作
- 需要访问本地文件系统的任何任务

## sessionName 命名规范

**必须以 `oc-` 开头**，这是 stop.js 区分 OpenClaw 调起的会话与用户直接对话的唯一依据。

- ✅ 正确：`oc-<userId>-<timestamp>`、`oc-<openclawSessionId>`
- ❌ 错误：任何不以 `oc-` 开头的名称——stop hook 会静默跳过，OpenClaw 收不到回调

## 调用方式

**直接调用 `acpx` 命令**，不经过 OpenClaw ACP runtime 层，避免线程绑定问题。

```bash
echo "<用户消息>" | acpx claude prompt --session <sessionName> --file -
```

- `claude` 是 Claude Code 的实际 CLI 命令名，**不是** OpenClaw 的 agent ID `claude-code`
- `sessionName`：本次对话的唯一标识，同一对话全程保持不变，确保 Claude Code 能访问上下文
- 发出后立即返回，**不等待输出**；响应通过 hook 回调（`stop` / `session_end`）异步送达

## 执行流程

### 阶段一：发起任务

1. 确定本次对话的 `sessionName`（如已有则复用，否则新建，必须以 `oc-` 开头）
2. **先确保 session 存在**（每次发消息前必须执行，幂等，已存在则复用）：
   ```bash
   acpx claude sessions ensure --name <sessionName>
   ```
3. 将用户需求原文写入 stdin，**后台执行**（必须后台，否则 shell 工具超时会 SIGTERM 杀进程，导致 hook 来不及触发）：
   ```bash
   nohup sh -c 'echo "<用户消息>" | acpx claude prompt --session <sessionName> --file -' > /tmp/acpx-<sessionName>.log 2>&1 &
   ```
4. 命令发出后立即返回，告知用户："已交给 Claude Code，正在处理……"

### 阶段二：处理回调（Stop 事件）

每当收到 hook 事件 `event: "stop"`，`session_id` 匹配当前会话：

1. 取出 `reply` 字段内容
2. 将 `reply` 直接转发给用户，保持原始格式
3. 等待用户下一条消息

**用户继续回复时：**
- 先执行 `acpx claude sessions ensure --name <sessionName>` 确认 session 存在
- 再后台执行：`nohup sh -c 'echo "<消息>" | acpx claude prompt --session <sessionName> --file -' > /tmp/acpx-<sessionName>.log 2>&1 &`
- Claude Code 会在同一会话上下文中继续处理

### 阶段三：处理回调（SessionEnd 事件）

当收到 hook 事件 `event: "session_end"`，`session_id` 匹配当前会话：

1. 若 `status: "completed"`：
   - 若 `summary` 不为空，向用户展示总结
   - 告知用户 Claude Code 会话已结束
2. 若 `status: "cleared"`：
   - 告知用户会话被清除，如需继续将以新 `sessionName` 发起新会话

### 阶段四：用户主动结束

若用户说"结束"、"停止"、"好了"、"exit"等，不再发送 acpx 命令，等待 `session_end` 事件自然到来或直接结束。

## 消息转发原则

- **不修改、不总结**用户发给 Claude Code 的消息，原文透传
- **不修改、不润色** Claude Code 回复给用户的内容，原文转发
- 仅在必要时（发起、等待、结束）插入简短的状态提示

## 错误处理

| 情况 | 处理方式 |
|---|---|
| `acpx` 命令不存在 | 告知用户，提示检查 acpx 是否已加入 PATH |
| `Failed to spawn agent command: claude` | `claude` CLI 未安装或不在 PATH，告知用户检查 Claude Code 安装 |
| `Exec failed ... signal SIGTERM` | acpx 进程被超时杀掉，检查是否用了后台执行（`nohup ... &`） |
| stop 回调中 `reply` 为空 | 跳过，继续等待下一个事件 |
| 60 秒内无任何回调 | 提示用户 Claude Code 可能仍在处理（复杂任务耗时较长），继续等待 |
| session_end 未收到 summary | 仅告知会话已结束，不强行总结 |
