# engram

Autonomous memory management for Claude Code using an Obsidian vault. Stores decisions, corrections, preferences, and patterns across sessions.

## What it does

- **Autonomous memory storage** – Agent writes corrections, decisions, preferences, and patterns without being asked
- **On-demand retrieval** – Pull relevant memories during work to inform decisions and avoid repeating mistakes
- **Vault organization** – Maintains a structured taxonomy; rebuilds itself as your memory grows
- **User commands** – Initialize vaults, sync with project state, optimize vault structure

## Installation

### Via marketplace

```
/plugin marketplace add rmngrc/engram
/plugin install engram@engram
```

### Manual

Clone into `.claude/`:

```bash
git clone https://github.com/rmngrc/engram ~/.claude/plugins/engram
```

## Setup

Create `.claude/memory.json` in your project root:

```json
{
  "vault": "/absolute/path/to/obsidian/vault"
}
```

The agent will prompt to create the vault directory if it doesn't exist.

## Usage

The agent automatically captures corrections, decisions, preferences, and patterns during work. Memories persist across sessions and inform future decisions.

### Commands

- **`/memory-init`** – Bootstrap a new memory vault for the current project
- **`/memory-sync`** – Sync vault with current project state; detect stale memories
- **`/memory-optimize`** – Audit and reorganize the vault (merges, prunes, summarizes)

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
