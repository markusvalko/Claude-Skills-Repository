---
name: case-look-up-account-case-history
description: >
  Searches all historical Salesforce support cases for the same account to identify
  prior occurrences of the same or similar issue, extract solutions that worked, and
  surface recurring patterns. Results are weighted highest in the final solution
  ranking because they reflect this specific customer's environment and configuration.
  Always use this skill as Module 5 immediately after Module 4
  (case-look-up-knowledge-base-documentation-search) has completed. Also trigger when
  someone asks "has this account had this issue before", "what did we do last time for
  this customer", "check this account's case history", "has this customer reported
  this before", or "what solutions worked for this account in the past" in the context
  of a support case. This is Module 5 of the case solution ranking system.
---

# Case Look Up Account Case History — Module 5

Searches all prior Salesforce cases for the same account using the Issue
Classification Object from Module 3. This module carries the highest base weight
in Module 12 because solutions that worked for this specific customer in their
specific environment are far more likely to work again than solutions drawn from
other accounts. Even partial matches are valuable — a prior case in the same product
area for this account is a stronger signal than a perfect match from a different
account.

---

## Prerequisites

The following must be available before this module runs:
- `AccountId` from Module 1 Enriched Case Object — required
- `Case ID` of the current case — required (to exclude from results)
- Issue Classification Object from Module 3 — required for search terms
- Account Context Object from Module 2 — used for recurrence signal validation

If `AccountId` is null, skip this module entirely and note "Account Case History
unavailable — case not linked to an account."

---

## Step-by-Step Execution

### Step 1 — Pull All Account Cases

Fetch all prior cases for this account excluding the current case. Cast a wide net
at this stage — filtering happens in Step 3.

```soql
SELECT
  Id,
  CaseNumber,
  Subject,
  Description,
  Status,
  Priority,
  Type,
  CreatedDate,
  ClosedDate,
  LAN_CSAT_Score__c,
  LAN_Product_Area__c,
  LAN_Product_Version__c,
  LAN_Lansweeper_Version__c,
  LAN_Number_of_Customer_Answers__c,
  Owner.Name
FROM Case
WHERE AccountId = '{ACCOUNT_ID}'
  AND Id != '{CASE_ID}'
ORDER BY CreatedDate DESC
LIMIT 200
```

Note the total count returned. If the account has more than 200 cases, note
"Account has {total} cases — search limited to most recent 200" in the output.

---

### Step 2 — Pull Case Threads for Candidate Cases

For each case returned in Step 1 that has a Subject or Description containing
any keyword from Keyword Set A or Keyword Set B (Module 3), fetch its email thread
to extract resolution details.

Apply keyword matching to Subject + Description first to identify candidates before
fetching threads — this avoids unnecessary API calls.

```soql
SELECT
  Id,
  Subject,
  TextBody,
  FromAddress,
  Incoming,
  MessageDate
FROM EmailMessage
WHERE ParentId IN ('{CANDIDATE_CASE_IDS}')
ORDER BY MessageDate ASC
LIMIT 500
```

Also fetch internal comments for candidate cases:

```soql
SELECT
  Id,
  ParentId,
  CommentBody,
  CreatedDate,
  CreatedBy.Name,
  IsPublished
FROM CaseComment
WHERE ParentId IN ('{CANDIDATE_CASE_IDS}')
ORDER BY CreatedDate ASC
LIMIT 200
```

Group all messages back to their parent case ID for analysis in Step 3.

---

### Step 3 — Score Each Historical Case for Relevance

Evaluate every case from Step 1 against the Issue Classification Object.
Apply the scoring rubric independently for each case and sum the points.

