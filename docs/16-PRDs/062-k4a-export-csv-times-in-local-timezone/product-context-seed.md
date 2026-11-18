# Context Seed — K4A: export CSV times in local timezone

- **Work item:** wi_df56eceacd302b79e165
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.7 — Type A trigger(s): requirements unclear, needs discovery. (Business rules conflict (EU vs. New York); timezone scope (single/configurable/per-region) and export coverage (main only vs. all types) remain unspecified. Backend logic needed but approach undefined.))
  - Requirement clarity: unclear
  - UX/design need: minor
  - Technical design clarity: unclear
  - Identified services: 1 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** MEDIUM (Hard month-end deadline with unclear scope creates rework risk; single backend service limits blast radius but timezone/data handling requires careful design upfront.)
- **Constraints/Deadline:** end of next month would be fine
- **Gate outcome:** pending review
- **Next-round owner:** (unassigned)

## Requirement Summary

## Requirement Summary

## Objective

Modify the K4A export CSV to display times in the local timezone for EU customers instead of UTC. The requester initially mentioned New York timezone but has now clarified this applies to EU customers only.

## Behavior & Rules

- Times currently exported in UTC should be converted to local timezone for EU customers
- Apply to the main/primary export CSV functionality
- Unclear whether other CSV exports or export formats should also be updated
- Timezone requirement applies to EU customers only; scope for other regions unclear

## Success Criteria

- Exported CSV times display in correct local timezone for EU customers
- EU customers confirm the timezone conversion is correct

## Constraints & Deadline

- Deadline: end of next month
- No other constraints stated

---

## Scope

- **Product:** K4A
- **Geography:** EU customers
- **UI/Frontend change needed:** Unclear — export functionality may involve UI changes, but this was not confirmed with the requester

---

**Note:** Before proceeding, please confirm:

1. Which timezone(s) should EU customers see — a single timezone (e.g., UTC+1 or UTC+2), or should it be configurable by customer location?
2. Does "main export CSV" cover all export types, or are there other exports that should remain in UTC?
3. What should non-EU customers see — UTC or their own local timezone?

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Next Round

- Which specific CSV exports does Acme Media use regularly, and do all of them need timezone conversion, or only certain report types?
- Should the timezone display be hardcoded to Eastern Time for all users, or should it be configurable by user preference or organization setting?
- Are there any downstream systems or integrations that consume these CSV exports and expect UTC times, and if so, how should we handle backward compatibility?
- Does Acme Media need the timezone offset or abbreviation (e.g., "ET" or "-05:00") displayed in the CSV alongside the converted time, or just the local time value?
- When daylight saving time transitions occur, should the system automatically handle the ET/EDT distinction, and how should historical exports be treated?
- Are there audit, compliance, or logging requirements that mandate UTC timestamps be retained somewhere in the export or system records for traceability?

## Intake Q&A

- **Q:** Which customer is asking, and what timezone should the export show? Do all their exports need this, or only certain ones?

  **A:** its Acme Media, they're in New York, i think just the main export CSV _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- k4a-mcp-server (backend): K4A export CSV generation and format

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- k4a-mcp-server — role: K4A export CSV generation and format; kind: backend; confidence: high
