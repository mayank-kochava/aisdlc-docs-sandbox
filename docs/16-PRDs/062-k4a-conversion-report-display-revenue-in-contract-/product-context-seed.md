# Context Seed — K4A conversion report: display revenue in contract currency

- **Work item:** wi_e817485fe6d2622be1bb
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.85 — Type A trigger(s): requirements unclear, needs discovery; more than 2 identified services (6). (Currency scope (which accounts/customers/regions) remains ambiguous despite Q&A; multi-service blast radius (6 services, backend+frontend) demands architectural clarity.))
  - Requirement clarity: unclear
  - UX/design need: minor
  - Technical design clarity: unclear
  - Identified services: 6 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): no (Type A trigger if yes)
- **Risk tier:** HIGH (Hard external deadline (renewal end-of-month), affects billing/revenue data, spans 6 services including frontend and data processing.)
- **Constraints/Deadline:** end of next month, they renew then
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

I'm ready to refine this requirement with you, but I don't see a question or requested edit in your message. What would you like to adjust, clarify, or confirm about the summary?

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
- cost-revenue-workers (backend): Revenue calculation and currency conversion for reports
- analytics-query (backend): Query layer for report data retrieval and formatting
- frontend-mos (frontend): K4A UI: conversion report display and currency localization
- accounts-api (backend): Account currency settings and customer contract details
- reporting-dispatcher (backend): Report generation workflow and delivery to customers

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: K4A conversion report generation and currency handling; kind: backend; confidence: high
- cost-revenue-workers — role: Revenue calculation and currency conversion for reports; kind: backend; confidence: high
- analytics-query — role: Query layer for report data retrieval and formatting; kind: backend; confidence: medium; endpoints: DELETE /analytics/accounts/{accountID}/filter-sets/{filterID}, POST /analytics/accounts/{accountID}/filter-sets, PUT /analytics/accounts/{accountID}/apps/{appID}/filter-sets/{filterID}, PUT /analytics/accounts/{accountID}/filter-sets
- frontend-mos — role: K4A UI: conversion report display and currency localization; kind: frontend; confidence: medium
- accounts-api — role: Account currency settings and customer contract details; kind: backend; confidence: medium
- reporting-dispatcher — role: Report generation workflow and delivery to customers; kind: backend; confidence: medium
