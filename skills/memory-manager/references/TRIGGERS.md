# TRIGGERS.md -- Autonomous Write Heuristics

When to write to the vault without being asked. These rules tell you what to detect, how to detect it, and what to store.

---

## 1. Correction Detection

A correction happens when the user tells you something you did was wrong, or overrides your approach.

### Signals

**Explicit rejection phrases:**
- "no", "no no", "nope"
- "don't do that", "stop doing X", "don't ever X"
- "that's wrong", "that's not right", "that's not how we do it"
- "I told you...", "I already said...", "how many times..."
- "undo that", "revert that", "put it back"

**Behavioral signals:**
- User manually rewrites something you just wrote
- User undoes your change and does it a different way
- User provides an alternative approach immediately after rejecting yours
- User expresses frustration at a repeated mistake ("again?", "every time...")

### What to store

- The specific behavior or pattern to avoid
- Why it's wrong (if stated or clear from context)
- What to do instead
- The project context if it's project-specific

Do not store one-off corrections for typos or simple mistakes. Store patterns -- things likely to recur.

### Example note

```markdown
---
created: 2024-11-15
updated: 2024-11-15
type: correction
project: my-api
tags:
  - git
  - workflow
---

Do not run `git push` without explicit user confirmation. User wants to review
diffs before anything goes remote. Always stop and ask: "Should I push this?"

Reason: user caught a bad commit going to main before review.
```

---

## 2. Decision Detection

A decision is a choice made during conversation that will constrain future work. These are easy to miss because they often happen mid-discussion without fanfare.

### Signals

**Explicit statements:**
- "let's go with X", "we'll use X"
- "we decided to use Y", "we're going with Y"
- "use X, not Y"
- "that's the approach"

**Implicit choices:**
- User picks one option after you present two or more
- User says "yeah" or "ok" after you propose an architecture
- Discussion converges on a solution and work begins -- the accepted approach is the decision
- User asks you to implement something specific rather than exploring options

**Architecture-significant choices** (always store these):
- Framework or library selection (React vs Vue, Prisma vs Drizzle)
- Database choice or schema design
- API design (REST vs GraphQL, versioning strategy)
- Deployment target or infrastructure
- Authentication strategy
- Directory structure or module boundaries
- Monorepo vs multi-repo

### What to store

- The decision itself (clear, unambiguous)
- Rationale (why this option, even if brief)
- Alternatives that were considered and rejected
- Date (use frontmatter `created`)

### Example note

```markdown
---
created: 2024-11-20
updated: 2024-11-20
type: decision
project: my-api
tags:
  - database
  - architecture
---

## ORM: Drizzle over Prisma

**Decision:** Use Drizzle ORM for all database access.

**Rationale:** Team prefers SQL-first approach; Drizzle's type inference is
sufficient without the overhead of Prisma's generate step. Faster cold starts.

**Rejected:** Prisma (too much magic, slower DX for schema iteration), raw SQL
(no type safety).
```

---

## 3. Preference Detection

A preference is a standing instruction about how you should behave -- in this session and future ones.

### Signals

**Style preferences:**
- "be more concise", "shorter responses", "stop writing essays"
- "use TypeScript, not JavaScript"
- "always write tests first"
- "use functional style, avoid classes"
- "I prefer [naming convention]"

**Workflow preferences:**
- "don't push without asking", "always ask before pushing"
- "commit often", "commit after each feature"
- "run tests before you commit"
- "ask me before installing new packages"
- "don't modify X without telling me"

**Communication preferences:**
- "stop summarizing what you just did"
- "explain your reasoning before you act"
- "don't use bullet points"
- "give me the full file, not a diff"
- "show me the command before running it"

### What to store

- The preference, stated plainly
- Scope: global (all projects) or project-specific
- Context if it came from a specific situation (helps you understand edge cases)

Omit `project` from frontmatter for global preferences.

### Example note

```markdown
---
created: 2024-11-18
updated: 2024-11-18
type: preference
tags:
  - communication
---

Do not summarize actions after completing them. User finds post-action summaries
redundant -- they can see what was done. Just report the result or ask what's next.

Context: user said "stop narrating" after a code edit.
```

---

## 4. Pattern Detection

A pattern is a convention used repeatedly in the project that isn't captured in config or linting. These often live only in people's heads until you notice them.

### Signals

**You notice it first:**
- You see the same code structure in 3+ places in the codebase
- A naming convention is consistent across all files of a type
- A specific way of handling errors, loading state, or side effects appears throughout
- A folder organization choice repeated across multiple modules

**User states it explicitly:**
- "we always do X when Y"
- "every [component/module/service] should have..."
- "the convention here is..."
- "don't do it that way, we do it like this everywhere else"

### What to store

- The pattern, described precisely
- When to apply it (conditions or context)
- A minimal example (inline code block)
- Where to find examples in the codebase (file path or folder)

Only store patterns that: (a) aren't obvious from the code, (b) aren't enforced by tooling, and (c) would recur in future work.

### Example note

```markdown
---
created: 2024-11-22
updated: 2024-11-22
type: pattern
project: my-api
tags:
  - error-handling
  - api
---

## API route error handling

All API route handlers wrap logic in try/catch and use the shared `ApiError`
class. Never throw raw errors from routes.

```ts
try {
  // handler logic
} catch (err) {
  throw new ApiError(500, "descriptive message", { cause: err });
}
```

Examples: `src/routes/users.ts`, `src/routes/billing.ts`.
```

---

## 5. Explicit Triggers

These don't require detection -- they're direct instructions or well-defined moments.

### "Remember this" (and similar)

Triggers when user says:
- "remember this", "remember that"
- "save this", "store this"
- "don't forget this"
- "keep this in mind for future sessions"

What to do: store exactly what they specified. Ask for clarification if it's vague. Use the most appropriate type based on content.

### End of significant task

When a multi-step task completes (feature built, bug fixed, refactor done), write a brief summary note:
- What was done
- Why (the goal or problem being solved)
- Key decisions made during the task (if not already stored)
- Anything that was tricky or non-obvious

Don't write this for trivial single-step tasks. Use judgment -- if you spent 10+ minutes on something and made meaningful decisions, it's worth capturing.

### After `/memory-optimize`

After a vault reorganization run, store a brief note in `_meta/` describing:
- What changed (folders merged, notes pruned, etc.)
- Note count before and after
- Date

This creates an audit trail and helps you understand vault history.

---

## 6. Confidence Threshold

Only write when you're reasonably confident something is memory-worthy.

**Lean toward NOT writing when:**
- You're unsure if the user's statement is a one-time thing or a standing rule
- The correction is minor and unlikely to recur
- A decision feels tentative ("maybe we'll use X", "let's try Y for now")
- You'd be guessing at rationale that wasn't stated

**The asymmetry:** A missed memory costs little -- the user can say "remember this." Noise in the vault costs more -- it clutters retrieval, wastes tokens, and degrades the signal-to-noise ratio over time.

**When in doubt, don't write.** If you're on the fence, you can note internally that something might be memory-worthy and wait to see if it comes up again. Recurrence is itself a signal.

**The exception:** If the user seems frustrated or explicitly emphasizes something ("I really need you to remember this"), write it even if you're not fully certain of the scope.
