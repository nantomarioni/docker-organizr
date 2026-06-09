# Agent guide — docker-organizr

Tool-agnostic guide for coding agents (Cursor, Claude Code, Copilot, Codex, …).
This is the container **packaging** repo for Organizr; it is small (a Dockerfile
+ an s6 rootfs overlay), so this single root file is the whole guide — there are
**no nested `AGENTS.md` files**.

## What this is

`docker-organizr` builds the **Docker image that runs Organizr** — an HTPC /
homelab services-organizer PHP app. It is the packaging counterpart to the
sibling app repo **`infra/Organizr`** (the PHP source). This repo contains *no
PHP application code*: it produces the runtime (web server + PHP + init system)
and pulls the Organizr source in at container startup.

- **Base image:** `ghcr.io/organizr/base:<date_tag>-<arch>` (pinned in the
  `Dockerfile` `FROM`, currently `2023-11-30_13`). That base is Alpine Linux
  with an **s6-overlay** init system, **nginx**, and **php-fpm** (PHP 8.1 — the
  cron script calls `/usr/bin/php81`; nginx talks to php-fpm over the unix
  socket `/var/run/php8-fpm.sock`).
- **Runtime model:** Organizr is **not baked into the image.** On every
  container start, `root/etc/cont-init.d/40-install` `git clone`s (or
  `git pull`s) the Organizr source from upstream **`https://github.com/causefx/Organizr`**
  into the `/config` volume at `/config/www/organizr`, on the branch selected by
  the `branch` env var. This means restarting the container updates Organizr,
  and the built-in updater never breaks the install.
- **Process model:** nginx serves `/config/www/organizr` on port 80, proxying
  `*.php` to php-fpm; a per-minute cron (`root/org/cron.sh`) runs Organizr's
  `cron.php`.

### Fork status (important)

This repo is a **fork** of the upstream `organizr/docker-organizr` (origin:
`git@github.com:nantomarioni/docker-organizr.git`). The `README.md`, image
references (`ghcr.io/organizr/organizr`), and CI credentials
(`username: roxedus`) still point at upstream — they describe the *upstream*
publish flow, not necessarily this fork's. Local divergence is a handful of
`40-install` tweaks + a base bump (see `git log`). Keep the upstream diff small
and rebase-friendly.

Note also that the runtime clones **upstream `causefx/Organizr`**, NOT the
sibling `infra/Organizr` fork (which is itself `nantomarioni/Organizr`, a fork
of `causefx/Organizr`). If you intend the image to ship your fork's PHP changes,
the `git clone` URL / `branch` in `40-install` is what you'd change — today it
does not.

## How to build / run

There is no Makefile or compose file in this repo. Builds are plain `docker build`
(multi-arch via buildx in CI).

### Build locally

The `FROM` line composes `${BASE_IMAGE}-${ARCH}`. For a local single-arch build
on x86-64 you can rely on the defaults:

```bash
# defaults: BASE_IMAGE=ghcr.io/organizr/base:2023-11-30_13, ARCH=linux-amd64
docker build -t docker-organizr:dev .

# explicit arch (matches the CI matrix: linux-amd64 | linux-arm64 | linux-arm-v7)
docker build --build-arg ARCH=linux-amd64 -t docker-organizr:dev .
```

`COPY root/ /` overlays the s6 rootfs onto the base image; that's the only build
step beyond the `FROM`.

### Run

```bash
docker run -d \
  --name=organizr \
  -p 8080:80 \
  -v /path/to/config:/config \
  -e PUID=1000 -e PGID=1000 \
  -e branch=v2-master \
  docker-organizr:dev
```

- **Port:** container `80` (`EXPOSE 80`).
- **Volume:** `/config` (`VOLUME /config`) — holds the cloned Organizr source
  (`/config/www/organizr`), the nginx site config
  (`/config/nginx/site-confs/default`), and all user data. This is the only
  persistent state.
