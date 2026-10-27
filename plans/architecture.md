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
Database name: **`Calyx`** (schema version 3). The legacy `CalyxDB` (integer `++id` keys) is migrated once on boot — see §9.4 — and deleted after a verified copy.

**Encryption at rest (schema v3+):** Every table except `vault` stores records as an envelope — a small set of plaintext index fields (needed for Dexie `where()` queries) plus one encrypted blob (`enc: { v, kid, iv, ct }`, AES-GCM-256) holding everything else. See `plans/encryption.md` for the full key hierarchy, migration, and backup format. The **`vault`** table (`key` primary key, single row `'keyring'`) holds the keyring — wrapped master-secret slots (password/recovery/device) — and is never included in export/import or future remote sync.

*   **`id`** fields across all tables are **string UUIDs** (`crypto.randomUUID()`), not auto-increment integers. This allows records from independent databases (imports, future sync) to merge without ID collisions.
*   Every mutable record carries an **`updatedAt`** timestamp (ms epoch), used for last-write-wins conflict resolution on import merge and future remote sync.
*   **`profiles`**: `id`, `name`, `systemPrompt` (persona prompt, layered after the global prompt), `createdAt`, `updatedAt`. Optional grouping container for related chats ("sessions") — see §8.
*   **`chats`**: `id`, `title`, `createdAt`, `updatedAt`, `isPinned`, `profileId` (indexed, nullable — `null` = unassigned/standalone chat). The per-chat `systemPrompt` override field was retired in schema v2 in favor of profile persona prompts; stripped from existing records on upgrade and on import of older backups.
*   **`messages`**: `id`, `chatId` (indexed), `role` (user/assistant), `content`, `createdAt`, `updatedAt`, `attachments` (array of objects: `{ type, name }` — image attachments additionally carry `data`/`mimeType` in-memory only, never persisted with the message), `memoryIds` (array, assistant messages only — memories extracted from that specific reply), `sessionProposal` (object, assistant messages only, optional — `{ title, purpose, status: 'pending'|'created'|'dismissed', chatId }`, see §7.8).
*   **`files`**: `id`, `chatId` (indexed), `name`, `type` (`text`/`pdf`), `hash` (indexed, SHA-256 of normalized text, used for dedup), `text`, `tokens` (estimated), `createdAt`, `updatedAt`. Chat-scoped document attachments, decoupled from individual messages so they persist across every turn.
*   **`memories`**: `id`, `chatId` (indexed — ownership, not a FK to the active chat context), `category` (`trait`/`value`/`goal`/`event`/`person`/`decision`/`agreement`/`preference`/`pattern`/`state`/`other`), `content`, `when` (nullable — the period the fact *refers to*, e.g. "childhood", distinct from `createdAt`), `source` (`model`/`user`/`manual`), `pinned` (bool, indexed), `sourceMessageId`, `createdAt`, `updatedAt`. See §8 for how these compose into profile-level context.
*   **`segments`**: `id`, `chatId` (indexed), `content` (dense structured summary), `fromMessageId`, `toMessageId`, `fromAt`, `toAt` (indexed), `tokens` (estimated), `createdAt`, `updatedAt`. Frozen summaries of compacted raw turns — see §8.
*   **`digests`**: `id` (= `chatId`), `chatId`, `content` (150–300 token incremental summary of the whole session), `coversUntilAt` (last message timestamp folded in), `createdAt`, `updatedAt`.
*   **`settings`**: Key-value pairs (`{ key, value, updatedAt }`).
    *   `systemPrompt` (Global — layered after the fixed built-in protocol block, before any profile persona prompt; see §8)
    *   `openaiApiKey`, `anthropicApiKey`, `geminiApiKey` — **secret**, see §9.1
    *   `selectedModel` (Default: `gpt-5.6-terra`, `claude-sonnet-5`, `gemini-3.1-pro-preview`)
    *   `theme` (light/dark)
    *   `pdfPageLimit` / `pdfSizeLimit`
    *   `docTokenWarn` (estimated token threshold for the chat-files warning toast; default `100000`)
    *   `maxOutputTokens`
    *   `utilityProvider` / `utilityModel` — provider+model used for segment summaries and session digests (default: same as chat)
    *   `contextBudget` (tokens, default `200000`), `compactTriggerPct` (default `70`), `minRawTurns` (default `20`) — drive segmented compaction, see §8
    *   `memoryBudgetPct` (default `30`, % of `contextBudget`) — caps how many profile memories are injected per turn
    *   `cornerstoneCategories` (default `['trait','value','goal','agreement']`) — memory categories always included regardless of budget
    *   `branchSessions` (default `true`) — whether the model may propose a new session for a drifting topic, see §7.8
    *   `autoLockMinutes` (default `15`, `0` = off) — idle auto-lock timeout, password mode only; see `plans/encryption.md` §6

