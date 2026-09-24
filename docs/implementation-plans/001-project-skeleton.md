# Implementation Plan 001: Project Skeleton

## Status

Implemented. This plan covers scaffolding only: solution and project layout, build wiring, launch/debug configuration for Visual Studio and VS Code, and one health-check endpoint. No CSV, MongoDB, queue, mapping, or validation behavior is added in this phase.

Two things were decided/discovered during implementation and are reflected below rather than in a separate changelog:

- The .NET 10 SDK's `dotnet new sln` now defaults to the new XML `.slnx` format instead of classic `.sln`. Asked and confirmed: this project uses **`Xplor.Importer.slnx`**, not the classic `.sln` format the rest of this document originally assumed (all references below have been updated accordingly). The multiple-startup-projects file is named `Xplor.Importer.slnx.launch.json` to match (same mechanism, adjusted suffix).
- `Xplor.Importer.BackgroundJobs` uses `Host.CreateApplicationBuilder` (the generic host), which reads the **`DOTNET_ENVIRONMENT`** variable for environment detection, not `ASPNETCORE_ENVIRONMENT` (that prefix is specific to the ASP.NET Core web host used by `Xplor.Importer.Api`). Using `ASPNETCORE_ENVIRONMENT` in `BackgroundJobs`' `launchSettings.json` was tried first and verified wrong (it logged `Hosting environment: Production`); both `launchSettings.json` and `.vscode/launch.json` for `BackgroundJobs` use `DOTNET_ENVIRONMENT` accordingly.

Read together with `docs/SYSTEM_ARCHITECTURE.md` and `CLAUDE.md`. Where this plan makes a concrete choice on something those documents leave open (target framework, port numbers, host type for `BackgroundJobs`), that choice is called out explicitly under "Decisions" so it can be corrected before implementation, per the project's own rule not to silently resolve open questions in code.

## 1. Scope of this phase

In scope:

- `Xplor.Importer.slnx` and the five projects from `docs/SYSTEM_ARCHITECTURE.md` section 3, plus their five test project counterparts, as empty/near-empty buildable projects wired together per the documented dependency graph.
- Shared MSBuild configuration (`Directory.Build.props`, `Directory.Packages.props`, `global.json`).
- `Xplor.Importer.Api`: minimal ASP.NET Core host with a single `/health` endpoint. Nothing else.
- `Xplor.Importer.BackgroundJobs`: a runnable worker host with a no-op placeholder background service. No queue integration yet.
- Launch configuration so both projects start together under debug in Visual Studio (multiple startup projects) and VS Code (a compound launch configuration), on ports that do not collide with the other OWNA repos on this machine.
- `.gitignore` additions appropriate for a mixed VS/VS Code .NET repo.

Out of scope (do not implement in this phase, even if convenient): ZIP/manifest reading, CSV parsing, MongoDB access, durable queue/work-item dispatch, authentication, OpenAPI beyond the default template, Dockerfiles, CI pipelines. These belong to later, separately-numbered implementation plans once the relevant mapping/decision artifacts referenced in `CLAUDE.md` are approved.

## 2. Decision: plain .NET projects, not .NET Aspire

**Recommendation: do not adopt .NET Aspire for this project. Use plain ASP.NET Core / Worker Service projects, matching `OWNAxInfoCareIntergration`.**

Reasoning:

