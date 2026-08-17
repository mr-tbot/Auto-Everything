---
name: auto-license-check
description: "Use when a codebase's licensing must be audited against how it will actually be released — before shipping, selling, open-sourcing, fundraising, or an acquisition; when the user asks about dependency licenses, GPL/AGPL/copyleft exposure, whether they can go closed-source, what license to pick, NOTICE or attribution files, \"/auto-license-check\"; or when about to declare a dependency's license fine, unknown, or unfixable."
---

# /auto-license-check

A deep audit of every license obligation attaching to this codebase, judged against how the user
actually intends to release it. Mission-critical and non-speculative: every finding cites the license
text or the package artifact it came from, and an unknown license is a **blocker**, never a guess.

**Not legal advice, and say so once.** You produce the inventory, the obligations, and the remedies
with their costs. Counsel decides what it means. Name license sections only when you have read them
in the actual license file — **never invent a section number, a case cite, or a clause.**

## Step 1 — The Intake (mandatory, ask before scanning)

The same dependency is fine in one release model and fatal in another. Do not scan until these are
answered — ask them as concrete options, not open questions:

1. **Release model** — open source / closed source / source-available / internal only / not decided.
2. **Intended license**, if open source, or "help me pick".
3. **Selling it?** — direct sale, subscription/SaaS, paid app, dual-licensed, free with paid support,
   or not commercial.
4. **How it reaches users** — SaaS the user never gets a copy of / downloadable binary / app store /
   OTA or sideload / library or SDK others embed / container image / on-prem appliance / firmware.
   **Answer per channel** — an app that ships to a store *and* to a test fleet has two answers, and
   they can differ.
5. **Who contributed** — solo, employees, outside contributors, contractors, AI assistance. Determines
   whether the user can relicense their own project at all.
6. **Deadline and trigger** — routine, pre-release, or diligence/acquisition.

Record the answers at the top of the report. Every finding is then judged against *that* target, and
a change to the target invalidates the audit.

If the user does not know their release model, that is the first deliverable — walk the four tiers
(permissive / weak copyleft / strong copyleft / source-available or proprietary) against their
selling and distribution answers, and get a decision before scanning.

## Step 2 — Scope The Distribution Boundary

Obligations attach on **conveying** — transferring a copy to another party — not on use, and not on
network topology.

- Use inside one legal entity: no conveying.
- To a subsidiary, affiliate, contractor, outsourced build vendor, JV partner, or an external tester
  via TestFlight / Firebase App Distribution / MDM: **that is conveying.** "Internal" means one legal
  person, not one org chart.
- App store, OTA, sideload, download: conveying. There is no app-store exemption.
- SaaS where no one receives a copy: not conveying — **except AGPL-3.0's network-use clause, which is
  exactly the trap this case walks into.**

Write down, per channel, what binary or bundle actually ships. That artifact is the audit's subject —
not the repo.

## Step 3 — Inventory, Cheapest First

Run in this order and stop escalating when the answer is clear. Do not open with ORT or FOSSology;
they are last resorts, not starting points.

**Tier 1 — the ecosystem's own resolver** (seconds, authoritative on *what* is present):

```bash
npm ls --all --omit=dev            # and: npm sbom --sbom-format cyclonedx
./gradlew :app:dependencies --configuration releaseRuntimeClasspath
cargo tree -e no-dev               # and: cargo deny check licenses
go list -deps -json ./... | jq -r .ImportPath
pip list --format=json             # better: uv pip list / poetry show --tree
swift package show-dependencies    # Package.resolved is the audit input
# CocoaPods: Podfile.lock is authoritative — read it. (`pod list` lists the whole
# spec repo, not your dependencies; it is not an inventory command.)
```

Separate **shipped** from **dev-only** here. A build tool that never enters the artifact carries no
distribution obligation, and counting it inflates the report into noise.

**Tier 2 — a fast license report** to triage the bulk. Eclipse `dash-license-tool` (ClearlyDefined-
backed, answers in seconds), `pip-licenses`, `license-checker`, `cargo about`, `go-licenses`. Exact
invocations vary by version — run `--help` and use what it prints rather than what you remember.
Known sharp edges: `cargo about generate` requires an `about.toml` that `cargo about init` creates;
`go-licenses` documents the `/v2` module path; `cargo deny`'s exit code is a **bitset** (0x1
advisories, 0x2 bans, 0x4 licenses, 0x8 sources), so `exit != 0` alone cannot tell you what failed.

**Tier 3 — file-content scanning** for anything Tier 2 reported as unknown, NOASSERTION, or
suspicious. This is the trustworthy layer:

```bash
# the binary is `scancode`, the package is `scancode-toolkit` — --spec is required
pipx run --spec scancode-toolkit scancode --license --copyright --json-pp scancode.json <path>
# or the modern pipeline runner: ScanCode.io
```

**Metadata vs content is the trust axis of this whole audit.** Package metadata is self-declared,
frequently wrong, and often absent. File-content detectors read the actual license text. When they
disagree, the file wins, and the disagreement itself is a finding.

