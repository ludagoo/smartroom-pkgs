# smartroom-pkgs

Package builds for SmartRoom Firmware, the Linux image that runs SmartRoom
classroom boxes on Apple Silicon Macs. For now it builds one thing: the
aarch64 `linux-aurora` kernel packages (`linux-aurora` and
`linux-aurora-headers`) from
[ludagoo/linux-aurora](https://github.com/ludagoo/linux-aurora) `smartroom/main`.

Phase 0 builds upstream Aurora unchanged. Nothing is signed or published yet.

## What is here

| Path | What |
| --- | --- |
| `pkgbuilds/linux-aurora/` | A verbatim copy of [omacom/omarchy-pkgs](https://github.com/omacom/omarchy-pkgs) `pkgbuilds/linux-aurora` (PKGBUILD, the 240 KB `config`, `configure.sh`, helpers) at the lock below. MIT, see `LICENSE.omarchy-pkgs`. |
| `omarchy-pkgs.lock` | The omarchy-pkgs commit our recipe copy and build tooling come from: `9259d5c7f7e8…` (2026-11-07). |
| PKGBUILD `_commit` | The kernel pin in that recipe: `b798f2b0c767…`, Omarchy's `7.2.11.aurora15-1` (edge). `smartroom/main` in the fork is based on it. |
| `.github/workflows/build-linux-aurora.yml` | The build, run by hand. |

## How a build works

Run **Actions → Build linux-aurora (aarch64) → Run workflow** (or
`gh workflow run build-linux-aurora.yml -R ludagoo/smartroom-pkgs -f kernel_ref=smartroom/main`).
`kernel_ref` is any branch, tag or commit of ludagoo/linux-aurora.

1. Runs on `ubuntu-24.04-arm`, GitHub's free arm64 runner for public repos (4 vCPU, 16 GB). Expect about 45–55 minutes.
2. Checks out this repo, omacom/omarchy-pkgs at `omarchy-pkgs.lock`, and the kernel at `kernel_ref` (depth 1).
3. Replaces omarchy-pkgs' `pkgbuilds/linux-aurora` with our copy and prints any difference in the run summary (none today).
4. Runs the kernel tree's `tools/aurora-ci/prepare-recipe.py`: it sets `_commit` to the checked-out commit, makes `pkgrel` `1.<run number>` (so a build never has the version of a published Omarchy package, e.g. `7.2.11.aurora15-1.3`, `uname` `7.2.11-15-1.3-ARCH`), and points the source at the checkout instead of cloning GitHub.
5. Runs omarchy-pkgs' own `bin/build --mirror edge --arch aarch64 --package linux-aurora`, which builds inside its Arch Linux ARM builder container. This is exactly what omacom's kernel CI does, so our packages differ from theirs only by the commit and pkgrel.
6. Uploads `linux-aurora-*.pkg.tar.zst`, `SHA256SUMS` and `recipe.json` as the workflow artifact `linux-aurora-aarch64-run<N>` (kept 30 days).

Builds are not bit-for-bit reproducible: `--mirror edge` installs current
Arch Linux ARM and Omarchy-edge build dependencies.

**Updating the recipe:** copy `pkgbuilds/linux-aurora` from a newer
omarchy-pkgs commit, put that commit in `omarchy-pkgs.lock`, and rebase
`smartroom/main` onto the new `_commit` (see CARRY.md in the fork). Keep the
copy verbatim unless a change is deliberate; deliberate changes show in every
run summary's diff.

## Trust model

A package from our repo is root on every room box: pacman takes a package from
the first repo that lists it, and `[smartroom]` will sit above `[omarchy]`.
So signing is split in two, and no signing exists yet.

- **Testing key.** May live in CI (a `publish` environment). Only test boxes trust it. Builds signed with it are for the test box.
- **Rooms key.** On a hardware token that Lucas holds (touch to sign); the offline primary key is in Lucas's 1Password. Never in CI, never on an agent's machine. Lucas promotes a build to rooms himself, after it has run on the test box (48 h) and its provenance checks out.
- Rooms trust only the rooms key, installed by a `smartroom-keyring` package that also handles subkey renewal and revocation.
- CI will publish only an allowlist of package names and never overwrite a published file.

The keys do not exist yet. Lucas creates them; an agent must not.

Agents (Claude, Cursor, Grok) run `gh` as ludagoo, so a GitHub environment
reviewer alone does not stop them. That is why the rooms key is on hardware.

## Not here yet

Signing, the `[smartroom]` pacman repository (a GitHub release tagged
`aarch64`), `smartroom-keyring`, build attestations, a scheduled rebuild.

## License

Our files: MIT (`LICENSE`). The recipe copy: MIT, Copyright (c) David
Heinemeier Hansson (`pkgbuilds/linux-aurora/LICENSE.omarchy-pkgs`). The kernel
it builds: GPL-2.0, source in ludagoo/linux-aurora.
