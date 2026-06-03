---
name: case-look-up-solution-synthesis-and-ranking
description: >
  Aggregates all module outputs from the case solution ranking system, applies the
  weighting model, deduplicates already-tried solutions, and produces a ranked list
  of potential solutions with confidence scores, source attribution, and handling
  guidance as a polished self-contained HTML report. Always use this skill as
  Module 12 — the final step — immediately after Module 11
  (case-look-up-escalation-signal-detection) has completed. Also trigger when
  someone asks "give me the ranked solutions", "what should we try", "produce the
  final report", "show me the solution ranking", or "what are the recommendations
  for this case" in the context of a support case. This is Module 12 of the case
  solution ranking system.
---

# Case Look Up Solution Synthesis & Ranking — Module 12

The final module. Reads all objects assembled by Modules 1–11 and produces a single
polished HTML report containing: a case and account summary, escalation directives
(if triggered), a ranked and weighted solution list with confidence scores and source
attribution, handling posture guidance, and suggested customer communication framing.

This module applies the complete weighting model, enforces deduplication from Module
7, and ensures the output is ready for an engineer to act on immediately.

---

## Prerequisites

All prior modules should have completed. If any module is missing its object,
note the gap and proceed — do not block on incomplete upstream data.

Required inputs (will degrade quality if absent):
- Module 1 — Enriched Case Object
- Module 3 — Issue Classification Object
- Module 7 — Deduplication Object (suppression list)
- Module 12 can produce a partial output from Module 1 + 3 alone if needed

Strongly recommended inputs:
- Module 2 — Account Context Object
- Module 4 — KB Results Object
- Module 5 — Account Case History Object
- Module 6 — Global Case History Object
- Module 8 — Jira Bug Object
- Module 9 — Gong Call Context Object
- Module 10 — Gainsight Review Object
- Module 11 — Escalation Signal Object

---

## Step-by-Step Execution

### Step 1 — Collect All Candidate Solutions

Gather every solution, resolution step, and article from all upstream modules
into a single flat candidate pool. Assign each a unique solution ID (`SOL-001`,
`SOL-002`, etc.) and tag its source.

| Source | Source Tag |
|---|---|
| KB article (Module 4) | `[KB]` |
| Account case history (Module 5) | `[ACCT]` |
| Global case history (Module 6) | `[GLOBAL]` |
| Jira workaround (Module 8) | `[JIRA-WA]` |
| Jira fix — upgrade path (Module 8) | `[JIRA-FIX]` |

For each candidate, record:
- Source tag and reference (article title, case number, Jira key)
- The resolution steps (numbered, action-level)
- CSAT score of source (if applicable)
- Version match classification
- Outcome confidence (CONFIRMED / ASSUMED / UNRESOLVED)

---

### Step 2 — Apply Deduplication Suppression

Cross every candidate solution against the Module 7 Deduplication Object.
Apply suppressions strictly:

- **Tier 1 (Hard Suppress):** Remove from candidate pool entirely. Do not include
  in ranked output. Note in a "Suppressed Solutions" footnote.
- **Tier 2 (Soft Suppress):** Keep in candidate pool but flag with note:
  *"Previously attempted — consider retrying with closer monitoring."*
  Reduce base weight by 40%.
- **Tier 3 (Downrank):** Keep in candidate pool with note:
  *"More thorough version of a previously attempted step."*
  Reduce base weight by 20%.
- **Late-arriving call items from Module 9:** Apply same suppression logic.

After suppression, the remaining candidates form the **active solution pool**.

---

### Step 3 — Apply the Weighting Model

Score every candidate in the active solution pool. Sum all applicable weights.

**Base Weights by Source:**