- The request itself ("the startup projects are the API and BackgroundJobs") describes the classic Visual Studio multiple-startup-project debug model. Aspire replaces that model: you set only the AppHost project as startup, and it fans out to every resource itself, including its own dashboard. Adopting Aspire would mean not doing what was asked.
- Aspire's main value is orchestrating many interdependent services plus fresh infrastructure containers it provisions itself (Postgres, Redis, etc.) with unified telemetry. This project's actual topology in v1 is two processes that both talk to OWNA's *existing* MongoDB (not a new database Aspire would stand up) and share one durable queue whose technology is still an open decision (`SYSTEM_ARCHITECTURE.md` section 11). That is a small, comparatively simple topology; Aspire's orchestration value is low here.
- No sibling OWNA repo uses Aspire. `OWNAxInfoCareIntergration` — the closest architectural precedent (an API host plus a separate Hangfire-hosted worker host, both against MongoDB) — uses plain ASP.NET Core projects with a Docker Compose file for local infrastructure (SQL Server for Hangfire storage) and no AppHost. Matching that keeps this repo consistent with house convention, which `CLAUDE.md` already asks us to prefer.
- Aspire can be revisited later if the system grows enough independent services and infrastructure dependencies that unified orchestration and tracing become worth the added model; that is not a v1 skeleton concern and should not be decided implicitly by scaffolding it now.

## 3. Decision: solution and folder layout

```text
Xplor-Importer/
  Xplor.Importer.slnx
  CLAUDE.md
  README.md
  .editorconfig
  .gitignore
  global.json
  Directory.Build.props
  Directory.Packages.props
  docs/
    SYSTEM_ARCHITECTURE.md
    CSV_IMPORT_PATTERN.md
    implementation-plans/
      001-project-skeleton.md
  src/
    Xplor.Importer.Core/
      Xplor.Importer.Core.csproj
    Xplor.Importer.Application/
      Xplor.Importer.Application.csproj
    Xplor.Importer.Infrastructure/
      Xplor.Importer.Infrastructure.csproj
    Xplor.Importer.Api/
      Xplor.Importer.Api.csproj
      Program.cs
      Properties/launchSettings.json
      appsettings.json
      appsettings.Development.json
    Xplor.Importer.BackgroundJobs/
      Xplor.Importer.BackgroundJobs.csproj
      Program.cs
      Properties/launchSettings.json
      appsettings.json
      appsettings.Development.json
  tests/
    Xplor.Importer.Core.Tests/
    Xplor.Importer.Application.Tests/
    Xplor.Importer.Infrastructure.Tests/
    Xplor.Importer.Api.Tests/
    Xplor.Importer.BackgroundJobs.Tests/
```

Updated from the previous revision of this plan: `Xplor.Importer.slnx` now lives at the **repository root**, as a sibling of both `src/` and `tests/`, rather than inside `src/`. This is a deliberate change from "source code and solution file inside `src`" so that the one solution file sits at the natural common ancestor of both `src/` and `tests/` and can reference every project (source and test) with equally simple relative paths, rather than the test projects needing `..\..\tests\...`-style paths from inside `src/`. It also matches the layout already sketched in `docs/SYSTEM_ARCHITECTURE.md` section 3 exactly.

This also simplifies the earlier note about `Directory.Build.props`/`Directory.Packages.props`/`global.json`/`.editorconfig` placement: they were already going to sit at repo root so that both `src/` and `tests/` inherit them via MSBuild's upward directory search from each `.csproj`; with the solution file now also at repo root, every shared file (solution, build props, package versions, SDK pin, editor config) lives at the same single level, and both `src/` and `tests/` are plain children of it. No project-reference path has to reach "up and back down" to find its neighbor.

## 4. Decision: target framework

Not specified in `docs/SYSTEM_ARCHITECTURE.md`. The two closest sibling repos disagree: `OWNAxInfoCareIntergration` targets **.NET 8**, `OwnaHQ` targets **.NET 9**.

**Decision: .NET 10** (`net10.0`), per explicit direction. This is also the more defensible choice independent of that direction: .NET 9 is a Standard-Term-Support release (~18 months from its November 2024 release, so it is already at or past end of support by the time this project starts), while .NET 10 is a Long-Term-Support release (~3 years of support from its November 2025 release) — the newer choice is also the one with the longer support runway, not just the newer number.

All five `src/` and five `tests/` projects target `net10.0`; `global.json` pins the SDK's major version accordingly (exact patch version to be filled in from whatever SDK is installed at implementation time).

