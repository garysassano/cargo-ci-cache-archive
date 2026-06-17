# Mise Tool Setup

Use `mise-action` as the preferred CI setup layer for Rust-adjacent tools and runtimes, especially on RunsOn runners with Magic Cache enabled.

This is not a Cargo cache approach. It is setup guidance that applies before the selected Cargo cache approach, such as `Swatinem/rust-cache` with an mtime-preserving checkout or a source-keyed target cache. Let Cargo caches handle Cargo home and target freshness; let mise handle repeated tool installation.

## Why

`mise-action` uses `actions/cache` for its mise directory. If `mise_dir` is not set, the action falls back to `MISE_DATA_DIR`, then XDG/default home paths. Set `MISE_DATA_DIR` so both mise and the action use the same cached tree for normal tool installs. Put mise-managed Rust state under that tree with `MISE_RUSTUP_HOME`, because Rust is installed through rustup rather than mise's normal `installs/` directory.

With RunsOn Magic Cache backing `actions/cache`, repeated installs of Zig, Rust toolchains/targets, `cargo-binstall`, `cargo-lambda`, `trunk`, and similar setup tools become effectively free after the cache is warm.

This removes the need for several separate setup/install actions and avoids paying repeated setup time in every matrix job.

## Recommended Shape

Use inline `mise_toml` in the workflow when the tool set is CI-specific:

```yaml
env:
  MISE_DATA_DIR: ${{ github.workspace }}/.mise
  MISE_RUSTUP_HOME: ${{ github.workspace }}/.mise/rustup

steps:
  - name: Setup Toolchain
    uses: jdx/mise-action@v4
    with:
      cache: true
      mise_toml: |
        [tools]
        zig = "0.16.0"
        rust = { version = "stable", components = "rustfmt", targets = "aarch64-unknown-linux-gnu" }
        cargo-binstall = "latest"
        "cargo:cargo-lambda" = "1.9.1"
```

For a Trunk/WebAssembly job:

```yaml
env:
  MISE_DATA_DIR: ${{ github.workspace }}/.mise
  MISE_RUSTUP_HOME: ${{ github.workspace }}/.mise/rustup

steps:
  - name: Setup Toolchain
    uses: jdx/mise-action@v4
    with:
      cache: true
      mise_toml: |
        [tools]
        rust = { version = "stable", components = "rustfmt", targets = "wasm32-unknown-unknown" }
        cargo-binstall = "latest"
        "cargo:trunk" = "0.21.14"
```

Install `cargo-binstall` first so mise can use prebuilt binaries where available instead of compiling tool CLIs.

Pin artifact build tools such as Zig, `cargo-lambda`, and Trunk. `latest` is convenient while experimenting, but a warm `mise-action` cache does not automatically invalidate when a new upstream Zig or Cargo tool release appears. The cache key includes the config text and restored cache path; if `zig = "latest"` previously resolved to `0.16.0`, repeated cached runs can continue using `0.16.0` until the cache key changes or the cache is refreshed. Pinning makes CI artifacts reproducible and makes version changes explicit. `cargo-binstall` is an installer mechanism for Cargo-backed tools, so using `latest` for it is acceptable.

Use `rust = "stable"` unless the repository declares a Rust version in `rust-toolchain.toml` or `workspace.package.rust-version`. This keeps app artifact builds aligned with normal Rust CI that also tracks stable. If the repository adopts a single checked-in Rust version, update the mise config to follow that source of truth.

Do not use `depends = ["rust", "cargo-binstall"]` to compensate for hidden config discovery problems. With a selected `$GITHUB_WORKSPACE/cached-worktree/app` layout, mise can discover the inline `mise_toml` naturally and Cargo-backed setup tools such as `cargo-lambda` work without `depends`.

The historical `No version is set for shim: cargo-lambda` failure was not an install failure. `mise install` installed `cargo-lambda`, and `mise ls` showed it. The later build failed because the shim ran from a worktree outside `$GITHUB_WORKSPACE`, could not discover `$GITHUB_WORKSPACE/mise.toml`, and therefore had no active version.

Prefer the mise Cargo backend for Cargo-distributed tools over the GitHub release backend:

- Use `"cargo:cargo-lambda"` for `cargo-lambda`.
- Use `"cargo:trunk"` for Trunk.

## Environment Variables

Required for warm setup caches:

```yaml
env:
  MISE_DATA_DIR: ${{ github.workspace }}/.mise
  MISE_RUSTUP_HOME: ${{ github.workspace }}/.mise/rustup

steps:
  - name: Setup Toolchain
    uses: jdx/mise-action@v4
    with:
      cache: true
```

