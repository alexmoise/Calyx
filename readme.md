# Calyx

A zero-backend, state-of-the-art AI workspace packaged entirely within a single, portable `index.html` file. Calyx delivers the visual fidelity, performance, and feature depth of enterprise chat applications while running strictly client-side in modern web browsers across Windows, macOS, and Linux.

---

## 1. What Calyx Is & The Gap It Fills

Current AI chat software forces a compromise between two extremes:

* **Fragile single-file scripts:** Lightweight and portable, but bare-bones. They rely on transient memory or `localStorage`, lack streaming control, break when handling attachments, and offer minimal interface polish.
* **Heavy multi-tier web platforms:** Polished and capable, but laden with npm dependencies, containerized backends, external database setups, telemetry, and hosting overhead. They introduce middleman servers between your machine and provider APIs.

Calyx eliminates this trade-off by occupying the empty space between them: **an uncompromised, zero-build desktop-grade interface delivered as a single self-contained artifact.**

### Key Architectural Pillars

* **Absolute Portability:** A single file. Double-click to open in any browser (`file://`), run from a local path, or host via static storage. No Node runtime, no compilation step, and no web server required.
* **Complete Data Sovereignty:** Direct client-to-API communication using Bring-Your-Own-Key (BYOK). Prompts, attachments, and keys never touch a proxy or third-party server.
* **Scalable Client-Side Persistence:** Replaces fragile storage limits with IndexedDB, enabling large-scale conversation history and relational chat metadata directly in the browser sandbox.
* **Universal Platform Parity:** Works identically across Windows, macOS, and Linux with zero OS-specific dependencies.

---

## 2. Quick Start

1. **Download** — Save [`index.html`](index.html) to your machine (clone the repo or download the raw file).
2. **Open** — Double-click `index.html`, or open it in any modern browser (Chrome, Edge, Firefox, Safari).
3. **Add API key(s)** — Click **Settings**, paste your API key for OpenAI, Anthropic, and/or Gemini.
4. **Set default model** — In the same Settings panel, enter the model name to use by default.
5. **(Optional) Set a system prompt** — Add a global system prompt to steer the assistant's behavior across chats.
6. **Start chatting** — Close Settings, type a message, and press `Enter`.

Your keys, prompts, and chat history stay in the browser's local IndexedDB — nothing is sent anywhere except directly to the provider you configured.

---

## 3. Iterative Roadmap

The roadmap is organized into progressive milestones. Phase 1 is complete and delivers a robust daily driver; subsequent phases introduce power-user UX, deep conversational control, and agentic protocols.

```
Phase 1 (Core MVP — done) ──> Phase 2 (Power UX) ──> Phase 3 (Branching) ──> Phase 4 (Agentic/MCP)

```

**Known Issue:** Attached images are never sent to the model APIs — base64 image data is dropped when a message is persisted, and only text content is included in the API history. Tracked for a fix in Phase 2.

---

### Phase 1: Foundation & The Polished MVP — ✅ Complete

*Goal: A bulletproof, elegant daily driver with native streaming, key management, and robust state storage.*

* **Single-File Architecture Setup:**
  * Declarative reactive UI powered by Vue 3 (global CDN build).
  * Responsive design system built with Tailwind CSS (CDN runtime).
  * Crisp vector iconography via Lucide icons.

* **Local Storage Engine:**
  * IndexedDB persistence powered by Dexie.js for asynchronous, non-blocking storage.
  * Schemas for `chats`, `messages`, and `settings`.

* **Unified API Adapter Layer:**
  * **OpenAI:** Streaming chat completions via native `fetch`.
  * **Anthropic:** Native browser calls utilizing the `anthropic-dangerous-direct-browser-access: "true"` header.
  * **Google Gemini:** Direct streaming endpoints via native `fetch` with token-by-token parsing.
  * Default provider/model selectable in Settings; API key stored per provider.

