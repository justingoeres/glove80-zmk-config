# Updating the firmware base — procedure & rationale

## TL;DR

Updating "the firmware" on this Glove80 is a **one-line change** to `.github/workflows/build.yml` — bump the `ref:` of the `moergo-sc/zmk` checkout to a newer release tag. It is **not** a `git merge` from any other repo.

This works because: this is a *config* repo — the firmware itself lives in a separate repo (`moergo-sc/zmk`) and is fetched at build time. There's nothing to merge, nothing to rebase, no upstream conflicts. Bump the ref, push, flash the resulting `.uf2`, re-pair if needed, done.

## "Are we forked from the glove80 repo?" — No, but close

It feels like a fork because the repo started from MoErgo's structure, but on GitHub it's actually a **template instance**, not a git fork:

```
gh repo view --json isFork,parent,templateRepository
{
  "isFork": false,
  "parent": null,
  "templateRepository": { "owner": { "login": "moergo-sc" }, "name": "glove80-zmk-config" }
}
```

The distinction matters:

|                             | Git fork                          | Template instance (this repo)                                         |
| --------------------------- | --------------------------------- | --------------------------------------------------------------------- |
| Created via                 | "Fork" button                     | "Use this template" button                                            |
| Shared git history          | Yes — common ancestor commit      | No — independent histories from day 1                                 |
| GitHub "Sync fork" UI       | Yes                               | No                                                                    |
| `git pull <upstream>` works | Cleanly (mostly)                  | Only with `--allow-unrelated-histories`, and conflicts will be brutal |
| Tracks upstream by default  | Yes (`origin`/`upstream` remotes) | No — only `origin` (your own repo) is configured                      |

So: **no automatic way to "sync upstream"** at the GitHub level, and trying to merge upstream's git tree directly would create unrelated-history conflicts on every file that has diverged (most importantly `config/glove80.keymap`, which is a 33 KB heavily-modified file vs. upstream's 9 KB vanilla template).

### Why "template instance" instead of "fork"?

Probably visibility. **GitHub forks inherit visibility from the parent** — you can't fork a public repo and make the fork private. To get a private starting point, you have to use "Use this template" (or clone and push to a new private repo manually). That trade-off — privacy in exchange for losing the fork's automatic sync UI — was the likely original motivation for this repo's structure. The repo has since gone public, so the privacy reason no longer applies, but the consequences persist: upstream sync is manual no matter what.

## "Can we just merge the glove80 firmware into ours and expect it to work?" — Two questions in one

The phrase "merge the glove80 firmware" conflates two different things:

### 1. The actual firmware code (ZMK) → not a merge, just a ref bump

The Glove80 firmware source lives in **`moergo-sc/zmk`** — a *separate* repo from the config template. This repo's `build.yml` checks it out at build time:

```yaml
- uses: actions/checkout@v4
  with:
    repository: moergo-sc/zmk
    ref: v24.02            # ← THIS is what controls the firmware version
    path: src
```

Updating the firmware = changing `ref: v24.02` to a newer tag (e.g., `ref: v25.11`). That's the entire change. Available tags: see [`moergo-sc/zmk/releases`](https://github.com/moergo-sc/zmk/releases).

For comparison, upstream's `moergo-sc/glove80-zmk-config` uses `ref: main` so it always picks up whatever's latest on moergo's `main` branch. That's lower friction but trades for stability — pinning a tag means the build is reproducible and won't suddenly change under you.

### 2. The config-repo template files → cherry-pick, don't merge

The template repo (`moergo-sc/glove80-zmk-config`) has its own files that *could* drift from this repo over time: `build.yml`, `default.nix`, `info.json`, `glove80.keymap` (the vanilla starting point), Dockerfile, etc. Comparing as of 2026-05:

| File                          | Upstream                                                      | This repo                                                                                     | Status                                  |
| ----------------------------- | ------------------------------------------------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------- |
| `.github/workflows/build.yml` | `ref: main`, builds on `pull_request` too, `cachix/*@v25/v14` | `ref: v24.02`, push-only, `cachix/*@v27/v15`, branch-named artifacts, ships `glove80.svg` too | **Diverged in both directions**         |
| `config/default.nix`          | Accepts `firmware ?` parameter                                | Hardcodes `import ../src {}`                                                                  | Functionally equivalent for CI          |
| `config/glove80.conf`         | Empty                                                         | Empty                                                                                         | Same                                    |
| `config/glove80.keymap`       | Vanilla template (~9 KB)                                      | Heavily customized (~33 KB)                                                                   | **Never sync — this repo owns it**      |
| `config/info.json`            | Layout Editor metadata                                        | Layout Editor metadata                                                                        | Likely safe to sync if upstream changes |
| `config/keymap.json`          | Layout Editor metadata                                        | Layout Editor metadata                                                                        | Tracks `glove80.keymap` — don't sync    |

