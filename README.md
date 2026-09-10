# Universal GitHub CI/CD Workflows

Reusable GitHub Actions workflows for consistent CI and safe, opt-in CD across repositories.

This repository is **public** and licensed under the **MIT License**, so any GitHub repository can call the reusable workflows directly, subject to that repository's own GitHub Actions policy.

Stable workflow references:

- **Universal CI v1** — `morexyz/github-workflows/.github/workflows/universal-ci.yml@v1`
- **Universal CD v1** — `morexyz/github-workflows/.github/workflows/universal-cd.yml@cd-v1`

Current immutable release references:

- **Universal CI v1.0.2** — `morexyz/github-workflows/.github/workflows/universal-ci.yml@ci-v1.0.2`
- **Universal CD v1.0.2** — `morexyz/github-workflows/.github/workflows/universal-cd.yml@cd-v1.0.2`

The design goal is simple: consumer repositories keep tiny caller workflows, while common orchestration lives here. Project-specific CI or deployment behavior stays in the consumer repository through explicit hooks.

## Architecture

```text
Developer / AI Agent
        ↓
      Branch
        ↓
        PR
        ↓
  Universal CI
        ↓
lint / typecheck / test / build
        ↓
      Review
        ↓
      Merge
        ↓
 optional Universal CD
        ↓
   dry-run or real deploy
        ↓
 verify GitHub Environment
        ↓
 predeploy → deploy → healthcheck
```

CI and CD are intentionally separate. **Universal CI never deploys.** Universal CD never performs a real deployment unless the caller explicitly opts in.

## Repository layout

```text
.github/workflows/
├── universal-ci.yml
├── universal-cd.yml
└── self-validation.yml

docs/
└── BRANCH_PROTECTION.md

AI_project_instruction.md
README.md
LICENSE
```

## Repository self-protection

This central workflow repository protects itself before shared-workflow changes reach `main`.

- `main` requires pull requests, linear history, deletion protection, and force-push protection.
- The aggregate `Self Validation` status check is required before merge.
- Pull requests must be up to date with `main` before the required check can satisfy the merge gate.
- `Self Validation` combines Actionlint, a Universal CI smoke test, and a Universal CD disabled/dry-run smoke test.
- `v1` and `cd-v1` are protected against deletion, force-pushes, and non-linear history while remaining deliberately fast-forwardable for reviewed backward-compatible releases.
- `ci-v*` and `cd-v*` release tags are protected against update and deletion after creation with no bypass actors.
- Creation of matching release tags is separately restricted to the configured repository-administrator bypass role.
- GitHub Release Immutability is enabled for new releases; `ci-v1.0.2` and `cd-v1.0.2` are verified immutable releases.
- External GitHub Actions are pinned to full commit SHAs, and the Actionlint archive used by self-validation is checked against a fixed SHA-256 digest.

See [`docs/BRANCH_PROTECTION.md`](docs/BRANCH_PROTECTION.md) for the maintained hardening model.

## Versioning

Use the maintained major-version branches for normal consumption:

```text
CI: @v1
CD: @cd-v1
```

These are mutable major-version pointers. They may advance only for backward-compatible, reviewed, validated fixes.

For the current fixed GitHub-immutable releases, use:

```text
CI: @ci-v1.0.2
CD: @cd-v1.0.2
```

Current immutable release mappings:

```text
ci-v1.0.2 → 26eb1c1b82dbe666e6c3ef77b3ce72d79941ac62
cd-v1.0.2 → 26eb1c1b82dbe666e6c3ef77b3ce72d79941ac62
```

The `Protect immutable releases` tag ruleset blocks updates and deletions for `ci-v*` and `cd-v*` tags after creation with no bypass actors. The separate `Restrict release tag creation` ruleset limits creation of those tags to the configured repository-administrator bypass role. GitHub Release Immutability is enabled for newly published releases.

Earlier `v1.0.0` and `v1.0.1` releases remain valid historical releases, but they were published before GitHub Release Immutability was enabled. Their tags remain protected by the repository ruleset.

