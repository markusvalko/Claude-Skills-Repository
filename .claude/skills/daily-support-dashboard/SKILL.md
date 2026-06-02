\---

name: daily-support-dashboard

description: >

&#x20; Generates a daily Lansweeper support cases dashboard as a polished, self-contained HTML

&#x20; report file. Use this skill whenever someone asks for a support dashboard, daily case

&#x20; report, case overview, support summary, or wants to see which accounts opened tickets

&#x20; yesterday — even if they phrase it casually like "show me today's cases", "what came in

&#x20; yesterday", "build the support report", "who's been hitting us with tickets", or

&#x20; "pull the dashboard". Always trigger when the request involves Salesforce support cases

&#x20; grouped by account, ARR, or Jira tickets — especially if they mention any combination

&#x20; of: ARR, Active ARR, pipeline, open Jira, case owners, case IDs, noise filtering, or

&#x20; yesterday's cases.

\---

&#x20;

\# Daily Support Cases Dashboard

&#x20;

Generates a prioritised, interactive HTML dashboard for the Lansweeper support team

covering cases opened \*\*yesterday\*\*. The report has two ranked sections plus a full

Jira ticket panel, and every row is clickable to reveal individual case IDs, subjects,

and owners pulled live from Salesforce.

&#x20;

\---

&#x20;

\## Data Sources

&#x20;

Pull all data fresh from Salesforce each run. Three queries are needed:

&#x20;

\### 1. Yesterday's Cases (with ARR + Owner)

```soql

SELECT Id, CaseNumber, Subject, AccountId, Account.Name,

&#x20;      Account.LAN\_Active\_ARR\_\_c, Status, Priority, Type, Origin,

&#x20;      Owner.Name, CreatedDate

FROM Case

WHERE CreatedDate = YESTERDAY

&#x20; AND AccountId != null

&#x20; AND RecordType.Name != 'Sales Support'

ORDER BY Account.LAN\_Active\_ARR\_\_c DESC NULLS LAST

LIMIT 200

```

&#x20;

\### 2. Open Opportunity Pipeline per Account

Sum open (non-Closed-Won, non-Closed-Lost) opportunity amounts for every account

that appeared in yesterday's cases:

```soql

SELECT AccountId, SUM(Amount) opp\_arr

FROM Opportunity

WHERE StageName NOT IN ('Closed Lost','Closed Won')

&#x20; AND Amount != null

&#x20; AND AccountId IN (SELECT AccountId FROM Case

&#x20;                   WHERE CreatedDate = YESTERDAY AND AccountId != null

&#x20;                   AND RecordType.Name != 'Sales Support')

GROUP BY AccountId

ORDER BY SUM(Amount) DESC

LIMIT 100

```

&#x20;

\### 3. Open Jira Tickets per Account

Count non-Done, non-Canceled Jira links for every account in yesterday's cases:

```soql

SELECT LAN\_Account\_\_c, COUNT(Id) jira\_count

FROM LAN\_Jira\_Case\_Link\_\_c

WHERE LAN\_Account\_\_c IN (SELECT AccountId FROM Case

&#x20;                         WHERE CreatedDate = YESTERDAY AND AccountId != null

&#x20;                         AND RecordType.Name != 'Sales Support')

&#x20; AND LAN\_Jira\_Case\_Status\_\_c != 'Done'

&#x20; AND LAN\_Jira\_Case\_Status\_\_c != 'Canceled'

GROUP BY LAN\_Account\_\_c

ORDER BY COUNT(Id) DESC

LIMIT 50

```

&#x20;

For accounts with Jira tickets, also pull the ticket details for Section 3:

```soql

SELECT LAN\_Account\_\_c, LAN\_Jira\_Case\_Id\_\_c, LAN\_Jira\_Case\_Status\_\_c,

&#x20;      LAN\_Blocking\_Issue\_\_c

FROM LAN\_Jira\_Case\_Link\_\_c

WHERE LAN\_Account\_\_c IN (...)   -- accounts with jira\_count > 0

&#x20; AND LAN\_Jira\_Case\_Status\_\_c != 'Done'

&#x20; AND LAN\_Jira\_Case\_Status\_\_c != 'Canceled'

LIMIT 200

```

&#x20;

The Jira ticket key lives in `LAN\_Jira\_Case\_Id\_\_c` as an HTML anchor tag like:

`<a href="https://lansweeper.atlassian.net/browse/LAN-12345">LAN-12345</a>`.

Extract the key and URL from it.

&#x20;

\---

