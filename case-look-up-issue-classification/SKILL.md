---
name: case-look-up-issue-classification
description: >
  Analyses the full case thread from the Module 1 Enriched Case Object and produces
  a structured issue classification — including problem statement, product area, error
  type, symptoms, root cause hypothesis, and what has already been attempted. This
  structured output is the query engine for all downstream search modules (KB lookup,
  case history search, Jira matching). Always use this skill as Module 3 immediately
  after Module 2 (case-look-up-account-context) has completed. Also trigger when
  someone asks "what is the issue on this case", "classify this case", "what is the
  customer's problem", "summarise the issue", or "what has already been tried" in the
  context of a support case. This is Module 3 of the case solution ranking system.
---

# Case Look Up Issue Classification — Module 3

Reads the full case thread from the Module 1 Enriched Case Object and uses Claude
to extract a precise, structured problem statement. This classification object is
what all downstream search modules query against — sloppy classification here means
noisy results everywhere downstream. The goal is to produce a signal-rich, jargon-
accurate description of the issue that can be matched against KB articles, past cases,
and Jira bugs with high precision.

---

## Prerequisites

Module 1 (look-up-case-and-enrich) must have completed. This module reads:
- Full case thread (emails, comments, feed) — required
- `Subject` — used as a classification seed
- `Description` — used as a classification seed
- `LAN_Product_Area__c` — pre-existing tag, used to validate or override
- `LAN_Product_Version__c` / `LAN_Lansweeper_Version__c` — version context
- Account Context Object from Module 2 — optional but used to enrich classification
  with recurring product area signals

If the case thread is empty and `Description` is also null, note "Insufficient case
content for classification" and produce a best-effort classification from `Subject`
only.

---

## Step-by-Step Execution

### Step 1 — Prepare the Classification Input

Assemble the following into a single input block for analysis:

```
CLASSIFICATION INPUT
Subject       : {Subject}
Description   : {Description}
Product Area  : {LAN_Product_Area__c or "Not set"}
Version       : {LAN_Product_Version__c or LAN_Lansweeper_Version__c or "Unknown"}
Thread        : {Full merged case thread from Module 1, chronological}
```

The thread should include all `[EMAIL-IN]`, `[EMAIL-OUT]`, `[COMMENT-INTERNAL]`,
`[COMMENT-PUBLIC]`, and `[FEED]` entries in order.

---

### Step 2 — Extract the Issue Classification

Analyse the full input and extract each of the following fields. For each field,
derive from evidence in the thread — do not guess or infer beyond what the thread
supports. If a field cannot be determined, mark it as `"Undetermined"`.

---

#### 2a. Problem Statement (Primary)
A single sentence, written from the customer's perspective, describing the core
issue. Be specific — include the component, action, and failure mode if visible.

**Good:** "Asset scanning fails to complete on Windows Server 2022 targets when
the Lansweeper service account lacks WMI permissions."

**Avoid:** "Customer has a scanning issue."

---

#### 2b. Problem Statement (Technical)
A second sentence written for a support engineer, using technical terminology,
describing the likely failure mechanism based on thread evidence.

---

#### 2c. Product Area
The primary product area affected. Validate against `LAN_Product_Area__c` if set —
if the thread contradicts the field value, use the thread and note the discrepancy.

Common Lansweeper product areas (non-exhaustive):
`Asset Scanning` · `Software Inventory` · `User Scanning` · `Network Discovery` ·
`Reporting` · `Dashboards` · `Deployment` · `Integrations` · `Cloud Scanning` ·
`Active Directory` · `Licensing` · `API` · `Performance` · `Installation / Upgrade` ·
`Access Control` · `Alerts` · `Relations`

---

#### 2d. Secondary Product Areas
Any additional product areas touched by the issue. May be empty.

---

#### 2e. Error Type
Classify the failure mode:

| Type | Description |
|---|---|
| `Functional Bug` | Feature not working as documented |
| `Performance` | Feature works but is slow, hanging, or resource-heavy |
| `Configuration` | Customer environment or settings issue |
| `Permission / Access` | Rights, credentials, or entitlement problem |
| `Compatibility` | Version, OS, or integration conflict |
| `Data Quality` | Incorrect, missing, or stale data in results |
| `Installation / Upgrade` | Setup, migration, or version upgrade failure |
| `Integration` | Third-party or API connection failure |
| `How-To / Usage` | Customer needs guidance, not a fix |
| `Known Bug` | Issue already matches a known Jira bug |
| `Undetermined` | Cannot classify from available thread |

---

#### 2f. Symptoms
A bullet list of observable symptoms described or implied in the thread. Each
symptom should be specific and verifiable (something an engineer could reproduce
or check). Maximum 8 symptoms.

---

#### 2g. Error Messages / Codes
Any exact error messages, codes, log excerpts, or status codes mentioned in the
thread. Quote these verbatim — they are high-signal for KB and Jira matching.

---

#### 2h. Environment Details
Extract any environment facts mentioned in the thread:

- OS / platform (Windows version, Linux distro, cloud provider)
- Lansweeper version (exact build if mentioned)
- Deployment type (on-premise, cloud, hybrid)
- Database type (SQL Server, PostgreSQL, etc.)
- Network topology mentions (domain, workgroup, VPN, firewall)
- Integration targets (AD, Azure, ServiceNow, etc.)
- Number of assets / scan targets if mentioned

---

#### 2i. What Has Already Been Tried
A bullet list of every solution, workaround, or troubleshooting step already
attempted — either by the customer or by the support engineer in this thread.
This list is passed to Module 7 (Already Tried Deduplication) and Module 12
(Solution Synthesis) to suppress already-attempted solutions from the ranked output.

Be thorough — include partial attempts and things that were tried but not confirmed
as failed. Mark each entry as:
- `[CUSTOMER]` — attempted by the customer before or during the case
- `[AGENT]` — suggested or performed by the support agent in this thread
- `[CONFIRMED FAILED]` — explicitly stated not to have worked
- `[OUTCOME UNKNOWN]` — attempted but result not confirmed in thread

---

#### 2j. Root Cause Hypothesis
Based on the thread evidence, what is the most likely root cause? This is a
hypothesis, not a diagnosis. State confidence level:
- `HIGH` — thread provides strong, consistent evidence pointing to one cause
- `MEDIUM` — thread suggests a likely cause but evidence is partial
- `LOW` — thread is ambiguous; multiple causes are plausible
- `INSUFFICIENT` — not enough information to hypothesise

---

#### 2k. Recurrence Signal
Based on the Module 2 Account Context Object (`product_areas_seen`,
`cases_last_30_days`), assess whether this appears to be a recurring issue
for this account:

- `RECURRING` — same product area seen in 2+ cases in last 30 days
- `POSSIBLY RECURRING` — same product area seen once before in last 30 days
- `FIRST OCCURRENCE` — no prior cases in same product area
- `UNKNOWN` — Module 2 data not available

---

#### 2l. Search Keywords
Generate two sets of search keywords derived from the classification above.
These are passed directly to downstream search modules.

**Keyword Set A — Broad (for KB and case history search):**
5–8 keywords or short phrases capturing the product area, error type, and
primary symptom. Prefer terms a support engineer would use when searching.

**Keyword Set B — Precise (for Jira bug matching):**
3–5 highly specific terms — error codes, component names, exact failure actions.
These should narrow to the specific defect rather than the general problem area.

---

### Step 3 — Confidence Assessment

Assess overall classification confidence based on thread quality:

| Level | Condition |
|---|---|
| `HIGH` | Thread has 3+ customer messages, clear symptoms, at least one error message |
| `MEDIUM` | Thread has 1–2 messages or symptoms but no error messages |
| `LOW` | Thread is a single message or description only |
| `MINIMAL` | Subject line only — no thread content available |

Note: LOW or MINIMAL confidence classifications should be flagged to the user
so they know downstream search results may be broader and less precise.

