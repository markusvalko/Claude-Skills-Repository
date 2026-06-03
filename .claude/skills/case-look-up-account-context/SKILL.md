---
name: case-look-up-account-context
description: >
  Pulls deep account context for a Salesforce support case — including ARR tier,
  CSM ownership, renewal date, industry, company size, open opportunities, and
  account health signals — to inform solution urgency and presentation in the case
  solution ranking system. Always use this skill as Module 2 immediately after
  Module 1 (look-up-case-and-enrich) has produced an Enriched Case Object. Also
  trigger when someone asks "who is the CSM for this account", "what's the ARR on
  this case", "how important is this customer", "what's the renewal date", or
  "give me background on this account" in the context of a support case. This is
  Module 2 of the case solution ranking system.
---

# Case Look Up Account Context — Module 2

Uses the `AccountId` from the Module 1 Enriched Case Object to pull comprehensive
account context. This module does not change which solutions are ranked — it informs
**how urgently** they should be acted on and **how they should be presented** to the
customer. High-ARR accounts near renewal require a different handling posture than
low-ARR accounts with no open opportunities.

---

## Prerequisites

Module 1 (look-up-case-and-enrich) must have run first. This module reads:
- `AccountId` — required for all queries
- `Account.LAN_Active_ARR__c` — already fetched in Module 1; used here for tier logic
- `Account.Name` — for display

If `AccountId` is null, skip this module and note "Account context unavailable —
case is not linked to an account" in the session output.

---

## Step-by-Step Execution

Run Steps 1–4 in parallel for efficiency.

---

### Step 1 — Pull Extended Account Fields

Fetch fields not already captured in Module 1:

```soql
SELECT
  Id,
  Name,
  Type,
  Industry,
  BillingCountry,
  BillingCity,
  NumberOfEmployees,
  Website,
  OwnerId,
  Owner.Name,
  LAN_Active_ARR__c,
  LAN_Total_Account_ARR__c,
  LAN_CSM__c,
  LAN_CSM__r.Name,
  LAN_Renewal_Date__c,
  LAN_Contract_Start_Date__c,
  LAN_Customer_Since__c,
  LAN_Account_Tier__c,
  LAN_Health_Score__c,
  LAN_NPS_Score__c,
  LAN_Support_Tier__c,
  LAN_Technical_Account_Manager__c,
  LAN_Technical_Account_Manager__r.Name
FROM Account
WHERE Id = '{ACCOUNT_ID}'
LIMIT 1
```

> **Field notes:**
> - `LAN_CSM__c` / `LAN_CSM__r.Name` — Customer Success Manager. Key contact for
>   escalation and renewal context.
> - `LAN_Renewal_Date__c` — renewal date. Cases within 90 days of renewal are flagged
>   as high urgency regardless of ARR tier.
> - `LAN_Account_Tier__c` — if populated, use this as the primary tier signal.
>   If null, derive tier from ARR using the tier logic below.
> - `LAN_Health_Score__c` — account health score if available. Low health + open case
>   is a compounding risk signal.
> - `LAN_Support_Tier__c` — support entitlement level (e.g. Standard, Premium, Enterprise).
>   Affects SLA expectations and escalation thresholds.
> - `LAN_Technical_Account_Manager__c` — TAM if assigned. Note alongside CSM in output.
> - `LAN_NPS_Score__c` — most recent NPS score. Informs tone of solution presentation.

---

### Step 2 — Pull Open Opportunities

Fetch open pipeline for this account to surface renewal and expansion risk:

```soql
SELECT
  Id,
  Name,
  Amount,
  StageName,
  CloseDate,
  Type,
  OwnerId,
  Owner.Name
FROM Opportunity
WHERE AccountId = '{ACCOUNT_ID}'
  AND StageName NOT IN ('Closed Won', 'Closed Lost')
  AND Amount != null
ORDER BY CloseDate ASC
LIMIT 10
```

Sum total open pipeline amount across all returned opportunities. Flag any opportunity
closing within 90 days as a near-term risk signal.

---

### Step 3 — Pull Account's Recent Case History (30 days)

