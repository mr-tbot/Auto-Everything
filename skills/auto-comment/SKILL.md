---
name: auto-comment
description: "Use when a codebase's comments need bringing into line with the code — inconsistent commenting across files written by different hands or models, comments that describe behavior that changed, missing doc comments on public API, or over-commented noise; when the user says fix the comments, document the code, add docstrings, or \"/auto-comment\"; or when about to leave a changed function with a comment that no longer describes it."
---

# /auto-comment

Bring the codebase's comments into line with what the code actually does — incrementally, resumably,
and without touching the comments that are load-bearing.

The failure this exists for: successive passes by different hands and different models. One adds a
comment to every line, one strips comments as noise, one writes none. What is left is a codebase where
comment density signals nothing about complexity and half the comments describe code that has since
changed.

**A comment that lies is worse than no comment**, because it is trusted. Stale comments are the
primary target of this skill; missing ones are secondary.

## Step 1 — Read Your Own State

State lives in `.agent/comment-state.json` (create it if absent, and make sure it is gitignored or
committed deliberately — either is fine, but decide):

```json
{
  "last_run": "2026-08-17T14:22:05Z",
  "last_commit": "f932834…",
  "conventions": { "…": "what was established in step 2" },
  "swept": ["src/core/", "src/api/"],
  "remaining": ["src/legacy/", "src/vendor-shim/"],
  "deliberate_skips": { "src/generated/": "codegen output, never edit" }
}
```

**On every run:**

1. No state file → this is a first run. Go to Step 2 and establish conventions.
2. State exists → **compute what changed since `last_commit`**:

```bash
git diff --name-only <last_commit>..HEAD
git log --oneline <last_commit>..HEAD | head -20
```

Then work in this order:

- **Changed files first.** Code that moved is where comments went stale. This is the highest-value
  work and it is what "keep comments in line with the codebase" actually means.
- **Then the backlog** — the next slice of `remaining`, so an untouched corner eventually gets swept.
- Update state at the end of every run, including a partial one. **A run that does not update state
  has broken the resumability that justifies the skill.**

Report at the start: when you last ran, how many commits ago, and what you are covering this run.

## Step 2 — Establish The Convention, Do Not Impose One

Read before writing. Determine what this codebase already does:

- **Doc-comment format** in use per language — and whether it is applied to public API only, or
  everything
- **Density**: does this project comment intent at the top of a function, or inline, or barely?
- **Voice**: imperative or descriptive, first person plural or impersonal, sentence case or fragment
- **Whether doc comments feed a generator** (a docs site, an API reference). If they do, their
  **syntax is functional** and breaking it breaks the docs build.

Sample the most recently-touched, most load-bearing files rather than the oldest ones — the newest
coherent convention is usually the intended one.

If the codebase genuinely has no convention, propose one, show an example, and get one confirmation
before applying it broadly. Do not invent a house style silently across a thousand files.

## Step 3 — Never Touch These

Some comments are not commentary. Removing or reformatting them changes behavior or breaks a build.
**Treat every one of these as read-only unless the user explicitly asks:**

| Category | Examples |
|---|---|
| **Lint and compiler directives** | `// eslint-disable-next-line`, `# noqa`, `# type: ignore`, `@ts-expect-error`, `@Suppress("DEPRECATION")`, `@SuppressLint`, `// swiftlint:disable`, `# rubocop:disable`, `// nolint`, `#pragma`, `// @formatter:off`, `// noinspection` |
| **Build/tooling pragmas** | `//go:build`, `//go:generate`, `# frozen_string_literal:`, shebangs, encoding declarations |
| **Legal** | Copyright headers, license blocks, `SPDX-License-Identifier` |
| **Generated-file markers** | `DO NOT EDIT`, `@generated`, codegen banners — and the whole file behind them |
| **Deliberate markers with context** | `TODO`/`FIXME`/`HACK`/`XXX` that carry a reason, a ticket, or a name |
| **Structured annotations** | Anything a tool parses: `@deprecated`, `@throws`, `@param`, DI or serialization hints |

A directive frequently carries an explanatory comment on the same line —
`@Suppress("DEPRECATION") // pre-API-33 registration path`. **Both halves stay.** The annotation is
functional; the trailing note is the only record of why it exists.

If a `TODO` looks stale, **do not delete it** — surface it in the report. Deleting the record of
deferred work is not tidying, it is forgetting.

## Step 4 — What To Write

**Comment the why, never the what.** The code already says what it does; the comment exists for what
the code cannot say.

Worth writing:

- **Why this approach** over the obvious one — the constraint, the bug it works around, the
  benchmark that decided it
