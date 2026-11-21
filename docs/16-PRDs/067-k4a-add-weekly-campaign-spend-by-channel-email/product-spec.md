---
id: product-spec
title: Product Specification
---

## 1. Business Logic & Context

### Problem

The marketing team at Meridian Travel currently pulls campaign spend data manually every Monday. This recurring manual effort is slow, error-prone, and does not scale ahead of their summer campaign launch. They need a reliable, automated weekly view of how much they are spending per marketing channel.

### Value

- **Eliminates recurring manual work:** replaces the weekly Monday data pull with an automated delivery.
- **Timely visibility:** marketing sees spend-by-channel on a predictable weekly cadence, supporting faster budget decisions ahead of the summer campaign.
- **In-product surface:** the summary data lives in the existing K4A dashboard; the weekly email includes a link to that dashboard (confirmed by the requester). No new K4A screen or UI build is required.

### Business Rules

- **BR-1 — Cadence:** The spend summary is generated and delivered on a weekly cadence. [DECISION REQUIRED: DEC-001]
- **BR-2 — Content:** Each summary breaks down campaign spend by marketing channel.
- **BR-3 — Recipient:** The summary is delivered to the marketing team at Meridian Travel. The recipient resolves to the marketing team distribution list, and their marketing ops lead controls who is on that list. Whether access to the data should additionally be limited by role or permission remains an open business decision. [DECISION REQUIRED: DEC-002]
- **BR-4 — Plan entitlement:** The customer is on the standard plan. The feature is gated on standard-plan entitlement; the system must verify entitlement before producing or delivering a summary. The standard plan already includes automated/scheduled recurring email reports, so enabling scheduled delivery for this customer requires no extra entitlement or feature work.
- **BR-5 — Dual surface:** The spend-by-channel summary is delivered as an email, and that email includes a link to the existing K4A dashboard where the underlying spend data already lives. No new on-screen view or UI build is created in K4A for this feature.
- **BR-6 — Channel set:** The breakdown includes a fixed set of five channels every week — paid search, paid social, display, email, and affiliate. This set is the same every week and does not vary by campaign.
- **BR-7 — Detail level:** The summary content shows total spend per channel. A prior-week comparison is included if it is low-effort to add; impressions and conversions are explicitly excluded. (Resolves DEC-005 / OQ-002.)

### Knowledge-Graph Reconciliation Note

Two independent codebase knowledge-graph lookups were run against this requirement and they **do not fully agree** on which backend services are involved. One lookup surfaced a reporting/notifications/accounts-oriented cluster; a second independent lookup surfaced a separate SAM-oriented cluster, with only the email-notification service appearing in both. This is a genuine topology ambiguity, not a formatting discrepancy: K4A may be layered on a SAM backend, with logically-equivalent roles appearing under two naming conventions. **No service map should be treated as settled until an engineer who knows the K4A/SAM topology confirms the real repositories.** The full per-service evidence is retained for the engineering team as a labeled appendix at the end of this document.

---

## 2. User Flows (Visuals)

```mermaid
flowchart TD
    A["Weekly generation (system-triggered)"] --> B{"Is customer entitled (standard plan)?"}
    B -->|"no (not entitled)"| C["Skip generation and record a skipped/blocked status"]
    B -->|"yes (entitled)"| D["Query spend-by-channel data for the week"]
    D --> E{"Is channel spend data complete?"}
    E -->|"no (missing/incomplete data)"| F["Apply missing-data handling and mark affected channels"]
    E -->|"yes (complete)"| G["Build spend-by-channel summary"]
    F --> G
    G --> H["Deliver summary email (including a link to the existing K4A dashboard) to marketing team"]
    H --> I{"Did email delivery succeed?"}
    I -->|"yes (delivered)"| J["Record delivery success"]
    I -->|"no (delivery failed)"| K["Retry with bounded backoff, then record failure and alert"]

    J --> T["Recipient clicks the email link to the existing K4A dashboard"]
    T --> U{"Is user authorized (entitlement/auth re-derived server-side)?"}
    U -->|"no (unauthorized)"| O["Deny access (server-side 403)"]
    U -->|"yes (authorized)"| P["Existing K4A dashboard shows spend-by-channel data"]
```

---

## 3. Functional Requirements (The Core)

### Feature Area A — Plan Entitlement Gating

#### FR-001 — Verify standard-plan entitlement before generating a summary
```gherkin
Given a customer is scheduled to receive the weekly spend-by-channel summary
When the weekly generation is triggered for that customer
Then the system verifies the customer's plan entitlement against the authoritative account source before any summary is generated
And generation proceeds only if the customer is entitled under their plan
```

