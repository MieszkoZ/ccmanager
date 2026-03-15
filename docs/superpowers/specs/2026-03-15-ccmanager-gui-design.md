# CCManager GUI — Design Spec

**Date:** 2026-03-15
**Approach:** B — Bun backend + React SPA in browser
**Status:** Approved

---

## Overview

Replace CCManager's Ink/TUI rendering layer with a React SPA served by a local Bun HTTP server. All existing services (SessionManager, WorktreeService, StateDetector ×8, ConfigReader, ProjectManager, hooks, auto-approval) remain unchanged in the Bun process. The browser provides the UI.

---

## Architecture

```
ccmanager --gui
  └─ Bun HTTP server (127.0.0.1:PORT)
  │    └─ Serves React SPA (static bundle)
  │    └─ REST API  (/api/*)
  │    └─ Single multiplexed WebSocket  (/ws)
  └─ Existing services — unchanged
  └─ Opens browser tab at http://127.0.0.1:PORT?token=<startup-token>
```

### Why Bun (not Node.js)
`Bun.Terminal` is the PTY driver. Switching to Node.js would require migrating to `node-pty`, which has different signal propagation, window-resize, and output-buffering behavior. All state detection heuristics (waiting_input, plan-selection prompts, etc.) are calibrated against `Bun.Terminal` output. Preserving Bun eliminates the riskiest rewrite.

---

## Security

- Bind exclusively to `127.0.0.1` (no LAN exposure)
- Generate a cryptographically random token at startup (32 bytes, hex-encoded)
- Embed token in the URL opened in the browser
- Validate token on every REST request (header) and WebSocket upgrade (query param)
- Model: identical to Jupyter notebook's single-use token pattern

---

## WebSocket Protocol

**Single multiplexed connection** per browser client (`/ws?token=<token>`).
Avoids Safari's per-origin concurrent WebSocket limits and TLS handshake overhead on loopback.

Frame types (JSON envelope, except PTY I/O which is binary):

```
// Binary frame: raw PTY output (server → client) or keyboard input (client → server)
// Header: 4 bytes big-endian uint32 session index + raw bytes
// Session index is a monotonically increasing integer assigned at session creation.
// Example: session index 3, 5 bytes of data → [0x00, 0x00, 0x00, 0x03, <5 bytes>]

// JSON control frames
{ type: "session:state", sessionId, state: "idle"|"busy"|"waiting_input"|"pending_auto_approval" }
{ type: "session:resize", sessionId, cols: number, rows: number }   // client → server
{ type: "session:signal", sessionId, signal: "SIGINT"|"SIGTERM" }   // client → server
{ type: "ping" } / { type: "pong" }                                 // keepalive, 10s interval
{ type: "replay:start", sessionId, byteCount: number }              // server → client on reconnect
{ type: "replay:end", sessionId }
{ type: "ack", sessionId, bytesConsumed: number }                   // client → server, flow control
```

Keyboard input from browser to PTY uses the same binary frame format as PTY output (4-byte session index header + UTF-8 encoded keypress bytes). The server routes it by session index to the correct `Bun.Terminal` stdin.

### Flow Control
Server tracks per-session pending ACK bytes. If unacknowledged bytes exceed **1MB**, server pauses PTY output delivery (does not pause the PTY itself — output continues into server-side buffer). Resumes when client ACKs bring the window below 512KB. Prevents browser memory exhaustion under verbose LLM output.

### Reconnect / Replay
Server keeps the last **100KB** of PTY output per session in a ring buffer. On reconnect, server sends `replay:start`, then delivers buffered output in **4KB chunks paced via 60fps** (server sends, client acknowledges render via ACK frames). Bulk write would freeze xterm.js parser on large sessions.

---

## State Detection Architecture

State detection stays entirely in the Bun process — it never moves to the browser.

```
PTY output stream (Bun.Terminal)
  ├─ → xterm/headless (existing StateDetector)
  │     └─ emits session state changes
  │           └─ WebSocket { type: "session:state", ... } broadcast
  └─ → replay ring buffer → WebSocket binary frames → browser xterm.js (render only)
```

The browser renders terminal output and displays state badges. It never interprets terminal content.

---

## Session Lifecycle

PTYs are owned by the Bun process, not the browser tab.

| Event | Behavior |
|---|---|
| Browser disconnects | PTY keeps running; 30-second keepalive grace period begins |
| Browser reconnects within grace | Session resumes; last 100KB replayed |
| No reconnect after grace | Session remains running (default) |
| `--kill-on-close` flag | Grace period skipped; all sessions killed immediately on disconnect |
| Explicit kill (UI or `ccmanager --kill`) | PTY destroyed immediately |

