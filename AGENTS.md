# Agent guide — docker-organizr

Primary agent guide for this repo. It is small (a Dockerfile + an s6 rootfs
overlay), so this single file is the whole guide — **no nested `AGENTS.md`**,
no `docs/`.

## What this is

`docker-organizr` builds the **upstream-model Docker image for Organizr** (the
homelab services dashboard). It contains *no PHP code*: it produces the runtime
(nginx + php-fpm + s6 init) and **`git clone`s the Organizr source at container
start**.

- **Base image:** `ghcr.io/organizr/base:<date_tag>-<arch>` (pinned in the
  `Dockerfile` `FROM`, currently `2023-11-30_13`): Alpine + **s6-overlay** +
  nginx + php-fpm (PHP 8.1 — cron calls `/usr/bin/php81`; socket
  `/var/run/php8-fpm.sock`).
- **Runtime model:** `root/etc/cont-init.d/40-install` clones/pulls
  **`https://github.com/causefx/Organizr`** into `/config/www/organizr` on the
  branch from the `branch` env var; restarting updates Organizr. nginx serves it
  on port 80; `root/org/cron.sh` runs `cron.php` every minute.

### Fork status

Fork of `organizr/docker-organizr` (origin `nantomarioni/docker-organizr`,
branch `master`). `README.md`, image refs (`ghcr.io/organizr/organizr`) and CI
credentials (`username: roxedus`) describe the **upstream** publish flow, not
this fork's. Local divergence: a few `40-install` tweaks + a base bump. Keep the
upstream diff small.

> **The homelab does not deploy this image.** The cluster runs the sibling
> `Organizr` fork's own self-contained image (`ghcr.io/nantomarioni/organizr`,
> see `../homelab-manifests/apps/organizr`). This repo also clones *upstream*
> `causefx/Organizr`, not the fork — to ship fork changes from here you would
> change the clone URL/branch in `40-install`.

## How to run things

No Makefile or compose. `COPY root/ /` is the only build step beyond `FROM`.

```bash
docker build -t docker-organizr:dev .                                  # defaults BASE_IMAGE=ghcr.io/organizr/base:2023-11-30_13, ARCH=linux-amd64
docker build --build-arg ARCH=linux-arm64 -t docker-organizr:dev .     # CI matrix: linux-amd64 | linux-arm64 | linux-arm-v7
docker run -d --name=organizr -p 8080:80 -v /path/to/config:/config -e PUID=1000 -e PGID=1000 -e branch=v2-master docker-organizr:dev
docker logs -f organizr && docker exec -it organizr /bin/bash
```

- `/config` is the only persistent state: the clone (`/config/www/organizr`),
  the nginx site conf (`/config/nginx/site-confs/default`), user data.
- `branch`: `v2-master`/`master` → `v2-master`; `v2-develop`/`develop`/`dev` →
  `v2-develop`; `manual` → skip auto-clone/update. `PUID`/`PGID` remap user
  `abc` (`lsiown -R abc:abc /config`). `fpm="false"` ENV is a legacy toggle.
- **CI** `.github/workflows/build.yml` (on push touching `Dockerfile`, `root/**`,
  the workflow; `skip ci` honoured): per-arch buildx pushes to Docker Hub +
  ghcr `organizr/organizr:<arch>`, then `imagetools create` stitches `:latest`.
  Publishing from this fork needs your own registry/credentials.

## Quality gate

No tests, no hadolint, no lint. The bar:

1. `docker build -t docker-organizr:dev .` succeeds (right `ARCH` off amd64).
2. The container starts, logs show the `Installing/Updating Organizr` banner and
   a successful clone, and `http://localhost:8080/` serves the setup/login page.
3. Changed a shell script? Run `shellcheck` (scripts carry `# shellcheck shell=bash`).

## Conventions

- **Dockerfile stays tiny**: pin the base via `${BASE_IMAGE}-${ARCH}`, `COPY
  root/ /`, `EXPOSE 80`, `VOLUME /config`. No build-time package installs —
  those belong in `organizr/base`. **Bumping PHP/nginx/Alpine = bumping the base
  date tag.**
- **s6-overlay init**: startup logic is `root/etc/cont-init.d/<NN>-name`
  (one-shot, numeric order; shebang `#!/usr/bin/with-contenv bash`).
- **Organizr is fetched at runtime, never COPYed**; clone URL hard-coded in
  `40-install`, branch via the `/Docker.txt` marker.
- **nginx site config** templated by `root/defaults/default` (`# V0.0.x` header);
  `40-install` migrates older user configs in place via `sed` when it sees an
  old version marker or the old `/config/www/Dashboard` path — preserve that
  pattern (bump the version comment) when changing served paths.
- **Tagging**: per-arch `:<arch>` + stitched `:latest`; the base image is the
  version pin, no semver.

## Where things live

```
Dockerfile                      # FROM organizr/base + COPY root/ / ; EXPOSE 80, VOLUME /config
root/etc/cont-init.d/40-install # pick branch, migrate nginx conf, clone/pull Organizr, perms, cron
root/org/cron.sh                # per-minute: php81 /config/www/organizr/cron.php (as abc)
root/defaults/default           # nginx site-conf template (root, php-fpm socket, /api/v2 route)
.github/workflows/build.yml     # multi-arch buildx → Docker Hub + ghcr; :latest stitch
organizr.xml ca_profile.xml     # Unraid Community-Apps template + profile
README.md LICENSE.md logo.gif   # upstream-oriented docs/assets
```

## Making a change — walkthrough

- **Bump the runtime** → `FROM` date tag; if the new base moves the php-fpm
  socket or PHP binary, also `root/defaults/default` (`fastcgi_pass`) and
  `root/org/cron.sh` (`/usr/bin/php81`).
- **Change served paths / nginx** → `root/defaults/default` + bump `# V0.0.x` +
  a matching `sed` migration in `40-install`.
- **Ship a different source/branch** → clone URL / branch mapping in `40-install`.
- **Add a startup step** → new `root/etc/cont-init.d/<NN>-name`.

Then rebuild, run, confirm logs + web UI; CI rebuilds on push.

## Ripple awareness

Open the sibling before declaring done. Siblings are `../<repo>` checkouts
(`github.com/nantomarioni/<repo>`).

- **`Organizr`** — coupled through Organizr's **runtime layout**: install path
  `/config/www/organizr` (`40-install` + nginx `root`); PHP version
  (`php81` in `cron.sh` + the base image); `40-install` rewrites
  `data/config/config.php` and `api/config/default.php` `'branch' => …` and runs
  `cron.php`; nginx `/api/v2` location ↔ `api/v2/index.php`. A layout, config
  path, cron location or PHP floor change there needs a matching change here
  (and vice-versa) — breakage is silent (container starts, cron/branch no-op).
  **Caveat:** this image clones upstream `causefx/Organizr`, so the fork's
  changes don't reach it unless the clone URL is re-pointed.
- **`homelab-manifests`** — none today (it deploys the fork's own image); only
  relevant if this image is ever re-pointed and adopted there.

## When stuck

- Organizr app behaviour / layout → `../Organizr` and <https://github.com/causefx/Organizr>.
- Base image contents (s6 services, php-fpm) → <https://github.com/organizr/docker-base>.
- Usage / parameters / migration history → `README.md` (upstream image refs).
- Unraid template → `organizr.xml` / `ca_profile.xml`.
