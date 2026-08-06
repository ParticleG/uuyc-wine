# Repository Guidelines

## Project Overview

`uuyc-wine` is an unofficial Arch Linux/AUR compatibility package for the proprietary Windows NetEase UU Remote client. It is a packaging and Wine-lifecycle repository, not a conventional compiled application: the root-level Bash launcher installs the official payload into a dedicated per-user Wine prefix, configures WebView2 and Wine compatibility settings, starts the background server, and launches the client or a `uuremote:` URI.

NetEase does not support this Wine setup. Preserve the boundary between repository-owned integration code and vendor binaries downloaded by `PKGBUILD`.

## Architecture & Data Flow

1. `PKGBUILD` downloads checksummed upstream installer/EULA artifacts, extracts the icon, and packages them with `uuyc-wine`, `uuyc-wine.desktop`, and version markers.
2. `uuyc-wine` parses CLI or desktop URI input, validates packaged data, and derives XDG data/state paths.
3. The launcher optionally migrates a recognized older prefix, using `flock` and `/proc/*/environ` checks to avoid moving an active prefix.
4. It normalizes the Wine environment and checks readiness from required files, Wine registry values, and the `.uuyc-wine-package-release` stamp.
5. Under a setup lock, it initializes or repairs the 64-bit prefix, installs UU Remote and bundled WebView2, applies registry compatibility settings, and atomically updates the release stamp.
6. It starts `GameViewerServer.exe`, polls for readiness, then uses `exec` to launch `GameViewer.exe` or dispatch the URI.
7. `.github/workflows/aur-publish.yml` publishes an already locally verified recipe from GitHub `main` to the separate AUR repository; it is not a validation pipeline.

Persistent state is filesystem- and registry-based. There is no module graph, dependency-injection container, application state library, or language-level async runtime.

## Key Directories

- Repository root: all substantive source and packaging files; there is no `src/` source tree.
- `.github/workflows/`: AUR publication workflow.
- `src/` and `pkg/`: ignored, generated `makepkg` work/output directories; never treat them as source.
- `${XDG_DATA_HOME:-$HOME/.local/share}/uuyc-wine/wineprefix`: default managed Wine prefix.
- `${XDG_STATE_HOME:-$HOME/.local/state}/uuyc-wine`: setup log, setup lock, and launcher state.
- `/usr/share/uuyc-wine`: packaged installer plus `upstream-version` and `package-release` markers.

## Development Commands

```bash
# Build and install the Arch package (documented local workflow)
makepkg -si

# Regenerate AUR metadata after every PKGBUILD change
makepkg --printsrcinfo > .SRCINFO

# Exercise launcher paths after installation
uuyc-wine --version
uuyc-wine --setup-only
uuyc-wine --repair
uuyc-wine --uri 'uuremote:...'
uuyc-wine

# Inspect setup evidence
less "${XDG_STATE_HOME:-$HOME/.local/state}/uuyc-wine/setup.log"
```

The maintainer checklist requires syntax, desktop-entry, source, package-build, migration, and launch checks but defines no canonical commands for most categories. Useful targeted checks, when the tools are installed, are:

```bash
bash -n uuyc-wine uuyc-wine.install
shellcheck -s bash uuyc-wine uuyc-wine.install
desktop-file-validate uuyc-wine.desktop
makepkg --verifysource
```

Treat these four as practical local checks, not repository-enforced scripts.

## Code Conventions & Common Patterns

