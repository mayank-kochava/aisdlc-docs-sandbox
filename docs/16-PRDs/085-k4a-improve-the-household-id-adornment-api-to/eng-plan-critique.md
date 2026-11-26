# Engineering Plan Critique — K4A Enhanced Error Reporting

## Part 1 — Coverage Matrix

| Spec Item | Covered By | Status |
|---|---|---|
| BR-1 | Affected Systems (api) "reason at 3 cause sites"; data-model KeyedHousehold.reason | covered |
| BR-2 | Decision Log "Carry reason as proto enum + field 4"; contracts enum three values | covered |
| BR-3 | Affected Systems "Related systems" (avro TTL informs 'no match' wording); data-model BR-3 rule | covered |
| BR-4 | Confirmed Assumption (privacy pattern); Risk "Privacy leak via free-text"; enum-only decision | covered |
| BR-5 | Confirmed Assumption (additive/backward-compat); Corrected Assumption (bare-nil→populated-empty) | covered |
| FR-001 | Phase 2 "GetHousehold empty path" leaves happy path unchanged; state diagram MatchFound | covered |
| FR-002 | Phase 2 "redis.Nil→NO_MATCH_FOUND"; data-model NO_MATCH_FOUND | covered |
| FR-003 | Affected Systems "Net-new device-path input validation"; Phase 2; OQ "malformed definition" | covered |
| FR-004 | Phase 2 "post-retry failure→LOOKUP_DEPENDENCY_UNAVAILABLE" | covered |
| FR-005 | Phase 2 "Ensure FR-005 — exactly one named reason"; data-model FR-005 rule | covered |
| FR-006 | **NONE** (see CG-001) | gap |
| FR-007 | Phase 2 privacy pass; data-model BR-4; contracts category-only comment | covered |
| FR-008 | Affected Systems "no raw id/ip in reason or logs; hash-before-log"; Phase 3 no-PII assertion | covered |
| FR-009 | Affected Systems "bounded-retry wrapper (3 attempts, exp backoff, honor Retry-After)"; Phase 2 | covered |
| FR-010 | Confirmed Assumption (additive); Risk (bare-nil consumer); contracts additive comment | covered |
| NFR-001 | Phase 4 "Confirm success-path p95 unchanged"; OQ baseline source | deferred |
| NFR-002 | Affected Systems retry wrapper; Phase 2 | covered |
| NFR-003 | Phase 4 "Wire NFR-003 alert rule (>1% for 5 min)"; Dependencies table | covered |
| NFR-004 | Phase 2 privacy pass; Constraints; data-model Privacy constraint | covered |
| NFR-005 | Affected Systems "no raw id/ip ... in logs"; Phase 3 no-PII assertion | covered |
| NFR-006 | **NONE** (see CG-002) | gap |
| NFR-007 | Standards (additive); Confirmed Assumption; contracts | covered |
| NFR-008 | Open Questions / Dependencies ("malformed" definition, consumption surface via OQ) | deferred |
| NFR-009 | Dependencies "Full proto consumer enumeration"; Risk (blocking gate Phase 4) | deferred |
| NFR-010 | Phase 2 "Ensure FR-005"; data-model FR-005 | covered |
| NFR-011 | Phase 4 "Privacy review ... as a release gate" | covered |
| NFR-012 | Corrected Assumption (frontend-mos ruled out, off affected list) | covered |

## Part 2 — Findings

**CG-001 — FR-006 (actionability) has no trace in the plan.**
FR-006 requires the diagnostic reason be *actionable* — explaining the cause "in terms the agent can act on" and enabling a next step. The plan carries three enum values (NO_MATCH_FOUND, etc.) but nowhere addresses how a bare enum ordinal becomes an agent-actionable message. The data-model's "Error Message (category only)" column hints at human-readable strings ("No matching household record"), but the plan explicitly prefers *enum-only* for privacy ("Prefer enum-only (cannot leak by construction)"). An enum-only wire contract with no accompanying message text is arguably *not* actionable for an agent reading a raw API response (DEC-003 says this is backend/API-only with no frontend). There is a genuine tension between the enum-only privacy decision and FR-006/BR-3 actionability, and the plan resolves neither — it neither commits to a companion message field nor explains how the enum alone satisfies actionability. This is a real gap: resolve whether actionability is delivered by (a) an enum the consumption surface maps to text, or (b) a companion category-only message string, and reflect that in the contract.

