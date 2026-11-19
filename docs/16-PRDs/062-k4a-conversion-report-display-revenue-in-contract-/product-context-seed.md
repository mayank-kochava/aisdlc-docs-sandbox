# Context Seed — K4A conversion report: display revenue in contract currency

- **Work item:** wi_e817485fe6d2622be1bb
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.85 — Type A trigger(s): requirements unclear, needs discovery; more than 2 identified services (5). (Account scope and currency logic are ambiguous (one account vs. all EU accounts); multi-service backend changes (5 services) require clear data-flow and schema design before dev.))
  - Requirement clarity: unclear
  - UX/design need: none
  - Technical design clarity: unclear
  - Identified services: 5 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): no (Type A trigger if yes)
- **Risk tier:** HIGH (Hard external deadline (end of month renewal), affects billing/revenue data across 5 services, and customer visibility into financial figures.)
- **Constraints/Deadline:** end of next month, they renew then
- **Gate outcome:** pending review
- **Next-round owner:** (unassigned)

## Requirement Summary

## Requirement Summary

## Objective

Show revenue in the contract currency (EUR) instead of USD on the K4A conversion report for Lumen Retail's main account in Munich. The change applies to EU customers only.

## Behavior & Rules

- Identify Lumen Retail's account(s) in the EU region
- Retrieve the contract currency for each account (EUR for Lumen Retail)
- Display revenue amounts in the contract currency on the conversion report instead of defaulting to USD
- Apply this logic to all EU customers, not just Lumen Retail

## Success Criteria

- Lumen Retail sees revenue displayed in EUR on their K4A conversion report
- EU customers see revenue in their respective contract currencies
- The change is reflected in downloaded reports/files

## Constraints & Deadline

- Must be complete by end of next month (customer renewal date)

## Scope

- **Product:** K4A
- **UI/Frontend change:** No

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Discovery/Design

- How should the system determine a customer's "contract currency" — is it stored in a single field, derived from their billing address region, or pulled from an external contract management system?
- What happens when an EU customer has multiple accounts or contracts with different currencies — should each account show its own contract currency, or should there be a primary currency setting?
- Are there existing exchange rate mechanisms in place, or do we need to build currency conversion logic (and if so, which rates and refresh frequency)?
- Should the USD amount be hidden entirely, shown alongside EUR, or configurable by the user/admin?
- How will this affect reporting consistency for multi-currency organizations (e.g., parent companies with subsidiary accounts across different EU countries)?
- Do downloaded reports need to include both original and converted currencies, metadata about the conversion, or just the contract currency amount?

## Intake Q&A

- **Q:** Which customer is this, and how many of their accounts or clients see revenue in USD when it should be in EUR?

  **A:** its Lumen Retail, just their one main account i think, they are in Munich _(answered by Mayank Ukey)_

- **Q:** Will Lumen Retail see this change on screen in K4A, in a file or report they download, or does nothing change for them? I'll assume the downloaded file only.

  **A:** yes, but only for EU customers _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): K4A conversion report generation and currency handling
- cost-v2 (backend): Revenue calculation and currency conversion logic
- cost-revenue-workers (backend): Revenue data aggregation and currency transformation
- accounts-api (backend): Account contract currency metadata and customer region configuration
- frontend-mos (frontend): K4A UI currency display and report download

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: K4A conversion report generation and currency handling; kind: backend; confidence: high
- cost-v2 — role: Revenue calculation and currency conversion logic; kind: backend; confidence: high
- cost-revenue-workers — role: Revenue data aggregation and currency transformation; kind: backend; confidence: medium
- accounts-api — role: Account contract currency metadata and customer region configuration; kind: backend; confidence: medium
- frontend-mos — role: K4A UI currency display and report download; kind: frontend; confidence: low