| Source | Base Weight | Condition |
|---|---|---|
| `[ACCT]` — same account, score ≥ 100 | 40 | Highest weight in system |
| `[ACCT]` — same account, score 50–99 | 28 | Strong account match |
| `[ACCT]` — same account, score 25–49 | 18 | Weak account match |
| `[GLOBAL]` — score ≥ 100 | 20 | Strong cross-account match |
| `[GLOBAL]` — score 50–99 | 14 | Solid global match |
| `[GLOBAL]` — score 25–49 | 8 | Weak global match |
| `[KB]` — score ≥ 100 | 15 | High confidence KB match |
| `[KB]` — score 60–99 | 12 | Solid KB match |
| `[KB]` — score 15–59 | 8 | Weak KB match |
| `[JIRA-FIX]` — version match | 25 | Fix exists for this version |
| `[JIRA-FIX]` — version mismatch | 15 | Fix exists, verify applicability |
| `[JIRA-WA]` — workaround documented | 18 | Workaround available |

**Multipliers (applied to base weight):**

| Multiplier | Factor | Condition |
|---|---|---|
| CSAT 4–5 | ×1.5 | Source case had high satisfaction |
| CSAT 1–3 | ×0.7 | Source case had low satisfaction |
| Confirmed outcome | +8 flat | Customer explicitly confirmed fix worked |
| Recurring solution (Module 5) | +12 flat | Same fix worked 2+ times for this account |
| GOLD signal (Module 6) | +8 flat | CSAT 5 + confirmed resolution |
| Fast resolution | +5 flat | Source case closed within 3 days |
| Version match | +5 flat | Same Lansweeper version as current case |
| Older version | −5 flat | Significantly older version |
| KB version match | +3 flat | KB article explicitly covers current version |
| KB KNOWN ISSUE tag | +5 flat | Article confirms documented problem |
| Soft suppress (Tier 2) | ×0.6 | Previously attempted, outcome unknown |
| Downrank (Tier 3) | ×0.8 | Partial overlap with prior attempt |
| Unresolved outcome | −8 flat | Source case closed without fix |

Compute a **final weighted score** for each candidate. Sort descending.

---

### Step 4 — Group Into Solution Tiers

After scoring, group candidates into tiers to give the engineer a clear
reading of confidence level:

| Tier | Score Range | Label |
|---|---|---|
| Tier A | Top score down to 70% of top score | **HIGHEST CONFIDENCE** |
| Tier B | 70%–40% of top score | **SOLID OPTIONS** |
| Tier C | Below 40% of top score | **WORTH TRYING** |

If a `[JIRA-FIX]` candidate exists with a version match, always promote it to
Tier A regardless of score — a confirmed fix should always lead.

Cap the output at **10 ranked solutions** total. If there are more, include
the top 10 and note how many were suppressed or fell below tier threshold.

---

### Step 5 — Prepare Handling Posture Block

Read the Handling Posture from Module 10 and the Escalation Signal Object from
Module 11. Prepare two output blocks:

**Escalation Block** (include if Module 11 severity ≥ MEDIUM):
- List all REQUIRED escalation actions in priority order
- Include the pre-drafted Jira bug filing details (Type 1) if triggered
- Include the CSM notification message (Type 3) if triggered
- Include support lead assignment note (Type 4) if triggered

**Handling Posture Block** (always include):
- State the posture level and its primary driving signal
- Note CSM name if notification is recommended
- Note any open commitments from Module 9 that must be acknowledged

---

### Step 6 — Prepare Customer Communication Framing

Based on Module 9 sentiment, Module 10 health, and the nature of the top-ranked
solutions, prepare a brief framing note for the engineer on how to communicate
the solution to the customer. This is guidance, not a drafted email.

| Scenario | Framing Guidance |
|---|---|
| Top solution is `[JIRA-FIX]` | Lead with confirmation that the issue is a known bug with a fix. Provide upgrade path clearly. Do not make the customer feel at fault. |
| Top solution is `[ACCT]` CONFIRMED | Lead with "we've seen this before in your environment — here's what resolved it last time." |
| Top solution is `[KB]` STEP-BY-STEP | Present as a structured guide. Offer to walk through together if needed. |
| Exhaustion Signal = DEEP | Acknowledge the customer's frustration with the length of the case. Lead with escalation path, not more steps. |
| Module 9 sentiment = Frustrated/At-Risk | Open with acknowledgement before presenting solutions. Do not lead with a list. |
| Module 11 severity = CRITICAL | Do not send technical steps without CSM coordination first. |

