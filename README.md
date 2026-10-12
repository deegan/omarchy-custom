# omarchy-custom

A personal pacman repository for [Omarchy](https://omarchy.org): **no AUR at runtime** — every
package here is built and hosted from our own infrastructure, from source or verified upstream
release artifacts, with no `yay`/`paru` step anywhere in the chain.

## What's in here

| Package | What it is |
|---|---|
| `goxlr-utility` | Unofficial GoXLR / GoXLR Mini configuration daemon + client, built from upstream's [GoXLR-on-Linux/goxlr-utility](https://github.com/GoXLR-on-Linux/goxlr-utility) source (AUR-only otherwise) |

## Using this repo

On the machine you want to install packages on, edit `/etc/pacman.conf`:

1. **Add this repo — above `[core]`/`[extra]`**, not below:
   ```ini
   [omarchy-custom]
   Server = http://<server-ip-or-hostname>:8081
   SigLevel = Optional TrustAll
   ```
   `SigLevel = Optional TrustAll` is required because this repo isn't GPG-signed by default (see
   [Signing](#signing) below). Only point this at a repo you trust — untrusted-repo package
   installation runs arbitrary code as root.

2. Sync and install:
   ```bash
   sudo pacman -Sy
   sudo pacman -S goxlr-utility
   ```

## Hosting your own instance

### Requirements

- Docker + Docker Compose — this is the only hard requirement. `bin/build` always builds each
  package inside a throwaway `archlinux` container, and `bin/publish` does the same for `repo-add`
  whenever it isn't already installed on the host, so nothing here requires the host to be Arch
  Linux or have an Arch toolchain.
- (Optional, faster) If the host already has `repo-add` in `PATH`, `bin/publish` uses it directly
  instead of spinning up a container. On Arch, that's the `pacman-contrib` package.

### Building and serving

```bash
git clone <this-repo-url> omarchy-custom
cd omarchy-custom
bin/build                 # builds every package under pkgbuilds/
                           # bin/build <pkgname> [<pkgname> ...] builds specific ones instead
bin/publish                # runs repo-add, refreshes the pacman database
docker compose up -d       # nginx now serves ./repo on :8081
```

`bin/build` runs each package in a clean, throwaway `archlinux` Docker container — it never builds
on the host, so the host doesn't need Arch's toolchain installed at all, just Docker. It also
mounts the local `repo/` into the build container as a `file://` pacman source (once `bin/publish`
has been run at least once), which would let a future package here resolve a dependency on another
package from this same repo.

`bin/publish` uses the host's `repo-add` if it finds one in `PATH`, and otherwise runs `repo-add` in
a throwaway `archlinux` container the same way — either way, no extra setup is needed on a
non-Arch host.

### Exposing it beyond a trusted LAN

`docker-compose.yml`'s `"8081:80"` port mapping binds to all interfaces by default, so the repo is
already reachable from other machines on the same network with no changes. If you want it reachable
over the internet rather than just a LAN:

- Put a reverse proxy (Caddy, nginx, Traefik) in front of it with a real TLS certificate — don't
  expose port 8081 directly to the internet over plain HTTP.
- Set up [signing](#signing) first. `SigLevel = Optional TrustAll` on the client is a reasonable
  tradeoff on a LAN you control; it's not one you want to ask anyone else to accept.

`restart: unless-stopped` in `docker-compose.yml` means the container comes back after a reboot as
long as Docker itself is enabled as a system service (`systemctl enable docker`).

If you also run `omarchy-gaming` (or another repo built the same way) on the same server, it
already binds `:8080` — that's why this repo uses `:8081` instead, so both can run as fully
independent containers side by side with no shared configuration between them.

## Signing

By default, `bin/publish` publishes an **unsigned** database (`repo-add` without `-s`), and
packages themselves aren't GPG-signed either. This is fine for local/LAN testing with
`SigLevel = Optional TrustAll`, but means there's no integrity check on anything this repo serves.

To sign:

1. Generate a GPG key (keep it outside this repo — it's not something to commit).
2. Set `GPG_KEY_ID` before running `bin/publish`:
   ```bash
   GPG_KEY_ID=<your-key-id> bin/publish
   ```
   If `repo-add` isn't on the host, `bin/publish` signs from inside its fallback container instead,
   mounting your host's `~/.gnupg` in so the key is still visible to `repo-add -s`.
3. Distribute the public key to clients and change `SigLevel` in their `pacman.conf` from
   `Optional TrustAll` to something that actually verifies, e.g. `Required DatabaseOptional`, after
   importing the key with `pacman-key`.

## Building packages

- `bin/build` with no arguments builds every `PKGBUILD` under `pkgbuilds/`. `bin/build <name>
  [<name> ...]` builds specific packages.
- `bin/publish` runs `repo-add` against everything currently in `repo/*.pkg.tar.zst` and refreshes
  the pacman database — using the host's `repo-add` if present, otherwise a throwaway `archlinux`
  container (same approach `bin/build` uses). Run it after every `bin/build`.
- To add a new package: create `pkgbuilds/<name>/PKGBUILD`, then `bin/build <name>` and
  `bin/publish`.

### Conventions

- **Binary repacks**: fetch the upstream release asset, verify its checksum against what the
  upstream release manifest/API already publishes — never download an artifact purely to compute a
  hash yourself if the checksum is already published somewhere.
- **Compiled packages**: always built in a clean `archlinux` Docker container via `bin/build`,
  never on the host.
- `pkgrel` resets to `1` on every `pkgver` bump.
- Packages that already exist unchanged in the official Arch repos don't belong here — only add one
  if it's patched, recompiled with different flags, or bundles opinionated config on top.
- Nothing under `repo/` is committed — it's build output, not source.

## Troubleshooting

- **`pacman -Sy` fails with a 404 on `omarchy-custom.db`, but `curl` can fetch the same URL fine**:
  almost always a typo in the `Server =` line (missing `://`, a leftover `$repo`/`$arch` template
  variable from a different repo layout, or a section-name mismatch — pacman requests
  `<section-name>.db`, so `[omarchy-custom]` must produce `omarchy-custom.db`, matching exactly).

## Known limitations

- The repo is unsigned by default (see [Signing](#signing)).

## License

This repo's own original content (the PKGBUILDs, `bin/build`/`bin/publish`, `docker-compose.yml`)
is [MIT](LICENSE), matching `omarchy`.

That covers our own glue code, not what it builds: every package here still carries whatever
license its upstream project uses — packaging something doesn't relicense it.
