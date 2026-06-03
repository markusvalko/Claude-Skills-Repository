---
name: case-look-up-gong-call-review
description: >
  Searches call and meeting transcripts for conversations involving the account that
  opened the support case, surfacing verbal context about the issue, environment
  details, prior frustrations, and off-ticket troubleshooting steps that never made
  it into Salesforce. Uses Spinach.ai as the primary transcript source. Always use
  this skill as Module 9 immediately after Module 8 (case-look-up-jira-bug-lookup)
  has completed. Also trigger when someone asks "did we have any calls with this
  customer about this", "check Gong for this account", "were there any meetings
  about this issue", "what did the customer say on the call", "search call transcripts
  for this account", or "has this come up on any calls" in the context of a support
  case. This is Module 9 of the case solution ranking system.
---

# Case Look Up Gong Call Review — Module 9

Searches meeting and call transcripts for the account that opened the case using
the Spinach.ai MCP connector. Calls often contain context that never surfaces in
a support ticket — the customer may have mentioned the issue in passing on a QBR,
described their environment in detail on an onboarding call, or expressed frustration
about a recurring problem that the support thread doesn't capture. This module
surfaces that context to enrich the solution ranking and flag anything that changes
how the issue should be handled or communicated.

This module does not produce ranked solutions directly. It produces a **context
enrichment layer** that Module 12 uses to calibrate tone, urgency, and completeness
of the final output.

---

## Prerequisites

The following must be available before this module runs:
- `Account.Name` from Module 1 Enriched Case Object — primary search term
- `Contact.Name` and `Contact.Email` from Module 1 — used to find calls by participant
- Issue Classification Object from Module 3 — provides keywords for transcript search
- Account Context Object from Module 2 — provides CSM name for participant filtering

Spinach.ai MCP connector must be available. If unavailable, skip this module
entirely and note "Call transcript search unavailable — Spinach.ai not connected."

---

## Step-by-Step Execution

Run Steps 1 and 2 in parallel for efficiency.

---

### Step 1 — Search by Account Name and Issue Keywords

Search Spinach.ai for meetings involving this account that contain discussion
relevant to the classified issue.

```
Tool: Spinach.ai:search_meetings
Query: "{Account.Name} {top keyword from Keyword Set A}"
is_external: true
vector_weight: 0.6
limit: 10
```

Run a second search focused on the issue itself:
```
Tool: Spinach.ai:search_meetings
Query: "{problem statement from Module 3} {Account.Name}"
is_external: true
vector_weight: 0.7
limit: 10
```

Run a third search using the verbatim error message if available:
```
Tool: Spinach.ai:search_meetings
Query: "{verbatim error message or error code}"
is_external: true
vector_weight: 0.2
limit: 5
```

Use `vector_weight: 0.2` for the error message search — exact technical strings
benefit from keyword-dominant matching. Use `vector_weight: 0.6–0.7` for
conceptual queries where the customer may have described the issue in non-technical
language.

---

### Step 2 — Search by Contact Participant

Search for all external meetings involving the primary case contact:

```
Tool: Spinach.ai:list_meetings
participant: "{Contact.Name}"
is_external: true
from_date: "{date 12 months ago}"
fields: ["id", "title", "date", "participants", "chapters", "summary_snippet"]
limit: 50
```

If `Contact.Name` is null, fall back to searching by account name participant
pattern — look for participants whose email domain matches the account's website
domain from Module 2.

Scan the returned meeting list for chapters whose summaries contain any keyword
from Keyword Set A or Keyword Set B. Flag those meetings as candidates for
full transcript fetch.

---

### Step 3 — Fetch Full Content for Candidate Meetings

For each meeting identified as a candidate in Steps 1 and 2, fetch full meeting
content:

```
Tool: Spinach.ai:get
id: "{meeting_id}"
include: ["summary", "action_items", "decisions", "chapters", "participants"]
```

Do not fetch transcripts at this stage — chapter summaries are sufficient for
most matches and transcripts are large. Only fetch the transcript for a specific
chapter if the chapter summary strongly suggests relevant technical detail:

```
Tool: Spinach.ai:get
id: "{meeting_id}"
chapter_index: {index of relevant chapter}
include: ["transcript"]
```

Cap transcript fetches at 3 chapters across all meetings to manage context volume.

---

### Step 4 — Extract Relevant Signals

