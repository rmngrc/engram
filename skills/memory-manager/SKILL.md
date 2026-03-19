---
name: memory-manager
description: Autonomous memory management for Claude Code using an Obsidian vault. Activates when .claude/memory.json exists in the project or when the user invokes /memory-init, /memory-sync, or /memory-optimize.
---

# Memory Manager

Manages a persistent Obsidian vault as long-term memory for any project. Stores decisions, corrections, preferences, patterns, and context across sessions. Retrieves relevant memories on demand to inform current work.

## Prerequisites

- `obsidian-cli` must be installed and available in PATH
- An Obsidian vault directory (agent will prompt to create if needed)

## Config Resolution

1. On first memory operation, read `.claude/memory.json` from the project root
2. If the file doesn't exist, ask the user for the vault path and create the file:
   ```json
   { "vault": "/absolute/path/to/vault" }
   ```
3. If the vault directory doesn't exist, ask the user whether to create it
4. Derive the vault name for obsidian-cli commands: use the vault directory's basename (e.g., `/Users/me/vaults/project-memory` → `vault="project-memory"`)
5. Optionally read `_meta/config.md` from the vault for behavior overrides (see defaults below)

### Default config values

If `_meta/config.md` is absent, use these defaults:

| Setting | Default |
|---------|---------|
| `auto_triggers` | `[correction, decision, preference, pattern]` |
| `suggest_threshold_notes` | `30` |
| `max_note_length` | `500` lines |
| `max_results` | `5` |

## Core Directives

**Lazy loading:** Never read the vault at startup. Only access it when a memory operation is needed.

**Taxonomy-first:** Always read `_meta/taxonomy.md` before accessing any other vault file. This is your map.

**Taxonomy caching:** After reading taxonomy, keep it in conversation context. Invalidate (re-read) after any write that modifies the taxonomy.

**Subagent rule:** If you are running as a subagent (dispatched by another agent), do NOT write to the memory vault. Include memory-worthy observations in your response to the parent agent. You may read from the vault.

**Error handling:** If an obsidian-cli command fails, retry once. If it fails again, inform the user with the error message. Never silently drop a memory.

**Concurrency:** This skill assumes single-agent writes. If two sessions target the same vault, last-write-wins. Use `/memory-sync` to repair inconsistencies.

## Autonomous Write Triggers

The agent writes to the vault without being asked when it detects:

- **Corrections** -- user corrects agent behavior (store the pattern to avoid)
- **Decisions** -- a project/architecture decision is made (store decision + rationale)
- **Preferences** -- user expresses a behavioral preference (store for future sessions)
- **Patterns** -- a recurring code/architecture pattern emerges (store for reference)

See [TRIGGERS.md](references/TRIGGERS.md) for detailed detection heuristics and examples.

## Explicit Write Triggers

The agent also writes when explicitly triggered:

- **User says "remember this"** -- store whatever they specify
- **End of significant task** -- store a summary of what was done and why
- **After optimization command** -- store results of vault reorganization

These don't require detection heuristics -- they're direct instructions or well-defined moments.

**Do NOT store:**
- Things derivable from code or git history
- Ephemeral task state
- Anything already in CLAUDE.md
- Duplicates (update the existing note instead)

## Write Flow

1. Decide something is memory-worthy (autonomous trigger or explicit request)
2. Read `_meta/taxonomy.md` to find where it belongs
3. Create a new note or append to existing (append if same entity/topic exists, new note otherwise)
4. Update `_meta/taxonomy.md` with any structural changes (new notes, new folders, updated descriptions)
5. Use `obsidian-cli` for all vault operations:
   - `obsidian create file="folder/name" content="..." vault="name"` for new notes
   - `obsidian append file="folder/name" content="..." vault="name"` for updates
   - `obsidian read file="path" vault="name"` to check existing content before appending

If obsidian-cli is unavailable, fall back to filesystem tools (Read, Write, Edit).

## Retrieval Protocol

Retrieval is on-demand only. Never preload memories at session start.

1. Read `_meta/taxonomy.md` (or use cached version)
2. Identify folder(s) relevant to the current task from taxonomy structure descriptions
3. Read specific notes (frontmatter first, full content only if needed)
4. If structured lookup finds nothing, fall back to `obsidian search query="..." vault="name"`
5. Extract only the relevant information -- never dump entire notes into context

If obsidian-cli is unavailable for search, fall back to Grep across the vault directory.

See [RETRIEVAL.md](references/RETRIEVAL.md) for detailed lookup strategy and token budget rules.

## Commands

### `/memory-init` -- Guided vault bootstrap

Bootstrap the memory vault from an existing project. Interactive guided process.

See [OPTIMIZATION.md](references/OPTIMIZATION.md) for the full procedure.

### `/memory-sync` -- Incremental update

Sync the vault with current project state. Detects stale memories, missing context, manual edits.

See [OPTIMIZATION.md](references/OPTIMIZATION.md) for the full procedure.

### `/memory-optimize` -- Vault reorganization

Audit and reorganize the vault for efficiency. Merges, prunes, summarizes.

See [OPTIMIZATION.md](references/OPTIMIZATION.md) for the full procedure.

## Vault Structure

No predefined folders. Build the structure organically.

**Rules:**
- New folder only when existing ones don't fit
- Keep top-level folders under ~15; nest subcategories instead
- Merge folders if overlap grows
- Always update taxonomy after structural changes

**`_meta/taxonomy.md` format:**

```markdown
---
last_updated: YYYY-MM-DD
last_synced: YYYY-MM-DD
note_count: N
---

## Structure

- `folder/` - Description of what this folder contains

## Index

- `folder/note.md` - One-line description
```

When the index exceeds ~100 entries, split into per-folder index files (`_meta/index/<folder>.md`) and keep the root taxonomy as folder-level summaries only.

## Note Format

```markdown
---
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: correction | decision | preference | pattern | person | context
project: project-name
tags:
  - relevant-tag
---

Concise content here.
```

**Type definitions:**

| Type | When used |
|------|-----------|
| `correction` | User corrected agent behavior |
| `decision` | Project/architecture decision with rationale |
| `preference` | User behavioral preference |
| `pattern` | Recurring code/architecture pattern |
| `person` | Team member or stakeholder profile |
| `context` | Background info that doesn't fit other types |

**Rules:**
- Frontmatter is mandatory
- `project` field supports multi-project vaults; omit for global memories
- Use `[[wikilinks]]` for cross-referencing between notes
- Keep notes concise -- written for retrieval, not prose
- Short for preferences/corrections, longer for decisions with rationale
