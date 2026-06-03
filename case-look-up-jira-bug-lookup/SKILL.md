---
name: case-look-up-jira-bug-lookup
description: >
  Searches Jira for bugs, known issues, and feature requests that match the classified
  issue from Module 3, cross-references prior escalation history from Modules 5 and 6,
  and determines whether the issue is a known defect, has a fix available, or should
  be escalated as a new bug. Results fundamentally change the nature of the solution
  recommendation — a known bug shifts the response from troubleshooting to workaround
  or fix ETA. Always use this skill as Module 8 immediately after Module 7
  (case-look-up-already-tried-deduplication) has completed. Also trigger when someone
  asks "is there a Jira for this", "is this a known bug", "has this been escalated",
  "check Jira for this issue", "is there a fix coming", or "should we file a bug" in
  the context of a support case. This is Module 8 of the case solution ranking system.
---

# Case Look Up Jira Bug Lookup — Module 8

Searches Jira using the Issue Classification Object from Module 3 and prior
escalation signals from Modules 5 and 6. A Jira match is a fundamentally different
signal from a case history match — it means the issue has been recognised at the
engineering level. When a matching Jira ticket exists, the recommended solution
shifts from troubleshooting steps to one of three outcomes: apply an available fix,
use a documented workaround, or set expectations around a fix ETA.

This module also determines whether the current case warrants filing a *new* Jira
ticket — a recommendation that is passed to Module 11 (Escalation Signal Detection)
for final evaluation.

---

## Prerequisites

The following must be available before this module runs:
- Issue Classification Object from Module 3 — provides Keyword Sets A and B,
  Product Area, Error Type, and verbatim Error Messages
- Account Case History Object from Module 5 — provides Prior Jira escalation keys
  from this account's prior cases
- Global Case History Object from Module 6 — provides case IDs of unresolved
  high-signal cases that may point to an unfiled bug
- Linked Jira Tickets from Module 1 — tickets already linked to the current case
  (these are reviewed but not re-searched)

---

## Step-by-Step Execution

### Step 1 — Review Already-Linked Jira Tickets

Before searching, check the Linked Jira Tickets list from Module 1. If tickets
are already linked to this case, retrieve their current details via the Atlassian
MCP connector:

For each linked ticket key (e.g. `LAN-12345`):
```
Fetch issue: LAN-{key}
Fields: summary, status, priority, assignee, reporter, created, updated,
        fixVersions, affectedVersions, labels, description, comments (last 5)
```

Classify each linked ticket:
- **RESOLVED WITH FIX** — status is Done/Released and a fix version is specified
- **IN PROGRESS** — actively being worked on
- **OPEN / BACKLOG** — acknowledged but not yet scheduled
- **WONT FIX / CLOSED** — engineering decided not to address
- **DUPLICATE** — merged into another ticket (retrieve the parent)

Note: even if tickets are already linked, continue with Steps 2–4 to find
*additional* related bugs the current case may not yet be linked to.

---

### Step 2 — Build Jira Search Queries

Construct JQL queries using the Issue Classification Object. Use the Atlassian
MCP connector to execute searches.

**Query A — Product area + keyword search (broad):**
```jql
project = LAN
AND issuetype in (Bug, "Known Issue")
AND (summary ~ "{keyword_1}" OR summary ~ "{keyword_2}"
  OR description ~ "{keyword_1}" OR description ~ "{keyword_2}")
AND status not in (Done, Resolved, Closed, "Won't Fix", Duplicate)
ORDER BY priority ASC, updated DESC
```
Use top 2 keywords from Keyword Set A. Run one query per keyword pair.

**Query B — Precise keyword search:**
```jql
project = LAN
AND issuetype in (Bug, "Known Issue", Improvement)
AND (summary ~ "{keyword_set_b_combined}"
  OR description ~ "{keyword_set_b_combined}")
ORDER BY priority ASC, updated DESC
```
Use all Keyword Set B terms combined. This targets the specific defect
rather than the general problem area.

**Query C — Verbatim error message search:**
Only run if `Error Messages / Codes` is populated in the Classification Object.
```jql
project = LAN
AND (summary ~ "{verbatim_error_string}"
  OR description ~ "{verbatim_error_string}")
ORDER BY updated DESC
```
Quoted exact strings are the highest-precision Jira query available.

**Query D — Prior escalation keys from Module 5:**
If the Account Case History Object contains prior Jira escalation keys, fetch
those tickets directly regardless of their current status — even resolved
tickets are relevant context.
```jql
issue in ({LAN-key1}, {LAN-key2}, ...)
```

**Query E — Resolved bugs in same product area (fix reference):**
Search for recently resolved bugs in the same product area — these may contain
fix instructions or version information relevant to the current case.
```jql
project = LAN
AND issuetype = Bug
AND status in (Done, Resolved)
AND component = "{product_area}"
AND resolved >= -365d
ORDER BY resolved DESC
```

Merge all query results. Deduplicate by Jira issue key. Note which queries
each ticket appeared in — multi-query matches score higher.

