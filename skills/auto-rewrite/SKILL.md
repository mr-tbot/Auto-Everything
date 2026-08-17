---
name: auto-rewrite
description: "Use when a codebase must be provably the user's own work — before a release, sale, funding round, acquisition, or open-sourcing; when code may have been copied from another project, Stack Overflow, or regurgitated by an AI; when the user says \"check for plagiarism\", \"is this code original\", \"clean-room\", \"copyright risk\", \"/auto-rewrite\"; or when about to declare copied code fine, or impossible to replace."
---

# /auto-rewrite

Find code in this repo that came from somewhere else, prove it either way, and give the user real
options for clearing it. Two questions per suspect region, in order:

1. **Is it actually copied?** — evidence, not vibes.
2. **If so, what is the cheapest lawful remedy?** — usually not a rewrite.

**Not legal advice, and say so once.** You are doing engineering triage: locating code, establishing
provenance, and costing remedies. A lawyer decides what it means. Name doctrines and cases by name
only — **never invent a page cite, a docket number, or a section number.** A fabricated citation
destroys the credibility of a report whose entire value is credibility.

## Step 0 — Scope And Trigger

Ask what prompted this, because it sets the depth and the deadline:

| Trigger | Depth |
|---|---|
| Routine hygiene | Passes 1–3, report, no escalation |
| Shipping / open-sourcing soon | Full sweep + remediation applied |
| Acquisition, funding, or enterprise diligence | Full sweep. Say plainly that the acquirer will run a commercial snippet scanner (Black Duck, FOSSID/Revenera, FOSSA, Snyk, Mend) against a far larger proprietary corpus than anything free, so treat your result as a floor, not a ceiling |
| A rightsholder has already made contact | **Stop. Escalate before writing findings down** — see Escalation |

Then write down the scope: which paths are the user's own work, which are declared third-party
(`vendor/`, `third_party/`, `Pods/`, `node_modules/`, `external/`), which are generated, which are
test fixtures. Exclusions get stated in the report — a silent exclusion reads as a clean bill of
health for code nobody looked at.

Declared third-party code is out of scope **here** and in scope for `auto-license-check`. The two
skills split on the question: this one asks "did we copy it", that one asks "may we use it".

## Pass 1 — Internal Duplication Sweep

Cheap, local, no network. Finds copy-paste inside the repo and gives you the hot files to
investigate externally.

```bash
npx --yes jscpd@5 . \
  --min-tokens 50 --min-lines 5 \
  --format typescript,kotlin,swift,java,python,go,rust \
  --ignore "**/node_modules/**,**/build/**,**/vendor/**,**/*.min.*,**/generated/**" \
  --reporters console,json --output ./.audit/jscpd --blame
```

`--format` and `--ignore` are not optional — pointed at a repo root unfiltered, jscpd lexes logs,
Markdown and JSON and reports them as duplication. `--blame` attaches authorship, which is the part
you actually want. jscpd finds Type-1 and Type-2 clones (exact, and renamed identifiers); a copy
that was edited afterwards slips through.

Cross-check with a second engine — agreement between a token-based and a line-based detector is a
real signal:

```bash
pmd cpd --minimum-tokens 100 --language kotlin --dir src \
  --ignore-identifiers --ignore-literals --format xml --report-file .audit/cpd.xml
```

PMD CPD takes **one `--language` per run** and defaults to `java`, so a polyglot repo needs one
invocation per language and a wrong flag yields a confident-looking zero. Exit code 4 means
duplications were found, not that the run failed.

For "did *our* tree come from *that* tree", use the reference-directory shape:

```bash
pipx run --spec copydetect copydetect -t ./our-src -r ./upstream-src -b ./boilerplate -O .audit/copydetect.html
```

**Do not use MOSS.** It uploads your source to a third-party server, is licensed for non-commercial
educational use, and publishes matches at an unlisted public URL. Dolos (`npm i -g @dodona/dolos`)
does the same winnowing algorithm locally. **CodeQL has no clone detection** — those queries were
removed in 2.8.3 and only ever worked on the retired LGTM service; any instruction to run them is
citing dead functionality.

