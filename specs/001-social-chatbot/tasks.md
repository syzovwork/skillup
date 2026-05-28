---
description: "Task list for feature 001-social-chatbot"
---

# Tasks: Multi-Brand Social Media Insights Chatbot

**Input**: Design documents from `/specs/001-social-chatbot/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: Test tasks are included because the constitution (Principle IV)
mandates unit tests for business logic and integration tests for AI pipeline
components.

**Organization**: Tasks are grouped by user story to enable independent
implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3)
- All file paths are repository-relative

## Path Conventions (from plan.md)

- Backend solution: `backend/src/SocialChatbot.{Api,Application,Domain,Infrastructure}`
- Backend tests: `backend/tests/SocialChatbot.{UnitTests,IntegrationTests}`
- Frontend: `frontend/src/`, `frontend/tests/`
- Data: `data/seed/`, `data/briefs/`
- Infra root files: `docker-compose.yml`, `.editorconfig`, `.gitignore`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Repo scaffolding, language toolchains, container deps.

- [ ] T001 Create top-level directory layout `backend/src/`, `backend/tests/`, `frontend/`, `data/seed/`, `data/briefs/` per `specs/001-social-chatbot/plan.md`
- [ ] T002 Initialize .NET 8 solution and projects under `backend/`: `SocialChatbot.sln` with `SocialChatbot.Api` (web), `SocialChatbot.Application` (classlib), `SocialChatbot.Domain` (classlib), `SocialChatbot.Infrastructure` (classlib), `SocialChatbot.UnitTests` (xunit), `SocialChatbot.IntegrationTests` (xunit); wire project references so dependencies point inward (Api → Application → Domain; Infrastructure → Domain; Api → Infrastructure for composition root only)
- [ ] T003 Add NuGet packages to the appropriate backend projects: `Microsoft.SemanticKernel`, `Microsoft.SemanticKernel.Connectors.OpenAI` (Application + Infrastructure), `Microsoft.Data.SqlClient` (Infrastructure), `UglyToad.PdfPig` (Infrastructure), `Serilog.AspNetCore` + `Serilog.Sinks.Console` (Api), `Microsoft.AspNetCore.Mvc.Testing` (IntegrationTests), `FluentAssertions` (test projects)
- [ ] T004 [P] Initialize React + TypeScript + Vite app in `frontend/` (`npm create vite@latest frontend -- --template react-ts`); add `axios` and `vitest` + `@testing-library/react`
- [ ] T005 [P] Create `docker-compose.yml` at repo root with services `sqlserver` (`mcr.microsoft.com/mssql/server:2022-latest`, host port 1433) and `chroma` (`chromadb/chroma:latest`, host port 8000) with named volumes for persistence; document `SA_PASSWORD` env var
- [ ] T006 [P] Create `.editorconfig` (4-space C#, 2-space TS/JSON) and `.gitignore` (bin/obj, node_modules, .vs, _.user, appsettings._.local.json, .env\*) at repo root
- [ ] T007 [P] Add backend code-style: `.editorconfig` rules covering `dotnet_diagnostic.IDE0005.severity = warning`, plus `dotnet format` pre-commit instructions in `backend/README.md`
- [ ] T008 [P] Add `data/seed/posts.csv` with the dummy dataset matching the schema in `specs/001-social-chatbot/data-model.md` §1 (≥ 3 brands `Aurora`, `Nimbus`, `Vela`; both platforms; ≥ 6 months ending current month; multiple `ContentFormat` values per brand)
- [ ] T009 [P] Add 3 sample brand-brief PDFs to `data/briefs/<Brand>/brief.pdf` for `Aurora`, `Nimbus`, `Vela` containing tone-of-voice, content requirements, and deadlines sections (text-based PDFs only)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Cross-cutting infrastructure that EVERY user story needs.

**⚠️ CRITICAL**: No user story work begins until Phase 2 is complete.

### Configuration & secrets

- [ ] T010 Configure `appsettings.json` in `backend/src/SocialChatbot.Api/appsettings.json` with non-secret defaults (`Dial:BaseUrl`, `Dial:ChatModel = gpt-4o-mini`, `Dial:EmbeddingModel = text-embedding-3-small`, `Sql:ConnectionString` template, `Chroma:BaseUrl = http://localhost:8000`); initialize User Secrets storage for `Dial:ApiKey` and `Sql:Password` per quickstart §2.1; ensure NO secret values are committed
- [ ] T011 Implement strongly-typed `DialOptions`, `SqlOptions`, `ChromaOptions` records in `backend/src/SocialChatbot.Infrastructure/Configuration/` and bind them in `Program.cs`

