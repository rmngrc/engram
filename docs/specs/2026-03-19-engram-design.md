# engram: Design Spec

Pure-skill AI coding agent plugin that uses obsidian-cli to autonomously manage an Obsidian vault as project memory.

## Plugin Structure

```
engram/
  .claude-plugin/
    plugin.json
    marketplace.json
  skills/
    memory-manager/
      SKILL.md
      references/
        TRIGGERS.md
        RETRIEVAL.md
        OPTIMIZATION.md
  README.md
```

Dependency: `obsidian-skills` (specifically `obsidian-cli` and `obsidian-markdown`).

Distributed as a standalone repo. Installable via AI coding agent marketplace or manually.

## Assumed CLI Surface Area

The skill depends on these `obsidian-cli` commands:

| Command | Purpose |
|---------|---------|
| `obsidian create file="path" content="..."` | Create a new note |
| `obsidian read file="path"` | Read a note's contents |
| `obsidian append file="path" content="..."` | Append to an existing note |
| `obsidian search query="..." vault="name"` | Full-text search across vault |
| `obsidian read path="_meta/taxonomy.md"` | Read taxonomy (structured lookup entry point) |

For operations needed by `/memory-optimize` (delete, move, rename), the agent uses filesystem tools (shell `mv`/`rm`, or file write/edit operations) since obsidian-cli doesn't expose these. The agent updates taxonomy after any such operation.

Vault targeting: `vault="name"` parameter on all commands. The skill resolves the vault name from config.

If a command is unavailable in the user's obsidian-cli version, the agent should fall back to filesystem operations via standard tools (file read, file write, content search).

## Configuration

### Project-side (`.engram/config.json`)

```json
{
  "vault": "/absolute/path/to/memory-vault"
}
```

Loaded lazily on first memory operation, not at startup. If missing, agent asks the user for the vault path and creates the file. If the vault directory doesn't exist, the agent asks the user whether to create it.

### Vault-side (`_meta/config.md`)

```yaml
---
auto_triggers:
  - correction
  - decision
  - preference
  - pattern
suggest_threshold_notes: 30  # notes per folder before suggesting /memory-optimize (counted from taxonomy index)
max_note_length: 500         # lines; suggest summarization above this
max_results: 5               # limit results per lookup to save tokens
---
```

Optional. Sensible defaults baked into skill instructions when absent. Agent can create/update as user tunes preferences.

## Vault Structure & Taxonomy

No predefined folders. The agent builds the structure organically based on content.

On first memory write the agent:

1. Creates the note in a folder it deems appropriate
2. Creates `_meta/taxonomy.md` documenting what it created and why

### `_meta/taxonomy.md` format

```markdown
---
last_updated: 2026-03-19
last_synced: 2026-03-19
note_count: 3
---

## Structure

- `people/` - User profiles, team members, stakeholder preferences
- `decisions/` - Architecture choices, trade-offs, rationale
- `_meta/` - Taxonomy, config, vault metadata

## Index

- `people/ramon.md` - User profile: role, preferences, corrections
- `decisions/auth-approach.md` - Chose JWT over sessions, rationale
- `decisions/db-migration.md` - Postgres migration strategy
```

Taxonomy is the agent's **only entry point** into the vault. One small file read instead of scanning.

### Evolution rules

- New folder only when existing ones don't fit
- Merge folders if overlap grows
- Keep top-level folders under ~15 as a soft cap; nest subcategories instead of proliferating top-level folders
- Always update taxonomy after any structural change
- Index section lists every note with a one-line description

### Taxonomy scaling

When the index exceeds ~100 entries, the agent should split the taxonomy into per-folder index files (`_meta/index/<folder>.md`) and keep the root taxonomy as a folder-level summary only. This keeps the entry-point read cheap.

### Manual edits and drift

If the user edits notes directly in Obsidian, taxonomy may drift. `/memory-sync` detects and repairs this by scanning the vault and reconciling with the taxonomy. The agent should also detect drift during normal retrieval (note not found where taxonomy says it is) and self-correct.

## Memory Write

### Autonomous triggers

Agent writes without being asked:

| Trigger (config key) | What gets stored | Example |
|----------------------|-----------------|---------|
| `correction` | Pattern to avoid | "Don't mock DB in tests" |
| `decision` | Decision + rationale + alternatives rejected | "Chose monorepo because..." |
| `preference` | Behavioral preference for future sessions | "User wants terse responses" |
| `pattern` | Recurring pattern worth remembering | "All API routes go through middleware X" |

### Explicit triggers

| Trigger | What gets stored |
|---------|-----------------|
| User says "remember this" | Whatever they specify |
| End of significant task | Summary of what was done and why |
| After optimization command | Results of vault reorganization |

### What NOT to store

- Things derivable from code or git history
- Ephemeral task state (use tasks/plans for that)
- Anything already in the project instructions file (e.g. CLAUDE.md, .cursorrules, AGENTS.md)
- Duplicate of an existing memory (update existing note instead)

### Write flow

1. Agent decides something is memory-worthy
2. Reads `_meta/taxonomy.md` to find where it belongs
3. Creates a new note or appends to existing (heuristic: append if same entity/topic exists, new note otherwise)
4. Updates `_meta/taxonomy.md` with any structural changes
5. Uses `obsidian-cli` for all vault operations

### Sub-agent rule

Sub-agents or delegated tasks cannot write to memory. The skill instructions include a directive: "If you are running as a sub-agent or delegated task (dispatched by another agent), do not write to the memory vault. Instead, include memory-worthy observations in your response to the parent agent." The main agent decides what to store.

### Error handling

