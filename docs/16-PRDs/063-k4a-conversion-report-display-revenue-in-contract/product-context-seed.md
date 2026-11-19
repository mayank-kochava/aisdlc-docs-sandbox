# Context Seed — K4A conversion report: display revenue in contract currency

- **Work item:** wi_4a32d7b3dfcace339a74
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.7 — Type A trigger(s): more than 2 identified services (6). (Revenue currency swap is well-defined; affects 6 backend services and requires data-access/currency-conversion logic redesign.))
  - Requirement clarity: clear
  - UX/design need: none
  - Technical design clarity: unclear
  - Identified services: 6 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): no (Type A trigger if yes)
- **Risk tier:** HIGH (Multi-service blast radius, billing/revenue data handling, and tight end-of-month deadline create significant delivery risk.)
- **Constraints/Deadline:** um by the end of this month.
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Make the K4A conversion report display revenue in the customer's contract currency (EUR instead of USD) in the downloaded file.

## Behavior & Rules

- The conversion report currently shows revenue in USD
- Revenue should be displayed in the contract currency instead
- This applies to downloaded files from K4A

## Success Criteria

- Downloaded conversion report shows revenue in contract currency rather than USD
- Report is accurate and reflects the customer's contract terms

## Constraints & Deadline

- Deadline: end of this month
- Customer identity to be confirmed by engineering

## Scope

Product: K4A
UI/Frontend change: No (file download content only)

## Open for engineering

- Which customer is asking for this?

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Next Discovery/Design Round

- What is the current data flow for revenue values in the conversion report — is the currency conversion happening at query time, presentation time, or stored in multiple formats?
- How are contract currencies currently stored and retrieved in the system — is there a reliable contract currency field linked to each customer/subscription record?
- Should the report header, column labels, or any other metadata also reflect the contract currency, or only the revenue values themselves?
- If a customer has multiple contracts in different currencies, how should the report handle or filter revenue data — single currency per report, or mixed currencies?
- Are there any downstream systems or integrations (accounting, billing, analytics) that depend on the revenue field being in USD specifically?
- What is the expected volume of customers who will need this feature, and are there other non-USD currencies we should plan for beyond EUR?

## Intake Q&A

- **Q:** Which customer is asking for this?

  **A:** Unknown, for engineering to confirm _(answered by Mayank Ukey)_

- **Q:** Where will the customer see this change: on screen in K4A, in a file they download, or nowhere? I'll assume the downloaded file only.

  **A:** yes, in downloaded file only. _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): K4A conversion report generation and data retrieval
- cost-v2 (backend): Revenue calculation and currency conversion logic
- cost-revenue-workers (backend): Revenue processing and aggregation for reports
- k4a-mcp-server (backend): K4A data export and downloaded file generation
- accounts-api (backend): Customer contract currency configuration retrieval
- champaign-api (backend): Campaign and customer metadata for currency context

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: K4A conversion report generation and data retrieval; kind: backend; confidence: high
- cost-v2 — role: Revenue calculation and currency conversion logic; kind: backend; confidence: high
- cost-revenue-workers — role: Revenue processing and aggregation for reports; kind: backend; confidence: high
- k4a-mcp-server — role: K4A data export and downloaded file generation; kind: backend; confidence: medium
- accounts-api — role: Customer contract currency configuration retrieval; kind: backend; confidence: medium
- champaign-api — role: Campaign and customer metadata for currency context; kind: backend; confidence: low