### Domain layer (no I/O)

- [ ] T012 [P] Implement `Post` record and `Platform`, `ContentFormat` enums in `backend/src/SocialChatbot.Domain/Posts/Post.cs` per data-model.md §3
- [ ] T013 [P] Implement `Brand` record in `backend/src/SocialChatbot.Domain/Briefs/Brand.cs`
- [ ] T014 [P] Implement `BriefChunk` record in `backend/src/SocialChatbot.Domain/Briefs/BriefChunk.cs`
- [ ] T015 [P] Implement `QuestionCategory` enum and `RoutingDecision`, `ChatTurn`, `ChatRole`, `ChatRequest`, `ChatResponse` records in `backend/src/SocialChatbot.Domain/Chat/`

### Persistence & data ingestion

- [ ] T016 Create SQL schema script `backend/src/SocialChatbot.Infrastructure/Sql/Schema/001_create_posts.sql` per data-model.md §1, including `CHECK` constraints and `IX_Posts_Brand_Date`, `IX_Posts_Platform_Date`, `IX_Posts_Date` indexes plus `Brands` table with FK from `Posts.Brand`
- [ ] T017 Implement `SqlBootstrapper` in `backend/src/SocialChatbot.Infrastructure/Sql/SqlBootstrapper.cs` that runs schema script on startup and bulk-inserts `data/seed/posts.csv` if `Posts` is empty
- [ ] T018 Implement `IPostsRepository` interface in `backend/src/SocialChatbot.Application/Tools/` and `PostsRepository` in `backend/src/SocialChatbot.Infrastructure/Sql/PostsRepository.cs` exposing typed query methods: `QueryAsync(MetricsQuery query, CancellationToken)`, `GetDatasetBoundsAsync()`, `ListBrandsAsync()` — all use parameterized SQL only

### DIAL clients (LLM + embeddings) with traceability

- [ ] T019 Implement `IDialChatClient` interface in `backend/src/SocialChatbot.Application/Llm/IDialChatClient.cs` and `DialChatClient` in `backend/src/SocialChatbot.Infrastructure/Dial/DialChatClient.cs`: wraps SK OpenAI connector pointed at DIAL base URL; emits Serilog structured log per call with fields `correlationId`, `promptTemplateId`, `promptVersion`, `model`, `tokensIn`, `tokensOut`, `latencyMs`, `outcome` (constitution Principle III)
- [ ] T020 Implement `IDialEmbeddingClient` and `DialEmbeddingClient` in `backend/src/SocialChatbot.Infrastructure/Dial/DialEmbeddingClient.cs` with the same structured logging

### Vector store

- [ ] T021 Implement `IBriefStore` in `backend/src/SocialChatbot.Application/Tools/IBriefStore.cs` and `ChromaBriefStore` in `backend/src/SocialChatbot.Infrastructure/Vector/ChromaBriefStore.cs`: supports `UpsertAsync(BriefChunk, embedding)`, `QueryAsync(string queryText, string? brandFilter, int topK)`, `CountAsync()`
- [ ] T022 Implement `PdfBriefIngestor` in `backend/src/SocialChatbot.Infrastructure/Ingestion/PdfBriefIngestor.cs` using PdfPig: walks `data/briefs/<Brand>/*.pdf`, extracts text per page, chunks at ~600 tokens with ~80-token overlap (research §5), generates embeddings via `IDialEmbeddingClient`, upserts to `ChromaBriefStore`; idempotent (skip when `CountAsync() > 0`)
- [ ] T023 Wire `SqlBootstrapper` and `PdfBriefIngestor` to run on host startup in `backend/src/SocialChatbot.Api/Program.cs` (sequential, awaited before the host accepts requests)

### Prompts (versioned)

