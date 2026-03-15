# CCManager GUI Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `ccmanager --gui` command that serves a React SPA over localhost, giving CCManager a browser-based GUI while keeping all existing TUI functionality intact.

**Architecture:** A Bun HTTP+WebSocket server wraps the existing services (SessionManager, WorktreeService, etc.) without modifying them. PTY output is streamed to the browser via a single multiplexed WebSocket; state detection stays server-side. The React SPA renders xterm.js terminal panels.

**Tech Stack:** Bun HTTP/WS (built-in), React 19, xterm.js (`@xterm/xterm`), Vite (SPA build), Vitest (tests)

**Spec:** `docs/superpowers/specs/2026-03-15-ccmanager-gui-design.md`

---

## File Structure

### New files

**Backend (`src/gui/`)**
| File | Responsibility |
|---|---|
| `src/gui/auth.ts` | Startup token generation + request validation |
| `src/gui/ringBuffer.ts` | Per-session 100KB circular replay buffer |
| `src/gui/ptyStreamer.ts` | Subscribes to SessionManager events; feeds ring buffers + WS clients |
| `src/gui/wsHandler.ts` | WebSocket upgrade handler; frame demux; client registry |
| `src/gui/apiRoutes.ts` | REST endpoint handlers (wraps existing services) |
| `src/gui/server.ts` | `Bun.serve` entry; wires auth + routes + WS; opens browser |

**Backend tests (`src/gui/`)**
| File | Tests |
|---|---|
| `src/gui/auth.test.ts` | Token format, validation |
| `src/gui/ringBuffer.test.ts` | Write, wrap-around, readAll |
| `src/gui/ptyStreamer.test.ts` | Ring buffer fill, WS delivery, flow control |
| `src/gui/apiRoutes.test.ts` | Each REST endpoint with mocked services |

**Frontend (`gui-frontend/`)**
| File | Responsibility |
|---|---|
| `gui-frontend/index.html` | Vite entry HTML |
| `gui-frontend/main.tsx` | React app bootstrap |
| `gui-frontend/App.tsx` | Top-level layout (sidebar + panel) |
| `gui-frontend/ws/wsClient.ts` | WebSocket wrapper; auto-reconnect; frame dispatch |
| `gui-frontend/ws/frameDemux.ts` | Binary/JSON frame parsing; routes to subscribers |
| `gui-frontend/hooks/useApi.ts` | `fetch` wrappers with auth token + error handling |
| `gui-frontend/hooks/useSessions.ts` | Session list + state derived from WS events |
| `gui-frontend/components/SessionSidebar.tsx` | Session list with state badges; click to switch |
| `gui-frontend/components/TerminalPanel.tsx` | xterm.js instance per session; keyboard/resize forwarding |
| `gui-frontend/components/WorktreeDialog.tsx` | Create / delete worktree dialogs |
| `gui-frontend/components/SettingsPanel.tsx` | Config read/write UI |
| `gui-frontend/components/ConnectionStatus.tsx` | Reconnect banner |

**Build / CLI**
| File | Change |
|---|---|
| `vite.config.ts` | New — Vite config: `gui-frontend/` → `dist/gui/` |
| `package.json` | Add Vite + xterm.js deps; add `build:gui` + `build:all` scripts |
| `src/cli.tsx` | Add `--gui` flag → start GUI server |

---

## Chunk 1: Project Setup + Spikes

### Task 1: Add frontend dependencies

**Files:**
- Modify: `package.json`
- Create: `vite.config.ts`
- Create: `gui-frontend/index.html`

- [ ] **Step 1: Add dependencies**

```bash
bun add --dev vite @vitejs/plugin-react
bun add @xterm/xterm
```

- [ ] **Step 2: Create `vite.config.ts`**

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  root: 'gui-frontend',
  build: {
    outDir: '../dist/gui',
    emptyOutDir: true,
  },
  server: {
    port: 5173,
  },
});
```

- [ ] **Step 3: Create `gui-frontend/index.html`**

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>CCManager</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/main.tsx"></script>
  </body>
</html>
```

- [ ] **Step 4: Verify Vite can build**

```bash
bun run vite build
```

Expected: `dist/gui/index.html` created.

- [ ] **Step 5: Commit**

```bash
git add vite.config.ts gui-frontend/index.html package.json bun.lockb
git commit -m "chore: add Vite + xterm.js dependencies for GUI"
```

---

### Task 2: Keyboard passthrough spike (go/no-go gate)

**Files:**
- Create: `gui-frontend/spike-keyboard.html` (temporary, deleted after spike)

This spike determines whether Approach B is viable. If Ctrl+C, Ctrl+D, Ctrl+Z fail in a major browser, pivot to Approach C (Electron). Run manually.

- [ ] **Step 1: Create spike page**

Create `gui-frontend/spike-keyboard.html`:

```html
<!DOCTYPE html>
<html>
<head><title>Keyboard Spike</title></head>
<body>
<pre id="log" style="font-family:monospace;font-size:14px;"></pre>
<div id="terminal" style="width:800px;height:400px;"></div>
<script type="module">
  import { Terminal } from '@xterm/xterm';
  import '@xterm/xterm/css/xterm.css';

  const log = document.getElementById('log');
  const term = new Terminal({ cursorBlink: true });
  term.open(document.getElementById('terminal'));

  // Log every key event xterm.js receives
  term.onKey(({ key, domEvent }) => {
    const entry = `key=${JSON.stringify(key)} code=${domEvent.code} ctrl=${domEvent.ctrlKey}\n`;
    log.textContent += entry;
  });

  term.write('Type Ctrl+C, Ctrl+D, Ctrl+Z, arrows, Enter. Check log above.\r\n');
  term.write('Pass: all keys appear in log. Fail: any key missing.\r\n');
</script>
</body>
</html>
```

- [ ] **Step 2: Serve and test**

```bash
bun run vite --config /dev/null --root gui-frontend
```

Open `http://localhost:5173/spike-keyboard.html` in Chrome, Firefox, and Safari on macOS.

**Pass criteria** (all must pass):
- Ctrl+C appears as `\x03`
- Ctrl+D appears as `\x04`
- Ctrl+Z appears as `\x1a`
- Arrow keys appear as `\x1b[A`, `\x1b[B`, `\x1b[C`, `\x1b[D`
- Enter appears as `\r`

**If any fail in any major browser:** Stop. Switch to Approach C (Electron). Do not proceed with Chunk 2.

- [ ] **Step 3: Delete spike file after validation**

```bash
rm gui-frontend/spike-keyboard.html
git commit -m "chore: keyboard passthrough spike — PASSED (or note failure)"
```

---

### Task 3: xterm.js 8-panel memory benchmark spike

**Files:**
- Create: `gui-frontend/spike-memory.html` (temporary)

- [ ] **Step 1: Create benchmark page**

Create `gui-frontend/spike-memory.html`:

```html
<!DOCTYPE html>
<html>
<head><title>Memory Spike</title></head>
<body>
<button id="start">Start flood</button>
<div id="panels" style="display:grid;grid-template-columns:repeat(4,1fr);gap:4px;"></div>
<script type="module">
  import { Terminal } from '@xterm/xterm';

  const terminals = [];
  const container = document.getElementById('panels');

  for (let i = 0; i < 8; i++) {
    const div = document.createElement('div');
    div.style.width = '400px';
    div.style.height = '200px';
    container.appendChild(div);
    const t = new Terminal({ cols: 80, rows: 24 });
    t.open(div);
    terminals.push(t);
  }

  document.getElementById('start').onclick = () => {
    // Flood all terminals with 1KB chunks at ~30fps for 30 minutes
    const chunk = 'A'.repeat(1024) + '\r\n';
    setInterval(() => {
      terminals.forEach(t => t.write(chunk));
    }, 33);
    // Check memory via performance.memory (Chrome only) every 30s
    if (performance.memory) {
      setInterval(() => {
        console.log(`Heap: ${(performance.memory.usedJSHeapSize / 1e6).toFixed(1)} MB`);
      }, 30000);
    }
  };
</script>
</body>
</html>
```

- [ ] **Step 2: Run benchmark (Chrome DevTools → Memory tab)**

```bash
bun run vite --config /dev/null --root gui-frontend
```

Open `http://localhost:5173/spike-memory.html`, click Start, run for 30 minutes.

**Pass criteria:** Heap stays below 500MB; CPU stays below 5% while idle (not flooding).

**If fails:** Reduce max concurrent visible panels to 4 in the frontend design before proceeding.

- [ ] **Step 3: Delete spike and commit results**

```bash
rm gui-frontend/spike-memory.html
git commit -m "chore: xterm.js memory spike — PASSED/FAILED (note panel limit)"
```

---

## Chunk 2: Backend — Auth + Ring Buffer

### Task 4: Startup token auth

**Files:**
- Create: `src/gui/auth.ts`
- Create: `src/gui/auth.test.ts`

- [ ] **Step 1: Write failing tests**

Create `src/gui/auth.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { generateStartupToken, validateToken } from './auth.js';

describe('generateStartupToken', () => {
  it('returns a 64-character hex string', () => {
    const token = generateStartupToken();
    expect(token).toMatch(/^[0-9a-f]{64}$/);
  });

  it('generates unique tokens on each call', () => {
    const a = generateStartupToken();
    const b = generateStartupToken();
    expect(a).not.toBe(b);
  });
});

describe('validateToken', () => {
  it('returns true for matching tokens', () => {
    const token = generateStartupToken();
    expect(validateToken(token, token)).toBe(true);
  });

  it('returns false for non-matching tokens', () => {
    expect(validateToken('abc', 'wrong')).toBe(false);
  });

  it('returns false for empty string', () => {
    expect(validateToken('abc', '')).toBe(false);
  });
});
```

- [ ] **Step 2: Run tests — expect failure**

