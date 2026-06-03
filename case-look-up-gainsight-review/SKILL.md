---
name: case-look-up-gainsight-review
description: >
  Pulls account health, churn prediction, NPS, onboarding scores, CS comments, and
  renewal signals from Salesforce to build a complete customer success picture for
  the case. This data does not change which solutions are ranked but fundamentally
  changes how urgently they are acted on and how they are presented. Always use this
  skill as Module 10 immediately after Module 9 (case-look-up-gong-call-review) has
  completed. Also trigger when someone asks "what is the health score for this
  account", "what does CS say about this customer", "is this customer at risk",
  "check the churn score", "what is the NPS for this account", or "give me the
  customer success view on this case" in the context of a support case. This is
  Module 10 of the case solution ranking system.
---

# Case Look Up Gainsight Review — Module 10

Pulls customer success signals from Salesforce Account fields to build a health and
risk picture for the account that opened the case. Lansweeper syncs key CS data
directly onto the Account object, so this module reads from Salesforce rather than
requiring a separate Gainsight connector.

This module does not rank solutions. It produces a **handling posture** — a
clear directive for Module 12 on how urgently to act, how to frame the response,
and whether any escalation or CSM notification is warranted alongside the technical
solution.

---

## Prerequisites

The following must be available before this module runs:
- `AccountId` from Module 1 Enriched Case Object — required
- `Account.Name` from Module 1 — for display
- Account Context Object from Module 2 — provides baseline ARR, renewal date,
  and urgency signals already computed; this module enriches and extends those signals
- Gong Call Context Object from Module 9 — provides sentiment and churn risk signals
  from calls to cross-reference with health data

If `AccountId` is null, skip this module and note "Gainsight review unavailable —
case not linked to an account."

---

## Step-by-Step Execution

### Step 1 — Fetch Extended Health and CS Fields

Fetch all customer success and health fields from the Account record. Several of
these were not pulled in Module 2's extended account query.

```soql
SELECT
  Id,
  Name,
  LAN_Health_Score_Number__c,
  LAN_Health_Score_Comments__c,
  LAN_Health_Score_Reason__c,
  LAN_Churn_Prediction_Score__c,
  LAN_NPS_Score__c,
  LAN_Onboarding_Survey_Score__c,
  LAN_Onboarding_Survey_Comments__c,
  LAN_Onboarding_Survey_Completed__c,
  LAN_Customer_Success_Manager__c,
  LAN_CS_Comments__c,
  LAN_Renewal_Date__c,
  LAN_Auto_Renew__c,
  LAN_Active_ARR__c,
  LAN_Account_Status__c,
  LAN_Annual_Business_Review_Last_Complete__c,
  LAN_Steady_State_Calls__c,
  LAN_Technical_Context__c,
  LAN_Tags__c,
  LAN_No_Customer_Support__c,
  LAN_Remove_Auto_Case_Closure__c
FROM Account
WHERE Id = '{ACCOUNT_ID}'
LIMIT 1
```

---

### Step 2 — Fetch Recent CS Activity from Account Feed

Pull recent internal activity posts, notes, and CS updates logged on the account:

```soql
SELECT
  Id,
  Type,
  Body,
  CreatedDate,
  CreatedBy.Name
FROM AccountFeed
WHERE ParentId = '{ACCOUNT_ID}'
  AND Type IN ('TextPost', 'CreateRecordEvent')
  AND CreatedDate = LAST_N_DAYS:90
ORDER BY CreatedDate DESC
LIMIT 20
```

These posts often contain CSM notes, escalation history, and context that enriches
the CS picture beyond what structured fields alone provide.

---

### Step 3 — Fetch Open Escalations and Tasks

Check for open tasks or activities assigned to the account that may indicate
ongoing CS engagement or escalation:

