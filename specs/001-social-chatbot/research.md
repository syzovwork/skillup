# Phase 0 — Research: Multi-Brand Social Media Insights Chatbot

**Feature**: 001-social-chatbot
**Date**: 2026-05-14

This document resolves every "NEEDS CLARIFICATION" in the Technical Context
and records the rationale for each technology / pattern choice.

---

## 1. DIAL chat model

**Decision**: Use `gpt-4o-mini` (or the closest equivalent the project's DIAL
deployment exposes under the OpenAI-compatible chat-completions interface) as
the **default** chat model for routing and answer synthesis. Make the model
identifier configurable via `Dial:ChatModel` in `appsettings.json` so it can
be swapped without code changes.

**Rationale**: `gpt-4o-mini` is fast enough to meet the p95 < 5s target for
short questions while being capable enough for the routing classification and
short answer synthesis used here. DIAL exposes OpenAI-compatible models so the
SK OpenAI connector works unchanged once pointed at the DIAL base URL.

**Alternatives considered**:
- `gpt-4o` — higher quality but ~2–3× slower and more expensive; reserve for
  evaluation comparisons, not the default.
- A non-OpenAI family model via DIAL (e.g. Anthropic / Mistral) — works but
  requires a different SK connector; out of scope for the PoC.

---

## 2. DIAL embedding model

**Decision**: Use `text-embedding-3-small` (1536 dims) via DIAL as the default
embedding model for brand-brief chunks. Configurable via
`Dial:EmbeddingModel`.

**Rationale**: 1536-dim embeddings give strong recall for short PDF chunks at
a fraction of the storage/CPU cost of `text-embedding-3-large`. Chroma's
default HNSW index handles this dimensionality comfortably for the expected
volume (≤ ~5k chunks for the PoC).

**Alternatives considered**:
- `text-embedding-3-large` — better recall on long-form text but overkill for
  ~200–1000 char chunks; doubles storage with little gain at PoC scale.
- `text-embedding-ada-002` — legacy; equal quality is available cheaper from
  the v3 family.

---

## 3. Chroma deployment topology

**Decision**: Run Chroma as a single local **Docker container** during
development (`chromadb/chroma:latest`), persisted to a host volume so
embeddings survive restarts. The .NET backend talks to it over HTTP on
`http://localhost:8000`. Connection settings come from configuration; no
cloud-hosted Chroma is used for the PoC.

**Rationale**: A local container is the simplest, lowest-friction option for
a single-developer PoC. It matches how Semantic Kernel's Chroma connector is
documented and gives us a real HTTP boundary (useful for integration tests
that swap in a recorded response).

**Alternatives considered**:
- Chroma in-process via Python embedding — not available for .NET; would
  require running a Python sidecar.
- Cloud-hosted Chroma — adds account setup and recurring cost for no PoC
  benefit.
- SQL-stored embeddings + cosine — workable but slower and pushes Infra
  responsibility into the data layer.

---

## 4. PDF text extraction

**Decision**: Extract text with **UglyToad.PdfPig** (MIT licence, pure .NET).
For each PDF: extract per-page text, strip headers/footers heuristically,
concatenate, then chunk.

**Rationale**: PdfPig is the standard pure-.NET PDF text extractor — no
external runtime, no licensing entanglement, good extraction quality on
digitally generated PDFs. The spec already declares scanned-image PDFs out of
scope, so no OCR step is required.

**Alternatives considered**:
- iText 7 — strong extractor but AGPL/commercial dual-licensed; avoided.
- PDFsharp — primarily for PDF generation, weaker extraction.
- External `pdftotext` (poppler) — adds a native dependency to the dev setup.

---

## 5. Chunking strategy

**Decision**: Split brief text into ~600-token chunks with ~80-token overlap,
preserving paragraph boundaries when possible. Attach metadata
`{ brand, source_file, page_start, page_end, section_hint }` to each chunk.

**Rationale**: 600/80 is the sweet spot most retrieval evals settle on for
1536-dim embeddings of business/marketing prose — long enough to carry one
guideline's full context, short enough that top-K = 4 fits the answer-prompt
budget. Metadata enables citation by brand name (FR-016) and lets the
retriever filter by brand when the question names one.

**Alternatives considered**:
- One chunk per page — too coarse; loses recall for short guidelines.
- One chunk per paragraph — too fine; balloons index size and fragments
  multi-paragraph guidelines.

---

## 6. Question routing