```bash
bun run test src/gui/auth.test.ts
```

Expected: FAIL — `auth.js` not found.

- [ ] **Step 3: Implement `src/gui/auth.ts`**

```typescript
import { randomBytes, timingSafeEqual } from 'crypto';

export function generateStartupToken(): string {
  return randomBytes(32).toString('hex');
}

export function validateToken(expected: string, provided: string): boolean {
  if (!provided || provided.length !== expected.length) return false;
  try {
    return timingSafeEqual(
      Buffer.from(expected, 'utf8'),
      Buffer.from(provided, 'utf8'),
    );
  } catch {
    return false;
  }
}
```

- [ ] **Step 4: Run tests — expect pass**

```bash
bun run test src/gui/auth.test.ts
```

Expected: all 5 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/gui/auth.ts src/gui/auth.test.ts
git commit -m "feat(gui): add startup token auth"
```

---

### Task 5: Ring buffer

**Files:**
- Create: `src/gui/ringBuffer.ts`
- Create: `src/gui/ringBuffer.test.ts`

- [ ] **Step 1: Write failing tests**

Create `src/gui/ringBuffer.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { RingBuffer } from './ringBuffer.js';

describe('RingBuffer', () => {
  it('readAll returns empty buffer when nothing written', () => {
    const buf = new RingBuffer(1024);
    expect(buf.readAll().length).toBe(0);
  });

  it('stores data written within capacity', () => {
    const buf = new RingBuffer(1024);
    const data = Buffer.from('hello world');
    buf.write(data);
    expect(buf.readAll()).toEqual(data);
  });

  it('stores multiple writes in order', () => {
    const buf = new RingBuffer(1024);
    buf.write(Buffer.from('hello '));
    buf.write(Buffer.from('world'));
    expect(buf.readAll().toString()).toBe('hello world');
  });

  it('evicts oldest bytes when capacity exceeded', () => {
    const buf = new RingBuffer(5);
    buf.write(Buffer.from('hello'));  // fills exactly
    buf.write(Buffer.from('world'));  // overwrites all
    const result = buf.readAll();
    expect(result.length).toBe(5);
    expect(result.toString()).toBe('world');
  });

  it('handles write larger than capacity', () => {
    const buf = new RingBuffer(3);
    buf.write(Buffer.from('abcdef'));
    const result = buf.readAll();
    expect(result.length).toBe(3);
    expect(result.toString()).toBe('def');
  });

  it('reports correct size', () => {
    const buf = new RingBuffer(1024);
    buf.write(Buffer.from('hello'));
    expect(buf.size).toBe(5);
  });

  it('clear empties the buffer', () => {
    const buf = new RingBuffer(1024);
    buf.write(Buffer.from('hello'));
    buf.clear();
    expect(buf.size).toBe(0);
    expect(buf.readAll().length).toBe(0);
  });
});
```

- [ ] **Step 2: Run tests — expect failure**

```bash
bun run test src/gui/ringBuffer.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement `src/gui/ringBuffer.ts`**

```typescript
/**
 * Circular byte buffer. When capacity is exceeded, oldest bytes are overwritten.
 * Used to store the last N bytes of PTY output per session for reconnect replay.
 */
export class RingBuffer {
  private readonly buf: Buffer;
  private readonly capacity: number;
  private head = 0;   // write position
  private _size = 0;  // valid bytes stored

  constructor(capacity: number) {
    this.capacity = capacity;
    this.buf = Buffer.allocUnsafe(capacity);
  }

  get size(): number {
    return this._size;
  }

  write(data: Buffer): void {
    const len = data.length;
    if (len === 0) return;

    if (len >= this.capacity) {
      // Only keep the last `capacity` bytes
      data.copy(this.buf, 0, len - this.capacity);
      this.head = 0;
      this._size = this.capacity;
      return;
    }

    const end = this.head + len;
    if (end <= this.capacity) {
      data.copy(this.buf, this.head);
      this.head = end % this.capacity;
    } else {
      // Wraps around
      const firstPart = this.capacity - this.head;
      data.copy(this.buf, this.head, 0, firstPart);
      data.copy(this.buf, 0, firstPart);
      this.head = len - firstPart;
    }
    this._size = Math.min(this._size + len, this.capacity);
  }

  readAll(): Buffer {
    if (this._size === 0) return Buffer.alloc(0);
    if (this._size < this.capacity) {
      return Buffer.from(this.buf.subarray(0, this._size));
    }
    // Full: head points to the oldest byte
    const result = Buffer.allocUnsafe(this.capacity);
    this.buf.copy(result, 0, this.head);
    this.buf.copy(result, this.capacity - this.head, 0, this.head);
    return result;
  }

  clear(): void {
    this.head = 0;
    this._size = 0;
  }
}
```

- [ ] **Step 4: Run tests — expect pass**

```bash
bun run test src/gui/ringBuffer.test.ts
```

Expected: all 7 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/gui/ringBuffer.ts src/gui/ringBuffer.test.ts
git commit -m "feat(gui): add ring buffer for session replay"
```

---

## Chunk 3: Backend — REST API

### Task 6: Add `replaceGlobalConfig` to `globalConfigManager`

The GUI settings panel needs to write an entire config object at once. The existing `globalConfigManager` only exposes field-specific setters. Add one thin method before writing the router.

**Files:**
- Modify: `src/services/config/globalConfigManager.ts`

- [ ] **Step 1: Add `replaceConfig` method and export the class**

In `src/services/config/globalConfigManager.ts`:

1. Check the class declaration — if it reads `class GlobalConfigManager` (not exported), change it to `export class GlobalConfigManager`.

2. Find the type used for the internal config object (look for `private config:` field — it's `ConfigurationData` from `src/types/index.ts`). Import it if not already imported.

3. Add this public method after `reload()`:

```typescript
/** Replace the entire global config with the given value and persist to disk. */
replaceConfig(newConfig: ConfigurationData): void {
  this.config = newConfig;
  this.saveConfig();
}
```

- [ ] **Step 2: Add `replaceConfig` to `projectConfigManager`**

In `src/services/config/projectConfigManager.ts`, add the same method (check its internal config field type — likely also `ConfigurationData` or a subset):

```typescript
replaceConfig(newConfig: Partial<ConfigurationData>): void {
  this.config = newConfig;
  this.saveConfig();
}
```

- [ ] **Step 3: Run typecheck**

```bash
bun run typecheck
```

Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add src/services/config/globalConfigManager.ts src/services/config/projectConfigManager.ts
git commit -m "feat(config): add replaceConfig methods for bulk config replacement"
```

---

### Task 7: REST endpoint handlers

**Files:**
- Create: `src/gui/apiRoutes.ts`
- Create: `src/gui/apiRoutes.test.ts`

The handlers receive a request and return a `Response`. They are pure functions injected with service instances — easy to test without a running server.

**Real method names from existing interfaces (verified):**
- `worktreeService.getWorktreesEffect()` ✓
- `worktreeService.createWorktreeEffect(worktreePath, branch, baseBranch, copySessionData?)` — note: `worktreePath` first, `branch` second
- `worktreeService.deleteWorktreeEffect(worktreePath, { deleteBranch? })`
- `worktreeService.getGitRootPath()` ✓ (verified at `src/types/index.ts:317`)
- `sessionManager.createSessionWithPresetEffect(worktreePath, presetId?)`
- `sessionManager.destroySession(worktreePath)` ✓
- `sessionManager.getSession(worktreePath)` ✓
- `session.stateMutex.getSnapshot()` ✓ (verified at `src/utils/mutex.ts`)
- `configReader.getConfiguration()` — returns merged config
- `globalConfigManager.replaceConfig(newConfig)` — added in Task 6
- `PUT /api/config/project` has no test (direct singleton import — intentional gap, acceptable)

- [ ] **Step 1: Write failing tests**

Create `src/gui/apiRoutes.test.ts`:

```typescript
import { describe, it, expect, vi } from 'vitest';
import { Effect } from 'effect';
import { createApiRouter } from './apiRoutes.js';

const mockWorktreeService = {
  getWorktreesEffect: vi.fn().mockReturnValue(Effect.succeed([])),
  createWorktreeEffect: vi.fn().mockReturnValue(Effect.succeed({})),
  deleteWorktreeEffect: vi.fn().mockReturnValue(Effect.succeed(undefined)),
};
const mockSessionManager = {
  sessions: new Map(),
  createSessionWithPresetEffect: vi.fn().mockReturnValue(Effect.succeed({})),
  destroySession: vi.fn(),
  getSession: vi.fn().mockReturnValue(undefined),
};
const mockConfigReader = {
  getConfiguration: vi.fn().mockReturnValue({ shortcuts: {}, statusHooks: [] }),
};
const mockGlobalConfigManager = {
  replaceConfig: vi.fn(),
};

function makeRouter() {
  return createApiRouter({
    worktreeService: mockWorktreeService as any,
    sessionManager: mockSessionManager as any,
    configReader: mockConfigReader as any,
    globalConfigManager: mockGlobalConfigManager as any,
  });
}

describe('GET /api/health', () => {
  it('returns 200 without auth', async () => {
    const router = makeRouter();
    const res = await router.handle(new Request('http://localhost/api/health'), 'valid-token');
    expect(res.status).toBe(200);
  });
});

describe('GET /api/config', () => {
  it('returns 401 without token', async () => {
    const router = makeRouter();
    const res = await router.handle(new Request('http://localhost/api/config'), 'valid-token');
    expect(res.status).toBe(401);
  });

  it('returns config with valid token', async () => {
    const router = makeRouter();
    const req = new Request('http://localhost/api/config', {
      headers: { Authorization: 'Bearer valid-token' },
    });
    const res = await router.handle(req, 'valid-token');
    expect(res.status).toBe(200);
    const body = await res.json();
    expect(body).toHaveProperty('shortcuts');
  });
});

describe('PUT /api/config/global', () => {
  it('calls replaceConfig with request body', async () => {
    const router = makeRouter();
    const newConfig = { shortcuts: { returnToMenu: { ctrl: true, key: 'r' } } };
    const req = new Request('http://localhost/api/config/global', {
      method: 'PUT',
      headers: { Authorization: 'Bearer valid-token', 'Content-Type': 'application/json' },
      body: JSON.stringify(newConfig),
    });
    const res = await router.handle(req, 'valid-token');
    expect(res.status).toBe(200);
    expect(mockGlobalConfigManager.replaceConfig).toHaveBeenCalledWith(newConfig);
  });
});

describe('GET /api/sessions/:id/state', () => {
  it('returns 404 for unknown session', async () => {
    const router = makeRouter();
    const req = new Request('http://localhost/api/sessions/unknown/state', {
      headers: { Authorization: 'Bearer valid-token' },
    });
    const res = await router.handle(req, 'valid-token');
    expect(res.status).toBe(404);
  });
});

describe('GET /api/worktrees', () => {
  it('returns worktrees array', async () => {
    const router = makeRouter();
    const req = new Request('http://localhost/api/worktrees', {
      headers: { Authorization: 'Bearer valid-token' },
    });
    const res = await router.handle(req, 'valid-token');
    expect(res.status).toBe(200);
    expect(await res.json()).toEqual([]);
  });
});
```

