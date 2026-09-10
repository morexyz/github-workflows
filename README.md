# Universal GitHub CI/CD Workflows

Reusable GitHub Actions workflows for consistent CI and safe, opt-in CD across repositories.

This repository provides two shared workflows:

- **Universal CI v1** — `morexyz/github-workflows/.github/workflows/universal-ci.yml@v1`
- **Universal CD v1** — `morexyz/github-workflows/.github/workflows/universal-cd.yml@cd-v1`

The goal is simple: consumer repositories keep tiny caller workflows, while common orchestration is maintained here. Project-specific validation and deployment behavior remains in the consumer repository when needed.

> **Access note:** this repository is currently private. Repositories owned by `morexyz` can use it because private reusable-workflow access has been enabled. Other GitHub users cannot directly call this private repository unless compatible access is granted. They can copy or fork these workflows into a repository they control. If this repository is later made public, direct public reuse becomes possible.

## Architecture

```text
Developer / AI Agent
        |
      Branch
        |
        PR
        |
  Universal CI
        |
 lint / typecheck / test / build
        |
      Review
        |
      Merge
        |
 optional Universal CD
        |
      dry-run?
      /     \
    yes     no
     |       |
 validate   verify GitHub Environment exists
             |
        GitHub Environment
             |
        predeploy hook
             |
          deploy hook
             |
        healthcheck hook
```

CI and CD are intentionally separate. **Universal CI never deploys.** Universal CD never runs a real deployment unless the caller explicitly opts in.

## Repository layout

```text
.github/workflows/
├── universal-ci.yml
└── universal-cd.yml

AI_project_instruction.md
README.md
LICENSE
```

## Stable references

Consumer repositories should normally use:

```text
CI: @v1
CD: @cd-v1
```

These are currently maintained Git branches, not immutable Git tags. They act as stable major-version pointers and may be advanced deliberately for backwards-compatible fixes after review and validation.

For maximum immutability and supply-chain reproducibility, pin the reusable workflow to an exact commit SHA instead.

---

# Universal CI v1

Universal CI detects supported root-level project files and runs conventional validation automatically.

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
| Python | `pyproject.toml`, `requirements.txt`, `requirements-dev.txt`, `setup.py`, or `setup.cfg` | Installs dependencies/project where safely inferable, compiles Python, runs Ruff/Pytest when available |
| Go | `go.mod` | `go vet ./...`, `go test ./...` |
| Rust | `Cargo.toml` | formatting, `cargo check`, tests |
| Solidity / Foundry | `foundry.toml` | `forge fmt --check`, `forge build`, `forge test` |

Detection in v1 is **root-level only**. Recursive monorepo discovery is intentionally not enabled.

## Custom CI hook

If a repository contains:

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

Use a custom hook for monorepos, unsupported stacks, service/database setup, unusual commands, or projects that require a strict validation sequence.

The workflow invokes the hook with Bash, so executable permission is not required, although keeping shell scripts executable is good repository hygiene.

## Node.js behavior

Universal CI supports npm, pnpm, Yarn, and Bun.

Package-manager selection order:

1. `package.json#packageManager`
2. recognized lockfile
3. npm fallback

Lockfiles are respected where possible:

- npm: `npm ci` with `package-lock.json` or `npm-shrinkwrap.json`
- pnpm: frozen-lockfile installation with `pnpm-lock.yaml`
- Yarn: immutable/frozen-lockfile installation with `yarn.lock`
- Bun: frozen-lockfile installation with `bun.lock` or `bun.lockb`

Only these declared package scripts run automatically:

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

If several supported root-level ecosystems are detected, their jobs may run in parallel. If `scripts/ci.sh` exists, it becomes authoritative and automatic ecosystem jobs are skipped.

## Unsupported repositories

If no supported root-level project type is detected, Universal CI succeeds rather than failing by default. Add `scripts/ci.sh` to define CI for that project.

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

For a real deployment, Universal CD also verifies that the selected GitHub Environment already exists in the caller repository before the deploy job is allowed to start. This prevents a misspelled environment name from silently becoming a newly created, unprotected deployment environment.

## Required permissions for a CD caller

Use:

```yaml
permissions:
  actions: read
  contents: read
```

`actions: read` is required for the real-deployment Environment existence check. `contents: read` is required to check out the consumer repository.

No GitHub write permission is required by Universal CD itself.

A reusable workflow cannot elevate token permissions above what the caller grants, so do not remove `actions: read` from a caller that may perform real deployment.

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

With the default manual input, deployment hooks do not execute. A user must explicitly choose a non-dry run for the real deploy path.

After a project is proven safe, the consumer may choose a protected branch, release, or another explicit trigger.

## Required deployment hook

The default required hook is:

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

For real deployment, the selected name must both:

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

If those inputs are empty, the workflow falls back to same-named GitHub `vars` values for `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_PATH`, and `DEPLOY_URL`.

