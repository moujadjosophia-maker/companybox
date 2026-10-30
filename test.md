# CompanyBox — Test Execution & Security / Code / Event Audit Report

> Canonical project copy also located at `CompanyBox/test.md` alongside `__tests__/App.test.tsx`.

## 1. Test Execution Summary

- **Test Runner**: Jest (`react-native` preset) + TypeScript (`tsc --noEmit`) + Static UTF-8 Glyph Audit
- **Test File**: `CompanyBox/__tests__/App.test.tsx`
- **Static Checks**:
  - `npx tsc --noEmit` → **PASS** (`0` errors across all 14 TypeScript/TSX modules)
  - Font-Icon / Zero-Emoji Scan → **PASS** (`0` emojis across `App.tsx` and `src/**/*`; all 26 `MaterialIcons.ttf` glyph codepoints verified in `cmap`)
- **Automated Test Results**: **8 / 8 Test Suites Passed (`100%`)** in `38.26s` (`--runInBand --verbose`)

```text
PASS __tests__/App.test.tsx (32.978 s)
  √ 1. Core v4 endpoints: voice/keyboard rename, quick quotas, and custom decimal addquota/setquota (12484 ms)
  √ 2. Core traffic shaping (512kbit / 2mbit / 10mbit), delete & single-worker restore (802 ms)
  √ 3. Firewall unblock & Restore All after blocking restores users to normal Active state (793 ms)
  √ 4. Phone Back navigation: Detail -> Previous Tab -> Double-back within 2.5s to exit (resets after timeout) (729 ms)
  √ 5. Language toggle cycles AUTO -> FR -> EN -> AUTO and auto-detects French device locale (385 ms)
  √ 6. Multilingual, phonetic sound-alike, fuzzy Levenshtein, and custom-name vocal actions (661 ms)
  √ 7. Security audit: private-LAN IPv4 allowlisting, path-traversal prevention, rename NVS sanitization, and quota bounds (173 ms)
  √ 8. Event & concurrency resilience: voice double-tap mutex, timeout cleanup, and ghost detail BackHandler recovery (873 ms)

Test Suites: 1 passed, 1 total
Tests:       8 passed, 8 total
```

### How to Re-Run Locally

```powershell
cd C:\Users\FDRkinggon\Desktop\licenceTech\CompanyBox\CompanyBox
npx tsc --noEmit
npx jest --runInBand --verbose
```

---

## 2. Core Functionalities Verified

| # | Area | Verified Behavior | Status |
|---|------|-------------------|--------|
| 1 | **Worker Rename (v4)** | Inline keyboard rename (`Save`) and per-worker voice rename send `GET /cmd/rename/<ip>/<encodedName>` with `AbortController` timeout and optimistic UI update. | PASS |
| 2 | **Quota Management (v4)** | Quick chips (`+5`, `+50`, `+500`, `+1024` MB) and `[OTHERS]` decimal input (`.` and `,`, `MB` and `GB`) accurately compute MB and call `/cmd/addquota/<ip>/<mb>` and `/cmd/setquota/<ip>/<mb>`. | PASS |
| 3 | **Traffic Control Shaping** | Segment buttons apply `512kbit` (Slow/Lent), `2mbit` (Normal), and `10mbit` (Fast/Rapide) via `/cmd/<rate>/<ip>`. | PASS |
| 4 | **Delete & Single Restore** | Deleting moves a worker to `Deleted` with a 4s `pendingDeleteRef` anti-flicker window; restoring calls `/cmd/restore/<ip>` **plus** `/cmd/rfid_add5/<ip>` so ESP32 clears `w.blocked`. | PASS |
| 5 | **Firewall Block / Unblock / Restore All** | `Unblock` dispatches `/cmd/rfid_add5/<ip>` (resetting `blocked=false`, `used_mb=0`, `GREEN`); `Restore all` calls `/cmd/restore/all` and sequentially unblocks every blocked/restored IP so all workers return to `Active`. | PASS |
| 6 | **Phone Back Navigation** | Android hardware/gesture Back navigates `WorkerDetailPage` → current Tab → previous Tab (`Deleted → Blocked → Active`) → 1st press on `Active` shows exit prompt (expires after `2500ms`) → 2nd consecutive press within `2500ms` calls `BackHandler.exitApp()`. | PASS |
| 7 | **Language Toggle (`AUTO` / `FR` / `EN`)** | Auto-detects system locale from `PhoneConfig.getWifiConfig().deviceLang` (with `Intl` fallback) and cycles `AUTO → FR → EN → AUTO`, translating all UI labels, units, empty states, and toasts. | PASS |
| 8 | **Bilingual + Phonetic/Fuzzy Voice NLP** | Recognizes English & French commands, cross-locale STT homophones (`"blog worker won"` → `block 1`, `"day block"` / `"des blocs"` / `"restaurant"` → `unblock`, `"soup reem"` / `"supreme"` → `delete`, `"rabbit"` → `10mbit`, `"long"` → `512kbit`), Levenshtein near-misses (`"debloker"`, `"suprimer"`, `"ralantir"`), voice quota (`"ajoute 50 mo"`), `"tout débloquer"`, and custom worker name targeting (`"blague Alise"` → `Alice Compta`). | PASS |