#### FR-002 — Block generation for a non-entitled customer
```gherkin
Given a customer is not entitled to the weekly spend-by-channel summary under their plan
When the weekly generation is triggered for that customer
Then the system does not generate or deliver a summary
And the system records a skipped/blocked status identifying the customer and the reason (not entitled)
```

#### FR-003 — Fail closed when entitlement cannot be verified
```gherkin
Given the authoritative account/entitlement source is unreachable
When the weekly generation is triggered for a customer
Then the system does not generate or deliver the summary for that run
And the system records a failure status indicating entitlement could not be verified
And the system fires an operational alert for the failed verification
```

#### FR-004 — Enforce entitlement server-side when the dashboard link is followed
```gherkin
Given a user follows the email link to view spend-by-channel data in the existing K4A dashboard
When the request reaches the backend
Then the backend re-derives the user's entitlement from the authoritative source on that request
And an ineligible user's request is rejected server-side with a 403, regardless of what the UI displays
```

### Feature Area B — Weekly Scheduling & Generation

#### FR-005 — Trigger weekly generation on the defined cadence
```gherkin
Given the weekly spend-by-channel summary feature is enabled for an entitled customer
When the defined weekly trigger time is reached
Then the system initiates generation of that customer's spend-by-channel summary for the covered week
```

#### FR-006 — Prevent overlapping concurrent runs for the same customer
```gherkin
Given a weekly generation run is already in progress for a customer
When another weekly generation trigger fires for the same customer before the first completes
Then the system does not start a second concurrent run for that customer
And the duplicate trigger is rejected rather than queued or duplicated
```

#### FR-007 — Idempotent generation for a re-run of the same week
```gherkin
Given a weekly summary has already been generated for a customer for a specific covered week
When generation is re-run or retried for that same customer and the same covered week
Then the system produces the same end state for that week rather than a duplicate summary
And no spend figures are double-counted as a result of the re-run
```

### Feature Area C — Spend-by-Channel Content

#### FR-008 — Summarize spend broken down by channel
```gherkin
Given spend data exists for a customer for the covered week
When the summary is generated
Then the summary presents campaign spend aggregated and broken down by each channel in the defined channel set
And a per-channel spend value is shown for each channel in that set
And the summary does not include impressions or conversions
```

#### FR-008a — Include a prior-week spend comparison when low-effort
```gherkin
Given a prior week's spend-by-channel summary exists for the customer
When the current week's summary is generated and a prior-week comparison can be derived at low implementation cost
Then the summary includes, per channel, a comparison of current-week spend to the prior week's spend
And if the prior-week comparison is not low-effort to add, it is omitted without blocking the rest of the summary
```

#### FR-009 — Handle a channel with missing or incomplete data
```gherkin
Given one or more channels have missing or incomplete spend data for the covered week
When the summary is generated
Then the summary is still produced for the channels with available data
And each channel with missing or incomplete data is explicitly marked as such rather than shown as zero spend
```

#### FR-010 — Handle a week with no spend data at all
```gherkin
Given no spend data is available for any channel for a customer for the covered week
When the summary is generated
Then the system produces a summary that clearly indicates no spend data is available for the week
And the summary is not silently suppressed
```

### Feature Area D — Email Delivery

#### FR-011 — Deliver the weekly summary email to the marketing team
```gherkin
Given a spend-by-channel summary has been successfully generated for an entitled customer
When delivery is initiated
Then the summary email is sent to the customer's marketing team distribution list (membership controlled by the customer's marketing ops lead)
And a delivery success status is recorded identifying the customer and the covered week
```

#### FR-012 — Retry a failed email delivery with bounded backoff
```gherkin
Given email delivery of a generated summary fails due to a downstream delivery error
When the system handles the failure
Then the system retries delivery with exponential backoff up to 3 attempts
And if all attempts fail, the system records a delivery-failed status and fires an operational alert
And the system never retries indefinitely
```

#### FR-013 — Distinct handling for named delivery failure conditions
```gherkin
Given email delivery fails
When the failure is a timeout, a delivery-service-unavailable error, or a rate-limited response
Then the system records a distinct named status for each of those conditions rather than a single generic error
And a rate-limited response's Retry-After header is honored rather than retrying on a fixed schedule
```

### Feature Area E — Email Link to the Existing K4A Dashboard

