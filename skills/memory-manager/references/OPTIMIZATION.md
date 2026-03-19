# OPTIMIZATION.md -- Command Procedures

Full procedures for `/memory-init`, `/memory-sync`, and `/memory-optimize`. Follow these steps mechanically.

---

## `/memory-init` -- Guided Vault Bootstrap

Bootstrap the memory vault for an existing project. Run this once per project to seed the vault with context that would otherwise require many sessions to accumulate.

**Step 1: Resolve config**

Check if `.engram/config.json` exists in the project root.

- If it exists, read it and confirm the vault path is valid.
- If it does not exist, ask the user: "What path should I use for the memory vault?" Create `.engram/config.json` with `{ "vault": "/absolute/path" }`. If the vault directory does not exist, ask whether to create it.

**Step 2: Infer project type**

Check the codebase to understand what kind of project this is before asking the user anything. Look for:

- `package.json` → Node/JS/TS project; read `name`, `description`, `scripts`, and top-level dependencies
- `Cargo.toml` → Rust project
- `pyproject.toml` or `setup.py` → Python project
- `go.mod` → Go project
- `Gemfile` → Ruby project
- `*.xcodeproj` or `Package.swift` → Swift/iOS project
- Root language files (e.g., `main.rs`, `index.ts`, `app.py`) as a fallback signal

Present your inference to the user: "This looks like a [TypeScript monorepo / Rust CLI / etc.]. Is that right?" Let them correct it.

**Step 3: Present category checklist**

Ask the user which categories to scan. Present as a numbered list:

```
Which categories should I scan? (enter numbers, comma-separated, or "all")

1. Architecture & tech stack
2. Key decisions in git history (last 50 commits)
3. Team / people context
4. Coding conventions & patterns
5. Known issues / pain points
6. User preferences (CLAUDE.md, .editorconfig, linting configs)
```

**Step 4: Receive user selection**

Accept "all" or a comma-separated list of numbers. If the user says nothing or presses enter, default to "all".

**Step 5: Scan selected categories**

For each selected category, use the appropriate tools:

**Architecture & tech stack**

- Read the project root directory listing
- Read config files: `package.json`, `tsconfig.json`, `Cargo.toml`, `pyproject.toml`, `docker-compose.yml`, `Makefile`, `.env.example`, etc.
- Read any `README.md` at the root
- Note the folder structure (top two levels is enough)
- Summarize: language, runtime, frameworks, key dependencies, build system, test framework, deployment approach

**Key decisions in git history**

- Run: `git log --oneline -50`
- Look for commit messages that indicate decisions: words like "switch", "replace", "migrate", "add", "remove", "refactor", "chose", "use X instead of Y", "revert"
- For each candidate, optionally read the commit diff (`git show <hash> --stat`) to understand scope
- Capture: what changed, why (if inferable from message), approximate date

**Team / people context**

- Run: `git log --format="%an <%ae>" | sort | uniq -c | sort -rn | head -20` to get contributors by commit count
- Check for `CODEOWNERS`, `MAINTAINERS`, or `OWNERS` files
- Check `package.json` `author` and `contributors` fields if present
- Capture: names, email domains (org context), rough contribution areas if inferable from file paths in commits

**Coding conventions & patterns**

- Read `.editorconfig` if present
- Read linting configs: `.eslintrc*`, `eslint.config.*`, `.prettierrc*`, `pyproject.toml` `[tool.ruff]` / `[tool.black]`, `rustfmt.toml`, `.rubocop.yml`
- Read project instructions file if present (e.g. `CLAUDE.md`, `.cursorrules`, `AGENTS.md`)
- Note: indent style, line length limits, quote style, naming conventions, any project-specific rules

**Known issues / pain points**

- Check `TODO`, `FIXME`, `HACK`, `XXX` comments: `grep -r "TODO\|FIXME\|HACK\|XXX" --include="*.{ts,js,py,rs,go}" -l` (show file list only)
- Check for any open issues files or `KNOWN_ISSUES.md`
- Note recurring TODO themes without reading every instance

**User preferences**

- Read the project instructions file if present (e.g. `CLAUDE.md`, `.cursorrules`, `AGENTS.md`, `GEMINI.md`, `.github/copilot-instructions.md`)
- Read `.editorconfig`
- Check for any `.engram/` directory contents beyond `config.json`
- Capture: behavioral preferences, workflow rules, style preferences the user has expressed

