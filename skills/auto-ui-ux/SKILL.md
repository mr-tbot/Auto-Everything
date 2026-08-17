---
name: auto-ui-ux
description: "Use when a project's UI needs to be brought to a professional finished state — when it was built in phases by different people or models and now has competing design idioms, when screens are inconsistent or half-finished, when the user asks to audit/polish/unify/fix the UI or UX, mentions design cohesion, accessibility, contrast, spacing, dark mode, buttons or controls that do nothing, or says \"/auto-ui-ux\"; or when about to declare a UI done without looking at it."
---

# /auto-ui-ux

Bring a codebase's interface to a finished, coherent, professional state — correctness, completeness,
and cohesion — and keep iterating until a full pass finds nothing.

The failure this exists for: a project built in phases, each phase by a different hand with a
different idea of what the UI should be. Two button components, three greys that are nearly the same
grey, one screen with empty states and five without. Nothing is broken enough to file, and the whole
thing reads as unfinished.

## Step 0 — Mode, Scope, Freeze

**Pick a mode.** Ask once, then commit:

- **INTERACTIVE** (default) — you ask before each decision that has taste in it: palette, type, assets,
  which idiom wins. Best when the user has opinions, brand assets, or a client.
- **AUTO** — you derive the target system from what is already in the codebase and align everything to
  it on your own judgment, asking nothing. Best for a backlog of drift, or an unattended pass. AUTO
  still reports every judgment call it made, and still stops for anything on the freeze list.

**Establish the freeze list before touching anything:** screens, components, or brand elements that
must not change. Anything the user is proud of, anything a client signed off, anything with an
established visual identity. Absent instruction, the freeze list includes the logo, brand colors, and
any screen the user names.

**Cohesion beats novelty.** You are extending an existing design language, not inventing one. Small
feature-driven tweaks are yours to make. A new design system, a font change, a palette change, or
restyling components that already work is a **proposal**, not an action — even in AUTO mode.

## Pass 1 — Inventory And Drift

Enumerate the whole surface before changing any of it: every screen/route/view, every component, and
every entry point. Then extract every hardcoded design value:

```bash
# colors: hex, rgb/rgba, hsl, and platform literals
rg -n --no-heading -e '#[0-9a-fA-F]{3,8}\b' -e 'rgba?\(' -e 'hsla?\(' -e 'Color\(0x' -e 'UIColor\(' -e 'Color\.[a-z]'
# spacing, radius, shadow, z-index
rg -n -e '\b(margin|padding|gap|inset)[^:]*:\s*[0-9]' -e 'border-radius|cornerRadius|clipShape' -e 'box-shadow|elevation|shadowRadius' -e 'z-index'
# type
rg -n -e 'font-family|fontFamily|FontFamily|\.font\(' -e 'font-size|fontSize|textSize|\.sp\b|\.dp\b'
```

Tally, don't just list. Forty distinct greys and six near-identical blues *is the finding*. Then hunt
the structural drift:

- Two components doing one job (two buttons, two modals, two date pickers, two toast systems)
- Mixed idioms in one product — Tailwind beside styled-components, Material widgets beside Cupertino,
  two icon sets, two date formats, two casing conventions in labels
- Screens that never got the treatment the others did
- Components with no dark, hover, pressed, focus, or disabled variant while their siblings have them

**Exclude before you count**: generated files, vendored UI kits, dead code that no route reaches,
per-brand theming that is intentional, and dark-mode variants — a second color set is not drift.

Record the inventory in `.audit/ui-inventory.md`. It is the checklist every later pass runs against.

## Pass 2 — Choose The Target System

**INTERACTIVE:** ask, with concrete options rather than open questions. A non-designer cannot answer
"what look do you want" but can absolutely pick between two rendered choices. Ask for:

- References and **anti-references** ("nothing like X" is the more useful answer)
- Brand assets: logo (SVG if it exists — raster logos cap the quality ceiling), icon set, fonts and
  **whether they are licensed for embedding**, imagery and its rights
- Palette direction, or approval of a palette you derived
- Density and tone: compact/comfortable, playful/serious

When the user has no assets, say what you will use instead and get one confirmation — do not stall the
whole pass on a logo.

**AUTO:** derive the target from evidence, in this order:

