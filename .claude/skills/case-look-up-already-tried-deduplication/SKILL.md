---
name: case-look-up-already-tried-deduplication
description: >
  Consolidates all solution steps already attempted in the current case thread with
  resolution steps surfaced by Modules 4, 5, and 6, and produces a deduplicated
  suppression list that prevents Module 12 from recommending solutions the customer
  or agent has already tried. Always use this skill as Module 7 immediately after
  Module 6 (case-look-up-global-case-history) has completed. Also trigger when
  someone asks "what has already been tried on this case", "what solutions have
  already been attempted", "don't suggest things we already tried", or "filter out
  steps already taken" in the context of a support case. This is Module 7 of the
  case solution ranking system.
---

# Case Look Up Already Tried Deduplication — Module 7

Reads the Already Tried list from the Module 3 Issue Classification Object alongside
all resolution steps extracted by Modules 4, 5, and 6, and produces a single
consolidated suppression list. Module 12 uses this list to exclude or down-rank
any solution that has already been attempted — ensuring the final ranked output
only surfaces genuinely new paths forward.

This module does not search for new data. It is a pure synthesis step that operates
entirely on objects already assembled by earlier modules.

---

## Prerequisites

The following objects must be available before this module runs:
- Issue Classification Object from Module 3 — provides the `Already Tried` list
  with `[CUSTOMER]`, `[AGENT]`, `[CONFIRMED FAILED]`, and `[OUTCOME UNKNOWN]` labels
- KB Results Object from Module 4 — provides resolution steps from matched articles
- Account Case History Object from Module 5 — provides resolution steps from prior
  account cases
- Global Case History Object from Module 6 — provides resolution steps from global
  cases

If Module 3's Already Tried list is empty ("Nothing attempted yet"), this module
still runs — it cross-references Modules 4–6 resolution steps to build a baseline
deduplication map for Module 12.

---

## Step-by-Step Execution

### Step 1 — Collect All Resolution Steps From Modules 4–6

Gather every resolution step extracted across the three upstream search modules.
Assign each step a source tag and a unique step ID for reference.

**From Module 4 (KB Articles):**
For each matched KB article that contains a `STEP-BY-STEP` or `WORKAROUND` type
tag, extract the numbered steps and label them:
```
[KB:{article_title_abbreviated}] Step N: {action}
```

**From Module 5 (Account Case History):**
For each retained prior case, extract the Resolution Steps list and label them:
```
[ACCT:{CaseNumber}] Step N: {action}
```

**From Module 6 (Global Case History):**
For each retained prior case, extract the Resolution Steps list and label them:
```
[GLOBAL:{CaseNumber}] Step N: {action}
```

Assign each extracted step a short unique ID: `S001`, `S002`, `S003`, etc.
across the full combined list.

---

### Step 2 — Normalise Step Language

Before matching, normalise all step descriptions to a canonical form to enable
fuzzy comparison. Apply these transformations:

- Lowercase all text
- Remove filler phrases: "please", "try to", "you should", "make sure to",
  "ensure that", "verify that", "check that", "it is recommended to"
- Collapse whitespace
- Expand common abbreviations: "svc acct" → "service account", "perms" → "permissions",
  "cfg" → "configuration", "svr" → "server"
- Strip trailing punctuation

Store both the original text and the normalised form for each step.
Matching uses the normalised form; output always uses the original text.

---

### Step 3 — Match Already Tried Steps Against Collected Steps

Take each item from the Module 3 `Already Tried` list and compare it against
every step collected in Step 1 using three matching levels:

**Level 1 — Exact Match**
Normalised text is identical or differs by only stop words.
→ Mark collected step as `SUPPRESS — exact match`

**Level 2 — Semantic Match**
Normalised text describes the same action even if worded differently.
Examples of semantic matches:
- "restarted the lansweeper server service" ≈ "restart the LsAgent service"
- "granted wmi permissions" ≈ "added service account to the WMI namespace security"
- "checked firewall rules" ≈ "verified inbound port 135 was open"

