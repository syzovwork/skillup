# Quickstart: Multi-Brand Social Media Insights Chatbot (PoC)

**Feature**: 001-social-chatbot
**Audience**: a developer who wants to run the PoC end-to-end on a single laptop.

---

## 1. Prerequisites

| Tool | Version | Why |
|------|---------|-----|
| .NET SDK | 8.x (LTS) | Backend |
| Node.js | 20.x LTS | Frontend (Vite) |
| Docker Desktop | latest | Runs Chroma + (optionally) MS SQL Server in containers |
| MS SQL Server | 2022 (Docker `mcr.microsoft.com/mssql/server:2022-latest`) or LocalDB | Structured metrics |
| DIAL API access | A valid DIAL API key and base URL | LLM + embeddings |

---

## 2. One-time setup

### 2.1 Clone and configure secrets

The DIAL API key is **never** committed. Set it via .NET User Secrets for
local dev (constitution Security Standards):

```bash
cd backend/src/SocialChatbot.Api
dotnet user-secrets init
dotnet user-secrets set "Dial:ApiKey" "<your-dial-key>"
dotnet user-secrets set "Dial:BaseUrl" "https://<your-dial-endpoint>"
dotnet user-secrets set "Dial:ChatModel" "gpt-4o-mini"
dotnet user-secrets set "Dial:EmbeddingModel" "text-embedding-3-small"
```

(For CI / containers, the same keys are read from environment variables
`Dial__ApiKey`, `Dial__BaseUrl`, etc.)

### 2.2 Start dependencies

```bash
# From repo root
docker compose up -d chroma sqlserver
```

`docker-compose.yml` (PoC) brings up:

- `chromadb/chroma:latest` on `localhost:8000` with a host-mounted volume.
- `mcr.microsoft.com/mssql/server:2022-latest` on `localhost:1433` with the
  `SA_PASSWORD` set via an env var (also kept out of source).

### 2.3 Seed the database

```bash
cd backend
dotnet run --project src/SocialChatbot.Infrastructure -- seed-sql \
    --csv ../data/seed/posts.csv
```

This creates `Brands` and `Posts` tables and bulk-loads the dummy data.

### 2.4 Ingest brand briefs

PDFs in `data/briefs/<BrandName>/*.pdf` are picked up automatically by the
API on startup (`PdfBriefIngestor` runs once if the Chroma collection is
empty, then idempotently). No manual step required — verify on first run via
log line `Ingested N chunks across M briefs`.

---

## 3. Run

```bash
# Terminal 1
cd backend
dotnet run --project src/SocialChatbot.Api
# → listening on http://localhost:5080

# Terminal 2
cd frontend
npm install
npm run dev
# → Vite dev server on http://localhost:5173
```

Open `http://localhost:5173`. The status dot should be green.

---

## 4. Try the spec's acceptance scenarios

Type each of these into the chat:

1. **Metrics-only (P1)** —
   *"Which post had the highest engagement last month?"*
   Expect: a single post identified by brand, platform, date, format, with
   the supporting engagement number. Badge: `Metrics`.

2. **Metrics-only (P1)** —
   *"Compare Instagram vs TikTok reach for April."*
   Expect: a side-by-side comparison with totals/averages. Badge: `Metrics`.

3. **Brief-only (P2)** —
   *"What does Aurora say about tone of voice?"*
   Expect: a brief-grounded answer citing Aurora. Badge: `Brand brief: Aurora`.

4. **Mixed (P3)** —
   *"Based on Aurora's guidelines and last month's data, what should I post next?"*
   Expect: a recommendation that names at least one Aurora guideline AND at
   least one metric observation. Badges: `Brand brief: Aurora` + `Metrics`.

5. **Out-of-data fallback** —
   *"What's our click-through rate?"*
   Expect: an honest "that metric isn't in this dataset" response. No badge,
   chip = `clarify` or `metrics` per router.

6. **Missing-brand brief** —
   *"What does Brand-That-Doesn't-Exist say about deadlines?"*
   Expect: "I don't have a brief on file for that brand."

---

## 5. Run the tests

```bash
# Backend
cd backend
dotnet test

# Frontend
cd ../frontend
npm test
```

Backend unit tests run without Docker. Backend integration tests require the
local SQL Server + Chroma containers and use recorded DIAL responses
(no live LLM calls in CI).

---

## 6. Observability

Every chat request emits a structured log line including:

- `correlationId`
- `category` (router decision)
- `promptTemplateId` and version (e.g. `metrics-answer@v1`)
- `model`
- `latencyMs`, `tokensIn`, `tokensOut`
- `outcome` (`ok` / `validation_failed` / `provider_error`)

Tail with:

```bash
dotnet run --project src/SocialChatbot.Api | jq -r 'select(.SourceContext|test("Chat")) | .'
```

---

## 7. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| 500 on first request with "Dial:ApiKey not configured" | User Secrets not set | Re-run §2.1 |
| Empty brief answers | Chroma collection empty | Check ingestion logs; ensure PDFs exist under `data/briefs/<Brand>/` |
| "Brand 'X' not found" | Brand name in question doesn't match `Brands.Name` | Case-insensitive substring match is supported; check seed data |
| Slow first response | Cold start of model + embedding cache | Subsequent calls warm; track p95 over ≥ 10 calls |
