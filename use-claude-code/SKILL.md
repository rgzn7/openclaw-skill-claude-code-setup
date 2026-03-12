---
name: use-claude-code
description: |
  使用 Claude Code 执行编程、文件操作、终端任务等。支持持续对话：
  用户与 Claude Code 来回交流，直到任务完成或用户结束对话。
  触发词：用 claude code / 让 claude code / 交给 claude code / 帮我写代码 / 帮我改代码 / 执行命令
metadata:
  openclaw:
    emoji: "💻"
---

# Use Claude Code

## 职责

本 skill 负责将适合 Claude Code 处理的任务委托给 claude-code agent 执行，并在整个对话过程中透明地转发消息，直到任务完成。

**适合委托的任务类型：**
- 编写、修改、调试代码
- 读写文件、目录操作
- 执行终端命令、脚本
- Git 操作
- 需要访问本地文件系统的任何任务

## 执行流程

### 阶段一：发起任务

1. 将用户的原始需求原文转发给 claude-code agent（通过 acp 工具，agent: `claude-code`）
2. 记录本次调起的 `session_id`（从 acp 调起结果或首次 stop 回调中获取）
3. 告知用户："已交给 Claude Code，正在处理……"

### 阶段二：处理回调（Stop 事件）

每当收到 hook 事件 `event: "stop"`：

1. 取出 `reply` 字段内容
2. 将 `reply` 直接转发给用户，保持原始格式
3. 等待用户下一条消息

**用户继续回复时：**
- 将用户消息发送到**同一个** session（使用记录的 `session_id`）
- 继续等待下一个 stop 事件

### 阶段三：处理回调（SessionEnd 事件）

当收到 hook 事件 `event: "session_end"`：

1. 若 `status: "completed"`：
   - 若 `summary` 不为空，向用户展示总结
   - 告知用户 Claude Code 会话已结束
2. 若 `status: "cleared"`：
   - 告知用户会话被清除，如需继续将发起新会话

### 阶段四：用户主动结束

若用户说"结束"、"停止"、"好了"、"exit"等，不再向 claude-code agent 发送消息，等待 session_end 事件自然到来或直接结束。

## 消息转发原则

- **不修改、不总结**用户发给 Claude Code 的消息，原文透传
- **不修改、不润色** Claude Code 回复给用户的内容，原文转发
- 仅在必要时（发起、等待、结束）插入简短的状态提示

## 错误处理

| 情况 | 处理方式 |
|---|---|
| acp 调起失败 | 告知用户，提示检查 acpx 插件和 Gateway 状态 |
| stop 回调中 `reply` 为空 | 跳过，继续等待下一个事件 |
| 30 秒内无任何回调 | 提示用户 Claude Code 可能仍在处理，继续等待 |
| session_end 未收到 summary | 仅告知会话已结束，不强行总结 |
