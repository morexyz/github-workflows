# AI Project Instruction

## Purpose

This repository is the central home for reusable GitHub Actions workflows shared by repositories owned by `morexyz`.

## Architecture

- Reusable workflows live under `.github/workflows/`.
- Consumer repositories keep only small caller workflows.
- CI is centralized here; project-specific behavior should stay in the consumer repository.
- A consumer repository can provide `scripts/ci.sh` to override generic auto-detection safely.
- CD will be added separately after Universal CI is proven on test repositories.

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

## Safety / Things Not To Change Casually

- Do not turn CI into automatic production deployment.
- Do not grant `contents: write` to Universal CI.
- Do not blindly execute arbitrary repository scripts beyond the documented custom hook and known validation script names.
- Do not force one Node/Python/PHP/etc. version across all repositories without keeping it configurable.
- Do not make absence of a supported project type fail by default; repositories may contain docs or unsupported stacks.

## Completed Steps

1. Created central repository `morexyz/github-workflows`.
2. Started Universal CI v1 on branch `feat/universal-ci-v1`.

## Next Steps

1. Add and review the reusable Universal CI workflow.
2. Merge Universal CI only after review.
3. Enable private-repository Actions access for this central repository if required by GitHub settings.
4. Add a minimal caller workflow to a disposable/test repository.
5. Verify actual GitHub Actions execution.
6. Only then design Universal CD as a separate opt-in workflow.
