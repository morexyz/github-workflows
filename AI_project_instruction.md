# AI Project Instruction

## Purpose

This repository is the central home for reusable GitHub Actions workflows shared by repositories owned by `morexyz`.

## Architecture

- Reusable workflows live under `.github/workflows/`.
- Consumer repositories keep only small caller workflows.
- CI is centralized here; project-specific behavior should stay in the consumer repository.
- A consumer repository can provide `scripts/ci.sh` to override generic auto-detection safely.
- CD is separate from CI and must always be explicitly enabled by the caller.
- Universal CD v1 is hook-based: deployment implementation stays in the consumer repository.

## Conventions

- Keep workflows conservative and deterministic.
- Never deploy from the Universal CI workflow.
- Do not assume one package manager, language, branch name, or deployment target for all repositories.
- Prefer lockfiles when installing dependencies.
- Run only checks that are clearly declared or conventional for the detected ecosystem.
- Do not require repository secrets for CI unless a future opt-in input explicitly enables that behavior.
- Keep permissions read-only unless a workflow has a documented reason to need more.
- Pin major versions of trusted GitHub Actions and keep third-party Actions minimal.
- A custom `scripts/ci.sh` is authoritative for repositories that need project-specific CI behavior.
- Production deployment must not be inferred from project type or repository contents.
- Real deployment must use a named GitHub Environment and should use environment protection rules where appropriate.
- Secrets must be passed explicitly to reusable CD workflows; do not assume cross-repository `secrets: inherit` for this personal-account repository layout.

## Universal CI v1

Supported automatic detection:

- Node.js / JavaScript / TypeScript: `package.json`
  - npm, pnpm, yarn, and bun lockfiles/packageManager are recognized.
  - Runs declared scripts among: `lint`, `typecheck`, `test`, `build`.
- PHP: `composer.json`
  - Validates Composer metadata, installs dependencies, lints project PHP files, then runs declared `lint`, `test`, and `build` Composer scripts.
- Python: `pyproject.toml`, `requirements.txt`, `requirements-dev.txt`, `setup.py`, or `setup.cfg`
  - Installs declared dependencies where safely inferable, compiles Python sources, and runs pytest/ruff only when available.
- Go: `go.mod`
  - Runs `go vet ./...` and `go test ./...`.
- Rust: `Cargo.toml`
  - Runs formatting check, `cargo check`, and tests.
- Solidity / Foundry: `foundry.toml`
  - Runs `forge fmt --check`, `forge build`, and `forge test`.

Detection is root-level in v1. Monorepo-specific recursive discovery is intentionally deferred until v1 is proven stable.

## Universal CD v1

Universal CD is intentionally generic and does not guess how to deploy a project.

- `enabled` defaults to `false`.
- `dry_run` defaults to `true`.
- A real deployment requires both `enabled: true` and `dry_run: false`.
- Real deployment runs inside the caller-selected GitHub Environment, default `production`.
- The caller repository owns deployment logic through `scripts/deploy.sh` by default.
- Optional hooks are `scripts/predeploy.sh` and `scripts/healthcheck.sh`.
- `working_directory` supports repositories where deployment hooks live below the root.
- Standard non-secret deployment values are exposed as `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_PATH`, and `DEPLOY_URL`.
- Standard optional secrets are `DEPLOY_TOKEN`, `DEPLOY_PASSWORD`, `DEPLOY_SSH_KEY`, and `DEPLOY_CREDENTIALS`.
- Hook paths must be repository-relative and cannot contain `..` path segments.
- A missing required deploy hook fails validation when CD is enabled.
- Disabled and dry-run modes never execute deployment hooks.

Deployment adapters such as SSH/VPS, Docker, static hosting, cloud platforms, and smart-contract deployment should be added separately rather than making Universal CD infer a deployment strategy. Smart-contract mainnet deployment must never be automatic.

## Safety / Things Not To Change Casually

- Do not turn CI into automatic production deployment.
- Do not grant `contents: write` to Universal CI or Universal CD without a documented need.
- Do not blindly execute arbitrary repository scripts beyond the documented CI/CD hooks and known validation script names.
- Do not force one Node/Python/PHP/etc. version across all repositories without keeping it configurable.
- Do not make absence of a supported CI project type fail by default; repositories may contain docs or unsupported stacks.
- Do not change Universal CD defaults so that a caller can deploy without explicit opt-in.
- Do not bypass GitHub Environment protection rules for production.
- Do not add automatic blockchain mainnet deployment to Universal CD.

## Completed Steps

1. Created central repository `morexyz/github-workflows`.
2. Implemented and Codex-reviewed Universal CI v1.
3. Enabled private-repository access for repositories owned by `morexyz`.
4. Validated Universal CI from `morexyz/simple-project` using a cross-repository reusable workflow call.
5. Merged Universal CI v1 to `main` and created stable `v1` ref.
6. Updated `morexyz/simple-project` to use `morexyz/github-workflows/.github/workflows/universal-ci.yml@v1`.
7. Started Universal CD v1 on branch `feat/universal-cd-v1`.

## Next Steps

1. Review Universal CD v1 for workflow syntax, permissions, environment/secrets behavior, and path safety.
2. Add a dry-run caller to a disposable/test repository and verify no deployment hook executes.
3. Add a harmless test deployment hook and verify explicit real-deployment gating in a non-production test environment.
4. Merge Universal CD only after review and successful caller tests.
5. Add deployment adapters separately as real project needs are identified.