- **Env vars:**
  - `branch` (default `v2-master`) — Organizr branch to clone. Valid:
    `v2-master`/`master` → `v2-master`; `v2-develop`/`develop`/`dev` →
    `v2-develop`; `manual` → skip the auto-clone/update (use a pre-existing
    `/config/www/organizr`).
  - `PUID` / `PGID` — uid/gid the `abc` user is mapped to (LinuxServer-style
    permission handling; `40-install` `lsiown -R abc:abc /config`).
  - `fpm="false"` is set in the `Dockerfile` ENV (legacy toggle; the migration
    notes in `README.md` say php-fpm-over-socket is now the only mode).
- Shell into a running container: `docker exec -it organizr /bin/bash`.
  Logs: `docker logs -f organizr`.

### CI build / publish

`.github/workflows/build.yml` runs on push when `Dockerfile`, `root/**`, or the
workflow itself changes (skippable with `skip ci` in the commit message):

1. **build** job — matrix over `linux-amd64 / linux-arm64 / linux-arm-v7`; sets
   up QEMU + buildx, logs into Docker Hub and ghcr.io, and
   `docker/build-push-action` builds + pushes one per-arch tag to **both**
   `organizr/organizr:<arch>` (Docker Hub) and `ghcr.io/organizr/organizr:<arch>`.
   (The repo name has `docker-` stripped to derive the image name.)
2. **publish** job — `docker buildx imagetools create` stitches the three
   per-arch tags into a single multi-arch `:latest` manifest on both registries.

> The published image namespace is upstream (`organizr/organizr`,
> `username: roxedus`); pushing from this fork would require your own
> registry/credentials. Don't assume the workflow publishes "your" image as-is.

## Quality gate

There is **no automated test suite, no hadolint config, and no lint step.** The
only gate is:

1. **The image builds cleanly:** `docker build -t docker-organizr:dev .`
   (with the right `ARCH` build-arg for non-amd64).
2. **The container starts and serves Organizr:** run it (above), watch
   `docker logs -f organizr` for the `Installing/Updating Organizr` banner and a
   successful `git clone`, then hit `http://localhost:8080/` and confirm the
   Organizr setup/login page loads.

If you change a shell script, sanity-check it with `shellcheck` (the scripts
already carry `# shellcheck shell=bash` directives), even though CI doesn't run
it.

## Conventions

- **Dockerfile is intentionally tiny:** pin the base via the `FROM`
  `${BASE_IMAGE}-${ARCH}` expression, `COPY root/ /`, declare `EXPOSE`/`VOLUME`.
  Don't add build-time package installs here — that belongs in the upstream
  `organizr/base` image. **Bumping the runtime (PHP/nginx/Alpine) = bumping the
  base-image date tag**, not editing this Dockerfile's body.
- **Init system is s6-overlay** (from the base image). Startup logic goes in
  `root/etc/cont-init.d/<NN>-name` scripts (one-shot, ordered by the numeric
  prefix); the only one here is `40-install`. Shebang is
  `#!/usr/bin/with-contenv bash` so env vars (`branch`, `PUID`, …) are available.
- **Organizr source is fetched at runtime, never COPYed in.** The clone target
  is `/config/www/organizr`, the branch comes from the `branch` env var (mapped
  via a `/Docker.txt` marker), and the source repo is hard-coded to
  `causefx/Organizr` in `40-install`.
- **nginx site config** is templated by `root/defaults/default` (versioned with a
  `# V0.0.x` header). `40-install` migrates older user configs in place via
  `sed` when it detects the old `# V0.0.x` marker or the old
  `/config/www/Dashboard` path — preserve that migration pattern (bump the
  version comment) if you change the served paths.
- **Permissions:** the container runs Organizr as user/group `abc` (remapped to
  `PUID`/`PGID`); `40-install` ends with `lsiown -R abc:abc /config`.
- **Image tagging/versioning:** per-arch tags `:<arch>` + a stitched `:latest`
  multi-arch manifest. There is no semver tag scheme in this repo — the base
  image is the version pin.

## Where things live

