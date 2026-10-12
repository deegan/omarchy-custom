# omarchy-custom

A personal Arch Linux pacman repository for Omarchy: no AUR at runtime, everything built and
hosted from our own infrastructure. Sibling project to `omarchy-gaming` (same pattern, scaled down
to one-off personal packages instead of a curated gaming stack).

## Repository structure

    omarchy-custom/
      pkgbuilds/          ← one directory per package, each containing a PKGBUILD
      repo/                ← built .pkg.tar.zst files + pacman database (gitignored)
      bin/
        build              ← build one or all packages
        publish            ← run repo-add and refresh the database
      docker-compose.yml   ← nginx serving ./repo on localhost:8081 for local testing
      CLAUDE.md            ← this file (assistant/contributor instructions)
      README.md            ← user-facing guide: using the repo, hosting your own instance, signing

## Packages

- `goxlr-utility` — unofficial GoXLR/GoXLR Mini config daemon+client. AUR-only upstream; built
  here from the tagged GitHub release source instead, since the user didn't want to add the AUR to
  this Omarchy install. One local patch: `daemon/Cargo.toml`'s reqwest feature is switched from
  `rustls-tls` to `native-tls`, because rustls' `ring` backend fails to link on this build
  toolchain (cargo drops the link-search path for `ring`'s build-script output when it ends up
  built twice in the unit graph — reproducible even from a clean build, not a caching artifact).
  `native-tls` (system OpenSSL) sidesteps `ring` entirely and only affects the daemon's
  firmware-update HTTPS check, nothing else.
- `cairn-tracker` — native Linux Hearthstone tracker (pure Python/PySide6, `arch=any`) from
  `../cairn` (upstream `Ratpido-dev/cairn`), built from the tagged `v1.0.4` GitHub source.
  Adapted from upstream's own `packaging/aur/PKGBUILD` (stale at 1.0.3). Adds `qt6-svg` to
  `depends` (the icon test fails without the Qt SVG plugin; pyside6 doesn't pull it in) and runs
  the test suite in `check()` with `QT_QPA_PLATFORM=offscreen`. The KWin follow-script that
  `install.sh` installs is deliberately not packaged (Omarchy is Hyprland); for Hyprland, window
  rules on app_id `cairn` are needed (see upstream README). The source sha256 was computed from the
  GitHub tag tarball (small; no upstream manifest exists).

## PKGBUILD conventions

- Binary repacks: fetch upstream release asset, verify sha256, repackage. Never download
  more than needed — read the upstream manifest/index to get the URL and checksum,
  don't fetch the artifact just to hash it.
- Compiled packages: use a clean `archlinux` Docker container for builds to ensure
  reproducibility. Do not build on the host directly.
- `pkgrel` resets to 1 on every `pkgver` bump.

## Local test workflow

```bash
# Start the local repo server
docker compose up -d

# Build a single package
bin/build goxlr-utility

# Refresh the pacman database
bin/publish

# Test from an Arch system or container:
# [omarchy-custom]
# Server = http://localhost:8081
# SigLevel = Optional TrustAll
pacman -Sy
pacman -S goxlr-utility
```

## What NOT to do

- Do not shell out to `yay` or any AUR helper. We are the AUR helper.
- Do not add packages that already exist in official Arch repos unchanged — only add
  them if we're patching, recompiling with different flags, or bundling config.
- Do not fetch full release artifacts to compute checksums. Read upstream manifests.
- Do not commit anything under `repo/` — built packages are artifacts, not source.

## Progress

Scaffolding is done: `docker-compose.yml` + `nginx/repo.conf`, `bin/build`/`bin/publish` (adapted
from `omarchy-gaming`'s, with the gaming-specific meta-package build ordering dropped since there's
only one package so far). `pkgbuilds/goxlr-utility/` has a working `PKGBUILD` (see the `native-tls`
patch above). `bin/build` and `bin/publish` have been run: `repo/` holds
`goxlr-utility-1.2.4-1-x86_64.pkg.tar.zst`, its `-debug` package and the `omarchy-custom` database
(built 2026-08-30). `goxlr-utility 1.2.4-1` is installed on this machine, and `/etc/pacman.conf` has
an `[omarchy-custom]` section pointing at `http://192.168.2.2:8083` (not the local `localhost:8081`
compose server). The local `docker compose` server is not currently running (the container on
port 8080 belongs to `omarchy-gaming`).

## Next steps

1. Run `docker compose up -d` and verify a clean `pacman -Sy && pacman -S goxlr-utility` against
   the local repo (`localhost:8081`), or confirm what serves `192.168.2.2:8083`.
2. Check for a newer upstream `goxlr-utility` release than 1.2.4.