1. **Frequency and prominence** — the color/type/spacing values that appear most, weighted toward the
   screens users actually reach.
2. **Recency** — `git log` the design files. The newest coherent idiom is usually the intended one.
3. **Load-bearing-ness** — the component with the most imports and the widest route coverage wins over
   the prettier one used twice.
4. **Brand invariants survive regardless** — brand colors and logo are never "outvoted" by frequency.

State the derived system and the reason each choice won, then apply it.

**Consolidating values** — the actual math, so this is defensible rather than vibes:

- Cluster colors by perceptual distance, not RGB distance. Use a real implementation —
  `culori`'s `differenceCiede2000()` (a curried factory: `differenceCiede2000()(a, b)`) or
  `colorjs.io`'s `deltaE(c, '2000')`. **Do not hand-implement CIEDE2000** — the hue-wrap and rotation
  terms are where implementations diverge. Note `chroma-js` before v2.2.0 named a different metric
  `deltaE`, so old comparisons are not comparable.
- Snap spacing to a scale, but **never snap a hairline** — a 1px border is not a 4px border.
- Type: pick a scale and keep the existing hierarchy's *relationships*, not just its values.

Do not over-tokenize. A genuinely one-off value stays a one-off value.

If the project has a token format, use it. **The W3C DTCG format has no concept of theme or mode** —
multi-mode is handled outside the format with parallel files sharing a key structure. Do not invent a
`$modes` property.

## Pass 3 — Correctness

Exact thresholds, cited. **WCAG 2.2 Level AA is the operative benchmark** — APCA is not the WCAG 3
contrast method (visual contrast was removed from WCAG 3 drafts in 2023 and its algorithm is
undetermined), so do not audit against it.

Contrast ratio is `(L1 + 0.05) / (L2 + 0.05)` with L1 the lighter relative luminance; range 1:1–21:1.

| Check | Requirement | Level |
|---|---|---|
| Text contrast (1.4.3) | 4.5:1 normal, 3:1 large | AA |
| "Large" text | ≥18pt / 24 CSS px, or ≥14pt bold / 18.66 CSS px bold | — |
| Non-text contrast (1.4.11) | 3:1 — on the pixels that **identify the component or its state**, not every border or decorative divider | AA |
| Target size (2.5.8) | 24×24 CSS px, with the spacing exception | AA |
| Target size enhanced (2.5.5) | 44×44 CSS px, no spacing exception | AAA |
| Focus visible (2.4.7) | `outline: none` with no replacement is a failure; use `:focus-visible`, indicator needs 3:1 | AA |
| Reflow, resize, text spacing | 1.4.10, 1.4.4, 1.4.12 | AA |

Platform minimums are separate obligations from WCAG: **Apple 44×44 pt**; **Android/Material 48×48 dp**
(≈9mm physical). Android states its contrast guidance in **sp**: below 18sp, or bold below 14sp,
requires 4.5:1.

**Measure every theme.** There is no dark-mode success criterion because every contrast criterion
applies independently to each color scheme you ship. Light-mode-only checking is the classic miss.

Automate what automates:

```bash
npx --yes @axe-core/cli http://localhost:3000 --tags wcag2a,wcag2aa,wcag21aa,wcag22aa
npx --yes pa11y-ci --sitemap http://localhost:3000/sitemap.xml
```

Valid axe tags include `wcag2a`, `wcag2aa`, `wcag21a`, `wcag21aa`, `wcag22aa`, `best-practice`, and
per-criterion tags like `wcag258`. There is no `wcag22a` and no `wcag21aaa`. axe returns
**`incomplete`** rather than `pass` when it cannot resolve a background (image, layered transparency) —
those are manual checks, not passes.

Native: Flutter encodes the platform minimums directly (`androidTapTargetGuideline` 48×48,
`iOSTapTargetGuideline` 44×44, `textContrastGuideline`), though its contrast check partitions colors
naively and should be treated as a smoke test. iOS has
`XCUIApplication.performAccessibilityAudit()` (Xcode 15+), which fails the test on findings with no
explicit assertion. Android's Accessibility Scanner runs the same Accessibility Test Framework as the
Espresso/Compose checks.

**State the honest coverage in the report:** automation catches roughly 20–30% of WCAG *success
criteria*, or ~57% of issue *instances* on real pages by Deque's measurement — the two numbers have
different denominators and neither means "mostly covered". Keyboard order, focus management, labeling
sense, and whether the flow is usable are manual.

