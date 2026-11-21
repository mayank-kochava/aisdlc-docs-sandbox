# Context Seed — K4A: add weekly campaign spend by channel email

- **Work item:** wi_0387f98c8dc122fe67b4
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.95 — Type A trigger(s): requirements unclear, needs discovery; significant new UX/UI design; more than 2 identified services (5); confirmed UI/frontend change required. (Recipient list unconfirmed and email timing undefined. Blast radius spans 5 services (backend reporting, frontend…)
  - Requirement clarity: unclear
  - UX/design need: significant
  - Technical design clarity: unclear
  - Identified services: 5 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): yes (Type A trigger if yes)
- **Risk tier:** HIGH (Hard December deadline, multi-service blast radius, and customer-facing K4A console UI changes create tight schedule with limited course-correction room.)
- **Constraints/Deadline:** before December end this year.
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Send Meridian Travel's marketing team a weekly email summary of campaign spend by channel, eliminating manual Monday pulls.

## Behavior & Rules

- Email delivery: weekly
- Content: campaign spend aggregated by channel
- Recipient list: to be confirmed by engineering with Meridian Travel
- Timing: to be determined (day/time of week)

## Success Criteria

- Marketing team receives automated weekly email summary
- Summary displays campaign spend broken down by channel
- Manual data pulls by marketing team are no longer necessary

## Constraints & Deadline

- Delivery: before December end this year

## Scope

**Product:** K4A

**UI/Frontend change needed:** Yes (on K4A console)

## Open for engineering

- Who at Meridian Travel's marketing team should receive this email?

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Next Round

- What specific channels should the spend breakdown include (e.g., Google Ads, Facebook, email, display, organic), and will this list change over time?
- Should the email summary include only current week spend, or do marketers need historical comparison data (prior week, month-over-month, year-over-year)?
- What level of detail should each channel row show—just total spend, or also metrics like impressions, clicks, conversions, or cost-per-action?
- How should the email handle campaigns that span multiple channels or don't map cleanly to a single channel category?
- What timezone and exact day/time would work best for delivery, and are there blackout periods (holidays, company shutdowns) to avoid?
- Should the email include any drill-down links or calls-to-action that direct marketers back to K4A console for deeper analysis, or is a static summary sufficient?

## Intake Q&A

- **Q:** Who at Meridian Travel's marketing team should receive this email?

  **A:** Unknown, for engineering to confirm _(answered by Mayank Ukey)_

- **Q:** Where will the customer see this change?

A. On screen in K4A
B. In a file they download
C. Nowhere
Pick a letter, or answer in your own words.
  **A:** on k4a console _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): Query & aggregate campaign spend by channel
- reporting-dispatcher (backend): Execute scheduled weekly email dispatch
- frontend-mos (frontend): K4A console UI displaying email config/subscription
- k4a-mcp-server (backend): K4A API backend for email subscription management
- sam-api (backend): SAM campaign spend data source for weekly reports

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: Query & aggregate campaign spend by channel; kind: backend; confidence: high
- reporting-dispatcher — role: Execute scheduled weekly email dispatch; kind: backend; confidence: high
- frontend-mos — role: K4A console UI displaying email config/subscription; kind: frontend; confidence: medium
- k4a-mcp-server — role: K4A API backend for email subscription management; kind: backend; confidence: high
- sam-api — role: SAM campaign spend data source for weekly reports; kind: backend; confidence: medium
