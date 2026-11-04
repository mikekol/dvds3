# dvds3 — project notes

OBS Studio plugin (C) that adds a DVD-style bouncing logo source. Targets OBS Studio 33+.

## Versioning

Uses the Twitchy-bois scheme (reference: `Q:\repos\Twitchy-bois`).

- `version.json` at the repo root: `{ "version": "MAJOR.MINOR.PATCH", "build": N }`.
  `version` is bumped by hand for releases; `build` is bumped automatically on every commit.
- `.githooks/pre-commit` is a POSIX shell script with no dependencies beyond git and sed. It sets `build` to the value committed at HEAD plus one, and stages `version.json`. Bypass a single commit with `git commit --no-verify`.
- The hooks are registered by CMake at configure time (`git config core.hooksPath .githooks`). This repo has no `package.json`, so there is no npm `prepare` step.
- CMake reads `version.json` and renders the full version `MAJOR.MINOR.PATCH+build` into `version.h`. The plugin logs it on load.
- `.gitattributes` forces LF line endings for `.githooks/*`, because the hook scripts break with CRLF.

## Build

See `README.md` ("Building"). The plugin links only OBS's exported `libobs` package, built from the OBS 33.0.0 tag.
