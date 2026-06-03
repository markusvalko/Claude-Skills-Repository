---
name: case-look-up-escalation-signal-detection
description: >
  Evaluates whether the issue pattern — based on case history, Jira results,
  deduplication exhaustion, and customer health signals — suggests this case should
  be escalated as a new Jira bug, escalated to engineering, or flagged for CSM and
  leadership intervention rather than resolved with existing solutions. Output a
  clear escalation recommendation if the threshold is met. Always use this skill as
  Module 11 immediately after Module 10 (case-look-up-gainsight-review) has
  completed. Also trigger when someone asks "should this be escalated", "does this
  need a Jira ticket", "is this a bug we should file", "should the CSM be looped in",
  "does this need engineering involvement", or "is this case beyond normal support"
  in the context of a support case. This is Module 11 of the case solution ranking
  system.
---

# Case Look Up Escalation Signal Detection — Module 11

Reads outputs from all prior modules and evaluates whether the current case warrants
escalation beyond standard support resolution. This module does not search for new
data — it synthesises signals already collected and produces a structured escalation
recommendation that Module 12 uses to prepend the appropriate escalation guidance
to the final ranked solution output.

Escalation is not binary. This module distinguishes between four escalation types
that require different actions and different audiences.

---

## Prerequisites

All prior modules must have completed. This module reads from:
- Module 3 — Issue Classification: Error Type, Root Cause Hypothesis, Recurrence Signal
- Module 4 — KB Results: KNOWN ISSUE tags
- Module 5 — Account Case History: Recurring Issue Flag, prior Jira escalation keys
- Module 6 — Global Case History: UNRESOLVED GOLD/STRONG cases, unresolved pattern count
- Module 7 — Deduplication: Exhaustion Signal (DEEP / PARTIAL / NONE)
- Module 8 — Jira Bug Lookup: New Bug Recommendation, ACKNOWLEDGED—NO FIX tickets,
  WONT FIX tickets, open high-priority bugs
- Module 9 — Gong Call Review: Commitments Made, Churn Risk Signal
- Module 10 — Gainsight Review: Handling Posture, Churn Assessment, Renewal Risk

---

## The Four Escalation Types

Each type triggers different actions and is evaluated independently. A case can
trigger multiple types simultaneously.

---

### Type 1 — ENGINEERING ESCALATION (New Jira Bug)
**Meaning:** The issue appears to be a product defect not yet tracked in Jira.
**Action:** File a new Jira bug. Link it to the current case. Provide workaround
if available. Set customer expectation on timeline.

**Trigger conditions (ALL must be true):**
- Module 8 New Bug Recommendation = YES
- Module 3 Error Type is `Functional Bug`, `Performance`, `Compatibility`,
  or `Data Quality`
- Module 3 Root Cause Hypothesis confidence is MEDIUM or HIGH
- At least one of:
  - Module 7 Exhaustion Signal = DEEP or PARTIAL
  - 2+ UNRESOLVED GOLD/STRONG cases in Module 6 with matching issue pattern
  - Module 5 Recurring Issue Flag = RECURRING for this account

---

### Type 2 — ENGINEERING MONITOR (Existing Bug — Action Needed)
**Meaning:** A matching Jira bug exists but is stalled, deprioritised, or has
no fix timeline, and the customer impact warrants pushing for prioritisation.
**Action:** Reference the existing Jira ticket. Escalate internally to have the
ticket prioritised. Provide workaround. Communicate ETA status to customer.

**Trigger conditions (ANY one sufficient):**
- Module 8 returns an `ACKNOWLEDGED — NO FIX` ticket with a score ≥ 60
- Module 8 returns a `FIX IN PROGRESS` ticket with no ETA and the account has
  ESCALATE or CRITICAL handling posture from Module 10
- Module 8 returns a `WONT FIX` ticket AND the account ARR tier is Enterprise
  AND Module 10 handling posture is ESCALATE or CRITICAL
- A high-priority open Jira bug (P1/P2/Critical) matches the issue and has not
  moved in 90+ days

---

### Type 3 — CSM ESCALATION (Relationship Risk)
**Meaning:** The technical issue is compounded by relationship or commercial risk
that requires CS involvement beyond standard support resolution.
**Action:** Notify CSM. Flag the case to team lead. Include relationship context
in solution framing. Coordinate response with CSM before sending.

**Trigger conditions (ANY one sufficient):**
- Module 10 Handling Posture = ESCALATE or CRITICAL
- Module 10 Churn Assessment = CONFIRMED RISK
- Module 10 Renewal Risk = URGENT RENEWAL RISK or MANUAL RENEWAL AT RISK
- Module 9 Commitments Made list is non-empty AND issue relates to a commitment
- Module 9 Churn Risk Signal = Yes
- Module 5 shows 3+ prior unresolved cases for this account in the same product area
- Account ARR tier = Enterprise AND this is the 3rd+ case in 30 days

