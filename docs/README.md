# Documentation

The documentation is organized by the role each page serves. Put information in one category and link to it from the others instead of repeating it.

| Category | Owns | Entry point |
| --- | --- | --- |
| Concepts | Stable explanations of Cargo state, freshness, cache semantics, and path coverage | [Concepts](concepts/README.md) |
| Approaches | Architecture choices, tradeoffs, status, and decisions | [Approaches](approaches/README.md) |
| Operations | Procedures, configuration guidance, diagnosis, and maintenance | [Operations](operations/README.md) |
| Evidence | Test questions, setup, observations, interpretation, and limitations | [Evidence](evidence/README.md) |
| RunsOn | The selected RunsOn deployment that combines the generic approach and operations guidance | [RunsOn Magic Cache](runs-on/README.md) |

## Page Conventions

| Page type | Expected shape |
| --- | --- |
| Concept | Scope, explanation/model, examples, caveats, official references |
| Approach | Status summary, related files, design/architecture, operational details, strengths, limitations, evidence, decision |
| Operation | Purpose, recommended procedure/configuration, ordering, caveats, references |
| Evidence | Question, test setup/progression, observations, interpretation, limitations, implications |

Not every page needs every heading, but pages should preserve this ownership and order where the content applies.
