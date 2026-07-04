# GapMiner — Architecture

**Document Type:** Planning / High-Level Architecture
**Source of truth:** `plan_docs/development-plan.md` (§1, §4, §6–§14) and the strategic feasibility document.
**Status:** Planning — describes *what* is built and *how* the components interact. No implementation here.
**Date:** 2026-07-04

---

## 1. System Purpose

The Gap Mining Platform is an **internal intelligence engine** — not a customer-facing product
(`development-plan.md` §1). Its mission is a single, automated pipeline:

1. **Scrape** 1-to-3-star reviews of competitor apps from digital marketplaces
   (Shopify App Store, Chrome Web Store, G2, Apple App Store).
2. **Analyze / Cluster** those negative reviews with LLMs to identify *substantial, monetizable
   feature gaps* (missing workflows — not bug fixes or cosmetic tweaks).
3. **Surface** ranked opportunities via an internal Blazor dashboard for human strategic review.

The platform does **not** build, ship, or monetize micro-SaaS applications; that is a downstream
activity governed by the strategic feasibility document.

---

## 2. The 3-Stage Pipeline

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                            STAGE 1 — SCRAPE                                  │
│  Apify actors ──▶ ScraperWorker (ScrapeReviewsJob) ──▶ Review store (Postgres)│
│                                                  └─▶ Redis "analyze" queue   │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                     STAGE 2 — ANALYZE / CLUSTER                              │
│  AIWorker: EmbedReviewsJob ──▶ pgvector embeddings                           │
│           pgvector KNN clustering ──▶ ReviewCluster[]                        │
│           AnalyzeGapsJob (map-reduce LLM) ──▶ FeatureGap store (Postgres)     │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        STAGE 3 — SURFACE                                     │
│  Minimal API (targets, gaps, jobs endpoints) ──▶ Blazor dashboard            │
│  Opportunity Matrix (severity × frequency scatter) ──▶ human review          │
└──────────────────────────────────────────────────────────────────────────────┘
```

- **Stage 1 (Scrape):** `ScrapeTargetCommand` enters via the API or a scheduled trigger →
  `ScraperWorker.ScraperReviewsJob` resolves the Apify actor, runs it synchronously with
  polling, maps dataset items to `Review` entities, and **bulk-inserts idempotently**
  (`ON CONFLICT DO NOTHING`). On success it enqueues `AnalyzeReviewsCommand`.
- **Stage 2 (Analyze/Cluster):** `AIWorker.EmbedReviewsJob` batches unembedded reviews into
  3072-dim vectors; pgvector KNN + C# k-means produces `ReviewCluster`s;
  `AnalyzeGapsJob` runs a **map-reduce over the LLM** (map = extract candidate gaps per
  cluster; reduce = deduplicate + rank into final `FeatureGap` records).
- **Stage 3 (Surface):** The Minimal API exposes paginated/sortable gap and target endpoints;
  the Blazor dashboard visualizes gaps on an Opportunity Matrix for human triage.

---

## 3. Source Projects & Responsibilities

The repository layout (`development-plan.md` §4) contains **9 projects under `src/`** plus a
test suite. Each project owns a single responsibility:

| # | Project | Layer | Responsibility |
|---|---|---|---|
| 1 | `GapMiner.AppHost` | Orchestration | Aspire host: declares Postgres (+pgvector), Redis, and project references; launches the dashboard. (`§T-0.2`) |
| 2 | `GapMiner.ServiceDefaults` | Cross-cutting | OpenTelemetry tracing/logging, health checks (`/health`, `/alive`); consumed via `builder.AddServiceDefaults()`. (`§T-0.3`) |
| 3 | `GapMiner.Domain` | Domain | Pure entities (`CompetitorTarget`, `Review`, `FeatureGap`), `SeverityScore` value object, `MarketplaceKind` enum. **No EF references.** (`§T-1.1`) |
| 4 | `GapMiner.Infrastructure` | Infrastructure | EF Core (`GapMinerDbContext`, configurations, migrations, repositories), Redis queue, Apify Refit client, SemanticKernel analyzer + prompt files. (`§4`) |
| 5 | `GapMiner.Application` | Application | CQRS command/query handlers (`IngestTarget`, `RunScraper`, `AnalyzeReviews`, `GetFeatureGaps`) and DTOs. (`§4`) |
| 6 | `GapMiner.Api` | Presentation | Minimal API gateway: `/api/v1/targets`, `/api/v1/gaps`, `/api/v1/jobs`, health, Swagger. (`§T-4.1`) |
| 7 | `GapMiner.ScraperWorker` | Background | Hangfire host executing `ScrapeReviewsJob`. (`§T-2.3`) |
| 8 | `GapMiner.AIWorker` | Background | Hangfire host executing `EmbedReviewsJob` and `AnalyzeGapsJob`. (`§9`) |
| 9 | `GapMiner.Web` | Presentation | Blazor Interactive Server dashboard: Dashboard, Targets, Jobs, Opportunity Matrix pages. (`§11`) |

**Test projects** (`tests/`): `Domain.Tests`, `Infrastructure.Tests`, `Application.Tests`,
`Api.Tests`, and `Integration.Tests` (end-to-end with Testcontainers — `§T-6.1`).

---

## 4. End-to-End Data Flow

```
[Operator adds target]
        │  POST /api/v1/targets  (Api → Application.IngestTargetHandler)
        ▼
