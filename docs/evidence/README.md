# Evidence

This category preserves the test setup, observations, and interpretation behind the repository's recommendations. Evidence pages support decisions; they do not own workflow guidance or repeat complete approach descriptions.

## Pages

| Page | Purpose |
| --- | --- |
| [`rust-cache` vs EBS snapshot](rust-cache-vs-snapshot.md) | Compares restored Cargo state and repeated-build behavior. |
| [Cached worktree and source-keyed target cache](cached-worktree-and-target-cache.md) | Records the source-mtime fix, exact-hit outliers, workaround, and native prototype. |
| [S3 Files experiment](s3-files.md) | Records network-filesystem layouts, timings, setup costs, and interpretation. |

## Evidence Rules

- Record the test shape, observed values, interpretation, and limitations.
- Keep current recommendations and tradeoffs in [`docs/approaches/`](../approaches/README.md).
- Keep procedures in [`docs/operations/`](../operations/README.md).
- Prefer focused evidence records over a chronological experiment diary.

## Page Shape

Evidence pages follow this order: question, test setup or progression, observations, interpretation, limitations, and implications.