&#x20;

\## Key Salesforce Fields

&#x20;

| Field | Object | Label | Notes |

|---|---|---|---|

| `LAN\_Active\_ARR\_\_c` | Account | Active ARR | Live contract revenue. \*\*Use this field for ARR.\*\* |

| `LAN\_Total\_Account\_ARR\_\_c` | Account | Total Account ARR | Broader roll-up; less reliable — avoid unless explicitly requested |

| `RecordType.Name` | Case | Record Type | Exclude `'Sales Support'` — these are sales-handled tickets, not support |

| `LAN\_Jira\_Case\_Link\_\_c` | Custom Object | Jira Case Links | Junction between Cases and Jira tickets |

| `LAN\_Jira\_Case\_Status\_\_c` | LAN\_Jira\_Case\_Link\_\_c | Jira Status | Filter: exclude 'Done' and 'Canceled' for open tickets |

| `LAN\_Jira\_Case\_Id\_\_c` | LAN\_Jira\_Case\_Link\_\_c | Jira Ticket | HTML hyperlink — parse key and URL from it |

| `LAN\_Blocking\_Issue\_\_c` | LAN\_Jira\_Case\_Link\_\_c | Blocking | Boolean — highlight if true |

| `Owner.Name` | Case | Case Owner | May be a user name or queue name (e.g. "Enterprise EMEA Support Queue") |

&#x20;

\---

&#x20;

\## Noise Filter

&#x20;

Before processing, discard cases that match \*\*any\*\* of these rules. Apply to every case

before grouping by account:

&#x20;

```

1\.  Type == "Spam"

2\.  Subject contains "requested access to the lansweeper community portal"  (case-insensitive)

3\.  Subject starts with "automatic reply" or "réponse automatique"           (case-insensitive)

4\.  Subject contains "mailbox not monitored"

5\.  Subject contains "out of office"

6\.  Subject contains "undeliverable"

7\.  Account.Name == "Lansweeper" or "SFDC Test"

8\.  Type == "Duplicate Case"

9\.  Subject contains "w-9" or "banking info"

10\. Type == "Post-Sales" AND subject contains "renewal", "purchase order", or "quotation"

11\. RecordType.Name == "Sales Support"

```

&#x20;

Report the filter counts in the hero: Total Inbound / Real Cases / Filtered Noise / Unique Accounts.

&#x20;

\---

&#x20;

\## 30-Day Case Counts (Section 2 only)

&#x20;

For the volume table, fetch the 30-day case count per account so you can compute a trend:

```soql

SELECT AccountId, COUNT(Id) cnt

FROM Case

WHERE CreatedDate = LAST\_N\_DAYS:30

&#x20; AND AccountId IN (<account IDs from yesterday's real cases>)

GROUP BY AccountId

```

Trend = ↑ if yesterday's count > (30-day count / 30) × 1.5, otherwise →.

&#x20;

\---

&#x20;

\## Dashboard Structure

&#x20;

The output is a \*\*single self-contained HTML file\*\* saved to `/mnt/user-data/outputs/`.

Name it `support\_cases\_dashboard\_YYYY-MM-DD.html` where the date is yesterday's date.

&#x20;

\### Hero Banner

Dark (`#0f1117`) hero with:

\- "Lansweeper Support Intelligence" eyebrow label (DM Mono, small caps)

\- Title: "Cases Dashboard / \[Date]" (DM Serif Display)

\- Four stat pills: Total Inbound · Real Cases · Filtered Noise · Unique Accounts

\### Section 01 — Top 20 by Active ARR

&#x20;

\*\*Section title (≤30 words, displayed below the section heading):\*\*

> "Highest-value customers who opened cases yesterday, ranked by live contract revenue. Expand any row to see individual case numbers and owners."

&#x20;

Accounts with active ARR > 0, sorted descending. Columns:

\- Rank · Account Name · \*\*Active ARR\*\* (with inline bar proportional to max) ·

&#x20; \*\*Expected ARR\*\* (pipeline, green) · Cases Yesterday · Open Jira (badge, clickable)

Each row is clickable to expand a sub-table showing every case opened:

`Case Number (linked to SF) | Subject | Owner`

&#x20;

The Salesforce case URL is: `https://lansweeper.my.salesforce.com/{Case.Id}`

&#x20;

\### Section 02 — Top 20 by Case Volume

&#x20;

\*\*Section title (≤30 words, displayed below the section heading):\*\*

> "Accounts generating the most noise yesterday with 30-day trend context. Expand any row to see individual case numbers and owners."