`MISE_DATA_DIR` is the mise runtime data directory. Normal mise-managed tools and shims live there. `mise-action` also uses `MISE_DATA_DIR` as its cache path when `mise_dir` is not provided, so setting this one env var is usually enough.

`mise_dir` is still useful as an explicit override, but it is not required when `MISE_DATA_DIR` is already set. If both are used, they should point at the same path. Setting only `mise_dir` is not equivalent to setting `MISE_DATA_DIR`, because `mise-action` does not export `MISE_DATA_DIR` for mise.

`MISE_RUSTUP_HOME` keeps Rust toolchains and rustup targets under the same cached tree. This matters because mise manages Rust through rustup; Rust does not live under mise's normal `installs/` directory. The official mise Rust docs state that Rust respects `RUSTUP_HOME` and `CARGO_HOME`, and that `MISE_RUSTUP_HOME` and `MISE_CARGO_HOME` can isolate mise's rustup/cargo state from other installations.

`MISE_OVERRIDE_CONFIG_FILENAMES` is required when `mise-action` uses inline `mise_toml` and later build steps run outside `$GITHUB_WORKSPACE`. The action writes inline `mise_toml` to `$GITHUB_WORKSPACE/mise.toml`; it does not write it to `mise_dir`, and `working_directory` does not change where the inline file is written. If the build runs from any worktree outside `$GITHUB_WORKSPACE`, mise's normal upward config search will not find `$GITHUB_WORKSPACE/mise.toml` unless this override is set.

If the build worktree is under `$GITHUB_WORKSPACE`, for example `$GITHUB_WORKSPACE/cached-worktree/app`, `MISE_OVERRIDE_CONFIG_FILENAMES` is not needed because mise can discover `$GITHUB_WORKSPACE/mise.toml` naturally. Prefer a descriptive directory name such as `cached-worktree` over `cached` because this workflow also caches mise data, Cargo target directories, and Rust/Cargo state.

A clear workspace layout is:

```yaml
env:
  CACHED_WORKTREE: ${{ github.workspace }}/cached-worktree
  CACHED_CARGO_TARGET_DIR: ${{ github.workspace }}/cached-cargo-target-${{ matrix.lambda.name }}
  CACHED_CONSOLE_TARGET_DIR: ${{ github.workspace }}/cached-console-ui-target
  MISE_DATA_DIR: ${{ github.workspace }}/.mise
  MISE_RUSTUP_HOME: ${{ github.workspace }}/.mise/rustup
```

Keep Cargo target directories outside `cached-worktree/app` so source checkout state and build output state remain separate, but keep them under `$GITHUB_WORKSPACE` so all restored/saved CI state is easy to inspect.

```mermaid
flowchart TD
    A[Restore cached worktree and<br/>configure registry credentials] --> B[mise-action runs]

    B --> C{mise_dir input set?}
    C -->|Yes| D[Action cache path uses<br/>mise_dir]
    C -->|No| E[Action cache path falls back to<br/>MISE_DATA_DIR]
    D --> D2[Action restores/saves<br/>its selected cache path]
    E --> D2

    B --> F[MISE_DATA_DIR env var]
    F --> F2[Normal mise tools install in<br/>$GITHUB_WORKSPACE/.mise]
    F2 --> G[zig, cargo-binstall, cargo-lambda, trunk]

    B --> H[MISE_RUSTUP_HOME env var]
    H --> I[Rustup stores Rust toolchains in<br/>$GITHUB_WORKSPACE/.mise/rustup]
    I --> J[rust stable and targets]

    B --> O[mise_toml input]
    O --> P[Writes config to<br/>$GITHUB_WORKSPACE/mise.toml]

    P --> Q{Build cwd under<br/>$GITHUB_WORKSPACE?}
    Q -->|Yes| K
    Q -->|No| R[Set MISE_OVERRIDE_CONFIG_FILENAMES]
    R --> K

    G --> K[Use tools during build]
    J --> K

    D2 --> L[Recommended: action cache path<br/>and MISE_DATA_DIR are the same tree]
    L --> M[Faster toolchain setup]
```

Usually not required:

- `RUSTUP_HOME`: prefer `MISE_RUSTUP_HOME` when Rust is managed by mise, because it makes ownership explicit and avoids changing non-mise rustup behavior.
- `depends`: not required for Cargo-backed setup tools when the later build can discover the same mise config that `mise-action` used. It may still be useful for documenting install order, but it should not be used as the fix for config visibility.

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
- Moving `MISE_DATA_DIR`, `MISE_RUSTUP_HOME`, cached worktrees, or cached target directories.

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
