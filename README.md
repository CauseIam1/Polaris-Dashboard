# Polaris Dashboard — Looking Glass Data Interface
## Version 4.6.1 — Holodeck Tactical Operations Interface (October 2026)

---

## Overview

The Polaris Dashboard is the primary operator console for eyes into Polaris' backend — an immersive, tactical command interface with a dark HUD aesthetic (cyan/teal glow, scanline overlay, bubbly rounded elements) optimized for desktop and mobile viewing.

The entire interface is a **single-file application**: `Polaris_page.html` (3,078 lines / ~153 KB of self-contained HTML + CSS + JS, no build step) served from a lightweight nginx container. All intelligence — chat, memory, voice synthesis, task execution, radar/vision telemetry — lives server-side in the **Polaris Gateway**.

| Component | Address |
|---|---|
| Dashboard UI (nginx static) | `http://localhost:7000/` or `http://192.168.50.51:7000/` |
| Polaris Gateway — REST + Socket.IO | `http://<host>:8082/` |
| Radar / Vision Socket.IO bridge | `http://<host>:8083/` (gateway container port 5001) |
| Active users | RICH, MATT |

**Version 4.6.1 headline features (compact-HUD + display-size + live split dial pass):**

- **Compact tactical top bar (~40 % shorter)** — `--header-height` 80 px → `clamp(38px, 4.5vh, 46px)`; slimmer nav buttons (`padding: 4px 10px`), tightened 5 px menu gaps, 15 px-class logo text, 7 px status dots, slimmed user selector and connection pill.
- **Tactical-ops-first split, operator-tunable (v4.6.1)** — the bottom **TACTICAL OPERATIONS** panel defaults to `flex: 0 0 70%` (chat ≈ 30 %); dragging the divider anywhere in **30 %–70 %** lets chat become the dominant pane (up to 70/30) and back. The drag writes inline `flex-basis` (the old `style.height` write was shadowed by the definite flex basis and did nothing); position persists in `localStorage['polaris_split']`. Chat paddings/bubbles/input row were trimmed (v4.6). Scale-agnostic.
- **⋯ Display Size control** — an HDI-style whole-page scale dial (70 %–150 %, step 5 %) in the nav bar, persisted in `localStorage` and applied pre-paint so there is no scaling flash (see *Display Size Control*).
- **Zoom-proofed shell** — nav height, deck height/margins and the edit-rule modal overlay are all divided by `var(--ui-scale)`; the layout snaps viewport-exact at *any* dial value and is pixel-identical at 100 %.
- **LEARNING tab** — third tab in TACTICAL OPERATIONS: Polaris's staged heuristics surface as review cards with **Approve / Reject / Edit**, plus a read-only LIVE MEMORY pane (see *LEARNING Tab*).

**Headline features (v4.4+ era, retained):**