**Future tables** (agents, MCP server definitions, skills, to-dos, etc.) must declare an `id` (UUID) primary key and an `updatedAt` field to automatically participate in export/import/sync — see §9. New fields on existing tables are encrypted by default (see §2); add to `PLAIN_FIELDS` only if genuinely needed as a Dexie index.

**Known risk — shared `file://` storage origin:** When opened via `file://` (double-click, local path), all local HTML files share a single browser-wide storage origin (literal origin `"file://"`, with no per-folder/per-filename isolation in Chromium). Consequently: (1) moving, renaming, or copying `index.html` anywhere on the same machine, in the same browser profile, continues to see the same `Calyx` database — this is expected and relied upon (e.g. portable-browser installs on a USB stick carry their profile, and thus the DB, with them across machines); (2) any other local HTML file opened in that same browser profile that also creates an IndexedDB named `Calyx` would read/write the *same physical database*, with no isolation — a theoretical collision/corruption risk. Mitigations (unique DB name, boot-time fingerprint check) were considered and deliberately deferred; this is accepted as a documented risk rather than an active defense.

## 3. Model API Abstraction Layer
A unified class/functions to handle API communication, supporting **streaming** for all providers.
*   **OpenAI Adapter:** Uses `fetch` to OpenAI's chat completions endpoint. Default model: `gpt-5.6-terra`.
*   **Anthropic Adapter:** Uses `fetch` to Anthropic's Messages API. Injects `anthropic-dangerous-direct-browser-access: "true"`. Default model: `claude-sonnet-5`.
*   **Gemini Adapter:** Uses `fetch` to Google's REST endpoints. Default model: `gemini-3.1-pro-preview` (or `models/gemini-3.1-pro-preview`).
*   *Future Proofing:* The architecture should allow querying the API to fetch available models dynamically in the future.

## 4. Multi-modal Processing Pipeline
*   **Images:** Read via `FileReader` to base64. Rendered as a name chip in the attachment tray (icon + filename only — visual thumbnails are a planned Phase 2 item, see `readme.md` §3). *(Known gap: base64 data is not persisted with the message and is not currently sent to model APIs.)*
*   **Documents (TXT/MD/PDF):** Extracted via `FileReader`/`pdf.js`, normalized (CRLF→LF, trailing-space strip, blank-line collapse), hashed (SHA-256) for chat-scoped deduplication, and stored in the `files` table — decoupled from any single message.
*   **Context Assembly:** On each turn, all chat `files` are rendered into a single `<documents><document index="n" name="...">…</document></documents>` block and injected into the system/instruction area (after the system prompt, documents last) rather than inlined into message `content`:
    *   **Anthropic:** `system` is sent as a block array; the documents block carries `cache_control: { type: "ephemeral" }` for prompt caching.
    *   **OpenAI:** Documents are appended to the `system` role message, relying on automatic prefix caching.
    *   **Gemini:** Documents are appended to `systemInstruction`, relying on implicit context caching.
*   Message `content` holds only the user's typed text (or an `Attached: name1, name2` placeholder when sending files with no typed text); the message's `attachments` array keeps lightweight `{ type, name }` chips for display.

## 5. UI / UX Layout
*   **Sidebar (Left):** "New Chat", search, chats grouped by profile section (+ "Unassigned"), "Settings" cog, "Export"/"Import Data" buttons.
*   **Chat View (Center):**
    *   **Header:** Editable chat title, session-number badge (within its profile), profile assign/un-assign dropdown, "Compact now", "Memories" panel toggle, pin toggle.
    *   **Message Stream:** Scrollable area. Auto-scrolls during streaming. Markdown rendered, sanitized, code highlighted with "Copy Code" buttons. A divider marks turns folded into a compaction segment (expandable to view the summary). Assistant messages carry dismissible chips for any memory saved from that reply.
    *   **Input Area:** Auto-resizing textarea, paperclip icon for uploads, file preview thumbnails, send button.
*   **Memory Panel (Right, toggleable):** Tabs for "This session" (editable/pinnable/deletable, manual-add form) and "Profile" (read-only live pool, out-of-context items greyed).
*   **Modals:** Settings window (API keys, models, prompts, theme, PDF limits, memory & compaction tuning), Profile editor (name, persona prompt, delete with optional cascade).