| Criterion | Points | Condition |
|---|---|---|
| Product area exact match | +30 | `LAN_Product_Area__c` matches primary product area |
| Product area partial match | +15 | `LAN_Product_Area__c` matches a secondary product area |
| Keyword Set B match | +25 | One or more precise keywords in Subject or thread |
| Keyword Set A match | +15 | Two or more broad keywords in Subject or thread |
| Error message match | +35 | Verbatim error string appears in thread |
| Same error type | +20 | Thread describes same failure mode (bug, config, permission, etc.) |
| Same environment | +10 | Same OS, deployment type, or integration mentioned |
| Version match | +10 | Same Lansweeper version as current case |
| Older version | −10 | Case is from a significantly older version |
| CSAT 4 or 5 | +20 | Case closed with high satisfaction — solution worked |
| CSAT 1, 2, or 3 | −10 | Case closed with low satisfaction — solution may not have worked |
| Case resolved / closed | +10 | Case reached a resolution state |
| Fast resolution | +10 | ClosedDate within 3 days of CreatedDate |
| Slow resolution | −5 | ClosedDate more than 30 days after CreatedDate |
| Recent (last 12 months) | +5 | CreatedDate within the last year |

**Maximum possible score: 190**

Retain cases with a score of 25 or above. Discard cases below this threshold —
they are coincidental keyword matches, not genuine issue matches.

---

### Step 4 — Extract Resolution from Each Candidate Case

For each retained case, read its thread and extract:

**Resolution Summary** — what was the final fix or workaround? Summarise in 2–3
sentences. If the case was closed without resolution, note "Closed without
confirmed resolution."

**Resolution Steps** — if the thread contains specific steps taken, list them
as a numbered sequence. These become candidate solutions in Module 12.

**Resolution Agent** — who on the support team resolved it? (For context only —
not displayed to the end user.)

**Outcome Confirmed** — did the customer explicitly confirm the fix worked?
- `CONFIRMED` — customer confirmed resolution in thread
- `ASSUMED` — case was closed but no explicit confirmation found
- `UNRESOLVED` — case closed without fix or customer went silent

**CSAT Signal** — apply CSAT weighting from the scoring rubric. A CSAT 4 or 5
with CONFIRMED outcome is the strongest possible signal in the entire system.

---

### Step 5 — Identify Recurring Patterns

Across all retained cases (not just the top-scoring ones), look for:

**Recurring Issue Flag** — if 2 or more retained cases share the same product
area and error type, flag this as a recurring issue for this account. Note:
- How many times it has occurred
- Date range of occurrences
- Whether it was fully resolved each time or recurred after a fix

**Recurring Solution Flag** — if the same resolution step appears across 2 or
more prior cases, flag it as a proven solution for this account. This gets a
significant weighting boost in Module 12.

**Escalation History** — note whether any prior similar cases were escalated to
Jira. If so, retrieve the Jira ticket keys for cross-reference in Module 8.

```soql
SELECT
  LAN_Jira_Case_Id__c,
  LAN_Jira_Case_Status__c,
  LAN_Case__c
FROM LAN_Jira_Case_Link__c
WHERE LAN_Case__c IN ('{RETAINED_CASE_IDS}')
LIMIT 50
```

---

### Step 6 — Assemble Account Case History Object

```
ACCOUNT CASE HISTORY OBJECT
─────────────────────────────────────────────
SEARCH SUMMARY
  Account             : {Account.Name}
  Total Cases Searched: {total pulled}
  Candidate Matches   : {count above threshold}
  Search Limited      : {Yes — capped at 200 / No}

RECURRING ISSUE SIGNAL
  Recurring Issue     : {Yes / No}
  Occurrences         : {count, or "N/A"}
  Date Range          : {earliest – most recent, or "N/A"}
  Recurring Solution  : {Yes / No}
  Prior Jira Links    : {list of Jira keys, or "None"}

TOP MATCHING PRIOR CASES (ranked by score)

Rank 1 — Score: {score}/190  |  CSAT: {score or "N/A"}
  Case Number     : {CaseNumber} (link: https://lansweeper.my.salesforce.com/{Id})
  Subject         : {Subject}
  Opened          : {CreatedDate}  |  Closed: {ClosedDate or "Open"}
  Version         : {LAN_Lansweeper_Version__c or "Not recorded"}
  Owner           : {Owner.Name}
  Outcome         : {CONFIRMED / ASSUMED / UNRESOLVED}
  Resolution      : {2–3 sentence summary}
  Steps Taken     :
    1. {step}
    2. {step}
    ...
  Weighting Notes : {e.g. "CSAT 5 + confirmed resolution → highest weight"}

Rank 2 — Score: {score}/190  |  CSAT: {score or "N/A"}
  ... (repeat for all retained cases, up to 10)

NO MATCHES
  {If zero cases above threshold: "No prior cases for this account match
  the current issue classification."}
─────────────────────────────────────────────
```