[Redis scrape queue]  gapminer:scrape:pending
        │  ScrapeReviewsJob dequeues (ScraperWorker)
        ▼
[Apify actor run] ── poll/15s, 60min max ──▶ dataset items
        │  map via Marketplace Actor Registry field mapping
        ▼
[Postgres reviews]  BulkInsertIgnoreDuplicates  (ON CONFLICT DO NOTHING)
        │  + update CompetitorTarget.LastScrapedAt
        ▼
[Redis analyze queue]  gapminer:analyze:pending   (AnalyzeReviewsCommand)
        │  AnalyzeGapsJob dequeues (AIWorker)
        ▼
[EmbedReviewsJob] ──▶ text-embedding-3-large ──▶ Review.Embedding vector(3072)
        ▼
[pgvector KNN clustering] ──▶ ReviewCluster[]  (top-5 representative reviews each)
        ▼
[AnalyzeGapsJob — MAP]   per-cluster LLM call (MapReviewsPrompt.txt) ──▶ candidates JSON
        ▼
[AnalyzeGapsJob — REDUCE] dedupe (cosine > 0.8) + LLM (ReduceGapsPrompt.txt) ──▶ ranked gaps
        ▼
[Postgres feature_gaps]  persisted via IFeatureGapRepository
        ▼
