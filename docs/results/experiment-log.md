# Experiment Log

This log preserves the major observations from the cache experiments. It is intentionally practical and result-oriented.

## Baseline Problem

Normal GitHub Actions checkouts rewrite source mtimes. Cargo uses source mtimes, target artifacts, dep-info, fingerprints, build-script outputs, and toolchain/config fingerprints to decide whether units are fresh.

The initial symptom was that Cargo rebuilt local workspace crates even when target artifacts were restored. The main reason was that source files appeared newer than restored target fingerprints after checkout.

## S3 Files Experiments

S3 Files was tested for Cargo target and registry state.

Important observations:

- Cargo could report logically clean state: `Compiling 0`, `Downloaded 0`, `Dirty 0`, `fingerprint error 0`.
- Actual no-op elapsed time remained high because Cargo still traversed target metadata and fingerprints.
- Pure S3 Files registry plus target produced forced no-op Cargo times around 35 to 39 seconds.
- Prewarm reduced the later Cargo step but moved cost into the prewarm step.
- Copying registry state from S3 Files to local disk was not viable; one copy test took about 932 seconds.
- `rust-cache` for registry plus S3 Files for target improved some runs to around 8 to 14 seconds of Cargo time, but local target cache remained better overall.
- S3 Files mount/setup costs included helper package/install overhead in the runner image used during testing.

Conclusion: S3 Files is not the right primitive for Cargo target no-op state in this workflow.

## EBS Snapshot Analysis

The analysis compared `Swatinem/rust-cache` with `runs-on/snapshot` for generic workspace builds.

Key empirical comparison:

| Method | Reuse `Compiling` lines | Reuse `Fresh` lines | Fingerprint / dirty indicators |
| --- | ---: | ---: | ---: |
| `Swatinem/rust-cache` + Magic Cache | 30 | 593 | 72 |
| `runs-on/snapshot` | 0 | 623 | 0 |

Interpretation:

- `runs-on/snapshot` most closely restores the filesystem Cargo saw last time.
- `rust-cache` restores a cleaned archive subset that is excellent for dependencies but not equivalent to a complete workspace build-state snapshot.

Conclusion: EBS snapshots are the strongest model for local no-op fidelity, but heavier operationally.

## Cached Worktree Checkout

A custom cached worktree checkout action was introduced to preserve source mtimes.

Behavior:

- Restore cached worktree from `actions/cache`.
- If `.git/HEAD` already equals `GITHUB_SHA`, skip fetch/checkout entirely.
- If the worktree is older, fetch and checkout in place.
- Git rewrites changed files and leaves unchanged files with previous mtimes.

Result:

- This fixed the main false rebuild source.
- Repeated same-SHA runs allowed many jobs to become Cargo no-ops.

## First No-S3 Implementation

Workflow shape:

```text
cached worktree checkout
per-job CARGO_TARGET_DIR
Swatinem/rust-cache cache-targets: true
cache-all-crates: true
cache-workspace-crates: true
```

Result:

- Most matrix jobs completed around 33 to 37 seconds.
- A representative no-op Cargo step finished around 0.31 seconds.
- A few outliers remained around 50 to 65 seconds.

Outliers involved generated-code/build-script crates and downstream binaries.

## Build Script Input Hints

Explicit `cargo:rerun-if-changed` hints were added to local generated-code build scripts:

```rust
println!("cargo:rerun-if-changed=build.rs");
println!("cargo:rerun-if-changed=path/to/generated-input");
```

This is good build-script hygiene, but it did not fix the repeated CI rebuild alone.

## `rust-cache` Exact-Hit Behavior

The repeated outliers persisted because `rust-cache` could restore an exact key that did not include workspace source contents. Since the key was exact, the post step reported `Cache up-to-date` and did not save the newly rebuilt workspace target artifacts.

Cycle observed:

```text
restore stale exact target cache
Cargo rebuilds some workspace crates
rust-cache post step says Cache up-to-date
rebuilt target state is not saved
next run restores same stale exact target cache
same crates rebuild again
```

## Source-Keyed Target Cache Workaround