For maximum explicit supply-chain pinning, use the exact hardened workflow commit SHA:

```yaml
uses: morexyz/github-workflows/.github/workflows/universal-ci.yml@26eb1c1b82dbe666e6c3ef77b3ce72d79941ac62
```

Breaking workflow contracts should use new major-version references rather than silently changing v1 behavior.

---

# Universal CI v1

Universal CI detects supported **root-level** project files and runs conventional checks automatically.

## Quick start

Create `.github/workflows/ci.yml` in the consumer repository:

```yaml
name: CI

on:
  pull_request:
  push:

permissions:
  contents: read

jobs:
  ci:
    uses: morexyz/github-workflows/.github/workflows/universal-ci.yml@v1
```

That is enough for a supported root-level project.

## Supported project types

| Project type | Detection | Default checks |
|---|---|---|
| Custom | `scripts/ci.sh` | Runs the repository-owned CI hook |
| Node.js / JavaScript / TypeScript | `package.json` | Installs dependencies; runs declared `lint`, `typecheck`, `test`, `build` scripts |
| PHP | `composer.json` | Composer validation/install, PHP lint, declared Composer `lint`, `test`, `build` scripts |
| Python | `pyproject.toml`, `requirements.txt`, `requirements-dev.txt`, `setup.py`, `setup.cfg` | Installs dependencies/project where safely inferable, compiles sources, runs Ruff/Pytest when available |
| Go | `go.mod` | `go vet ./...`, `go test ./...` |
| Rust | `Cargo.toml` | format check, `cargo check`, tests |
| Solidity / Foundry | `foundry.toml` | `forge fmt --check`, `forge build`, `forge test` |

Detection is root-level only in v1. For monorepos or unusual layouts, use the custom CI hook.

## Custom CI hook

If the consumer repository contains:

```text
scripts/ci.sh
```

that hook takes precedence over all automatic ecosystem jobs.

Example:

```bash
#!/usr/bin/env bash
set -euo pipefail

npm ci
npm run lint
npm test
npm run build
```

Use `scripts/ci.sh` for monorepos, service/database setup, unsupported stacks, unusual commands, or projects that require a strict validation sequence.

The shared workflow runs it with Bash, so executable permission is not required, although keeping scripts executable is good repository hygiene.

## Node.js behavior

Universal CI supports npm, pnpm, Yarn, and Bun.

Package-manager selection order:

1. `package.json#packageManager`
2. recognized lockfile
3. npm fallback

Lockfile behavior:

- npm: `npm ci` with `package-lock.json` or `npm-shrinkwrap.json`
- pnpm: `pnpm install --frozen-lockfile` with `pnpm-lock.yaml`
- Yarn: immutable/frozen-lockfile install with `yarn.lock`
- Bun: frozen-lockfile install with `bun.lock` or `bun.lockb`

Only declared scripts among these names run automatically:

```text
lint
typecheck
test
build
```

A missing script is skipped, not treated as a failure.

## CI inputs

| Input | Default |
|---|---|
| `node-version` | `22` |
| `pnpm-version` | `10.26.1` |
| `yarn-version` | `1.22.22` |
| `bun-version` | `1.2.22` |
| `corepack-version` | `0.34.0` |
| `python-version` | `3.12` |
| `php-version` | `8.3` |
| `go-version` | `1.24.x` |
| `rust-toolchain` | `stable` |

Example override:

```yaml
jobs:
  ci:
    uses: morexyz/github-workflows/.github/workflows/universal-ci.yml@v1
    with:
      node-version: '20'
      python-version: '3.11'
```

## Multiple detected ecosystems

If several supported root-level ecosystems are detected, their jobs may run in parallel. If `scripts/ci.sh` exists, it is authoritative and automatic ecosystem jobs are skipped.

## Unsupported repositories

If no supported root-level project type is detected, Universal CI succeeds rather than failing by default. Add `scripts/ci.sh` to define project-specific CI.

---

