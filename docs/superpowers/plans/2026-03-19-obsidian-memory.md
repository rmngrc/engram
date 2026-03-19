# obsidian-memory Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a pure-skill Claude Code plugin that uses obsidian-cli to autonomously manage an Obsidian vault as project memory.

**Architecture:** A set of SKILL.md instruction files (no code) that teach Claude Code when/how to read and write memories in an Obsidian vault. The vault structure is agent-built and tracked via a taxonomy file. Three user commands (`/memory-init`, `/memory-sync`, `/memory-optimize`) provide explicit control.

**Tech Stack:** Markdown skill files, obsidian-cli, Obsidian vault (filesystem)

**Spec:** `docs/superpowers/specs/2026-03-19-obsidian-memory-design.md`

---

## File Structure

```
obsidian-memory/
  .claude-plugin/
    plugin.json                         # Plugin metadata
    marketplace.json                    # Marketplace listing
  skills/
    memory-manager/
      SKILL.md                          # Core skill: activation, config, directives, commands
      references/
        TRIGGERS.md                     # Autonomous write trigger rules and heuristics
        RETRIEVAL.md                    # Hybrid lookup strategy and token budget
        OPTIMIZATION.md                 # /memory-optimize and /memory-sync procedures
  README.md                            # User-facing docs: install, configure, usage
  docs/
    superpowers/
      specs/
        2026-03-19-obsidian-memory-design.md   # (already exists)
      plans/
        2026-03-19-obsidian-memory.md          # (this file)
```

---

### Task 1: Plugin Scaffolding

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Create plugin.json**

```json
{
  "name": "obsidian-memory",
  "version": "0.1.0",
  "description": "Autonomous memory management for Claude Code using an Obsidian vault. Use when working on any project that has a .claude/memory.json config file or when the user invokes /memory-init, /memory-sync, or /memory-optimize.",
  "author": {
    "name": "rmngrc",
    "url": "https://github.com/rmngrc"
  },
  "repository": "https://github.com/rmngrc/obsidian-memory",
  "license": "MIT",
  "keywords": [
    "memory",
    "obsidian",
    "knowledge-management",
    "agent-memory",
    "autonomous"
  ]
}
```

- [ ] **Step 2: Create marketplace.json**

```json
{
  "name": "obsidian-memory",
  "owner": {
    "name": "rmngrc",
    "url": "https://github.com/rmngrc"
  },
  "plugins": [
    {
      "name": "obsidian-memory",
      "source": "./",
      "description": "Autonomous memory management using Obsidian vaults",
      "version": "0.1.0"
    }
  ]
}
```

- [ ] **Step 3: Verify structure**

Run: `find .claude-plugin -type f | sort`
Expected:
```
.claude-plugin/marketplace.json
.claude-plugin/plugin.json
```

- [ ] **Step 4: Commit**

```bash
git add .claude-plugin/
git commit -m "feat: add plugin scaffolding"
```

---

### Task 2: Core SKILL.md

This is the main file. It defines activation conditions, config resolution, core directives, and the three user commands. It links to reference files for detailed procedures.

**Files:**
- Create: `skills/memory-manager/SKILL.md`

- [ ] **Step 1: Create the skills directory**

```bash
mkdir -p skills/memory-manager/references
```

- [ ] **Step 2: Write SKILL.md**

The SKILL.md must contain these sections in order:

1. **Frontmatter** -- name and description with activation trigger
2. **Overview** -- one paragraph explaining what this skill does
3. **Prerequisites** -- obsidian-cli must be installed
4. **Config Resolution** -- how to find `.claude/memory.json`, read vault path, handle missing config
5. **Core Directives** -- the rules the agent follows at all times:
   - Lazy loading (never read vault at startup)
   - Taxonomy-first access (always read `_meta/taxonomy.md` before anything else)
   - Cache taxonomy after first read, invalidate after writes
   - Subagent rule (read-only for subagents)
   - Error handling (retry once, then inform user)