## 5. Decision: `BackgroundJobs` host shape in this phase

`docs/SYSTEM_ARCHITECTURE.md` section 3 describes `BackgroundJobs` as "a continuously running worker, not a scheduler or polling-based sync service," consuming durable work items. It does not require an HTTP surface.

For this skeleton phase: `BackgroundJobs` is a plain **.NET Generic Host Worker Service** (`Microsoft.Extensions.Hosting`, `dotnet new worker` shape) with a single placeholder `BackgroundService` that does nothing but log a heartbeat and respect cancellation. It does **not** get a Kestrel/HTTP listener or a `/health` endpoint in this phase — only `Xplor.Importer.Api` gets the health endpoint, per the request.

A port pair is still reserved for `BackgroundJobs` now (section 7) even though nothing listens on it yet, so that whenever it does need an HTTP surface (a readiness probe, or a Hangfire-style dashboard, depending on how the durable-queue decision in `docs/SYSTEM_ARCHITECTURE.md` section 11 is resolved) the port is already documented and collision-free rather than picked under time pressure later.

## 6. Project reference graph

Matches `docs/SYSTEM_ARCHITECTURE.md` section 3 exactly:

```text
Xplor.Importer.Api             -> Application, Infrastructure
Xplor.Importer.BackgroundJobs  -> Application, Infrastructure
Xplor.Importer.Infrastructure  -> Application, Core
Xplor.Importer.Application     -> Core
Xplor.Importer.Core            -> (none)
```