Learn the SPDX distinction and use it in the report: **declared** (what the package claims) vs
**concluded** (what a scan of the files supports), and `NONE` (no license found) vs `NOASSERTION`
(not determined) — those two mean different things and collapsing them hides the blockers.

**Tier 4 — the shipped artifact itself.** This is where audits are won, because it is the only view
that includes what the resolvers never saw:

```bash
unzip -l app-release.aab | grep -iE 'META-INF|LICENSE|NOTICE'
unzip -l app-release.aab | grep 'base/lib/.*\.so'
bundletool build-apks --mode=universal --bundle=app.aab --output=app.apks   # expand before inspecting
```

For Android, **the AAB is what the store ingests**, not the APK, and Play re-signs and re-splits, so
the installed artifact is not byte-identical to the upload. Do **not** use Syft as an APK/AAB
inspector — its `apk` cataloger is Alpine Linux packages, not Android, and it will return a
near-empty SBOM that looks like success. Unzip and scan instead. For iOS, CocoaPods generates
`Pods-acknowledgements.plist`; SwiftPM has **no license field at all**, so resolved pins must be
checked by hand.

## Step 4 — What Every Tool Misses

Account for these by hand. They are where the real exposure lives:

- **Prebuilt native binaries** — `.so`, `.a`, `.aar`, `.framework`, `.dll`, vendored C/C++ trees.
  Version and license are compiled in and invisible to every scanner.
- **OpenSSL** — the most commonly bundled native library. ≤1.1.1 is the OpenSSL/SSLeay dual license
  with an advertising clause and is **GPL-incompatible**; 3.0+ is Apache-2.0. Which one is inside a
  prebuilt `.so` is version-pinned and invisible. BoringSSL is a further mixture with no single SPDX
  id.
- **FFmpeg's build-time license election** — default build is LGPL-2.1-or-later; `--enable-gpl` or
  linking x264/x265/xvid forces GPL-2.0-or-later; `--enable-nonfree` produces a binary that is **not
  redistributable at all**. `ffmpeg -version` prints the configuration line — that is the evidence,
  and it is a named audit step.
- **Fonts** — SIL OFL-1.1 requires the license ship with the font, forbids selling the font on its
  own (bundled in an app is fine), reserves Reserved Font Names, and keeps modified versions under
  OFL. Nothing in `fonts/` or `res/font/` is scanned by default.
- **Assets** — icons, images, sounds, 3D models, model weights. CC-BY-NC and "free for
  non-commercial use" land here and kill a commercial release.
- **Proprietary SDK terms** — Firebase, Google Maps Platform, AdMob, Meta SDK, analytics and crash
  reporters. These are not OSS licenses, no scanner detects them, and for a shipped app they are
  frequently the more dangerous obligations.
- **Store policy obligations**, which are independent of license obligations: Play requires
  attribution for Play services; Apple requires third-party license disclosure at review. Both can be
  violated while every OSS license is satisfied.
- **Inbound licensing of the user's own code** — whether a CLA or DCO exists. Without one, the repo's
  LICENSE file does not by itself cover outside contributions, and the user may be unable to
  relicense their own product.
- **Git submodules and copied snippets** — the latter is `auto-rewrite`'s job; hand it off rather than
  guessing.

## Step 5 — Judge Against The Target

For each shipped dependency, resolve in order:

1. **Which license actually applies.** Dual/multi-licensing means an **election** the user must make
   and record — an SPDX `OR` is a choice, not an ambiguity. `AND` means both apply. `WITH` means an
   exception modifies the base. Per-file licenses override the repo's LICENSE file.
2. **Does the obligation trigger** on this channel? (Conveying vs use vs network-use.)
3. **What must the user do** — retain notices, ship license text, provide attribution, provide
   Corresponding Source, permit relinking, keep modifications under the same license?
4. **Is it compatible** with the intended outbound license, in the right direction? Compatibility is
   one-way: permissive flows into copyleft, never the reverse. `GPL-2.0-only` and `-or-later` are
   different answers.

Two facts that decide most real cases and are usually missed:

- **Corresponding Source is not "a source tarball."** It is everything needed to generate, install
  and run the object code and to modify the work — including build scripts and interface definition
  files. A git URL does not discharge it. A written offer must stand for **three years** and, under
  GPL-2.0 §3(b), runs to **any third party**, not only the user's customers.
- **The System Libraries exception** decides whether a GPL app linking the platform libc, the NDK, or
  Apple's system frameworks must ship those as Corresponding Source. Read the clause in the actual
  license text before concluding either way — this is the single most consequential clause in a
  mobile audit.

Classify every dependency:

| Class | Meaning |
|---|---|
| **Blocker** | Incompatible with the stated release model. Ship = violation |
| **Obligation unmet** | Compatible, but attribution / notice / source / offer is missing |
| **Election required** | Dual-licensed; the user must choose and record it |
| **Unknown** | No license found, NOASSERTION, or metadata and files disagree. **Treated as a blocker until resolved** |
| **Clear** | Compatible, obligations already met |

## Step 6 — The Report

