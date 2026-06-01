# Changelog

All notable changes to the reusable workflows. Tags follow `v<MAJOR>`.

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
