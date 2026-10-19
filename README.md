# wowsilicon-arm64

Builds [WoWSilicon](https://github.com/WoWSilicon/WoWSilicon) around the native
arm64 Wine of [athei/wine-build](https://github.com/athei/wine-build), both
from source, with WoWSilicon's Wine patches on top, without waiting for a
release of either.

`build.yml` builds the bundle the way wine-build's own `release.yml` does:
the same runner, layout, `bin/ci` scripts, Homebrew action, caches and
`bundle-wine.sh --strict-floor --symbols`, then `package.sh`. On top of it:

- Wine is the ref in wine-build's `pins/wine-src.ref` (or `wine_ref`), plus
  `patches/wine`; FEX is the ref in `pins/fex-src.ref` (or `fex_ref`), plus
  `patches/fex`. Mesa, DXMT, mtld3d and llvm-mingw are what wine-build pins.
- Each patch is committed, since `package.sh` names commits and refuses
  uncommitted changes.
- The signed loader replaces `release.yml`'s signing step (below).

The artifact holds the bundle tarball, its SHA-256, `package.sh`'s
`notes.md` (what everything was built from, the Homebrew library versions
included) and `BUILD-INFO.txt` (this repository's patches); the symbols are
a separate artifact.

## FEX

`patches/fex/0001` makes the whole last page of an executable image section
executable, as Windows' page protection does, instead of stopping at its
`VirtualSize`. WoWSilicon's patched `DivxDecoder.dll` (the hook that loads
`libDllLdr.dll`) puts its entry point in the slack after `.text`, and FEX
faulted on it.

## The app

`app.yml` (run by `build.yml` after the bundle, or alone with `wine_run_id`
naming an earlier run) builds `WoWSilicon/WoWSilicon` at v3.2.2 through its
own `make bundle`, with `patches/app` and the icon in `app/icon.png`:

- `0001` signs the app without `--deep`, which would re-sign `wine.app` and
  drop its entitlements, and makes `tools/wine-runtime/validate.sh` check an
  arm64 bundle. Game Mode needs nothing from the app: `wine.app` declares
  `LSSupportsGameMode`.
- `0002` removes the update feed, so Sparkle never replaces it with the
  official build. Otherwise it is WoWSilicon, in place of the official app:
  the same name, bundle identifier, settings and default bottle
  (`~/WoWSilicon`), which has to be a prefix of this Wine.
- `0003` starts Wine through `Contents/MacOS/wowsilicon-launch`, which
  clears `__CFBundleIdentifier` and execs Wine as its own responsible
  process. Every child of WoWSilicon inherits WoWSilicon's
  `__CFBundleIdentifier`, and with it WoW ran about 7 % slower: 135 against
  145 fps at one spot, measured with WoWSilicon's whole environment from a
  shell, with and without that one variable.
- `0004` drops `LSApplicationCategoryType` (games), by which Game Mode took
  the launcher itself for a game. The game is still one: `wine.app`
  declares `LSSupportsGameMode`.
- `0005` lists Wine processes in process instead of running `ps` and parsing
  the whole process table: libproc's process list and each process's
  arguments, of which only the ones `WineProcessMonitor`'s classification
  could accept are passed on, in `ps`'s format (about 27 ms a poll against
  165 ms, the same processes on a system running Wine). While a process
  WoWSilicon started runs, it does not poll: that process's exit requests a
  poll, and a 60-second poll is the fallback. That cost showed as
  WoWSilicon at 5–27 % CPU beside the game.

The app leaves out what only building against Wine or testing it needs:
`include`, the import and static libraries, the SDK tools in `bin`, the man
pages and `lib/wine/tests`, about 150 MB. The bundle artifact keeps them.

x87 needs no app setting: FEX runs the game's x87 code, and the Rosetta x87
paths the app passes are ignored by this Wine.

## The loader

The loader needs `com.apple.developer.cross-architecture-support`, which macOS
honours only with a provisioning profile that grants it and a signature by a
certificate of the profile's team. That certificate is athei's, so the signed
`wine.app` is taken from a wine-build release (`loader_release`).
`patches/wine-build/0001` adds
`LOADER_APP` to `bundle-wine.sh`: it compares the loader this build made with
the app's, segment layout and every section's contents, refuses any
difference, and then bundles the app with its signature unchanged. The
loader's embedded `Info.plist` carries Wine's package version (`11.19`), so a
Wine of another version needs a release of that version.

## Patches

`patches/wine` are WoWSilicon's patches (`WoWSilicon/WoWSilicon`,
`Packaging/WineRuntime/patches/`) rebased onto `athei/wine`, and two of
wine-arm64-ci's own (`0007`, `0008`); the list, and the ones left out and
why, is in
[wine-arm64-ci](https://github.com/mybaixinyu/wine-arm64-ci#patches).

## Run

```bash
gh workflow run build.yml --repo mybaixinyu/wowsilicon-arm64
gh workflow run app.yml --repo mybaixinyu/wowsilicon-arm64 -f wine_run_id=<build.yml run>
```

`wine_build_ref` defaults to a wine-build commit, whose pins name the rest;
`loader_release` to the release whose `wine.app` matches its Wine. The
artifacts are `wowsilicon-arm64-<describe>` (above) and
`app-wowsilicon-arm64-<version>-<commit>.tar.xz`, `WoWSilicon.app` with
`BUILD-INFO.txt`.
