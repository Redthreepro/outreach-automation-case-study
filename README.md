# Compliance-guarded outreach automation — case study

**A listing-triggered email automation for a home-inspection company: the v1 built with every guardrail first and deliberately held back, and the production rebuild on n8n and Google Apps Script that has run since January 2026.**
Ryan Faber / Red Three Pro. Built for a Michigan home-inspection company. Private.

> Documentation only. Source and data are private; no agent, customer, or listing data appears here. Numbers for the production system come from the email provider's statistics page and the system's own configuration and dashboard sheets, captured September 2026. Numbers for v1 come from its repository and database.

**Status:** in production. 4,019 emails sent between January 4 and August 9, 2026 at 94.97% delivery and 0.00% spam complaints, with a spreadsheet control plane the office can operate and a kill switch that has been exercised.

---

## The problem

When a home goes under contract, the buyer needs an inspection within days, and the listing agent often influences who gets called. The company already received MLS "new pending" notifications. Put a well-and-septic coupon in front of the right agent at exactly that moment, automatically, without ever becoming the company that spams agents. The referral relationships are worth more than any campaign.

The twenty-line version of this sends email. The version that could be trusted to run unattended needed brakes before it needed an engine.

## Part 1: v1, built with the brakes first (December 2025)

A small Node service, seven source files, one job.

```mermaid
flowchart LR
  E[MLS 'new pending' email] --> P[Parse<br/>MLS #, address,<br/>city/state/ZIP, price]
  P -->|parse_failed| X1[Stop, log reason]
  P --> Z{ZIP in<br/>service area?}
  Z -->|no| X2[out_of_service_area]
  Z --> A[Match listing agent<br/>against directory]
  A -->|agent_not_found| X3[Stop, log reason]
  A --> S[Send policy]
  S -->|send_disabled| X4[Blocked]
  S -->|suppressed| X5[Blocked]
  S -->|daily_cap_reached| X6[Blocked]
  S -->|agent_cooldown_active| X7[Blocked]
  S -->|allowed| M[Transactional send]
  M --> L[(SQLite<br/>sends, suppression)]
```

**Send policy, four checks in order, each logged with a reason when it blocks:** a kill switch that must literally equal `true` (off is the default); a suppression list with admin endpoints; a daily cap across all recipients (25 by default); a per-agent cooldown (21 days by default). Every non-send returns a structured reason, so a batch of notifications reads as a table of outcomes. A demo mode redirects every send to an internal address so the pipeline can run against real notifications with nobody receiving anything.

**Why v1 was held back.** The final integration needed the service to read a mailbox, and which mailbox, whose credentials, and what else that exposed were the company's decisions, not mine. And the guardrails covered volume, frequency, and opt-out but not consent, sender identification, or disclosure language. So v1 stopped one integration short: one recorded send, a test on 2025-12-15, kill switch off since. Adding a suppression list after the first angry reply means the first angry reply already happened.

## Part 2: the production rebuild (January 2026 to present)

Once the mailbox and compliance questions had answers, the system was rebuilt with the v1 guardrails as the baseline rather than a rewrite. It is two halves: **self-hosted n8n** ingests and qualifies listings, and **Google Apps Script inside the control workbook** runs every check, sends, logs, and reports. Three design choices define it.

```mermaid
flowchart LR
  subgraph n8n["Self-hosted n8n (every 2 h, 7 AM–9 PM)"]
    T[Schedule] --> C1[Read config sheet<br/>stop if disabled]
    C1 --> S[Listing scraper]
    S --> N[Normalize]
    N --> W[Well-records lookup<br/>local service]
    W --> F[Filter]
  end
  F -->|append| L[(Listings tab)]
  subgraph GS["Google Sheets + Apps Script (every 30 min)"]
    L --> RP[runPipeline]
    K[(Config sheet)] --> RP
    DNC[(Agent DB · DNC)] --> RP
    RP --> G{Guardrails<br/>switches · window · caps<br/>domain limit · cooldown · drive time}
    G -->|send| E[Email provider]
    G -->|hold / skip + reason| L
    E --> L
    RP --> D[(Dashboard sheet)]
    DS[sendDailySummary] --> O[Office inbox]
    AS[dailyArchiveSweep] --> AR[(Archive)]
  end
  CP[Control panel sidebar<br/>kill switch · status · DNC · stats] --- K
```


### A spreadsheet is the control plane

Every operating parameter lives in a configuration sheet the office can read and edit without touching a workflow. The workflow reads it on every run.

![Guardrail configuration sheet](images/03-guardrail-config.png)

What the sheet controls, as captured in September 2026:

| Control | Setting |
|---|---|
| Sending enabled, automation enabled, n8n enabled | three independent switches, all must be on |
| System kill switch, manual pause | two ways to stop, one for emergencies and one for "not today" |
| Test mode, dry-run mode | run the whole pipeline and send nothing, or send only to an internal address |
| Send window | 07:00 to 21:00, active days configurable |
| Max per run, max per day | 40 per run, 100 per day |
| Per-domain daily limit | 25, so no single brokerage domain gets flooded |
| Cooldown | 7 days per agent |
| Drive-time qualification | 90-minute hard cap, drive-time lookups capped per run |
| External API budget | calls per run capped, with a sleep between calls |
| Ramp-up mode | start date and a rising daily cap, so volume grows gradually rather than arriving all at once |
| Whitelist domains, daily summary | recipients for the daily digest; send-to-self domain for testing |

