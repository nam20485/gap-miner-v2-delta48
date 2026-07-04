# Workflow Execution Plan — project-setup

**Workflow:** `project-setup` (dynamic workflow)
**Workflow File:** `ai_instruction_modules/ai-workflow-assignments/dynamic-workflows/project-setup.md`
**Repository:** `nam20485/gap-miner-v2-delta48`
**Project:** Gap Mining Platform
**Date:** 2026-07-04
**Status:** Awaiting Approval

---

## 1. Overview

This document is the execution plan for the `project-setup` dynamic workflow. It covers
**how each workflow assignment will be executed** in the context of this repository — it is
NOT the application implementation plan (which is produced later by `create-app-plan`).

- **Workflow name:** `project-setup`
- **Workflow file reference:** `ai_instruction_modules/ai-workflow-assignments/dynamic-workflows/project-setup.md`
- **Project:** Gap Mining Platform — an internal intelligence engine that scrapes
  competitor app reviews (1–3 star), LLM-analyzes them for monetizable feature gaps, and
  surfaces ranked opportunities via a Blazor dashboard.
- **Total main assignments:** 6 (sequential)
- **Pre-script event:** `create-workflow-plan` (this document)
- **Post-assignment events (after EACH main assignment):** `validate-assignment-completion` + `report-progress`
- **Post-script event:** apply `orchestration:plan-approved` label to the application plan issue
- **What the workflow accomplishes:** initializes the repository (branch protection, project
  board, labels, file renames, setup PR), produces the application plan issue with milestones,
  scaffolds the .NET project structure, creates the AGENTS.md guidance file, debriefs
  learnings, and merges the setup PR.

### Workflow Execution Order

```
[pre-script-begin]
  └─ create-workflow-plan  ← THIS STEP (produces plan_docs/workflow-plan.md)

[initiate-new-repository]  (sequential main assignments)
  ├─ 1. init-existing-repository
  │     └─ {validate-assignment-completion, report-progress}
  ├─ 2. create-app-plan
  │     └─ {validate-assignment-completion, report-progress}
  ├─ 3. create-project-structure
  │     └─ {validate-assignment-completion, report-progress}
  ├─ 4. create-agents-md-file
  │     └─ {validate-assignment-completion, report-progress}
  ├─ 5. debrief-and-document
  │     └─ {validate-assignment-completion, report-progress}
  └─ 6. pr-approval-and-merge   ($pr_num from step 1; self-approval OK; CI loop required)
        └─ {validate-assignment-completion, report-progress}

[post-script-complete]
  └─ apply orchestration:plan-approved to the application plan issue (from step 2)
```

---

## 2. Project Context Summary

Key facts extracted from `plan_docs/` that influence how each assignment executes.

### 2.1 Application Purpose

The Gap Mining Platform is an **internal intelligence engine** (NOT a customer-facing product).
Its mission:

1. Scrape 1-to-3-star reviews of competitor apps from digital marketplaces (Shopify App Store,
   Chrome Web Store, G2, Apple App Store) via Apify.
2. Cluster and analyze negative reviews using LLMs (Semantic Kernel) to identify substantial,
   monetizable feature gaps.
3. Surface ranked opportunities via an internal Blazor dashboard for human strategic review.

The platform does NOT build, ship, or monetize micro-SaaS applications — that is a downstream
activity described in the strategic feasibility document.

### 2.2 Technology Stack (from development-plan.md §3)

| Component | Technology | Version (per plan) |
|---|---|---|
| Runtime | .NET SDK | **8.0.x (LTS)** ⚠ see Open Question Q1 |
| Orchestration | .NET Aspire | 8.2.x |
| Web UI | Blazor Web App (Interactive Server) | 8.0 |
| API | ASP.NET Core Minimal API | 8.0 |
| Database | PostgreSQL | 16.x |
| Vector Extension | pgvector | 0.7.x |
| ORM | Entity Framework Core | 8.0.x |
| Queue / Cache | Redis (StackExchange.Redis) | 7.x |
| Job Scheduling | Hangfire | 1.8.x |
| AI Orchestration | Microsoft.SemanticKernel | 1.20.x |
| LLM Provider | Azure OpenAI (GPT-4o) or Anthropic Claude 3.5 Sonnet | Latest stable |
| Embeddings | text-embedding-3-large or voyage-3 | Latest stable |
| Scraping | Apify API | v2 REST |
| HTTP Client | Refit | 7.x |
| Validation | FluentValidation | 11.x |
| Testing | xUnit + NSubstitute + Testcontainers | Latest stable |
| Code Quality | SonarAnalyzer.CSharp + StyleCop.Analyzers | Latest stable |

