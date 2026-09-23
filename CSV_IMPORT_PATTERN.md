# CSV Import Pattern

## Purpose

This document defines the CSV parsing, validation, transformation, and persistence pattern for the Xplor importer. Read it together with `SYSTEM_ARCHITECTURE.md`.

The pattern is:

```text
CSV file
  -> shared CSV reader
  -> typed source row
  -> table-specific validation and transformation
  -> validated domain/destination contract
  -> persistence adapter
```

The parser mechanics are shared. The source model, CSV mapping, validation rules, transformation, and persistence behavior remain explicit per table.

## Design Summary

The implementation uses:

- one generic `ReadCsv<T>` method;
- one typed source model for each CSV file;
- one CsvHelper `ClassMap<T>` for each file;
- one migration method for each file;
- a transformation object for business normalization and lookup resolution;
- a domain factory or mapper for destination-contract creation.

It does **not** use the AutoMapper object-mapping library. The term `AutoMap` refers to CsvHelper's `ClassMap<T>.AutoMap(...)` method.

## 1. Shared CSV Reader

Centralize file reading and record materialization in a generic asynchronous method:

```csharp
private async IAsyncEnumerable<T> ReadCsv<T>(
    string sourceFile,
    ClassMap<T> classMap,
    [EnumeratorCancellation] CancellationToken cancellationToken)
{
    using var reader = new StreamReader(sourceFile);
    using var csv = new CsvHelper.CsvReader(
        reader,
        CultureInfo.InvariantCulture);

    csv.Context.RegisterClassMap(classMap);

    while (await csv.ReadAsync())
    {
        yield return csv.GetRecord<T>();
    }
}
```

This gives every table the same basic behavior for:

- opening the file;
- creating the `CsvReader`;
- registering the table mapping;
- reading asynchronously;
- materializing a typed row.

The reader should remain ignorant of business rules. It should not know how a child, guardian, attendance, fee, or family is mapped to OWNA.

### Manifest boundary

The Xplor reader must sit behind a ZIP manifest boundary. It should additionally validate:

- ZIP entry names and path traversal;
- required and optional tables;
- encoding, delimiter, quoting, and headers;
- duplicate table entries and duplicate headers;
- compressed and uncompressed size limits;
- row counts and source file hashes.

The generic reader should receive a validated table stream or safe stored path, not an arbitrary client filename.

## 2. Typed Source Models

Each supported CSV table should have a wire model that represents the source contract, for example:

```csharp
public sealed record ChildRow(
    string ChildId,
    string? DateOfBirth,
    string? ModifiedAt,
    string? Balance,
    string? CulturalBackground,
    string? GovernmentReference);
```

For Xplor, keep these models separate from:

- validated domain records;
- OWNA destination contracts;
- MongoDB persistence models;
- API request and response DTOs.

This prevents CSV column names and parsing concerns from leaking into MongoDB writes.

## 3. CsvHelper Mapping

### 3.1 Automatic member mapping

Use CsvHelper automatic mapping only when headers and property names are verified to align:

```csharp
public sealed class RoomMap : ClassMap<RoomRow>
{
    public RoomMap()
    {
        AutoMap(CultureInfo.InvariantCulture);
    }
}
```

`AutoMap` is useful for straightforward columns, but it is not a contract decision. It relies on convention and should not be used to silently accept schema drift.

### 3.2 Automatic mapping with explicit overrides

Automatic mapping may be followed by explicit overrides for fields that need special conversion or null handling:

```csharp
public sealed class ChildMap : ClassMap<ChildRow>
{
    public ChildMap()
    {
        AutoMap(CultureInfo.InvariantCulture);

        Map(row => row.DateOfBirth)
            .TypeConverter<DateOnlyStringConverter>();

        Map(row => row.ModifiedAt)
            .TypeConverter<DateTimeStringConverter>();

        Map(row => row.Balance)
            .TypeConverter<NanToNullDecimalConverter>();

        Map(row => row.CulturalBackground)
            .TypeConverterOption.NullValues("NULL");
    }
}
```

This hybrid form is the preferred starting point when the source headers are mostly conventional but several columns have non-standard representations.

### 3.3 When to use explicit mapping

Use explicit `Map(...)` declarations instead of relying on `AutoMap` when:

