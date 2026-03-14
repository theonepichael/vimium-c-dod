# Security Audit — Vimium-C DoD Fork

**Version audited:** 2.12.3 (upstream) → 2.12.3-dod.1 (this fork)
**Audit date:** 2026-03-13
**Framework:** NIST SP 800-171 rev2 / CMMC Level 2
**Threat model:** Browser extension running on DoD contractor workstations handling CUI

---

## 1. Executive Summary

This document records the security findings from auditing the vimium-c browser extension for DoD use and the mitigations applied. The original codebase had **critical** supply chain and code execution risks that have been addressed. Several residual risks remain and are tracked as open issues.

---

## 2. Supply Chain Risk (CRITICAL)

### 2.1 Chinese origin
| Finding | Risk |
|---|---|
| Author email `gdh1995@qq.com` (QQ, Chinese platform) | NIST 800-161 supply chain flag |
| `README-zh.md` — primary maintainer is Chinese | Jurisdiction risk |
| `npm-shrinkwrap.json` locked `source-map-support` to `registry.nlark.com` (Alibaba/Taobao Chinese mirror) | Package could have been tampered |

**Mitigation applied:** Updated `npm-shrinkwrap.json` to resolve from `registry.npmjs.org`.

**Residual risk / Recommendation:** Full rewrite of security-critical components is recommended for any production DoD ATO. This fork should be treated as a reference implementation.

### 2.2 Dependency vulnerabilities (build-time)
| Package | Severity | Finding |
|---|---|---|
| `braces < 3.0.3` | High | Uncontrolled resource consumption (DoS) |
| `ajv < 6.14.0` | Moderate | ReDoS via `$data` option |
| `brace-expansion 1.0.0–1.1.11` | Moderate | ReDoS |

**Note:** These are build-tool dependencies (gulp, eslint), not present in the distributed extension bundle. Risk is confined to the CI/build environment.

**Mitigation pending:** `npm audit fix --force` blocked on gulp 5 breaking change. Track in #5.

---

## 3. Permissions Analysis

### 3.1 Required permissions
| Permission | Justification | DoD Risk | Status |
|---|---|---|---|
| `<all_urls>` (host) | Inject content scripts everywhere | HIGH — universal page access | Retained (required for function); restrict via enterprise policy |
| `tabs` | Tab titles/URLs for omnibar | HIGH — full browsing history access | Retained (core feature) |
| `webNavigation` | Per-site enable/disable rules | MEDIUM | Retained |
| `scripting` | MV3 script injection | MEDIUM | Retained |
| `history` | History search in omnibar | HIGH | Retained |
| `clipboardRead/Write` | Copy URL, paste-and-go | MEDIUM | Retained |
| `sessions` | Session restore commands | LOW | Retained |
| `storage` | Settings persistence (local only) | LOW | Retained — sync removed from optional |
| `tabGroups` | Tab group management | LOW | Retained |
| `favicon` | Show favicons in omnibar | LOW | Retained |
| `offscreen` | Clipboard operations in MV3 | LOW | Retained |
| ~~`notifications`~~ | Desktop notifications | MEDIUM — OS-level info disclosure | **REMOVED** |
| ~~`search`~~ | Browser search API | LOW | **REMOVED** (not core) |

### 3.2 Optional permissions removed
- `downloads` / `downloads.shelf` — file system access, removed
- `cookies` — cookie read/write, removed
- `contentSettings` — modify per-site browser settings, removed
- `chrome://*/*` optional host permission — removed

### 3.3 web_accessible_resources
**Original:** `content/*`, `front/vomnibar*`, `lib/*` exposed to ALL websites (`<all_urls>`)

**Issue:** Any website could load extension scripts by guessing URLs (predictable, `use_dynamic_url: false`). This is a significant attack surface — could be used to probe extension behavior or exploit vulnerabilities in extension scripts.

**Hardened:** Only `front/vomnibar.html` exposed to `chrome-extension://*/*` with `use_dynamic_url: true`.

---

## 4. Code Execution Attack Surface

### 4.1 `javascript:` URL execution (CRITICAL — FIXED)

**Finding:** The extension had two code paths that executed arbitrary JavaScript in page context:

1. `content/dom_ui.ts:evalIfOK` — received `javascript:` URLs from background, executed via page script injection
2. `background/open_urls.ts:openJSUrl` — dispatched `javascript:` URLs to content scripts or via `tabs.executeScript`

These paths allowed any `javascript:` URL (from bookmarks, omnibar input, keyboard command) to run arbitrary JS in the active tab's context.

**Mitigation:** Both functions return immediately without execution. `javascript:` links are silently dropped.

**Impact on functionality:** Users cannot open/follow `javascript:` links. This is intentional.

### 4.2 MAIN-world `simple_eval.js` injection (CRITICAL — FIXED)

**Finding:** `content/injected_end.ts` and `pages/loader.ts` dynamically loaded `lib/simple_eval.js` into the MAIN world of every web page via `<script src>` injection. `simple_eval.ts` is a 1784-line full JavaScript expression evaluator that ran in the same context as page scripts.

**Risk:** If any page script could trigger the `tryEval` function, it had access to a full sandbox escape via the native `Function` constructor path in `simple_eval.ts` (line 200–203).

**Mitigation:** Both injection sites replaced with a stub that returns `undefined` and logs a warning. `simple_eval.js` is no longer injected into any page.