- **Holodeck split-view** — resizable top chat pane + bottom **TACTICAL OPERATIONS** panel with three tabs: **TASK FEED** (live Gateway task events), **MEMORY VAULT** (full CRUD + semantic search over Polaris's long-term memory) and **LEARNING** (staged-heuristic review).
- **Hardened sequential TTS voice pipeline** — one voice at a time, error-storm eliminated (see TTS section).
- **Per-message audio controls** — every Polaris bubble carries a ▶ replay button and a × close button.
- **Miss Pi Vision Stream** dropdown HUD — live camera-detection telemetry and narration.
- **CSI Radar Tracking** dropdown HUD — position/velocity/motion telemetry on an animated canvas scope.
- Dropdown HUD panels for Session Diagnostics and Memory Architecture.

---

## Changelog: v4.5 → v4.6.1

| Change | Detail |
|---|---|
| Compact header (density pass) | `--header-height` 80 px → `clamp(38px, 4.5vh, 46px)`; `.logo-text` down to `clamp(13px, 1.5vw, 15px)`; `.nav-btn` `padding: 4px 10px`; `.nav-menu` gap 5 px; `.radar-status-indicator` dots 7 px; user selector, connection pill and the TACTICAL OPERATIONS header/tab row all tightened. Net top-bar height down ~40 %. |
| Chat condensed, tactical widened | `.holodeck-bottom-section` default sizing `flex: 0 0 76%` (was 70 %) → TACTICAL OPS 76 % / chat ≈ 24 %; chat container, bubble and input paddings trimmed. `min-height: 180px; max-height: 88%` retained. |
| v4.6.1 split dial (operator) | Default reset to `flex: 0 0 70%` (TACTICAL 70 % / chat 30 %) and the divider made actually draggable across **30 %–70 %** — inline `flex-basis` is the live control (a definite flex-basis shadows `height`, so the old `style.height` drag was a silent no-op; `.collapsed` gains `flex-basis: 0 !important` so the ▼ toggle works too). Layout transitions freeze during drag for cursor-exact tracking. Position persists in `localStorage['polaris_split']` (clamped 30–70 on restore). |
| ⋯ Display Size menu | New nav button `uiScaleBtn` (⋯) → `uiScaleDropdown`: cyan range slider **70 %–150 % (step 5 %)**, live `%` label, **RESET 100 %** button. Wired through the existing `toggleDropdown` machinery (single dropdown open at a time; `×` and click-outside close). |
| Whole-page zoom dial | `:root { --ui-scale }` × `body { zoom: var(--ui-scale) }` — one dial scales everything; JS only ever sets the CSS variable (`applyUiScale`, clamped 70–150, 2-dp). Persistence in `localStorage['polaris_ui_scale']` with `try/catch`; a pre-paint inline `<script>` in `<head>` re-applies the saved scale **before first paint** (no unscaled flash); `initUiScale` IIFE re-syncs the label + slider thumb defensively on load. |
| Zoom-proofed geometry | Every surface that measures the viewport divides by the dial: `.nav-bar` height, `.holodeck-container` height + margin-top, `.edit-modal-overlay` full-screen dims (`calc(100vw\|100vh / var(--ui-scale))`). The resize-handle computes its percentage from the live container rect, so it is scale-agnostic by construction. Pixel-identical at 100 %. |
| LEARNING tab | Third TACTICAL OPERATIONS tab (ported from the legacy page into the LGI HUD theme): **STAGED RULES** review cards with APPROVE / REJECT / EDIT, a read-only **LIVE MEMORY** pane, a `STAGED n • LIVE n` counts pill and a REFRESH button. Edit flow uses a modal overlay (rule text, category, confidence) that PUTs the amended rule to the gateway. |

## Changelog: v4.4 → v4.5

| Change | Detail |
|---|---|
| Phantom camera handshake deleted (v1.0.6) | The vision socket no longer emits `vision_connect` on page load — a dashboard VIEWER is not a camera. The fake announce (camera `miss-pi-front` @ 640x480, a MissPi-era artifact) overwrote the gateway's `vision_device_name`, fired a bogus *"Vision stream online"* JARVIS announcement on every page load, and left the stream flag ghosted after tab close. Viewers now only `join_vision`; detections/narration flow unchanged. |
| Honest stream-state pill (v1.0.7) | The STREAM status line no longer keys off the viewer's own socket state: the `join_vision` ack (`vision_history.stream_active`) carries the CAMERA's real state on join, typed `jarvis_announcement` events positively drive `STREAM ACTIVE` / `STREAM INACTIVE`, and page load stays on `WAITING FOR STREAM` until server truth arrives. |
| Dead `polaris_vision` listener removed (v1.0.7) | The gateway's `polaris_vision` fan-out emitted into room `radar_dashboard`, which the vision socket never joined — the listener could never have fired (orphaned on both ends). Deleted; narration already flows via `vision_update`. |

## Changelog: v4.3 → v4.4

| Change | Detail |
|---|---|
| SSH Sandbox tab retired | The bottom panel's second tab is now **MEMORY VAULT** — a full document browser/editor for the Polaris memory vault (previously the SSH Sandbox command runner). |
| TTS error-storm fix | Every cleanup site now uses the spec-safe `resetAudioElement()` helper instead of `ttsPlayer.src = ''`. The old empty-string reset re-invoked the media load algorithm → instant `error` event → `onerror` re-assigned `src = ''` → infinite error loop (~60 console errors from ~3 requests), with stale queued errors cascading into repeated `/api/tts` gateway hits. |
| onerror re-entrancy guard | `setupPlayerHandlers` now ignores error events when the element has no `src` attribute — a genuine playback failure logs exactly once. |
| Sequential TTS queue | One shared hidden `<audio id="ttsPlayer">` element; messages synthesize and play strictly one at a time in arrival order. |
| Per-message replay | ▶ re-synthesizes a message on demand (jumps the queue head); × closes the bubble and stops/dequeues its audio. |
| Miss Pi Vision dropdown | New nav dropdown streaming live detection results (class, confidence, narration) from the vision pipeline. |
| Stale-doc corrections | Chat window is the **last 10 messages total** (not 5 per sender); fallback history sync runs every **10 s** with WebSocket as primary (not a 3 s poll); Radar/Vision connect via Socket.IO on **:8083** (not a raw WebSocket on :5001); bottom panel resizes to **20 %–70 %** of container height (not fixed pixel bounds). |

---

## System Architecture

```
┌───────────────────────────────────────────────────────────────┐
│            OPERATOR BROWSER (desktop / RedMagic phone)          │
│        Polaris_page.html — single-file tactical HUD            │
└──────┬─────────────────────────────────────────┬──────────────┘
       │ static HTML/JS (HTTP :7000)               │ Socket.IO + REST
       ▼                                          ▼
┌──────────────────────┐      ┌─────────────────────────────────────┐
│ POLARIS DASHBOARD    │      │ POLARIS GATEWAY (Dell Mainframe)     │
│ nginx:alpine, :7000  │      │                                       │
│ COPY'd index.html    │      │ REST :8082                            │
│ wildcard CORS + SPA  │      │  /api/chat   /api/history/{user}      │
│ fallback             │      │  /api/tts    /api/tts/cancel           │
└──────────────────────┘      │  /api/memory/stats                    │
                              │  /api/memory/vault/* (8 endpoints)   │
                              │  /api/heuristics[/staged]             │
                              │  /api/heuristics/approve|reject|edit  │
                              │                                       │
                              │ Socket.IO :8082                       │
                              │  chat_message · tool_execution         │
                              │  task_started/_completed/_failed     │
                              │                                       │
                              │ Socket.IO :8083 (container :5001)     │
                              │  radar_update/_status/_snapshot      │
                              │  vision_update/_history              │
                              │  jarvis_announcement (typed)         │
                              └────────────┬──────────────────────────┘
                                           │
                    ┌──────────────────────┼─────────────────────┐
                    ▼                      ▼                     ▼
             SQLite memory         ChromaDB vectors        Kokoro TTS engine
             (structured store,    (nomic-embed-text,      (24 kHz WAV synthesis
              chat + heuristics +   vault_documents          for /api/tts)
              memory vault)        collection)
```

---

## Holodeck Split-View Layout

```
┌──────────────────────────────────────────────────────────────┐
│ NAV: POLARIS │ USER selector │ Chat History │ Memory │        │  ← top bar height
│      Radar │ Vision │ ⋯ │ WS status                           │     clamp(38px, 4.5vh, 46px)
├──────────────────────────────────────────────────────────────┤
│ TOP — CHAT (flex remainder ≈ 30 %)                             │
│   [pinned message input] [AUDIO RESPONSE toggle]              │
│   reverse-chronological feed (10-message window)              │
├──── RESIZE HANDLE (drag: bottom pane 30 % – 70 %) ────────────┤
│ BOTTOM — TACTICAL OPERATIONS (default 70 %)                    │
│   [▼/▲ TACTICAL OPS collapse] [TASK FEED] [MEMORY VAULT] [LEARNING] │
│   active tab content area                                     │
└──────────────────────────────────────────────────────────────┘
```

- **Collapse toggle** — collapses the bottom pane to a slim `▼ TACTICAL OPS` bar; expands to `▲ TACTICAL OPS`. Collapsing forces `flex-basis: 0 !important` so it beats any dragged inline basis (v4.6.1 fix).
- **Resize (v4.6.1 dial)** — drag the grip handle to set the bottom pane as a percentage of the holodeck container, clamped to **30 %–70 %** (dragging fully down makes chat dominant at 70/30; the un-dragged default is the CSS `flex: 0 0 70 %`). The drag writes **inline `flex-basis`** — a definite flex-basis shadows `height`, so this is the control surface that actually works; dragging also clears the collapsed/expanded class state. Both panes' layout transitions freeze on `mousedown` (restored on `mouseup`) so the divider tracks the cursor exactly. The last dragged value persists to `localStorage['polaris_split']` and is re-applied (clamped 30–70, try/catch) on load; the percent math reads the live container rect, so it stays exact under any `--ui-scale`.

## Navigation Bar & Dropdown HUD Panels

Only one dropdown is open at a time; `×` closes. The **Radar** and **Vision** buttons carry live status dots that mirror their socket connection state. The bar itself is deliberately compact (`--header-height: clamp(38px, 4.5vh, 46px)`): slim buttons, tightened 5 px menu gaps, a 15 px-class logo glyph and 7 px status dots.

| Nav button | Dropdown panel | Contents |
|---|---|---|
| Chat History | **Session Diagnostics** | SESSION ID (`SESSION_{USER}_{epoch}`), ACTIVE USER, MESSAGES, DURATION (`mm:ss`), and a **Sliding Window** preview of the last 10 messages (40-char snippets). Updates every 1 s. |
| Memory | **Memory Architecture** | SQLite block (USERS / MESSAGES / HEURISTICS / OPS-SEC) and ChromaDB block with GLOBAL / USER / CONTEXT tabs (vector counts) + OPS/SEC. Polls `/api/memory/stats` every 5 s; ops/sec computed from counter deltas between polls. |
| Radar | **CSI Radar Tracking** | 350 × 350 canvas scope + telemetry rows (see CSI Radar section). |
| Vision | **Miss Pi Vision Stream** | Detection telemetry + narration (see Miss Pi Vision section). |
| `⋯` (`uiScaleBtn`) | **Display Size** | Whole-page scale dial — slider 70 %–150 % (step 5 %), live `%` label, **RESET 100 %** (see *Display Size Control*). |

## Display Size Control (⋯ HDI Dial)

The nav's `⋯` button (`uiScaleBtn`) opens the **Display Size** dropdown — a single HDI-style dial that scales the entire page, not just fonts.

**Architecture — one CSS variable does everything:**

```css
:root { --ui-scale: 1; }          /* dial value, unitless (0.70 – 1.50) */
body { zoom: var(--ui-scale); }   /* the whole page scales off this line */
```

- **JS never resizes anything itself** — `applyUiScale(pct)` clamps **70–150**, writes `--ui-scale = pct/100` (2-dp) on `document.documentElement`, syncs the live `%` label and slider thumb, and persists the integer to `localStorage`.
- **Persistence key** — `polaris_ui_scale`; every read/write is wrapped in `try/catch` (blocked storage degrades gracefully — the scale still applies for the session).
- **Pre-paint apply** — a tiny inline `<script>` in `<head>` reads the key, clamps to 70–150 and sets `--ui-scale` **before first paint**, so loading at a non-default scale shows no unscaled flash. The `initUiScale` IIFE then re-syncs label + thumb defensively on module load.
- **RESET 100 %** — calls the same `applyUiScale(100)` path (storage updated immediately).
- **Dropdown behavior** — rides the standard `toggleDropdown('uiScale')` machinery: one dropdown open at a time; `×` and click-outside close.
- **Slider mechanics** — `70–150`, step `5`; custom cyan webkit/moz thumb styling (`.ui-scale-dropdown`).
- **Graceful degradation** — engines without CSS `zoom` (Firefox before 126) simply ignore the dial; the rest of the UI is unaffected.

**Zoom-proofing — everything that measures the viewport divides by the dial:**

| Site | Compensation |
|---|---|
| `.nav-bar` height (fixed header) | `calc(var(--header-height) / var(--ui-scale))` |
| `.holodeck-container` height / margin-top | `calc((100vh - var(--header-height)) / var(--ui-scale))` and `calc(var(--header-height) / var(--ui-scale))` |
| `.edit-modal-overlay` full-screen dims | `calc(100vw / var(--ui-scale))` × `calc(100vh / var(--ui-scale))` |
| Drag-resize handle % math | reads the live container `getBoundingClientRect()` — scale-agnostic by construction |

At 100 % the dial is a no-op: the layout is pixel-identical to the pre-dial page.

## Chat System

**Identity** — user selector offers **RICH** / **MATT** (default `rich`).

**Transport** — Socket.IO to `:8082` is the primary channel. Sending a message POSTs `/api/chat` with `{prompt, user_id, client_source: 'dashboard_browser', session_id: '<user>_default'}` and **renders the user bubble locally immediately**; Polaris's reply renders from the room `chat_message` echo (healthy socket) or from the POST body itself while the socket stays down (v1.0.4 chat repair — the gateway only broadcasts the echo after its LLM turns complete, and a dead socket never broadcasts at all; the old wait-for-echo-only render path was the "dashboard goes silent" failure). Incoming `chat_message` events are filtered to the active user or Polaris, with user-side echoes collapsed against the local bubble by sender+content within a 150 s window; `tool_execution` events are logged to console only (the follow-up chat message reports completion).

**Fallback sync** — every 10 s, `syncChatHistory()` fetches `/api/history/{user}` and content-diffs against the local `messageHistory`; any message the WebSocket missed is inserted. Failures are silent (WebSocket remains primary). (v1.0.4 chat repair) The endpoint serves the **paired** unification format (`{history: [{user_message, ai_response, timestamp, sender}]}`) — the loaders previously checked a flat `data.messages` field that no longer exists, so both this safety net and the on-load history render were silently dead; both now expand pairs into user + Polaris bubbles. Every Socket.IO *re*-connect also re-pulls history to catch up on messages missed while the socket was down, a 15 s heartbeat prods a stalled socket, and reconnection is unlimited (was capped at 10 attempts ≈ permanent silence after any outage).

**Rendering** — `addMessageToChat()` prepends newest at the top and the feed keeps only the **last 10 messages total**. Polaris bubbles get audio controls (▶ replay / × close). Content containing `{"tool": …}` JSON is reduced to the human-readable text preceding the tool call. On page load, `fetchChatHistory()` renders the last 10 history messages newest-first with **no auto-play**, expanding each paired `{user_message, ai_response}` history entry into its two bubbles (v1.0.4 chat repair). `fetchChatHistory()` carries a stale-response guard so racing fetches (reconnect catch-up vs page init) can't interleave renders.

---

## TTS Voice Pipeline (v4.4 Hardened)

**Model** — one shared, hidden `<audio id="ttsPlayer">` element driven by a strictly sequential queue.

**Playback flow**

1. A Polaris message arrives → `playTTSResponse()` (gated by the **AUDIO RESPONSE** checkbox) → `enqueueTTS()` (skips if that message is already playing or queued; supports `atFront` priority).
2. `pumpTTSQueue()` plays **one message at a time**, in arrival order:
   - `resetAudioElement(ttsPlayer)` — clears `onended`/`onerror` handlers, revokes the previous blob object URL, then `removeAttribute('src')` + `load()`.
   - `POST /api/tts {text}` (abortable via `AbortController`) → Kokoro WAV blob → `URL.createObjectURL()` → assign `src` → `setupPlayerHandlers()` → `play()`.
   - Polls every 200 ms until the message finishes or is stopped, then advances the queue.
3. Stops — `stopAllAudioPlayers()` aborts every per-message controller and resets the element; `cancelCurrentStream()` additionally POSTs `/api/tts/cancel` and tears down any MediaSource.

**Why `resetAudioElement()` exists (the v4.4 fix)** — assigning `ttsPlayer.src = ''` re-invokes the media load algorithm; an empty source fails instantly and fires an `error` event; the `onerror` handler then assigned `src = ''` again → infinite error loop. Because error events queue asynchronously, stale errors could also fire **after** the next message's blob loaded — killing fresh playback and cascading into repeated `/api/tts` requests. A src-less element (`removeAttribute('src')` + `load()`) fires **no** error event, making cleanup silent. As belt-and-braces, `onerror` begins with a re-entrancy guard — `if (!ttsPlayer.getAttribute('src')) return;` — so a genuine decode/network failure still logs exactly once.

**Per-message controls** — ▶ re-synthesizes that message on demand (inserts at the queue front); × closes the bubble, dequeues it, and aborts playback if active. The **AUDIO RESPONSE** toggle gates auto-play only — manual ▶ always works.

**Known-good console signatures** — one `[TTS] Synthesizing message …` and one `[TTS] Finished message …` per message; **zero** `[TTS] Audio playback error` entries in normal operation.

**Legacy note** — `streamTTSAudio()` (`/api/tts-stream`, MediaSource chunked streaming) is retained as dead code; the live pipeline uses whole-blob `/api/tts`.

## Task Feed System

- Connects Socket.IO to `:8082`, emits `join_tasks {room: 'tasks_dashboard'}`.
- Events → feed rows: `task_started` → **▶** TASK STARTED · `task_completed` → **✓** TASK COMPLETED · `task_failed` → **✗** TASK FAILED. Each row shows `task_id • HH:MM:SS`.
- Newest at top; rolling window keeps the **last 50 items**.
- Auto-reconnect: unlimited attempts (v1.0.5 — was capped at 10, which meant permanent feed silence after any outage ≥ ~10 s), 1 s initial delay, 10 s delay max; websocket transport with polling fallback.

## Memory Vault System

Full operator access to Polaris's long-term distilled knowledge — the Obsidian-style markdown vault bind-mounted at `/polaris_memory_vault` (categories 00-Core → 05-Attachments), embedded into the ChromaDB `vault_documents` collection via `nomic-embed-text` with a 30 s watchdog re-embedding changed files.

**UI**

- **Status bar** — gateway online indicator, doc / indexed counts, vault path label.
- **Browser panel** — category filter (ALL / CORE / ARCHITECTURE / USER / LESSONS / JOURNAL / ATTACHMENTS), search input + SEARCH, search-mode selector (**HYBRID / KEYWORD / SEMANTIC**), BROWSE / NEW / REINDEX buttons, and the document list (title, meta, snippet — up to 200 docs).
- **Editor panel** — document title + meta, SAVE / DELETE actions, a compose bar (category + title) revealed when composing a NEW document, full-width textarea, inline status messages. The textarea stays disabled until a document is selected or NEW is pressed. CORE is deliberately absent from the new-document category list (protected doctrine).

## LEARNING Tab — Heuristic Review Console (v4.6)

Third tab in TACTICAL OPERATIONS (ported from the legacy page into the LGI HUD theme). This is the operator's gate on Polaris's self-proposed behavioral rules: **nothing becomes doctrine without an Approval here.**

**Toolbar** — STAGED RULES / LIVE MEMORY sub-tabs, a `STAGED n • LIVE n` counts pill, and a REFRESH button. `initLearningHeuristics()` pulls both panes on init and re-polls every **20 s** (`refreshLearningPanels`); switching to LIVE MEMORY re-fetches immediately.

**STAGED RULES pane** — rule cards from `GET /api/heuristics/staged`: category pill, `Confidence: x.xx`, rule text, and three actions:
- **Approve** → `POST /api/heuristics/approve/<id>` — promotes the rule into the live heuristics store.
- **Reject** → `DELETE /api/heuristics/reject/<id>` — drops the proposal.
- **Edit** → opens the **EDIT STAGED RULE** modal (rule text, category select, confidence input) → `PUT /api/heuristics/edit/<id>` with `{rule, category, confidence}` — the gateway ignores the legacy `{text}` key (contract note kept inline in the source). Save shows a success/error toast and re-pulls staged rules; Cancel/backdrop-dismiss clears the pending edit.

**LIVE MEMORY pane** — read-only cards over the active heuristics store (`GET /api/heuristics`), newest first, capped at **50 cards** in the DOM.

**Defensive parsing** — `extractRuleArray()` tolerates both gateway envelopes (the `/staged` endpoint returns an array; the active store returns `{heuristics: [...]}`); all rule fields pass through `escapeHtml()` before injection; empty results render a friendly "nothing staged" state; fetch failures log once to console (no error-storm).

---

## CSI Radar Integration

- Socket.IO to `:8083` (gateway container port 5001), emits `join_radar {room: 'radar_dashboard'}`.
- Events — `radar_update` (x, y, velocity, motion_detected, motion_confidence), `radar_status`, and `radar_snapshot` (initial state on connect: `current_position[]`, velocity, motion fields).
- **Canvas scope** (350 × 350) — concentric range rings, crosshairs, N/E/S/W compass labels, glowing position dot with a fading trail (**max 20 points**).
- **Telemetry rows** — POSITION X, POSITION Y, VELOCITY, MOTION (DETECTED / NONE), CONFIDENCE (%), STATUS (CONNECTED / DISCONNECTED / ERROR).

## Miss Pi Vision Stream

- Socket.IO to `:8083` (same bridge as radar), emits `join_vision {room: 'vision_dashboard'}` — viewer membership only: this socket never claims to be a camera (v1.0.6 removed the phantom `vision_connect` handshake).
- Events — `vision_update` (detections array → count, primary object `CLASS (CONFIDENCE %)`, narration), `vision_history` (last 10 detections **+ the camera's real `stream_active` state** on join), `jarvis_announcement` (typed camera on/off announcements: `connection` / `disconnection` / `detection`).
- **Honest status line** — keyed to CAMERA truth, never to viewer socket state (v1.0.7): `WAITING FOR STREAM` on page load until server truth arrives → `STREAM ACTIVE` (green) when the gateway reports the camera live via `vision_history.stream_active` or a typed `connection` announcement / `STREAM INACTIVE` (red) on a genuine camera offline (typed `disconnection` announce, stale-feed watchdog, or camera-socket death). Viewer DISCONNECTED still shows `STREAM INACTIVE`; a viewer reconnect shows `WAITING FOR STREAM` until the server speaks.
- **UI** — italic narration quote box (default "Polaris is watching…"), DETECTIONS, PRIMARY OBJECT, LAST UPDATE (`HH:MM:SS`), STREAM rows.

## API Reference

**REST** (base `http://<host>:8082`)

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/chat` | Send prompt `{prompt, user_id, client_source: 'dashboard_browser', session_id: '<user>_default'}` |
| GET | `/api/history/{user}` | Chat history (oldest-first; paired `{user_message, ai_response}` entries; UI renders the last 10) |
| POST | `/api/tts` | Kokoro TTS synthesis `{text}` → WAV blob |
| POST | `/api/tts/cancel` | Cancel in-flight synthesis `{session_id}` |
| GET | `/api/memory/stats` | SQLite + ChromaDB counters (Memory Architecture panel) |
| GET | `/api/memory/vault/stats` | Vault totals (docs / indexed) |
| GET | `/api/memory/vault/list?limit=200&category=` | Browse vault documents |
| POST | `/api/memory/vault/query` | Search `{query, mode: hybrid\|keyword\|semantic}` |
| GET | `/api/memory/vault/doc?path=` | Fetch one document |
| POST | `/api/memory/vault/write` | Create a document |
| POST | `/api/memory/vault/update` | Save edits |
| POST | `/api/memory/vault/delete` | Delete a document |
| POST | `/api/memory/vault/reindex` | Re-embed the vault into ChromaDB |
| GET | `/api/heuristics/staged` | Staged rule proposals awaiting review (array payload → Learning tab STAGED RULES) |
| GET | `/api/heuristics` | Active heuristics store `{heuristics: [...]}` → LIVE MEMORY pane |
| POST | `/api/heuristics/approve/{id}` | Promote a staged rule into the live store |
| DELETE | `/api/heuristics/reject/{id}` | Discard a staged proposal |
| PUT | `/api/heuristics/edit/{id}` | Amend a staged rule `{rule, category, confidence}` |

**Socket.IO channels**

| Port | Room (join emit) | Events |
|---|---|---|
| 8082 | — | `chat_message`, `tool_execution` |
| 8082 | `tasks_dashboard` (`join_tasks`) | `task_started`, `task_completed`, `task_failed` |
| 8083 | `radar_dashboard` (`join_radar`) | `radar_update`, `radar_status`, `radar_snapshot` |
| 8083 | `vision_dashboard` (`join_vision`) | `vision_update`, `vision_history`, `jarvis_announcement` |

All Socket.IO clients use `transports: ['websocket', 'polling']` with unlimited reconnection (v1.0.5: the radar/vision/task-feed clients were capped at 10 attempts — ~10 s of outage then permanent silence with a dead feed and no visible retry; the chat socket was fixed in v1.0.4 and the rest now match: Infinity attempts, 1 s initial delay, 10 s delay max). The client library is Socket.IO **4.5.4** loaded from the CDN in `<head>`.

---

## Build & Deploy

The HTML is **baked into the image** (no bind mount) — every source change requires a rebuild:

```
# Rebuild + redeploy (only the dashboard service; never touch ollama)
docker compose build polaris-dashboard
docker compose up -d polaris-dashboard

# Verify
docker ps --filter name=polaris-dashboard
curl -s http://localhost:7000/ | head
```

- **Compose service** — `polaris-dashboard` in `/home/causeiam/docker-containers/docker-compose.yml`; build context `./freeroam/polaris-dashboard`; ports `7000:80`; `depends_on: questdb`.
- **Dockerfile** — `FROM nginx:alpine`; `COPY nginx.conf → /etc/nginx/conf.d/default.conf`; `COPY Polaris_page.html → /usr/share/nginx/html/index.html`.
- **nginx** — `try_files $uri $uri/ /index.html` SPA fallback; wildcard CORS headers; `OPTIONS` preflight → `204`.
- After a rebuild, **hard-refresh the browser (Ctrl+Shift+R)** to bypass cached HTML.

## Browser Verification Checklist (v4.6)

**Shell & panels**
- [ ] POLARIS logo renders; WS indicator shows ONLINE (cyan)
- [ ] User selector offers RICH / MATT
- [ ] Chat input pinned below nav; AUDIO RESPONSE checkbox visible
- [ ] Messages appear newest-at-top; feed prunes to the last 10
- [ ] Chat History / Memory / Radar / Vision / ⋯ Display Size dropdowns open, show live values, close via ×
- [ ] Memory panel: SQLite + ChromaDB (GLOBAL/USER/CONTEXT) counts refresh every 5 s
- [ ] Session Diagnostics: SESSION ID / ACTIVE USER / MESSAGES / DURATION tick every second

**Holodeck bottom panel**
- [ ] Expand/collapse toggle (▼/▲ TACTICAL OPS) works
- [ ] Drag resize handle; bottom pane clamps between 30 % and 70 % (default 70 %) — pulled fully down the chat pane is dominant (70/30)
- [ ] Reload after a drag: the split returns to the persisted position (`localStorage['polaris_split']`); storage-blocked browsers fall back to the 30/70 default
- [ ] TASK FEED / MEMORY VAULT / LEARNING tabs switch cleanly
- [ ] Task Feed rows arrive with ▶/✓/✗ icons + timestamps; window caps at 50
- [ ] Memory Vault: BROWSE lists docs; HYBRID/KEYWORD/SEMANTIC search return; open → edit → SAVE persists; NEW composes with category + title; DELETE removes; REINDEX succeeds

**LEARNING (heuristic review)**
- [ ] Staged rules load as cards with category pill, confidence and APPROVE / REJECT / EDIT; the "nothing staged" empty state renders
- [ ] APPROVE promotes the rule (card disappears); REJECT discards it (card disappears)
- [ ] EDIT modal opens pre-filled; saving persists via the gateway PUT and a toast confirms; Cancel / backdrop dismiss restores state
- [ ] LIVE MEMORY lists active heuristics newest-first (≤ 50 cards); counts pill tracks `STAGED n • LIVE n`
- [ ] Background 20 s refresh keeps both panes current without focus

**Display size (⋯ dial)**
- [ ] Slider scales the whole page live across 70 %–150 %; the `%` label tracks it
- [ ] Reload at a non-default scale shows no unscaled flash (pre-paint script)
- [ ] RESET 100 % restores the default layout instantly and persists
- [ ] Fixed header, deck edges and the edit-rule modal stay viewport-exact at 70 % and 150 %
- [ ] Drag-resize of the bottom pane still clamps correctly at non-100 % scales

**Voice**
- [ ] Each Polaris reply auto-plays once: one `/api/tts` request, one finish log, zero `[TTS] Audio playback error`
- [ ] Back-to-back messages queue and play strictly one at a time (no overlap, no cascade)
- [ ] ▶ replays an old message immediately; × on a playing message stops its audio
- [ ] Unchecking AUDIO RESPONSE silences auto-play; re-checking restores it (manual ▶ still works)

**Sensors**
- [ ] Radar: canvas rings/crosshairs/compass render; POSITION/VELOCITY/MOTION/CONFIDENCE/STATUS update live
- [ ] Vision: WAITING FOR STREAM on load until server truth; STREAM ACTIVE + narration while the LG camera streams; STREAM INACTIVE (red) on a genuine camera offline; page reload fires NO bogus *"Vision stream online"* announcement (v1.0.7 honest-pill check)

**Resilience**
- [ ] Restarting the gateway → all four sockets auto-reconnect; indicators recover
- [ ] Messages missed by WebSocket are recovered by the 10 s fallback sync
- [ ] Mobile viewport sanity check (Red Magic 9 Pro over LAN)

## Technical Notes

- **Single-file architecture** — no bundler, no framework; vanilla JS + Socket.IO 4.5.4 (CDN). All state lives in module-scope variables (`currentUserId`, `messageHistory`, `ttsQueue`, `messageAudioPlayers`, per-system socket handles).
- **Timing constants** — memory stats 5 s · session diagnostics 1 s · TTS completion poll 200 ms · chat fallback sync 10 s · socket reconnect 10 × 1 s · task-feed window 50 · radar trail 20 · chat window 10 · vault list 200 · vault stats 30 s · Learning panels 20 s (LIVE pane re-fetches on open) · UI scale clamp 70–150, step 5 · split drag clamp 30–70 % (persisted as polaris_split).
- **Duplicate safety** — the sending device never appends locally; rendering happens exclusively via the gateway's Socket.IO broadcast, plus content-based diffing in the fallback sync.
- **Reverse-chronological window** — the UI intentionally keeps only the last 10 bubbles; full history remains in the gateway's SQLite store.
- **Mobile** — `clamp()`-based typography and touch-friendly targets; validated on the Red Magic 9 Pro.
- **Honest vision status** — the STREAM pill tracks gateway-side camera truth only (`vision_history.stream_active` + typed `jarvis_announcement`); the viewer's own socket state supplies only the interim `WAITING FOR STREAM`.
- **Display-size dial** — `--ui-scale` (0.70–1.50) on `:root` × `body { zoom }`; JS only ever sets the variable (`applyUiScale`); persisted as `localStorage['polaris_ui_scale']` and re-applied pre-paint by the inline head script (no flash). Viewport-measuring surfaces (nav, deck, edit-modal overlay) divide by the dial; the drag-resize math is rect-based and scale-agnostic. Pixel-identical at 100 %.
- **Split-view dial (v4.6.1)** — default `flex: 0 0 70%` (tactical-dominant); drag range 30 %–70 % of the deck. The divider writes **inline `flex-basis`** because a definite flex-basis shadows `height` on a flex item — the v4.5–v4.6 `style.height` drag was a silent no-op, and `.collapsed` needed `flex-basis: 0 !important` to collapse at all. Persisted as `localStorage['polaris_split']` (30–70 clamp on restore, try/catch); both panes' layout transitions freeze during drag for cursor-exact tracking.
- **LEARNING gate** — staged heuristics surface for operator Approve/Reject/Edit via the gateway `/api/heuristics/*` endpoints (20 s poll); a rule only enters the live store on explicit Approve. Both gateway payload shapes are tolerated; every injected field is HTML-escaped.
- **History** — this document supersedes v4.5 (the v4.6 compact-HUD + display-size + LEARNING pass; v4.6.1 adds the live split-view dial); earlier versions remain retrievable from git history.