6. **Autonomous Write Triggers** -- brief summary with link to `references/TRIGGERS.md`
7. **Write Flow** -- the 5-step write procedure from the spec
8. **Retrieval Protocol** -- brief summary with link to `references/RETRIEVAL.md`
9. **Commands** -- `/memory-init`, `/memory-sync`, `/memory-optimize` brief descriptions with links to `references/OPTIMIZATION.md`
10. **Note Format** -- frontmatter template, type definitions, rules

Full content for SKILL.md:

````markdown
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
````

- [ ] **Step 3: Verify file exists and has frontmatter**

Run: `head -5 skills/memory-manager/SKILL.md`
Expected: frontmatter with `name: memory-manager`

- [ ] **Step 4: Commit**

```bash
git add skills/memory-manager/SKILL.md
git commit -m "feat: add core SKILL.md for memory-manager"
```

---

### Task 3: TRIGGERS.md Reference

> **Note:** Tasks 3-5 provide detailed outlines rather than verbatim content. The implementing agent writes the prose following the outline structure. This is intentional -- these files require nuanced instructional writing that benefits from the agent's judgment within the constraints of the outline.

Detailed detection heuristics for autonomous write triggers.

**Files:**
- Create: `skills/memory-manager/references/TRIGGERS.md`

- [ ] **Step 1: Write TRIGGERS.md**

Content must cover:

1. **Correction detection** -- heuristics for recognizing when the user is correcting the agent:
   - Phrases like "no", "don't do that", "stop doing X", "that's wrong", "I told you..."
   - User undoing agent's work
   - User providing an alternative approach after rejecting agent's
   - What to store: the pattern to avoid, why, and what to do instead
   - Example note content

2. **Decision detection** -- heuristics for recognizing project decisions:
   - Explicit: "let's go with X", "we decided to use Y"
   - Implicit: user picks between options after discussion
   - Architecture-significant choices (framework, database, API design, deployment)
   - What to store: the decision, rationale, alternatives considered
   - Example note content

3. **Preference detection** -- heuristics for behavioral preferences:
   - Style: "be more concise", "use TypeScript", "always write tests first"
   - Workflow: "don't push without asking", "commit often"
   - Communication: "stop summarizing", "explain your reasoning"
   - What to store: the preference and context
   - Example note content

4. **Pattern detection** -- heuristics for recurring patterns:
   - Agent notices the same code structure appearing across the project
   - User explains "we always do X when Y"
   - Architectural conventions not captured in config/linting
   - What to store: the pattern, when to apply it, example
   - Example note content

5. **Explicit triggers** -- in addition to autonomous detection, the agent writes when:
   - User explicitly says "remember this" or similar
   - A significant task completes (store summary of what was done and why)
   - After `/memory-optimize` runs (store results)
   - These don't require heuristics -- they're direct instructions or well-defined moments

6. **Confidence threshold** -- only write when reasonably confident. If unsure whether something is memory-worthy, err on the side of NOT writing. The cost of a missed memory is low (user can say "remember this"); the cost of noise is high (clutters vault, wastes tokens).

- [ ] **Step 2: Commit**

```bash
git add skills/memory-manager/references/TRIGGERS.md
git commit -m "feat: add TRIGGERS.md reference for autonomous write heuristics"
```

---

### Task 4: RETRIEVAL.md Reference

Detailed lookup strategy and token budget rules.

**Files:**
- Create: `skills/memory-manager/references/RETRIEVAL.md`

- [ ] **Step 1: Write RETRIEVAL.md**

Content must cover:

1. **When to retrieve** -- detailed situations:
   - Starting a task: check for relevant decisions/patterns in that domain
   - Encountering a person mentioned in conversation: look up their profile
   - About to make a decision: check for prior similar decisions
   - User asks "do you remember": search the vault
   - Uncertain about a preference: check correction/preference notes
   - Do NOT retrieve speculatively or "just in case"

