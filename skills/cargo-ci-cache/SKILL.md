---
name: cargo-ci-cache
description: Use when choosing, explaining, updating, or debugging Rust/Cargo CI cache strategies, GitHub Actions Cargo cache workflows, Swatinem/rust-cache behavior, source-mtime checkout issues, filesystem snapshots, S3 Files experiments, or Cargo fingerprint rebuild diagnostics.
---

# Cargo CI Cache

Use this skill to apply the cache research archived in this repository to Rust/Cargo CI workflows.

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

Use `docs/operations/diagnosing-rebuilds.md` and `examples/workflows/cargo-fingerprint-diagnostics.yml` for concrete commands.

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

Use `docs/approaches/s3-files.md` only as an experiment record or as background for non-Cargo shared filesystem workloads.

## Updating This Archive

When editing this repository:

- Keep conclusions in `docs/approaches/README.md`.
- Keep empirical numbers in `docs/results/empirical-results.md`.
- Keep chronological history in `docs/results/experiment-log.md`.
- Keep procedures in `docs/operations/`.
- Keep examples generic under `examples/`.
- Run `git diff --check`, `actionlint examples/workflows/*.yml`, and YAML parsing after workflow edits.