→ Mark collected step as `SUPPRESS — semantic match`

**Level 3 — Partial Overlap**
Normalised text describes a related but not identical action — the already-tried
step is a subset or superset of the collected step.
Example: "checked permissions" (already tried) vs. "audited all service account
permissions across target OUs" (collected — more thorough version of the same idea)

→ Mark collected step as `DOWNRANK — partial overlap`
  Do not fully suppress; the more thorough version may still be worth attempting.

Steps with no match at any level are marked `INCLUDE — not yet attempted`.

---

### Step 4 — Apply Outcome Modifiers

For steps marked `SUPPRESS`, apply the outcome label from the Module 3 Already
Tried entry to add context for Module 12:

| Already Tried Label | Suppression Note for Module 12 |
|---|---|
| `[CONFIRMED FAILED]` | `SUPPRESS — tried and confirmed failed` |
| `[CUSTOMER]` + no outcome | `SUPPRESS — customer attempted, outcome unknown` |
| `[AGENT]` + no outcome | `SUPPRESS — agent suggested, outcome unknown` |
| `[OUTCOME UNKNOWN]` | `SUPPRESS — attempted but result unconfirmed` |

Steps confirmed failed should be explicitly called out in Module 12 output
so the engineer knows not only to skip them but also to acknowledge to the
customer that those paths have been exhausted.

---

### Step 5 — Build the Suppression List

Compile the final suppression list with three tiers:

**Tier 1 — Hard Suppress (do not include in Module 12 output):**
All steps marked `SUPPRESS — tried and confirmed failed`
All steps marked `SUPPRESS — exact match` regardless of outcome

**Tier 2 — Soft Suppress (include in Module 12 only as a footnote):**
All steps marked `SUPPRESS — customer attempted, outcome unknown`
All steps marked `SUPPRESS — agent suggested, outcome unknown`
All steps marked `SUPPRESS — attempted but result unconfirmed`
Rationale: these were tried but the result is unknown — they may still be worth
a second attempt with better execution, so Module 12 notes them as "previously
attempted — consider retrying with closer monitoring."

**Tier 3 — Downrank (include in Module 12 at reduced weight):**
All steps marked `DOWNRANK — partial overlap`
These are shown in Module 12 with a note: "A related step was previously
attempted — this is a more thorough version."

All remaining steps with no match are passed to Module 12 as `INCLUDE` with
full weight.

---

### Step 6 — Identify Exhaustion Signals

After applying suppressions, evaluate whether the solution space appears to be
narrowing. Flag any of the following conditions:

**Deep Exhaustion Signal** — 60% or more of all collected steps are Tier 1
hard-suppressed. This means most known solutions have been tried and failed.
→ Flag: `EXHAUSTION — most known solutions attempted. Escalation likely needed.`

**Partial Exhaustion Signal** — 30–59% of collected steps are Tier 1 suppressed.
→ Flag: `PARTIAL EXHAUSTION — several solutions attempted. Review remaining steps carefully.`

**Low Suppression** — fewer than 30% of steps suppressed. Normal state for
early-stage cases.
→ No flag. Plenty of untried paths remain.

---

### Step 7 — Assemble the Deduplication Object

