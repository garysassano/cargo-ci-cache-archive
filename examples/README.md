# Examples

These examples are intentionally generic. They are meant to preserve the workflow shapes and ordering constraints discovered during the cache experiments, not to be copied without adapting package names, paths, runner labels, credentials, and deployment steps.

## Workflows

| Example | Purpose | Referenced by |
| --- | --- | --- |
| [`rust-cache-mtime-checkout.yml`](workflows/rust-cache-mtime-checkout.yml) | Recommended default: `Swatinem/rust-cache` plus an mtime-preserving cached worktree checkout. | [`docs/approaches/rust-cache-mtime-checkout.md`](../docs/approaches/rust-cache-mtime-checkout.md) |
| [`rust-cache-source-keyed-target-cache.yml`](workflows/rust-cache-source-keyed-target-cache.yml) | Proven workaround: `rust-cache` for Cargo home plus separate source-keyed target cache restored after `rust-cache`. | [`docs/approaches/rust-cache-source-keyed-target-cache.md`](../docs/approaches/rust-cache-source-keyed-target-cache.md) |
| [`ebs-snapshot.yml`](workflows/ebs-snapshot.yml) | EBS/filesystem snapshot layout for high-fidelity Cargo no-op behavior. | [`docs/approaches/ebs-snapshot.md`](../docs/approaches/ebs-snapshot.md) |
| [`cargo-lambda-snapshot-matrix.yml`](workflows/cargo-lambda-snapshot-matrix.yml) | Sanitized RunsOn snapshot fork workflow shape for per-Lambda Cargo build-state snapshots. | [`docs/approaches/ebs-snapshot.md`](../docs/approaches/ebs-snapshot.md) |
| [`s3-files-cargo-target.yml`](workflows/s3-files-cargo-target.yml) | S3 Files target-cache experiment shape. Not recommended for Cargo target no-op state based on these experiments. | [`docs/approaches/s3-files.md`](../docs/approaches/s3-files.md) |
| [`cargo-fingerprint-diagnostics.yml`](workflows/cargo-fingerprint-diagnostics.yml) | Diagnostic workflow for fingerprint and dirty-rebuild investigation. | [`docs/operations/diagnosing-rebuilds.md`](../docs/operations/diagnosing-rebuilds.md) |

## Local Actions

| Example | Purpose |
| --- | --- |
| [`actions/cached-worktree-checkout/action.yml`](actions/cached-worktree-checkout/action.yml) | Generic composite action for checking out into a restored Git worktree without rewriting unchanged file mtimes. |
| [`actions/snapshot/action.yml`](actions/snapshot/action.yml) | Local RunsOn snapshot fork with keyed snapshot streams and smart-save support. |
| [`actions/s3-files-mount/action.yml`](actions/s3-files-mount/action.yml) | Composite action for installing and mounting S3 Files with cached client setup. |
