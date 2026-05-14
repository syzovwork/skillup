<!--
SYNC IMPACT REPORT
==================
Version change: (initial template) → 1.0.0
Bump rationale: First ratified version — all placeholders replaced with concrete principles.

Modified principles:
  - [PRINCIPLE_1_NAME] → I. Code Quality & Clean Architecture
  - [PRINCIPLE_2_NAME] → II. Tech Stack Governance
  - [PRINCIPLE_3_NAME] → III. AI Traceability & Prompt Discipline (NON-NEGOTIABLE)
  - [PRINCIPLE_4_NAME] → IV. Testing Discipline
  - [PRINCIPLE_5_NAME] → V. Spec-Driven Development (NON-NEGOTIABLE)

Added sections:
  - Performance & Security Standards (replaces [SECTION_2_NAME])
  - Development Workflow & Quality Gates (replaces [SECTION_3_NAME])

Removed sections: none

Templates requiring updates:
  - ✅ .specify/templates/plan-template.md — "Constitution Check" gate references this file generically; no edits required.
  - ✅ .specify/templates/spec-template.md — no constitution-specific placeholders detected; no edits required.
  - ✅ .specify/templates/tasks-template.md — task categories compatible (unit/integration tests, AI pipeline); no edits required.
  - ✅ .specify/templates/checklist-template.md — generic; no edits required.

Follow-up TODOs: none
-->

# SkillUp AI Chatbot PoC Constitution

## Core Principles

### I. Code Quality & Clean Architecture

All code MUST follow clean architecture boundaries: domain logic is isolated from
infrastructure (LLM clients, database, HTTP). Separation of concerns is enforced at
project/folder level — presentation, application, domain, and infrastructure
layers do not leak types across boundaries except via explicit contracts. Naming
MUST be intention-revealing (no abbreviations like `mgr`, `svc2`, `tmp`); types and
methods read as domain vocabulary. Dependencies point inward (toward the domain),
never outward. Public APIs and modules require XML doc comments only where the
"why" is non-obvious.

**Rationale**: A PoC that cannot be refactored cleanly into production is a sunk
cost. Architectural discipline keeps the option open at marginal extra cost.

### II. Tech Stack Governance

The approved stack is fixed for this PoC and MUST NOT be substituted without an
amendment to this constitution:

- **Backend**: .NET (current LTS) / C# using the **Semantic Kernel SDK** as the
  orchestration layer for all LLM interactions, function/tool calling, and memory
  abstractions.
- **Frontend**: **React** (functional components + hooks).
- **Structured data**: **Microsoft SQL Server**. Schema changes go through
  migrations; no ad-hoc DDL.
- **LLM access**: **Azure AI Foundry** and/or **EPAM DIAL API**. No direct calls
  to other providers (OpenAI, Anthropic, etc.) from application code.

Introducing a new framework, database engine, or LLM provider requires a written
justification in `plan.md` under "Complexity Tracking" and a constitution
amendment (MINOR bump minimum).

**Rationale**: Stack lock-in for a PoC prevents accidental sprawl and keeps the
evaluation comparable to the production target environment.

### III. AI Traceability & Prompt Discipline (NON-NEGOTIABLE)

Every LLM interaction MUST satisfy all of the following:

1. **Traceability**: each call emits a structured log/telemetry record containing
   correlation ID, prompt template ID + version, model identifier, token usage,
   latency, and outcome (success / validation failure / provider error). Logs
   MUST be queryable per user session.
2. **Versioned prompts**: prompts are stored as files under source control
   (e.g., `prompts/<name>.v<N>.md` or equivalent Semantic Kernel prompt
   functions). Inline string-concatenated prompts in application code are
   PROHIBITED.
3. **Response validation**: every LLM response is validated before being
   returned to the user. Validation includes, at minimum: schema/format check
   for structured outputs, refusal/empty-output detection, and a content-safety
   gate. Unvalidated raw model output MUST NOT cross the API boundary.

**Rationale**: Without traceability and prompt versioning, AI behavior is
unreproducible and unauditable. Response validation is the last line of defense
against hallucinations and unsafe content reaching end users.

### IV. Testing Discipline

