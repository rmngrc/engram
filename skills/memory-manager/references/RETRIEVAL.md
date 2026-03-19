# RETRIEVAL.md -- Lookup Strategy and Token Budget

When and how to retrieve memories from the vault. Follow these procedures mechanically.

---

## 1. When to Retrieve

Retrieve only when there is a concrete reason. Do not retrieve speculatively or "just in case."

**Retrieve when:**

- **Starting a task in a domain** -- before writing code, making a plan, or touching a system area, check for stored decisions or patterns in that domain. Example: about to touch the auth module? Check for auth-related decisions and patterns.

- **A person is mentioned** -- when a team member or stakeholder is named in conversation, look up their profile before responding. People notes contain context that shapes how you should interact or interpret their feedback.

- **About to make a decision** -- before proposing an architecture choice, library selection, or design approach, check whether a prior decision already covers it. Don't re-litigate settled choices.

- **User asks "do you remember"** -- treat this as an explicit retrieval request. Search immediately. Report what you find or confirm that nothing is stored.

- **Uncertain about a preference** -- if you're unsure whether the user prefers a particular style, workflow, or communication pattern, check preference and correction notes before guessing.

**Do NOT retrieve when:**

- You're about to answer a factual question unrelated to the project
- The task is trivial and self-contained (a one-liner, a quick lookup)
- You already retrieved relevant memories earlier in this conversation and nothing has changed
- You're speculating that something *might* be useful -- retrieve only when you have a specific reason

---

## 2. Hybrid Lookup Strategy

Follow these steps in order. Stop when you have what you need.

**Step 1: Read `_meta/taxonomy.md`**

Use the cached version if you already read it this session and no write has occurred since. If the cache is invalid or absent, read the file now:

```
obsidian read file="_meta/taxonomy.md" vault="<vault-name>"
```

If obsidian-cli is unavailable, read the file directly from the vault directory.

**Step 2: Identify relevant folders**

From the taxonomy's `## Structure` section, identify which folder(s) are likely to contain what you need. Match based on folder descriptions, not just names. If nothing looks relevant, skip to Step 5.

**Step 3: Identify specific notes**

From the taxonomy's `## Index` section (or per-folder index files under `_meta/index/` if the vault is large), find specific notes within the relevant folders. The one-line descriptions in the index are your filter -- don't read a note if its description clearly doesn't match.

**Step 4: Read the identified notes**

Read frontmatter and the first paragraph first. Only read the full note if the summary suggests it's relevant.

```
obsidian read file="<folder/note.md>" vault="<vault-name>"
```

If a note exceeds 50 lines and only part of it is relevant, extract the section that matches your query. Do not load the entire note into context.

Stop here if you found what you needed.

**Step 5: Fall back to search**

If structured lookup found nothing, run a keyword search:

```
obsidian search query="<relevant terms>" vault="<vault-name>"
```

If obsidian-cli is unavailable, search file contents across the vault directory using relevant terms as the pattern.

**Step 6: Read top matches from search**

From search results, read only the top matches up to `max_results` (default: 5). Apply the same rule as Step 4 -- frontmatter and first paragraph first, full content only if warranted.

---

## 3. Multi-Project Filtering

When retrieving from a vault that serves multiple projects, filter results by project scope.

**Rules:**

- Include notes where `project` frontmatter matches the current project name
- Always include notes with no `project` field -- these are global and apply to all projects
- Exclude notes where `project` is set to a different project

**Finding the current project name:**

1. Check `.engram/config.json` in the project root -- it may contain a `project` field
2. If not, use the project directory's basename (e.g., `/Users/me/dev/my-api` → `my-api`)
3. If still unclear, ask the user before filtering

When doing keyword search (Step 5), post-filter results by project after retrieving them -- don't rely on the search to apply frontmatter filtering automatically.

---

## 4. Token Budget Rules

Retrieval must stay cheap. These are guardrails to prevent context bloat.

- **Never read more than `max_results` notes per lookup** (default: 5). If you need more, re-evaluate the query -- a broader search usually means the query is too vague.

- **Read frontmatter + first paragraph before committing to the full note.** Most of the time, the summary is enough. Only read the full note if the summary is relevant and you need detail.

- **If a note exceeds 50 lines,** extract only the section relevant to your query. Do not dump the whole note into context.

- **Cache the taxonomy.** After reading `_meta/taxonomy.md`, keep it in conversation context. Re-read it only after a write that modifies the taxonomy, or if a note isn't found where the taxonomy says it should be.

- **Target < 2000 tokens per retrieval operation.** This is a guideline, not a hard cutoff. If you're consistently going over, narrow the query or increase selectivity in Step 3.

---

## 5. Empty Vault Handling

If the vault is new or nearly empty:

- If `_meta/taxonomy.md` doesn't exist, skip all lookup steps and proceed without warning
- If the taxonomy exists but has no index entries, proceed without warning
- Do not notify the user that no memories were found unless they asked "do you remember" explicitly

Memories accumulate naturally through writes. An empty vault is not a problem.

---

## 6. Drift Detection and Self-Correction

Taxonomy and vault can drift out of sync (manual edits, interrupted writes, file moves). Correct drift on the spot -- don't block retrieval on it.

**Case 1: Taxonomy says a note exists, but it doesn't**

When you try to read a note and it's missing:
1. Remove the stale entry from `_meta/taxonomy.md` immediately
2. Continue retrieval -- treat that note as not found and proceed to the next candidate or fall back to search

**Case 2: Search returns a note not listed in taxonomy**

When a search result references a note that isn't in the taxonomy index:
1. Read the note
2. Add it to `_meta/taxonomy.md` under the appropriate folder
3. Continue -- the note is now in scope

In both cases, do not pause or ask the user about the drift. Note it internally and flag it when the user runs `/memory-sync` for full reconciliation.
