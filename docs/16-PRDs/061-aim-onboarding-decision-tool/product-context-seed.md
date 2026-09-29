---
id: product-context-seed
title: Initiative One Pager — AIM Onboarding Tool
---

## Problem Statement

> "At the moment we give them a spreadsheet. It's got all questions. It's ignorant of their business model. There is no decision tree there." — Robert Pawlowicz, Lead Data Scientist

The current AIM onboarding process is open-ended by design: clients are asked what they want, they ask for everything, and the team says yes. This creates scope creep, delays model builds, and over-engineers ETL pipelines — the #1 internal pain point identified across every persona interviewed (CSM, Data Engineering, Data Science) in Gary's 8-interview discovery exercise (April 2026). The Klook incident is the defining example: Kochava promised a feature not yet built, Robert had to build a new framework from scratch, the client was lost, and the TikTok partnership was damaged.

## Visual Context (Prototype)

Status: High-fidelity prototype complete as of 15 May 2026 (Claude Design — AIM Onboarding.html). Reviewed and signed off across 7 stakeholder sessions (Robert, Jacob, Kade, Rhia, Anupam, Jaya, Sachin/Satish). Design system updated to canonical AIM X tokens (colors_and_type.css).

### Architecture: 8-Step Wizard + Output Screen

The onboarding tool is a **linear wizard (8 steps)** followed by a **generated output screen with 4 tabs**. The full flow is: (1) User lands on a welcome screen and starts fresh (or loads saved progress from localStorage); (2) They answer 8 steps of questions (phases 1–8); (3) A 1.6-second loading animation plays — phase 9 ("Generating your AIM setup…"); (4) The output screen is shown — phase 10 — with all four tabs pre-populated from their answers; (5) Incomplete answers show a validation banner with jump-links back to the offending step. A hovering Setup Summary panel (right edge, steps 2–6) provides a live model configuration read-back of every answer so far, including an estimated model count. State is auto-saved to localStorage on every change and restored on load. Steps 7 (Objectives) and 8 (Summary) do not show the panel — step 8 has its own full review layout.

A Demo mode (the `BRIGHTFIT_DEMO` constant in `app.jsx`) pre-fills all answers with a fictional subscription fitness app called BrightFit and jumps straight to the output screen — useful for demos and testing. The print export uses this mode.

The wizard has 8 steps (phases 1–8). Phase 9 is the loading animation; phase 10 is the output screen. Every step is accessible from the left sidebar nav at any time. The sidebar shows ✓ for completed steps and a red dot for validation errors (evaluated when output is reached). "Next" on step 8 triggers phase 9 (1.6s loader) then phase 10. **No phases are skipped** — phases 1–8 map directly to steps 1–8.

### Wizard Steps — Full Detail

#### Step 1 — Your Team (Phase 1)

Collects company identity and key contacts. These names appear verbatim on the generated Scope of Work.

- Company name — text input. Required. Error: "Please enter your company name."
- Project lead(s) — repeating name + email row. At least one required. "+ Add another lead" adds rows; × removes. Error if no row has both fields filled.
- Data lead(s) — same repeating row pattern. At least one required. "Same as project lead" checkbox (dataLeadSameAsProject) syncs data leads from project leads in real time — data lead rows hidden when ticked.
- Feeds output: Company name and lead names populate the Scope of Work header and signoff block.

#### Step 2 — About Your Product (Phase 2)

Establishes the product name, platforms, conversion split, modelling approach, and primary market. Drives model count in Setup Summary panel.

- App/brand name — text input. Required.
- 2.1 Platform selection — three toggle cards: iOS, Android, Web. Multi-select. At least one required. iOS triggers ATT/SKAN callout note. Drives model count.
- 2.2 Conversion share allocator — shown when ≥2 platforms selected. Slider + stepper per platform; must total 100%. With 2 platforms, moving one auto-adjusts the other. With 3 platforms, moving one proportionally redistributes among the other two.
- 2.3 Spend separable? — shown only when Web + ≥1 mobile platform selected. Three options: Yes/No/Not sure. "Not sure" reveals optional free-text textarea.
- 2.3 Cross-journey migration? — shown after spend separability answered. Same three options with a "See an example" disclosure. "Not sure" reveals optional textarea. Drives unified vs separate model recommendation.
- 2.4 Recommended model structure — shown after both 2.3 answers given. Two choice cards: Unified/Separate. Auto-recommended badge shown on system preference. User can override.
- 2.5 Primary market — select: US, UK, Germany, France, Canada, Other. "Other" reveals free-text input. Required. Locked until ≥1 platform selected.
- Feeds output: Platform list, share%, modelling approach, and region populate Scope of Work and Model Config. Setup Summary panel shows estimated model count from step 2 onward.