This matches the mental model of the existing TUI: closing the terminal window doesn't kill sessions.

---

## REST API

All requests require `Authorization: Bearer <token>` header.

```
GET  /api/projects                — list projects (multi-project mode)
GET  /api/worktrees               — list worktrees for current project
POST /api/worktrees               — create worktree
                                    body: { branch, path,
                                      copySessionData?: boolean }
                                    copySessionData copies ~/.claude/projects/<source>
                                    to ~/.claude/projects/<target> (preserves conversation history)
DELETE /api/worktrees/:id         — delete worktree; body: { deleteBranch?: boolean }
POST /api/sessions                — start session on worktree; body: { worktreeId, preset? }
DELETE /api/sessions/:id          — kill session
GET  /api/sessions/:id/state      — current session state
GET  /api/config                  — read merged config (global + project, project values win)
PUT  /api/config/global           — write global config (~/.config/ccmanager/config.json)
PUT  /api/config/project          — write project config (.ccmanager.json in git root)
GET  /api/health                  — server liveness (no auth required)
```

---

## Frontend (React SPA)

**Stack:** React 19 + TypeScript + xterm.js + Vite (build)

**Static bundle:** Vite builds the SPA to `dist/gui/`. The Bun HTTP server serves files from this directory. The `dist/gui/` directory is included in the npm package (`"files"` field in `package.json`). The `--gui` build step runs `vite build` as part of `npm run build`.

**Layout:**
```
┌─────────────────────────────────────────┐
│  Session sidebar   │  Terminal panel     │
│  (worktree list    │  (xterm.js,         │
│   + state badges)  │   active session)   │
│                    │                     │
│  [+ New worktree]  │                     │
└─────────────────────────────────────────┘
```

Key components:
- `SessionSidebar` — lists worktrees with state badges (idle/busy/waiting/auto-approval); click to switch active panel
- `TerminalPanel` — xterm.js instance per session (lazy-mount; only active panel is attached to DOM; others keep buffers in memory)
- `WorktreeDialog` — create/delete/merge worktree
- `SettingsPanel` — config editor (shortcuts, presets, auto-approval, hooks)
- `ConnectionStatus` — reconnect indicator

**xterm.js panel strategy:** Instantiate one `Terminal` per session at session start. Only the active panel renders to a DOM node; inactive panels remain in detached state (buffers preserved, no DOM reflow). This bounds browser memory regardless of session count.

---

## Pre-Implementation Spikes (go/no-go gates)

| Spike | Duration | Pass condition | Fail action |
|---|---|---|---|
| Keyboard passthrough | 1–2 days | Ctrl+C, Ctrl+D, Ctrl+Z, arrow keys work in xterm.js on macOS Chrome + Firefox + Safari with a live `claude` session | Pivot to Approach C (Electron) |
| 8-panel memory benchmark | 1 day | <500MB browser RAM, <5% idle CPU after 30 min of active output across 8 sessions | Reduce max visible panels or pivot |

---

## Approach C: Fallback (Electron)

If the keyboard passthrough spike fails, implement using Electron:

- Main process: Node.js (not Bun) — `node-pty` replaces `Bun.Terminal`
- State detection: headless xterm instance in main process fed from `node-pty` output
- IPC: `contextBridge` / `ipcMain.handle` for all service calls (async wrapper shim over Effect-ts layer)
- Build: `electron-builder` + `electron-rebuild` in CI; `electron-updater` for auto-update
- Migration: 1-week `node-pty` spike + 1-week state detection re-validation across claude/gemini/codex CLIs
- Estimated effort: 6–8 weeks vs. 4–5 weeks for Approach B

---

## Effort Estimate

**Approach B (Bun + SPA):** ~6 weeks

| Phase | Duration |
|---|---|
| Spikes (keyboard + memory) | 3 days |
| Bun HTTP/WS server + auth | 3 days |
| REST API + session lifecycle | 3 days |
| WebSocket PTY streaming + flow control | 4 days |
| React SPA + xterm.js panels | 7 days |
| Worktree dialogs + config UI | 4 days |
| CLI entry point + `--gui` flag | 1 day |
| Testing + polish | 3 days |

---

## Out of Scope (v1)

- Remote access (multi-machine) — localhost only
- Mobile / tablet browser support
- Dark/light theme toggle (uses system preference via CSS `prefers-color-scheme`)
- PWA / installable web app
