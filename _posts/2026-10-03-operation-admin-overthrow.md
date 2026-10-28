---
title: "Operation Admin Overthrow — How I Took 1st at a BSides CTF Without Exploiting Anything"
date: 2026-10-03
categories: [ctf]
tags: [ctf, osint, web, writeup]
excerpt: "Five linked puzzles, zero exploits. Every secret was already in reach — the challenge was recognizing what I was staring at. Final flag: OSINT DETECTIVE."
toc: true
toc_label: "The chain"
---

[Ostrich Lab](https://ostrichlab.io/)'s puzzle track at BSides — *Operation Admin Overthrow*. Five linked challenges, each one coughing up the URL path to the next. The theme is "hyper-intelligent ostriches stage a coup"; the real theme is **client-side trust is a disease**. I finished the chain and took **1st**. The final flag decodes to `OSINT DETECTIVE`.

Nothing here needed an exploit. Every secret was sitting somewhere the client could already reach — encoding dressed up as encryption, coordinates stuffed in a DNS record, passwords shipped straight to the browser. The challenge wasn't breaking in. It was looking in the right place and recognizing what I was staring at.

> There's also a formal, PJPT-style **[findings report for this CTF](/reports/operation-admin-overthrow/)** — same event, written up as a professional pentest deliverable with severities, CWE/OWASP mappings, and remediation.

#### The chain at a glance

| # | Challenge | The trick | Dropped you at |
|---|-----------|-----------|----------------|
| 1 | Punch Card | Hollerith → base32 → URL | `/wordsearch` |
| 2 | Scrambled Feathers | Cardan grille (the card over the grid) | `/GEOSINTCYOPS` |
| 3 | Rescue Recon | DNS `LOC` record → Street View OSINT | `/<geourl>` |
| 4 | Jailbreak | Unlock codes shipped to the browser | `/j5justsuebcekvcfinkesvsf` |
| 5 | Final | base32 of the URL path itself | `OSINT DETECTIVE` |

---

### 1 — The Punch Card

A physical IBM 80-column card. Standard Hollerith encoding: one character per column, two zone rows up top plus rows 0–9. One hole in a column is a digit; two holes is a letter.

#### Decoding Hollerith by hand

Zone punch picks the letter group, digit punch picks the position in it:

| Upper (zone) hole | + lower hole 1–9 → |
|-------------------|--------------------|
| top row           | A–I |
| 2nd row           | J–R |
| row 0             | S–Z (digits 2–9) |

Read columns 1–72 off the card by eye. Output was all caps, digits 2–7, trailing `====`. That alphabet + that padding is **base32** wearing a trench coat. Learn to spot it on sight; it shows up twice in this CTF.

```bash
echo 'NB2HI4DTHIXS633WMVZHI2DSN53S433TORZGSY3INRQWELTJN4XXO33SMRZWKYLSMNUA====' \
  | base32 -d
# https://overthrow.ostrichlab.io/wordsearch
```

- `base32 -d` — decode, not encode. Base32 is case-insensitive and uses `A–Z` + `2–7`, which is exactly why it survives being punched into a card as uppercase.

The "Reorder number 1337" printed on the card is flavor text. Note it anyway — these people reuse numbers, and it pays off in challenge 4.

### 2 — Scrambled Feathers (the word search)

A 26×12 letter grid stuffed with decoy words — BOTNET, ZERODAY, PAYLOAD, RANSOM, OSTRICH. All noise. The title said **Cardan** Word Search, and that one word is the entire solution.

#### What a Cardan grille is

A stencil with holes. You lay it over a block of text and only the letters showing through the holes form the message. Classic steganography, pre-dates computers by centuries. **The punch card is the stencil.** The holes I decoded in step 1 have a second job.

#### Lining the card up with the grid

- Grid is 12 columns wide; the card has 12 punch rows. They match.
- Grid is 26 rows tall; the grid's row labels read **13** and **38** → that's 26 card *columns* (13 through 38).
- So each grid row maps to one card column. The holes punched in that column pick which letters in that row survive.

I didn't know the card's physical orientation (which long edge is "up"), so I didn't guess — I brute-forced all four and let English pick the winner:

```text
top→bottom, reading the 12-row from the right:
FINDYOURFLAGWHENYOUVISITTHEPATHGEOSINTCYOPS
```

→ `FIND YOUR FLAG WHEN YOU VISIT THE PATH GEOSINTCYOPS`. The "ostriches can't read the Roman alphabet" line in the brief was pure misdirection.

### 3 — Rescue Recon (the GEOSINT one)

A login page titled "System Access **LOC**ation." Username = street name, password = Container Id, format `E#######C`. The capital **L-O-C** is the tell.

#### DNS LOC records

`LOC` is a real DNS record type ([RFC 1876](https://datatracker.ietf.org/doc/html/rfc1876)) that stores physical coordinates. Almost nobody deploys it on purpose, which makes it a perfect place to hide something. Don't just query `A`/`TXT` on a target and move on — enumerate the weird record types.

```bash
dig LOC geo1.overthrow.ostrichlab.io
# 41 56 45.900 N 70 57 22.900 W 18.00m 50m 10m 10m
```

- `dig LOC <host>` — ask specifically for the LOC record. A default `dig` won't show it.

That's `41.946083, -70.956361` — the grounds of **Old Colony Correctional Center**, Bridgewater MA. A prison. On brand for "locked away somewhere in the world."

#### The part no tool does for you

I dropped the coords into Street View and used my eyes. That's the whole point of a GEOSINT challenge:

- **Username** → Google labels the road **`Admin`**. Not "Administration Road," which is what the full name resolves to — and that's exactly why the obvious answer kept getting rejected.
- **Password** → a shipping container on the lot, stenciled `E11` over `13410C` → **`E1113410C`**.

Two lessons here, both of which cost me time:

1. **Match the author's data source, not the one you prefer.** OpenStreetMap called the road "Administration Road." The challenge wanted what *Google* shows — `Admin`. When a GEOSINT answer won't take, you're probably reading the right place off the wrong map.
2. **Stop hammering the check endpoint.** When the creds bounce, the answer is in the imagery, not in a wordlist. Spraying a login form is loud, it's lazy, and on a real target the account lockout would've ended my afternoon. Fix your inputs, submit once.

### 4 — Jailbreak (Oscar the Ostrich)

A prison control panel: select a cell, enter the unlock code. The code check runs **in the browser**:

```javascript
fetch('prisoner_data.json')
  .then(r => r.json())
  .then(data => {
    if (data.cells[cellNumber].unlock_code === unlockCode) { /* POST /prisoncheck */ }
  });
```

Every prisoner's unlock code is sitting in a JSON file the page fetches on load. You don't guess it, you don't brute it. You read the file the app handed you.

```bash
curl -s https://overthrow.ostrichlab.io/prisoner_data.json \
  | jq '.cells | to_entries[] | select(.value.name | test("Oscar"))'
# Cell 17 — Oscar the Ostrich — unlock_code: "31337"
```

- `curl -s` — silent, no progress bar cluttering the pipe.
- `jq '... | select(.value.name | test("Oscar"))'` — filter the cells down to the one whose name matches "Oscar" instead of eyeballing the whole dump.

Cell **17**, code **`31337`** — eleet, and there's the `1337` callback from the punch card. Select it, submit, and the *real* server-side check (`POST /prisoncheck`) fires and redirects.

#### Defender's corner

Switching hats. This is the single most common sin in the whole chain and it's the same one that pops real apps every week:

- **Never ship the secret to the client.** `prisoner_data.json` containing unlock codes should never leave the server. The browser is the attacker's machine — anything it can fetch, the attacker has.
- **Never do the authorization check client-side.** The `if (unlock_code === ...)` in JavaScript is theater. The only check that counts is the one on the server (`/prisoncheck`). Here they had both — the client check was pure decoration, and decoration doesn't stop anyone with devtools.
- **Rule of thumb:** if removing the JavaScript would expose data or bypass a control, the control was never real.

### 5 — The final URL

The redirect landed on `/j5justsuebcekvcfinkesvsf`. The server returns the jailbreak page for *any* path (catch-all routing), so the page is a dead end — the **path string itself** is the payload. Same encoding as challenge 1; if you trained your eye in step 1 you flag it instantly.

```bash
echo 'J5JUSTSUEBCEKVCFINKESVSF' | base32 -d
# OSINT DETECTIVE
```

- Base32 is case-insensitive, so uppercasing the lowercase URL path first is cosmetic — `base32 -d` handles it either way.

Done. Oscar walks.

---

### What actually got exercised

Not exploitation. **Recognition and observation.** The skills under test:

- Spotting base32 by its alphabet and padding — twice, and the second time for free because I learned it the first.
- Reading a hint *literally* (`LOC`, `Cardan`) instead of trying to be clever about it.
- Knowing obscure primitives exist at all — DNS `LOC` records, Hollerith, Cardan grilles.
- Matching the challenge author's data source (Google's road label, not OSM's).
- Understanding that client-side code and client-fetched JSON are already in your hands the moment the page loads.

Every secret in this CTF was reachable without breaking a single thing. That's not sloppy puzzle design — **that's the lesson.** Encoding is not encryption. Client-side validation is not validation. A secret the browser can read is a secret the attacker already has.

### On using a tool to win

I used an AI assistant to decode the encodings fast. I disclosed it to the organizers at submission. I took 1st anyway — and there's no asterisk on that.

The objective was to capture the flag, and I captured it. An attacker uses whatever gets them in; that's the entire premise of the exercise, not a loophole in it. The tool decoded Hollerith and base32 in seconds — but the judgment calls, the Street View legwork, the wrong-map correction, and every single submission were mine. That's how real engagements run, too: you reach for the fastest thing that works, you document what you used, and you keep moving.

Here's the part I'll stand on: **everyone in that room could have done exactly what I did.** The encodings were recognizable. The `LOC` record was there for anyone who thought to query it. The JSON was sitting in the browser for every competitor who loaded the page. None of it was hidden from them.

I'm the one who saw it and did it. That's the whole difference — and I'll take it.