### 2.3 Repository Layout (target, from development-plan.md §4)

```
GapMiner/
├── GapMiner.sln
├── Directory.Build.props        # Shared MSBuild properties
├── Directory.Packages.props     # Central Package Management
├── .editorconfig
├── global.json
├── README.md
├── src/
│   ├── GapMiner.AppHost/                # Aspire orchestrator
│   ├── GapMiner.ServiceDefaults/        # Shared Aspire service defaults
│   ├── GapMiner.Domain/                 # Pure domain entities
│   ├── GapMiner.Infrastructure/         # EF Core, Redis, Apify, Semantic Kernel
│   ├── GapMiner.Application/            # Use cases / command handlers
│   ├── GapMiner.Api/                    # Minimal API gateway
│   ├── GapMiner.ScraperWorker/          # Background worker: scraping
│   ├── GapMiner.AIWorker/               # Background worker: LLM analysis
│   └── GapMiner.Web/                    # Blazor dashboard
├── tests/
│   ├── GapMiner.Domain.Tests/
│   ├── GapMiner.Infrastructure.Tests/
│   ├── GapMiner.Application.Tests/
│   ├── GapMiner.Api.Tests/
│   └── GapMiner.Integration.Tests/      # End-to-end with Testcontainers
└── docs/
    ├── prompts/                         # Version-controlled prompt library
    └── adr/                             # Architecture Decision Records
```

**Phases (development-plan.md):** Phase 0 Foundation → Phase 1 Domain & Data → Phase 2 Scraper
Pipeline → Phase 3 Intelligence Pipeline → Phase 4 API Gateway → Phase 5 Blazor Dashboard →
Phase 6 Integration Testing & Hardening. 16 atomic tasks (T-0.1 … T-6.3).

### 2.4 Current Repository State (delta48)

- The repository is a project instance cloned from the `intel-agency/ai-new-workflow-app-template`.
- Primary app spec is `plan_docs/development-plan.md` (757 lines). There is **NO**
  `ai-new-app-template.md` / `new app spec.md` — `development-plan.md` is the authoritative spec
  (same situation as sibling repo `gap-miner-v2-lima63`).
- `.devcontainer/devcontainer.json` `name` = `gap-miner-v2-delta48` → needs `-devcontainer` suffix
  per `init-existing-repository` AC #6.
