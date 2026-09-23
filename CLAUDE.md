# Xplor Importer

## Mission

Build a repeatable, one-off CSV-driven importer that migrates an Xplor Education export ZIP into OWNA's MongoDB. The importer must be safe to rerun for a centre, deterministic, auditable, and usable in dry-run mode before any write.

[SYSTEM_ARCHITECTURE.md](SYSTEM_ARCHITECTURE.md) is the sole project source of truth for scope, solution boundaries, dependency rules, runtime responsibilities, data practices, persistence constraints, security, and testing standards. When code or another document conflicts with it, stop and resolve the conflict before implementation.

## Working rules

- Read `SYSTEM_ARCHITECTURE.md` before planning, implementing, testing, or reviewing a change.
- Do not implement an affected slice until the relevant source CSV samples, OWNA consuming code, and unresolved assumptions have been reviewed and recorded in an approved decision or mapping artifact.
- Prefer the existing OWNA migration conventions from `OWNAxInfoCareIntergration`: `sourcetype`, `externalid`, composite source keys, native Mongo upserts, `SetOnInsert` for immutable fields, phased writes, migration audit records, and orphan-sweep protection.
- Never invent an OWNA field shape from a name alone. Verify it from nearby OWNA read/write code or an approved mapping artifact.
- Keep transformation logic deterministic. AI may help draft mapper skeletons or identify anomalies, but it must not decide production values, silently normalize ambiguous data, or replace validation.
- Default to dry-run. Production MongoDB writes require an explicit run mode, centre mapping, input manifest, validation report, and human approval.
- Never log credentials, connection strings, payment card data, bank details, CRNs, or unnecessary child/guardian personal data.
- Keep the API request-bound and the migration execution in `Xplor.Importer.BackgroundJobs`; never run migration phases or fire-and-forget tasks from an endpoint.
- Do not add sync behavior, UI, recurring scheduling, destructive reconciliation, or unrelated cleanup unless `SYSTEM_ARCHITECTURE.md` is explicitly updated.

## Required design constraints

1. Read the ZIP as a manifest of named CSV tables; validate required tables, headers, encoding, delimiters, row counts, duplicate keys, and referential integrity before mapping.
2. Separate wire CSV records, validated domain records, OWNA mapping, and persistence. Do not let CSV parsing details leak into Mongo writes.
3. Use an explicit Xplor-centre to OWNA-centre mapping. Never infer a destination centre from a child, email address, or free text.
4. Stamp imported records with `sourcetype = Xplor` and a stable `externalid`. Use centre-qualified keys where OWNA records are centre-scoped.
5. Preserve dependency order: centre group, centre, rooms/fees/discounts, staff, parents, children, relationships/families, bookings/enrolments, attendance, billing/ledger/bond, then approved QKFS/Kindy data.
6. Treat missing source rows as non-destructive by default. Do not deactivate or delete records without an explicit, separately approved reconciliation policy.
7. Produce row-count reconciliation, rejected-row details, warnings, entity counts, timings, and a human-readable run summary for every dry-run and import.
8. Treat payment and government-identification fields as sensitive. Use least privilege and redacted diagnostics.

## Non-negotiable open decisions

Do not silently resolve these in code:

- staff matching across multiple centres;
- QKFS to OWNA Kindy Funding column mapping;
- PrimaryCarerChangeHistory retention;
- BookingPatternProposal/BookingPatternCreation retention (large approval/versioning history tables with no matching OWNA collection; decide full-history vs. current-state-only);
- v1 table scope, especially AuditLog, SuperAdmin, and Xplorer;
- polymorphic created/modified-by links;
- centre mapping and handling of records with no destination centre.

Record decisions in a project artifact before implementation.

## Expected workflow

1. Read `SYSTEM_ARCHITECTURE.md`, then inspect the source export samples and relevant OWNA reference code for the requested slice.
2. Write or update a mapping/decision artifact before implementing a new entity.
3. Define acceptance criteria and the smallest focused validation that can disprove the implementation.
4. Add parser and validation tests before persistence tests.
5. Implement one dependency slice at a time with dry-run coverage.
6. Validate idempotency, rerun behavior, partial failures, sensitive-data handling, and row reconciliation.
7. Review the diff for accidental production-write paths, API-hosted migration work, and scope creep.

## Project skills

Use the local skills when their trigger matches:

- `xplor-csv-contract`: inspect and validate export ZIP/table contracts.
- `owna-mongo-mapping`: establish evidence-backed Xplor-to-OWNA mappings and dependency order.
- `safe-migration-validation`: design dry-run, idempotency, reconciliation, and safety checks.
- `qkfs-mapping-investigation`: investigate QKFS versus OWNA Kindy Funding without guessing.
- `feature-implementation`: implement an approved feature slice across the solution with focused validation.
- `unit-testing`: add or refine deterministic unit tests for Core and Application behavior.

## Definition of done for importer work

- The mapping and assumptions are documented.
- Dry-run behavior is testable and does not write to MongoDB.
- Every write path has stable source identity and idempotent upsert semantics.
- Invalid and ambiguous rows are rejected or quarantined with reasons.
- Counts and warnings reconcile to source input.
- Tests cover reruns, duplicate source rows, missing references, partial failures, and sensitive fields.
- No claim of production readiness is made without evidence and explicit approval.
