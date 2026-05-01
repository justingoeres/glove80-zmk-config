# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## ⚠ This repo is public

Everything in this directory ships to a public GitHub repo. **Do not write personal facts, real names, email addresses, host names, network details, work-context info, credentials, or anything else not strictly about Glove80/ZMK into any tracked file** — including this `CLAUDE.md`, the `README.md`, keymap comments, or commit messages. Treat the maintainer in third-person-neutral ("the maintainer", "the user") rather than by name. If a discussion needs context that's personal, hold it in conversation and don't persist it.

## What this repo is

A personal ZMK firmware configuration for the MoErgo Glove80 wireless split keyboard. There is no application source code — the "build" is a firmware compilation that runs in CI against an upstream ZMK checkout. Editing here means editing the keymap and rebuilding firmware, not running services or tests locally.

## Your job in this repo

Help the maintainer **maintain, debug, and modify** their fancypants keyboard setup. Concretely that means: tweaking keymaps and layers, adjusting hold-tap / tap-dance / home-row-mod timing and behavior, diagnosing why a key feels wrong (accidental shifts, missed taps, mod sticking), wiring new behaviors into the right layer at the right key position, and keeping the generated render in sync. Most "bugs" here are timing/ergonomic, not compile-time — when something feels off, suspect `tapping-term-ms`, `quick-tap-ms`, `flavor`, `hold-trigger-key-positions`, or `hold-trigger-on-release` before suspecting the binding itself.

## Build & render

There is no local build target. Two GitHub Actions workflows handle everything:

- `.github/workflows/build.yml` — on every push and via `workflow_dispatch`. Checks out this repo plus `moergo-sc/zmk` at the pinned ref (`v24.02`) into `src/`, then runs `nix-build config -o combined` to produce `glove80.uf2`. The artifact is uploaded as `glove80-<branch>-build-<run>.<attempt>.uf2`. Flash that file to the keyboard.
- `.github/workflows/draw-keymaps.yml` — on PRs to `main` that touch `config/*.keymap`, `config/*.dtsi`, or `keymap_drawer.config.yaml`. Calls `caksoylar/keymap-drawer`'s reusable workflow, which commits an updated `glove80.svg` and `glove80.yaml` back to the PR branch. Expect those two files to show up in diffs you didn't write.

If you need to reproduce the firmware build locally, you need Nix and the `moergo-sc/zmk` repo cloned alongside as `src/`; then `nix-build config -o combined`. This is rarely worth doing — push and let CI build.

## Architecture

### `config/glove80.keymap` is the only file most edits touch

It's a devicetree source file processed by the C preprocessor before ZMK compiles it. Important conventions:

- **Layer indexes are `#define`s at the top.** Current layers: `Base` (0), `Lower` (1), `Cursor` (2), `Spaces` (3), `Gaming` (4), `LabVIEW` (5), `Magic` (6). Reordering these `#define`s silently rebinds every `&mo`/`&to`/`&lt` reference — change names, not the numeric order, unless you intend to remap.
- **Custom behaviors live in `behaviors { ... }` blocks** (tap-dances, hold-taps, mod-morphs). The file has multiple `/ { ... };` root blocks that get merged by the devicetree compiler — that's normal, not a bug to clean up.
- **Layer bindings** are inside `keymap { ... }` and laid out as a 2D grid of `&kp`/`&mo`/`&macro_*`/etc., one row per physical row across both halves. Whitespace and column alignment matter only for human readability; the drawer parses positions, not formatting.
- `config/glove80.conf` is the Kconfig fragment (currently empty). Add ZMK feature flags here, not in the keymap.
- `config/default.nix` wires the build: it imports `../src` (the upstream ZMK checkout), overrides `board` for `glove80_lh` and `glove80_rh`, and combines them via `firmware.combine_uf2`. Don't edit this unless changing how the firmware is assembled.
- `config/info.json` and `config/keymap.json` are emitted by the **Glove80 Layout Editor** webapp. They are not consumed by the Nix build — `glove80.keymap` is the source of truth. They exist so the Layout Editor can round-trip; keep them in sync if you use the editor.

### Layers

Defined in order at the top of `glove80.keymap` as `LAYER_*` `#define`s. The Base layer is the canonical QWERTY surface; everything else is an overlay reached via thumb-cluster keys, tap-dances, or layer-tap behaviors. `&trans` falls through to the layer below; `&none` blocks the key.

