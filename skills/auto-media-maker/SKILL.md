---
name: auto-media-maker
description: "Use when a project needs product video made or kept current — onboarding, tutorials, walkthroughs, launch stingers, social cutdowns, release-notes clips; when the user mentions HeyGen, avatar video, demo video, tutorial video, product ads, or says \"/auto-media-maker\"; or when code has changed and the videos that teach it may now be stale."
---

# /auto-media-maker

Build and maintain a project's video series — and keep track of which videos the code has since
outrun.

**The standing law: a project's video series is part of its surface area.** When the product changes,
the videos that teach it are stale until proven otherwise. A stale tutorial is not cosmetic — someone
follows it, it does not match, and that becomes a support ticket or a refund.

The hard part is not making video. It is that six weeks later the login screen moved, the tutorial
still shows the old one, and nobody notices until a customer follows it.

## Step 1 — Find Or Establish The Pipeline

**Ask for the media pipeline repo on first run** — the repo holding credentials, project configs and
video state, kept separate from the product repo so keys never live beside code. Persist its path in
`.agent/media.json` (gitignored) so later runs skip this.

If a pipeline exists, **read its conventions and drive its CLI — do not build a parallel one.** A
mature pipeline typically provides:

| Command | Purpose |
|---|---|
| `doctor` | tools + credentials present |
| `init <slug> --repo PATH` | register a project |
| `add-video <proj> <slug> --kind …` | scaffold script, memory, build |
| `status` / `check` / `plan` | inventory, what the code outran, the work order |
| `render` / `build` / `bless` | narration, assemble, mark "matches the code as of now" |

with per-project config (brand, avatar/voice ids, `voice_and_copy` including a `never_say` list, demo
account, capture rig, project-wide `watch` paths) and per-video config (kind, segments, each segment's
`watch` paths and its `on_change` action).

If no pipeline exists, offer to create the minimum: a project config, a per-video config with
per-segment `watch` paths, a script file, and a content-hash record. **The tracking is the point** —
video without it rots invisibly.

## Step 2 — First-Run Interview

Ask in this order, and ask concretely.

**Avatar and voice.** Present the actual available avatars and voices from the account rather than
describing them — `assets`-style listing, then let the user pick. Male/female, and the voice
separately, because the pairing is a taste call. Get one avatar+voice per project unless the user wants
different presenters per video kind.

**Style, per type, separately** — tutorials and advertisements are different crafts:

- **Tutorial / onboarding**: clean, calm, professional. Screen-led. The presenter serves the screen.
- **Advertisement / stinger**: the one that is allowed to be striking. Rhythm, contrast, a real idea.

Ask each set separately, then offer *"use the same look for both"* as an explicit option. Cover: pace,
register, music presence, caption style, aspect ratio, and length target.

**Brand.** Ask for a media folder — logos, colors, fonts, footage, reference films, and
**anti-references** ("nothing like this"), which are usually the more useful answer.

**Copy rules.** Capture the `never_say` list verbatim — banned claims, retired taglines, legal
phrasing. **Check every script against it before spending a single credit**, because a banned phrase
caught in text costs a keystroke and caught in a delivered cut costs a re-render, a rebuild, and a
re-upload everywhere it is embedded.

## Step 3 — Plan The Series

