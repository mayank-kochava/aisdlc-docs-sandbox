# Context Seed — K4A: add weekly campaign spend by channel summary

- **Work item:** wi_11d2fc7b87f0b78307dd
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.95 — Type A trigger(s): significant new UX/UI design; more than 2 identified services (5); confirmed UI/frontend change required. (Behavior and success criteria are well-defined; however.))
  - Requirement clarity: clear
  - UX/design need: significant
  - Technical design clarity: unclear
  - Identified services: 5 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): yes (Type A trigger if yes)
- **Risk tier:** HIGH (Hard external deadline (~5 weeks), multi-service blast radius, and new UI component introduce substantial schedule and integration risk.)
- **Constraints/Deadline:** about 5 weeks, before their summer campaign launch
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Provide Meridian Travel's marketing team with a weekly summary of campaign spend by channel displayed on screen in K4A, eliminating their manual Monday pull process.

## Behavior & Rules

- Summary aggregates campaign spend grouped by channel
- Delivery cadence: weekly
- Date format: dd-mm-yyyy
- Visible on screen within K4A interface

## Success Criteria

Marketing team can view weekly campaign spend by channel in K4A without manual extraction; ready before summer campaign launch.

## Constraints & Deadline

- Target: live within approximately 5 weeks, before summer campaign launch
- Customer plan: standard

## Scope

- **Product:** K4A
- **UI/Frontend change:** Yes

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Discovery/Design

- Which specific channels does Meridian Travel want to aggregate spend across (e.g., Google Ads, Facebook, email, social, display networks), and do any channels need sub-grouping or exclusion?
- Should the weekly summary show spend totals only, or do marketing team members also need to see performance metrics (clicks, conversions, ROI, impressions) alongside spend to act on the data?
- Who in the marketing team needs access to this summary, and are there any permission or data-sensitivity rules that should limit who can view spend by channel?
- When exactly should the weekly summary refresh and be ready to view (e.g., Monday 9am, end of business Sunday), and which timezone applies?
- How should historical weekly data be retained and accessed—does the team need to compare spend trends week-over-week or year-over-year, or only view the current week's snapshot?
- What is the current source or system where campaign spend data lives, and is there an existing API or automated feed we can connect to, or will manual data entry be required each week?

## Intake Q&A

- **Q:** What format and cadence does Meridian's marketing team need?

  **A:** dd-mm-yyyy _(answered by Mayank Ukey)_

- **Q:** Where will Meridian Travel see this change?

A. On screen in K4A
B. In a file they download
C. Nowhere
D. In an email or message
Pick a letter, or answer in your own words.
  **A:** On screen in K4A _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- frontend-mos (frontend): K4A UI: display weekly spend summary on screen
- reporting-api (backend): Query and return campaign spend by channel data
- reporting-subscriptions-worker (backend): Execute weekly scheduled summary generation and delivery
- cost-v2 (backend): Provide spend/cost data aggregated by channel
- accounts-api (backend): Verify customer plan level (standard plan check)

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- frontend-mos — role: K4A UI: display weekly spend summary on screen; kind: frontend; confidence: high
- reporting-api — role: Query and return campaign spend by channel data; kind: backend; confidence: high
- reporting-subscriptions-worker — role: Execute weekly scheduled summary generation and delivery; kind: backend; confidence: high
- cost-v2 — role: Provide spend/cost data aggregated by channel; kind: backend; confidence: high
- accounts-api — role: Verify customer plan level (standard plan check); kind: backend; confidence: medium
