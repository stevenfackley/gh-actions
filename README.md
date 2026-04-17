# gh-actions — Reusable workflows & composite actions

Shared CI/CD for `stevenfackley/*` repos. Pin to `@v1`. Breaking changes → `@v2`.

## Reusable workflows (workflow_call)

| File | Purpose | Source of truth |
|------|---------|-----------------|
| `.github/workflows/ci-dotnet.yml` | .NET 10 build + test + AOT + deps audit + telemetry scan | StackAlchemist + axon-main |
| `.github/workflows/ci-python.yml` | Python 3.11 lint + pytest with optional Postgres | undertow-engine |
| `.github/workflows/ci-node.yml` | Node 22 lint + typecheck + vitest + docker build | steveackleyorg |
| `.github/workflows/ci-astro.yml` | Astro 6 + Postgres 16 + Playwright E2E | steveackleyorg |
| `.github/workflows/deploy-ec2-ssh.yml` | EC2 via SSH + docker compose + health check | steveackleyorg |
| `.github/workflows/deploy-k3s-helm.yml` | K3s via local-registry sync + Helm upgrade | p1-opshub |
| `.github/workflows/secret-scan.yml` | grep for telemetry SDKs + AWS keys + PEM | axon-main |
| `.github/workflows/triage-issues.yml` | Move referenced issues to Ready For Test | p1-opshub |
| `.github/workflows/release.yml` | Tag `YYYYMMDD_<Project>_Release` | new |

## Composite actions

| Action | Purpose |
|--------|---------|
| `.github/actions/setup-env` | Writes `.env` from secrets with chmod 600 |
| `.github/actions/docker-build-push` | BuildKit cache + GHCR login + `sha-<short>` tag |
| `.github/actions/verify-image` | p1-opshub's pre-deploy GHCR manifest check |
| `.github/actions/r2-upload` | Cloudflare R2 sync (v2) |

## Usage

```yaml
# .github/workflows/ci.yml in any repo
name: ci
on:
  pull_request:
  push: { branches: [main] }
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
jobs:
  ci:
    uses: stevenfackley/gh-actions/.github/workflows/ci-dotnet.yml@v1
    with:
      project-name: my-service
      solution-path: src/MyService.slnx
    secrets: inherit
```

## Versioning

- `@v1` — initial extraction. Stable.
- `@main` — tracks latest; use only for development.
- `@v2+` — breaking input/output changes. Repos opt in individually.

See CHANGELOG.md for per-version details.

## Known limitations (v1)

1. **`deploy-k3s-helm.yml` is p1-opshub-shaped.** It requires a `helm-script` (`.sh` file present on the self-hosted runner), `helm-set-key`, `deployment-name`, `namespace`, `local-image-name`, and `short-sha`. Not yet a general-purpose reusable workflow. New K3s-deployed repos should copy p1-opshub's wiring end-to-end, or wait for a generalized `deploy-k3s-helm-basic.yml`.
2. **`ci-astro.yml` defaults `workspace: "apps/site"`** (steveackleyorg monorepo shape). Single-package repos must pass `workspace: ""`. The workflow also expects `test:unit` / `test:integration` npm scripts alongside `test:e2e`.
3. **`triage-issues.yml` requires `after_sha`**. Callers must pass `with: { after_sha: ${{ github.sha }} }`.
