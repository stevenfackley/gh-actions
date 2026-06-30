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
| `.github/actions/r2-upload` | Cloudflare R2 directory sync (wrangler + API token) |
| `.github/actions/wait-for-postgres` | Host-side Postgres readiness gate (`pg_isready` loop) |

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

## Postgres in CI

`ci-dotnet`, `ci-go`, `ci-node`, `ci-python`, and `ci-astro` each start Postgres
as a job-level service container. **That `services:` block cannot be factored into
a single shared definition** — this is a hard GitHub Actions limitation, not an
oversight:

- Actions workflow YAML does **not** expand YAML anchors/aliases, so there is no
  `&pg` / `*pg` reuse.
- Composite actions can only declare `steps:` — they **cannot** declare job-level
  keys like `services:`.
- A nested reusable workflow runs as a **separate job on a separate runner**, so a
  service container it starts is not reachable from the caller job's steps.

The only mechanisms that would "DRY" it are a `docker run`-based composite (trades
the robust native `services:` lifecycle for hand-rolled container management) or
standardizing every consumer's image/credentials (a behavior change). Neither is
worth it for live consumer pipelines, so the blocks stay per-workflow. Keep them
in sync against this **canonical stanza**:

```yaml
services:
  postgres:
    # run-postgres toggles the service: empty image == no container.
    image: ${{ inputs.run-postgres && 'postgres:16' || '' }}
    env:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: test_db
    ports:
      - 5432:5432
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5
```

Per-workflow variance (intentional — must match each workflow's injected
connection string, so do **not** blindly unify):

| Workflow | image | user / password | db | health-cmd | service always on? |
|----------|-------|-----------------|----|------------|--------------------|
| `ci-dotnet` | `postgres:16` | `test` / `test` | `test_db` | `pg_isready` | no (`run-postgres`) |
| `ci-go` | `postgres:16` | `test` / `test` | `test_db` | `pg_isready` | no (`run-postgres`) |
| `ci-python` | `postgres:16` | `test` / `test` | `test_db` | `pg_isready` | no (`run-postgres`) |
| `ci-node` | `postgres:16-alpine` | `testuser` / `testpassword` | `test_db` | `pg_isready` | no (`run-postgres`) |
| `ci-astro` | `postgres:16-alpine` | `testuser` / `testpassword` | `e2etest` | `pg_isready -U testuser -d e2etest` | yes (E2E job) |

> The `ci-astro` health-cmd carries `-U testuser -d e2etest` deliberately — bare
> `pg_isready` defaulted to the runner OS user and failed the gate (fixed in v1).
> That per-db readiness fragility is exactly what `wait-for-postgres` centralizes.

### `wait-for-postgres`

A composite that blocks until Postgres is accepting authenticated connections,
probed from the runner host (the same path the test process uses). Use it in a
repo's own workflow when you want a single, shared home for readiness logic
instead of tuning each `--health-cmd`:

```yaml
- uses: stevenfackley/gh-actions/.github/actions/wait-for-postgres@v1
  with:
    user: testuser
    database: e2etest
```

> It is **not** wired into the reusable `ci-*` workflows: a reusable workflow can
> only reference a same-repo composite by a hardcoded `@ref`, which would couple
> the action to the floating `v1` tag. Those workflows keep their inline service
> health-check; new/consumer workflows can adopt the composite directly.

## Versioning

- `@v1` — initial extraction. Stable.
- `@main` — tracks latest; use only for development.
- `@v2+` — breaking input/output changes. Repos opt in individually.

See CHANGELOG.md for per-version details.

## Known limitations (v1)

1. **`deploy-k3s-helm.yml` is p1-opshub-shaped.** It requires a `helm-script` (`.sh` file present on the self-hosted runner), `helm-set-key`, `deployment-name`, `namespace`, `local-image-name`, and `short-sha`. Not yet a general-purpose reusable workflow. New K3s-deployed repos should copy p1-opshub's wiring end-to-end, or wait for a generalized `deploy-k3s-helm-basic.yml`.
2. **`ci-astro.yml` defaults `workspace: "apps/site"`** (steveackleyorg monorepo shape). Single-package repos must pass `workspace: ""`. The workflow also expects `test:unit` / `test:integration` npm scripts alongside `test:e2e`.
3. **`triage-issues.yml` requires `after_sha`**. Callers must pass `with: { after_sha: ${{ github.sha }} }`.
4. **`r2-upload` `delete-stale` is not implemented.** Upload/overwrite works; pruning stale remote keys needs R2 S3-API credentials (Access Key ID/Secret), a different secret than the Cloudflare API token the action takes. `delete-stale: true` fails fast rather than silently not deleting.
