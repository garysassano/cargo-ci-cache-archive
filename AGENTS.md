# Agent Instructions

This repository archives Rust/Cargo CI cache research, decisions, evidence, and copyable GitHub Actions examples. Optimize edits for accuracy, low duplication, and easy retrieval by other agents.

## Agent Skill

Use `.agents/skills/cargo-ci-cache/SKILL.md` for tasks that apply this archive to a Rust CI workflow, compare cache strategies, diagnose Cargo rebuilds, or modify the local snapshot and S3 Files action examples. The skill is repository-scoped and links back to canonical documents and examples.

For simple repository maintenance, use the routing table below directly. Read only the pages relevant to the task instead of loading the entire archive.

## Canonical Entry Points

| Task | Use |
| --- | --- |
| Understand documentation ownership | `docs/README.md` |
| Choose or compare cache approaches | `docs/approaches/README.md` |
| Apply the selected RunsOn Magic Cache design | `docs/runs-on/README.md` |
| Configure fast CI tool setup | `docs/operations/mise-tool-setup.md` |
| Explain Cargo freshness/no-op behavior | `docs/concepts/cargo-freshness-model.md` |
| Map Cargo state paths to cache coverage | `docs/concepts/cargo-path-coverage.md` |
| Explain cache primitives | `docs/concepts/cache-primitives.md` |
| Explain `Swatinem/rust-cache` inputs and cleanup | `docs/concepts/rust-cache-behavior.md` |
| Diagnose rebuilds | `docs/operations/diagnosing-rebuilds.md` |
| Review measured evidence | `docs/evidence/README.md` |
| Refresh examples and assumptions | `docs/operations/maintenance-checklist.md` |
| Copy workflow shapes | `examples/README.md` and `examples/workflows/` |
| Understand the local snapshot fork | `examples/actions/snapshot/README.md` |
| Understand the S3 Files mount action | `examples/actions/s3-files-mount/action.yml` |

## Current Conclusions

Preserve these conclusions unless new evidence is added to `docs/evidence/`:

- Recommended default: `Swatinem/rust-cache` with an mtime-preserving cached worktree checkout.
- Recommended setup layer on RunsOn: `mise-action` backed by Magic Cache for Rust, Zig, and helper tools.
- Keep the selected RunsOn deployment canonical in `docs/runs-on/README.md`.
- Proven workaround: split Cargo home and full target caching, with a source-keyed target cache restored after `rust-cache`.
- EBS/filesystem snapshots provide the strongest local no-op fidelity, but with higher operational complexity.
- S3 Files was rejected for Cargo target no-op state in these experiments because remote metadata/read behavior dominated even when Cargo was logically clean.
- Do not mix full filesystem snapshots with `rust-cache` on the same `target/` or `$CARGO_HOME` paths.

Treat these as archived conclusions, not timeless upstream facts. Before changing action versions, service behavior, or recommendations that depend on current external behavior, follow `docs/operations/maintenance-checklist.md` and verify the relevant upstream documentation.

## Duplication Rules

- Keep approach selection and tradeoffs in `docs/approaches/README.md`.
- Keep test setup, observations, measurements, interpretation, and limitations in focused pages under `docs/evidence/`.
- Do not maintain a chronological experiment log; move durable findings into the relevant concept, approach, operation, or evidence page.
- Keep diagnostic procedures in `docs/operations/diagnosing-rebuilds.md`.
- Keep `Swatinem/rust-cache` input and cleanup semantics in `docs/concepts/rust-cache-behavior.md`.
- Keep RunsOn runner, Magic Cache, and S3 backend guidance in `docs/runs-on/README.md`.
- Keep copyable workflow examples in `examples/workflows/`.
- Link to canonical pages instead of repeating long tables or result summaries.

## Page Conventions

- Concept pages explain stable models and semantics, followed by caveats and official references where applicable.
- Approach pages use this order where applicable: status summary, related files, design/architecture, operational details, strengths, limitations, evidence, decision.
- Operation pages contain a purpose, recommended procedure or configuration, ordering, caveats, and references.
- Evidence pages contain a question, test setup or progression, observations, interpretation, limitations, and implications.
- Category `README.md` files use a short ownership statement followed by a `Page | Purpose` table or a decision matrix.

## Markdown Style

- Keep each normal prose paragraph on one source line and let the renderer wrap it to the available width.
- Do not manually hard-wrap prose. Preserve structural line breaks in lists, tables, blockquotes, code fences, Mermaid diagrams, YAML, and front matter.
- Use "with" when naming combinations of cache approaches. Avoid `w/`, `+`, and "plus" as alternate labels for the same relationship.

## Example Maintenance

When editing workflow examples:

- Check current GitHub-owned action majors for `actions/checkout`, `actions/cache`, `actions/upload-artifact`, and `actions/download-artifact`.
- Keep `jdx/mise-action@v4`, `Swatinem/rust-cache@v2`, and `dtolnay/rust-toolchain@stable` unless there is a deliberate reason to change them.
- Preserve source-keyed target cache ordering: restore `rust-cache` first with `cache-targets: false`, then restore the full target cache.
- Preserve the generic nature of examples; do not add app-specific package names, secrets, runner labels, or deployment steps unless a page explicitly documents them as examples.

## Validation

Run these checks after relevant edits:

```bash
git diff --check
actionlint examples/workflows/*.yml
yq eval-all --exit-status 'true' examples/workflows/*.yml examples/actions/*/action.yml
(cd examples/actions/snapshot && go test ./...)
```

If `actionlint`, `yq`, or `go` is unavailable, say so and use a compatible tool declared in `~/.config/mise/config.toml`.

## Scope

This archive documents conclusions and reusable examples. Do not add live deployment credentials, organization-specific runner labels, or private repository details. If information comes from another local repository or opencode session history, summarize it as sanitized evidence and place it in the appropriate canonical page.