- [ ] T024 [P] Create router prompt `backend/src/SocialChatbot.Application/Prompts/router.v1.md` returning strict JSON `{ "category": "metrics"|"brief"|"mixed"|"clarify", "brand": "<name|null>", "reason": "<short>" }` per research §6
- [ ] T025 [P] Create metrics-answer prompt `backend/src/SocialChatbot.Application/Prompts/metrics-answer.v1.md` that instructs the model to cite only values present in the tool result (FR-019)
- [ ] T026 [P] Create brief-answer prompt `backend/src/SocialChatbot.Application/Prompts/brief-answer.v1.md` that instructs the model to cite the brand name (FR-016) and refuse to fabricate when chunks are empty (FR-017)
- [ ] T027 [P] Create mixed-answer prompt `backend/src/SocialChatbot.Application/Prompts/mixed-answer.v1.md` that instructs the model to include at least one brief-derived element and one metric-derived element (FR-018, SC-005)

### Application services (skeletons used by all stories)

- [ ] T028 Implement `PromptSanitizer` in `backend/src/SocialChatbot.Application/Chat/PromptSanitizer.cs` redacting emails, phone numbers, and ≥ 12-digit runs (constitution Security Standards)
- [ ] T029 Implement `ResponseValidator` skeleton in `backend/src/SocialChatbot.Application/Chat/ResponseValidator.cs` exposing `Validate(ChatResponse, ValidationContext)` with three checks (empty/refusal, source-attribution, numeric-grounding); each check returns a `ValidationResult` (story phases extend the rules)
- [ ] T030 Implement `QuestionRouter` in `backend/src/SocialChatbot.Application/Chat/QuestionRouter.cs` that loads `router.v1.md` via SK and parses + validates the JSON output (uses `IDialChatClient`)
- [ ] T031 Implement `ChatService` in `backend/src/SocialChatbot.Application/Chat/ChatService.cs` exposing `Task<ChatResponse> AnswerAsync(ChatRequest, CancellationToken)`; for Phase 2 it only wires routing + sanitization + validator and returns a stub answer per category (the per-category answer prompts are filled in by story phases)

### API surface

- [ ] T032 Implement `POST /api/chat` endpoint in `backend/src/SocialChatbot.Api/Endpoints/ChatEndpoint.cs` matching `specs/001-social-chatbot/contracts/chat-api.openapi.yaml` (request schema, 200/400/422/500 response shape, correlation id middleware, request cancellation honored)
- [ ] T033 Implement `GET /api/health` endpoint returning per-dependency status (SQL, Chroma, DIAL) per the OpenAPI contract
- [ ] T034 Add Serilog request-logging middleware in `backend/src/SocialChatbot.Api/Program.cs` with JSON output and a `CorrelationId` enricher

### Frontend foundation

- [ ] T035 [P] Implement `frontend/src/api/chatClient.ts` posting to `/api/chat` with `AbortController`, reading base URL from `import.meta.env.VITE_API_BASE_URL` (default `http://localhost:5080`)
- [ ] T036 [P] Implement `frontend/src/components/MessageList.tsx`, `MessageInput.tsx`, `SourceBadge.tsx`, `ChatWindow.tsx`, and `App.tsx` per `specs/001-social-chatbot/contracts/frontend-ui-contract.md` (layout, ARIA roles, badges, category chip, history capped at 10 turns)
- [ ] T037 [P] Add `frontend/src/App.test.tsx` and `frontend/src/components/ChatWindow.test.tsx` using Vitest + Testing Library asserting render of empty state and that Send triggers a POST

### Foundational tests