| #   | Name      | Purpose                                                                                                                                                                                                                                                                                                                             |
| --- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0   | `Base`    | QWERTY with **Miryoku-style home-row mods** (`homey_left/right` for ALT/CTRL/GUI, `index_left/right` for SHIFT on D/K). Includes `lk_qu` (linger Q→qu), `tap_dance_caps_word` (tap = caps-word, double = CapsLock), `shift_spc_uscr` (shift+space → underscore — left-shift only), and the `magic` hold-tap on the bottom-left key. |
| 1   | `Lower`   | Numpad on right hand, F1–F12, media keys (vol/brightness/transport), and explicit `&to N` layer-jump keys for switching to other layers from a single chord. Reached via the `lower` tap-dance on the inner thumb keys (tap/hold = `&mo`, double-tap = `&to`).                                                                      |
| 2   | `Cursor`  | Vim-ish navigation cluster: arrows on right hand, line-jump on the left (`LG(LEFT/RIGHT)` = beginning/end of line on macOS), edit shortcuts (`LG(Z/X/C/V)`, Backspace/Delete). Set up for **macOS** — uses `LG(...)` (Cmd) for clipboard and line nav. Reached via `td_cursor` and `thumb_caps`.                                    |
| 3   | `Spaces`  | OS workspace / virtual-desktop switching using `RC(LEFT/RIGHT/UP/DOWN)`. Reached via the `&mo LAYER_Spaces` thumb key and from Cursor.                                                                                                                                                                                              |
| 4   | `Gaming`  | Plain QWERTY **without** home-row mods so games don't fire mod-tap on held WASD. Mods are on dedicated thumb-cluster keys instead. Enter via `&to LAYER_Gaming` (typically from Lower); leave via `&to LAYER_Base` on the inner thumb.                                                                                              |
| 5   | `LabVIEW` | LabVIEW-specific shortcuts using `LG(...)` (Close/Run/Edit/cut/copy/paste). Reached via `&to LAYER_LabVIEW` from Lower.                                                                                                                                                                                                             |
| 6   | `Magic`   | System layer: Bluetooth profile selection (`bt_0..bt_3`, `BT_CLR`, `BT_CLR_ALL`), RGB underglow controls, `&out OUT_USB`, `&bootloader`, `&sys_reset`. Reached by **holding** the bottom-left magic key (`&magic LAYER_Magic LAYER_Base`) — the tap action runs the RGB-status macro.                                               |

A `Factory` layer (7) and `Blank`/`Trans` template layers exist commented out at the bottom of the keymap — handy starting points when adding a new layer.

### Custom behaviors worth knowing

All defined in the big `behaviors { ... }` block around lines 165–400:

- `homey_left` / `homey_right` — `balanced` hold-tap, `hold-trigger-on-release`, opposite-hand-only triggering. Held = mod, tapped = letter. Tuned via `HOMEY_TAPPING_TERM` (280) and `TYPING_STREAK_TERM` (160, via `global-quick-tap`).
- `index_left` / `index_right` — `tap-preferred` flavor for the index-finger SHIFT keys; **no** `hold-trigger-on-release` and **no** `global-quick-tap` so SHIFT activates fast (key repeat enabled via `KEY_REPEATING_TERM`). The most recent commit `cb42dbe` swapped which finger gets `index_` vs `homey_` to fix accidental shift non-activations — be careful when touching this.
- `magic` — hold = `&mo LAYER_Magic`, tap = RGB status macro. Lives on the bottom-outer corners.
- `lower`, `td_cursor` — tap-dances where first tap/hold = `&mo`, second tap = `&to` (sticky). Lets one key both peek and lock a layer.
- `thumb`, `thumb_caps` — Miryoku thumb-cluster hold-taps (`&mo` on hold, `&kp` / `&caps_macro` on tap).
- `lk_qu` — "linger" tap behavior: Q tapped quickly enough types `qu`.
- `shift_spc_uscr` — shift+space → underscore, gated to left-shift only (per `fb725ac`).

### Generated artifacts

`glove80.svg` and `glove80.yaml` at the repo root are committed by the `draw-keymaps` workflow. Don't hand-edit them — they'll be overwritten on the next PR. `keymap_drawer.config.yaml` controls how they're rendered (key labels, colors, glyph substitutions).

### Reference

`reference/Sunaku's Layout v22 (Engrammer).keymap` is a third-party layout kept for inspiration/comparison. Not part of any build.

## Glove80 hardware operations (pairing, output, recovery)

