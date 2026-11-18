---
id: product-spec
title: Product Specification
---

# K4A MCP — Date Format Standardization (YYYY-MM-DD)

## 1. Business Logic & Context

### Problem

A gaming customer consuming K4A MCP responses finds the current date values confusing. The date format(s) returned by the K4A MCP are inconsistent with the customer's expectations, creating ambiguity in how date values are interpreted. The current format and the exact location(s) of date fields within the K4A MCP response are **not yet known** and must be confirmed by engineering before implementation.

### Value

Returning dates in the unambiguous ISO 8601 `YYYY-MM-DD` format removes interpretation ambiguity (e.g. no confusion between day-first and month-first ordering), reducing customer confusion and support overhead for the gaming customer and any future consumer of K4A MCP date values.

### Business Rules

- **BR-1:** All date values returned in K4A MCP responses MUST be formatted as `YYYY-MM-DD` (four-digit year, two-digit month, two-digit day, hyphen-separated).
- **BR-2:** The identification of which date fields exist in the K4A MCP response, and their current format, is a prerequisite to implementation and is owned by engineering discovery.
- **BR-3:** The customer-facing consumption surface (direct API response, console display, or both) determines which layer(s) must emit the standardized format. [DECISION REQUIRED: DEC-001]
- **BR-4:** Whether date values carry a time-of-day component and how timezone/locale is handled must be resolved before a date field can be deterministically reformatted. [DECISION REQUIRED: DEC-002]
- **BR-5:** Whether existing stored/logged date values are reformatted retroactively, or whether only newly generated responses adopt the format, must be settled. [DECISION REQUIRED: DEC-003]

### Affected-Services Reconciliation (requirement-agent cross-check)

An independent knowledge-graph lookup did not cleanly match the Context Seed's named backend service. The Context Seed names a single backend owner of date formatting; the independent lookup surfaced **two candidate backend names** for what is likely the same service (one at high confidence, one at medium confidence flagged as an "alternate" that "may own date format"). This naming discrepancy means **which backend repository actually owns the date-serialization output is unconfirmed**, and resolving it is step zero for implementation. A frontend console that displays K4A MCP dates to the customer was named in the Context Seed but not re-surfaced by the independent lookup; its involvement depends on whether the customer consumes dates via the API directly, via the console, or both (see BR-3). [DECISION REQUIRED: DEC-004]

## 2. User Flows (Visuals)

```mermaid
flowchart TD
    A[Consumer requests data from K4A MCP] --> B{K4A MCP builds response}
    B --> C{Response contains date field(s)?}
    C -->|No date fields present| D[Return response unchanged]
    C -->|Date field(s) present| E{Date value resolvable to a calendar date?}
    E -->|Yes| F[Format each date value as YYYY-MM-DD]
    F --> G[Return response with standardized date format]
    G --> H{Consumer surface}
    H -->|Direct API consumer| I[Consumer receives YYYY-MM-DD in payload]
    H -->|Console display| J[Console renders date to gaming customer]
    J --> K{Console re-formats the date?}
    K -->|No, passes through| L[Customer sees YYYY-MM-DD]
    K -->|Yes, re-formats| M[Console display logic must preserve YYYY-MM-DD]
    M --> L

    E -->|No - null / malformed / unparseable value| N[Error State: value cannot be reformatted]
    N --> O[Preserve original value, do not corrupt it; surface a recorded formatting-failure indication]
```

## 3. Functional Requirements

> Scoping note: Several requirements below are gated on discovery outcomes (current format, field locations, consumption surface). Where a requirement depends on an unresolved decision, the dependency is stated in its `Given` precondition so a downstream agent does not implement against an assumed answer.

### Feature Area A — Backend Date Serialization

**FR-001 — Standardize date output format**
```gherkin
Given the K4A MCP server produces a response containing one or more date fields
  And each such field has been identified during engineering discovery
When the response is serialized for return to the consumer
Then every identified date value is rendered in YYYY-MM-DD format
  And the year is four digits, the month is two digits, the day is two digits, hyphen-separated
```

**FR-002 — Apply format consistently across all date fields**
```gherkin
Given engineering discovery has enumerated every date field in the K4A MCP response payload
When any response containing those fields is returned
Then all enumerated date fields use YYYY-MM-DD format
  And no enumerated date field retains the prior format
```

