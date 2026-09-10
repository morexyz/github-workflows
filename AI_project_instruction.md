# AI Project Instruction

## Purpose

This repository is the public central home for reusable GitHub Actions workflows maintained by `morexyz` and usable by external GitHub repositories.

## Architecture

- Reusable workflows live under `.github/workflows/`.
- Consumer repositories keep only small caller workflows.
- CI is centralized here; project-specific behavior should stay in the consumer repository.
- A consumer repository can provide `scripts/ci.sh` to override generic auto-detection safely.
- CD is separate from CI and must always be explicitly enabled by the caller.
- Universal CD v1 is hook-based: deployment implementation stays in the consumer repository.

## Stable references

- Universal CI v1: `morexyz/github-workflows/.github/workflows/universal-ci.yml@v1`
- Universal CD v1: `morexyz/github-workflows/.github/workflows/universal-cd.yml@cd-v1`

Keep CI and CD release refs independent so updating one does not silently change the other.

These stable references are maintained branches, not immutable tags. Advance them only for backward-compatible, reviewed, validated fixes. Consumers requiring immutable supply-chain pinning should use exact commit SHAs.

## Universal CI v1

Supported automatic root-level detection:

- Node.js / JavaScript / TypeScript: `package.json`
  - npm, pnpm, Yarn, and Bun are supported.
  - Runs declared scripts among `lint`, `typecheck`, `test`, and `build`.
- PHP: `composer.json`
  - Validates Composer metadata, installs dependencies, lints PHP, and runs declared Composer `lint`, `test`, and `build` scripts.
- Python: `pyproject.toml`, `requirements.txt`, `requirements-dev.txt`, `setup.py`, or `setup.cfg`
  - Supports Python 3.8+.
  - Installs dependencies/project where safely inferable, compiles sources, and runs Ruff/Pytest when available.
- Go: `go.mod`
  - Runs `go vet ./...` and `go test ./...`.
- Rust: `Cargo.toml`
  - Runs formatting, `cargo check`, and tests.
- Solidity / Foundry: `foundry.toml`
  - Runs `forge fmt --check`, `forge build`, and `forge test`.

Detection is root-level in v1. Use `scripts/ci.sh` for monorepos, unsupported stacks, or specialized validation flows.

## Universal CD v1

Universal CD is intentionally generic and does not guess how to deploy a project.

- `enabled` defaults to `false`.
- `dry_run` defaults to `true`.
- A real deployment requires both `enabled: true` and `dry_run: false`.
- Real deployment runs inside the caller-selected GitHub Environment, default `production`.
- `allowed_environments` restricts valid Environment names and defaults to `production`.
- Before any real deployment job starts, validation calls the GitHub Environment API in the caller repository and requires the selected Environment to already exist.
- Environment verification uses caller context and caller `GITHUB_TOKEN` with `actions: read`; it fails closed on missing Environment or insufficient permission.
- Universal CD callers need `actions: read` and `contents: read`.
- The caller repository owns deployment logic through `scripts/deploy.sh` by default.
- Optional hooks are `scripts/predeploy.sh` and `scripts/healthcheck.sh`.
- `working_directory` supports repositories where hooks live below the root.
- Standard non-secret values are `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_PATH`, and `DEPLOY_URL`.
- Optional secrets are `DEPLOY_TOKEN`, `DEPLOY_PASSWORD`, `DEPLOY_SSH_KEY`, and `DEPLOY_CREDENTIALS`.
- Hook paths must be repository-relative and cannot contain `..` path segments.
- Hook execution uses `bash --` so hook names cannot be interpreted as Bash options.
- A missing required deploy hook fails validation when CD is enabled.
- Disabled and dry-run modes never execute deployment hooks.
- If multiple repositories share one external deployment target, the caller must set `shared_target: true` and confirm that the deployment implementation enforces an external lock via `external_lock_confirmed: true`.

Deployment adapters such as SSH/VPS, Docker, static hosting, cloud platforms, and smart-contract deployment should remain explicit rather than being inferred automatically. Smart-contract mainnet deployment must never be automatic.

## Public-use rules

- The repository is public and licensed under MIT.
- Public and private consumer repositories may reference the shared workflows directly, subject to their own GitHub Actions policy.
- Every consumer repository still needs its own small `.github/workflows/*.yml` caller.
- Repository or organization policies may restrict reusable/external workflows; those policies are the consumer's responsibility.

## Safety / Things Not To Change Casually

- Do not turn CI into automatic production deployment.
- Do not grant GitHub write permissions to Universal CI or Universal CD without a documented need.
- Do not remove `actions: read` from CD callers that may perform real deployment.
- Do not blindly execute arbitrary repository scripts beyond documented CI/CD hooks and known validation script names.
- Do not force one Node/Python/PHP/etc. version across all repositories without keeping it configurable.
- Do not make absence of a supported CI project type fail by default.
- Do not change Universal CD defaults so a caller can deploy without explicit opt-in.
- Do not bypass GitHub Environment existence checks or protection rules for production.
- Do not add automatic blockchain mainnet deployment to Universal CD.
- Do not rely on repository-scoped GitHub concurrency to coordinate different repositories that deploy to one shared external target.

## Validation completed

1. Universal CI v1 was Codex-reviewed and validated cross-repository from `morexyz/simple-project`.
2. Stable CI ref `@v1` was created and tested.
3. Universal CD v1 was reviewed and fixed for Environment-name safety, cross-repository target locking semantics, and Bash hook invocation safety.
4. Dry-run behavior was validated with a sentinel deploy hook that would fail if executed.
5. A harmless real deployment was validated in a non-production `cd-test` Environment with predeploy/deploy/healthcheck sequencing.
6. A stricter Environment-existence guard was added after final security review showed that an allowlist alone could not prove Environment configuration.
7. The guard was tested negatively with a nonexistent Environment and positively with the existing `cd-test` Environment.
8. Stable CD ref `@cd-v1` was advanced to the hardened release and validated directly from the consumer repository.
9. README usage documentation and MIT License were added.
10. The repository was made public for direct external reuse.

## Next steps

1. Protect `main`, `v1`, and `cd-v1` with suitable branch/ruleset controls when administration tooling is available.
2. Add deployment adapters only when real project requirements justify them.
3. Revisit recursive monorepo CI discovery and additional ecosystems only when concrete repositories need them.
4. Keep README and internal instructions synchronized with stable workflow contracts.