```
docker-organizr/
├── Dockerfile                     # FROM organizr/base + COPY root/ / ; EXPOSE 80, VOLUME /config
├── root/                          # s6 rootfs overlay (COPYed to / in the image)
│   ├── etc/cont-init.d/40-install # startup: pick branch, migrate nginx conf, git clone/pull Organizr, set perms, add cron
│   ├── org/cron.sh                # per-minute job: php81 /config/www/organizr/cron.php (as user abc)
│   └── defaults/default           # nginx site-conf template (served root, php-fpm socket, /api/v2 route)
├── .github/workflows/build.yml    # multi-arch buildx → Docker Hub + ghcr; :latest manifest stitch
├── organizr.xml                   # Unraid Community-Apps template (image, port, /config path, PUID/PGID)
├── ca_profile.xml                 # Unraid CA repository/profile metadata
├── logo.gif                       # Organizr logo asset (referenced by docs, not used at build/runtime)
├── README.md                      # upstream-oriented usage docs (image refs point at upstream)
└── LICENSE.md
```

## Making a change — walkthrough

- **Bump PHP / nginx / Alpine (the runtime):** these come from the base image.
  Edit the `FROM` tag in `Dockerfile` (the `${BASE_IMAGE}` default date, e.g.
  `2023-11-30_13` → a newer `organizr/base` tag), rebuild, and verify the
  container still serves Organizr. If the new base moves the php-fpm socket or
  PHP binary path, also update `root/defaults/default` (`fastcgi_pass unix:…`)
  and `root/org/cron.sh` (`/usr/bin/php81`).
- **Change served paths / nginx behavior:** edit `root/defaults/default`, bump
  its `# V0.0.x` header, and add a matching in-place `sed` migration block in
  `root/etc/cont-init.d/40-install` so existing `/config` volumes get upgraded
  (mirror the existing `Dashboard → organizr` / `/api/v2` migrations).
- **Change which Organizr source / branch is shipped:** edit the `git clone`
  URL and/or branch mapping in `root/etc/cont-init.d/40-install`. (Today it
  clones upstream `causefx/Organizr`.)
- **Add a startup step:** add a new `root/etc/cont-init.d/<NN>-name` script
  (`#!/usr/bin/with-contenv bash`), numbered to order it relative to `40-install`.

After any change: rebuild the image, run it, and confirm logs + the web UI
(see Quality gate). CI rebuilds/publishes on push to the watched paths.

## Ripple awareness — the `infra/Organizr` pairing

This image and the sibling **`infra/Organizr`** PHP app are a packaging/app
pair, coupled through Organizr's **runtime layout**:

- **Install path:** the app expects to live at `/config/www/organizr`
  (`40-install` clones there; `root/defaults/default` sets nginx `root` there).
- **PHP version:** `root/org/cron.sh` hard-codes `php81` and the base image
  provides php-fpm; if Organizr starts requiring a newer PHP, the **base image**
  and these references must move together.
- **Branch/config plumbing:** `40-install` rewrites Organizr's
  `data/config/config.php` and `api/config/default.php` `'branch' => …` value
  and runs its `cron.php` — those file paths are part of Organizr's layout. If
  the app moves/renames them, this repo breaks silently (the container starts
  but cron/branch handling no-ops).
- **API routing:** the nginx `/api/v2` location must match Organizr's API entry
  (`api/v2/index.php`).

So: a change to Organizr's directory layout, entrypoint/cron file locations,
config-file paths, or required PHP version **requires a matching change here**
(usually in `40-install`, `root/defaults/default`, `root/org/cron.sh`, or the
base-image tag) — and vice-versa. **Caveat:** the image clones *upstream*
`causefx/Organizr`, so changes to the local `infra/Organizr` fork do **not**
reach the running container unless you re-point the clone URL. See
`infra/AGENTS.md` (the "Organizr pairing" note) one directory up.

## When stuck

- **Organizr app behavior / layout** → the sibling `infra/Organizr` repo and
  upstream `https://github.com/causefx/Organizr`.
- **The base image** (what's installed, s6 service definitions, php-fpm setup)
  → `https://github.com/organizr/docker-base` (`ghcr.io/organizr/base`).
- **s6-overlay / `cont-init.d` semantics** → the s6-overlay docs.
- **Usage / parameters / migration history** → `README.md` (note: its image
  references describe the *upstream* publish, see Fork status).
- **Unraid template** → `organizr.xml` / `ca_profile.xml`.
