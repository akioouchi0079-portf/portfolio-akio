# Browser Forensics: Comparing Data Recoverability Across Firefox, Brave, and Microsoft Edge

**Type:** Academic digital forensics project
**Environment:** Kali Linux
**Tools:** DB Browser for SQLite, foremost, strings, grep, jq, browser DevTools

---

## Overview

Digital investigators frequently need to reconstruct a user's browsing activity from artifacts left behind on disk — cookies, history, and cache. But not all browsers store this data the same way, and the privacy features built into modern browsers actively shape what's recoverable.

This project compared three browsers — **Firefox**, **Brave**, and **Microsoft Edge** — across three dimensions:

1. Where and how each browser stores cookie and history data
2. How each browser's built-in tracking-protection feature works, and what it does to evidence
3. How accessible (or encrypted/obscured) each browser's cache data is to an investigator

The goal was to understand the practical trade-off between **user privacy** and **forensic accessibility** — and to translate that into concrete recommendations for a business.

---

## Methodology

For each browser, the same three-step process was applied:

1. **Locate storage**: identify the on-disk file paths for cookie, history, and cache data (Firefox profile directory vs. Chromium `User Data` structure)
2. **Extract and inspect**: use SQLite to directly query the underlying databases (`Cookies.sqlite` / `places.sqlite` for Firefox; `Cookies` / `History` SQLite DBs for Brave and Edge), and use browser DevTools to cross-reference what's visible from the UI
3. **Analyze tracking-protection internals**: inspect preference files (`prefs.js`, Chromium `Preferences` JSON) using `grep` and `jq` to find configuration keys governing tracking protection, and compare against documented behavior

Cache files were additionally probed with `strings` (Firefox) and `foremost` (Brave/Edge) to attempt recovery of embedded media and page remnants.

---

## Key Findings

### Data storage location

| Browser | Cookie storage | History storage | Format |
|---|---|---|---|
| Firefox | `~/.mozilla/firefox/<profile>/cookies.sqlite` | `places.sqlite` | SQLite, plainly queryable |
| Brave | `~/.config/BraveSoftware/Brave-Browser/Default/Cookies` | `History` | SQLite (Chromium schema) |
| Microsoft Edge | `~/.config/microsoft-edge/Default/Cookies` | `History` | SQLite (Chromium schema) |

All three store cookies and history in SQLite databases that could be directly queried with standard SQL (e.g., `SELECT url, title, visit_count FROM moz_places` / `FROM urls`), making history reconstruction straightforward across all three browsers.

### Tracking protection architecture

- **Firefox — Enhanced Tracking Protection**: blocklist-based (disconnect.me lists), per-site cookie isolation, and automatic purging of "bounce tracker" cookies after a short inactivity window. Configuration was fully visible in `prefs.js` via `grep -i "privacy"` / `grep -i "tracking"`.
- **Brave — Shields**: EasyList/EasyPrivacy filter lists, third-party cookie blocking by default, fingerprinting mitigation, and automatic HTTPS upgrading. Aggregate blocking stats were recoverable via `jq '.brave' Preferences`, but detailed per-site tracking-prevention state was not present in the expected preference keys.
- **Microsoft Edge — Tracking Prevention**: three-tier system (Basic/Balanced/Strict) using a heuristic approach to first-party/third-party relationship detection. Blocked-tracker counts and per-site status were visible in the UI, but — like Brave — the corresponding preference file lacked a clear `privacy` key; only an unrelated `privacy_sandbox` key was present.

**Notable gap**: both Chromium-based browsers (Brave, Edge) obscure their tracking-protection configuration from plaintext preference files in a way Firefox does not. This is a finding worth investigating further — it's likely stored in a separate protected preferences file, a hashed/HMAC-verified store, or a local database rather than plain JSON, and would be a good next step for anyone extending this work.

### Cache accessibility

- **Firefox**: cache entries under `cache2/entries/` were directly readable with `strings`, yielding recoverable page/video remnants (e.g., a cached YouTube page).
- **Brave & Edge**: cache files were not human-readable with `strings`; `foremost` file-carving was attempted but failed to complete in a reasonable time even with elevated permissions. DevTools' Cache Storage inspector was used as a practical workaround, surfacing HTTP response metadata (content type, length, cache timestamp) even where raw file carving failed.

---

## Business Security Recommendations

Based on the technical findings, the following controls were recommended for organizational browser security:

- Enforce automatic browser updates (patches close the majority of known vulnerabilities)
- Set tracking-protection to the strictest available mode (Firefox: Strict / Brave: Shields Up / Edge: Strict)
- Configure auto-clear of cookies and history on browser exit for sensitive roles
- Restrict extensions to a vetted allowlist
- Deploy a password manager to reduce credential-reuse risk — a large share of breaches still trace back to compromised credentials

---

## Reflection & Next Steps

The most useful technical skill built during this project was direct SQLite querying of live browser databases — a transferable skill for any forensic or incident-response context involving endpoint artifacts. The tooling failures (foremost timing out, permission issues) were as instructive as the successes: they highlighted the practical unreliability of generic carving tools against modern, densely packed cache formats, and the value of having a fallback method (DevTools inspection) ready.

If extended, this project would benefit from:
- Locating the actual storage mechanism for Chromium tracking-protection state (likely `Secure Preferences` or a separate local store)
- Testing cache carving with a longer timeout and instrumented profiling to determine why `foremost` failed to complete
- Extending the comparison to mobile browser variants, which store data very differently