### Service area is data, not code

A county table with region and an active flag decides eligibility. Turning a county on or off is a cell edit.

![Service-area county table](images/02-service-area-counties.png)

### The operator sees the system's state at a glance

A dashboard sheet refreshes after each run: capacity (ramp-up day, daily cap, sent today, remaining), backlog by status, last run and errors, today's deliverability (sent, hard bounces, spam complaints), and the three system settings that matter. When the kill switch is on, the header says so in red.

![Control dashboard](images/01-control-dashboard.png)

### The workflows

**Listing ingest.** Runs every two hours between 7 AM and 9 PM, seven days a week. It reads the configuration sheet first and stops if automation is off. Otherwise it calls a hosted scraper for new listings, normalizes each one, checks it against a locally hosted well-records lookup (the well/septic qualification), merges the result, filters, and appends qualified listings to the Listings tab that the send workflow consumes.

![Listing-ingest workflow](images/05-workflow-listing-ingest.png)

Each qualified listing lands in the Listings tab as one row. The 59 columns, grouped (names only; the rows are real properties and stay private):

| Group | Columns | What it tells you |
|---|---|---|
| Listing | Address, City, State, ZIP, Postal_Code, County, Latitude, Longitude, Price, List_Price, MLS_Number, MLS_Status, Listing_URL, Property_URL, Property_ID, List_Date, Beds, Baths, Sqft, Property_Type, Photo | the normalized listing |
| Agent | Agent_Name, Agent_Email, Agent_Phone, Office_Name | the recipient, matched from the agent database |
| Well qualification | well_nearby, nearest_well_id, distance_feet, threshold_feet, counties_searched, county_resolved_from | the local well-records lookup writes back the nearest well, its distance, and the threshold it was judged against, plus which county tables were searched and how the county was resolved |
| Routing | Closest_Inspector, Backup_Inspector, Closest_Drive_Min, Backup_Closest_Drive_Min, Drive_Min | nearest and backup inspector by real drive time |
| Eligibility and state | Eligible, Ineligible_Reason, Status, Status_Updated_At, Processing_Lock, Outreach_Key, Sent_Timestamp, Notes, Fulfillment_ID | the state machine: a reason for every ineligible row, a lock so two runs can't process the same row, and a dedupe key so an agent is never emailed twice for one listing |
| Click tracking | Track_Yes_URL, Track_Meet_URL, Track_No_URL, Clicked_Action, Clicked_At | three tracked calls to action per email (yes, let's meet, no), so the response is measured per action, not just "opened" |
| Provenance | Source, Date_Imported, Imported_At, Imported_From, Imported_Batch_ID | which scrape produced the row |
| Archive | Archived_Flag, Archived_At, Archived_Timestamp | the nightly sweep's bookkeeping |

Three of those groups are the difference between a script and a system. A processing lock and a dedupe key are what let a 30-minute cron run safely against a shared sheet. An explicit ineligible-reason column is why the daily summary is readable. And three tracked links per email means the 3.35% click rate above is broken down by what the agent actually chose.

Two things this canvas shows without any data on it: the enable check is the second node, before anything external is called, and the well lookup is a local service, so the qualification that matters most doesn't depend on a third party.

### The well-records lookup: a statewide public dataset made queryable

The company's coupon is for well-and-septic evaluations, so the single most valuable qualification is "does this property have a well." Michigan publishes water-well records as per-county GIS exports. Those exports, 39 counties covering the service area with a lithology table alongside each, were loaded onto the same machine as a locally hosted lookup service that the ingest workflow queries per listing. No per-listing API cost, no rate limit, no third party in the path.

![Well-records dataset by county](images/06-well-records-counties.png)

![One county's shapefile set](images/07-well-records-county-shapefile.png)

A single county's well table runs to tens of megabytes; the largest in the service area is over 50 MB with a 75 MB lithology table. Michigan has no statewide septic registry, so the septic side of the qualification comes from the well side and the listing data rather than a records lookup.

**Send, guardrails, and operations: Apps Script.** Everything after ingest lives in the workbook's script project, a 20-plus-file codebase whose file list reads as the feature list: agent database and new-agent handling, do-not-contact, drive time, emails and email testing, MLS API, market controls, click tracking, audit, archive and archive de-duplication, daily summary, dashboard, kill-switch UI, and a control panel served as a sidebar and a full page.

![Apps Script project](images/08-apps-script-project.png)

The file open in that screenshot is the menu builder, and its comment records a lesson worth keeping: the custom menu must render even if reading the kill-switch status throws, because a kill switch the operator can't reach is the worst possible failure mode for a kill switch. The menu gives the office one-click access to toggle sending, check system status, open the control panel, preview the email template, send a test to a custom address, and run the maintenance jobs.

Three time-driven triggers run the system:

![Triggers](images/09-apps-script-triggers.png)

| Function | Cadence | Does |
|---|---|---|
| `runPipeline` | every 30 minutes | for each listing at status Send: drive-time qualification, do-not-contact, cooldown, send window, per-run, per-day and per-domain caps; send through the provider; write status, timestamp, and skip reason back; refresh the dashboard |
| `sendDailySummary` | nightly | the day's sends, holds, skips, errors, and deliverability to the office |
| `dailyArchiveSweep` | nightly | move finished rows to the archive and de-duplicate |

The execution log shows the cadence and the cost: `runPipeline` every half hour on the minute, 17 to 25 seconds per run, completing every time in the window shown. The `cp_` functions are the control-panel sidebar loading settings, inspectors, the DNC list, stats, and the activity log when the office opens it.

![Executions](images/10-apps-script-executions.png)

## Results

Provider statistics for January 4 through August 9, 2026:

![Provider statistics](images/04-provider-statistics.png)

| Metric | Value |
|---|---|
| Emails sent | **4,019** |
| Delivered | **94.97%** |
| Estimated openers | 52.79% (trackable 36.95%) |
| Unique clickers | 3.35% |
| Hard bounce | 2.04% |
| Soft bounce | 2.84% |
| Blocked | 0.30% |
| **Spam complaints** | **0.00%** |

The daily chart shows the shape the guardrails produce: sends arrive in capped daily plateaus (the 40-per-run and 100-per-day limits, later raised under ramp-up), with gaps where the send window, the weekend schedule, or a pause held them back. A zero complaint rate across four thousand cold emails to real estate agents is the number the whole design was built to protect.

Booked inspections attributable to the campaign are tracked in the company's scheduling system and are not published here.

## Decisions worth explaining

**Guardrails before the first send, in both versions.** v1's four checks became the production system's floor. The production system added the ones v1 lacked: per-domain limits, a send window, ramp-up, and a dry-run mode. Nothing was removed.

**The spreadsheet is the application.** The office manager will never open n8n, but they open this workbook every day. So the workbook is where the pipeline runs (Apps Script), where it's configured (a KEY/Value sheet), where it's monitored (the dashboard), and where it's controlled (a custom menu and a sidebar with the kill switch). n8n is only used for the part Sheets can't do well: scheduled scraping and a call to a local service. A pause, a cap change, or a kill can happen from a phone.

**Fail closed, three switches deep.** Sending, automation, and the workflow engine each have their own enable flag, plus a kill switch and a manual pause above them. Any one of five things off means nothing sends.

**Ramp-up instead of launch.** Volume started small on a fixed date and rose on a schedule. Deliverability reputation is built, not declared, and a sudden spike from a fresh sender is how legitimate mail gets classified as spam.

**Reasons, not booleans.** Every skip is logged with why. The daily summary is readable because of it.

## What's not done

- Per-function logic inside the Apps Script project is described from file and function names, not from a line-by-line review.
- v1's caps skip the check when unset rather than failing closed. The production system's three-switch design supersedes it.
- Consent and disclosure handling is present in the production system but not documented here until the export review.

## By the numbers

| | |
|---|---|
| v1 | 7 source files, 4 policy checks, 8 structured outcomes, 1 test send (2025-12-15), never launched |
| Production | in operation January 2026 to present: self-hosted n8n (ingest) + Google Apps Script (everything else) |
| Pipeline cadence | `runPipeline` every 30 minutes, 17–25 s per run; nightly summary and archive sweep |
| Script project | 20+ files: agent DB, DNC, drive time, emails, MLS API, market controls, clicks, audit, archive, dashboard, kill-switch UI, control panel |
| Sent | 4,019 (Jan 4 to Aug 9, 2026) |
| Delivered / hard bounce / complaints | 94.97% / 2.04% / 0.00% |
| Guardrails | 5 stop controls, 4 volume caps, send window, 7-day cooldown, 90-minute drive cap, ramp-up |
| Control plane | 3 spreadsheets: configuration, county coverage, dashboard; a 59-column Listings tab as the state store |
| Ingest cadence | every 2 hours, 7 AM to 9 PM, 7 days |
| Well-records dataset | 39 counties of state GIS exports, hosted locally as a lookup service |

## How it was built

I defined the workflow, the qualification rules, the guardrails and their order, the spreadsheet-as-the-application design, the n8n/Apps Script split, and the decision to hold v1 until the compliance posture was right. AI coding agents wrote the v1 service, the Apps Script project, and the n8n workflows under that direction. I verified v1 end to end in demo mode before deciding not to launch it, and I operate the production system, watching the daily summary and the deliverability numbers.

---

Part of the [Red Three Pro portfolio](https://github.com/Redthreepro). See also: [Field-inspection AI platform](https://github.com/Redthreepro/inspection-ai-platform-case-study). Questions: redthreepro@gmail.com