```soql
SELECT
  Id,
  Subject,
  Status,
  Priority,
  ActivityDate,
  Description,
  Owner.Name
FROM Task
WHERE WhatId = '{ACCOUNT_ID}'
  AND Status NOT IN ('Completed', 'Deferred')
  AND ActivityDate >= TODAY
ORDER BY ActivityDate ASC
LIMIT 10
```

---

### Step 4 — Interpret Health Score

Interpret `LAN_Health_Score_Number__c` using the following scale. If the field
is null, derive a best-effort health assessment from the signals available.

| Score Range | Health Label | Implication |
|---|---|---|
| 80–100 | 🟢 HEALTHY | Standard handling. No urgency escalation from health alone. |
| 60–79 | 🟡 MODERATE | Elevated attention warranted. Timely resolution important. |
| 40–59 | 🟠 AT RISK | Proactive CSM loop-in recommended alongside technical resolution. |
| 0–39 | 🔴 CRITICAL | Immediate escalation path required. CSM must be notified. |
| null | ⬜ UNKNOWN | Assess from churn score, NPS, and CS comments instead. |

If the score is null, derive health label from:
1. `LAN_Churn_Prediction_Score__c` — high churn score → lower health
2. `LAN_NPS_Score__c` — NPS < 6 → AT RISK signal; NPS < 4 → CRITICAL signal
3. `LAN_CS_Comments__c` — read for explicit risk language
4. `LAN_Account_Status__c` — flag any non-standard status values
5. Module 9 sentiment — Frustrated or At-Risk → lower health assumption

---

### Step 5 — Interpret Churn Prediction Score

Interpret `LAN_Churn_Prediction_Score__c` if populated:

| Score Range | Churn Risk Label |
|---|---|
| 0–20 | LOW — minimal churn indicators |
| 21–50 | MODERATE — some signals present |
| 51–75 | ELEVATED — meaningful churn risk |
| 76–100 | HIGH — strong churn indicators |
| null | UNKNOWN |

Cross-reference churn prediction with Module 9 sentiment signal:
- High churn score + Frustrated/At-Risk sentiment from calls → **CONFIRMED RISK**
- High churn score + Neutral/Positive sentiment → **MONITOR — data vs. tone mismatch**
- Low churn score + Frustrated sentiment from calls → **EMERGING RISK — calls more current than score**
- Low churn score + Positive sentiment → **STABLE**

---

### Step 6 — Assess Renewal Risk

Combine renewal date (from Module 2) with health and churn signals:

| Condition | Renewal Risk |
|---|---|
| Renewal ≤ 90 days + Health AT RISK or CRITICAL | 🔴 URGENT RENEWAL RISK |
| Renewal ≤ 90 days + Health MODERATE | 🟡 ELEVATED RENEWAL RISK |
| Renewal ≤ 90 days + Health HEALTHY | 🟢 RENEWAL IMMINENT — healthy |
| Renewal 91–180 days + Health AT RISK or CRITICAL | 🟡 RENEWAL WATCH |
| Renewal > 180 days or unknown | Standard — no renewal urgency |
| `LAN_Auto_Renew__c = false` + AT RISK/CRITICAL health | 🔴 MANUAL RENEWAL AT RISK |

---

### Step 7 — Review CS Comments and Context Fields

Read the following fields for qualitative context that enriches the handling posture:

**`LAN_CS_Comments__c`** — CSM's free-text notes on the account. May contain
recent escalation history, customer behaviour patterns, known sensitivities,
or relationship context. Extract any note directly relevant to the current case
or to how the response should be framed.

**`LAN_Health_Score_Reason__c`** — the stated reason for the current health score.
If this references technical issues in the same product area as the current case,
flag it as a compounding signal.

**`LAN_Health_Score_Comments__c`** — additional health commentary. Read for any
specific mentions of support experience or product friction.

**`LAN_Technical_Context__c`** — technical environment notes recorded by the CS
or SA team. May contain environment details not captured in the case thread.
Pass any new details to Module 12 as supplementary environment context.

**`LAN_Onboarding_Survey_Comments__c`** — if populated, provides early signal
on the customer's relationship with the product and support team.

