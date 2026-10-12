<div align="center">

# omarchy-custom

**A personal pacman repository for [Omarchy](https://omarchy.org) — no AUR at runtime.**

Every package is built from upstream source in a clean Docker container and served from your own
box. No `yay`, no `paru`, no AUR helper anywhere in the chain.

![Arch Linux](https://img.shields.io/badge/Arch_Linux-pacman-1793D1?logo=archlinux&logoColor=white)
![Docker](https://img.shields.io/badge/built_with-Docker-2496ED?logo=docker&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

</div>

---

## What is this?

Some software only lives in the AUR. If you'd rather not add the AUR to an Omarchy install, you can
still get it: this repo holds the `PKGBUILD`s, builds them reproducibly, and publishes the result as
a normal pacman repository. On the client it's just one extra section in `pacman.conf`, and after
that `pacman -S <pkg>` works like it does for any other package.

It's a small, personal sibling of [`omarchy-gaming`](https://github.com/deegan/omarchy-gaming):
the same idea, scaled down to a handful of one-off packages instead of a curated gaming stack.

## Packages

| Package | Description | Upstream |
|---|---|---|
| **`goxlr-utility`** | Unofficial config daemon + client for the TC-Helicon GoXLR and GoXLR Mini. Installs a udev rule and reloads udev on install/upgrade/remove. | [GoXLR-on-Linux/goxlr-utility](https://github.com/GoXLR-on-Linux/goxlr-utility) |
| **`cairn-tracker`** | Native Linux Hearthstone tracker (Python/PySide6). Reads the game's own logs: no Wine, no injection. | [Ratpido-dev/cairn](https://github.com/Ratpido-dev/cairn) |

<details>
<summary><b>Packaging notes</b> (what differs from upstream)</summary>

**`goxlr-utility`** is built from the tagged GitHub release source. One local patch: the daemon's
`reqwest` feature is switched from `rustls-tls` to `native-tls`. rustls' `ring` backend fails to
link on this build toolchain (cargo drops the link-search path for `ring`'s build-script output when
it's built twice in the unit graph, even from a clean build). `native-tls` uses the system OpenSSL
and only affects the daemon's firmware-update HTTPS check.

**`cairn-tracker`** is adapted from upstream's own `packaging/aur/PKGBUILD` (which lags a release
behind). It adds `qt6-svg` to `depends` (the icon test fails without the Qt SVG plugin) and runs the
test suite in `check()` with `QT_QPA_PLATFORM=offscreen`. Upstream's KWin follow-script is
deliberately left out, since Omarchy runs Hyprland. Use window rules on app_id `cairn` instead (see
upstream's README).

</details>

## Using the repo

On the machine you want to install packages on, add this to `/etc/pacman.conf` **above** `[core]`
and `[extra]`:

```ini
[omarchy-custom]
SigLevel = Optional TrustAll
Server = http://<server-ip-or-hostname>:8081
```

Then:

```bash
sudo pacman -Sy
sudo pacman -S goxlr-utility cairn-tracker
```

> [!WARNING]
> `SigLevel = Optional TrustAll` is needed because the repo isn't GPG-signed by default (see
> [Signing](#signing)). Installing a package runs arbitrary code as root, so only point this at a
> repo you control.

## How it works

```
 pkgbuilds/<name>/PKGBUILD
          │
          │  bin/build            clean archlinux container → makepkg
          ▼
 repo/<name>-<ver>.pkg.tar.zst
          │
          │  bin/publish          repo-add → omarchy-custom.db
          ▼
 repo/ ──► nginx container (:8081) ──► pacman -Sy on your machines
```

| Piece | Role |
|---|---|
| `pkgbuilds/` | One directory per package, each with a `PKGBUILD` (and any `.install` scripts). |
| `bin/build` | Builds one or all packages in a throwaway `archlinux` Docker container, so the host never needs an Arch toolchain. Pacman's package cache and cargo's crate registry live in named volumes so rebuilds are fast. Finished packages are moved into `repo/`. |
| `bin/publish` | Runs `repo-add` over `repo/*.pkg.tar.zst` to (re)generate `omarchy-custom.db`. Uses the host's `repo-add` if present, otherwise runs it in a container. Signs if `GPG_KEY_ID` is set. |
| `bin/check-updates` | Compares each PKGBUILD's `pkgver` against upstream using [nvchecker](https://github.com/lilydjwg/nvchecker). |
| `docker-compose.yml` | An nginx container serving `./repo` read-only on port `8081`. |
| `repo/` | Build output: packages and the pacman database. **Gitignored.** |

## Hosting your own

**Requirements:** Docker and Docker Compose. That's it. (If `repo-add` is on the host, from
`pacman-contrib` on Arch, `bin/publish` uses it directly.)

```bash
git clone git@github.com:deegan/omarchy-custom.git
cd omarchy-custom

bin/build                  # build everything (or: bin/build goxlr-utility)
bin/publish                # generate the pacman database
docker compose up -d       # serve ./repo on http://localhost:8081
```

Verify from any Arch system or container:

```ini
[omarchy-custom]
SigLevel = Optional TrustAll
Server = http://localhost:8081
```

```bash
sudo pacman -Sy && sudo pacman -S goxlr-utility
```

### Exposing it beyond your LAN

The `8081:80` mapping binds all interfaces, so other machines on your network can already reach it.
For the internet:

- Put a reverse proxy (Caddy, nginx, Traefik) with real TLS in front. Don't expose 8081 over plain HTTP.
- Set up [signing](#signing) first.

`restart: unless-stopped` brings the container back after a reboot, provided Docker is enabled
(`systemctl enable docker`). Port `8081` is used so it can run next to `omarchy-gaming`, which
takes `8080`.

## Keeping packages up to date

```bash
bin/check-updates          # report packages with a newer upstream release (exits non-zero if any)
bin/check-updates --fix    # bump pkgver, reset pkgrel to 1, and update the sha256
bin/build <pkg> && bin/publish
```

`--fix` downloads the new (small) tag archive once to hash it. A bump can break patches, notably
`goxlr-utility`'s `native-tls` `sed`, so always rebuild and check the result. Every package needs a
section in [`bin/nvchecker.toml`](bin/nvchecker.toml), otherwise it is silently never checked.

## Adding a package

1. Create `pkgbuilds/<name>/PKGBUILD`.
2. Add a section for it in `bin/nvchecker.toml`.
3. `bin/build <name>`, then `bin/publish`.

**Conventions**

- Compiled packages are always built in the clean container via `bin/build`, never on the host.
- Binary repacks: read the checksum from the upstream manifest or API. Don't download an artifact
  just to hash it.
- `pkgrel` resets to `1` on every `pkgver` bump.
- Don't add packages that exist unchanged in the official repos. Only add ones that are patched,
  rebuilt with different flags, or bundle config.
- Never commit anything under `repo/`.

## Signing

By default `bin/publish` produces an **unsigned** database and the packages aren't signed either, so
nothing this repo serves has an integrity check. Fine on a trusted LAN, not beyond it.

To sign:

1. Generate a GPG key (keep it out of this repo).
2. Publish with it:
   ```bash
   GPG_KEY_ID=<your-key-id> bin/publish
   ```
   If `repo-add` isn't on the host, `bin/publish` mounts `~/.gnupg` into its container to sign.
3. Import the public key on clients with `pacman-key`, then tighten `SigLevel` from
   `Optional TrustAll` to something that verifies, e.g. `Required DatabaseOptional`.

## Troubleshooting

**`pacman -Sy` returns 404 for `omarchy-custom.db`, but `curl` fetches it fine.** Almost always a
typo in the `Server =` line (missing `://`, a stray `$repo`/`$arch` from another layout). Pacman
requests `<section-name>.db`, so `[omarchy-custom]` must match `omarchy-custom.db` exactly.

## License

The repo's own content (PKGBUILDs, `bin/` scripts, `docker-compose.yml`) is [MIT](LICENSE). That
covers the glue code only: every package keeps the license of its upstream project.
