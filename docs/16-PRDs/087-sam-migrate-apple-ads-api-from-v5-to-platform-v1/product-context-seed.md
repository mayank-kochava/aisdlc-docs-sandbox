# Context Seed — SAM: migrate Apple Ads API from v5 to Platform v1

- **Work item:** wi_29223c9281447d42b93b
- **Requester:** Mayank Ukey
- **Product/channel:** SAM
- **Classification:** Type A (confidence 0.7 — Type A trigger(s): more than 2 identified services (7). (Behavior and success criteria are well-defined: migrate v5→v1 endpoints, headers, models while maintaining functionality.))
  - Requirement clarity: clear
  - UX/design need: none
  - Technical design clarity: unclear
  - Identified services: 7 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** HIGH (Hard Dec 28 deadline, multi-service blast radius, authentication/external API surface.)
- **Constraints/Deadline:** Dec 28 2026. No specific customer, this is for our own use.
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Migrate Search Ads Maven from Apple Ads Campaign Management API v5 to Platform API v1 by updating API endpoints, headers, request/response handling, and data models to align with v1, covering campaign, ad group, and keyword management.

## Behavior & Rules

- Update all API calls from v5 endpoints to v1 endpoints as documented in the Apple Ads Platform API v1 specification
- Align request/response handling with v1 requirements, including header formats and authentication
- Update data models to match v1 schema for campaigns, ad groups, and keywords
- Maintain existing functionality for campaign management, optimization, and reporting

## Success Criteria

- All campaign, ad group, and keyword operations function correctly against Platform API v1
- Campaign management, optimization, and reporting workflows continue to work without user-facing disruption
- No dependency on deprecated v5 endpoints

## Constraints & Deadline

- Deadline: December 28, 2026
- This is an internal migration; no external customers are affected
- Apple is retiring v5; migration is preventive to avoid service loss

## Scope

**Product:** SAM
**UI/Frontend change needed:** Unclear — to be confirmed

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Discovery/Design

- How will we validate that all v1 endpoint responses contain the data fields currently used by SAM's campaign optimization and reporting features, or do we need to implement fallback logic/data transformations for missing fields?
- What authentication changes are required between v5 and v1 (token format, refresh logic, permission scopes), and do we need to support both authentication methods during a transition period?
- Are there breaking changes in v1's campaign, ad group, or keyword data models (field names, enums, structure) that require UI updates, database schema changes, or user workflow adjustments?
- How should we handle API rate limits or pagination differences between v5 and v1, and do current rate limits in SAM's optimization engine need adjustment?
- What is the rollback strategy if issues are discovered post-migration, and do we need to maintain dual-API support or versioning for a grace period?
- Are there v1-specific features or capabilities (new fields, endpoints, analytics) that could improve SAM's functionality, or is this strictly a 1:1 parity migration?

## Intake Q&A

- **Q:** Search Ads Maven needs to move from Apple Ads v5 to v1 by Dec 28 2026. What breaks today if we don't migrate—are customers unable to manage campaigns, or is this a preventive update because Apple is sunsetting v5?

  **A:** Like I said, nothing's broken right now. It's preventive because Apple is retiring v5. Users are internal. Is that not enough? _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- k4a-sam-backend (unknown): Implements Apple Search Ads v5→v1 API migration
- sam-agent (unknown): Consumes migrated Apple Ads API field schemas and contracts
- sam-api (unknown): Consumes Apple Ads API responses after v5→v1 migration
- sam-campaign-data-sync (unknown): Syncs campaign/ad-group/keyword data from migrated Apple API
- sam-performance-reporter (unknown): Reports on campaign performance using migrated API data
- sam-bulk-actions (unknown): Executes bulk campaign/keyword operations via migrated Apple API
- sam-util (unknown): Utility/helper layer for Apple Ads API v5→v1 migration

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- k4a-sam-backend — role: Implements Apple Search Ads v5→v1 API migration; kind: unknown; confidence: high
- sam-agent — role: Consumes migrated Apple Ads API field schemas and contracts; kind: unknown; confidence: medium
- sam-api — role: Consumes Apple Ads API responses after v5→v1 migration; kind: unknown; confidence: medium
- sam-campaign-data-sync — role: Syncs campaign/ad-group/keyword data from migrated Apple API; kind: unknown; confidence: high
- sam-performance-reporter — role: Reports on campaign performance using migrated API data; kind: unknown; confidence: medium
- sam-bulk-actions — role: Executes bulk campaign/keyword operations via migrated Apple API; kind: unknown; confidence: medium
- sam-util — role: Utility/helper layer for Apple Ads API v5→v1 migration; kind: unknown; confidence: medium
