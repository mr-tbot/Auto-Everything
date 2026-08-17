---
name: auto-doc
description: "Use when documentation must be brought back in line with the code — README, CHANGELOG, docs sites, API docs, wikis, GitHub/Forgejo repo surfaces, and external project-management tools; when the user asks to update docs, check whether docs are stale, verify the changelog, audit feature claims, or says \"/auto-doc\"; or before a release, when shipping a UI or API change, or when about to call documentation current without checking."
---

# /auto-doc

Find every place this project describes itself, check each against what the code actually does now,
and fix what has drifted.

**The standing law: a project's documentation is part of its surface area.** When the product
changes, the docs that teach it are stale until proven otherwise. A stale tutorial is not cosmetic —
someone follows it, it does not match, and that becomes a support ticket, a bad review, or a refund.

## Step 1 — Enumerate The Doc Surface

Four categories. Miss a category and the audit reports clean on documentation nobody checked.

**In-repo:** README, CHANGELOG, `docs/` trees, doc-site sources (Docusaurus, MkDocs, Sphinx,
VitePress, mdBook, Starlight, Jekyll), ADRs, `CONTRIBUTING`, `SECURITY`, code comments and docstrings,
`--help` output, man pages, and generated API docs (TypeDoc, Sphinx autodoc, rustdoc, Dokka, DocC,
godoc, Doxygen, OpenAPI).

**Repo host:** description, topics, homepage, the wiki, Pages, Releases and their notes, issue/PR
templates, `CODEOWNERS`, community-health files, social preview, labels, milestones, Projects.

**External:** Notion, Confluence, Jira, Linear, GitBook, Outline, BookStack, and friends. Detect which
from inside the repo — URLs in README/CONTRIBUTING/templates, ticket-ID patterns (`PROJ-123`) in
commit messages and branch names, CI integrations, `.env` keys, MCP server config — rather than
asking cold.

**In-product prose, which is the surface everyone forgets:** onboarding copy, empty-state text, error
messages, menu labels, and their translations. `strings.xml`, `Localizable.strings`, `.arb`, i18n
JSON. A removed feature leaves its menu label behind, and when the English string finally gets fixed,
forty locales keep the old promise.

Store listings count too, and they are the highest-severity claim source — a false claim on a store
page outranks a wrong line in a tutorial. Pull them into diffable files rather than eyeballing the
console: `fastlane deliver download_metadata` for App Store Connect, `fastlane supply init` for Play.

## Step 2 — Pin The Claim Corpus

Before comparing anything, decide **which version the docs are supposed to describe.** A README on a
release tag describing the released version is correct; the same README judged against `main` looks
wrong. Pin it explicitly — `git show v2.3.0:README.md` versus the working tree — and say in the report
which basis you used. Skipping this manufactures drift findings for a project doing it right.

## Step 3 — Mechanical Freshness

These are cheap, objective, and catch most rot. Run them first.

**Dead links** — `lychee` is the fastest and handles non-Markdown too:

```bash
lychee --cache --max-cache-age 1d --max-retries 3 './**/*.md'
npx linkinator ./docs --recurse --check-fragments --retry --retry-errors
```

Set `GITHUB_TOKEN` for `lychee` or GitHub links rate-limit into false failures. `linkinator`'s
`--check-fragments` catches anchors that moved, which is the failure a plain link check misses.

**Code samples that no longer run** — a doc sample that does not compile is a bug report waiting to
be filed:

```bash
pytest --doctest-modules --doctest-glob='*.rst'
cargo test --doc
mdbook test
npx embedme --verify "docs/**/*.md"    # samples must match the real source file
```

**References that no longer resolve:** documented paths that do not exist, documented CLI flags the
parser no longer accepts (walk `--help` and diff), documented env vars absent from the code,
documented endpoints absent from the router, and screenshots older than the UI they show.

**Version agreement** across README badge, package manifest, git tag, doc-site config, and the
in-product about screen. Disagreement here is common and trivially checkable.

**Docstring coverage** where the project cares: `interrogate --fail-under 95 src`, `sphinx-build -b
coverage` — and if you use the Sphinx coverage builder, set `coverage_modules` explicitly. Left
empty it only measures modules already reachable in the doc tree, so a completely undocumented module
is not reported as uncovered; it simply is not counted, and the audit returns clean by measuring
nothing.

## Step 4 — Feature-Claim Parity, Both Directions

Extract every claim the project makes about itself, map each to code evidence, then do the reverse.

