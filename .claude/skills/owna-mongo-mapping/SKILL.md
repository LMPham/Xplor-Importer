---
name: owna-mongo-mapping
description: Use when mapping Xplor entities to OWNA MongoDB collections, defining source keys, designing phased imports, verifying field shapes, or implementing deterministic upsert mappers.
---

# OWNA Mongo Mapping

## Goal

Create an evidence-backed mapping from normalized Xplor tables to OWNA's denormalized MongoDB documents without inventing contracts.

## Evidence hierarchy

1. `SYSTEM_ARCHITECTURE.md` for project boundaries and non-negotiable constraints.
2. Existing OWNA production read/write code and established migration services for destination field contracts.
3. Approved mapping and decision artifacts for this importer.
4. Existing tests and representative, redacted documents.
5. Names, intuition, or generated suggestions are not evidence.

## Mapping procedure

For each entity, document:

- source table and source primary key;
- destination collection and verified field names/types;
- centre scope and destination-centre mapping;
- stable `sourcetype` and `externalid` formula;
- required versus optional fields;
- normalization rules and rejected values;
- relationship dependencies;
- immutable fields using `SetOnInsert`;
- update-owned fields and conflict behavior;
- count and reconciliation rules;
- unresolved assumptions and owner.

## Persistence rules

- Use native MongoDB upserts or bulk writes consistent with the existing OWNA migration service.
- Make identity filters include `sourcetype` and `externalid`; never match only on mutable names or email.
- Use centre-qualified external IDs whenever the same source entity can be represented separately per centre.
- Preserve generated IDs, PINs, passwords, and import timestamps with insert-only semantics where required.
- Make reruns converge to the same document and never create duplicates.
- Avoid destructive deletes and orphan deactivation unless a separate approved policy exists.
- Keep side effects explicit. Do not call UI workflows that seed unrelated records unless their behavior has been reviewed.

## Dependency order

Centre group -> centre -> rooms/fees/discounts -> staff -> parents -> children -> relationships/families -> bookings/enrolments -> attendance -> billing/ledger/bond -> approved QKFS/Kindy data.

## Review questions

- Can this mapper run twice with the same input without changing identity or count?
- What happens when a referenced row is missing or duplicated?
- Is the destination field shape verified from OWNA code?
- Does a shared source person require one destination record per centre?
- Could a malformed or truncated export deactivate existing records?
- Are sensitive source fields being copied, logged, or retained intentionally?
