# Targeted interview questions

Ask only questions the intake phase did not already answer. Ask one question at a time when the answer needs thinking; batch simple yes/no questions. Do not ask all of these in every interview.

## 1. Domain model

- What are the main things (nouns) this feature deals with? (entities, records, objects)
- For each entity: is it newly introduced, or does it already exist elsewhere in the system?
- What is each entity's lifecycle? (created → updated → archived → deleted, or something else)
- What relationships exist between entities? One-to-one, one-to-many, many-to-many?
- Which entity "owns" which? (multi-tenancy, user ownership, workspace scoping)
- Are any entities shared across tenants, or strictly per-tenant?

## 2. Actions

- What verbs describe what the system does? (create, accept, publish, emit, reject, reconcile, ...)
- For each verb: what triggers it, what inputs it takes, what it produces, and what side effects it has (DB writes, messages published, files written)?
- Which actions are synchronous (caller waits) vs asynchronous (fire-and-forget)?
- Which actions are idempotent, which are not, and why?
- Which actions must be authenticated? Which require specific roles or permissions?

## 3. Non-functional requirements

- Expected request rate / message rate / job rate?
- Latency target for user-facing operations? p50 and p95, if known.
- Data volume: rows per day/month, bytes per record?
- Growth assumption: is this going to 10× in a year?
- Availability expectation: single-region, multi-region, zero-downtime upgrades?
- Security posture: does this handle PII, secrets, payments, or regulated data?
- Any specific compliance requirement? (GDPR, HIPAA, SOC2)

## 4. Cross-cutting concerns (check each explicitly)

Ask each row as a one-line question. Valid answers: "applies — <policy>", "does not apply — <reason>", "project default".

- **Authentication:** which authn scheme? (JWT, API key, session, mTLS, none)
- **Authorisation:** role-based, attribute-based, tenant-scoped only, or none?
- **Logging:** structured JSON with correlation ID (project default), or something different?
- **Metrics:** which Prometheus metrics should this emit? Any specific names already decided?
- **Tracing:** OTel spans on the hot path? Any particular sampling rate?
- **Rate limiting:** any endpoint here that should be throttled?
- **Pagination:** which endpoints return lists? Cursor or offset?
- **Input validation:** any field with a non-obvious constraint (max length, regex, forbidden values)?
- **Error handling:** standard envelope (project default) or specialised?
- **Configuration:** which env vars does this introduce? Any secret material?
- **Health checks:** does this add a new dependency (DB, NATS subject, external service) that readiness should reflect?
- **Migrations:** schema changes involved? Destructive (data loss) or additive?
- **Graceful shutdown:** any long-running operation that must drain cleanly?
- **CORS:** any new HTTP endpoint called from the browser directly?
- **Multi-tenancy:** how is tenancy encoded? (subject wildcards, column, schema, row-level security)

## 5. Known unknowns

- What parts of this are you unsure about right now?
- Are there alternative approaches you considered and rejected? Why?
- Is there an earlier attempt at this (code, design doc, prototype) worth looking at?
- Who else needs to weigh in before this design is finalised?

## When to stop interviewing

Stop when:
- Every mandatory template section has enough content to be drafted without inventing facts.
- Every cross-cutting row has an answer (including "does not apply — <reason>").
- Remaining uncertainty has been captured as an Assumption (defaults) or Open Question (architectural).

Do not keep asking to fill in rough edges — the design doc exists to surface uncertainty, not eliminate it.
