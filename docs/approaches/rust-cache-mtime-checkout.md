# `Swatinem/rust-cache` With Mtime-Preserving Checkout

This is the recommended default approach.

## Related Files

| File | Purpose |
| --- | --- |
| [Workflow example](../../examples/workflows/rust-cache-mtime-checkout.yml) | End-to-end cached worktree plus `Swatinem/rust-cache` workflow. |
| [Cached worktree action](../../examples/actions/cached-worktree-checkout/action.yml) | Composite action that checks out into a restored worktree without rewriting unchanged file mtimes. |

## Design

```text
actions/cache restores cached worktree
custom checkout checks out source in place and preserves unchanged mtimes
Swatinem/rust-cache restores Cargo home and target state
Cargo builds with explicit CARGO_TARGET_DIR
```

## Why It Works

Normal checkout rewrites source mtimes. Cargo can treat rewritten source files as newer than restored target fingerprints, so local workspace crates rebuild even if contents are unchanged.

The cached worktree checkout avoids that false invalidation:

- If the worktree is already at `GITHUB_SHA`, it skips checkout entirely.
- If the worktree is older, Git checks out the new commit in place.
- Git rewrites changed files only, so unchanged files keep stable mtimes.

`Swatinem/rust-cache` then handles Cargo home and dependency-oriented target state.

## Recommended Settings

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    workspaces: ./app -> ../../target-for-job
    cache-targets: true
    cache-all-crates: true
    cache-workspace-crates: true
    shared-key: app-target-v1
```

Use a stable explicit target directory:

```yaml
env:
  CARGO_TARGET_DIR: /tmp/cargo-target-one-job
```

## Strengths

- Maintained upstream cache action.
- Minimal custom logic.
- Fixes the biggest false rebuild source: source mtime churn.
- Avoids network filesystem metadata latency.
- Good repeated-run performance for most jobs.

## Limitations

- `rust-cache` target keys intentionally do not include workspace source contents.
- Exact cache hits can restore stale workspace artifacts and then skip saving rebuilt target state.
- Build-script/generated-code workspace crates can rebuild repeatedly in some jobs.

## Related Alternatives And Upstream Work

These approaches address the same source-mtime class of problem, but they are not equivalent to the source-keyed target-cache workaround.

| Approach | What it does | Notes |
| --- | --- | --- |
| [`chetan/git-restore-mtime-action`](https://github.com/chetan/git-restore-mtime-action) | Rewrites checked-out file mtimes from Git history, usually requiring `fetch-depth: 0`. | Can reduce false rebuilds from checkout mtime churn, but uses synthetic commit-time mtimes rather than preserving the previous CI worktree's mtimes. |
| Retimer-style cached mtime state | Saves prior source mtimes and restores them before running Cargo. | Closer to the cached-worktree idea, but adds another state file/cache to maintain. A later comment on [`Swatinem/rust-cache#155`](https://github.com/Swatinem/rust-cache/issues/155) reported that this still left the final project rebuilding as `stale, unknown reason`. |
| Cargo checksum freshness | Upstream Cargo work in [`rust-lang/cargo#14137`](https://github.com/rust-lang/cargo/pull/14137), tracked under checksum freshness, uses checksums instead of mtimes for rebuild detection. | This is the cleanest long-term direction, but it is not the stable default documented by this archive and build-script coverage was still called out as incomplete in the upstream discussion. |

These alternatives mainly target source mtime churn. They do not fix `rust-cache` exact-hit behavior where stale workspace target state is restored and not saved again because the target cache key ignores workspace source contents.

## Observed Result

Repeated same-SHA workflows with this approach produced most matrix jobs around 33 to 37 seconds. A few jobs with generated-code/build-script dependency chains were slower, around 50 to 65 seconds.

## When To Use

Use this as the default for most Rust GitHub Actions CI workflows where:

- You want maintained upstream behavior.
- You can tolerate occasional generated-code/build-script outliers.
- You value simplicity over maximum theoretical Cargo no-op fidelity.