For each fetched meeting, read the full content and extract any of the following
signal types. Only extract what is genuinely present — do not infer or fill gaps.

---

#### 4a. Issue Mentions
Any direct or indirect reference to the classified issue in the call.
Record: what was said, who said it (customer vs. internal), meeting date.

---

#### 4b. Environment Details
Any technical environment information mentioned verbally that may not be in the
case thread:
- Specific server configurations, OS versions, network topology
- Integration targets mentioned (AD, Azure, ServiceNow, etc.)
- Number of scan targets, sites, or assets
- Deployment constraints (air-gapped, strict firewall, VPN-only access)

Compare against the Environment Details from Module 3. Flag any new details not
already captured.

---

#### 4c. Prior Frustrations or Escalation Mentions
Any language suggesting:
- The customer has experienced this or a related issue before
- The customer expressed urgency, frustration, or dissatisfaction
- A prior promise was made by the Lansweeper team ("we'll fix this in the next
  release", "your CSM will follow up on this")
- The issue was raised and not actioned

---

#### 4d. Off-Ticket Troubleshooting
Any troubleshooting steps, workarounds, or configuration changes mentioned
verbally that do not appear in the Salesforce case thread. These are added
to the Already Tried context from Module 7 — pass them with the label
`[CALL — {meeting date}]`.

---

#### 4e. Relationship and Sentiment Signals
- Overall sentiment of the customer in recent calls (positive, neutral, frustrated,
  at-risk language)
- Any churn signals, competitive mentions, or escalation threats
- Praise or positive feedback about the support team or product

These are passed to Module 12 as tone guidance for the solution presentation —
they do not affect solution ranking.

---

#### 4f. Commitments Made
Any specific commitments made by Lansweeper staff on the call that are relevant
to this case:
- Fix timelines promised
- Feature requests acknowledged and noted
- Follow-up actions assigned

---

### Step 5 — Deduplicate Against Known Thread Content

Cross-reference all extracted signals against the case thread from Module 1 and
the Already Tried list from Module 3. Remove any signal that is already
documented in the case thread — only surface *new* information from calls.

For off-ticket troubleshooting steps (4d), pass new items to the Deduplication
Object from Module 7 as late-arriving `[CALL]` entries. Module 12 will apply
the same suppression logic to these as to any other already-tried item.

---

### Step 6 — Assess Call Coverage

Evaluate the quality of the call search:

| Level | Condition |
|---|---|
| `RICH` | 3+ relevant meetings found with issue-relevant content |
| `MODERATE` | 1–2 relevant meetings found |
| `SPARSE` | Meetings found but no issue-relevant content in transcripts |
| `NONE` | No meetings found for this account/contact |

Note: `NONE` or `SPARSE` is a normal and expected result for many cases —
not all customers have regular calls with the Lansweeper team. Do not treat
absence of call data as a negative signal.

---

### Step 7 — Assemble Gong Call Context Object

```
GONG CALL CONTEXT OBJECT
─────────────────────────────────────────────
SEARCH SUMMARY
  Account Searched      : {Account.Name}
  Contact Searched      : {Contact.Name or "Not available"}
  Meetings Found        : {total from search}
  Candidate Meetings    : {count with relevant content}
  Transcripts Fetched   : {count of chapter transcripts pulled}
  Call Coverage         : {RICH / MODERATE / SPARSE / NONE}

RELEVANT MEETINGS (chronological, most recent first)

Meeting 1
  Title       : {title}
  Date        : {date}
  URL         : {url}
  Participants: {names}
  Relevance   : {1-sentence summary of why this meeting is relevant}

  Issue Mentions:
    {What was said, by whom, date}

  New Environment Details:
    {Any new technical details not in Module 3, or "None"}

  Prior Frustrations / Escalation Mentions:
    {Relevant language, or "None"}

  Off-Ticket Troubleshooting:
    {Steps mentioned verbally not in case thread, or "None"}

  Commitments Made:
    {Any promises or follow-ups committed, or "None"}

  Relationship / Sentiment:
    {Tone and sentiment signal, or "Neutral / not assessed"}

Meeting 2
  ... (repeat for all relevant meetings)

LATE-ARRIVING ALREADY TRIED ITEMS (for Module 7 supplement)
  {[CALL — date]: step description}
  {or "None — no off-ticket troubleshooting found"}

TONE GUIDANCE FOR MODULE 12
  Customer Sentiment    : {Positive / Neutral / Frustrated / At-Risk / Unknown}
  Commitments to Address: {List of any open commitments, or "None"}
  Churn Risk Signal     : {Yes / No / Unknown}
─────────────────────────────────────────────
```

---

## Output to User

```
📞 GONG CALL REVIEW — {Account.Name}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Meetings Searched : {total}  |  Relevant Found: {candidate count}
Call Coverage     : {RICH / MODERATE / SPARSE / NONE}
Customer Sentiment: {sentiment signal}

RELEVANT MEETINGS:
  {Date} — {title}
  {1-sentence relevance summary}
  New context: {brief note on any new environment details or off-ticket steps}

  {Date} — {title}
  ...

LATE-ARRIVING ALREADY TRIED:
  {[CALL — date]: step, or "None found"}

COMMITMENTS TO ADDRESS:
  {List, or "None found"}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Gong call review complete. Ready for Module 10 (Gainsight Review).
```

If no meetings found:
```
ℹ️  No external meetings found for {Account.Name} / {Contact.Name}.
    This is a normal result — call transcript context is supplementary.
    Proceeding to Module 10 with no call enrichment.
```

If at-risk or churn signals detected:
```
⚠️  RELATIONSHIP SIGNAL — Customer expressed {frustration / churn risk /
    escalation language} on {date} call. Module 12 will flag this for
    urgent handling guidance.
```

---

## How Downstream Modules Use This Object

| Module | Consumes |
|---|---|
| Module 7 — Already Tried Dedup | Late-arriving off-ticket troubleshooting steps (passed as supplement) |
| Module 10 — Gainsight Review | Sentiment signal and churn risk flag — cross-referenced with health data |
| Module 11 — Escalation Signal | Commitments made + churn risk signal — open commitments lower escalation threshold |
| Module 12 — Solution Synthesis | Tone guidance, sentiment, new environment details, commitment context |

---

## Weighting Passed to Module 12

Call review results do not directly weight solution steps. Instead they produce
three modifier outputs that Module 12 applies to the overall response:

| Signal | Module 12 Behaviour |
|---|---|
| New environment details found | Prepend environment note to solution steps that depend on those details |
| Off-ticket steps found | Passed to Module 7 supplement — suppressed or downranked in final output |
| Frustrated / At-Risk sentiment | Add urgency note and escalation path to Module 12 output |
| Open commitments found | Module 12 explicitly references the commitment in solution framing |
| Positive sentiment | Module 12 maintains collaborative, partnership tone |
| Churn risk signal | Module 12 flags for CSM notification alongside solution output |

---

## Error Handling

| Condition | Behaviour |
|---|---|
| Spinach.ai MCP unavailable | Skip module. Note "Call transcript search unavailable." Pass empty object. |
| Contact.Name is null | Search by account name only. Note reduced search precision. |
| No meetings found for account | Set coverage to NONE. Pass empty object. Do not block workflow. |
| Meetings found but all internal | Note "Only internal meetings found — no external customer calls available." |
| Transcript fetch fails | Use chapter summary only. Note "Transcript unavailable for this chapter." |
| Meeting content is irrelevant after fetch | Discard meeting from output. Do not include for completeness. |
| More than 10 candidate meetings | Prioritise by recency and relevance score. Cap at 5 meetings for full fetch. |

---

## Notes on Search Quality

- Use `vector_weight: 0.6–0.7` for issue and problem statement queries —
  customers describe technical problems in non-technical language on calls,
  so semantic matching is more valuable than keyword matching here.
- Use `vector_weight: 0.2` for exact technical strings like error codes or
  product component names — these benefit from keyword-dominant matching.
- The `is_external: true` filter is important — internal Lansweeper meetings
  are not relevant to customer issue context and will pollute results.
- Meeting titles alone are often insufficient to assess relevance — always
  scan chapter summaries before deciding whether to fetch full content.
- Sentiment signals from calls are inherently subjective. Flag language that
  is clearly frustrated or at-risk ("we're evaluating alternatives", "this is
  unacceptable", "we need this fixed or we'll have to reconsider") but do not
  interpret neutral or ambiguous language as negative.
- Never surface verbatim customer quotes from call transcripts in any output
  shown to the customer. Call content is for internal context only.