---

### Step 7 — Build the HTML Report

Generate a single self-contained HTML report saved to `/mnt/user-data/outputs/`.
Name it: `case_solution_ranking_{CaseNumber}_{YYYY-MM-DD}.html`

Use the Lansweeper design system from the daily-support-dashboard skill:
```css
--bg:    #f5f4f0
--surf:  #ffffff
--dark:  #0f1117
--acc:   #2d6ef7
--grn:   #2dac6a
--red:   #e84c4c
--txt:   #1a1a1a
--muted: #6b6b6b
--bdr:   #e0dfd8
Fonts: DM Serif Display (headings), DM Sans (body), DM Mono (numbers, IDs, scores)
```

---

#### Report Structure

**Hero Banner** (dark `#0f1117`):
- Eyebrow: "Lansweeper Case Intelligence" (DM Mono, small caps)
- Title: "Solution Ranking / Case {CaseNumber}"
- Subtitle: "{Account.Name} — {Account ARR tier} — {today's date}"
- Four stat pills: Solutions Found · Suppressed · Escalation Severity · Handling Posture

---

**Section 01 — Case Summary**

Two-column layout:

Left column:
- Case number, subject, status, priority, owner
- Opened date, days open, customer message count
- Product area, version, error type (from Module 3)

Right column:
- Account name (linked to Salesforce), ARR, tier
- CSM name, renewal date + days remaining
- Health score pill (colour-coded by label), churn risk

Below columns: Issue Classification block
- Customer-facing problem statement
- Root cause hypothesis with confidence level
- Already Tried count (Tier 1 hard-suppressed count in red)

---

**Section 02 — Escalation Directives**
*(Only render this section if Module 11 severity ≥ MEDIUM)*

Dark amber (`#f5a623`) section header to signal urgency.

For each triggered escalation type, a card containing:
- Type label and severity badge
- Conditions that triggered it (collapsed by default, expandable)
- Required action in a clearly styled action box
- For Type 1: expandable pre-drafted Jira bug report fields
- For Type 3: pre-drafted CSM notification text in a copyable block

If severity = CRITICAL, add a prominent red banner at the top of this section:
*"Do not send technical solutions to the customer until CSM coordination is complete."*

---

**Section 03 — Ranked Solutions**

Main section. For each solution in the active pool (up to 10):

**Source URL Construction**

Use the following lookup table to build source links for each solution card:

| Source Tag | URL Pattern | Source |
|---|---|---|
| `[KB]` | URL passed directly from Module 4 KB Results Object (article URL field) | Module 4 |
| `[ACCT]` | `https://lansweeper.my.salesforce.com/{CaseId}` | Module 5 |
| `[GLOBAL]` | `https://lansweeper.my.salesforce.com/{CaseId}` | Module 6 |
| `[JIRA-FIX]` | `https://lansweeper.atlassian.net/browse/{JiraKey}` | Module 8 |
| `[JIRA-WA]` | `https://lansweeper.atlassian.net/browse/{JiraKey}` | Module 8 |

**Solution card** containing:
- Rank number and tier badge (HIGHEST CONFIDENCE / SOLID / WORTH TRYING)
- Weighted score and source tag
- Solution title (derived from the resolution summary)
- Source reference: rendered as a clickable `<a href="{url}" target="_blank">` anchor. Display text = case number for SF cases, Jira key for Jira tickets, article title for KB articles. If URL is unavailable from upstream module, display the identifier as plain text with a note "(link unavailable)".
- CSAT badge (if source case), version match flag, outcome badge
- Numbered resolution steps — action-level, ready to execute
- Collapsed metadata section (expandable): score breakdown, multipliers applied,
  module that sourced this solution

