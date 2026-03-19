# obsidian-memory: Design Spec

Pure-skill Claude Code plugin that uses obsidian-cli to autonomously manage an Obsidian vault as project memory.

## Plugin Structure

```
obsidian-memory/
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

Distributed as a standalone repo. Installable via Claude Code marketplace or manually.

## Configuration

### Project-side (`.claude/memory.json`)

```json
{
  "vault": "/absolute/path/to/memory-vault"
}
```

Loaded lazily on first memory operation, not at startup. If missing, agent asks the user for the vault path and creates the file.

### Vault-side (`_meta/config.md`)

```yaml
---
auto_triggers:
  - user_corrections
  - project_decisions
  - architecture_patterns
optimization:
  suggest_threshold: 30
  max_note_length: 500
retrieval:
  max_results: 5
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
- Keep folder count reasonable (agent's judgment)
- Always update taxonomy after any structural change
- Index section lists every note with a one-line description

## Memory Write

### Autonomous triggers

Agent writes without being asked:

| Trigger | What gets stored | Example |
|---------|-----------------|---------|
| User correction | Pattern to avoid | "Don't mock DB in tests" |
| Project decision | Decision + rationale + alternatives rejected | "Chose monorepo because..." |
| User preference | Behavioral preference for future sessions | "User wants terse responses" |
| Architecture pattern | Recurring pattern worth remembering | "All API routes go through middleware X" |

### Explicit triggers

| Trigger | What gets stored |
|---------|-----------------|
| User says "remember this" | Whatever they specify |
| End of significant task | Summary of what was done and why |
| After optimization command | Results of vault reorganization |

### What NOT to store

- Things derivable from code or git history
- Ephemeral task state (use tasks/plans for that)
- Anything already in CLAUDE.md
- Duplicate of an existing memory (update existing note instead)

### Write flow

1. Agent decides something is memory-worthy
2. Reads `_meta/taxonomy.md` to find where it belongs
3. Creates a new note or appends to an existing one (agent's judgment)
4. Updates `_meta/taxonomy.md` with any structural changes
5. Uses `obsidian-cli` for all vault operations

### Subagent rule

Subagents cannot write to memory. If a subagent encounters something memory-worthy, it reports back. The main agent decides whether to store it.

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

### Token budget rules

- Never read more than `max_results` notes per lookup (default 5)
- Prefer reading note frontmatter/first paragraph before full content
- If a note is long, extract only the relevant section
- Cache taxonomy in conversation context after first read

## User Commands

### `/memory-init` -- Guided vault bootstrap

For adding the skill to an existing project:

1. Agent asks what kind of project this is (or infers from codebase)
2. Presents scannable categories:
   - Architecture & tech stack
   - Key decisions in git history
   - Team/people context
   - Coding conventions & patterns
   - Known issues / pain points
   - User preferences (from CLAUDE.md or similar)
3. User picks which ones matter (or "all")
4. Agent scans selected categories, presents summary of findings
5. User reviews and approves or trims
6. Agent writes to vault and builds taxonomy

### `/memory-sync` -- Incremental update

1. Agent reads current project state (code structure, recent git history, CLAUDE.md)
2. Compares against vault contents
3. Flags stale memories, missing context, outdated decisions
4. Proposes updates, user approves
5. Writes changes and updates taxonomy

### `/memory-optimize` -- Vault reorganization

1. Read taxonomy
2. Audit for: overfull folders, duplicates, stale content, verbose notes, miscategorized entries
3. Propose changes to user (merge, move, summarize, delete)
4. User approves, agent executes
5. Update taxonomy

**Proactive suggestions:** during normal retrieval, agent flags inefficiency but doesn't act without approval.

## Note Format

```markdown
---
created: 2026-03-19
updated: 2026-03-19
type: decision | preference | pattern | person | context
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

### Rules

- Frontmatter mandatory (enables cheap filtering by type/project/tags)
- `type` maps to autonomous trigger categories
- `project` supports multi-project vaults
- Wikilinks for cross-referencing
- Body is concise -- written for retrieval, but human-readable
- Agent uses judgment on length: short for preferences, longer for decisions with rationale
- One note per memory for distinct items, append for incremental updates (agent decides)
