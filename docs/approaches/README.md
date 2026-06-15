# Approaches

This category owns architecture choices, tradeoffs, status, and decisions. Individual pages keep implementation details and link to focused evidence records for measured results.

## Decision Matrix

| Approach | Status | Best when | Main tradeoff | Page | Example |
| --- | --- | --- | --- | --- | --- |
| `Swatinem/rust-cache` with mtime-preserving checkout | Recommended Cargo cache default | You want maintained/simple CI that can produce warm Cargo no-op builds. | Affected local path workspace members can still rebuild on exact target-cache hits. | [rust-cache-mtime-checkout.md](rust-cache-mtime-checkout.md) | [workflow](../../examples/workflows/rust-cache-mtime-checkout.yml) |
| `Swatinem/rust-cache` with source-keyed target cache | Proven workaround | Repeated local path workspace-member rebuilds are expensive enough to justify custom cache composition. | More workflow logic and strict restore ordering. | [rust-cache-source-keyed-target-cache.md](rust-cache-source-keyed-target-cache.md) | [workflow](../../examples/workflows/rust-cache-source-keyed-target-cache.yml) |
| EBS snapshot / filesystem snapshot | Archived alternative | You need maximum local no-op fidelity and can own snapshot lifecycle complexity. | Heavier infrastructure, credential scrubbing, and snapshot scoping. | [ebs-snapshot.md](ebs-snapshot.md) | [workflow](../../examples/workflows/ebs-snapshot.yml) |
| S3 Files | Rejected for Cargo target state | You need shared file-system access for another workload. | Cargo target metadata traversal was too slow/variable for these tests. | [s3-files.md](s3-files.md) | [workflow](../../examples/workflows/s3-files.yml) |

## Selection Guidance

- Use `Swatinem/rust-cache` with the mtime-preserving cached worktree checkout as the starting point for Cargo state.
- Use [mise tool setup](../operations/mise-tool-setup.md) alongside any approach when repeated Rust/Zig/helper-tool setup time matters.
- Move to the source-keyed target cache only when affected local path workspace members repeatedly rebuild and the cost is worth the extra workflow code.
- Use filesystem snapshots only when exact Cargo no-op fidelity matters more than operational complexity.
- Do not use S3 Files for Cargo target no-op state based on these experiments; the remaining time was dominated by remote metadata/read behavior.

## Compatibility Rule

Do not combine `Swatinem/rust-cache` with a full Cargo build-state snapshot for the same `target/` or `$CARGO_HOME` paths. See the [canonical compatibility rule](../concepts/cargo-path-coverage.md#compatibility-rule-canonical) for why.

## Architecture Diagrams

Keep diagrams beside the canonical explanation of the behavior they represent:

`mise-action` also uses `actions/cache`, so RunsOn Magic Cache can accelerate tool setup through the same backend. This is complementary to `rust-cache`; it does not replace Cargo freshness state.

- [`rust-cache` with mtime-preserving checkout](rust-cache-mtime-checkout.md#architecture)
- [`rust-cache` with source-keyed full target cache](rust-cache-source-keyed-target-cache.md#architecture)
- [RunsOn Magic Cache backend and job flow](../deployments/runs-on/README.md)
- [Filesystem snapshot lifecycle](ebs-snapshot.md#architecture)
- [S3 Files network-filesystem experiment](s3-files.md#architecture)
- [Cargo freshness decision model](../concepts/cargo-freshness-model.md)

## Page Shape

Approach pages follow this order where applicable: status summary, related files, design/architecture, operational details, strengths, limitations, evidence, and decision.