**CG-002 — NFR-006 (access control preserved) has no trace in the plan.**
NFR-006 requires the enhancement not widen who can call the lookup API or read responses, enforced server-side. The plan nowhere confirms the existing auth/authorization posture on `GetHousehold`/`GetHouseholds`, nor that the new populated-empty response path doesn't inadvertently return data to a caller the old bare-nil path would have gated. This is likely trivially satisfied (no new RPC, no access-path change), but it is asserted nowhere — not in Affected Systems, not as a Confirmed Assumption, not as an Open Question. Per two-way coverage this is a silent gap: add a one-line Confirmed Assumption that the auth middleware/interceptor is unchanged and gates both RPCs identically, with a cited source.

**CG-003 — NFR-001 baseline is deferred but gates a Phase 4 release gate with no owner-confirmed number.**
Tracked correctly as an Open Question and Dependency ("Success-path p95 baseline + monitoring source", Open, needed by Phase 4). This is a routine deferred item, not blocking Phase 0/1 — flagging only to confirm it is correctly *not* treated as a Phase 0/1 blocker. No action required beyond keeping it on the Phase 4 gate.

**CG-004 — Batch-granularity decision is referenced as a "Register" item but there is no Register/Decisions-Required table in the plan.**
Multiple entries ("see Register", "batch-granularity decision") in Affected Systems and Phases 1/3 point to a decision register that does not exist in this document. The conditional `GetHouseholdsResponse.reason` field (proto tag 2), the batch-loop restructure complexity (M/H), and Phase 1/Phase 3 prerequisites all hinge on this decision. The plan lets Phase 1 proceed "on a provisional default," which is acceptable — but the decision itself is neither in the Open Questions table nor a Dependencies row. **This is a blocking finding for internal consistency:** a decision that gates Phase 3 scope and conditionally changes the Phase 1 proto contract (whether tag 2 ships) must be an explicit, tracked Open Question with an owner, not a dangling "see Register" reference to a table that isn't present. Add it to Open Questions with its blocking scope (Phase 3; conditionally Phase 1 contract shape).

**CG-005 — "Invalid input format" / "malformed" definition gates Phase 2 and is a cross-repo evidence gap.**
The plan correctly flags (Corrected Assumptions, Risk, Dependencies) that `HashDeviceIDAndType` returns no error and "malformed" is undefined, with `householdutils` internals "unverified (external, not vendored)." FR-003 cannot be implemented without this definition, and it is listed Open / needed-by Phase 2. This is correctly tracked as a Phase 2 blocker (not Phase 0/1), so it is *not* a blocking finding for G2 start — but note the evidence gap on `householdutils`: the validation logic depends on behavior of an external, non-vendored package the plan admits it could not inspect. Recommend confirming `householdutils` format semantics before Phase 2, or the net-new validation may be built against assumptions.

**CG-006 — NFR-012 / frontend-mos rule-out leaves "where agents read reasons" unconfirmed, which bears on CG-001.**
The plan correctly corrects frontend-mos to not-present (Read-verified, two passes) and marks it off the affected list. But it also notes "Where agents actually read reasons remains unconfirmed (OQ)." This consumption-surface question is the same one DEC-003/DEC-005/OQ-005 defer, and it is the hinge for whether enum-only is actionable (CG-001). It is appropriately deferred per the spec's own phasing — flagging only that CG-001's resolution and this OQ are coupled and should be resolved together.

---

## Verdict

**Not ready to proceed to G2 as-is.** Two blocking findings must be addressed first:

1. **CG-004** — the batch-granularity decision is referenced ("see Register") but has no tracked Open Question / owner, despite conditionally changing the Phase 1 proto contract (tag 2) and gating Phase 3 scope. Either the "Register" table was dropped in authoring or the decision was never formally raised; it must appear as an explicit, owned Open Question.
2. **CG-001** — FR-006 (actionability) is entirely untraced, and it collides with the plan's own enum-only privacy preference. This is a spec requirement with no coverage anywhere, not even a deferral.

Additionally, **CG-002** (NFR-006 access control) should be closed with a one-line evidence-backed assumption before handoff. CG-003/CG-005/CG-006 are correctly tracked as deferred Phase-2+/Phase-4 items and are not blocking for G2, but CG-005's `householdutils` evidence gap should be closed before Phase 2 begins.