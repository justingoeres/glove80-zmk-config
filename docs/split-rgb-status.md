# Split-keyboard RGB on Glove80 — bug status (as of 2026-05)

## TL;DR

The "RGB control only affects the left half" bug **has been fixed upstream** and is included in moergo's `v25.01` release and every release after it. **This repo's CI still pins `moergo-sc/zmk@v24.02`** (`.github/workflows/build.yml`), which is from **2024-02-16** — about 7 months before the fix shipped. So on the firmware actually running on this Glove80, the bug is still present. Bumping the pinned ref to **`v25.11`** (the current latest moergo release, 2025-11-26) would pick up the fix.

This note is research only — no code, keymap, or workflow changes. See "Why we kept the workaround" at the bottom for the existing in-keymap workaround.

## The bug

When `&rgb_ug RGB_*` is invoked from inside a macro, hold-tap, mod-morph, tap-dance, or sticky-key (collectively: "nested behaviors"), the RGB command is only delivered to the **central** half (the left half on Glove80). The peripheral (right) half doesn't receive the command. Calling `&rgb_ug ...` directly from a keymap binding works fine on both halves — the bug is specifically about *nested* invocations.

Concretely in this repo, the consequence is the commented-out RGB-on-layer-switch macros at `config/glove80.keymap:96–113` (and the surrounding comment block):

```c
//    Layer switch with RGB is disabled because an
//    upstream ZMK bug means RGBs in macros only
//    light up the left (main) side of the keyboard
```

That comment is accurate against the firmware actually flashed today. It would no longer be accurate after a bump to v25.01+.

## Upstream resolution

| Item                | Reference                                                                                                                                                | Date                  |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| Original bug report | [zmkfirmware/zmk #1347 — `&rgb_ug RGB_COLOR_HSB on split keyboard is not global when used within macro`](https://github.com/zmkfirmware/zmk/issues/1347) | filed 2022-06-16      |
| Related bug         | [zmkfirmware/zmk #1494](https://github.com/zmkfirmware/zmk/issues/1494) (broader "behavior locality" bucket)                                             | —                     |
| Earlier attempt     | [zmkfirmware/zmk #1630](https://github.com/zmkfirmware/zmk/pull/1630) (covered macros only)                                                              | —                     |
| **Final fix**       | [zmkfirmware/zmk #2409 — `fix(split): Make locality work for nested behaviors`](https://github.com/zmkfirmware/zmk/pull/2409)                            | **merged 2024-09-23** |
| Issue #1347 closed  | by maintainer caksoylar referencing #2409                                                                                                                | 2024-09-23            |

PR #2409 generalizes the locality fix beyond macros: it covers macros, mod-morphs, hold-taps, tap-dances, sticky keys, and combos invoking global behaviors like RGB underglow. The implementation adds a `source` field to `zmk_behavior_binding_event` so nested behaviors propagate the originating side correctly when split-locality routes a command to "global" / both halves.

## Moergo's release pipeline

`moergo-sc/zmk` cherry-picks the fix as commit [`9e36ebd5 feat(split): Make locality work nested behavior invocations`](https://github.com/moergo-sc/zmk/commit/9e36ebd52587c0364562057466a458fdcfcdc685). Spot-checking which moergo release tags contain that commit:

| moergo tag      | Tagged     | Contains the fix?            |
| --------------- | ---------- | ---------------------------- |
| `v23.12`        | 2023-12-18 | ❌                            |
| `v24.02`        | 2024-02-16 | ❌ ← **what this repo pins**  |
| `v24.08-beta.1` | 2024-07-24 | ❌                            |
| `v25.01`        | 2025-01-21 | ✅ first release with the fix |
| `v25.05`        | 2025-05-14 | ✅                            |
| `v25.08`        | 2025-08-27 | ✅                            |
| `v25.11`        | 2025-11-26 | ✅ current latest             |

(Verified by walking `app/src/behaviors/behavior_macro.c` history at each tag via the GitHub API — the fix commit appears in v25.01 and every later tag, and is absent from v24.02.)

A note on the commit's authored date: the commit shows `2023-01-17` because it carries the original PR #2409 author date (and a co-author trailer to user `tokazio`, who first reported the regression in their Sofle config). The actual landing in moergo's tree was after upstream merge.

## What this repo currently pins vs. upstream template

- **This repo:** `.github/workflows/build.yml` pins `repository: moergo-sc/zmk` at `ref: v24.02`. The build runs against a 2-year-old ZMK that does not have the fix.
- **Upstream `moergo-sc/glove80-zmk-config` template:** now uses `ref: main` — i.e., it rides moergo's `main` branch and gets the fix automatically.

So the upstream template's stance is "ride main"; this fork froze its ref at v24.02 some time ago and the world moved on.

## Implications (no action taken)

If the pinned ref is ever bumped to a release ≥ `v25.01` (most likely target: `v25.11` or whatever's current at the time):

1. The commented-out layer-switch RGB macros at `config/glove80.keymap:96–113` could be uncommented and would work on both halves. Worth re-reading the `to_layer_4_rgb_on` / `to_layer_0_rgb_off` definitions before re-enabling — they were written against an older ZMK API and may need minor adjustments.
2. Other RGB-in-macro / RGB-in-tap-dance compositions that were avoided would become available.
3. Unrelated changes between v24.02 and v25.11 will also come along — Bluetooth bond format, behaviors API surface, kconfig defaults, board defs. After flashing, MoErgo's docs recommend `BT_CLR_ALL` + re-pair on every host, and a "Configuration factory reset" if the right half misbehaves.
4. Worth a build-and-test pass on a feature branch before merging — exactly the kind of change that benefits from the existing CI artifact + flash-and-test loop.

## Why we kept the workaround

Glove80 owners who were on early-2024 firmware learned to write RGB triggers either at the *outer* keymap level (works on both halves) or simply to leave the layer-switch RGB feedback off. The current keymap takes the second path: macros for layer-switch + RGB are defined but commented out, and the bottom-left "magic" key's tap action is `rgb_ug_status_macro` (single-binding `&rgb_ug RGB_STATUS`) rather than a multi-step macro — that single binding is *not* a nested invocation, so it works fine on both halves even on v24.02. Worth preserving that pattern: top-level `&rgb_ug` bindings work everywhere; nested `&rgb_ug` only works once we're on v25.01+.

## Sources

- ZMK issue tracker — [#1347 (closed)](https://github.com/zmkfirmware/zmk/issues/1347), [#1494](https://github.com/zmkfirmware/zmk/issues/1494), [#3017 (open, related architecture concern)](https://github.com/zmkfirmware/zmk/issues/3017)
- ZMK PR — [#2409 (merged 2024-09-23)](https://github.com/zmkfirmware/zmk/pull/2409)
- moergo-sc/zmk commit — [`9e36ebd5`](https://github.com/moergo-sc/zmk/commit/9e36ebd52587c0364562057466a458fdcfcdc685)
- moergo-sc/zmk releases — [GitHub releases page](https://github.com/moergo-sc/zmk/releases)
- moergo-sc/glove80-zmk-config — [upstream template `build.yml`](https://github.com/moergo-sc/glove80-zmk-config/blob/main/.github/workflows/build.yml) (rides `main`)
- MoErgo troubleshooting docs — [Glove80 Troubleshooting FAQs](https://docs.moergo.com/glove80-troubleshooting-faqs/)
- ZMK feature docs — [Lighting](https://zmk.dev/docs/features/lighting), [Split keyboards](https://zmk.dev/docs/features/split-keyboards), [Underglow behavior](https://zmk.dev/docs/keymaps/behaviors/underglow)