`.audit/license-report.md`, blockers first. Header states the intake answers, the channels audited,
the artifact inspected per channel, which tools ran, and **what was not covered**.

Per dependency: name, version, declared license, concluded license, where the evidence came from,
which channel it ships on, the obligation, the classification, and the remedy with its cost.

Never write a license conclusion you did not read the text for. "Probably MIT" is not an output —
`NOASSERTION, unresolved` is, and it is a blocker.

## Step 7 — Remediate

Stop at the first rung that clears it:

1. **Comply.** Usually the whole job: generate `NOTICE` / `THIRD-PARTY-LICENSES` with per-dependency
   copyright lines and full license text, wire it into an in-app "Licenses" screen, host the written
   offer where required. Offer to generate this — it is the single most common missing deliverable.
2. **Elect.** Pick the permissive side of a dual license and record the election in the SBOM and the
   notice file.
3. **Upgrade or reconfigure.** A newer version, or a differently-configured build (the FFmpeg case),
   often changes the license outright.
4. **Restructure the boundary.** Separate process, dynamic linking, or a service split can change
   which obligation attaches. Propose it, do not promise it — the answer is license-specific and
   fact-specific.
5. **Replace** with a compatibly-licensed equivalent.
6. **Relicense the user's own project** — only with every copyright holder's agreement, or a CLA that
   assigned the rights. Check the contributor list before offering this.
7. **Remove the feature.**

**If a violation is already live**, cure terms decide the strategy: GPL-3.0 §8 provides reinstatement
paths on ceasing the violation; GPL-2.0 §4 has **no cure period in its text** — though many major
GPL-2.0 holders, including the Linux kernel and the GPL Cooperation Commitment signatories, have
committed to GPL-3.0-style cure. Whether cure is available is therefore a **per-copyright-holder**
question, not a per-license one. Read the terms; do not assume either way.

## Offer To Set The Repo's License

When the user picks a license, changing it means changing **every** place it is asserted, or the
inconsistency becomes its own finding:

- `LICENSE` / `LICENSE.md` at the repo root, full text, correct year and holder
- The license field in every package manifest (`package.json`, `Cargo.toml`, `pyproject.toml`,
  `*.podspec`, `build.gradle`, `*.csproj`, `composer.json`)
- SPDX headers in source files, if the project uses them (`reuse lint` checks this)
- README badge and any license section
- The store listing, docs site, and website footer
- `NOTICE` / `THIRD-PARTY-LICENSES` for inbound dependencies
- A `CONTRIBUTING`/CLA/DCO decision for inbound contributions

Confirm the choice with the user before writing any of it, and never change an existing license
without explicit approval — that is not a cleanup, it is a legal act.

## "No Answer Exists" Is Not An Answer

Before reporting a dependency as unresolvable:

1. Read the actual license file in the package, not the metadata.
2. Check the upstream repository, its release tags, and its `NOTICE`.
3. Check ClearlyDefined for a curated concluded license
   (`curl -4 -s 'https://api.clearlydefined.io/definitions/<type>/<provider>/<ns>/<name>/<rev>'` — the
   `-4` matters, the host's IPv6 path black-holes).
4. Run a file-content scan on the extracted package.
5. Ask the upstream maintainer — a GitHub issue asking "what license is this under?" resolves it
   permanently and takes minutes.
6. Only then report it unresolved, with what you tried, and treat it as a blocker.

Report a genuine limit explicitly and with evidence. Never downgrade an unknown to "probably fine".

## Rationalization Table

| Excuse | Reality |
|---|---|
| "package.json says MIT" | Metadata is self-declared and often wrong. Scan the files |
| "It's open source, so it's free to use" | Every OSS license has conditions. Some are incompatible with selling |
| "We're SaaS, copyleft doesn't reach us" | AGPL's network-use clause is written for exactly that case |
| "It's internal use only" | Internal means one legal entity. Contractors and external testers are conveying |
| "It's just a dev dependency" | Verify it never enters the artifact. Then it is genuinely out of scope |
| "GPL apps are on the App Store, so it's fine" | Store presence is not a legal analysis. Check the terms for the actual channel |
| "Only one file is GPL" | Copyleft scope is about the combined work, not the file count |
| "We'll add attribution later" | Later is after distribution, which is after the obligation attached |
| "The scanner said 0 issues" | Check what it scanned. Prebuilt natives, fonts, and assets are invisible to it |
| "Unknown license, probably permissive" | Unknown is a blocker. No license means no permission |
| "We can relicense our own code" | Not without every contributor, absent a CLA |
| "Nobody enforces this" | Enforcement is not the standard, and diligence will find it before a court does |

## Red Flags — Keep Working

- Scanning before the intake questions are answered
- One answer for a project that ships through more than one channel
- A license conclusion drawn from metadata alone
- "Probably", "typically", or "usually" in a finding — this audit does not speculate
- A cited license section you did not read in the license file
- Reporting clean without opening the shipped artifact
- No accounting for prebuilt natives, fonts, assets, or proprietary SDK terms
- Changing a LICENSE file without explicit approval
- Treating an unknown license as anything other than a blocker