- **Non-obvious invariants and preconditions** — what must be true for this to be correct, what the
  caller must guarantee, what happens when it is not
- **Units, ranges, and coordinate systems** — milliseconds or seconds, zero- or one-indexed, which
  timezone, which color space
- **Concurrency** — which thread this runs on, what lock is held, what is safe to call from where
- **Boundaries with the outside world** — protocol quirks, vendor bugs, spec sections, and a link
- **Public API doc comments**, in the project's format, describing contract rather than
  implementation
- **The pointer to the real explanation** — a spec, an issue, an ADR

Not worth writing, and actively harmful:

- `// increment i`, `// constructor`, `// getter for name`
- A doc comment that restates the signature in prose
- Section-divider banners that decay the moment code moves
- Commented-out code — it is in git; delete it and say so in the report
- A comment repeating a name that already reads clearly

**If a comment would only restate the code, the answer is usually a better name, not a comment.**
Say that in the report rather than adding noise.

## Step 5 — Verify Before You Rewrite

For each comment you touch, check it against what the code does **now**:

- Does it describe behavior that still exists?
- Does it name parameters, return values, exceptions or fields that still exist under those names?
- Does it reference a file, function, ticket or URL that still resolves?
- Does the doc comment's declared types and thrown exceptions match the signature?

A stale comment gets **corrected**, not deleted — unless what it described is genuinely gone, in
which case say so in the report.

**When a comment and the code disagree, do not assume the comment is wrong.** Sometimes the comment
records the intended behavior and the code drifted — that is a bug, not a comment defect. Flag it
rather than quietly rewriting the comment to match a possible bug. That is the single highest-value
find this skill produces.

## Step 6 — Apply In Reviewable Batches

Comment changes are enormous in line count and near-zero in risk, which makes them uniquely easy to
hide a real change inside. Protect against that:

- **Batch by directory or module**, and commit each batch separately
- **Comments-only commits contain only comment changes.** Never mix a logic fix into a commenting
  pass — if you find a bug, report it separately or fix it in its own commit
- Verify the build and tests still pass after each batch — a broken doc-comment syntax, a removed
  pragma, or a mangled multi-line string will show up here
- Keep the diff reviewable; a thousand-file comment sweep in one commit gets rubber-stamped

For doc comments that feed a generator, **build the docs** after the batch and confirm they still
render.

## Step 7 — Report And Save State

`.audit/comment-report.md`:

- Files touched this run, and what changed by category (stale corrected, missing added, noise removed)
- **Comment/code contradictions** — the possible bugs, listed first
- Stale `TODO`s surfaced, not deleted
- Places where a rename would beat a comment
- What remains in the backlog and roughly how much

Then write state: `last_run`, `last_commit` (the SHA you audited to), the swept list, and the
remaining list. Every run, including partial ones.

## Cadence

This is a maintenance skill. Reasonable triggers: before a release, after a large refactor or a
model-written feature, when onboarding someone, or on a period the user picks. On each run it reports
how long it has been and what drifted since.

It stays inside the code. README, changelog, docs sites and wikis belong to `/auto-doc` — hand off
rather than overlapping, and say so when a comment change implies an external doc change.

## Rationalization Table

| Excuse | Reality |
|---|---|
| "This function has no comment, add one" | Only if there is a why to record. A restated signature is noise |
| "The comment is wrong, fix the comment" | The code may be wrong instead. Flag the contradiction |
| "This TODO is ancient, delete it" | Surface it. Deleting deferred work is forgetting, not tidying |
| "`# noqa` is just a comment" | It is a directive. Removing it breaks the lint gate or the build |
| "I'll tidy the copyright headers" | Legal text. Do not touch |
| "This file is generated but under-commented" | Do not edit generated files. Fix the generator or leave it |
| "I'll fix this small bug while I'm here" | Not in a comments-only commit. Separate it |
| "One big commit is easier to review" | It is easier to *not* review. Batch it |
| "The project has no style, I'll pick one" | Propose it and get confirmation before a thousand files |
| "I'll update state at the end of the whole job" | Update it every run, including partial. That is the resume point |

## Red Flags — Stop

- Removing or reformatting a lint directive, pragma, license header, or generated-file marker
- Deleting a `TODO`/`FIXME` rather than reporting it
- Rewriting a comment to match code without considering that the code may be the defect
- Mixing logic changes into a comments-only pass
- Adding comments that restate the code
- Editing files behind a `DO NOT EDIT` banner
- Running without reading state, or finishing without writing it
- A comment sweep that was not built or tested afterwards
- Imposing a house style on an established codebase without asking
