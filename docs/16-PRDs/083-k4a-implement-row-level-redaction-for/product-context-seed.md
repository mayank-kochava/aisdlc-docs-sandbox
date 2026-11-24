# Context Seed — K4A: Implement row-level redaction for eventGrainEligibility

- **Work item:** wi_c5b1d0bc8ab4d4cd146b
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.95 — Type A trigger(s): requirements unclear, needs discovery; significant new UX/UI design; more than 2 identified services (5). (Redaction rules, current data visibility, and UI impact are all explicitly deferred to design phase; affects 5 services across backend and frontend.))
  - Requirement clarity: unclear
  - UX/design need: significant
  - Technical design clarity: unclear
  - Identified services: 5 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** HIGH (Data visibility/redaction carries compliance and PII sensitivity; multi-service blast radius with frontend involvement amplifies scope and integration risk.)
- **Constraints/Deadline:** yes, correct this is k4a request.
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Implement row-level redaction for the Amazon Ads eventGrainEligibility data in K4A.

## Behavior & Rules

Row-level redaction logic to be determined during detailed design, pending confirmation of current data visibility and access patterns.

## Success Criteria

- eventGrainEligibility rows are redacted according to eligibility rules (rules to be defined)
- Redaction is applied consistently at the row level in K4A

## Constraints & Deadline

None stated.

## Scope

**Product:** K4A

**UI/Frontend change needed:** Unclear — to be confirmed during detailed design.

## Open for engineering

- What does the customer see today when they access eventGrainEligibility data in K4A?

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Discovery/Design

- What specific eligibility criteria should determine whether a row is redacted (e.g., user permissions, advertiser status, data freshness, campaign state)?
- How should redacted rows be presented to users — hidden completely, shown with placeholder text, or indicated as "redacted" with an explanation?
- Are there different redaction rules for different user roles or advertiser types, or is one consistent rule applied to all?
- How often does eligibility status change, and do redaction rules need to be re-evaluated in real-time or can they be computed periodically?
- What happens to downstream systems, reports, or dashboards that consume eventGrainEligibility data — do they also need redaction logic, or only K4A?
- Should users be able to request access to redacted rows, and if so, what approval or escalation workflow is required?

## Intake Q&A

- **Q:** What does the customer see today when they access eventGrainEligibility data in K4A?

  **A:** Unknown, for engineering to confirm _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- control (backend): Owns Amazon Ads data ingestion and field definitions
- sanitization-api (backend): Implements row-level redaction/masking logic
- analytics-query (backend): Serves eventGrainEligibility queries to K4A consumers
- reporting-api (backend): Exposes eventGrainEligibility in customer-facing reports
- frontend-mos (frontend): K4A UI: displays eventGrainEligibility data to customers

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- control — role: Owns Amazon Ads data ingestion and field definitions; kind: backend; confidence: high
- sanitization-api — role: Implements row-level redaction/masking logic; kind: backend; confidence: high
- analytics-query — role: Serves eventGrainEligibility queries to K4A consumers; kind: backend; confidence: high; endpoints: DELETE /analytics/accounts/{accountID}/filter-sets/{filterID}, POST /analytics/accounts/{accountID}/filter-sets, PUT /analytics/accounts/{accountID}/apps/{appID}/filter-sets/{filterID}, PUT /analytics/accounts/{accountID}/filter-sets
- reporting-api — role: Exposes eventGrainEligibility in customer-facing reports; kind: backend; confidence: high
- frontend-mos — role: K4A UI: displays eventGrainEligibility data to customers; kind: frontend; confidence: medium