&#x20;

All accounts with real cases yesterday, sorted by count descending. Columns:

\- Rank · Account · Active ARR · Expected ARR · Yesterday · Last 30 Days · Trend · Open Jira

Same expandable case detail rows as Section 1.

&#x20;

\### Section 03 — Open Jira Tickets

&#x20;

\*\*Section title (≤30 words, displayed below the section heading):\*\*

> "Active product bugs and feature requests linked to customer accounts. Tickets marked Done or Canceled are excluded. Click any ticket to open it in Jira."

&#x20;

Cards for each account that has open Jira tickets. Show:

\- Account name · ticket count

\- Per ticket: ticket key (linked to `https://lansweeper.atlassian.net/browse/{KEY}`) · status badge · blocking flag if true

Jira badges in Section 1 \& 2 scroll to the corresponding card in Section 3 when clicked.

&#x20;

\---

&#x20;

\## Design System

&#x20;

Use these values consistently. Load fonts from Google Fonts CDN.

&#x20;

```css

\--bg:     #f5f4f0   /\* page background \*/

\--surf:   #ffffff   /\* card/table background \*/

\--dark:   #0f1117   /\* hero + section headers \*/

\--acc:    #2d6ef7   /\* links, Jira badges, bar fill \*/

\--grn:    #2dac6a   /\* Expected ARR column \*/

\--red:    #e84c4c   /\* high Jira counts (≥10), trend up \*/

\--txt:    #1a1a1a

\--muted:  #6b6b6b

\--bdr:    #e0dfd8

&#x20;

Fonts: DM Serif Display (headings), DM Sans (body), DM Mono (numbers, labels, case IDs)

```

&#x20;

Status dot colours for case sub-tables:

\- In Progress → blue `#2d6ef7`

\- Awaiting Reply → amber `#f5a623`

\- Open → purple `#9b59b6`

\- Pending \* → grey `#7f8c8d`

\- Assigned → steel `#3498db`

\- Resolved → green `#2dac6a`

\- Closed → light grey `#bdc3c7`

Jira status badge colours:

\- In Progress → `#e8f0ff / #1e4bb8`

\- To Do → `#f0efe9 / #5c5650`

\- New → `#eef8f0 / #1a6626`

\- On Hold → `#fdf0f0 / #b83232`

\- Testing / External Testing → `#fff7e6 / #925d00`

\- Reviewing → `#f5f0ff / #5a2db8`

\---

&#x20;

\## Step-by-Step Execution

&#x20;

1\. Run all four Salesforce queries in parallel (cases, opps, jira counts, jira details).

2\. Apply the noise filter to the cases result.

3\. Group real cases by AccountId; for each account collect: Active ARR, case list

&#x20;  (id, num, subject, status, owner), opp ARR, jira count.

4\. Build Section 1 list: accounts with ARR > 0, sorted by ARR desc, top 20.

5\. Build Section 2 list: all accounts sorted by case count desc, top 20.

&#x20;  Add 30-day counts and trend.

6\. Build Section 3 data: accounts with jira\_count > 0, de-duplicate ticket keys

&#x20;  (one Jira key may link to multiple cases — show it once).

7\. Derive the Top 10 Insights (see section below).

8\. Render the \*\*dashboard HTML\*\* using the design system above.

9\. Render the \*\*insights HTML\*\* as a second file (see section below).

10\. Save both files to `/mnt/user-data/outputs/`:

&#x20;   - `support\_cases\_dashboard\_{YYYY-MM-DD}.html`

&#x20;   - `support\_cases\_insights\_{YYYY-MM-DD}.html`

&#x20;   and call `present\_files` with both so the user can download them.

\---

&#x20;

\## Top 10 Insights Export

&#x20;

After generating the dashboard data, produce a second standalone HTML file containing

\*\*exactly 10 insights\*\* — synthesised observations drawn from the day's data. This is

a narrative intelligence layer on top of the raw tables.

&#x20;

\### How to derive insights

&#x20;

Look across all three data sets (cases, ARR, Jira) and identify the most notable

patterns and risk signals. Each insight should be actionable or decision-relevant.

Good insight types to look for:

&#x20;

\- \*\*High-value account in distress\*\* — top-ARR account with multiple cases + open Jiras

\- \*\*Spike vs baseline\*\* — account with yesterday's count far above 30-day average

\- \*\*Large pipeline at risk\*\* — account with high opp ARR opening support cases

\- \*\*Jira backlog concern\*\* — account with many open tickets (≥5) still opening new cases

