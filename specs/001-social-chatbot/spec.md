# Feature Specification: Multi-Brand Social Media Insights Chatbot

**Feature Branch**: `001-social-chatbot`
**Created**: 2026-05-14
**Status**: Draft
**Input**: User description: "Build a conversational AI chatbot for social media managers who work with multiple brands on Instagram and TikTok. The core problem: social media managers spend too much time manually switching between analytics dashboards to understand content performance. There is no unified way to ask natural language questions across platforms or cross-reference performance data with brand-specific guidelines. The chatbot must support three types of questions: (1) Data questions over structured metrics, (2) Document questions over brand briefs, (3) Mixed questions combining both. The structured data covers: post date, platform (Instagram or TikTok), brand name, content format, reach, impressions, likes, comments, shares, saves. Data is realistic dummy data for the PoC. The unstructured data consists of PDF brand briefs containing tone of voice guidelines, content requirements, and campaign deadlines. The user interacts via a chat interface. The system must correctly route each question to the appropriate data source, or combine both when needed."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ask data questions across platforms (Priority: P1)

A social media manager opens the chatbot and asks natural-language questions
about post performance across the brands they manage on Instagram and TikTok.
Instead of opening separate analytics dashboards, they get a single,
conversational answer summarizing the metrics, including comparisons across
platforms, content formats, brands, and time ranges.

**Why this priority**: This is the primary value driver — it eliminates the
dashboard-switching pain that motivates the project. Without it, the chatbot
has no reason to exist. It is also the foundation on which document and mixed
questions are layered.

**Independent Test**: Load the realistic dummy metrics dataset, ask each of
the following without any brand-brief PDFs ingested, and confirm answers are
factually correct against the dataset:

- "Which post had the highest engagement last month?"
- "Compare Instagram vs TikTok reach for April."
- "What content format performs best on TikTok?"

**Acceptance Scenarios**:

1. **Given** the dummy metrics dataset is loaded, **When** the user asks
   *"Which post had the highest engagement last month?"*, **Then** the chatbot
   returns the single post (brand, platform, date, format) whose engagement
   (likes + comments + shares + saves) is highest in the last full calendar
   month and shows the supporting numbers.
2. **Given** the dummy metrics dataset is loaded, **When** the user asks
   *"Compare Instagram vs TikTok reach for April"*, **Then** the chatbot
   returns total and average reach for each platform for April with a clear
   side-by-side comparison.