#### Step 3 — Marketing Setup (Phase 3)

Collects media budget, channel mix, attribution reliability, and campaign type split. Drives tier recommendation via the UA/UE routing flag.

- 3.1 Total paid media budget — slider with Monthly/Annual toggle. Monthly steps: $50K→$10M+. Annual steps: $5M→$200M+. Required. Stored as budgetMonthly/budgetAnnual + budgetPeriod.
- 3.2 Offline media — Yes/No. Required. If Yes: digital/offline spend split slider appears (default 80% digital/20% offline); offline schema included in Data Schema tab.
- 3.3 Digital media types — multi-select chips: Self-attributed (Meta, Google, TikTok, ASA), Standard attributed (DSP/programmatic), Affiliates/partner networks, Other. At least one required.
- 3.4 Attribution data reliability — Yes/No/Not sure. Required. If Yes or Not sure: chip multi-select of gap categories (Offline media, DSP, Affiliates, Branded/awareness, Mobile networks, Influencer, Podcasts, Not sure, Other). "Not sure" shows a reassurance callout.
- 3.5 Coverage confidence — "Are you confident your MMP captures the majority of installs?" Yes/No/Not sure. Required.
- 3.6 Organic/paid split per platform — slider per platform. % of installs that are paid. Defaults: iOS 65%, Android 80%, Web 50%.
- 3.7 Campaign types — three toggle cards: User Acquisition (UA), User Engagement (UE), Brand. At least one required. Toggling a type initialises its campaign share immediately at an equal split.
- 3.8 Budget split across campaign types — shown when ≥1 campaign type selected. Slider + stepper per active type. 2 types: constrained to 100%. 3 types: proportional redistribution. Total shown with ok/bad state. Warning banners fire when a type is below 5% or above 90%.
- Feeds output: Campaign shares → uaUeRoutingFlag and brandRoutingFlag (computed live). These feed the tier algorithm and Model Config tab. Offline spend → offline media schema row in Data Schema.

#### Step 4 — Business Model & Conversion Funnel (Phase 4)

Determines the KPI event and funnel structure. Required events are pre-selected and locked; optional events togglable. Changing business model resets the funnel.

- Business model — three choice cards: Subscription / E-commerce / Gaming. Required. Changing this resets the funnel.
- Gaming revenue type — shown only for Gaming: IAP/Ads/Both. Required if Gaming selected.
- App funnel events — checklist per business type. Required events locked on; optional togglable. Minimum 3 on. Each event has an editable name field and "Set as Primary KPI" button. One KPI must be set. Defaults: Subscription → Install + Registration + Trial Start + Subscription Start (KPI: Trial Start); E-commerce → Install + Registration + First Purchase (KPI: First Purchase). Funnel connector lines shown between events.
- Web funnel — shown if Web platform selected. Same pattern for web events. Minimum 2 on.
- LTV model — shown after funnel configured. "Include LTV model?" Yes/No. If Yes: "Cohort revenue data available?" Yes/No/Partial. If Partial: choice of partial or skip.
- Feeds output: Funnel events → dep_var slugs in Model Config. KPI event → Scope of Work. LTV cohort selection → conditional cohort rows in Data Schema (7-day, 30-day, 90-day, 360-day revenue).

#### Step 5 — Data Sources (Phase 5)

Collects MMP, web attribution, and ad spend data source configuration. Determines direct vs client-supplied split in Data Schema. "Other" selections inject custom connector TBC milestones into the Timeline.

- 5.0 Historical data per platform — tile selection per selected platform: 12–24 months / 24–36 months / 36 months+. 12–24 months shows a warning banner (seasonality partially modelled); 24+ months shows green confirmation.
- 5.1 MMP selection — shown if mobile platform selected. Chip: AppsFlyer, Adjust, Singular, Branch, Kochava, Other/none. Required. "Other/none" → custom connector TBC milestone in Timeline.
- MMP collection method — shown after MMP selected (except "Other/none"). Direct API (Recommended badge) / File / cloud. File → storage type chips (Google Drive, S3).
- AppsFlyer cohort access — shown only if AppsFlyer selected. Yes/No sub-question with explanatory note.
- 5.2 Web attribution sources — shown if Web platform selected. Multi-select chips: Google Analytics 4 (directly connectable), Google Drive, S3, Database, Other. "Other" checkbox → free-text input. Required if Web.
- 5.3 Ad spend data — collection method first (Direct API / File / Manual). Then source chips: MMP direct, Supermetrics, Google Drive, S3, Database, Other. "Other" → text input + Timeline TBC milestone. Required.
- Feeds output: MMP + collection method → provision plan in Data Schema (direct vs client-supplied). "Other" selections → custom connector TBC milestones in Timeline tab. History → seasonality note in Scope of Work.