**FR-003 — Preserve non-date and empty values**
```gherkin
Given a K4A MCP response field that is null, empty, or not a date value
When the response is serialized
Then that field is returned unchanged
  And no date formatting is applied to it
```

**FR-004 — Handle unparseable or malformed date values**
```gherkin
Given a field identified as a date field holds a value that cannot be resolved to a valid calendar date
When the response is serialized
Then the original value is preserved without corruption
  And a formatting-failure condition is recorded for operational visibility
  And the failure of one field does not block formatting of other valid date fields in the same response
```

**FR-005 — Time-of-day and timezone handling**
```gherkin
Given a date field whose source value carries a time-of-day and/or timezone component
  And a decision has been made on whether responses expose date-only or date-plus-time [see BR-4]
When the value is formatted
Then the value is rendered according to that decision
  And the same rule is applied uniformly to every date field exposing the same semantic
```

### Feature Area B — Consumer Consumption Surface

**FR-006 — Direct API consumer receives standardized format**
```gherkin
Given a consumer calls the K4A MCP API directly
  And FR-001 is in effect
When the consumer receives the response
Then all date values in the payload are in YYYY-MM-DD format
```

**FR-007 — Console display preserves standardized format**
```gherkin
Given the gaming customer views K4A MCP date values through the console display surface
  And the backend emits dates in YYYY-MM-DD format
When the console renders a date value
Then the value shown to the customer is in YYYY-MM-DD format
  And the console does not re-format the value into a different date format that would override the backend output
```

**FR-008 — Single authoritative formatting layer**
```gherkin
Given the consumption surface for the gaming customer has been confirmed (direct API, console, or both) [see BR-3]
When the format standardization is implemented
Then exactly one layer is the authoritative source of the YYYY-MM-DD format for a given consumption path
  And no second layer re-parses and re-formats the date in a way that could diverge from or silently override the authoritative output
```

### Feature Area C — Backward Compatibility & Downstream Consumers

**FR-009 — Identify and account for downstream consumers**
```gherkin
Given engineering discovery investigates systems that consume K4A MCP date responses
When a downstream consumer that parses the current date format is found
Then the impact of the format change on that consumer's parsing logic is documented before rollout
  And either the consumer is confirmed compatible with YYYY-MM-DD or a coordinated change is sequenced
```

**FR-010 — No silent breakage of existing consumers**
```gherkin
Given the K4A MCP date format changes from its current format to YYYY-MM-DD
When the change is released
Then any existing consumer whose parsing depends on the prior format is accounted for per FR-009
  And the change does not fail silently for a consumer that was not identified during discovery, insofar as discovery coverage is documented
```

**FR-011 — Scope of retroactive application**
```gherkin
Given a decision on whether the format applies to existing stored/logged date values or only to newly generated responses [see BR-5]
When the change is deployed
Then date values in scope per that decision are returned in YYYY-MM-DD format
  And date values explicitly out of scope per that decision are left in their prior format
```

I have completed Sections 0–3. Now reporting the decisions I flagged.

## 4. Information Requirements

The system must track the following business information to support the Functional Requirements. This is a format-standardization change, so the information set is deliberately narrow — the critical items are the discovery outputs that scope the work, not new persisted entities.

- **Date Field Inventory** — The enumerated list of every date-bearing field in the K4A MCP response payload, with each field's current format. System-derived via engineering discovery (per BR-2). This is the prerequisite input to FR-001, FR-002, and FR-005; without it the format change cannot be applied completely or verifiably.
- **Current Date Format per Field** — The present serialization format of each identified date field (e.g. ordering, separators, presence of time-of-day). System-derived / discovery output. Matters because the transformation from current to `YYYY-MM-DD` (FR-001) and the detection of consumers that parse the old format (FR-009) both depend on knowing the starting format.
- **Date Value Semantic (date-only vs date-plus-time, timezone/locale)** — Whether each date field's source value carries a time-of-day and/or timezone component. System-derived from the source data, pending the human decision on exposure (BR-4, FR-005). Matters because a value carrying time/timezone cannot be deterministically reduced to `YYYY-MM-DD` without a stated rule.
- **Consumption Surface** — Whether the gaming customer receives dates via the direct API response, the console display, or both. Discovery output pending human confirmation (BR-3). Matters because it determines which layer is the authoritative formatting source (FR-008) and whether the console needs any change (FR-007).
- **Downstream Consumer Registry** — The set of external systems or APIs that consume K4A MCP date responses and parse the current format. System-derived via discovery (FR-009). Matters because an unidentified consumer parsing the old format is the primary silent-breakage risk (FR-010).
- **Formatting-Failure Indication** — A recorded signal that a field identified as a date could not be resolved to a valid calendar date (FR-004). System-derived at serialization time. Matters for operational visibility into malformed source values without corrupting the returned payload.
- **Retroactive Scope Flag** — Whether the format applies to existing stored/logged date values or only to newly generated responses (BR-5, FR-011). Human decision, consumed by the system at deployment. Matters because it bounds which data the change touches.