#### FR-014 — Include a link to the existing K4A dashboard in the email
```gherkin
Given a spend-by-channel summary email is delivered to an entitled customer
When the email is composed
Then the email includes a link to the existing K4A dashboard where the spend-by-channel data lives
And no new on-screen view or UI build is created in K4A for this feature
```

#### FR-015 — Enforce authorization when the dashboard link is followed
```gherkin
Given a recipient follows the email's link to the existing K4A dashboard
When the request reaches the backend
Then the backend re-derives the user's entitlement and viewer authorization from the authoritative source on that request
And an unauthorized or ineligible user is rejected server-side with a 403, regardless of what the UI displays
```

Now reporting the decisions I flagged in Sections 0–3.

## 4. Information Requirements

The system must track the following business information to support the Functional Requirements above. Attribute-level definitions, types, and storage are intentionally deferred to eng-plan.

- **Weekly spend-by-channel summary** — the generated artifact for a given customer and covered week, containing per-channel spend. System-derived, built by aggregating spend data. This is the core content delivered by email (FR-008, FR-011); the email links to the existing K4A dashboard where the same data is viewable (FR-014).
- **Per-channel spend figure** — the spend amount attributed to each channel in the defined channel set for the covered week. System-derived from the underlying campaign spend data. Drives the breakdown in FR-008.
- **Prior-week per-channel spend comparison** — per channel, the prior week's spend figure used to compute a current-vs-prior comparison when low-effort to derive. System-derived from the prior covered week's summary. Supports FR-008a.
- **Channel set** — the fixed list of marketing channels included in the breakdown: paid search, paid social, display, email, and affiliate (see Section 1, BR-6). The same five channels appear every week and the set does not vary by campaign. Determines which channels appear in every summary.
- **Channel data-completeness state** — per channel, whether spend data for the covered week is complete, incomplete, or missing. System-derived. Drives the missing-data marking in FR-009 and FR-010.
- **Covered week** — the time window a given summary represents. System-derived from the generation trigger. Needed to label summaries (FR-008) and to key idempotent re-runs (FR-007).
- **Customer plan entitlement** — whether the customer's plan permits the weekly summary feature. External, verified against the authoritative account source. The standard plan already includes automated/scheduled recurring email reports, so no new entitlement is required for this feature. Gates generation and the on-screen view (FR-001–FR-004).
- **Marketing-team recipient list** — the marketing team distribution list the email is delivered to for a customer, with membership controlled by the customer's marketing ops lead (see Section 1, BR-3). Used in FR-011.
- **Viewer authorization state** — whether a given user following the email link to the existing K4A dashboard is authorized to see the spend data. External/system-derived from the authoritative auth source on each request. Gates FR-004 and FR-015.
- **Generation run status** — per customer and covered week, the outcome of a generation run (succeeded, skipped/blocked, entitlement-unverifiable, failed). System-derived. Supports FR-002, FR-003, and operational visibility.
- **Email delivery status** — per customer and covered week, the outcome of delivery (delivered, retrying, timed-out, service-unavailable, rate-limited, failed). System-derived. Supports FR-011–FR-013.
- **Audit record of generation and delivery events** — an append-only trail of who/what triggered each run, the entitlement decision, and the delivery outcome. System-derived. Supports the audit guardrail in Section 5.

---

## 5. Non-Functional Requirements & Guardrails

### Performance & Capacity

- **NFR-001 — Dashboard view latency:** When a recipient follows the email link, the existing K4A dashboard's spend-by-channel view must return within a p95 latency target measured under representative concurrent load (e.g. N concurrent requests, p95 ≤ X ms). The concrete p95 target and concurrency level are an engineering decision to be set in eng-plan based on the chosen aggregation path.
- **NFR-002 — Spend-query rate cap:** The spend-by-channel aggregation runs against a shared data/query resource. Both the generation (batch) query path and the on-screen (read) query path must have their own separate rate caps rather than a single shared value, to protect the shared resource. Concrete caps are an engineering decision for eng-plan.
- **NFR-003 — Weekly delivery timeliness:** Each entitled customer's weekly summary must be generated and delivered within a bounded window after its weekly trigger time, so the marketing team receives it on a predictable cadence replacing the Monday manual pull. The exact delivery-window target and the day/time of the weekly trigger are a product decision. [DECISION REQUIRED: DEC-006]
- **NFR-004 — Generation & delivery monitoring:** The system must alert when the generation-failure rate or delivery-failure rate exceeds 1% for 5 minutes, sustained, and when any single customer's weekly summary fails to deliver after all retries. This directly protects the customer-facing email reliability called out in the HIGH risk tier.

