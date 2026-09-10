# Branch Protection and Self Validation

This repository protects shared workflow infrastructure with repository rulesets, pinned workflow dependencies, and an internal self-validation workflow.

## Protected refs

### `main`

The active `Protect main` ruleset requires:

- changes to reach `main` through a pull request;
- linear history;
- deletion protection;
- non-fast-forward / force-push protection;
- the `Self Validation` status check to pass;
- the pull request branch to be up to date with `main` before merge.

There are currently no required approving reviews. The required validation gate is automated through `Self Validation`.

### `v1` and `cd-v1`

The active `Protect stable workflow refs` ruleset protects both maintained stable workflow branches against deletion, force-pushes, and non-linear history.

These refs are intentionally not configured to require pull requests because they are moving major-version pointers that may need deliberate fast-forward advancement after a compatible release is reviewed and validated.

Never force-update these refs. Advance them only with a fast-forward to a reviewed, validated backward-compatible commit.

### `ci-v*` and `cd-v*`

The active `Protect immutable releases` tag ruleset protects fixed workflow release tags against update and deletion after creation.

Published fixed release tags must never be moved or reused. Publish a new versioned tag for every immutable workflow release.

Current fixed releases:

- `ci-v1.0.0` → `5695f6e2ea04c6d6eb7cd5aa45d33effc9f370f3`
- `cd-v1.0.0` → `84216d819ac00d1096db2b2ae343769ede0f8ef3`

## Workflow dependency pinning

External GitHub Actions used by this repository are pinned to full commit SHAs. A human-readable major or channel name is kept in an inline comment, for example:

```yaml
uses: actions/checkout@fbc6f3992d24b796d5a048ff273f7fcc4a7b6c09 # v5
```

Do not replace a full SHA with a movable branch or tag reference. To upgrade an Action, resolve and review the intended upstream ref, replace the SHA deliberately, and validate the change through a pull request.

Downloaded tooling used by self-validation must also be integrity-pinned. The Actionlint binary is verified against a repository-owned fixed SHA-256 value before execution.

## Self Validation

`.github/workflows/self-validation.yml` runs on pull requests and pushes to `main`.

It validates the repository through three independent checks:

1. `Actionlint` validates GitHub Actions workflow syntax and semantics.
2. `Universal CI smoke test` calls the repository's current `universal-ci.yml` revision.
3. `Universal CD disabled smoke test` calls the current `universal-cd.yml` revision with deployment disabled and dry-run enabled.

A final aggregate job named `Self Validation` fails unless all three validations succeed. The `Protect main` ruleset requires this aggregate status check.

## Release-ref rule

Normal consumers may use the moving compatibility refs:

- CI: `@v1`
- CD: `@cd-v1`

Consumers that want a fixed published release may use:

- CI: `@ci-v1.0.0`
- CD: `@cd-v1.0.0`

For maximum explicit supply-chain pinning, consumers may reference the exact workflow commit SHA directly.

A documentation-only change to `main` does not justify advancing either stable ref. Stable refs should advance only when the corresponding reusable workflow release itself needs a backward-compatible update.
