
# USCIS Insights Hub

> **An internal employer intelligence and case monitoring platform built on public USCIS and DOL data — zero licensing cost beyond Microsoft 365.**

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Modules](#modules)
3. [Tech Stack](#tech-stack)
4. [Data Sources](#data-sources)
5. [Data Model](#data-model)
6. [Architecture](#architecture)
7. [Power BI Report Structure](#power-bi-report-structure)
8. [Power Apps](#power-apps)
9. [Power Automate Flows](#power-automate-flows)
10. [Project To-Do List & Timeline](#project-to-do-list--timeline)
11. [Portfolio Pitch](#portfolio-pitch)
12. [Resources & Links](#resources--links)

---

## Project Overview

USCIS Insights Hub combines two powerful intelligence modules into a single Microsoft Power Platform solution:

- **Module 1 — Employer Intel:** Who is sponsoring H-1B workers, at what approval rate, at what wage level, and how often are they getting denied?
- **Module 2 — Processing Watch:** How long is USCIS currently taking per form type and field office — and how has that changed over time?

This project is designed to demonstrate end-to-end Power Platform proficiency (Power BI + Power Apps + Power Automate), domain expertise in U.S. employment immigration, and the ability to build a production-quality internal tool using only free public government data and an existing Microsoft 365 license.

---

## Modules

| Module | Primary Dataset | Primary Tool | Business User |
|---|---|---|---|
| Employer Intel | DOL OFLC + USCIS H-1B Employer Hub | Power BI + Power Apps | HR, Talent Acquisition, Immigration Counsel |
| Processing Watch | USCIS Processing Times (weekly snapshots) | Power BI + Power Automate | Case Managers, Immigration Attorneys, Compliance Teams |

---

## Tech Stack

| Layer | Tool | Role |
|---|---|---|
| Analytics & Visualization | Power BI Desktop / Service | 6-page interactive report |
| End-User Application | Power Apps (Canvas App) | Employer lookup + Case estimator |
| Automation & Ingestion | Power Automate (Cloud Flows) | Scheduled data pulls, alerts, PDF generation |
| Data Storage | SharePoint Lists / OneDrive Excel | Free, M365-native, connectable to all three tools |
| Data Source | DOL.gov, USCIS.gov (public CSVs) | Zero-cost public government data |

> **Free Tier Note:** Register for a [Microsoft 365 Developer Tenant](https://developer.microsoft.com/en-us/microsoft-365/dev-program) (free, renewable every 90 days) to unlock full E5 licenses including Power BI Pro, premium Power Automate connectors, and Power Apps per-app plans for up to 25 users.

---

## Data Sources

| Dataset | URL | Format | Update Cadence | Module |
|---|---|---|---|---|
| DOL OFLC H-1B Performance Data | https://www.dol.gov/agencies/eta/foreign-labor/performance | CSV | Quarterly | Module 1 |
| USCIS H-1B Employer Data Hub | https://www.uscis.gov/tools/reports-and-studies/h-1b-employer-data-hub | CSV | Quarterly | Module 1 |
| USCIS Processing Times | https://egov.uscis.gov/processing-times/ | Manual / Scraped | Weekly | Module 2 |
| USCIS Form Performance Data | https://www.uscis.gov/tools/reports-and-studies/immigration-forms-data | Excel | Quarterly | Module 2 (supporting) |
| DHS Yearbook of Immigration Statistics | https://www.dhs.gov/immigration-statistics/yearbook | Excel | Annual | Module 2 (macro page) |

---

## Data Model

### Table 1: `OFLC_FILINGS`
Sourced from DOL OFLC quarterly H-1B performance files.

| Column | Type | Notes |
|---|---|---|
| `case_number` | Text (PK) | Unique LCA identifier |
| `employer_name` | Text | Clean/normalize for joins |
| `employer_state` | Text | 2-letter abbreviation |
| `employer_city` | Text | |
| `naics_code` | Text | Industry classification |
| `soc_code` | Text | Occupational classification |
| `job_title` | Text | As filed |
| `wage_rate_offered` | Decimal | Annual |
| `prevailing_wage` | Decimal | DOL benchmark |
| `wage_level` | Integer | 1–4 (compliance signal) |
| `case_status` | Text | Certified / Denied / Withdrawn |
| `decision_date` | Date | |
| `fiscal_year` | Integer | |

---

### Table 2: `H1B_EMPLOYER_HUB`
Sourced from USCIS H-1B Employer Data Hub quarterly files.

| Column | Type | Notes |
|---|---|---|
| `employer_name` | Text (FK → OFLC) | Fuzzy-matched on ingest |
| `fiscal_year` | Integer | |
| `initial_approvals` | Integer | |
| `initial_denials` | Integer | |
| `continuing_approvals` | Integer | |
| `continuing_denials` | Integer | |
| `naics_code` | Text | |

---

### Table 3: `PROCESSING_TIMES_SNAPSHOT`
Built and maintained by Power Automate Flow 1. This is original data — no public historical archive exists.

| Column | Type | Notes |
|---|---|---|
| `snapshot_date` | Date (PK composite) | Date the snapshot was captured |
| `form_type` | Text (PK composite) | I-485, I-130, N-400, I-140, etc. |
| `field_office` | Text (PK composite) | Field office or service center name |
| `processing_time_low` | Decimal | Months (lower bound) |
| `processing_time_high` | Decimal | Months (upper bound) |
| `form_category` | Text | Family / Employment / Humanitarian / Naturalization |

---

### Table 4: `PROCESSING_TIME_ALERTS`
Written by Power Automate; read by Power BI and Power Apps.

| Column | Type | Notes |
|---|---|---|
| `alert_id` | Text (PK) | Auto-generated |
| `form_type` | Text | |
| `field_office` | Text | |
| `alert_date` | Date | |
| `previous_time` | Decimal | Months |
| `new_time` | Decimal | Months |
| `delta_months` | Decimal | Positive = slower, Negative = faster |
| `alert_type` | Text | Spike / Improvement / SLA_Breach |

---

## Architecture

```
[DOL.gov CSVs]          [USCIS.gov Processing Times]
       │                          │
       ▼                          ▼
[Power Automate]────────[Power Automate]
  Flow 2: Quarterly         Flow 1: Weekly
  OFLC Ingestor             Snapshot Archiver
       │                          │
       ▼                          ▼
[SharePoint Lists]────────────────────────────────────┐
  OFLC_FILINGS                                        │
  H1B_EMPLOYER_HUB                                    │
  PROCESSING_TIMES_SNAPSHOT                           │
  PROCESSING_TIME_ALERTS                              │
       │                                              │
       ├──────────────────────┬───────────────────────┘
       ▼                      ▼
[Power BI Report]      [Power Apps]
  6-page dashboard       Employer Lookup Tool
  DAX measures           Case Processing Estimator
  Drill-through               │
  Embedded in Apps            ▼
                        [Power Automate]
                          Flow 3: Alert Notifier
                          Flow 4: PDF Generator
```

---

## Power BI Report Structure

### Page 1 — Executive Overview
- KPI Cards: Total H-1B Filings YTD | National Approval Rate % | Avg Processing Time (I-485) | Active Alerts
- Clustered Bar: Top 10 Sponsoring Employers (approvals vs. denials split)
- Line Chart: Processing time trend (last 52 weeks) for top 3 form types
- Slicers: Fiscal Year | State | Industry

### Page 2 — Employer Intelligence Deep Dive
- Matrix: Employer × Fiscal Year — approval rate, denial rate, avg wage offered vs. prevailing
- Scatter Plot: Wage Level (X) vs. Denial Rate (Y), sized by filing volume
- Map Visual: H-1B filing volume by employer HQ state, colored by approval rate
- Drill-through enabled → Page 3

### Page 3 — Employer Profile (Drill-Through)
- YOY trend line: approvals vs. denials over 5 fiscal years
- Top 5 SOC codes / job titles sponsored
- Wage distribution histogram vs. national prevailing wage benchmark
- Wage level breakdown (Level I–IV compliance signal)
- Key DAX: `Denial Rate = DIVIDE([Initial Denials], [Initial Approvals] + [Initial Denials])`

### Page 4 — Processing Time Dashboard
- Line Chart: Processing time over time per form type (full historical archive)
- Heatmap Table: Form × Office with conditional formatting (Green < 6mo / Yellow 6–12mo / Red 12+mo)
- Bar Chart: Longest current processing times ranked
- Slicer: Form Category (Family / Employment / Naturalization / Humanitarian)

### Page 5 — Alert History & Anomalies
- Table Visual: All alerts — date, form, office, delta, type (sortable/filterable)
- Bar Chart: Alert frequency by form type (most volatile forms)
- KPI Cards: Alerts This Month | Biggest Single Spike | Most Improved Form
- *Entirely auto-populated by Power Automate — zero manual refresh needed*

### Page 6 — Workforce Policy Macro View
- H-1B cap utilization by year (65K cap + 20K master's exemption)
- Approval rate by country of birth (India, China, Mexico, Other) — DHS Yearbook
- Naturalization trend: applications vs. ceremonies completed
- Executive-level context connecting employer data to national immigration policy

---

## Power Apps

### App 1: Employer Lookup Tool

| Screen | Description |
|---|---|
| Search | Text input for employer name; dropdowns for State, Industry, Fiscal Year; results gallery with approval rate, avg wage level, trend arrow |
| Employer Profile Card | KPI row, embedded Power BI tile, wage compliance indicator (Green/Yellow/Red), Export PDF button |
| Benchmarking View | Side-by-side comparison of up to 3 employers; "Send Comparison Report" triggers Power Automate email |

### App 2: Case Processing Estimator

| Screen | Description |
|---|---|
| Estimator Input | Dropdowns for form type and field office; date picker for receipt date |
| Estimate Result | Current processing window, estimated completion date range, historical sparkline, status flag (On Track / Extended / Significantly Delayed), "Set Alert" button |
| My Alerts | Gallery of user-set alerts; current status vs. alert creation date; weekly email toggle per alert |

---

## Power Automate Flows

### Flow 1: Weekly Processing Time Archiver
- **Trigger:** Recurrence — every Sunday at 8:00 AM
- **Logic:** HTTP GET from USCIS processing times → parse response → for each form/office, check if snapshot exists for this week → if not, create new SharePoint item → calculate delta vs. prior week → if delta > 2 months, create alert record → send summary email → trigger Power BI dataset refresh

### Flow 2: Quarterly OFLC Data Ingestor
- **Trigger:** Recurrence — 1st of each month
- **Logic:** Check DOL OFLC page for new quarterly file → compare to last ingestion date in config list → if new data, download CSV → parse and clean → batch create items in SharePoint → update config → send notification email → trigger Power BI refresh

### Flow 3: User Alert Notifier
- **Trigger:** Recurrence — every Monday at 7:00 AM
- **Logic:** Get all alerts from last 7 days → get all user subscriptions from Power Apps → match subscriptions to relevant form/office alerts → send personalized email per user with processing time delta

### Flow 4: Employer Profile PDF Generator
- **Trigger:** Instant (HTTP request from Power Apps button)
- **Logic:** Receive employer name → query SharePoint for employer data → compose HTML summary → convert to PDF via OneDrive → send email to requestor with PDF attachment

---

## Project To-Do List & Timeline

> Check off each item as you complete it. Items are grouped by week and ordered by dependency.

---

### ✅ Phase 0 — Environment Setup
- [ ] Register for [Microsoft 365 Developer Tenant](https://developer.microsoft.com/en-us/microsoft-365/dev-program) (free E5, 90-day renewable)
- [ ] Confirm Power BI Desktop installed locally
- [ ] Confirm Power Apps access via M365 tenant
- [ ] Confirm Power Automate access via M365 tenant
- [ ] Create project folder structure: `/USCIS-Insights-Hub/data/raw`, `/cleaned`, `/sharepoint-schemas`, `/powerbi`, `/powerapps`, `/automate`
- [ ] Create a SharePoint site for the project (e.g., "USCIS Insights Hub") to house all lists and files

---

### 📦 Phase 1 — Data Acquisition & Cleaning (Week 1)

**DOL OFLC Data**
- [ ] Download latest H-1B performance data CSV from [DOL OFLC](https://www.dol.gov/agencies/eta/foreign-labor/performance)
- [ ] Download 2–3 prior fiscal year files for trend analysis
- [ ] Open in Excel / Power Query; review column names and data types
- [ ] Filter to `VISA_CLASS = H-1B` if combined file
- [ ] Standardize `EMPLOYER_NAME` column (trim, uppercase, remove punctuation)
- [ ] Standardize `CASE_STATUS` to: Certified / Denied / Withdrawn
- [ ] Confirm `WAGE_RATE_OF_PAY_FROM` is annual (convert if hourly)
- [ ] Drop unnecessary columns; export cleaned file to `/data/cleaned/oflc_h1b_cleaned.csv`

**USCIS H-1B Employer Data Hub**
- [ ] Download employer hub CSV from [USCIS](https://www.uscis.gov/tools/reports-and-studies/h-1b-employer-data-hub)
- [ ] Review columns: initial approvals, initial denials, continuing approvals, continuing denials
- [ ] Standardize `EMPLOYER_NAME` column to match OFLC naming convention
- [ ] Export cleaned file to `/data/cleaned/h1b_hub_cleaned.csv`

**Processing Times (Manual Seed)**
- [ ] Visit [USCIS Processing Times](https://egov.uscis.gov/processing-times/)
- [ ] Manually record current processing times for key forms: I-485, I-130, I-140, N-400, I-765, I-131
- [ ] Record for at least 3–4 service centers / field offices each
- [ ] Enter into a staging Excel file with columns: `snapshot_date`, `form_type`, `field_office`, `processing_time_low`, `processing_time_high`, `form_category`
- [ ] Repeat manual capture weekly until Flow 1 is live (target: 4–6 manual snapshots before automation)

---

### 🗄️ Phase 2 — SharePoint List Setup (Week 1–2)

- [ ] Create SharePoint List: `OFLC_FILINGS` — add all columns per data model above
- [ ] Create SharePoint List: `H1B_EMPLOYER_HUB` — add all columns per data model above
- [ ] Create SharePoint List: `PROCESSING_TIMES_SNAPSHOT` — add all columns per data model above
- [ ] Create SharePoint List: `PROCESSING_TIME_ALERTS` — add all columns per data model above
- [ ] Create SharePoint List: `USER_ALERT_SUBSCRIPTIONS` — columns: `user_email`, `form_type`, `field_office`, `active` (Yes/No)
- [ ] Create SharePoint List: `FLOW_CONFIG` — columns: `config_key`, `config_value` (stores last ingestion date, etc.)
- [ ] Import cleaned OFLC CSV into `OFLC_FILINGS` list (Power Automate or manual import)
- [ ] Import H-1B Hub CSV into `H1B_EMPLOYER_HUB` list
- [ ] Import manual processing time seed data into `PROCESSING_TIMES_SNAPSHOT`
- [ ] Validate row counts and spot-check 10 records per list

---

### 📊 Phase 3 — Power BI Core Build (Weeks 2–3)

**Setup**
- [ ] Open Power BI Desktop → Get Data → SharePoint Online List
- [ ] Connect to all 4 SharePoint lists
- [ ] Build Date Table in DAX:
  ```
  DateTable = CALENDAR(DATE(2019,1,1), TODAY())
  ```
- [ ] Add calculated columns to Date Table: Year, Quarter, Month, FiscalYear
- [ ] Build relationships: Date Table → `OFLC_FILINGS.decision_date`, `PROCESSING_TIMES_SNAPSHOT.snapshot_date`
- [ ] Build relationship: `OFLC_FILINGS.employer_name` ↔ `H1B_EMPLOYER_HUB.employer_name`

**DAX Measures (write before building visuals)**
- [ ] `Total Filings = COUNTROWS(OFLC_FILINGS)`
- [ ] `Initial Approvals = SUM(H1B_EMPLOYER_HUB[initial_approvals])`
- [ ] `Initial Denials = SUM(H1B_EMPLOYER_HUB[initial_denials])`
- [ ] `Denial Rate = DIVIDE([Initial Denials], [Initial Approvals] + [Initial Denials])`
- [ ] `Avg Wage Offered = AVERAGE(OFLC_FILINGS[wage_rate_offered])`
- [ ] `Wage vs Prevailing Ratio = DIVIDE([Avg Wage Offered], AVERAGE(OFLC_FILINGS[prevailing_wage]))`
- [ ] `YOY Filing Change = [Total Filings] - CALCULATE([Total Filings], SAMEPERIODLASTYEAR(DateTable[Date]))`
- [ ] `Current Processing Low = CALCULATE(MAX(PROCESSING_TIMES_SNAPSHOT[processing_time_low]), LASTDATE(PROCESSING_TIMES_SNAPSHOT[snapshot_date]))`
- [ ] `Processing Time Delta = [Current Processing Low] - CALCULATE([Current Processing Low], DATEADD(DateTable[Date], -4, WEEK))`

**Pages**
- [ ] Build Page 1: Executive Overview (KPIs, top employers bar, processing line, slicers)
- [ ] Build Page 2: Employer Intelligence Deep Dive (matrix, scatter, map)
- [ ] Build Page 3: Employer Profile drill-through (YOY line, SOC bar, wage histogram, wage level donut)
- [ ] Configure drill-through: right-click on Page 2 employer → navigates to Page 3
- [ ] Build Page 4: Processing Time Dashboard (line chart, heatmap table, ranked bar, conditional formatting)
- [ ] Build Page 5: Alert History (alerts table, frequency bar, KPI cards)
- [ ] Build Page 6: Macro View (cap utilization line, country breakdown, naturalization trend)
- [ ] Apply consistent color theme across all pages
- [ ] Add bookmarks for slicer panel show/hide on each page
- [ ] Publish report to Power BI Service workspace

---

### 📱 Phase 4 — Power Apps Build (Weeks 4–5)

**App 1: Employer Lookup Tool**
- [ ] Create new Canvas App (tablet layout)
- [ ] Screen 1: Add search text input + state/industry/year dropdowns
- [ ] Screen 1: Connect gallery to `H1B_EMPLOYER_HUB` SharePoint list with delegation
- [ ] Screen 1: Add "View Profile" button → navigate to Screen 2
- [ ] Screen 2: Bind KPI labels to selected gallery record (approval rate, avg wage, top title)
- [ ] Screen 2: Add Power BI tile (embedded report visual for selected employer)
- [ ] Screen 2: Add wage compliance indicator using conditional color formula:
  ```
  If(WageRatio >= 1, Green, If(WageRatio >= 0.9, Yellow, Red))
  ```
- [ ] Screen 2: Add "Export PDF" button → trigger Flow 4 via Power Automate connector
- [ ] Screen 3: Add employer name inputs (3 total) for side-by-side comparison
- [ ] Screen 3: Build comparison table using Gallery with hardcoded column headers
- [ ] Screen 3: Add "Send Report" button → trigger email flow

**App 2: Case Processing Estimator**
- [ ] Create new Canvas App (phone layout)
- [ ] Screen 1: Add form type dropdown (distinct values from `PROCESSING_TIMES_SNAPSHOT`)
- [ ] Screen 1: Add field office dropdown (filtered by form type)
- [ ] Screen 1: Add date picker for receipt date
- [ ] Screen 1: Add "Estimate" button → navigate to Screen 2
- [ ] Screen 2: Calculate and display current processing window (lookup from SharePoint)
- [ ] Screen 2: Display estimated completion date range
- [ ] Screen 2: Add status flag label (On Track / Extended / Significantly Delayed)
- [ ] Screen 2: Add "Set Alert" button → write record to `USER_ALERT_SUBSCRIPTIONS` list
- [ ] Screen 3: Gallery of current user's alert subscriptions
- [ ] Screen 3: Toggle control to activate/deactivate each alert
- [ ] Screen 3: Display current processing time vs. time when alert was set

---

### ⚙️ Phase 5 — Power Automate Flows (Weeks 6–7)

**Flow 1: Weekly Processing Time Archiver**
- [ ] Create scheduled cloud flow (recurrence: weekly, Sunday 8AM)
- [ ] Add HTTP action: GET USCIS processing times page
- [ ] Add Parse JSON / Parse HTML action to extract form/office/time values
- [ ] Add "Get items" action from `PROCESSING_TIMES_SNAPSHOT` (filter: this week)
- [ ] Add condition: if no record exists → create new item
- [ ] Add condition: if delta > 2 months → create item in `PROCESSING_TIME_ALERTS`
- [ ] Add Send Email action with snapshot summary
- [ ] Add HTTP POST to Power BI REST API to trigger dataset refresh
- [ ] Test with manual trigger; verify new row appears in SharePoint

**Flow 2: Quarterly OFLC Data Ingestor**
- [ ] Create scheduled cloud flow (recurrence: monthly, 1st at 6AM)
- [ ] Add "Get items" from `FLOW_CONFIG` to retrieve last ingestion date
- [ ] Add HTTP GET to DOL OFLC data page
- [ ] Add condition: if new file detected vs. last ingestion date
- [ ] Add Parse CSV action for new file content
- [ ] Add "Create item" loop for each new record (batch where possible)
- [ ] Update `FLOW_CONFIG` with new ingestion date
- [ ] Add Send Email confirmation
- [ ] Add Power BI refresh trigger

**Flow 3: User Alert Notifier**
- [ ] Create scheduled cloud flow (recurrence: weekly, Monday 7AM)
- [ ] Get all items from `PROCESSING_TIME_ALERTS` (last 7 days)
- [ ] Get all items from `USER_ALERT_SUBSCRIPTIONS` (active = Yes)
- [ ] For each subscription: filter alerts by form_type + field_office match
- [ ] If match found: Send personalized email with delta summary
- [ ] Test with a seeded subscription record and a matching alert record

**Flow 4: Employer Profile PDF Generator**
- [ ] Create instant cloud flow with HTTP request trigger
- [ ] Add "Get items" from `OFLC_FILINGS` filtered by employer name
- [ ] Add "Get items" from `H1B_EMPLOYER_HUB` filtered by employer name
- [ ] Compose HTML body with employer summary table
- [ ] Add Send Email action with HTML body (or OneDrive PDF conversion if available)
- [ ] Test by calling from Power Apps button

---

### 🔗 Phase 6 — Integration & Polish (Week 7–8)

- [ ] Embed Power BI report inside Power Apps App 1 (Employer Lookup → Employer Profile screen)
- [ ] Test drill-through from Power BI to Power Apps (deep link via URL navigation)
- [ ] Validate all SharePoint data connections refresh correctly in Power BI Service
- [ ] Enable scheduled refresh in Power BI Service (daily)
- [ ] Add tooltip descriptions to all Power BI visuals explaining immigration context (e.g., explain wage levels, RFE implications)
- [ ] Add loading spinners to Power Apps screens during data fetch
- [ ] Add error handling to all Power Automate flows (send failure email if any step errors)
- [ ] Test all 4 flows end-to-end with real data

---

### 🚀 Phase 7 — Final Deliverables (Week 8)

- [ ] Build Power BI Page 6 macro view (cap utilization, country breakdown, naturalization trend)
- [ ] Final UI polish on all Power Apps screens (consistent fonts, colors, icons)
- [ ] Record a 5–7 minute walkthrough demo video covering both modules
- [ ] Write a one-page Solution Brief (problem → approach → architecture → business value)
- [ ] Publish Power BI report link + Power Apps share link for portfolio
- [ ] Push project folder to GitHub (`dustindoesdata/uscis-insights-hub`)
- [ ] Write LinkedIn post announcing the project

---

## Portfolio Pitch

> *"USCIS Insights Hub is a zero-cost employer intelligence and case monitoring platform built entirely on public government data and Microsoft 365. It combines DOL and USCIS H-1B employer records with automated weekly snapshots of USCIS processing times — data that has no public historical archive — to give HR teams, immigration compliance officers, and legal staff a real-time and longitudinal view of the U.S. employment immigration landscape. Power Automate ingests and archives the data, Power BI surfaces the analytics, and Power Apps puts actionable lookup tools directly in the hands of non-technical users."*

---

## Resources & Links

| Resource | URL |
|---|---|
| DOL OFLC H-1B Performance Data | https://www.dol.gov/agencies/eta/foreign-labor/performance |
| USCIS H-1B Employer Data Hub | https://www.uscis.gov/tools/reports-and-studies/h-1b-employer-data-hub |
| USCIS Processing Times | https://egov.uscis.gov/processing-times/ |
| USCIS Immigration Forms Data | https://www.uscis.gov/tools/reports-and-studies/immigration-forms-data |
| DHS Yearbook of Immigration Statistics | https://www.dhs.gov/immigration-statistics/yearbook |
| Microsoft 365 Developer Program | https://developer.microsoft.com/en-us/microsoft-365/dev-program |
| Power BI REST API (Dataset Refresh) | https://learn.microsoft.com/en-us/rest/api/power-bi/datasets/refresh-dataset |
| USCIS H-1B Cap Info | https://www.uscis.gov/working-in-the-united-states/h-1b-specialty-occupations |
| DOL Prevailing Wage Info | https://www.dol.gov/agencies/eta/foreign-labor/wages |

---

*Last updated: May 2026 | Author: Dustin | GitHub: dustindoesdata*