### Reliability & Idempotency

- **NFR-005 — Bounded retry on delivery:** Email delivery failures retry with exponential backoff, capped at 3 attempts, never indefinitely (reinforces FR-012). A 429/rate-limited response honors the Retry-After header (FR-013).
- **NFR-006 — Idempotent weekly generation:** Generation is idempotent on the (customer + covered week) key: a retried or re-run generation for the same key overwrites/updates the existing summary rather than producing a duplicate or double-counting spend (reinforces FR-007).
- **NFR-007 — No overlapping scheduled runs:** The weekly scheduler must not start an overlapping run for the same customer while one is in progress; concurrent re-runs are disabled at the scheduler level as a complement to idempotency (reinforces FR-006).

### Security & Access Control

- **NFR-008 — Server-side entitlement/authorization enforcement:** Entitlement and viewer authorization must be enforced server-side and re-derived from the authoritative source on every request. A direct API call from an ineligible plan or unauthorized user must be rejected (403) regardless of UI state; the gate must fail closed if the auth/account source is unreachable (reinforces FR-003, FR-004, FR-015).
- **NFR-009 — No sensitive data in logs:** The system must never log raw recipient email addresses, credentials, secrets, or tokens. Where a recipient identity must appear for operational diagnosis, it must be masked.
- **NFR-010 — Audit trail for generation and delivery:** Each generation run and delivery attempt must write an append-only, immutable audit record capturing actor/trigger identity, customer/account identity, action (generate / deliver / skip), covered week, timestamp, and outcome/status. Records must not be editable or deletable by any role or code path. Audit writes must be asynchronous so they never block generation or delivery, but an audit-write failure must fire an immediate operational alert rather than fail silently. Minimum retention: 12 months. [DECISION REQUIRED: DEC-007]

### Risk Mitigation

The Context Seed classifies this work as **HIGH risk** (hard 5-week deadline, multi-service blast radius, customer-facing email reliability, new UI/UX). Each listed risk and its mitigation condition:

- **NFR-011 — Kill-switch for the feature:** Because this is HIGH risk and spans multiple services, the weekly summary feature (generation + delivery) must be disableable via configuration without a code deploy. An on-call engineer must be able to disable it during an incident without separate approval. New-request propagation target: 5 minutes; a run already in flight at the time of the flip completes under prior behavior. *Mitigates: multi-service blast radius / customer-facing reliability.*
- **NFR-012 — Email reliability mitigation:** Mitigated when FR-011–FR-013, NFR-004, and NFR-005 are all in place: delivery is retried with bounded backoff, each named failure condition is distinctly recorded, and sustained failure triggers an alert. *Mitigates: customer-facing email reliability.*
- **NFR-013 — Service-topology mitigation:** The two knowledge-graph lookups disagree on the real service map (see Section 1 reconciliation note). This risk is mitigated only when an engineer who knows the K4A/SAM topology confirms the actual repositories before implementation begins. No build work should proceed against an unconfirmed service map. *Mitigates: multi-service blast radius / unclear technical design.*
- **NFR-014 — Deadline mitigation:** The hard 5-week deadline (before the summer campaign launch) is mitigated only if the remaining open scope decisions in Section 1 (recipient resolution, cadence) are resolved early, since each materially affects scope. Plan entitlement is now resolved — the standard plan already includes scheduled recurring email reports (DEC-003/OQ-004) — and detail level is resolved per BR-7. The out-of-scope phasing in Section 6 protects the deadline by deferring non-critical-path content.
- **NFR-015 — Rollback:** On a decision to roll back, the feature owner (Product owner, operationally the on-call engineer during an incident) triggers the kill-switch in NFR-011. Rollback disables future generation and delivery only; already-delivered emails and already-generated on-screen summaries are not retroactively reverted. Timing target follows the 5-minute kill-switch propagation.

### Backward Compatibility

- **NFR-016 — Extend existing reporting/notification paths without breaking existing consumers:** This feature extends existing reporting, scheduling, and notification capabilities (per the engineering reference below) that may already serve other reports/customers. Any existing scheduled-report or notification behavior, and any existing consumers of the spend-aggregation data, must remain unchanged. If the work turns out to be genuinely additive with no existing consumers at risk, that must be stated explicitly rather than assumed.

---

## 6. Out of Scope & Phasing

### Phase 1 (In Scope — Critical Path)