- [ ] T038 [P] Unit test `PromptSanitizerTests` in `backend/tests/SocialChatbot.UnitTests/PromptSanitizerTests.cs` covering email / phone / long-digit redaction
- [ ] T039 [P] Unit test `QuestionRouterTests` in `backend/tests/SocialChatbot.UnitTests/QuestionRouterTests.cs` using an `IDialChatClient` fake that returns canned JSON; asserts routing decisions for metrics, brief, mixed, and clarify inputs
- [ ] T040 [P] Unit test `ResponseValidatorTests` in `backend/tests/SocialChatbot.UnitTests/ResponseValidatorTests.cs` covering empty answer, missing brand citation, hallucinated number not in tool result
- [ ] T041 [P] Integration test fixture `WebAppFactory` in `backend/tests/SocialChatbot.IntegrationTests/Fixtures/WebAppFactory.cs`: spins up `WebApplicationFactory` with `IDialChatClient` and `IDialEmbeddingClient` replaced by recorded-response fakes, seeded SQL (LocalDB or testcontainers), and an in-memory `IBriefStore` substitute
- [ ] T042 [P] Integration test `HealthEndpointTests` in `backend/tests/SocialChatbot.IntegrationTests/HealthEndpointTests.cs` asserting `/api/health` returns 200 with all three dependencies reported
- [ ] T079 [US1] Wire ChatService to detect category=clarify from router response and populate ChatResponse.clarifyingQuestion field; return HTTP 200 with clarifyingQuestion set and no sourcesUsed — specs/001-social-chatbot/src/Application/Services/ChatService.cs
- [ ] T080 [US1] Extend ChatWindow.tsx to render clarifyingQuestion prominently when present in API response (distinct visual style from normal answer, e.g. question bubble) — specs/001-social-chatbot/src/Frontend/src/components/ChatWindow.tsx

**Checkpoint**: Foundation ready — user story implementation can now begin.

---

## Phase 3: User Story 1 - Ask data questions across platforms (Priority: P1) 🎯 MVP

**Goal**: Manager asks natural-language questions about Instagram + TikTok
post metrics and gets a single conversational answer that is factually
correct against the dummy `Posts` dataset.

**Independent Test**: With the seeded dataset and no brand-brief PDFs ingested
(or briefs ignored), ask each of:

- "Which post had the highest engagement last month?"
- "Compare Instagram vs TikTok reach for April."
- "What content format performs best on TikTok?"

Verify answers are factually consistent with `data/seed/posts.csv` and the
`category` chip is `metrics`.

### Tests for User Story 1

- [ ] T043 [P] [US1] Unit test `MetricsSqlToolTests` in `backend/tests/SocialChatbot.UnitTests/MetricsSqlToolTests.cs` against an in-memory `IPostsRepository` fake covering: top-N by engagement, platform comparison (Instagram vs TikTok over date range), best-performing content format per platform, brand-scoped filtering, dataset-bounds query
- [ ] T044 [P] [US1] Integration test `ChatEndpoint_MetricsOnly_Tests` in `backend/tests/SocialChatbot.IntegrationTests/ChatEndpoint_MetricsOnly_Tests.cs` posting the three spec questions with recorded DIAL responses; asserts `category == "metrics"`, `sourcesUsed == ["metrics"]`, and the answer text contains specific tokens (brand name, date, metric value) from the seeded dataset
- [ ] T045 [P] [US1] Integration test in the same file asserting the out-of-data fallback for "What's our click-through rate?" returns an honest "not available" answer (SC-007)

### Implementation for User Story 1

- [ ] T046 [US1] Implement `MetricsQuery` record (brand, platform, contentFormat, dateFrom, dateTo, metric, aggregation, topN, comparisonAxis) in `backend/src/SocialChatbot.Application/Tools/MetricsQuery.cs`
- [ ] T047 [US1] Implement `MetricsSqlTool` in `backend/src/SocialChatbot.Application/Tools/MetricsSqlTool.cs` as an SK kernel function exposing methods `TopByEngagement`, `ComparePlatforms`, `BestFormat`, `GetDatasetBounds`, `Search` — every call goes through `IPostsRepository` with parameterized queries (research §9); annotate with `KernelFunction` + parameter descriptions so the LLM can pick correct arguments
- [ ] T048 [US1] Extend `ChatService.AnswerAsync` to handle `QuestionCategory.Metrics`: load `metrics-answer.v1.md`, register `MetricsSqlTool` as a kernel plugin, invoke with chat history and sanitized user message, run `ResponseValidator` with `NumericGroundingCheck` enabled against the tool's last result set
- [ ] T049 [US1] Extend `ResponseValidator.NumericGroundingCheck` in `backend/src/SocialChatbot.Application/Chat/ResponseValidator.cs` to extract integer/decimal tokens from the answer and verify each appears in the tool result set or in dataset-bounds metadata (FR-019)
- [ ] T050 [US1] In `ChatService`, when `MetricsSqlTool.GetDatasetBounds` shows the user's referenced date range is outside the data, emit an honest fallback answer instead of calling the model with empty data (FR-021)
- [ ] T051 [US1] Emit `sourcesUsed = ["metrics"]` on the `ChatResponse` for metrics-only answers in `ChatService`
- [ ] T052 [US1] Frontend: render the `Metrics` source badge in `frontend/src/components/SourceBadge.tsx` and verify `ChatWindow` displays it when `sourcesUsed` contains `"metrics"`

