# Handoff: open work on dvds3

State as of the `v1.2.0+1` release (`master`, tag `v1.2.0+1`), plus the dual-ID change that follows it.

## Release

- **Release notes are wrong.** `v1.2.0+1` says existing "Screensaver" sources keep working. The source is named "Birdie Screensaver 2" in the scene that prompted this work. The notes should be corrected with `gh release edit v1.2.0+1 --repo mikekol/dvds3`.
- **The next release needs both IDs.** `v1.2.0+1` registers only `obs_dvd_screensaver`. The dual-ID change (`dvds3_source` plus `obs_dvd_screensaver`) is on `master` after the tag and needs its own release.
- **Release assets are Windows x64 only.** Linux and macOS are not built.

## Runtime verification

- **Not tested in a running OBS with the dual-ID build.** The last OBS check was the earlier single-ID build. Install the new MSI (`build\dvd-screensaver-<version>.msi`) and check `%APPDATA%\obs-studio\logs` for both `dvds3_source` and `obs_dvd_screensaver` registering without errors.
- **OBS folder state.** `C:\Program Files\obs-studio\obs-plugins\64bit` holds the `dvd-screensaver.dll` from an earlier MSI test, not the current build. The old `obs-dvd-screensaver.dll` is in `build\backup-old-plugin`. Remove that backup once the new build loads.
- **Origin of the old DLL is unknown.** The repo has never used the name `obs-dvd-screensaver`. The DLL that OBS had installed was built against the pre-OBS-33 API and registered `obs_dvd_screensaver`. Its author and version are not known.

## CI

- **Linux job is commented out** in `.github/workflows/nightly.yml`. Re-enable it and check that `libobs-dev` on `ubuntu-26.04` provides the libobs CMake config. The job fails early if the version or config is missing.
- **Windows job passes** on `windows-latest` with Visual Studio 18 2026, and it uploads the MSI and zip. Watch for the runner image dropping VS 2026.
- **Node.js:** actions are on Node 24 (`checkout@v7`, `cache@v6`, `upload-artifact@v7`). The run shows no deprecation annotation.

## Installer

- **Install location:** the MSI defaults to `C:\Program Files\obs-studio`. The registry lookup for OBS was removed because it read the 32-bit view. Users with OBS elsewhere pick the folder on the installer's folder page, or pass `OBSDIR=` to `msiexec`.
- **Build with `-arch x64`.** WiX builds x86 by default, which installs into `Program Files (x86)`. CI and the local build both pass the flag.
- **README placement:** the MSI puts `README.txt` in the plugin's data folder, not the OBS root as the old Inno installer did.
- **No Start menu shortcuts.** The Inno installer had them.

## Versioning

- **Hook is shell-only.** `.githooks/pre-commit` needs only git and sed. It bumps the build number on every commit, not on every push.
- **Hook registration:** CMake sets `core.hooksPath` at configure time. Fresh clones need one configure run.
- **Line endings:** `core.autocrlf` is on, and git warns about CRLF on `version.json` and other text files. `.gitattributes` only covers `.githooks/*`. Consider extending it to other text files.

## Build environment

- **PDB lock:** `build\plugin` failed with `LNK1201` on the PDB. Deleting the stale PDB fixed it. The cause is unknown, so a fresh build directory is the fallback.
- **Pre-existing warnings:** two `C4090` const-qualifier warnings in `src/dvd.c`, from `obs_data_get_string` assignments. Harmless.

## Repository

- **The fork is still attached to `univrsal/dvds3`.** Breaking it needs a GitHub-side step: the web UI's "Leave fork network" option, or GitHub Support. Pushes already go only to `origin` (`mikekol/dvds3`).
- **Stale README example:** the Versioning example still shows `1.1.0`. The current version is in `version.json`.
