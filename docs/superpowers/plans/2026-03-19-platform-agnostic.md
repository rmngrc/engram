# Platform-Agnostic Engram Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make engram platform-agnostic so any AI coding tool can use it, not just Claude Code.

**Architecture:** Genericize all skill files to remove Claude Code-specific tool names, paths, and concepts. Change config path from `.claude/memory.json` to `.engram/config.json`. Add platform adapter docs for manual installation. Keep `.claude-plugin/` for Claude Code marketplace distribution.

**Tech Stack:** Markdown skill files, obsidian-cli, Obsidian vault (filesystem)

**Spec:** Designed during brainstorming session 2026-03-19.

---

## File Structure

```
engram/
  .claude-plugin/
    plugin.json                         # Claude Code marketplace metadata (updated description)
    marketplace.json                    # Claude Code marketplace listing (unchanged)
  skills/
    memory-manager/
      SKILL.md                          # Core skill (genericized)
      references/
        TRIGGERS.md                     # Autonomous write heuristics (genericized)
        RETRIEVAL.md                    # Lookup strategy (genericized)
        OPTIMIZATION.md                 # Command procedures (genericized)
  README.md                            # Updated: multi-platform install docs
  docs/
    superpowers/
      specs/
        2026-03-19-engram-design.md
      plans/
        2026-03-19-engram.md
        2026-03-19-platform-agnostic.md # (this file)
```

No new files created. All changes are edits to existing files.

---

## Changes Summary

### What changes across all skill files

| Claude Code-specific | Generic replacement |
|---|---|
| `Read`, `Write`, `Edit` tools | "read the file", "create/update the file", "edit the file" |
| `Grep` / `Glob` tools | "search file contents", "find files by pattern" |
| `Bash` tool | "run in the terminal" |
| `.claude/memory.json` | `.engram/config.json` |
| `CLAUDE.md` / `~/.claude/CLAUDE.md` | "project instructions file (e.g. CLAUDE.md, .cursorrules, AGENTS.md, GEMINI.md)" |
| "subagent (dispatched by another agent)" | "sub-agent or delegated task (dispatched by a parent agent)" |
| "conversation context" (for caching) | "session context" |
| `.claude/` directory | `.engram/` directory |

### What stays the same

- obsidian-cli commands (platform-independent)
- Vault structure, taxonomy, note format
- Write triggers and detection heuristics
- Retrieval strategy logic
- Command procedures (`/memory-init`, `/memory-sync`, `/memory-optimize`)
- `.claude-plugin/` directory (Claude Code distribution format, untouched except description)

---

### Task 1: Genericize SKILL.md

**Files:**
- Modify: `skills/memory-manager/SKILL.md`

- [ ] **Step 1: Update frontmatter description**

Change:
```
description: Autonomous memory management for Claude Code using an Obsidian vault. Activates when .claude/memory.json exists in the project or when the user invokes /memory-init, /memory-sync, or /memory-optimize.
```
To:
```
description: Autonomous memory management using an Obsidian vault. Activates when .engram/config.json exists in the project or when the user invokes /memory-init, /memory-sync, or /memory-optimize.
```

- [ ] **Step 2: Update overview paragraph**

Change "Manages a persistent Obsidian vault as long-term memory for any project." -- this is already generic, keep it.

- [ ] **Step 3: Update Config Resolution section**

Replace all `.claude/memory.json` references with `.engram/config.json`. The JSON structure stays the same:
```json
{ "vault": "/absolute/path/to/vault" }
```

- [ ] **Step 4: Update Core Directives**

Change subagent rule from:
> If you are running as a subagent (dispatched by another agent), do NOT write to the memory vault. Include memory-worthy observations in your response to the parent agent.

To:
> If you are running as a sub-agent or delegated task (dispatched by a parent agent), do NOT write to the memory vault. Include memory-worthy observations in your response to the parent agent.

- [ ] **Step 5: Update "Do NOT store" list**

Change:
> Anything already in CLAUDE.md

To:
> Anything already in the project instructions file (CLAUDE.md, .cursorrules, AGENTS.md, etc.)

- [ ] **Step 6: Update Write Flow fallback**

Change:
> If obsidian-cli is unavailable, fall back to filesystem tools (Read, Write, Edit).

To:
> If obsidian-cli is unavailable, fall back to filesystem operations (read, create, and edit files directly).

- [ ] **Step 7: Update Retrieval Protocol fallback**

Change:
> If obsidian-cli is unavailable for search, fall back to Grep across the vault directory.

