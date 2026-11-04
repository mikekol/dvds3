# dvds3
![nightly](https://github.com/univrsal/dvds3/workflows/nightly/badge.svg)
Will it hit the corner?

obs-studio plugin, which adds a dvd screen saver source type
Maintained my Mike Kolitz (TheCap'm)

![logo](./data/dvd.png)

## Building

Requires OBS Studio 33.0.0 or newer. The plugin's only build dependency is libobs, which OBS exports as a CMake package.

```sh
cmake -S . -B build -DCMAKE_INSTALL_PREFIX=<install-dir>
cmake --build build --config RelWithDebInfo
```

### Windows (x64)

libobs must be built from the OBS 33 source and installed (Development component) first. Then configure the plugin with:

| Option | Value |
| --- | --- |
| `libobs_DIR` | `<obs-sdk>/cmake` (contains `libobsConfig.cmake`) |
| `w32-pthreads_DIR` | `<obs-sdk>/cmake` (contains `w32-pthreadsConfig.cmake`) |
| `SIMDe_INCLUDE_DIR` | `<obs-deps>/include` (contains `simde/simde-common.h`) |

```powershell
cmake -S . -B build64 -G "Visual Studio 18 2026" -A x64 `
  -Dlibobs_DIR="<obs-sdk>\cmake" `
  -Dw32-pthreads_DIR="<obs-sdk>\cmake" `
  -DSIMDe_INCLUDE_DIR="<obs-deps>\include"
cmake --build build64 --config RelWithDebInfo
```

- `<obs-deps>` is the extracted `windows-deps-<date>-x64.zip` from [obs-deps releases](https://github.com/obsproject/obs-deps/releases).
- The `w32-pthreads_DIR` and `SIMDe_INCLUDE_DIR` overrides are needed because libobs' package config does not find its own dependencies when installed to a custom prefix.
- The `libobs_DIR` override is needed because the package is not on CMake's default search path.

The build output is `build64/RelWithDebInfo/dvd-screensaver.dll`.

### Linux (x64)

The Ubuntu 26.04 archive ships libobs 32.1, which is too old, so build libobs from the OBS 33.0.0 tag first. The CI job does the same. Install the build dependencies listed in the `Install build dependencies` step of `.github/workflows/nightly.yml`, then:

```sh
git clone --depth 1 --branch 33.0.0 https://github.com/obsproject/obs-studio.git
cmake -S obs-studio -B obs-studio/build -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_INSTALL_PREFIX=<obs-sdk> \
  -DENABLE_UI=OFF -DENABLE_PLUGINS=OFF -DENABLE_SCRIPTING=OFF -DENABLE_BROWSER=OFF \
  -DENABLE_FRONTEND_API=OFF -DENABLE_TESTS=OFF -DENABLE_AJA=OFF -DENABLE_VLC=OFF \
  -DENABLE_NEW_MPEGTS_OUTPUT=OFF -DCOPIED_DEPENDENCIES=NO -DCOPY_DEPENDENCIES=NO
cmake --build obs-studio/build --target libobs obs-frontend-api --parallel
cmake --install obs-studio/build --component Development

cmake -S . -B build -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -Dlibobs_DIR=<obs-sdk>/lib/cmake/libobs
cmake --build build --parallel
```

The output is `build/dvd-screensaver.so`. Install it with `cmake --install build --prefix <dir>`, which writes the zip layout (`dvd-screensaver/bin/64bit` and `dvd-screensaver/data`).

The frontend subtree of the OBS source is configured even with `ENABLE_UI=OFF`, so the Qt6 and `nlohmann-json` development packages are needed too.

### macOS (universal)

Build libobs the same way, with the universal architectures and the OBS macOS dependencies:

- `<obs-deps>` is the extracted `macos-deps-<date>-universal.tar.xz` from [obs-deps releases](https://github.com/obsproject/obs-deps/releases).
- Add `-DCMAKE_OSX_ARCHITECTURES="arm64;x86_64" -DCMAKE_OSX_DEPLOYMENT_TARGET=13.0 -DCMAKE_PREFIX_PATH=<obs-deps>` to both the libobs and the plugin configure steps.
- The plugin configure uses `-Dlibobs_DIR=<obs-sdk>/lib/cmake/libobs`.

The plugin is `build/dvd-screensaver.so`. It has no `lib` prefix, which OBS needs. The macOS build is not signed or notarized.

## Continuous integration

`.github/workflows/nightly.yml` has three jobs:

- **Linux x64** builds libobs 33.0.0 from source (cached), then produces the zip layout and a `.deb`.
- **macOS universal** builds libobs 33.0.0 against the obs-deps universal tarball, then produces the zip layout.
- **Windows x64** builds libobs 33.0.0 from source (cached), then produces the zip and the MSI.

Each job uploads its output as a workflow artifact.

## Versioning

The plugin version lives in `version.json` at the repo root:

```json
{ "version": "1.1.0", "build": 0 }
```

- `version` is the `MAJOR.MINOR.PATCH` release. Bump it by hand for releases.
- `build` is incremented automatically by the `.githooks/pre-commit` hook on every commit.

The full version is rendered as semver build metadata, `1.1.0+N`. The plugin logs it when OBS loads it, and CMake generates it into `version.h`.

The hooks are registered when CMake configures the project (`git config core.hooksPath .githooks`). To register them without configuring, run that command yourself. The hook is a plain shell script with no dependencies beyond git and sed. To skip the bump for one commit, use `git commit --no-verify`.
