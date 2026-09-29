---
id: data-schema-brightfit-worked-example
title: "Data Schema (Tab 3) — BrightFit Worked Example (for Jaya sign-off)"
---

## Data Schema (Tab 3) — BrightFit Worked Example

**Date:** 2026-06-12
**Purpose:** Show exactly what the Data Schema tab (Tab 3) outputs for the BrightFit demo advertiser, so Data Engineering (Jaya) can review/sign off the columns, format, collection plan, and the funnel→column mapping before build.
**Authoritative sources:** Anupam's client doc *"CSV format and file arrangement requirements"* + *"AIM Data Schema — Event data example.xlsx"* (Template sheet); cross-checked against live DB `aim.advertiser_schema_mappings` (global) + `aim.validation_rules`.

---

### 1. BrightFit inputs that drive the schema (from the wizard)

| Field | Value | Schema effect |
|---|---|---|
| Platforms | iOS, Android (no Web) | `operating_system` ∈ {ios, android}; iOS ⇒ `skad_installs` relevant; no web rows |
| Primary market | US | `country_code = us` |
| Business model | Subscription | funnel = subscription dep_vars (below) |
| Funnel | Install · Registration · **Trial Start (KPI)** · Subscription Start | maps to install/registration/conversion columns (see §4) |
| LTV | Yes, cohort data available | **cohort columns required** (flattened, days 0–360) |
| Data history | 24–36 months (both platforms) | full seasonal coverage; no effect on columns |
| MMP | AppsFlyer · **Direct API** · cohort access = yes | install/event/cohort data auto-collected via API |
| Ad spend | **Direct API** · sources: MMP-direct + Supermetrics | spend + delivery metrics auto-collected via connector |
| Offline media | No | no offline rows/columns |
| Campaign types | UA + UE (no Brand) | `campaign_type` ∈ {UA, UE} |
| Incremental study | No (Phase 1 onboarding, not an incrementality study) | **exclude all `_incr_` / `_lift_` columns** |

---

### 2. The flow (and the open question)

The Data Schema tab generates BrightFit's **tailored CSV template** (§3). The flow (Gary B8):

> Download tailored template (Data Schema tab) → client fills it → **upload to the existing Validate Onboarding tab** to check the format → ETL ingests.

**⚠ OPEN — question #1 for the meeting (do NOT assume): what is auto-collected vs client-provided?** I previously assumed "MMP = Direct API ⇒ event data auto-collected ⇒ no CSV." That is unverified and probably wrong: Anupam ships this template *to create the data schema*, which implies the **event/conversion/cohort data is client-provided regardless of MMP method**. "Direct API / connector" may apply only to *ad spend* (Supermetrics) and/or attribution, not the event schema. **Confirm with Jaya/Anupam** which of these is API-pulled vs file-provided:

| Data | Provided how? |
|---|---|
| Install & in-app event data (installs, registrations, conversions) | **? client CSV vs AppsFlyer API** |
| LTV cohort data (revenue/registrations/retentions × day) | **? client CSV vs AppsFlyer cohort API** |
| iOS SKAN / ATT | **? Kochava API** |
| Ad spend + delivery (clicks/impressions/total_cost) | likely Supermetrics connector (confirm) |

Per spec §4.2 the tab renders the **green "no uploads needed"** state ONLY if every source is truly direct; otherwise it shows the template download per the answer above. Until confirmed, treat BrightFit as **template-download** (the column proposal stands either way; only the delivery method changes).

---

### 3. The deliverable — a tailored CSV template (download)

**Tab 3's primary action = "Download CSV template"** — the customer-facing file (Anupam's "Sheet 1"), but **tailored to BrightFit's answers** (only the columns AIM needs, not the full master schema). The client fills it and provides it. Flattened-cohort CSV, one row per `full_date × app_id × operating_system × country_code × campaign` grain. `_incr_`/`_lift_` excluded (incremental-study only).

**Exact header — BrightFit template (37 columns):**

```csv
full_date,app_id,operating_system,country_code,network_name,campaign_id,campaign_name,campaign_type,publisher_name,channel,daus,installs,skad_installs,clicks,impressions,total_cost,revenue,registrations,retentions,revenue_0d,revenue_1d,revenue_7d,revenue_30d,revenue_90d,revenue_360d,registrations_0d,registrations_1d,registrations_7d,registrations_30d,registrations_90d,registrations_360d,retentions_0d,retentions_1d,retentions_7d,retentions_30d,retentions_90d,retentions_360d
```

Grouped (same columns, for review):

**Dimensions (10):**

```text
full_date, app_id, operating_system, country_code, network_name,
campaign_id, campaign_name, campaign_type, publisher_name, channel
```

