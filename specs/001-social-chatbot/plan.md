# Implementation Plan: Multi-Brand Social Media Insights Chatbot

**Branch**: `001-social-chatbot` | **Date**: 2026-05-14 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-social-chatbot/spec.md`

## Summary

A PoC conversational assistant for social media managers who work across
Instagram and TikTok. The user asks natural-language questions in a chat UI;
the backend classifies each question as **metrics-only**, **brief-only**, or
**mixed**, and answers using a SQL tool over a `posts` table, a RAG tool over
PDF brand briefs (chunked + embedded into a Chroma vector store), or both. The
backend is a single .NET REST endpoint; orchestration and tool routing are
implemented with **Semantic Kernel**. LLM and embedding calls go through the
**DIAL API**. The frontend is a minimal React chat UI.

## Technical Context

**Language/Version**: C# / .NET 8 (LTS) for backend; TypeScript + React 18 for frontend.
**Primary Dependencies**:
- Backend: `Microsoft.SemanticKernel`, `Microsoft.SemanticKernel.Connectors.OpenAI`
  (configured to point at DIAL), `Microsoft.Data.SqlClient`, a Chroma .NET client
  (`ChromaDB.Client` or HTTP via `HttpClient`), `UglyToad.PdfPig` for PDF text
  extraction, `Serilog` for structured logging.
- Frontend: `react`, `react-dom`, `vite`, `axios` (or fetch), `tailwindcss` (optional).

**Storage**:
- **MS SQL Server** (LocalDB or container) for the `Posts` table.
- **Chroma** vector store (local Docker container) for brand-brief chunks.

**Testing**:
- Backend: **xUnit** for unit tests, **xUnit + WebApplicationFactory** for
  integration tests; recorded LLM responses for determinism.
- Frontend: **Vitest** + **React Testing Library**.

**Target Platform**: Local developer machines (Windows / macOS / Linux) for the
PoC. No production deployment.

**Project Type**: Web application (separate `backend/` and `frontend/`).

**Performance Goals**:
- p95 end-to-end chat response **< 5 seconds** (per spec SC-003).
- First streamed token within **2 seconds** when streaming is enabled.
- SQL queries < 200 ms p95 on the dummy dataset.
- Vector search < 200 ms p95 with top-K ≤ 8 over ~5k chunks.

**Constraints**:
- Single-user PoC; no auth, no multi-tenant isolation.
- All LLM/embedding traffic via DIAL only (constitution Principle II).
- No outbound LLM call may include unsanitized sensitive data (constitution
  Security Standards).
- Secrets read from environment variables / .NET User Secrets only; never
  committed.

**Scale/Scope**:
- ≥ 3 brands, both platforms, ≥ 6 months of dummy `Posts` data (≈ 1k–5k rows).
- 3–10 brand-brief PDFs, ≈ a few hundred chunks each after splitting.
- 1 concurrent user (PoC).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| **I. Code Quality & Clean Architecture** | Pass | Backend split into `Api` (presentation), `Application` (orchestration / SK), `Domain` (entities, routing rules), `Infrastructure` (SQL, Chroma, DIAL client, PDF ingestion). Dependencies point inward. |
| **II. Tech Stack Governance** | Pass | .NET + Semantic Kernel, React, MS SQL Server, DIAL API are all on-policy. **Chroma** as the vector store is explicitly permitted by Principle II since constitution v1.1.0 (Amendment log: 2026-05-14). The Complexity Tracking entry below is retained as an architectural record. |
| **III. AI Traceability & Prompt Discipline (NON-NEGOTIABLE)** | Pass | All LLM calls go through an `IDialChatClient` wrapper that emits structured Serilog records (correlation id, prompt template id + version, model, token usage, latency, outcome). Prompts live as files under `backend/Prompts/*.v1.md` and are loaded as Semantic Kernel prompt functions; no inline string concatenation. A `ResponseValidator` runs schema + safety + empty-output checks before the answer is returned. |
| **IV. Testing Discipline** | Pass | Unit tests cover the router classifier, SQL query builder, brief retriever, and response validator without touching the LLM. Integration tests cover the full chat pipeline against a recorded DIAL response and a seeded SQL/Chroma instance. |
| **V. Spec-Driven Development (NON-NEGOTIABLE)** | Pass | Spec at `specs/001-social-chatbot/spec.md` is approved; this plan derives from it; tasks will trace back to FRs. |
| **Performance Standards** | Pass | p95 < 5s targeted; vector queries bounded by top-K and indexed (HNSW by Chroma default); long calls cancellable via `CancellationToken` from the API. |
| **Security Standards** | Pass | Sanitization layer (`PromptSanitizer`) redacts PII patterns before any DIAL call. Secrets via env vars / User Secrets. Frontend never holds LLM keys — all LLM traffic flows through the .NET backend. |

**Gate**: Passes. Vector-store choice is sanctioned by constitution v1.1.0
(Principle II Vector store clause).

## Project Structure

### Documentation (this feature)

```text
specs/001-social-chatbot/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
│   ├── chat-api.openapi.yaml
│   └── frontend-ui-contract.md
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
backend/
├── src/
│   ├── SocialChatbot.Api/                  # ASP.NET Core minimal API host
│   │   ├── Program.cs
│   │   ├── Endpoints/ChatEndpoint.cs
│   │   └── appsettings.json                # no secrets; DIAL key via env var
│   ├── SocialChatbot.Application/          # SK orchestration, prompts, validators
│   │   ├── Chat/
│   │   │   ├── ChatService.cs              # entry point used by the endpoint
│   │   │   ├── QuestionRouter.cs           # SK function: classify question
│   │   │   ├── ResponseValidator.cs
│   │   │   └── PromptSanitizer.cs
│   │   ├── Tools/
│   │   │   ├── MetricsSqlTool.cs           # SK function for SQL queries
│   │   │   └── BriefRagTool.cs             # SK function for RAG retrieval
│   │   └── Prompts/
│   │       ├── router.v1.md
│   │       ├── metrics-answer.v1.md
│   │       ├── brief-answer.v1.md
│   │       └── mixed-answer.v1.md
│   ├── SocialChatbot.Domain/               # entities + pure logic, no I/O
│   │   ├── Posts/Post.cs
│   │   ├── Briefs/Brand.cs
│   │   ├── Briefs/BriefChunk.cs
│   │   └── Chat/QuestionCategory.cs
│   └── SocialChatbot.Infrastructure/       # adapters: SQL, Chroma, DIAL, PDF
│       ├── Sql/PostsRepository.cs
│       ├── Vector/ChromaBriefStore.cs
│       ├── Ingestion/PdfBriefIngestor.cs   # runs at startup
│       ├── Dial/DialChatClient.cs
│       └── Dial/DialEmbeddingClient.cs
└── tests/
    ├── SocialChatbot.UnitTests/
    │   ├── QuestionRouterTests.cs
    │   ├── MetricsSqlToolTests.cs
    │   ├── ResponseValidatorTests.cs
    │   └── PromptSanitizerTests.cs
    └── SocialChatbot.IntegrationTests/
        ├── ChatEndpoint_MetricsOnly_Tests.cs
        ├── ChatEndpoint_BriefOnly_Tests.cs
        ├── ChatEndpoint_Mixed_Tests.cs
        └── Fixtures/                       # recorded DIAL responses, seed SQL, sample PDFs

frontend/
├── src/
│   ├── components/
│   │   ├── ChatWindow.tsx
│   │   ├── MessageList.tsx
│   │   ├── MessageInput.tsx
│   │   └── SourceBadge.tsx                 # shows metrics / brief / mixed source
│   ├── api/chatClient.ts
│   ├── App.tsx
│   └── main.tsx
└── tests/
    └── ChatWindow.test.tsx

data/
├── seed/posts.csv                          # dummy metrics seed
└── briefs/                                 # sample PDF brand briefs
```

**Structure Decision**: Option 2 (web application). Two top-level packages —
`backend/` (.NET solution with four layered projects) and `frontend/` (React +
Vite). Plus a `data/` directory shipping the dummy seed metrics and sample
brand briefs used by the integration tests and by the demo.

## Phase 0 — Research

See [research.md](./research.md). All unknowns from the user's tech-stack
input have been resolved with concrete decisions and rationale:

- DIAL chat model selection
- DIAL embedding model selection
- Chroma deployment topology
- PDF text extraction approach and chunking strategy
- Question routing approach (SK function calling vs explicit classifier)
- Streaming vs non-streaming responses for the PoC

## Phase 1 — Design & Contracts

See:

- [data-model.md](./data-model.md) — entities, fields, validation rules
- [contracts/chat-api.openapi.yaml](./contracts/chat-api.openapi.yaml) — REST contract
- [contracts/frontend-ui-contract.md](./contracts/frontend-ui-contract.md) — UI behavior contract
- [quickstart.md](./quickstart.md) — how to run the PoC end-to-end

### Post-design Constitution re-check

Re-evaluated after writing research/data-model/contracts/quickstart: still
**passes**. No new violations introduced; the design keeps prompts versioned,
LLM calls wrapped in the logging client, responses validated, and the SQL +
vector stores behind repository/store interfaces in the Infrastructure layer.

## Complexity Tracking

| Architectural Decision (recorded) | Why Needed | Alternative Rejected Because |
|-----------------------------------|------------|------------------------------|
| Chroma used as the vector store (sanctioned by constitution v1.1.0 Principle II Vector store clause) | Need vector similarity search over PDF chunks; MS SQL Server's vector type is not GA on the LocalDB targets used for the PoC, and Semantic Kernel has a mature Chroma connector. | Storing embeddings in a SQL table with manual cosine math is slower and noisier to set up; Azure AI Search would add cloud setup overhead disproportionate to a PoC. |
| Four backend projects instead of one | Clean Architecture (constitution Principle I) — keeps Domain free of SK/SQL/HTTP dependencies, lets us unit-test routing and validation in isolation. | A single project would couple the router to the SQL driver and the SK kernel, making unit testing of business logic require integration plumbing. |