To:
> If obsidian-cli is unavailable for search, fall back to searching file contents across the vault directory.

- [ ] **Step 8: Verify no remaining Claude Code-specific references**

Search `SKILL.md` for: `Claude Code`, `.claude/`, `Read,`, `Write,`, `Edit,`, `Grep`, `Glob`, `Bash`, `CLAUDE.md` (without surrounding generic text).

- [ ] **Step 9: Commit**

```bash
git add skills/memory-manager/SKILL.md
git commit -m "refactor: genericize SKILL.md for platform-agnostic use"
```

---

### Task 2: Genericize RETRIEVAL.md

**Files:**
- Modify: `skills/memory-manager/references/RETRIEVAL.md`

- [ ] **Step 1: Update Step 1 fallback**

Change:
> If obsidian-cli is unavailable, use the filesystem Read tool on the vault directory.

To:
> If obsidian-cli is unavailable, read the file directly from the vault directory.

- [ ] **Step 2: Update Step 5 fallback**

Change:
> If obsidian-cli is unavailable, use the Grep tool across the vault directory with relevant terms as the pattern.

To:
> If obsidian-cli is unavailable, search file contents across the vault directory using relevant terms as the pattern.

- [ ] **Step 3: Update Multi-Project Filtering section**

Change `.claude/memory.json` reference to `.engram/config.json`.

- [ ] **Step 4: Verify no remaining Claude Code-specific references**

Search for: `Claude`, `.claude`, `Read tool`, `Grep tool`, `Write tool`.

- [ ] **Step 5: Commit**

```bash
git add skills/memory-manager/references/RETRIEVAL.md
git commit -m "refactor: genericize RETRIEVAL.md for platform-agnostic use"
```

---

### Task 3: Genericize OPTIMIZATION.md

**Files:**
- Modify: `skills/memory-manager/references/OPTIMIZATION.md`

- [ ] **Step 1: Update /memory-init Step 1**

Replace `.claude/memory.json` with `.engram/config.json`. Update the creation instruction:
> Create `.engram/config.json` with `{ "vault": "/absolute/path" }`.

- [ ] **Step 2: Update /memory-init Step 5 -- User preferences category**

Change:
> Read `CLAUDE.md` at the project root and at `~/.claude/CLAUDE.md` if accessible

To:
> Read the project instructions file if present (e.g. `CLAUDE.md`, `.cursorrules`, `AGENTS.md`, `GEMINI.md`, `.github/copilot-instructions.md`)

Change:
> Check for any `.claude/` directory contents beyond `memory.json`

To:
> Check for any `.engram/` directory contents beyond `config.json`

- [ ] **Step 3: Update /memory-init Step 5 -- Conventions category**

Change:
> Read `CLAUDE.md` or `.claude/CLAUDE.md` if present

To:
> Read project instructions file if present (e.g. `CLAUDE.md`, `.cursorrules`, `AGENTS.md`)

- [ ] **Step 4: Update /memory-sync Step 4**

Change:
> Read current `CLAUDE.md`, `.editorconfig`, and linting configs.

To:
> Read the current project instructions file (e.g. `CLAUDE.md`, `.cursorrules`, `AGENTS.md`), `.editorconfig`, and linting configs.

- [ ] **Step 5: Update /memory-optimize Step 5**

Change:
> Use filesystem tools (Read, Write, Edit, Bash with `mv` / `rm`)

To:
> Use filesystem operations (read, write, edit files; use `mv` / `rm` in the terminal)

- [ ] **Step 6: Verify no remaining Claude Code-specific references**

Search for: `Claude`, `.claude`, `Read,`, `Write,`, `Edit,`, `Grep`, `Glob`, `Bash tool`.

- [ ] **Step 7: Commit**

```bash
git add skills/memory-manager/references/OPTIMIZATION.md
git commit -m "refactor: genericize OPTIMIZATION.md for platform-agnostic use"
```

---

### Task 4: Genericize TRIGGERS.md

**Files:**
- Modify: `skills/memory-manager/references/TRIGGERS.md`

- [ ] **Step 1: Scan for Claude Code-specific references**

TRIGGERS.md is mostly about detection heuristics and is likely already generic. Scan for: `Claude`, `.claude`, tool names, `CLAUDE.md`, `subagent`.

- [ ] **Step 2: Apply any needed changes**

Based on scan results. Expected: minimal or no changes since TRIGGERS.md focuses on human behavioral signals.

- [ ] **Step 3: Commit (if changes made)**