---

## Output to User

```
🗂️  ACCOUNT CASE HISTORY — {Account.Name}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Cases Searched  : {total}  |  Matches Found: {count above threshold}
Recurring Issue : {Yes / No}  |  Recurring Solution: {Yes / No}

TOP MATCHES:
  #1 [Score: {score}] Case {CaseNumber} — {Subject}
     Opened: {date}  |  CSAT: {score or "N/A"}  |  Outcome: {CONFIRMED/ASSUMED/UNRESOLVED}
     Resolution: {1-sentence summary}

  #2 [Score: {score}] Case {CaseNumber} — {Subject}
     Opened: {date}  |  CSAT: {score or "N/A"}  |  Outcome: {CONFIRMED/ASSUMED/UNRESOLVED}
     Resolution: {1-sentence summary}

  ... (up to 5 shown in summary; full list in Account Case History Object)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Account case history complete. Ready for Module 6 (Global Case History).
```

If recurring issue detected:
```
🔁 RECURRING ISSUE DETECTED — This problem has occurred {count} times for
   this account ({date range}). Prior resolution steps are weighted heavily
   in the final ranking.
```

If no matches found:
```
ℹ️  No prior matching cases found for this account. This appears to be a
    first occurrence of this issue in this environment. Global case history
    (Module 6) carries full weight for solution ranking.
```

---

## How Downstream Modules Use This Object

| Module | Consumes |
|---|---|
| Module 7 — Already Tried Dedup | Resolution Steps from prior cases — cross-referenced to suppress repeats |
| Module 8 — Jira Bug Lookup | Prior Jira escalation keys — used to check if bug is already tracked |
| Module 11 — Escalation Signal | Recurring Issue Flag — repeated occurrences lower escalation threshold |
| Module 12 — Solution Synthesis | Full ranked list with scores, resolution steps, CSAT weighting |

---

## Weighting Passed to Module 12

Account case history results carry the highest base weight in the system:

| Condition | Base Weight | Notes |
|---|---|---|
| Same account match, score ≥ 100 | 40 | Highest weight in entire system |
| Same account match, score 50–99 | 28 | Strong match |
| Same account match, score 25–49 | 18 | Weak match — include with lower weight |
| CSAT 4–5 multiplier | ×1.5 | Applied to base weight |
| CSAT 1–3 multiplier | ×0.7 | Applied to base weight |
| Confirmed outcome | +8 bonus | Customer explicitly confirmed fix |
| Recurring solution | +12 bonus | Same fix worked multiple times for this account |
| Unresolved outcome | −10 penalty | Prior case closed without fix |

---

## Error Handling

| Condition | Behaviour |
|---|---|
| AccountId is null | Skip module. Note "Account Case History unavailable." |
| No prior cases exist | Note "No prior cases for this account." Pass empty object to Module 12. |
| Thread fetch fails for a candidate case | Use Subject + Description only for that case. Note partial data. |
| All prior cases are open (no resolutions) | Include with ASSUMED outcome. Note no confirmed resolutions available. |
| More than 200 cases on account | Cap at 200 most recent. Note count and cap in output. |
| Jira link query fails | Skip escalation history. Note "Prior Jira links unavailable." |

---

## Notes on Scoring Behaviour

- A case with CSAT 5, confirmed resolution, and a verbatim error message match is
  the single most valuable result in the entire solution ranking system. When this
  combination occurs, it should be prominently flagged in Module 12's output.
- Cases closed as duplicates (`Type = "Duplicate Case"`) should be excluded from
  scoring — they don't contain independent resolution information.
- If `LAN_CSAT_Score__c` is null, do not apply either the positive or negative CSAT
  modifier — treat it as neutral, not as a low score.
- Resolution steps extracted here should be written at the action level
  ("Granted the service account WMI permissions on target hosts") not the
  conversation level ("Agent told customer to check permissions") — Module 12
  needs actionable steps, not a narrative.
