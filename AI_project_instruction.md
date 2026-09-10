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
- Real deployment must use a named GitHub Environment that already exists in the caller repository and should use environment protection rules where appropriate.
- Universal CD requires caller permissions `actions: read` and `contents: read`; no GitHub write permission is required by the shared workflow.
- Repository-level deployment secrets may be passed explicitly. Environment-level secrets should be defined on the selected consumer-repository Environment and are resolved by the job that targets that Environment.

## Stable References

- Universal CI v1: `morexyz/github-workflows/.github/workflows/universal-ci.yml@v1`
- Universal CD v1: `morexyz/github-workflows/.github/workflows/universal-cd.yml@cd-v1`

Keep CI and CD release refs independent so updating one does not silently change the other.

These stable references are currently maintained branches rather than immutable tags. They may move forward only for backwards-compatible, reviewed, validated fixes. Consumers that require immutable supply-chain pinning should use exact commit SHAs.

## Universal CI v1

Supported automatic detection:

- Node.js / JavaScript / TypeScript: `package.json`
  - npm, pnpm, yarn, and bun lockfiles/packageManager are recognized.
  - Runs declared scripts among: `lint`, `typecheck`, `test`, `build`.
- PHP: `composer.json`
  - Validates Composer metadata, installs dependencies, lints project PHP files, then runs declared `lint`, `test`, and `build` Composer scripts.
- Python: `pyproject.toml`, `requirements.txt`, `requirements-dev.txt`, `setup.py`, or `setup.cfg`
  - Supports Python 3.8 or newer.
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
- `allowed_environments` restricts which environment names may be used and defaults to `production`.
- Before any real deployment job starts, the validation job calls the GitHub Environment API in the caller repository and requires the selected Environment to already exist.
- Environment verification uses the caller repository context and caller `GITHUB_TOKEN` with `actions: read`; it fails closed on missing Environment or insufficient permission.
- The caller repository owns deployment logic through `scripts/deploy.sh` by default.
- Optional hooks are `scripts/predeploy.sh` and `scripts/healthcheck.sh`.
- `working_directory` supports repositories where deployment hooks live below the root.
- Standard non-secret deployment values are exposed as `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_PATH`, and `DEPLOY_URL`.
- Standard optional secrets are `DEPLOY_TOKEN`, `DEPLOY_PASSWORD`, `DEPLOY_SSH_KEY`, and `DEPLOY_CREDENTIALS`.
- Hook paths must be repository-relative and cannot contain `..` path segments.
- Hook execution uses `bash --` so hook names cannot be interpreted as Bash options.
- A missing required deploy hook fails validation when CD is enabled.
- Disabled and dry-run modes never execute deployment hooks.
- If multiple repositories share one external deployment target, the caller must explicitly set `shared_target: true` and confirm that the deployment hook provides an external lock via `external_lock_confirmed: true`.

Deployment adapters such as SSH/VPS, Docker, static hosting, cloud platforms, and smart-contract deployment should be added separately rather than making Universal CD infer a deployment strategy. Smart-contract mainnet deployment must never be automatic.

## Safety / Things Not To Change Casually

- Do not turn CI into automatic production deployment.
- Do not grant GitHub write permissions to Universal CI or Universal CD without a documented need.
- Do not remove `actions: read` from CD callers that may perform real deployment; environment existence verification depends on it.
- Do not blindly execute arbitrary repository scripts beyond the documented CI/CD hooks and known validation script names.
- Do not force one Node/Python/PHP/etc. version across all repositories without keeping it configurable.
- Do not make absence of a supported CI project type fail by default; repositories may contain docs or unsupported stacks.
- Do not change Universal CD defaults so that a caller can deploy without explicit opt-in.
- Do not bypass GitHub Environment existence checks or protection rules for production.
- Do not add automatic blockchain mainnet deployment to Universal CD.
- Do not rely on repository-scoped GitHub concurrency to coordinate different repositories that deploy to one shared external target.

## Completed Steps

1. Created central repository `morexyz/github-workflows`.
2. Implemented and Codex-reviewed Universal CI v1.
3. Enabled private-repository access for repositories owned by `morexyz`.
4. Validated Universal CI from `morexyz/simple-project` using a cross-repository reusable workflow call.
5. Merged Universal CI v1 to `main` and created stable `v1` ref.
6. Updated `morexyz/simple-project` to use `morexyz/github-workflows/.github/workflows/universal-ci.yml@v1`.
7. Implemented Universal CD v1 and addressed Codex findings covering environment allowlisting, cross-repository shared-target locking, and Bash hook invocation safety.
8. Codex re-review of Universal CD v1 found no major issues.
9. Verified cross-repository dry-run behavior in `morexyz/simple-project`; the real deploy job was skipped and the sentinel deploy hook was not executed.
10. Verified a harmless real deployment flow in the `cd-test` environment; predeploy, deploy, healthcheck, and summary all succeeded without any external target or credentials.
11. Merged Universal CD v1 to `main` and created stable `cd-v1` ref.
12. Closed the temporary Universal CD test PR in `morexyz/simple-project` without merging test-only hooks into `master`.
13. Added a stricter Environment-existence guard after a final security review identified that an allowlist alone could not prove the target Environment was already configured.
14. Verified the new guard with a negative consumer test where the same nonexistent Environment name was supplied in both `environment_name` and `allowed_environments`; validation failed and deploy was skipped.
15. Verified the positive path against the existing `cd-test` Environment; validation, deploy, healthcheck, and summary succeeded.

## Next Steps

1. Merge the Environment-existence hardening only after review is clean and advance `cd-v1` deliberately.
2. Maintain a complete public-facing README aligned with the actual stable workflow contracts.
3. Add deployment adapters only when a real repository needs them, such as SSH/VPS, Docker, static hosting, or a specific cloud platform.
4. Configure GitHub Environments and approval/protection rules in each consumer repository before enabling production deployment.
5. Keep smart-contract deployment separate; never infer or automatically enable blockchain mainnet deployment.
6. Revisit monorepo CI discovery and additional language ecosystems only when concrete repositories require them.