## 5. Non-Functional Requirements & Guardrails

### Performance

- **NFR-001 — No added latency from reformatting.** The date-formatting step MUST NOT materially degrade K4A MCP response latency. Target: response-time p95 after the change remains within the pre-change p95 baseline plus a negligible margin, measured under representative concurrent load. The concrete baseline value and acceptable margin are not supplied by the Context Seed. [DECISION REQUIRED: DEC-005]
- **NFR-002 — Formatting is a bounded, in-line operation.** Date formatting MUST be a deterministic in-memory transformation applied during response serialization and MUST NOT introduce an external call, additional round-trip, or unbounded operation into the response path.

### Correctness & Resilience

- **NFR-003 — Deterministic, uniform formatting.** Given the same source value and the same resolved time/timezone rule (BR-4), the formatted output MUST be identical on every request. The same formatting rule MUST apply uniformly to every date field sharing the same semantic (reinforces FR-002, FR-005).
- **NFR-004 — No payload corruption on failure.** A formatting failure on one field (FR-004) MUST NOT corrupt, drop, or block other fields in the same response, and MUST NOT cause the overall request to error. A formatting-failure indication is recorded for operational visibility; an elevated rate of such failures SHOULD alert when the failure rate exceeds a defined threshold for a sustained window. The specific rate threshold and window are not supplied by the Context Seed. [DECISION REQUIRED: DEC-006]

### Backward Compatibility

- **NFR-005 — Existing consumers accounted for before cutover.** This change modifies the output contract of an existing service with potential existing consumers. Before release, every identified downstream consumer that parses the current format MUST be confirmed compatible with `YYYY-MM-DD` or have a coordinated change sequenced (per FR-009, FR-010). The discovery coverage MUST be documented so unidentified-consumer risk is explicit rather than assumed absent.
- **NFR-006 — Single authoritative formatting layer.** For any given consumption path, exactly one layer owns emission of the `YYYY-MM-DD` format; no second layer may re-parse and re-format in a way that diverges from or silently overrides it (reinforces FR-008). This prevents the cross-layer trap where the fix lands in the wrong layer or is double-applied.

### Security & Data Handling

- **NFR-007 — No sensitive data exposure in failure logging.** The formatting-failure indication (FR-004, NFR-004) and any associated operational logging MUST NOT log raw sensitive or user-identifying data. If a problematic value must be referenced for diagnosis, it is referenced by field identity, not by logging a raw sensitive payload value.
- **NFR-008 — Access and visibility unchanged.** This change alters only the *format* of date values, not *who* can see which data. The change MUST NOT widen access to any date field or expose a date field to a consumer not already entitled to it.

### Risk Mitigation

Addressing each risk surfaced by the Context Seed (MEDIUM risk tier; cross-layer blast radius with unknown scope):