[Api GET /api/v1/gaps] ──▶ Blazor Opportunity Matrix (severity × frequency)
```

---

## 5. Key Architectural Patterns

### 5.1 CQRS — Commands & Handlers
`GapMiner.Application` separates write-side **commands** (`IngestTargetCommand`,
`RunScraperCommand`, `AnalyzeReviewsCommand`) from read-side **queries**
(`GetFeatureGaps`). Naming follows `development-plan.md` §5:
`{Verb}{Noun}Command` / `{Verb}{Noun}Handler`.

### 5.2 Repository Pattern
Repositories in `Infrastructure.Persistence.Repositories` encapsulate EF Core access:
- `ICompetitorTargetRepository` — CRUD + `ExistsByUrlAsync`.
- `IReviewRepository` — `BulkInsertIgnoreDuplicatesAsync`, `GetUnembeddedAsync`, `GetByTargetIdAsync`.
- `IFeatureGapRepository` — `AddAsync`, `GetRankedAsync(sortBy, skip, take)`.

All async methods accept `CancellationToken` (rule R4). (`§T-1.3`)

### 5.3 Idempotency — `ON CONFLICT DO NOTHING`
A unique composite index on `Review(CompetitorTargetId, SourceReviewId)` plus
`ON CONFLICT DO NOTHING` bulk-inserts means re-running the same scrape (race condition on
re-scrape, `§14`) **never duplicates reviews**. (`§T-1.2`, `§T-1.3`)

### 5.4 Map-Reduce LLM Analysis
`AnalyzeGapsJob` (`§T-3.4`) is a classic map-reduce over the LLM:
- **Map:** each review cluster → one `MapReviewsPrompt.txt` call → candidate gaps JSON
  (chunked to ≤ 8,000 tokens/call to avoid context overflow, `§14`).
- **Reduce:** aggregate candidates, deduplicate by title cosine similarity > 0.8, one
  `ReduceGapsPrompt.txt` call → final ranked, schema-validated `FeatureGap[]` (max 10).
- Strict JSON schema validation + one retry on invalid output; dead-letter on second failure.

### 5.5 Redis Job Queues
`RedisJobQueue<TCommand>` (`§T-1.4`) implements `IJobQueue<T>` over a Redis List
(FIFO via `RPUSH`/`LPOP`, blocking `BRPOP`-style dequeue). Keys follow
`gapminer:{domain}:{action}` (`§5`), e.g. `gapminer:scrape:pending`,
`gapminer:analyze:pending`. Hangfire sits above for scheduling/retry.

### 5.6 Dead-Letter & Retry
Every job carries `MaxRetryAttempts` (default 3); failed jobs land in a `DeadLetterQueue`
Redis list for human review (`§T-6.3`).

### 5.7 Secrets Hygiene
API keys/tokens (Apify, Azure OpenAI, Anthropic) come **only** from `IConfiguration` /
Aspire secret parameters / environment variables — never `appsettings.json` (rule R5). Serilog
scrubs `*Password*`, `*Token*`, `*Key*` properties (`§14`).

---

## 6. Aspire Orchestration

`GapMiner.AppHost` (`§T-0.2`) is the single entry point for local development:

```csharp
var builder = DistributedApplication.CreateBuilder(args);
var postgres = builder.AddPostgres("postgres").WithDataVolume().AddDatabase("gapminer");
var redis   = builder.AddRedis("redis");
builder.AddProject<Projects.GapMiner_Api>("api")           .WithReference(postgres).WithReference(redis);
builder.AddProject<Projects.GapMiner_ScraperWorker>("scraper-worker").WithReference(postgres).WithReference(redis);
builder.AddProject<Projects.GapMiner_AIWorker>("ai-worker")        .WithReference(postgres).WithReference(redis);
builder.AddProject<Projects.GapMiner_Web>("web")           .WithReference(api);
builder.Build().Run();
```

- Postgres is configured with `HasPostgresExtension("vector")` and the `pgvector/pgvector:pg16`
  image so EF migrations find the extension (`§14` risk mitigation).
- Connection strings flow into downstream projects via `WithReference(...)` and are read through
  `IConfiguration`.
- `GapMiner.ServiceDefaults` adds OpenTelemetry + health checks uniformly; the Aspire dashboard
  (port ~18888) shows all traces, metrics, and resource states.
- `ApplyMigrations` runs migrations at AppHost startup in dev only (`§T-1.2`).

---

## 7. Domain Model Summary

| Entity / VO | Key fields | Constraints |
|---|---|---|
| `CompetitorTarget` | `Id`, `Name`, `MarketplaceKind`, `MarketplaceUrl`, `CreatedAt`, `LastScrapedAt` | URL validated per marketplace |
| `Review` | `Id`, `CompetitorTargetId`, `StarRating` (1–5), `ReviewText`, `ReviewAuthor`, `DatePosted`, `SourceReviewId`, `Embedding` | unique (`CompetitorTargetId`,`SourceReviewId`); `Embedding` = `vector(3072)` nullable |
| `FeatureGap` | `Id`, `CompetitorTargetId`, `Title`, `DetailedDescription`, `SeverityScore`, `MentionFrequency`, `SuggestedTechStack`, `ActionableImplementationPlan`, `IdentifiedAt` | index on `SeverityScore DESC, MentionFrequency DESC` |
| `SeverityScore` (VO) | `double` | range [1.0, 10.0] enforced via factory method |
| `MarketplaceKind` (enum) | `ShopifyAppStore`, `ChromeWebStore`, `AppleAppStore`, `G2` | unknown → `MarketplaceNotSupportedException` |

(`development-plan.md` §T-1.1, §T-1.2)

---

## 8. API Surface (Stage 3)

| Method | Route | Purpose |
|---|---|---|
| POST | `/api/v1/targets` | Ingest competitor target; validate URL; enqueue `ScrapeTargetCommand` |
| GET | `/api/v1/targets` | List targets (paginated) |
| GET | `/api/v1/targets/{id}` | Target detail |
| POST | `/api/v1/targets/{id}/reanalyze` | Manually trigger `AnalyzeReviewsCommand` |
| GET | `/api/v1/gaps` | Paginated, sortable (`severity|frequency`), filterable by `targetId` |
| GET | `/api/v1/jobs` | Hangfire job history (last 100) |
| GET | `/health`, `/alive` | Health (via ServiceDefaults) |

Routes are kebab-case plural (`§5`); validation via FluentValidation → uniform `ProblemDetails`
errors; OpenAPI via Swashbuckle (`§T-4.1`).

---

## 9. Prompt Library (Version-Controlled)

Prompts live as `.txt` in `src/GapMiner.Infrastructure/AI/Prompts/` and are loaded at runtime
(`development-plan.md` §13) — **never inline strings** (handoff checklist `§17`):

| File | Used by | Role |
|---|---|---|
| `EmbeddingPrompt.txt` | `EmbedReviewsJob` | Wraps review text for embedding similarity |
| `MapReviewsPrompt.txt` | `AnalyzeGapsJob` (map) | Extract candidate gaps from one cluster → strict JSON |
| `ReduceGapsPrompt.txt` | `AnalyzeGapsJob` (reduce) | Deduplicate + rank + produce actionable plan → strict JSON (max 10) |

Both map/reduce prompts enforce **strict JSON output** validated against the schema in `§T-3.4`.

---

## 10. Observability & Quality Gates

- **Tracing:** OpenTelemetry spans for scrape jobs, LLM calls, embedding calls (`§T-6.2`).
- **Metrics:** `gapminer.reviews.scraped`, `gapminer.gaps.identified`, `gapminer.llm.tokens.consumed`.
- **Logs:** Serilog JSON sink with secret-property scrubbing.
- **Quality gates (`§15` Definition of Done):** zero warnings (`TreatWarningsAsErrors`),
  zero new StyleCop/SonarAnalyzer violations, XML docs on all public APIs, ≥ 80% coverage on
  `Application` and `Infrastructure` layers (`§17`).

---

## 11. Cross-References to `development-plan.md`

| Topic | Section |
|---|---|
| Strategic context | §1 |
| Agent operating rules | §2 |
| Technology stack (original) | §3 |
| Repository layout | §4 |
| Naming conventions | §5 |
| Phase 0 Foundation (T-0.1–T-0.3) | §6 |
| Phase 1 Domain & Data (T-1.1–T-1.4) | §7 |
| Phase 2 Scraper Pipeline (T-2.1–T-2.3) | §8 |
| Phase 3 Intelligence Pipeline (T-3.1–T-3.4) | §9 |
| Phase 4 API Gateway (T-4.1) | §10 |
| Phase 5 Blazor Dashboard (T-5.1–T-5.4) | §11 |
| Phase 6 Integration & Hardening (T-6.1–T-6.3) | §12 |
| Prompt library | §13 |
| Known risks & mitigations | §14 |
| Definition of Done | §15 |
| Parallel execution map | §16 |

> Runtime/SDK versioning (.NET 10 reconciliation) is documented in `plan_docs/tech-stack.md` §9.
