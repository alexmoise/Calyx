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
*   **`messages`**: `id`, `chatId` (indexed), `role` (user/assistant), `content`, `createdAt`, `attachments` (array of objects: `{ type, data, name }`).
*   **`settings`**: Key-value pairs.
    *   `systemPrompt` (Global)
    *   `openaiApiKey`, `anthropicApiKey`, `geminiApiKey`
    *   `selectedModel` (Default: `gpt-5.6-terra`, `claude-sonnet-5`, `gemini-3.1-pro-preview`)
    *   `theme` (light/dark)
    *   `pdfPageLimit` / `pdfSizeLimit`

## 3. Model API Abstraction Layer
A unified class/functions to handle API communication, supporting **streaming** for all providers.
*   **OpenAI Adapter:** Uses `fetch` to OpenAI's chat completions endpoint. Default model: `gpt-5.6-terra`.
*   **Anthropic Adapter:** Uses `fetch` to Anthropic's Messages API. Injects `anthropic-dangerous-direct-browser-access: "true"`. Default model: `claude-sonnet-5`.
*   **Gemini Adapter:** Uses `fetch` to Google's REST endpoints. Default model: `gemini-3.1-pro-preview` (or `models/gemini-3.1-pro-preview`).
*   *Future Proofing:* The architecture should allow querying the API to fetch available models dynamically in the future.

## 4. Multi-modal Processing Pipeline
*   **Images:** Read via `FileReader` to base64. Rendered as thumbnails. Passed to vision models.
*   **Documents (PDF):** Extracted using `pdf.js`. Configurable limits in settings to prevent token overflow. Text is prepended to the user's message or injected into the system prompt.

## 5. UI / UX Layout
*   **Sidebar (Left):** "New Chat", search, pinned chats, recent chats, "Settings" cog, "Export Data" button.
*   **Chat View (Center):**
    *   **Header:** Editable chat title, pin toggle.
    *   **Message Stream:** Scrollable area. Auto-scrolls during streaming. Markdown rendered, sanitized, code highlighted with "Copy Code" buttons.
    *   **Input Area:** Auto-resizing textarea, paperclip icon for uploads, file preview thumbnails, send button.
*   **Modals:** Settings window (API keys, models, prompts, theme, PDF limits).

## 6. Data Portability
*   **Export:** Ability to export `chats`, `messages`, and `settings` (excluding API keys) as a JSON file for backup and portability across devices.