**Checkpoint**: US1 is end-to-end functional and independently demoable.

---

## Phase 4: User Story 2 - Ask document questions over brand briefs (Priority: P2)

**Goal**: Manager asks brand-specific questions (tone of voice, content
requirements, deadlines) and gets answers grounded in the ingested PDF
briefs, citing the brand.

**Independent Test**: With the three sample briefs ingested, ask:

- "What does Aurora say about tone of voice?"
- "What are the deliverable deadlines for Nimbus?"

Verify answers reflect the brief content, the `category` chip is `brief`, and
the `Brand brief: <Brand>` badge appears.

### Tests for User Story 2

- [ ] T053 [P] [US2] Unit test `PdfBriefIngestorTests` in `backend/tests/SocialChatbot.UnitTests/PdfBriefIngestorTests.cs` using a fixture PDF (text-only): asserts chunk count ≈ expected, overlap exists, metadata `brand` + `source_file` + `page_start` populated, idempotency (second run does not duplicate when `CountAsync() > 0`)
- [ ] T054 [P] [US2] Unit test `BriefRagToolTests` in `backend/tests/SocialChatbot.UnitTests/BriefRagToolTests.cs` against an in-memory `IBriefStore` fake covering: brand-filtered retrieval, empty-result handling for an unknown brand, top-K bound = 4
- [ ] T055 [P] [US2] Integration test `ChatEndpoint_BriefOnly_Tests` in `backend/tests/SocialChatbot.IntegrationTests/ChatEndpoint_BriefOnly_Tests.cs`: posts brief questions with recorded DIAL responses; asserts `category == "brief"`, `sourcesUsed == ["brief:<Brand>"]`, answer contains the brand name (FR-016)
- [ ] T056 [P] [US2] Integration test in the same file asserting that asking about an unknown brand returns "no brief on file" without fabrication (FR-017)

### Implementation for User Story 2

- [ ] T057 [US2] Implement `BriefRagTool` in `backend/src/SocialChatbot.Application/Tools/BriefRagTool.cs` as an SK kernel function: `RetrieveAsync(string queryText, string? brand, int topK = 4)` returning `IReadOnlyList<BriefChunk>`; uses `IBriefStore.QueryAsync` with brand filter when supplied
- [ ] T058 [US2] Extend `ChatService.AnswerAsync` to handle `QuestionCategory.Brief`: load `brief-answer.v1.md`, register `BriefRagTool` as a kernel plugin, invoke; require `ResponseValidator.SourceAttributionCheck` to confirm the answer cites the brand
- [ ] T059 [US2] Extend `ResponseValidator.SourceAttributionCheck` in `backend/src/SocialChatbot.Application/Chat/ResponseValidator.cs` to enforce brand-name presence in answer text for `Brief` and `Mixed` categories (FR-016, SC-004)
- [ ] T060 [US2] In `ChatService`, when `BriefRagTool.RetrieveAsync` returns zero chunks for the requested brand, emit a deterministic "no brief on file for {brand}" response without calling the answer prompt (FR-017)
- [ ] T061 [US2] Emit `sourcesUsed = ["brief:<Brand>"]` on the `ChatResponse` for brief-only answers in `ChatService`
- [ ] T062 [US2] Frontend: render the `Brand brief: <Brand>` badge in `SourceBadge.tsx` parsing the `brief:<Brand>` token; update `ChatWindow.test.tsx` to assert badge appears

**Checkpoint**: US1 + US2 both work; brief questions cite the correct brand and decline gracefully when the brief is missing.

---

## Phase 5: User Story 3 - Ask mixed questions combining metrics and briefs (Priority: P3)

**Goal**: Manager asks strategic questions that require both the brief
guidelines AND recent metrics for a brand, and gets a single recommendation
referencing both sources.