## 6. Data Portability
*   **Export:** Streams every Dexie table (`db.tables`, generic — no hardcoded list) into a gzip-compressed NDJSON file (`calyx-backup-<ISO>.ndjson.gz`), excluding secret fields (§9.1). Uses `showSaveFilePicker` where supported (user picks the save location), falling back to a normal `<a download>` otherwise.
*   **Import:** Accepts the current `.ndjson.gz` format or a legacy v2 `.json` export. Two modes:
    *   **Replace** — wipes all local data (secrets excepted) and restores from the file.
    *   **Merge** — adds file records to local data; on an `id` collision, the record with the greater `updatedAt` wins.
*   See §9 for the full format, validation, and secret-handling specification.

## 7. Profiles & Persistent Memory
Full specification and design rationale: [`plans/long-context-memory.md`](long-context-memory.md).

### 7.1 Profiles & Sessions
A **profile** (`profiles` table) is an optional, named container grouping chats that belong to the same ongoing relationship (e.g. one patient, one client company). Each chat assigned to a profile acts as a **session**; session order/number is derived from `chat.createdAt`, not stored. Chats can be assigned, re-assigned, or un-assigned at any time with no data migration — membership is just `chat.profileId`.

### 7.2 Membership-Derived Memory
Memories are **owned by the chat they originated in** (`memory.chatId`). A profile's available memory is computed live as the union of memories from every chat *currently* assigned to that profile — nothing is copied or duplicated. Assigning a chat brings its memories into scope immediately (for new and already-open sessions); un-assigning removes them just as immediately. Conflicting facts across sessions are **never reconciled automatically** (no LLM-driven supersede/merge) — both remain visible, ordered chronologically with session labels, and the chat model is instructed to interpret differences as evolution rather than error (see the protocol block, §7.5).

### 7.3 Memory Capture
The model appends inline `<memory category="..." when="..." source="...">fact</memory>` tags to its own replies (`extractMemoryTags`/`stripMemoryTagsForDisplay` in `index.html`). Calyx strips these tags from the displayed/stored message, hides partial tags while streaming, validates category/length, and persists one `memories` row per tag with a back-reference (`message.memoryIds`) for UI attribution (chips, undo). No tool-calling loop is required — this works identically across all three providers.

### 7.4 Context Assembly & Memory Selection
Per turn, context is assembled stable-to-volatile for maximum prompt-cache reuse (see `buildContextForChat` in `index.html`):

1. Built-in protocol block (fixed, §7.5)
2. Global system prompt (`settings.systemPrompt`)
3. Profile persona prompt (`profile.systemPrompt`, if assigned)
4. `<documents>` block (chat-scoped files, §4)
5. Other sessions in the profile: `<session_digests>` + `<profile_memories>`, chronological, grouped by session
6. Current session: `<conversation_summary>` (compaction segments, §7.6)
7. Current session: `<session_memories>` + `<recalled_memories>` + `<calyx_context>` (date/time, profile, session number)
8. Recent raw turns, verbatim, user messages timestamped (`[YYYY-MM-DD HH:MM]`)
9. New user message

Anthropic requests mark blocks 4–6 with `cache_control: { type: "ephemeral" }`; OpenAI/Gemini rely on automatic/implicit prefix caching with the same ordering.

