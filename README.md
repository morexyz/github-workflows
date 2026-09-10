# Universal GitHub CI/CD Workflows

Reusable GitHub Actions workflows for consistent CI and opt-in CD across repositories.

This repository currently provides two stable reusable workflows:

- **Universal CI v1** — `morexyz/github-workflows/.github/workflows/universal-ci.yml@v1`
- **Universal CD v1** — `morexyz/github-workflows/.github/workflows/universal-cd.yml@cd-v1`

The design keeps consumer repositories small: each project owns its application code and any project-specific CI/CD hooks, while common orchestration lives here.

> **Access note:** this repository is currently private. Repositories owned by `morexyz` can use it when repository Actions access is enabled. Other users cannot directly call this private repository unless they are granted compatible access; they can copy/fork the workflows into a repository they control. If this repository is later made public, public direct reuse becomes possible.

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
 validate  GitHub Environment
             |
        predeploy hook
             |
          deploy hook
             |
        healthcheck hook
```

CI and CD are intentionally separate. Universal CI never deploys. Universal CD never runs a real deployment unless the caller explicitly opts in.

## Repository layout

```text
.github/workflows/
├── universal-ci.yml
└── universal-cd.yml

AI_project_instruction.md
README.md
```

## Stable references

Use the stable references in consumer repositories rather than `main`:

```text
CI: @v1
CD: @cd-v1
```

At the moment these are maintained Git branches rather than immutable Git tags. They are intended as stable major-version pointers. Pinning an exact commit SHA gives stronger immutability if your security policy requires it.

---

# Universal CI v1

Universal CI detects the project type from root-level files and runs conventional checks for supported ecosystems.

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
| Node.js / JS / TS | `package.json` | Install dependencies; run declared `lint`, `typecheck`, `test`, `build` scripts |
| PHP | `composer.json` | Composer validation/install, PHP lint, declared Composer `lint`, `test`, `build` scripts |
| Python | `pyproject.toml`, `requirements.txt`, `requirements-dev.txt`, `setup.py`, or `setup.cfg` | Dependency/project install where inferable, compile sources, run Ruff/Pytest when installed |
| Go | `go.mod` | `go vet ./...`, `go test ./...` |
| Rust | `Cargo.toml` | formatting, `cargo check`, tests |
| Solidity / Foundry | `foundry.toml` | `forge fmt --check`, `forge build`, `forge test` |

Detection in v1 is **root-level only**. Recursive monorepo discovery is not enabled.

## Custom CI hook

If a repository contains:

```text
scripts/ci.sh
```

that hook takes precedence over automatic ecosystem jobs.

Example:

```bash
#!/usr/bin/env bash
set -euo pipefail

npm ci
npm run lint
npm test
npm run build
```

The workflow invokes it with Bash, so executable permission is not required, although keeping shell scripts executable is still good repository hygiene.

Use a custom CI hook when a project is a monorepo, requires services/databases, has an unsupported stack, or needs a specific validation sequence.

## Node.js behavior

Universal CI recognizes npm, pnpm, Yarn, and Bun.

Package-manager selection is based on `package.json#packageManager` first, then lockfiles, with npm as the fallback. Lockfiles are respected where possible:

- npm: `npm ci` when `package-lock.json` or `npm-shrinkwrap.json` exists
- pnpm: `pnpm install --frozen-lockfile` when `pnpm-lock.yaml` exists
- Yarn: immutable/frozen-lockfile installation when `yarn.lock` exists
- Bun: frozen lockfile installation when `bun.lock` or `bun.lockb` exists

Only these declared package scripts are run automatically:

```text
lint
 typecheck
 test
 build
```

A missing script is skipped; it is not treated as a failure.

## CI inputs

You can override tool versions from the caller:

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

Example:

```yaml
jobs:
  ci:
    uses: morexyz/github-workflows/.github/workflows/universal-ci.yml@v1
    with:
      node-version: '20'
      python-version: '3.11'
```

## Multiple detected ecosystems

If more than one supported root-level ecosystem is detected, the corresponding jobs may run in parallel. A custom `scripts/ci.sh` disables the automatic ecosystem jobs and becomes authoritative.

## Unsupported repositories

If no supported root-level type is detected, Universal CI completes successfully instead of failing by default. For such projects, add `scripts/ci.sh`.

---

# Universal CD v1

Universal CD is intentionally deployment-platform-neutral. It does **not** guess whether a project uses SSH, Docker, Kubernetes, Vercel, a static host, a cloud provider, or a blockchain deployment.

The consumer repository owns deployment implementation through shell hooks.

## Safety defaults

Universal CD defaults to:

```text
enabled = false
dry_run = true
environment_name = production
allowed_environments = production
```

A real deployment requires both:

```yaml
enabled: true
dry_run: false
```

This two-switch model is deliberate.

## Recommended CD flow

Prefer a caller triggered by `workflow_dispatch` while initially adopting CD. After a project is proven safe, you can additionally trigger deployment on a protected branch, release, or other explicit event.

Example `.github/workflows/cd.yml`:

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

With the default manual input, this validates without deploying. A user must explicitly choose a non-dry run for the real deploy job to execute.

## Required deploy hook

For enabled CD, the default required hook is:

```text
scripts/deploy.sh
```

Example skeleton:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Implement the project's deterministic deployment here.
# Example: upload an artifact, run an SSH deployment command,
# publish a container, or invoke a provider CLI.
```

A missing deploy hook causes validation to fail when CD is enabled.

## Optional hooks

These hooks are optional:

```text
scripts/predeploy.sh
scripts/healthcheck.sh
```

Execution order for a real deployment is:

```text
predeploy.sh (if present)
        ↓
deploy.sh
        ↓
healthcheck.sh (if present)
```

Any failing hook fails the deployment workflow.

## Working directory

For projects whose deployment code is below the repository root:

```yaml
with:
  working_directory: apps/backend
```

Hook paths are relative to that working directory.

Example:

```yaml
with:
  working_directory: apps/backend
  deploy_script: scripts/deploy.sh
```

resolves to the equivalent repository location:

```text
apps/backend/scripts/deploy.sh
```

Absolute hook paths and `..` traversal segments are rejected.

## GitHub Environments

Real deployment jobs are attached to the caller-selected GitHub Environment:

```yaml
with:
  environment_name: production
  allowed_environments: production
```

Configure the corresponding Environment in the consumer repository and add protection rules appropriate for that repository, such as required reviewers where available.

The explicit `allowed_environments` check also reduces the risk of an accidental typo selecting a newly created, unprotected environment.

For multiple valid environments:

```yaml
with:
  environment_name: staging
  allowed_environments: staging,production
```

Keep production approval and secret policy in the consumer repository.

## Deployment variables

Universal CD exposes these conventional non-secret values to repository hooks:

```text
DEPLOY_HOST
DEPLOY_USER
DEPLOY_PATH
DEPLOY_URL
DEPLOY_ENVIRONMENT
```

The first four can be supplied as workflow inputs:

```yaml
with:
  deploy_host: example.internal
  deploy_user: deploy
  deploy_path: /srv/example
  deploy_url: https://example.com
```

If an input is empty, the reusable workflow falls back to same-named GitHub `vars` values for `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_PATH`, and `DEPLOY_URL`.

Do not put credentials in these non-secret inputs or variables.

## Deployment secrets

Supported optional secret channels are:

```text
DEPLOY_TOKEN
DEPLOY_PASSWORD
DEPLOY_SSH_KEY
DEPLOY_CREDENTIALS
```

Pass only the secrets a project actually needs, and map them explicitly from the consumer workflow. Do not depend on broad secret inheritance for this setup.

Example:

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

Repository hooks receive these values as environment variables only during the real deployment job.

## CD inputs

| Input | Default | Purpose |
|---|---|---|
| `enabled` | `false` | Explicit CD opt-in |
| `dry_run` | `true` | Validate without executing deployment hooks |
| `environment_name` | `production` | GitHub Environment for a real deploy |
| `allowed_environments` | `production` | Comma-separated environment allowlist |
| `shared_target` | `false` | Declares that multiple repositories may deploy to one external target |
| `external_lock_confirmed` | `false` | Confirms the deploy hook implements an external cross-repository lock |
| `working_directory` | `.` | Directory containing hook paths |
| `predeploy_script` | `scripts/predeploy.sh` | Optional predeploy hook |
| `deploy_script` | `scripts/deploy.sh` | Required deployment hook |
| `healthcheck_script` | `scripts/healthcheck.sh` | Optional health check |
| `deploy_host` | empty | Optional non-secret host |
| `deploy_user` | empty | Optional non-secret user |
| `deploy_path` | empty | Optional non-secret path |
| `deploy_url` | empty | Optional non-secret deployment URL |

## Shared deployment targets

GitHub Actions concurrency groups are repository-scoped. They do not serialize two different repositories that deploy to the same host/cluster.

If multiple repositories can target the same external deployment destination, declare it:

```yaml
with:
  shared_target: true
  external_lock_confirmed: true
