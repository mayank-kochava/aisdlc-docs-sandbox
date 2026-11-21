---
id: product-spec-r2
title: Product Specification
---

# Product Specification — K4A Weekly Attribution Report PDF Download

## 1. Business Logic & Context

### Problem

Brightway Retail's finance team has no self-service way to obtain attribution data in a portable, review-ready format. Attribution data lives inside the K4A platform, but finance needs a periodic, fixed-format snapshot they can file, circulate, and reconcile against their books — specifically ahead of year-end close. Today that export does not exist, so finance cannot complete its year-end attribution review without manual intervention.

### Value

Delivering a weekly attribution report as a downloadable PDF gives the finance team a repeatable, standalone artifact they can retrieve on their own schedule. The near-term driver is a hard year-end-close deadline (~5 weeks from request); the durable value is an ongoing weekly finance-grade reporting capability for K4A customers.

### Business Rules

- **Cadence:** The attribution report covers a weekly period and is regenerated on a weekly basis.
- **Format:** The report is delivered as a PDF.
- **Audience:** The finance team of the subscribing customer account (initial customer: Brightway Retail) is the intended consumer.
- **Plan tier:** Brightway Retail is on the **standard plan**. Whether the weekly PDF attribution report is included in the standard plan or is a gated/entitled capability is not settled by the input. [DECISION REQUIRED: DEC-001]
- **Delivery mechanism:** The report is delivered exclusively via on-demand PDF download in the K4A console UI. Automatic email push to finance inboxes is not required and is out of scope.
- **Report content:** The specific data fields and metrics the report must contain — and whether that content differs by finance team member or department — are not settled by the input. [DECISION REQUIRED: DEC-003]
- **Historical window:** Whether each weekly report contains only the most recent week of attribution data or a rolling window of prior weeks is not settled by the input. [DECISION REQUIRED: DEC-004]
- **Data sensitivity:** The report carries finance/billing-sensitive attribution data; access must be restricted to authorized users of the owning account and enforced server-side.

### Known Scope / Blast-Radius Note

This change spans multiple backend services and the K4A frontend console. The automated codebase analysis produced **two non-identical service lists**: the primary analysis named six services (including a K4A entitlement/plan-check service), while an independent cross-check did **not** re-find that entitlement service and instead surfaced two additional backend query/summary services at low confidence. The blast radius is therefore **not fully settled** and must be confirmed during engineering discovery. (The full service-level grounding evidence is attached as a labelled appendix at the end of this document.) Given this uncertainty combined with HIGH risk and a hard 5-week deadline, explicit discovery time should be budgeted before build estimation.

---

## 2. User Flows (Visuals)

```mermaid
flowchart TD
    A[Finance user signs in to K4A console] --> B{Authorized finance user<br/>on an entitled account?}
    B -- No --> C[Access denied: report not available<br/>server-side rejection, not UI-hiding only]
    B -- Yes --> D[Open Weekly Attribution Report view]
    D --> E{A generated weekly report<br/>is available?}
    E -- No report yet --> F[Show empty/pending state:<br/>no report available for this week]
    E -- Yes --> G[Display available weekly report with period label]
    G --> H[User requests PDF download]
    H --> I{Report artifact retrievable<br/>from backend?}
    I -- Success --> J[PDF delivered to user<br/>download access recorded in audit log]
    I -- Dependency/generation failure --> K[Show specific failure message<br/>timeout vs unavailable vs not-ready<br/>failure recorded with distinct status]
    K --> L[User may retry download]
    L --> I

    subgraph Weekly generation (system-triggered)
    M[Weekly schedule fires] --> N{Report already generated<br/>for this period?}
    N -- Yes --> O[Idempotent: no duplicate,<br/>overwrite/upsert existing period record]
    N -- No --> P[Generate weekly report from attribution data]
    P --> Q{Attribution data source reachable?}
    Q -- Yes --> R[Produce PDF artifact for the weekly period]
    Q -- No --> S[Fail with recorded status + operational alert;<br/>report marked not-ready for the period]
    end

    R --> E
```

---

## 3. Functional Requirements