Do not store credentials in these non-secret inputs or variables.

## Deployment secrets

Supported optional secret names are:

```text
DEPLOY_TOKEN
DEPLOY_PASSWORD
DEPLOY_SSH_KEY
DEPLOY_CREDENTIALS
```

There are two relevant patterns.

### Repository-level secrets

A repository-level secret can be passed explicitly from the caller:

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

Pass only what the project actually needs. Broad secret inheritance is intentionally not required by this setup.

### Environment-level secrets

GitHub treats Environment secrets specially with reusable workflows. The caller job itself cannot attach an `environment` while it is a reusable-workflow call. Universal CD attaches the Environment to its internal real-deployment job.

Therefore, if you use Environment-level secrets, define the supported secret names directly on the selected consumer-repository Environment (for example `production`). GitHub makes those Environment secrets available to the job that targets that Environment. If a same-named Environment secret and a passed secret both exist, the Environment-scoped value takes precedence for that job.

This is useful for keeping production credentials bound to the protected production Environment.

## CD inputs

| Input | Default | Purpose |
|---|---|---|
| `enabled` | `false` | Explicit CD opt-in |
| `dry_run` | `true` | Validate without executing deployment hooks |
| `environment_name` | `production` | GitHub Environment for a real deployment |
| `allowed_environments` | `production` | Comma-separated environment allowlist |
| `shared_target` | `false` | Declares that several repositories may deploy to the same external target |
| `external_lock_confirmed` | `false` | Confirms the project deploy hook implements an external cross-repository lock |
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

If several repositories can target one external destination:

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

## Example production-oriented consumer layout

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

CI caller:

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

CD caller:

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

Then put actual deployment behavior in `scripts/deploy.sh` and a meaningful post-deployment check in `scripts/healthcheck.sh`.

## Production recommendations

1. Create a dedicated `production` GitHub Environment before enabling real CD.
2. Configure required reviewers/protection rules where supported.
3. Keep production credentials in GitHub Secrets/Environments, never repository files.
4. Begin with manual `workflow_dispatch` and `dry_run: true` by default.
5. Make `scripts/deploy.sh` deterministic and preferably idempotent.
6. Add a meaningful `scripts/healthcheck.sh`.
7. Do not let an AI agent or application bypass the protected GitHub deployment path for production.
8. Pin reusable workflows to exact commit SHAs when immutable supply-chain pinning is required.

## Smart contracts / blockchain

Universal CD does not automatically deploy smart contracts and must never infer or automatically perform a blockchain mainnet deployment.

Keep blockchain deployment in an explicitly designed consumer hook or a specialized future adapter with network, signer, approval, and confirmation safeguards.

---

# Private repository setup

This central repository is currently private.

For a compatible private consumer repository to call it, the central repository's GitHub Actions access policy must permit that consumer. In the current `morexyz` setup, access has been enabled for repositories owned by the same account.

Every consumer still needs its own small `.github/workflows/*.yml` caller. Reusable workflows are not automatically injected into projects.

A public caller cannot consume a reusable workflow stored only in a private repository. Users outside this private-repository access boundary should copy/fork the workflows into a repository they control, or use this repository directly if it becomes public later.

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

# Versioning and pinning

Current stable references:

```text
Universal CI v1: @v1
Universal CD v1: @cd-v1
```

These references are separate so CI and CD can evolve independently.

They are maintained branch references and therefore mutable. Use an exact commit SHA when your security policy requires immutable workflow code:

```yaml
uses: morexyz/github-workflows/.github/workflows/universal-ci.yml@<commit-sha>
```

When maintaining this repository, advance a stable reference only after the change has been reviewed and tested from a consumer repository.

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

No real production server, cloud deployment destination, or blockchain network was used during these tests.

# Known v1 boundaries

Universal CI v1 does not yet include automatic recursive monorepo discovery or built-in Java/Gradle/Maven, .NET, Ruby, or generic Docker-project detection. Use `scripts/ci.sh` for unsupported or specialized CI requirements.

Universal CD v1 intentionally contains no platform-specific deployment adapter. Use repository-owned hooks today; SSH/VPS, Docker/container, static-hosting, cloud, or blockchain adapters can be added separately when real project requirements justify them.

# Contributing / changing shared workflows

Shared-workflow changes can affect many repositories. Treat them as infrastructure changes:

1. Work on a feature/fix branch.
2. Inspect the exact workflow diff.
3. Review reusable-workflow syntax, permissions, inputs, secrets, expressions, and shell safety.
4. Validate with a disposable consumer repository.
5. Exercise negative safety tests as well as success paths.
6. Test dry-run before any real deployment path.
7. Merge only after validation succeeds.
8. Advance stable version refs deliberately.

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

# License

This project is licensed under the MIT License. See [`LICENSE`](LICENSE) for the full text.