Fetch case volume over the past 30 days to establish whether this is a high-frequency
support account:

```soql
SELECT
  Id,
  CaseNumber,
  Subject,
  Status,
  Priority,
  CreatedDate,
  ClosedDate,
  LAN_CSAT_Score__c,
  LAN_Product_Area__c,
  LAN_Product_Version__c,
  Owner.Name
FROM Case
WHERE AccountId = '{ACCOUNT_ID}'
  AND CreatedDate = LAST_N_DAYS:30
  AND Id != '{CASE_ID}'
ORDER BY CreatedDate DESC
LIMIT 25
```

Compute:
- `cases_last_30_days` — total count
- `avg_csat_last_30_days` — average of non-null CSAT scores
- `open_cases_count` — count where Status not in ('Closed', 'Resolved')
- `product_areas_seen` — distinct list of `LAN_Product_Area__c` values

---

### Step 4 — Pull All-Time Case Statistics

```soql
SELECT
  COUNT(Id) total_cases_all_time,
  MIN(CreatedDate) first_case_date
FROM Case
WHERE AccountId = '{ACCOUNT_ID}'
  AND Id != '{CASE_ID}'
```

This tells downstream modules whether this account is a long-term frequent contact
or a relatively new/infrequent support user.

---

### Step 5 — Derive ARR Tier and Urgency Signals

Use the following logic to assign an ARR tier if `LAN_Account_Tier__c` is null:

| Active ARR | Tier |
|---|---|
| ≥ $100,000 | Enterprise |
| $25,000 – $99,999 | Mid-Market |
| $5,000 – $24,999 | SMB |
| < $5,000 or null | Low / Unknown |

Then evaluate urgency signals:

| Signal | Condition | Flag |
|---|---|---|
| Renewal imminent | `LAN_Renewal_Date__c` within 90 days | 🔴 HIGH |
| Renewal imminent | `LAN_Renewal_Date__c` within 180 days | 🟡 MEDIUM |
| Large open pipeline | Open opportunity Amount > $50,000 | 🔴 HIGH |
| Low health score | `LAN_Health_Score__c` < 50 (if 0–100 scale) | 🔴 HIGH |
| Low NPS | `LAN_NPS_Score__c` < 7 | 🟡 MEDIUM |
| High case frequency | `cases_last_30_days` > 10 | 🟡 MEDIUM |
| Multiple open cases | `open_cases_count` > 3 | 🟡 MEDIUM |
| Enterprise tier | ARR Tier = Enterprise | Escalate faster |
| No CSM assigned | `LAN_CSM__r.Name` is null | ⚠️ NOTE |

Collect all triggered flags into an `urgency_signals` list. The overall urgency level
is the highest flag triggered:
- Any 🔴 HIGH signal → **URGENT**
- Only 🟡 MEDIUM signals → **ELEVATED**
- No signals → **STANDARD**

---

### Step 6 — Assemble Account Context Object

```
ACCOUNT CONTEXT OBJECT
─────────────────────────────────────────────
ACCOUNT IDENTITY
  Account Name        : {Name}
  Account ID          : {Id}
  Type                : {Type}
  Industry            : {Industry}
  Location            : {BillingCity}, {BillingCountry}
  Company Size        : {NumberOfEmployees} employees
  Customer Since      : {LAN_Customer_Since__c or first_case_date or "Unknown"}
  Website             : {Website}

CONTRACT & REVENUE
  Active ARR          : ${LAN_Active_ARR__c}
  Total ARR           : ${LAN_Total_Account_ARR__c}
  ARR Tier            : {Derived tier or LAN_Account_Tier__c}
  Support Tier        : {LAN_Support_Tier__c or "Not set"}
  Contract Start      : {LAN_Contract_Start_Date__c or "Unknown"}
  Renewal Date        : {LAN_Renewal_Date__c or "Unknown"}
  Days to Renewal     : {Calculated or "Unknown"}

OPEN PIPELINE
  Total Open Opps     : {count}
  Total Open Amount   : ${sum}
  Nearest Close Date  : {earliest CloseDate}
  {For each opp: Name | Stage | Amount | Close Date | Owner}

ACCOUNT HEALTH
  Health Score        : {LAN_Health_Score__c or "Not available"}
  NPS Score           : {LAN_NPS_Score__c or "Not available"}

PEOPLE
  Account Owner       : {Owner.Name}
  CSM                 : {LAN_CSM__r.Name or "⚠️ Not assigned"}
  TAM                 : {LAN_Technical_Account_Manager__r.Name or "Not assigned"}

CASE HISTORY
  All-Time Cases      : {total_cases_all_time}
  First Case Date     : {first_case_date}
  Cases Last 30 Days  : {cases_last_30_days}
  Currently Open      : {open_cases_count}
  Avg CSAT (30d)      : {avg_csat_last_30_days or "N/A"}
  Product Areas (30d) : {product_areas_seen}

URGENCY ASSESSMENT
  Overall Urgency     : {URGENT / ELEVATED / STANDARD}
  Signals Triggered   : {List of urgency_signals, or "None"}
─────────────────────────────────────────────
```