> Grouping: **Report Generation & Scheduling**, **Report Access & Download**, **Access Control & Entitlement**, **Resilience, Idempotency & Observability**, **Operational Safety (Kill-Switch & Audit)**.
>
> Several requirements below reference product decisions that are not settled by the input (report content, historical window, delivery mechanism, plan entitlement). Where a requirement depends on one of those, it is written against a stated working default so an implementation agent can proceed, and the dependency is flagged. Each such decision is tagged once at its first occurrence in Section 1 and referenced — not re-tagged — here.

### Report Generation & Scheduling

**FR-001 — Weekly report generation on schedule**
```gherkin
Given an account is enrolled to receive the weekly attribution report
When the weekly generation schedule fires for that account's reporting period
Then the system generates an attribution report covering that weekly period
And the generated report is produced in PDF format
And the report is associated with the specific weekly period it covers
```

**FR-002 — Report covers the defined weekly period**
```gherkin
Given a weekly report generation run for a given period
When the report content is assembled
Then the report includes attribution data for that weekly period
And the report is labelled with the weekly period boundaries it represents
```
> Working default: each report contains the most recent completed week of attribution data only (no rolling multi-week history). Depends on the historical-window decision flagged in Section 1.

**FR-003 — Report content sourced from attribution data**
```gherkin
Given the attribution data source for an account is available
When a weekly report is generated
Then the report content is derived from that account's attribution data for the period
And no attribution data from another account is included in the report
```
> Working default: the report contains the standard attribution metrics already exposed for the account for the period. The exact field/metric set depends on the report-content decision flagged in Section 1.

**FR-004 — Weekly schedule does not run concurrently for the same period**
```gherkin
Given a weekly generation run for an account period is already in progress
When the schedule would otherwise trigger another run for the same account and period
Then the second run does not start concurrently
And only one generation run executes for a given account and period at a time
```

### Report Access & Download

**FR-005 — Finance user downloads the available weekly report**
```gherkin
Given an authorized finance user is signed in to the K4A console on an entitled account
And a generated weekly report is available for that account
When the user requests to download the report
Then the system delivers the report as a PDF
And the download is scoped to that user's own account data only
```
> Delivery is via on-demand UI download in the K4A console only; automatic email push is not required (delivery-mechanism decision resolved in Section 1).

**FR-006 — Sensitive report delivered via time-limited secure link**
```gherkin
Given a finance user has requested a weekly attribution report download
When the system provides access to the generated PDF artifact
Then the artifact is served via a time-limited secure link rather than an unauthenticated raw file location
And the link is no longer usable after its validity window expires
```

**FR-007 — No report available state**
```gherkin
Given an authorized finance user opens the weekly attribution report view
And no weekly report has yet been generated for the current period
When the view loads
Then the system presents an explicit "no report available for this period" state
And no partial, stale, or placeholder PDF is offered for download
```

**FR-008 — Report period is identifiable to the user**
```gherkin
Given one or more generated weekly reports are available to a finance user
When the user views the available report(s)
Then each available report displays the weekly period it covers
So the user can confirm which week they are downloading
```

### Access Control & Entitlement

**FR-009 — Server-side authorization on every download**
```gherkin
Given a request to download a weekly attribution report
When the request reaches the backend
Then the system re-derives the requester's authorization from the authoritative source on that request
And a request from a user not authorized for the owning account is rejected with a forbidden response
And authorization is never satisfied by UI-hiding or a client-supplied claim alone
```

**FR-010 — Fail closed when authorization cannot be determined**
```gherkin
Given a request to download or view a weekly attribution report
When the authorization/entitlement source cannot be reached or evaluated
Then the system denies access rather than allowing it
And the denial is surfaced to the user as an access error, not a generic failure
```

**FR-011 — Plan-tier entitlement enforcement**
```gherkin
Given an account requests access to the weekly attribution report feature
When the system evaluates the account's plan entitlement for this feature
Then access is granted only if the account's plan includes the weekly attribution report capability
And an account whose plan does not include the capability is rejected server-side with a forbidden response
```
> Working default: treat the weekly attribution report as included for the standard plan so Brightway Retail (standard plan) receives it within the deadline. Whether the capability is actually standard-plan-included or gated depends on the plan-entitlement decision flagged in Section 1; if it is gated, this requirement blocks delivery to Brightway until entitlement is granted.

### Resilience, Idempotency & Observability

