# Phase 1 — Data Model: Multi-Brand Social Media Insights Chatbot

**Feature**: 001-social-chatbot
**Date**: 2026-05-14

---

## Overview

Two data domains:

- **Structured metrics** in MS SQL Server (one table: `Posts`, plus a
  reference list of brands).
- **Brand-brief chunks** in Chroma (a single collection
  `brand_briefs` of embedded text chunks with metadata).

Plus a transient **Conversation** model held in the React client and echoed
back to the backend on each request — not persisted server-side.

---

## 1. Relational schema (MS SQL Server)

### Table `Posts`

| Column           | Type                | Nullable | Notes                                                                  |
|------------------|---------------------|----------|------------------------------------------------------------------------|
| `PostId`         | `BIGINT` IDENTITY   | NOT NULL | Primary key.                                                           |
| `PostDate`       | `DATE`              | NOT NULL | The date the post was published.                                       |
| `Platform`       | `VARCHAR(16)`       | NOT NULL | One of `Instagram`, `TikTok`. Enforced via `CHECK` constraint.         |
| `Brand`          | `NVARCHAR(64)`      | NOT NULL | Brand name. Must match a value in `Brands.Name`.                       |
| `ContentFormat`  | `VARCHAR(32)`       | NOT NULL | One of `Image`, `Carousel`, `Reel`, `Story`, `Video`, `LiveStream`.    |
| `Reach`          | `INT`               | NOT NULL | ≥ 0.                                                                   |
| `Impressions`   | `INT`               | NOT NULL | ≥ Reach.                                                               |
| `Likes`          | `INT`               | NOT NULL | ≥ 0.                                                                   |
| `Comments`       | `INT`               | NOT NULL | ≥ 0.                                                                   |
| `Shares`         | `INT`               | NOT NULL | ≥ 0.                                                                   |
| `Saves`          | `INT`               | NOT NULL | ≥ 0.                                                                   |

**Indexes**:

- Clustered: `PostId`.
- `IX_Posts_Brand_Date` on `(Brand, PostDate)` — primary access pattern.
- `IX_Posts_Platform_Date` on `(Platform, PostDate)` — cross-platform comparisons.
- `IX_Posts_Date` on `(PostDate)` — date-range queries.

**Computed convenience (in queries, not stored)**:

- `Engagement = Likes + Comments + Shares + Saves`.
- `EngagementRate = Engagement * 1.0 / NULLIF(Reach, 0)`.

**Validation rules (FR-013, FR-019, FR-021)**:

- `Platform IN ('Instagram','TikTok')` — `CHECK` constraint.
- `Reach <= Impressions` — `CHECK` constraint (impressions ≥ reach by definition).
- All counter columns ≥ 0 — `CHECK` constraints.
- `Brand` must exist in `Brands.Name` — `FOREIGN KEY` (see below).

### Table `Brands`

| Column     | Type             | Nullable | Notes                                  |
|------------|------------------|----------|----------------------------------------|
| `BrandId`  | `INT` IDENTITY   | NOT NULL | PK.                                    |
| `Name`     | `NVARCHAR(64)`   | NOT NULL | Unique. Display + join key.            |
| `BriefDoc` | `NVARCHAR(256)`  | NULL     | Filename of the brand brief PDF, or `NULL` if no brief on file. |

**Indexes**: unique index on `Name`.

`Posts.Brand` references `Brands.Name` (`FOREIGN KEY (Brand) REFERENCES Brands(Name)`).

### Seed dataset (for FR-013)

The seed CSV at `data/seed/posts.csv` must contain:

- ≥ 3 brands (e.g. `Aurora`, `Nimbus`, `Vela`).
- Both platforms (`Instagram`, `TikTok`) for each brand.
- ≥ 6 months of dates ending at the current month.
- Multiple `ContentFormat` values per brand to support the "format performs best" question.
- Realistic distributions (no obvious nulls, no zero-everything rows).

---

## 2. Vector collection (Chroma)

### Collection `brand_briefs`

Each item:

- `id`: deterministic string `<brand>::<source_file>::<chunk_index>`.
- `embedding`: float vector from DIAL `text-embedding-3-small` (1536 dims).
- `document`: the chunk text (≈ 600 tokens, ≈ 80-token overlap with neighbours).
- `metadata`:
  - `brand` — must equal a `Brands.Name` value (required for FR-016 citation).
  - `source_file` — original PDF filename.
  - `page_start`, `page_end` — int page numbers within the PDF.
  - `section_hint` — best-effort heading captured during chunking; nullable.
  - `chunk_index` — int.

**Retrieval rules**:

- When the router emits a `brand`, retrieval is filtered by
  `metadata.brand == <brand>` and top-K = 4.
- When no brand is known, retrieval falls back to unfiltered top-K = 6
  followed by a "did you mean brand X?" clarification if results span more
  than one brand (FR-008).

---

## 3. Domain entities (in code, `SocialChatbot.Domain`)

These are pure C# types — no I/O, no SK references. They are the unit of
exchange between the Application and Infrastructure layers.

### `Post`

```csharp
public sealed record Post(
    long PostId,
    DateOnly PostDate,
    Platform Platform,
    string Brand,
    ContentFormat ContentFormat,
    int Reach,
    int Impressions,
    int Likes,
    int Comments,
    int Shares,
    int Saves)
{
    public int Engagement => Likes + Comments + Shares + Saves;
}
```

### Enums

```csharp
public enum Platform { Instagram, TikTok }

public enum ContentFormat { Image, Carousel, Reel, Story, Video, LiveStream }

public enum QuestionCategory { Metrics, Brief, Mixed, Clarify }
```

### `BriefChunk`

```csharp
public sealed record BriefChunk(
    string Id,
    string Brand,
    string SourceFile,
    int PageStart,
    int PageEnd,
    string Text,
    string? SectionHint);
```

### `RoutingDecision`

```csharp
public sealed record RoutingDecision(
    QuestionCategory Category,
    string? Brand,
    string Reason);
```

### `ChatTurn` and `ChatRequest`

```csharp
public enum ChatRole { User, Assistant }

public sealed record ChatTurn(ChatRole Role, string Content);

public sealed record ChatRequest(
    string Message,
    IReadOnlyList<ChatTurn> History);  // last N turns from the client

public sealed record ChatResponse(
    string Answer,
    QuestionCategory Category,
    IReadOnlyList<string> SourcesUsed,   // e.g. ["metrics", "brief:Aurora"]
    string CorrelationId);
```

---

## 4. Validation rules summary

| Rule | Source FR / SC | Enforced where |
|------|----------------|----------------|
| Platform ∈ {Instagram, TikTok} | FR-009 | SQL `CHECK`, `Platform` enum |
| Reach ≤ Impressions | sanity | SQL `CHECK` |
| Brand exists in `Brands.Name` | FR-014, FR-017 | SQL FK + router check |
| Posts cover ≥ 6 months, multi-brand, both platforms | FR-013 | Seed CSV + integration test |
| Brief chunk has `metadata.brand` set | FR-016, SC-004 | Ingestion writes it; retriever asserts non-null |
| Question router returns one of {Metrics, Brief, Mixed, Clarify} | FR-004 | JSON schema validator on router output |
| Numbers in answer present in tool result | FR-019 | `ResponseValidator.NumericGroundingCheck` |
| Brand brief absent ⇒ no fabricated content | FR-017 | `BriefRagTool` returns empty; answer prompt instructs honest fallback |
| Out-of-data range ⇒ honest fallback | FR-021, SC-007 | `MetricsSqlTool` exposes dataset bounds; answer prompt acknowledges |

---

## 5. Notable non-decisions (deferred past PoC)

- No `Conversations` table — conversation history is client-resident
  (research §8). When multi-user support is added, this becomes a real entity.
- No per-user identity model — PoC is single-tenant (spec Assumptions).
- No analytics on the chatbot's own usage (latency / accuracy tracking) at
  the data-model level; these are emitted as structured logs only.