**Independent Test**: With both data and Brand X's brief loaded, ask "Based on
Aurora's guidelines and last month's data, what should I post next?" and
verify the answer (a) names at least one Aurora guideline, (b) cites at least
one metric value from Aurora's recent posts, and (c) is internally consistent.

### Tests for User Story 3

- [ ] T063 [P] [US3] Integration test `ChatEndpoint_Mixed_Tests` in `backend/tests/SocialChatbot.IntegrationTests/ChatEndpoint_Mixed_Tests.cs` with recorded DIAL responses for a mixed question; asserts `category == "mixed"`, `sourcesUsed` contains both `"metrics"` and `"brief:Aurora"`, answer contains a number from the metric tool result AND a brand-name citation
- [ ] T064 [P] [US3] Integration test in the same file asserting that when a brand has a brief but no metrics in scope, the mixed answer says metrics are missing and falls back to brief-only guidance (spec §US3 Acceptance Scenario 2)
- [ ] T065 [P] [US3] Unit test in `backend/tests/SocialChatbot.UnitTests/MixedAnswerValidationTests.cs` extending `ResponseValidator` checks: asserts that mixed answers without at least one metric AND at least one brief-derived clause are rejected and trigger the single retry (SC-005)

### Implementation for User Story 3

- [ ] T066 [US3] Extend `ChatService.AnswerAsync` to handle `QuestionCategory.Mixed`: load `mixed-answer.v1.md`, register BOTH `MetricsSqlTool` and `BriefRagTool` as kernel plugins, invoke; collect tool-call evidence to feed validation
- [ ] T067 [US3] Extend `ResponseValidator` with a `MixedCoverageCheck` that requires at least one numeric grounding hit AND one brand-name citation; on first failure, retry once with a stricter "include sources" system addendum (research §10)
- [ ] T068 [US3] In `ChatService`, when Mixed routing produces zero metric rows for the named brand, fall back to brief-only answering and surface `sourcesUsed = ["brief:<Brand>"]` with a note that metrics were unavailable for the requested range (FR-021)
- [ ] T069 [US3] Emit `sourcesUsed = ["metrics", "brief:<Brand>"]` on the `ChatResponse` for successful mixed answers in `ChatService`

**Checkpoint**: All three user stories are independently demonstrable.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Quality, performance, and demo readiness across all stories.

- [ ] T070 [P] Add `docker-compose.yml` health checks for `sqlserver` and `chroma` so `docker compose up -d` reports readiness; document in `backend/README.md`
- [ ] T071 [P] Write `backend/README.md` covering build, test, configuration, and seed/ingestion steps from `specs/001-social-chatbot/quickstart.md`
- [ ] T072 [P] Write `frontend/README.md` with run/test instructions
- [ ] T073 [P] Add performance smoke test `backend/tests/SocialChatbot.IntegrationTests/Performance_Smoke_Tests.cs` measuring p95 over 20 calls per category against recorded DIAL responses; assert p95 < 5000 ms (SC-003) — note this measures non-LLM latency budget only
- [ ] T074 [P] Add accessibility check to `frontend/tests/`: verify `aria-label` on input, `role="log"` on message list, badges have readable text (UI contract §Accessibility)
- [ ] T075 Verify constitution compliance: secret scan over the repo (e.g. `gitleaks detect --no-git`) and run from CI script; ensure no `Dial:ApiKey` literal appears anywhere except User Secrets / env documentation
- [ ] T076 Run the quickstart end-to-end (`specs/001-social-chatbot/quickstart.md` §4) manually against all six demo questions and record results in `specs/001-social-chatbot/quickstart-validation.md`
- [ ] T077 [P] Add CI workflow `.github/workflows/ci.yml` running `dotnet build`, `dotnet test`, `npm run test --prefix frontend`, and the secret scan from T075
- [ ] T078 Final pass: ensure every functional requirement FR-001 .. FR-021 in `specs/001-social-chatbot/spec.md` is traced by at least one task above (record the trace in a comment block at the end of this file if any are missing)
- [ ] T081 Create data/eval/benchmark.jsonl with 30 labeled questions: 10 metrics, 10 brief, 10 mixed; each entry includes question, expected_category, expected_brand (where applicable), and key_facts the answer must contain — specs/001-social-chatbot/data/eval/benchmark.jsonl
- [ ] T082 Implement evaluation harness (script or test project) that POSTs each benchmark question to POST /api/chat, compares returned category to expected_category, and checks key_facts presence in answer; outputs accuracy % and routing F1 score — `specs/001-social-chatbot/tools/eval/EvalRunner.cs`
- [ ] T083 Gate ship decision on SC-001 (≥ 90% factual accuracy) and SC-002 (≥ 95% routing accuracy) by running T082 harness as part of Phase 6 checklist; results must be committed to `data/eval/results.json` — `specs/001-social-chatbot/data/eval/results.json`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)** — no dependencies; can start immediately.
- **Foundational (Phase 2)** — depends on Setup; **BLOCKS all user stories**.
- **User Story 1 (Phase 3, P1)** — depends only on Phase 2.
- **User Story 2 (Phase 4, P2)** — depends only on Phase 2; independent of US1 (touches different prompt, different tool).
- **User Story 3 (Phase 5, P3)** — depends on Phase 2; uses tools created in US1 and US2 phases, so practically runs after US1 and US2 are merged (or while their tools are wired but before validators are fully tightened).
- **Polish (Phase 6)** — depends on the stories you intend to ship.

