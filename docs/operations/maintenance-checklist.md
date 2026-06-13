# Maintenance Checklist

Use this checklist when refreshing the archive or copying its examples into a live repository.

## Example Versions

- Check current GitHub-owned action majors for `actions/checkout`, `actions/cache`, `actions/upload-artifact`, and `actions/download-artifact`.
- Keep non-GitHub actions intentionally pinned or floating by policy. For example, this archive keeps `Swatinem/rust-cache@v2` and `dtolnay/rust-toolchain@stable` because those are the intended upstream interfaces.
- Run `actionlint examples/workflows/*.yml` when `actionlint` is available.
- Parse all example YAML files after edits.

## Cargo Cache Semantics

- Re-check `Swatinem/rust-cache` release notes before changing the recommendation, especially around target keys, `cache-workspace-crates`, incremental state, and save cleanup behavior.
- Re-check Cargo checksum freshness status before changing source-mtime guidance.
- Keep the source-keyed target-cache workaround documented until upstream target keys include workspace source state or an equivalent mechanism exists.
- Preserve the warning not to mix full filesystem snapshots with `rust-cache` on the same `target/` or `$CARGO_HOME` paths.

## AWS / RunsOn Notes

- Re-check current S3 Files docs before using the S3 Files page for new experiments.
- Re-check `runs-on/snapshot` inputs and snapshot identity behavior before copying the snapshot example.
- Keep credential scrubbing guidance on any snapshot layout that places `$CARGO_HOME` under the snapshot root.

## Results Hygiene

- Keep empirical numbers in `docs/results/empirical-results.md`, not duplicated across approach pages.
- Keep chronological experiment history in `docs/results/experiment-log.md`.
- Keep diagnostic procedures in `docs/operations/diagnosing-rebuilds.md`.
