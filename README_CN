# setup-claude-code

OpenClaw skill — 一键配置 Claude Code ↔ OpenClaw 双向集成。

## 做什么

在 OpenClaw 里触发后，自动完成所有配置，无需手动改任何文件：

- 生成专用 hook token
- 创建 `~/.claude/hooks/session-end.js`（Claude 会话结束时回调 OpenClaw）
- 更新 `~/.claude/settings.json`（注册 SessionEnd hook、写入 token、开启 bypassPermissions）
- 更新 `~/.openclaw/openclaw.json`（配置 hooks 接收端、acp 调起端、agents.list）
- 安装 acpx 插件

完成后 OpenClaw 可以：

- 主动调起 Claude Code 执行任务（通过 acpx）
- 在 Claude Code 会话结束时收到结构化回调事件

## 安装

### 方式一：clone 后加到 extraDirs（推荐）

```bash
git clone https://github.com/<your-username>/setup-claude-code ~/.openclaw/skills/setup-claude-code
```

在 `~/.openclaw/openclaw.json` 的 `skills.load.extraDirs` 加入：

```json
"skills": {
  "load": {
    "extraDirs": [
      "/path/to/existing/skills",
      "/Users/<you>/.openclaw/skills"
    ]
  }
}
```

然后重启 OpenClaw Gateway。

### 方式二：手动放置

把本仓库的 `SKILL.md` 放到任意目录，将该目录加入 `skills.load.extraDirs`。

## 使用

安装完成后，在 OpenClaw 对话里说：

```
setup claude code
```

或：

```
配置 claude code 集成
```

skill 会自动执行所有步骤并输出结果摘要。

## 前提条件

- OpenClaw 已安装并运行
- Node.js 可用（用于执行 session-end.js hook 脚本）
- Claude Code CLI 已安装

## 架构

```
OpenClaw main agent
      │
      │ 调起 claude-code agent（via acpx）
      ▼
  Claude Code 执行任务
      │
      │ SessionEnd hook 触发
      ▼
  session-end.js
      │
      │ POST /hooks/claude-code
      ▼
  OpenClaw Gateway
      │
      │ hooks.mappings 匹配
      ▼
  main agent 收到回调
```

## 兼容性

- 支持 QClaw（Electron 封装版）和标准 OpenClaw CLI 环境
- 幂等：重复运行不破坏已有配置，token 不重新生成