---

### Step 3 — Score Each Jira Ticket for Relevance

Apply the scoring rubric to every returned ticket. Sum all applicable points.

| Criterion | Points | Condition |
|---|---|---|
| Summary exact keyword match (Set B) | +35 | Precise keyword in ticket summary |
| Summary keyword match (Set A) | +20 | Broad keyword in ticket summary |
| Description keyword match (Set B) | +20 | Precise keyword in ticket description |
| Description keyword match (Set A) | +10 | Broad keyword in ticket description |
| Verbatim error message match | +40 | Exact error string in summary or description |
| Product area / component match | +20 | Ticket component matches classified product area |
| Error type match | +15 | Ticket type aligns with classified error type |
| Multi-query match | +10 | Appeared in 2+ queries |
| Prior escalation match | +15 | Key surfaced from Module 5 account history |
| Version affected match | +10 | `affectedVersions` includes customer's version |
| Version affected older | −5 | `affectedVersions` only covers older versions |
| High priority (P1/P2/Critical) | +10 | Engineering priority signal |
| Fix version available | +15 | `fixVersions` is populated — a fix exists or is planned |
| Recently updated (last 90 days) | +5 | Active engineering attention |

**Maximum possible score: 225**

Retain tickets with a score of 30 or above. No hard cap on results — all
relevant bugs should be surfaced. In practice expect 0–8 strong matches.

---

### Step 4 — Classify Each Retained Ticket

For each retained ticket assign a resolution classification that directly
informs how Module 12 presents the recommendation:

| Classification | Condition | Implication for Solution |
|---|---|---|
| `FIX AVAILABLE` | Status Done/Resolved + fixVersion populated | Upgrade to fix version or apply patch |
| `FIX IN PROGRESS` | Status In Progress or In Review | Share ETA if available; apply workaround meanwhile |
| `WORKAROUND DOCUMENTED` | Open/Backlog but comments or description contain a workaround | Apply the documented workaround |
| `ACKNOWLEDGED — NO FIX` | Open/Backlog with no workaround documented | Set customer expectation; escalate urgency if high ARR |
| `WONT FIX` | Status Won't Fix or closed without resolution | Explain limitation; explore alternative approaches |
| `RESOLVED OLD VERSION` | Done but fixVersion is older than customer's version | Customer may already have fix — verify version |
| `DUPLICATE` | Marked as duplicate — has parent ticket | Retrieve and assess parent ticket |

---

### Step 5 — Extract Workarounds and Fix Details

For each retained ticket, extract:

**Fix Details** (if `FIX AVAILABLE` or `FIX IN PROGRESS`):
- Fix version(s): `{fixVersions}`
- Release date if known
- Upgrade path or patch instructions from ticket description or comments

**Workaround Steps** (if `WORKAROUND DOCUMENTED` or any open ticket):
- Read last 10 comments on the ticket for documented workarounds
- Extract numbered steps if present
- Note who documented the workaround (engineering vs. support)

**Affected Versions:**
- List all `affectedVersions` from the ticket
- Compare against customer's version from Module 1
- Flag: version affected / not listed / version unknown

**ETA** (if `FIX IN PROGRESS`):
- Check `fixVersions` for a planned release version
- If no fix version, note "No ETA available"
- Do not invent or estimate ETAs — only report what is in the ticket

---

### Step 6 — Assess New Bug Filing Recommendation

Evaluate whether the current case warrants filing a new Jira bug. Apply this
decision logic:

**Recommend NEW BUG FILING if ALL of the following are true:**
1. No matching Jira ticket found with score ≥ 30
2. Root Cause Hypothesis confidence from Module 3 is MEDIUM or HIGH
3. Error Type from Module 3 is `Functional Bug`, `Performance`, `Compatibility`,
   or `Data Quality` — not `How-To / Usage` or `Configuration`
4. At least one of:
   - Exhaustion Signal from Module 7 is DEEP or PARTIAL
   - 2+ unresolved GOLD/STRONG cases in Module 6 with the same issue pattern
   - Current case has CSAT implications (high-ARR account, renewal signal)

**Recommend MONITOR (do not file yet) if:**
- A related ticket exists but confidence of match is borderline (score 30–50)
- Issue may be configuration-related rather than a product defect

**No new bug needed if:**
- A matching ticket already exists (score ≥ 50)
- Error Type is `How-To / Usage`, `Configuration`, or `Permission / Access`
- Root Cause Hypothesis confidence is LOW or INSUFFICIENT

---

### Step 7 — Assemble Jira Bug Object

