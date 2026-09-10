# Branch Protection and Self Validation

This repository protects shared workflow infrastructure with repository rulesets, pinned workflow dependencies, GitHub Release Immutability, and an internal self-validation workflow.

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

Two separate tag rulesets protect release tags so creation authority is isolated from immutability protections.

`Protect immutable releases`:

- targets `refs/tags/ci-v*` and `refs/tags/cd-v*`;
- blocks updates;
- blocks deletions;
- has no bypass actors.

`Restrict release tag creation`:

- targets the same release-tag patterns;
- restricts creation;
- grants the configured repository-administrator role an `always` bypass for creation only.

Keeping creation in a separate ruleset is deliberate. The administrator bypass must not be added to `Protect immutable releases`, because published release tags must remain non-bypassable for update and deletion.

Published fixed release tags must never be moved, reused, or deleted. Publish a new versioned tag for every release.

Current immutable releases:

- `ci-v1.0.2` → `26eb1c1b82dbe666e6c3ef77b3ce72d79941ac62`
- `cd-v1.0.2` → `26eb1c1b82dbe666e6c3ef77b3ce72d79941ac62`

GitHub Release Immutability is enabled for newly published releases. Both current `v1.0.2` releases were verified through the GitHub API with `immutable: true`.

Earlier `v1.0.0` and `v1.0.1` releases predate GitHub Release Immutability enablement. Their release tags remain protected by the repository tag ruleset, but the GitHub Release API reports those earlier release objects as non-immutable.

## Workflow dependency pinning

External GitHub Actions used by this repository are pinned to full commit SHAs. A human-readable major or channel name is kept in an inline comment, for example:

```yaml
uses: actions/checkout@fbc6f3992d24b796d5a048ff273f7fcc4a7b6c09 # v5
```

Do not replace a full SHA with a movable branch or tag reference. To upgrade an Action, resolve and review the intended upstream ref, replace the SHA deliberately, and validate the change through a pull request.

Downloaded tooling used by self-validation must also be integrity-pinned. The Actionlint Linux amd64 archive is verified against a repository-owned fixed SHA-256 value before execution.

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

Consumers that want the current fixed GitHub-immutable releases should use:

- CI: `@ci-v1.0.2`
- CD: `@cd-v1.0.2`

Both current immutable tags resolve to:

```text
26eb1c1b82dbe666e6c3ef77b3ce72d79941ac62
```

For maximum explicit supply-chain pinning, consumers may reference that exact workflow commit SHA directly.

A documentation-only change to `main` does not justify advancing either stable ref. Stable refs should advance only when the corresponding reusable workflow release itself needs a backward-compatible update.