**Decision**: Implement routing as a **dedicated Semantic Kernel prompt
function** (`router.v1.md`) that returns a small JSON object:
`{ "category": "metrics" | "brief" | "mixed" | "clarify", "brand": "<name|null>", "reason": "<short>" }`.
The router runs first; the answer is then produced by one of three
category-specific prompt functions (`metrics-answer.v1.md`,
`brief-answer.v1.md`, `mixed-answer.v1.md`). Tools (`MetricsSqlTool`,
`BriefRagTool`) are exposed to the latter two via SK function calling so the
LLM only requests data it actually needs.

**Rationale**: An explicit classifier step is easier to evaluate and log than
implicit routing via "let the LLM pick a tool" — critical for SC-002 (≥ 95%
routing accuracy) and constitution Principle III (traceability). The router
output is a small, fixed JSON schema, making it cheap to validate.

**Alternatives considered**:
- Pure SK function-calling with both tools exposed and no classifier — works
  but routing decisions become opaque; harder to measure SC-002.
- Hand-written regex/keyword classifier — fast but brittle on real
  managers' phrasing; loses on edge cases the LLM handles well.

---

## 7. Streaming vs non-streaming responses

**Decision**: Non-streaming for v1. Return the full validated response in a
single HTTP 200 with JSON body. Add streaming (SSE) later if the latency
budget is tight.

**Rationale**: The PoC's p95 < 5s budget is achievable without streaming for
the typical short answers expected here, and non-streaming keeps the
ResponseValidator (Principle III) trivial — it can inspect and reject the
whole output before sending. Adding SSE later only requires changing the
endpoint and the React client, not the orchestration.

**Alternatives considered**:
- SSE streaming from the start — better perceived latency but complicates
  validation (must validate after streaming completes or per-chunk).

---

## 8. Conversation memory

**Decision**: Maintain conversation history **client-side** in React state.
On each request, the React client posts the last N turns (default N = 10)
along with the new user message. The backend is stateless between requests.

**Rationale**: PoC scope (FR-002, Assumptions) requires only intra-session
memory, no cross-session persistence. Keeping the server stateless removes
the need for a `Conversations` table for the PoC and makes the API trivial to
test.

**Alternatives considered**:
- Server-side session store (in-memory) — adds state for no observable user
  benefit at PoC scale.
- SK ChatHistory persisted to SQL — premature for the PoC; revisit when
  multi-user support is added.

---

## 9. SQL query generation

**Decision**: Do **not** let the LLM write raw SQL. Implement
`MetricsSqlTool` as a Semantic Kernel function with a typed parameter object
(brand, platform, content_format, date_from, date_to, metric, aggregation,
top_n, comparison_axis). The LLM selects values; the tool composes the SQL
using parameterized queries against `Posts`.

**Rationale**: Keeps SQL safe (no injection surface), enforces FR-019 (numbers
are traceable to the dataset), and makes the tool unit-testable without an
LLM. Constitution Principle III's traceability is satisfied — every metrics
answer can show the exact tool invocation parameters in logs.

**Alternatives considered**:
- LLM-generated SQL via "text-to-SQL" — adds an injection/safety surface and
  a quality wildcard; unnecessary because the schema is fixed and tiny.

---

## 10. Response validation

**Decision**: `ResponseValidator` runs three checks before any answer leaves
the API:

1. **Empty/refusal detection** — non-empty content, no canned-refusal
   substring pattern when the question was in scope.
2. **Source-attribution check** — if the router picked `brief` or `mixed`, the
   answer text MUST contain a brand-name citation (FR-016, SC-004); if
   `mixed`, the response MUST also include at least one metric value (SC-005).
3. **Numeric grounding check** — any number in the answer that purports to be
   a metric value MUST appear in the SQL tool's last result set
   (FR-019).

Failures are returned to the orchestrator, which retries once with a stricter
"include sources" instruction, then surfaces a graceful fallback message.

**Rationale**: Direct execution of constitution Principle III's
"responses must be validated before returning to the user".

**Alternatives considered**:
- Trust the LLM and post-log only — violates the constitution.

---

## 11. Sanitization layer

**Decision**: `PromptSanitizer` redacts common PII patterns (email addresses,
phone numbers, long digit sequences likely to be card/account numbers) from
the user message and from any document text before it is included in a DIAL
prompt. The seed dataset and briefs used in the PoC contain no real PII, but
the sanitizer is wired in unconditionally so the security pattern is in place
from day one.

**Rationale**: Constitution Security Standards: "No sensitive data is sent to
an LLM provider without sanitization."

**Alternatives considered**:
- Skip sanitization for the PoC because dummy data is safe — rejected: the
  point of the security principle is to make the pattern non-optional.

---

## Open items / TODOs

None — all unknowns resolved.
