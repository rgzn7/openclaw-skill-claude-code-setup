# setup-claude-code

OpenClaw skill — one-command setup for Claude Code ↔ OpenClaw two-way integration.

## What It Does

Once triggered in OpenClaw, it automatically completes all configuration without any manual file editing:

- Generates a dedicated hook token
- Creates `~/.claude/hooks/session-end.js` (called back to OpenClaw when a Claude session ends)
- Updates `~/.claude/settings.json` (registers SessionEnd hook, writes token, enables bypassPermissions)
- Updates `~/.openclaw/openclaw.json` (configures hooks receiver, acp launcher, agents.list)
- Installs the acpx plugin

After setup, OpenClaw can:

- Actively launch Claude Code to execute tasks (via acpx)
- Receive structured callback events when a Claude Code session ends
- Automatically extract Claude Code's last reply as a task summary and relay it to the user via the main agent

## Installation

### Option 1: Clone and add to extraDirs (Recommended)

```bash
git clone https://github.com/<your-username>/setup-claude-code ~/.openclaw/skills/setup-claude-code
```

Add the path to `skills.load.extraDirs` in `~/.openclaw/openclaw.json`:

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

Then restart OpenClaw Gateway.

### Option 2: Manual placement

Copy `SKILL.md` from this repo into any directory, then add that directory to `skills.load.extraDirs`.

## Usage

After installation, say the following in an OpenClaw conversation:

```
setup claude code
```

or:

```
configure claude code integration
```

The skill will automatically execute all steps and output a result summary.

## Prerequisites

- OpenClaw installed and running
- Node.js available (required to run the session-end.js hook script)
- Claude Code CLI installed

## Architecture

```
OpenClaw main agent
      |
      | launch claude-code agent (via acpx)
      v
  Claude Code executes task
      |
      | SessionEnd hook fires
      v
  session-end.js
      |
      | POST /hooks/claude-code
      v
  OpenClaw Gateway
      |
      | hooks.mappings matched
      v
  main agent receives callback
```

## Compatibility

- Supports both QClaw (Electron-wrapped) and standard OpenClaw CLI environments
- Idempotent: repeated runs do not break existing configuration, token is not regenerated