### User Story Dependencies (within stories)

- Models / domain types → tools (services) → ChatService wiring → endpoint surface → frontend badge.
- Tests for a story SHOULD be written before or alongside its implementation tasks (constitution Principle IV).

### Parallel Opportunities

- Phase 1: T004–T009 are all `[P]` (different files).
- Phase 2 domain types T012–T015 are `[P]` together; prompts T024–T027 are `[P]` together; foundational tests T038–T042 are `[P]` together.
- Phase 3 tests T043–T045 are `[P]`; story-level implementation tasks T046–T052 are sequential because they touch shared files (`ChatService`, `ResponseValidator`).
- Same pattern in Phase 4 (T053–T056 `[P]`, T057–T062 sequential) and Phase 5 (T063–T065 `[P]`, T066–T069 sequential).
- Phase 6 polish tasks T070–T074 and T077 are `[P]`.

---

## Parallel Example: User Story 1

```bash
# Run all US1 test scaffolding tasks in parallel:
Task T043: "Unit test MetricsSqlToolTests in backend/tests/SocialChatbot.UnitTests/MetricsSqlToolTests.cs"
Task T044: "Integration test ChatEndpoint_MetricsOnly_Tests for three spec questions"
Task T045: "Integration test for out-of-data fallback ('click-through rate')"

# Then implement sequentially because they share ChatService / ResponseValidator:
Task T046 → T047 → T048 → T049 → T050 → T051 → T052
```

---

## Implementation Strategy

### MVP First (User Story 1 only)

1. Phase 1 (Setup) → Phase 2 (Foundational) → Phase 3 (US1).
2. STOP and demo: the three P1 metric questions answer correctly.
3. Ship as PoC milestone 1.

### Incremental Delivery

1. Foundation + US1 → MVP demo (metrics-only chatbot).
2. - US2 → demo brand-brief Q&A with brand citation.
3. - US3 → demo the mixed-reasoning differentiator.
4. Polish phase → quickstart validation pass, performance smoke, README.

### Parallel Team Strategy

With multiple developers after Phase 2 completes:

- Dev A: US1 (metrics) — tools, validator numeric grounding, frontend badge.
- Dev B: US2 (briefs) — ingestion + RAG tool + brand-citation validator.
- Dev C (joins later): US3 (mixed) — depends on tools from A and B but can prep the prompt and tests in parallel.

---

## Notes

- `[P]` = different files, no in-phase dependencies.
- Every story-phase task carries its `[US#]` tag for traceability back to
  `spec.md`.
- Tests-first is encouraged for unit tests of pure logic (router, validator,
  SQL tool); integration tests can be written alongside endpoint wiring with
  recorded DIAL responses.
- Commit after each task or coherent group; checkpoint markers indicate
  natural demo / review boundaries.
- Constitution traceability: tasks T019, T020, T024–T027 (versioned prompts +
  traced LLM calls), T028 (sanitization), T029/T049/T059/T067 (response
  validation), and T038–T042 + T043–T045 + T053–T056 + T063–T065 (unit +
  integration tests for the AI pipeline) collectively satisfy Principles III
  and IV.
