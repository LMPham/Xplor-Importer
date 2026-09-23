---
name: safe-migration-validation
description: Use when designing dry-run mode, migration reports, idempotency checks, row reconciliation, failure recovery, or production-readiness validation for the Xplor importer.
---

# Safe Migration Validation

## Goal

Prove that an import is understandable, repeatable, bounded, and safe before it writes to live OWNA MongoDB.

## Required modes

- **Inspect**: read the ZIP and produce a contract report; no domain writes.
- **Dry-run**: perform parsing, validation, mapping, key resolution, and projected writes; no Mongo mutation.
- **Import**: write only after explicit approval and a passing dry-run report.

The mode must be explicit and fail closed. Never infer import mode from environment defaults.

## Required checks

- source file hashes and centre mapping are recorded;
- required tables and headers match the approved contract;
- source and accepted/rejected counts reconcile;
- foreign-key failures and ambiguous matches are visible;
- projected destination identity keys are unique;
- rerunning the same input produces no new identities;
- partial failure behavior is defined per phase;
- no unexpected collection, delete, deactivate, or non-Xplor write is possible;
- sensitive values are redacted from logs and reports;
- report includes phase counts, warnings, errors, and timings.

## Test scenarios

At minimum cover: empty ZIP, missing table, changed header, duplicate source key, missing reference, duplicate reference, two-centre shared person, malformed date, invalid enum, truncated export, interrupted phase, rerun after partial success, identical rerun, and conflicting existing OWNA record.

## Safety rules

- Do not use production credentials in tests or fixtures.
- Do not run write-mode commands against production during development.
- Use an isolated database or a mocked Mongo abstraction for persistence tests.
- Keep any reconciliation or orphan sweep disabled until explicitly approved.
- A warning is not a pass when it can alter identity, centre ownership, money, attendance, or government reporting.

## Release evidence

A readiness recommendation must link the input manifest, mapping decisions, test results, dry-run report, and approval record. If any of those are absent, report the importer as not ready for production.
