# Auto-Audit

A single agent skill that refuses to let a project be called finished before it actually is.

`auto-audit` drives an iterative **audit → fix → build → verify → adversarially review → audit again**
loop across every platform a project ships to, and keeps looping until a complete pass turns up
nothing. It treats "that capability doesn't exist yet" as the start of a research task rather than a
stopping point, and it treats "it compiles" as evidence of nothing at all.

It works with Claude Code, and with any other agent runtime that reads skills from `~/.agents/skills/`.

## Why

Agents are good at producing code that compiles, launches, renders a UI, and moves zero bytes. The
failure shape recurs: a class registered against an interface whose method silently returns nothing,
a callback wired nowhere, a counter incremented on the wrong side of the failure so the metrics look
healthy. All of it survives a build, a launch, and a smoke test.

The second failure shape is stopping early — one clean-looking pass, one verified platform,
extrapolated to the rest, and a "known limitation" quietly written into the docs instead of raised
with the person who owns the decision.

Auto-Audit is written against both. It is a discipline skill: a loop, a definition of "finished", a
rationalization table, and a red-flag list the agent can self-check against when it catches itself
negotiating.

## What it does

- **Enumerates the targets first.** Every platform the project ships to — mobile OSes, desktop OSes,
  browsers, server runtimes, architectures, containers, embedded boards, CLI shells, distribution
  channels — written down as the checklist every later stage runs against. The host machine the agent
  happens to be running on counts as a target, not as a neutral build box.
- **Audits per platform branch.** Any `#ifdef`, `Platform.OS`, feature detection, or per-target build
  flavor gets audited on each side. One branch working says nothing about its sibling.
- **Defines "verified" per target.** A physical device for mobile, each supported engine for browsers,
  a fresh install consumed by a scratch project for libraries, the powered board for embedded. Logs
  read, and the line that proves data moved quoted back.
- **Refuses quiet degradation.** A genuine platform limit gets reported explicitly, with the specific
  missing API as evidence. An unreachable target gets reported unverified, never counted as done.
- **Leaves shipping to you.** After a clean pass it lists the project media the new work invalidated
  and reports ready. Pushing, tagging, and store submission wait for explicit approval.

## Install

Clone the repo and symlink the skill into your agent's skills directory:

```bash
git clone https://github.com/mr-tbot/Auto-Audit.git
ln -s "$PWD/Auto-Audit/skills/auto-audit" ~/.claude/skills/auto-audit
```

For runtimes that use the cross-runtime alias, symlink into `~/.agents/skills/` instead. Copying the
directory works just as well if you would rather not track updates.

Skills are discovered when a session starts, so restart your agent afterwards.

## Usage

Invoke it directly:

```
/auto-audit
```

Or let it trigger on its own. The skill description fires on phrasing like "finish this", "don't stop
until it's done", "100% complete", "keep doing passes" — and, more usefully, whenever the agent is
about to declare a feature done, a capability impossible, a platform unsupported, or a build ready to
ship.

## Structure

```
skills/auto-audit/SKILL.md    # the whole skill
```

That is the entire project. The skill is self-contained by design.

## License

MIT
