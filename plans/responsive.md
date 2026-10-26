# Responsive Layout Plan — APPROVED (all recommended options D1–D7)

## Decisions (confirmed: Recommended column)

| # | Decision | Recommended | Alternative |
|---|---|---|---|
| D1 | Breakpoints | Sidebar drawer `< lg` (1024); memory overlay `< xl` (1280) | Both `< md` (768) — tablet portrait then shows sidebar + chat at ~480px |
| D2 | Drawer animation | 200ms `translate-x` slide, `motion-reduce:transition-none` | No animation (instant show/hide) |
| D3 | Hamburger placement | Header far-left, before title | Floating button bottom-left |
| D4 | Header crowding `< sm` (375px phone can't fit title + 6 tap-size controls) | Below `sm`: hide Theme toggle (exists in Settings) + Pin (exists in sidebar row); profile `<select>` `max-w-[96px]` | (b) Move profile select + session badge to a second header row below `sm`; (c) "⋯" overflow menu — needs extra state, not recommended |
| D5 | Settings structure | Sectioned nav (left rail `≥ lg`, horizontal tabs `< lg`), 4 sections now, extensible | Single scroll page with grouped section headings (no new state) |
| D6 | Touch target detection | `pointer: coarse` media (input-type based) | Width-based `max-lg:` (enlarges on narrow desktop windows too) |
| D7 | Desktop settings card size | `max-w-4xl`, fixed `h-[min(85vh,760px)]` | `max-w-3xl`, auto height up to `85vh` |

Scope: [`index.html`](../index.html) only. Tailwind v4 browser CDN + Vue 3. No new deps. Desktop (≥ `xl`, 1280px) output identical to today.

## 1. Breakpoint strategy

| Viewport | Sidebar | Memory panel | Settings |
|---|---|---|---|
| `< lg` (<1024: phones, tablet portrait) | Off-canvas drawer | Overlay | Full-screen |
| `lg` – `xl` (1024–1279: tablet landscape, small laptops) | Inline (as today) | Overlay | Large centered card |
| `≥ xl` (≥1280: desktop) | Inline (as today) | Inline `w-80` (as today) | Large centered card |

Reasoning: sidebar `w-72` + memory `w-80` = 608px; below 1280 two inline panels squeeze chat column to under ~450px.

Touch sizing uses Tailwind v4 `pointer-coarse:` variant (input-type based, not width) → desktop mouse sizing never changes, even on narrow windows. Touch laptops with mouse primary stay unchanged.

## 2. State changes

- Root: `sidebarOpen = ref(false)`. Set `false` inside [`selectChat()`](../index.html:1703) (covers `createChat` + `open-chat`).
- `SettingsModal`: local `section = ref('general')` (only if D5 = tabs).
- No other state.

## 3. Component changes

### 3.1 Global
- Viewport meta: add `viewport-fit=cover` (required for `env(safe-area-inset-*)`).
- `#app`: `h-screen` → `h-dvh` (fixes iOS URL-bar 100vh overflow; identical on desktop).
- One custom utility in `<style type="text/tailwindcss">`: `@utility tap` → under `@media (pointer: coarse)` sets `min-width/min-height: 2.75rem` (44px) + flex centering. Added to every icon-only button. DRY vs repeating 4 variant classes per button.
- iOS zoom (D6): single CSS rule `@media (pointer: coarse) { input, select, textarea { font-size: max(16px, 1em) } }` — covers chat textarea, search, memory inputs, settings, profile editor. No `maximum-scale=1` (breaks pinch-zoom, WCAG 1.4.4).

### 3.2 Sidebar
- `<aside>` gets: `max-lg:fixed max-lg:inset-y-0 max-lg:left-0 max-lg:z-40 transition-[transform,visibility]` + bound class `sidebarOpen ? '' : 'max-lg:-translate-x-full max-lg:invisible'`. `invisible` removes off-screen drawer from tab order / screen readers.
- Backdrop: `<div v-if="sidebarOpen" class="lg:hidden fixed inset-0 z-30 bg-black/50" @click="sidebarOpen = false">`.
- Safe-area: drawer top/bottom padding `max-lg:pt-[env(safe-area-inset-top)] max-lg:pb-[env(safe-area-inset-bottom)]`.
- Hover-only actions (chat-list-item pin/rename/delete, profile refresh/edit) use `group-hover:visible` — **unreachable on touch** (v4 `hover:` only fires on hover-capable devices). Add `pointer-coarse:visible`.

### 3.3 Header
- Hamburger button (`Menu` icon, `lg:hidden tap`) as first element, left of title, `aria-label`, `aria-expanded`.
- Safe-area top: `h-14` → `min-h-14 pt-[env(safe-area-inset-top)] box-content`-equivalent (height grows only on notched devices).
- Narrow-width crowding (D4): 375px phone fits ~5 tap-size buttons + title. Need to drop items below `sm`.
- Icon buttons get `tap`.

### 3.4 Memory panel
- `<aside>` adds: `max-xl:fixed max-xl:inset-y-0 max-xl:right-0 max-xl:z-40 max-xl:w-full max-xl:max-w-sm max-xl:shadow-xl` + safe-area paddings.
- Backdrop: `v-if="showMemoryPanel && currentChat"`, `xl:hidden`, click → `showMemoryPanel = false`.
- Small action buttons (pin/edit/delete/close/add) get `tap`.

### 3.5 Input bar
- Container `p-4` → `p-4 pb-[max(1rem,env(safe-area-inset-bottom))]`.
- Attach/Send `w-8 h-8` get `tap`.
- Textarea font covered by 3.1 rule.

### 3.6 Message bubbles
- User, assistant, streaming bubbles: `max-w-[80%]` → `max-w-[90%] sm:max-w-[80%]`.
- Code block copy button (JS className, `opacity-0 group-hover:opacity-100`): add `pointer-coarse:opacity-100`.

### 3.7 Toasts
- `bottom-4` → `bottom-[max(1rem,env(safe-area-inset-bottom))]`; mobile: `max-lg:left-4` so long messages wrap instead of overflowing.

### 3.8 Settings modal (D5)
Container:
- `< lg`: `fixed inset-0`, `h-dvh w-full`, no rounding, no outer padding, safe-area padding on header/footer.
- `≥ lg`: `max-w-4xl h-[min(85vh,760px)] rounded-2xl` — fixed height so content doesn't jump.
- Layout: sticky header (title + close) / scrollable body / sticky footer (Done).

Recommended structure — sectioned with nav (scales to roadmap):

| Section | Contents now | Future (roadmap) |
|---|---|---|
| General | Theme, Global System Prompt | Encryption / master password |
| Providers & Models | Default provider, model, API keys | Model dropdown, LM Studio/Ollama endpoints |
| Memory & Context | Utility provider/model, budgets, cornerstone, branching | Context cache |
| Documents & Output | PDF limits, doc token warn, max output tokens | Image compression |
| *(future)* Agents, Pricing, MCP, Sync | — | Own sections |

- `≥ lg`: left vertical nav (`w-52`) + content pane.
- `< lg`: horizontal scrollable tab strip under header.
- Within sections: `grid sm:grid-cols-2/3` for numeric fields (single column on phones).

## 4. Execution order
1. Global (meta, `h-dvh`, `tap` utility, iOS font rule)
2. Sidebar drawer + backdrop + `sidebarOpen` + hover-action fix
3. Header (hamburger, safe-area, crowding, tap)
4. Memory panel overlay + backdrop
5. Input bar, bubbles, copy button, toasts
6. Settings modal restructure
7. Manual verification via Chrome DevTools emulation: 375×812 touch, 820×1180 touch, 1024×768, 1440×900 (desktop diff = none)
