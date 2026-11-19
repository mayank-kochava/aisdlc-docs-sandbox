# Context Seed — K4A install report: show install times in account timezone

- **Work item:** wi_d1acc6089b53b4513387
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.7 — Type A trigger(s): requirements unclear, needs discovery. (Scope ambiguity (screen vs. download only) and timezone-conversion logic across 2 backend services remain unconfirmed; customer expectation is clear but delivery surface is not.))
  - Requirement clarity: unclear
  - UX/design need: minor
  - Technical design clarity: unclear
  - Identified services: 2 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** MEDIUM (Hard month-end deadline, multi-service blast radius, and timezone/data-handling complexity create moderate execution risk despite narrow UX footprint.)
- **Constraints/Deadline:** end of this month would be fine
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Requirement Summary

## Objective

Change the install time column in the K4A install report CSV to display in the account's configured timezone instead of UTC. Globex Media has requested this change and their account is set to Berlin timezone.

## Behavior & Rules

- The install time values in the CSV export should be converted from UTC to the timezone configured on the customer's account
- Globex Media's account timezone should be used as the reference for this conversion

## Success Criteria

- Install time column in the K4A install report CSV shows times in the account's configured timezone
- Globex Media can download the report and verify times match their expected Berlin timezone

## Constraints & Deadline

- Deadline: end of this month
- Scope of change (whether this affects only downloaded CSV files or also on-screen displays in K4A) needs to be confirmed by engineering

## Scope

- **Product:** K4A
- **UI/Frontend change needed:** Unclear — requires confirmation whether the change applies to downloaded files only or also to on-screen report displays

## Open for engineering

- Will Globex Media see this change on screen in K4A, in a file or report they download, or does nothing change for them? I'll assume the downloaded file only.

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Next Discovery Round

- How should the system handle accounts without a configured timezone—should it default to UTC, use the user's browser timezone, or require explicit configuration before the report can be generated?
- Beyond the K4A install report CSV, are there other reports or data exports in the product that currently display timestamps in UTC and should also be converted to account timezone for consistency?
- When a user changes their account's timezone setting, should previously downloaded CSV files remain in their original timezone, or is there an expectation to re-generate historical reports in the new timezone?
- Does Globex Media (or any customer) need the ability to toggle between UTC and account timezone when viewing or exporting reports, or is account timezone the single source of truth for all timestamp displays?
- If a multi-user account has members in different timezones, should all users see the same account-timezone timestamps, or should individual user timezone preferences override the account setting?
- Are there audit, compliance, or logging requirements that mandate timestamps remain in UTC anywhere in the system, even if customer-facing reports display in account timezone?

## Intake Q&A

- **Q:** Which customer is asking for this, and what timezone should the install time show in their CSV?

  **A:** its Globex Media, they want it in whatever time zone is set on their account, I think theyre in Berlin _(answered by Mayank Ukey)_

- **Q:** Will Globex Media see this change on screen in K4A, in a file or report they download, or does nothing change for them? I'll assume the downloaded file only.

  **A:** Unknown, for engineering to confirm _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- k4a-mcp-server (backend): K4A install report CSV generation and timezone handling
- accounts-api (backend): Account timezone configuration consumed by report generation

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- k4a-mcp-server — role: K4A install report CSV generation and timezone handling; kind: backend; confidence: high
- accounts-api — role: Account timezone configuration consumed by report generation; kind: backend; confidence: medium