Code-evidence extraction, per stack: route enumeration (`flask --app x routes --sort match
--all-methods` — `--sort match` matters, because when two rules overlap the claim is only true for
whichever matches first), CLI subcommand walking, public API surface (`cargo public-api`, `go doc`,
`api-extractor run`, `griffe`, japicmp, and for Kotlin `./gradlew apiDump` / `apiCheck`), UI entry
points, and feature-flag inventories.

Sort every claim into one of seven buckets:

| Bucket | Meaning |
|---|---|
| **PRESENT** | Claim maps to code evidence. Fine |
| **ABSENT** | Claimed, no implementation found. Fix the docs or build the feature |
| **STALE** | Implementation was removed, claim survived. Date it: `git log -S'<symbol>' --oneline -- <path>` — a claim whose implementation died three releases ago is worse than one that died last week |
| **ORPHANED** | Implementation exists, no user can reach it. This is a *reachability* question, not a dead-code question — trace from real entry points (main, routes, Activity/Composable, CLI commands), because "nothing calls this" and "no user can get here" are different |
| **CONDITIONAL** | True behind a flag, a platform, a paid tier, or a config. Not a defect — an unlabeled one is |
| **ASPIRATIONAL** | "Coming soon", "beta", "planned for 2.5". The defect is future tense written as present tense, or a roadmap date that has passed |
| **UNDOCUMENTED** | Exists and is user-reachable, documented nowhere |

Note that `cargo-semver-checks` and similar run against the **default feature set** — a feature-gated
API is exactly where a claim audit goes wrong, so run per shipped feature combination and union the
results.

## Step 5 — Config Parity, With The Guard Rails

Every config option should be documented *and* surfaced in whatever settings UI the product has, and
file-based config should agree with the UI. The diff is easy; getting it *right* is where this goes
wrong, so these guard rails are mandatory:

```bash
# 1. extract both sides — pick the recipe that matches the actual project layout
# legacy Android Views preferences:
rg -PoN --no-filename '(?:android|app):key="\K[^"]+' app/src/main/res/xml/ | LC_ALL=C sort -u > ui_keys.txt
# Compose + DataStore: keys are declared, then referenced by IDENTIFIER, never by string literal
rg -PoN '(?:boolean|int|string|long|float|double|stringSet)PreferencesKey\("\K[^"]+' src/ | LC_ALL=C sort -u > code_keys.txt

# 2. MANDATORY: an empty side manufactures a finding for every key on the other side
[ -s ui_keys.txt ] && [ -s code_keys.txt ] || { echo "extraction produced nothing — wrong path or wrong project layout"; exit 1; }

# 3. compare in the same collation you sorted in
LC_ALL=C comm -13 ui_keys.txt code_keys.txt   # config the code reads that no UI exposes
LC_ALL=C comm -23 ui_keys.txt code_keys.txt   # UI controls that write config nothing reads
```

That non-empty assertion is not paranoia. Run the legacy-Views recipe against a Compose+DataStore
project and it yields zero UI keys, `comm` dutifully reports **every** config key as unexposed, and
the report reads as a catastrophic finding when the truth is you grepped the wrong layout. The empty
result is silent — `rg` exits 1 on no match and an empty file looks like a legitimate answer.

Where keys are bound to identifiers (`val LOW_LATENCY = booleanPreferencesKey("low_latency")`, used as
`SettingsKeys.LOW_LATENCY`), diff **identifiers**, not string literals, or every call site looks like
a gap.

`\K` requires `rg -P`; without it you get a parse error and exit 2 — loud, and any exit-code check
catches it. The silent killer is the wrong path, which is what the assertion above is for.

## Step 6 — Changelog And Release Notes

Keep a Changelog defines exactly six change-type headings — **Added, Changed, Deprecated, Removed,
Fixed, Security** — plus `Unreleased` at the top. Anything else is drift.

SemVer: `feat` → MINOR, `fix` → PATCH, a `!` or `BREAKING CHANGE:` footer → MAJOR. Build metadata is
ignored for precedence. And SemVer's rule 3 — *once released, the contents of that version MUST NOT be
modified* — is the spec basis for the hard rule below: **never rewrite a historical changelog entry.**
Add a new entry that corrects it.

If the project uses a generator (`release-please`, `changesets`, `git-cliff`, `towncrier`,
`semantic-release`), drive the generator rather than hand-editing its output, or the next run reverts
you.

Check that every shipped version has an entry, that entries describe user-visible change rather than
commit subjects, and that a release actually exists for each tag.

## Step 7 — The Repo Host Surface