**FR-012 — Idempotent weekly generation per period**
```gherkin
Given a weekly report for a given account and period has already been generated
When generation runs again for the same account and the same period (e.g. a retry or re-delivered trigger)
Then the system overwrites/updates the existing report record for that period
And no duplicate report record is created for the same account and period
And no side effect is double-counted
```

**FR-013 — Bounded retry on attribution data source failure**
```gherkin
Given a weekly report generation run needs attribution data
When the attribution data source is temporarily unavailable
Then the system retries a bounded number of times with exponential backoff
And does not retry indefinitely
And if a rate-limit response with a Retry-After signal is returned, the system honors that interval rather than retrying on a fixed schedule
```
> Working default: 3 retry attempts with exponential backoff, matching the established platform convention.

**FR-014 — Distinct, user-facing failure conditions on download**
```gherkin
Given a finance user requests a weekly report download
When the request fails
Then the system presents a message specific to the failure condition
And a timeout, a dependency-unavailable condition, and a report-not-yet-ready condition each produce their own distinct message
And each failure condition is recorded with its own distinct status rather than collapsed into one generic error
```

**FR-015 — Generation failure raises an operational alert**
```gherkin
Given a scheduled weekly report generation run fails after exhausting retries
When the run terminates unsuccessfully
Then the report is marked not-ready for that period
And an operational alert is raised for the failure
And the failure never passes silently
```

**FR-016 — No sensitive data in logs**
```gherkin
Given any step of report generation, delivery, or access logs diagnostic information
When a log record is written
Then raw finance/billing-sensitive attribution data is not written to logs
And no credentials, secrets, or tokens are written to logs
```

### Operational Safety (Kill-Switch & Audit)

**FR-017 — Config-based kill-switch for the feature**
```gherkin
Given the weekly attribution report feature is live
When an on-call engineer needs to disable it during an incident
Then the feature can be disabled via configuration without a code deploy
And disabling it requires no separate approval or escalation step
And new requests reflect the disabled state within an established propagation window
```
> Working default: 5-minute propagation for new requests; in-flight generation runs and downloads submitted before the flip complete under the prior behavior. (Engineering default — operational mechanism.)

**FR-018 — Audit record on report download/access**
```gherkin
Given an authorized finance user successfully downloads a weekly attribution report
When the download completes
Then an audit record is written capturing actor identity (user ID and account ID), the report resource identity (account and weekly period), the action, a timestamp, and the outcome
And the audit record is written asynchronously so it does not block the download
And a failure to write the audit record raises an immediate operational alert rather than failing silently
```

**FR-019 — Audit records are immutable**
```gherkin
Given an audit record has been written for a report access or download
When any role or code path attempts to edit or delete it
Then the attempt is not permitted
And the audit record remains append-only and unchanged
```
> Working default: minimum audit retention of 12 months, within the established platform range; a longer period applies only if a specific finance/compliance agreement requires it. (Engineering/compliance default.)

Now reporting the business trade-offs I flagged in Sections 0–3.

## 4. Information Requirements

The system must track the following business information to support the functional requirements above:

- **Weekly Attribution Report (artifact):** The generated PDF for a given account and weekly period. System-derived (produced by the generation run from attribution data). This is the core deliverable the finance team retrieves.
- **Reporting Period:** The weekly period boundaries each report covers (start/end of the covered week). System-derived from the generation schedule. Needed so a user can confirm which week they are downloading (FR-002, FR-008) and so generation stays idempotent per period (FR-012).
- **Report Generation Status:** The outcome of each weekly generation run for an account and period — e.g. not-yet-run, in-progress, ready, not-ready/failed. System-derived. Drives the "no report available" state (FR-007), the distinct failure messaging (FR-014), and the not-ready marking on failure (FR-015).
- **Attribution Data (source content):** The account's attribution metrics for the period, consumed from the upstream attribution data source. External/upstream to this feature. This is what the report content is derived from (FR-003); the exact field/metric set depends on the report-content decision flagged in Section 1.
- **Account & Plan Entitlement:** The owning account's identity and whether its plan includes the weekly attribution report capability. System-derived / sourced from the authoritative entitlement source. Drives plan-tier enforcement (FR-011) and account-scoping of report access (FR-003, FR-005).
- **Requesting User Identity & Authorization:** The signed-in user, their account association, and their finance-team authorization. Sourced from the authoritative auth source on each request. Drives server-side authorization (FR-009, FR-010).
- **Report Enrollment / Subscription:** Which accounts are enrolled to receive the weekly report. System-derived / configured. Determines which accounts the weekly schedule generates for (FR-001).
- **Secure Download Link:** The time-limited access reference by which a user retrieves a generated PDF. System-derived at download time. Supports secure, expiring delivery of sensitive artifacts (FR-006).
- **Audit Record:** The record of each report access/download — actor identity, resource identity (account + period), action, timestamp, outcome. System-derived. Supports the audit requirements (FR-018, FR-019).
- **Feature Kill-Switch State:** The configuration flag enabling/disabling the feature. Configured/operational. Supports incident response (FR-017).

