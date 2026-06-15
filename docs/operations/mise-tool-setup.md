# Mise Tool Setup

Use `mise-action` as the preferred CI setup layer for Rust-adjacent tools and runtimes, especially on RunsOn runners with Magic Cache enabled.

This is not a Cargo cache approach. It is setup guidance that applies before the selected Cargo cache approach, such as `Swatinem/rust-cache` with an mtime-preserving checkout or a source-keyed target cache. Let Cargo caches handle Cargo home and target freshness; let mise handle repeated tool installation.

## Why

`mise-action` uses `actions/cache` for its data directory. With RunsOn Magic Cache backing `actions/cache`, repeated setup of Zig, Rust toolchains/targets, `cargo-binstall`, `cargo-lambda`, `trunk`, and similar tools becomes very fast after the cache is warm.

This removes the need for several separate setup/install actions and avoids paying repeated setup time in every matrix job.

## Recommended Shape

Use inline `mise_toml` in the workflow when the tool set is CI-specific:

```yaml
- name: Setup Toolchain
  uses: jdx/mise-action@v4
  with:
    cache: true
    cache_key_prefix: mise-v1
    mise_toml: |
      [tools]
      zig = "0.16.0"
      rust = { version = "stable", components = "rustfmt", targets = "aarch64-unknown-linux-gnu" }
      cargo-binstall = "latest"
      "cargo:cargo-lambda" = "latest"
```

For a Trunk/WebAssembly job:

```yaml
- name: Setup Toolchain
  uses: jdx/mise-action@v4
  with:
    cache: true
    cache_key_prefix: mise-v1
    mise_toml: |
      [tools]
      rust = { version = "stable", components = "rustfmt", targets = "wasm32-unknown-unknown" }
      cargo-binstall = "latest"
      "cargo:trunk" = "latest"
```

Prefer the mise Cargo backend for Cargo-distributed tools over the GitHub release backend:

- Use `"cargo:cargo-lambda"` for `cargo-lambda`.
- Use `"cargo:trunk"` for Trunk.

No explicit `depends` option is needed in these examples. The Cargo backend declares Rust as a required dependency and `cargo-binstall` as an optional dependency. When `rust`, `cargo-binstall`, and `cargo:*` tools are present in the same install set, mise orders them so the Cargo tools wait for Rust and `cargo-binstall`. Mise then uses `cargo-binstall` by default when it is available, avoiding source compilation when a compatible prebuilt binary exists.

Use an explicit `depends` option only for an additional project-specific ordering constraint that the backend does not already declare.

`cache_key_prefix: mise-v1` is an explicit cache-layout namespace, not the action major. Increment it when changing the mise cache layout or setup policy in a way that should start a fresh tool cache.

## Environment Variables

Recommended:

```yaml
env:
  MISE_DATA_DIR: ${{ github.workspace }}/.mise
  RUSTUP_HOME: ${{ github.workspace }}/.mise/rustup
```

`MISE_DATA_DIR` keeps mise installs under a path that `mise-action` caches.

`RUSTUP_HOME` keeps Rust toolchains and rustup targets under the mise cache. This matters because mise manages Rust through rustup; Rust does not live under mise's normal `installs/` directory.

Usually not required:

- `MISE_RUSTUP_HOME`: redundant when `RUSTUP_HOME` is already set to the intended path.
- `MISE_OVERRIDE_CONFIG_FILENAMES`: not set by `mise-action`, but generally unnecessary after `mise-action` exports the resolved `PATH`, `RUSTUP_HOME`, and `RUSTUP_TOOLCHAIN` for later steps.

Avoid:

- Do not set `CARGO_HOME` or `MISE_CARGO_HOME` under the mise cache when registry credentials are written there. Let `rust-cache` own Cargo home caching and credential-sensitive Cargo paths.

## Ordering

Run setup in this order:

1. Restore/check out the workspace using the repository's mtime-preserving strategy.
2. Configure private registry credentials such as Shipyard.
3. Run `mise-action` for toolchains and setup tools.
4. Restore `rust-cache` / target caches.
5. Run Cargo builds with explicit `CARGO_TARGET_DIR`.

`mise-action` should happen before `rust-cache` so `rust-cache` sees the Rust environment that the build will use.

## Target-Key Rule

Changing setup tooling can change Cargo's build semantics even when application source does not change. If using a source-keyed target cache, bump the target-key namespace after changing any of these:

- Rust home/toolchain location.
- Rust targets.
- Build flags such as `--locked` vs `--frozen`.
- Target triples.
- Profiles or features.
- Tool wrappers or setup backend, such as switching from `dtolnay/rust-toolchain` and installer actions to mise.

Example:

```yaml
target-key: mise-locked-v1-${{ steps.app-source-key.outputs.hash }}
```

The first run after a namespace bump should seed the new target cache. The immediate follow-up run is the one that should prove warm no-op behavior.

## What This Does Not Solve

Mise setup caching makes tool installation fast. It does not by itself prove Cargo units fresh. Cargo no-op behavior still depends on source mtimes, target fingerprints, dep-info files, build-script outputs, registry source paths, and consistent build semantics.

Keep using the selected Cargo cache approach, such as `Swatinem/rust-cache` with mtime-preserving checkout, and use a source-keyed target cache when affected local path workspace members repeatedly rebuild and justify the extra cache composition.

## Official References

- [`mise-action` README and cache configuration](https://github.com/jdx/mise-action)
- [`mise-action` input definitions](https://github.com/jdx/mise-action/blob/main/action.yml)
- [Mise Cargo backend and `cargo-binstall` behavior](https://mise.jdx.dev/dev-tools/backends/cargo.html)
- [Mise tool dependency ordering](https://mise.jdx.dev/dev-tools/)
- [Cargo backend dependency declarations](https://github.com/jdx/mise/blob/40c2a2373c2f85f7f6cadfdfe377db9060686076/src/backend/cargo.rs#L103-L109)
- [Install dependency graph construction](https://github.com/jdx/mise/blob/40c2a2373c2f85f7f6cadfdfe377db9060686076/src/toolset/tool_deps.rs#L48-L66)
- [Open Rust cache interaction issue](https://github.com/jdx/mise-action/issues/215)
