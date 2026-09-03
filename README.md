# Homebox + Traefik + Let's Encrypt on Docker Compose

[![Deployment Verification](https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml/badge.svg?branch=main)](https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

This repository deploys Homebox (a fast inventory and organization system for your home, backed by embedded SQLite) behind Traefik with automatic Let's Encrypt TLS.

## Getting started

```bash
# 1. Clone
git clone https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose
cd homebox-traefik-letsencrypt-docker-compose

# 2. Create the two Docker networks the stack expects
docker network create traefik-network
docker network create homebox-network

# 3. Copy the environment template and fill in required values
cp .env.example .env
$EDITOR .env
# ^ Required: HBOX_AUTH_API_KEY_PEPPER, HOMEBOX_HOSTNAME,
#   TRAEFIK_HOSTNAME, TRAEFIK_ACME_EMAIL, TRAEFIK_BASIC_AUTH.

# 4. Deploy
docker compose -f homebox-traefik-letsencrypt-docker-compose.yml -p homebox up -d
```

Within a minute `https://${HOMEBOX_HOSTNAME}` serves the registration page. The first account registered is yours: open it right after deploy, and consider setting `HBOX_OPTIONS_ALLOW_REGISTRATION=false` afterwards.

### What success looks like

```bash
docker compose -f homebox-traefik-letsencrypt-docker-compose.yml -p homebox ps
curl -fsk "https://${HOMEBOX_HOSTNAME}/api/v1/status"   # {"health":true,...}
```

### Common first-deploy issues

- **Cert issuance fails.** DNS hasn't propagated or port 80 isn't reachable from the internet.
- **`docker compose up` fails with `set in .env`.** A required variable is empty; the error names it. Current Homebox requires the API key pepper.
- **Networks not found.** Step 2 was skipped.

## Supply chain trust

Two images ([`traefik`](https://hub.docker.com/_/traefik) and [`ghcr.io/sysadminsmedia/homebox`](https://github.com/sysadminsmedia/homebox/pkgs/container/homebox)) pinned to `tag@sha256:<digest>` as interpolation defaults in the compose `x-images` block. Earlier revisions deployed from the floating `main` tag. Every pull could land on an untagged development build; the pin ends that. `git pull` alone delivers the tested combination.

Two override levels exist per image. `<PREFIX>_IMAGE_VERSION` in `.env` swaps only the version of that image (Compose then pulls the tag, without a digest) and leaves every other pin as tested; `<PREFIX>_IMAGE_TAG` replaces the whole reference, digest included. The variable names are listed in `.env.example`. Nested defaults need Docker Compose v2.5 or newer (2022); v2.0 to v2.4 leave the inner `${...}` unexpanded and `docker compose up` fails with an invalid reference instead of deploying something unexpected.

The daily `check-pin-freshness` CI job re-resolves each pin against its registry and compares the pinned versions against the latest upstream releases. GitHub Actions are pinned by commit SHA; Dependabot keeps those fresh.

## Production checklist

- [ ] **Register your account immediately, then disable open registration** (`HBOX_OPTIONS_ALLOW_REGISTRATION=false`).
- [ ] **Strong pepper**: 64 random characters; regenerate the Traefik dashboard hash.
- [ ] **Back up the `homebox-data` volume**: it holds the SQLite database and uploaded photos.
- [ ] **Verify Let's Encrypt cert issuance** in the Traefik logs on first start.

## Unattended updates

Releases are the update channel: a tag is cut only after CI has built the pinned images, booted the full stack, and passed the smoke tests. `update.sh` moves a deployment to the newest tag and nothing else:

```bash
./update.sh --dry-run   # show what would be applied
./update.sh             # update within the current major and redeploy
```

Put it on a timer for hands-off minor/patch updates:

```bash
# crontab -e
17 5 * * *  /opt/homebox-traefik-letsencrypt-docker-compose/update.sh >> /var/log/homebox-update.log 2>&1
```

The script refuses to cross a MAJOR template version on its own: majors are breaking by definition and their release notes exist to be read. After reading them, `./update.sh ‑‑allow-major` performs the jump. It also refuses to touch a checkout with local modifications: your customization belongs in `.env`, which updates never overwrite.

This is deliberately a host-side script and not a container in the stack: an in-stack updater needs the Docker socket (root on the host) and turns "someone pushed to a repo" into "someone deployed to your machine" with no operator in the loop. A cron job under your own user updates only to tagged, CI-verified states and leaves the trust boundary where it was.

## Resource limits

Every service carries memory and CPU limits plus reservations as compose-level defaults: the same values CI boots the stack under. Override any of them in `.env` (the knobs and their defaults are listed in `.env.example`, e.g. `TRAEFIK_MEMORY_LIMIT=512m`) and the override survives every `git pull`. If a service is OOM-killed under real load, `docker inspect <container> ‑‑format '{{.State.OOMKilled}}'` says so; raise its `_MEMORY_LIMIT` and recreate.

## Backups

The `backups` container runs on a loop: an initial delay (`HOMEBOX_BACKUP_INIT_SLEEP`, default 30m), then every `HOMEBOX_BACKUP_INTERVAL` (default 24h) it takes a consistent copy of each SQLite database (`homebox.db`) through Python's `sqlite3` backup API - no application stop - and a `tar.gz` of the rest of the data directory (live database files excluded), into the `homebox-backups` volume; files older than `HOMEBOX_BACKUP_PRUNE_DAYS` (default 7) are pruned. Each artefact logs `... backup OK: <file> (<bytes> bytes)` or `FAILED` (kept as `<file>.failed`). Grep the log for `FAILED` from your monitoring.

**Verify backups are running:**

```bash
docker compose -p homebox logs backups | tail -5
docker compose -p homebox exec backups ls -la /srv/homebox/backups/
```

**Restore** a backup set with the interactive script (`chmod +x homebox-restore-data.sh` once): it stops homebox, unpacks the data archive over the data directory, restores each database from its consistent copy, and starts homebox again.

```bash
./homebox-restore-data.sh
```

**Off-host replication.** Backups live in a named volume on the same host. Bind-mount `HOMEBOX_BACKUPS_PATH` to a directory covered by your off-host backup solution (restic, rclone, Borg, S3 sync).

## Container hardening

Every service runs with `security_opt: no-new-privileges:true`, so a process cannot gain privileges through setuid binaries even if it escapes its initial capability set. Infrastructure containers (the reverse proxy, databases, caches, backups) run with `cap_drop: [ALL]` and add back only what their entrypoints need: `NET_BIND_SERVICE` for Traefik to bind :80/:443, `CHOWN`/`SETUID`/`SETGID` (and friends) for database images to own their data directory and drop to their service user. Application containers keep the default capability set on purpose: upstream images assume it, and a wrong guess there is a boot loop in production rather than a hardening win. CI boots the stack under exactly these settings on every push, so what ships is what was tested.

## Testing

The [Deployment Verification](https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml?query=branch%3Amain) workflow runs on every push, pull request, and every day at 06:00 UTC: actionlint, Trivy scans of both pinned images, the weekly freshness check, and a deploy-and-test job that boots the stack and requires `/api/v1/status` to report healthy through Traefik.

### Backup and restore, proven

`tests/e2e-backup-restore.sh` runs against the live stack and is what CI executes after the smoke test. The scenario that matters most is the restore roundtrip: the application is stopped, the baseline database copy is put back, and a row inserted after the baseline is gone. The tests stop the application briefly and write into its data directory. Run them on a staging copy with short intervals in `.env` (`HOMEBOX_BACKUP_INIT_SLEEP=15s`, `HOMEBOX_BACKUP_INTERVAL=60s`), never on production.

```bash
chmod +x tests/e2e-backup-restore.sh
./tests/e2e-backup-restore.sh
```

## Security notes

- Credentials are read from `.env` at deploy time; `.env` is gitignored and compose fails fast on missing required variables.
- **Pre-rotation advisory.** Releases before v1.0.0 (2026-09-01) shipped a tracked `.env` with a generated-looking API key pepper and SMTP relay credentials. Rotate both if your deployment reused them.
- SMTP is off by default.

---

## About the maintainer

<div align="center">

**Maintained by [Vladimir Mikhalev](https://github.com/heyvaldemar)** · Docker Captain · IBM Champion · AWS Community Builder

[YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1) · [Blog](https://heyvaldemar.com) · [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)

</div>
