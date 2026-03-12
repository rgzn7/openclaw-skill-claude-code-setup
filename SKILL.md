---
name: setup-claude-code
description: |
  配置 Claude Code ↔ OpenClaw 双向集成。自动完成所有配置文件修改、hook 脚本创建、
  acpx 插件安装。用户无需手动改任何配置文件。
  触发词：setup claude code / 配置 claude code / 初始化 claude code 集成
metadata:
  openclaw:
    emoji: "🔧"
---

# Setup Claude Code Integration

## 职责

本 skill 负责自动完成 Claude Code ↔ OpenClaw 双向集成的全部配置，包括：

1. 生成专用 hook token
2. 创建/更新 `~/.claude/hooks/session-end.js`
3. 更新 `~/.claude/settings.json`（hooks、env、permissions）
4. 更新 `~/.openclaw/openclaw.json`（hooks、acp、agents.list）
5. 安装 acpx 插件
6. 提示用户重启 gateway

## 执行步骤

### Step 1：读取当前状态

```
读取 ~/.claude/settings.json
读取 ~/.openclaw/openclaw.json
检查 ~/.claude/hooks/session-end.js 是否存在
```

判断是否已有 `OPENCLAW_HOOK_TOKEN`：
- 若已有，**复用原 token，不重新生成**
- 若没有，生成新 token：`openssl rand -hex 16` 前加前缀 `clhook_`

### Step 2：写入 session-end.js

目标路径：`~/.claude/hooks/session-end.js`

写入以下内容（将 `<TOKEN>` 替换为实际 token）：

```js
#!/usr/bin/env node

const http = require('http');
const fs = require('fs');

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

function extractSummary(transcriptPath) {
  try {
    const lines = fs.readFileSync(transcriptPath, 'utf8').trim().split('\n');
    // 从后往前找最后一条 assistant 文本消息
    for (let i = lines.length - 1; i >= 0; i--) {
      const entry = JSON.parse(lines[i]);
      const msg = entry.message || entry;
      if (msg.role === 'assistant' && Array.isArray(msg.content)) {
        const textBlock = msg.content.find(b => b.type === 'text');
        if (textBlock && textBlock.text) return textBlock.text.trim();
      }
    }
  } catch (_) {}
  return null;
}

async function main() {
  const stdinText = await readStdin();

  let raw = {};
  if (stdinText) {
    try { raw = JSON.parse(stdinText); } catch (_) { raw = { text: stdinText }; }
  }

  const token = process.env.OPENCLAW_HOOK_TOKEN;
  if (!token) {
    console.error('[session-end hook] OPENCLAW_HOOK_TOKEN not set, skipping');
    process.exit(0);
  }

  const transcriptPath = raw.transcript_path || raw.transcriptPath || null;
  const summary = transcriptPath ? extractSummary(transcriptPath) : null;

  const payload = JSON.stringify({
    source: 'claude-code',
    event: 'session_end',
    session_id: raw.session_id || raw.sessionId || null,
    status: raw.reason === 'clear' ? 'cleared' : 'completed',
    timestamp: new Date().toISOString(),
    cwd: process.env.PWD || null,
    summary,
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
    console.error('[session-end hook] request failed:', err.message);
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

### Step 3：更新 ~/.claude/settings.json

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
    "SessionEnd": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node /Users/<username>/.claude/hooks/session-end.js"
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
- `hooks.SessionEnd` 已存在时，检查是否已有 session-end.js 的条目，有则跳过

### Step 4：更新 ~/.openclaw/openclaw.json

需要确保以下配置段存在（不覆盖其他已有字段）：

**tools 段：**
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

**hooks 段：**
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
      "deliver": true
    }
  ]
}
```

**acp 段：**
```json
"acp": {
  "enabled": true,
  "backend": "acpx",
  "defaultAgent": "claude-code",
  "allowedAgents": ["claude-code", "claude"],
  "maxConcurrentSessions": 4
}
```

**agents.list 条目：**
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

**写入原则：**
- 直接读取 JSON、修改内存中的对象、再写回文件
- 写回前备份原文件到 `openclaw.json.bak.setup`
- token 与 `settings.json` 中保持一致

### Step 5：安装 acpx 插件

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

### Step 6：完成提示

输出安装摘要，告知用户：

```
✅ session-end.js 已创建
✅ ~/.claude/settings.json 已更新
✅ ~/.openclaw/openclaw.json 已更新
✅ acpx 插件已安装（或已存在）

⚠️  需要重启 OpenClaw Gateway 使配置生效：
   - QClaw 用户：在 QClaw 应用中重启，或等待自动重启
   - 命令行用户：openclaw gateway restart

🔑 Hook Token：<token>（已写入两侧配置，请勿泄露）
```

## 幂等性要求

- 同一机器重复运行不会破坏已有配置
- Token 不重新生成（复用已有的）
- 已存在的字段不覆盖
- 重复运行的结果与第一次运行相同

## 错误处理

| 情况 | 处理方式 |
|---|---|
| `~/.claude/` 目录不存在 | 创建目录后继续 |
| `settings.json` 不是合法 JSON | 报错，不覆盖，提示用户手动检查 |
| `openclaw.json` 写入失败 | 从备份恢复，报错说明原因 |
| acpx 安装失败 | 报错但不中止，继续完成其余步骤，最后提示手动安装 |
