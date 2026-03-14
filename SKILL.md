---
name: setup-claude-code
description: |
  配置 Claude Code 在 OpenClaw 中的 ACP 集成环境。目标是启用两种能力：
  (1) 持续会话：通过 ACP thread/session 让 Claude Code 保持独立长期会话，结果通过 stop hook 回流到发起渠道；
  (2) 研究回流：通过 ACP run + streamTo=parent 让 Claude Code 在当前会话内完成一次研究并把结果带回（仅适用于有 parent session 的上下文）。
  触发词：setup claude code / 配置 claude code / 初始化 claude code 集成
metadata:
  openclaw:
    emoji: "🔧"
---

# Setup Claude Code Integration

## 职责

本 skill 负责配置 Claude Code 在 OpenClaw 中的 ACP 集成，分为两类能力：

1. 启用 Claude Code 的 ACP runtime（主链路）
2. 确保持久会话可用（`thread: true` + `mode: "session"`，结果通过 hook 回流）
3. 确保研究回流可用（`mode: "run"` + `streamTo: "parent"`，仅限有 parent session 的上下文）
4. 安装并验证 acpx / ACP 相关依赖
5. 配置 Claude Code stop hook（持续会话模式的必需回流链路）
6. 提示用户重启 gateway

**核心原则：**
- **持续会话**的结果回流依赖 stop hook：聊天渠道发起时没有 parent session，必须走 hook 把结果推回发起方
- **研究回流**依赖 `streamTo=parent`：仅在 OpenClaw agent 上下文内调用时有效，聊天渠道不适用
- 两种模式回流机制不同，不要混用
- hook 是持续会话的**主链路**，不是 debug 附件

## Claude Code 的两种推荐模式

### 模式 A：持续会话（独立长期协作）

适用于：
- "开个 Claude Code 线程"
- "让 Claude Code 持续做这个项目"
- "给 Claude Code 一个独立会话继续聊"
- 从聊天渠道（Telegram、QQBot 等）发起的任何长期协作

推荐参数：

```json
{
  "runtime": "acp",
  "agentId": "claude-code",
  "thread": true,
  "mode": "session"
}
```

特点：
- 会创建并绑定独立 thread / 持续 session，OpenClaw 代管 session ID
- 每轮任务完成后，Claude Code 通过 stop hook 把结果推回发起渠道
- **不附带 `streamTo: "parent"`**，聊天渠道无 parent session，加了也无效
- **必须配置 stop hook + openclaw hooks 段**，否则持续会话的结果无法回流

### 模式 B：研究回流（当前会话内完成一轮研究并带回结果）

适用于：
- "让 Claude Code 研究一下这个问题"
- "让 Claude Code 看完后把结论带回来"
- "用 Claude Code 分析一下，然后告诉我结果"
- **仅限从 OpenClaw agent 上下文内发起（有 parent session）**

推荐参数：

```json
{
  "runtime": "acp",
  "agentId": "claude-code",
  "thread": false,
  "mode": "run",
  "streamTo": "parent"
}
```

特点：
- Claude Code 在 child ACP session 中执行一轮任务
- 进度和最终结果通过标准 ACP parent-return 回到当前会话
- **仅适用于有 parent session 的调用上下文**（如 OpenClaw agent-to-agent 调用）
- 从聊天渠道直接发起时无 parent session，此模式结果无法回流，应改用模式 A

## 执行步骤

### Step 1：读取当前状态

优先检查 ACP 主链路是否已经具备：

```text
读取 ~/.openclaw/openclaw.json
检查 acp.enabled
检查 acp.backend
检查 acp.defaultAgent
检查 acp.allowedAgents
检查 agents.list 中是否已存在 claude-code 条目
检查 acpx 插件是否已安装
检查 acpx 是否可执行（acpx --version）
```

然后再读取：

```text
读取 ~/.claude/settings.json
检查 ~/.claude/hooks/stop.js 是否存在
检查 settings.json 中是否已有 OPENCLAW_HOOK_TOKEN
```

判断是否已有 `OPENCLAW_HOOK_TOKEN`：
- 若已有，**复用原 token，不重新生成**
- 若没有，生成新 token：`openssl rand -hex 16` 前加前缀 `clhook_`

### Step 2：更新 ~/.openclaw/openclaw.json（主配置）