* **Streaming:**
  * Real-time Server-Sent Events (SSE) stream consumption with live-rendered assistant bubble and animated "thinking" indicator.

* **Core Conversation UX:**
  * Create, select, and delete chat sessions (with confirmation on delete).
  * Inline chat renaming (double-click title or list item) with auto-generated title proposal based on the initial user turn.
  * Chat pinning to keep priority threads fixed at the top of the sidebar.
  * Sidebar search filtering chats by title.
  * Auto-resizing prompt input field (`Enter` to send).

* **File Attachments:**
  * Image ingestion (file picker) with base64 encoding; attachment shown as a removable name chip before send.
  * PDF text extraction via `pdf.js`, bounded by configurable page and file-size limits.
  * Plain-text (`.txt`, `.md`) ingestion via `FileReader`, with whitespace normalization and SHA-256 content deduplication.
  * **Chat-Level Documents:** Text/PDF attachments are pinned to the chat (not the message) and reused across every turn, with a chip bar showing per-file and total estimated token counts (configurable warning threshold).
  * **Prompt Caching:** Chat documents are assembled into a `<documents>` block placed in the system/instruction area — Anthropic requests mark it `cache_control: ephemeral`; OpenAI and Gemini benefit from automatic prefix caching.

* **Render Pipeline & Content Security:**
  * Markdown parsing with `marked.js`.
  * DOM sanitization using `DOMPurify` to eliminate XSS risks from untrusted model outputs.
  * Code block syntax highlighting with `highlight.js`, accompanied by a one-click "Copy" button.

* **Settings & Security Window:**
  * Modal drawer to store API keys directly to IndexedDB (OpenAI, Anthropic, Gemini).
  * Default provider and model (free-text) configuration.
  * Global system prompt configuration.
  * Configurable PDF page-count and file-size limits.
  * Light/dark theme toggle (persisted).

* **Data Portability:**
  * **Full export:** Every stored table — chats, messages, files, settings, and any future table added to the database — is streamed into a single gzip-compressed NDJSON backup file (`calyx-backup-<timestamp>.ndjson.gz`), with no size-driven memory ceiling. API keys and any other credential-shaped field (tokens, secrets, passwords, authorization headers) are always excluded.
  * **Full import:** Restore from a Calyx backup in two modes — **Replace** (wipe local data and restore from the file) or **Merge** (add file records to local data, newest `updatedAt` wins on conflict). API keys are never imported; your locally configured keys are always kept.
  * The backup file is validated end-to-end (format, schema compatibility, record integrity) before any data is written, and the browser's native "Save As" dialog is used where supported so you choose exactly where backups are written.

* **Feedback:**
  * Toast notifications for errors and confirmations.

---

### Phase 2: Power User UX & Document Intelligence

*Goal: Turn Calyx into a high-utility knowledge tool with rich output rendering and richer client-side ingestion.*

* **Chat Organization & Discovery:**
  * **Full-Text Search:** Extend search to match message bodies, not just chat titles.

* **Client-Side Document & Vision Ingestion:**
  * **Drag-and-Drop & Clipboard Paste:** Ingest images by dragging onto the window or pasting from clipboard, in addition to the file picker.
  * **Visual Thumbnails:** Render actual image previews in the attachment tray instead of icon-only chips.

* **Render Pipeline Enhancements:**
  * LaTeX mathematical formula rendering via KaTeX.

* **Formatted Chat Export & Archival:**
  * **Markdown Export:** Clean, human-readable `.md` output containing timestamps, system prompts, and message bodies.
  * **Self-Contained HTML Export:** Styled, standalone read-only transcripts ready for sharing.

* **Generation Control:**
  * **Stop Generation:** Instant cancellation of an in-flight response using `AbortController`.

* **Model Selection:**
  * Curated per-provider model dropdown (replacing the current free-text model field).

---

### Phase 3: Advanced Conversation Control & Specialized Agents

