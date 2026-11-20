# Context Seed — K4A: reduce Android install reporting latency to same day

- **Work item:** wi_d99aab51154aa16d2d78
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.85 — Type A trigger(s): requirements unclear, needs discovery; more than 2 identified services (5). (Root cause of 24-hour lag is unidentified; affects 5 services (backend + frontend), requiring data-pipeline and schema investigation before dev can begin.))
  - Requirement clarity: unclear
  - UX/design need: minor
  - Technical design clarity: unclear
  - Identified services: 5 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** HIGH (Hard month-end deadline, multi-service blast radius, and unknown latency source create execution risk despite straightforward success criteria.)
- **Constraints/Deadline:** before the end of next month
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Reduce the latency of Android install number reporting in K4A so that daily totals appear in the dashboard within the same day, matching Orion Mobile's own reporting timeline instead of appearing approximately 24 hours late.

## Behavior & Rules

- Android install counts in K4A currently display the same totals as Orion Mobile's internal dashboard, but delayed by approximately one day
- The requirement is to bring these numbers into sync with real-time or same-day reporting

## Success Criteria

- Android install numbers in K4A dashboard appear on the same day they are recorded, eliminating the ~24-hour lag
- Totals match Orion Mobile's internal dashboard without delay

## Constraints & Deadline

- Deadline: end of next month

## Scope

- Product: K4A
- Platform: Android install reporting only
- UI/Frontend change needed: unclear — requires clarification (dashboard display may need updates, but this was not confirmed)

## Engineering Analysis

(To be completed during design review)

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Discovery/Design

- Where exactly in the data pipeline does the ~24-hour delay occur—is it in data collection, aggregation, transmission from Android sources, K4A processing, or dashboard rendering?
- What is Orion Mobile's internal dashboard pulling data from that allows it to report same-day, and can K4A tap into that same source or data stream directly?
- Are there technical or contractual constraints with Android data providers (Play Store, Firebase, or other SDKs) that currently prevent same-day reporting, and if so, can they be worked around?
- What volume and frequency of install events does the system need to handle, and does the current architecture support real-time or hourly aggregation instead of daily batches?
- Should the dashboard display truly real-time numbers that update throughout the day, or is end-of-day same-day reporting (e.g., updated by midnight) sufficient to meet business needs?
- Are there data validation, deduplication, or reconciliation steps in the current pipeline that intentionally wait 24 hours for completeness, and can they be restructured to run sooner without sacrificing accuracy?

## Intake Q&A

- **Q:** When you say the numbers show up a day late, do you mean Orion Mobile sees the same totals but delayed by about 24 hours, or are the numbers themselves different between the two dashboards?

  **A:** same totals i think, just late, like a day behind _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- k4a-sam-backend (unknown): Syncs Android install data to K4A dashboard; latency issue likely here
- reporting-dispatcher (backend): Processes and dispatches reporting data for dashboard display
- cost-revenue-workers (backend): Workers that may delay or batch data aggregation before display
- frontend-mos (frontend): K4A dashboard UI; needs to verify data freshness display and caching
- event-manager-api (backend): Event ingestion pipeline; validates timing of Android install events

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- k4a-sam-backend — role: Syncs Android install data to K4A dashboard; latency issue likely here; kind: unknown; confidence: high
- reporting-dispatcher — role: Processes and dispatches reporting data for dashboard display; kind: backend; confidence: medium
- cost-revenue-workers — role: Workers that may delay or batch data aggregation before display; kind: backend; confidence: medium
- frontend-mos — role: K4A dashboard UI; needs to verify data freshness display and caching; kind: frontend; confidence: medium
- event-manager-api — role: Event ingestion pipeline; validates timing of Android install events; kind: backend; confidence: medium; endpoints: DELETE /event/delete, DELETE /event/{id}, GET /event/{id}, GET /events/list, PATCH /event/{id}