---

## 5. Non-Functional Requirements & Guardrails

### Performance & Capacity

- **NFR-001 — Download responsiveness:** When a generated report is already available, the PDF download request should respond within a p95 latency target measured under expected concurrent finance-user load (p95 ≤ 2s for the download-initiation response, measured across representative concurrent download requests). Engineering may refine the exact threshold against real artifact sizes.
- **NFR-002 — Read-path rate limiting:** The report view/download (read) path must enforce its own rate limit, separate from any backend generation limits, to protect the shared artifact-serving and attribution-query resources from excessive on-demand requests.
- **NFR-003 — Shared attribution source protection:** Because report generation reads from a shared attribution data source consumed by other systems, generation must operate within an explicit rate cap against that source so added reporting load does not degrade existing consumers.
- **NFR-004 — Generation throughput for cadence:** The weekly generation path must complete all enrolled accounts' reports within the weekly window with margin, so that a report is reliably available before each covered week's consumers need it.

### Availability & Monitoring

- **NFR-005 — Generation failure alerting:** Raise an operational alert when the weekly generation error rate exceeds 1% for 5 minutes, sustained, or when any scheduled run fails after exhausting retries (reinforces FR-015).
- **NFR-006 — Resource utilization alerting:** Raise an operational alert when artifact-serving or generation resource utilization exceeds 80% for 5 minutes, sustained.
- **NFR-007 — Audit-write failure alerting:** Any failure to persist an audit record raises an immediate operational alert rather than failing silently (reinforces FR-018).

### Security & Access Control

- **NFR-008 — Server-side enforcement:** All authorization and plan-entitlement checks are enforced server-side and re-derived from the authoritative source on every request; a direct API call from an ineligible role or account is rejected (403) regardless of UI state (reinforces FR-009, FR-011).
- **NFR-009 — Fail-closed access posture:** Access/entitlement checks fail closed — if the auth or entitlement source is unreachable, access is denied, never granted by default (reinforces FR-010). This is the appropriate posture because the report carries finance/billing-sensitive data where a false allow is worse than a false deny.
- **NFR-010 — Account data isolation:** A report must never contain or expose another account's attribution data; account scoping is enforced on both generation and retrieval.
- **NFR-011 — Sensitive-data handling in logs:** Raw finance/billing-sensitive attribution data, credentials, secrets, and tokens are never written to logs (reinforces FR-016).
- **NFR-012 — Secure, expiring delivery:** The generated PDF is delivered via a time-limited secure link, never an unauthenticated raw file location or a persistent public URL (reinforces FR-006).
- **NFR-013 — Audit immutability & retention:** Audit records are append-only, cannot be edited or deleted by any role or code path, and are retained for a minimum of 12 months within the established platform range; a longer retention period applies only if a specific finance/compliance agreement requires it (reinforces FR-019).

### Resilience & Idempotency

- **NFR-014 — Bounded downstream retry:** Calls to the attribution data source use bounded retry with exponential backoff (3 attempts, the established platform default), never indefinite retry, and honor a 429 `Retry-After` signal rather than a fixed schedule (reinforces FR-013).
- **NFR-015 — Idempotent generation:** Weekly generation is idempotent on the account + weekly-period key; a retried or re-delivered trigger overwrites/upserts the existing period record and never creates a duplicate or double-counts a side effect (reinforces FR-012). As a complementary safeguard, the scheduler must prevent overlapping concurrent runs for the same account and period (reinforces FR-004).
- **NFR-016 — Distinct failure classification:** Timeout, dependency-unavailable, and report-not-ready conditions are tracked as distinct statuses with distinct user-facing messages, never collapsed into one generic error (reinforces FR-014).

