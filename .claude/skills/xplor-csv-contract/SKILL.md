---
name: xplor-csv-contract
description: Use when inspecting an Xplor export ZIP, discovering CSV table schemas, validating headers and encodings, checking row counts or referential integrity, or designing the CSV ingestion boundary.
---

# Xplor CSV Contract

## Goal

Turn an Xplor export ZIP into a validated, versioned input contract before any domain mapping or MongoDB write.

## Workflow

1. Inventory every ZIP entry and identify the table name using an explicit naming rule.
2. Capture encoding, delimiter, quoting, header names, duplicate headers, blank headers, and row counts.
3. Compare observed tables and headers with the approved source-contract artifact for the importer version. If no approved contract exists, produce an observed inventory and stop before mapping.
4. Classify tables as required, optional, unsupported, or unknown. Unknown tables must be reported, not silently ignored.
5. Validate primary-key uniqueness and foreign-key references for each table where the source contract supports them.
6. Detect duplicate rows, malformed dates, invalid booleans, numeric overflow, empty identifiers, and inconsistent centre identifiers.
7. Emit a machine-readable manifest plus a redacted human report. Include file hashes so a run can be reproduced.

## Rules

- Preserve raw values until a typed validation step has accepted them.
- Do not trim or case-fold identifiers unless the source contract explicitly permits it.
- Keep row numbers and source file names on every validation error.
- Treat missing required tables and unexpected schema changes as a blocked run.
- Treat optional-table absence as a warning only when the mapping decision says it is safe.
- Do not load all CSVs into memory by default; use streaming or bounded reads for large exports.
- Never include raw payment, bank, CRN, or child/guardian values in ordinary logs.

## Deliverables

- Export manifest and table inventory.
- Typed source record contracts or DTO definitions.
- Validation result with errors, warnings, and rejected-row counts.
- Fixture ZIPs or representative CSV fixtures for parser tests.
- Explicit compatibility decision for every observed schema variant.

## Stop conditions

Stop before mapping when table identity, key fields, centre ownership, or encoding cannot be established reliably. Ask for a sample or record an approved assumption instead of guessing.