# Universal CD v1

Universal CD is intentionally platform-neutral. It does **not** guess whether a project deploys with SSH, Docker, Kubernetes, a cloud CLI, static hosting, or blockchain tooling.

The consumer repository owns deployment implementation through shell hooks.

## Safety defaults

```text
enabled = false
dry_run = true
environment_name = production
allowed_environments = production
shared_target = false
external_lock_confirmed = false
```

A real deployment requires both:

```yaml
enabled: true
dry_run: false
```

This two-switch gate is deliberate.

For a real deployment, Universal CD also verifies that the selected GitHub Environment already exists in the **caller repository** before the deploy job can start. This prevents a misspelled environment name from silently becoming a newly created, unprotected deployment environment.

## Required caller permissions

Use:

```yaml
permissions:
  actions: read
  contents: read
```

`actions: read` is required for the Environment existence check. `contents: read` is required to check out the consumer repository.

Universal CD itself does not require GitHub write permission.

## Recommended manual CD caller

Start with `workflow_dispatch` and dry-run enabled by default:

```yaml
name: CD

on:
  workflow_dispatch:
    inputs:
      dry_run:
        description: Validate only
        required: true
        type: boolean
        default: true

permissions:
  actions: read
  contents: read

jobs:
  deploy:
    uses: morexyz/github-workflows/.github/workflows/universal-cd.yml@cd-v1
    with:
      enabled: true
      dry_run: ${{ inputs.dry_run }}
      environment_name: production
      allowed_environments: production
```

With the default manual input, deployment hooks do not execute. A user must explicitly disable dry-run for a real deployment.

After a project is proven safe, the consumer may choose a protected branch, release, or another explicit trigger.

## Required deployment hook

Default required hook:

```text
scripts/deploy.sh
```

Example skeleton:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Put this project's deterministic deployment logic here.
```

When CD is enabled, a missing deploy hook fails validation.

## Optional hooks

```text
scripts/predeploy.sh
scripts/healthcheck.sh
```

Real-deployment order:

```text
predeploy.sh (if present)
        ↓
deploy.sh
        ↓
healthcheck.sh (if present)
```

Any failing hook fails the workflow.

## Working directory and custom hook paths

Example:

```yaml
with:
  working_directory: apps/backend
  predeploy_script: scripts/predeploy.sh
  deploy_script: scripts/deploy.sh
  healthcheck_script: scripts/healthcheck.sh
```

The hooks above resolve under `apps/backend/`.

Absolute hook paths and `..` traversal segments are rejected. Hook invocation also terminates Bash options before the path so a script name cannot be interpreted as a Bash command-line option.

## GitHub Environments

Create the deployment Environment in the **consumer repository before real deployment**.

Example:

```yaml
with:
  environment_name: production
  allowed_environments: production
```

For several valid environments:

```yaml
with:
  environment_name: staging
  allowed_environments: staging,production
```

For real deployment, the selected name must:

1. appear in `allowed_environments`, and
2. already exist as a GitHub Environment in the caller repository.

Configure protection rules appropriate for each consumer repository, such as required reviewers where supported by the account/repository plan.

## Deployment variables

Repository hooks receive these conventional non-secret values:

```text
DEPLOY_HOST
DEPLOY_USER
DEPLOY_PATH
DEPLOY_URL
DEPLOY_ENVIRONMENT
```

The first four can be supplied directly:

```yaml
with:
  deploy_host: example.internal
  deploy_user: deploy
  deploy_path: /srv/example
  deploy_url: https://example.com
```

If those inputs are empty, Universal CD falls back to same-named GitHub `vars` values for `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_PATH`, and `DEPLOY_URL`.

Do not store credentials in these non-secret values.

## Deployment secrets

Supported optional secret names:

```text
DEPLOY_TOKEN
DEPLOY_PASSWORD
DEPLOY_SSH_KEY
DEPLOY_CREDENTIALS
```

### Repository-level secrets

Pass repository-level secrets explicitly:

```yaml
jobs:
  deploy:
    uses: morexyz/github-workflows/.github/workflows/universal-cd.yml@cd-v1
    with:
      enabled: true
      dry_run: false
      environment_name: production
      allowed_environments: production
    secrets:
      DEPLOY_SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
