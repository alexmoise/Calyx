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
*   **`chats`**: `id`, `title`, `updatedAt`, `createdAt`, `isPinned`, `systemPrompt` (override).
*   **`messages`**: `id`, `chatId` (indexed), `role` (user/assistant), `content`, `createdAt`, `attachments` (array of objects: `{ type, name }` — image attachments additionally carry `data`/`mimeType` in-memory only, never persisted with the message).
*   **`files`** (v2): `id`, `chatId` (indexed), `name`, `type` (`text`/`pdf`), `hash` (indexed, SHA-256 of normalized text, used for dedup), `text`, `tokens` (estimated), `createdAt`. Chat-scoped document attachments, decoupled from individual messages so they persist across every turn.
*   **`settings`**: Key-value pairs.
    *   `systemPrompt` (Global)
    *   `openaiApiKey`, `anthropicApiKey`, `geminiApiKey`
    *   `selectedModel` (Default: `gpt-5.6-terra`, `claude-sonnet-5`, `gemini-3.1-pro-preview`)
    *   `theme` (light/dark)
    *   `pdfPageLimit` / `pdfSizeLimit`
    *   `docTokenWarn` (estimated token threshold for the chat-files warning toast; default `100000`)

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
*   **Export:** Ability to export `chats`, `messages`, `files`, and `settings` (excluding API keys) as a JSON file for backup and portability across devices.