- the source header differs from the property name;
- the source contains aliases or version-specific headers;
- a field is sensitive or must be deliberately excluded;
- a value needs a verified date, decimal, enum, or sentinel conversion;
- accepting a missing column would be unsafe;
- the same source value could map to multiple destination meanings.

Example:

```csharp
public sealed class ExplicitChildMap : ClassMap<ChildRow>
{
    public ExplicitChildMap()
    {
        Map(row => row.ChildId)
            .Name("ChildId");

        Map(row => row.DateOfBirth)
            .Name("DOB")
            .TypeConverter<StrictDateOnlyConverter>();

        Map(row => row.GovernmentReference)
            .Name("CRN");
    }
}
```

For Xplor, sensitive fields should be explicit even when automatic mapping could read them. This makes retention, redaction, and approved field ownership visible in code review.

## 4. Per-File Migration Methods

Process each file through a separate table processor while reusing the same reader:

```csharp
private async Task ProcessChildrenAsync(
    string sourceFile,
    CancellationToken cancellationToken)
{
    var rows = ReadCsv(
        sourceFile,
        new ChildMap(),
        cancellationToken);

    await foreach (var row in rows
        .WithCancellation(cancellationToken))
    {
        var validation = _childValidator.Validate(row);
        if (!validation.IsValid)
        {
            _report.Reject(row.ChildId, validation.Errors);
            continue;
        }

        var validatedChild =
            ValidatedChild.Create(row, validation.Value);

        var destination =
            _childMapper.Map(validatedChild, _runContext);

        await _childWritePort.UpsertAsync(
            destination,
            cancellationToken);
    }
}
```

The phase orchestrator invokes table processors in a fixed dependency order:

```csharp
await ProcessCentresAsync(
    manifest.GetRequiredTable("Center"),
    cancellationToken);

await ProcessRoomsAsync(
    manifest.GetRequiredTable("Room"),
    cancellationToken);

await ProcessParentsAsync(
    manifest.GetRequiredTable("Guardian"),
    cancellationToken);

await ProcessChildrenAsync(
    manifest.GetRequiredTable("Child"),
    cancellationToken);
```

This is not a separate parser implementation per file. It is a separate processing policy per file using one parser engine.

For Xplor, the same idea should be expressed as dependency phases rather than a large handler with unrelated methods:

```text
phase 1: centre group, centre
phase 2: rooms, fees, discounts
phase 3: staff
phase 4: parents, children
phase 5: relationships, families
phase 6: bookings, enrolments
phase 7: attendance
phase 8: billing, ledger, bond
phase 9: approved QKFS/Kindy data
```

Each table processor should receive a validated manifest/table contract and shared run context. It should not reopen arbitrary ZIP entries or bypass run-level reconciliation.

## 5. Validation and Transformation

Perform business conversion after parsing and validation rather than putting business rules in the CsvHelper map:

```csharp
public sealed class ChildDestinationMapper
{
    public ChildDestination Map(
        ValidatedChild child,
        ImportRunContext context)
    {
        return new ChildDestination(
            ExternalId: BuildExternalId(
                child.SourceChildId,
                context.SourceCentreId),
            SourceType: "Xplor",
            CentreId: context.DestinationCentreId,
            FirstName: child.FirstName,
            LastName: child.LastName,
            DateOfBirth: child.DateOfBirth);
    }
}
```

This is the correct boundary for rules such as:

- source status to destination status;
- preferred-name resolution;
- address and contact normalization;
- lookup resolution;
- source-specific defaults;
- rejected or unresolved business values.

The validated record is then passed to the deterministic destination mapper:

```csharp
var validatedChild = ValidatedChild.Create(row, validation.Value);
var destination = childMapper.Map(validatedChild, runContext);
```

For Xplor, use the equivalent sequence:

```text
wire CSV row
  -> validated source record
  -> deterministic Xplor-to-OWNA transformation
  -> destination contract
  -> collection-specific persistence adapter
```

Do not map directly from `CsvReader.GetRecord<T>()` into a MongoDB document.

## 6. Error Handling and Reconciliation

Record expected row-level failures as redacted rejection details and keep unexpected exceptions distinct:

