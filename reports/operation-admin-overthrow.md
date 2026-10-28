---
title: "Operation Admin Overthrow — Findings Report"
permalink: /reports/operation-admin-overthrow/
toc: true
toc_label: "Report"
toc_sticky: true
---

*This is a formal, report-format version of the [narrative CTF writeup](/2026/10/03/operation-admin-overthrow/), structured on a standard PJPT-style pentest report template. This was a CTF, not a paid engagement — it's framed as a findings report for portfolio and practice. The target was a deliberately vulnerable event app; the "findings" are the real weakness classes each puzzle maps to.*

---

## Cover Page

- **Client Name:** Ostrich Lab — *Operation Admin Overthrow* (BSides CTF)
- **Report Title:** Capture-the-Flag Findings Report — Web & OSINT Challenge Chain
- **Date:** 2026-10-03 (testing) / 2026-10-05 (report)
- **Project #:** CTF-2026-OAO-01
- **Version:** 1.0
- **Classification:** Public (CTF writeup — no real client data)

---

## Confidentiality Statement

> This document describes a public Capture-the-Flag exercise run by Ostrich Lab at BSides. The target application (`overthrow.ostrichlab.io`) was built to be solved and contains no real customer data. Nothing here is confidential; it is published as a technical writeup. Standard confidentiality language is retained below for format fidelity but does not restrict this document.

---

## Disclaimer

> A penetration test is a snapshot in time. The findings reflect the application as it existed during the CTF window and not any changes made afterward. This was a time-boxed event exercise, not a full security evaluation, and testing was limited to the intended challenge path plus light observation of the surrounding application behavior.

---

## Contact Information