*Goal: Provide nonlinear thinking tools, precise cost tracking, and persona-driven agent workflows.*

* **Nonlinear Conversation Control:**
  * **Chat Forking:** Branch a new conversation thread from any historical message turn, preserving context up to that point.
  * **Message Editing & Regeneration:** Edit any previous user prompt and re-run inference from that juncture without deleting previous attempts.
  * **Alternative Message Swapping:** Carousel through regenerated assistant responses (Turn 1 of N).

* **Granular Context Controls:**
  * Inference parameter controls (Temperature, Top-P, Max Output Tokens).
  * **Context Cache:** Leverage provider-side prompt/context caching (e.g., Anthropic, OpenAI) to reduce redundant token cost on repeated or growing context.

* **Profiles & Persistent Memory — ✅ Implemented:**
  * **Profiles:** Optional named containers (e.g., "Jane D. — Therapy", "Acme Corp — Advisory") that group related chats ("sessions") and carry a persona system prompt, layered after the global prompt.
  * **Inline Memory Capture:** The model continuously assesses the conversation and appends `<memory category="..." when="...">` tags to its replies; Calyx parses, saves, and hides them, surfacing each as a dismissible chip under the message.
  * **Membership-Derived Profile Memory:** Memories are owned by the chat they originated in; a profile's context is the live union of memories from every chat currently assigned to it. Assigning/un-assigning a chat changes context immediately — no copying, no cross-chat edits.
  * **Chronological Grounding:** Memories are never edited or superseded by the app — differences over time are left for the model to interpret as evolution, using per-memory `when`/`category`/timestamp metadata and session labels.
  * **Deterministic Memory Selection:** Pinned memories and configurable "cornerstone" categories (trait, value, goal, agreement by default) are always included; remaining memories fill by recency up to a configurable budget; memories that don't fit are still recoverable via a zero-dependency BM25 keyword-recall pass matched against the live conversation.
  * **Session Digests:** Each chat maintains a short, incrementally-updated digest injected (instead of full memories) when referencing other sessions in a profile, refreshed on chat switch, lazily before sending, or via a per-profile "Refresh digests" action.
  * **Segmented Rolling Compaction:** Older turns beyond a configurable raw-turn window are summarized into frozen "segments" (never re-summarized individually); only the oldest segments are merged further when the segment budget is exceeded. Original messages are never deleted. Auto-triggers at a configurable % of the context budget, or on demand via "Compact now".
  * **Memory Review Panel:** Per-chat panel to view, pin, edit, delete, and manually add memories, plus a read-only view of the live profile memory pool with out-of-context items greyed out.

* **Browser-Native Agentic Tools:**
  * **Specialized Agents System:** Create custom "Agents" (e.g., Law Advisor, Medical Advisor, Finance Specialist) by configuring a title, provider, model, specific inference parameters (temperature, effort, etc.), and an agent-specific system prompt.
  * **Skill Files:** Enhance agents by uploading local files or linking remote URLs containing specialized instructions or structured tools. These skills are "advertised" to the agent, allowing it to read them dynamically when necessary.
  * **Agent-Driven Conversations:** Start new chats by selecting a specific agent.
  * **Mid-Conversation Agent Switching:** Change the active agent at any point during a chat session. The new agent inherits the entire conversation history and context seamlessly, replacing the simpler "Mid-Conversation Provider Switching" concept.
  * **Native Task Management (ToDo):** Built-in tool calling allowing the LLM to read, write, and manage a persistent ToDo list stored in IndexedDB, rendered natively in the UI.

* **Dual-Model Compare Mode:**
  * Side-by-side prompt dispatch: send the identical prompt simultaneously to two providers to evaluate reasoning, latency, and tone side by side.

