---
name: remove-dumb-comments
description: >
  Cleanup pass that strips low-value comments from code. Run it over a diff, a file, or a
  directory to delete narration that restates the code, docstrings that just name the symbol,
  section dividers that echo the data, change-history and chat-context notes, perishable
  measurements, and commented-out code, while keeping the comments that explain a non-obvious
  why. Invoke when asked to remove dumb comments, clean up comments, deslop comments, or review
  a diff for comment noise. For deciding whether to write a NEW comment, the same gate applies.
argument-hint: "[<file | dir | \"diff\">]"
model: sonnet
---

# Remove dumb comments

The default state of code is no comment. Clear names and small functions carry the meaning; a
comment survives only when it tells a reader something the code on the same screen cannot.

Be ruthless. If you are arguing with yourself about whether a comment "kind of orients the
reader" or "is nice context," that argument is the reason to delete it. **Borderline is a
delete.** Leniency is how these accumulate.

## The gate

For every comment in scope, answer one question:

> **What does this tell a future reader that the code right next to it does not?**

If the answer restates the code, a symbol's name, a type, or data sitting on the adjacent
lines, delete it. The fix for a line that needs narration is a better name, a smaller function,
or a type, not a sentence describing it.

A comment earns its place only by answering a _why_ the code cannot:

- ✅ `// Stripe sends the amount in cents; the rest of our system uses dollars`
- ✅ `# ATOMIC_REQUESTS is off, so wrap the two writes that must commit together`
- ✅ `// Kept in sync with the enum in migrations/0042; update both`

## Do this first: scope and safety

1. **Establish scope.** The argument names it: a path, a directory, or `"diff"` for the current
   change. Default to the current diff against the base branch. Only evaluate comments inside
   that scope.
2. **Stay in scope.** Do not sweep pre-existing comments in files the change does not otherwise
   touch, unless the user explicitly asks for a whole-repo pass. Churning untouched code buries
   the real change and re-litigates decisions that were not yours to reopen.
3. **Never remove functional pragmas.** These look like comments but change behavior, tooling,
   or output. Leave them alone:
   - Linter/type directives: `// biome-ignore`, `// eslint-disable*`, `// @ts-expect-error`,
     `// @ts-ignore`, `// prettier-ignore`, `# noqa`, `# type: ignore`, `# mypy:`, `# pylint:`,
     `# ruff:`, `# fmt: off`, `//nolint`, `//go:*` directives.
   - Coverage/build markers: `/* istanbul ignore */`, `# pragma: no cover`, `// coverage:`.
   - License and copyright headers, and shebang lines (`#!/...`).
   - Doctests: a Python docstring containing `>>>` is executable; do not strip it.
4. **Respect the project's docstring convention.** Some ecosystems require doc comments on
   exported symbols (Go package/exported docs, some Python public APIs). Keep those, but a
   docstring that only restates the symbol name is still a delete (see below). Match the
   surrounding density: do not strip a well-commented module bare, and do not leave one lonely
   divider standing after its siblings are gone.

## Delete these

### Narration that restates the code

- ❌ `# increment the counter` above `counter += 1`
- ❌ `// loop over users` above `for (const user of users)`
- ❌ `# return the result` above `return result`

If a block needs narration to be followed, extract a well-named function instead.

### Docstrings that just describe the obvious

This is the most common offender and the easiest to rationalize into keeping. A docstring or
field comment that names what the symbol already says is noise.

- ❌ `/** The setting's visible label, shown as the result row. */` above `label: string`
  (the field is named `label`; where it renders is visible at the call site)
- ❌ `/** Render the control on its own line under the label. */` above `stacked?: boolean`
  (the name plus a `flex-col`/`flex-row` body already say it)
- ❌ `/** Optional trailing control, aligned with the label. */` above `action?: ReactNode`
  (the `?` says optional; the render says where it goes)
- ❌ `"""Gets the user by id."""` on `get_user_by_id`
- ❌ A story/example export whose name already says what it demonstrates (`GeneralPage`,
  `NotificationDeliveryCards`) carrying a docstring that just describes the render

A component or function docstring survives **only** when it states intent or a contract the
signature cannot carry: why the abstraction exists, an invariant, an ordering guarantee, the
matching semantics. "What it renders" or "what it returns" is not that.

- ✅ `/** ... every whitespace-separated token must match the label, keywords, or page name. */`
  (AND-across-tokens semantics the name does not imply)