These are the operations that come up when something goes sideways with the keyboard itself. All of them live on the **Magic layer** (LAYER_Magic, 6), entered by **holding** the bottom-outer corner key (`&magic LAYER_Magic LAYER_Base`). On this keymap the relevant bindings are roughly:

| Operation                         | Behavior                                 | Position on Magic layer                                                        |
| --------------------------------- | ---------------------------------------- | ------------------------------------------------------------------------------ |
| Select Bluetooth profile 0–3      | `bt_0` / `bt_1` / `bt_2` / `bt_3` macros | thumb cluster + row-5 inner thumbs (each macro also runs `&out OUT_BLE` first) |
| Switch output to USB              | `&out OUT_USB`                           | center thumb (bottom row)                                                      |
| Clear *current* BT pairing        | `&bt BT_CLR`                             | top-left corner (F1 position)                                                  |
| Clear *all* BT pairings           | `&bt BT_CLR_ALL`                         | top-right corner (F10 position)                                                |
| Enter bootloader (firmware flash) | `&bootloader`                            | row-4 outer corners (both halves)                                              |
| System reset                      | `&sys_reset`                             | row-5 outer corners (both halves)                                              |
| RGB underglow controls            | `&rgb_ug RGB_*`                          | left letter cluster (HUI/SAI/SPI/BRI/TOG; HUD/SAD/SPD/BRD/EFF)                 |

### Pairing / re-pairing playbook

