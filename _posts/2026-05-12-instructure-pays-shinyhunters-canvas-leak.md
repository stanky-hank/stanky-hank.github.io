---
title: "Instructure Pays ShinyHunters to Stop a 3.65TB Canvas Leak"
date: 2026-05-12
categories: [breaches]
tags: [breach, ransomware, data-exfiltration]
excerpt: "275 million records walked out the door, and the 'proof of deletion' is worth nothing."
---

Instructure — the parent company of Canvas LMS — was breached by ShinyHunters and paid a ransom to stop the publication of 3.65TB of stolen data. It's a textbook case of what "we'll handle security later" actually costs, so it's worth walking through.

## What happened

The attackers got initial access by exploiting a vulnerability in the **Free-for-Teacher** environment's support-ticket system. From there they exfiltrated roughly **275 million records** spanning usernames, emails, course names, enrollment data, and user messages — across about **9,000 organizations**, including schools and universities.

Instructure initially believed the breach was contained. Then, on May 7, a second wave of unauthorized activity hit and **~330 institutional Canvas portals were defaced** with extortion messages. With a May 12 ransom deadline looming, Instructure negotiated an agreement, got the data "back," and obtained what it called "digital confirmation of data destruction."

## The part that's meaningless

That phrase — *digital confirmation of data destruction* — is worth sitting with, because it's worth nothing.

**There is no technical mechanism that verifies a threat actor deleted their copies of exfiltrated data.** None. A criminal who just extorted you saying "it's gone, pinky swear" is not evidence; it's a receipt for a payment. Instructure's own hedge says it plainly: "there is never complete certainty when dealing with cyber criminals." Correct. So the 275 million records should be treated as permanently in the wild, regardless of what was paid.

## Where the real damage lands

The downstream risk here is **phishing**, and it's severe. 275 million records with names, emails, enrollment status, and message history hand attackers everything they need to impersonate school administrators, IT support, or financial-aid offices in follow-on attacks. As Halcyon put it:

> Leaked records can be used to impersonate school administrators, IT support, or financial aid offices in follow-on attacks. Students, parents, and personnel at affected institutions should be considered [compromised], and institutions should issue phishing advisories and direct communications immediately.

The people carrying that risk — students, parents, staff — had no say in which platform held their data. That decision was made for them, and the due diligence that was supposed to protect them clearly wasn't there.

## The math nobody wants to do

Add it up: ransom paid (amount undisclosed), remediation work, reputational damage, and every affected institution now needing to issue phishing advisories. Instructure has also said it's now going to "improve its cybersecurity posture" and run a comprehensive review — which raises the obvious question of why that wasn't deemed necessary *before* 275 million records walked out the door.

Every time one of these hits, the story is the same: the cost of doing security *proactively* was always smaller than this. It just didn't show up on a quarterly report, so it lost the argument.

---

*Source: [The Hacker News — Instructure Reaches Ransom Agreement with ShinyHunters to Stop 3.65TB Canvas Leak](https://thehackernews.com/2026/05/instructure-reaches-ransom-agreement.html). Analysis is my own.*