Read and update with the CLI rather than the web UI so the change is reproducible:

```bash
gh repo view --json description,homepageUrl,repositoryTopics
gh repo edit --description "..." --homepage "..." --add-topic ...
gh release list && gh release view v1.2.0
gh api repos/{owner}/{repo}/community/profile --jq '.files | keys'
```

The **wiki is a separate git repository** at `https://github.com/OWNER/REPO.wiki.git` — clone, commit,
push like anything else. It must be initialized through the web UI once before that URL exists, so if
the clone fails with "repository not found", that is why, and it is a thing to ask the user to do
rather than a bug to work around.

Forgejo/Gitea have their own API with the same shapes; use it when that is where the repo lives.

## Step 8 — External Systems: Ask First

Reading an external wiki or tracker is fine. **Writing to one is a visible action in a shared space —
confirm before the first write, every session.** Mass-editing tickets, restructuring a team's
Confluence space, or posting to Discussions on your own initiative is how this skill would make
enemies.

If credentials are missing, ask for a token with the minimum scope needed and name the scope. Do not
hunt the filesystem for one.

## Write Versus Check — Label Every Command

Doc tooling loves commands that quietly rewrite tracked files. Always know which mode you are in:

| Tool | Check (read-only, safe in CI) | Write (mutates tracked files) |
|---|---|---|
| terraform-docs | `--output-check` | `--output-mode inject` / `replace` |
| api-extractor | `api-extractor run` | `api-extractor run --local` |
| generated CLI docs | regenerate then `git diff --exit-code` | regenerate in place |
| embedme | `--verify` | bare invocation |

`terraform-docs --output-mode replace` overwrites a README's entire contents. Run the check form
first, show the diff, then write.

## The Output Contract

Every finding is one record, so findings can be diffed release over release:

```
claim_id · verbatim claim text · claim source file:line · bucket ·
evidence file:line (or "none") · precondition (for CONDITIONAL) ·
severity (from claim source: store listing > README > docs site > code comment) ·
proposed fix · the gate that would prevent recurrence
```

Write it to `.audit/doc-report.md`, highest severity first, with the coverage statement at the top:
what was enumerated, what was pinned to which version, what was excluded, and what remains unsampled.

## Triage — Rank The Work, Not Just The Findings

A README with 300 bullets and two store listings cannot be exhaustively verified on every run. Order
the work: **purchase-decision claims first** (store listing, pricing, top-of-README), then anything a
user follows step-by-step (onboarding, tutorials, quickstart), then reference material, then the long
tail — sampled.

Then say so. "Top 40 claims verified, tail unsampled" is an honest report. Silence about the tail
reads as a clean bill of health for documentation nobody looked at.

## Keep The Gate

A doc fix without a gate comes back. When you fix a finding, add the check that would have caught it —
a link-check job, a `--doctest` run in CI, an `apiCheck` gate, a doc-site build in the release
workflow. Name the gate in the finding record.

## Rationalization Table

| Excuse | Reality |
|---|---|
| "The docs build, so they're fine" | Building proves syntax, not truth |
| "I updated the README" | That is one of four surfaces. The wiki, the store listing and the in-app copy also claim things |
| "No links are broken" | Link checking is the cheapest check, not the audit |
| "The feature list looks right" | Run the reverse direction. Undocumented features are half the finding set |
| "The parity diff shows 49 gaps" | Check both inputs are non-empty first. An empty side fabricates a gap per key |
| "Translations can wait" | Translated prose is prose users read. A removed feature still promises itself in forty locales |
| "I'll fix the old changelog entry" | Released versions are immutable. Add a correcting entry |
| "It's documented, just behind a flag" | Then it is CONDITIONAL and the precondition belongs in the doc |
| "'Coming soon' is a claim we can leave" | Only if it is labeled as future tense and its date has not passed |
| "I'll update the team wiki while I'm here" | Shared space. Ask first |
| "The docs describe the release, not main" | Correct — so pin the corpus and say which basis you used |

## Red Flags — Keep Working

- Reporting clean after checking only the README
- A parity diff run without asserting both inputs are non-empty
- Comparing against `main` when the docs describe a release, or the reverse, without saying which
- Rewriting a historical changelog entry
- Running a write-mode doc generator before showing the check-mode diff
- Writing to a wiki, tracker, or Discussions without asking
- Force-pushing a wiki, or rewriting notes on a published release
- No coverage statement, so partial work reads as complete
- Fixing findings without adding the gate that prevents recurrence
