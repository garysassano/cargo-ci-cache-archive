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
| `$CARGO_HOME/bin` | Cargo-installed binaries | Covered if `cache-bin=true` and represented in Cargo's installed-crate metadata; directly extracted binaries are removed during save cleanup | Covered if `CARGO_HOME` is under snapshot root |
| `$XDG_CACHE_HOME/cargo-zigbuild` | Cargo helper cache/state | Not covered unless separately cached | Covered if `XDG_CACHE_HOME` is under snapshot root |
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
See [`Swatinem/rust-cache` Behavior](rust-cache-behavior.md) for input defaults,
true/false examples, and the exact cleanup rules used by the documented
approaches.

Important `rust-cache` save behavior:

```text
keeps dependency-oriented target artifacts
workspace crate artifacts require cache-workspace-crates=true
usually removes most extracted registry/src content
cleans unused dependencies
removes pre-existing cargo bin entries
```

An EBS snapshot preserves the mounted filesystem subtree. If the path is under the snapshot root and was not removed before the post step, it is saved.

## Choosing Coverage

| State | Preferred coverage | Why |
| --- | --- | --- |
| `target/` | `rust-cache` for the selected default; add the source-keyed full-target cache only when measured rebuilds justify it | Cargo freshness depends on artifacts, dep-info, fingerprints, build script outputs, and stable filesystem metadata. |
| `$CARGO_HOME/registry`, `$CARGO_HOME/git` | `rust-cache` | These are dependency download and source inputs that `rust-cache` is designed to manage. |
| `$XDG_CACHE_HOME/cargo-zigbuild` | Explicit `actions/cache` entry if preserving the helper cache is worthwhile | This is Cargo-helper state outside Cargo home and `target/`. |
| Cargo-installed helper binaries | Custom AMI or setup action preferred; `rust-cache` is useful for Cargo-registered installs | These are setup state, not freshness proof. |
| Rust toolchain and rustup targets | Custom AMI preferred, otherwise a setup action's cache | Toolchain state is large and stable. |
| Zig compiler install | Custom AMI preferred, otherwise a setup action's cache | Stable tool state. |
| Zig tarball/download cache | `actions/cache` | Immutable download archives fit keyed archive cache semantics. |
| Node/package/deployment dependencies | Ecosystem cache or custom AMI | Outside Cargo freshness. |

The archived EBS snapshot approach covers these paths with greater filesystem
continuity, but it is not the selected deployment because of its operational
and lifecycle complexity.

## Important Warning

Avoid combining `Swatinem/rust-cache` with a full Cargo build-state snapshot for the same paths. `rust-cache` restores and prunes an archive-oriented subset, while a snapshot depends on preserving filesystem continuity. Mixing both for `target/` or `$CARGO_HOME` can rewrite files, alter mtimes, and reduce the snapshot value for local no-op behavior.