```
JIRA BUG OBJECT
─────────────────────────────────────────────
SEARCH SUMMARY
  Queries Run           : {count}
  Already-Linked Tickets: {count from Module 1}
  Tickets Found         : {total from search}
  Retained (score ≥ 30) : {count}
  New Bug Recommended   : {YES / MONITOR / NO}

ALREADY-LINKED TICKETS (from current case)
  {Key} — {Summary} | Status: {status} | Classification: {classification}

TOP MATCHING JIRA TICKETS (ranked by score)

Rank 1 — Score: {score}/225
  Ticket Key      : {LAN-XXXXX}
    (link: https://lansweeper.atlassian.net/browse/LAN-XXXXX)
  Summary         : {summary}
  Status          : {status}
  Priority        : {priority}
  Classification  : {FIX AVAILABLE / FIX IN PROGRESS / etc.}
  Affected Versions: {list}  |  Version Match: {✅ / ⚠️ / ⬇️ / 〰️}
  Fix Version     : {fixVersions or "None planned"}
  Workaround      : {extracted steps, or "None documented"}
  Fix Details     : {upgrade path or patch info, or "N/A"}
  Matched Queries : {A / B / C / D / E}

Rank 2 — Score: {score}/225
  ... (repeat for all retained tickets)

NEW BUG ASSESSMENT
  Recommendation  : {YES / MONITOR / NO}
  Reasoning       : {1–2 sentences explaining the recommendation}

NO MATCHES
  {If zero tickets above threshold: "No matching Jira bugs found for this
  issue classification."}
─────────────────────────────────────────────
```

---

## Output to User

```
🐛 JIRA BUG LOOKUP — Case {CaseNumber}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Tickets Searched  : {total}  |  Matches Found: {retained count}
New Bug Needed    : {YES / MONITOR / NO}

TOP MATCHES:
  #1 [Score: {score}] {LAN-KEY} — {Summary}
     Status: {status}  |  Classification: {classification}
     Version Match: {flag}  |  Fix: {fix version or "None"}
     {1-sentence workaround or fix summary}

  #2 [Score: {score}] {LAN-KEY} — {Summary}
     Status: {status}  |  Classification: {classification}
     Version Match: {flag}  |  Fix: {fix version or "None"}
     {1-sentence workaround or fix summary}

  ... (up to 5 shown in summary; full list in Jira Bug Object)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Jira bug lookup complete. Ready for Module 9 (Gong Call Review).
```

If FIX AVAILABLE found:
```
✅ FIX AVAILABLE — {LAN-KEY} has a confirmed fix in version {fixVersion}.
   Verify customer is on or can upgrade to this version.
```

If new bug recommended:
```
🆕 NEW BUG RECOMMENDED — No matching Jira ticket found. This case pattern
   suggests a product defect. Module 11 will evaluate escalation.
```

---

## How Downstream Modules Use This Object

| Module | Consumes |
|---|---|
| Module 11 — Escalation Signal | New Bug Recommendation, ACKNOWLEDGED—NO FIX tickets, high-priority open bugs |
| Module 12 — Solution Synthesis | Full ticket list with classifications, workarounds, fix details, version flags |

---

## Weighting Passed to Module 12

Jira results influence Module 12 differently from case history — they change the
*type* of solution recommended, not just the ranking:

| Condition | Weight / Behaviour |
|---|---|
| FIX AVAILABLE, version match | 25 — lead with upgrade/patch instruction |
| FIX AVAILABLE, version mismatch | 15 — verify applicability first |
| WORKAROUND DOCUMENTED | 18 — present workaround as primary path |
| FIX IN PROGRESS | 10 — present ETA + interim workaround |
| ACKNOWLEDGED — NO FIX | 5 — set expectation; note in customer communication |
| WONT FIX | 3 — acknowledge limitation; explore alternatives |
| New Bug Recommended | Escalation flag to Module 11 — no solution weight |

---

## Error Handling

| Condition | Behaviour |
|---|---|
| Atlassian MCP unavailable | Note "Jira search unavailable." Pass empty object. Do not block workflow. |
| Query returns zero results | Run fallback query using Product Area only before declaring no matches |
| Linked ticket key is malformed HTML | Parse best-effort. Note if key cannot be extracted. |
| Ticket is marked Duplicate | Retrieve parent ticket automatically and assess parent instead |
| Fix version listed but no upgrade path in ticket | Note fix version exists but no instructions available — recommend checking release notes |
| affectedVersions field is empty | Apply `VERSION_UNKNOWN` flag. Do not assume the ticket applies or does not apply. |

---

## Notes on Jira Search Behaviour

- JQL `~` operator performs full-text search including stemming — "scanning" will
  match "scanned", "scanner" etc. This is intentional for recall.
- Verbatim error message queries (Query C) are the most precise queries in the
  system. When an exact error string matches a Jira ticket summary, treat this
  as near-certain confirmation the issue is known.
- Tickets in `Won't Fix` status should still be surfaced and classified — they
  are important context for setting customer expectations. Do not discard them
  from the output.
- The `fixVersions` field in Jira is populated by engineering when a fix is
  committed to a release. If it is null on a closed/done ticket, the fix may
  have been delivered without being formally tagged — note this ambiguity rather
  than assuming no fix exists.
- Do not surface internal Jira ticket descriptions verbatim in any customer-facing
  output from Module 12. Extract the actionable information (fix version,
  workaround steps, ETA) and present that instead.
