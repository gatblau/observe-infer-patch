# Component spec template (Phase 3 & 4)

Produce one section per component using this exact structure.

```markdown
### SPEC: <ComponentName>
**File:** `<path/to/file.go>` | **Package:** `<package_name>` | **Phase:** <N> | **Dependencies:** <list>

#### Purpose
<One paragraph. What this component does and why it exists.>

#### Shared Context
<Duplicate the shared types, config keys, and env vars this spec references. Do not cross-link to Phase 2C — copy them in. A reader must implement from this section alone.>

#### Public Interface
<Exact signatures: function names, parameters, return types. For HTTP: method + path + request/response schema. For message consumers: subject pattern + payload schema.>

##### Example
<Concrete request/response or input/output. Real values, not placeholders.>

#### Internal Logic
<Numbered step-by-step. Each step states: what is done, which dependency is called, which errors are possible, what is logged.>

#### Data Model
<DDL for tables owned by this component (CREATE TABLE, indexes, constraints). Include rollback considerations for destructive changes.>

#### Error Table
| Condition | Status | Code | Response Body |
|-----------|--------|------|---------------|
| <e.g. JSON parse failure> | 400 | VAL_ERR | `{"error":"...", "code":"VAL_ERR"}` |
| <e.g. DB unavailable> | 503 | DB_DOWN | `{"error":"...", "code":"DB_DOWN"}` |

Minimum two rows. Include every failure mode the Internal Logic references.

#### Acceptance Criteria (Gherkin)
Feature: <ComponentName>

  Scenario: Happy path
    Given ...
    When ...
    Then ...

  Scenario: Edge case — <describe>
    Given ...
    When ...
    Then ...

  Scenario: Error path — <describe>
    Given ...
    When ...
    Then ... returns <status> with code <CODE>

Minimum three scenarios: happy, edge, error.

#### Performance, Security, Observability
- **Performance targets:** <p95 latency, throughput, memory ceiling>
- **Security:** <auth required, input validation, PII handling, rate limits>
- **Observability:** <metric names with labels, log fields, trace span names>
```

## Rules when applying this template

- Every section is mandatory. If a section genuinely does not apply (e.g. no DDL for a pure in-memory component), write `N/A — <reason>` rather than omitting.
- Keep each component spec self-contained: a code-generating LLM must be able to implement it with only this section in context.
- Enforce British English throughout.
- No banned phrases (see `banned-phrases.md`).