需要确保以下配置段存在（不覆盖其他已有字段）：

#### tools 段

```json
"tools": {
  "sessions": {
    "visibility": "all"
  },
  "agentToAgent": {
    "enabled": true
  }
}
```

#### acp 段

```json
"acp": {
  "enabled": true,
  "backend": "acpx",
  "defaultAgent": "claude-code",
  "allowedAgents": ["claude-code", "claude"],
  "maxConcurrentSessions": 4
}
```

#### agents.list 条目

在 `agents.list` 数组中添加（若 id 为 `claude-code` 的条目已存在则跳过）：

```json
{
  "id": "claude-code",
  "runtime": {
    "type": "acp",
    "acp": {
      "agent": "claude",
      "backend": "acpx",
      "mode": "persistent"
    }
  }
}
```

#### hooks 段（持续会话必需）

确保以下配置存在（不覆盖已有字段）：

```json
"hooks": {
  "enabled": true,
  "token": "<生成的token>",
  "path": "/hooks",
  "defaultSessionKey": "hook:claude-code",
  "allowRequestSessionKey": false,
  "allowedSessionKeyPrefixes": ["hook:"],
  "mappings": [
    {
      "match": {
        "path": "claude-code",
        "source": "claude-code"
      },
      "action": "agent",
      "agentId": "main",
      "messageTemplate": "{{message}}",
      "deliver": true
    }
  ]
}
```

**说明：**
- 这是持续会话模式结果回流的主链路，不是 debug 附件
- Claude Code 每轮完成后通过 stop hook POST 到 `/hooks/claude-code`，OpenClaw 再把结果投递到目标 agent

#### 写入原则

- 直接读取 JSON、修改内存中的对象、再写回文件
- 写回前备份原文件到 `openclaw.json.bak.setup`
- hook token 要与 `settings.json` 中保持一致

### Step 3：安装 acpx 插件

检查 `openclaw.json` 的 `plugins.entries.acpx` 是否已存在且 `enabled: true`：
- 若已安装，跳过
- 若未安装，执行安装

**安装方式（按优先级尝试）：**

1. 若存在 `qclaw-openclaw` skill（QClaw 环境），使用其提供的 wrapper 脚本：
   ```bash
   bash <qclaw-openclaw-skill-dir>/scripts/openclaw-mac.sh plugins install acpx
   ```

2. 若全局 `openclaw` CLI 可用：
   ```bash
   openclaw plugins install acpx
   ```

3. 若两者都不可用，告知用户手动执行安装命令

### Step 3b：将 acpx 加入全局 PATH

检查 `acpx` 是否已在全局 PATH 中（`which acpx`）：
- 若已存在且可用（`acpx --version` 正常返回），跳过
- 若不存在或报错，创建包装脚本

**注意：不能用软链接。** acpx 的启动脚本用 `$basedir` 推算 `cli.js` 路径，软链接会导致 basedir 指向错误目录，报 `Cannot find module`。必须用包装脚本写死绝对路径。

acpx 实际路径从 `openclaw.json` 的 `plugins.load.paths` 中找到包含 `acpx` 的 extensions 目录，然后：
- cli.js：`<extensionsDir>/acpx/node_modules/acpx/dist/cli.js`
- NODE_PATH：`<openclawDir>/node_modules/.pnpm/acpx@<version>_<hash>/node_modules/acpx/dist/node_modules:<openclawDir>/node_modules/.pnpm/acpx@<version>_<hash>/node_modules/acpx/node_modules:<openclawDir>/node_modules/.pnpm/acpx@<version>_<hash>/node_modules:<openclawDir>/node_modules/.pnpm/node_modules`

**按优先级尝试写入包装脚本：**

1. `/opt/homebrew/bin/acpx`（macOS Apple Silicon）
2. `/usr/local/bin/acpx`（macOS Intel）

写入内容（将路径替换为实际值）：

```sh
#!/bin/sh
export NODE_PATH="<node_path>"
exec node <extensionsDir>/acpx/node_modules/acpx/dist/cli.js "$@"
```

```bash
chmod +x /opt/homebrew/bin/acpx
```

写入后执行 `acpx --version` 验证，若正常返回版本号则成功。若两个目录都无写权限，告知用户手动执行。