```

Pass only what the project actually needs.

### Environment-level secrets

GitHub treats Environment secrets specially with reusable workflows. Universal CD attaches the selected Environment to its internal real-deployment job.

If you use Environment-level secrets, define the supported secret names directly on the selected Environment in the consumer repository. Those Environment-scoped values are available to the job that targets that Environment. If a same-named Environment secret and a passed secret both exist, the Environment-scoped value takes precedence for that job.

This keeps production credentials bound to the protected production Environment.

## CD inputs

| Input | Default | Purpose |
|---|---|---|
| `enabled` | `false` | Explicit CD opt-in |
| `dry_run` | `true` | Validate without executing deployment hooks |
| `environment_name` | `production` | GitHub Environment for real deployment |
| `allowed_environments` | `production` | Comma-separated environment allowlist |
| `shared_target` | `false` | Declares that several repositories may deploy to the same external target |
| `external_lock_confirmed` | `false` | Confirms the project deploy hook enforces an external cross-repository lock |
| `working_directory` | `.` | Directory containing hook paths |
| `predeploy_script` | `scripts/predeploy.sh` | Optional predeploy hook |
| `deploy_script` | `scripts/deploy.sh` | Required deploy hook |
| `healthcheck_script` | `scripts/healthcheck.sh` | Optional health check |
| `deploy_host` | empty | Optional non-secret host |
| `deploy_user` | empty | Optional non-secret user |
| `deploy_path` | empty | Optional non-secret path |
| `deploy_url` | empty | Optional non-secret deployment URL |

## Shared external targets

GitHub Actions concurrency groups are repository-scoped. Two different repositories can therefore deploy concurrently even if they target the same host or cluster.

If several repositories can deploy to one external destination:

```yaml
with:
  shared_target: true
  external_lock_confirmed: true