* **Token & Cost Telemetry:**
  * **Live Conversation Cost:** Calculate and display live cost per chat, updated at the end of each message stream based on exact API usage statistics.
  * **Dynamic Pricing Engine:** Read real prices directly from the API response if available. If not, fallback to manual price inputs (global per model) configured in Settings.
  * **Adaptive Display:** If prices are available, display the cost in dollars ($). If prices are missing, display raw token counts. If manual prices are added later, historical token counts automatically convert to dollar displays.

---

### Phase 4: Frontier Integrations & Agentic Protocols

*Goal: Elevate Calyx into an extensible local agentic workstation.*

* **Model Context Protocol (MCP) Integration:**
  * Native client support for Anthropic's Model Context Protocol.
  * Connect the browser UI to local or remote MCP servers via WebSocket bridges (`ws://localhost:...`) or SSE endpoints.
  * Automatic tool discovery, schema parsing, and model-driven tool execution loops directly inside the chat interface.

* **Client-Side Media Compression:**
  * Browser-side Canvas/Wasm image optimization to scale and compress oversized images before API dispatch, cutting token costs and latency.

* **Offline/Hybrid Local LLM Support:**
  * WebGPU-accelerated local inference fallback (via WebLLM/Wasm) for zero-connectivity prompt handling.
  * Seamless switching between local on-device models and frontier cloud APIs.

* **Local Server Endpoints (LM Studio / Ollama):**
  * New provider adapter targeting OpenAI-compatible chat completions APIs exposed by local inference servers (LM Studio, Ollama, and similar).
  * Settings: configurable base URL (default `http://localhost:1234/v1`), optional API key, and free-text model name.
  * Supports `localhost`, LAN IPs/hostnames, and publicly resolvable hostnames — enabling on-machine, LAN-hosted, or remotely reachable local-model servers.

* **Hardened Data Encryption:**
  * Optional master-password database encryption using Web Crypto API (AES-GCM) to encrypt stored API keys and chat archives at rest.

* **Remote Storage Sync (Cloud/Self-Hosted):**
  * Multi-device synchronization using Origin Private File System (OPFS) combined with WebDAV or S3-compatible APIs (Google Drive, OneDrive, AWS, self-hosted).
  * Syncs individual chat files rather than a monolithic database to gracefully handle offline usage and merge conflicts.
  * Zero-trust architecture: all synced data is encrypted client-side (AES-GCM) before leaving the browser.

---

## 4. Architecture Constraints

These rules are non-negotiable for all future development iterations and must be enforced at every phase.

* **Export/Import Parity:** The backup pipeline iterates all Dexie tables generically — any new table or field is included automatically with no code change required. The only permitted exclusion is secret-shaped fields (API keys, tokens, passwords, authorization headers), which are stripped on export and ignored on import by a central pattern-based registry (see `plans/architecture.md` §7.1). No non-secret setting may exist that cannot round-trip through a backup file.
* **Remote Sync Coverage:** Every setting and piece of user content must be included in the remote sync payload when a remote storage target is configured. Partial sync is not permitted.
* **Minimal HTML Footprint:** `index.html` must remain as small as possible. Heavy libraries, renderers, and optional features must be loaded dynamically from CDNs at runtime — never bundled inline.
* **Pinned CDN Versions:** All CDN dependencies must reference a specific, immutable version (e.g., `@1.2.3`). Unpinned or `@latest` references are prohibited to prevent upstream breaking changes.
* **Offline Resilience:** Core functionality must remain operational without network access. CDN-loaded assets must degrade gracefully or be cached; the user must never be left with a broken interface due to a failed CDN fetch.
* **Local Storage as Source of Truth:** IndexedDB is the canonical data store. Remote storage is a synchronization target only — it never overrides local state without explicit user action.
* **Mandatory Client-Side Encryption:** Everything synced to remote storage must be encrypted client-side (AES-GCM via Web Crypto API) before transmission. No plaintext user data may leave the browser.
* **Native-First Dependencies:** Prefer native browser APIs and ES module patterns over adding new external libraries. Every new dependency must be justified against an equivalent native implementation.