Walk the catalog and decide what this project needs — onboarding, payment, admin/web panel, feature
tutorials, stinger, social cutdowns, release notes. **Record the skips and why** ("no payment video,
product is free"), so nobody re-asks in four months.

Work in priority order: **onboarding first, payment second, operator surfaces third, feature tutorials
after, stinger last.** Onboarding and payment are the ones that cost money when they are wrong.

## Step 4 — Cut For Small Future Lifts

This is the decision that determines whether the next change is minutes or an afternoon.

- **One beat per idea, one `watch` list per beat.** A segment declares the source files whose rendered
  surface it shows. When those files change, that segment — and only that segment — is flagged.
- **Each segment declares its own `on_change` action**: re-shoot the footage, re-voice the narration,
  both, or flag for human review.
- **Narration is per-line and content-addressed**, so an unchanged line is never re-rendered. Never
  work around that.
- **Never overwrite a shipped deliverable.** New cut, new `vN` filename. The old file is the evidence
  for why the new one exists.

## Step 5 — Write Scripts That Do Not Sound Like Advertising

Show the thing working, say what it does, stop talking.

**Banned by default**: hype adjectives, urgency, stacked imperatives, zinger closes, infomercial
cadence, "imagine if", "in today's world", and rhetorical questions to open. If a line would sound
wrong said aloud by the person who built the thing, it is wrong.

Write for the ear: short sentences, one clause of new information at a time, and a beat of silence
after anything the viewer needs to read on screen. Tutorials earn attention by being useful, not by
being energetic. Advertisements earn it with a real idea in the first two seconds — not with volume.

Then check the script against `never_say` before rendering.

## Step 6 — Capture And Cut, Steadily

Three standing constraints, each learned from a series that shipped wrong:

**Steady.** A UI beat gets **no push**. `zoompan` truncates its crop origin to whole pixels each frame,
so a slow zoom moves the picture in 1px jumps — that is what "the screens all shake" means.
Supersampling before the zoom reduces it but is damage control, not permission. If a beat feels dead,
fix it upstream: a window where something actually happens, then a shorter beat, then a cut to a
different surface. Push is the last resort and never above **1.03–1.05**.

**Unhurried.** Target **0.85×–1.05×**. Slowing down is free; speeding up never is — sped-up UI is
exactly what a viewer calls rushed, and the frame-rate conversion adds judder on top. **Never fix an
overlong window by speeding it up.** Cut a shorter window or shoot again. Hold on a result long enough
to read it.

**Capture clean.** Script the drive; never hand-scroll during a take. Hide the cursor unless it is the
subject. Let the UI settle before and after every action. Never film production writes. Never show real
customer data, a real card number, or a placeholder domain — not blurred, not briefly.

## Step 7 — Additional Tools, On The First Pass Only

Offer to extend the toolkit **during first-run setup**: image generation for thumbnails and poster
frames, video generation for abstract B-roll, music and sound design, and voice alternatives.

Rules that keep this from becoming slop:

- **Set these up on the first pass only.** On later runs, do not add tools unless the user asks.
- **Generated B-roll is seasoning, not substance.** A product video's job is showing the product. Where
  a real screen recording exists, it beats generated footage every time.
- **Check the commercial-use terms before anything ships**, especially for music: several services'
  cheaper tiers exclude paid advertising, and the grant differs by plan. Confirm per service, per plan.
- **Verify the current model and pricing before using any of them** — this area moves monthly, so read
  the provider's current docs rather than relying on a remembered model name.

## Step 8 — Watch It, Then Bless It

**Never bless a build you did not watch, with sound, all the way through.** That is the rule that
keeps the whole system honest: blessing an unwatched build teaches the tracker that a broken video is
correct.

Then record the version, and write the memory entry — newest first: what changed and **who asked for
it**, every deliberate compromise (a mocked screen, a placeholder name, a state that cannot exist on
the bench), and which takes were rejected and why. Undocumented compromises get "fixed" by the next
person and break the cut.

## Step 9 — Every Later Run: Re-Check, Re-Cut Only What Moved

**Keep the video task list current on every run — this is not optional.** The task list is the skill's
memory, and a run that does not update it has broken the tracking that justifies the pipeline.

1. Run the pipeline's `check` — which videos, which beats, which files moved.
2. Read the plan and work each flagged beat by its declared action.
3. **A diff is not automatically a re-cut.** Refactors, renames and formatter sweeps move files without
   moving anything visible. Look at the screen; if nothing a viewer can see changed, **re-bless with a
   note saying so** — that is a correct outcome, not a workaround. The fingerprint is a tripwire, not
   an oracle.
4. Re-render only changed lines, re-shoot only changed windows.
5. Watch, bless, bump the version, write the memory.

Also handle the cases `check` cannot see: copy and legal language changes invalidate work that looks
fine, and **screen copy is copy** — a retired claim visible in footage is exactly as much of a problem
as one that is spoken.

## Cost

Renders cost credits; captures cost time. Before a big rebuild, say roughly what it will cost and offer
the smaller version. A one-word fix in a twelve-beat tutorial should be minutes. **If a proposed job is
not shaped like that, question the shape before spending.**

## Rationalization Table

| Excuse | Reality |
|---|---|
| "The file changed, re-cut the beat" | Look at the screen first. Invisible diffs get a re-bless with a note |
| "It renders fine, ship it" | Never bless a build you did not watch, with sound |
| "The window is long, speed it up" | Never. Cut a shorter window or shoot again |
| "The beat feels static, add a push" | Stillness is the look. Fix it upstream; push is the last resort |
| "One more hype line will help" | Show the thing working, say what it does, stop |
| "I'll re-render the whole script" | Only changed lines. That rule is what makes maintenance cheap |
| "I'll overwrite the old cut" | New version, new filename. The old file is the evidence |
| "Generated B-roll will look impressive" | The product is the subject. Real screens beat generated ones |
| "The music is from an AI tool, it's fine" | Check the plan's commercial terms. Several exclude paid ads |
| "I'll update the task list at the end" | Update it every run. The tracking is the product |
| "Nobody will notice the old screen" | Someone follows it. That is the support ticket |

## Red Flags — Stop

- Rendering before checking the script against `never_say`
- Blessing a cut you did not watch end to end with sound
- Speeding footage up to fit, or pushing a UI beat past 1.05
- Re-rendering unchanged narration lines
- Overwriting a shipped deliverable instead of cutting a new version
- Filming production writes, real customer data, or a placeholder domain
- Adding new generation tools on a maintenance run without being asked
- Shipping generated music or footage without checking commercial terms for that plan
- Finishing a run without updating the video task list
- Publishing anywhere without explicit approval