- ✅ `/** Segmented control ... so the choices read at a glance instead of hiding in a dropdown. */`
  (states why this exists instead of the default)

### Section dividers that echo the data

- ❌ `// General` above a run of entries that all carry `category: "general"`
- ❌ `// Notifications`, `// Advanced`, and the rest of the same set

A divider that is also **inaccurate** is a double delete: `// Pages without per-row entries`
above a block that does contain per-row entries is both redundant and wrong. A divider survives
only if it names a grouping the data does not express, and names it correctly.

### Change history and chat context

Never record how the code got here. That belongs in the commit message and PR description,
where it is attached to the diff and searchable. In the source it is noise that goes stale
immediately.

- ❌ `# previously used a set here, switched to a list for ordering`
- ❌ `// per PR #1234` / `# as discussed` / `# changed because the old way broke`
- ❌ `# AI: generated this helper` / `// agent: refactored` / `// round two: ...`
- ❌ `# TODO(2024-01): remove after migration` left in long after the migration

### Perishable measurements and current-state stamps

Measured timings, counts, and rates rot silently: nothing forces them to update, and a rotted
number misleads the next person sizing a timeout or a shard count. The same goes for "currently"
and "today" hedges, because the sentence states the same fact without them. State the durable
relationship, not the snapshot.

- ❌ `# skip the ~20 min build` when the durable fact is that the build is expensive
- ❌ `// regardless of the theme currently applied` where dropping "currently" states the same fact
- ❌ `# ~20 minutes in June, past 25 by July` (trend narration is change history)

Numbers that stay: a dated snapshot (`# as of August 2024, Homebrew ships 4.13.2`), a restated
adjacent code literal (`# runs that took >5 min (300 seconds)` beside the `300`), a platform
constant (`# GitHub's comment size limit (~64KB)`), a target or budget (`# Target: ~15 min per
shard`), or cited evidence with a link that dates it.

### Commented-out code

Delete it. Version history has it if it is ever needed again. Commented-out code is ambiguous to
the next reader, who cannot tell whether it is a note, a rollback plan, or an accident.

## Keep these

- A **why** that is not obvious from the code: a workaround, a performance trade-off, a spec
  quirk, an ordering constraint.
- A **warning** about a consequence that lives elsewhere: "changing this breaks the cache key",
  "callers rely on this being sorted".
- A **pointer** to context a reader cannot reconstruct from the repo: a link to the spec or
  ticket, or the reason a surprising value was chosen.
- A **contract** the signature does not carry: matching semantics, an enforced invariant, "the
  Record type forces an entry per category, so a new page can't ship unnamed".

## Style for the survivors

- **Explain why, not what.** The what is in the code.
- **No em-dash.** The tell is the clipped two-part phrase joined by a dash: `// batch here — avoids N+1`.
  Replace the dash with a real connective (`because`, `so that`, `which means`, `to avoid`) or two
  sentences: `// batch here to avoid an N+1 against the membership table`. The fix is the
  connective, not more words.
- **Be precise and technical.** Name the actual conditions, values, and consequences. Length
  follows content: one line when one line covers it, more when the reasoning needs it.
- **Preserve existing why-comments when moving code.** Do not drop a real why just because you
  are relocating the function.

## Procedure

1. Resolve scope and list the candidate comment lines. Language-agnostic starting greps
   (candidates, not verdicts):
   - Line and block comments in the diff: `git diff <base> -- <scope> | grep -nE '^\+.*(//|/\*|\*|#)'`
   - One-line docstrings and JSDoc: `grep -rnE '/\*\*.*\*/' <scope>`
   - Section-divider echoes: `grep -rnE '^\s*(//|#) [A-Z][a-z]+\s*$' <scope>`
   Skim the functional pragmas out of the candidate list before judging (step 3 in scope/safety).
2. Run each candidate through the gate. When unsure, delete.
3. Apply the deletions. Fix em-dashes and "currently"/"today" hedges in the comments that
   survive.
4. Verify nothing structural broke. Comment removal must not change behavior, so run the
   touched area's build, lint, and tests. A failure means you cut a functional pragma or a
   doctest; restore it.
5. Report what you cut and what you kept, grouped by reason, in a few lines. For each keeper
   that was close, state the why in one line so the next reviewer does not re-litigate it.