#### Step 6 — External Factors (Phase 6, Optional)

No hard validation. Allows the client to flag factors AIM cannot detect from data alone.

- 6.1 Auto-included factors — two non-interactive read-only cards: Seasonality (✓ Included, or ⚠ Partial if history is 12–24 months) and National holidays (✓ Included, shows client's region).
- 6.2 Additional external factors — multi-select choice grid: Promotional activity, Competitor activity, Major product launch/rebrand, PR/earned media spike, Macro economic event, Regulatory change, Natural disaster/crisis, Other. Warning callout: client must be able to supply data for flagged factors.
- Free-text description — shown when any factor selected. Textarea for brief description of what happened, which market, rough significance. No dates or data required at this stage.
- Feeds output: Selected factors and notes appear in Scope of Work and Model Config.

#### Step 7 — Objectives (Phase 7, All Fields Optional)

Three open free-text fields. No validation. Purpose is to frame the kickoff call and give the onboarding manager context on client goals. Hint at bottom: "Optional — your answers are shared with your onboarding manager to frame the kickoff call."

- What are you hoping to achieve with your AIM model? (objectivesGoal) — textarea. Hint: "e.g. understand which channels are driving subscriptions, reduce wasted spend." Placeholder: "We want to understand which channels are driving our paid subscriptions…"
- What does success look like? (objectivesSuccess) — textarea. Hint: "e.g. a clear view of channel ROI, confident budget decisions." Placeholder: "We'd consider this successful if we can confidently shift budget between channels…"
- What are your marketing goals for the next 12 months? (objectivesMarketing) — textarea. Hint: "e.g. grow paid subscribers by 40%, reduce CPA by 20%." Placeholder: "Grow paid subscriber base by 40% YoY while maintaining or improving blended CPA."
- Feeds output: All three fields appear in the Step 8 review summary. Not currently surfaced in any output tab — should feed Scope of Work in production implementation.

#### Step 8 — Review Your Answers (Phase 8)

Full read-back of all answers across steps 1–7. No new input collected. The CTA "Generate my AIM setup" triggers the loader then output.

- Seven sections, each with an Edit button jumping directly to that step: Your team · Product · Marketing · Business & funnel · Data sources · External factors · Objectives.
- Each section renders key/value rows from state. Unanswered fields show a — placeholder at 45% opacity.
- The Setup Summary hover panel is hidden at this step — the full review layout replaces it.
- "Generate my AIM setup" CTA: setPhase(9) → setTimeout(() => setPhase(10), 1600).
- No validation runs here. Users can generate output with incomplete answers. Validation surfaces as a banner on the output screen with jump-back links per issue.

### Output Screen — 4 Tabs

The output screen (`src/output.jsx`, `window.OutputScreen`) renders at phase 10. It has a header showing the app name and an Edit answers button, then a tier recommendation summary (computed by `recommendTier(state)` — AIM X or AIM Pro), then a tab strip with three tabs visible to all users and one CSM-only tab. The **default tab on load is Scope of Work (tab 1)**. The tier recommendation is computed at the OutputScreen level and passed as a prop into each tab — it is not a tab in its own right. `TabTier` exists as a component in output.jsx but is currently unused in the tab strip.

**Tab 1 — Scope of Work (All Users, Default Tab):** A generated document in monospaced doc-card style, pre-populated from all wizard answers. Sections include: engagement details, platform & region, campaign types, business model & KPI, funnel events, data sources, external factors, and model structure. Each section has an editable inline notes textarea that expands on click. Approval block at the bottom: checkbox + name field. When approved, stamps the date and triggers the timeline start date. Stubbed "Download Scope of Work (PDF)" button — opens a modal in the prototype; needs real PDF generation in production. `onApproved` callback sets `approvedAt` state in OutputScreen, which is passed to the Timeline tab to anchor milestone dates.

**Tab 2 — Timeline (All Users):** A delivery plan showing estimated milestones from onboarding kickoff to first insights. Milestones differ by tier — AIM X has 7 milestones (fastest ~7 weeks); AIM Pro has 8 milestones including a Results Review (~9+ weeks). Date anchoring: Dates cascade from `approvedAt` (SoW approval timestamp) if set, otherwise from today. All milestone start dates are the next Monday on or after their calculated date. Cascade delay logic: If a milestone is marked Delayed and a new date is saved, all subsequent milestone start dates shift by the corresponding number of weeks. A delay banner shows total weeks of accumulated delay. Per-milestone interactions: Each row has a status selector (In progress / Complete / Delayed), a delay date picker (shown when Delayed is selected), a notes section (add/edit/delete notes), and an expand/collapse toggle. Status changes and note actions are all written to an audit Change Log at the bottom of the tab. Custom connector rows: If the user selected "Other/none" for MMP, web attribution, or ad spend, a custom connector row is injected into the milestone list with timing TBC. Author name: A persistent name field at the top of the tab is stamped onto every change log entry. In production this must be replaced with the authenticated session user's email — see Open Questions.

**Tab 3 — Your Data Schema (All Users):** Shows exactly what data the client needs to provide for their AIM model. Split into two sections: "Kochava collects automatically" (direct API connections) and "You need to provide" (file/cloud upload required). The provision plan is built dynamically by `buildProvisionPlan(state)` — which data sources are included, and whether they are direct or client-supplied, is derived entirely from wizard answers. Each client-supplied item shows a full schema table (field name, format, required/optional, notes) and a stubbed CSV template download button. Conditional schema rows: Offline media schema included only if `usesOffline === true`. LTV cohort rows (7-day, 30-day, 90-day, 360-day revenue) included only if LTV is in scope (`wantsLtv === true` and cohort data is available). Web schema included only if Web platform is selected. If all sources are direct (no client uploads needed), a green confirmation banner is shown instead of schema tables. The full shape of this tab is TBD — see Open Questions.

**Tab 4 — Model Config (CSM Only):** Internal document for the Lead Data Scientist and AIM delivery team. Only visible when `mode === "csm"`. Contains the full technical model configuration derived from wizard answers: fitting method, tier, platform(s), region, modelling approach (unified/separate); campaign types in scope, UA/UE routing flag, brand routing flag, campaign merge recommendation; media structure and channel list; funnel events as dep_var slugs (e.g. `subscription_start`), primary KPI flagged; SKAN configuration (iOS only) and ATT rate estimate; data source configuration and delivery method per source; LTV scope and cohort availability.

#### User Modes

The tool has two user modes. **Client mode** is the self-service path a client completes themselves. **CSM mode** is used by Kochava's Customer Success Managers on behalf of a client — it surfaces additional internal guidance notes (e.g. how each answer affects the Redshift partition, which platforms justify a standalone model, etc.).

#### Design System

The prototype uses the navy `#1E4B97` primary, card style, tab strip, and KPI card layout informed by the live Kochava MMM Insights dashboard visual language. It now uses the canonical AIM X design system tokens (`colors_and_type.css`) rather than inline custom properties.

## Proposed Solution Summary

An 8-step linear wizard embedded in the K4A console that guides clients (or CSMs) through AIM onboarding in under 10 minutes — producing a deterministic Tier Recommendation, Timeline, Data Schema, Draft Scope of Work, and internal Model Config automatically from structured answers. Replaces open-ended discovery with guardrailed, progressive-disclosure UX that eliminates scope creep at source.

The tool is upstream of the ETL configuration UI (Mayank/Kai screens). It determines what is needed; ETL handles how to get it. Questionnaire answers feed Robert's app_config collection in MongoDB.

Implementation phases:

- Phase 1 (now): Prototype → engineer uses outputs to configure ETL manually
- Phase 2: Wizard auto-populates ETL form fields in the UI
- Phase 3: Full auto-configuration — client self-serve, no CSM required for AIM X

## High-Level Product Requirements (Core Capabilities)

- Capability 1: 8-step linear wizard with progressive disclosure — Sub-questions only appear once their parent is answered. Locked sections are visually desaturated (opacity 0.32). Never asks for information that isn't immediately relevant. Steps: Who are you? → About your product → Marketing setup → Business & funnel → Data sources → External factors → Objectives → Summary.
- Capability 2: Live Setup Summary panel — Hovering panel on right edge (steps 2–6) reflects every answer in real time, including estimated model count. Hidden at Step 8 (replaced by full review layout). Builds client confidence during completion, eliminates surprise at output stage.
- Capability 3: Deterministic tier recommendation (AIM X vs AIM Pro) — Three-input algorithm: (1) monthly spend ≥ $250K AND history ≥ 12 months, (2) ≥ 2 non-MMP data sources, (3) uaUeRoutingFlag = aim_pro_eligible. AIM Pro can be manually downgraded to AIM X; AIM X can only be upgraded if auto-rec is Pro.
- Capability 4: Auto-generated Scope of Work (Tab 1 — default) — Editable inline textareas per section. Approval checkbox + name field that anchors Timeline milestone dates. Stubbed PDF export.
- Capability 5: Delivery Timeline (Tab 2) — Milestone plan auto-generated by tier. AIM X: 7 milestones (~7 weeks). AIM Pro: 8 milestones (~9+ weeks). Cascade delay logic — marking a milestone Delayed shifts all downstream dates. Full audit change log per milestone.
- Capability 6: Auto-generated Data Schema (Tab 3) — Filtered to exactly what the client needs based on their answers. Split into "collected automatically" vs "you need to provide." CSV template download per schema type. Conditional schema rows (offline media, LTV cohort, iOS SKAN, web).
- Capability 7: Model Config tab (CSM only — Tab 4) — Internal document for the data science team showing fitting method, tier, routing flags, dep_var slugs, SKAN config, ATT rate estimate. Only visible in CSM mode.
- Capability 8: Step 8 full review screen — Read-back of all answers across steps 1–7. Each section has an Edit button jumping directly to that step. "Generate my AIM setup" CTA triggers loader → output.
- Capability 9: Client objectives capture (Step 7) — Three optional free-text fields: goal, success definition, 12-month marketing targets. Feeds kickoff call framing. State keys: objectivesGoal, objectivesSuccess, objectivesMarketing.
- Capability 10: State persistence + save-and-return — All state auto-saved to localStorage on every change. Restored on load. Welcome screen offers "continue saved progress."
- Capability 11: Demo mode — BRIGHTFIT_DEMO constant pre-fills a fictional subscription fitness app (BrightFit) and jumps to output screen. Used for sales demos, testing, and print export.
- Capability 12: Non-blocking validation — Users can proceed through steps with incomplete answers. Validation surfaces at output stage as a banner with jump-back links to offending steps. Hard blocks only apply on "Next" when critical fields are empty.

## Key Business Logic (Implemented in Prototype)

### Tier Recommendation Algorithm

Defined in `recommendTier(state)` in `output.jsx`. Starts at AIM X, upgrades to AIM Pro if any of three conditions are met: (1) Spend + history: monthly spend ≥ $250K AND history is 12–24 or 24+ months; (2) Data sources: ≥ 2 additional non-MMP data sources connected; (3) UA/UE routing flag: `uaUeRoutingFlag === "aim_pro_eligible"` — set when both UA and UE are present at mid share (10–90%). AIM Pro can always be manually downgraded to AIM X. AIM X can only be upgraded if the auto-recommendation is Pro.

### UA/UE Routing Flag

Computed live on every state update inside `set()` in `app.jsx`. Three possible values: `aim_pro_eligible` — both UA and UE present at mid share (10–90%), can qualify for Pro tier; `aim_x_preferred` — either absent, or one is skewed (<10% or >90%), AIM X is the better fit; `""` — UA/UE answers not yet completed.

### Funnel Defaults by Business Type

Subscription: Install (required), Registration, Trial Start, Subscription Start, Revenue. KPI default: `subscription_start`. E-commerce/IAP: Install (required), Registration, First Purchase (required + default KPI), Total Purchases, Revenue. Gaming follows e-commerce funnel internally. If the user changes business model mid-flow, the funnel resets. The funnel object carries a `business` field to detect this.

### Unified vs Separate Models (Web + Mobile)

Only triggered when Web AND a mobile platform are both selected. If spend is not separable OR cross-journey migration is "yes" → recommend unified. If spend is separable AND no cross-journey migration → recommend separate. Auto-set but overridable.

### Timeline Milestone Logic

Milestones differ by tier: AIM X has 7 milestones (fastest ~7 weeks); AIM Pro has 8 milestones including a Results Review (~9+ weeks). Dates cascade from `approvedAt` (SoW approval timestamp) if set, otherwise from today. All milestone start dates are the next Monday on or after their calculated date. If a milestone is marked Delayed and a new date is saved, all subsequent milestone start dates shift by the corresponding number of weeks. A delay banner shows total weeks of accumulated delay.

## Design Principles (Implemented)

| # | Principle | Implementation |
|---|---|---|
| 01 | Progressive disclosure | Sub-questions hidden until parent answered; locked sections use opacity 0.32 |
| 02 | Recommendations over blank slates | Default funnel events pre-selected by business type; "Recommended" badge on system preference |
| 03 | Contextual reassurance | Every anxiety-inducing answer has inline follow-up copy (attribution gaps, short history, iOS ATT) |
| 04 | Depth hidden by default | Technical detail (MMP delivery, AppsFlyer cohort, campaign merge) only appears as progressive sub-questions |
| 05 | Live read-back builds confidence | Setup Summary panel reflects every answer in real time including model count estimate |
| 06 | Non-blocking validation | Users proceed with incomplete answers; validation banner at output with jump-links |
| 07 | Outputs are immediately useful | SoW is editable and approvable; Schema tab has copy-ready field names + CSV download |
| 08 | No visual noise | No emoji, no decorative gradients; minimal 1.6px icon stroke set used sparingly |
| 09 | Mode parity | CSM and Client share the same UI structure; CSM additions are inline notes, never a separate screen |

## Business Case & Value

| Metric | Current State | Target State |
|---|---|---|
| Onboarding setup time | 4–8 weeks target, frequently overrun | 4–8 weeks consistently achieved |
| Scope creep incidents | Multiple per quarter per CSM | < 1 per quarter per CSM |
| CSM effort per AIM X client | High — manual schema work, open-ended discovery | Near-zero — wizard self-serve, SoW auto-generated |
| Model config time (Robert) | Hours/days of manual interpretation | < 10 minutes from wizard completion |
| Client data team clarity | Low — over-provide out of caution | High — exact schema with only required fields |
| Enterprise customisation for mid-market | Common (Klook, Cabify examples) | Eliminated — Enterprise path is explicit and opt-in |
| AIM X zero-touch viability | Not possible without this tool | Enabled — this is the prerequisite |
| Revenue protection | At risk from scope creep incidents | Protected — constrained agreements, no over-commitment |

## Target Audience (Personas)

- Primary — Client self-serve (Phase 3): Client data team / marketing manager — completes the wizard before the onboarding call. Receives tier recommendation, exact data schema, and draft SoW.
- Primary — CSM-assisted (now): CSM (Kade, Jacob) — runs the wizard with or on behalf of the client. Receives Model Config tab (CSM-only) and internal guidance notes per answer.
- Secondary — Data Engineering (Jaya, Anupam): Receives pre-validated schema and ETL config before ETL work begins. No ambiguous specs.
- Secondary — Data Science (Robert): Receives auto-generated model config populated into app_config MongoDB. No manual interpretation.
- Anti-persona 1: Enterprise clients requiring bespoke model design — explicitly routed to AIM Enterprise path.
- Anti-persona 2: Clients with in-house data science teams wanting to configure the model themselves — AIM Pro advanced settings.
- Anti-persona 3: Clients with genuinely novel business models not mapping to Subscription / E-commerce / Gaming — bespoke/enterprise path.

## Risks & Dependencies

### Technical Risks

- LLM config generation accuracy — auto-generated app_config must be validated by Robert before CSMs rely on it. Human review mandatory during pilot.
- Missing wizard fields — channels, hasNoLta, noLtaChannels, campaignMerge, campaignMergeChoice, split exist in the state shape but are not surfaced in the current wizard UI. They are referenced in output tabs and populated in demo mode only. Decision needed: collected in this wizard, or pulled from CRM?
- Backend schema service not yet built — short term: hardcoded frontend logic. Medium term: backend API service (Satish to design).
- Phase 9 loader hardcoded — 1.6-second timeout with no real async work behind it. In production this should be replaced with an actual API call to generate/save the configuration.

### Business Risks

- CSM resistance — moving from judgment-based to constrained flow requires change management. Frame as protection from over-commitment, not restriction.
- AIM X zero-touch mandate (Sachin Dutta) — any manual engineering or CSM effort per AIM X onboarding breaks the model. This wizard must achieve near-zero-touch for AIM X or AIM X cannot scale.
- Brand campaign ad stock — brand campaigns require ~30-day ad stock vs 7-day for UA/UE. Mixed-network campaign-level combination is unknown territory — must be flagged in SoW.

### Open Questions (from Handover Note §11)

- Missing wizard fields: Decide whether history/spend/channels/noLta/additionalRegions are collected here or pulled from CRM/another source.
- CSM mode trigger: How does a CSM identify themselves — URL param ?mode=csm, internal login, or separate URL? Logic is in place; needs a trigger.
- Submission/handoff flow: What happens after "Generate my AIM setup" — email to CSM, CRM record write, or in-app review step? No backend/data persistence currently exists (all state is localStorage only).
- Multi-region: additionalRegions exists in state and Setup Summary panel but has no UI step. Region-based model count multipliers not implemented.
- mmm-insights.jsx: Contains a full MMM Insights dashboard preview component. Not imported in the wizard flow. If intended as a "preview of what you'll get" → slot as fifth output tab. If not, remove before engineer handoff.
- PDF/CSV export: "Download Scope of Work (PDF)" and "CSV template" buttons are currently stubbed (modal explains they're not built). Real implementations needed.
- Objectives → Scope of Work: Step 7 objectives (objectivesGoal, objectivesSuccess, objectivesMarketing) are not currently surfaced in any output tab. Should feed Scope of Work in production implementation.
- Data Schema tab — shape and delivery TBD (discuss with Satish, Anupam & Jaya): The Data Schema tab is currently partially stubbed. Two directions have been raised:
    - Option A — Dynamic downloadable schema file: Schema generated entirely from the user's onboarding answers. Conditional logic already identified: offline media → include/exclude offline spend schema; LTV opted out → remove all cohort fields (cohort_day_7_revenue, cohort_day_30_revenue, retention columns); platform selection gates platform-specific fields (iOS SKAN fields, web pixel fields); MMP selection surfaces MMP-specific column names (AppsFlyer vs Adjust etc.). Output is a downloadable CSV/Excel template pre-configured with exactly the fields the client needs — no more, no less.
    - Option B — API key/data source connection: Rather than (or alongside) a manual schema download, allow the user to input API credentials for each data source they selected — MMP API key, Google Analytics property ID, etc. Enables direct data pull rather than manual file upload.
    - These two options are not mutually exclusive. Decision needed with Satish, Anupam, and Jaya.
- Timeline audit log — authenticated user identity: The timeline currently prompts for a free-text "your name" field. In production, the user will be authenticated — their email should be read from session/auth context and stamped automatically on every timeline event. No manual name entry needed.

### Dependencies

- Robert's app_config schema document — must be shared with Gary before Phase 2 spec can be finalised
- Airbyte architecture (Satish) — internal use approved; external/client-facing pending legal. AIM is the exposed layer.
- Sales Funnel Events tab in ETL UI (Anupam + Mayank) — confirmed required, not yet built
- Airflow trigger API exposed in ETL UI (Anupam) — required for Phase 2 auto-population
- Anupam to share updated master data schema (agreed 8 May 2026, pending)
- Jaya's external factors / market data schema tables (pending via Anupam)

## Out of Scope

- Campaign-level attribution — AIM Pro only; not in AIM X scope
- Offline media handling — separate ETL project; post-ingestion tagging only
- In-form region assignment — notifications to onboarding@aim.io centrally; no regional routing in the form
- Airbyte direct client access — AIM is the client-facing layer; Airbyte is internal infrastructure
- Full data quality validation — schema/format validation is in scope; deep data quality is not
- AIM Enterprise bespoke configuration — edge cases route explicitly to Enterprise path
- Look-back periods and weekly grouping — system/engineering config, not customer-facing
- Offline/cross-platform network identification — happens after ingestion, not in questionnaire

## Code Architecture (for Engineer Handoff)

The app is a single-page React app delivered as an HTML file with inline Babel. All JSX components are loaded as separate script files and exposed to each other via `window.*` globals. No build step, no npm, no bundler — the standalone export works completely offline.

File structure:

- AIM Onboarding.html — entry point; CSS tokens bridge + component styles, `<div id="root">`, script tags
- colors_and_type.css — AIM X design system tokens (sourced from vuetify.ts + styles.scss)
- src/app.jsx — App root, phase router, state machine, validation, BRIGHTFIT_DEMO
- src/output.jsx — OutputScreen, TabTier, TabSchema, TabScope, TabConfig, TabTimeline
- src/steps.jsx — Step1Mode, Step4Funnel, Step5Channels, Step6Mmp, Step7Objectives, Step8Review, etc.
- src/step2-app.jsx — Step2AppInfo
- src/setup-summary.jsx — SetupSummary hover panel
- src/tweaks-panel.jsx — TweaksPanel, useTweaks hook
- src/icons.jsx — Icon component (monoline SVG set) + RecBadge
- src/mmm-insights.jsx — MMM Insights dashboard preview (currently unused)

Load order matters. Script tags must be ordered: `icons.jsx` → `tweaks-panel.jsx` → `step2-app.jsx` → `steps.jsx` → `setup-summary.jsx` → `output.jsx` → `app.jsx`. Each file reads `window.*` globals set by earlier scripts. `app.jsx` must be last as it calls `ReactDOM.createRoot`.

Phase routing: Phases 1–8 map directly to steps. Phase 9 = 1.6s loading animation. Phase 10 = output screen. **No phases are skipped.** All state lives in a single flat object updated via a `set(patch)` helper that merges partial updates and recomputes `uaUeRoutingFlag` and `brandRoutingFlag` on every change.

Engineer handoff note: The component structure maps cleanly onto a React app with React Router. State shape and validation logic move almost unchanged; replace window.* globals with named imports. The :root block in AIM Onboarding.html uses bridge aliases (--navy, --ink, etc.) pointing to canonical AIM X tokens. In the production implementation, components should reference AIM X token names directly and the bridge layer should be removed.

## Engineering Impact Matrix

| Top-level | Next-level | Impacted | Lift Factor |
|---|---|---|---|
| Reporting | New Report Type | Draft Scope of Work generation + PDF export | 3 |
| | Modify existing reporting values / columns | Dynamic timeline with cascade delay logic + audit log | 2 |
| Analytics | New Analytics View (non superset) | 8-step wizard UI (K4A console embed) | 3 |
| | New Analytics View (non superset) | Output screen — 4-tab results view | 3 |
| | Modify existing view | Data Schema tab (filtered subset + CSV download) | 2 |
| Attribution | Modify non-SAN attribution logic / waterfall | AIM X vs AIM Pro tier recommendation algorithm | 2 |
| Postbacks | Introduce new values to scope of postbacks | Submission/notification flow (TBD — email vs CRM) | 2 |
| SDK | — | Not impacted | — |

Additional engineering work outside the impact matrix:

- MongoDB app_config auto-population from wizard answers (Robert's LLM layer) — High lift
- Backend schema API service to filter master schema based on answers (Satish) — Medium lift, Phase 2
- CSM mode trigger mechanism (URL param / login / separate URL) — Low lift
- Real PDF export for Scope of Work — Medium lift
- Real CSV template generation per schema type — Low lift
- Timeline backend persistence (currently localStorage only) — High lift
- Multi-region collection step + model count multiplier — Medium lift, Phase 2
- Schema lock enforcement on SoW approval — Low lift
- Submission/CRM handoff flow — Medium lift (TBD on architecture decision)
- Authenticated user identity for Timeline audit log — Low lift (requires auth integration)
- Step 7 objectives → Scope of Work output — Low lift

## Key References

- Primary source: AIM Onboarding.html — Claude Design prototype, 15 May 2026
- Handover note: AIM Onboarding – Handover Note, 15 May 2026 16:18 (this document's source of truth — Rev 3, 15pp)
- Playbook file: projects/2026-04-onboarding-decision-tree.md (AIM Projects playbook)
- Related project: projects/2026-04-aim-x.md
- Related project: projects/2026-04-etl-platform.md
- Config Schema Audit: technical-refs/aim-config-schema/app-config-variable-behavior-audit.md
- AI Config System Design: technical-refs/aim-config-schema/ai-assisted-config-system-design.md
- AIM SOP Matrix: AIM Knowledge V2 Ragi (id: 95fb4166-95b3-4796-a45b-d900fdfa31eb)
- AIM Technical White Paper: AIM Knowledge V2 Ragi (id: 95fb4166-95b3-4796-a45b-d900fdfa31eb)
- Stakeholder sessions: Robert (Apr + 5 May + 6 May), Jacob + Kade (7 May), Anupam (8 May), Rhia Taplin (8 May), Jaya (8 May), Sachin + Satish (12 May)

## Revisions

| Date | Change | Source |
|---|---|---|
| 15 May 2026 | Initial creation — synthesised from 7 stakeholder sessions, 8 internal interviews, AIM knowledge base | Gary Danks / AISDLC Phase 1 |
| 15 May 2026 | Rev 2 — Full rewrite from Claude Design Handover Note (15 May 2026 14:52). Wizard structure, output tabs, business logic, code architecture, open questions all updated. | Gary Danks / AISDLC Phase 1 Rev 2 |
| 15 May 2026 | Rev 3 — Added two new open questions: Data Schema tab shape/delivery (Option A vs B) and Timeline audit log authenticated identity. | Gary Danks / AISDLC Phase 1 Rev 3 |
| 15 May 2026 | Rev 4 — Updated wizard from 6-step to 8-step based on live prototype screenshot (Step 1 of 8). Added Steps 7 & 8 as flagged gaps. | Gary Danks / AISDLC Phase 1 Rev 4 |
| 15 May 2026 | Rev 5 — Complete rewrite from Claude Design Handover Note (15 May 2026 16:18, 15pp). Steps 7 (Objectives) and 8 (Review) now fully documented. Output tabs restructured: Tab 1=SoW (default), Tab 2=Timeline (NEW — milestones, cascade delay, audit log), Tab 3=Data Schema, Tab 4=Model Config. Step 3 and Step 5 substantially expanded with field-level detail. Phase routing confirmed: phases 1–8 map directly to steps, no phases skipped. All open questions updated. | Gary Danks / AISDLC Phase 1 Rev 5 |

---

Generated by: Claude claude-opus-4-5 — AISDLC Phase 1 Context Seed Rev 5 (Kochava) · 15 May 2026