```

`external_lock_confirmed: true` is a declaration that **the consumer repository's deployment implementation really enforces an external lock**. Universal CD does not create a cross-repository lock itself.

`shared_target: true` without this confirmation fails validation.

## CD output

Universal CD exposes one reusable-workflow output:

```text
deployed
```

It is `true` only when the real deploy job completes successfully. Disabled and dry-run executions return `false`.

## Example consumer layout

```text
my-project/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── cd.yml
├── scripts/
│   ├── deploy.sh
│   └── healthcheck.sh
└── application files...
```

## Production recommendations

1. Create a dedicated `production` GitHub Environment before enabling real CD.
2. Configure required reviewers/protection rules where supported.
3. Keep production credentials in GitHub Secrets/Environments, never repository files.
4. Begin with manual `workflow_dispatch` and `dry_run: true` by default.
5. Make `scripts/deploy.sh` deterministic and preferably idempotent.
6. Add a meaningful `scripts/healthcheck.sh`.
7. Do not let an AI agent or application bypass the protected GitHub deployment path for production.
8. Pin reusable workflows to an immutable release tag or exact commit SHA when supply-chain immutability is required.

## Smart contracts / blockchain

Universal CD does not automatically deploy smart contracts and must never infer or automatically perform a blockchain mainnet deployment.

Keep blockchain deployment in an explicitly designed consumer hook or a specialized future adapter with network, signer, approval, and confirmation safeguards.

---

# Public repository usage

This repository is public. Public and private consumer repositories can reference the reusable workflows directly, subject to the consumer repository's GitHub Actions policy.

Every consumer still needs its own small `.github/workflows/*.yml` caller. Reusable workflows are not automatically injected into projects.

If a repository's Actions policy restricts external or reusable workflows, allow this repository or use an immutable release tag or exact commit SHA according to that repository's security policy.

# Permissions model

Universal CI:

```yaml
permissions:
  contents: read
```

Universal CD:

```yaml
permissions:
  actions: read
  contents: read
```

The reusable workflows do not grant themselves write access to repository contents.

# Validation status

The workflows have been exercised cross-repository using `morexyz/simple-project` as a disposable consumer.

Universal CI validation covered shared-workflow resolution, root project detection, Node setup/install flow, and successful workflow completion.

Universal CD validation covered:

- Codex review and fixes for Environment-name safety, cross-repository shared-target concurrency semantics, and Bash hook invocation safety
- dry-run behavior with a sentinel deployment hook that would fail if accidentally executed
- harmless real execution in a non-production `cd-test` Environment
- predeploy/deploy/healthcheck sequencing
- explicit verification that a deliberately nonexistent Environment fails validation before the deploy job can start
- verification using the caller repository and caller token context
- direct consumer validation through the released `@cd-v1` reference after the Environment guard hardening

Repository-level validation also includes the required aggregate `Self Validation` check on changes targeting `main`.

The dependency-pinning hardening was validated both inside this repository and through a disposable cross-repository consumer. Stable refs `@v1` and `@cd-v1` now point to hardened commit `26eb1c1b82dbe666e6c3ef77b3ce72d79941ac62`.

Current releases `ci-v1.0.2` and `cd-v1.0.2` were published after GitHub Release Immutability was enabled, were verified with `immutable: true`, and resolve to the same hardened commit. Their tags are additionally protected against update/deletion by a no-bypass tag ruleset, while creation is governed separately.

No real production server, cloud deployment destination, or blockchain network was used during these tests.

# Known v1 boundaries

Universal CI v1 does not yet include automatic recursive monorepo discovery or built-in Java/Gradle/Maven, .NET, Ruby, or generic Docker-project detection. Use `scripts/ci.sh` for unsupported or specialized CI requirements.

Universal CD v1 intentionally contains no platform-specific deployment adapter. Use repository-owned hooks today; SSH/VPS, Docker/container, static-hosting, cloud, or blockchain adapters can be added separately when real project requirements justify them.

# Contributing / changing shared workflows

Shared-workflow changes can affect many repositories. Treat them as infrastructure changes:

1. Work on a feature/fix branch.
2. Inspect the exact workflow diff.
3. Review reusable-workflow syntax, permissions, inputs, secrets, expressions, shell safety, and dependency pins.
4. Validate with a disposable consumer repository.
5. Exercise negative safety tests as well as success paths.
6. Test dry-run before any real deployment path.
7. Merge only after validation succeeds.
8. Advance stable version refs deliberately with fast-forward only.
9. Publish a new immutable release tag for a versioned release; never move, reuse, or delete an existing fixed release tag.
10. Keep the release-tag creation bypass isolated from the no-bypass update/delete ruleset.

Do not use a production repository as the first consumer test for a shared workflow change.

# Security model summary

- CI never deploys.
- CD is disabled by default.
- Enabled CD is dry-run by default.
- Real deployment requires explicit `enabled: true` and `dry_run: false`.
- Real deployment requires an allowlisted Environment name.
- The selected Environment must already exist before the deploy job starts.
- Deployment hooks are consumer-owned and path validated.
- Shared external targets require explicit external-lock confirmation.
- GitHub token permissions remain read-only.
- Deployment credentials are scoped through GitHub secrets/environments rather than repository files.
- Production and blockchain mainnet deployment are never inferred automatically.
- Changes targeting `main` must pass the aggregate `Self Validation` gate and be current with `main` before merge.
- External GitHub Actions are pinned to full commit SHAs.
- The Actionlint archive used by self-validation is verified against a fixed SHA-256 digest.
- Fixed `ci-v*` and `cd-v*` release tags are protected against update and deletion with no bypass actors.
- Creation of matching release tags is separately restricted to the configured repository-administrator bypass role.
- GitHub Release Immutability is enabled for new releases; the current `v1.0.2` releases are verified immutable.

# License

This project is licensed under the MIT License. See [`LICENSE`](LICENSE) for the full text.