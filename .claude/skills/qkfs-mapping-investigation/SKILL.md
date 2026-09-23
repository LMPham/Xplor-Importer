---
name: qkfs-mapping-investigation
description: Use when comparing Xplor QKFS tables with OWNA Kindy Funding, CCS, K4A, QGrants, or other government-program data models, especially before deciding whether to import QKFS fields.
---

# QKFS Mapping Investigation

## Goal

Resolve whether and how the seven Xplor QKFS tables map to OWNA's Kindy Funding or CCS-facing models. This is an investigation skill, not permission to guess or write government-reporting data.

## Required comparison

Compare every Xplor field in `QkfsCenter`, `QkfsChild`, `QkfsChildEvidence`, `QkfsProgram`, `QkfsProgramPause`, `QkfsProgramChild`, and `QkfsProgramEducator` against verified OWNA models and consuming code, including:

- identity and centre scope;
- child eligibility and evidence;
- program and enrolment periods;
- pauses and attendance/hours;
- educator or teacher representation;
- reporting/acquittal/forecast semantics;
- source dates, status values, and evidence retention.

## Rules

- Do not equate QKFS with federal CCS without evidence.
- Do not write a QKFS value into a similarly named OWNA field merely because the type matches.
- Separate data that is safe to retain from data that changes government submissions.
- Mark fields as exact match, transformable, lossy, unsupported, or unresolved.
- Capture source provenance and the decision owner for every unresolved field.
- Prefer deferral or quarantine over silent loss for government or eligibility data.

## Deliverables

- Column-by-column comparison matrix.
- Explicit recommendation: import, transform, quarantine, or exclude.
- Test fixtures for accepted and rejected eligibility/status values.
- Follow-up questions for product/compliance owners.
- Approval record before implementation.