\- \*\*Unowned / queue-assigned cases\*\* — high-value account whose cases landed in a queue

&#x20; rather than a named engineer

\- \*\*Noise pattern\*\* — if noise volume is unusually high, call it out

\- \*\*New account escalation\*\* — account not typically seen in recent 30 days opening cases

\- \*\*Cross-section signal\*\* — account appears in both top ARR and top volume lists

\- \*\*Blocking Jira\*\* — any account with `LAN\_Blocking\_Issue\_\_c = true` on an open ticket

\- \*\*Owner concentration\*\* — one engineer carrying a disproportionate share of cases

Rank insights roughly by urgency / revenue impact. Number them 1–10.

&#x20;

\### Insights HTML structure

&#x20;

A clean, printable single-page HTML file using the same design system (fonts, colours).

No interactive JS required — this is a read-only report.

&#x20;

\*\*Layout:\*\*

\- Same dark hero banner as the dashboard, but title is "Support Intelligence Brief / \[Date]"

\- Subtitle: "Top 10 insights derived from \[N] real cases across \[M] accounts"

\- 10 insight cards in a single-column layout, each containing:

&#x20; - \*\*Insight number\*\* (01–10) in DM Mono, accent colour

&#x20; - \*\*Insight title\*\* — a short bold headline (≤10 words)

&#x20; - \*\*Body\*\* — 2–4 sentences explaining the signal, the accounts involved, and

&#x20;   the recommended action or watch item

&#x20; - \*\*Signal tags\*\* — small pills showing relevant data points

&#x20;   (e.g. "ARR $555K", "3 cases", "12 open Jiras", "↑ above avg")

&#x20; - A subtle left-border colour that reflects severity:

&#x20;   - 🔴 Red `#e84c4c` — immediate attention (blocking Jira, very high ARR + multiple cases)

&#x20;   - 🟡 Amber `#f5a623` — monitor closely (spike, queue assignment, pipeline risk)

&#x20;   - 🔵 Blue `#2d6ef7` — informational (pattern, context, low urgency)

\*\*Example card:\*\*

&#x20;

```

01  CIVICA — HIGH JIRA BACKLOG WITH ACTIVE CASES

&#x20;   Civica (Active ARR $86K) has 13 open Jira tickets and opened 2 new cases yesterday,

&#x20;   the highest Jira count of any account in the queue. The ongoing backlog suggests

&#x20;   unresolved product issues are driving repeat contact. Recommend a dedicated engineering

&#x20;   review to clear blockers before the renewal window.

&#x20;   \[ARR $86K] \[2 cases] \[13 open Jiras] \[↑ 30d avg]

```

&#x20;

\---

&#x20;

\## Customisation Flags

&#x20;

If the user explicitly requests a different ARR field, use `LAN\_Total\_Account\_ARR\_\_c`

instead of `LAN\_Active\_ARR\_\_c`. Note in the dashboard footer which field was used.

&#x20;

If the user asks for a specific date instead of yesterday, adjust the `CreatedDate`

filter accordingly (e.g. `CreatedDate = 2026-05-11T00:00:00Z`).

&#x20;

If the user asks to include noise cases (e.g. "show everything"), skip the filter and

note it in the hero banner.

&#x20;

\---

&#x20;

\## Notes on Data Quirks

&#x20;

\- \*\*OCI Cloud SAS\*\* and similar accounts may generate OOO floods — these get filtered.

\- \*\*Genuit Group PLC\*\* generates large "Mailbox Not Monitored" floods — all filtered.

\- \*\*Sales Support cases\*\* (`RecordType.Name == 'Sales Support'`) are excluded at both the SOQL query level and the noise filter. These are handled by sales, not support, and should never appear in the dashboard.

\- Some accounts have `LAN\_Active\_ARR\_\_c = 0` (not null) — these are real accounts but

&#x20; with no current contract revenue. Include them in Section 2 but exclude from

&#x20; Section 1 (ARR section requires ARR > 0).

\- Jira `LAN\_Jira\_Case\_Id\_\_c` is stored as raw HTML. Parse the href and link text.

&#x20; The same Jira key may appear multiple times (once per linked case) — deduplicate

&#x20; by key when rendering Section 3 cards. Report the SF record count vs. unique key

&#x20; count in the card header when they differ.

\- `Owner.Name` can be a queue (e.g. "Enterprise EMEA Support Queue", "AMER Support

&#x20; Queue", "Sales Queue"). Display as-is.