| Name | Title | Contact Info |
|---|---|---|
| Charles Leonard | Tester (CTF competitor) | — |
| Challenge author | Ostrich Lab | [ostrichlab.io](https://ostrichlab.io/) |

---

## Assessment Overview

- **Engagement dates:** 2026-10-03 (single-day CTF)
- **Standards:** None formal — CTF ruleset. No prohibition on tooling; AI assistance was used for decoding and disclosed to the organizer at submission.
- **Phases:** Planning → Discovery → Attack → Reporting
- **Result:** Full chain solved. 1st place. Final flag: `OSINT DETECTIVE`.

> **Methodology & tooling note.** Encoding/decoding and lookups (Hollerith, base32, `dig`, `curl`, `jq`) were AI-assisted. The OSINT (physical-site observation in Street View, correcting a wrong street name) and all answer submissions were performed by the tester. Disclosed at submission; CTF rules barred no tools.

---

## Assessment Components

- **Assessment type:** Black-box web application + OSINT challenge chain.
- **Description:** A five-stage linked puzzle hosted at `overthrow.ostrichlab.io`. Each stage, when solved, yields the URL path to the next. The chain emulates a mixed attacker: recognizing weak data-protection schemes (encoding mistaken for encryption), enumerating uncommon infrastructure (DNS records), performing open-source reconnaissance of a physical location, and exploiting client-side trust in a web app. No network foothold was provided or required; everything was reachable from an ordinary browser and a shell.

---

## Finding Severity Ratings

| Severity | CVSS v3 Range | Definition |
|---|---|---|
| Critical | 9.0–10.0 | Exploitation is straightforward and usually results in system-level compromise. Patch immediately. |
| High | 7.0–8.9 | Exploitation is more difficult but could cause elevated privileges, data loss, or downtime. Patch ASAP. |
| Moderate | 4.0–6.9 | Vulnerabilities exist but aren't directly exploitable or require extra steps. Patch after high-priority items. |
| Low | 0.1–3.9 | Non-exploitable, but reduces attack surface. Patch during next maintenance window. |
| Informational | N/A | No vulnerability. Notes on items observed, strong controls, or documentation. |

---

## Scope

| Assessment | Details |
|---|---|
| Web application | `overthrow.ostrichlab.io` and `geo1.overthrow.ostrichlab.io` |
| Starting artifact | A physical IBM punch card handed to competitors |

**Scope Exclusions:** No DoS, no infrastructure attacks against Cloudflare/hosting, no targeting of other competitors. Limited to the intended challenge path.

**Client Allowances:** Open CTF. Any tooling permitted. The punch card was the sole provided starting point.

---

## Executive Summary

A single physical punch card unlocked a five-stage chain that ended in full "compromise" (the final flag) without exploiting a single memory bug, injection, or server-side flaw. Every secret was either **encoded rather than encrypted**, **published in infrastructure that is rarely inspected**, or **handed directly to the browser**. The chain is a clean illustration of one theme: *obscurity and client-side trust are not access controls.*

The most serious issue (Finding F-01) is a textbook client-side authentication failure — unlock codes were shipped to the browser in a JSON file and validated in JavaScript. Any visitor could read every credential with no privileges. In a real application this is a complete authentication bypass.

### Attack Summary

| Step | Action | Recommendation |
|---|---|---|
| 1 | Decoded the punch card (Hollerith → base32) to recover a hidden URL path | Don't rely on encoding to protect resource locations (F-03) |
| 2 | Used the punch card as a Cardan grille over a word-search grid to recover the next path | — (intended puzzle mechanic; no server weakness) |
| 3 | Queried a DNS `LOC` record to recover physical coordinates, then OSINT'd the site for login credentials | Don't publish sensitive data in obscure DNS records (F-02) |
| 4 | Read hardcoded unlock codes from a client-fetched JSON file; authenticated as the target | Move secrets and auth checks server-side (F-01, Critical) |
| 5 | base32-decoded the final URL path to recover the flag | Don't rely on encoding to protect resource locations (F-03) |

### Security Strengths

- **Final validation was server-side.** Each stage posted to a server endpoint (`/wordsearchcheck`, `/geosintcheck`, `/prisoncheck`) that confirmed the answer, so flags could not be forged purely in the browser without first recovering the correct value.

### Security Weaknesses

- Secrets and authorization logic exposed to the client (root cause of F-01).
- Sensitive data published in an uncommon-but-public location — a DNS `LOC` record (F-02).
- Sensitive resource paths protected only by reversible encoding (F-03).
- No rate limiting or lockout on authentication endpoints (F-04).

---

## Vulnerability Summary & Report Card

Critical: **1**  High: **0**  Moderate: **1**  Low: **2**  Informational: **0**

| Finding | Severity | Recommendation |
|---|---|---|
| F-01 Client-Side Authentication & Hardcoded Credentials | Critical | Perform all authentication server-side; never ship secrets to the client |
| F-02 Sensitive Information Disclosure via DNS `LOC` Record | Moderate | Remove sensitive data from public DNS; audit all record types |
| F-03 Security Through Obscurity (Encoding as Access Control) | Low | Protect sensitive paths with authentication, not reversible encoding |
| F-04 Missing Rate Limiting / Lockout on Auth Endpoints | Low | Throttle and lock out repeated failed authentication attempts |

---

## Technical Findings

### Finding F-01: Client-Side Authentication & Hardcoded Credentials (Critical)

| Field | Value |
|---|---|
| **Description** | The "Jailbreak" stage (`/j5justsuebcekvcfinkesvsf`) performs its unlock-code check in client-side JavaScript against a JSON file (`prisoner_data.json`) fetched by the browser on page load. That file contains every cell's `unlock_code` in cleartext. Any unauthenticated visitor can download it and read all credentials directly — no guessing, no brute force. The subsequent `POST /prisoncheck` is server-validated, but the credential needed to pass it is already in the attacker's hands. |
| **Risk — Likelihood** | High. Requires only opening browser dev tools or a single `curl`; no skill or special access. The file is served to everyone who loads the page. |
| **Risk — Impact** | High. Full authentication bypass — an attacker can "unlock" any cell, including the target (Oscar, Cell 17). In a real system this is complete credential disclosure and auth bypass. |
| **System(s)** | `overthrow.ostrichlab.io` — `/j5justsuebcekvcfinkesvsf`, `/prisoner_data.json`, `/prisoncheck` |
| **Tools Used** | `curl`, `jq`, browser dev tools |
| **References** | OWASP A01:2021 Broken Access Control; CWE-603 Use of Client-Side Authentication; CWE-798 Use of Hard-coded Credentials |

**Evidence**

Client-side check, in page JavaScript:

```javascript
if (data.cells[cellNumber].unlock_code === unlockCode) { /* POST /prisoncheck */ }
```

Credential recovered directly from the client-fetched file:

```bash
curl -s https://overthrow.ostrichlab.io/prisoner_data.json \
  | jq '.cells | to_entries[] | select(.value.name | test("Oscar"))'
# Cell 17 — Oscar the Ostrich — unlock_code: "31337"
```

**Remediation:** Never transmit secrets to the client or perform authorization in client-side code. The browser is attacker-controlled — anything it can fetch, the attacker has, and any check it runs, the attacker can skip. Keep `unlock_code` values server-side only; have the server receive the submitted code and perform the comparison. Rule of thumb: if removing the JavaScript would expose data or bypass a control, the control was never real.

---

### Finding F-02: Sensitive Information Disclosure via DNS `LOC` Record (Moderate)

| Field | Value |
|---|---|
| **Description** | The "Rescue Recon" login (`/GEOSINTCYOPS`) hints at "LOCation." The host `geo1.overthrow.ostrichlab.io` publishes a DNS `LOC` record (RFC 1876) containing precise physical coordinates. `LOC` is a rarely-inspected record type, but it is fully public. The coordinates (`41.946083, -70.956361`) resolve to a real location (Old Colony Correctional Center, Bridgewater MA), which seeds the next stage's OSINT. |
| **Risk — Likelihood** | Moderate. Trivial to retrieve for anyone who thinks to query it (`dig LOC`), but most attackers only check `A`/`TXT` records, so it hides in plain sight. |
| **Risk — Impact** | Moderate. Information disclosure. In a real engagement, geolocation or other data planted in obscure DNS records can expose physical sites, internal naming, or infrastructure the owner assumed was private. |
| **System(s)** | `geo1.overthrow.ostrichlab.io` (DNS) |
| **Tools Used** | `dig` |
| **References** | OWASP A01:2021 / Information Disclosure; CWE-200 Exposure of Sensitive Information |

**Evidence**

```bash
dig LOC geo1.overthrow.ostrichlab.io
# 41 56 45.900 N 70 57 22.900 W 18.00m 50m 10m 10m
```

**Remediation:** Treat all DNS record types as public. Do not store sensitive location or infrastructure data in `LOC`, `TXT`, or other records on the assumption that nobody looks. Periodically enumerate your own zone across record types and remove anything that discloses more than intended.

---

### Finding F-03: Security Through Obscurity — Encoding as Access Control (Low)

| Field | Value |
|---|---|
| **Description** | Multiple stage transitions protect the next resource path using only reversible encoding, not authentication. The punch card encodes a URL in Hollerith → base32; the final stage's URL *path itself* is base32 (`j5justsuebcekvcfinkesvsf` → `OSINT DETECTIVE`). Anyone who recognizes the encoding reverses it instantly — base32's `A–Z2–7` alphabet and `=` padding are a giveaway. Encoding obscures a value; it does not protect it. |
| **Risk — Likelihood** | Low-to-Moderate. Requires recognizing and reversing an encoding — easy for a practitioner, a speed bump for others. No secret is actually needed. |
| **Risk — Impact** | Low. Exposes resource locations that were intended to be hidden. On its own it leaks paths; combined with weak downstream controls (F-01) it contributes to full compromise. |
| **System(s)** | `overthrow.ostrichlab.io` — punch-card URL, final path |
| **Tools Used** | Manual Hollerith decoding, `base32` (coreutils) |
| **References** | CWE-656 Reliance on Security Through Obscurity |

**Evidence**

```bash
echo 'NB2HI4DTHIXS633WMVZHI2DSN53S433TORZGSY3INRQWELTJN4XXO33SMRZWKYLSMNUA====' | base32 -d
# https://overthrow.ostrichlab.io/wordsearch

echo 'J5JUSTSUEBCEKVCFINKESVSF' | base32 -d
# OSINT DETECTIVE
```

**Remediation:** Protect sensitive resources with authentication and authorization, not with the assumption that their location or encoding is unknown. Encoding is a transport/representation choice, never a control. Assume any encoded value an attacker can see is already decoded.

---

### Finding F-04: Missing Rate Limiting / Lockout on Authentication Endpoints (Low)

| Field | Value |
|---|---|
| **Description** | The authentication endpoint `/geosintcheck` accepted repeated failed credential submissions with no throttling, delay, or lockout. During testing, roughly 50 failed `POST` attempts (varying the username/password) were submitted in quick succession and every one returned a normal `{"correct":false}` with no rate limiting observed. |
| **Risk — Likelihood** | High to attempt. Any attacker can script repeated submissions. |
| **Risk — Impact** | Low in this CTF (answers were OSINT-derived, not brute-forceable in practice), but in a real app the absence of lockout enables credential stuffing and brute force. Noted honestly because it was directly observed — and spraying the endpoint was the wrong approach; the answer was in the imagery, not a wordlist. |
| **System(s)** | `overthrow.ostrichlab.io` — `/geosintcheck` |
| **Tools Used** | `curl` |
| **References** | OWASP A07:2021 Identification & Authentication Failures; CWE-307 Improper Restriction of Excessive Authentication Attempts |

**Evidence:** ~50 sequential failed submissions, no throttle or lockout response.

**Remediation:** Enforce rate limiting and progressive lockout on authentication endpoints (e.g., exponential backoff, temporary lockout after N failures, CAPTCHA or IP throttling). Log and alert on bursts of failed attempts.

---

## Steps to Final Flag (Compromise Chain)

*Full-resolution version of the Attack Summary. Each step is paired with the Finding it exercised.*

1. **Punch card → URL** *(F-03).* Decoded the physical card's Hollerith punches to text, recognized base32, decoded to `https://overthrow.ostrichlab.io/wordsearch`.
2. **Cardan grille → path.** Used the punch card as a stencil over the word-search grid (title hint: "Cardan"). Card's 12 punch rows map to the grid's 12 columns; row labels 13/38 map to card columns 13–38. Brute-forced the 4 orientations; the valid one read `FIND YOUR FLAG WHEN YOU VISIT THE PATH GEOSINTCYOPS`. *(Intended puzzle mechanic — no server weakness.)*
3. **DNS `LOC` → coordinates → OSINT credentials** *(F-02).* `dig LOC geo1.overthrow.ostrichlab.io` → `41.946083, -70.956361` (Old Colony Correctional Center). Street View recon recovered the login: username **`Admin`** (Google's road label — not "Administration Road," which is why the obvious answer failed) and password **`E1113410C`** (stenciled on a shipping container on the lot).
4. **Client-side creds → authenticate** *(F-01).* Read `prisoner_data.json` from the browser, recovered Oscar's cell (**17**) and unlock code (**`31337`**), submitted, passed the server check, redirected to the final path.
5. **base32 path → flag** *(F-03).* The final URL path `j5justsuebcekvcfinkesvsf` base32-decoded to **`OSINT DETECTIVE`**.

---

*Companion piece: the [full narrative writeup](/2026/10/03/operation-admin-overthrow/) with per-step reasoning and defender notes.*
