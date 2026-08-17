---
name: auto-brand-parity
description: "Use when a product's brand must be consistent everywhere it appears — app icons across Windows, macOS, Linux, iOS, Android and the web, favicons and PWA manifests, store listings, splash screens, logos, colors and fonts; when the user mentions branding, icons, logo, favicon, app icon, brand assets, or says \"/auto-brand-parity\"; or when about to generate an icon set or call branding consistent without comparing platforms."
---

# /auto-brand-parity

Establish what this product's brand actually is, then make every platform agree with it — and say
plainly where an asset is missing, wrong-sized, or derived from the wrong master.

The failure this exists for: the website has the current logo, the Android icon is two versions old,
the macOS icon was exported from a screenshot, the store listing uses a third blue, and nothing is
broken enough to notice until they sit side by side.

## Step 1 — Establish The Brand (interactive by default)

Ask before assuming, and ask for artifacts rather than descriptions:

1. **A vector master.** SVG, AI, PDF, or a Figma frame. Everything below derives from it. **A raster
   logo caps the quality ceiling permanently** — at 16px an upscaled PNG is mush and there is no
   recovering it. If none exists, say so as a finding and offer to trace or rebuild one.
2. **The palette**, as values: brand color, accent, background, and the light/dark variants.
3. **Fonts**, and — the part that gets skipped — **whether they are licensed for embedding** in an
   app, which is a different grant from web use.
4. **A monochrome / single-color variant**, which several platforms require and nobody has ready.
5. **Existing brand guidelines**, clear-space and minimum-size rules if they exist.
6. **A media/reference folder** if the user has one, for inspiration or actual assets to use.

If the user has none of this, derive a candidate from the most load-bearing existing asset, show what
you derived, and get one confirmation before generating anything from it.

## Step 2 — Inventory What Exists

Find every asset the product ships, per platform, with its real dimensions:

```bash
rg --files -g '{*.ico,*.icns,*.iconset,*.png,*.svg,*.webp,*.jpg}' \
   -g '{**/AppIcon.appiconset/**,**/mipmap-*/**,**/res/drawable*/**,**/Assets.xcassets/**}' \
   -g '{**/public/**,**/static/**,**/build/icons/**,**/snap/gui/**}' -g '!node_modules' -g '!.git'

identify -format '%f %wx%h %[colorspace]\n' path/to/*.png     # ImageMagick
file *.ico *.icns
```

Then compare them against each other, not just against the spec: **same artwork? same palette? same
version?** Extract the dominant colors from each rendered icon and diff them — three blues across
app, site and store listing is the classic finding and it is invisible unless you measure.

## Step 3 — Platform Requirements

Where the spec is fixed, hold to it exactly.

### Windows

`.ico` must contain, at minimum, **16, 24, 32, 48, 256**. Add **20 and 40** — Windows resolves by
exact match, then takes the *next size up and scales down*, **never up**, so 125% and 250% displays
fall back to 24 and 48 and look soft. That is the "fuzzy only on this laptop" bug. Alt+Tab wants a
**40**. Only the **256** entry should be PNG-compressed; compressing the rest saves nothing and
uncompressed 256 bloats the file.

MSIX/Store, if packaging: `Square44x44Logo` and `Square150x150Logo` at minimum 100/200/400% scale,
plus `AppList.targetsize-*` at 16/20/24/30/32/36/40/48/60/64/72/80/96/256 — **and the
`_altform-unplated` variants**, without which every user gets a grey plate behind the icon in the
taskbar with no build warning. Use `altform-lightunplated` for light theme; **never combine
`theme-light` with `altform-unplated`**. The Store still requires `MedTile` scale-100 and `StoreLogo`
even though Windows 11 ignores tiles.

### macOS

`.iconset` filenames *are* the API, and they are point-based:

```
icon_16x16.png=16   icon_16x16@2x.png=32    icon_32x32.png=32    icon_32x32@2x.png=64
icon_128x128.png=128  icon_128x128@2x.png=256  icon_256x256.png=256
icon_256x256@2x.png=512  icon_512x512.png=512  icon_512x512@2x.png=1024
iconutil -c icns -o MyIcon.icns MyIcon.iconset
```

**There is no `icon_1024x1024.png`** — the 1024 slot is `icon_512x512@2x.png`, and writing the former
gets it silently ignored. Getting the point-vs-pixel mapping backwards builds fine and looks blurry.

Current macOS icons are **layered, 1024×1024, square and unmasked**. The system applies the rounded
rectangle. **Pre-applying the mask, a drop shadow, or blur breaks the system's specular highlights and
produces jagged edges** — ship flat, square, shadow-free layers. Color spaces: sRGB, Gray Gamma 2.2, or
Display P3.

### Linux

Install into the hicolor theme at the standard sizes (16, 22, 24, 32, 36, 48, 64, 72, 96, 128, 192,
256, 512, plus `@2` variants), because **icon lookup short-circuits per theme** — one 16px icon in your
theme beats a 512px icon in a parent theme, so a partial ladder is worse than none.

