# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

_(no unreleased changes yet)_

## [1.0.0] - 2026-09-01

First semver release. Brings this template to the fleet standard established
in [keycloak-traefik-letsencrypt-docker-compose](https://github.com/heyvaldemar/keycloak-traefik-letsencrypt-docker-compose)
v1.2.0.

### Changed

- **Homebox was deployed from the floating `main` tag — now pinned to
  0.26.2** by `tag@sha256:digest`, alongside Traefik 3.7 (3.2's Docker
  client cannot talk to Docker Engine 29), in the compose `x-images`
  block. `git pull` delivers the tested combination.
- **SMTP is optional**: the SMTP variables default to empty and mail is
  inert until configured.

### Security

- **Credentials untracked from git.** The tracked `.env` carried the API
  key pepper and SMTP relay credentials — rotate both if reused.

### Added

- **Deployment Verification workflow**: actionlint; Trivy scans of both
  pinned images; weekly `check-pin-freshness` (digest drift + Homebox
  and Traefik release lag); and a deploy-and-test job that boots the
  stack and requires `/api/v1/status` to report healthy through Traefik.

[Unreleased]: https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/heyvaldemar/homebox-traefik-letsencrypt-docker-compose/releases/tag/v1.0.0
