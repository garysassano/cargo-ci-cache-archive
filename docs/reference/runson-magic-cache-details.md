# RunsOn Magic Cache Details

This page preserves the detailed state ownership and backend-flow diagrams behind the [RunsOn Magic Cache deployment](../deployments/runs-on/README.md).

## Cache Ownership

Each layer owns a different kind of state:

| State | Owner | Why |
| --- | --- | --- |
| Source worktree and unchanged source mtimes | `actions/cache` with the cached-worktree checkout | Prevents normal checkout from making unchanged source files appear newer than restored Cargo outputs. |
| Rust toolchain, rustup targets, Zig, and Cargo-distributed helper tools | `jdx/mise-action` | Mise installs and caches tools under `MISE_DATA_DIR`; these are setup state, not Cargo freshness state. |
| Cargo registry and Git dependency state | `Swatinem/rust-cache` | Keeps dependency downloads and sources aligned with the workspace dependency graph. |
| Dependency and workspace-library target state | `Swatinem/rust-cache` | Restores the target metadata and artifacts used by Cargo's freshness checks, subject to `rust-cache` cleanup. |
| Final build output location | Explicit stable `CARGO_TARGET_DIR` | Keeps restored target paths consistent between jobs. |

Keep these ownership boundaries strict. In particular, declare stable helper tools in mise instead of installing them separately with `cargo install`, and do not place `CARGO_HOME` under `MISE_DATA_DIR`.

## Backend Boundary

```mermaid
flowchart LR
    subgraph job[GitHub Actions job on RunsOn]
        worktree[actions/cache worktree archive]
        mise[mise-action tool setup]
        tools[MISE_DATA_DIR and MISE_RUSTUP_HOME]
        rust_cache[Swatinem/rust-cache]
        cargo_home[CARGO_HOME registry and Git state]
        target[Cleaned target subset]
        cargo[Cargo build]

        mise --> tools
        tools --> cargo
        rust_cache --> cargo_home
        rust_cache --> target
        worktree --> cargo
        cargo_home --> cargo
        target --> cargo
    end

    magic[RunsOn Magic Cache sidecar]
    s3[(S3 cache objects)]

    worktree <-->|actions/cache protocol| magic
    mise <-->|actions/cache protocol| magic
    rust_cache <-->|actions/cache protocol| magic
    magic <-->|archive transfer| s3
```

The S3 backend changes cache transport and storage. It does not change cache keys, archive extraction, `rust-cache` cleanup, or exact-hit save behavior.

## Job Sequence

```mermaid
sequenceDiagram
    participant Job as GitHub Actions job
    participant Magic as RunsOn Magic Cache
    participant S3 as S3 cache backend
    participant Cargo as Cargo

    Job->>Magic: Restore cached worktree key
    Magic->>S3: Read worktree archive
    S3-->>Magic: Archive
    Magic-->>Job: Extract prior worktree
    Job->>Job: Checkout new commit in place
    Job->>Job: Configure registry credentials when required

    Job->>Magic: mise-action restore
    Magic->>S3: Read mise tool archive
    S3-->>Magic: Archive
    Magic-->>Job: Restore mise and rustup state
    Job->>Job: Install any missing configured tools
    Job->>Magic: Save mise cache on cache miss
    Magic->>S3: Write mise tool archive

    Job->>Magic: Swatinem/rust-cache restore
    Magic->>S3: Read Cargo cache archive
    S3-->>Magic: Archive
    Magic-->>Job: Restore Cargo home and target subset

    Job->>Cargo: Run Cargo, cargo-lambda, or trunk build
    Cargo-->>Job: Build artifacts

    Job->>Job: rust-cache save cleanup
    Job->>Magic: Save Cargo cache on cache miss
    Magic->>S3: Write Cargo cache archive
    Job->>Magic: Save worktree on exact-key miss
    Magic->>S3: Write worktree archive
```
