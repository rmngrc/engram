<p align="center">
  <pre align="center">

  ███████╗███╗   ██╗ ██████╗ ██████╗  █████╗ ███╗   ███╗
  ██╔════╝████╗  ██║██╔════╝ ██╔══██╗██╔══██╗████╗ ████║
  █████╗  ██╔██╗ ██║██║  ███╗██████╔╝███████║██╔████╔██║
  ██╔══╝  ██║╚██╗██║██║   ██║██╔══██╗██╔══██║██║╚██╔╝██║
  ███████╗██║ ╚████║╚██████╔╝██║  ██║██║  ██║██║ ╚═╝ ██║
  ╚══════╝╚═╝  ╚═══╝ ╚═════╝ ╚═╝  ╚═╝╚═╝  ╚═╝╚═╝     ╚═╝

  </pre>
  <strong>Persistent memory for AI coding agents</strong>
  <br />
  <em>Stores what your agent learns. Recalls it when it matters.</em>
  <br /><br />
  <a href="#installation">Install</a> &middot; <a href="#setup">Setup</a> &middot; <a href="#usage">Usage</a> &middot; <a href="#configuration">Config</a>
</p>

---

## How it works

Your AI agent gains long-term memory backed by an Obsidian vault. No code, no runtime -- just markdown instructions that teach any agent to remember.

<table>
<tr>
<td width="50%" valign="top">

### Writes automatically

The agent detects and stores memory-worthy moments:

> **Corrections** -- "Don't mock the DB in tests"
>
> **Decisions** -- "Chose Drizzle over Prisma because..."
>
> **Preferences** -- "User wants terse responses"
>
> **Patterns** -- "All API routes use middleware X"

</td>
<td width="50%" valign="top">

### Retrieves on demand

Before acting, the agent checks what it already knows:

> Starting auth work? Check stored auth decisions.
>
> Person mentioned? Look up their profile.
>
> About to pick a library? See if you already decided.
>
> User asks "remember X"? Search the vault.

</td>
</tr>
</table>

### Organizes itself

The vault grows organically. A taxonomy file acts as the table of contents -- one cheap read gives the agent a full map. When things get messy, `/memory-optimize` cleans house.

---

## Installation

### Claude Code

```
/plugin add rmngrc/engram
```

### Any AI tool

```bash
git clone https://github.com/rmngrc/engram ~/.engram
```

Point your tool to `~/.engram/skills/memory-manager/SKILL.md`:

| Tool | Where to reference |
|------|-------------------|
| Cursor | `.cursor/rules/` |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Gemini CLI | `GEMINI.md` |
| Codex | `AGENTS.md` |
| Windsurf | `.windsurfrules` |
| Other | System prompt or instructions config |

> **Roadmap** -- `npx engram init` for automated setup

---

## Setup

Create `.engram/config.json` in your project root:

```json
{
  "vault": "/absolute/path/to/obsidian/vault"
}
```

The agent will prompt to create the vault directory if it doesn't exist.

---

## Usage

The agent captures memories during normal work. No special commands needed for day-to-day use.

### Commands

| Command | What it does |
|---------|-------------|
| `/memory-init` | Bootstrap a new memory vault from your project |
| `/memory-sync` | Sync vault with current project state |
| `/memory-optimize` | Audit and reorganize (merge, prune, summarize) |

---

## Configuration

Drop `_meta/config.md` in your vault to override defaults:

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

| Setting | Default | What it controls |
|---------|---------|-----------------|
| `auto_triggers` | `[correction, decision, preference, pattern]` | What the agent writes automatically |
| `suggest_threshold_notes` | `30` | Notes per folder before suggesting optimization |
| `max_note_length` | `500` | Lines per note before suggesting summarization |
| `max_results` | `5` | Max memories per retrieval |

---

## Requirements

- [`obsidian-cli`](https://github.com/Yakitrak/obsidian-cli) installed and in PATH
- An Obsidian vault (created automatically on first run)

---

## Why "engram"?

In neuroscience, an **engram** is the physical trace a memory leaves in the brain. The term was coined by German zoologist Richard Semon in 1904, from the Greek *en* (in) + *gramma* (something written) -- literally "something written in."

For decades, engrams were theoretical. Scientists knew memories had to live somewhere in the brain, but couldn't pinpoint where or how. It wasn't until the 2010s that researchers at MIT were able to identify and even reactivate specific engram cells in mice, proving that discrete clusters of neurons really do encode individual memories.

The parallel to this project is direct. AI coding agents are stateless by default -- every session starts from zero, and everything learned in conversation vanishes when the session ends. Engram gives them a physical trace: a structured Obsidian vault where corrections, decisions, preferences, and patterns are written down and retrieved when relevant. Each note in the vault is, quite literally, something written in -- an engram for your agent.
