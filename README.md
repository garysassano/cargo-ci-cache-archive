# Knowledge: Cargo CI Cache

Agent-oriented reference notes, decisions, evidence, and copyable GitHub Actions examples for Rust/Cargo CI cache behavior.

## Current Answer

Use `jdx/mise-action` for Rust, Zig, and Cargo-distributed helper tools. For Cargo state, use `Swatinem/rust-cache` with workspace-crate caching and an mtime-preserving cached worktree checkout. Add the source-keyed full-target cache only when affected local path workspace members repeatedly rebuild on exact `rust-cache` hits.

The canonical decision record is [Decisions](docs/decisions/README.md). The selected RunsOn implementation is [RunsOn Magic Cache](docs/deployments/runs-on/README.md).

## Choose A Path

| If you are here to... | Start with |
| --- | --- |
| Get the answer fast | [Quickstart](docs/quickstart.md) |
| Copy the selected RunsOn workflow | [RunsOn Magic Cache](docs/deployments/runs-on/README.md) |
| Build a provider-neutral Cargo cache | [`Swatinem/rust-cache` with mtime-preserving checkout](docs/approaches/rust-cache-mtime-checkout.md) |
| Compare cache approaches | [Approaches](docs/approaches/README.md) |
| Diagnose recompilation | [Diagnosing Cargo Rebuilds In CI](docs/operations/diagnosing-rebuilds.md) |
| Understand why this works | [Cargo Freshness Model](docs/concepts/cargo-freshness-model.md) |
| Review the measurements | [Evidence](docs/evidence/README.md) |
| Copy workflow examples | [Examples](examples/README.md) |

## Core Mental Model

Cargo no-op behavior requires mutually consistent proof: same source contents and mtimes, same workspace path, same target artifacts and metadata, same dependency source paths, and the same toolchain/build context.

If one piece is missing, stale, moved, or newer than expected, Cargo can mark units dirty and rebuild.

## Documentation Map

| Section | Owns |
| --- | --- |
| [Decisions](docs/decisions/README.md) | Current conclusions and superseded decisions |
| [Approaches](docs/approaches/README.md) | Approach selection, tradeoffs, and workflow links |
| [Examples](examples/README.md) | Copyable workflow and local-action shapes |
| [Operations](docs/operations/README.md) | Setup, diagnosis, and maintenance procedures |
| [Concepts](docs/concepts/README.md) | Stable models and cache semantics |
| [Evidence](docs/evidence/README.md) | Test setup, observations, interpretation, and limitations |
| [Reference](docs/reference/README.md) | Dense technical details kept out of first-read pages |