```csharp
catch (SourceRowException ex)
{
    report.Reject(
        tableName: "Child",
        rowNumber: ex.RowNumber,
        sourceKeyHash: ex.SourceKeyHash,
        errorCode: ex.ErrorCode);

    logger.LogWarning(
        "Rejected source row {RowNumber} from {TableName} with {ErrorCode}",
        ex.RowNumber,
        "Child",
        ex.ErrorCode);
}
```

Validation should reject invalid values explicitly:

```csharp
if (string.IsNullOrWhiteSpace(row.ChildId))
{
    errors.Add("child.source-id.required");
}
```

### Required reconciliation behavior

Xplor must make error handling more explicit and auditable:

- distinguish malformed source rows, unresolved references, ambiguous matches, phase failures, run failures, and cancellation;
- retain source file and row number for every rejected row;
- reconcile total, accepted, rejected, skipped, projected, and written counts;
- fail the run when required table or header validation fails;
- never log raw child, guardian, payment, CRN, or credential values;
- ensure dry-run never invokes write ports;
- preserve the run report after worker restart or partial phase completion.

A row-level `try/catch` must not hide a phase-level failure or turn an unsafe ambiguity into a warning.

## 7. Recommended Xplor Abstractions

Use these architectural boundaries:

```text
Infrastructure
  IZipManifestReader
  ICsvTableReader
  CsvTableMap<T>
  source row records

Application
  ITableValidator<T>
  ITableTransformer<TSource, TValidated>
  IImportPhaseProcessor
  phase orchestration
  reconciliation and report building

Core
  validated records
  stable source identities
  run state and invariants
  destination contracts where appropriate

Infrastructure
  collection-specific Mongo adapters
  run store
  durable queue implementation
  report and file storage
```

Example table processor shape:

```csharp
public interface ITableProcessor<TSource>
{
    Task<TableProcessingResult> ProcessAsync(
        ImportRunContext context,
        IAsyncEnumerable<TSource> rows,
        CancellationToken cancellationToken);
}
```

The processor can use the shared reader and table map, but it owns only table-specific validation and transformation. It should delegate persistence to an explicit collection port.

## 8. Reuse Decision

Follow these practices:

1. shared generic asynchronous CSV reading;
2. typed source model per table;
3. `AutoMap` plus explicit converter overrides where appropriate;
4. one table-specific processing method or processor per source table;
5. separate transformation before domain/persistence mapping;
6. fixed dependency order;
7. cancellation propagation through async enumeration.

Avoid these unsafe variants:

1. arbitrary file-path input instead of a validated ZIP manifest;
2. implicit schema acceptance through `AutoMap` alone;
3. row skipping without complete reconciliation;
4. logging exception messages that may contain source data;
5. direct application-owned dictionaries as the only identity and recovery mechanism;
6. processing that cannot resume safely after worker restart.

The resulting Xplor design is therefore **one reusable CSV engine, explicit table contracts, table-specific processors, deterministic transformations, and durable run-level orchestration**.

## 9. Memory and Batching

Every table uses the same streaming reader (`ReadCsv<T>` / `IAsyncEnumerable<T>`). There is no per-table exception to how a file is opened or read. What differs by table is **retention**: what, if anything, is kept in memory after a row has been read and processed.

### 9.1 Why retention differs, and why that is not an arbitrary per-table guess

An observed export shows the real shape of this problem:

| Table | Size |
| --- | --- |
| LedgerPrimaryCarer | ~2.07 GB |
| AuditLog | ~1.65 GB |
| SessionBooking | ~507 MB |
| BookingPatternProposal | ~305 MB |
| BookingPatternCreation | ~239 MB |
| Attendance | ~124 MB |
| WeeklyBooking | ~65 MB |
| GuardianScheduledPayment | ~18 MB |
| All remaining ~39 tables combined (Centre, Room, Fee, Guardian, Child, Educator, RoomCapacity, every `Qkfs*` table, etc.) | < 6 MB combined |

Roughly 96% of export volume sits in five tables. This is not incidental to one sample — it is structural. A reference/dimension table's size is bounded by how many people, places, or things a business has (children, guardians, rooms), which stays in the thousands even at multi-centre scale. A fact/history table's size is bounded by entity count multiplied by time (days x years x sessions), which is unbounded in principle. The retention rule below follows from that structural difference, not from a per-table size guess that could go stale for a larger customer.

### 9.2 The two retention modes

