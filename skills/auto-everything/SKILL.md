---
name: auto-everything
description: "Use when a project should be taken from wherever it is to genuinely shippable in one pass — issues triaged and fixed, code audited, UI and docs and media and infrastructure caught up; when the user says do everything, full pass, get this release-ready, run all the audits, or \"/auto-everything\"; or when several of the Auto-* skills would each apply and running them in the right order matters."
---

# /auto-everything

Run the whole family in dependency order, carrying findings forward, until the project is genuinely
shippable — and stop at the line where shipping is your decision.

**Why order matters.** These skills feed each other. Fixing a wiring break changes what the UI audit
sees. Replacing a dependency changes the license report. Any code change invalidates documentation and
video. Running them in the wrong order means doing the same work twice and reporting stale findings.

## Step 0 — Balance First, Always

Start with **`/auto-balance`**. It sizes this session's parallelism, claims any devices needed, and
decides whether the machine can take the work at all. Everything after it is expensive; finding out
mid-build that three other sessions are compiling is the wrong time.

If balance says the machine is saturated, the correct move is to wait or reduce scope — say so rather
than pressing on.

## Step 1 — Intake: What Are We Actually Fixing?

Run **`/auto-issue-fix`** in **read-only intake mode** first. Pull the reported problems from every
tracker and crash reporter, classify them, and merge them into one work list. Do not write anything
back yet — replies come after fixes are real.

This front-loads reality: the backlog decides the pass's scope, and a crash affecting users outranks
anything you would have found by browsing.

Then agree the scope with the user: everything, or this release only.

## Step 2 — Provenance And Licensing, Before You Write Code

Run **`/auto-rewrite`** and **`/auto-license-check`** now, not later, because both can force
*architectural* changes. Discovering after a week of polish that a core dependency is AGPL, or that a
load-bearing file was copied from an incompatible project, invalidates everything built on top.

These two run cleanly in parallel — they share no state and touch nothing.

Their blockers become work items ahead of everything else.

## Step 3 — Make It Actually Work

Run **`/auto-audit`**. This is the spine of the pass: every claimed feature traced to a real runtime
path, on every platform the project targets, iterating until a pass finds nothing.

Fix the intake list from Step 1 here too — with a regression test per fix, per auto-issue-fix's
discipline. A fix without a failing-then-passing test is a guess.

**Do not proceed while auto-audit is still finding defects.** Everything downstream describes,
photographs or deploys this code; documenting a broken build produces accurate documentation of a
broken build.

## Step 4 — Interface

Run **`/auto-ui-ux`**. Its wiring pass will surface dead controls — and **a wiring break sends you back
to Step 3**, because a disconnected control means a wiring pass was skipped somewhere. That loop is
part of the design, not a detour.

Run **`/auto-brand-parity`** alongside it. They overlap at the edges (icons, palette, contrast of brand
colors used as accents) and are best reconciled together rather than fighting each other later.

## Step 5 — Tell The Truth About It

Only now, with the code settled, do the descriptive passes — anything earlier documents a moving
target.

- **`/auto-doc`** — README, changelog, docs site, wikis, feature claims, in-product copy, store
  listings.
- **`/auto-media-maker`** — its `check` names exactly which videos and which beats the code outran.

These parallelize well. Both are cheap to re-run if Step 3 reopens.

## Step 6 — Infrastructure

Run **`/auto-web`** — read-only first, always. It reports whether what is deployed matches what you
now have, and what would have to change.

It sits here because deploying is downstream of everything: there is no point reconciling
infrastructure against code that is about to change again.

## Step 7 — Close The Loop

Return to **`/auto-issue-fix`**, now in **write mode**, under its gates:

- Reply to each report with the actual fix and the commit or PR that carries it.
- Close what is genuinely fixed, with the honest `state_reason`.
- File new issues for what this pass found and did not fix.
- Update the trackers.

Per-item approval still applies. A batch pass is exactly the situation that produces mass-comment spam,
so the gates matter more here, not less.

## Step 8 — The Handoff

Produce one consolidated report, `.audit/everything-report.md`:

- What each skill found and what was fixed
- What remains, ranked, with an estimate
- Every deliberate compromise, and why
- What could not be verified, and what it would take
- **The ship checklist**: what is ready, and what is explicitly waiting on your decision

**Then stop.** Nothing in this family pushes, tags, submits to a store, deploys, or posts publicly on
its own. The pass ends with a recommendation and a clean tree, and shipping is a separate, explicit
instruction.

## Iterate

One pass is never enough — every pass so far has found real defects the previous one missed, including
in code written earlier in the same session.

Re-enter at the earliest step whose inputs changed:

| What changed | Re-enter at |
|---|---|
| Code fixed | Step 3, then everything downstream |
| A wiring break found | Step 3 |
| A dependency swapped | Step 2 |
| UI changed | Step 4, then docs and media |
| Only prose changed | Step 5 |

Stop when a complete cycle finds nothing new. Report the cycle count.

## Running It Sanely

**Checkpoint between steps.** Commit at each boundary so a step can be reverted alone. A single
enormous diff across eleven concerns is unreviewable, and unreviewable work does not get reviewed.

**Report progress as you go**, not only at the end. A pass this long that goes silent is
indistinguishable from a pass that hung.

**Budget honestly.** Before starting, estimate roughly what the full pass costs in time and credits and
offer a scoped-down version — release-critical only, or a single platform. `/auto-media-maker` and
`/auto-audit` are the expensive ones.

**Respect every sub-skill's gates.** Running under one command does not upgrade anyone's permissions.
Anything the individual skill would have asked about, this one asks about too — store-review replies,
production changes, license changes, publishing. The orchestration is a convenience, never a
consent-laundering mechanism.

## Rationalization Table

| Excuse | Reality |
|---|---|
| "I'll do docs first, they're easy" | You will document code that is about to change |
| "Licensing can wait until the end" | An AGPL dependency is architectural. Find it before you build on it |
| "auto-audit still has findings, but let's proceed" | Everything downstream describes this code. Fix it first |
| "One big commit at the end is cleaner" | It is unreviewable, and it cannot be partially reverted |
| "The user said do everything, so I can ship" | Everything ends at ready. Shipping is a separate instruction |
| "Running under one command, so I don't need to ask" | Gates are per action, not per invocation |
| "One pass is enough this time" | It never has been. Run the cycle until it comes back empty |
| "The machine is busy but I'll start anyway" | Balance runs first for a reason |
| "I'll batch all the issue replies at the end" | That is the mass-comment pattern. Per-item approval still applies |

## Red Flags — Stop

- Starting without `/auto-balance`
- Documenting or filming code that `/auto-audit` still has open findings against
- Running licensing or provenance after the implementation work
- Writing to trackers before the fixes are real and verified
- One giant uncommitted diff spanning every concern
- Treating the orchestration as blanket approval for a sub-skill's gated action
- Declaring the pass complete after one cycle
- Shipping anything — that is never this skill's call