2. **Hybrid lookup strategy (step by step):**
   - Step 1: Read `_meta/taxonomy.md` (or use cached version if available and not invalidated)
   - Step 2: From taxonomy Structure section, identify which folder(s) are relevant to the current need
   - Step 3: From taxonomy Index section, identify specific notes within those folders
   - Step 4: Read the identified notes. Start with frontmatter + first paragraph. Only read full content if the summary matches.
   - Step 5: If no match found via structured lookup, use `obsidian search query="relevant terms" vault="name"`
   - Step 6: From search results, read only the top matches (up to `max_results`)

3. **Multi-project filtering:**
   - When retrieving, filter by `project` frontmatter field matching the current project
   - Notes without a `project` field are global -- always include them
   - If the current project isn't known, check `.claude/memory.json` or the project directory name

4. **Token budget rules:**
   - Never read more than `max_results` notes per lookup (default: 5)
   - Read frontmatter + first paragraph before committing to full note read
   - If a note exceeds 50 lines, extract only the section relevant to the query
   - Cache taxonomy in conversation context; re-read only after a write or if a note isn't found where expected
   - A single retrieval operation should cost < 2000 tokens total (aim for this, not a hard limit)

5. **Empty vault handling:**
   - If taxonomy doesn't exist or has no entries, proceed without warning
   - Memories will accumulate naturally from writes

6. **Drift detection and self-correction during retrieval:**
   - If taxonomy says a note exists but it doesn't: remove the stale entry from taxonomy immediately (self-correct)
   - If search returns a note not in taxonomy: add it to taxonomy on the spot (self-correct)
   - In both cases, proceed with the retrieval -- don't block on drift
   - Flag to user during `/memory-sync` for full reconciliation

- [ ] **Step 2: Commit**

```bash
git add skills/memory-manager/references/RETRIEVAL.md
git commit -m "feat: add RETRIEVAL.md reference for lookup strategy"
```

---

### Task 5: OPTIMIZATION.md Reference

Full procedures for `/memory-init`, `/memory-sync`, and `/memory-optimize`.

**Files:**
- Create: `skills/memory-manager/references/OPTIMIZATION.md`

- [ ] **Step 1: Write OPTIMIZATION.md**

Content must cover:

1. **`/memory-init` -- Guided vault bootstrap:**
   - Step 1: Check if `.claude/memory.json` exists. If not, ask user for vault path and create it.
   - Step 2: Ask user what kind of project this is (or infer from codebase -- check language files, package.json, Cargo.toml, etc.)
   - Step 3: Present scannable categories as a checklist:
     - [ ] Architecture & tech stack
     - [ ] Key decisions in git history (last 50 commits)
     - [ ] Team/people context
     - [ ] Coding conventions & patterns
     - [ ] Known issues / pain points
     - [ ] User preferences (from CLAUDE.md, .editorconfig, linting configs)
   - Step 4: User picks categories (or "all")
   - Step 5: Scan selected categories. For each, use appropriate tools:
     - Architecture: read project structure, config files, dependency manifests
     - Git decisions: `git log --oneline -50`, look for decision-indicating commit messages
     - Conventions: read linting configs, CLAUDE.md, .editorconfig
     - People: check git log for contributors, any CODEOWNERS file
   - Step 6: Present findings as a summary. Let user approve, trim, or add context.
   - Step 7: Write approved memories to vault, create taxonomy.
   - Step 8: Set `last_synced` in taxonomy to today.

2. **`/memory-sync` -- Incremental update:**
   - Step 1: Read taxonomy, get `last_synced` date
   - Step 2: Scan vault filesystem vs taxonomy index:
     - Notes in vault but not in taxonomy → add to taxonomy
     - Taxonomy entries pointing to missing notes → remove from taxonomy
   - Step 3: Check git history since `last_synced`: `git log --since="YYYY-MM-DD" --oneline`
     - Look for decision-indicating changes
   - Step 4: Compare CLAUDE.md / project config against stored preferences
     - Flag conflicts or new preferences
   - Step 5: Present proposed updates to user
   - Step 6: User approves, agent executes writes
   - Step 7: Check taxonomy scaling (split if index exceeds ~100 entries)
   - Step 8: Update `last_synced` in taxonomy