3. **Given** the dummy metrics dataset is loaded, **When** the user asks a
   question that requires filtering by brand (e.g. *"Which brand had the most
   saves last week?"*), **Then** the chatbot scopes the answer to that filter
   correctly.

---

### User Story 2 - Ask document questions over brand briefs (Priority: P2)

The manager asks questions about a specific brand's guidelines that are
contained in PDF brand briefs (tone of voice, content requirements, campaign
deadlines). The chatbot returns the relevant information grounded in the brief,
citing which brand brief the answer came from.

**Why this priority**: Brand-brief lookup is the second most common pain
point — managers re-read PDFs to find the same answers repeatedly. This story
is valuable on its own (a brand-brief Q&A assistant) and is a prerequisite for
the mixed-question story.

**Independent Test**: Ingest a small set of dummy brand-brief PDFs (at least 3
brands) and ask:

- "What does Brand X say about tone of voice?"
- "What are the deliverable deadlines for Brand Y?"

Answers must reflect the actual content of the briefs and cite the source
brand brief.

**Acceptance Scenarios**:

1. **Given** Brand X's brief has been ingested and explicitly states a tone-of-
   voice guideline, **When** the user asks *"What does Brand X say about tone of
   voice?"*, **Then** the chatbot returns that guideline grounded in the brief
   and cites Brand X's brief as the source.
2. **Given** Brand Y's brief lists campaign deadlines, **When** the user asks
   *"What are the deliverable deadlines for Brand Y?"*, **Then** the chatbot
   lists the deadlines from Brand Y's brief.
3. **Given** the user asks about a brand whose brief has not been ingested,
   **When** they ask a brief-related question, **Then** the chatbot states it
   has no brief for that brand rather than fabricating an answer.

---

### User Story 3 - Ask mixed questions combining metrics and briefs (Priority: P3)

The manager asks a strategic question that requires both the brand's
guidelines and the recent performance data — for example,
*"Based on Brand X guidelines and last month's data, what type of post should
I create next?"*. The chatbot reasons over both sources and returns a single,
grounded recommendation that references the brief constraints and the metric
evidence.

**Why this priority**: This is the differentiator of the product — neither a
dashboard nor a document-search tool can answer this alone. It builds on
stories 1 and 2 and is most valuable once those work reliably.

**Independent Test**: With both metrics data and Brand X's brief loaded, ask
the mixed example above and confirm the response (a) cites at least one
specific guideline from Brand X's brief, (b) cites at least one metric-based
insight derived from last month's data for Brand X, and (c) makes a
recommendation consistent with both.

**Acceptance Scenarios**:

1. **Given** Brand X's brief and last month's metrics for Brand X are loaded,
   **When** the user asks *"Based on Brand X guidelines and last month's data,
   what type of post should I create next?"*, **Then** the chatbot returns a
   recommendation that explicitly references (a) at least one brief-derived
   constraint and (b) at least one metric-derived observation.
2. **Given** Brand X's brief exists but no metrics are available for Brand X,
   **When** the user asks a mixed question, **Then** the chatbot states that
   metrics are missing and answers only with brief-grounded guidance rather
   than fabricating performance numbers.

---

### Edge Cases

- The user asks an ambiguous question that could apply to either source
  (e.g. *"What should I post next?"* with no brand named). The chatbot asks a
  brief clarifying question (e.g. *"Which brand?"*) instead of guessing.
- The user asks about a date range with no data (e.g. *"Show me last year's
  performance"* when only the last 6 months are in the dataset). The chatbot
  states the range covered by the data rather than returning empty results
  silently.
- The user asks for a metric the dataset does not contain (e.g. *"What's my
  click-through rate?"*). The chatbot states which metrics are available.
- A brand brief is uploaded but unreadable (scanned image, corrupted). The
  chatbot surfaces the ingestion failure and treats the brand as having no
  brief for question-answering.
- Two brands have similar names. The chatbot disambiguates by asking which one
  the user means before answering.
- The user's question is off-topic (e.g. weather, unrelated coding help). The
  chatbot declines politely and restates its scope.

## Requirements *(mandatory)*

### Functional Requirements

**Chat interaction**

- **FR-001**: The system MUST provide a chat interface where the user can
  send a natural-language question and receive a natural-language answer in
  the same conversation thread.
- **FR-002**: The system MUST preserve at least the current conversation's
  prior turns and use them as context when interpreting follow-up questions
  within that session.
- **FR-003**: The system MUST display, alongside each answer, a clear
  indication of which source(s) were used: structured metrics, brand brief(s)
  (with brand name), or both.

**Question routing**

- **FR-004**: The system MUST classify every incoming user question into one
  of three categories before answering: metrics-only, brief-only, or mixed.
- **FR-005**: When the question is metrics-only, the system MUST answer
  using only the structured metrics dataset.
- **FR-006**: When the question is brief-only, the system MUST answer using
  only ingested brand-brief content.
- **FR-007**: When the question is mixed, the system MUST consult both the
  metrics dataset and the relevant brand brief(s) and produce a single answer
  that is consistent with both.
- **FR-008**: If routing confidence is low or the brand is ambiguous, the
  system MUST ask a clarifying question instead of guessing.

**Structured metrics**

- **FR-009**: The system MUST store and query a structured dataset of social
  media post records with at least these attributes: post date, platform
  (Instagram or TikTok), brand name, content format, reach, impressions,
  likes, comments, shares, saves.
- **FR-010**: The system MUST be able to filter and aggregate metrics by any
  combination of: brand, platform, content format, and time range
  (day / week / month / arbitrary range).
- **FR-011**: The system MUST be able to rank posts by a metric or composite
  metric (e.g. engagement = likes + comments + shares + saves) and return the
  top result(s) with their context (brand, platform, date, format).
- **FR-012**: The system MUST be able to compare two groups (e.g. Instagram
  vs TikTok, two brands, two formats) over a stated time range and return the
  comparison in a way the user can read at a glance.
- **FR-013**: The dataset MUST be realistic dummy data covering multiple
  brands, both platforms, multiple content formats, and a time range of at
  least 6 months ending at the current date.

**Brand briefs (unstructured)**

- **FR-014**: The system MUST accept brand-brief documents in PDF format,
  one or more per brand, and associate each brief with a specific brand
  identifier used in the metrics dataset.
- **FR-015**: The system MUST be able to answer questions about brief
  content covering at minimum: tone of voice, content requirements, and
  campaign deadlines.
- **FR-016**: Answers grounded in a brand brief MUST cite the brand name
  whose brief was used.
- **FR-017**: The system MUST refuse to invent brief content for a brand
  whose brief has not been ingested; instead it MUST state that no brief is
  on file for that brand.

**Mixed reasoning**

- **FR-018**: For mixed questions, the answer MUST include (a) at least one
  observation derived from the metrics data and (b) at least one constraint
  or guideline drawn from the relevant brand brief, both clearly attributable
  to their source.

**Quality & safety**

- **FR-019**: The system MUST not fabricate metric values; numbers in
  answers MUST be traceable to the underlying dataset.
- **FR-020**: The system MUST decline politely and restate its scope when
  asked questions outside social media performance and brand briefs.
- **FR-021**: When the user asks about a date range, brand, platform, or
  metric not present in the data, the system MUST say so explicitly rather
  than returning an empty or invented result.

### Key Entities

- **Post Metric Record**: A single row representing one published post's
  performance. Attributes: post date, platform (Instagram or TikTok), brand,
  content format, reach, impressions, likes, comments, shares, saves.
- **Brand**: A managed brand with a unique name/identifier used to link
  metric records to its brief(s).
- **Brand Brief**: A PDF document belonging to a brand, containing tone of
  voice, content requirements, and campaign deadlines.
- **Conversation**: An ordered sequence of user questions and chatbot
  answers within a single session, including the source attribution per
  answer.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: On a benchmark set of at least 30 test questions covering all
  three categories (metrics-only, brief-only, mixed) in equal proportion, the
  chatbot returns a factually correct answer for at least **90%** of cases,
  judged against the known dummy dataset and the brief contents.
- **SC-002**: The chatbot correctly classifies the question category
  (metrics / brief / mixed) for at least **95%** of the benchmark set.
- **SC-003**: For at least **90%** of the benchmark set, the user sees the
  first part of the answer within **5 seconds** of submitting their question.
- **SC-004**: For every answer that uses a brand brief, the response cites
  the brand whose brief was used in **100%** of cases.
- **SC-005**: For mixed questions, **100%** of answers reference both at
  least one brief-derived element and at least one metric-derived element.
- **SC-006**: In a usability check with a small panel of social media
  managers, at least **80%** report they would prefer the chatbot to manually
  switching between dashboards for the question types in scope.
- **SC-007**: When asked questions outside scope or about missing data, the
  chatbot returns an honest "not available / out of scope" response rather
  than a fabricated answer in at least **95%** of trials.

## Assumptions

- **Audience**: Users are professional social media managers comfortable
  with brand and platform terminology; they are not expected to write queries
  or filters by hand.
- **Scope of platforms**: Only Instagram and TikTok are in scope for the
  PoC. Other platforms (YouTube, X/Twitter, LinkedIn, etc.) are out of scope.
- **Scope of data**: The PoC uses realistic dummy data, not live integrations
  with the Instagram or TikTok APIs. Real-data integration is out of scope.
- **Metric set**: Only the listed metrics (reach, impressions, likes,
  comments, shares, saves) are available. Click-through rate, follower growth,
  ad spend, etc. are out of scope.
- **Brief format**: Brand briefs arrive as digitally generated, text-based
  PDFs. OCR of scanned-image PDFs is out of scope for the PoC; such files are
  treated as ingestion failures.
- **Users and access**: The PoC is a single-tenant demonstration; per-user
  authentication, role-based permissions, and multi-tenant data isolation are
  out of scope.
- **Language**: All briefs and questions are in English for the PoC.
- **Session persistence**: Conversation history is preserved within a single
  active session. Long-term cross-session memory is out of scope for the PoC.
- **Date semantics**: Relative dates in questions ("last month", "April",
  "last week") are interpreted relative to today's date as configured on the
  server.
