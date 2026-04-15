# Security Scan Report: `windows-vs2022-new` branch

**Date:** 2026-04-10
**Verdict:** No malicious code, backdoors, or data exfiltration found. This branch is a Windows/VS2022 port of an existing screensaver app. There are several security issues to address (most pre-existing).

---

## CRITICAL

| # | Issue | Location | New in this branch? |
|---|-------|----------|---------------------|
| 1 | **TLS certificate validation completely disabled** — `CURLOPT_SSL_VERIFYHOST` and `CURLOPT_SSL_VERIFYPEER` both set to 0. All HTTPS traffic is vulnerable to MITM. Tracked in [e-dream-ai/client#546](https://github.com/e-dream-ai/client/issues/546). | `Networking.cpp:311-313` | No (pre-existing) |
| 2 | **OpenSSL 1.0.2k (Jan 2017)** — EOL since Dec 2019, years of unpatched CVEs. Only affects Windows build. Tracked in [e-dream-ai/client#551](https://github.com/e-dream-ai/client/issues/551). | `openssl-1.0.2k/` | No (pre-existing) |

## HIGH

| # | Issue | Location | New in this branch? |
|---|-------|----------|---------------------|
| 3 | **Server-controlled URL passed to ShellExecuteA** — `frontendUrl` from server JSON is opened without scheme validation. A compromised server (trivial given #1) could execute arbitrary schemes. Triggerable remotely via WebSocket `"web"` event. | `client.h:2376-2378`, `PlatformUtils_win.cpp:251` | Yes |
| 4 | **Buffer overflow** — `strcpy` into 2048-byte static buffer with only a debug-mode ASSERT guard. | `RendererDD.cpp:87` | No (pre-existing) |
| 5 | **Mixed allocator UB** — `realloc()` on `_aligned_malloc()` memory (must use `_aligned_realloc` on Windows). Heap corruption risk. | `AlignedBuffer.cpp:148` | No (pre-existing) |
| 6 | **Pointer truncation on Win64** — pointer cast to `unsigned long` (32-bit on Windows LLP64) corrupts addresses above 4GB. | `AlignedBuffer.cpp:200` | No (pre-existing) |

## MEDIUM

| # | Issue | Location |
|---|-------|----------|
| 7 | No null-checks on WebSocket message `dynamic_pointer_cast` — malformed server messages crash the client. | `EDreamClient.cpp:2195+` |
| 8 | Session token sent as query parameter (logged by servers/proxies). | `EDreamClient.cpp:2494` |
| 9 | MD5 used for content integrity (collision-prone, especially with TLS disabled). | `PlatformUtils_win.cpp:352` |
| 10 | `string_view::data()` passed to Win32 API with no null-termination guarantee. | `PlatformUtils_win.cpp:251` |

## LOW / HYGIENE

| # | Issue | Location |
|---|-------|----------|
| 11 | **Debug .exe committed to repo** (`e-dreamd.exe`) — not malicious, but can't be audited and bloats the repo. Should `git rm --cached`. | `RuntimeMSVC/e-dreamd.exe` |
| 12 | NSIS installer writes DLLs to `$WINDIR` (legacy pattern, may trigger AV). | `nsis_installer.nsi:87-98` |
| 13 | `usleep()` implementation takes microseconds but passes directly to `Sleep()` (milliseconds) — sleeps 1000x too long. | `msvc_fix.cpp:27` |
| 14 | 115 vendored binary files (boost, ffmpeg, openssl libs) — not verifiable against upstream checksums. | Various |

## What this branch changes

The diff is clean in intent — thread-safety fixes (mutex in PlaylistManager), improved error handling (CacheManager), build tooling (VS2022 .vcxproj, build.py, release.py), D3D11 display code, socket.io integration, and settings UI. No obfuscated code, no hidden endpoints, no credential theft, no unauthorized persistence mechanisms. All server connections go to `*.infinidream.ai` domains.

## Recommendations before merging

1. **Fix #3 now** (the one HIGH issue new to this branch) — validate `frontendUrl` starts with `https://` before passing to `ShellExecuteA`
2. **Plan to fix #1 and #2** — TLS and OpenSSL are the biggest real-world risks but are pre-existing
3. **Remove the debug .exe** from git tracking
4. The rest can be tracked as tech debt