`.desktop` `Icon=` takes a **name without extension** when it is not an absolute path. Flatpak wants
`/app/share/icons/hicolor/$size/apps/$APPID.png` with a reverse-DNS app id (dashes only in the last
component). AppImage requires `AppRun`, `.DirIcon`, and **exactly one** `.desktop` file in the AppDir
root. Snap takes `snap/gui/icon.svg`, 40×40 to 512×512, 256×256 recommended, under 256 KB.

### Mobile and stores — discover, do not assume

These specs move between OS releases, and a stale dimension table is worse than none. **Read the
requirement from the toolchain rather than from memory**, then validate:

- iOS: the asset catalog / Icon Composer set in the project is the authority; build and let Xcode
  report what is missing. Note the **alpha-channel prohibition** for App Store icons, and the modern
  dark/tinted variants.
- Android: adaptive icons have a foreground/background layer with a **safe zone** — a logo drawn to the
  full canvas gets cropped by the circular mask. Generate through Android Studio's Image Asset tool or
  `flutter_launcher_icons`/`tauri icon` and inspect the result at mask boundaries. Provide the
  **monochrome** layer for themed icons or the icon vanishes on some launchers.
- Stores: pull the current requirements from the console/API and diff your assets against them, rather
  than trusting a table.

Where you cannot confirm a dimension, **say so and check it** — never emit an icon set built on a
guessed spec.

### Web

Favicon set, `apple-touch-icon`, and a web app manifest with a **`maskable`** icon (its safe zone is
smaller than the canvas — a full-bleed logo gets cropped on Android). `theme-color`, including its
`prefers-color-scheme` form. Open Graph and Twitter card images. An SVG favicon needs explicit
dark-mode handling; it does not adapt on its own.

## Step 4 — Generate From The Master, Never From A Sibling

Every derived asset comes from the vector master at its target size. **Never** upscale, and never
regenerate one platform's icon from another platform's PNG — that compounds resampling and drifts the
palette.

Downscale with awareness: a logo with fine detail turns to mush at 16px. The correct answer is a
**simplified mark** for small sizes, not a sharper resampler. Check the small sizes visually, at
100%, on both light and dark backgrounds.

Watch the color pipeline: sRGB vs Display P3 shifts, hex values taken from a print palette, and gamma
shifts introduced by naive downscaling.

## Step 5 — Verify By Looking

Render every generated icon at its real target size and **actually view it** — in a grid, light and
dark, next to the current website logo and store listing. Cohesion is only visible in comparison.

Check specifically: mask cropping at circular and squircle boundaries, legibility at the smallest
size, the monochrome variant against both backgrounds, and whether the brand color used as a UI accent
still clears contrast requirements (hand that check to `/auto-ui-ux`).

## Step 6 — Report

`.audit/brand-report.md`: per platform, per asset — present/missing, actual vs required dimensions,
which master it derives from, and whether it matches the current brand. Flag anything derived from a
raster source or from another platform's export.

Include what you could not verify and why. Then list the regeneration commands, and **ask before
overwriting any existing asset** — a hand-tuned small icon is often better than a generated one, and
overwriting it silently is a real loss.

## Rationalization Table

| Excuse | Reality |
|---|---|
| "I'll generate the set from the PNG" | Raster masters cap quality permanently. Get or rebuild the vector |
| "The .ico has 16/32/48/256" | Add 20 and 40, or 125%/250% displays downscale and look soft |
| "I'll name the big one icon_1024x1024.png" | No such slot. It is `icon_512x512@2x.png`, and yours is ignored |
| "I rounded the corners and added a shadow" | The system does that. Pre-applying it breaks highlights |
| "The Android icon looks right in the preview" | Check it under the circular mask. Safe zone is smaller than canvas |
| "Windows 11 ignores tiles, skip them" | The Store still rejects a package without MedTile and StoreLogo |
| "One PNG in hicolor is enough" | Lookup short-circuits per theme. A partial ladder beats you |
| "I'll regenerate iOS from the Android asset" | Compounded resampling and palette drift. Always from the master |
| "The brand blue is #2196F3 everywhere" | Measure it. Three near-identical blues is the usual finding |
| "The font is fine, we use it on the site" | Web use and app embedding are different grants |
| "I don't know the current store spec, I'll use the usual one" | Read it from the toolchain or the console. Never guess a dimension |

## Red Flags — Stop

- Generating any icon set from a raster source without flagging it
- Emitting a dimension you did not verify against the toolchain or the current spec
- Regenerating one platform's assets from another platform's export
- Pre-applying platform masking, rounding, or shadows
- Shipping an adaptive icon without checking it under the mask
- Overwriting an existing hand-tuned asset without asking
- Declaring brand consistency without rendering the platforms side by side
- Ignoring the monochrome/themed variant because the color one looks fine
