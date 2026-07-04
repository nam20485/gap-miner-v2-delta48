# GapMiner — Technology Stack

**Document Type:** Planning / Technology Stack Reference
**Source of truth:** `plan_docs/development-plan.md` §3 (original spec) reconciled against the runtime environment.
**Status:** Planning — exact package versions are locked during the `create-project-structure` scaffolding step via `Directory.Packages.props`.
**Date:** 2026-07-04

---

## 1. Runtime & SDK

| Component | Technology | Version (this plan) | Original spec (§3) |
|---|---|---|---|
| Runtime / SDK | .NET SDK | **10.0.x** (devcontainer ships `10.0.102`) | 8.0.x (LTS) — **deviated, see §9** |
| SDK pin | `global.json` | `version: 10.0.100`, `rollForward: latestMinor`, `allowPrerelease: false` | n/a |
| WebAssembly SDK | `Microsoft.NET.Sdk.WebAssembly` | `10.0.100` (already in `global.json`) | n/a |
| Orchestration | .NET Aspire | **9.x** (.NET 10–compatible line) | 8.2.x |

> **Why Aspire 9.x, not 8.2.x:** Aspire 8.2 targets .NET 8. The .NET Aspire 9.x line is the
> first release that supports hosting on .NET 10. The AppHost and ServiceDefaults projects will
> target `net10.0`. The hosting integration packages (`Aspire.Hosting.PostgreSQL`,
> `Aspire.Hosting.Redis`) move to the 9.x line accordingly.

---

## 2. Web & API Layer

| Component | Technology | Version (this plan) | Original spec (§3) |
|---|---|---|---|
| Web UI | Blazor Web App (Interactive Server) | .NET 10 | 8.0 |
| Component library | MudBlazor **or** Radzen.Blazor | Latest stable (selected in Phase 5) | named in §T-5.1 |
| Charts (Opportunity Matrix) | `Blazor-ApexCharts` **or** `AntDesign.Charts` | Latest stable | named in §T-5.4 |
| API | ASP.NET Core Minimal API | .NET 10 | 8.0 |
| OpenAPI / Swagger | Swashbuckle.AspNetCore | Latest .NET 10–compatible | implied by §T-4.1 |

---

## 3. Data Layer

| Component | Technology | Version (this plan) | Original spec (§3) |
|---|---|---|---|
| Database | PostgreSQL | **16.x** (unchanged) | 16.x |
| Vector extension | pgvector | **0.7.x** (unchanged); container `pgvector/pgvector:pg16` | 0.7.x |
| ORM | Entity Framework Core | **10.x** (.NET 10 line) | 8.0.x |
| Postgres EF provider | `Npgsql.EntityFrameworkCore.PostgreSQL` | **10.x** | 8.0.10 |
| pgvector EF provider | `Pgvector.EntityFrameworkCore` | **0.2.x+** (.NET 10–compatible) | 0.2.0 |

> The `Review.Embedding` column is mapped as `vector(3072)` to match the
> `text-embedding-3-large` dimensionality (see `development-plan.md` §T-1.2).

---

## 4. Queue, Cache & Scheduling

| Component | Technology | Version (this plan) | Original spec (§3) |
|---|---|---|---|
| Queue / Cache | Redis | **7.x** (unchanged) via `StackExchange.Redis` | 7.x |
| Job scheduling | Hangfire | **1.8.x** (latest stable on the 1.8 line) | 1.8.x |
| Hangfire storage | Hangfire.PostgreSql **or** in-memory (dev) | Latest .NET 10–compatible | implied |

---

## 5. AI / LLM Stack

| Component | Technology | Version (this plan) | Original spec (§3) |
|---|---|---|---|
| AI orchestration | `Microsoft.SemanticKernel` | **1.3x.x** (.NET 10–compatible) | 1.20.x |
| LLM provider | Azure OpenAI (GPT-4o) **or** Anthropic Claude 3.5 Sonnet | Latest stable (switchable) | same |
| Embeddings | `text-embedding-3-large` (OpenAI) **or** `voyage-3` | Latest stable; 3072-dim | same |
| Prompt storage | Version-controlled `.txt` in `src/GapMiner.Infrastructure/AI/Prompts/` | n/a | §13 |

> **Why SemanticKernel 1.3x.x, not 1.20.x:** SemanticKernel 1.20.x predates first-class
> .NET 10 targeting. The 1.3x line carries the .NET 10 / netstandard compatibility guarantees
> required for `TreatWarningsAsErrors` to compile clean. Exact version pinned in
> `Directory.Packages.props` during scaffolding.

---

## 6. Integrations

| Component | Technology | Version (this plan) | Original spec (§3) |
|---|---|---|---|
| Scraping | Apify API | v2 REST (unchanged) | v2 REST |
| HTTP client | Refit (`Refit.HttpClientFactory`) | **8.x** (.NET 10–compatible) — fall back to 7.x if 8.x proves incompatible during scaffolding | 7.x |
| Resilience | Polly (`Microsoft.Extensions.Http.Polly`) | Latest .NET 10–compatible | implied by §T-2.1 |
| Validation | FluentValidation | **11.x** (latest stable; 12.x if .NET 10–required at lock time) | 11.x |