- [ ] **Step 2: Run tests — expect failure**

```bash
bun run test src/gui/apiRoutes.test.ts
```

Expected: FAIL — `apiRoutes.js` not found.

- [ ] **Step 3: Implement `src/gui/apiRoutes.ts`**

```typescript
import { Effect } from 'effect';
import type { SessionManager } from '../services/sessionManager.js';
import type { IWorktreeService } from '../types/index.js';
import type { ConfigReader } from '../services/config/configReader.js';
import type { globalConfigManager as _gcm } from '../services/config/globalConfigManager.js';
type GlobalConfigManager = typeof _gcm;
import { validateToken } from './auth.js';

interface RouterDeps {
  worktreeService: IWorktreeService;
  sessionManager: SessionManager;
  configReader: ConfigReader;
  globalConfigManager: GlobalConfigManager;
  projectManager?: { getProjects(): Promise<Array<{ path: string; name: string }>> };
}

export interface ApiRouter {
  handle(req: Request, startupToken: string): Promise<Response>;
}

function json(data: unknown, status = 200): Response {
  return new Response(JSON.stringify(data), {
    status,
    headers: { 'Content-Type': 'application/json' },
  });
}

function unauthorized(): Response { return json({ error: 'Unauthorized' }, 401); }
function notFound(msg = 'Not found'): Response { return json({ error: msg }, 404); }

function requireAuth(req: Request, startupToken: string): boolean {
  const auth = req.headers.get('Authorization');
  const token = auth?.startsWith('Bearer ') ? auth.slice(7) : null;
  return token !== null && validateToken(startupToken, token);
}

async function runEffect<A>(eff: Effect.Effect<A, any, never>): Promise<{ ok: true; value: A } | { ok: false; error: string }> {
  const result = await Effect.runPromise(Effect.either(eff));
  if (result._tag === 'Right') return { ok: true, value: result.right };
  return { ok: false, error: String(result.left) };
}

export function createApiRouter(deps: RouterDeps): ApiRouter {
  const { worktreeService, sessionManager, configReader, globalConfigManager } = deps;

  async function handle(req: Request, startupToken: string): Promise<Response> {
    const url = new URL(req.url);
    const path = url.pathname;
    const method = req.method;

    if (path === '/api/health' && method === 'GET') {
      return json({ status: 'ok' });
    }

    if (!requireAuth(req, startupToken)) return unauthorized();

    // GET /api/worktrees
    if (path === '/api/worktrees' && method === 'GET') {
      const r = await runEffect(worktreeService.getWorktreesEffect());
      return r.ok ? json(r.value) : json({ error: r.error }, 500);
    }

    // POST /api/worktrees
    if (path === '/api/worktrees' && method === 'POST') {
      const body = await req.json() as { branch: string; path: string; baseBranch?: string; copySessionData?: boolean };
      const r = await runEffect(
        worktreeService.createWorktreeEffect(body.path, body.branch, body.baseBranch ?? 'HEAD', body.copySessionData),
      );
      return r.ok ? json({ ok: true }, 201) : json({ error: r.error }, 500);
    }

    // DELETE /api/worktrees/:id
    const worktreeDeleteMatch = path.match(/^\/api\/worktrees\/(.+)$/);
    if (worktreeDeleteMatch && method === 'DELETE') {
      const id = decodeURIComponent(worktreeDeleteMatch[1]!);
      const body = await req.json().catch(() => ({})) as { deleteBranch?: boolean };
      const r = await runEffect(worktreeService.deleteWorktreeEffect(id, { deleteBranch: body.deleteBranch }));
      return r.ok ? json({ ok: true }) : json({ error: r.error }, 500);
    }

    // POST /api/sessions
    if (path === '/api/sessions' && method === 'POST') {
      const body = await req.json() as { worktreeId: string; preset?: string };
      const r = await runEffect(sessionManager.createSessionWithPresetEffect(body.worktreeId, body.preset));
      return r.ok ? json({ ok: true }, 201) : json({ error: r.error }, 500);
    }

    // DELETE /api/sessions/:id  (must check before /state match)
    const sessionDeleteMatch = path.match(/^\/api\/sessions\/([^/]+)$/);
    if (sessionDeleteMatch && method === 'DELETE') {
      const id = decodeURIComponent(sessionDeleteMatch[1]!);
      sessionManager.destroySession(id);
      return json({ ok: true });
    }

    // GET /api/sessions/:id/state
    const sessionStateMatch = path.match(/^\/api\/sessions\/(.+)\/state$/);
    if (sessionStateMatch && method === 'GET') {
      const id = decodeURIComponent(sessionStateMatch[1]!);
      const session = sessionManager.getSession(id);
      if (!session) return notFound('Session not found');
      const stateData = session.stateMutex.getSnapshot();
      return json({ state: stateData.state });
    }

    // GET /api/config
    if (path === '/api/config' && method === 'GET') {
      return json(configReader.getConfiguration());
    }

    // PUT /api/config/global
    if (path === '/api/config/global' && method === 'PUT') {
      try {
        const body = await req.json();
        globalConfigManager.replaceConfig(body);
        return json({ ok: true });
      } catch (err) {
        return json({ error: String(err) }, 500);
      }
    }

    // PUT /api/config/project — writes .ccmanager.json in git root
    // NOTE: projectConfigManager singleton is imported directly (not injected) since
    // there is no existing interface for project config writes. This is the minimal touch.
    if (path === '/api/config/project' && method === 'PUT') {
      try {
        const body = await req.json();
        const { projectConfigManager } = await import('../services/config/projectConfigManager.js');
        projectConfigManager.replaceConfig(body);
        return json({ ok: true });
      } catch (err) {
        return json({ error: String(err) }, 500);
      }
    }

    // GET /api/projects — multi-project mode: list available git repos
    // NOTE: projectManager is injected as optional; single-project mode returns [{path, name}] for cwd.
    if (path === '/api/projects' && method === 'GET') {
      try {
        const projects = deps.projectManager
          ? await deps.projectManager.getProjects()
          : [{ path: process.cwd(), name: worktreeService.getGitRootPath().split('/').pop() ?? 'current' }];
        return json(projects);
      } catch (err) {
        return json({ error: String(err) }, 500);
      }
    }

    return notFound();
  }

  return { handle };
}
```

- [ ] **Step 4: Run tests — expect pass**

```bash
bun run test src/gui/apiRoutes.test.ts
```

Expected: all 6 tests pass.

- [ ] **Step 5: Run typecheck**

```bash
bun run typecheck
```

Fix any type errors. Common fixes needed:
- `IWorktreeService` import path may differ — check `src/types/index.ts` for the exported interface name
- `GlobalConfigManager` may need explicit type export from its file

- [ ] **Step 6: Commit**

```bash
git add src/gui/apiRoutes.ts src/gui/apiRoutes.test.ts
git commit -m "feat(gui): add REST API route handlers"
```

---

## Chunk 4: Backend — WebSocket + PTY Streaming

### Task 8: PTY streamer

Subscribes to `SessionManager` events, maintains ring buffers, delivers binary frames to registered WS clients.

**Files:**
- Create: `src/gui/ptyStreamer.ts`
- Create: `src/gui/ptyStreamer.test.ts`

- [ ] **Step 1: Write failing tests**

Create `src/gui/ptyStreamer.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { EventEmitter } from 'events';
import { PtyStreamer } from './ptyStreamer.js';
import { RingBuffer } from './ringBuffer.js';

function makeSession(path: string) {
  return {
    worktreePath: path,
    stateMutex: { getSnapshot: () => ({ state: 'idle' }) },
  };
}

describe('PtyStreamer', () => {
  let emitter: EventEmitter;
  let streamer: PtyStreamer;

  beforeEach(() => {
    emitter = new EventEmitter();
    streamer = new PtyStreamer(emitter as any, { replayBytes: 1024 });
  });

  it('assigns a session index when session is created', () => {
    emitter.emit('sessionCreated', makeSession('/a'));
    expect(streamer.getSessionIndex('/a')).toBe(0);
  });

  it('assigns incrementing indices', () => {
    emitter.emit('sessionCreated', makeSession('/a'));
    emitter.emit('sessionCreated', makeSession('/b'));
    expect(streamer.getSessionIndex('/b')).toBe(1);
  });

  it('writes PTY data to ring buffer', () => {
    emitter.emit('sessionCreated', makeSession('/a'));
    emitter.emit('sessionData', makeSession('/a'), 'hello');
    expect(streamer.getReplayBuffer('/a')!.size).toBeGreaterThan(0);
  });

  it('delivers PTY data to registered client', () => {
    const send = vi.fn();
    emitter.emit('sessionCreated', makeSession('/a'));
    streamer.registerClient({ send } as any);
    emitter.emit('sessionData', makeSession('/a'), 'hi');
    expect(send).toHaveBeenCalledWith(expect.any(Buffer));
  });

  it('removes ring buffer when session is destroyed', () => {
    emitter.emit('sessionCreated', makeSession('/a'));
    emitter.emit('sessionDestroyed', makeSession('/a'));
    expect(streamer.getReplayBuffer('/a')).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run tests — expect failure**

```bash
bun run test src/gui/ptyStreamer.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement `src/gui/ptyStreamer.ts`**