---

### Type 4 — SUPPORT LEAD ESCALATION (Case Complexity)
**Meaning:** The case has exceeded normal support scope due to depth of
troubleshooting, time invested, or complexity of the issue — regardless of
commercial risk.
**Action:** Assign to senior engineer or support lead. Do not close or auto-rotate.
Consider scheduling a technical call with the customer.

**Trigger conditions (ANY one sufficient):**
- Module 7 Exhaustion Signal = DEEP
- Module 3 Root Cause Hypothesis confidence = INSUFFICIENT after 5+ customer messages
- `LAN_Number_of_Customer_Answers__c` from Module 1 ≥ 8 with no confirmed resolution
- Case has been open for 14+ days with Status not Closed or Resolved
- Module 5 shows the same issue recurred 3+ times for this account without
  permanent resolution

---

## Step-by-Step Execution

### Step 1 — Evaluate All Four Escalation Types

Work through each type in order. For each type, check every trigger condition
against the available module outputs. Record which conditions are met.

Do not short-circuit — evaluate all four types even if Type 1 is triggered.
A case can and often should trigger multiple types (e.g. a deep technical bug
in a high-ARR account near renewal will trigger Types 1, 3, and possibly 4).

---

### Step 2 — Assess Escalation Confidence

For each triggered escalation type, assign a confidence level:

| Level | Condition |
|---|---|
| `HIGH` | 3+ trigger conditions met for that type |
| `MEDIUM` | 2 trigger conditions met |
| `LOW` | 1 trigger condition met — borderline, flag but don't force |

LOW confidence escalations are flagged as **recommendations** in the output.
MEDIUM and HIGH confidence escalations are flagged as **required actions**.

---

### Step 3 — Compute Combined Escalation Severity

Based on which types are triggered and at what confidence, assign an overall
escalation severity:

| Severity | Condition |
|---|---|
| `CRITICAL` | Type 3 or 4 triggered at HIGH confidence, OR Type 1 + Type 3 both triggered |
| `HIGH` | Any single type triggered at HIGH confidence |
| `MEDIUM` | Any type triggered at MEDIUM confidence |
| `LOW` | Only LOW confidence triggers |
| `NONE` | No escalation conditions met |

---

### Step 4 — Generate Escalation Actions

For each triggered type, produce a concrete, actionable recommendation. These
are passed directly to Module 12 for inclusion in the final output.

**Type 1 — Engineering (New Bug):**
```
ACTION: File new Jira bug
  Project     : LAN
  Issue Type  : Bug
  Summary     : {suggested summary from Module 3 problem statement}
  Description : {key symptoms, error messages, environment details from Module 3}
  Priority    : {suggested: P1 if CRITICAL health/renewal, P2 if ESCALATE, P3 otherwise}
  Affected Ver: {LAN_Lansweeper_Version__c from Module 1}
  Linked Cases: {current CaseNumber + any related cases from Module 5/6}
  Labels      : {Product Area from Module 3}
```

**Type 2 — Engineering Monitor (Existing Bug):**
```
ACTION: Escalate existing Jira ticket {LAN-KEY}
  Current Status  : {status}
  Requested Action: {prioritise / provide ETA / re-evaluate Won't Fix decision}
  Customer Impact : {ARR tier + health posture}
  Linked Cases    : {current case + any others sharing this bug}
```

**Type 3 — CSM Escalation:**
```
ACTION: Notify CSM — {CSM name or "unassigned — notify team lead"}
  Notify via  : {Slack / Email}
  Message     : "Case {CaseNumber} for {Account.Name} requires CS attention.
                 {Primary trigger reason}. Please coordinate response before
                 sending technical resolution."
  Priority    : {Urgent / High / Normal based on severity}
```

**Type 4 — Support Lead:**
```
ACTION: Assign to support lead / senior engineer
  Reason      : {Primary trigger condition}
  Recommended : {Schedule technical call / Senior engineer review / Do not auto-close}
```

---

### Step 5 — Check for Positive Signals (Downgrade Escalation)

Before finalising, check for signals that suggest escalation may not be necessary
despite trigger conditions being met. Apply these downgrade conditions:

- Module 8 found a `FIX AVAILABLE` ticket with a version match → downgrade Type 1
  (a fix exists — file the bug link but escalation is less urgent)
- Module 10 Health = HEALTHY + Module 9 Sentiment = Positive + Renewal > 180 days
  → downgrade Type 3 from REQUIRED to RECOMMENDED even if other triggers met
- Module 3 Error Type = `How-To / Usage` or `Configuration` → suppress Type 1
  (not a product defect regardless of exhaustion signal)
- `LAN_Number_of_Customer_Answers__c` < 3 → suppress Type 4 (case is too early
  to assess complexity exhaustion)

Document any downgrades applied and the reason.

---

### Step 6 — Assemble Escalation Signal Object