```
DEDUPLICATION OBJECT
─────────────────────────────────────────────
SUPPRESSION SUMMARY
  Total Steps Collected     : {count from Modules 4–6}
  Already Tried Items       : {count from Module 3}
  Tier 1 — Hard Suppressed  : {count}
  Tier 2 — Soft Suppressed  : {count}
  Tier 3 — Downranked       : {count}
  INCLUDE (untried)         : {count}
  Exhaustion Signal         : {DEEP / PARTIAL / NONE}

HARD SUPPRESSED STEPS (Tier 1)
  {Step ID} [{source tag}]: {original step text}
  Reason: {CONFIRMED FAILED / exact match}

SOFT SUPPRESSED STEPS (Tier 2)
  {Step ID} [{source tag}]: {original step text}
  Reason: {outcome label}
  Note for Module 12: "Previously attempted — consider retrying with monitoring"

DOWNRANKED STEPS (Tier 3)
  {Step ID} [{source tag}]: {original step text}
  Matched To: {already-tried item it partially overlaps}
  Note for Module 12: "More thorough version of a previously attempted step"

INCLUDE STEPS (untried — full weight)
  {Step ID} [{source tag}]: {original step text}
  ... (full list)
─────────────────────────────────────────────
```

---

## Output to User

```
🔁 DEDUPLICATION — Case {CaseNumber}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Steps Collected : {total}  |  Already Tried: {count}
Hard Suppressed : {Tier 1 count}  |  Soft Suppressed: {Tier 2 count}
Downranked      : {Tier 3 count}  |  Clear to Include: {INCLUDE count}
Exhaustion      : {DEEP / PARTIAL / NONE}

CONFIRMED FAILED (will not appear in recommendations):
  {Bulleted list of Tier 1 CONFIRMED FAILED steps}

PREVIOUSLY ATTEMPTED — OUTCOME UNKNOWN (flagged in recommendations):
  {Bulleted list of Tier 2 steps}

FRESH PATHS REMAINING:
  {Count of INCLUDE steps across KB, account history, and global history}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Deduplication complete. Ready for Module 8 (Jira Bug Lookup).
```

If Deep Exhaustion Signal triggered:
```
⚠️  EXHAUSTION SIGNAL — The majority of known solutions for this issue
    classification have already been attempted without resolution. This
    case is a strong escalation candidate. Module 11 (Escalation Signal
    Detection) will evaluate further.
```

---

## How Downstream Modules Use This Object

| Module | Consumes |
|---|---|
| Module 11 — Escalation Signal | Exhaustion Signal — Deep Exhaustion is a primary escalation trigger |
| Module 12 — Solution Synthesis | Full suppression list — Tier 1 excluded, Tier 2 footnoted, Tier 3 downranked, INCLUDE at full weight |

---

## Error Handling

| Condition | Behaviour |
|---|---|
| Module 3 Already Tried list is empty | Skip Steps 3–4. All collected steps pass as INCLUDE. Note "No prior attempts recorded." |
| Module 4 KB object is empty | Skip KB step collection. Note in summary. |
| Module 5 Account History object is empty | Skip account history step collection. Note in summary. |
| Module 6 Global History object is empty | Skip global history step collection. Note in summary. |
| All modules 4–6 returned no resolution steps | Deduplication object is empty. Pass note to Module 12: "No resolution steps available to deduplicate." |
| Semantic matching is ambiguous | When in doubt, mark as DOWNRANK rather than SUPPRESS — it is safer to include a borderline step with a caution note than to suppress a potentially valid solution |

---

## Notes on Matching Quality

- Semantic matching requires judgment. When two steps describe the same underlying
  action with different surface language, suppress. When they describe related but
  genuinely different actions, downrank.
- The goal is to protect the engineer's time and the customer's trust — recommending
  something that was already tried and failed damages credibility. When uncertain,
  err toward DOWNRANK with a note rather than hard suppression.
- Steps from Module 5 (account history) that are marked SUPPRESS because they
  were tried in a *prior case* — not the current one — should be labelled clearly:
  "Tried in Case {CaseNumber} — {outcome}." The current engineer may not have
  visibility into that prior resolution attempt.
- Do not suppress steps solely because they appear in a CONFIRMED FAILED case from
  Module 6 (global history). A step that failed for a different customer in a
  different environment may still be valid for this customer. Only suppress based
  on the Module 3 Already Tried list (current case) and Module 5 (this account's
  prior cases).