---

## Output to User

Present a concise account brief after assembling the context object:

```
🏢 ACCOUNT CONTEXT — {Account.Name}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Tier        : {ARR Tier}  |  Active ARR: ${LAN_Active_ARR__c}
Support     : {LAN_Support_Tier__c or "Standard"}
Renewal     : {LAN_Renewal_Date__c} ({days to renewal} days)
CSM         : {LAN_CSM__r.Name or "⚠️ Unassigned"}
Health      : {LAN_Health_Score__c or "N/A"}  |  NPS: {LAN_NPS_Score__c or "N/A"}

Open Pipeline : ${total open opp amount} across {count} opportunities
Case History  : {total_cases_all_time} all-time  |  {cases_last_30_days} in last 30 days
               {open_cases_count} currently open

⚡ Urgency     : {URGENT / ELEVATED / STANDARD}
   Signals     : {urgency_signals list, or "None triggered"}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Account context complete. Ready for Module 3 (Issue Classification).
```

---

## How Downstream Modules Use This Data

| Module | Uses From This Object |
|---|---|
| Module 3 — Issue Classification | `product_areas_seen`, `cases_last_30_days` for recurring issue detection |
| Module 5 — Account Case History | `total_cases_all_time`, `open_cases_count` as baseline |
| Module 9 — Solution Synthesis | `urgency_level`, `urgency_signals` to set solution presentation tone |
| Module 9 — Solution Synthesis | `LAN_Renewal_Date__c`, `LAN_Health_Score__c` to flag handling priority |
| Module 9 — Solution Synthesis | `avg_csat_last_30_days` to calibrate communication style |

---

## Error Handling

| Condition | Behaviour |
|---|---|
| AccountId is null | Skip module entirely. Note in session: "No account linked — account context unavailable." |
| `LAN_Renewal_Date__c` is null | Set days_to_renewal to "Unknown". Do not trigger renewal urgency flags. |
| `LAN_Health_Score__c` is null | Skip health score signal. Note "Health score not available." |
| No open opportunities found | Set pipeline to $0 / 0 opps. Do not trigger pipeline urgency flag. |
| `LAN_CSM__r.Name` is null | Flag as "⚠️ CSM not assigned" — this is a notable gap, not a data error. |
| All urgency fields null | Set urgency to STANDARD with note "Insufficient data to assess urgency." |

---

## Notes on Field Quirks

- `LAN_Renewal_Date__c` may not be populated for all accounts. If null and the account
  has `LAN_Contract_Start_Date__c`, do not attempt to infer renewal date — flag as
  unknown rather than guess.
- `LAN_Health_Score__c` scale may vary — if the value is above 1 and below 100 treat
  as a 0–100 scale; if between 0 and 1 treat as a decimal (multiply by 100 for display).
- `NumberOfEmployees` may be null for some accounts — omit the field rather than
  displaying zero.
- Open opportunity `Amount` may include renewal amounts — do not treat all open pipeline
  as net-new. Use for urgency signalling only, not financial reporting.