### Risk Mitigation

The Context Seed classifies this work as **HIGH risk** (hard year-end deadline, multi-service blast radius, finance/billing data sensitivity). Each risk and its mitigation condition:

- **NFR-017 — Hard year-end deadline (~5 weeks):** The feature must be delivered and a report available for Brightway Retail's finance team before year-end close. Mitigated only when the plan-entitlement question (flagged in Section 1) is resolved early enough that it does not block Brightway's access, and when discovery time to resolve the service-boundary ambiguity (below) is budgeted explicitly rather than assumed. The MVP scope in Section 6 is deliberately minimal to protect this deadline.
- **NFR-018 — Multi-service blast radius / unsettled scope:** The codebase analysis produced two non-identical service lists (see Section 1 and the Engineering Reference appendix), so the blast radius is not confirmed. Mitigated when engineering discovery confirms the affected services and resolves the scheduling vs. PDF-generation ownership overlap before build estimation. A config-based kill-switch (FR-017) must exist so the feature can be disabled quickly if a multi-service interaction misbehaves in production.
- **NFR-019 — Finance/billing data sensitivity:** The report exposes sensitive attribution data. Mitigated when server-side authorization (NFR-008), fail-closed posture (NFR-009), account isolation (NFR-010), secure expiring delivery (NFR-012), no-sensitive-data-in-logs (NFR-011), and immutable audit logging (NFR-013) are all in place and verified.

### Rollback

- **NFR-020 — Rollback:** Because this is HIGH-risk, multi-service work, rollback is handled via the config-based kill-switch (FR-017). **Trigger:** report delivery of incorrect/corrupted data, an access-control failure, or sustained generation failures. **Authority:** on-call engineer, no separate escalation required. **Timing:** new requests reflect the disabled state within the established propagation window (5-minute default per FR-017). **Data handling:** disabling is forward-only — already-generated reports are not retroactively withdrawn; only new generation and download requests are affected going forward.

### Backward Compatibility

- **NFR-021 — Backward compatibility:** This feature extends existing reporting and attribution services that have existing consumers (notably the shared attribution data source). Existing report types, existing attribution-data read patterns, and existing consumers of those services must remain unchanged by this work. New reporting load must not alter behavior or degrade performance for existing consumers of the shared attribution source (relates to NFR-003).

---

## 6. Out of Scope & Phasing

### In Scope (Phase 1 — MVP)

- Weekly generation of an attribution report in PDF format for enrolled accounts (initial: Brightway Retail).
- On-demand download of the available weekly report through the K4A console.
- Server-side authorization and plan-entitlement enforcement.
- Secure, time-limited delivery; immutable audit logging of downloads.
- Config-based kill-switch, bounded retry, idempotent generation, and operational alerting.

### Out of Scope (Future Phases)

| Item | Rationale / Notes | Phase |
|------|-------------------|-------|
| Email/automatic push delivery of the report to finance inboxes | Delivery is on-demand UI download only (decided in Section 1); email push is explicitly not required. If email is ever requested, it would be a follow-on. | Future |
| Rolling multi-week / historical window in each report | Historical-window is unsettled (flagged in Section 1); MVP defaults to most-recent-completed-week only. | Future |
| Per-member / per-department report content variation | Content variation is unsettled (flagged in Section 1); MVP produces a single standard report per account. | Future |
| In-console report preview (view before download) | Preview was surfaced as a possible UI capability in the analysis but is not required for the download objective. | Future |
| Delivery-status display in the UI | Surfaced as a possible UI capability but not required for the core download objective. | Future |
| Configurable / custom report cadence (other than weekly) | The requirement is specifically weekly. | Future |
| Self-service report enrollment/subscription management UI | MVP enrollment can be configured operationally for the initial customer. | Future |
| Rollout to customers beyond the initial enrolled account(s) | MVP targets Brightway Retail's year-end need; broader rollout follows once the capability is proven. | Future |

---

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (grounded context, not implementation direction)

Primary codebase-analysis service list (6 services):

- **reporting-api** (backend): Attribution report generation and delivery API.
- **reporting-dispatcher** (backend): Scheduled weekly report dispatch and delivery mechanism.
- **reporting-subscriptions-worker** (backend): Weekly subscription scheduling and PDF report generation.
- **frontend-mos** (frontend): K4A UI — download link, report preview, delivery status display.
- **k4a-mcp-server** (backend): K4A MCP server coordination and plan entitlement check.
- **pubsuite-attribution** (backend): Attribution data source the K4A report references and displays (read dependency).

