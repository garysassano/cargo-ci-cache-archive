# `Swatinem/rust-cache` Vs `runs-on/snapshot` Evidence

This page records the empirical comparison between `Swatinem/rust-cache` and the archived `runs-on/snapshot` EBS filesystem-restore approach.

## Question

How much reusable Cargo state does each method restore, and does Cargo perform any compilation on the next identical build?

## Test Setup

The comparison workflow ran the same generic Cargo command in two jobs:

```bash
cargo build --workspace --locked -v
```

Both jobs enabled Cargo fingerprint diagnostics:

```bash
CARGO_LOG=cargo::core::compiler::fingerprint=trace
```

The jobs were:

| Job | Method |
| --- | --- |
| `rust-cache-magic-cache` | `Swatinem/rust-cache@v2` using RunsOn Magic Cache as the `actions/cache` backend |
| `snapshot-cache` | `runs-on/snapshot` EBS-backed filesystem restore |

The workflow captured before/after counts for:

```text
target files
fingerprint files
build-script files
incremental files
registry source files
```

## Observations

### Seed Run

The first run seeded both methods.

| Run | Method | `Compiling` lines | `Fresh` lines | Fingerprint / dirty indicators |
| --- | ---: | ---: | ---: | ---: |
| Seed | `Swatinem/rust-cache` with Magic Cache | 623 | 0 | 1584 |
| Seed | `runs-on/snapshot` EBS restore | 623 | 0 | 1584 |

### Reuse Run

The second run measured reuse.

| Run | Method | `Compiling` lines | `Fresh` lines | Fingerprint / dirty indicators |
| --- | ---: | ---: | ---: | ---: |
| Reuse | `Swatinem/rust-cache` with Magic Cache | 30 | 593 | 72 |
| Reuse | `runs-on/snapshot` EBS restore | 0 | 623 | 0 |

### State Before Second Build

| State before second build | `Swatinem/rust-cache` with Magic Cache | `runs-on/snapshot` EBS restore |
| --- | ---: | ---: |
| Exact cache hit | `true` | not applicable |
| `CARGO_INCREMENTAL` | `0` | `0` in this workflow |
| Target files | 5724 | 6021 |
| Fingerprint files | 2902 | 3040 |
| Dep-info files | 695 | 758 |
| Build-script files | 970 | 993 |
| Incremental files | 0 | 0 |
| Registry source files | 5343 | 38230 |

### State After Second Build

The `Swatinem/rust-cache` job recreated additional state during the second build:

| State after second `rust-cache` build | Count |
| --- | ---: |
| Target files | 6021 |
| Fingerprint files | 3040 |
| Dep-info files | 758 |
| Build-script files | 993 |
| Registry source files | 38230 |

The `runs-on/snapshot` job already had the complete state before the second build and did not need to grow these counts during build.

### Magic Cache Object Shape

A read-only object listing of the RunsOn cache bucket showed that Magic Cache stores `actions/cache` entries as keyed objects shaped like:

```text
cache/v1/<owner>/<repo>/<git-ref>/<cache-version>/<cache-key>
```

Representative sanitized cache keys looked like:

```text
v0-rust-<job>-Linux-x64-<rust-env-hash>-<lock-hash>
setup-zig-tarball-zig-x86_64-linux-<version>
setup-rustcargo-v1-linux-<hash>
```

This matches the expected model: RunsOn Magic Cache changes the backend used by `actions/cache`, but the result is still keyed archive/cache content. It is not a mounted filesystem snapshot.

## Interpretation

The `Swatinem/rust-cache` job had an exact cache hit, but it restored a pruned/reconstructed subset of Cargo state. During the build, Cargo recreated missing extracted registry sources, target metadata, fingerprints, dep-info, and build-script state.

That is why the second run still had:

```text
30 Compiling lines
72 fingerprint / dirty indicators
```

The EBS snapshot job restored the complete post-build filesystem state from the previous run. Cargo could prove every unit fresh:

```text
0 Compiling lines
623 Fresh lines
0 fingerprint / dirty indicators
```

## Limitations

In this empirical workflow, `CARGO_INCREMENTAL=0` was present. `Swatinem/rust-cache` explicitly exports `CARGO_INCREMENTAL=0`, and Rust setup actions may also set it when unset. Therefore this run compares Cargo freshness metadata reuse, not rustc incremental reuse.

If incremental compilation is enabled and `target/**/incremental` is under the snapshot root, filesystem snapshots can preserve it. `Swatinem/rust-cache` intentionally disables and cleans incremental state.

## Implications

- Use this evidence to understand why a complete filesystem restore can produce a stricter Cargo no-op than dependency-oriented archive cleanup.
- Do not treat the snapshot result as the selected deployment recommendation; operational tradeoffs remain documented in the [approach comparison](../approaches/README.md).
- See the [`rust-cache` approach](../approaches/rust-cache-mtime-checkout.md) and [EBS snapshot approach](../approaches/ebs-snapshot.md) for implementation guidance.