```typescript
import type { EventEmitter } from 'events';
import type { SessionManager } from '../services/sessionManager.js';
import { RingBuffer } from './ringBuffer.js';
import { logger } from '../utils/logger.js';

const REPLAY_BYTES = 100 * 1024;       // 100KB default
const FLOW_PAUSE_THRESHOLD = 1024 * 1024;   // 1MB unACKed → pause
const FLOW_RESUME_THRESHOLD = 512 * 1024;    // 512KB → resume

interface GuiClient {
  send(data: Buffer | string): void;
  close?(): void;
}

interface ClientState {
  client: GuiClient;
  unacked: number;  // bytes sent but not yet ACKed
  paused: Map<string, boolean>;  // sessionPath → paused
}

export class PtyStreamer {
  private sessionIndex = new Map<string, number>();  // path → index
  private indexToPath = new Map<number, string>();   // index → path
  private nextIndex = 0;
  private ringBuffers = new Map<string, RingBuffer>();
  private clients = new Set<ClientState>();
  private readonly replayBytes: number;

  constructor(
    private readonly emitter: Pick<EventEmitter, 'on' | 'off'>,
    options: { replayBytes?: number } = {},
  ) {
    this.replayBytes = options.replayBytes ?? REPLAY_BYTES;
    this.emitter.on('sessionCreated', this.onCreated);
    this.emitter.on('sessionData', this.onData);
    this.emitter.on('sessionStateChanged', this.onStateChanged);
    this.emitter.on('sessionDestroyed', this.onDestroyed);
    this.emitter.on('sessionExit', this.onDestroyed);
  }

  private onCreated = (session: { worktreePath: string }) => {
    const idx = this.nextIndex++;
    this.sessionIndex.set(session.worktreePath, idx);
    this.indexToPath.set(idx, session.worktreePath);
    this.ringBuffers.set(session.worktreePath, new RingBuffer(this.replayBytes));
    this.broadcastJson({ type: 'session:created', sessionId: session.worktreePath, index: idx });
  };

  private onData = (session: { worktreePath: string }, data: string) => {
    const buf = Buffer.from(data, 'utf8');
    this.ringBuffers.get(session.worktreePath)?.write(buf);
    const idx = this.sessionIndex.get(session.worktreePath);
    if (idx === undefined) return;
    const frame = buildBinaryFrame(idx, buf);
    this.broadcastBinary(session.worktreePath, frame);
  };

  private onStateChanged = (session: { worktreePath: string; stateMutex: any }) => {
    const state = session.stateMutex.getSnapshot().state;
    this.broadcastJson({ type: 'session:state', sessionId: session.worktreePath, state });
  };

  private onDestroyed = (session: { worktreePath: string }) => {
    const idx = this.sessionIndex.get(session.worktreePath);
    this.sessionIndex.delete(session.worktreePath);
    if (idx !== undefined) this.indexToPath.delete(idx);
    this.ringBuffers.delete(session.worktreePath);
    this.broadcastJson({ type: 'session:destroyed', sessionId: session.worktreePath });
  };

  private broadcastBinary(sessionPath: string, frame: Buffer) {
    for (const clientState of this.clients) {
      const paused = clientState.paused.get(sessionPath) ?? false;
      if (paused) continue;
      clientState.client.send(frame);
      clientState.unacked += frame.length;
      if (clientState.unacked > FLOW_PAUSE_THRESHOLD) {
        clientState.paused.set(sessionPath, true);
      }
    }
  }

  private broadcastJson(data: object) {
    const msg = JSON.stringify(data);
    for (const clientState of this.clients) {
      clientState.client.send(msg);
    }
  }

  registerClient(client: GuiClient): void {
    this.clients.add({ client, unacked: 0, paused: new Map() });
  }

  unregisterClient(client: GuiClient): void {
    for (const cs of this.clients) {
      if (cs.client === client) {
        this.clients.delete(cs);
        return;
      }
    }
  }

  /** Called when client sends an ACK frame. */
  handleAck(client: GuiClient, sessionId: string, bytesConsumed: number): void {
    for (const cs of this.clients) {
      if (cs.client !== client) continue;
      cs.unacked = Math.max(0, cs.unacked - bytesConsumed);
      if (cs.unacked < FLOW_RESUME_THRESHOLD) {
        cs.paused.set(sessionId, false);
      }
      return;
    }
  }

  /** Returns buffered data for reconnect replay. */
  getReplay(sessionPath: string): Buffer | null {
    return this.ringBuffers.get(sessionPath)?.readAll() ?? null;
  }

  getPathByIndex(index: number): string | undefined {
    return this.indexToPath.get(index);
  }

  // Test helpers
  getSessionIndex(path: string): number | undefined {
    return this.sessionIndex.get(path);
  }

  getReplayBuffer(path: string): RingBuffer | undefined {
    return this.ringBuffers.get(path);
  }

  destroy(): void {
    this.emitter.off('sessionCreated', this.onCreated);
    this.emitter.off('sessionData', this.onData);
    this.emitter.off('sessionStateChanged', this.onStateChanged);
    this.emitter.off('sessionDestroyed', this.onDestroyed);
    this.emitter.off('sessionExit', this.onDestroyed);
  }
}

/** Binary frame: 4-byte big-endian session index + raw bytes */
export function buildBinaryFrame(sessionIndex: number, data: Buffer): Buffer {
  const frame = Buffer.allocUnsafe(4 + data.length);
  frame.writeUInt32BE(sessionIndex, 0);
  data.copy(frame, 4);
  return frame;
}

/** Parse binary frame header. Returns { sessionIndex, payload }. */
export function parseBinaryFrame(frame: Buffer): { sessionIndex: number; payload: Buffer } {
  return {
    sessionIndex: frame.readUInt32BE(0),
    payload: frame.subarray(4),
  };
}
```

- [ ] **Step 4: Run tests — expect pass**

```bash
bun run test src/gui/ptyStreamer.test.ts
```

Expected: all 5 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/gui/ptyStreamer.ts src/gui/ptyStreamer.test.ts
git commit -m "feat(gui): add PTY streamer with ring buffer and flow control"
```

---

### Task 9: WebSocket handler

**Files:**
- Create: `src/gui/wsHandler.ts`

No unit tests here — this is thin glue between Bun's WS API, `PtyStreamer`, and `SessionManager`. Tested in end-to-end.

- [ ] **Step 1: Create `src/gui/wsHandler.ts`**

```typescript
import type { ServerWebSocket } from 'bun';
import type { SessionManager } from '../services/sessionManager.js';
import { PtyStreamer, parseBinaryFrame } from './ptyStreamer.js';
import { validateToken } from './auth.js';
import { logger } from '../utils/logger.js';

interface WsData {
  token: string;
}

const CHUNK_SIZE = 4 * 1024;       // 4KB replay chunks
const CHUNK_INTERVAL_MS = 1000 / 60; // ~60fps

export function createWsHandlers(
  streamer: PtyStreamer,
  sessionManager: SessionManager,
  startupToken: string,
) {
  return {
    open(ws: ServerWebSocket<WsData>) {
      if (!validateToken(startupToken, ws.data.token)) {
        ws.close(4001, 'Unauthorized');
        return;
      }
      streamer.registerClient(ws);
      logger.info('GUI WebSocket client connected');
    },

    message(ws: ServerWebSocket<WsData>, message: string | Buffer) {
      if (typeof message === 'string') {
        handleJsonFrame(ws, message, streamer, sessionManager);
      } else {
        handleBinaryFrame(ws, message as Buffer, sessionManager, streamer);
      }
    },

    close(ws: ServerWebSocket<WsData>) {
      streamer.unregisterClient(ws);
      logger.info('GUI WebSocket client disconnected');
    },

    error(ws: ServerWebSocket<WsData>, error: Error) {
      logger.info(`GUI WebSocket error: ${error.message}`);
      streamer.unregisterClient(ws);
    },
  };
}

function handleJsonFrame(
  ws: ServerWebSocket<WsData>,
  raw: string,
  streamer: PtyStreamer,
  sessionManager: SessionManager,
) {
  let msg: Record<string, unknown>;
  try {
    msg = JSON.parse(raw);
  } catch {
    return;
  }

  switch (msg.type) {
    case 'session:resize': {
      const session = sessionManager.getSession(msg.sessionId as string);
      if (session) {
        session.process.resize(msg.cols as number, msg.rows as number);
      }
      break;
    }
    case 'session:signal': {
      const session = sessionManager.getSession(msg.sessionId as string);
      if (session && msg.signal === 'SIGINT') {
        session.process.write('\x03');
      } else if (session && msg.signal === 'SIGTERM') {
        session.process.kill();
      }
      break;
    }
    case 'replay:request': {
      const sessionId = msg.sessionId as string;
      const replay = streamer.getReplay(sessionId);
      if (!replay || replay.length === 0) {
        ws.send(JSON.stringify({ type: 'replay:end', sessionId }));
        return;
      }
      ws.send(JSON.stringify({ type: 'replay:start', sessionId, byteCount: replay.length }));
      sendChunked(ws, sessionId, replay);
      break;
    }
    case 'ack': {
      streamer.handleAck(ws, msg.sessionId as string, msg.bytesConsumed as number);
      break;
    }
    case 'ping': {
      ws.send(JSON.stringify({ type: 'pong' }));
      break;
    }
  }
}

