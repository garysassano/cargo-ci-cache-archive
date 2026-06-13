---
name: cargo-ci-cache
description: Use when choosing, explaining, updating, or debugging Rust/Cargo CI cache strategies, GitHub Actions Cargo cache workflows, Swatinem/rust-cache behavior, source-mtime checkout issues, filesystem snapshots, S3 Files experiments, or Cargo fingerprint rebuild diagnostics.
---

# Cargo CI Cache

Use this skill to apply the cache research archived in this repository to Rust/Cargo CI workflows.

## Repository References

This is a repository-scoped skill. Resolve the linked references relative to this
`SKILL.md`; do not assume `docs/` or `examples/` exists inside the skill directory.

Read only the references needed for the task:

| Task | Read |
| --- | --- |
| Choose or compare approaches | [Approach comparison](../../../docs/approaches/README.md) |
| Explain Cargo freshness | [Cargo freshness model](../../../docs/concepts/cargo-freshness-model.md) |
| Map state paths to cache coverage | [Cargo path coverage](../../../docs/concepts/cargo-path-coverage.md) |
| Diagnose rebuilds | [Diagnosing rebuilds](../../../docs/operations/diagnosing-rebuilds.md) |
| Review measured evidence | [Empirical results](../../../docs/results/empirical-results.md) |
| Copy workflow shapes | [Examples](../../../examples/README.md) |
| Refresh recommendations or versions | [Maintenance checklist](../../../docs/operations/maintenance-checklist.md) |

## Application Workflow

When applying this skill to another repository:

1. Inspect its Cargo workspace, `.cargo/config*`, toolchain files, target
   directory settings, and existing CI workflow before recommending changes.
2. Identify whether the goal is dependency reuse, repeated-run Cargo no-op
   behavior, rebuild diagnosis, or infrastructure-level snapshot fidelity.
3. Read only the archive references that match that goal.
4. Treat measured results as evidence from the recorded experiments, not as
   universal timing guarantees.
5. Verify current upstream action inputs and service behavior before changing
   versions or relying on behavior that may have changed.
6. Adapt generic paths, package selections, credentials, and runner assumptions
   to the target repository.
7. Run the target repository's existing validation plus `actionlint` and YAML
   parsing for edited workflows.

## Core Decision

Default to `Swatinem/rust-cache` plus an mtime-preserving cached worktree checkout.

Use alternatives only when the workload justifies them:

| Need | Recommendation |
| --- | --- |
| Maintained, simple, fast repeated-run CI | `Swatinem/rust-cache` plus mtime-preserving checkout |
| True no-op for repeated generated-code/build-script outliers | Source-keyed full target cache restored after `rust-cache` |
| Maximum local no-op fidelity and acceptable infra overhead | Filesystem snapshot / EBS snapshot layout |
| Shared filesystem for Cargo target no-op state | Do not use S3 Files based on these experiments |

## Diagnostic Workflow

When a cached Cargo build still recompiles:

1. Enable `CARGO_LOG=cargo::core::compiler::fingerprint=trace`.
2. Count `Compiling`, `Fresh`, `dirty`, `fingerprint`, and `stale` lines.
3. Confirm unchanged source files keep old mtimes.
4. Confirm workspace path is stable.
5. Confirm `target/**/.fingerprint/**`, `target/**/deps/*.d`, and `target/**/build/**` exist after restore.
6. Check whether a cache exact hit prevents saving newly rebuilt target state.
7. For build scripts, check `cargo:rerun-if-changed` hints, but do not assume hints fix stale target-cache restores.

Use [diagnosing rebuilds](../../../docs/operations/diagnosing-rebuilds.md)
and the
[diagnostic workflow](../../../examples/workflows/cargo-fingerprint-diagnostics.yml)
for concrete commands.

## Source-Keyed Target Cache Rules

Use this only when `rust-cache` target caching repeatedly restores stale workspace artifacts.

Required order:

```text
restore cached source worktree
setup registry credentials and toolchain
restore rust-cache with cache-targets: false
compute source key
restore full target directory with actions/cache
build with explicit CARGO_TARGET_DIR
```

Do not restore the full target directory before `rust-cache`; `rust-cache` cleanup can remove workspace artifacts.

Fast source key shape:

```bash
hash="$({ git rev-parse HEAD:app; git ls-files -s app; } | sha256sum | cut -d ' ' -f1)"
```

A dependency-closure key using `cargo metadata` can be more precise, but it was too expensive in the recorded CI tests.

## Filesystem Snapshot Rules

Use snapshots only when operational overhead is acceptable.

Snapshot-friendly layout:

```text
SNAPSHOT_ROOT=/mnt/build-snapshot
SNAPSHOT_WORKSPACE=/mnt/build-snapshot/workspace
CARGO_HOME=/mnt/build-snapshot/cargo-home
CARGO_TARGET_DIR=/mnt/build-snapshot/workspace/app/target
XDG_CACHE_HOME=/mnt/build-snapshot/xdg-cache
```

Do not put unrelated setup-action caches or large toolchain downloads under the snapshot root unless deliberately snapshotting them. Scrub credential-bearing files before snapshot save.

## S3 Files Rule

S3 Files can present S3 buckets as shared file systems, but this archive rejected it for Cargo target no-op state. Cargo can become logically clean on S3 Files while still spending time traversing many small metadata, fingerprint, dep-info, and build-script files remotely.

Use the [S3 Files experiment record](../../../docs/approaches/s3-files.md)
only as background for non-Cargo shared filesystem workloads.

## Updating This Archive

When editing this repository:

- Keep conclusions in [the approach comparison](../../../docs/approaches/README.md).
- Keep empirical numbers in [empirical results](../../../docs/results/empirical-results.md).
- Keep chronological history in [the experiment log](../../../docs/results/experiment-log.md).
- Keep procedures in [`docs/operations/`](../../../docs/operations/README.md).
- Keep examples generic under [`examples/`](../../../examples/README.md).
- Run `git diff --check`, `actionlint examples/workflows/*.yml`, and YAML parsing after workflow edits.