**Non-cohort metrics — BrightFit-relevant subset:**

```text
daus, installs, skad_installs, clicks, impressions, total_cost,
revenue, registrations, retentions
```

- `skad_installs` — included (iOS selected).
- `clicks, impressions, total_cost` — from ad-spend connector.
- **Excluded for BrightFit:** `views`, `ad_revenue` (ad-monetization — BrightFit monetizes via subscription, not ads), `new_orders`/`total_orders` only if Trial/Subscription map to them (see §4 — open).

**Cohort metrics (flattened, days 0/1/7/30/90/360) — LTV = yes:**

```text
revenue_0d  revenue_1d  revenue_7d  revenue_30d  revenue_90d  revenue_360d
registrations_0d … registrations_360d
retentions_0d … retentions_360d
```

- Revenue cohort is the core LTV signal for a subscription app. Registrations + retentions cohorts support trial→paid conversion and churn modelling.
- **Excluded cohort families for BrightFit:** `views_Xd`, `ad_revenue_Xd` (no ad monetization); `new_orders_Xd`/`total_orders_Xd` pending the §4 mapping decision.

---

### 4. Funnel-event → system-column mapping (⚠ Jaya sign-off needed)

BrightFit's dep_vars don't all have an obvious 1:1 system column. Proposed mapping (DB `advertiser_schema_mappings` informs the unambiguous ones):

| BrightFit funnel event | Proposed system column | Confidence | Note |
|---|---|---|---|
| Install | `installs` | ✅ high | `{{installs}} → installs` |
| Registration | `registrations` | ✅ high | `{{registration}}/{{signups}} → registrations` |
| **Trial Start (KPI)** | `new_orders` | ⚠ **confirm** | "first conversion" slot; no dedicated `trial` column exists |
| **Subscription Start** | `total_orders` (or `registrations`) | ⚠ **confirm** | `{{subscriptions}} → registrations` exists, but that collides with Registration; `total_orders` may better represent paid conversions |
| (Revenue) | `revenue` + `revenue_Xd` | ✅ high | subscription revenue + LTV cohort |

**Open question for Jaya:** confirm the column each subscription event maps to — especially **Trial Start** (the KPI/dep_var) and **Subscription Start**. This determines whether `new_orders(_Xd)` / `total_orders(_Xd)` are in BrightFit's schema. Everything else is settled.

---

### 5. Format & file arrangement (reference — applies when a source is client-provided)

- **Flattened cohort format (required):** one row with a column per cohort day; NOT a long `cohort` column.
- **CSV**, UTF-8. Column names free (mapped on the mapping page); not all columns mandatory.
- **Folder layout (two supported):** (1) hierarchical `YYYY-MM/YYYY-MM-DD/1.csv,2.csv` (filename = order only); (2) flat `YYYY-MM/YYYY_MM_DD_n.csv` (filename must encode date+index). Dedicated AIM folder, only the data to share.
- Downloadable CSV template = the Anupam example dataset (Template sheet), filtered to BrightFit's columns.

---

### 6. Sign-off checklist for the Jaya meeting

1. **Collection model (the big one):** which data is **API-pulled vs client-provided**? Is the event/cohort CSV *always* client-provided (so BrightFit downloads + fills the template), or does AppsFlyer Direct API remove the need for it? (§2) This decides whether BrightFit sees the template at all.
2. **Validate Onboarding loop:** confirm the flow = download tailored template → fill → upload to the existing Validate Onboarding tab to check format (Gary B8).
3. **Column set** (§3) correct for subscription + LTV on iOS/Android? (`skad_installs`, `daus`, `retentions`, revenue/registrations/retentions cohorts in; views/ad_revenue out.)
4. **Funnel→column mapping** (§4) — confirm Trial Start + Subscription Start columns (decides new_orders/total_orders inclusion).
5. **Cohort days** 0/1/7/30/90/360 correct, or a subset? **`_incr_`/`_lift_` excluded** for non-incremental — confirm.
6. **Tailored vs full-master:** OK that the tool generates a *filtered* template (only needed columns) vs today's full-master sheet?

---

### 7. How this generalizes (for the build — F6 `buildProvisionPlan`)

The same logic produces any advertiser's Tab 3: **dimensions always** + **non-cohort metrics implied by selected funnel events + ad-spend** + **cohort families when LTV=yes** + **`skad_installs` if iOS** + **web columns if Web** + `_incr_/_lift_` only for incremental studies. Each non-direct source ⇒ a "you provide" card with this schema + CSV template + the §5 arrangement. BrightFit is the all-direct, subscription+LTV instance of that.
