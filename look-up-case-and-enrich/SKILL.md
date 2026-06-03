---
name: look-up-case-and-enrich
description: >
  Looks up a Salesforce support case by case number and returns a fully enriched
  case object ready for downstream analysis modules. Use this skill whenever someone
  provides a case number and wants to investigate, analyse, or find solutions for a
  support case — even if they phrase it casually like "look up case 12345", "what's
  going on with case 00123456", "pull this case", "enrich case number X", "load case
  details", or "I want to find solutions for case X". Always trigger this skill as
  the first step before any case analysis, solution ranking, or issue classification
  workflow. This is Module 1 of the case solution ranking system.
---

# Look Up Case and Enrich — Module 1

Retrieves a Salesforce support case by case number and enriches it with full account
context, complete case thread, and all metadata needed by downstream modules. This is
the foundation layer — all other modules (issue classification, solution ranking,
Jira lookup, etc.) depend on the structured output this module produces.

---

## Step-by-Step Execution

### Step 1 — Resolve Case Number to Case ID

Salesforce case numbers are human-readable (e.g. `00123456`). Resolve to the internal
record ID first:

```soql
SELECT Id, CaseNumber
FROM Case
WHERE CaseNumber = '{INPUT_CASE_NUMBER}'
LIMIT 1
```

If no result is returned, tell the user: "Case number {X} was not found in Salesforce.
Please verify the case number and try again." Then stop.

---

### Step 2 — Pull Core Case Fields

Using the resolved Case Id, fetch full case metadata:

```soql
SELECT
  Id,
  CaseNumber,
  Subject,
  Description,
  Status,
  Priority,
  Type,
  Origin,
  CreatedDate,
  ClosedDate,
  LastModifiedDate,
  AccountId,
  Account.Name,
  Account.LAN_Active_ARR__c,
  Account.Type,
  Account.Industry,
  Account.BillingCountry,
  OwnerId,
  Owner.Name,
  ContactId,
  Contact.Name,
  Contact.Email,
  LAN_Jira_Case_Link__c,
  LAN_Number_of_Customer_Answers__c,
  LAN_CSAT_Score__c,
  LAN_Product_Version__c,
  LAN_Product_Area__c,
  LAN_Lansweeper_Version__c
FROM Case
WHERE Id = '{CASE_ID}'
LIMIT 1
```

> **Field notes:**
> - `LAN_Active_ARR__c` — live contract revenue. Use this field for ARR; do not use
>   `LAN_Total_Account_ARR__c` unless explicitly requested.
> - `LAN_Number_of_Customer_Answers__c` — total customer message count, used for
>   activity weighting in downstream modules.
> - `LAN_CSAT_Score__c` — CSAT score if case is closed. May be null for open cases.
> - `LAN_Product_Version__c` / `LAN_Lansweeper_Version__c` — capture both fields;
>   downstream modules use this for version-match weighting.
> - `LAN_Product_Area__c` — product area tag if set by the agent.

---

### Step 3 — Pull Full Case Thread (Emails + Comments)

⚠️ **MANDATORY CHECKLIST: All three queries below (EmailMessages, CaseComments, CaseFeed) MUST be executed before proceeding to Step 4. Do not skip any query. After executing each query, explicitly confirm it is complete before running the next.**

Fetch all customer-facing emails and internal comments in chronological order. These
are the raw inputs for issue classification in Module 3.

**EmailMessages (customer-facing thread):**
```soql
SELECT
  Id,
  Subject,
  TextBody,
  FromAddress,
  FromName,
  ToAddress,
  MessageDate,
  Incoming,
  Status
FROM EmailMessage
WHERE ParentId = '{CASE_ID}'
ORDER BY MessageDate ASC
LIMIT 100
```

**CaseComments (internal notes):**
```soql
SELECT
  Id,
  CommentBody,
  CreatedDate,
  CreatedBy.Name,
  IsPublished
FROM CaseComment
WHERE ParentId = '{CASE_ID}'
ORDER BY CreatedDate ASC
LIMIT 50
```

**CaseFeed (activity + chatter posts):**
```soql
SELECT
  Id,
  Type,
  Body,
  CreatedDate,
  CreatedBy.Name
FROM CaseFeed
WHERE ParentId = '{CASE_ID}'
  AND Type IN ('TextPost', 'CreateRecordEvent', 'ChangeStatusPost')
ORDER BY CreatedDate ASC
LIMIT 50
```

Merge all three into a single chronological thread sorted by date. Label each entry
with its source: `[EMAIL-IN]`, `[EMAIL-OUT]`, `[COMMENT-INTERNAL]`, `[COMMENT-PUBLIC]`,
`[FEED]`.

---

### Step 4 — Pull Linked Jira Tickets

Check whether this case already has Jira tickets linked:

```soql
SELECT
  Id,
  LAN_Jira_Case_Id__c,
  LAN_Jira_Case_Status__c,
  LAN_Blocking_Issue__c
FROM LAN_Jira_Case_Link__c
WHERE LAN_Case__c = '{CASE_ID}'
LIMIT 20
```