**`LAN_Tags__c`** — account tags that may indicate special handling requirements
(e.g. strategic account, at-risk, early adopter, beta customer).

**`LAN_No_Customer_Support__c`** — if true, flag prominently. This account has
a specific support configuration that may affect how the case is handled.

---

### Step 8 — Derive Handling Posture

Synthesise all signals into a single **Handling Posture** directive for Module 12.
This is the primary output of this module.

| Posture | Conditions | Module 12 Behaviour |
|---|---|---|
| `STANDARD` | Healthy + no renewal urgency + no churn risk | Provide ranked solutions. Standard tone. |
| `PRIORITISE` | Moderate health OR elevated renewal risk OR moderate churn | Provide ranked solutions. Note time-sensitivity. Recommend prompt follow-up. |
| `ESCALATE` | AT RISK health OR high churn OR urgent renewal risk | Provide ranked solutions AND escalation path. Loop in CSM. Flag to team lead. |
| `CRITICAL` | CRITICAL health OR confirmed risk + renewal ≤ 90 days | Solutions secondary to relationship triage. Immediate CSM and leadership notification. |

Handling Posture is set to the highest level triggered by any single signal.

---

### Step 9 — Assemble Gainsight Review Object

```
GAINSIGHT REVIEW OBJECT
─────────────────────────────────────────────
HEALTH SIGNALS
  Health Score          : {LAN_Health_Score_Number__c or "Not set"}
  Health Label          : {🟢 HEALTHY / 🟡 MODERATE / 🟠 AT RISK / 🔴 CRITICAL / ⬜ UNKNOWN}
  Health Reason         : {LAN_Health_Score_Reason__c or "Not recorded"}
  Health Comments       : {LAN_Health_Score_Comments__c or "None"}
  Churn Prediction      : {LAN_Churn_Prediction_Score__c or "Not set"}
  Churn Risk Label      : {LOW / MODERATE / ELEVATED / HIGH / UNKNOWN}
  Churn Assessment      : {CONFIRMED RISK / MONITOR / EMERGING RISK / STABLE}
  NPS Score             : {LAN_NPS_Score__c or "Not available"}
  Onboarding Score      : {LAN_Onboarding_Survey_Score__c or "Not available"}
  Onboarding Comments   : {LAN_Onboarding_Survey_Comments__c or "None"}

RENEWAL SIGNALS
  Renewal Date          : {LAN_Renewal_Date__c or "Unknown"}
  Days to Renewal       : {calculated or "Unknown"}
  Auto Renew            : {LAN_Auto_Renew__c or "Unknown"}
  Renewal Risk          : {label from Step 6}

CS CONTEXT
  CSM                   : {LAN_Customer_Success_Manager__c or "Not assigned"}
  Account Status        : {LAN_Account_Status__c or "Standard"}
  Last ABR              : {LAN_Annual_Business_Review_Last_Complete__c or "None recorded"}
  Steady State Calls    : {LAN_Steady_State_Calls__c or "Not set"}
  CS Comments           : {LAN_CS_Comments__c or "None"}
  Technical Context     : {LAN_Technical_Context__c or "None"}
  Account Tags          : {LAN_Tags__c or "None"}
  No Support Flag       : {LAN_No_Customer_Support__c — YES/NO}
  Auto Case Closure Off : {LAN_Remove_Auto_Case_Closure__c — YES/NO}

RECENT ACCOUNT FEED (last 90 days)
  {Date — Author: Summary of post, or "No recent activity"}

OPEN TASKS
  {Subject | Due Date | Owner | Priority, or "No open tasks"}

CROSS-REFERENCE WITH MODULE 9
  Call Sentiment        : {from Gong Call Context Object}
  Combined Assessment   : {CONFIRMED RISK / MONITOR / EMERGING RISK / STABLE}

SUPPLEMENTARY ENVIRONMENT DETAILS
  {Any new technical context from LAN_Technical_Context__c not in Module 3, or "None"}

HANDLING POSTURE
  Posture               : {STANDARD / PRIORITISE / ESCALATE / CRITICAL}
  Primary Signal        : {The single highest-severity signal that drove the posture}
  CSM Notification      : {Required / Recommended / Not required}
  Escalation Path       : {Description of recommended escalation action, or "None"}
─────────────────────────────────────────────
```

