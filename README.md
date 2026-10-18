# wowsilicon-arm64

Builds the native arm64 Wine of [athei/wine-build](https://github.com/athei/wine-build)
from source with WoWSilicon's Wine patches on top, without waiting for a
wine-build release.

- Wine is `athei/wine` at any commit, plus `patches/wine`.
- FEX, Mesa (KosmicKrisp and the Vulkan loader), DXMT and the Wine SDK are
  built by wine-build's own `build-wine.sh` at the pinned commits; mtld3d
  comes from the release `pins/redist.env` names.
- `bundle-wine.sh --runtime-only` assembles the bundle and runs its smoke
  test.

## The loader

The loader needs `com.apple.developer.cross-architecture-support`, which macOS
honours only with a provisioning profile that grants it and a signature by a
certificate of the profile's team. That certificate is athei's, so the signed
`wine.app` is taken from a wine-build release (`loader_release`).
`patches/wine-build/0001-bundle-a-prebuilt-signed-loader-app.patch` adds
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
```

The inputs default to what `arm64-11.19-1` was built from. The artifact is
`wowsilicon-arm64-<describe>.tar.xz`, a `wine/` tree with `BUILD-INFO.txt`.