**Step 6: Present findings for approval**

Present a structured summary of findings per category. For each item:

```
[Category Name]
- Finding 1
- Finding 2
...
```

Ask: "Should I store all of these, or are there items to trim or add context to?" Let the user edit inline or by number.

**Step 7: Write memories to vault**

For each approved finding:

1. Determine the appropriate folder (create if needed, following vault structure rules from SKILL.md)
2. Create notes using `obsidian create` or filesystem Write tool
3. Use proper frontmatter with `type`, `created`, `project`, and relevant `tags`
4. Update `_meta/taxonomy.md` with new structure and index entries

Group related findings into single notes where it makes sense (e.g., all conventions in one `conventions/project-conventions.md` note).

**Step 8: Set `last_synced`**

Update the `last_synced` field in `_meta/taxonomy.md` frontmatter to today's date.

Report to the user: "Vault initialized. Wrote N notes across X folders."

---

## `/memory-sync` -- Incremental Update

Sync the vault with current project state. Run this periodically or when you suspect the vault is stale.

**Step 1: Read taxonomy and get sync baseline**

Read `_meta/taxonomy.md`. Extract `last_synced` from frontmatter. If `last_synced` is absent, treat the entire git history as in scope (and warn the user the vault was initialized without a sync date).

**Step 2: Audit vault filesystem vs taxonomy index**

Compare the taxonomy index entries against the actual vault filesystem:

- Notes present in vault but not in taxonomy → flag as "unindexed notes"
- Taxonomy entries pointing to files that don't exist → flag as "broken taxonomy entries"

List these discrepancies. Don't resolve them yet.

**Step 3: Check git history since `last_synced`**

Run: `git log --since="YYYY-MM-DD" --oneline` (substitute the actual `last_synced` date).

Scan the output for decision-indicating commit messages using the same heuristics as `/memory-init` Step 5 (keywords: "switch", "replace", "migrate", "add", "remove", "refactor", "chose", "use X instead of Y").

For candidate commits, check if a corresponding memory already exists (search taxonomy index for the topic). Flag new decisions not yet stored.

**Step 4: Compare config against stored preferences**

Read the current project instructions file (e.g. `CLAUDE.md`, `.cursorrules`, `AGENTS.md`), `.editorconfig`, and linting configs.

Compare against any `preferences/` or `conventions/` notes in the vault.

Flag:
- New rules in config files not reflected in vault
- Vault notes that contradict current config (conflicts)
- Vault notes referencing files or settings that no longer exist

**Step 5: Present proposed updates**

Show the user a grouped list of proposed changes:

```
Unindexed notes (will add to taxonomy):
- decisions/use-drizzle.md

Broken taxonomy entries (will remove from index):
- patterns/old-pattern.md (file missing)

New decisions from git history:
- "migrate from jest to vitest" (commit abc1234) → store in decisions/

Config changes to capture:
- .eslintrc now enforces "no-console" → update conventions/

Conflicts:
- vault says "use tabs" but .editorconfig now says "indent_style = space"
```

Ask: "Approve all, or specify which items to skip?"

**Step 6: Execute approved updates**

For each approved item:

- Unindexed notes: add entries to taxonomy index
- Broken entries: remove from taxonomy index (do not delete the note reference, just clean index)
- New decisions: create notes using write flow from SKILL.md
- Config changes: update or append to existing convention/preference notes
- Conflicts: update the vault note to reflect current state; add a dated comment noting the change

**Step 7: Check taxonomy scaling**

Count the total entries in `_meta/taxonomy.md` index section.

If the count exceeds ~100 entries:
- Create per-folder index files at `_meta/index/<folder>.md`
- Move that folder's entries from the root taxonomy to the per-folder file
- Keep the root taxonomy as folder-level summaries only (one line per folder)
- Inform the user: "Index split into per-folder files due to size."

**Step 8: Update `last_synced`**

Set `last_synced` in `_meta/taxonomy.md` frontmatter to today's date.

Report: "Sync complete. Added N entries, removed N broken entries, stored N new memories."

---

## `/memory-optimize` -- Vault Reorganization