## Pass 2 — External Provenance

Does this code exist upstream? Run cheapest-first.

**Software Heritage — the exact-match oracle.** ~350M archived origins including dead forges. The
lookup key is `sha1_git`, **not** `sha1` — the same file returns opposite answers under the two
hashes:

```bash
H=$(git hash-object suspect.c)
curl -s -X POST -H 'Content-Type: application/json' \
  -d "[\"swh:1:cnt:$H\"]" https://archive.softwareheritage.org/api/1/known/
```

Batch up to **1000 SWHIDs** per POST (1200 returns HTTP 413); anonymous limit is **120 requests per
hour per IP**. `known: true` on a file the user claims to have written is a strong copy signal. Two
hard limits: it is **exact-match only** — one reformat, one renamed variable, one stripped header and
it returns false — and it will not tell you *where* the file came from, because
`/api/1/provenance/whereis/` requires allowlisted permission and returns 401 to everyone else. Pull
the archived bytes to diff with `/api/1/content/sha1_git:<h>/raw/`.

**ClearlyDefined — turns "looks like lodash" into "IS lodash 4.17.21 `package/_apply.js`".** It
publishes a per-file hash manifest for published packages:

```bash
S=$(sha256sum suspect.js | cut -d' ' -f1)
curl -4 -s 'https://api.clearlydefined.io/definitions/npm/npmjs/-/lodash/4.17.21' \
  | jq --arg s "$S" '.files[] | select(.hashes.sha256==$s)'
```

The `-4` is load-bearing — the host's IPv6 address black-holes and the request hangs to timeout.
Coverage is published packages only; code lifted straight out of a git repo has no definition.

**Sourcegraph — best free cross-forge full-text and regex search:**

```bash
curl -sG --max-time 60 --data-urlencode 'q=context:global "DISTINCTIVE LITERAL" count:100 fork:yes archived:yes' \
  --data 'v=V3' --data 't=literal' https://sourcegraph.com/.api/search/stream
```

`fork:yes archived:yes` is mandatory for this job — forks and archived repos are excluded by default,
and the fork is exactly where copied code usually lives. Index is ~2M repos, so **absence is not
evidence**.