- Workspace file `gap-miner-v2-delta48.code-workspace` already matches the repo name (no rename needed).
- `global.json` currently pins SDK `10.0.100` (DISCREPANCY with the plan's 8.0.x — see Open Question Q1).
- `.github/protected-branches_ruleset.json` exists (99 lines) but contains non-importable fields
  (`id`, `source`, `source_type`) and `bypass_actors` referencing org-only actor types that are
  invalid for a user repo — these MUST be stripped before import.
- `.github/ISSUE_TEMPLATE/` contains `application-plan.md`, `copilot-task.md`, `epic.md`, `story.md`.
- `.github/.labels.json` exists (188 lines, single source of truth for labels).
- `.github/workflows/validate.yml` runs CI jobs: `lint` (actionlint, gitleaks, markdownlint),
  `scan` (gitleaks), `test` (bash + Pester).

### 2.5 Environment Constraints

- The devcontainer prebuild image has **.NET SDK 10.0.102**, Node 24.14.0, Bun 1.3.10, uv 0.10.9.
- **PowerShellGet is broken in the container**: `pwsh ./scripts/validate.ps1 -All` fails ONLY on the
  PSScriptAnalyzer/Pester steps (Install-Module broken). JSON-syntax checks and the bash test suite
  (~59 tests) PASS. This is a pre-existing environment defect, not task-related; CI runs Pester on a
  healthy runner so this does not block CI.
- `git push` over HTTPS requires `gh auth setup-git` first (no credential helper by default).

---

## 3. Assignment Execution Plan

Each assignment below is documented in execution order with its goal, key acceptance criteria,
project-specific notes, prerequisites, dependencies, risks, and events.

---

### Pre-Script Event: `create-workflow-plan`

| Field | Content |
|---|---|
| **Goal** | Produce this workflow execution plan; present for approval; commit to repo. |
| **Key Acceptance Criteria** | Workflow file read & understood; every referenced assignment traced & read; all `plan_docs/` read; structured plan produced; presented & approved; committed as `plan_docs/workflow-plan.md`. |
| **Project-Specific Notes** | Primary spec is `development-plan.md` (no `ai-new-app-template.md`); reconciliation needed for .NET SDK version (8.0.x plan vs 10.0.x container). |
| **Prerequisites** | Access to dynamic workflow file + assignment files (remote canonical URLs); `plan_docs/` present. |
| **Dependencies** | None (first step). |
| **Risks / Challenges** | None material — read-only synthesis. |
| **Events** | This IS the pre-script event. |

---

### Main Assignment 1: `init-existing-repository`

| Field | Content |
|---|---|
| **Goal** | Initialize the repository: create branch, import branch-protection ruleset, create GitHub Project, import labels, rename devcontainer/workspace files, open the setup PR. |
| **Key Acceptance Criteria** | (0) new branch `dynamic-workflow-project-setup` created FIRST; (1) branch protection ruleset imported; (2) GitHub Project created; (3) project linked to repo; (4) project columns Not Started/In Progress/In Review/Done; (5) labels imported from `.github/.labels.json`; (6) filenames changed to match project name (devcontainer `-devcontainer` suffix); (7) PR created from branch → `main`. |
| **Project-Specific Notes** | • Branch name: `dynamic-workflow-project-setup`.<br>• Ruleset import: strip `id`, `source`, `source_type`, AND `bypass_actors` (the org-only `OrganizationAdmin`/`EnterpriseOwner` actor types are invalid for a user-scoped repo and cause 422 payload-validation errors). The `integration_id: 15368` in required_status_checks can remain.<br>• Ruleset requires `lint` + `scan` status-check contexts, CodeQL code_scanning, code_quality errors, copilot_code_review, required_linear_history, pull_request (require_last_push_approval, required_review_thread_resolution, 1 approving review).<br>• `gh` token works WITHOUT `administration:write` scope (the failure mode is 422 payload validation, not 403).<br>• devcontainer.json `name`: `gap-miner-v2-delta48` → `gap-miner-v2-delta48-devcontainer`.<br>• Workspace file already correctly named — no action.<br>• Use `scripts/import-labels.ps1` for labels; `gh auth setup-git` before first push. |
| **Prerequisites** | GitHub auth with scopes `repo`, `project`, `read:project`, `read:org`, `workflow`. |
| **Dependencies** | None (first main assignment). Produces `$pr_num` consumed by assignment 6. |
| **Risks / Challenges** | (R1) Ruleset 422 if `bypass_actors` not stripped — mitigated by known fix. (R2) `gh auth setup-git` missed → push fails — run it first. (R3) Idempotency: check whether a ruleset named `protected-branches` already exists before POSTing. |
| **Events** | post-assignment-complete → `validate-assignment-completion` + `report-progress`. |

**Critical output:** `$pr_num` (the PR number opened here) — REQUIRED input for `pr-approval-and-merge`.

---

### Main Assignment 2: `create-app-plan`

| Field | Content |
|---|---|
| **Goal** | Produce the application plan (PLANNING ONLY — no code): document the plan as a GitHub issue using the application-plan issue template, create `plan_docs/tech-stack.md` + `plan_docs/architecture.md`, create milestones, link the issue to the GitHub Project, assign it to the first milestone, apply labels. |
| **Key Acceptance Criteria** | App template analyzed; project structure documented; Appendix-A template used; detailed phase breakdown; all components/dependencies planned; specified tech stack followed; mandatory requirements (testing/docs/containerization) addressed; risks & mitigations; quality standards; plan ready for dev; plan documented in an issue; milestones created & linked; issue added to Project; issue assigned to first milestone; labels applied (typically `state:planning` + `documentation`). |
| **Project-Specific Notes** | • Primary spec is `plan_docs/development-plan.md` (already a detailed, phase-structured plan). The `create-app-plan` issue should synthesize it into the `.github/ISSUE_TEMPLATE/application-plan.md` format.<br>• Create 7 milestones matching the plan's phases: Phase 0 Foundation, Phase 1 Domain & Data, Phase 2 Scraper Pipeline, Phase 3 Intelligence Pipeline, Phase 4 API Gateway, Phase 5 Blazor Dashboard, Phase 6 Integration & Hardening (due dates from the plan's day estimates, anchored 2026-07-04).<br>• Assign plan issue to "Phase 0: Foundation" milestone.<br>• **Label convention:** this repo uses `namespace:label` naming — `state:planning` exists; there is NO bare `planning` label. Apply `state:planning` + `documentation`.<br>• Tech-stack reconciliation: container has .NET SDK 10.0.102 but `development-plan.md` specifies 8.0.x. Document the chosen stack in `plan_docs/tech-stack.md` and flag the decision. See Open Question Q1.<br>• Do NOT apply `orchestration:plan-approved` here — that is the post-script event's job. |
| **Prerequisites** | Assignment 1 complete (Project board + labels exist). |
| **Dependencies** | GitHub Project + label set from assignment 1. |
| **Risks / Challenges** | (R4) .NET SDK version conflict (8 vs 10) — document the decision; the container only has SDK 10 so implementations will use 10 unless downgraded. |
| **Events** | pre-assignment-begin → `gather-context`; on-assignment-failure → `recover-from-error`; post-assignment-complete → `validate-assignment-completion` + `report-progress`. |