- Weekly, scheduled generation of a spend-by-channel summary for the entitled customer.
- Standard-plan entitlement gating (server-side).
- Email delivery of the summary to the marketing-team recipients, with bounded retry and failure handling.
- A link in the delivered email to the existing K4A dashboard where the spend-by-channel data is viewable (no new K4A screen or UI build).
- Missing/incomplete channel-data handling and marking.

### Out of Scope / Future Phases

| Item | Rationale | Phase |
|------|-----------|-------|
| Additional content dimensions: impressions, conversions, budget comparison | Not required for the core spend-by-channel objective; explicitly excluded per BR-7 (DEC-005/OQ-002). Prior-week comparison is in scope for Phase 1 only if low-effort (FR-008a) | Future |
| Filtering/segmentation of the summary (by campaign, region, product line, intra-week period) | Named as an open question, not critical path to the weekly spend-by-channel objective | Future |
| Self-service scheduling/subscription management UI (users configuring cadence or recipients themselves) | Cadence and recipients are fixed configuration in Phase 1; self-service management is net-new UX, and Phase 1 adds no new K4A screen at all (the email simply links to the existing dashboard) | Future |
| Rollout to customers beyond Meridian Travel / plans beyond standard | Requirement is scoped to one customer on the standard plan | Future |
| Configurable per-recipient or role-based access controls over who gets the data | Raised as an open question; Phase 1 delivers to the configured marketing-team recipients | Future |
| Secure time-limited download/export of the summary | Not requested; email + on-screen view satisfy the stated objective | Future |
| Staged percentage-based rollout tooling | Single-customer Phase 1 does not require staged rollout; kill-switch (NFR-011) covers incident response | Future |

---

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (grounded context, not implementation direction)

Two independent knowledge-graph lookups produced **two different service maps** with only the email-notification service in common. Both are recorded here as given fact; the real repositories must be confirmed by an engineer who knows the K4A/SAM topology (see NFR-013) before scoping.

**Lookup 1 — reporting/notifications cluster (Context Seed "Affected Services"):**

- **reporting-api** (backend) — queries and aggregates spend-by-channel data.
- **reporting-dispatcher** (backend) — schedules and delivers the weekly email summaries.
- **frontend-mos** (frontend) — displays the summary on-screen in the K4A console.
- **notifications-api** (backend) — sends the email notification to the marketing team. *(Appears in both lookups.)*
- **accounts-api** (backend) — verifies customer plan entitlement (standard-plan check).
- **k4a-mcp-server** (backend) — K4A API backend layer; needs an integration point.

**Lookup 2 — SAM-oriented cluster (independent cross-check):**

- **k4a-sam-backend** (backend, medium) — may own the campaign spend email logic.
- **sam-report-scheduler** (backend, medium) — schedules/sends periodic SAM reports like weekly emails.
- **sam-performance-reporter** (backend, medium) — generates SAM performance/spend reports by channel.
- **sam-campaign-data-sync / k4a-sam-campaign-data-sync** (backend, low) — sync campaign spend data consumed by the report; a potential upstream data-freshness dependency.
- **sam-api** (backend, low) — exposes campaign spend fields by channel.
- **notifications-api** (backend, low) — may deliver the weekly email notification. *(Appears in both lookups.)*
- **reporting-subscriptions-worker** (backend, low) — processes scheduled report email subscriptions; overlaps reporting-dispatcher's role.

**Reconciliation:** There appear to be ~6 logical roles (aggregate spend data, sync data, schedule, send email, verify entitlement, display on-screen) spread across two naming conventions. No evidence block cites a concrete symbol, endpoint, file, or table — all evidence is role-level. Collapse these to actual repositories before scoping.

## Revision Summary

_Applied reviewer's request._

- Rewrote the in-product surface value bullet to reflect that the email links to the existing K4A dashboard rather than a new on-screen view.
- Updated BR-5 so the dual surface is email + a link to the existing K4A dashboard, not a new on-screen view.
- Replaced the new on-screen K4A view subgraph in the flow diagram with the email linking to the existing K4A dashboard, keeping server-side authorization.
- Reframed FR-004 to enforce entitlement when the email's link to the existing K4A dashboard is followed.
- Replaced the on-screen-view feature area (FR-014/015/016) with requirements for an email link to the existing K4A dashboard and its server-side authorization.
- Updated the summary information requirement to reference the email link to the existing K4A dashboard instead of an on-screen view.
- Removed the FR-016 reference from the channel data-completeness state since FR-016 (on-screen missing-channel marking) no longer exists.
- Repointed the covered-week summary-labeling reference from FR-014 to FR-008 since FR-014 is no longer an on-screen display.
- Updated viewer authorization state to reference following the dashboard link and the corrected FR citations.
- Reframed NFR-001 latency from a new on-screen view to the existing K4A dashboard reached via the email link.
- Added FR-015 to the NFR-008 server-side enforcement cross-reference for the dashboard-link authorization gate.
- Updated Phase 1 scope to deliver an email link to the existing K4A dashboard rather than a new on-screen display.
- Clarified the self-service UI out-of-scope rationale to note Phase 1 adds no new K4A screen, only a link to the existing dashboard.


