
id: product-spec
title: Product Specification

Product Specification: AIM Onboarding Decision Tool

AISDLC Phase: 2 — Product Spec
PRD: 061-aim-onboarding-decision-tool
PM: Gary Danks
Date: 18 May 2026
Version: v3 — Updated post engineering sync
Status: Ready to merge  

## 1. Executive Summary

The AIM Onboarding Decision Tool is an 8-step guided wizard embedded in the K4A console that replaces Kochava's current open-ended onboarding questionnaire (a static spreadsheet) with a constrained, progressive-disclosure flow. It collects everything the data science team needs to configure an AIM model — platforms, business model, KPI, funnel events, data sources, and objectives — and auto-generates four output documents: a draft Scope of Work, a delivery Timeline, a filtered Data Schema, and an internal Model Config. The tool is the prerequisite for AIM X zero-touch onboarding and the primary mechanism for eliminating scope creep from the AIM delivery process.

## 2. Problem & Goals

Problem Statement

"At the moment we give them a spreadsheet. It's got all questions. It's ignorant of their business model. There is no decision tree there." — Robert Pawlowicz, Lead Data Scientist

The current AIM onboarding process is open-ended: clients are asked what they want, they ask for everything, and the team says yes. This creates scope creep, delays model builds, and over-engineers ETL pipelines — the #1 internal pain point identified across every persona interviewed (CSM, Data Engineering, Data Science) in Gary's 8-interview discovery exercise (April 2026).

The Klook incident is the defining case study: Kochava promised a feature not yet built, Robert had to build a new framework from scratch, the client was lost, and the TikTok partnership was damaged.

Goals

Eliminate scope creep at source — constrained wizard produces a deterministic, agreed Scope of Work before any engineering begins

Enable AIM X zero-touch onboarding — wizard self-serve path requires no CSM or engineering effort per client

Reduce model configuration time — auto-generated app_config from wizard answers; Robert's current manual interpretation time drops from hours/days to < 10 minutes

Give clients exact data requirements — filtered Data Schema shows only what they need, not the full master schema

Reduce CSM discovery burden — structured wizard replaces open-ended call

Success Metrics

Metric

Baseline

Target

Measurement

Onboarding scope changes post-SoW

Multiple per quarter per CSM

< 1 per quarter per CSM

SoW amendment log

Model config time (Robert)

Hours/days per client

< 10 minutes

Time tracked from wizard completion to app_config finalised

AIM X onboarding touches (CSM + Engineering)

High — manual schema work

Near-zero

CSM effort log per client

Client data schema delivery time

Days (client over-provides, team filters)

Same day as wizard completion

ETL pipeline start date

Onboarding call prep time (CSM)

High

Eliminated for AIM X

CSM survey

## 3. User Stories

CSM — Kade / Jacob

As a CSM, I want to run a structured onboarding questionnaire with a new AIM client so that I receive a pre-populated Model Config, Data Schema, and draft Scope of Work without manual interpretation work.

Acceptance criteria:

CSM can launch the tool in CSM mode and complete all 8 steps on behalf of a client

Model Config tab (Tab 4) is visible and populated in CSM mode

CSM mode surfaces internal guidance notes per answer (Redshift partition notes, platform model justification, etc.)

CSM can approve the Scope of Work on behalf of the client, anchoring the Timeline

CSM mode is triggered via a defined mechanism (URL param, login, or separate URL — see Open Questions)

Client — Data Team / Marketing Manager

As a new AIM client, I want to complete an onboarding questionnaire in under 10 minutes so that I know exactly what tier I need, what data to provide, and what my delivery timeline looks like — without needing a call first.

Acceptance criteria:

Client can complete all 8 steps unassisted in client mode

Output screen shows a clear tier recommendation (AIM X or AIM Pro) with rationale

Data Schema tab shows only the data they need to provide based on their answers

Client can approve the Scope of Work in-tool

Client can save progress and return to a partially completed wizard

Model Config tab (internal CSM document) is NOT visible in client mode

Data Scientist — Robert

As a Lead Data Scientist, I want to receive a complete, validated model configuration from the onboarding wizard so that I can begin the AIM model build without a discovery call or manual interpretation.

Acceptance criteria:

Model Config tab outputs all fields required for app_config MongoDB entry

Funnel events are expressed as dep_var slugs

UA/UE routing flag and brand routing flag are computed and shown

SKAN configuration and ATT rate estimate included for iOS

Data source delivery method per source is specified

Data Engineer — Jaya / Anupam

As a Data Engineer, I want to receive a filtered, client-specific data schema from the onboarding wizard so that I can begin ETL configuration without waiting for a discovery call or schema negotiation.

Acceptance criteria:

Data Schema tab shows only schema rows relevant to the client's answers

Each client-supplied item includes a full schema table (field name, format, required/optional)

CSV template download available per schema type

"Other" data source selections generate a TBC milestone in the Timeline tab

## 4. Feature Requirements

4.1 Core Features (P1 — Must Ship)