```

Setting `external_lock_confirmed: true` is a declaration that **your repository-owned deployment hook actually implements an external lock**. Universal CD does not create that lock for you.

If `shared_target: true` is used without that confirmation, validation fails.

## CD output

Universal CD exposes:

```text
deployed
```

It is `true` only when the real deployment job completes successfully. Disabled and dry-run executions return `false`.

## Production recommendations

For production repositories:

1. Use a dedicated GitHub Environment such as `production`.
2. Add required reviewers/protection rules where supported by your GitHub plan and repository type.
3. Keep production secrets in GitHub Secrets/Environments, not in repository files.
4. Start with `workflow_dispatch` and dry-run enabled by default.
5. Make `scripts/deploy.sh` deterministic and idempotent when possible.
6. Add a meaningful `scripts/healthcheck.sh`.
7. Do not let an AI agent or application directly bypass the GitHub deployment workflow for production.
8. Pin the reusable workflow to an exact commit SHA when immutable supply-chain pinning is required.

## Smart contracts / blockchain

Universal CD does not automatically deploy smart contracts and must not automatically deploy to a blockchain mainnet.

Keep blockchain deployment in a dedicated project-owned hook or a future specialized adapter with explicit network, signer, approval, and confirmation safeguards.

---

# Private repository setup

This central repository is currently private. For another private repository owned by the same account to call these workflows, GitHub Actions access for the central repository must permit repositories owned by that account to use its reusable workflows.

In this setup, that access has been enabled for repositories owned by `morexyz`.

A consumer still needs its own small `.github/workflows/*.yml` caller file; reusable workflows are not automatically injected into repositories.

# Permissions

Both Universal CI and Universal CD currently declare:

```yaml
permissions:
  contents: read
```

They are intentionally read-only at the GitHub token level. A repository-owned deployment hook may still act on external systems using explicitly supplied credentials.

Do not add broad GitHub write permissions unless a specific future workflow requires them and the reason is documented.

# Versioning

Current stable pointers:

```text
v1     → Universal CI v1 line
cd-v1  → Universal CD v1 line
```

Consumer repositories should normally use these stable pointers.

For maximum reproducibility/security, pin to an exact commit SHA instead. When backwards-compatible fixes are released, the maintained major-version branch may be advanced deliberately after review and validation.

Breaking changes should use a new major-version reference rather than silently changing the v1 contract.

# Validation status

The current versions have been validated cross-repository from `morexyz/simple-project`.

Universal CI validation covered reusable-workflow resolution, root project detection, Node setup/install flow, and successful execution through the shared workflow.

Universal CD validation included:

- Codex review and fixes for environment-name safety, shared-target concurrency semantics, and Bash hook invocation safety
- a cross-repository dry-run where a sentinel deploy hook would fail if accidentally executed
- a harmless real execution in a non-production `cd-test` environment where predeploy, deploy, healthcheck, and summary all completed successfully

No external production server, cloud deployment target, or blockchain network was used for the real-execution validation.

# Known v1 boundaries

Universal CI v1 intentionally does not yet provide automatic recursive monorepo discovery or built-in Java/Gradle/Maven, .NET, Ruby, or generic Docker-project detection. Use `scripts/ci.sh` for unsupported or specialized CI requirements.

Universal CD v1 intentionally does not include platform-specific deployment adapters. Use repository-owned hooks; specialized SSH/VPS, Docker/container, static-hosting, cloud, or blockchain adapters can be added separately when needed.

# Example minimal consumer repository

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

Safe manual CD caller:

```yaml
name: CD

on:
  workflow_dispatch:
    inputs:
      dry_run:
        description: Validate only
        type: boolean
        required: true
        default: true

permissions:
  contents: read

jobs:
  deploy:
    uses: morexyz/github-workflows/.github/workflows/universal-cd.yml@cd-v1
    with:
      enabled: true
      dry_run: ${{ inputs.dry_run }}
      environment_name: production
      allowed_environments: production
    secrets:
      DEPLOY_SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
```

Then place the actual deployment behavior in `scripts/deploy.sh` and validation in `scripts/healthcheck.sh`.

# Contributing / changing the shared workflows

Changes to shared workflows affect many consumer repositories, so treat them as infrastructure changes:

1. Work on a feature branch.
2. Review the exact workflow diff.
3. Run a review focused on reusable-workflow syntax, permissions, inputs/secrets, and shell safety.
4. Validate through a disposable consumer repository.
5. Test dry-run paths before real deployment paths.
6. Merge only after successful validation.
7. Advance stable version references deliberately.

Do not use a production repository as the first test consumer for a shared workflow change.

# Security model summary

- CI never deploys.
- CD is disabled by default.
- Enabled CD is dry-run by default.
- Real deployment requires explicit `enabled: true` and `dry_run: false`.
- Real deployment requires an allowed GitHub Environment name.
- Deployment hooks are repository-owned and path validated.
- Shared external targets require an explicit external-lock confirmation.
- GitHub token permissions remain read-only.
- Deployment credentials must be supplied explicitly.
- Production/mainnet deployment is never inferred automatically.
