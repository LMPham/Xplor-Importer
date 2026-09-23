---
name: unit-testing
description: Use when creating, refining, or reviewing unit tests for Xplor Importer Core or Application behavior, including run-state transitions, validators, deterministic mappings, orchestration, cancellation, reconciliation, and dry-run write exclusion.
---

# Unit Testing

## Goal

Provide fast, deterministic tests that prove business behavior without filesystem, network, queue, clock, or MongoDB dependencies.

## Workflow

1. Read `CLAUDE.md`, `docs/SYSTEM_ARCHITECTURE.md`, and the behavior under test.
2. Identify the observable contract, important invariant, and failure boundary.
3. Follow the repository's existing test framework, naming, assertion, and fixture conventions.
4. Arrange inputs with builders or fixtures only when they reduce meaningful duplication.
5. Exercise behavior through its public API; avoid testing private implementation details.
6. Assert the result, state transition, emitted port call, or absence of a write.
7. Run the narrowest test filter first, then the affected test project.

## Required Coverage by Behavior

- **Run state:** valid transitions, invalid transitions, cancellation, and terminal-state protection.
- **Validation:** accepted boundary values and rejected malformed, missing, unknown, or ambiguous values.
- **Mapping:** deterministic output, stable identity, centre qualification, and unknown-value rejection.
- **Orchestration:** phase order, failure stopping rules, cancellation propagation, and reconciliation totals.
- **Dry-run:** identical parsing and mapping path with no invocation of MongoDB write ports.
- **Job handling:** duplicate delivery is safe, claimed work is not processed concurrently, and retryable versus terminal failures are classified.

## Test Rules

- Use synthetic data with no real child, guardian, payment, CRN, credential, or customer export values.
- Inject time, identifiers, storage, queues, and persistence through explicit abstractions where determinism requires it.
- Mock or fake Application ports, not domain entities or value objects.
- Test one behavior per test and make failures explain the broken contract.
- Do not use delays, live services, production configuration, or order-dependent shared state.
- Move ZIP, CSV, MongoDB, queue-provider, serialization, and HTTP-host checks to the appropriate integration test project.
- A test that only verifies mock setup or implementation call order without business significance is not sufficient coverage.

## Completion Evidence

- New or changed behavior has a happy-path unit test.
- Material invalid input and state boundaries are covered.
- Tests pass repeatedly and do not mutate external state.
- Integration-test gaps are recorded when behavior cannot be proven at unit scope.