---

## 3. What the Tests & Audit Revealed (Weaknesses Found & Fixed)

During testing and static/dynamic inspection across **Security**, **Code Quality**, and **Event Functionality**, **8 concrete weaknesses** were uncovered and **fixed** in this build:

### A. Security Findings & Fixes

1. **`[SEC-01]` Unvalidated Router/Target IP Interpolation (SSRF & URL Path Traversal) — FIXED**
   - **What was revealed**: `routerIp` (from DHCP `gatewayIp`) and `targetIp` (from `/api/state` keys) were interpolated directly into `http://${routerIp}:8000/cmd/${action}/${targetIp}`. Because `AndroidManifest.xml` enables cleartext HTTP for LAN router communication, a spoofed DHCP gateway (e.g. `93.184.216.34`) could cause cleartext requests to leave the private LAN, and a malformed worker key (e.g. `192.168.10.2/../../admin`) could traverse URL paths on the router.
   - **Fix applied**: Added `isPrivateLanIpv4()` (strict RFC1918 `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` allowlist), `isValidTargetIp()`, and `SAFE_ACTION_RE` command allowlisting in `src/security.ts` and `src/hooks/useRouterController.ts`. Spoofed public gateways and non-IP targets are rejected before any socket opens.

2. **`[SEC-02]` Persistent ESP32 JSON-Corruption Injection via Worker Rename — FIXED**
   - **What was revealed**: On the ESP32 v4 firmware (`sketch_oct29a.ino`), `w.label` is a 32-byte `char[32]` saved to NVS Flash and concatenated into `buildStateJson()` after stripping only `"` — **not** stripping trailing backslashes `\`, slashes `/`, or control characters. Renaming a worker to `Admin\` would escape the closing quote in `"label":"Admin\"`, permanently corrupting `/api/state` JSON across reboots.
   - **Fix applied**: Added `sanitizeWorkerName()` (`src/security.ts`) stripping `\`, `"`, `/`, `<`, `>`, `` ` ``, and ASCII control bytes `[\u0000-\u001F\u007F]`, collapsing whitespace, and capping length to `28` characters (`maxLength={28}` on `TextInput` + sanitization on both keyboard and voice rename paths).

3. **`[SEC-03]` Partial-Numeric Bypass (`"10abc"`) & `uint64_t` Quota Overflow — FIXED**
   - **What was revealed**: `parseFloat("10abc")` silently evaluated to `10` instead of rejecting invalid characters, and entering huge GB values (e.g. `999999999999` GB) could overflow `(uint64_t)(mb * 1024 * 1024)` on the ESP32 firmware (wrapping quota around and unexpectedly blocking the worker).
   - **Fix applied**: Added `parseSafeQuotaMb()` (`src/security.ts`) enforcing strict `/^\d+(?:\.\d+)?$/` syntax and a `MAX_QUOTA_MB = 1,048,576` (`1 TB`) ceiling across both UI input and voice quota commands.