If an obsidian-cli command fails:
1. Retry once
2. If still failing, inform the user with the error message
3. Do not silently drop the memory -- suggest the user check vault permissions, Obsidian lock status, or disk space

### Concurrency

The skill is designed for single-agent writes (enforced by the sub-agent rule). If two independent AI coding agent sessions target the same vault, last-write-wins applies to taxonomy. `/memory-sync` can repair inconsistencies after the fact. The skill does not implement locking.

## Memory Retrieval

Retrieval is never automatic at startup. The agent only fetches when needed.

### When the agent fetches

| Situation | What it looks up |
|-----------|-----------------|
| Starting a task in a known domain | Relevant decisions, patterns |
| Interacting with a mentioned person | That person's profile/preferences |
| About to make a decision similar to a past one | Previous decision + rationale |
| User asks "do you remember..." | Search vault for the topic |
| Agent is uncertain about a preference | Check for stored correction/preference |

### Hybrid lookup strategy

1. **Read `_meta/taxonomy.md`** -- one cheap read, full map
2. **Structured lookup** -- pick folder(s) most likely relevant based on taxonomy descriptions
3. **Read specific notes** -- only matching ones, frontmatter/first paragraph first when possible
4. **Fall back to search** -- `obsidian search` across vault if structured lookup finds nothing
5. **Return minimal context** -- extract only what's relevant, don't dump entire notes

### Multi-project filtering

When retrieving, the agent filters by the current project using the `project` frontmatter field. Notes without a `project` field are considered global (applicable to all projects).

### Token budget rules

- Never read more than `max_results` notes per lookup (default 5)
- Prefer reading note frontmatter/first paragraph before full content
- If a note is long, extract only the relevant section
- Cache taxonomy in conversation context after first read; invalidate cache after any write that modifies taxonomy

### Empty vault

If taxonomy doesn't exist or is empty, the agent proceeds normally without surfacing a warning. Memories accumulate naturally from the first write.

## User Commands

### `/memory-init` -- Guided vault bootstrap

For adding the skill to an existing project:

1. Agent asks what kind of project this is (or infers from codebase)
2. Presents scannable categories:
   - Architecture & tech stack
   - Key decisions in git history (last 50 commits max to bound cost)
   - Team/people context
   - Coding conventions & patterns
   - Known issues / pain points
   - User preferences (from project instructions file, e.g. CLAUDE.md, .cursorrules, AGENTS.md)
3. User picks which ones matter (or "all")
4. Agent scans selected categories, presents summary of findings
5. User reviews and approves or trims
6. Agent writes to vault and builds taxonomy

### `/memory-sync` -- Incremental update

1. Read taxonomy to get last_updated timestamp and current index
2. Scan vault filesystem for notes not in taxonomy (manual additions) and taxonomy entries pointing to missing notes (deletions)
3. Check recent git history (since `last_synced` in taxonomy frontmatter) for decision-worthy changes
4. Compare project instructions file and project config against stored preferences
5. Propose updates to user (new memories, stale removals, corrections)
6. User approves, agent writes changes and updates taxonomy

### `/memory-optimize` -- Vault reorganization

1. Read taxonomy
2. Audit for: folders exceeding `suggest_threshold_notes`, near-duplicate notes, stale content (references to removed code/files), verbose notes exceeding `max_note_length`, miscategorized entries
3. Propose changes to user (merge, move, summarize, delete)
4. User approves, agent executes
5. Update taxonomy

**Proactive suggestions:** during normal retrieval, agent flags inefficiency but doesn't act without approval.

### Note deletion

Notes are deleted only during `/memory-optimize` with user approval. No soft-delete or archive -- deleted means removed from vault and taxonomy. The agent never deletes notes autonomously outside of the optimization flow.

## Note Format

```markdown
---
created: 2026-03-19
updated: 2026-03-19
type: correction | decision | preference | pattern | person | context
project: my-app
tags:
  - auth
  - backend
---

JWT chosen over sessions for API auth.

## Rationale
Stateless, scales horizontally, team already familiar.

## Alternatives considered
- Sessions with Redis: rejected, added infra complexity
- OAuth only: rejected, need internal service-to-service auth too

## Related
- [[api-middleware]]
- [[scaling-strategy]]
```

### Type definitions

| Type | When used |
|------|-----------|
| `correction` | User corrected agent behavior -- store the pattern to avoid |
| `decision` | A project/architecture decision with rationale |
| `preference` | User behavioral preference for future sessions |
| `pattern` | Recurring code/architecture pattern worth remembering |
| `person` | Profile of a team member, stakeholder, or user |
| `context` | Background information that doesn't fit other types |

### Rules

- Frontmatter mandatory (enables cheap filtering by type/project/tags)
- `type` values align with autonomous trigger config keys plus `person` and `context` for non-triggered writes
- `project` supports multi-project vaults; omit for global memories
- Wikilinks for cross-referencing
- Body is concise -- written for retrieval, but human-readable
- Agent uses judgment on length: short for preferences, longer for decisions with rationale
- Append to existing note if same entity/topic, new note otherwise

## SKILL.md Structure Outline

The main `SKILL.md` should contain:

1. **Activation conditions** -- when the skill applies (any project with `.engram/config.json` or when user invokes `/memory-*` commands)
2. **Config resolution** -- how to find and read vault path and config
3. **Core directives** -- autonomous trigger rules, write flow, sub-agent rule
4. **Retrieval protocol** -- the hybrid lookup strategy, token budget
5. **Command definitions** -- `/memory-init`, `/memory-sync`, `/memory-optimize`

Reference files (`TRIGGERS.md`, `RETRIEVAL.md`, `OPTIMIZATION.md`) contain detailed instructions for each subsystem, keeping the main SKILL.md focused and readable.