Recent upstream commits to the template (last few years):

- 2025-05-26 — Windows `build.bat` script, LF line endings
- 2024-04-04 — Dockerfile improvements, syntax highlighting for `.keymap`
- 2023-10-07 — Add Dockerfile for local builds

The template moves slowly. Most updates are infrastructure (Docker, build scripts) that this repo doesn't use anyway because builds run in CI on Nix.

## Recommended procedure to update firmware

This is a feature-branch task. Follow it like any other change in this repo.

```bash
git checkout main && git pull
git checkout -b feature/firmware-bump-v25-11   # or whatever target version
```

Edit `.github/workflows/build.yml`:

```diff
       - uses: actions/checkout@v4
         with:
           repository: moergo-sc/zmk
-          ref: v24.02
+          ref: v25.11
           path: src
```

Then:

```bash
git add .github/workflows/build.yml
git commit -m "🔧 Bump moergo-sc/zmk pinned ref to v25.11"
git push -u origin feature/firmware-bump-v25-11
gh pr create --title "🔧 Bump firmware base to moergo-sc/zmk v25.11"
```

CI runs the Nix build. If green, download the artifact (`gh run download <id> --dir ./builds/`) and flash both halves.

### Post-flash checklist

After flashing a new firmware base, in order:

1. **Confirm both halves boot** and the keymap looks right.
2. **`BT_CLR_ALL` and re-pair every host.** MoErgo's docs explicitly recommend this on major version jumps because the BLE bond format can change between releases.
3. **Configuration factory reset on the right half**, per MoErgo's troubleshooting FAQ: charge the right half via USB, power-cycle both halves, toggle RGB on with `magic+t`, confirm RGB works on both halves, toggle RGB off, wait ≥1 minute before powering off — needed for the new config to persist.
4. **Sanity-check the keymap layers.** Each layer should still behave as in the keymap file. ZMK occasionally renames keycodes or changes behavior semantics between major versions — read the release notes between the old and new tag.
5. **Re-evaluate the workarounds in `glove80.keymap`** — see `docs/split-rgb-status.md`. The commented-out RGB-on-layer-switch macros at `config/glove80.keymap:96–113` may be safe to re-enable on `v25.01+`.

## Picking a target version

Look at [`moergo-sc/zmk/releases`](https://github.com/moergo-sc/zmk/releases) and pick the latest stable (non-beta, non-pre-release). At time of writing (2026-05) the recommendation is `v25.11`.

Avoid `ref: main` for this repo. Upstream uses it because they're the maintainers and they want continuous testing; for a personal config you want reproducibility. Pinning a tag means a build kicked off today and a build kicked off in six months will produce identical firmware unless you choose to bump.

## What this repo's `build.yml` already does *better* than upstream

Worth noting before you ever consider syncing `build.yml` from upstream — this repo is **ahead** in three places:

- `cachix/install-nix-action@v27` (upstream still on v25)
- `cachix/cachix-action@v15` (upstream still on v14)
- Branch-named artifact filenames (`glove80-<branch>-build-<n>.<m>.uf2`) — upstream uploads a generic `glove80.uf2` that is a pain to tell apart across runs.
- Uploads `glove80.svg` alongside the firmware — useful for at-a-glance review.

Don't accidentally lose any of that by bulk-overwriting `build.yml` from upstream. If you ever do sync, do it as a per-line cherry-pick.

## Sources

- [`moergo-sc/zmk` releases](https://github.com/moergo-sc/zmk/releases) — pick the target tag from here
- [`moergo-sc/glove80-zmk-config`](https://github.com/moergo-sc/glove80-zmk-config) — upstream template
- [Glove80 Troubleshooting FAQs (MoErgo)](https://docs.moergo.com/glove80-troubleshooting-faqs/) — re-pair / RGB factory reset procedure
- [`docs/split-rgb-status.md`](./split-rgb-status.md) — companion note: which v25.x release first contains the split-RGB fix