- **Fact/history tables** (`Attendance`, `SessionBooking`, `LedgerPrimaryCarer`, `AuditLog` if in scope, `WeeklyBooking`, `BookingPatternProposal`, `BookingPatternCreation`, `GuardianScheduledPayment`, and any other table whose row count scales with time rather than headcount): read a row, validate, transform, place it in the current bounded write batch, and retain nothing once that batch has flushed. Never accumulate these tables into a `List<T>` for the whole file.
- **Reference/dimension tables** (`Centre`, `Room`, `Fee`, `Discount`, `Guardian`, `Child`, `Staff`/`Educator`, `RoomCapacity`, the `Qkfs*` tables, and similar): while streaming the table once, build a small retained index for the phase — at minimum a `HashSet<string>` of valid source IDs for foreign-key checks, and a `Dictionary<string, T>` only where a later phase genuinely needs a full-row lookup (for example, denormalizing a room name). This index lives only as long as the phase that depends on it and is proportionate to the table's real size (under a few MB, per the data above).

A fact table is never promoted to the retained-index role, and a reference table never needs bounded-batch treatment on the write side because its total volume does not require it. Do not decide this per table by name; decide it by whether the table's row count is headcount-bound or time-bound.

### 9.3 Why foreign-key validation should use the CSV-derived index, not a live Mongo lookup

An alternative would be validating a fact-table row's foreign keys against OWNA's Mongo collections written by earlier phases (a batched `$in` query per write-batch of distinct referenced IDs), instead of an in-memory CSV-derived index. This was considered and rejected for v1:

- `RunMode.Inspect` must validate the export's internal consistency (headers, PK-before-FK, referential integrity) without depending on a destination database or centre mapping. A Mongo-based check would make `Inspect` require a live OWNA connection it should not need.
- The CSV-derived index is already cheap: reference tables are proven small (9.1), so building it costs negligible memory and avoids extra round-trips and an ordering dependency on a prior phase having fully committed.

Foreign-key validation therefore happens in two layers: source-side (does the FK exist within the export itself, checked via the CSV-derived index, available in every run mode including `Inspect`), and phase-order-side (a later phase only processes rows whose parent phase completed; an unresolved or rejected parent record is not a target for a child row, per `SYSTEM_ARCHITECTURE.md` section 6's field-ownership rules).

### 9.4 Bounded write batching

For fact/history tables, accumulate `WriteModel<TDocument>` upserts into a bounded buffer, flush via `BulkWriteAsync(..., IsOrdered = false)` when the buffer fills or the stream ends, then clear and continue streaming. A starting batch size of 500-1000 is reasonable and should be tunable per entity (smaller for wider documents, larger for narrow ones such as `Attendance`). Keep a parallel list of source-row identities aligned to each batch's write-model order, so a partial `BulkWriteAsync` failure (`MongoBulkWriteException.WriteErrors[i].Index`) can still be attributed back to the source row for the rejection report. Never defer all writes for a fact table until the end of the file; this is the specific pattern to avoid (see 9.5).

### 9.5 What not to copy from `OWNAxInfoCareIntergration`

That system's fetch services (for example `InfoCareDataFetchService.FetchChildrenAndGuardiansAsync`) accumulate an entire business's dataset into `List<T>` before any Mongo write, and its migration services then issue one unbounded `BulkWriteAsync` per entity type at the end. This is safe there only because a single InfoCare/KidSoft business, fetched from a paginated live API, stays small. It is not safe here: a single Xplor export can contain gigabytes in a handful of tables (9.1), so the fetch-everything-then-write-everything shape must not be reused for those tables, even though its `sourcetype`/`externalid` upsert conventions should be.

### 9.6 ZIP entry access

The manifest reader opens the stored ZIP file from disk (never a `MemoryStream` of the whole archive) and, per table, opens a single `ZipArchiveEntry` stream at a time, passed directly into `ReadCsv<T>`'s `StreamReader`. Never call `ReadToEnd()`/buffer a whole entry into a `string` or `byte[]` first.

### 9.7 Testing implication

Infrastructure tests (`SYSTEM_ARCHITECTURE.md` section 10) should include a large-file/streaming category using synthetic multi-hundred-MB fixtures shaped like `LedgerPrimaryCarer` and `SessionBooking`, in addition to small correctness-focused fixtures. Memory-retention bugs typically pass on a 10-row test file and fail only at real volume.