Discrepancy from the independent cross-check: the fresh lookup did **not** re-find `k4a-mcp-server`, and additionally surfaced `koch-query-api` (query layer feeding attribution report data) and `summary-report-cloud` (generates summary report content for attribution), both at low confidence. The scheduling responsibility (reporting-dispatcher vs. reporting-subscriptions-worker) and the PDF-generation responsibility (reporting-api vs. reporting-subscriptions-worker) each appear in more than one service. Engineering must confirm the true blast radius and resolve these ownership overlaps during discovery before build estimation.

## Revision Summary

_Applied reviewer's request._

- Resolved DEC-002/OQ-001 in Section 1 business rules: delivery is on-demand UI download only, no email.
- Updated FR-005 note to state on-demand UI download only, email not required, per resolved delivery decision.
- Updated out-of-scope email row rationale to reflect that on-demand UI download only is now a decision, not a default.


---
## Open Questions & Decisions Register

_Durable, ID'd reference for HITL 2 (G1) and beyond — inline `[DECISION REQUIRED: DEC-00N]` tags in the sections above point back here. Reflects the latest round only._

### Decisions Required (business trade-offs — needs a human call)

_Each carries an **Owner** (accountable for supplying the answer) and a **Provisional Value** (the working assumption engineering may build against until the decision lands), so none of these fully block engineering — implementation proceeds on the provisional value and adjusts if the confirmed decision differs._

| ID | Decision | Owner | Provisional Value (build against until confirmed) |
|----|----------|-------|-----------------------------------------------------|
| **DEC-001** | Is the weekly attribution report PDF included in the standard plan, or is it a gated premium capability that Brightway Retail would need an upgrade or special entitlement to receive? | Product owner | Treat the weekly attribution report as included in the standard plan, so Brightway Retail receives it within the year-end deadline. Revisit if the feature is meant to be a premium upsell. |
| **DEC-002** | How should the finance team receive the weekly report — as an on-demand download in the K4A console, pushed automatically to their email inboxes each week, or both? | Product owner | On-demand UI download in the K4A console for Phase 1; add automatic weekly email delivery only if explicitly required before year-end close. |
| **DEC-003** | What specific attribution metrics and data fields must the weekly report contain, and does that content need to differ by finance team member or department? | Product owner (with finance stakeholder) | Include the standard attribution metrics already exposed for the account for the weekly period, in a single uniform report for all finance users (no per-person or per-department variation) for Phase 1. |
| **DEC-004** | Should each weekly report contain only the most recent completed week of attribution data, or a rolling window covering several prior weeks? | Product owner (with finance stakeholder) | Each report contains only the most recent completed week of attribution data; add a rolling/historical window only if finance confirms they need it for year-end reconciliation. |

### Open Questions (evidence gaps — needs more input)

- **OQ-002**: What specific data fields and metrics must the weekly attribution report contain? We need this to confirm whether the existing attribution data source already exposes everything the report needs.
- **OQ-003**: Should each weekly report cover only the most recent week, or include a rolling window of historical attribution data? If historical, how many weeks?
- **OQ-004**: Is this weekly PDF report included in the standard plan (Brightway's tier), or is it a premium/entitled feature that should be gated? There's a plan-entitlement enforcement point that needs a decision here.
- **OQ-005**: For the finance team's UI, do you want just a download link, or also an in-app report preview and a delivery/generation status indicator? This directly drives the frontend scope.
- **OQ-006**: Are there requirements around report security, access controls, or audit trails for the downloaded PDFs, given this is finance/billing-sensitive data?
- **OQ-007**: Is there an existing attribution report or format the finance team uses today that this PDF should match or build on, rather than designing from scratch?
- **OQ-008**: Does finance need the report on a fixed day each week, or is weekly cadence with availability before year-end close sufficient?

### Questions Resolved

- **OQ-001**: OQ-001 resolved: the weekly attribution report is delivered as an on-demand PDF download in the K4A UI only, with no email delivery — making frontend download core to the build and removing dispatcher/email services from scope.

---
> **Classification:** Type A