---

## 7. Testing Stack

| Component | Technology | Version (this plan) | Original spec (§3) |
|---|---|---|---|
| Unit framework | xUnit | Latest stable (.NET 10–compatible) | Latest stable |
| Mocking | NSubstitute | Latest stable | Latest stable |
| Integration containers | Testcontainers (`Testcontainers.PostgreSql`, `Testcontainers.Redis`) | Latest stable | Latest stable |
| Web integration | `WebApplicationFactory` (built-in) | .NET 10 | implied by §T-4.1 |

---

## 8. Code Quality & Observability

| Component | Technology | Version (this plan) | Original spec (§3) |
|---|---|---|---|
| Static analysis | `SonarAnalyzer.CSharp` | Latest stable | Latest stable |
| Style | `StyleCop.Analyzers` | Latest stable | Latest stable |
| Structured logging | Serilog (`Serilog.AspNetCore`, JSON sink) | Latest .NET 10–compatible | §T-6.2 |
| Telemetry | OpenTelemetry (via Aspire ServiceDefaults) | Aspire 9.x –bundled | §T-0.3, §T-6.2 |

### Central Package Management
- `Directory.Packages.props` enables Central Package Management and is the **single source of
  truth** for every package version (`development-plan.md` §T-0.1).
- `Directory.Build.props` enables `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`,
  nullable reference types, and implicit usings.

### Build conventions (from `global.json` + §5)
- File-scoped namespaces, 4-space indentation, CRLF line endings (`.editorconfig`).
- Conventional Commits: `feat(scope): description`.

---

## 9. Deviation From Original Spec — .NET SDK 8 → 10 Reconciliation

**Decision:** The platform targets **.NET SDK 10.0.x / .NET 10** instead of the .NET SDK 8.0.x
specified in `development-plan.md` §3.

**Rationale (first-hand evidence):**

1. **Runtime constraint.** The prebuilt devcontainer image
   (`ghcr.io/nam20485/workflow-orchestration-prebuild/devcontainer:main-latest`) ships
   **.NET SDK 10.0.102** only. There is **no .NET SDK 8 installed** in the execution environment.

2. **Existing pin.** The repository's `global.json` already pins:
   ```json
   { "sdk": { "version": "10.0.100", "rollForward": "latestMinor", "allowPrerelease": false } }
   ```
   `rollForward: latestMinor` means the installed `10.0.102` SDK satisfies the pin. Pinning
   `global.json` to an `8.0.x` SDK would make **every** `dotnet build`, `dotnet test`, and
   `dotnet run` command fail with `SDK not found`, blocking all 16 tasks.

3. **Therefore** the plan adopts .NET 10 as the runtime and selects the .NET 10–compatible line
   of every package that the original spec pinned to a .NET 8 version. The notable migrations:

   | Package (original → .NET 10 line) | Original | This plan |
   |---|---|---|
   | .NET SDK | 8.0.x | **10.0.x** |
   | .NET Aspire | 8.2.x | **9.x** |
   | EF Core | 8.0.x | **10.x** |
   | `Npgsql.EntityFrameworkCore.PostgreSQL` | 8.0.10 | **10.x** |
   | `Microsoft.SemanticKernel` | 1.20.x | **1.3x.x** |
   | Refit | 7.x | **7.x or 8.x** (locked at scaffolding) |

4. **Exact version locking.** Per `development-plan.md` §2.1 rule R2 ("Never invent package
   versions"), the precise final versions are **not** invented here. They will be locked during
   the `create-project-structure` scaffolding step by restoring against the live NuGet feed and
   writing resolved versions into `Directory.Packages.props`. The versions in this document are
   the .NET 10–compatible *lines*; exact patch versions are determined at restore time.

5. **No functional impact.** Every API surface used by the plan (Minimal API, Blazor Interactive
   Server, EF Core migrations, `HasPostgresExtension("vector")`, Hangfire jobs, Redis lists,
   SemanticKernel chat/embedding services) is available and stable on .NET 10.

---

## 10. Summary Table (Quick Reference)

| Area | Technology | Target line |
|---|---|---|
| Runtime | .NET 10 | SDK 10.0.x |
| Orchestration | .NET Aspire | 9.x |
| Web | Blazor Interactive Server | .NET 10 |
| API | ASP.NET Core Minimal API | .NET 10 |
| DB | PostgreSQL 16 + pgvector 0.7 | unchanged |
| ORM | EF Core 10 + Npgsql 10 + pgvector EF | .NET 10 line |
| Queue/Cache | Redis 7 (StackExchange.Redis) | unchanged |
| Jobs | Hangfire 1.8 | unchanged |
| AI | SemanticKernel 1.3x + Azure OpenAI / Claude | .NET 10 line |
| Embeddings | text-embedding-3-large (3072-dim) | unchanged |
| Scraping | Apify v2 + Refit 7/8 | .NET 10 line |
| Validation | FluentValidation 11 | unchanged |
| Testing | xUnit + NSubstitute + Testcontainers | latest |
| Quality | SonarAnalyzer + StyleCop | latest |
| Observability | Serilog + OpenTelemetry (Aspire) | latest |