- Use Bash-specific constructs; `uuyc-wine` starts with `#!/usr/bin/bash` and `set -euo pipefail`.
- Use lowercase `snake_case` for functions/variables, `readonly` for immutable globals, and `local` inside functions.
- Follow the launcher's four-space indentation, `[[ ... ]]`, arithmetic `(( ... ))`, `case` parsing, and Bash arrays for argument forwarding.
- Quote expansions, preserve argument boundaries with `"$@"`/`"${array[@]}"`, use `--` for destructive file operations, and prefer `printf` over `echo`.
- Route fatal failures through `die`; point setup failures to `setup.log`. Handle expected failures explicitly with `if ! ...`, `|| return 1`, or narrowly scoped `|| true`.
- Installer exit status is not authoritative: UU Remote and WebView2 installers may return nonzero after writing valid state. Preserve the existing file/registry polling and post-install validation.
- Concurrency is process-based: advisory `flock`, double-checked setup state, detached server startup, and bounded polling loops. Preserve lock ordering and recheck state after acquiring a lock.
- Environment variables are the injection mechanism: `UUYC_WINE_PREFIX`, `UUYC_WINE_SHARE_DIR`, XDG variables, and `WINEDEBUG`. The launcher intentionally clears `WINEARCH` and WebView2 overrides outside controlled setup.
- Keep runtime state out of the repository. Do not edit generated Wine-prefix contents as source.
- Keep ShellCheck suppressions narrow and justified; existing exceptions are `SC2218` and `SC2016`.

## Important Files

- `uuyc-wine`: primary source and executable entry point; owns CLI parsing, migration, setup, validation, locking, and launch.
- `PKGBUILD`: canonical package metadata, dependencies, upstream URLs/checksums, build steps, installed layout, and file modes.
- `.SRCINFO`: generated AUR projection of `PKGBUILD`; regenerate it, do not maintain it independently.
- `README.md`: canonical user workflow, migration/troubleshooting guidance, and maintainer release checklist.
- `uuyc-wine.desktop`: desktop launcher and `x-scheme-handler/uuremote` registration.
- `uuyc-wine.install`: pacman removal hook; intentionally preserves per-user data.
- `.github/workflows/aur-publish.yml`: SHA-pinned, publish-only GitHub-to-AUR workflow using `AUR_SSH_PRIVATE_KEY`.
- `REUSE.toml` and `LICENSE`: licensing metadata for repository-owned files; the upstream EULA remains separate.

## Runtime/Tooling Preferences

- Target Arch Linux/AUR on `x86_64`; do not generalize hardcoded `/usr/bin` tools or procfs behavior to other distributions without an explicit portability change.
- Runtime requires `wine>=11.1`, `hicolor-icon-theme`, `procps-ng`, and `util-linux`; `libnotify` is optional. Package construction requires `7zip`.
- Use Arch `makepkg`; there is no Node, Bun, Python, container, Makefile, or language package manager/toolchain.
- Preserve fixed SHA-256 source checksums and intentional modes: launcher `0755`, data/desktop/license files `0644`.
- Keep GitHub Actions pinned to commit SHAs and never commit `AUR_SSH_PRIVATE_KEY` or runtime logs containing credentials, device identifiers, or remote-assistance codes.
- Packaging-only behavior changes increment `pkgrel`; upstream updates change `pkgver`, reset `pkgrel=1`, update sources/checksums, and regenerate `.SRCINFO`.

## Testing & QA

There is no automated test suite, test framework, fixture harness, coverage tooling, or coverage threshold. `PKGBUILD` has no `check()` function, and CI only publishes to AUR. A green publish workflow proves publication, not correctness.

Match verification to the change:

- CLI/package metadata: `uuyc-wine --version`, help, invalid/missing arguments, URI input, and arguments after `--`.
- Prefix/setup changes: use a disposable `UUYC_WINE_PREFIX`; run `--setup-only`, then `--repair`, and inspect `setup.log` plus required files/registry state.
- Migration changes: use disposable XDG paths and verify successful recognized migration, refusal while the old prefix is active, no destination overwrite/merge, and migration bypass with `UUYC_WINE_PREFIX`.
- Launch changes: verify server readiness followed by client launch and `uuremote:` dispatch in a graphical Wine session.
- Packaging changes: validate sources/checksums, build the package, confirm installed paths/modes, regenerate `.SRCINFO`, and review its diff against `PKGBUILD` intent.
- Desktop changes: validate the desktop file and exercise `Exec=uuyc-wine %u` plus the `uuremote:` MIME handler.

Favor real observable state over installer return codes, mocks, or source-text assertions.