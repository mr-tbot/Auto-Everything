# Auto-Everything

A family of agent skills that refuse to let a project be called finished before it demonstrably is.

Each one takes a different definition of "done" and holds the line on it: the code actually runs, the
dependencies are actually licensed, the interface actually works, the docs actually match, the
infrastructure actually agrees with the repo. They work standalone, and `/auto-everything` runs the
whole set in dependency order.

They exist because agents — and people on a deadline — are good at producing work that *looks*
finished. A feature that compiles and moves zero bytes. A button that renders and does nothing. A
README describing a flag that was renamed two releases ago. Nothing is broken enough to file, and the
whole thing quietly isn't done.

## The skills

| Skill | Refuses to let you ship |
|---|---|
| **auto-audit** | A feature that compiles but moves zero bytes. Iterative audit → fix → build → verify → adversarial review, across every platform the project targets — the machine the agent runs on included — until a pass finds nothing |
| **auto-rewrite** | Code that came from somewhere else. Duplication sweep, provenance search against the archives that actually hold the world's source, git forensics — then a remediation ladder where rewriting is the fifth option, not the first |
| **auto-license-check** | A dependency whose license nobody read. Asks how you actually release before it scans, opens the shipped artifact rather than trusting metadata, and treats an unknown license as a blocker |
| **auto-ui-ux** | An interface built in phases that never got unified — and controls that render beautifully while doing nothing. Drift, WCAG 2.2 AA in every theme, the full state matrix, and a wiring pass that traces every control to a real effect |
| **auto-doc** | Documentation, changelogs, wikis and feature claims the code has outrun — including the in-product copy and the forty locales still promising a removed feature |
| **auto-web** | Deployed infrastructure that has drifted from the repo. Read-only first, and it never touches production on its own |
| **auto-brand-parity** | Icons, logos and brand assets that disagree across Windows, macOS, Linux, iOS, Android and the web |
| **auto-media-maker** | Onboarding, tutorial and launch video that silently went stale when the product moved |
| **auto-balance** | Sessions that fight each other for CPU, RAM and hardware. Coordinates parallel builds, arbitrates devices, and sizes its own agent count so you never hit account limits |
| **auto-issue-fix** | Reported bugs sitting unread across GitHub, Jira, Linear, Slack, ClickUp and your crash reporter. Finds them, fixes them with a regression test, replies with the fix, updates the tracker — reading freely, writing only under gates |
| **auto-everything** | All of the above as one pass, in dependency order — issues in, code fixed, docs and media caught up, nothing shipped without your say-so |

## What they have in common

**Evidence, not assertion.** "Finished" means the runtime path demonstrably works, with the log line,
the hash, or the screenshot to prove it. Not that it compiles, not that a comment says so.

**Exact commands, verified.** Every command, threshold and API shape in these skills was checked
against the tool or the spec — several were executed against real endpoints — and then put through an
adversarial pass whose only job was to refute them. That pass caught fabricated case citations, a
parity recipe with a 100% false-positive rate, resource limits that silently do nothing, and mutating
commands documented as read-only. An invented flag is the entire failure mode for skills like these.

**They stop before shipping.** Every one of them reports ready and hands the decision back. None
pushes, tags, submits, deploys, or posts publicly on its own.

**"It's impossible" is a research task, not a conclusion.** Each skill has an explicit section on what
to try before reporting a limit, and requires evidence — the specific missing API, the entitlement not
granted — rather than an assumption.

## Install

Either install the whole family through
[BOT-CODE-MODS](https://github.com/mr-tbot/BOT-CODE-MODS), which also sets up a persistent system
prompt across Claude Code and VS Code Chat:

```bash
git clone https://github.com/mr-tbot/BOT-CODE-MODS.git
cd BOT-CODE-MODS && bash install.sh          # Windows: powershell -ExecutionPolicy Bypass -File .\install.ps1
```

Or install these skills alone:

```bash
git clone https://github.com/mr-tbot/Auto-Everything.git
cd Auto-Everything
for s in skills/*/; do ln -sfn "$PWD/$s" ~/.claude/skills/$(basename "$s"); done
```

Runtimes using the cross-runtime alias take `~/.agents/skills/` instead. Skills load when a session
starts, so restart your agent afterwards.

## Usage

Invoke any of them by name:

```
/auto-audit          /auto-doc            /auto-balance
/auto-rewrite        /auto-web            /auto-issue-fix
/auto-license-check  /auto-brand-parity   /auto-everything
/auto-ui-ux          /auto-media-maker
```

Or let them trigger themselves. Each description fires on the moment it applies — when an agent is
about to declare a feature done, wave through a license it never read, call a UI finished without
looking at it, or report documentation current without checking.

## Structure

```
skills/<name>/SKILL.md    # one self-contained skill per directory
```

That is the whole project. Adding a skill is adding a directory.

## License

MIT
