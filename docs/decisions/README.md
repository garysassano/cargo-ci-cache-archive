# Decisions

This page is the single source of truth for the archive's current conclusions. Other pages, including the root `README.md` and `AGENTS.md`, summarize these decisions briefly and link here instead of restating them. When a conclusion changes, update it here first, record the superseded version in [history](history.md), then adjust the summaries that point here.

Treat these as archived conclusions, not timeless upstream facts. They reflect the evidence under [`docs/evidence/`](../evidence/README.md) and external behavior at the time of testing. Before changing any decision that depends on current action versions or service behavior, follow the [maintenance checklist](../operations/maintenance-checklist.md) and verify the relevant upstream documentation.

## Current Conclusions

| # | Decision | Status | Basis |
| --- | --- | --- | --- |
| D1 | Use `Swatinem/rust-cache` with an mtime-preserving cached worktree checkout as the default Cargo cache approach. | Recommended | [Approach](../approaches/rust-cache-mtime-checkout.md), [evidence](../evidence/cached-worktree-and-target-cache.md) |
| D2 | Use `jdx/mise-action` backed by an `actions/cache` backend as the setup layer for Rust, Zig, and Cargo-distributed helper tools. | Recommended | [Mise tool setup](../operations/mise-tool-setup.md) |
| D3 | Keep the source-keyed full-target cache (split Cargo home and target, target restored after `rust-cache`) as a proven workaround for repeated workspace rebuild outliers. | Proven workaround | [Approach](../approaches/rust-cache-source-keyed-target-cache.md), [evidence](../evidence/cached-worktree-and-target-cache.md) |
| D4 | Treat EBS/filesystem snapshots as the strongest local no-op fidelity option, but as an archived alternative because of operational and lifecycle complexity. | Archived alternative | [Approach](../approaches/ebs-snapshot.md), [evidence](../evidence/rust-cache-vs-snapshot.md) |
| D5 | Do not use S3 Files for Cargo target or registry no-op state; remote metadata/read behavior dominated even when Cargo was logically clean. | Rejected for this use | [Approach](../approaches/s3-files.md), [evidence](../evidence/s3-files.md) |
| D6 | Do not mix a full filesystem snapshot with `rust-cache` on the same `target/` or `$CARGO_HOME` paths. | Compatibility rule | [Approach comparison](../approaches/README.md), [path coverage](../concepts/cargo-path-coverage.md) |

## Detailed Findings

These supporting findings explain why the decisions above hold. Keep the underlying explanation in the linked canonical pages; this section captures only the durable conclusion.

- Normal `actions/checkout` can make Cargo rebuild local workspace crates because it rewrites source mtimes. Preserving source mtimes with a cached Git worktree is the highest-value, low-complexity fix. See [the freshness model](../concepts/cargo-freshness-model.md) and [diagnosing rebuilds](../operations/diagnosing-rebuilds.md).
- `Swatinem/rust-cache` is excellent for dependency-oriented Cargo home and registry caching, but its target caching is not equivalent to a full local target snapshot. `cache-workspace-crates: true` helps, yet exact target-cache hits can still restore stale workspace target state because source contents are not part of the target key. See [`rust-cache` behavior](../concepts/rust-cache-behavior.md).
- `mise-action` accelerates repeated toolchain and helper-tool setup; it does not by itself prove Cargo units fresh. See [mise tool setup](../operations/mise-tool-setup.md).
- Cargo no-op behavior requires a consistent set of proof artifacts (source contents, source mtimes, workspace path, target artifacts, dep-info, fingerprints, build-script outputs, dependency source paths, toolchain, profile, features, flags, and relevant env). If one is missing, stale, moved, or newer than expected, Cargo can mark units dirty. See [the freshness model](../concepts/cargo-freshness-model.md).

## Selected Deployment

The recommended approaches are deployed on RunsOn with Magic Cache. See the [RunsOn deployment](../deployments/runs-on/README.md) for the runner, Magic Cache, S3 backend, and combined workflow shape. Deployment-specific guidance lives there, not in the generic approach pages.

## Changing A Decision

1. Confirm new measured evidence exists under [`docs/evidence/`](../evidence/README.md).
2. Record the prior conclusion in [history](history.md) with what changed and why.
3. Update the affected row or finding above.
4. Update the brief summaries that link here (root `README.md`, `AGENTS.md`, and any approach page status fields).