The proven workaround split cache responsibility:

```text
rust-cache manages Cargo home only
actions/cache manages full per-job target directory
target cache key includes source state
target cache restore happens after rust-cache
```

Incorrect ordering tested:

```text
restore target cache
then rust-cache
```

This still rebuilt because `rust-cache` cleanup could remove target artifacts.

Correct ordering:

```text
rust-cache restore
then restore target cache
then build
```

Result:

- All previously slow jobs became true Cargo no-ops.
- Cargo build phases were around 0.24 to 0.31 seconds.
- All tested matrix jobs finished around 34 to 37 seconds in the final verification run.

Decision:

- Keep this workaround documented.
- Do not select it as the default yet because it adds cache composition complexity.
- Ask upstream `Swatinem/rust-cache` for source-keyed target-cache support.

## Native `rust-cache` Target-Key Prototype

A local `rust-cache` prototype added a native `target-key` input that splits Cargo
home and target caches inside `rust-cache` itself. The target cache key includes a
user-provided source fingerprint, while Cargo home keeps the existing dependency
key strategy.

Test shape:

```text
cached worktree checkout
rust-cache with cache-targets=true
cache-workspace-crates=true
target-key=<build-mode>-<source-hash>
build with explicit per-job CARGO_TARGET_DIR
```

Important result:

- A seed run created the new source-keyed target caches.
- A repeated run restored exact Cargo-home and target-cache hits for all tested
  binary and UI jobs.
- Cargo build phases were around 0.3 seconds.
- No `Compiling` lines were emitted in the repeated run.

Keying lesson:

- The target key must include build-command semantics, not only source state.
- Adding `--locked` changed Cargo freshness enough that the first run against the
  old source-only target key rebuilt some workspace artifacts.
- Prefixing the target key with a namespace such as `locked-v1-` created a new
  lineage; after seeding that lineage, repeated runs were no-op again.

Cargo flag lesson:

- `--locked` is appropriate for CI artifact builds and should be part of the
  target-key namespace when introduced.
- `--locked` does not suppress `Updating crates.io index`; it only prevents
  modifying `Cargo.lock`.
- `--frozen` / `--offline` failed with the `rust-cache`-managed Cargo home,
  because offline mode needs complete local registry/index state while
  `rust-cache` intentionally prunes Cargo home.

Tool-cache lesson:

- `cache-bin=true` did not remove the need for a dedicated setup layer for helper
  commands such as `cargo-lambda` and `trunk` on every job in the tested workflow.
- Treat Cargo-installed helper binaries and frontend-managed tools as setup state,
  not Cargo freshness proof.
- Preinstall stable helper tools in the runner image, use `mise-action`/setup
  action caching when available, or cache the tool's own cache directory explicitly.
- Inline `mise_toml` is written under `$GITHUB_WORKSPACE`; later Cargo commands
  must run where mise can discover that config. A cached worktree under
  `$GITHUB_WORKSPACE/cached-worktree/app` worked without overrides.
- With config discovery fixed, `zig = "latest"` and `"cargo:cargo-lambda" = "latest"`
  were valid setup-tool declarations, but a warm `mise-action` cache can keep using
  the previously resolved versions. The selected workflow pins artifact-affecting
  tools instead so upgrades are explicit, while leaving `cargo-binstall` on
  `latest` because it is an installer mechanism. Rust stays on `stable` to match
  the repository's Rust CI unless the repo later adds a checked-in Rust version
  source.
- `depends = ["rust", "cargo-binstall"]` on mise Cargo tools is not the fix for
  `No version is set for shim`; that error came from config discovery, not missing
  installation.
- For Trunk, the remaining repeated-run time came mostly from its own pipeline:
  pre-build hook, helper tool downloads/installs, wasm processing, and dist
  application, rather than from missing Cargo target artifacts.

Decision:

- Keep the external source-keyed target-cache workaround as the documented copyable
  approach until native `target-key` support is available from upstream
  `Swatinem/rust-cache`.
- If native support lands upstream, update the examples to remove the separate
  target `actions/cache` step and use `target-key` directly.
