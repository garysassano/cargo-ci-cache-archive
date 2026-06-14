# Documentation

The documentation is organized by the role each page serves. Put information in one category and link to it from the others instead of repeating it.

## Decisions

Start here for the archive's conclusions and their current status.

| Page | Owns | Entry point |
| --- | --- | --- |
| Decisions | The single source of truth for current conclusions and how to change them | [Decisions](decisions/README.md) |
| Decision History | Superseded or revised conclusions and why they changed | [History](decisions/history.md) |

## Documentation Roles

Each role owns one kind of content. These four categories are the stable taxonomy of the archive.

| Category | Owns | Entry point |
| --- | --- | --- |
| Concepts | Stable explanations of Cargo state, freshness, cache semantics, and path coverage | [Concepts](concepts/README.md) |
| Approaches | Architecture choices, tradeoffs, status, and decisions | [Approaches](approaches/README.md) |
| Operations | Procedures, configuration guidance, diagnosis, and maintenance | [Operations](operations/README.md) |
| Evidence | Test questions, setup, observations, interpretation, and limitations | [Evidence](evidence/README.md) |

## Deployments

Deployments are concrete platform realizations of the approaches above. They are not a documentation role; they compose the generic approach and operations guidance for a specific platform and link back to it rather than restating it.

| Deployment | Owns | Entry point |
| --- | --- | --- |
| RunsOn Magic Cache | The selected RunsOn runner, Magic Cache, S3 backend, and combined workflow shape | [RunsOn Magic Cache](deployments/runs-on/README.md) |

## Page Conventions

| Page type | Expected shape |
| --- | --- |
| Concept | Scope, explanation/model, examples, caveats, official references |
| Approach | Status summary, related files, design/architecture, operational details, strengths, limitations, evidence, decision |
| Operation | Purpose, recommended procedure/configuration, ordering, caveats, references |
| Evidence | Question, test setup/progression, observations, interpretation, limitations, implications |
| Deployment | Ownership statement, platform-specific deltas, workflow shape, maintenance, related pages |

Not every page needs every heading, but pages should preserve this ownership and order where the content applies.