A Glove80 holds **4 independent BLE profiles** (slots 0–3). Each slot pairs with one host. To switch between hosts, just select the slot (Magic + the slot's key). To switch back to the wired connection, Magic + center thumb (`OUT_USB`).

When a host *was* paired and now refuses to connect, **the pairing has to be cleared on both ends** — clearing only one side leaves a stale association that prevents re-pairing:

1. On Glove80: hold Magic, select the affected slot, then Magic + top-left to `BT_CLR`.
2. On the host: remove the Glove80 from its Bluetooth settings (Windows: Settings → Bluetooth → Remove Device; macOS: System Settings → Bluetooth → ⓘ → Forget; Linux: `bluetoothctl remove <mac>`).
3. Re-initiate pairing from the host. The Glove80 advertises automatically when the slot is empty.

To wipe everything and start clean: Magic + top-right (`BT_CLR_ALL`), then forget the keyboard on every host that had it.

### Flashing firmware

Each half flashes **independently** but with the **same `.uf2` file** — `combine_uf2` packages both halves' images into one file and the bootloader picks the right one. Repeat the procedure once per half.

**Get the file.** After pushing, open the repo on GitHub → Actions → the most recent `Build` run → scroll to **Artifacts** → download `glove80-<branch>-build-<n>.<m>.uf2`. Unzip; the result is a single `.uf2` file. (Filename doesn't matter to the bootloader — no need to rename to `CURRENT.UF2`.)

**Enter bootloader on each half.** Two methods:

1. **From a working keymap (preferred).** Hold the Magic key, tap `&bootloader`. On the factory default layout that's **Magic + Esc** (left half) / **Magic + `'`** (right half). On this keymap `&bootloader` is on the **row-4 outer corner** of each half, so it's Magic + that corner key. Whichever half's Magic key you held is the half that enters bootloader.
2. **Power-up method (fallback when ZMK is wedged).** Power the half off, plug it in via USB, then power on while holding **C6R6 + C3R3** (Magic + E on left, Magic + PgDn on right, on factory default). Use this if the keyboard isn't responding well enough for method 1.

**Confirm bootloader mode** by the LED next to the power switch:

- **Slow pulsing red** = in bootloader, USB connected, ready to receive a UF2. ✅
- **Fast flashing red** = in bootloader, but no USB host detected (check cable / port).
- **Off / solid** = not in bootloader; try again.

A USB mass-storage drive named something like `GLV80LHBOOT` / `GLV80RHBOOT` will appear on the host.

**Flash.** Copy the `.uf2` onto that mass-storage drive (Finder drag, `cp`, whatever). On a successful write the drive disappears, the half reboots into the new firmware, and the LED returns to its normal state. If the drive doesn't disappear, the flash didn't take — check the cable (some charge-only USB-C cables won't enumerate data) and retry.

**Repeat for the other half.** Halves run their own firmware images and their own Bluetooth state — flashing only one leaves them mismatched, which usually shows up as one side's keys not registering or the halves failing to pair to each other.

**After a major version change**, MoErgo recommends a factory reset (clear all BT pairings via `BT_CLR_ALL`, then re-pair) since the BLE bond format can change between releases.

Reference: [MoErgo: Customizing key layout & loading firmware](https://docs.moergo.com/glove80-user-guide/customizing-key-layout/), [Building firmware (Layout Editor guide)](https://docs.moergo.com/layout-editor-guide/building-firmware/).

### Adding / reassigning layers

Two paths:

- **Glove80 Layout Editor** (web app, simpler) — round-trips through `config/keymap.json` and `config/info.json`. Available layer behaviors there: `&mo` (momentary), `&lt` (layer-tap), `&to` (jump-and-stick), `&sl` (sticky-layer for next keypress), plus MoErgo's pseudo-behaviors `&lower` and `&layer` for double-tap-to-stick. Layout Editor does **not** expose `&tog`; if you want toggle, edit the keymap directly.
- **Editing `glove80.keymap` directly** (the path used in this repo) — preferred for anything beyond the basics. To add a new layer:
  1. Add a `#define LAYER_<Name> N` at the top, keeping numeric order intact.
  2. Add a `layer_<Name> { bindings = < ... >; };` node inside `keymap { }` — copy the commented `layer_Trans` template at the bottom as a starting point.
  3. Bind a key somewhere to enter it (`&mo LAYER_<Name>`, `&to LAYER_<Name>`, or a tap-dance like the existing `lower` / `td_cursor`).
  4. Push — CI rebuilds firmware and re-renders `glove80.svg`.

  To **remove** a layer, also strip every `&mo`/`&to`/`&tog`/`&lt` reference to its number; `keymap-drawer` will fail loudly if any references survive.

  Reference docs:
  - [Operating Glove80 wirelessly](https://docs.moergo.com/glove80-user-guide/operating-glove80-wirelessly/) — Bluetooth profiles, output switching, re-pairing
  - [Layout Editor guide](https://docs.moergo.com/layout-editor-guide/layout-editing/) — Layout Editor's behavior catalog
  - [Glove80 troubleshooting FAQ](https://docs.moergo.com/glove80-troubleshooting-faqs/) — recovery, stuck halves, etc.
  - [ZMK behaviors reference](https://zmk.dev/docs/behaviors) — canonical docs for `hold-tap`, `tap-dance`, `mod-morph`, etc.

## Branching & naming

Workflow is **branch → push → PR → merge to `main`**. Every push triggers `build`; PRs touching keymap files trigger `draw-keymaps` which auto-commits the rendered SVG/YAML back to the branch (so a PR usually has a `keymap-drawer render` commit on top of the human ones).

Branch prefixes observed in history (use these — don't invent new ones):

- `feature/<kebab-case-topic>` — by far the most common; used for new layers, behavior tweaks, ergonomic experiments, build/CI changes. Examples: `feature/cursor-layer`, `feature/labview-layer`, `feature/home-row-mods-adjust`, `feature/shift-space-underscore`, `feature/keymap-drawer`.
- `bugfix/<kebab-case-topic>` — for fixing broken behavior. Example: `bugfix/lower-not-working`.
- `develop` — exists on the remote but is dormant; merges go directly from `feature/*` to `main`. Don't branch from `develop`.

A few naming-convention notes that come from reading the keymap, not the branches:

- **Behavior identifiers in `behaviors { }`** are `snake_case` and prefixed by what they do: `homey_*`, `index_*`, `thumb*`, `td_*` (tap-dance), `lk_*` (linger), `magic`, `shift_spc_uscr`. The `miryoku_*` labels in `label = "..."` are descriptive only — match them when adding new behaviors in the same family.
- **Layer node names** in `keymap { }` are `layer_<Name>` with `PascalCase` after the underscore, mirroring the `LAYER_<Name>` `#define`. Keep both in sync when renaming.
- **Macros** are `snake_case` with a `_macro` suffix (e.g., `rgb_ug_status_macro`, `caps_macro`).

## Commit & PR style

- Commit messages use **gitmoji prefix only** — just the emoji, no `feat:` / `fix:` word label (e.g., `✨ add Cursor layer`, not `✨ feat: add Cursor layer`). Existing history uses the older conventional-commit `feat:`/`fix:` style; new commits should follow the gitmoji-only convention.
- Don't manually commit `glove80.svg` or `glove80.yaml` — let the `draw-keymaps` workflow do it. If a PR's render looks stale, touching the keymap (even a comment edit — note the `/* comment to trigger keymap-drawing */` line near the top) re-triggers it.
- The `keymap-drawer` workflow re-runs only when files in its `paths:` filter change.