### Step 4：更新 ~/.claude/settings.json

需要确保以下字段存在（不覆盖其他已有字段，使用 JSON merge 方式）：

```json
{
  "env": {
    "OPENCLAW_HOOK_TOKEN": "<生成的token>"
  },
  "permissions": {
    "defaultMode": "bypassPermissions"
  },
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node /Users/<username>/.claude/hooks/stop.js"
          }
        ]
      }
    ]
  }
}
```

**注意：**
- `command` 路径中的用户名从 `$HOME` 环境变量或 `~` 展开获取
- 若 `env`、`permissions`、`hooks` 已存在，只补充缺少的字段，不覆盖已有内容

### Step 5：写入 ~/.claude/hooks/stop.js

目标路径：`~/.claude/hooks/stop.js`

写入以下内容：

```js
#!/usr/bin/env node

const http = require('http');

function readStdin() {
  return new Promise((resolve) => {
    let data = '';
    process.stdin.setEncoding('utf8');
    process.stdin.on('data', (chunk) => { data += chunk; });
    process.stdin.on('end', () => resolve(data.trim()));
    process.stdin.on('error', () => resolve(''));
    setTimeout(() => resolve(data.trim()), 100);
  });
}

async function main() {
  const stdinText = await readStdin();

  let raw = {};
  if (stdinText) {
    try { raw = JSON.parse(stdinText); } catch (_) { raw = { text: stdinText }; }
  }

  const token = process.env.OPENCLAW_HOOK_TOKEN;
  if (!token) {
    console.error('[stop hook] OPENCLAW_HOOK_TOKEN not set, skipping');
    process.exit(0);
  }

  const reply = raw.last_assistant_message || null;
  if (!reply) {
    process.exit(0);
  }

  const payload = JSON.stringify({
    source: 'claude-code',
    event: 'stop',
    session_id: raw.session_id || null,
    timestamp: new Date().toISOString(),
    cwd: raw.cwd || null,
    message: reply,
    reply,
    raw,
  });

  const options = {
    hostname: 'localhost',
    port: 18789,
    path: '/hooks/claude-code',
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Content-Length': Buffer.byteLength(payload),
      'Authorization': `Bearer ${token}`,
    },
  };

  const req = http.request(options, (res) => {
    res.resume();
    res.on('end', () => process.exit(0));
  });

  req.on('error', (err) => {
    console.error('[stop hook] request failed:', err.message);
    process.exit(0);
  });

  req.setTimeout(5000, () => {
    req.destroy();
    process.exit(0);
  });

  req.write(payload);
  req.end();
}

main();
```

### Step 6：完成提示

输出安装摘要，告知用户：

```text
✅ ACP 已启用
✅ Claude Code agent 已注册到 agents.list
✅ acpx 插件已安装（或已存在）
✅ acpx 已加入全局 PATH（或已存在）
✅ stop.js 已创建
✅ ~/.claude/settings.json 已更新（env + hooks.Stop）
✅ ~/.openclaw/openclaw.json 已更新（hooks 段）
✅ 持续会话模式可用（thread + session，结果通过 hook 回流）
✅ 研究回流模式可用（run + streamTo=parent，需有 parent session）

⚠️ 需要重启 OpenClaw Gateway 使配置生效：
   - QClaw 用户：在 QClaw 应用中重启，或等待自动重启
   - 命令行用户：openclaw gateway restart

🔑 Hook Token：<token>（已写入两侧配置，请勿泄露）
```

## 幂等性要求

- 同一机器重复运行不会破坏已有配置
- Token 不重新生成（复用已有的）
- 已存在的字段不覆盖
- 重复运行的结果与第一次运行相同
- 不因重复执行而把持续会话模式错误改成研究回流模式，或反之

## 错误处理

| 情况 | 处理方式 |
|---|---|
| `~/.claude/` 目录不存在 | 创建目录后继续 |
| `settings.json` 不是合法 JSON | 报错，不覆盖，提示用户手动检查 |
| `openclaw.json` 写入失败 | 从备份恢复，报错说明原因 |
| acpx 安装失败 | 报错但不中止，继续完成其余步骤，最后提示手动安装 |
| stop hook 写入失败 | 报错，持续会话模式回流不可用，提示用户手动检查 |