`Core`, `Application`, and `Infrastructure` are created as empty class libraries with no starter class (delete the template's default `Class1.cs`); an empty C# class library is a valid, buildable project with zero source files, and "nothing implemented yet" should be taken literally here. Their only content in this phase is the `.csproj` and the project references above, proving the dependency graph compiles in the correct direction (a reference from `Core` to anything else, for example, should fail to build if accidentally introduced).

## 7. Decision: dev ports, checked against the other 9 OWNA repos

Every other OWNA repo on this machine was checked for committed development port numbers (`launchSettings.json`, `web.config` self-references, `docker-compose*.yml`). Findings:

| Repo | Project/service | Port(s) | Source |
| --- | --- | --- | --- |
| `OwnaWCF` | (WCF service, IIS Express) | https 44317 | `.csproj` `<IISUrl>` |
| `OwnaHRWCF` | (WCF service, IIS Express) | http 11877 | `.csproj` `<IISUrl>` |
| `OwnaWebsite` | portal-website (IIS Express) | https 44333 | `web.config` self-reference (`OwnaUrl`) |
| `Portal` | portal (IIS Express) | https 44344 | `web.config` self-reference (`PortalUrl`) |
| `Portal` | widgets Vite dev server | https 5173 | `web.config` (`WidgetsDevServer`) |
| `Portal` | referenced SSO authority (not necessarily Portal itself) | 5001 | `web.config` (`SSOAuthority`) — treat as reserved out of caution |
| `OwnaHQ` | `Owna.HQ` (Razor Pages) | http 5019 / https 7248 | `Properties/launchSettings.json` |
| `OwnaHQ` | `Owna.HQ.Api` | http 5057 / https 7101 | `Properties/launchSettings.json` |
| `OwnaHQ` | `Owna.Hangfire` (dashboard/worker) | http 5165 / https 7291 | repo-root `launchSettings.json` |
| `OwnaHQ` | local dev SQL Server (docker) | 1433 | `docker-compose.yml` |
| `OwnaHQ` | local dev MongoDB (docker) | 27017 | `dev/local-environment/docker-compose.yml` |
| `OWNAxInfoCareIntergration` | `OwnaInfoCareSync.Api` | http 5266 / https 7275 (IIS Express: http 13170 / ssl 44327) | `Properties/launchSettings.json` |
| `OWNAxInfoCareIntergration` | `OwnaInfoCareSync.Server` (Blazor + Hangfire) | http 5005 / https 7009 | `Properties/launchSettings.json` |
| `OWNAxInfoCareIntergration` | `OwnaInfoCareSync.Client` (Blazor WASM dev host) | http 5258 / https 7231 | `Properties/launchSettings.json` |
| `OWNAxInfoCareIntergration` | `OwnaInfoCareSync.Mcp` | https 55950 / http 55951 | `Properties/launchSettings.json` |

`OwnaConsole`, `OwnaHRPayroll`, and `OwnaHRConsole` had no committed dev port found in the repo (no `<IISUrl>`, no self-referencing `web.config` entry, no `launchSettings.json`). These are likely legacy WebForms projects where Visual Studio auto-assigns and locally persists an IIS Express port on first run, uncommitted. Treat this as a residual unknown rather than a confirmed non-conflict: if any of these three are ever run at the same time as this project, verify their actual assigned port before trusting the table above.

**Chosen ports for this project** — clear of every port above, and of the well-known defaults (5000/5001, 8080/8081, 1433, 27017, 3306, 5432, 6379) commonly used by other unrelated local tooling:

| Project | HTTP | HTTPS |
| --- | --- | --- |
| `Xplor.Importer.Api` | 5299 | 7299 |
| `Xplor.Importer.BackgroundJobs` (reserved; no listener in this phase) | 5300 | 7300 |

## 8. `Xplor.Importer.Api` contents

- `Program.cs`: minimal hosting model, `WebApplication.CreateBuilder` / `MapGet("/health", ...)` returning `200 OK` with a small JSON body (status, UTC timestamp). No controllers, no authentication, no other endpoints.
- Updated after this plan's initial implementation, by explicit request: `builder.Services.AddOpenApi()` + `Scalar.AspNetCore`'s `MapScalarApiReference()` are wired up, but gated behind `app.Environment.IsDevelopment()` — the OpenAPI document (`/openapi/v1.json`) and the Scalar UI (`/scalar/v1`) only exist in Development; outside Development the same branch instead calls `app.UseHsts()` / `app.UseHttpsRedirection()`. `/health` itself is unconditional in both environments. Verified at runtime: in Development, `/health`, `/openapi/v1.json`, and `/scalar/v1` all return `200`; with `ASPNETCORE_ENVIRONMENT=Production`, `/health` still returns `200` and `/scalar/v1` correctly returns `404`. `launchSettings.json`'s `launchUrl` points at `scalar/v1` accordingly. This is dev-tooling/API documentation, not application business logic, so it doesn't conflict with this phase's "no business logic" scope — but it is a genuine (small) addition beyond what this plan originally scoped out, recorded here rather than silently.
- `Properties/launchSettings.json`: `http` and `https` profiles on the ports in section 7. `ASPNETCORE_ENVIRONMENT=Development`.
- `appsettings.json` / `appsettings.Development.json`: present but empty of anything beyond default logging configuration — no MongoDB connection string, no queue configuration yet.

## 9. `Xplor.Importer.BackgroundJobs` contents

- `Program.cs`: `Host.CreateApplicationBuilder`, one registered `BackgroundService` (e.g. `HeartbeatWorker`) that logs "running" on an interval and observes `CancellationToken`, and nothing else.
- `Properties/launchSettings.json`: a single `Project` profile with `DOTNET_ENVIRONMENT=Development` (the generic `Host.CreateApplicationBuilder` host reads `DOTNET_ENVIRONMENT`, not `ASPNETCORE_ENVIRONMENT` — that prefix only applies to the ASP.NET Core web host used by `Api`) and no `applicationUrl` (no HTTP listener in this phase, per section 5).
- `appsettings.json` / `appsettings.Development.json`: present, empty beyond default logging configuration.

## 10. Debug/run configuration

### Visual Studio

- Solution-level "Multiple startup projects" set to start both `Xplor.Importer.Api` and `Xplor.Importer.BackgroundJobs` with action `Start`. Stored in the committed companion file `Xplor.Importer.slnx.launch.json` (Visual Studio's supported mechanism for source-controlled multi-project startup configuration), so it works for every developer who opens the solution rather than only as a local, uncommitted preference.

### VS Code

- `.vscode/launch.json` with one `coreclr` configuration per project (`Api`, `BackgroundJobs`) plus a `compounds` entry (`Launch Api + BackgroundJobs`) that starts both together, mirroring the Visual Studio multi-startup behavior.
- `.vscode/tasks.json` with a single solution-wide `build` task (`dotnet build Xplor.Importer.slnx`) that both `launch.json` entries depend on via `preLaunchTask`.
- Relies on the C# Dev Kit (or OmniSharp) extension being installed; this plan does not add a `.devcontainer` — out of scope unless requested separately.

## 11. `.gitignore` additions

Standard .NET/VS/VS Code entries not already covered by whatever is currently in the repo: `bin/`, `obj/`, `.vs/`, `*.user`, `.vscode/*.log`. Keep `.vscode/launch.json` and `.vscode/tasks.json` tracked (not ignored), since they are shared debug configuration, not personal state.

## 12. Acceptance checklist for this phase

- [x] `dotnet build` succeeds from the repo-root `Xplor.Importer.slnx` with zero warnings under the `Directory.Build.props` warnings-as-errors setting. Verified.
- [x] `dotnet run --project src/Xplor.Importer.Api` starts and `GET http://localhost:5299/health` returns `200 OK` (`{"status":"Healthy","utc":"..."}`). Verified.
- [x] `dotnet run --project src/Xplor.Importer.BackgroundJobs` starts and logs its heartbeat once per second, `Hosting environment: Development`. Verified (stopped via process kill in this environment; VS/VS Code Ctrl+C behavior should be reconfirmed locally, but the host observes `CancellationToken` correctly).
- [x] `dotnet test` succeeds (zero tests) across all five test projects — the template placeholder tests were removed since this phase adds no behavior to test. Verified.
- [ ] Pressing F5 in Visual Studio with the solution's multiple-startup-projects setting launches both processes. Not verified here — no Visual Studio instance in this environment; requires confirmation on a developer machine.
- [ ] Running the VS Code compound launch configuration launches both processes and attaches the debugger to both. Not verified here — requires confirmation with the C# Dev Kit/OmniSharp extension installed.
- [x] No project other than `Xplor.Importer.Api` exposes an HTTP endpoint. Verified by inspection.
- [x] No source file in `Core`, `Application`, or `Infrastructure` beyond the bare `.csproj`. Verified.
- [ ] Confirm actual dev ports for `OwnaConsole`, `OwnaHRPayroll`, and `OwnaHRConsole` before running this project alongside any of them, since no committed port was found for those three (section 7). Still outstanding — not verifiable from repo contents alone.

## 13. Explicitly deferred to later implementation plans

- Any ZIP/CSV/manifest reading (`xplor-csv-contract` skill territory).
- Any MongoDB connectivity or driver setup.
- Durable queue technology selection and wiring between `Api` and `BackgroundJobs` (`docs/SYSTEM_ARCHITECTURE.md` section 11 lists this as an open decision).
- `RunMode`, run-state persistence, or any of the `Core` invariants described in `docs/SYSTEM_ARCHITECTURE.md` section 6.
- Authentication/authorization on the `Api` (section 9 of `docs/SYSTEM_ARCHITECTURE.md` requires this before anything beyond `/health` is added).
- Dockerfiles, CI/CD pipeline definitions.

Each of these should get its own numbered plan under `docs/implementation-plans/` once its prerequisite mapping/decision artifacts exist, per `CLAUDE.md`'s expected workflow.
