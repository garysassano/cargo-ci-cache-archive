# Agent Instructions

This repository archives Rust/Cargo CI cache research, decisions, results, and copyable GitHub Actions examples. Optimize edits for accuracy, low duplication, and easy retrieval by other agents.

## Canonical Entry Points

| Task | Use |
| --- | --- |
| Choose or compare cache approaches | `docs/approaches/README.md` |
| Explain Cargo freshness/no-op behavior | `docs/concepts/cargo-freshness-model.md` |
| Map Cargo state paths to cache coverage | `docs/concepts/cargo-path-coverage.md` |
| Explain cache primitives | `docs/concepts/cache-primitives.md` |
| Diagnose rebuilds | `docs/operations/diagnosing-rebuilds.md` |
| Refresh examples and assumptions | `docs/operations/maintenance-checklist.md` |
| Copy workflow shapes | `examples/README.md` and `examples/workflows/` |

## Current Conclusions

Preserve these conclusions unless new evidence is added to `docs/results/`:

- Recommended default: `Swatinem/rust-cache` plus an mtime-preserving cached worktree checkout.
- Proven workaround: split Cargo home and full target caching, with a source-keyed target cache restored after `rust-cache`.
- EBS/filesystem snapshots provide the strongest local no-op fidelity, but with higher operational complexity.
- S3 Files was rejected for Cargo target no-op state in these experiments because remote metadata/read behavior dominated even when Cargo was logically clean.
- Do not mix full filesystem snapshots with `rust-cache` on the same `target/` or `$CARGO_HOME` paths.

## Duplication Rules

- Keep approach selection and tradeoffs in `docs/approaches/README.md`.
- Keep empirical numbers in `docs/results/empirical-results.md`.
- Keep chronological experiment history in `docs/results/experiment-log.md`.
- Keep diagnostic procedures in `docs/operations/diagnosing-rebuilds.md`.
- Keep copyable workflow examples in `examples/workflows/`.
- Link to canonical pages instead of repeating long tables or result summaries.

## Example Maintenance

When editing workflow examples:

- Check current GitHub-owned action majors for `actions/checkout`, `actions/cache`, `actions/upload-artifact`, and `actions/download-artifact`.
- Keep `Swatinem/rust-cache@v2` and `dtolnay/rust-toolchain@stable` unless there is a deliberate reason to change them.
- Preserve source-keyed target cache ordering: restore `rust-cache` first with `cache-targets: false`, then restore the full target cache.
- Preserve the generic nature of examples; do not add app-specific package names, secrets, runner labels, or deployment steps unless a page explicitly documents them as examples.

## Validation

Run these checks after relevant edits:

```bash
git diff --check
actionlint examples/workflows/*.yml
ruby -e 'require "yaml"; Dir["examples/workflows/*.{yml,yaml}"].each { |f| YAML.load_file(f) }'
```

If `actionlint` is unavailable, say so and still run YAML parsing.

## Scope

This archive documents conclusions and reusable examples. Do not add live deployment credentials, organization-specific runner labels, or private repository details. If information comes from another local repository or opencode session history, summarize it as sanitized evidence and place it in the appropriate canonical page.
