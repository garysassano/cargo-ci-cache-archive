# RunsOn Magic Cache

This page maps the repository's recommended Cargo cache approach onto RunsOn. RunsOn Magic Cache supplies an S3-backed implementation of the `actions/cache` protocol; `Swatinem/rust-cache` still decides which Cargo paths are restored, cleaned, and saved.

## Use This Shape

| Layer | Purpose |
| --- | --- |
| RunsOn runner with Magic Cache / S3 backend | Cache transport and storage. |
| Cached worktree | Stable source mtimes. |
| `jdx/mise-action` | Rust, Zig, targets, and helper tools. |
| `Swatinem/rust-cache` | Cargo home and target state. |
| Stable explicit `CARGO_TARGET_DIR` | Consistent build output path. |

Do not add an EBS filesystem snapshot to this design. It is a separate archived approach with different restore and lifecycle semantics.

## Copy The Workflow

Use the complete [RunsOn, mise, and `rust-cache` workflow](../../../examples/workflows/runs-on-mise-rust-cache.yml). Its required order is:

1. Enable `extras=s3-cache` and run `runs-on/action@v2`.
2. Restore and update the cached worktree.
3. Configure registry credentials when required.
4. Run `mise-action` so the build toolchain and helper tools are active.
5. Restore `rust-cache` using the explicit ownership settings from [`rust-cache` behavior](../../concepts/rust-cache-behavior.md).
6. Build with a stable explicit `CARGO_TARGET_DIR`.

## RunsOn-Specific Deltas

RunsOn does not require different `mise-action` or `rust-cache` inputs. [Magic Cache](https://runs-on.com/docs/performance/caching/actions/) transparently replaces the `actions/cache` storage backend, while mise and `rust-cache` retain their normal path selection, keying, cleanup, and save behavior.

| Concern | RunsOn choice | Reason |
| --- | --- | --- |
| Cache backend | Enable `extras=s3-cache` and run `runs-on/action@v2` before any cache step. | Magic Cache redirects the `actions/cache` protocol to the RunsOn S3 backend for the worktree, mise, and `rust-cache` entries. |
| `MISE_DATA_DIR` | Stable job-local path such as `${{ github.workspace }}/.mise`. | `mise-action` caches this directory through `actions/cache`, which Magic Cache backs with S3. |
| `MISE_RUSTUP_HOME` | Directory under `MISE_DATA_DIR`. | Keeps mise-managed rustup toolchains, components, and targets in the same S3-backed mise cache without changing non-mise rustup behavior. |
| `CARGO_TARGET_DIR` | Explicit stable path. | Keeps restored target paths consistent between jobs on ephemeral runners. |
| `CARGO_HOME` | Not under `MISE_DATA_DIR`. | Cargo home can hold registry credentials and is already owned and cleaned by `rust-cache`. |

Keep ownership boundaries strict: declare stable helper tools in mise instead of `cargo install`, and do not let `rust-cache` and mise both own `$CARGO_HOME/bin`.

These settings can produce warm Cargo no-op builds, but they still use dependency-oriented `rust-cache` target cleanup rather than a complete target snapshot. If affected local path workspace members repeatedly rebuild on exact cache hits, use the [source-keyed full-target workaround](../../approaches/rust-cache-source-keyed-target-cache.md).

## Ownership

This page owns only the selected RunsOn deployment: runner setup, Magic Cache/S3 backend assumptions, RunsOn-specific deltas, and the combined workflow shape.

Generic Cargo approach selection stays in [Approaches](../../approaches/README.md), tool setup mechanics stay in [Mise Tool Setup](../../operations/mise-tool-setup.md), `rust-cache` input semantics stay in [`Swatinem/rust-cache` Behavior](../../concepts/rust-cache-behavior.md), and measurements stay under [Evidence](../../evidence/README.md).

The detailed state ownership table and backend/job-flow diagrams are in [RunsOn Magic Cache Details](../../reference/runson-magic-cache-details.md).

## Maintenance

Before changing this platform shape, verify the current RunsOn runner-label syntax, Magic Cache setup, S3 backend behavior, and `runs-on/action` major. Keep those platform-specific assumptions on this page rather than copying them into generic Cargo approach pages.

## Related Pages

- [Quickstart](../../quickstart.md)
- [Decisions](../../decisions/README.md)
- [Recommended cache approach](../../approaches/rust-cache-mtime-checkout.md)
- [Mise Tool Setup](../../operations/mise-tool-setup.md)
- [RunsOn Magic Cache Details](../../reference/runson-magic-cache-details.md)
- [`Swatinem/rust-cache` vs `runs-on/snapshot` evidence](../../evidence/rust-cache-vs-snapshot.md)
- [Observed RunsOn cache object shape](../../evidence/rust-cache-vs-snapshot.md#magic-cache-object-shape)
