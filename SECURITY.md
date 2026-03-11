# Security / 安全说明

---

## English

### What This Skill Changes

Running this skill modifies your local Claude Code configuration in the following ways:

- **`permissions.defaultMode: "bypassPermissions"`** is written into `~/.claude/settings.json`
- A `SessionEnd` hook script is created at `~/.claude/hooks/session-end.js`
- A shared hook token is stored in both `~/.claude/settings.json` and `~/.openclaw/openclaw.json`

### What `bypassPermissions` Means

With `bypassPermissions` enabled, Claude Code will **skip all user confirmation prompts** for every tool call, including:

- Reading and writing arbitrary files on your filesystem
- Executing shell commands
- Making network requests
- Modifying other configuration files

This is intentional for unattended automation: when OpenClaw launches Claude Code as a background agent, there is no user present to approve each action interactively.

### Risks

- Any task dispatched to Claude Code via OpenClaw will execute **without per-action confirmation**.
- If the OpenClaw agent or its prompt is compromised, it could instruct Claude Code to perform destructive or unintended actions silently.
- The hook token grants OpenClaw Gateway the ability to receive session-end callbacks. Keep it confidential.

### Recommended Usage

- Use only in **trusted local environments** where you control both OpenClaw and Claude Code.
- Do not use on shared machines or in environments where other users or processes can influence the tasks sent to Claude Code.
- Review the tasks OpenClaw dispatches to Claude Code before enabling full automation.

### How to Revert

To restore per-action confirmation prompts, open `~/.claude/settings.json` and set:

```json
"permissions": {
  "defaultMode": "default"
}
```

Or remove the `permissions` block entirely. Claude Code will then resume asking for confirmation on sensitive tool calls.

To fully remove the integration:

1. Delete `~/.claude/hooks/session-end.js`
2. Remove the `SessionEnd` entry from `hooks` in `~/.claude/settings.json`
3. Remove the `hooks` and `acp` sections added to `~/.openclaw/openclaw.json`

---

## 中文

### 此 Skill 修改了哪些内容

运行此 skill 会对本地 Claude Code 配置做以下修改：

- 在 `~/.claude/settings.json` 中写入 **`permissions.defaultMode: "bypassPermissions"`**
- 在 `~/.claude/hooks/session-end.js` 创建 SessionEnd hook 脚本
- 在 `~/.claude/settings.json` 和 `~/.openclaw/openclaw.json` 中各存储一份共享 hook token

### `bypassPermissions` 的含义

开启 `bypassPermissions` 后，Claude Code 对每一个工具调用都会**跳过用户确认弹窗**，包括：

- 读写本地文件系统中的任意文件
- 执行 Shell 命令
- 发起网络请求
- 修改其他配置文件

这是无人值守自动化场景的必要设置：当 OpenClaw 以后台 agent 方式调起 Claude Code 时，没有用户在场逐一审批操作。

### 风险说明

- 通过 OpenClaw 下发给 Claude Code 的任何任务都会**不经逐步确认**直接执行。
- 若 OpenClaw agent 或其 prompt 被篡改，可能在用户无感知的情况下指示 Claude Code 执行破坏性或非预期的操作。
- hook token 允许 OpenClaw Gateway 接收会话结束回调，请妥善保管，不要泄露。

### 适用场景建议

- 仅在**受信任的本地环境**中使用，且你同时控制 OpenClaw 和 Claude Code 两侧。
- 不要在共享机器或其他用户/进程可影响任务输入的环境中使用。
- 在开启完全自动化前，先审查 OpenClaw 会向 Claude Code 下发哪些任务。

### 如何撤销

若需恢复逐步确认提示，打开 `~/.claude/settings.json`，将以下字段改为：

```json
"permissions": {
  "defaultMode": "default"
}
```

或直接删除 `permissions` 块。Claude Code 随后会恢复对敏感工具调用的确认询问。

若需彻底移除集成：

1. 删除 `~/.claude/hooks/session-end.js`
2. 从 `~/.claude/settings.json` 的 `hooks` 中移除 `SessionEnd` 条目
3. 从 `~/.openclaw/openclaw.json` 中移除 skill 添加的 `hooks` 和 `acp` 相关字段