```bash
git add skills/memory-manager/references/TRIGGERS.md
git commit -m "refactor: genericize TRIGGERS.md for platform-agnostic use"
```

---

### Task 5: Update plugin.json description

**Files:**
- Modify: `.claude-plugin/plugin.json`

- [ ] **Step 1: Update description field**

Change:
```
"description": "Autonomous memory management for Claude Code using an Obsidian vault. Use when working on any project that has a .claude/memory.json config file or when the user invokes /memory-init, /memory-sync, or /memory-optimize."
```
To:
```
"description": "Autonomous memory management using an Obsidian vault. Use when working on any project that has a .engram/config.json config file or when the user invokes /memory-init, /memory-sync, or /memory-optimize."
```

- [ ] **Step 2: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "refactor: update plugin.json description for platform-agnostic config path"
```

---

### Task 6: Rewrite README.md

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Rewrite README with multi-platform install docs**

New content structure:

```markdown
# engram

Autonomous memory management for AI coding agents using an Obsidian vault. Works with any AI tool that can follow markdown instructions.

## What it does

- **Autonomous memory storage** -- writes corrections, decisions, preferences, and patterns without being asked
- **On-demand retrieval** -- pulls relevant memories to inform decisions and avoid repeating mistakes
- **Vault organization** -- maintains a structured taxonomy; reorganizes as memory grows
- **User commands** -- initialize vaults, sync with project state, optimize vault structure

## Installation

### Claude Code (marketplace)

\```
/plugin add rmngrc/engram
\```

### Any AI tool (manual)

Clone engram to a shared location:

\```bash
git clone https://github.com/rmngrc/engram ~/.engram
\```

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

\```json
{
  "vault": "/absolute/path/to/obsidian/vault"
}
\```

The agent will prompt to create the vault directory if it doesn't exist.

## Usage

The agent automatically captures corrections, decisions, preferences, and patterns during work. Memories persist across sessions and inform future decisions.

### Commands

- **`/memory-init`** -- bootstrap a new memory vault for the current project
- **`/memory-sync`** -- sync vault with current project state; detect stale memories
- **`/memory-optimize`** -- audit and reorganize the vault (merges, prunes, summarizes)

## Configuration

Create `_meta/config.md` in your vault to override defaults:

\```yaml
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
\```

| Setting | Default | Purpose |
|---------|---------|---------|
| `auto_triggers` | `[correction, decision, preference, pattern]` | What the agent writes automatically |
| `suggest_threshold_notes` | `30` | Notes per folder before suggesting `/memory-optimize` |
| `max_note_length` | `500` | Max lines per note before suggesting summarization |
| `max_results` | `5` | Max memories to retrieve per query |

## Requirements

- `obsidian-cli` installed and available in PATH
- Obsidian vault directory (created automatically on first run if needed)
```

Note: the `\` before triple backticks above is an escape for this plan doc. The actual README should use normal triple backticks.

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: rewrite README for multi-platform installation"
```

---

### Task 7: Update design spec

**Files:**
- Modify: `docs/superpowers/specs/2026-03-19-engram-design.md`

- [ ] **Step 1: Update config path references**

Replace `.claude/memory.json` with `.engram/config.json` throughout.

- [ ] **Step 2: Update any Claude Code-specific language**

Replace `CLAUDE.md` references with generic "project instructions file" language. Replace tool name references with generic descriptions.

- [ ] **Step 3: Commit**

```bash
git add docs/superpowers/specs/2026-03-19-engram-design.md
git commit -m "docs: update design spec for platform-agnostic config"
```

---

### Task 8: Verification

- [ ] **Step 1: Search entire repo for remaining Claude Code-specific references**

Search for each of these patterns across all files (excluding `.git/` and this plan file):
- `.claude/memory` (old config path)
- `Claude Code` (product name in instructions)
- `Read,` / `Write,` / `Edit,` / `Grep` / `Glob` as tool names (not as generic verbs)
- `CLAUDE.md` without surrounding generic alternatives

Expected: no matches in skill files, README, or design spec. Only matches in `.claude-plugin/` (expected, it's the Claude Code adapter) and plan/historical docs.

- [ ] **Step 2: Verify .engram/config.json is consistently referenced**

Search for `.engram/config.json` -- should appear in SKILL.md, OPTIMIZATION.md, README.md.

- [ ] **Step 3: Verify skill files parse correctly**

Check that SKILL.md frontmatter is intact:
```bash
head -4 skills/memory-manager/SKILL.md
```

- [ ] **Step 4: Final commit if any fixes needed**

```bash
git status
```

Expected: clean working tree.
