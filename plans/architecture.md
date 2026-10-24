# AI Chat UI Architecture Blueprint

## 1. Core Constraints & Tech Stack
*   **Format:** Strictly a single `index.html` file for maximum portability.
*   **Reactivity:** Vue 3 (Global Build via CDN). Components will be defined using `<template id="...">` blocks within the HTML.
*   **Storage:** Dexie.js (IndexedDB wrapper) for local, persistent storage.
*   **Styling:** Tailwind CSS (Runtime via CDN).
*   **Rendering:** Marked.js + highlight.js (Markdown parsing and code highlighting).
*   **Security:** DOMPurify (Sanitization of marked output).
*   **Icons:** Lucide (via CDN).
*   **PDF Parsing:** PDF.js (via CDN) for client-side text extraction.

## 2. Storage Schema (IndexedDB / Dexie)
Database name: **`Calyx`** (schema version 1). The legacy `CalyxDB` (integer `++id` keys) is migrated once on boot — see §7 — and deleted after a verified copy.

*   **`id`** fields across all tables are **string UUIDs** (`crypto.randomUUID()`), not auto-increment integers. This allows records from independent databases (imports, future sync) to merge without ID collisions.
*   Every mutable record carries an **`updatedAt`** timestamp (ms epoch), used for last-write-wins conflict resolution on import merge and future remote sync.
*   **`chats`**: `id`, `title`, `createdAt`, `updatedAt`, `isPinned`, `systemPrompt` (override).
*   **`messages`**: `id`, `chatId` (indexed), `role` (user/assistant), `content`, `createdAt`, `updatedAt`, `attachments` (array of objects: `{ type, name }` — image attachments additionally carry `data`/`mimeType` in-memory only, never persisted with the message).
*   **`files`**: `id`, `chatId` (indexed), `name`, `type` (`text`/`pdf`), `hash` (indexed, SHA-256 of normalized text, used for dedup), `text`, `tokens` (estimated), `createdAt`, `updatedAt`. Chat-scoped document attachments, decoupled from individual messages so they persist across every turn.
*   **`settings`**: Key-value pairs (`{ key, value, updatedAt }`).
    *   `systemPrompt` (Global)
    *   `openaiApiKey`, `anthropicApiKey`, `geminiApiKey` — **secret**, see §7.1
    *   `selectedModel` (Default: `gpt-5.6-terra`, `claude-sonnet-5`, `gemini-3.1-pro-preview`)
    *   `theme` (light/dark)
    *   `pdfPageLimit` / `pdfSizeLimit`
    *   `docTokenWarn` (estimated token threshold for the chat-files warning toast; default `100000`)
    *   `maxOutputTokens`

**Future tables** (agents, memories, MCP server definitions, skills, to-dos, etc.) must declare an `id` (UUID) primary key and an `updatedAt` field to automatically participate in export/import/sync — see §7.

## 3. Model API Abstraction Layer
A unified class/functions to handle API communication, supporting **streaming** for all providers.
*   **OpenAI Adapter:** Uses `fetch` to OpenAI's chat completions endpoint. Default model: `gpt-5.6-terra`.
*   **Anthropic Adapter:** Uses `fetch` to Anthropic's Messages API. Injects `anthropic-dangerous-direct-browser-access: "true"`. Default model: `claude-sonnet-5`.
*   **Gemini Adapter:** Uses `fetch` to Google's REST endpoints. Default model: `gemini-3.1-pro-preview` (or `models/gemini-3.1-pro-preview`).
*   *Future Proofing:* The architecture should allow querying the API to fetch available models dynamically in the future.

