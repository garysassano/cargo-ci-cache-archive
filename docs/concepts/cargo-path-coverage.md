# Cargo Path Coverage

This page lists the Cargo paths that matter between builds and how the major CI cache approaches cover them. The key question is not only whether a file exists after restore, but whether Cargo can use the restored state to prove a build unit is fresh.

## Restore Coverage

| Path | Purpose | `Swatinem/rust-cache` | EBS snapshot / filesystem restore |
| --- | --- | ---: | ---: |
| `<workspace>/Cargo.toml` | Package graph, targets, dependency declarations | Hashed into key only | Covered if workspace is under snapshot root |
| `<workspace>/Cargo.lock` | Exact dependency graph | Hashed into key only | Covered if workspace is under snapshot root |
| `<workspace>/.cargo/config.toml` | Registry, flags, target config | Hashed into key if found | Covered if workspace is under snapshot root |
| `<workspace>/**/src/*.rs` | Source inputs | Recreated by checkout | Covered if workspace is under snapshot root |
| `<workspace>/**/build.rs` | Build script source | Recreated by checkout | Covered if workspace is under snapshot root |
| Generated workspace files | Source/codegen inputs | Recreated by checkout or build | Covered if under snapshot root |
| `$CARGO_HOME/registry/cache` | Compressed crate archives | Covered | Covered if `CARGO_HOME` is under snapshot root |
| `$CARGO_HOME/registry/src` | Extracted dependency sources | Usually recreated or partially preserved | Covered if `CARGO_HOME` is under snapshot root |
| `$CARGO_HOME/registry/index` | Registry metadata | Covered and cleaned | Covered if `CARGO_HOME` is under snapshot root |
| `$CARGO_HOME/git/db` | Git dependency bare repos | Covered and cleaned | Covered if `CARGO_HOME` is under snapshot root |
| `$CARGO_HOME/git/checkouts` | Git dependency source trees | Covered and cleaned for used refs | Covered if `CARGO_HOME` is under snapshot root |
| `$CARGO_HOME/bin` | Cargo-installed binaries | Covered if `cache-bin=true` | Covered if `CARGO_HOME` is under snapshot root |
| `$XDG_CACHE_HOME/cargo-zigbuild` | Cargo helper cache/state | Not covered unless separately cached | Covered if `XDG_CACHE_HOME` is under snapshot root |
| Trunk tool cache, for example `$XDG_CACHE_HOME/dev.trunkrs.trunk` | Trunk-managed helper binaries such as `wasm-bindgen`, `wasm-opt`, and `tailwindcss` | Not covered unless separately cached | Covered if under snapshot root |
| `target/<profile>/deps/*.rlib` | Compiled library artifacts | Dependency-oriented; workspace artifacts require `cache-workspace-crates` and still follow `rust-cache` key behavior | Covered |
| `target/<profile>/deps/*.rmeta` | Rust metadata for downstream crates | Dependency-oriented | Covered |
| `target/<profile>/deps/*.d` | Dep-info freshness/input tracking | Dependency-oriented | Covered |
| `target/<profile>/.fingerprint/**` | Cargo unit freshness metadata | Dependency-oriented | Covered |
| `target/<profile>/build/**` | Build script outputs and `OUT_DIR` state | Dependency-oriented | Covered |
| `target/<profile>/incremental/**` | rustc incremental recompilation state | Disabled and cleaned by `rust-cache` | Covered if enabled and under snapshot root |
| `target/<profile>/<final-binary>` | Final executable output | Workspace final artifacts are usually not the primary target | Covered |
| `target/<target-triple>/<profile>/**` | Cross-compiled artifacts | Dependency-oriented | Covered |
| Non-Cargo tool cache dirs | Setup action cache/tool state | Only if separately cached or added via `cache-directories` | Covered if under snapshot root |

## Save Behavior

`Swatinem/rust-cache` restore and save behavior is intentionally not symmetric. On save, it prunes state before uploading the archive.

Important `rust-cache` save behavior:

```text
keeps dependency-oriented target artifacts
workspace crate artifacts require cache-workspace-crates=true
usually removes most extracted registry/src content
cleans unused dependencies
removes pre-existing cargo bin entries
```

`cache-bin=true` should not be treated as a general setup-tool cache. In the
tested workflow it was not the right home for stable helper tools such as
`cargo-lambda` or `trunk`; those are setup state now handled by `mise-action` and
RunsOn Magic Cache. Binaries restored into `$CARGO_HOME/bin` also exist before the
`rust-cache` post step computes what changed during the job, so they can be
considered pre-existing and removed before the next save. For stable CI helper
tools, prefer a custom runner image, the setup action's own cache, or an explicit
tool cache. Use `rust-cache` for `$CARGO_HOME/bin` only when that tradeoff is
acceptable.

An EBS snapshot preserves the mounted filesystem subtree. If the path is under the snapshot root and was not removed before the post step, it is saved.

## Best Home For Each State Type

| State | Best home | Why |
| --- | --- | --- |
| `target/` | EBS snapshot for maximum no-op fidelity, or `rust-cache` for simpler dependency-oriented caching | Cargo freshness depends on artifacts, dep-info, fingerprints, build script outputs, and stable filesystem metadata. |
| `$CARGO_HOME/registry`, `$CARGO_HOME/git` | `rust-cache` for practical dependency caching, EBS snapshot for full filesystem continuity | Extracted sources and mtimes can matter for perfect no-op behavior. |
| `$XDG_CACHE_HOME/cargo-zigbuild` | EBS snapshot or explicit cache if the helper cache matters | This is Cargo-helper state created by a Cargo build frontend. |
| Trunk tool cache | Custom AMI, explicit cache, or snapshot | Trunk downloads helper tools outside Cargo target state. Cache these paths separately if their install time matters. |
| Cargo-installed helper binaries | Custom AMI or `mise-action` cache preferred | These are setup state, not freshness proof; `rust-cache cache-bin` may not persist them reliably across runs. |
| Rust toolchain and rustup targets | Custom AMI preferred, otherwise setup action/Magic Cache | Toolchain state is large and stable. |
| Zig compiler install | Custom AMI preferred, otherwise setup action/Magic Cache | Stable tool state. |
| Zig tarball/download cache | `actions/cache` or RunsOn Magic Cache | Immutable download archives fit keyed archive cache semantics. |
| Node/package/deployment dependencies | Ecosystem cache or custom AMI | Outside Cargo freshness. |

## Important Warning

Avoid combining `Swatinem/rust-cache` with a full Cargo build-state snapshot for the same paths. `rust-cache` restores and prunes an archive-oriented subset, while a snapshot depends on preserving filesystem continuity. Mixing both for `target/` or `$CARGO_HOME` can rewrite files, alter mtimes, and reduce the snapshot value for local no-op behavior.