---

## Output to User

```
🏥 GAINSIGHT REVIEW — {Account.Name}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Health Score    : {score or "N/A"}  ({health label})
Churn Score     : {score or "N/A"}  ({churn risk label})
NPS             : {score or "N/A"}
Renewal         : {date} ({days} days)  |  Auto-Renew: {Yes/No/Unknown}
Renewal Risk    : {label}
CSM             : {name or "⚠️ Not assigned"}

Combined Risk   : {CONFIRMED RISK / MONITOR / EMERGING RISK / STABLE}
Health Reason   : {LAN_Health_Score_Reason__c or "Not recorded"}

CS Comments     :
  {LAN_CS_Comments__c truncated to 3 sentences, or "None"}

⚡ HANDLING POSTURE : {STANDARD / PRIORITISE / ESCALATE / CRITICAL}
   Primary Signal   : {driving signal}
   CSM Notification : {Required / Recommended / Not required}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Gainsight review complete. Ready for Module 11 (Escalation Signal Detection).
```

If ESCALATE or CRITICAL posture triggered:
```
🚨 {ESCALATE / CRITICAL} POSTURE — {primary signal description}.
   CSM ({CSM name or "unassigned"}) should be notified alongside
   the technical resolution. Module 12 will include escalation
   guidance in the final output.
```

---

## How Downstream Modules Use This Object

| Module | Consumes |
|---|---|
| Module 11 — Escalation Signal | Handling Posture, Churn Assessment, Renewal Risk — compound escalation triggers |
| Module 12 — Solution Synthesis | Handling Posture → tone, urgency, CSM notification flag, escalation path |
| Module 12 — Solution Synthesis | Supplementary environment details from `LAN_Technical_Context__c` |

---

## Error Handling

| Condition | Behaviour |
|---|---|
| AccountId is null | Skip module. Note "Gainsight review unavailable." |
| All health fields null | Derive posture from available signals (churn, NPS, Module 9 sentiment). Note "Health score not set — assessment based on available signals." |
| `LAN_Health_Score_Number__c` null but CS Comments populated | Read comments for explicit risk language and derive label manually |
| AccountFeed query returns no results | Note "No recent CS activity recorded." Continue. |
| Task query returns no results | Note "No open tasks." Continue. |
| `LAN_No_Customer_Support__c` is true | Flag prominently in output. Add note to Module 12: "Verify support entitlement before proceeding." |
| CSM not assigned | Note "⚠️ CSM not assigned" in output. If posture is ESCALATE or CRITICAL, flag to team lead instead. |

---

## Notes on Field Interpretation

- `LAN_Health_Score_Number__c` is the primary health signal. If it conflicts with
  `LAN_Churn_Prediction_Score__c` or Module 9 sentiment, note the conflict rather
  than silently resolving it — Module 12 should be aware of mixed signals.
- `LAN_CS_Comments__c` is a free-text field maintained by the CSM. It may be
  outdated — check `LastModifiedDate` on the Account record if recency matters.
- `LAN_Technical_Context__c` may contain environment details more current than
  what is in the case thread — always pass new details found here to Module 12
  as supplementary context.
- `LAN_Tags__c` may contain pipe-separated or comma-separated values. Parse and
  list individually rather than displaying as a raw string.
- `LAN_Steady_State_Calls__c` indicates whether this account has regular cadence
  calls with the CS team. If false or null for an AT RISK account, this is an
  additional risk signal — the account may be disengaged.
- Do not surface health scores, churn scores, or CS comments in any customer-facing
  output from Module 12. This data is for internal handling guidance only.
