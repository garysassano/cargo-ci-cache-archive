# Maintenance Checklist

Use this checklist when refreshing the archive or copying its examples into a live repository.

## Example Versions

- Check current GitHub-owned action majors for `actions/checkout`, `actions/cache`, `actions/upload-artifact`, and `actions/download-artifact`.
- Keep non-GitHub actions intentionally pinned or floating by policy. For example, this archive keeps `Swatinem/rust-cache@v2` and `dtolnay/rust-toolchain@stable` because those are the intended upstream interfaces.
- Re-check `jdx/mise-action` inputs and cache-key behavior when changing mise setup examples.
- Run `actionlint examples/workflows/*.yml` when `actionlint` is available.
- Parse all example YAML files after edits.

## Cargo Cache Semantics

- Re-check `Swatinem/rust-cache` release notes before changing the recommendation, especially around target keys, `cache-workspace-crates`, incremental state, and save cleanup behavior.
- Re-check the official [Cargo checksum freshness documentation](https://doc.rust-lang.org/nightly/cargo/reference/unstable.html#checksum-freshness) and [tracking issue](https://github.com/rust-lang/cargo/issues/14136) before changing source-mtime guidance.
- Keep the source-keyed target-cache workaround documented until upstream target keys include workspace source state or an equivalent mechanism exists.
- When using source-keyed target caches, include build-command semantics in the target key, for example `locked-v1-<source-hash>`, and bump the namespace after changing build flags, target triples, profiles, features, or wrappers.
- Also bump the target-key namespace after changing setup semantics that affect Cargo's environment, such as moving Rust/rustup home, switching from Rust/Zig installer actions to mise, or changing Cargo helper installation backends.
- Prefer `--locked` for CI artifact builds. Do not switch to `--frozen` / `--offline` with `rust-cache` unless complete local registry/index state is known to be restored.
- Prefer `mise-action` with inline `mise_toml` for stable setup tools such as Zig, Rust targets, `cargo-lambda`, `trunk`, and `cargo-binstall`; do not rely on `rust-cache cache-bin=true` as the only cache for those tools when setup time matters.
- Preserve the warning not to mix full filesystem snapshots with `rust-cache` on the same `target/` or `$CARGO_HOME` paths.

## Platform Guidance

- Keep selected RunsOn Magic Cache guidance and its current-version checks in
  [`docs/runs-on/README.md`](../runs-on/README.md).

## Archived AWS Experiments

- Re-check current S3 Files docs before using the S3 Files page for new
  experiments.
- Re-check `runs-on/snapshot` inputs and snapshot identity behavior before copying the snapshot example.
- Keep credential scrubbing guidance on any snapshot layout that places `$CARGO_HOME` under the snapshot root.
- Keep snapshot and S3 Files local action examples generic; do not reintroduce app-specific names, secrets, or runner labels.

## Results Hygiene

- Keep empirical numbers in `docs/results/empirical-results.md`, not duplicated across approach pages.
- Keep chronological experiment history in `docs/results/experiment-log.md`.
- Keep diagnostic procedures in `docs/operations/diagnosing-rebuilds.md`.