- **Risk: Unknown current format and field locations.** Mitigated when the Date Field Inventory and current-format-per-field (Section 4) are produced and documented by engineering discovery before implementation (BR-2, FR-001, FR-002). Implementation MUST NOT proceed against an assumed format.
- **Risk: Silent breakage of downstream consumers parsing the old format.** Mitigated when the Downstream Consumer Registry is produced, each consumer is confirmed compatible or coordinated, and discovery coverage is documented (FR-009, FR-010, NFR-005). A format change produces no compile error, so this must be verified, not assumed.
- **Risk: Fix lands in the wrong layer (backend vs console).** Mitigated when the consumption surface is confirmed (BR-3) and a single authoritative formatting layer is designated (FR-008, NFR-006), so the customer actually sees `YYYY-MM-DD` and the change is neither invisible nor double-applied.
- **Risk: Wrong repository touched due to backend name collision.** Mitigated when the naming discrepancy between the candidate backend services (see the reconciliation note in Section 1) is resolved and the single service owning date serialization is confirmed before code changes begin.
- **Risk: Ambiguous time/timezone handling produces inconsistent dates.** Mitigated when the date-only vs date-plus-time and timezone/locale rule is decided (BR-4, FR-005) and applied uniformly (NFR-003).
- **Risk: Cross-layer coordination.** Because the change spans backend and (potentially) frontend, release MUST be sequenced so the backend format contract and any console display behavior change land in a compatible order, avoiding a window where the console breaks on a new backend format or overrides it.

## 6. Out of Scope & Phasing

### Phase 1 (this requirement)

- Returning K4A MCP date values in `YYYY-MM-DD` format, scoped to the date fields enumerated by engineering discovery.
- Ensuring the gaming customer perceives the standardized format on their confirmed consumption surface (per BR-3).

### Explicitly Out of Scope for Phase 1

| Item | Rationale |
| --- | --- |
| Retroactive reformatting of existing stored data/logs | Depends on an unresolved decision (BR-5, FR-011); default posture is new responses only unless decided otherwise — see the retroactive-scope decision flagged in Section 4. |
| Adding or changing which date fields exist in the response | This is a format change only; field set, additions, and removals are not in scope. |
| Changing the meaning, timezone semantics, or time-of-day inclusion of dates beyond what BR-4 resolves | Only the string format is standardized; semantic changes beyond the BR-4 decision are a separate effort. |
| Broader locale/internationalization of date presentation | Not requested; the requirement is a single fixed ISO format, not locale-aware presentation. |
| Reformatting dates returned by services other than the K4A MCP | Requirement is scoped to the K4A MCP response surface only. |
| Modifying downstream consumers' own internal date handling beyond coordinating the contract change | Discovery documents impact (FR-009); re-engineering a downstream consumer is that team's own work, not this requirement. |

### Rollback

Because this is a cross-layer change with unknown scope (MEDIUM risk), rollback is called out explicitly. Trigger: an elevated formatting-failure rate (NFR-004) or a confirmed downstream consumer breakage after release. Owning authority and the specific timing target for rollback are not supplied by the Context Seed. [DECISION REQUIRED: DEC-007] Rollback reverts the serialization behavior for new responses going forward; it does not retroactively alter responses already returned to consumers.

---
## Open Questions & Decisions Register

_Durable, ID'd reference for HITL 2 (G1) and beyond — inline `[DECISION REQUIRED: DEC-00N]` tags in the sections above point back here. Reflects the latest round only._

### Decisions Required (business trade-offs — needs a human call)

_Each carries an **Owner** (accountable for supplying the answer) and a **Provisional Value** (the working assumption engineering may build against until the decision lands), so none of these fully block engineering — implementation proceeds on the provisional value and adjusts if the confirmed decision differs._