/** Keyboard input: binary frame from browser → PTY stdin */
function handleBinaryFrame(
  ws: ServerWebSocket<WsData>,
  frame: Buffer,
  sessionManager: SessionManager,
  streamer: PtyStreamer,
) {
  if (frame.length < 4) return;
  const { sessionIndex, payload } = parseBinaryFrame(frame);
  const sessionPath = streamer.getPathByIndex(sessionIndex);
  if (!sessionPath) return;
  const session = sessionManager.getSession(sessionPath);
  if (session && payload.length > 0) {
    session.process.write(payload.toString('utf8'));
  }
}

function sendChunked(ws: ServerWebSocket<WsData>, sessionId: string, data: Buffer) {
  let offset = 0;
  function sendNext() {
    if (offset >= data.length) {
      ws.send(JSON.stringify({ type: 'replay:end', sessionId }));
      return;
    }
    const chunk = data.subarray(offset, offset + CHUNK_SIZE);
    ws.send(chunk);
    offset += CHUNK_SIZE;
    setTimeout(sendNext, CHUNK_INTERVAL_MS);
  }
  sendNext();
}
```

- [ ] **Step 2: Run typecheck**

```bash
bun run typecheck
```

Fix any type errors.

- [ ] **Step 3: Commit**

```bash
git add src/gui/wsHandler.ts
git commit -m "feat(gui): add WebSocket frame handler"
```

---

## Chunk 5: Backend — Server + Session Lifecycle

### Task 10: Bun HTTP server + session lifecycle

**Files:**
- Create: `src/gui/server.ts`

- [ ] **Step 1: Create `src/gui/server.ts`**

```typescript
import { existsSync } from 'fs';
import { join } from 'path';
import { fileURLToPath } from 'url';
import { exec } from 'child_process';
import { generateStartupToken, validateToken } from './auth.js';
import { PtyStreamer } from './ptyStreamer.js';
import { createApiRouter } from './apiRoutes.js';
import { createWsHandlers } from './wsHandler.js';
import type { SessionManager } from '../services/sessionManager.js';
import type { WorktreeService } from '../services/worktreeService.js';
import type { ConfigReader } from '../services/config/configReader.js';
import { logger } from '../utils/logger.js';

const __dirname = fileURLToPath(new URL('.', import.meta.url));
const STATIC_DIR = join(__dirname, '../../dist/gui');
const DEFAULT_PORT = 7288;
const KEEPALIVE_GRACE_MS = 30_000;

export interface GuiServerOptions {
  port?: number;
  killOnClose?: boolean;
  noOpen?: boolean;
}

export async function startGuiServer(
  sessionManager: SessionManager,
  worktreeService: WorktreeService,
  configReader: ConfigReader,
  options: GuiServerOptions = {},
) {
  const port = options.port ?? DEFAULT_PORT;
  const startupToken = generateStartupToken();
  const streamer = new PtyStreamer(sessionManager);
  const apiRouter = createApiRouter({ worktreeService, sessionManager, configReader });
  const wsHandlers = createWsHandlers(streamer, sessionManager, startupToken);

  let disconnectTimer: ReturnType<typeof setTimeout> | null = null;

  const server = Bun.serve<{ token: string }>({
    port,
    hostname: '127.0.0.1',

    fetch(req, server) {
      const url = new URL(req.url);

      // WebSocket upgrade
      if (req.headers.get('Upgrade') === 'websocket') {
        const token = url.searchParams.get('token') ?? '';
        if (!validateToken(startupToken, token)) {
          return new Response('Unauthorized', { status: 401 });
        }
        const upgraded = server.upgrade(req, { data: { token } });
        return upgraded ? undefined : new Response('Upgrade failed', { status: 500 });
      }

      // REST API
      if (url.pathname.startsWith('/api/')) {
        return apiRouter.handle(req, startupToken);
      }

      // Static SPA files
      if (existsSync(STATIC_DIR)) {
        const filePath = url.pathname === '/' ? '/index.html' : url.pathname;
        const fullPath = join(STATIC_DIR, filePath);
        if (existsSync(fullPath)) {
          return new Response(Bun.file(fullPath));
        }
        // SPA fallback: serve index.html for unknown paths
        return new Response(Bun.file(join(STATIC_DIR, 'index.html')));
      }

      return new Response('GUI not built. Run: bun run build:gui', { status: 503 });
    },

    websocket: {
      ...wsHandlers,
      open(ws) {
        wsHandlers.open(ws);
        // Cancel pending disconnect timer on reconnect
        if (disconnectTimer) {
          clearTimeout(disconnectTimer);
          disconnectTimer = null;
        }
      },
      close(ws, code, reason) {
        wsHandlers.close(ws);
        if (options.killOnClose) {
          logger.info('GUI: --kill-on-close set, destroying all sessions');
          sessionManager.sessions.forEach((_, path) => sessionManager.destroySession(path));
          return;
        }
        // Grace period: keep sessions alive for 30s waiting for reconnect
        disconnectTimer = setTimeout(() => {
          disconnectTimer = null;
          logger.info('GUI: keepalive grace period expired, sessions remain running');
        }, KEEPALIVE_GRACE_MS);
      },
    },
  });

  const url = `http://127.0.0.1:${port}?token=${startupToken}`;
  logger.info(`CCManager GUI running at ${url}`);

  if (!options.noOpen) {
    openBrowser(url);
  } else {
    console.log(`Open: ${url}`);
  }

  return { server, streamer, url };
}

function openBrowser(url: string) {
  const cmd =
    process.platform === 'darwin' ? `open "${url}"` :
    process.platform === 'win32' ? `start "${url}"` :
    `xdg-open "${url}"`;
  exec(cmd, err => {
    if (err) logger.info(`Could not open browser automatically. Open: ${url}`);
  });
}
```

- [ ] **Step 2: Run typecheck**

```bash
bun run typecheck
```

Fix any type errors before proceeding.

- [ ] **Step 3: Smoke test the server manually**

```bash
# Build first so static dir exists (even if empty)
mkdir -p dist/gui && echo '{}' > dist/gui/index.html
bun run --no-env-file dist/cli.js --gui
```

Expected: server starts, browser opens (or URL printed with `--no-open`), `/api/health` returns `{"status":"ok"}`.

- [ ] **Step 4: Commit**

```bash
git add src/gui/server.ts
git commit -m "feat(gui): add Bun HTTP server with session lifecycle"
```

---

### Task 11: Wire `--gui` flag into CLI

**Files:**
- Modify: `src/cli.tsx`

- [ ] **Step 1: Read current `src/cli.tsx`**

Check how flags are currently parsed (uses `meow`).

- [ ] **Step 2: Add `--gui` flag**

In `src/cli.tsx`, add to the `meow` flags definition:

```typescript
gui: { type: 'boolean', default: false },
noOpen: { type: 'boolean', default: false },
killOnClose: { type: 'boolean', default: false },
guiPort: { type: 'number', default: 7288 },
```

And add the early-exit branch before the Ink render:

```typescript
if (cli.flags.gui) {
  const { startGuiServer } = await import('./gui/server.js');
  await startGuiServer(
    sessionManager,
    worktreeService,
    configReader,
    {
      port: cli.flags.guiPort,
      noOpen: cli.flags.noOpen,
      killOnClose: cli.flags.killOnClose,
    },
  );
  // Keep process alive — server runs until SIGINT
  await new Promise(() => {});
  process.exit(0);
}
```

- [ ] **Step 3: Build and test**

```bash
bun run build
bun run --no-env-file dist/cli.js --gui --no-open
# Expect: "CCManager GUI running at http://127.0.0.1:7288?token=..."
curl http://127.0.0.1:7288/api/health
# Expect: {"status":"ok"}
```

- [ ] **Step 4: Add `build:gui` and `build:all` scripts to `package.json`**

```json
"build:gui": "vite build",
"build:all": "bun run build && bun run build:gui"
```

- [ ] **Step 5: Commit**

```bash
git add src/cli.tsx package.json
git commit -m "feat(gui): add --gui flag to CLI entry point"
```

---

## Chunk 6: Frontend — WS Client + App Shell

### Task 12: WebSocket client + frame demux

**Files:**
- Create: `gui-frontend/ws/wsClient.ts`
- Create: `gui-frontend/ws/frameDemux.ts`

- [ ] **Step 1: Create `gui-frontend/ws/frameDemux.ts`**

```typescript
/** Parses incoming WebSocket frames and dispatches to typed handlers. */

export type JsonFrame =
  | { type: 'session:created'; sessionId: string; index: number }
  | { type: 'session:destroyed'; sessionId: string }
  | { type: 'session:state'; sessionId: string; state: string }
  | { type: 'replay:start'; sessionId: string; byteCount: number }
  | { type: 'replay:end'; sessionId: string }
  | { type: 'pong' };

export interface BinaryFrame {
  sessionIndex: number;
  payload: Uint8Array;
}

export interface FrameHandlers {
  onJson: (frame: JsonFrame) => void;
  onBinary: (frame: BinaryFrame) => void;
}

