# Cargo CI Cache Archive

This repository preserves research and operational notes about making Rust/Cargo builds fast in GitHub Actions CI/CD.

It covers:

- How Cargo decides whether a unit is fresh or dirty.
- Why source mtimes, target fingerprints, dep-info, and build-script outputs matter.
- How `Swatinem/rust-cache` behaves and where it is most useful.
- How cached checkouts can preserve source mtimes and avoid false rebuilds.
- How EBS snapshot-style filesystem restore compares with archive caches.
- Why S3 Files was not a good fit for Cargo target no-op state in these tests.
- A proven but not currently selected workaround using a source-keyed target cache.
- How `mise-action` can make repeated toolchain and tool setup very fast on RunsOn Magic Cache.

## Current Recommendation

Use `mise-action` for Rust-adjacent tool setup, then use `Swatinem/rust-cache` in the normal way, combined with a checkout/worktree strategy that preserves unchanged source file mtimes. If true full Cargo no-op behavior becomes necessary, the source-keyed target-cache workaround is documented and proven.

This is a summary. The [Decisions](docs/decisions/README.md) page is the single source of truth for the archive's conclusions and their status, and the [RunsOn deployment](docs/deployments/runs-on/README.md) documents how the recommended approach is deployed.

## Start Here

| Goal | Entry point |
| --- | --- |
| See the current conclusions and their status | [Decisions](docs/decisions/README.md) |
| Understand the documentation organization | [Documentation Map](docs/README.md) |
| Choose or compare cache approaches | [Approach Comparison](docs/approaches/README.md) |
| Deploy the recommended approach on RunsOn | [RunsOn Magic Cache](docs/deployments/runs-on/README.md) |
| Configure fast CI tool setup | [Mise Tool Setup](docs/operations/mise-tool-setup.md) |
| Understand why Cargo rebuilds | [Cargo Freshness Model](docs/concepts/cargo-freshness-model.md) |
| Check which paths each approach preserves | [Cargo Path Coverage](docs/concepts/cargo-path-coverage.md) |
| Review measured evidence | [Evidence](docs/evidence/README.md) |
| Diagnose a cached build that still recompiles | [Diagnosing Rebuilds](docs/operations/diagnosing-rebuilds.md) |
| Copy or adapt workflow examples | [Examples](examples/README.md) |
| Refresh this archive later | [Maintenance Checklist](docs/operations/maintenance-checklist.md) |

## Core Mental Model

Cargo no-op behavior requires a consistent set of proof artifacts:

```text
same source contents
same source mtimes
same workspace path
same target artifacts
same dep-info files
same fingerprints
same build-script outputs
same registry/git dependency source paths
same toolchain, profile, features, flags, and relevant env
```

If one of these is missing, stale, moved, or newer than expected, Cargo can mark units dirty and rebuild.

## Important Findings

The durable findings behind the recommendation are maintained on the [Decisions](docs/decisions/README.md) page so they have a single source of truth. In short:

- Normal `actions/checkout` rewrites source mtimes and can force false rebuilds; a cached Git worktree is the highest-value low-complexity fix.
- `Swatinem/rust-cache` is excellent for dependency-oriented caching but its target caching is not a full local target snapshot.
- `mise-action` is the preferred setup layer for toolchains and helper tools.
- EBS snapshots give the strongest no-op fidelity at higher operational cost; S3 Files was rejected for Cargo target no-op state.

See [Decisions](docs/decisions/README.md) for the full list, status, and supporting evidence links.