**Critical output:** the application plan **issue number** — REQUIRED for the post-script label application.

---

### Main Assignment 3: `create-project-structure`

| Field | Content |
|---|---|
| **Goal** | Create the actual .NET solution scaffolding: `GapMiner.sln`, the 9 `src/` projects, 5 `tests/` projects, version pinning (`global.json`, `Directory.Packages.props`, `Directory.Build.props`, `.editorconfig`), Docker/compose, CI/CD foundation, README, `.ai-repository-summary.md`, docs structure. |
| **Key Acceptance Criteria** | Solution/project structure created per tech stack; all project files/dirs established; initial config files (version pinning, Docker); basic CI/CD pipeline structure; documentation structure; dev environment configured & validated; initial commit; stakeholder approval; repository summary doc created; **ALL GitHub Actions pinned to full commit SHAs**. |
| **Project-Specific Notes** | • Use `dotnet new` to scaffold projects against the repo layout in §2.3.<br>• `Directory.Packages.props` must enable Central Package Management and declare ALL package versions from development-plan.md §3 reference.<br>• `Directory.Build.props`: `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`, nullable, implicit usings.<br>• `.editorconfig`: file-scoped namespaces, 4-space indent, CRLF.<br>• `global.json`: reconcile SDK version (see Q1).<br>• Docker healthchecks: do NOT use `curl` (base image may lack it) — use Python stdlib `urllib`.<br>• The repo ALREADY has `.github/workflows/validate.yml` — do not duplicate top-level keys; add a build workflow for the .NET solution if appropriate, SHA-pinned.<br>• This is the heaviest assignment by file count — stay within the plan's exact structure; do not invent extra projects.<br>• `.ai-repository-summary.md` at repo root, linked from README. |
| **Prerequisites** | Assignment 2 complete (app plan + tech-stack.md + architecture.md exist). |
| **Dependencies** | Application plan (issue + `plan_docs/tech-stack.md` + `plan_docs/architecture.md`). |
| **Risks / Challenges** | (R5) Aspire 8.2.x templates may require SDK 8 — if committing to SDK 10, verify Aspire 8.2 compat or upgrade to a 10-compatible Aspire version. (R6) Large file count — must validate `dotnet build` succeeds with zero warnings before commit. (R7) Any new workflow MUST SHA-pin actions (supply-chain requirement). |
| **Events** | post-assignment-complete → `validate-assignment-completion` + `report-progress`. |

