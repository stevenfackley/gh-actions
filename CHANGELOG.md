# Changelog

All notable changes to the reusable workflows. Tags follow `v<MAJOR>`.

## [Unreleased]

### Added
- `wait-for-postgres` composite action: blocks until Postgres accepts
  authenticated connections (probed from the runner host via `pg_isready`, with
  `psql` and TCP fallbacks). Single shared home for the readiness logic that
  failed in the v1 `ci-astro` incident. Not wired into the reusable `ci-*`
  workflows — a reusable workflow can only reference a same-repo composite by a
  hardcoded `@ref`, which would couple it to the floating `v1` tag — so consumer
  workflows adopt it directly.

### Changed
- `r2-upload` composite finished for the upload path: per-file `Content-Type`
  guessing, null-safe directory walk, source/prefix normalization, configurable
  `wrangler-version`/`node-version`, `setup-node@v4` -> `@v6`. `delete-stale`
  remains unimplemented and now **fails fast** (was a silent no-op) because a
  reliable prune needs R2 S3-API credentials, not the Cloudflare API token.

### Notes
- The Postgres `services:` block (duplicated across `ci-dotnet`, `ci-go`,
  `ci-node`, `ci-python`, `ci-astro`) was **not** de-duplicated: Actions does not
  expand YAML anchors, composite actions cannot declare job-level `services:`,
  and a nested reusable workflow's service runs on a separate runner. README now
  documents the canonical stanza + intentional per-workflow variance. No `ci-*`
  workflow behavior changed.

## [v1] — 2026-05-31 (re-tagged)

### Fixed
- `ci-astro.yml`: postgres service healthcheck now passes `-U testuser -d e2etest`
  to `pg_isready`. Without those flags, `pg_isready` defaulted to the runner's
  OS user (root), repeatedly logged `FATAL: role "root" does not exist`, and
  eventually failed the service health gate — blocking the entire E2E job.

## [v1] — 2026-04-16

### Added
- Initial extraction from production repos:
  - `ci-dotnet.yml` from StackAlchemist + axon-main
  - `ci-python.yml` from undertow-engine
  - `ci-node.yml` from steveackleyorg
  - `ci-astro.yml` from steveackleyorg
  - `deploy-ec2-ssh.yml` from steveackleyorg
  - `deploy-k3s-helm.yml` from p1-opshub
  - `secret-scan.yml` from axon-main (banned telemetry packages + AKIA/PEM grep)
  - `triage-issues.yml` from p1-opshub
  - `release.yml` (new — tags `YYYYMMDD_<Project>_Release`)
- Composite actions: `setup-env`, `docker-build-push`, `verify-image`, `r2-upload` (v2 stub)