export function parseFrame(
  data: string | ArrayBuffer,
  handlers: FrameHandlers,
): void {
  if (typeof data === 'string') {
    try {
      handlers.onJson(JSON.parse(data) as JsonFrame);
    } catch {
      console.warn('Invalid JSON WS frame', data);
    }
    return;
  }
  const buf = new Uint8Array(data);
  if (buf.length < 4) return;
  const view = new DataView(buf.buffer);
  const sessionIndex = view.getUint32(0, false); // big-endian
  handlers.onBinary({ sessionIndex, payload: buf.subarray(4) });
}

/** Build a binary frame for keyboard input: 4-byte session index + UTF-8 bytes */
export function buildInputFrame(sessionIndex: number, input: string): ArrayBuffer {
  const encoded = new TextEncoder().encode(input);
  const buf = new ArrayBuffer(4 + encoded.length);
  new DataView(buf).setUint32(0, sessionIndex, false);
  new Uint8Array(buf).set(encoded, 4);
  return buf;
}
```

- [ ] **Step 2: Create `gui-frontend/ws/wsClient.ts`**

```typescript
import { parseFrame, buildInputFrame, type FrameHandlers, type JsonFrame } from './frameDemux.js';

const PING_INTERVAL_MS = 10_000;
const RECONNECT_DELAY_MS = 2_000;

type Listener<T> = (data: T) => void;

export class WsClient {
  private ws: WebSocket | null = null;
  private pingTimer: ReturnType<typeof setInterval> | null = null;
  private jsonListeners = new Set<Listener<JsonFrame>>();
  private binaryListeners = new Set<Listener<{ sessionIndex: number; payload: Uint8Array }>>();
  private closed = false;

  constructor(private readonly url: string) {}

  connect(): void {
    if (this.closed) return;
    this.ws = new WebSocket(this.url);
    this.ws.binaryType = 'arraybuffer';

    this.ws.onopen = () => {
      this.startPing();
      this.requestReplayForAllSessions();
    };

    this.ws.onmessage = (e) => {
      parseFrame(e.data, {
        onJson: (frame) => this.jsonListeners.forEach(l => l(frame)),
        onBinary: (frame) => this.binaryListeners.forEach(l => l(frame)),
      });
    };

    this.ws.onclose = () => {
      this.stopPing();
      if (!this.closed) {
        setTimeout(() => this.connect(), RECONNECT_DELAY_MS);
      }
    };

    this.ws.onerror = () => {
      // onclose will fire after onerror — reconnect handled there
    };
  }

  disconnect(): void {
    this.closed = true;
    this.stopPing();
    this.ws?.close();
  }

  send(data: string | ArrayBuffer): void {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(data);
    }
  }

  sendInput(sessionIndex: number, input: string): void {
    this.send(buildInputFrame(sessionIndex, input));
  }

  sendJson(data: object): void {
    this.send(JSON.stringify(data));
  }

  requestReplay(sessionId: string): void {
    this.sendJson({ type: 'replay:request', sessionId });
  }

  onJson(listener: Listener<JsonFrame>): () => void {
    this.jsonListeners.add(listener);
    return () => this.jsonListeners.delete(listener);
  }

  onBinary(listener: Listener<{ sessionIndex: number; payload: Uint8Array }>): () => void {
    this.binaryListeners.add(listener);
    return () => this.binaryListeners.delete(listener);
  }

  get connected(): boolean {
    return this.ws?.readyState === WebSocket.OPEN;
  }

  private startPing(): void {
    this.pingTimer = setInterval(() => {
      this.sendJson({ type: 'ping' });
    }, PING_INTERVAL_MS);
  }

  private stopPing(): void {
    if (this.pingTimer) {
      clearInterval(this.pingTimer);
      this.pingTimer = null;
    }
  }

  private requestReplayForAllSessions(): void {
    // Sessions will be re-listed via REST; replay requested per session after listing
  }
}

/** Singleton — created once in main.tsx with the startup token from the URL */
let _client: WsClient | null = null;

export function initWsClient(token: string, port = 7288): WsClient {
  const url = `ws://127.0.0.1:${port}/ws?token=${token}`;
  _client = new WsClient(url);
  _client.connect();
  return _client;
}

export function getWsClient(): WsClient {
  if (!_client) throw new Error('WsClient not initialized');
  return _client;
}
```

- [ ] **Step 3: Typecheck frontend (Vite)**

```bash
bun run vite build 2>&1 | head -30
```

Expected: no type errors in `ws/` files (build may fail on missing components — that's fine at this stage).

- [ ] **Step 4: Commit**

```bash
git add gui-frontend/ws/
git commit -m "feat(gui): add WebSocket client and frame demux"
```

---

### Task 13: React app shell + API hooks

**Files:**
- Create: `gui-frontend/main.tsx`
- Create: `gui-frontend/App.tsx`
- Create: `gui-frontend/hooks/useApi.ts`
- Create: `gui-frontend/hooks/useSessions.ts`

- [ ] **Step 1: Create `gui-frontend/main.tsx`**

```tsx
import React from 'react';
import { createRoot } from 'react-dom/client';
import { App } from './App.js';
import { initWsClient } from './ws/wsClient.js';

const params = new URLSearchParams(window.location.search);
const token = params.get('token') ?? '';
const wsClient = initWsClient(token);

// Expose token for API calls
(window as any).__CCM_TOKEN = token;

createRoot(document.getElementById('root')!).render(<App wsClient={wsClient} />);
```

- [ ] **Step 2: Create `gui-frontend/hooks/useApi.ts`**

```typescript
function getToken(): string {
  return (window as any).__CCM_TOKEN ?? '';
}

function authHeaders(): HeadersInit {
  return { Authorization: `Bearer ${getToken()}`, 'Content-Type': 'application/json' };
}

export async function apiFetch<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${BASE}${path}`, {
    ...init,
    headers: { ...authHeaders(), ...(init?.headers ?? {}) },
  });
  if (!res.ok) {
    const err = await res.json().catch(() => ({ error: res.statusText }));
    throw new Error((err as any).error ?? res.statusText);
  }
  return res.json() as Promise<T>;
}

// Use the same port the SPA was served from (supports --gui-port)
const BASE = `http://127.0.0.1:${window.location.port}`;

export const api = {
  getWorktrees: () => apiFetch<any[]>('/api/worktrees'),
  createWorktree: (body: object) => apiFetch('/api/worktrees', { method: 'POST', body: JSON.stringify(body) }),
  deleteWorktree: (id: string, body?: object) =>
    apiFetch(`/api/worktrees/${encodeURIComponent(id)}`, { method: 'DELETE', body: JSON.stringify(body ?? {}) }),
  getSessions: () => apiFetch<Record<string, string>>('/api/sessions'),
  createSession: (body: object) => apiFetch('/api/sessions', { method: 'POST', body: JSON.stringify(body) }),
  deleteSession: (id: string) => apiFetch(`/api/sessions/${encodeURIComponent(id)}`, { method: 'DELETE' }),
  getConfig: () => apiFetch<any>('/api/config'),
  putGlobalConfig: (body: object) => apiFetch('/api/config/global', { method: 'PUT', body: JSON.stringify(body) }),
  putProjectConfig: (body: object) => apiFetch('/api/config/project', { method: 'PUT', body: JSON.stringify(body) }),
};
```

- [ ] **Step 3: Create `gui-frontend/hooks/useSessions.ts`**

```typescript
import { useState, useEffect } from 'react';
import type { WsClient } from '../ws/wsClient.js';
import { api } from './useApi.js';

export interface SessionInfo {
  worktreePath: string;
  index: number;
  state: string;
  branch?: string;
}

export function useSessions(wsClient: WsClient) {
  const [sessions, setSessions] = useState<Map<string, SessionInfo>>(new Map());
  const [indexMap, setIndexMap] = useState<Map<number, string>>(new Map()); // index → path

  // Load initial worktrees
  useEffect(() => {
    api.getWorktrees().then(worktrees => {
      // Worktrees may or may not have active sessions; state arrives via WS
    }).catch(console.error);
  }, []);

  // Subscribe to WS session events
  useEffect(() => {
    const unsubJson = wsClient.onJson(frame => {
      setSessions(prev => {
        const next = new Map(prev);
        switch (frame.type) {
          case 'session:created':
            next.set(frame.sessionId, {
              worktreePath: frame.sessionId,
              index: frame.index,
              state: 'busy',
            });
            setIndexMap(m => new Map(m).set(frame.index, frame.sessionId));
            break;
          case 'session:destroyed':
            next.delete(frame.sessionId);
            break;
          case 'session:state':
            const existing = next.get(frame.sessionId);
            if (existing) next.set(frame.sessionId, { ...existing, state: frame.state });
            break;
        }
        return next;
      });
    });
    return unsubJson;
  }, [wsClient]);

  return { sessions, indexMap };
}
```

- [ ] **Step 4: Create `gui-frontend/App.tsx`**

```tsx
import React, { useState } from 'react';
import { SessionSidebar } from './components/SessionSidebar.js';
import { TerminalPanel } from './components/TerminalPanel.js';
import { ConnectionStatus } from './components/ConnectionStatus.js';
import { useSessions } from './hooks/useSessions.js';
import type { WsClient } from './ws/wsClient.js';

interface AppProps {
  wsClient: WsClient;
}