Parse `LAN_Jira_Case_Id__c` — it is stored as an HTML anchor tag. Extract the Jira
key (e.g. `LAN-12345`) and the URL from the href.

---

### Step 5 — Pull Account Case History Summary

Fetch a summary count of past cases for this account to give downstream modules
context about case volume and frequency:

```soql
SELECT COUNT(Id) total_cases, MAX(CreatedDate) most_recent
FROM Case
WHERE AccountId = '{ACCOUNT_ID}'
  AND Id != '{CASE_ID}'
```

Also fetch the 5 most recent prior cases for surface context:

```soql
SELECT Id, CaseNumber, Subject, Status, CreatedDate,
       LAN_CSAT_Score__c, LAN_Product_Area__c, LAN_Product_Version__c
FROM Case
WHERE AccountId = '{ACCOUNT_ID}'
  AND Id != '{CASE_ID}'
ORDER BY CreatedDate DESC
LIMIT 5
```

---

### Step 6 — Assemble the Enriched Case Object

Combine all data into a single structured object. This object is passed to all
downstream modules. Store it clearly so it can be referenced throughout the session.

```
ENRICHED CASE OBJECT
─────────────────────────────────────────────
CASE IDENTITY
  Case Number       : {CaseNumber}
  Case ID           : {Id}
  Subject           : {Subject}
  Status            : {Status}
  Priority          : {Priority}
  Type              : {Type}
  Origin            : {Origin}
  Created           : {CreatedDate}
  Closed            : {ClosedDate or "Open"}
  Last Modified     : {LastModifiedDate}
  CSAT Score        : {LAN_CSAT_Score__c or "N/A — case open or not rated"}
  Customer Messages : {LAN_Number_of_Customer_Answers__c}

PRODUCT CONTEXT
  Product Area      : {LAN_Product_Area__c or "Not set"}
  Product Version   : {LAN_Product_Version__c or "Not set"}
  Lansweeper Version: {LAN_Lansweeper_Version__c or "Not set"}

ACCOUNT CONTEXT
  Account Name      : {Account.Name}
  Account ID        : {AccountId}
  Account Type      : {Account.Type}
  Industry          : {Account.Industry}
  Country           : {Account.BillingCountry}
  Active ARR        : {Account.LAN_Active_ARR__c}
  Total Prior Cases : {total_cases}
  Most Recent Prior : {most_recent}

PEOPLE
  Case Owner        : {Owner.Name}
  Contact           : {Contact.Name} <{Contact.Email}>

LINKED JIRA TICKETS
  {For each ticket: KEY | Status | Blocking: Yes/No}
  {or "None linked"}

RECENT ACCOUNT CASE HISTORY (last 5)
  {CaseNumber | Subject | Status | Created | CSAT | Product Area | Version}

FULL CASE THREAD ({total message count} messages)
  {Chronological thread with source labels}
─────────────────────────────────────────────
```

---

## Output to User

After assembling the enriched case object, present a **clean summary** to the user
before proceeding to any downstream module. Format as follows:

```
📋 CASE {CaseNumber} — {Subject}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Account     : {Account.Name}  |  ARR: ${LAN_Active_ARR__c}
Status      : {Status}  |  Priority: {Priority}  |  Owner: {Owner.Name}
Opened      : {CreatedDate}   |  Messages: {LAN_Number_of_Customer_Answers__c}
Version     : {LAN_Lansweeper_Version__c or "Not set"}
Product Area: {LAN_Product_Area__c or "Not set"}
CSAT        : {LAN_CSAT_Score__c or "Open / not yet rated"}
Jira Linked : {count of linked tickets, or "None"}
Prior Cases : {total_cases} total for this account

Case thread loaded: {total message count} messages across emails and comments.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Case enrichment complete. Proceeding automatically to Module 2 (Account Context).
```

---

## Error Handling

| Condition | Behaviour |
|---|---|
| Case number not found | Stop and inform user. Do not proceed. |
| Account ID is null | Warn user — account context and history modules will be skipped downstream. |
| No email messages found | Note in output. Thread may be description-only. |
| Version fields both null | Flag as "Version unknown" — downstream version-match scoring will not apply. |
| Jira object query fails | Note "Jira link data unavailable" — do not fail the whole enrichment. |

---

## Auto-Chain Instruction

After presenting the enriched case summary to the user, **immediately and automatically proceed to run Modules 2 through 12 in sequence without waiting for user confirmation between steps.** Do not pause, ask for direction, or wait for the user to prompt the next module. The full pipeline should run end to end from a single case number input.

---

## Notes on Salesforce Field Quirks

- `LAN_Jira_Case_Id__c` is raw HTML — always parse the `href` and link text; never
  display the raw HTML to the user.
- `Owner.Name` may be a queue name (e.g. "Enterprise EMEA Support Queue") rather than
  a person. Display as-is and note if it appears to be unassigned to a named engineer.
- `LAN_CSAT_Score__c` will be null for all open cases — this is expected, not an error.
- `LAN_Number_of_Customer_Answers__c` counts customer replies only, not agent messages.
  A high number signals a complex or unresolved case.
- Some accounts have `LAN_Active_ARR__c = 0` (not null) — this is a real account with
  no current contract revenue, not a data error.
