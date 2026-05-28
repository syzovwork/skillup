# Frontend UI Contract

**Feature**: 001-social-chatbot
**Surface**: React single-page chat UI consumed by social media managers.

This document is the contract between the React client and the backend. It
states what the UI MUST display and how it MUST behave so that the spec's
acceptance scenarios and constitution-level guarantees hold end-to-end.

---

## Screens

The PoC has a single screen: `/`.

### Layout

- **Header** (top): app title `"Social Insights Chat"` and a small status
  dot reflecting `/api/health` (green / amber / red).
- **Message list** (center, scrollable): chronological list of user and
  assistant turns. Newest at the bottom.
- **Message input** (bottom): a single-line text input + Send button. Enter
  submits; Shift+Enter inserts a newline.

### Empty state

When `history` is empty, show a one-line tip:
*"Ask about your Instagram or TikTok performance, or about a brand brief."*

---

## Message rendering

Every assistant message MUST render the following sub-elements:

1. **Answer text** — markdown allowed (lists, bold). HTML stripped.
2. **Source badges** — one badge per entry in `sourcesUsed`:
   - `metrics` → blue badge labelled `Metrics`.
   - `brief:<Brand>` → green badge labelled `Brand brief: <Brand>`.
3. **Category chip** — small grey chip showing `category` value
   (`metrics`, `brief`, `mixed`, `clarify`).
4. **Correlation id** — visible on hover/long-press of the message bubble,
   for support and traceability (constitution Principle III).
5. **Clarifying question** — when `category == "clarify"`, render
   `clarifyingQuestion` prominently and treat the user's next message as the
   answer to it.

---

## Request lifecycle

On Send:

1. Append `{role: "user", content: <text>}` to client-side `history`.
2. POST `/api/chat` with `{ message, history }` where `history` is the last
   **10** turns (per research §8).
3. While in flight: show a typing indicator under the user message.
4. On 200: append `{role: "assistant", content: response.answer}` plus the
   badges/chip metadata from the response.
5. On 4xx/5xx: append an assistant turn with text
   *"Something went wrong. (correlationId: ...)"* and a red error badge.
   Do NOT silently retry.
6. On timeout (> 30 s): cancel the request and show the same error pattern.

Requests MUST be cancellable: a second Send while a request is in flight
cancels the first and starts the new one (uses `AbortController`).

---

## Configuration

- API base URL is read from `VITE_API_BASE_URL` (default
  `http://localhost:5080`).
- The frontend MUST NOT read any DIAL key, model name, or other LLM provider
  configuration. The .NET backend is the only holder of LLM credentials.
  (Constitution Security Standards.)

---

## Accessibility

- Input has an accessible label `"Your question"`.
- Message bubbles use ARIA roles `log` (list) and `listitem` (each turn).
- Source badges include readable text (not icons only).
- Status dot includes an `aria-label` describing the current health state.

---

## Out of scope for the PoC

- User authentication, multi-user separation.
- Persisting chat history across page reloads.
- Streaming token-by-token rendering (research §7 — added later if needed).
- File uploads from the UI (brand briefs are loaded server-side from
  `data/briefs/` at startup).
- Theming, localization (English only per spec Assumptions).