**GitHub code search** works but is sharper-edged than it looks: authentication required, its own
10 requests/minute bucket, a 1000-result cap, files over 384 KB unindexed, default branch only, no
regex — and **scoping qualifiers silently return zero** (`repo:`, `user:`, `org:` and `path:` queries
returned 0 even for GitHub's own documented examples). Use it for unscoped global searches; do not
use it to confirm a hit inside a named repository. Expect occasional HTTP 408 on broad phrases and
retry.

```bash
gh search code 'DISTINCTIVE LITERAL' --limit 50 --json repository,path,textMatches
gh api /rate_limit --jq '.resources.code_search'
```

**ScanOSS** matches snippets against an OSS knowledge base. It transmits fingerprints, so use
`scanoss-py scan --obfuscate` on client or NDA'd code, and generate fingerprints locally first
(`scanoss-py fingerprint -o`) if you want to inspect what leaves the machine.

## Pass 3 — Local Forensics

Signals that survive when search finds nothing:

- **Git pickaxe.** `git log -S'<literal>'` finds additions/removals of a string; `git log -G'<regex>'`
  matches diff text and therefore catches blocks that were edited afterwards. `-S` silently misses an
  edited copy. Add `--all --full-history -- <path>` for files deleted and re-added,
  `--find-copies-harder` for moves.
- **Import events.** A single commit adding thousands of lines with no incremental history, no
  authorship trail, and a message like "add feature" is the shape of a paste.
- **Style divergence.** A file whose brace style, naming, comment voice, error handling, or tab width
  disagrees with everything around it. Formatters mask this — check the commit that introduced it.
- **Header archaeology.** Copyright lines, SPDX tags, license blocks, `@author` tags, TODOs naming a
  different product, non-English comments, URLs in comments, and — the loudest of all — a stripped
  header (`git log -p` the file's first commit).
- **AST-level diff.** `difft --display side-by-side ours.c theirs.c` compares structure, so it
  survives reformatting and reordering that defeat `git diff --ignore-all-space`.
- **Stack Overflow vintage.** SO answers are CC BY-SA, and the version is fixed by the answer's post
  date: **2.5** before 2011-02-08, **3.0** from 2011-02-08 to 2018-05-02, **4.0** after. That is a
  share-alike attribution license landing in a proprietary codebase — a real finding, not a nitpick.

## Triage — Do Not Manufacture Findings

A false accusation is worse than a missed one. Before a match becomes a finding, rule out:

- **Not copyrightable in the first place**: short snippets, standard algorithms, API names and
  signatures, facts, and expression dictated by function (merger) or by convention (scènes à faire).
- **Common idiom**: the pattern every project in that framework produces. Test it — search the
  literal; tens of thousands of hits means idiom, not theft.
- **Generated or scaffolded**: `create-react-app`, protobuf output, Room/Core Data boilerplate, IDE
  templates, Gradle wrappers.
- **Direction of travel**: the user's project may be the upstream. Compare first-commit dates
  (`gh api graphql` for `createdAt`) before assuming which way it flowed.
- **Same author**: the user may have written it elsewhere first — which raises an ownership question
  (employment, contractor terms, IP assignment) rather than a copying one, and that is often the real
  issue.

Classify every survivor:

| Class | Meaning |
|---|---|
| **Confirmed copy, compliant** | Copied, license permits it, obligations met. No action |
| **Confirmed copy, non-compliant** | Copied, obligations unmet (attribution, license text, copyleft). Remediate |
| **Confirmed copy, incompatible** | License cannot coexist with the user's release model. Blocker |
| **Probable copy, unresolved** | Strong signal, no upstream identified. Investigate or escalate |
| **Cleared** | Ruled out above. Say why in the report — this is the useful half |

## The Report

One file, `.audit/provenance-report.md`, ordered by severity. Per finding: file and line range, what
matched, the evidence (hash, URL, search that found it, commit SHA), the suspected upstream and its
license, the classification, and the remediation options with a real cost estimate for each.

State the coverage honestly at the top: what was scanned, what was excluded, which tools ran, and
what those tools structurally cannot see. A provenance report that implies completeness it does not
have is the failure mode.

## Remediation Ladder

Stop at the first rung that clears the problem. A rewrite is the *fifth* option, not the first.

0. **Ask for permission or a license.** Cheapest and most overlooked. But check contributor count and
   whether a CLA exists first — on a multi-contributor project with no assignment, no single
   maintainer can grant it.
1. **Confirm it is a problem at all.** Uncopyrightable, licensed-compatible, or already permitted?
   Then there is nothing to fix.
2. **Comply.** Most permissive licenses want attribution and license text — add `NOTICE` /
   `THIRD-PARTY-LICENSES`, restore the header, record it in the SBOM. Minutes of work.
3. **Isolate.** Move the code behind a boundary that changes the obligation (a separate process, a
   dynamically-linked module, a service). Whether this works is license-specific and lawyer territory
   — propose it, do not promise it.
4. **Replace.** Swap in a differently-licensed equivalent library. Usually cheaper than a rewrite and
   the result is maintained by someone else.
5. **Clean-room rewrite.** See below.
6. **Remove the feature.** Always on the table; sometimes correct.

**What does not clear anything** — and an agent will be tempted by all four: renaming variables,
reordering functions, reformatting, translating to another language, or feeding the original to an
LLM and shipping its output. Each produces a derivative work while destroying the evidence that you
knew. The LLM path has a second edge: output with no human authorship may not be protectable as the
user's own, so they can end up with code that is *both* derivative of the original *and* unownable.

**A rewrite fixes the forward product only.** It does not extinguish liability for copies already
built, shipped, or distributed. Say this plainly whenever you propose rung 5 — users universally
assume otherwise.

## Clean-Room Procedure

Two roles, and the wall between them is the entire point:

1. **Analyst** reads the tainted code and writes a *specification*: inputs, outputs, behavior,
   invariants, edge cases, wire formats, performance envelope. No code, no pseudocode that mirrors
   structure, no variable names, no comments carried across.
2. **Reviewer** checks the spec contains only functional requirements — the filtration step. Anything
   that is expression rather than function gets struck.
3. **Implementer** has never seen the original, works only from the approved spec, and says so in
   writing.
4. **Record it as you go**: who saw what, when, which spec version, which commits. Contemporaneous
   records are what make the claim credible later.

You can act as Analyst *or* Implementer in a session, never both. If you have already read the
original in this conversation, you are the Analyst — the implementation goes to a fresh agent or
person with the spec only, and you say so rather than quietly writing it yourself.

## Escalation And Privilege

Escalate to counsel — before writing findings down — when: a rightsholder has made contact, a
copyleft license is in a shipped proprietary binary, the copied code is load-bearing to the product,
an acquisition or diligence process is live, or the source is a former employer.

**Why "before":** a self-generated, dated analysis concluding the user's code matches upstream is
discoverable, and is close to ideal evidence that they knew. Work performed at the direction of
counsel may be privileged; the same work done alone is not. This does not license destroying
anything — preserve what already exists — it governs what *new* documents get created and under whose
direction.

Two more traps worth naming in the report: stripping a copyright header can be an independent wrong
(removal of copyright management information), and stamping the user's own copyright onto a vendored
third-party subtree can be a worse one (false CMI). "Clean up the headers" is not a safe reflex in
either direction.

## "It Can't Be Rewritten" Is Not A Finding

Every capability in a copied file was implemented by someone from a specification, and the
specification is what is being reimplemented. Before reporting a rewrite as infeasible:

1. Identify what the code actually does — the observable contract, not its structure.
2. Look for an alternative implementation. If a differently-licensed library does the same job, rung
   4 beats rung 5.
3. Check whether the difficulty is the *algorithm* (usually not copyrightable) or the *expression*
   (rewritable by definition).
4. Only then report the real constraint — a patent, a trade secret, a certification tied to the
   binary, a spec under NDA — explicitly and with evidence.

## Rationalization Table

| Excuse | Reality |
|---|---|
| "It's only a few lines" | De minimis is qualitative, not a line count. Short and distinctive still counts |
| "It was on Stack Overflow, so it's public" | Public ≠ unlicensed. SO answers are CC BY-SA — share-alike, with attribution |
| "It's on GitHub with no LICENSE file" | No license means no permission granted. That is the most restrictive case, not the least |
| "I renamed everything" | Renaming produces a derivative work and evidence you knew |
| "I'll have the LLM rewrite it" | Still derivative, and the output may not be protectable as the user's |
| "The rewrite fixes it" | Fixes the forward product. Copies already distributed remain what they are |
| "Nobody will ever find it" | The acquirer runs a commercial snippet scanner against a bigger corpus than yours |
| "The search found nothing, so it's clean" | Every index here is partial and mostly exact-match. Absence is not evidence |
| "This tool says 0 duplicates" | Check the invocation. Wrong `--language`, a file-list argument, or an over-broad `--ignore` all produce a confident zero |
| "I'll just strip the headers" | That is an independent wrong, and it deletes your innocent-infringement mitigation |
| "We can just relicense it" | Not without every copyright holder, unless a CLA assigned the rights |
| "It's impossible to replace" | Name the specific constraint. Otherwise it is a research task |

## Red Flags — Keep Working

- Reporting "clean" when only Pass 1 ran, or when the searches all returned zero
- A finding with no hash, URL, or commit as evidence
- A citation with a page number, docket number, or section number you did not verify
- Proposing a rewrite before checking whether the license already permits the use
- Renaming, reformatting, or LLM-laundering a file and calling it remediated
- Writing a detailed match analysis after learning a rightsholder is already asking
- Excluding a directory from the sweep without saying so in the report
- Concluding "impossible to rewrite" without naming the constraint
