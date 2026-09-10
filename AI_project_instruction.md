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

Moving major-version references:

- Universal CI v1: `morexyz/github-workflows/.github/workflows/universal-ci.yml@v1`
- Universal CD v1: `morexyz/github-workflows/.github/workflows/universal-cd.yml@cd-v1`

Current immutable release references:

- Universal CI v1.0.2: `morexyz/github-workflows/.github/workflows/universal-ci.yml@ci-v1.0.2`
- Universal CD v1.0.2: `morexyz/github-workflows/.github/workflows/universal-cd.yml@cd-v1.0.2`

Keep CI and CD release refs independent so updating one does not silently change the other.

The moving stable references are maintained branches. Advance them only for backward-compatible, reviewed, validated fixes. The fixed `ci-v*` and `cd-v*` release tags are protected against update and deletion by the `Protect immutable releases` tag ruleset. Creation of matching release tags is separately restricted by `Restrict release tag creation` and allowed only to the configured bypass role. GitHub Release Immutability is enabled for new releases. Consumers requiring maximum explicit supply-chain pinning may still use exact commit SHAs.

Current immutable release mappings:

- `ci-v1.0.2` → `26eb1c1b82dbe666e6c3ef77b3ce72d79941ac62`
- `cd-v1.0.2` → `26eb1c1b82dbe666e6c3ef77b3ce72d79941ac62`

## Repository hardening

The repository now has active rulesets and internal validation:

- `main` is protected by the `Protect main` ruleset.
- Changes to `main` require a pull request, linear history, deletion protection, and non-fast-forward/force-push protection.
- `main` requires the aggregate `Self Validation` status check to pass.
- Required status checks use strict branch freshness, so a pull request must be up to date with `main` before merge.
- `v1` and `cd-v1` are protected by the `Protect stable workflow refs` ruleset against deletion, force-pushes, and non-linear history.
- Stable refs intentionally remain fast-forwardable so reviewed backward-compatible releases can advance them deliberately.
- `ci-v*` and `cd-v*` tags are protected by `Protect immutable releases` against update and deletion after creation, with no bypass actors.
- Creation of `ci-v*` and `cd-v*` tags is restricted by the separate `Restrict release tag creation` ruleset; repository administrators are the configured always-allow bypass role for creation only.
- GitHub Release Immutability is enabled for newly published releases; current `ci-v1.0.2` and `cd-v1.0.2` releases were verified with `immutable: true`.
- External GitHub Actions used by the workflows are pinned to full commit SHAs.
- The Actionlint archive used by self-validation is verified against a repository-owned fixed SHA-256 digest before execution.
- `.github/workflows/self-validation.yml` runs Actionlint plus safe smoke tests of Universal CI and Universal CD, then reports one aggregate `Self Validation` result.

See `docs/BRANCH_PROTECTION.md` for the maintained hardening model.

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
- Do not disable or bypass `Self Validation` for ordinary changes to `main`.
- Do not force-update `v1` or `cd-v1`; advance them only by deliberate fast-forward after compatible validation.
- Do not update or delete published `ci-v*` or `cd-v*` fixed release tags.
- Do not broaden the release-tag creation bypass beyond the dedicated creation ruleset without an explicit security review.
- Do not replace full-SHA Action pins with movable tags or branches.

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
11. `Self Validation` was added and successfully exercised with Actionlint, Universal CI smoke validation, Universal CD disabled smoke validation, and an aggregate result.
12. `Protect main` was activated with PR, linear-history, deletion, force-push protection, required `Self Validation`, and strict up-to-date-before-merge behavior.
13. `Protect stable workflow refs` was activated for `v1` and `cd-v1` with deletion, force-push, and linear-history protection while retaining deliberate fast-forward release updates.
14. `Protect immutable releases` was activated for `ci-v*` and `cd-v*` tags with update and deletion protection and no bypass actors.
15. External GitHub Actions were pinned to full commit SHAs, Actionlint integrity verification was hardened, and the changes passed repository and cross-repository validation.
16. Stable refs `@v1` and `@cd-v1` were fast-forwarded to hardened commit `26eb1c1b82dbe666e6c3ef77b3ce72d79941ac62`.
17. GitHub Release Immutability was enabled.
18. Immutable releases `ci-v1.0.2` and `cd-v1.0.2` were published and verified with `immutable: true` at hardened commit `26eb1c1b82dbe666e6c3ef77b3ce72d79941ac62`.
19. `Restrict release tag creation` was activated separately so creation is limited to the configured repository-administrator bypass while update/delete protection remains non-bypassable under the separate immutable-release ruleset.

## Next steps

1. Add deployment adapters only when real project requirements justify them.
2. Revisit recursive monorepo CI discovery and additional ecosystems only when concrete repositories need them.
3. Keep README, branch-protection documentation, and internal instructions synchronized with stable workflow contracts, repository rulesets, dependency pins, and immutable releases.
