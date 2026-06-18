# Quickstart

Use this page when you want the current answer without reading the full archive.

## Recommended Default

For GitHub Actions Rust builds, start with:

- Mtime-preserving cached worktree.
- `Swatinem/rust-cache` with `cache-workspace-crates: true`.
- Explicit stable `CARGO_TARGET_DIR`.
- `mise-action` for Rust-adjacent tool setup.

This is the maintained, low-complexity path that can produce warm Cargo no-op builds. The canonical decision record is [Decisions](decisions/README.md).

## Copy The Right Shape

| Need | Use |
| --- | --- |
| Selected RunsOn deployment | [RunsOn Magic Cache](deployments/runs-on/README.md) and [`runs-on-mise-rust-cache.yml`](../examples/workflows/runs-on-mise-rust-cache.yml) |
| Provider-neutral Cargo cache | [`Swatinem/rust-cache` with mtime-preserving checkout](approaches/rust-cache-mtime-checkout.md) and [`rust-cache-mtime-checkout.yml`](../examples/workflows/rust-cache-mtime-checkout.yml) |
| Tool setup with Rust, Zig, `cargo-lambda`, or Trunk | [Mise Tool Setup](operations/mise-tool-setup.md) |
| Rebuild diagnosis | [Diagnosing Cargo Rebuilds In CI](operations/diagnosing-rebuilds.md) |

## When To Escalate

Use the [source-keyed full-target cache workaround](approaches/rust-cache-source-keyed-target-cache.md) only when measured logs show affected local path workspace members repeatedly rebuilding on exact `rust-cache` hits and the rebuild cost is material.

Do not use S3 Files for Cargo target no-op state based on this archive's experiments. Keep EBS/filesystem snapshots as an archived alternative for cases where maximum filesystem fidelity matters more than lifecycle complexity.

## Mental Model

Cargo can skip compilation only when source inputs, source mtimes, workspace paths, target artifacts, dep-info, fingerprints, build-script outputs, dependency source paths, toolchain, flags, profile, features, and relevant environment agree with each other.

The short explanation is [Cargo Freshness Model](concepts/cargo-freshness-model.md). The detailed signal table is [Cargo Freshness Signals](reference/cargo-freshness-signals.md).
