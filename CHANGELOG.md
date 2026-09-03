# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

_(no unreleased changes yet)_

## [1.5.0] - 2026-09-03

### Added

- **Per-image version overrides.** Every pin in the `x-images` block is
  now `${<PREFIX>_IMAGE_TAG:-repo:${<PREFIX>_IMAGE_VERSION:-tag@sha256:digest}}`.
  Set `<PREFIX>_IMAGE_VERSION` in `.env` to run a different version of one
  image while every other pin stays as tested (Compose pulls that tag
  without a digest), or `<PREFIX>_IMAGE_TAG` to replace the whole
  reference as before. A deployment that sets neither is unchanged. The
  freshness job, the Trivy matrix and the fleet digest automation resolve
  the nested default before reading a pin. Needs Docker Compose v2.5 or
  newer (2022): v2.0 to v2.4 leave the inner `${...}` unexpanded and
  `docker compose up` fails with an invalid reference instead of
  deploying something unexpected.

## [1.4.0] - 2026-09-02

### Security

- **Container hardening.** Every service runs with
  `security_opt: no-new-privileges:true` (no privilege escalation via
  setuid binaries even if a process escapes its initial capability
  set). Infrastructure containers (the reverse proxy, databases,
  caches, backups) drop every Linux capability and add back only what
  their entrypoints need (bind :80/:443, chown a data directory, drop to
  the service user). Application containers keep the default capability
  set: upstream images assume it, and a wrong guess there is a boot loop
  in production, not a hardening win. CI boots the stack under these
  settings on every push.

### Added

- **`tests/e2e-backup-restore.sh`**: scenarios against the live stack,
  run by CI on every push: the required-variable guard fires, a backup
  set is produced, the archive is readable, the database copy passes `PRAGMA integrity_check`, a cycle that cannot
 write its archive is reported as `FAILED`, **restore 
  replaces the data** (the application is stopped, the baseline database copy is put back, and a row inserted after the baseline is gone), and pruning removes only old files.

## [1.3.1] - 2026-09-02

### Fixed

- A database file that does not exist yet (the application creates it on
  first start) is skipped with a note instead of being reported as a
  failed backup; the first cycle after a fresh install no longer logs
  `FAILED`.

## [1.3.0] - 2026-09-02

### Added

- **A `backups` service** for the inventory database and uploaded attachments: on a loop it takes a consistent copy of each SQLite database (`homebox.db`) through Python's `sqlite3` backup API - no application stop - and a `tar.gz` of the rest of the data directory (live database files excluded), logs `OK` or `FAILED` per artefact (a failed archive is kept as `.failed`), and prunes only its own files. Schedule knobs (`HOMEBOX_BACKUP_INIT_SLEEP`, `HOMEBOX_BACKUP_INTERVAL`, `HOMEBOX_BACKUP_PRUNE_DAYS`, path and names) have defaults listed in `.env.example`.
- **`homebox-restore-data.sh`**: interactive restore of a backup set: stops homebox, unpacks the data archive, restores each database copy, starts homebox.
- CI waits for the first backup cycle and proves the archives are readable and the database copy passes `PRAGMA integrity_check`.

## [1.2.0] - 2026-09-02

### Added

- **Resource limits on every service, as `.env`-overridable defaults.**
  Each service now carries memory and CPU limits plus reservations
  (`<SERVICE>_MEMORY_LIMIT`, `_CPU_LIMIT`, `_MEMORY_RESERVATION`,
  `_CPU_RESERVATION`, defaults listed in `.env.example`). Set any of
  them in `.env` and the override survives every `git pull`. The
  defaults are what CI boots the stack under, so they are known to be
  enough for a fresh install; raise a limit if a service is OOM-killed
  under your real load (`docker inspect` shows `OOMKilled=true`).

## [1.1.0] - 2026-09-02

### Added

- **`update.sh`**: unattended updates to the newest tagged release,
  and nothing else: a tag is cut only after CI has booted the pinned
  images and passed the smoke tests, so "update to the latest tag" means
  "update to a combination a machine has already run". It refuses to
  cross a major version on its own (`--allow-major` after reading the
  notes), refuses a checkout with local modifications, and supports
  `--dry-run`. Put it on a cron timer for hands-off minor/patch updates.

## [1.0.0] - 2026-09-01

First semver release. Brings this template to the fleet standard established
in [keycloak-traefik-letsencrypt-docker-compose](https://github.com/heyvaldemar/keycloak-traefik-letsencrypt-docker-compose)
v1.2.0.

### Changed

- **Homebox was deployed from the floating `main` tag, now pinned to
  0.26.2** by `tag@sha256:digest`, alongside Traefik 3.7 (3.2's Docker
  client cannot talk to Docker Engine 29), in the compose `x-images`
  block. `git pull` delivers the tested combination.
- **SMTP is optional**: the SMTP variables default to empty and mail is
  inert until configured.

### Security

- **Credentials untracked from git.** The tracked `.env` carried the API
  key pepper and SMTP relay credentials. Rotate both if reused.

### Added

- **Deployment Verification workflow**: actionlint; Trivy scans of both
  pinned images; weekly `check-pin-freshness` (digest drift + Homebox
  and Traefik release lag); and a deploy-and-test job that boots the
  stack and requires `/api/v1/status` to report healthy through Traefik.

[Unreleased]: https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose/compare/v1.5.0...HEAD
[1.5.0]: https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose/compare/v1.4.0...v1.5.0
[1.4.0]: https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose/compare/v1.3.1...v1.4.0
[1.3.1]: https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose/compare/v1.3.0...v1.3.1
[1.3.0]: https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose/compare/v1.2.0...v1.3.0
[1.2.0]: https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose/releases/tag/v1.0.0