Wizard Shell & Navigation

Requirement: 8-step linear wizard with stepper/breadcrumb navigation (K4A portal stepper component — not a custom left sidebar)

Behaviour: Every step is accessible from the stepper at any time. Stepper shows ✓ for completed steps and a red dot for validation errors (evaluated at output). "Next" on Step 8 triggers the 1.6s loader (Phase 9) then the output screen (Phase 10).

Validation: Non-blocking. Users can proceed with incomplete answers. Hard blocks apply only on "Next" when critical fields are empty. Full validation runs only when the output screen is reached — surfaces as a banner with jump-back links to offending steps.

State persistence: All state auto-saved to localStorage on every change. Restored on page load. Welcome screen offers "continue saved progress."

Phase routing: Phases 1–8 map directly to wizard steps 1–8. Phase 9 = loader. Phase 10 = output screen. No phases are skipped.

Step 1 — Your Team

Requirement: Collect company identity and key contacts

Fields:

Company name — text input, required

Project lead(s) — repeating name + email rows. At least one required. "+ Add another lead" / × remove. "Same as project lead" checkbox syncs data leads in real time.

Data lead(s) — same repeating row pattern. At least one required (unless synced from project lead)

Validation: Error if company name empty. Error if no project lead row has both name and email. Error if no data lead row (and not synced).

Feeds output: Company name and lead names populate the Scope of Work header and signoff block verbatim.

Step 2 — About Your Product

Requirement: Establish product name, platforms, conversion split, model approach, and primary market

Fields:

App/brand name — required

Platform selection — multi-select toggle cards: iOS, Android, Web. At least one required. iOS selection triggers ATT/SKAN callout note.

Conversion share allocator — shown when ≥2 platforms selected. Sliders + steppers per platform; must total 100%. 2 platforms: moving one auto-adjusts the other. 3 platforms: proportional redistribution.

Spend separable? — shown only when Web + ≥1 mobile platform. Yes / No / Not sure. "Not sure" reveals optional free-text textarea.

Cross-journey migration? — shown after spend separability answered. Yes / No / Not sure with "See an example" disclosure. Drives unified vs separate model recommendation.

Recommended model structure — shown after both questions above answered. Choice cards: Unified / Separate. Auto-recommended badge on system preference. User can override.

Primary market — select: US, UK, Germany, France, Canada, Other. "Other" reveals free-text. Required. Locked until ≥1 platform selected.

Feeds output: Platform list, share%, modelling approach, and region populate Scope of Work, Model Config, and Setup Summary panel model count estimate.

Step 3 — Marketing Setup

Requirement: Collect media budget, channel mix, attribution reliability, and campaign type split

Fields:

Total paid media budget — slider with Monthly/Annual toggle. Monthly: $50K→$10M (hardcoded range — $50K is the AIM minimum; do not lower). Annual: $5M→$200M+. Required. Stored as budgetMonthly/budgetAnnual + budgetPeriod. The slider range is not configurable — it serves to estimate advertiser volume, and $50K is the hard minimum for AIM eligibility.

Offline media — Yes/No. Required. If Yes: digital/offline spend split slider (default 80/20); offline schema included in Data Schema.

Digital media types — multi-select chips: Self-attributed (Meta, Google, TikTok, ASA), Standard attributed (DSP/programmatic), Affiliates/partner networks, Other. At least one required.

Attribution data reliability — Yes/No/Not sure. Required. If Yes or Not sure: chip multi-select of gap categories (Offline media, DSP, Affiliates, Branded/awareness, Mobile networks, Influencer, Podcasts, Not sure, Other). "Not sure" shows a reassurance callout.

Coverage confidence — "Are you confident your MMP captures the majority of installs?" Yes/No/Not sure. Required.

Organic/paid split per platform — slider per platform showing % of installs that are paid. Defaults: iOS 65%, Android 80%, Web 50%.

Campaign types — toggle cards: User Acquisition (UA), User Engagement (UE), Brand. At least one required. Toggling a type initialises its share at an equal split.

Budget split across campaign types — shown when ≥1 campaign type selected. Slider + stepper per active type. 2 types: constrained to 100%. 3 types: proportional redistribution. Warning banners if a type is <5% or >90%.

Feeds output: Campaign shares → uaUeRoutingFlag and brandRoutingFlag (computed live on every state update). These feed the tier algorithm and Model Config. Offline spend → offline media schema row in Data Schema.

Step 4 — Business Model & Conversion Funnel

Requirement: Determine the KPI event and funnel structure

Fields:

Business model — choice cards: Subscription / E-commerce / Gaming. Required. Changing this resets the funnel.

Gaming revenue type — shown only for Gaming: IAP / Ads / Both. Required if Gaming selected.

App funnel events — checklist per business type. Required events locked on; optional events togglable. Minimum 3 on. Each event has an editable name field and "Set as Primary KPI" button. One KPI must be set.