**Impact on functionality:** The `vimium://eval`, `vimium://exec`, `vimium://expr` URL handlers in the omnibar now return empty results. The math evaluator (`vimium://e 2+2`) still works via the separate `math_parser.js` module.

### 4.3 `innerHTML` with browser-origin data (MEDIUM — FIXED)

**Finding:** `front/vomnibar.ts` inserted browser history data (page titles, URLs) directly into `innerHTML` without HTML encoding. A malicious page title like `<img src=x onerror=...>` could inject active HTML into the omnibar.

**Mitigation:** Added `VUtils_.escapeHtml_()` to encode all `id===1` template values (the data-origin slot) before HTML insertion.

**Note:** The vomnibar runs in a `chrome-extension://` page with `script-src 'self'` CSP, which limits script injection risk. However, attribute-based XSS (event handlers) could still execute without the fix.

### 4.4 Static `innerHTML` assignment (LOW — FIXED)
`pages/options_wnd.ts` used `innerHTML` for a static Edge-browser tip string. Replaced with `textContent`.

### 4.5 `innerHTML` in `content/commands.ts` (LOW — ACCEPTED)
`content/commands.ts:577` uses `innerHTML` to inject the extension's own HUD UI template. The `html` value is a compile-time string from the extension bundle — not user-controlled data. Risk is accepted as low; flagged for future migration to DOMParser path (matching the Firefox branch).

---

## 5. Network / Telemetry

| Finding | Status |
|---|---|
| `update_url: https://clients2.google.com/service/update2/crx` — sends extension ID + version to Google on each browser update check | **REMOVED** from manifest |
| No other outbound network connections found in source | Confirmed — no fetch/XHR/sendBeacon calls |
| `connect-src 'none'` added to extension CSP | **ADDED** — blocks any accidental fetch from extension pages |

---

## 6. Content Security Policy

| Context | Before | After |
|---|---|---|
| extension_pages (MV3) | `script-src 'self'; style-src 'self' 'unsafe-inline'; object-src 'none'` | `script-src 'self'; style-src 'self' 'unsafe-inline'; object-src 'none'; connect-src 'none'` |
| extension_pages (MV2) | `script-src 'self'; style-src 'self' 'unsafe-inline'; object-src 'none'` | Same + `connect-src 'none'` |

**Residual:** `'unsafe-inline'` remains in `style-src` because extension HTML pages use inline `style="..."` attributes and `<style>` blocks extensively. Removing it requires migrating all inline styles to external stylesheets. Tracked as a pending hardening task.

---

## 7. Data Storage

- Settings stored in `chrome.storage.local` only (not `.sync`)
- `storage.sync` is not in the permissions list
- No encryption of stored settings — acceptable for keyboard shortcut config; must not store sensitive data
- **Recommendation:** Add a settings validation schema to prevent injection via malformed settings import

---

## 8. Incognito Mode

Set to `not_allowed`. The extension will not inject into incognito windows, consistent with DoD workstation policies restricting private browsing.

---

## 9. NIST SP 800-171 Control Mapping

| Control | ID | How addressed |
|---|---|---|
| Limit system access to types of transactions and functions | AC-2 | Permissions minimized; optional perms removed |
| Control information flow | AC-4 | `connect-src 'none'`; no outbound requests; `update_url` removed |
| Identify and authenticate users | IA-2 | N/A — extension does not authenticate |
| Perform maintenance | MA-2 | Dependency audit; lockfile cleaned |
| Protect communications | SC-7 | No network; extension-only storage |
| Control and monitor user-installed software | CM-7 | Extension distributed via enterprise policy, not Web Store |
| Software, firmware, and information integrity | SI-7 | `javascript:` exec disabled; eval injection removed; HTML escaping |
| Perform periodic scans | SI-3 | `npm audit` integrated; static analysis recommended |

---

## 10. Open Risks / Pending Tasks

| # | Risk | Severity | Status |
|---|---|---|---|
| 1 | `style-src 'unsafe-inline'` in extension CSP | Medium | Open — requires style migration |
| 2 | `content/commands.ts` uses innerHTML for HUD template | Low | Accepted (compile-time data) |
| 3 | `npm audit`: braces/ajv/brace-expansion vulns in build tools | High (build-env) | Pending |
| 4 | Chinese author origin — full rewrite recommended for production ATO | Critical (supply chain) | Architectural decision required |
| 5 | `<all_urls>` host permission required for core function | High | Mitigate via enterprise allowlist policy |
| 6 | No settings import validation schema | Medium | Open |
| 7 | `lib/simple_eval.ts` still exists in repo — should be removed | Low | Pending cleanup |

---

## 11. Deployment Recommendations

For DoD workstation deployment:

1. **Enterprise policy only** — distribute via managed Chrome/Firefox enterprise policy; do not publish to Web Store
2. **Allowlist domains** — use `managed_storage` or enterprise policy to restrict `<all_urls>` to known work domains
3. **Code signing** — sign the CRX package with the organization's code signing certificate
4. **STIG compliance** — verify extension does not conflict with DoD STIGs for Chrome (V-221558 through V-221627)
5. **Audit log** — consider adding a tamper-evident audit log of settings changes
6. **Rewrite recommendation** — for formal ATO, rewrite the background service worker, content scripts, and manifest from scratch; retain only the keybinding/navigation logic from this codebase