- **Unit tests** are MANDATORY for all business/domain logic. Tests do not
  depend on the LLM, the database, or network I/O — collaborators are stubbed at
  the port boundary.
- **Integration tests** are MANDATORY for AI pipeline components: prompt
  rendering, Semantic Kernel plan execution, retrieval/vector-search wiring,
  response validators, and DIAL/Azure AI Foundry client adapters. Integration
  tests MAY use recorded/replayed LLM responses to stay deterministic.
- A pull request that adds or modifies business logic or AI pipeline behavior
  without corresponding tests MUST be rejected at review.
- Coverage is a signal, not a target; failing-test-first is preferred but not
  mandated for this PoC tier.

**Rationale**: AI pipelines fail silently far more often than traditional code.
Integration tests around the pipeline edges catch regressions that unit tests
structurally cannot.

### V. Spec-Driven Development (NON-NEGOTIABLE)

No implementation work begins without (a) an approved `spec.md` and (b) an
approved `plan.md` for the feature. "Approved" means reviewed by the project
owner and committed to the repository. Tasks generated via `/speckit-tasks` MUST
trace back to requirements in the spec. Drift between code and spec is a defect
and MUST be resolved by amending the spec or reverting the code — not by
silently diverging.

**Rationale**: SDD compliance is the contract that makes this project's
documentation, traceability, and AI-assisted workflow worth the overhead.

## Performance & Security Standards

**Performance**:

- The chatbot MUST respond within acceptable interactive latency for a PoC:
  target p95 end-to-end response time **< 5 seconds** for typical queries.
  Streaming responses count from first-token-emitted.
- Vector / semantic search queries MUST be optimized: appropriate indexes
  (e.g., HNSW or provider-native), top-K bounded, embedding dimensionality
  pinned per collection. Queries returning > 200 ms at p95 MUST be profiled.
- Long-running LLM calls MUST be cancellable from the client.

**Security**:

- **No sensitive data** (PII, secrets, internal credentials, customer records
  flagged as confidential) is sent to an LLM provider without sanitization /
  redaction. A sanitization layer MUST run on all outbound prompts.
- **API keys and connection strings MUST NEVER be hardcoded** or committed.
  Secrets live in environment variables, Azure Key Vault, or .NET User Secrets
  during local development. Repository scans for secret patterns are part of
  CI.
- All inputs from the user are treated as untrusted and pass through prompt-
  injection mitigations (system-prompt isolation, allow-listed tool calls).
- The frontend never holds LLM provider keys; all LLM traffic flows through the
  .NET backend.

## Development Workflow & Quality Gates

- Every feature follows: `/speckit-specify` → `/speckit-clarify` (if needed) →
  `/speckit-plan` → `/speckit-tasks` → `/speckit-implement`.
- The `plan.md` "Constitution Check" gate MUST be filled and pass before
  Phase 0 research and again after Phase 1 design. Violations require entries
  in the "Complexity Tracking" table with explicit justification.
- Pull requests require: passing unit + integration tests, no hardcoded
  secrets, all new LLM calls logged per Principle III, and reference to the
  governing spec/plan.
- Code review MUST verify constitution compliance, not only correctness.

## Governance

This constitution supersedes ad-hoc practices, individual preferences, and
prior informal conventions. In a conflict between this document and any other
guidance (READMEs, chat history, agent prompts), this document wins until
amended.

**Amendment procedure**:

1. Propose the change in a PR that edits `.specify/memory/constitution.md`.
2. Include a Sync Impact Report (see HTML comment at top of this file) listing
   the version bump rationale and all dependent templates/docs.
3. Update all templates flagged in the report in the same PR.
4. Project owner approval is required to merge.

**Versioning policy** (semantic):

- **MAJOR**: backward-incompatible removal or redefinition of a principle, or
  removal of a governance section.
- **MINOR**: new principle or section added; material expansion of guidance.
- **PATCH**: clarifications, wording, typo fixes, non-semantic refinements.

**Compliance review**: every PR description MUST state "Constitution: vX.Y.Z
checked" and call out any complexity-tracking justifications. Periodic reviews
(at least once per milestone) reconcile drift between code and constitution.

**Version**: 1.0.0 | **Ratified**: 2026-05-14 | **Last Amended**: 2026-05-14
