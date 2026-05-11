# IR-2026-001 — RAPBEEF Incident Investigation Report

> **Credential Compromise, Spearphishing, and Data Exfiltration at OWL Records**
> 
> Analyst: Muhammad Essam · Handle: [0xM4R7YR](https://github.com/0xM4R7YR) · Severity: `HIGH` · Status: `CONTAINED`

---
## Investigation Report Documentation
I did a Investigation Report for that specific case as part of my learning journey, you can find it on my github in PDF format
<img width="1760" height="990" alt="Untitled design" src="https://github.com/user-attachments/assets/d4f6cbba-81eb-4775-bcb4-a60718bbcae2" />

## ⚠️ Before Reading Please Read
For Q&A styled investigation, please feel free to [check my Medium Writeup]([https://medium.com/@0xm4r7yr](https://medium.com/@0xm4r7yr/rapbeef-kc7-walkthrough-45ea11e956be)) where I solve the same investigation - Q&A Style

## Overview

This repository contains the full incident investigation report that I shared as part of my learning journey for **RAPBEEF** — a simulated two-phase cyber operation conducted against OWL Records and its primary artist, Dwake, within the [KC7 cybersecurity training platform](https://kc7cyber.com/).

The scenario was built by former Microsoft DART analysts and mirrors real-world SOC investigation methodology. All company names, individuals, IP addresses, and artifacts are entirely simulated for training purposes.

---

## The Incident — TL;DR

A threat actor, assessed with moderate confidence to be a contractor hired by Dollar Currency Records (DCR), compromised OWL Records through two sequential phases requiring **zero malware and zero zero-day exploits**.

| Metric | Finding |
|---|---|
| Operation duration | April 10 – April 30, 2024 (20 days) |
| Initial access method | KBA bypass via OSINT from public rap lyrics |
| Accounts compromised | 2 — corporate email + Instagram |
| Phishing emails delivered | 13 (role-targeted: Rappers only) |
| Time-to-click (spearphish) | 56 minutes after delivery |
| Time-to-auth (post-click) | 60 minutes after click |
| Data exfiltrated | `DwakesDirtySecrets.zip` from Drafts folder |
| Overall severity | **HIGH** |

---

## Attack Chain Summary
### Detailed OWL Records Attack Chain Graph Analysis
<img width="1952" height="1426" alt="image" src="https://github.com/user-attachments/assets/3d56cc90-894c-4e1e-ba53-c8dc78778db1" />

### Phase 1 — Reconnaissance and Account Takeover

1. **Web Reconnaissance** — 19 HTTP requests to OWL Records' website from attacker IP `18.66.52.227`, methodically converging on Dwake's artist profile and the password reset flow.
2. **OSINT via Rap Lyrics** — Two lines from Dwake's public diss track directly answered two KBA security questions:
   - *"Got the Washington name from my mom's side, son"* → Mother's maiden name: **Washington**
   - *"Used to play with little Fluffy, now I'm runnin' with the wolves"* → Childhood pet: **Fluffy**
3. **KBA Bypass & Email Compromise** — Password reset submitted with harvested answers. Full inbox control achieved at `10:57` on April 10th.
4. **Instagram Takeover** — "Forgot password" flow routed recovery to the now-controlled inbox. Instagram fell in the same session. Embarrassing image posted the following day.

### Phase 2 — Phishing Campaign

1. **PassiveDNS Pivot** — Attacker IP resolved to phishing domain `betterlyrics4u[.]com` via PassiveDNS historical records.
2. **Campaign Scope** — 13 emails delivered from two sender addresses, exclusively targeting employees with the **Rapper** job title.
3. **Psychological Timing** — Dwake's spearphish arrived **five days** after the Instagram incident — long enough for humiliation to settle, short enough that it was still raw.
4. **Click & Credential Harvest** — Dwake clicked the malicious link at `12:03:12Z` on April 15th, 56 minutes after delivery. The attacker authenticated with stolen credentials 60 minutes later.
5. **Mailbox Recon & Exfiltration** — Systematic keyword searches (`scandal`, `confidential`, `leak`, `secret`, `sensitive`, `meeting_notes`) across April 16th. On April 30th at `14:23:02Z`, `DwakesDirtySecrets.zip` was exfiltrated from Dwake's Drafts folder.

---

## MITRE ATT&CK Mapping

| Technique | ID |
|---|---|
| Gather Victim Identity Info via Public Media | T1589 |
| Modify Authentication Process — Security Questions | T1556 |
| Compromise Accounts — Email Accounts | T1586.002 |
| Phishing — Spearphishing Link | T1566.002 |
| Valid Accounts | T1078 |
| Email Collection — Local | T1114 |
| Exfiltration Over Web Service | T1567 |
| Search Open Technical Databases — Passive DNS | T1596.001 |

---

## Indicators of Compromise (IOCs)

| Indicator | Type | Context |
|---|---|---|
| `18.66.52[.]227` | IP | Attacker exit node — recon, auth, phishing infra |
| `betterlyrics4u[.]com` | Domain | Phishing domain — PassiveDNS pivot |
| `ghostwritersanonymous@protonmail.com` | Email | Primary sender — 11 phishing emails |
| `wemakebeatz@gmail.com` | Email | Secondary sender — 2 phishing emails |
| `.../share/online/published/enter` | URL | Credential harvesting page |
| `dwake_audrey@owl-records.com` | Email | Compromised account |
| `10.10.0.5` | IP | Dwake's internal IP — click confirmed |
| `DwakesDirtySecrets.zip` | File | Exfiltrated archive — 30/04/2024 14:23Z |
| `8f8d7baf48abc18667315891d4c6a507` | Hash | Password hash captured in auth log |

---

## Key Investigative Decisions

**Why start with `InboundNetworkEvents` instead of `AuthenticationEvents`?**  
The investigation was intel-led, triggered by a human intelligence tip providing the attacker IP. At the moment of the tip, the attacker had not yet touched the authentication layer. Starting from `AuthenticationEvents` — the default for alert-driven triage — would have missed Phase 1 entirely. Matching your starting log source to the current attack phase is a core investigative discipline.

**Why did PassiveDNS unlock everything?**  
A single confirmed IOC (the attacker IP) was queried against PassiveDNS historical records, resolving to `betterlyrics4u[.]com`. From that single domain, the full 13-email phishing campaign was reconstructed. One pivot. Two phases exposed.

**What made this attack succeed?**  
No malware. No exploit. The attacker walked through doors that were already open:
- Deprecated KBA authentication (NIST SP 800-63B deprecated it in 2017)
- An artist with no security training and public OSINT exposure via his own lyrics
- An email security filter that returned `CLEAN` on a role-targeted spearphish
- No behavioral monitoring inside authenticated mailbox sessions

---

## Defensive Recommendations

**Immediate**
- Eliminate KBA security questions. Replace with FIDO2/passkeys or hardware MFA. This single change removes the entire Phase 1 attack vector.
- Block newly registered domains at the email gateway (domain age ≤ 30 days would have blocked delivery entirely).
- Flag outbound HTTP (non-HTTPS) link clicks from internal IPs.

**Behavioral Monitoring**
- Alert on bulk keyword searches (`scandal`, `confidential`, `leak`) within a single authenticated mailbox session — this pattern is not normal user behavior.
- Alert on password reset events for high-value accounts.

**Organizational**
- Security policy failure caused this incident, not user failure. The artist cannot be blamed for a mechanism the organization chose.
- Public-facing staff (artists, executives) require bespoke OPSEC briefings — standard security awareness training does not account for elevated OSINT exposure.

---

## KQL Query Reference

```kql
// Q1 — Attacker IP Reconnaissance
InboundNetworkEvents
| where timestamp between (datetime("2024-04-10T00:00:00") .. datetime("2024-04-11T00:00:00"))
| where src_ip has "18.66.52.227"

// Q2 — KBA Bypass Confirmation
InboundNetworkEvents
| where url has_any("washington", "fluffy")
| where src_ip has "18.66.52.227"

// Q3 — PassiveDNS Pivot
PassiveDns
| where ip == "18.66.52.227"

// Q4 — Full Phishing Campaign Scope
Email
| where link has "betterlyrics4u.com"

// Q5 — Targeted Employee Role Identification
let _targets = Email
| where link has "betterlyrics4u.com"
| distinct recipient;
Employees
| where email_addr in (_targets)

// Q6 — Click Confirmation
OutboundNetworkEvents
| where url == "http://betterlyrics4u.com/share/online/published/enter"
| where src_ip == "10.10.0.5"

// Q7 — Attacker Authentication
AuthenticationEvents
| where username == "dwaudrey"
| where src_ip == "18.66.52.227"

// Q8 — Post-Compromise Mailbox Activity
InboundNetworkEvents
| where timestamp between (datetime("2024-04-12T00:00:00") .. datetime("2024-05-01T00:00:00"))
| where url has "dwaudrey"
| where src_ip has "18.66.52.227"
```

---

## About This Report

This investigation was completed on the [KC7 cybersecurity training platform](https://kc7cyber.com/) — a free, browser-based SOC simulation environment built by former Microsoft DART analysts. The scenario uses real-world log sources (KQL queries against `InboundNetworkEvents`, `OutboundNetworkEvents`, `Email`, `PassiveDns`, `AuthenticationEvents`, `Employees`) and mirrors production incident investigation methodology.

If you have any comments or suggestions please feel free to contact me

Published as part of Muhammad Essam's public cybersecurity portfolio.

---

*Written by Muhammad Essam · 0xM4R7YR · IR-2026-001*