3. **`/memory-optimize` -- Vault reorganization:**
   - Step 1: Read taxonomy
   - Step 2: Audit for issues:
     - Folders exceeding `suggest_threshold_notes` (default 30) → suggest splitting or consolidating
     - Near-duplicate notes (similar titles, overlapping content) → suggest merging
     - Stale content (references to files/functions that no longer exist in codebase) → suggest removal
     - Verbose notes exceeding `max_note_length` (default 500 lines) → suggest summarization
     - Miscategorized notes (type doesn't match folder) → suggest moving
   - Step 3: Present all proposed changes as a grouped list. Include what will be merged/moved/deleted/summarized.
   - Step 4: User approves (all, some, or none)
   - Step 5: Execute approved changes using filesystem tools (mv, rm) since obsidian-cli doesn't support these
   - Step 6: Check taxonomy scaling: if index exceeds ~100 entries, split into per-folder index files (`_meta/index/<folder>.md`) and keep root taxonomy as folder-level summaries only
   - Step 7: Update taxonomy
   - Step 8: Report summary: "Merged N notes, deleted N stale entries, summarized N verbose notes, moved N miscategorized"

4. **Proactive optimization suggestions:**
   - During normal retrieval, if the agent notices any of the issues listed in step 2 above, flag it:
     > "Your `decisions/` folder has 35 notes. Consider running `/memory-optimize` to consolidate."
   - Never act on optimization without user approval
   - At most one suggestion per session (don't nag)

5. **Note deletion policy:**
   - Notes are only deleted during `/memory-optimize` with explicit user approval
   - No soft-delete or archive mechanism
   - The agent never deletes notes autonomously outside optimization

- [ ] **Step 2: Commit**

```bash
git add skills/memory-manager/references/OPTIMIZATION.md
git commit -m "feat: add OPTIMIZATION.md reference for commands"
```

---

### Task 6: README.md

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write README.md**

Content:

1. **Title and one-line description**
2. **What it does** -- 3-4 bullet points
3. **Installation** -- marketplace and manual methods
4. **Setup** -- create `.claude/memory.json`, point to vault
5. **Usage** -- brief description of autonomous behavior + the 3 commands
6. **Configuration** -- vault-side `_meta/config.md` options
7. **Requirements** -- obsidian-cli, Obsidian vault

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README"
```

---

### Task 7: Integration Test (Manual Verification)

No automated tests since this is a pure-skill plugin. Manual verification instead.

- [ ] **Step 1: Verify plugin structure**

```bash
find . -type f -not -path './.git/*' -not -path './docs/*' | sort
```

Expected:
```
./.claude-plugin/marketplace.json
./.claude-plugin/plugin.json
./README.md
./skills/memory-manager/SKILL.md
./skills/memory-manager/references/OPTIMIZATION.md
./skills/memory-manager/references/RETRIEVAL.md
./skills/memory-manager/references/TRIGGERS.md
```

- [ ] **Step 2: Verify SKILL.md frontmatter**

```bash
head -4 skills/memory-manager/SKILL.md
```

Expected:
```
---
name: memory-manager
description: Autonomous memory management for Claude Code using an Obsidian vault...
---
```

- [ ] **Step 3: Verify all internal links resolve**

Check that SKILL.md references to `references/TRIGGERS.md`, `references/RETRIEVAL.md`, and `references/OPTIMIZATION.md` point to files that exist.

```bash
ls skills/memory-manager/references/
```

Expected: `OPTIMIZATION.md  RETRIEVAL.md  TRIGGERS.md`

- [ ] **Step 4: Verify plugin.json is valid JSON**

```bash
python3 -c "import json; json.load(open('.claude-plugin/plugin.json')); print('valid')"
python3 -c "import json; json.load(open('.claude-plugin/marketplace.json')); print('valid')"
```

Expected: `valid` for both

- [ ] **Step 5: Final commit with all files**

Verify nothing is unstaged:

```bash
git status
```

Expected: clean working tree