## Pass 4 — Wiring: Does The Control Actually Do Anything?

A UI audit that only judges appearance will pass a screen full of buttons that do nothing. Every
interactive element gets traced from the control to the effect.

For each control, name the handler, then name what the handler actually reaches — a state change, a
network call, a file write, a rendered frame. **A control whose trace ends in nothing is broken**, no
matter how good it looks. The recurring shapes:

- An `onClick` / `onPress` bound to an empty lambda, a `TODO()`, a no-op, or a handler that only logs
- A handler wired to the wrong target — the copy of the function that is no longer called, an older
  duplicate component, a stale route
- A control bound to state nothing reads, or reading state nothing writes (the config-parity diff in
  the previous pass finds the settings version of this)
- A form that validates and never submits; a submit that posts to a dead endpoint
- A feature flag defaulting off, so the control renders and is inert
- A navigation target that no longer exists, or a deep link no route claims
- A callback parameter left null, defaulted, or dropped through a refactor
- A control disabled by a condition that can never become true

Find them by tracing, not by looking:

```bash
# handlers that are empty, TODO, or log-only
rg -n -B2 'on(Click|Press|Tap|Change|Submit)\s*=\s*\{\s*\}' 
rg -n 'on(Click|Press|Tap)\s*=\s*\{[^}]*\b(TODO|FIXME|Log\.|console\.log|print)\b[^}]*\}'
# declared-but-never-referenced handlers and routes
rg -n 'fun handle[A-Z]\w*|const handle[A-Z]\w*' -o | sort -u   # then grep each name for a call site
```

Then **drive it**. Static tracing finds the obvious cases; the subtle ones only appear when you click
the control and read the logs. Anything that survives Pass 6's capture step gets exercised, not just
photographed.

**When a break is found:**

1. Report it as a wiring defect, separate from cosmetic findings — it is a different severity and a
   different fix.
2. Offer to correct it, with the specific fix named: which handler, which target, what it should
   reach.
3. **Then run `/auto-audit` before continuing the loop.** A disconnected control is rarely alone —
   it means a wiring pass was skipped somewhere, and auto-audit is the skill that traces runtime paths
   end to end across the whole project rather than screen by screen. Fold its findings back in and
   re-enter this loop at Pass 1.

A UI pass that fixes the paint on a dead button has made the product worse: it now looks finished.

## Pass 5 — Completeness

Per screen, the state matrix. A screen missing these is unfinished no matter how it looks:

**initial · loading · partial · empty · error · offline · permission-denied · unauthenticated ·
rate-limited · success**

Then per flow:

- **Navigation** — Android back (gesture and hardware), iOS swipe-back, browser back, deep links,
  state restoration after process death, modal dismissal rules, unsaved-changes guards
- **Forms** — validation timing, inline errors, error focus management, keyboard types, autofill,
  submit-on-enter. **A disabled submit button is an anti-pattern**: it is typically unfocusable, often
  skipped by screen readers, exempt from contrast rules so it renders illegibly, and it removes the
  mechanism that would have told the user what is wrong.
- **Destructive actions** — confirmation, and undo where undo is possible
- **Cross-cutting** — dark mode, RTL, localization and string expansion, dynamic type / font scaling,
  safe areas, keyboard avoidance, breakpoints, landscape, tablet, reduced motion, first-run and
  empty-database states
- **Every interactive element** has hover, pressed, focus, and disabled states — or a documented reason
  it does not

**Config parity:** every config option gets a UI surface in the existing settings pattern, and
file-based config stays in sync with it — unless the user says an option is deliberately file-only.
An option that exists in a config file and nowhere in the UI is an unfinished feature.

## Pass 6 — Apply, In Reviewable Batches

Never one giant diff. Batch by screen or by token — whichever produces a diff a human can actually
approve — and checkpoint between batches. Back up before editing per the project's convention, and
log what changed so a single change can be reverted on its own.

Token migrations go through codemods where the codebase supports it, staged per directory, verified
per batch. A find-and-replace across a design system is how you discover that three of those greys
were load-bearing.

## Pass 7 — Look At It

A UI change is not done because the code changed. Capture and **actually view** the result, for every
theme and breakpoint you touched:

