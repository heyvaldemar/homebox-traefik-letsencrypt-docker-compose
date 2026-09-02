# Homebox + Traefik + Let's Encrypt — Docker Compose

[![Deployment Verification](https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml/badge.svg?branch=main)](https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

This repository deploys **Homebox** — a fast inventory and organization system for your home, backed by embedded SQLite — behind **Traefik** with automatic **Let's Encrypt TLS**.

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

Within a minute `https://${HOMEBOX_HOSTNAME}` serves the registration page. **The first account registered is yours** — open it right after deploy, and consider setting `HBOX_OPTIONS_ALLOW_REGISTRATION=false` afterwards.

### What success looks like

```bash
docker compose -f homebox-traefik-letsencrypt-docker-compose.yml -p homebox ps
curl -fsk "https://${HOMEBOX_HOSTNAME}/api/v1/status"   # {"health":true,...}
```

### Common first-deploy issues

- **Cert issuance fails.** DNS hasn't propagated or port 80 isn't reachable from the internet.
- **`docker compose up` fails with `set in .env`.** A required variable is empty; the error names it — current Homebox requires the API key pepper.
- **Networks not found.** Step 2 was skipped.

## Supply chain trust

Two images — [`traefik`](https://hub.docker.com/_/traefik) and [`ghcr.io/sysadminsmedia/homebox`](https://github.com/sysadminsmedia/homebox/pkgs/container/homebox) — pinned to `tag@sha256:<digest>` as interpolation defaults in the compose `x-images` block. Earlier revisions deployed from the floating `main` tag — every pull could land on an untagged development build; the pin ends that. `git pull` alone delivers the tested combination.

The weekly `check-pin-freshness` CI job re-resolves each pin against its registry and compares the pinned versions against the latest upstream releases. GitHub Actions are pinned by commit SHA; Dependabot keeps those fresh.

## Production checklist

- [ ] **Register your account immediately, then disable open registration** (`HBOX_OPTIONS_ALLOW_REGISTRATION=false`).
- [ ] **Strong pepper** — 64 random characters; regenerate the Traefik dashboard hash.
- [ ] **Back up the `homebox-data` volume** — it holds the SQLite database and uploaded photos.
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

The script refuses to cross a MAJOR template version on its own — majors are breaking by definition and their release notes exist to be read. After reading them, `./update.sh --allow-major` performs the jump. It also refuses to touch a checkout with local modifications: your customization belongs in `.env`, which updates never overwrite.

This is deliberately a host-side script and not a container in the stack: an in-stack updater needs the Docker socket (root on the host) and turns "someone pushed to a repo" into "someone deployed to your machine" with no operator in the loop. A cron job under your own user updates only to tagged, CI-verified states and leaves the trust boundary where it was.

## Testing

The [Deployment Verification](https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml?query=branch%3Amain) workflow runs on every push, pull request, and every Monday at 06:00 UTC: actionlint, Trivy scans of both pinned images, the weekly freshness check, and a deploy-and-test job that boots the stack and requires `/api/v1/status` to report healthy through Traefik.

## Security Notes

- Credentials are read from `.env` at deploy time; `.env` is gitignored and compose fails fast on missing required variables.
- **Pre-rotation advisory.** Releases before v1.0.0 (2026-09-01) shipped a tracked `.env` with a generated-looking API key pepper and SMTP relay credentials. Rotate both if your deployment reused them.
- SMTP is off by default.

---

## About the maintainer

<div align="center">

**Maintained by [Vladimir Mikhalev](https://github.com/heyvaldemar)** — Docker Captain · IBM Champion · AWS Community Builder

[YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1) · [Blog](https://heyvaldemar.com) · [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)

</div>