Audit and reorganize the vault. This is the only command that deletes or moves notes. All changes require explicit user approval.

**Step 1: Read taxonomy**

Read `_meta/taxonomy.md` (and any per-folder index files if they exist). Build a full picture of the vault structure and index.

**Step 2: Audit for issues**

Check each of the following. Record every issue found.

**Overcrowded folders**

For each folder in the taxonomy, count its notes. If a folder exceeds `suggest_threshold_notes` (default: 30), flag it. Propose either:
- Splitting into subfolders (if notes cluster around distinct subtopics)
- Consolidating several short notes into a single longer one (if notes are thin)

**Near-duplicate notes**

Scan for notes with similar titles within the same folder. For pairs that look similar, read both and compare content. If they cover the same topic with significant overlap, flag for merging. Propose a merged title and note which one to keep as the base.

**Stale content**

For notes that reference specific files, functions, or classes (e.g., `src/utils/oldHelper.ts`, `function parseToken`), check if those targets still exist in the codebase. Search the codebase to verify. Flag notes where the referenced code is gone.

**Verbose notes**

For notes exceeding `max_note_length` (default: 500 lines), flag for summarization. Propose keeping the key decisions/preferences and trimming historical narrative.

**Miscategorized notes**

Check each note's `type` frontmatter field against its folder location. Flag mismatches:
- A note with `type: correction` in a `decisions/` folder
- A note with `type: preference` in `patterns/`

Propose moving each to the correct folder.

**Step 3: Present all proposed changes**

Group by issue type and present as a single list before touching anything:

```
Overcrowded folders:
- decisions/ has 42 notes. Suggest splitting into decisions/architecture/ and decisions/tooling/

Near-duplicates (suggest merging):
- patterns/api-error-handling.md + patterns/error-handling-api.md → patterns/api-error-handling.md

Stale content (suggest deletion):
- context/old-monorepo-setup.md references src/packages/ which no longer exists

Verbose notes (suggest summarization):
- decisions/auth-redesign.md is 620 lines. Suggest trimming to key decisions.

Miscategorized notes (suggest moving):
- decisions/prefer-pnpm.md has type: preference → move to preferences/
```

Ask: "Approve all, some (list numbers), or none?"

**Step 4: Receive user approval**

Accept "all", "none", or a comma-separated list of item numbers. Execute only what is approved.

**Step 5: Execute approved changes**

Use filesystem operations (read, write, edit files; use `mv` / `rm` in the terminal) since obsidian-cli does not support move or delete operations.

- **Split folder**: Create new subfolders, move note files, update taxonomy
- **Merge notes**: Read both notes, write a merged version to the kept file path, delete the other file
- **Delete stale notes**: Remove the file, remove from taxonomy index
- **Summarize verbose notes**: Read the note, write a condensed version back to the same path (preserve frontmatter, update `updated` date)
- **Move miscategorized notes**: Move file to correct folder, update taxonomy index entry

After each operation, verify the file state is correct before moving to the next.

**Step 6: Check taxonomy scaling**

Same as `/memory-sync` Step 7. If the index exceeds ~100 entries after reorganization, split into per-folder index files.

**Step 7: Update taxonomy**

Rewrite or update `_meta/taxonomy.md` to reflect all structural changes: new folders, removed notes, moved notes, updated descriptions. Set `last_updated` to today. Update `note_count`.

**Step 8: Report summary**

Print a final summary:

```
Optimization complete.
- Merged N notes
- Deleted N stale entries
- Summarized N verbose notes
- Moved N miscategorized notes
- Split N overcrowded folders
```

---

## Proactive Optimization Suggestions

During normal retrieval operations, if the agent notices any of the audit issues from `/memory-optimize` Step 2, it should flag it once:

> "Your `decisions/` folder has 35 notes. Consider running `/memory-optimize` to consolidate."

Rules:
- Never act on optimization without explicit user approval
- Raise at most one suggestion per session (don't repeat it)
- Suggestions are informational only -- the agent takes no action until the user runs the command

---

## Note Deletion Policy

Notes are only deleted during `/memory-optimize` with explicit user approval at Step 4.

- No soft-delete mechanism
- No archive folder
- Deletion is permanent (file removed from vault)
- The agent never deletes notes autonomously outside of an approved `/memory-optimize` run