## 4. Multi-modal Processing Pipeline
*   **Images:** Read via `FileReader` to base64. Rendered as thumbnails. *(Known gap: base64 data is not persisted with the message and is not currently sent to model APIs.)*
*   **Documents (TXT/MD/PDF):** Extracted via `FileReader`/`pdf.js`, normalized (CRLF→LF, trailing-space strip, blank-line collapse), hashed (SHA-256) for chat-scoped deduplication, and stored in the `files` table — decoupled from any single message.
*   **Context Assembly:** On each turn, all chat `files` are rendered into a single `<documents><document index="n" name="...">…</document></documents>` block and injected into the system/instruction area (after the system prompt, documents last) rather than inlined into message `content`:
    *   **Anthropic:** `system` is sent as a block array; the documents block carries `cache_control: { type: "ephemeral" }` for prompt caching.
    *   **OpenAI:** Documents are appended to the `system` role message, relying on automatic prefix caching.
    *   **Gemini:** Documents are appended to `systemInstruction`, relying on implicit context caching.
*   Message `content` holds only the user's typed text (or an `Attached: name1, name2` placeholder when sending files with no typed text); the message's `attachments` array keeps lightweight `{ type, name }` chips for display.

## 5. UI / UX Layout
*   **Sidebar (Left):** "New Chat", search, pinned chats, recent chats, "Settings" cog, "Export Data" button.
*   **Chat View (Center):**
    *   **Header:** Editable chat title, pin toggle.
    *   **Message Stream:** Scrollable area. Auto-scrolls during streaming. Markdown rendered, sanitized, code highlighted with "Copy Code" buttons.
    *   **Input Area:** Auto-resizing textarea, paperclip icon for uploads, file preview thumbnails, send button.
*   **Modals:** Settings window (API keys, models, prompts, theme, PDF limits).

## 6. Data Portability
*   **Export:** Streams every Dexie table (`db.tables`, generic — no hardcoded list) into a gzip-compressed NDJSON file (`calyx-backup-<ISO>.ndjson.gz`), excluding secret fields (§7.1). Uses `showSaveFilePicker` where supported (user picks the save location), falling back to a normal `<a download>` otherwise.
*   **Import:** Accepts the current `.ndjson.gz` format or a legacy v2 `.json` export. Two modes:
    *   **Replace** — wipes all local data (secrets excepted) and restores from the file.
    *   **Merge** — adds file records to local data; on an `id` collision, the record with the greater `updatedAt` wins.
*   See §7 for the full format, validation, and secret-handling specification.

## 7. Backup Format, Secret Registry & Import Pipeline
Full specification: [`plans/import-export.md`](import-export.md).

### 7.1 Secret Field Registry
A single pattern governs which fields are treated as secrets everywhere (export stripping, import ignoring, UI masking):

```js
const SECRET_KEY_PATTERN = /(apikey|token|secret|password|authorization)$/i;
```

Any object key (including nested, e.g. a future MCP server's `headers.Authorization`) matching this pattern is:
*   **Stripped** from every export (never written to a backup file).
*   **Ignored** on every import (never overwritten from a file).
*   **Always preserved locally**, even when using Replace mode.

**When adding any new secret-bearing field** (API keys, MCP server auth tokens/headers, future OAuth tokens, etc.), name it so it ends in `apiKey`, `Token`, `Secret`, `Password`, or `Authorization` to be covered automatically. If a field cannot be named this way, it must be added explicitly to the registry in `index.html` and documented here.

### 7.2 Backup File Format
NDJSON, one JSON object per line, gzip-compressed:
```
{"type":"header","format":"calyx-backup","version":3,"dbSchema":1,"exportedAt":"…"}
{"type":"record","table":"chats","data":{…}}
…
{"type":"footer","counts":{"chats":N,"messages":N,…}}
```
The footer's per-table counts are checked against the records actually read during import validation, catching truncated/corrupt files before any write occurs.

### 7.3 Import Pipeline
Two-pass: (1) full-file validation (header format/schema version, record shape, footer count match) with zero writes; (2) batched (500 records) `bulkPut` writes in a Replace or Merge transaction. See `plans/import-export.md` for the full flow diagram.

### 7.4 Legacy Database Migration
On boot, if the old integer-keyed `CalyxDB` exists and the new `Calyx` database is empty, all records are copied once with freshly generated UUIDs (chat references remapped), verified by count, and `CalyxDB` is deleted. Runs entirely in local IndexedDB — no files are written, and no network access occurs.