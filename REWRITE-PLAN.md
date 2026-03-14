Now I have a thorough understanding of the codebase. Let me produce the comprehensive implementation plan.

---

# REWRITE PLAN: Vimium-C DoD Edition — Clean-Room Implementation Blueprint

**Document version:** 1.0
**Prepared:** 2026-03-13
**Audience:** Development team, security reviewer, ATO package
**Basis:** Exploration of upstream fork at `/home/yanil/Workspace/vimium-c-dod`, NIST SP 800-171 rev2, CMMC Level 2, DoD STIG V5R4 for Chrome/Edge/Firefox

---

## 1. Project Structure

The rewrite lives in a completely new repository. No source files are carried forward from the upstream. Directory and file naming follows a security-first, readable-first convention — no single-letter module names.

```
vimium-dod/
├── .github/
│   └── workflows/
│       ├── ci.yml               # lint, type-check, unit tests, npm audit
│       └── release.yml          # build, sign, artifact upload
├── src/
│   ├── background/              # Service worker (MV3) — no DOM access
│   │   ├── worker.ts            # Entry: registers all runtime listeners
│   │   ├── settings.ts          # Read/write chrome.storage.local with schema validation
│   │   ├── exclusions.ts        # Per-site enable/disable rule engine
│   │   ├── key-mappings.ts      # Parse user keybinding config; build FSM
│   │   ├── commands.ts          # Command registry — maps command names to handlers
│   │   ├── tab-commands.ts      # Tab CRUD, focus, group operations
│   │   ├── history-nav.ts       # Back/forward, session commands
│   │   ├── omnibar-completions.ts  # Query history, bookmarks, tabs for omnibar
│   │   ├── port-manager.ts      # Track content-script ports per tab/frame
│   │   ├── marks-store.ts       # Global marks persistence
│   │   └── message-types.ts     # Exhaustive discriminated union of all messages
│   ├── content/                 # ISOLATED world only — no MAIN world injection
│   │   ├── entry.ts             # Bootstrap: check exclusion, connect port, install handlers
│   │   ├── keyboard.ts          # Keydown capture, FSM evaluation, mode dispatch
│   │   ├── scroll.ts            # Smooth/instant scrolling, viewport math
│   │   ├── link-hints.ts        # Hint overlay generation, label assignment, key matching
│   │   ├── find-mode.ts         # In-page find bar, next/prev navigation
│   │   ├── visual-mode.ts       # Caret, range selection by keyboard
│   │   ├── marks.ts             # Local mark set/jump, dispatch to background for global
│   │   ├── insert-mode.ts       # Focus tracking, pass-through key handling
│   │   ├── omnibar-client.ts    # Open/close omnibar iframe, relay messages
│   │   ├── hud.ts               # Heads-up display overlay (text only, no innerHTML)
│   │   ├── dom-utils.ts         # Safe wrappers: textContent only, no innerHTML
│   │   └── port.ts              # chrome.runtime.connect wrapper, reconnect logic
│   ├── pages/
│   │   ├── options/
│   │   │   ├── options.html     # Settings UI — no inline scripts, no inline styles
│   │   │   ├── options.ts       # Settings page logic
│   │   │   └── options.css      # External stylesheet only
│   │   ├── omnibar/
│   │   │   ├── omnibar.html     # Vomnibar — loaded in sandboxed iframe
│   │   │   ├── omnibar.ts       # Omnibar UI logic
│   │   │   └── omnibar.css
│   │   └── action/
│   │       ├── action.html      # Browser action popup
│   │       ├── action.ts
│   │       └── action.css
│   └── shared/
│       ├── constants.ts         # Enum-only: command IDs, message types, key codes
│       ├── settings-schema.ts   # Zod-free schema: hand-written validator/coercer
│       └── url-utils.ts         # URL validation — strips javascript:, data:, etc.
├── _locales/
│   └── en/
│       └── messages.json        # English only; no zh, no fr, no sp
├── icons/                       # PNG icons produced from SVG source
│   ├── icon16.png
│   ├── icon32.png
│   ├── icon48.png
│   └── icon128.png
├── manifest.json                # The definitive MV3 manifest (see section 3)
├── tsconfig.json                # Strict mode; no AMD; ESNext modules
├── esbuild.config.mjs           # Zero-dep bundler script
├── eslint.config.mjs            # Flat config with eslint-plugin-security
├── vitest.config.ts
├── package.json                 # Pinned deps; no Chinese mirrors; sha512 lockfile
└── package-lock.json            # registry.npmjs.org only; all sha512 integrity hashes
```

### What each top-level source directory owns