```bash
npx playwright screenshot --viewport-size=390,844 http://localhost:3000/ shot-mobile.png
# in tests: await expect(page).toHaveScreenshot('home-light-1280.png', { fullPage: true, maxDiffPixelRatio: 0.01 })
adb exec-out screencap -p > android.png
xcrun simctl io booted screenshot ios.png   # simulator — use a real device where policy requires it
flutter test --update-goldens
```

Headless Chrome's `--screenshot` works **only** in headless mode — the same command without
`--headless` writes no file and exits silently — and never captures beyond the window box.

Capture from real hardware where it is attached and where the project's device policy requires it —
a simulator renders fonts, safe areas and scaling differently enough to hide exactly the bugs this
pass is looking for.

Kill the flake sources before diffing: disable animations, mask timestamps and dynamic content, pin
device scale factor, install the fonts. A green visual diff where both sides are broken is not a pass.

Lay captures side by side across screens to judge cohesion — that comparison is the only way drift is
visible, because each screen looks fine alone.

## The Anti-Slop Constraints

An AI "redesign" has a recognizable smell. If the result has these, it is worse than what it replaced:

- System default fonts, or the same three fonts every generated site uses
- Unmotivated gradients, glassmorphism, or a purple-to-blue hero
- Emoji standing in for icons
- Centered hero + three feature cards, regardless of what the product does
- Motion with no purpose, or inconsistent durations and easing
- A palette with no relationship to the brand that was already there
- Cosmetic polish over screens whose actual problem is a missing empty state

The constraint that prevents all of it: **derive from what exists, change the minimum that achieves
cohesion, and justify every choice against evidence in the codebase or an answer from the user.**

Never restyle something that already works to match a preference nobody expressed.

## Iterate Until Finished

Loop: inventory → target → correctness → **wiring** → completeness → apply → look → inventory again.
Stop when a full pass finds nothing. One pass never does — the second pass always finds screens the
first one changed inconsistently.

**If any pass found a wiring break, the loop is not done until `/auto-audit` has run and its findings
are folded back in.** A dead control is evidence that something upstream was never connected.

**Finished** means: one visual language across every screen; the state matrix filled on every screen;
AA thresholds met in every theme; platform target minimums met; every interactive element has its full
state set **and demonstrably does something**; and you have *looked at* every screen you touched, in
every theme and breakpoint.

## Rationalization Table

| Excuse | Reality |
|---|---|
| "The code change is correct, so the UI is fine" | You have not seen it. Capture it and look |
| "It looks fine on my viewport" | You shipped breakpoints, themes, and locales. Check them |
| "axe passed" | Automation covers 20–30% of criteria. `incomplete` is not `pass` |
| "Dark mode is just inverted colors" | Every contrast criterion applies again. Measure it twice |
| "This screen isn't important" | It is in the product. Unfinished screens are what "unfinished" means |
| "I'll unify the components later" | Later is another phase, which is how the drift got here |
| "The user didn't mention the empty state" | Users report the polished thing, not the missing one |
| "A redesign would be cleaner" | Cohesion with what exists beats novelty. Propose, don't impose |
| "AUTO mode means I don't explain" | AUTO means no questions, not no reasoning. Report every judgment |
| "Disabled the button until the form is valid" | Anti-pattern — unfocusable, unreadable, and it hides the error |
| "It's a 1px border, snap it to the scale" | Hairlines are not spacing. Some values are intentional |
| "The button renders correctly" | Rendering is not wiring. Trace the handler to a real effect |
| "The handler exists, so it works" | It may target the copy nobody calls. Follow it to the end |
| "I'll note the dead control and move on" | A dead control means a wiring pass was skipped. Fix it, then run /auto-audit |

## Red Flags — Keep Working

- Claiming a UI change works without a screenshot you looked at
- Auditing contrast in one theme only
- Restyling components on the freeze list, or changing fonts/palette without sign-off
- A single enormous diff across the whole design system
- Consolidating tokens without perceptual-distance math behind the clusters
- Reporting "polished" while screens still lack empty, error, or offline states
- Deriving a target system in AUTO mode without stating why each choice won
- Polishing a control without tracing what it actually does
- Finding a wiring break and continuing the loop without running /auto-audit
- Ending after one pass