---

### Main Assignment 4: `create-agents-md-file`

| Field | Content |
|---|---|
| **Goal** | Create a root `AGENTS.md` that gives AI coding agents precise, actionable context: project overview, validated setup/build/test commands, project structure, code style, testing instructions, architecture notes, PR/commit guidelines, common pitfalls. |
| **Key Acceptance Criteria** | `AGENTS.md` at repo root; project overview (purpose + tech stack); setup/build/test commands VERIFIED to work; code style section; project structure section; testing instructions; PR/commit guidelines; standard Markdown; commands validated by running; committed & pushed; approval obtained. |
| **Project-Specific Notes** | • An `AGENTS.md` already exists at repo root (the orchestration-system one). The project-specific `AGENTS.md` should be tailored to the GapMiner .NET application while preserving any still-relevant orchestration context — or the existing file may already serve double duty. The create-agents-md-file assignment targets the APPLICATION project; reconcile so the file remains coherent.<br>• Commands to validate & document: `dotnet build GapMiner.sln`, `dotnet test`, restore, Aspire run (`dotnet run --project src/GapMiner.AppHost`).<br>• Cross-reference `README.md`, `.ai-repository-summary.md`, and `plan_docs/` — complement, don't duplicate.<br>• Document the `state:planning` label convention and Conventional Commits (`feat(scope): description`). |
| **Prerequisites** | Assignments 1–3 complete (project structure exists so commands can be validated). |
| **Dependencies** | Built solution from assignment 3 (so commands can be run & verified). |
| **Risks / Challenges** | (R8) Overwriting the existing orchestration `AGENTS.md` could lose orchestration guidance — must reconcile carefully. |
| **Events** | post-assignment-complete → `validate-assignment-completion` + `report-progress`. |

---

### Main Assignment 5: `debrief-and-document`

| Field | Content |
|---|---|
| **Goal** | Produce a comprehensive debriefing report capturing learnings, deviations, and improvements; save the execution trace; commit to repo. |
| **Key Acceptance Criteria** | Detailed report following the structured template (executive summary, workflow overview, deliverables, lessons learned, what worked/improved, errors & resolutions, challenges, suggested changes, metrics, future recommendations); report in `.md`; all deviations documented; reviewed & approved; committed & pushed; execution trace saved as `debrief-and-document/trace.md`. |
| **Project-Specific Notes** | • Flag the .NET SDK version decision, the ruleset `bypass_actors` strip, the PowerShellGet container defect, and any CI remediation cycles as ACTION ITEMS where plan-impacting.<br>• The trace must capture all commands run, files created/modified, and orchestrator interactions.<br>• Plan Adjustment Mandate: recommend filing issues or updating later-phase descriptions for any plan-impacting findings. |
| **Prerequisites** | Assignments 1–4 complete. |
| **Dependencies** | All prior assignment outputs (for metrics, deviations, errors). |
| **Risks / Challenges** | None material — documentation synthesis. |
| **Events** | post-assignment-complete → `validate-assignment-completion` + `report-progress`. |

---

### Main Assignment 6: `pr-approval-and-merge`