```
ESCALATION SIGNAL OBJECT
─────────────────────────────────────────────
OVERALL ESCALATION SEVERITY : {CRITICAL / HIGH / MEDIUM / LOW / NONE}

ESCALATION TYPES TRIGGERED

  Type 1 — Engineering (New Bug)  : {REQUIRED / RECOMMENDED / NOT TRIGGERED}
    Confidence    : {HIGH / MEDIUM / LOW / N/A}
    Conditions Met: {list of trigger conditions met}
    Downgrades    : {any downgrade conditions applied, or "None"}
    Action        : {structured Jira filing details from Step 4, or "N/A"}

  Type 2 — Engineering Monitor    : {REQUIRED / RECOMMENDED / NOT TRIGGERED}
    Confidence    : {HIGH / MEDIUM / LOW / N/A}
    Conditions Met: {list}
    Jira Ticket   : {LAN-KEY to escalate, or "N/A"}
    Action        : {escalation request details, or "N/A"}

  Type 3 — CSM Escalation         : {REQUIRED / RECOMMENDED / NOT TRIGGERED}
    Confidence    : {HIGH / MEDIUM / LOW / N/A}
    Conditions Met: {list}
    CSM           : {name or "Unassigned — escalate to team lead"}
    Action        : {notification details from Step 4, or "N/A"}

  Type 4 — Support Lead           : {REQUIRED / RECOMMENDED / NOT TRIGGERED}
    Confidence    : {HIGH / MEDIUM / LOW / N/A}
    Conditions Met: {list}
    Action        : {assignment/call recommendation, or "N/A"}

KEY SIGNALS DRIVING ESCALATION
  {Bulleted list of the 3–5 most important signals across all modules
   that contributed to the escalation assessment}

NO ESCALATION NOTE
  {If severity = NONE: "No escalation conditions met. Case is within normal
   support resolution scope. Proceed to solution ranking."}
─────────────────────────────────────────────
```

---

## Output to User

```
🚨 ESCALATION SIGNAL DETECTION — Case {CaseNumber}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Severity      : {CRITICAL / HIGH / MEDIUM / LOW / NONE}

  Type 1 — New Bug      : {REQUIRED / RECOMMENDED / NOT TRIGGERED}
  Type 2 — Existing Bug : {REQUIRED / RECOMMENDED / NOT TRIGGERED}
  Type 3 — CSM Alert    : {REQUIRED / RECOMMENDED / NOT TRIGGERED}
  Type 4 — Lead Review  : {REQUIRED / RECOMMENDED / NOT TRIGGERED}

KEY SIGNALS:
  {Bulleted list of top 3–5 driving signals}

{If any REQUIRED actions:}
REQUIRED ACTIONS:
  {Numbered list of concrete steps from Step 4}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Escalation detection complete. Ready for Module 12 (Solution Synthesis & Ranking).
```

If severity = NONE:
```
✅ No escalation required — case is within normal support resolution scope.
   Proceeding to solution ranking.
```

---

## How Downstream Modules Use This Object

| Module | Consumes |
|---|---|
| Module 12 — Solution Synthesis | Full Escalation Signal Object — prepended to final output when severity ≥ MEDIUM |
| Module 12 — Solution Synthesis | Type 1 Jira filing details — included as an action item in the engineer's output |
| Module 12 — Solution Synthesis | Type 3 CSM notification — included as a parallel action alongside technical steps |

---

## Error Handling

| Condition | Behaviour |
|---|---|
| Any upstream module object is missing | Evaluate with available data. Note which module data is absent. Do not block. |
| All module objects empty | Set severity to NONE with note "Insufficient data for escalation assessment." |
| Module 8 returned no Jira results | Type 1 and 2 evaluated from other signals only. Note absence of Jira data. |
| CSM is unassigned (Type 3 triggered) | Replace CSM notification with team lead notification. Flag unassigned CSM in output. |
| Module 10 posture is STANDARD but Type 3 conditions met from other modules | Trust the trigger conditions over the posture. Note the conflict. |

---

## Notes on Escalation Philosophy

- Escalation recommendations exist to protect both the customer and the support
  engineer. An engineer who keeps trying the same approaches on a case that needs
  engineering involvement is wasting time and eroding customer trust.
- LOW confidence escalations are shown as recommendations, not requirements.
  The engineer retains judgment on whether to act on them.
- Multiple escalation types triggering simultaneously is a strong compound signal —
  a Type 1 + Type 3 combination (product defect in a high-risk account) almost
  always warrants CRITICAL severity regardless of individual confidence levels.
- The downgrade logic in Step 5 exists to prevent alert fatigue. If a fix already
  exists and the account is healthy, a technically-triggered escalation flag is
  noise rather than signal. Suppress it.
- Escalation actions generated in Step 4 are drafts — they include all the
  information needed to act but the engineer reviews before filing or sending.
  Module 12 presents them as ready-to-use starting points, not automated actions.
