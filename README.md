# engram

Autonomous memory management for AI coding agents using an Obsidian vault. Works with any AI tool that can follow markdown instructions.

## What it does

- **Autonomous memory storage** -- writes corrections, decisions, preferences, and patterns without being asked
- **On-demand retrieval** -- pulls relevant memories to inform decisions and avoid repeating mistakes
- **Vault organization** -- maintains a structured taxonomy; reorganizes as memory grows
- **User commands** -- initialize vaults, sync with project state, optimize vault structure

## Installation

### Claude Code (marketplace)

```
/plugin add rmngrc/engram
```

### Any AI tool (manual)

Clone engram to a shared location:

```bash
git clone https://github.com/rmngrc/engram ~/.engram
```

Then point your AI tool to the skill file at `~/.engram/skills/memory-manager/SKILL.md`. How you do this depends on the tool:

| Tool | Where to reference the skill |
|------|------------------------------|
| **Cursor** | Add an include or reference in `.cursor/rules/` |
| **GitHub Copilot** | Reference from `.github/copilot-instructions.md` |
| **Gemini CLI** | Reference from `GEMINI.md` |
| **Codex** | Reference from `AGENTS.md` |
| **Windsurf** | Reference from `.windsurfrules` |
| **Other** | Include the skill file path in your tool's system prompt or instructions config |

### Roadmap

- `npx engram init` -- automated setup that detects your AI tool and wires up the config

## Setup

Create `.engram/config.json` in your project root:

```json
{
  "vault": "/absolute/path/to/obsidian/vault"
}
```

The agent will prompt to create the vault directory if it doesn't exist.

## Usage

The agent automatically captures corrections, decisions, preferences, and patterns during work. Memories persist across sessions and inform future decisions.

### Commands

- **`/memory-init`** -- bootstrap a new memory vault for the current project
- **`/memory-sync`** -- sync vault with current project state; detect stale memories
- **`/memory-optimize`** -- audit and reorganize the vault (merges, prunes, summarizes)

## Configuration

Create `_meta/config.md` in your vault to override defaults:

```yaml
---
auto_triggers:
  - correction
  - decision
  - preference
  - pattern
suggest_threshold_notes: 30
max_note_length: 500
max_results: 5
---
```

| Setting | Default | Purpose |
|---------|---------|---------|
| `auto_triggers` | `[correction, decision, preference, pattern]` | What the agent writes automatically |
| `suggest_threshold_notes` | `30` | Notes per folder before suggesting `/memory-optimize` |
| `max_note_length` | `500` | Max lines per note before suggesting summarization |
| `max_results` | `5` | Max memories to retrieve per query |

## Requirements

- `obsidian-cli` installed and available in PATH
- Obsidian vault directory (created automatically on first run if needed)