4. **`[SEC-04]` Residual Firmware-Level Advisory (Documented)**
   - The ESP32 v4 HTTP server on port `8000` does not implement TLS or token authentication by hardware/firmware design. Keep the management Wi-Fi WPA2/WPA3 password restricted to authorized operators, or add a lightweight shared token header if untrusted guests can reach port `8000`.

---

### B. Code Quality & Resilience Findings & Fixes

5. **`[CODE-01]` Dangling Poll Timer Leak & Missing Command Timeouts — FIXED**
   - **What was revealed**: In `fetchState()`, `clearTimeout(timeoutId)` was placed *after* `await fetch(...)` instead of inside `finally`, leaking an active `1800ms` timer whenever `fetch()` rejected early. Additionally, `/cmd/...` requests had no `AbortController` timeout, risking 60–75s OS TCP timeouts when the router IP was unreachable.
   - **Fix applied**: Centralized all network calls through `fetchWithTimeout()` (`src/security.ts`), which guarantees `clearTimeout(timeoutId)` inside `finally` (`1800ms` for polling, `2500ms` for commands).

6. **`[CODE-02]` Untrusted `/api/state` Schema Crash Vector — FIXED**
   - **What was revealed**: If `/api/state` returned a partial/corrupted worker object (such as `label: null` or `quota_mb: "NaN"`), downstream calls like `w.label.match(...)` or `.toFixed()` could throw runtime exceptions during render.
   - **Fix applied**: Added `sanitizeWorkersMap()` (`src/security.ts`) at the JSON ingestion boundary to normalize labels, rates, non-negative finite quota/usage numbers, booleans, and app counters.

---

### C. Event Functionality Findings & Fixes

7. **`[EVT-01]` Synchronous Double-Tap Race on Voice Recognition — FIXED**
   - **What was revealed**: Relying solely on React `listening` state allowed two rapid taps (<16ms apart, before re-render commit) on `VoiceBall` or per-worker voice-rename buttons to invoke Android `RecognizerIntent` twice concurrently, triggering `VOICE_BUSY` and prematurely resetting `listening`/`heard` while the first recognizer dialog was still active.
   - **Fix applied**: Added synchronous mutex guards (`voiceSessionActive` in `src/voiceNlp.ts` and `voiceBusyRef` in `src/hooks/useRouterController.ts`) that block duplicate taps on frame 0 and silently ignore `VOICE_BUSY` without clobbering UI state.

8. **`[EVT-02]` Ghost `selectedWorkerIp` Consuming Hardware Back Event After Disconnect — FIXED**
   - **What was revealed**: If an operator opened `WorkerDetailPage` for a worker and that worker subsequently disappeared from `/api/state`, `selectedWorker` evaluated to `null` (returning the UI to the Tabs screen), **but** `selectedWorkerIp` remained non-null. The next phone Back gesture was swallowed clearing the invisible `selectedWorkerIp` instead of moving to the previous tab.
   - **Fix applied**: Added an automatic cleanup `useEffect` in `App.tsx` that resets `selectedWorkerIp` to `null` whenever a viewed worker is no longer in `workers` or `deleted`, and gated `BackHandler` on `selectedWorkerVisibleRef.current`.

9. **`[EVT-03]` Native `PhoneConfigModule.kt` Orphaned Promise Recovery — FIXED**
   - **What was revealed**: If Android destroyed/recreated the host `Activity` while the system speech prompt was open, `voicePromise` in `PhoneConfigModule.kt` could remain non-null indefinitely and block subsequent voice commands with `VOICE_BUSY`.
   - **Fix applied**: Added `override fun invalidate()` listener/promise cleanup and a `45,000ms` stale-promise timestamp recovery in `android/app/src/main/java/com/companybox/PhoneConfigModule.kt`.

