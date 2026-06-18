# Approaches

This category owns architecture choices, tradeoffs, status, and decisions. Current conclusions are canonical in [Decisions](../decisions/README.md); this page helps pick the right approach.

## Decision Tree

```text
Need a Cargo CI cache?
  Start with Swatinem/rust-cache and an mtime-preserving cached worktree.

Repeated local path workspace members still rebuild on exact cache hits?
  Add the source-keyed full-target cache workaround.

Need maximum full-filesystem no-op fidelity and can own lifecycle complexity?
  Consider the archived EBS/filesystem snapshot approach.

Considering S3 Files for Cargo target no-op state?
  Do not use it for this purpose based on these experiments.
```

Use [mise tool setup](../operations/mise-tool-setup.md) alongside any approach when repeated Rust/Zig/helper-tool setup time matters.

## Decision Matrix

| Approach | Status | Best when | Main tradeoff | Page | Example |
| --- | --- | --- | --- | --- | --- |
| `Swatinem/rust-cache` with mtime-preserving checkout | Recommended Cargo cache default | You want maintained/simple CI that can produce warm Cargo no-op builds. | Affected local path workspace members can still rebuild on exact target-cache hits. | [rust-cache-mtime-checkout.md](rust-cache-mtime-checkout.md) | [workflow](../../examples/workflows/rust-cache-mtime-checkout.yml) |
| `Swatinem/rust-cache` with source-keyed target cache | Proven workaround | Repeated local path workspace-member rebuilds are expensive enough to justify custom cache composition. | More workflow logic and strict restore ordering. | [rust-cache-source-keyed-target-cache.md](rust-cache-source-keyed-target-cache.md) | [workflow](../../examples/workflows/rust-cache-source-keyed-target-cache.yml) |
| EBS snapshot / filesystem snapshot | Archived alternative | You need maximum local no-op fidelity and can own snapshot lifecycle complexity. | Heavier infrastructure, credential scrubbing, and snapshot scoping. | [ebs-snapshot.md](ebs-snapshot.md) | [workflow](../../examples/workflows/ebs-snapshot.yml) |
| S3 Files | Rejected for Cargo target state | You need shared file-system access for another workload. | Cargo target metadata traversal was too slow/variable for these tests. | [s3-files.md](s3-files.md) | [workflow](../../examples/workflows/s3-files.yml) |

## Compatibility Rule

Do not combine `Swatinem/rust-cache` with a full Cargo build-state snapshot for the same `target/` or `$CARGO_HOME` paths. See the [canonical compatibility rule](../concepts/cargo-path-coverage.md#compatibility-rule-canonical) for why.

## Architecture Diagrams

Keep diagrams beside the canonical explanation of the behavior they represent:

- [`rust-cache` with mtime-preserving checkout](rust-cache-mtime-checkout.md#architecture)
- [`rust-cache` with source-keyed full target cache](rust-cache-source-keyed-target-cache.md#architecture)
- [RunsOn Magic Cache backend and job flow](../deployments/runs-on/README.md)
- [Filesystem snapshot lifecycle](ebs-snapshot.md#architecture)
- [S3 Files network-filesystem experiment](s3-files.md#architecture)
- [Cargo freshness decision model](../concepts/cargo-freshness-model.md)