| Directory | Runtime context | DOM access | Network access |
|---|---|---|---|
| `src/background/` | MV3 service worker | None | None (connect-src 'none') |
| `src/content/` | Isolated content script world | Tab page DOM (read/react only) | None |
| `src/pages/` | Extension pages (chrome-extension://) | Own page DOM | None |
| `src/shared/` | Compiled into consumers — no runtime of its own | N/A | N/A |

---

## 2. Tech Stack

### TypeScript — version 5.4.x (pinned)

**Rationale:** TypeScript 5.x `const` type parameters, satisfies operator, and stricter generic inference materially reduce the class of runtime bugs. The upstream used TS 4.9 with `noLib: true` and AMD modules — both choices create friction with modern tooling. We use standard `ESNext` module output, not AMD, because the service worker and content scripts are loaded as ES modules by the browser natively.

Flags locked in `tsconfig.json`:
- `strict: true` (covers strictNullChecks, noImplicitAny, strictFunctionTypes, etc.)
- `noUncheckedIndexedAccess: true` — every array read is guarded
- `exactOptionalPropertyTypes: true` — eliminates undefined-as-value bugs
- `target: "ES2022"` — class fields, top-level await, structuredClone available natively
- `module: "ESNext"`, `moduleResolution: "bundler"`
- No `noLib: true` — use the standard TypeScript lib; do not fight the compiler

### Build tool — esbuild 0.21.x (pinned)

**Rationale:** esbuild is a single binary with no transitive npm dependencies (written in Go). This eliminates the gulp + rollup + gulp-typescript + gulp-concat chain that introduced `braces`/`ajv` vulnerabilities in the upstream build environment. esbuild natively handles TypeScript, path aliases, tree-shaking, and minification. The configuration is a single `.mjs` file checked into the repo.

Build outputs one bundle per entry point:
- `dist/background/worker.js` (ES module)
- `dist/content/entry.js` (IIFE to avoid global leakage)
- `dist/pages/options/options.js`
- `dist/pages/omnibar/omnibar.js`
- `dist/pages/action/action.js`

No dynamic imports. No code splitting. Each bundle is self-contained and byte-stable across builds (deterministic by pinned tool version).

### Test framework — Vitest 1.x (pinned)

**Rationale:** Vitest uses the same esbuild pipeline, requires no Babel transform, and runs in Node with JSDOM for DOM tests. It has native TypeScript support and produces tap-compatible output that GitHub Actions can consume. The upstream used a bespoke test runner with no clear framework — Vitest gives a maintainable, auditable baseline.

Unit tests run in Node (no browser required). Browser-level integration tests use Playwright with the extension loaded from `dist/`.

### Linter — ESLint 9.x flat config + eslint-plugin-security 3.x (pinned)

**Rationale:** `eslint-plugin-security` flags `eval()`, `new Function()`, unsafe regex, non-literal RegExp construction, and `innerHTML` assignments. These rules directly address the upstream's critical findings. All rules run in CI; any violation blocks merge.

No Prettier — formatting is enforced by ESLint's built-in formatter rules. This reduces the dependency surface.

### No runtime dependencies

The extension bundle has zero npm runtime dependencies. All logic is hand-written TypeScript. Build-time dependencies (esbuild, typescript, vitest, eslint) are `devDependencies` only and are not included in the extension package.

---

## 3. Manifest.json Design

```jsonc
{
  "manifest_version": 3,
  "name": "__MSG_name__",
  "short_name": "__MSG_short_name__",
  "description": "__MSG_description__",
  "version": "1.0.0",
  "version_name": "1.0.0-dod",
  "minimum_chrome_version": "121",
  "default_locale": "en",

  // NO update_url — distributed via enterprise policy only
  // NO author field referencing upstream

  "background": {
    "service_worker": "background/worker.js",
    "type": "module"
  },

  "action": {
    "default_popup": "pages/action/action.html",
    "default_icon": {
      "16": "icons/icon16.png",
      "32": "icons/icon32.png",
      "48": "icons/icon48.png",
      "128": "icons/icon128.png"
    },
    "default_title": "__MSG_name__"
  },

  "options_ui": {
    "page": "pages/options/options.html",
    "open_in_tab": true
  },

  "content_scripts": [
    {
      "matches": ["<all_urls>"],
      "js": ["content/entry.js"],
      "run_at": "document_start",
      "all_frames": true,
      "match_about_blank": false,
      "match_origin_as_fallback": false
      // NO "world": "MAIN" — isolated world only, always
    }
  ],

  "permissions": [
    "tabs",
    "history",
    "storage",
    "sessions",
    "scripting",
    "webNavigation",
    "clipboardWrite",
    "favicon",
    "offscreen"
  ],
  // Deliberately omitted vs upstream: clipboardRead, tabGroups, bookmarks (optional only)
  // Removed: notifications, search, downloads, cookies, contentSettings

  "optional_permissions": ["bookmarks"],

  "host_permissions": ["<all_urls>"],
  "optional_host_permissions": [],

  "content_security_policy": {
    "extension_pages": "script-src 'self'; style-src 'self'; object-src 'none'; connect-src 'none'"
    // style-src 'unsafe-inline' REMOVED — all styles are external .css files
  },

  "web_accessible_resources": [
    {
      "resources": ["pages/omnibar/omnibar.html"],
      "matches": ["<all_urls>"],
      "use_dynamic_url": true
      // Dynamic URL prevents enumeration by external pages
      // Only the omnibar html is exposed — no scripts, no lib files
    }
  ],

  "incognito": "not_allowed",
  "offline_enabled": true,

  "commands": {
    "createTab":    { "description": "__MSG_cmd_createTab__" },
    "goBack":       { "description": "__MSG_cmd_goBack__" },
    "goForward":    { "description": "__MSG_cmd_goForward__" },
    "previousTab":  { "description": "__MSG_cmd_previousTab__" },
    "nextTab":      { "description": "__MSG_cmd_nextTab__" },
    "reloadTab":    { "description": "__MSG_cmd_reloadTab__" },
    "userCustom1":  { "description": "__MSG_cmd_userCustom1__" },
    "userCustom2":  { "description": "__MSG_cmd_userCustom2__" }
  }
}
```

### Manifest security rationale

**`clipboardRead` removed:** The upstream needed it for paste-and-go functionality that used JavaScript evaluation. Since `javascript:` URL evaluation is removed, paste-and-go is either plain URL navigation (safe) or not supported. If paste-and-go without eval is desired, the offscreen document path can read clipboard via the Clipboard API without declaring `clipboardRead` in the manifest.

**`tabGroups` removed from required, made optional:** Tab group management is a nice-to-have; removing it from required permissions reduces the extension's privilege surface for installations that don't need it.

**`match_about_blank: false`:** The upstream set this to `true` which injected into `about:blank` frames. This is not needed for core Vim navigation and increases attack surface.

**`style-src 'unsafe-inline'` removed:** All extension pages must use external CSS files. This is a design constraint that propagates to every HTML page in the extension. The upstream's audit flagged this as a residual risk — the rewrite eliminates it by design.

**No `key` field:** The upstream had a hardcoded extension public key in the manifest. This is unnecessary for enterprise policy deployment and leaks key material into the public repo.

---

## 4. Architecture — Communication Design

The extension has three runtime contexts that communicate exclusively by message passing. There is no shared memory. There is no eval. There is no script injection into MAIN world.

### 4.1 Contexts and their roles

**Background Service Worker** (`src/background/`)
- Owns all persistent state: settings, exclusion rules, mark positions (global), key mapping FSM
- Responds to content-script requests via `chrome.runtime.onConnect` and `chrome.runtime.onMessage`
- Calls browser APIs: `tabs`, `history`, `webNavigation`, `sessions`, `storage`
- Never has access to page DOM

**Content Script** (`src/content/`) — one instance per frame
- Runs in ISOLATED world — cannot access page's JavaScript globals
- Captures keyboard events, runs mode FSM, renders overlays (hints, HUD, find bar)
- Opens omnibar by creating an `<iframe>` pointing to the omnibar page
- Communicates with background via a long-lived port (one per top-level frame)
- Communicates with sub-frames via `postMessage` to child frame content scripts

**Extension Pages** (`src/pages/`)
- Options page: reads/writes settings via `chrome.storage.local` directly
- Omnibar page: loaded inside an iframe injected by content script; communicates via `postMessage` (not chrome.runtime.connect) because it is technically a web page context inside the content script's iframe
- Action popup: reads current tab state, shows enable/disable toggle

### 4.2 Message type system

All messages are defined in `src/shared/constants.ts` as a discriminated union:

```typescript
// Pattern — every message has a mandatory `kind` discriminant
type ContentToBackground =
  | { kind: "keyCommand"; command: CommandId; count: number; frameId: number }
  | { kind: "requestSettings"; }
  | { kind: "reportUrl"; url: string; }
  | { kind: "markSet"; markChar: string; position: MarkPosition; global: boolean }
  | { kind: "markJump"; markChar: string; global: boolean }
  | { kind: "omnibarQuery"; query: string; engines: SugType }
  | { kind: "omnibarNavigate"; url: string }
  | { kind: "copyToClipboard"; text: string }
  | { kind: "historyBack"; count: number }
  | { kind: "historyForward"; count: number }

type BackgroundToContent =
  | { kind: "settingsPayload"; settings: ContentSettings }
  | { kind: "exclusionResult"; excluded: boolean; passKeys: string }
  | { kind: "omnibarResults"; suggestions: Suggestion[] }
  | { kind: "executeCommand"; command: CommandId; options: CommandOptions }
  | { kind: "markPosition"; position: MarkPosition | null }
```

The background's `port-manager.ts` maintains a `Map<tabId, Map<frameId, Port>>`. On port connect, the background checks the sender URL against the exclusion list and sends back either a `settingsPayload` (active) or `exclusionResult: excluded: true` (disabled for this site). The content script in a disabled tab tears itself down and disconnects.

### 4.3 Content script to sub-frame communication

When the user presses `f` to enter hint mode, the top-level content script needs hint data from all sub-frames. Sub-frame content scripts expose a `MessageEvent` listener on their window. The top-level script uses:

```
window.frames[n].postMessage({ kind: "collectHints", viewBox: ... }, extensionOrigin)
```

Sub-frames respond with `{ kind: "hintsResponse", hints: [...] }` postMessage back. All `postMessage` communications use the extension's `chrome.runtime.id`-derived origin as the target origin — never `"*"`.

### 4.4 Content script to omnibar communication

The omnibar iframe is created with a `src` of `chrome.runtime.getURL("pages/omnibar/omnibar.html") + "?token=<random>"`. The token is a cryptographically random string generated with `crypto.getRandomValues`. The omnibar page reads the token from its URL and uses it to authenticate subsequent `postMessage` traffic. This prevents any other page from injecting messages into the omnibar.

Message flow:
1. Content script creates iframe, stores token
2. Content script sends `{ kind: "activate", token, options }` via `postMessage` to iframe's `contentWindow`
3. Omnibar sends `{ kind: "ready", token }` back to `window.parent`
4. Content script verifies token, then relays omnibar queries to background via port
5. Background sends results back to content script port
6. Content script forwards results to omnibar via `postMessage`
7. On selection, omnibar sends `{ kind: "selected", token, url }` to parent
8. Content script verifies token, passes URL to background for navigation

No `chrome.runtime.connect` from within the omnibar page — all communication is parent/child `postMessage`.

---

## 5. Feature Implementation Order

Features are ordered by dependency depth. Each phase must be code-reviewed and security-checked before the next begins.

### Phase 1 — Infrastructure (weeks 1–2)

1. Repository bootstrap: `package.json` with pinned devDeps, lockfile, `.npmrc` pointing to `registry.npmjs.org`, esbuild config, tsconfig, ESLint config
2. `manifest.json` (complete, final)
3. `src/shared/constants.ts` — all enums, message types, command IDs
4. `src/background/worker.ts` — bare service worker: registers listeners, no features yet
5. `src/background/settings.ts` — schema definition, read/write, migration stub
6. `src/background/port-manager.ts` — connect/disconnect tracking, per-tab frame map
7. `src/content/entry.ts` — bare content script: connect port, receive settings, tear down if excluded
8. `src/content/port.ts` — connect wrapper with reconnect logic
9. CI pipeline skeleton: `npm run typecheck`, `npm run lint`, `npm audit --audit-level=moderate`, `npm test`

**Security gate:** npm audit must pass. ESLint with security plugin must pass. TypeScript must compile with zero errors.

### Phase 2 — Keyboard engine and scrolling (weeks 3–4)

10. `src/content/keyboard.ts` — keydown listener, mode FSM, count prefix parsing, repeat handling
11. `src/content/scroll.ts` — `j/k/gg/G/d/u/h/l` commands; smooth scroll using `requestAnimationFrame`; no eval for scroll amount calculation
12. `src/background/commands.ts` — command registry; connect keyboard FSM output to command dispatch
13. `src/content/hud.ts` — HUD overlay using `document.createElement` and `textContent` only
14. `src/content/insert-mode.ts` — focus detection, pass-through when in text fields

**Security gate:** Zero `innerHTML` assignments in content scripts. Zero `eval()` calls. Confirm ISOLATED world only (no `world: MAIN`).

### Phase 3 — Link hints (weeks 5–6)

15. `src/content/link-hints.ts` — element collection, visibility filtering, label assignment, overlay rendering, keypress matching, activation
16. `src/shared/url-utils.ts` — URL sanitizer: block `javascript:`, `data:text/html`, `vbscript:`; only allow `http:`, `https:`, `ftp:`, extension-origin URLs for specific actions
17. `src/content/link-hints.ts` hint action dispatch — click, open-in-new-tab, copy URL; all routed through URL sanitizer

**Security gate:** URL sanitizer unit tests must achieve 100% branch coverage. No hint action can trigger JavaScript execution.

### Phase 4 — Find mode (weeks 7–8)

18. `src/content/find-mode.ts` — regex construction from user input (with `try/catch`; failed regex shows error in HUD); `window.find()` or Range-based implementation; next/prev match; match count display

**Security gate:** Confirm user input is passed to `RegExp()` only inside a try/catch; the resulting `RegExp` object is never serialized back to a string for re-evaluation.

### Phase 5 — Tab management (weeks 9–10)

19. `src/background/tab-commands.ts` — `t` (new tab), `x` (close tab), `J/K` (prev/next tab), `^` (alternate tab), `<c-s-t>` (restore tab)
20. `src/background/history-nav.ts` — `H/L` (back/forward); uses `chrome.sessions` for undo-close

### Phase 6 — Omnibar / Vomnibar (weeks 11–13)

21. `src/background/omnibar-completions.ts` — history, bookmarks (if permission granted), tab search, search engine dispatch
22. `src/pages/omnibar/omnibar.html` + `omnibar.ts` — iframe UI; results rendered with `document.createElement` and `textContent`; no `innerHTML` for suggestion content
23. `src/content/omnibar-client.ts` — iframe lifecycle, token exchange, relay logic

**Security gate:** Confirm all suggestion text (page titles, URLs from history) is inserted via `textContent` never `innerHTML`. Verify token authentication flow. Confirm `postMessage` uses extension origin as target, never `"*"`.

### Phase 7 — Visual/caret mode and marks (weeks 14–15)

24. `src/content/visual-mode.ts` — caret movement by keyboard using Selection API; range extension; yank selection to clipboard; line mode
25. `src/content/marks.ts` — local marks (`m<char>`, `'<char>`) stored as scroll position + hash; global marks stored via background port to `chrome.storage.local`

### Phase 8 — Options page and exclusions (weeks 16–17)

26. `src/pages/options/options.html` + `options.ts` — settings form; key mapping editor; exclusion list editor; all rendered by DOM manipulation
27. `src/background/exclusions.ts` — URL pattern matching (string prefix, domain glob, URLPattern API, regex); evaluated on every port connect and every `webNavigation.onCommitted`
28. `src/background/key-mappings.ts` — parse user keybinding text (one mapping per line); build FSM; validate command names against registry

### Phase 9 — Hardening and polish (weeks 18–19)

29. Pagination (`[[` / `]]`) — find `rel=prev/next` or heuristic anchor text matching
30. `src/pages/action/action.html` — popup: show enabled/disabled status, link to options
31. Icon generation from SVG source (no dependency on upstream's `pngjs` pipeline)
32. Full i18n review — strip all non-English message keys; ensure no Chinese fallback strings remain

---

## 6. Component Breakdown

### 6.1 Background service worker

| File | Purpose |
|---|---|
| `background/worker.ts` | Single entry point. Registers `chrome.runtime.onInstalled`, `onConnect`, `commands.onCommand`, `webNavigation.onCommitted`. Calls `initSettings()` then enters event-driven idle. |
| `background/settings.ts` | Defines `SettingsSchema` (hand-written validator with no external library). `loadSettings()` reads from `chrome.storage.local`, validates each key, applies defaults for missing keys. `saveSettings()` validates before writing. Exposes typed `getSetting<K>()` and `setSetting<K>()`. |
| `background/port-manager.ts` | `PortManager` class: `Map<tabId, Map<frameId, chrome.runtime.Port>>`. On connect: validates sender origin (must be `<all_urls>` match), checks exclusion, sends settings payload. On disconnect: removes from map. Provides `broadcastToTab(tabId, msg)` for settings-change notifications. |
| `background/exclusions.ts` | `ExclusionEngine`: takes the exclusion rules array from settings, compiles each pattern to a matcher (StringPrefix, URLPattern, or RegExp). `check(url)` returns `{ excluded: boolean, passKeys: string }`. All regex compilation is done at settings-load time, not per-request. |
| `background/key-mappings.ts` | `parseKeyMappings(text)`: line-by-line parser for `map <key> <command> [options]` syntax. Returns a validated `KeyFSM` (finite state machine represented as a nested Map). Emits parse errors into a log but never `throw`s or crashes the service worker. |
| `background/commands.ts` | Registry of all command IDs to handler functions. All handlers are `async (options, port) => void`. No command handler calls `eval()` or `new Function()`. |
| `background/tab-commands.ts` | Handlers for tab creation, closing, navigation, reordering, grouping. Uses `chrome.tabs.*` and `chrome.sessions.*`. |
| `background/history-nav.ts` | `goBack(tabId, count)`, `goForward(tabId, count)`. Uses `chrome.tabs.goBack`/`goForward` (Chrome 72+). Falls back to posting a message to the content script which calls `history.back()`. |
| `background/omnibar-completions.ts` | `query(text, engines)` fans out to history searcher, bookmark searcher, tab searcher in parallel via `Promise.all`. Each searcher returns typed `Suggestion[]`. Results are merged, ranked, and sent back. No network calls. |
| `background/marks-store.ts` | Persists global marks (mark char → `{url, scrollX, scrollY, hash}`) in `chrome.storage.local` under a reserved key. Exposes `setGlobalMark`, `getGlobalMark`, `clearGlobalMark`. |
| `background/message-types.ts` | The single source of truth for all message discriminated unions. Imported by both background and content. No logic — types only. |

### 6.2 Content scripts

| File | Purpose |
|---|---|
| `content/entry.ts` | Executed at `document_start`. Creates port, sends URL, awaits settings. If excluded: disconnects and returns. Otherwise: installs keyboard listener, constructs mode controller. Handles port reconnect on service worker wake. |
| `content/port.ts` | `connect()` wrapper: calls `chrome.runtime.connect`. Handles `onDisconnect` with exponential backoff reconnect (max 3 retries). All send operations are queued during reconnect. |
| `content/keyboard.ts` | Installs `keydown` listener at capture phase. Maintains current mode stack. Each keydown is passed to the active mode's handler. Modes: normal, insert, hint, find, visual, mark, passthrough. Key sequence accumulation with timeout reset. |
| `content/scroll.ts` | `scrollBy(x, y, smooth)` and `scrollTo(x, y)` with frame-rate-aware animation. Smart target element selection: finds the deepest scrollable ancestor. `j/k` scroll document or focused scrollable. `d/u` half-page. `gg/G` top/bottom. `h/l` horizontal. |
| `content/link-hints.ts` | `HintSession`: collects all interactive elements (`a[href]`, `button`, `input`, `[role=button]`, `[tabindex]`), filters by visibility via `getBoundingClientRect`, assigns alphanumeric labels, injects a `<div>` overlay (not into MAIN world), handles label keystrokes, activates or dismisses. All label text via `textContent`. |
| `content/find-mode.ts` | `FindSession`: opens find HUD. User input constructs a `RegExp` in try/catch. `window.find()` used for native match; custom Range API used for match count and wrap detection. `n/N` advance forward/backward. |
| `content/visual-mode.ts` | `VisualSession`: enters caret mode via `Selection.collapse()`. Arrow-like commands extend selection using `Selection.modify()`. `y` yanks selection text to clipboard via background port. Line mode toggles line-granularity. |
| `content/marks.ts` | Local marks: `Map<char, {scrollX, scrollY, hash}>` in module scope. Global marks: sent to background port. Jump: `scrollTo` with hash navigation for hash-carrying marks. |
| `content/insert-mode.ts` | Detects focus entering an editable element. Pushes `insert` onto mode stack (suppresses all Vim keys except `<Escape>`). Detects blur, pops mode stack. |
| `content/omnibar-client.ts` | Creates the omnibar `<iframe>` using the dynamic URL. Manages token exchange handshake. Relays queries from omnibar to background port and results back. Handles omnibar close. |
| `content/hud.ts` | `showHUD(message: string, durationMs?: number)`. Creates a fixed-position `<div>` with `textContent` (never innerHTML). Removes itself after duration or on ESC. |
| `content/dom-utils.ts` | `safeText(el, text)` — sets `textContent`. `createElement(tag, attrs)` — safe element factory. `removeElement(el)`. No `innerHTML` exports. |

### 6.3 Extension pages

| File | Purpose |
|---|---|
| `pages/options/options.html` | Static HTML only. All scripts via `<script src="...">`. No inline scripts. No inline styles. External CSS only. |
| `pages/options/options.ts` | Reads settings via `chrome.storage.local`. Renders key mapping editor and exclusion table by DOM construction. Validates and saves on submit. Shows parse errors from key-mapping parser. |
| `pages/omnibar/omnibar.html` | Loaded inside iframe. Input field, suggestion list. No inline scripts. |
| `pages/omnibar/omnibar.ts` | Receives `postMessage` from parent content script. Renders suggestions as `<li>` elements with `textContent`. Sends selected URL back to parent. Handles keyboard: up/down arrow, Enter, Escape. |
| `pages/action/action.html` | Popup: shows enabled/disabled status for current tab, button to open options. |

### 6.4 Shared modules

| File | Purpose |
|---|---|
| `shared/constants.ts` | `enum CommandId`, `enum MessageKind`, `enum SugType`, key code constants. Import in both background and content with no side effects. |
| `shared/settings-schema.ts` | `validateSettings(raw: unknown): Settings` — throws on invalid type, coerces valid values, applies defaults. No third-party validator library. Hand-written type guards. |
| `shared/url-utils.ts` | `isSafeUrl(url: string): boolean` — returns `true` only for `http:`, `https:`, `ftp:`, `file:` (conditional), and the extension's own origin. Explicitly blocks `javascript:`, `data:`, `vbscript:`, `blob:` by scheme check before any other processing. |

---

## 7. Security Checklist

### Phase 1 gate — Infrastructure
- [ ] `npm audit` exits 0 with no vulnerabilities
- [ ] `eslint-plugin-security` installed and all rules enabled
- [ ] `tsconfig.json` has `strict: true`, `noUncheckedIndexedAccess: true`
- [ ] Manifest has no `update_url`
- [ ] Manifest `incognito: "not_allowed"`
- [ ] `content_security_policy.extension_pages` has no `unsafe-inline`, no `unsafe-eval`, no `connect-src` except `'none'`
- [ ] No `world: "MAIN"` in any `content_scripts` entry
- [ ] `.npmrc` locks registry to `https://registry.npmjs.org/`

### Phase 2 gate — Keyboard/scroll
- [ ] Grep `eval(`, `new Function(`, `innerHTML`, `outerHTML`, `insertAdjacentHTML` in `src/content/` returns zero matches
- [ ] All `postMessage` calls specify a target origin (never `"*"`)
- [ ] Content script does not call any `chrome.tabs.*` API (background only)

### Phase 3 gate — Link hints
- [ ] `url-utils.ts` has unit tests for: `javascript:alert(1)`, `JAVASCRIPT:alert(1)` (case variants), `java\tscript:`, `data:text/html,<script>`, `vbscript:`, `http://example.com` (allowed), `https://example.com` (allowed)
- [ ] Hint activation path always calls `isSafeUrl()` before navigation
- [ ] No hint can trigger a click on a `javascript:` href without URL check blocking it

### Phase 4 gate — Find mode
- [ ] User regex input wrapped in `try { new RegExp(input) } catch { /* show error */ }`
- [ ] DoS test: input of 10,000 `(` characters — must not hang (ReDoS guard: cap input length at 512 chars)

### Phase 6 gate — Omnibar
- [ ] Suggestion rendering uses `textContent` — not `innerHTML` — for page titles and URLs from history
- [ ] Token is generated with `crypto.getRandomValues(new Uint8Array(32))` → hex string (not `Math.random`)
- [ ] `postMessage` to omnibar uses `chrome.runtime.getURL("")` as target origin
- [ ] Omnibar page has no `chrome.runtime.connect` calls — all communication is `postMessage` only
- [ ] Omnibar HTML page has `<meta http-equiv="Content-Security-Policy" content="script-src 'self'; style-src 'self'; default-src 'none'">` as belt-and-suspenders

### Phase 8 gate — Options page
- [ ] Settings import validates every field against `settings-schema.ts` before writing
- [ ] Exclusion pattern input is sanitized: maximum length 2000 chars; reject patterns that compile to a regex with catastrophic backtracking (test: `/^(a+)+$/` style patterns — flag and reject if `RegExp` compilation takes >5ms)
- [ ] Key mapping parser rejects any `map` target that is not in the command registry (no arbitrary command injection)

### Continuous checks (every PR)
- [ ] `npm audit --audit-level=moderate` — zero findings
- [ ] ESLint passes with no suppressions added to security rules
- [ ] TypeScript compile with zero errors and zero suppressions of `@ts-expect-error` on security-critical paths
- [ ] No new permissions added to `manifest.json` without documented justification
- [ ] No new `web_accessible_resources` entries
- [ ] `grep -r "eval(" src/` returns zero matches
- [ ] `grep -r "new Function" src/` returns zero matches
- [ ] `grep -r "innerHTML" src/` is reviewed — any match requires explicit security justification comment

---

## 8. Testing Strategy

### 8.1 Unit tests (Vitest, Node environment)

**`settings-schema.ts`**
- Valid settings object round-trips correctly
- Each invalid field type is rejected and default applied
- Unknown keys are stripped (no prototype pollution via key injection)

**`url-utils.ts`**
- Full matrix of scheme variants including Unicode lookalikes (e.g., `ϳavascript:`)
- Case variants: `JAVASCRIPT:`, `Javascript:`, `jAvAsCrIpT:`
- Percent-encoded variants: `java%73cript:` (must be decoded before scheme check)
- `data:` variants: `data:text/html`, `data:application/xhtml+xml`
- All `http://` and `https://` URLs allowed
- Relative URLs treated as allowed (browser resolves them)

**`exclusions.ts`**
- Domain match: `example.com` matches `https://example.com/path`
- Wildcard: `*.example.com` matches `https://sub.example.com`
- Regex pattern: `^https://secure\\.example\\.com/` matches correctly
- URLPattern syntax tested against spec
- No catastrophic backtracking in compiled regex (timed test)

**`key-mappings.ts`**
- Valid `map j scrollDown` parsed correctly
- Unknown command name `map j unknownCommand` produces parse error, not crash
- `unmap j` removes existing mapping
- `unmapAll` clears all custom mappings
- Malformed lines produce per-line errors without halting the parse of subsequent lines
- Deeply nested `run` chaining does not produce infinite recursion

**`omnibar-completions.ts`**
- History results are deduplicated and ranked correctly
- Query of empty string returns recently visited items
- `isSafeUrl` is called on all returned suggestion URLs

**`scroll.ts` (with JSDOM)**
- `scrollBy` respects viewport boundaries
- Animation frame callback cleans up on destruction

### 8.2 Integration tests (Playwright, real Chrome/Firefox)

Use Playwright's `launchPersistentContext` with `--load-extension=dist/` and `--disable-extensions-except=dist/`.

**Smoke tests**
- Extension loads without errors (no `chrome.runtime.lastError` on startup)
- Content script connects to background (check port map via test harness)
- Navigating to `https://example.com` shows extension icon as enabled

**Link hints**
- Press `f`, verify hint overlay appears
- Type hint label characters, verify link is followed or opened in new tab
- Press `<Escape>`, verify hints dismissed cleanly

**Scrolling**
- Press `j` five times, verify `window.scrollY` increases
- Press `gg`, verify `window.scrollY === 0`
- Press `G`, verify page scrolled to bottom

**Find mode**
- Press `/`, type a search term, verify match highlighted
- Press `n`, verify next match selected
- Press `<Escape>`, verify find mode exited

**Omnibar**
- Press `o`, verify omnibar iframe appears
- Type a URL, press Enter, verify navigation
- Press `<Escape>`, verify omnibar dismissed

**Tab commands**
- Press `t`, verify new tab opened
- Press `x`, verify current tab closed
- Press `J`/`K`, verify tab focus moves

**Security tests**
- Navigate to a page with `javascript:` links, press `f`, activate the hint — verify no JavaScript executes
- Import a crafted settings JSON with `"<script>"` in an exclusion pattern — verify it is rejected by schema validator
- Inject a `postMessage` with a wrong token to the omnibar iframe — verify it is ignored

### 8.3 Manual testing checklist (pre-release)

- Test on Chrome 121, Chrome latest, Firefox ESR, Firefox latest, Edge 121+
- Verify `about:blank` frames: extension is NOT active (match_about_blank: false)
- Verify incognito windows: extension is NOT active (incognito: not_allowed)
- Verify options page: all settings persist across browser restart
- Verify exclusion: add `https://example.com` to exclusion list, verify extension disabled on that domain
- Verify global marks persist across tab close/reopen
- Verify custom key mapping takes effect without browser restart
- Verify `npm audit` on the installed `node_modules` (not just lockfile) reports zero findings
- Run Chrome's "Analyze extension" tool in the Extensions developer panel and review all findings

---

## 9. Build and CI Pipeline

### 9.1 Development build

```
npm run build:dev
```

Runs `esbuild` with `sourcemap: "inline"`, no minification. Output to `dist/`. The `dist/` directory can be loaded directly as an unpacked extension in Chrome or Firefox.

### 9.2 Production build

```
npm run build:prod
```

Runs `esbuild` with:
- `minify: true`
- `sourcemap: false`
- `define: { DEBUG: "false" }`
- `legalComments: "none"` — strips all comments including license headers from output (license.txt is included separately in the package)
- `drop: ["debugger", "console"]` — strips all console.log calls

Output to `dist/`. Build is deterministic: same source + same pinned esbuild version = byte-identical output.

### 9.3 CI pipeline (GitHub Actions)

**On every push and PR:**

```yaml
# ci.yml
jobs:
  validate:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci --prefer-offline    # uses lockfile, no network beyond registry
      - run: npm audit --audit-level=moderate
      - run: npx tsc --noEmit          # type check without emitting
      - run: npm run lint               # eslint with security plugin
      - run: npm test                   # vitest unit tests
      - run: npm run build:prod         # verify build succeeds
```

**On tag push (release):**

```yaml
# release.yml
jobs:
  release:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci --prefer-offline
      - run: npm audit --audit-level=moderate
      - run: npm run build:prod
      - name: Package extension
        run: |
          cd dist
          zip -r ../vimium-dod-${{ github.ref_name }}.zip .
      - name: Generate SHA-256 manifest
        run: |
          sha256sum vimium-dod-${{ github.ref_name }}.zip > vimium-dod-${{ github.ref_name }}.zip.sha256
      - name: Upload release artifacts
        uses: actions/upload-artifact@v4
        with:
          name: extension-package
          path: |
            vimium-dod-*.zip
            vimium-dod-*.zip.sha256
```

### 9.4 Signing for enterprise deployment

The Chrome CRX package is signed using the organization's code signing private key, held in CI secrets (not in the repository). The signing step runs after the zip is produced:

```
chrome --pack-extension=dist/ --pack-extension-key=$SIGNING_KEY_PATH
```

The resulting `.crx` file and its `.sha256` hash are uploaded as release artifacts. The enterprise Chrome policy pushes the `.crx` from an internal web server; the extension's `update_url` points to that internal server (set in the enterprise-deployed manifest, not the source manifest).

For Firefox, the extension is signed via `web-ext sign` using an AMO API key for self-hosted distribution, or via the organization's Firefox enterprise policy using unsigned XPI (Firefox allows unsigned extensions via enterprise policy).

### 9.5 Dependency provenance

All npm packages in `package-lock.json` must:
- Resolve from `registry.npmjs.org` (enforced by `.npmrc`)
- Have a `sha512` integrity hash in the lockfile
- Pass `npm audit` at moderate severity or above
- Have no transitive dependencies that resolve from non-npmjs registries

A CI step verifies this by running `npm ls --json` and checking all `resolved` fields against an allowlist.

---

## 10. NIST SP 800-171 Control Coverage

The table maps each NIST 800-171 rev2 control family to the rewrite components that address it. Controls that are not applicable to a browser extension are noted as N/A with justification.

### 3.1 Access Control

| Control | ID | Component | How addressed |
|---|---|---|---|
| Limit system access to authorized users | 3.1.1 | `manifest.json`, `background/settings.ts` | Extension is not a multi-user system; access is limited to the logged-in OS user. Enterprise deployment via policy restricts who can install. |
| Limit system access to types of transactions | 3.1.2 | `manifest.json` permissions | Minimal permissions declared; `optional_permissions` used for non-core features (bookmarks); no downloads, cookies, contentSettings |
| Control information flow | 3.1.3 | `manifest.json` CSP, `background/port-manager.ts` | `connect-src 'none'` blocks all outbound network from extension pages. No `update_url`. Content scripts only communicate to extension background — not to external hosts. |
| Separate duties | 3.1.4 | N/A | Single-user tool; separation of duties not applicable to a keyboard navigation extension. |
| Employ least privilege | 3.1.5 | `manifest.json`, `shared/url-utils.ts` | Only 9 declared permissions vs 14 in upstream. `clipboardRead` removed. All permissions justified. URL sanitizer prevents privilege escalation via link activation. |
| Control use of privileged accounts | 3.1.6 | N/A | Extension does not manage accounts. |

### 3.4 Configuration Management

| Control | ID | Component | How addressed |
|---|---|---|---|
| Establish baseline configurations | 3.4.1 | `manifest.json`, `src/shared/settings-schema.ts` | Manifest is the authoritative security baseline. Settings schema enforces known-good defaults on every load. Unknown keys stripped. |
| Establish a configuration management policy | 3.4.2 | CI pipeline, `package-lock.json` | All dependency versions pinned. Build is deterministic and reproducible. Release artifacts include SHA-256 hashes. |
| Track and control changes | 3.4.3 | Git history, CI release pipeline | All changes require PR review. Releases are tagged commits. CI blocks merges that fail security checks. |
| Analyze security impact of changes | 3.4.4 | CI security gates | `npm audit`, ESLint security plugin, and `grep` for banned patterns run on every PR. |
| Define allowable software | 3.4.6 | `manifest.json` permissions, enterprise policy | Enterprise policy defines which extensions are allowed. This extension requests no permissions beyond what is needed. |
| Prevent use of unauthorized software components | 3.4.7 | `package.json` zero runtime dependencies, CI provenance check | Zero runtime dependencies in extension bundle. Build-time dependencies validated against npmjs.org with sha512 integrity. |

### 3.5 Identification and Authentication

| Control | ID | Component | How addressed |
|---|---|---|---|
| Identify system users | 3.5.1 | N/A | Extension does not authenticate users. Identity is managed by the OS and browser. |
| Authenticate system users | 3.5.2 | N/A | Same as above. |
| Use multifactor authentication | 3.5.3 | N/A | Same as above. Browser authentication is managed by DoD STIG controls outside this extension. |

### 3.8 Media Protection

| Control | ID | Component | How addressed |
|---|---|---|---|
| Protect CUI on digital media | 3.8.1 | `background/settings.ts` | Settings stored in `chrome.storage.local` only — no `storage.sync` (which would transmit to Google servers). Settings contain only keyboard shortcuts, not CUI. |

### 3.11 Risk Assessment

| Control | ID | Component | How addressed |
|---|---|---|---|
| Periodically assess risk | 3.11.1 | CI `npm audit`, periodic manual review | `npm audit` runs on every CI run. Schedule quarterly manual review of dependency updates and new CVEs. |

### 3.12 Security Assessment

| Control | ID | Component | How addressed |
|---|---|---|---|
| Periodically assess security controls | 3.12.1 | Security checklist (Section 7) | Per-phase security gates documented. Pre-release manual checklist. |
| Plan and implement remediation | 3.12.2 | GitHub Issues, CI blocking | Security findings from ESLint/audit block CI. Issues tracked in repository. |

### 3.13 System and Communications Protection

| Control | ID | Component | How addressed |
|---|---|---|---|
| Monitor and control communications | 3.13.1 | `manifest.json` CSP `connect-src 'none'` | Extension pages cannot make any network connections. Content scripts cannot make fetch/XHR calls to external hosts. |
| Employ architectural security design | 3.13.2 | Architecture (Section 4) | Message-passing only between contexts. No MAIN world injection. No shared memory. No eval. Omnibar token authentication prevents cross-context injection. |
| Separate user and system functionality | 3.13.3 | ISOLATED world content scripts | Content scripts run in ISOLATED world — they cannot access page JavaScript globals and page scripts cannot access extension globals. |
| Prevent unauthorized/unintended information transfer | 3.13.4 | CSP, `url-utils.ts` | `connect-src 'none'` prevents data exfiltration from extension pages. URL sanitizer prevents navigation to `javascript:` or `data:` URLs that could exfiltrate data via browser execution. |
| Implement subnetwork protection | 3.13.5 | N/A | Network boundary management is handled by the network infrastructure, not this extension. |
| Protect communication sessions | 3.13.8 | `postMessage` token auth (Section 4.4) | Omnibar `postMessage` channel authenticated by cryptographically random token. Prevents cross-site message injection. |
| Protect CUI during transmission | 3.13.8 | N/A | Extension does not transmit CUI. All storage is local. |
| Terminate sessions after defined period | 3.13.9 | N/A | Extension is stateless with respect to sessions; it follows the browser's session model. |

### 3.14 System and Information Integrity

| Control | ID | Component | How addressed |
|---|---|---|---|
| Identify, report, and correct flaws | 3.14.1 | CI pipeline, security checklist | Automated flaw detection on every commit. Findings logged in repository issues. |
| Provide protection from malicious code | 3.14.2 | Zero eval, no MAIN world, URL sanitizer | The three critical upstream findings (eval injection, MAIN world script injection, javascript: URL execution) are eliminated by architectural design in the rewrite. ESLint rules prevent re-introduction. |
| Monitor security alerts | 3.14.3 | `npm audit` CI integration | `npm audit` on every CI run surfaces new CVEs against build dependencies. |
| Update malicious code protection | 3.14.4 | Pinned dependencies, CI provenance | All dependencies pinned with sha512 integrity. No auto-update of build tools. |
| Perform periodic scans | 3.14.6 | ESLint, TypeScript strict, quarterly review | Static analysis runs continuously. Quarterly dependency audit and review of new browser extension security advisories. |
| Protect against unsolicited network traffic | 3.14.7 | `connect-src 'none'`, no `update_url` | Extension initiates no outbound connections. No telemetry. No auto-update check. |

---

## Implementation Notes for the Development Team

### On clean-room compliance

No file from the upstream `vimium-c-dod` repository may be copied into the rewrite. This includes TypeScript source files, JavaScript build outputs, CSS, HTML, icon files, and type definitions. All implementation must be written from scratch by the development team working from functional specifications and browser extension documentation.

The **behavioral specification** (what features do, how they respond to keypresses, what the UI looks like) may be informed by the upstream as a reference. The **code** that implements those behaviors must be independently authored.

The one exception is `_locales/en/messages.json`: the message key names and English string values may be authored from scratch with reference to the upstream's key names, as the message keys are functional identifiers that need to match the manifest's `__MSG_...` references. The strings themselves should be authored independently.

### On the 1784-line eval engine

`lib/simple_eval.ts` in the upstream is a complete JavaScript expression evaluator. It must not appear in the rewrite in any form. The "math evaluator" feature (`vimium://e 2+2`) is explicitly on the removed features list. No math evaluation whatsoever is supported. If a user types a math expression in the omnibar, it is treated as a search query.

### On Firefox compatibility

The MV3 Firefox implementation has known differences from Chrome:
- Firefox does not support `chrome.offscreen` — clipboard write in Firefox MV3 uses `navigator.clipboard.writeText` directly from the content script (available in Firefox 63+ content script context)
- Firefox `service_worker` requires the `background.scripts` key in Firefox-specific builds until Firefox 121 where MV3 service workers stabilized
- `chrome.sessions.restore` is not available in Firefox — the undo-close tab feature falls back to omitting this in Firefox builds

The build system uses esbuild's `define` feature to compile browser-specific code paths. A `BROWSER` constant is defined at build time (`chrome` or `firefox`). TypeScript declaration merging in `src/shared/constants.ts` handles the conditional types.

### On the `<all_urls>` host permission

This permission is unavoidable for a keyboard navigation extension that must work on every website. The DoD mitigation is:

1. Enterprise Chrome policy `ExtensionSettings` can restrict the extension's host permissions at deployment time to a domain allowlist, overriding `<all_urls>` to only the organization's known web domains
2. The exclusion list in settings provides user-level control
3. `webNavigation.onCommitted` is used to disable the extension on pages matching exclusion rules before any content script work occurs

This design is documented in the ATO package as a compensating control.

---

### Critical Files for Implementation

- `/home/yanil/Workspace/vimium-c-dod/manifest.json` - Definitive baseline for the new manifest design; all security-relevant fields (permissions, CSP, web_accessible_resources, incognito) are derived from this hardened version
- `/home/yanil/Workspace/vimium-c-dod/SECURITY-AUDIT.md` - Complete threat model and residual risk inventory; every finding maps to a design decision in the rewrite plan
- `/home/yanil/Workspace/vimium-c-dod/background/port-manager.ts` (as `ports.ts`) - Reference for the port lifecycle, per-tab frame mapping, and message routing patterns that must be re-implemented cleanly
- `/home/yanil/Workspace/vimium-c-dod/content/link-hints.ts` - Canonical behavioral specification for the hint system (data structures, label assignment algorithm, visibility filtering logic) to inform the clean-room implementation
- `/home/yanil/Workspace/vimium-c-dod/background/exclusions.ts` - Reference for the exclusion rule matching engine (pattern types: StringPrefix, URLPattern, RegExp) and the behavioral contract for the clean-room exclusions module