Memory selection (`selectMemoriesForContext`) is **fully deterministic — no LLM judgment** — applied when profile memories exceed `memoryBudgetPct` of `contextBudget`:
*   Pinned memories: always included.
*   `cornerstoneCategories` (default trait/value/goal/agreement): always included, every session.
*   All session digests: always included (low-res fallback for sessions whose memories don't fit).
*   Remaining budget: filled with the newest remaining memories (recency).
*   Anything still excluded is recoverable per-turn via a zero-dependency BM25 keyword match (`bm25Rank`) against the new message + recent turns, injected as `<recalled_memories>` (top 20 matches).
*   If pinned + cornerstone + digests alone exceed the budget, everything is still sent and a warning toast is shown.

### 7.5 Built-in Protocol Block
A fixed, non-editable instruction block (`PROTOCOL_BLOCK` constant in `index.html`) is always sent first, ahead of the user-editable global system prompt. It explains the context blocks the model receives, instructs it to treat chronological differences between memories as evolution rather than contradiction, and defines the `<memory>` tag capture syntax and categories. Kept non-editable so memory capture keeps working even if the user clears or rewrites the global prompt.

### 7.6 Segmented Rolling Compaction
Raw turns beyond a configurable window (`minRawTurns`) are folded into frozen summary **segments** (`segments` table) rather than being deleted or truncated. Each segment is generated once by a configurable **utility model** (`utilityProvider`/`utilityModel`, default: same as chat) and is never re-summarized individually; only when total segment tokens exceed a budget are the **oldest two segments merged**, keeping repeated re-summarization (the main source of drift) to a minimum. Compaction triggers automatically once the assembled context exceeds `compactTriggerPct` of `contextBudget`, or on demand via "Compact now". Original messages are never deleted — the UI shows a "turns summarized" divider with the raw transcript still viewable.

### 7.7 Session Digests
Each chat maintains one incrementally-updated **digest** (`digests` table, ~150–300 tokens, generated by the utility model) summarizing the whole session, used in place of full memories when referencing *other* sessions in a profile (§7.4, block 5). A digest is stale once its chat has messages newer than `coversUntilAt`. Refresh triggers: on leaving a chat (background), lazily before assembling context for a profile chat (blocking, ensures correctness), or via a per-profile "Refresh digests" action that catches every stale chat in that profile. A chat's own digest is never injected into itself.

### 7.8 Model-Proposed Session Branching
Full specification: [`plans/branch-session.md`](branch-session.md).

When a profile chat's topic drifts, the model may propose parking the new topic in a separate session of the same profile by appending `<session_proposal title="...">purpose</session_proposal>` to its reply. The tag is parsed and stripped like `<memory>` tags; Calyx shows a confirmation card (editable title, Create/Dismiss) under the message — no chat is created without an explicit click. The proposal and its status (`pending`/`created`/`dismissed`) persist on the originating assistant message (`messages.sessionProposal`), surviving reloads.

On Create: a new chat is added to the same `profileId` (current chat stays active/selected), a `decision` memory is saved in the **origin** chat recording the agreement, and a `goal` memory is saved in the **new** chat pointing back to the originating session — the link between sessions is carried entirely through memories, no `parentChatId` field. Gated by `settings.branchSessions` (default on) **and** `chat.profileId` being set; standalone chats never see the tag or the instructions. The protocol wording is domain-neutral with a moderate bar, explicitly deferring to the global/profile system prompt for tuning.

## 8. Agentic Roadmap Hooks
Future tables (agents, MCP server definitions, skills, to-dos, etc.) must declare an `id` (UUID) primary key and an `updatedAt` field to automatically participate in export/import/sync — see §9.

## 9. Backup Format, Secret Registry & Import Pipeline
Full specification: [`plans/import-export.md`](import-export.md).

### 9.1 Secret Field Registry
A single pattern governs which fields are treated as secrets everywhere (export stripping, import ignoring, UI masking):

```js
const SECRET_KEY_PATTERN = /(apikey|token|secret|password|authorization)$/i;
```

Any object key (including nested, e.g. a future MCP server's `headers.Authorization`) matching this pattern is:
*   **Stripped** from every export (never written to a backup file).
*   **Ignored** on every import (never overwritten from a file).
*   **Always preserved locally**, even when using Replace mode.

**When adding any new secret-bearing field** (API keys, MCP server auth tokens/headers, future OAuth tokens, etc.), name it so it ends in `apiKey`, `Token`, `Secret`, `Password`, or `Authorization` to be covered automatically. If a field cannot be named this way, it must be added explicitly to the registry in `index.html` and documented here.

### 9.2 Backup File Format
Current format is an **encrypted `.calyx` container** (magic bytes, plaintext JSON header, chunked AES-GCM ciphertext). The plaintext payload inside is the same gzip NDJSON stream as before:
```
{"type":"header","format":"calyx-backup","version":5,"dbSchema":3,"exportedAt":"…"}
{"type":"record","table":"chats","data":{…}}
…
{"type":"footer","counts":{"chats":N,"messages":N,…}}
```
The footer's per-table counts are checked against the records actually read during import validation, catching truncated/corrupt files before any write occurs. Full encryption/key-wrapping spec: `plans/encryption.md` §8.

### 9.3 Import Pipeline
Two-pass: (1) full-file validation (header format/schema version, record shape, footer count match) with zero writes; (2) batched (500 records) `bulkPut` writes (sealed with the local vault's keys) in a Replace or Merge transaction. See `plans/import-export.md` for the full flow diagram and `plans/encryption.md` §9 for the encrypted-format unwrap flow. Records are passed through `sanitizeRecord` (secret stripping + removal of the retired `chats.systemPrompt` field) so older backups import cleanly against the current schema. Two formats are accepted: the current encrypted `.calyx` container, and the legacy plaintext gzip NDJSON (`calyx-backup` v4) format for read-only migration off older Calyx versions. The one-time legacy `CalyxDB` migration and legacy plain-JSON (v2) import path were removed after confirming no deployment still depended on them.

### 9.4 Encryption Exclusions
The `vault` table (encryption keyring) is excluded from the generic export/import table loop — it never leaves the device and has no portable meaning across instances with different keys. See `plans/encryption.md` for the full key hierarchy, record envelope, migration, and backup encryption design.