---
## Open Questions & Decisions Register

_Durable, ID'd reference for HITL 2 (G1) and beyond — inline `[DECISION REQUIRED: DEC-00N]` tags in the sections above point back here. Reflects the latest round only._

### Decisions Required (business trade-offs — needs a human call)

_Each carries an **Owner** (accountable for supplying the answer) and a **Provisional Value** (the working assumption engineering may build against until the decision lands), so none of these fully block engineering — implementation proceeds on the provisional value and adjusts if the confirmed decision differs._

| ID | Decision | Owner | Provisional Value (build against until confirmed) |
|----|----------|-------|-----------------------------------------------------|
| **DEC-001** | What day and time each week should the spend summary be generated and delivered? | Product owner (with Meridian Travel marketing team) | Generate and deliver every Monday morning in the customer's local time zone, since the team currently pulls this data manually every Monday. |
| **DEC-002** | Who exactly on the Meridian Travel marketing team should receive the email, and should access to the data be limited by role or permission? | Product owner (with Meridian Travel account contact) | Deliver to a customer-managed marketing distribution list configured per account, and restrict the on-screen view to users with a marketing or reporting role on the account; the account admin maintains the recipient list. |
| **DEC-004** | Which marketing channels should the spend breakdown include, and is that channel list the same every week or can it change by campaign? | Product owner (with Meridian Travel marketing team) | Include the customer's active channels, defaulting to paid search, social media, email, display, and affiliate, derived dynamically each week from the channels that have campaign activity. |
| **DEC-006** | What day and time should the weekly summary be sent, and how soon after that trigger must it arrive? | Product owner (Mayank Ukey) with Meridian Travel marketing team | Generate and deliver every Monday at 08:00 in the customer's local timezone, delivered within 2 hours of the trigger, since the manual pull it replaces is done every Monday. |
| **DEC-007** | How long must we retain the audit records of when each weekly summary was generated and delivered? | Compliance/Security with Product owner | Retain generation and delivery audit records for a minimum of 12 months, matching Kochava's standard platform audit retention, unless a Meridian Travel agreement requires longer. |

### Decisions Resolved

- **DEC-003**: Duplicate of OQ-004; same answer settles it — standard plan already includes scheduled email reports with no extra entitlement work.
- **DEC-005**: DEC-005 is a duplicate of OQ-002 and is settled by the same answer: total spend per channel plus optional prior-week comparison, no impressions or conversions.

### Open Questions (evidence gaps — needs more input)

- **OQ-006**: How should the email behave when a channel's spend data is missing or incomplete for the week, and what data latency is acceptable before send?
- **OQ-007**: Are there filtering/segmentation needs (by campaign, region, product line, or sub-period within the week), or is it one flat weekly total per channel?
- **OQ-008**: Is there an existing spend-by-channel aggregation and an existing per-customer recurring-email capability we can reuse, or should we budget for building both from scratch within the 5-week window?

### Questions Resolved

- **OQ-001**: Resolved OQ-001/DEC-004 (BR-6): the spend breakdown includes a fixed weekly set of five channels — paid search, paid social, display, email, and affiliate — the same every week, not varying by campaign.
- **OQ-002**: Each email needs total spend per channel only, plus a prior-week comparison if it's easy to add; no impressions or conversions. This answer also settles DEC-005, which asks the same question in other words.
- **OQ-003**: OQ-003 answered: the 'marketing team' recipient resolves to the marketing team distribution list, with membership controlled by the customer's marketing ops lead.
- **OQ-004**: Resolved OQ-004 (and DEC-003, which asks the same question): the standard plan tier already includes scheduled/automated recurring email reports, so enabling scheduled delivery for this customer requires no extra entitlement or feature work.
- **OQ-005**: Resolved OQ-005 — the customer sees nothing new on-screen; the weekly email includes a link to the existing K4A dashboard, and no new screen or UI build is needed in K4A.

---
> **Classification:** Type A
