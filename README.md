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
- How `mise-action` can make toolchain and tool installation effectively free on RunsOn Magic Cache.

## Current Recommendation

Use `mise-action` for Rust-adjacent tool setup, then use `Swatinem/rust-cache` in the normal way, combined with a checkout/worktree strategy that preserves unchanged source file mtimes.

This is the pragmatic default because it makes repeated setup of Zig, Rust targets, `cargo-lambda`, `trunk`, and similar tools cheap on RunsOn Magic Cache, relies on maintained upstream behavior, avoids custom target-cache choreography in every workflow, and still gets most matrix jobs into the fast repeated-run path. The selected RunsOn deployment is documented in the [RunsOn guide](docs/runs-on/README.md). If true full Cargo no-op behavior becomes necessary, the source-keyed target-cache workaround is documented and proven.

## Start Here

| Goal | Entry point |
| --- | --- |
| Choose or compare cache approaches | [Approach Comparison](docs/approaches/README.md) |
| Deploy the recommended approach on RunsOn | [RunsOn Magic Cache](docs/runs-on/README.md) |
| Configure fast CI tool setup | [Mise Tool Setup](docs/operations/mise-tool-setup.md) |
| Understand why Cargo rebuilds | [Cargo Freshness Model](docs/concepts/cargo-freshness-model.md) |
| Check which paths each approach preserves | [Cargo Path Coverage](docs/concepts/cargo-path-coverage.md) |
| Review measurements and experiment history | [Empirical Results](docs/results/empirical-results.md), [Experiment Log](docs/results/experiment-log.md) |
| Diagnose a cached build that still recompiles | [Diagnosing Rebuilds](docs/operations/diagnosing-rebuilds.md) |
| Copy or adapt workflow examples | [Examples](examples/README.md) |
| Refresh this archive later | [Maintenance Checklist](docs/operations/maintenance-checklist.md) |

## Agent Skill

The repository-scoped skill is at
[`.agents/skills/cargo-ci-cache/SKILL.md`](.agents/skills/cargo-ci-cache/SKILL.md).
Agents that discover `.agents/skills/` can use it to select relevant research,
diagnostics, workflow examples, and local action references without loading the
entire archive.

The skill intentionally links to canonical repository files instead of copying
them into the skill directory. Consume it from this repository checkout;
copying only the skill folder will omit those references.

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

- Normal `actions/checkout` can make Cargo rebuild local workspace crates because it rewrites source mtimes.
- Preserving source mtimes with a cached Git worktree is the highest-value low-complexity fix.
- `Swatinem/rust-cache` is excellent for dependency-oriented caching and Cargo home restoration.
- `mise-action` is the preferred setup layer for Rust toolchains, targets, Zig, and Cargo-installed helper tools when backed by RunsOn Magic Cache.
- `Swatinem/rust-cache` target caching is not equivalent to a full local target snapshot.
- `cache-workspace-crates: true` helps, but exact cache hits can still restore stale workspace target state because source contents are not part of the target key.
- EBS snapshots can reproduce local no-op behavior most faithfully, but operational overhead and workflow complexity are higher.
- S3 Files is not a good fit for Cargo target no-op state because metadata traversal and read latency dominate even when Cargo is logically clean.