Funnel defaults: Subscription → Install (required) + Registration + Trial Start + Subscription Start (KPI: Trial Start) + Revenue. E-commerce → Install (required) + Registration + First Purchase (required, KPI: First Purchase) + Total Purchases + Revenue. Gaming follows e-commerce funnel.

Web funnel — shown if Web platform selected. Same pattern for web events. Minimum 2 on.

LTV model — shown after funnel configured. "Include LTV model?" Yes/No. If Yes: "Cohort revenue data available?" Yes/No/Partial. If Partial: choice of partial or skip.

Edge cases: If business model changes mid-flow, funnel resets. The funnel object carries a business field to detect this. Funnel connector lines shown between events.

Feeds output: Funnel events → dep_var slugs in Model Config. KPI event → Scope of Work. LTV cohort selection → conditional cohort rows in Data Schema (7-day, 30-day, 90-day, 360-day revenue).

Step 5 — Data Sources

Requirement: Collect MMP, web attribution, and ad spend data source configuration

Fields:

Historical data per platform — tile selection per selected platform: 12–24 months / 24–36 months / 36 months+. 12–24 months shows a warning banner (seasonality partially modelled); 24+ months shows green confirmation.

MMP selection — shown if mobile platform selected. Chip: AppsFlyer, Adjust, Singular, Branch, Kochava, Other/none. Required. "Other/none" → custom connector TBC milestone injected into Timeline.

MMP collection method — Direct API (Recommended badge) / File / cloud. File → storage type chips (Google Drive, S3). Shown after MMP selected (except "Other/none").

AppsFlyer cohort access — Yes/No sub-question with explanatory note. Shown only if AppsFlyer selected.

Web attribution sources — shown if Web selected. Multi-select chips: Google Analytics 4 (directly connectable), Google Drive, S3, Database, Other. "Other" checkbox → free-text. Required if Web.

Ad spend data — collection method first (Direct API / File / Manual). Then source chips: MMP direct, Supermetrics, Google Drive, S3, Database, Other. "Other" → text input + Timeline TBC milestone. Required.

Feeds output: MMP + collection method → provision plan in Data Schema (direct vs client-supplied). "Other" selections → custom connector TBC milestones in Timeline tab. History → seasonality note in Scope of Work.

Step 6 — External Factors (Optional)

Requirement: Allow client to flag factors AIM cannot detect from data alone

No hard validation on this step.

Fields:

Auto-included factors — two read-only cards: Seasonality (✓ Included, or ⚠ Partial if history is 12–24 months) and National holidays (✓ Included, shows client's region).

Additional external factors — multi-select grid: Promotional activity, Competitor activity, Major product launch/rebrand, PR/earned media spike, Macro economic event, Other. Note: "Regulatory change" and "Natural disaster/crisis" have been intentionally consolidated into "Macro economic event" to reduce cognitive load. 6 factors total — prototype is correct, original spec listing 8 was superseded. Warning callout: client must be able to supply data for flagged factors.

Free-text description — shown when any factor selected. Textarea for brief description (what happened, which market, rough significance). No dates or data required.

Feeds output: Selected factors and notes appear in Scope of Work and Model Config.

Step 7 — Objectives (Optional)

Requirement: Capture client goals to frame the kickoff call. All fields optional. No validation.

Fields:

What are you hoping to achieve with your AIM model? (objectivesGoal) — textarea

What does success look like? (objectivesSuccess) — textarea

What are your marketing goals for the next 12 months? (objectivesMarketing) — textarea

Hint shown at bottom: "Optional — your answers are shared with your onboarding manager to frame the kickoff call."

Feeds output: All three fields must appear in the Step 8 review summary AND in a dedicated Objectives section in Tab 1 (Scope of Work). This is a Phase 1 requirement — Anupam to implement during Vue port.

Step 8 — Review Your Answers

Requirement: Full read-back of all wizard answers before generating output

Behaviour: No new input collected. Seven sections, each with an Edit button jumping directly to that step: Your team · Product · Marketing · Business & funnel · Data sources · External factors · Objectives. Each section renders key/value rows from state. Unanswered fields show a — placeholder at 45% opacity.

Setup Summary panel is hidden at this step — the full review layout replaces it.

CTA: "Generate my AIM setup" — triggers setPhase(9) → setTimeout(() => setPhase(10), 1600).

No validation runs here. Validation surfaces as a banner on the output screen with jump-back links.

Output Screen — 4 Tabs

The output screen renders at Phase 10. Header shows app name and an "Edit answers" button. Tier recommendation summary computed by recommendTier(state) (AIM X or AIM Pro) shown above the tab strip. Default tab on load is Tab 1 (Scope of Work).

Tab 1 — Scope of Work (All Users · Default)

Generated document in monospaced doc-card style, pre-populated from all wizard answers

Sections: engagement details, platform & region, campaign types, business model & KPI, funnel events, data sources, external factors, model structure

Each section has an editable inline notes textarea that expands on click

Approval block: name field + job title field (both required to activate Approve button). Captures approverName, approverJobTitle, and full approvedAt timestamp. Deliberate design: requiring an official job title formalises the signoff as a named, titled individual — not just a free-text entry.

"Download Scope of Work (PDF)" button is included in Phase 1 — real implementation required (not stubbed). Agreed in engineering sync 2 June 2026.

onApproved callback passes approvedAt to the Timeline tab

Post-approval behaviour: Once approved, the wizard questionnaire form is removed from view. Only the output tabs (SoW, Timeline, Data Schema) remain visible. Further edits require a CSM-led re-approval cycle.

Unlock to edit: Core generated content (scope, tier, data schema) is locked after approval. Inline notes remain editable. An explicit "Unlock to edit" action resets approvedAt and re-surfaces the approval block.

Tab 2 — Timeline (All Users)

Delivery plan showing estimated milestones from onboarding kickoff to first insights

AIM X: 7 milestones (~7 weeks minimum). AIM Pro: 8 milestones including a Results Review (~9+ weeks)

Date anchoring: cascade from approvedAt if set, otherwise from today. All start dates are the next Monday on or after their calculated date

Cascade delay logic: marking a milestone "Delayed" + saving a new date shifts all subsequent milestone dates by the corresponding weeks. A delay banner shows total accumulated delay

Per-milestone interactions: status selector (In progress / Complete / Delayed), delay date picker, notes section (add/edit/delete), expand/collapse toggle

All status changes and note actions written to an audit Change Log at the bottom of the tab

Custom connector rows injected for "Other/none" MMP, web attribution, or ad spend selections

⚠️ Author name field must be replaced with authenticated session user's email in production

Tab 3 — Data Schema (All Users)

Split into: "Kochava collects automatically" and "You need to provide"

Provision plan built dynamically by buildProvisionPlan(state) from wizard answers

Conditional schema rows:

Offline media schema included only if usesOffline === true

LTV cohort rows (7-day, 30-day, 90-day, 360-day revenue) only if wantsLtv === true and cohort data available

Web schema only if Web platform selected

iOS SKAN fields only if iOS selected

MMP-specific column names based on MMP selection (AppsFlyer vs Adjust etc.)

Each client-supplied item shows a full schema table (field name, format, required/optional, notes) and a stubbed CSV template download

If all sources are direct: green confirmation banner shown instead of schema tables

⚠️ Full shape of this tab is TBD — see §11 Open Questions (Option A vs Option B decision needed with Satish, Anupam, Jaya)

Tab 4 — Model Config (CSM Only)

⛔ OUT OF SCOPE FOR PHASE 1. Model Config tab is excluded from Phase 1 development. Agreed in engineering sync 2 June 2026.

Phase 2: CSM-only tab visible when mode === "csm". Will contain: fitting method, tier, platform(s), region, modelling approach (unified/separate); campaign types in scope, uaUeRoutingFlag, brandRoutingFlag, campaign merge recommendation; media structure and channel list; funnel events as dep_var slugs with primary KPI flagged; SKAN configuration (iOS only) and ATT rate estimate; data source configuration and delivery method per source; LTV scope and cohort availability.

Pending: Gary to obtain app_config schema from Robert to finalise field mapping for Phase 2 spec.

Tier Recommendation Algorithm

Defined in recommendTier(state) in output.jsx. Starts at AIM X. Upgrades to AIM Pro if any of three conditions are met:

Spend + history: monthly spend ≥ $250K AND history is 12–24 or 24+ months

Data sources: ≥ 2 additional non-MMP data sources connected

UA/UE routing flag: uaUeRoutingFlag === "aim_pro_eligible" — set when both UA and UE are present at mid share (10–90%)

AIM Pro can always be manually downgraded to AIM X. AIM X can only be upgraded if the auto-recommendation is Pro.

Setup Summary Panel

Hovering panel on right edge of wizard, visible steps 2–6 only

Reflects every answer in real time including: platform list, model count estimate, business model, KPI event, MMP, data sources

Hidden at Step 8 — replaced by the full review layout

Model count estimate updates live as platform and region answers change

User Modes

Mode

Who

Access

client

New AIM client completing the wizard themselves

Steps 1–8, Tabs 1–3

csm

Kochava CSM completing on behalf of a client

Steps 1–8, Tabs 1–3 (Tab 4 is Phase 2). Additional guidance notes visible per answer.

Primary persona: The tool is designed primarily for the advertiser to complete. The CSM reviews the generated Scope of Work after the client submits — they do not lead the wizard completion by default.

CSM mode trigger: Super Admin role. CSM access is managed via the existing K4A Super Admin role — no new toggle or separate URL required. Anyone logged in with Super Admin permissions (e.g. Kade, Jacob) automatically accesses CSM mode. This resolves the open question from v2.

4.2 Extended Features (P2 — Should Ship)

PDF export for Scope of Work — Moved to Phase 1. See §4.1 Tab 1 output screen. PDF download is a real Phase 1 implementation.

CSV template download per schema type — real implementation of the per-schema "Download template" button. Generates a CSV/Excel file pre-configured with exactly the fields the client needs based on their answers.

Step 7 Objectives → Scope of Work — Moved to Phase 1. See §4.1 Step 7. Objectives must feed Tab 1 (SoW). Assigned to Anupam.

Multi-region collection — Phase 1 includes an "Add another region" action at the end of the onboarding flow, allowing clients to initiate a new independent onboarding session for a different region. Regions are processed sequentially, not in parallel (each region is its own onboarding instance). UI: a dropdown menu (not a button toggle) allows switching between regions and apps mid-session, even if those regions are at different stages of approval. Full model count multiplier logic remains Phase 2.

Submission/handoff flow — "Generate my AIM setup" does NOT trigger an automated notification. The loader animation is retained as a marketing/polish element only (visual feedback for the user). Notification to the onboarding team (Kade + Jacob) is triggered only when the client hits "Book my onboarding meeting" — this sends an email notification to the CSM team to initiate the offline meeting. The meeting itself is offline; a manual "CSM Approval" button becomes active in the tool only after the offline meeting has taken place. This button allows the process to move to the final SoW approval stage. Architecture decision confirmed in engineering sync 2 June 2026.

CSM mode trigger — Resolved. Super Admin role in K4A. See §4.1 User Modes.

mmm-insights.jsx exploration — the prototype contains an unused MMM Insights dashboard preview component. If intended as a "preview of what you'll get," slot it as a fifth output tab.

4.3 Out of Scope (Explicit)

Campaign-level attribution — AIM Pro only; not in AIM X scope

Offline media handling — separate ETL project; post-ingestion tagging only

In-form region assignment — onboarding notifications sent centrally; no regional routing in the form

Airbyte direct client access — AIM is the client-facing layer; Airbyte is internal infrastructure

Full data quality validation — schema/format validation is in scope; deep data quality is not

AIM Enterprise bespoke configuration — novel business models route explicitly to Enterprise path

Look-back periods and weekly grouping — system/engineering config, not customer-facing

Offline/cross-platform network identification — happens post-ingestion, not in questionnaire

## 5. User Flows

Flow 1: Client Self-Serve (AIM X Path)

Client receives a link to the AIM Onboarding Tool (client mode)

Lands on welcome screen. Option to start fresh or load saved progress.

Completes Steps 1–8 (can navigate forward/back via stepper at any time)

Setup Summary panel updates live throughout steps 2–6

Step 8: reviews all answers. Clicks "Generate my AIM setup."

1.6s loader (marketing animation — no backend call at this stage). Output screen appears.

Tier recommendation shown: AIM X (or AIM Pro with rationale).

Reviews Tab 1 (Scope of Work). Clicks "Book my onboarding meeting" — email notification sent to Kade/Jacob (CSM team).

Offline onboarding meeting takes place.

CSM activates "CSM Approval" button post-meeting. SoW approval block becomes available.

Client approves SoW (name + job title + timestamp). SoW stamps approvedAt. Questionnaire form hidden — only output tabs remain.

Reviews Tab 2 (Timeline). Sees milestone dates anchored from approval date.

Reviews Tab 3 (Data Schema). Downloads CSV template for required data.

Sends data to Kochava. ETL begins.

Flow 2: CSM-Assisted Onboarding

CSM logs in with Super Admin role — CSM mode activates automatically

CSM supports advertiser through wizard or reviews completed submission

CSM sees additional guidance notes per answer throughout

Output screen: Tabs 1–3 visible in Phase 1 (Tab 4 Model Config is Phase 2)

Offline onboarding meeting takes place

CSM activates "CSM Approval" button post-meeting

Client provides final SoW approval (name + job title + timestamp)

CSM manages Timeline milestone status updates (client has read-only view)

ETL team uses Data Schema tab to begin pipeline configuration

Flow 3: Demo / Sales

Open tool with BRIGHTFIT_DEMO constant enabled

All answers pre-filled with BrightFit fictional subscription app

Jumps directly to output screen

All 4 tabs populated and reviewable for sales demonstrations

## 6. UX / Design Reference

Source: Claude Design prototype AIM Onboarding.html (15 May 2026). High-fidelity, reviewed by 7 stakeholders. Design system updated to canonical AIM X tokens.

Key Design Decisions Already Made

#

Principle

Decision

01

Progressive disclosure

Sub-questions hidden until parent answered; locked sections at opacity 0.32, pointer-events: none

02

Recommendations over blank slates

Default funnel events pre-selected by business type; system preference auto-recommended with badge

03

Contextual reassurance

Attribution gaps → "AIM is designed to handle weak attribution." iOS → "no extra work needed." Short history → warned, not blocked.

04

Non-blocking validation

All validation deferred to output screen; hard blocks only on "Next" for empty required fields

05

Live read-back

Setup Summary panel on right edge (steps 2–6). Model count estimate updates live.

06

Mode parity

CSM and Client share the same UI structure. CSM additions are inline notes — never a separate screen.

07

No visual noise

No emoji, no decorative gradients, minimal 1.6px icon set

08

Immediately useful outputs

SoW is editable and approvable; Schema tab has copy-ready field names + CSV download

Design System

The production implementation must use the AIM X / Kochava MOS design system tokens from colors_and_type.css:

Primary: --brand-primary-1 (#1E4B97), --brand-primary-2 (#1B4388)

Secondary (focus rings, links): --brand-secondary (#0C72EE)

Page background: --surface-2 (#F8F7F7)

Cards: --shadow-card, --border (#E1E1E1), --radius-md (8px)

Buttons: --radius-sm (4px), 36px height, --text-btn-* (14px/600/0.175px ls)

Errors: --alert-red (#BE202E)

Success: --alert-green (#427900)

Warnings: --alert-orange (#BA4E00)

Info callouts: --alert-blue (#095ABD)

The prototype currently uses bridge aliases (--navy, --ink, --line, --card, etc.) pointing to canonical AIM X tokens. In production, remove the bridge layer entirely — reference AIM X token names directly.

## 7. Data Requirements

Inputs Collected by Wizard

Data point

State key

Type

Required?

Company name

companyName

string

Yes

Project leads

projectLeads

Array of {name, email}

Yes (min 1)

Data leads

dataLeads

Array of {name, email}

Yes (min 1, or sync from project leads)

App/brand name

appName

string

Yes

Platforms

platforms

Array of "iOS"|"Android"|"Web"

Yes (min 1)

Platform conversion shares

platformShares

{iOS: n, Android: n, Web: n} (must total 100)

Yes if ≥2 platforms

Modelling approach

modelling

"single"|"separate"|"unified"

Yes if Web + mobile

Primary market

region / regionOther

string

Yes

Total budget

budgetMonthly / budgetAnnual + budgetPeriod

number + "monthly"|"annual"

Yes

Offline media

usesOffline / offlineSplitPct

boolean + 0–100

Yes

Digital media types

digitalMediaTypes

Array of strings

Yes (min 1)

Attribution gaps

hasAttrGaps / attrGapCategories

"yes"|"no"|"not_sure" + string array

Yes

Coverage confidence

coverageConfidence

"yes"|"no"|"not_sure"

Yes

Organic/paid split

paidSplit

{iOS: n, Android: n, Web: n}

Auto-populated with defaults

Campaign types + shares

ua/uaShare, ue/ueShare, brand/brandShare

boolean + "lt10"|"mid"|"gt90"

Yes (min 1 type)

Business model

business

"subscription"|"ecommerce"|"gaming"

Yes

Funnel events + KPI

funnel

{business, items[], kpi, kpiConfirmed, names{}}

Yes

Web funnel

webFunnel

Same shape as funnel

Yes if Web selected

LTV scope

wantsLtv / ltvCohortAvail / ltvPartialChoice

boolean chain

Yes if funnel configured

MMP

mmp / mmpCollection / mmpFileStorage

string + delivery method

Yes if mobile platform

AppsFlyer cohort access

appsflyerCohortAccess

boolean

Yes if AppsFlyer selected

Web attribution sources

webAttrSources

Array of strings

Yes if Web selected

Ad spend collection

spendCollection / adSpendSources

delivery method + source array

Yes

Data history

history

"12_24"|"24_36"|"36_plus" per platform

Yes

External factors

externalFactors / externalFactorsNotes

Array of factor IDs + free text

Optional

Objectives

objectivesGoal, objectivesSuccess, objectivesMarketing

strings

Optional

Computed: UA/UE routing flag

uaUeRoutingFlag

"aim_pro_eligible"|"aim_x_preferred"|""

Computed

Computed: Brand routing flag

brandRoutingFlag

boolean

Computed

⚠️ Fields in state shape but NOT currently collected in wizard UI:
channels, hasNoLta, noLtaChannels, campaignMerge, campaignMergeChoice, split, additionalRegions — referenced in output tabs and populated in demo mode only. Decision needed: collected here, or pulled from CRM?

Outputs Produced

Output

Format

Destination

Tier recommendation (AIM X or AIM Pro)

Computed value

Output screen header + all tabs

Scope of Work

Generated document (HTML → PDF)

Tab 1. Client approval in-tool. PDF export (Phase 2).

Delivery Timeline

Milestone list with dates

Tab 2. Cascade delay logic. Audit log.

Data Schema

Filtered schema tables + CSV templates

Tab 3. CSV download per schema type.

Model Config

Internal technical document

Tab 4 (CSM only). Fed to Robert → app_config MongoDB.

uaUeRoutingFlag

Computed string

Model Config. Tier algorithm input.

brandRoutingFlag

Computed boolean

Model Config.

## 8. Integration Points

System

Integration type

Owner

Notes

K4A console (Kochava for Advertisers)

Embed — wizard hosted inside K4A as a console view

Anupam + Mayank

Wizard is upstream of ETL configuration UI

MongoDB app_config collection

Write — wizard answers → model config → Robert's LLM layer → app_config

Robert

Phase 2 only. Model Config tab out of scope for Phase 1. LLM layer to be designed; human review mandatory during pilot.

ETL UI (Mayank/Kai screens)

Read — wizard determines WHAT; ETL handles HOW

Anupam + Mayank

Phase 2: wizard auto-populates ETL form fields

Airflow trigger API

Trigger — Phase 2 only, auto-start pipeline from wizard completion

Anupam

Required for Phase 2; not in scope for Phase 1 (MVP)

Authentication / session

Read — authenticated user email → Timeline audit log author field

TBD

Replaces free-text name field in Timeline tab

CRM

Write — submission/handoff flow post-SoW approval

TBD

Architecture decision required — see Open Questions

Airbyte

Internal only — Kochava uses Airbyte for ETL infrastructure

Satish

AIM is the client-facing layer; Airbyte is internal. Client does NOT interact with Airbyte directly.

## 9. Technical Constraints & Considerations

React app architecture: Prototype is a single-page React app with inline Babel, no build step. Production implementation should use a proper React app with React Router. State shape and validation logic move almost unchanged; replace window.* globals with named imports.

Load order: In the prototype, script tags must be ordered: icons.jsx → tweaks-panel.jsx → step2-app.jsx → steps.jsx → setup-summary.jsx → output.jsx → app.jsx. This becomes import ordering in production.

State shape: All state lives in a single flat object updated via a set(patch) helper. uaUeRoutingFlag and brandRoutingFlag are recomputed on every state update.

Design system bridge: The prototype uses bridge aliases (--navy, --ink, etc.) → remove in production; use AIM X token names directly.

localStorage only: No server, no user account, no submission endpoint in the prototype. Production requires a submission flow and backend persistence.

Phase 9 loader: Currently a hardcoded 1.6-second timeout. In production, replace with a real API call to generate/save the configuration — loader duration should reflect real processing time.

Brand campaign ad stock: Brand campaigns require ~30-day ad stock vs 7-day for UA/UE. Mixed-network campaign-level combination is unknown territory — must be flagged explicitly in the generated Scope of Work.

AIM X zero-touch mandate (Sachin Dutta): Any manual engineering or CSM effort per AIM X onboarding breaks the product model. The wizard must achieve near-zero-touch for AIM X or AIM X cannot scale. This is a hard constraint on Phase 3.

## 10. Non-Functional Requirements

Performance: Wizard steps load instantly (client-side only in Phase 1). Output screen generation < 500ms (all client-side computation).

Security: Wizard answers include client company name, lead contact details, and budget data. Must be treated as commercial confidential. No wizard data should be logged client-side or in browser dev tools in production.

Authentication: CSM mode requires authenticated identity. Client mode should require authentication in production (client must be identifiable for CRM handoff). Free-text name field in Timeline must be replaced with session user email.

Scalability: Tool must support concurrent completion by multiple CSMs without state conflicts. Each wizard session is independent (localStorage-isolated per browser session in Phase 1; user-account-isolated in production).

Accessibility: Wizard must meet WCAG 2.1 AA. All form controls keyboard-navigable. Error messages associated with fields via aria-describedby. Focus management on step transitions.

Browser support: Chrome, Safari, Firefox, Edge (latest 2 major versions). Mobile browsers not required for Phase 1 (CSM-led flow is desktop).

Offline: Phase 1 prototype works offline (no server calls). Production requires connectivity for submission flow.

## 11. Open Questions for Engineering

Data Schema tab architecture — Option A (dynamic downloadable schema) vs Option B (API credential collection) vs both? — Owner: Satish + Anupam + Jaya. Blocks full Data Schema tab implementation.

Missing wizard fields — channels, hasNoLta, noLtaChannels, campaignMerge, campaignMergeChoice, split, additionalRegions exist in state but have no wizard UI. Collected here, or pulled from CRM/another source? — Owner: Robert + Gary.

CSM mode trigger — ✅ Resolved 2 June 2026. Super Admin role in K4A. No new mechanism required. — Owner: Anupam.

Submission/handoff flow — ✅ Resolved 2 June 2026. "Generate" triggers loader animation only (no notification). Notification (email to Kade/Jacob) fires on "Book my onboarding meeting". CSM Approval button manually activated post-meeting. — Owner: Gary + Sachin.

Multi-region UX — ✅ Resolved 2 June 2026. "Add another region" action at end of flow. Sequential processing (not parallel). Dropdown to switch between regions/apps. — Owner: Gary + Robert.

mmm-insights.jsx — Retain as a fifth output tab ("here's what you'll see") or remove before engineer handoff? — Owner: Gary.

PDF export — ✅ Moved to Phase 1. Approach TBD (print stylesheet recommended). — Owner: Anupam.

Objectives → Scope of Work — ✅ Phase 1 requirement confirmed 2 June 2026. Step 7 objectives must populate Tab 1 SoW. — Owner: Anupam.

Phase 9 loader replacement — What API endpoint does the loader call in production? What data does it save? What does it return? — Owner: Satish + Anupam.

app_config MongoDB auto-population — Robert's LLM layer to translate Model Config tab output into app_config fields. What is the exact schema? What fields remain manual? Human review process during pilot? — Owner: Robert + Gary. Dependency: Robert to share app_config schema document.

Anupam's master data schema — Agreed 8 May 2026. Needed to finalise conditional schema logic in Data Schema tab. — Owner: Anupam.

## 12. Engineering Impact Matrix

Top-level

Next-level

Impacted component

Lift

Reporting

New Report Type

Scope of Work generation + PDF export

3

Reporting

Modify existing reporting

Timeline with cascade delay + audit log

2

Analytics

New Analytics View

8-step wizard UI (K4A console embed)

3

Analytics

New Analytics View

Output screen — 4-tab results view

3

Analytics

Modify existing view

Data Schema tab (filtered subset + CSV download)

2

Attribution

Modify non-SAN logic

AIM X vs AIM Pro tier recommendation algorithm

2

Postbacks

Introduce new values

Submission/notification flow (TBD)

2

SDK

—

Not impacted

—

Additional work outside the matrix:

Work item

Owner

Lift

Phase

MongoDB app_config auto-population (LLM layer)

Robert

High

2

Backend schema API service (filter master schema by answers)

Satish

Medium

2

CSM mode trigger mechanism

Anupam

Low

1

Real PDF export for Scope of Work

Anupam

Medium

1

Real CSV template generation per schema type

Anupam

Low

1

Timeline backend persistence (currently localStorage only)

Satish

High

2

Multi-region collection step + model count multiplier

Anupam

Medium

2

Submission/CRM handoff flow

TBD

Medium

2

Authenticated user identity for Timeline audit log

Anupam

Low

1

Step 7 Objectives → Scope of Work

Anupam

Low

1

Remove bridge alias layer, reference AIM X tokens directly

Engineering

Low

1

## 13. Revision History

Version

Date

Author

Changes

v1

18 May 2026

Gary Danks

Initial draft — generated from Context Seed Rev 5 via AISDLC Phase 2

v2

21 May 2026

Gary Danks

Updated following AIM X prototype review session with Robert Pawlowicz (21 May 2026). New inputs: Google/iOS tweaks outstanding on Robert; offline attribution pool confirmed live in model trainer; data freshness indicator agreed (card turns transparent orange + warning triangle); dynamic daily pacing confirmed for AIM X investment page; budget target range (±5%) confirmed feasible; CPA to be added back to channel cards and table; marginal CPA terminology confirmed; 'Run Incremental Test' upsell CTA noted; KPI trend chart forward projection lines agreed; AIM X vs Pro boundary for offline clarified (retrospective offline analysis = Pro); direct API connection confirmed as hard requirement (file sharing not acceptable).

v3

3 June 2026

Gary Danks

Updated following AIM Product Weekly Sync (2 June 2026) with Mayank Ukey, Satish Karunanithi, Anupam Bhattacharyya, Sachin Dutta. Changes: (1) Navigation updated to K4A stepper component (not custom sidebar). (2) Budget slider range hardcoded $50K–$10M confirmed. (3) CSM mode trigger resolved — Super Admin role. (4) Model Config tab (Tab 4) moved to Phase 2. (5) PDF download moved to Phase 1. (6) Approval block: name + job title + timestamp confirmed. (7) Post-approval: questionnaire hidden, output tabs only remain. (8) Primary persona confirmed as advertiser; CSM reviews post-submission. (9) "Generate" notification removed — notification fires on "Book my onboarding meeting" only (email to Kade/Jacob). Manual CSM Approval button added to workflow. (10) Multi-region: sequential processing, "Add another region" at end of flow, dropdown for region/app switching. (11) Step 7 objectives → SoW confirmed as Phase 1. (12) External factors list confirmed at 6 (not 8) — prototype is source of truth. (13) Autosave on tab switch + manual save button confirmed. (14) Staggered delivery strategy adopted for AIM X feature rollout.

## 14. Source Documents

Context seed: docs/16-PRDs/061-aim-onboarding-decision-tool/product-context-seed.md (Rev 5)

UX design: Claude Design AIM Onboarding.html — 15 May 2026 prototype

Handover note: AIM Onboarding – Handover Note, 15 May 2026 16:18 (15pp — definitive UX source)

Playbook project file: projects/2026-04-onboarding-decision-tree.md

Related project: projects/2026-04-aim-x.md

Related project: projects/2026-04-etl-platform.md

AIM knowledge base: AIM Knowledge V2 Ragi (id: 95fb4166-95b3-4796-a45b-d900fdfa31eb)

Stakeholder sessions: Robert (Apr + 5 May + 6 May), Jacob + Kade (7 May), Anupam (8 May), Rhia Taplin (8 May), Jaya (8 May), Sachin + Satish (12 May)

Generated by: AISDLC Phase 2 Product Spec v1 (Kochava) · 18 May 2026
Updated v2: Claude claude-opus-4-5 · 21 May 2026