export function App({ wsClient }: AppProps) {
  const { sessions, indexMap } = useSessions(wsClient);
  const [activeSessionId, setActiveSessionId] = useState<string | null>(null);

  const sessionList = Array.from(sessions.values());
  const activeSession = activeSessionId ? sessions.get(activeSessionId) ?? null : null;

  return (
    <div style={{ display: 'flex', height: '100vh', fontFamily: 'monospace', background: '#1e1e1e', color: '#ccc' }}>
      <ConnectionStatus wsClient={wsClient} />
      <SessionSidebar
        sessions={sessionList}
        activeSessionId={activeSessionId}
        onSelect={setActiveSessionId}
        wsClient={wsClient}
      />
      <div style={{ flex: 1, overflow: 'hidden' }}>
        {activeSession ? (
          <TerminalPanel
            key={activeSession.worktreePath}
            sessionId={activeSession.worktreePath}
            sessionIndex={activeSession.index}
            wsClient={wsClient}
          />
        ) : (
          <div style={{ padding: 24, color: '#666' }}>Select a session from the sidebar</div>
        )}
      </div>
    </div>
  );
}
```

- [ ] **Step 5: Commit**

```bash
git add gui-frontend/main.tsx gui-frontend/App.tsx gui-frontend/hooks/
git commit -m "feat(gui): add React app shell, API hooks, session state hook"
```

---

## Chunk 7: Frontend — Terminal Panel + Connection Status

### Task 14: xterm.js terminal panel

**Files:**
- Create: `gui-frontend/components/TerminalPanel.tsx`
- Create: `gui-frontend/components/ConnectionStatus.tsx`

- [ ] **Step 1: Create `gui-frontend/components/TerminalPanel.tsx`**

```tsx
import React, { useEffect, useRef } from 'react';
import { Terminal } from '@xterm/xterm';
import '@xterm/xterm/css/xterm.css';
import type { WsClient } from '../ws/wsClient.js';

interface TerminalPanelProps {
  sessionId: string;
  sessionIndex: number;
  wsClient: WsClient;
}

export function TerminalPanel({ sessionId, sessionIndex, wsClient }: TerminalPanelProps) {
  const containerRef = useRef<HTMLDivElement>(null);
  const termRef = useRef<Terminal | null>(null);

  useEffect(() => {
    const term = new Terminal({
      cursorBlink: true,
      theme: { background: '#1e1e1e', foreground: '#ccc' },
      fontSize: 14,
      fontFamily: 'Menlo, Monaco, "Courier New", monospace',
    });
    termRef.current = term;

    if (containerRef.current) {
      term.open(containerRef.current);
    }

    // Forward keyboard input to PTY
    term.onKey(({ key }) => {
      wsClient.sendInput(sessionIndex, key);
    });

    // Forward resize to server
    term.onResize(({ cols, rows }) => {
      wsClient.sendJson({ type: 'session:resize', sessionId, cols, rows });
    });

    // Request replay of buffered output
    wsClient.requestReplay(sessionId);

    // Subscribe to binary PTY output
    const unsubBinary = wsClient.onBinary(({ sessionIndex: idx, payload }) => {
      if (idx !== sessionIndex) return;
      term.write(payload);
      // ACK bytes consumed for flow control
      wsClient.sendJson({ type: 'ack', sessionId, bytesConsumed: payload.length });
    });

    return () => {
      unsubBinary();
      term.dispose();
    };
  }, [sessionId, sessionIndex, wsClient]);

  return (
    <div
      ref={containerRef}
      style={{ width: '100%', height: '100%', padding: 4 }}
    />
  );
}
```

- [ ] **Step 2: Create `gui-frontend/components/ConnectionStatus.tsx`**

```tsx
import React, { useState, useEffect } from 'react';
import type { WsClient } from '../ws/wsClient.js';

export function ConnectionStatus({ wsClient }: { wsClient: WsClient }) {
  const [connected, setConnected] = useState(wsClient.connected);

  useEffect(() => {
    const unsub = wsClient.onJson(frame => {
      if (frame.type === 'pong') setConnected(true);
    });
    // Poll connection state
    const timer = setInterval(() => setConnected(wsClient.connected), 1000);
    return () => { unsub(); clearInterval(timer); };
  }, [wsClient]);

  if (connected) return null;

  return (
    <div style={{
      position: 'fixed', top: 0, left: 0, right: 0, zIndex: 100,
      background: '#c0392b', color: 'white', padding: '4px 12px', fontSize: 13, textAlign: 'center',
    }}>
      Reconnecting to CCManager server…
    </div>
  );
}
```

- [ ] **Step 3: Attempt a full Vite build**

```bash
bun run build:gui
```

Fix any missing import/type errors until build succeeds.

- [ ] **Step 4: Commit**

```bash
git add gui-frontend/components/TerminalPanel.tsx gui-frontend/components/ConnectionStatus.tsx
git commit -m "feat(gui): add xterm.js terminal panel and connection status banner"
```

---

## Chunk 8: Frontend — Session Sidebar + Worktree Dialogs

### Task 15: Session sidebar

**Files:**
- Create: `gui-frontend/components/SessionSidebar.tsx`

- [ ] **Step 1: Create `gui-frontend/components/SessionSidebar.tsx`**

```tsx
import React, { useState } from 'react';
import type { SessionInfo } from '../hooks/useSessions.js';
import type { WsClient } from '../ws/wsClient.js';
import { WorktreeDialog } from './WorktreeDialog.js';
import { api } from '../hooks/useApi.js';

const STATE_COLORS: Record<string, string> = {
  idle: '#27ae60',
  busy: '#f39c12',
  waiting_input: '#e74c3c',
  pending_auto_approval: '#8e44ad',
};

const STATE_LABELS: Record<string, string> = {
  idle: '⬤ idle',
  busy: '⬤ busy',
  waiting_input: '⬤ waiting',
  pending_auto_approval: '⬤ approving',
};

interface SessionSidebarProps {
  sessions: SessionInfo[];
  activeSessionId: string | null;
  onSelect: (id: string) => void;
  wsClient: WsClient;
}

