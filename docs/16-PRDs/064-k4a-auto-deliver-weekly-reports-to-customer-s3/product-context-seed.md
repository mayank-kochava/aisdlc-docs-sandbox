# Context Seed — K4A: auto-deliver weekly reports to customer S3

- **Work item:** wi_74dcfa8e2ad7397f510c
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.85 — Type A trigger(s): requirements unclear, needs discovery; more than 2 identified services (6). (S3 delivery mechanism is specified, but credential-setup process, error handling, and retry logic are undefined. Affects 6 services across reporting, credentials, and scheduling…)
  - Requirement clarity: unclear
  - UX/design need: none
  - Technical design clarity: unclear
  - Identified services: 6 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): no (Type A trigger if yes)
- **Risk tier:** HIGH (Involves PII/data delivery, multi-service integration, customer credentials management, and hard 6-week deadline with little slack.)
- **Constraints/Deadline:** (none given)
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Enable Northwind Media to receive their weekly attribution report automatically delivered to their own S3 bucket instead of downloading the CSV manually.

## Behavior & Rules

- Weekly attribution reports are pushed to the customer's S3 bucket on a scheduled basis
- The report files are zipped before delivery
- Northwind Media will need assistance setting up their S3 bucket and credentials as part of this implementation

## Success Criteria

- Northwind Media receives their weekly attribution report in their S3 bucket automatically
- Report files arrive zipped
- Customer has completed S3 setup with K4A support

## Constraints & Deadline

- Delivery needed before their quarterly review in approximately six weeks
- Northwind Media is on the enterprise plan

## Scope

Product: K4A
Feature: Weekly attribution report S3 delivery only
UI/Frontend change: No

## Engineering Analysis

KG-identified services: (pending — awaiting your confirmation of the full scope before engineering design review)

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Discovery/Design

- How will Northwind Media provide and rotate their AWS credentials securely, and what credential format and storage approach will K4A use to protect them?
- Should the system retry failed S3 deliveries automatically, and if so, what should the retry logic be (frequency, max attempts, notification to customer)?
- Will Northwind Media need visibility into delivery status (success/failure logs, timestamps) through a dashboard or email notifications, or is a silent delivery model acceptable?
- What naming convention and folder structure should reports follow in their S3 bucket, and can the customer customize this or should it be standardized?
- If Northwind Media's S3 bucket permissions or credentials become invalid, how should the system behave—continue retrying, pause silently, or alert K4A support and the customer?
- Are there any compliance or audit requirements (e.g., delivery logs, proof of receipt, data retention policies) that K4A needs to meet for this enterprise customer?

## Intake Q&A

- **Q:** Northwind Media wants their weekly attribution report pushed to their S3 bucket instead of downloading it manually. Do they have S3 credentials ready, or would they need help setting that up?

  **A:** they would need help setting it up i think, they dont have a bucket ready yet _(answered by Mayank Ukey)_

- **Q:** Where will the customer see this change?

• On screen in K4A
• In a file they download
• Nowhere
I'd go with the downloaded file only, and I'll assume that unless you say otherwise.
  **A:** Unknown, for engineering to confirm _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): Attribution report generation and delivery API
- reporting-dispatcher (backend): Scheduled report distribution and S3 export handling
- credentials-api (backend): Store and manage customer S3 bucket credentials
- frontend-mos (frontend): K4A UI: S3 credential setup form and configuration UI
- sam-report-scheduler (backend): Schedule weekly attribution report generation and delivery
- accounts-api (backend): Track customer plan tier (enterprise) and feature entitlements

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: Attribution report generation and delivery API; kind: backend; confidence: high
- reporting-dispatcher — role: Scheduled report distribution and S3 export handling; kind: backend; confidence: high
- credentials-api — role: Store and manage customer S3 bucket credentials; kind: backend; confidence: high
- frontend-mos — role: K4A UI: S3 credential setup form and configuration UI; kind: frontend; confidence: medium
- sam-report-scheduler — role: Schedule weekly attribution report generation and delivery; kind: backend; confidence: high
- accounts-api — role: Track customer plan tier (enterprise) and feature entitlements; kind: backend; confidence: medium
