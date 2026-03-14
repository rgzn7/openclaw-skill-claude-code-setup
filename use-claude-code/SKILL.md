---
name: use-claude-code
description: |
  使用 Claude Code 执行编程、文件操作、终端任务等。支持两种模式：
  (1) 一次性研究模式：Claude Code 完成一轮任务后通过 streamTo=parent 将结果回流到当前 OpenClaw 会话（仅限有 parent session 的上下文，即从 OpenClaw agent 内部发起）；
  (2) 持续协作模式：通过 direct acpx 维持独立 Claude Code 会话，结果通过 stop hook 回流到发起渠道（聊天渠道、agent 均适用）。
  触发词：用 claude code / 让 claude code / 交给 claude code / 帮我写代码 / 帮我改代码 / 执行命令
metadata:
  openclaw:
    emoji: "💻"
---

# Use Claude Code

## 职责

本 skill 负责将适合 Claude Code 处理的任务委托给 Claude Code，并根据任务类型选择正确模式。

**适合委托的任务类型：**
- 编写、修改、调试代码
- 读写文件、目录操作
- 执行终端命令、脚本
- Git 操作
- 需要访问本地文件系统的任何任务
- 需要独立研究或长期协作的本地工程任务

## 核心原则

1. **一次性研究模式** 与 **持续协作模式** 是两种不同模式，不要混用。
2. **两种模式的实现路径和回流机制完全不同**：
   - 研究模式：ACP run + `streamTo=parent`，结果直接回到当前会话（**仅限有 parent session 的上下文**）
   - 持续协作模式：direct acpx（因为聊天渠道不支持线程绑定，ACP session/thread 路不通），结果通过 stop hook 回流
3. **agentId 说明**：ACP run 调用时用 `"claude"`（ACP harness 实际 id），不是 `"claude-code"`（OpenClaw agents.list 包装名）

## 两种推荐模式

### 模式 A：一次性研究模式

适用于：
- "让 Claude Code 研究一下这个问题"
- "让 Claude Code 看完后把结论带回来"
- "用 Claude Code 分析这个 bug，然后告诉我"
- **仅限从 OpenClaw agent 上下文内发起（有 parent session）**

推荐调用：

```json
{
  "runtime": "acp",
  "agentId": "claude",
  "mode": "run",
  "thread": false,
  "streamTo": "parent"
}
```

特点：
- Claude Code 在一次 run-mode ACP 子会话中完成任务
- 进度与结果通过标准 ACP parent-return 回到当前 OpenClaw 会话
- 从聊天渠道直接发起时无 parent session，此模式不适用，应改用持续协作模式

### 模式 B：持续协作模式

适用于：
- "开个 Claude Code 线程长期做"
- "给 Claude Code 一个独立会话继续做这个项目"
- "后面我还要继续跟它聊"
- 从聊天渠道（Telegram、QQBot 等）发起的任何长期协作

实现方式：**direct acpx**（ACP session/thread 因聊天渠道不支持线程绑定而不可用）

```bash
acpx claude sessions ensure --name <sessionName>
echo "<用户消息>" | acpx claude prompt --session <sessionName> --file -
```

特点：
- `sessionName` 由 OpenClaw 维护，代管会话身份，全程保持不变
- 每轮任务完成后，Claude Code 通过 stop hook 把结果推回发起渠道
- **stop hook 是此模式的结果回流主链路**

## sessionName 命名规范

持续协作模式中，sessionName **必须以 `oc-` 开头**，便于在 acpx session 列表中识别 OpenClaw 调起的会话。

- ✅ 正确：`oc-<userId>-<timestamp>`、`oc-<openclawSessionId>`
- ❌ 错误：任何不以 `oc-` 开头的名称

> 注意：stop.js hook 收到的是 Claude Code 内部的 `session_id`（UUID），与 acpx 的 `--session <name>` 是不同字段，无法在 hook 里按 name 过滤。全量回调均会送达 OpenClaw。

## 执行流程

### 阶段一：识别模式

收到用户请求后，先判断属于哪类：

**选择一次性研究模式**（需满足：从 agent 上下文发起，有 parent session）：
- 分析一下 / 看看这个问题 / 跑一轮研究后把结论告诉我

**选择持续协作模式**（默认用于聊天渠道，或需要长期会话时）：
- 开个 Claude Code 会话 / 线程 / 后面我要继续跟它聊 / 让它持续做这个项目

### 阶段二：一次性研究模式执行

1. 用 `sessions_spawn(runtime="acp")` 启动 Claude Code run
2. 带上：
   - `agentId: "claude"`
   - `mode: "run"`
   - `streamTo: "parent"`
3. 告知用户：已交给 Claude Code 处理，结果会回当前会话
4. 等待标准 ACP 回流结果

### 阶段三：持续协作模式执行

#### 发起任务

1. 确定本次对话的 `sessionName`（如已有则复用，否则新建，必须以 `oc-` 开头）
2. **先确保 session 存在**（幂等，已存在则复用）：
   - 当前实测 `acpx claude sessions ensure` **不支持 `--cwd` 参数**
   - 若用户**指定了项目目录** `<projectDir>`，应当先切到该目录再执行：
     ```bash
     cd <projectDir> && acpx claude sessions ensure --name <sessionName>
     ```
   - 否则：
     ```bash
     acpx claude sessions ensure --name <sessionName>
     ```
3. 将用户需求原文写入 stdin，**后台执行**：
   - 若用户**指定了项目目录** `<projectDir>`：
     ```bash
     nohup sh -c 'cd <projectDir> && echo "<用户消息>" | acpx claude prompt --session <sessionName> --file -' > /tmp/acpx-<sessionName>.log 2>&1 &
     ```
   - 否则：
     ```bash
     nohup sh -c 'echo "<用户消息>" | acpx claude prompt --session <sessionName> --file -' > /tmp/acpx-<sessionName>.log 2>&1 &
     ```
4. 命令发出后立即返回，告知用户：已交给 Claude Code，正在处理

#### 处理 hook 回调

当收到 hook 事件时：

**Stop 事件**
1. 取出 `reply` 字段内容
2. 将 `reply` 转发给用户
3. 等待用户下一条消息

**SessionEnd 事件**
1. 若 `status: "completed"`：
   - 若 `summary` 不为空，向用户展示总结
   - 告知用户 Claude Code 会话已结束
2. 若 `status: "cleared"`：
   - 告知用户会话被清除，如需继续将以新 `sessionName` 发起新会话

## 消息转发原则

### 一次性研究模式

- 允许 OpenClaw 用正常助手口吻把 Claude Code 的结果带回或转述给用户
- 不要求逐字原样转发，重点是保持结果语义正确、上下文可继续

### 持续协作模式

- 用户消息原文透传给 Claude Code
- Claude Code 的 reply 尽量原样转发
- 仅在必要时插入简短状态提示

## 错误处理

| 情况 | 处理方式 |
|---|---|
| `acpx` 命令不存在 | 告知用户检查 acpx 是否已加入 PATH（参见 setup-claude-code skill） |
| `Failed to spawn agent command: claude` | `claude` CLI 未安装或不在 PATH，告知用户检查 Claude Code 安装 |
| `Exec failed ... signal SIGTERM` | direct acpx 进程被超时杀掉，检查是否用了后台执行（`nohup ... &`） |
| stop 回调中 `reply` 为空 | 跳过，继续等待下一个事件 |
| 60 秒内无任何回调 | 提示用户 Claude Code 可能仍在处理，继续等待 |
| session_end 未收到 summary | 仅告知会话已结束，不强行总结 |