export function SessionSidebar({ sessions, activeSessionId, onSelect, wsClient }: SessionSidebarProps) {
  const [showNewDialog, setShowNewDialog] = useState(false);

  return (
    <div style={{ width: 240, minWidth: 200, background: '#252526', display: 'flex', flexDirection: 'column', borderRight: '1px solid #333' }}>
      <div style={{ padding: '12px 12px 8px', fontSize: 13, color: '#888', fontWeight: 600, textTransform: 'uppercase', letterSpacing: 1 }}>
        Sessions
      </div>

      <div style={{ flex: 1, overflowY: 'auto' }}>
        {sessions.map(s => (
          <div
            key={s.worktreePath}
            onClick={() => onSelect(s.worktreePath)}
            style={{
              padding: '8px 12px',
              cursor: 'pointer',
              background: s.worktreePath === activeSessionId ? '#37373d' : 'transparent',
              borderLeft: s.worktreePath === activeSessionId ? '2px solid #007acc' : '2px solid transparent',
              userSelect: 'none',
            }}
          >
            <div style={{ fontSize: 13, color: '#ccc', overflow: 'hidden', textOverflow: 'ellipsis', whiteSpace: 'nowrap' }}>
              {s.worktreePath.split('/').pop()}
            </div>
            <div style={{ fontSize: 11, color: STATE_COLORS[s.state] ?? '#888', marginTop: 2 }}>
              {STATE_LABELS[s.state] ?? s.state}
            </div>
          </div>
        ))}
      </div>

      <div style={{ padding: 8, borderTop: '1px solid #333' }}>
        <button
          onClick={() => setShowNewDialog(true)}
          style={{ width: '100%', padding: '6px 0', background: '#007acc', color: 'white', border: 'none', borderRadius: 3, cursor: 'pointer', fontSize: 13 }}
        >
          + New Worktree
        </button>
      </div>

      {showNewDialog && (
        <WorktreeDialog
          mode="create"
          onClose={() => setShowNewDialog(false)}
          onConfirm={async (branch, path, copySessionData) => {
            await api.createWorktree({ branch, path, copySessionData });
            setShowNewDialog(false);
          }}
        />
      )}
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add gui-frontend/components/SessionSidebar.tsx
git commit -m "feat(gui): add session sidebar with state badges"
```

---

### Task 16: Worktree dialogs

**Files:**
- Create: `gui-frontend/components/WorktreeDialog.tsx`

- [ ] **Step 1: Create `gui-frontend/components/WorktreeDialog.tsx`**

```tsx
import React, { useState } from 'react';

interface CreateProps {
  mode: 'create';
  onClose: () => void;
  onConfirm: (branch: string, path: string, copySessionData: boolean) => Promise<void>;
}

interface DeleteProps {
  mode: 'delete';
  worktreePath: string;
  onClose: () => void;
  onConfirm: (deleteBranch: boolean) => Promise<void>;
}

type WorktreeDialogProps = CreateProps | DeleteProps;

export function WorktreeDialog(props: WorktreeDialogProps) {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  if (props.mode === 'create') {
    return <CreateDialog {...props} loading={loading} setLoading={setLoading} error={error} setError={setError} />;
  }
  return <DeleteDialog {...props} loading={loading} setLoading={setLoading} error={error} setError={setError} />;
}

function CreateDialog({ onClose, onConfirm, loading, setLoading, error, setError }: CreateProps & any) {
  const [branch, setBranch] = useState('');
  const [path, setPath] = useState('');
  const [copy, setCopy] = useState(false);

  async function submit(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true); setError(null);
    try { await onConfirm(branch, path, copy); }
    catch (err) { setError(String(err)); }
    finally { setLoading(false); }
  }

  return (
    <Overlay onClose={onClose}>
      <h3 style={{ margin: '0 0 16px' }}>New Worktree</h3>
      <form onSubmit={submit}>
        <label style={labelStyle}>Branch name</label>
        <input value={branch} onChange={e => setBranch(e.target.value)} required style={inputStyle} />
        <label style={labelStyle}>Directory path</label>
        <input value={path} onChange={e => setPath(e.target.value)} required style={inputStyle} />
        <label style={{ ...labelStyle, display: 'flex', alignItems: 'center', gap: 8, cursor: 'pointer' }}>
          <input type="checkbox" checked={copy} onChange={e => setCopy(e.target.checked)} />
          Copy session data from current worktree
        </label>
        {error && <div style={{ color: '#e74c3c', marginTop: 8, fontSize: 13 }}>{error}</div>}
        <div style={{ display: 'flex', gap: 8, marginTop: 16, justifyContent: 'flex-end' }}>
          <button type="button" onClick={onClose} style={cancelBtnStyle}>Cancel</button>
          <button type="submit" disabled={loading} style={confirmBtnStyle}>{loading ? 'Creating…' : 'Create'}</button>
        </div>
      </form>
    </Overlay>
  );
}

function DeleteDialog({ worktreePath, onClose, onConfirm, loading, setLoading, error, setError }: DeleteProps & any) {
  const [deleteBranch, setDeleteBranch] = useState(false);

  async function submit() {
    setLoading(true); setError(null);
    try { await onConfirm(deleteBranch); }
    catch (err) { setError(String(err)); }
    finally { setLoading(false); }
  }

  return (
    <Overlay onClose={onClose}>
      <h3 style={{ margin: '0 0 12px' }}>Delete Worktree</h3>
      <p style={{ color: '#ccc', margin: '0 0 12px', fontSize: 14 }}>Delete <strong>{worktreePath}</strong>?</p>
      <label style={{ ...labelStyle, display: 'flex', alignItems: 'center', gap: 8, cursor: 'pointer' }}>
        <input type="checkbox" checked={deleteBranch} onChange={e => setDeleteBranch(e.target.checked)} />
        Also delete the branch
      </label>
      {error && <div style={{ color: '#e74c3c', marginTop: 8, fontSize: 13 }}>{error}</div>}
      <div style={{ display: 'flex', gap: 8, marginTop: 16, justifyContent: 'flex-end' }}>
        <button type="button" onClick={onClose} style={cancelBtnStyle}>Cancel</button>
        <button onClick={submit} disabled={loading} style={{ ...confirmBtnStyle, background: '#c0392b' }}>{loading ? 'Deleting…' : 'Delete'}</button>
      </div>
    </Overlay>
  );
}

function Overlay({ children, onClose }: { children: React.ReactNode; onClose: () => void }) {
  return (
    <div
      style={{ position: 'fixed', inset: 0, background: 'rgba(0,0,0,0.6)', display: 'flex', alignItems: 'center', justifyContent: 'center', zIndex: 200 }}
      onClick={e => { if (e.target === e.currentTarget) onClose(); }}
    >
      <div style={{ background: '#2d2d2d', borderRadius: 6, padding: 24, minWidth: 360, boxShadow: '0 4px 24px rgba(0,0,0,0.5)' }}>
        {children}
      </div>
    </div>
  );
}

const labelStyle: React.CSSProperties = { display: 'block', fontSize: 13, color: '#888', marginBottom: 4 };
const inputStyle: React.CSSProperties = { width: '100%', background: '#1e1e1e', border: '1px solid #555', borderRadius: 3, padding: '6px 8px', color: '#ccc', marginBottom: 12, boxSizing: 'border-box' };
const cancelBtnStyle: React.CSSProperties = { padding: '6px 16px', background: 'transparent', border: '1px solid #555', color: '#ccc', borderRadius: 3, cursor: 'pointer' };
const confirmBtnStyle: React.CSSProperties = { padding: '6px 16px', background: '#007acc', color: 'white', border: 'none', borderRadius: 3, cursor: 'pointer' };
```

- [ ] **Step 2: Full build check**

```bash
bun run build:all
```

Expected: TypeScript compiles, Vite builds SPA to `dist/gui/`.

- [ ] **Step 3: Commit**

```bash
git add gui-frontend/components/WorktreeDialog.tsx
git commit -m "feat(gui): add worktree create/delete dialogs"
```

---

## Chunk 9: Frontend — Settings Panel

### Task 17: Settings panel

**Files:**
- Create: `gui-frontend/components/SettingsPanel.tsx`

- [ ] **Step 1: Create `gui-frontend/components/SettingsPanel.tsx`**

```tsx
import React, { useEffect, useState } from 'react';
import { api } from '../hooks/useApi.js';

export function SettingsPanel({ onClose }: { onClose: () => void }) {
  const [config, setConfig] = useState<any>(null);
  const [raw, setRaw] = useState('');
  const [saving, setSaving] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [saved, setSaved] = useState(false);

  useEffect(() => {
    api.getConfig().then(c => {
      setConfig(c);
      setRaw(JSON.stringify(c, null, 2));
    }).catch(e => setError(String(e)));
  }, []);

  async function save() {
    setSaving(true); setError(null); setSaved(false);
    try {
      const parsed = JSON.parse(raw);
      await api.putGlobalConfig(parsed);
      setSaved(true);
    } catch (e) {
      setError(String(e));
    } finally {
      setSaving(false);
    }
  }

  return (
    <div style={{ position: 'fixed', inset: 0, background: 'rgba(0,0,0,0.6)', display: 'flex', alignItems: 'center', justifyContent: 'center', zIndex: 200 }}>
      <div style={{ background: '#2d2d2d', borderRadius: 6, padding: 24, width: 600, maxHeight: '80vh', display: 'flex', flexDirection: 'column' }}>
        <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: 16 }}>
          <h3 style={{ margin: 0, color: '#ccc' }}>Settings (Global Config)</h3>
          <button onClick={onClose} style={{ background: 'none', border: 'none', color: '#888', cursor: 'pointer', fontSize: 18 }}>✕</button>
        </div>

        {!config && !error && <p style={{ color: '#888' }}>Loading…</p>}

        {error && <p style={{ color: '#e74c3c', fontSize: 13 }}>{error}</p>}

        {config && (
          <>
            <textarea
              value={raw}
              onChange={e => setRaw(e.target.value)}
              style={{
                flex: 1, minHeight: 300, background: '#1e1e1e', border: '1px solid #555',
                color: '#ccc', fontFamily: 'monospace', fontSize: 13, padding: 12, borderRadius: 3, resize: 'vertical',
              }}
            />
            <div style={{ display: 'flex', gap: 8, marginTop: 12, justifyContent: 'flex-end', alignItems: 'center' }}>
              {saved && <span style={{ color: '#27ae60', fontSize: 13 }}>Saved!</span>}
              <button onClick={onClose} style={{ padding: '6px 16px', background: 'none', border: '1px solid #555', color: '#ccc', borderRadius: 3, cursor: 'pointer' }}>Cancel</button>
              <button onClick={save} disabled={saving} style={{ padding: '6px 16px', background: '#007acc', color: 'white', border: 'none', borderRadius: 3, cursor: 'pointer' }}>
                {saving ? 'Saving…' : 'Save'}
              </button>
            </div>
          </>
        )}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Wire settings button into `App.tsx`**

Add a settings icon/button to the sidebar footer in `App.tsx`:

```tsx
// Add import
import { SettingsPanel } from './components/SettingsPanel.js';

// Add state
const [showSettings, setShowSettings] = useState(false);

// In sidebar area, add a settings button (pass as prop or inline in App)
// After </SessionSidebar> and before terminal panel:
{showSettings && <SettingsPanel onClose={() => setShowSettings(false)} />}
```

Pass `onOpenSettings={() => setShowSettings(true)}` to `SessionSidebar` and add a gear icon button in its footer.

- [ ] **Step 3: Final build**

```bash
bun run build:all
bun run typecheck
```

Both must pass with no errors.

- [ ] **Step 4: Commit**

```bash
git add gui-frontend/components/SettingsPanel.tsx gui-frontend/App.tsx
git commit -m "feat(gui): add settings panel for global config editing"
```

---

## Chunk 10: Integration + End-to-End Smoke Test

### Task 18: End-to-end smoke test

- [ ] **Step 1: Start full GUI stack**

```bash
bun run build:all
bun run --no-env-file dist/cli.js --gui --no-open
```

- [ ] **Step 2: Verify REST endpoints**

```bash
TOKEN="<paste token from server output>"
curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:7288/api/health
# Expected: {"status":"ok"}

curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:7288/api/worktrees
# Expected: JSON array of worktrees

curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:7288/api/config
# Expected: merged config JSON
```

- [ ] **Step 3: Verify SPA loads**

Open `http://127.0.0.1:7288?token=<TOKEN>` in browser.

Expected:
- Sidebar shows any active sessions
- "Select a session" placeholder shown
- Connection status banner absent (connected)

- [ ] **Step 4: Verify terminal session**

In the sidebar, start a session on the current worktree. Terminal panel should appear and show PTY output. Type in the terminal and verify keystrokes reach the CLI.

- [ ] **Step 5: Verify reconnect replay**

Refresh the browser tab. Terminal output should replay from the ring buffer. Session should remain running.

- [ ] **Step 6: Add `dist/gui` to `.gitignore`**

```bash
echo "dist/gui/" >> .gitignore
git add .gitignore
git commit -m "chore: ignore dist/gui build output"
```

- [ ] **Step 7: Update package.json `files` field to include `dist/gui`**

In `package.json`, ensure:
```json
"files": ["dist", "bin"]
```
(already includes `dist/`, which covers `dist/gui/`)

- [ ] **Step 8: Final commit**

```bash
git add package.json
git commit -m "feat(gui): complete Approach B — Bun backend + React SPA GUI"
```

---

## Summary

| Chunk | Deliverable | Gate |
|---|---|---|
| 1 | Project setup + spikes | Keyboard spike pass = proceed with B |
| 2 | Auth + ring buffer | Tested units |
| 3 | REST API | Tested against mocked services |
| 4 | WS + PTY streaming | Tested units; smoke test WS protocol |
| 5 | Server + CLI flag | `ccmanager --gui` starts server |
| 6 | WS client + app shell | Vite builds |
| 7 | Terminal panels | xterm.js renders PTY output in browser |
| 8 | Sidebar + dialogs | Create/delete worktrees from UI |
| 9 | Settings panel | Config editable in UI |
| 10 | E2E smoke test | Full stack works end-to-end |