| Field | Content |
|---|---|
| **Goal** | Complete the full PR approval & merge for the setup PR (`$pr_num` from assignment 1): CI verification & remediation loop, code-review delegation, review-comment resolution, approval, merge, branch deletion, issue closure. |
| **Key Acceptance Criteria** | CI checks pass before review (remediation loop up to 3 attempts); code review delegated to `code-reviewer` (NOT self-review) — but **self-approval by the orchestrator is acceptable** per the workflow's special handling; auto-reviewer comments waited for; `ai-pr-comment-protocol.md` executed; review threads resolved (GraphQL evidence); approval obtained; merge performed (`result` = `merged`/`pending`/`failed`); source branch deleted; related issues closed. |
| **Project-Specific Notes** | • `$pr_num` comes from assignment 1.<br>• **Self-approval is acceptable** for this automated setup PR (per project-setup.md special handling) — no human stakeholder approval required.<br>• **CI remediation loop MUST still run** — if `lint`/`scan`/`test` checks fail, attempt up to 3 fix cycles before escalating.<br>• Known: `validate.yml` runs actionlint + gitleaks + markdownlint + bash + Pester. Plan docs are excluded from markdownlint. Watch for markdownlint failures on newly-created `.md` files.<br>• Branch protection (from assignment 1 ruleset) requires `lint` + `scan` contexts, linear history, 1 approving review, last-push approval, review-thread resolution, Copilot review. Self-approval + admin merge may be needed if ruleset blocks the bot.<br>• On success: delete `dynamic-workflow-project-setup` branch; close setup-related issues. |
| **Prerequisites** | Assignments 1–5 complete; PR has commits from all prior assignments. |
| **Dependencies** | `$pr_num`; green CI; branch-protection satisfaction. |
| **Risks / Challenges** | (R9) Branch protection may block bot self-merge (require_last_push_approval, 1 approving review) — may require a ruleset bypass or admin merge. (R10) markdownlint/actionlint failures on new files — remediate within 3 cycles. (R11) Copilot/auto-reviewer comments must be resolved before merge. |
| **Events** | post-assignment-complete → `validate-assignment-completion` + `report-progress`. |

---

## 4. Sequencing Diagram

```
create-workflow-plan (pre-script)  ──► APPROVAL GATE
        │
        ▼
init-existing-repository  ──────────► [$pr_num] ──► {validate, report}
        │
        ▼
create-app-plan  ──────────────────► [plan issue #] ──► {validate, report}
        │
        ▼
create-project-structure  ─────────► [.NET scaffolding] ──► {validate, report}
        │
        ▼
create-agents-md-file  ────────────► [AGENTS.md] ──► {validate, report}
        │
        ▼
debrief-and-document  ─────────────► [report + trace] ──► {validate, report}
        │
        ▼
pr-approval-and-merge ($pr_num)  ──► [merged PR] ──► {validate, report}
        │
        ▼
post-script-complete  ─────────────► apply orchestration:plan-approved to plan issue
```

**Hard dependencies:**
- `create-app-plan` needs the Project board + labels from `init-existing-repository`.
- `create-project-structure` needs the app plan + tech-stack.md + architecture.md.
- `create-agents-md-file` needs a built solution to validate commands.
- `pr-approval-and-merge` needs `$pr_num` from `init-existing-repository`.
- The post-script label needs the plan issue number from `create-app-plan`.

---

## 5. Open Questions

These ambiguities should be confirmed before or during the relevant assignment.

### Q1: .NET SDK version — 8.0.x (plan) vs 10.0.x (container/global.json)

`plan_docs/development-plan.md` §3 specifies **.NET SDK 8.0.x (LTS)** and §2.1 R2 says "Never
invent package versions. Use the exact versions in §3." However:
- The devcontainer prebuild image ships **.NET SDK 10.0.102** only.
- The repo's existing `global.json` pins **10.0.100**.

**Options:**
- (a) Honor the plan: downgrade pin to 8.0.x — but the container has no SDK 8 installed, so `dotnet`
  commands would fail unless SDK 8 is added.
- (b) Reconcile upward: adopt **.NET SDK 10.0.x** + Aspire/EF Core/Blazor 10-compatible versions,
  updating the tech-stack to what the container actually provides. This keeps the dev environment
  functional and is the pragmatic path.
- (c) Keep the plan's 8.0.x versions for package references but set `rollForward` so SDK 10 can
  build them (risky — major-version skew).

**Recommendation:** Option (b) — adopt SDK 10.0.x (matching container + global.json) and document
the stack in `plan_docs/tech-stack.md` during `create-app-plan`, noting the deviation from the
original 8.0.x plan. **Needs stakeholder confirmation.**