---

### Step 4 — Assemble the Issue Classification Object

```
ISSUE CLASSIFICATION OBJECT
─────────────────────────────────────────────
CLASSIFICATION CONFIDENCE   : {HIGH / MEDIUM / LOW / MINIMAL}

PROBLEM STATEMENT
  Customer View   : {2b — customer-facing sentence}
  Technical View  : {2c — engineer-facing sentence}

CLASSIFICATION
  Product Area    : {2d — primary}
  Secondary Areas : {2e — list, or "None"}
  Error Type      : {2f}
  Recurrence      : {2l — RECURRING / POSSIBLY RECURRING / FIRST OCCURRENCE / UNKNOWN}

ROOT CAUSE HYPOTHESIS
  Hypothesis      : {2j — description}
  Confidence      : {HIGH / MEDIUM / LOW / INSUFFICIENT}

SYMPTOMS
  {Bulleted list from 2g}

ERROR MESSAGES / CODES
  {Verbatim excerpts from 2h, or "None found"}

ENVIRONMENT
  OS / Platform   : {from 2i}
  Version         : {from 2i}
  Deployment Type : {from 2i}
  Database        : {from 2i}
  Network Notes   : {from 2i}
  Integrations    : {from 2i}
  Asset Count     : {from 2i, or "Not mentioned"}

ALREADY TRIED
  {Bulleted list from 2k with [CUSTOMER] / [AGENT] / [CONFIRMED FAILED] /
   [OUTCOME UNKNOWN] labels, or "Nothing attempted yet"}

SEARCH KEYWORDS
  Keyword Set A (Broad)   : {comma-separated list}
  Keyword Set B (Precise) : {comma-separated list}
─────────────────────────────────────────────
```

---

## Output to User

Present a clean classification summary after assembling the object:

```
🔍 ISSUE CLASSIFICATION — Case {CaseNumber}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Issue       : {Customer-facing problem statement}
Product Area: {Primary product area}
Error Type  : {Error type}
Recurrence  : {Recurrence signal}
Confidence  : {Classification confidence}

Root Cause Hypothesis ({confidence level}):
{Hypothesis text}

Already Tried ({count} items):
{Bulleted list with labels}

Search Keywords:
  Broad   → {Keyword Set A}
  Precise → {Keyword Set B}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Issue classification complete. Ready for Module 4 (KB / Documentation Search).
```

If classification confidence is LOW or MINIMAL, add:
```
⚠️  Low classification confidence — downstream search results will be broader.
    Consider asking the customer for more detail before proceeding.
```

---

## How Downstream Modules Use This Object

| Module | Consumes |
|---|---|
| Module 4 — KB Search | Keyword Set A, Product Area, Error Type |
| Module 5 — Account Case History | Product Area, Keyword Set A, Recurrence Signal |
| Module 6 — Global Case History | Keyword Set A + B, Product Area, Error Type, Version |
| Module 7 — Already Tried Dedup | Already Tried list (full, with labels) |
| Module 8 — Jira Bug Lookup | Keyword Set B, Error Messages, Product Area |
| Module 11 — Escalation Signal | Error Type, Root Cause Hypothesis, Recurrence Signal |
| Module 12 — Solution Synthesis | Full classification object for context and framing |

---

## Error Handling

| Condition | Behaviour |
|---|---|
| Thread is empty, Description is null | Classify from Subject only. Set confidence to MINIMAL. |
| Product area cannot be determined | Set to "Undetermined". Do not guess. |
| No error messages in thread | Set Error Messages to "None found". Do not fabricate. |
| Already Tried list is empty | Set to "Nothing attempted yet" — valid state for new cases. |
| Module 2 data unavailable | Set Recurrence Signal to UNKNOWN. Do not block classification. |
| Thread is very long (50+ messages) | Summarise older messages; analyse the most recent 20 in full detail. Always read the first 3 messages and last 10 messages in full regardless of total length. |
