---
name: feature-implementation
description: Use when implementing an approved Xplor Importer feature or user story across Core, Application, Infrastructure, Api, or BackgroundJobs, including endpoint behavior, durable job dispatch, orchestration, validation, storage adapters, and focused tests.
---

# Feature Implementation

## Goal

Deliver the smallest complete, tested feature slice that conforms to `docs/SYSTEM_ARCHITECTURE.md` and does not silently settle unresolved migration decisions.

## Preconditions

- Read `CLAUDE.md` and `docs/SYSTEM_ARCHITECTURE.md`.
- Identify acceptance criteria and the affected run-state transition.
- Confirm any required source contract, OWNA mapping, centre mapping, and security decision is approved.
- Stop before implementation when a required decision or destination field contract is missing.

## Workflow

1. Locate the owning use case and nearest existing pattern.
2. State the intended behavior, layer ownership, and one focused falsification check.
3. Add or update contracts without leaking API, CSV, or MongoDB models across boundaries.
4. Implement business behavior in `Core` or `Application` and external concerns behind Application ports.
5. Keep request validation, authorization, HTTP translation, and enqueueing in `Api`.
6. Keep durable work consumption, retries, cancellation, and migration execution in `BackgroundJobs`.
7. Add focused tests for the happy path and material boundary or failure cases.
8. Run the narrowest relevant test, then broader build or test validation when warranted.
9. Review the diff for production-write bypasses, sensitive-data exposure, and unrelated changes.

## Implementation Rules

- Preserve explicit state transitions and idempotency by `runId` and stable source identity.
- Pass `CancellationToken` through asynchronous I/O.
- Return structured failures; do not swallow exceptions or silently coerce invalid data.
- Do not start fire-and-forget tasks from API endpoints.
- Do not let dry-run invoke write ports.
- Do not introduce recurring scheduling, destructive reconciliation, or new production write paths without an architecture decision.
- Prefer focused, reversible changes over speculative frameworks or generic repositories.

## Completion Evidence

- Acceptance criteria are traceable to code and tests.
- Focused tests and the affected project build pass.
- API and worker responsibilities remain separated.
- Any unresolved decision, skipped validation, or residual production risk is reported explicitly.