Tier A cards: white background with left border `--acc` (#2d6ef7)
Tier B cards: white background with left border `--grn` (#2dac6a)
Tier C cards: `#fafaf8` background with left border `--bdr` (#e0dfd8)

Soft-suppressed solutions (Tier 2): shown with amber left border and
caution note. Do not hide these — the engineer should see them.

---

**Section 04 — Handling Posture & Communication Guidance**

Clean card layout:

Left card — Handling Posture:
- Posture level with colour-coded badge
- Primary driving signal
- CSM notification status (Required / Recommended / Not required)
- Open commitments to acknowledge (if any from Module 9)

Right card — Communication Framing:
- Framing guidance from Step 6
- Sentiment note from Module 9
- Any relationship risk flags from Module 10

---

**Section 05 — Suppressed Solutions Footnote**

Collapsed section (expand to view):
- Tier 1 hard-suppressed solutions listed with reason
- Confirmation that these have been excluded from ranking
- Note: "These were tried and confirmed failed — they will not reappear
  in any future run of this tool for this case."

---

**Footer**
- Modules run: list of all 12 modules with ✅ completed / ⚠️ partial / ❌ unavailable
- Data freshness: timestamp of when each Salesforce query ran
- Source counts: KB articles searched, cases reviewed, Jira tickets checked,
  meetings reviewed
- Disclaimer: "Solution rankings are based on historical case data and
  documentation. Engineer judgment is always required before acting."

---

## Output to User

After saving the HTML file, present a brief terminal summary before calling
`present_files`:

```
✅ SOLUTION RANKING COMPLETE — Case {CaseNumber}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Account       : {Account.Name}  ({ARR tier})
Issue         : {1-sentence problem statement}
Solutions     : {count} ranked  |  {suppressed count} suppressed

TOP RECOMMENDATION [{Tier A label}]:
  {Source tag}  Score: {score}
  {Solution title}
  Step 1: {first action step}
  Step 2: {second action step}
  ...

Escalation    : {severity}  {types triggered}
Posture       : {handling posture}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Full report ready below ↓
```

Then call `present_files` with the HTML report path.

---

## Error Handling

| Condition | Behaviour |
|---|---|
| Active solution pool is empty after suppression | Report "No unsuppressed solutions found. All known approaches have been attempted." Escalation section becomes primary output. |
| Module 11 object missing | Omit Section 02. Note "Escalation assessment unavailable." |
| Module 10 posture missing | Default to STANDARD posture. Note "Gainsight data unavailable — posture defaulted to STANDARD." |
| Module 7 suppression list missing | Proceed without suppression. Note "Deduplication data unavailable — all solutions included unfiltered." |
| Fewer than 3 solutions in pool | Include all available. Do not pad with low-quality results to reach a minimum. |
| All solutions are Tier C | Note "No high-confidence solutions found — consider escalation." Escalation section elevated to primary. |
| Source URL missing from upstream object | Display identifier as plain text. Append `(link unavailable)` in muted style. Do not omit the source reference entirely. |

---

## Notes on Output Quality

- The report is written for a support engineer, not the customer. Do not soften
  technical language or omit complexity for readability.
- Source attribution must be present on every solution card. An engineer needs
  to be able to trace every recommendation back to its origin.
- Account names from other customers (Module 6 global sources) must not appear
  in the rendered HTML. Reference as "Similar case" or "Prior case in same
  product area" only.
- Verbatim customer quotes from call transcripts (Module 9) must not appear
  in any section of the report.
- Internal Jira ticket descriptions must not be reproduced verbatim — extract
  and present only the actionable elements (fix version, workaround steps, ETA).
- The report is a decision-support tool. The engineer always reviews and approves
  before acting on any escalation draft or sending any customer communication.