### Q2: Existing root AGENTS.md (orchestration system) vs application AGENTS.md

A detailed orchestration-system `AGENTS.md` already exists at the repo root. The
`create-agents-md-file` assignment wants an application-focused `AGENTS.md`. **Decision needed:**
replace, merge, or keep the orchestration file and add application guidance into it. Recommendation:
augment the existing file with an application section so both audiences are served.

### Q3: Branch-protection vs automated self-merge

The imported ruleset requires 1 approving review + `require_last_push_approval` + review-thread
resolution. The workflow permits self-approval, but the ruleset may still block a bot self-merge.
**Decision needed:** if the merge is blocked, allow a ruleset bypass actor or an admin
(`--admin`) merge for this automated setup PR.

---

## 6. Risk Register

| ID | Risk | Impact | Likelihood | Mitigation | Owner Assignment |
|---|---|---|---|---|---|
| R1 | Ruleset import 422 (org-only bypass_actors) | Blocks branch protection | High | Strip `id`, `source`, `source_type`, `bypass_actors` before POST (validated on sibling repo) | init-existing-repository |
| R2 | `git push` fails (no credential helper) | Blocks all commits | High | Run `gh auth setup-git` before first push | init-existing-repository |
| R3 | Non-idempotent ruleset creation | Duplicate rulesets | Low | Check existing ruleset by name before POST | init-existing-repository |
| R4 | .NET SDK version conflict (8 vs 10) | Build failures | High | Reconcile to SDK 10 in tech-stack.md (Q1) | create-app-plan |
| R5 | Aspire 8.2 incompatibility with SDK 10 | Scaffolding/build failures | Medium | Verify Aspire version compat during create-project-structure; upgrade if needed | create-project-structure |
| R6 | `dotnet build` warnings (TreatWarningsAsErrors) | Build fails | Medium | Resolve all warnings before commit | create-project-structure |
| R7 | Non-SHA-pinned actions in workflows | CI/supply-chain failure | Medium | Pin all `uses:` to full SHAs with semver comment | create-project-structure |
| R8 | Overwriting orchestration AGENTS.md | Loss of orchestration guidance | Medium | Augment, don't replace (Q2) | create-agents-md-file |
| R9 | Branch protection blocks bot self-merge | Merge blocked | Medium | Bypass actor or admin merge (Q3) | pr-approval-and-merge |
| R10 | markdownlint/actionlint on new files | CI red | Medium | Run local validation; fix within 3 remediation cycles | pr-approval-and-merge |
| R11 | PowerShellGet broken in container | Local validate.ps1 partial | Known | Bash tests + JSON checks pass locally; CI runs Pester on healthy runner | All |
| R12 | Copilot/auto-reviewer comments unresolved | Merge blocked | Medium | Wait 60-120s; resolve all threads via GraphQL | pr-approval-and-merge |

---

## 7. Mandatory Constraints (carried from the workflow + repo rules)

1. **Action SHA Pinning** — every `uses:` in every workflow file MUST be pinned to a full 40-char
   commit SHA with a trailing `# vX.Y.Z` comment. No `@v4` / `@main` tags.
2. **No duplicate top-level YAML keys** (`name:`, `on:`, `jobs:`) in workflow files.
3. **Preserve `__EVENT_DATA__`** placeholder in orchestrator-agent-prompt.md.
4. **Orchestrator delegation-depth ≤ 2**; orchestrator never writes code directly.
5. **No hardcoded secrets**; synthetic test values must not match real provider prefixes.
6. **Change validation** — run `pwsh ./scripts/validate.ps1 -All` (or bash-equivalent subset given
   the container's PowerShellGet defect) before every commit; fix all failures; do not push until
   clean; monitor CI to green after push.
7. **Labels** are defined in `.github/.labels.json` (single source of truth); use
   `scripts/import-labels.ps1`.
8. **Commit messages** follow Conventional Commits: `feat(scope): description`.

---

*End of workflow execution plan. Awaiting stakeholder approval before proceeding to
`init-existing-repository`.*
