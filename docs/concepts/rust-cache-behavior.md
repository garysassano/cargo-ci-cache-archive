# `Swatinem/rust-cache` Behavior

This page documents the `Swatinem/rust-cache@v2` behavior that matters to the
approaches in this archive. It is not a replacement for the upstream input
reference; it explains how selected inputs affect restored and saved Cargo
state.

## Restore And Save Are Different

`Swatinem/rust-cache` restores the configured Cargo home and target cache
paths, but it does not save an untouched copy of them. Before saving, it cleans
the registry, Git cache, Cargo binaries, and target directories.

For target profiles, the current `v2` cleanup removes profile-root files and
keeps package-matching entries under:

```text
target/<profile>/build/
target/<profile>/.fingerprint/
target/<profile>/deps/
```

This is dependency-oriented caching, not a complete target snapshot.

## Relevant Inputs

| Input | Default | `false` | `true` |
| --- | --- | --- | --- |
| [`cache-all-crates`](https://github.com/Swatinem/rust-cache/blob/v2/action.yml#L39-L42) | `false` | Clean the Cargo registry down to crates selected from the current workspace dependency graph. | Skip registry crate archive and extracted-source cleanup, retaining crates beyond the current dependency graph. |
| [`cache-bin`](https://github.com/Swatinem/rust-cache/blob/v2/action.yml#L55-L58) | `true` | Do not add `$CARGO_HOME/bin` and Cargo's installed-crate metadata files to the cache paths. | Cache binaries tracked by Cargo's installed-crate metadata, subject to binary cleanup before save. |
| [`cache-targets`](https://github.com/Swatinem/rust-cache/blob/v2/action.yml#L32-L35) | `true` | Cache Cargo home state without configured workspace target directories. | Add configured target directories to the cache paths; clean them before save. |
| [`cache-workspace-crates`](https://github.com/Swatinem/rust-cache/blob/v2/action.yml#L43-L46) | `false` | Exclude workspace members from the target-cleanup allowlist; retain dependency package artifacts. | Add workspace members to the allowlist so matching workspace target artifacts also survive cleanup. |

`cache-all-crates` affects Cargo registry cleanup.
`cache-workspace-crates` affects target artifact cleanup. They do not enable
the same behavior.

## `cache-workspace-crates` Example

Suppose the repository is:

```text
app/
├── Cargo.toml          # workspace members: app and crates/common
├── src/
└── crates/
    └── common/
        ├── Cargo.toml
        └── src/
```

The application depends on a local library and a registry crate:

```toml
[dependencies]
common = { path = "crates/common" }
serde = "1"
```

Cargo normally makes an
[in-tree path dependency a workspace member](https://doc.rust-lang.org/cargo/reference/workspaces.html#the-members-and-exclude-fields),
so `crates/common` is a workspace crate in this example.

With the default `cache-workspace-crates: false`:

```text
serde and other dependencies:
  matching target artifacts are retained

app and crates/common workspace members:
  matching target artifacts are removed during save cleanup
```

With `cache-workspace-crates: true`:

```text
serde and other dependencies:
  matching target artifacts are retained

app and crates/common workspace members:
  matching target artifacts are also retained
```

The flag follows workspace membership, not `path = ...` by itself:

- A path dependency inside the workspace normally becomes a member and is
  covered when the flag is `true`.
- A path dependency outside the workspace root is already treated as a
  dependency and does not need this flag.
- A path crate explicitly excluded from the workspace is not covered merely
  because its dependency uses `path = ...`.

For retained packages, `v2` keeps matching entries under profile `build/`,
`.fingerprint/`, and `deps/` for package names and library/proc-macro target
names. It does not preserve the entire target tree or profile-root binaries.

## Choosing Values

For the recommended mtime-preserving checkout approach:

```yaml
cache-targets: true
cache-workspace-crates: true
```

- Keep `cache-all-crates` at `false` unless the workflow downloads registry crates
  outside the workspace dependency graph, such as a tool compiled through
  `cargo install` or an install action's source-build fallback.
- Set `cache-bin` according to whether the workflow has Cargo-registered
  installed tools to preserve.
- Keep `cache-targets: true` because the approach needs target metadata and
  artifacts in addition to stable source mtimes. It is already the default,
  but writing it explicitly makes the architecture clear.
- Use `cache-workspace-crates: true` when repeated-run reuse of workspace
  library artifacts is desired.

These options do not guarantee a complete or current target snapshot. An exact
cache hit is not replaced during the post step, and the target key does not
include all workspace source contents. This can repeatedly restore stale
workspace artifacts; use the source-keyed target-cache workaround when that is
measurable.

## Tool Example: taiki-e Prebuilt Tools

`taiki-e/install-action` officially supports tools including `cargo-lambda`
and `trunk` through prebuilt GitHub Release archives. It installs tools backed
by Rust crates under `$CARGO_HOME/bin` only when the active `cargo` executable
also comes from that directory. Otherwise it falls back to
`$HOME/.install-action/bin`.

`cache-all-crates` is unrelated to this executable: it controls registry crate
cleanup, not `$CARGO_HOME/bin`.

`cache-bin: true` also does not make these taiki-installed executables reusable
through `rust-cache`. If an executable is placed in `$CARGO_HOME/bin`,
`rust-cache` removes it during save because taiki's release extraction does
not register it in Cargo's `.crates2.json` installation metadata. If taiki
uses its fallback directory, that path is outside the `rust-cache` Cargo-home
paths entirely.

Use explicit `cache-bin: true` when the workflow installs tools through
`cargo install` or another path that updates Cargo's installation metadata.
It is not useful for caching the normal taiki-e release installation of
these tools.

`cache-all-crates: true` becomes relevant only if installation falls back to
compiling the tool from registry sources and the workflow wants to retain all
of those downloaded crate archives and sources.

The install action also does not currently skip a supported tool when a
matching executable is already present. Its upstream idempotent-install issue
remains open, and the maintainer states that the existing check applies only
to `cargo-binstall` itself. Supported `cargo-lambda` and `trunk` versions
proceed through the release download and extraction path.

Official implementation references:

- [Supported tools](https://github.com/taiki-e/install-action/blob/main/TOOLS.md)
- [`cargo-lambda` release manifest](https://github.com/taiki-e/install-action/blob/main/manifests/cargo-lambda.json)
- [`trunk` release manifest](https://github.com/taiki-e/install-action/blob/main/manifests/trunk.json)
- [Supported-tool download path](https://github.com/taiki-e/install-action/blob/7a79fe8c3a13344501c80d99cae481c1c9085912/main.sh#L946-L992)
- [Cargo binary directory selection](https://github.com/taiki-e/install-action/blob/7a79fe8c3a13344501c80d99cae481c1c9085912/main.sh#L616-L639)
- [Open idempotent-install request](https://github.com/taiki-e/install-action/issues/577)
- [`cache-bin` input](https://github.com/Swatinem/rust-cache/blob/v2/action.yml#L55-L58)
- [`cache-bin` path selection](https://github.com/Swatinem/rust-cache/blob/v2/src/config.ts#L272-L280)
- [`cache-bin` save cleanup](https://github.com/Swatinem/rust-cache/blob/v2/src/cleanup.ts#L77-L111)

## Upstream Implementation

The behavior above follows the upstream `v2` implementation:

- [Input definitions](https://github.com/Swatinem/rust-cache/blob/v2/action.yml#L32-L46)
- [Cache-path selection](https://github.com/Swatinem/rust-cache/blob/v2/src/config.ts#L272-L289)
- [Save-time package selection](https://github.com/Swatinem/rust-cache/blob/v2/src/save.ts#L39-L60)
- [Workspace target selection](https://github.com/Swatinem/rust-cache/blob/v2/src/workspace.ts#L6-L38)
- [Target cleanup](https://github.com/Swatinem/rust-cache/blob/v2/src/cleanup.ts#L35-L75)
- [Exact-hit save skip](https://github.com/Swatinem/rust-cache/blob/v2/src/save.ts#L24-L28)
