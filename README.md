# wowsilicon-arm64

Builds [WoWSilicon](https://github.com/WoWSilicon/WoWSilicon) around the native
arm64 Wine of [athei/wine-build](https://github.com/athei/wine-build), both
from source, with WoWSilicon's Wine patches on top, without waiting for a
release of either.

- Wine is `athei/wine` at any commit, plus `patches/wine`.
- FEX, Mesa (KosmicKrisp and the Vulkan loader), DXMT and the Wine SDK are
  built by wine-build's own `build-wine.sh` at the pinned commits; mtld3d
  comes from the release `pins/redist.env` names. FEX gets `patches/fex`.
- `bundle-wine.sh` assembles the full bundle, with Wine's graphics tests, and
  runs its smoke test. The runner's virtual GPU cannot create a Vulkan
  instance, so `SMOKE_SKIP_GPU=1` (`patches/wine-build/0003`) leaves the
  Vulkan and wined3d device checks to a real Mac.
- `patches/wine-build/0002` bundles Homebrew libraries that their siblings
  load by `@rpath` (`libbrotlicommon` of current `freetype`), which the
  bundle step left out, so FreeType failed to load.

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

- `0001` puts the bundle in `Contents/Resources/steamapps/common/Wine`, with
  `Wine` linking to it, since macOS Game Mode takes a process whose real
  path contains `steamapps/common` for a game; signs the app without
  `--deep`, which would re-sign `wine.app` and drop its entitlements; and
  makes `tools/wine-runtime/validate.sh` check an arm64 bundle.
- `0002` makes it `WoWSilicon arm64` (`io.github.mybaixinyu.wowsilicon-arm64`)
  beside the official app: its settings live in `Application Support/WoWSilicon-arm64`,
  its default bottle is `~/WoWSilicon-arm64`, Wine is told where its audio
  control files are, and it has no update feed, so Sparkle never replaces it
  with the official build.

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
`Packaging/WineRuntime/patches/`) rebased onto `athei/wine`; the list, and the
ones left out and why, is in
[wine-arm64-ci](https://github.com/mybaixinyu/wine-arm64-ci#patches).

## Run

```bash
gh workflow run build.yml --repo mybaixinyu/wowsilicon-arm64
gh workflow run app.yml --repo mybaixinyu/wowsilicon-arm64 -f wine_run_id=<build.yml run>
```

The inputs default to what `arm64-11.19-1` was built from. The artifacts are
`wowsilicon-arm64-<describe>.tar.xz`, a `wine/` tree with `BUILD-INFO.txt`,
and `app-wowsilicon-arm64-<version>-<commit>.tar.xz`, `WoWSilicon-arm64.app`
with `BUILD-INFO.txt`.