| ID | Decision | Owner | Provisional Value (build against until confirmed) |
|----|----------|-------|-----------------------------------------------------|
| **DEC-001** | Which consumption surface(s) the gaming customer actually uses for K4A MCP dates — direct API response, the console display, or both — is unconfirmed. This determines which layer(s) must emit YYYY-MM-DD and where the authoritative formatting lives (BR-3, FR-006/007/008). | Product owner (Mayank Ukey) with input from Engineering discovery | Treat the backend API response as the authoritative formatting layer (format dates to YYYY-MM-DD at serialization in the K4A MCP backend). Have the console pass the value through unchanged rather than re-formatting. This covers the direct-API consumer unconditionally and makes the console correct by default; revisit only if discovery shows the customer consumes dates exclusively via a console that re-formats. |
| **DEC-002** | Whether K4A MCP date values carry a time-of-day component and how timezone/locale should be handled is unknown. YYYY-MM-DD is date-only, so a field currently carrying a timestamp needs an explicit rule on whether to drop or separately expose the time, and in which timezone the calendar date is derived (BR-4, FR-005). | Product owner (Mayank Ukey) with Engineering discovery confirming which fields carry time/timezone | Expose date-only YYYY-MM-DD for customer-facing confusion-reduction. Where a source value carries a timestamp, derive the calendar date in UTC and render date-only, applied uniformly across all date fields. If any field semantically requires the time component, flag it during discovery before dropping it. |
| **DEC-003** | Whether the format change applies retroactively to existing stored/logged date values or only to newly generated responses going forward is unspecified (BR-5, FR-011). | Product owner (Mayank Ukey) with Engineering on feasibility/cost of a backfill | Apply going forward only — new responses adopt YYYY-MM-DD; existing stored/logged values are left untouched. This is the lower-risk, lower-cost default consistent with the 'not urgent' constraint. Only pursue a retroactive reformat if the customer explicitly requires historical responses to match. |
| **DEC-004** | Which backend repository actually owns the date-serialization output is unconfirmed — the independent KG cross-check surfaced two candidate backend names for what is likely the same service (one high-confidence, one medium-confidence 'alternate'), and could not cleanly reconcile them with the Context Seed's single named backend. 'Which repo do I even touch' must be resolved before implementation. | Engineering (repo-owning team to confirm service identity/layout) | Treat the high-confidence backend named in the Context Seed as the owner of date serialization and target it first. Before coding, have an engineer who knows the repo layout confirm whether the two candidate names are one service, an old/new split, or two live services — and if a duplicate is live, ensure the fix covers it rather than leaving a stale path emitting the old format. |
| **DEC-005** | Latency baseline and acceptable added-latency margin for the reformatting step (NFR-001) are not supplied — a concrete p95 baseline and margin are needed to make the performance guardrail verifiable. | Engineering (kochava-k4a-mcp-server owner), with Product sign-off | Post-change response p95 must stay within the current p95 baseline + 5ms, measured under representative concurrent load; formatting is an in-memory transform so near-zero added latency is expected. |
| **DEC-006** | Formatting-failure alert threshold and sustained window (NFR-004) are not supplied — needed to make the failure-rate alert actionable without noise. | Engineering / On-call (operational owner) | Alert when the date-formatting failure rate exceeds 1% of date fields processed for 5 minutes, sustained (Kochava's standard alerting shape). |
| **DEC-007** | Rollback owning authority and timing target for this cross-layer change (Section 6 Rollback) are not supplied. | Engineering lead / On-call for the backend service, with Product (Mayank Ukey) informed | On-call engineer for the K4A MCP backend may roll back without separate approval; target full rollback of serialization behavior for new responses within 30 minutes of a confirmed breakage or sustained failure alert; no retroactive reversion of already-returned responses. |

### Open Questions (evidence gaps — needs more input)

- **OQ-001**: kochava-k4a-mcp-server: The KG shows two backend names — 'kochava-k4a-mcp-server' and 'k4a-mcp-server' — for what may be the same service. Is this one repo, two, or an old/new split? Which one is live and should be changed?
- **OQ-002**: kochava-k4a-mcp-server: Which specific date fields exist in the K4A MCP response payload, and what format is each in today? Engineering needs this confirmed before scoping (the KG didn't surface the fields).
- **OQ-003**: kochava-k4a-mcp-server: Are there downstream systems/APIs (besides MOS) that parse the current date format? A format change could silently break them — do we need a compatibility window or versioned response?
- **OQ-004**: kochava-k4a-mcp-server: Should dates include a time component, or date-only? And is timezone/locale a factor for how YYYY-MM-DD is produced?
- **OQ-005**: frontend-mos: Does the gaming customer see these dates via the raw API response, via the MOS Console UI, or both? This determines whether frontend work is needed at all.
- **OQ-006**: frontend-mos: Does MOS re-parse/re-format the backend's date string for display, or render it as-is? This decides whether the fix belongs in backend, frontend, or both.
- **OQ-007**: Scope: Should the new YYYY-MM-DD format apply only to new responses going forward, or also retroactively to existing stored data/logs?
- **OQ-008**: Scope: What's the relative priority and engineering capacity for this vs other K4A work, given there's no deadline?

---
> **Classification:** Type B — requirement-agent's independent re-check DISAGREES with intake's original Type A call. The desired behavior is clearly specified (return dates as YYYY-MM-DD); the only open items are engineering implementation details (which fields, current format, downstream parsers) across the k4a-mcp-server backend and MOS frontend, which is engineering-led design, not product discovery.
