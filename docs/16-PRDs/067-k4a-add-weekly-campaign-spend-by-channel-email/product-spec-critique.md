# Spec Critique: Meridian Travel Weekly Spend-by-Channel Summary

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-21

## Summary

- **Critical Issues:** 0
- **High Issues:** 6
- **Medium Issues:** 7
- **Blocking:** No

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | FR-005 triggers generation "for an entitled customer," but the entitlement check (FR-001/FR-002) and the scheduling trigger (FR-005) have no defined ordering guarantee against a mid-week plan downgrade. If entitlement is checked at trigger time but the plan changes before delivery (FR-011), it is undefined whether a summary already built for a now-ineligible customer is still delivered. | High | 3 (FR-001/005/011) | Add an FR requiring entitlement to be re-verified immediately before delivery, not only before generation, and define behavior if it changed between the two points. |
| CG-002 | The flowchart branches `G -> L` (render on-screen) with no entitlement/authorization gate on the write of the on-screen artifact; authorization (FR-004) only gates the *read*. If a summary is generated for a customer who loses entitlement, FR-015's "not yet available" state vs. a stale rendered summary is undefined. | High | 2, 3 (FR-004/014/015) | Define whether on-screen rendering is suppressed/invalidated when entitlement is revoked after generation. |
| CG-003 | FR-003 fails closed when the account source is unreachable at *generation*, but FR-004 fail-closed behavior for the *on-screen read* when the auth source is unreachable is only asserted in NFR-008, not in any FR with Given/When/Then. The on-screen read path has no testable fail-closed requirement. | High | 3 (FR-004), 5 (NFR-008) | Add an FR: "Given the auth/account source is unreachable, When a user requests the on-screen view, Then the request is rejected server-side (fail closed)." |

No issue rises to the Critical threshold (data loss / security breach / implementation-blocking) given the fail-closed posture and the explicit DECISION REQUIRED markers. The entitlement-timing and on-screen-revocation gaps are serious and should be closed before engineering, but each has a safe default and does not block starting.

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "must return within a p95 latency target ... (e.g. N concurrent requests, p95 ≤ X ms)" | Placeholder targets `N` and `X` are not testable; no automated test can assert against unbound variables. | Given N=200 concurrent on-screen requests, When the summary view is requested, Then p95 response time ≤ 800ms. (Fill real numbers in eng-plan and bind them here.) |
| NFR-002 | "must have their own separate rate caps ... Concrete caps are an engineering decision" | No numeric cap = nothing to test. | Given the batch path exceeds its cap of C_batch queries/sec, When another batch query is issued, Then it is throttled; same for C_read on the read path. |
| NFR-003 | "generated and delivered within a bounded window after its weekly trigger time" | "Bounded window" is undefined; DEC-006 open. | Given the weekly trigger fires at T, When generation+delivery runs, Then delivery completes within W minutes of T (define W). |
| FR-007 | "produces the same end state for that week" | "Same end state" is not directly observable/assertable without defining the comparable artifact. | Add: Then the stored summary for (customer, covered week) is byte-identical to / supersedes the prior artifact, and the generation-run count for that key increments without creating a second summary row. |
| FR-012 | "exponential backoff up to 3 attempts" | Backoff base/interval unspecified — two implementations both "pass." | Given attempt N fails, When retrying, Then the delay follows base*2^(N-1) with base = B seconds (define B), for N ≤ 3. |

All 16 FRs are written in valid Gherkin — no prose FRs found (see Section 5).

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Multi-service blast radius | Yes — NFR-011 kill-switch (config, no deploy, 5-min propagation) | PASS | Well specified, including in-flight run behavior. |
| Customer-facing email reliability | Yes — FR-011–013, NFR-004, NFR-005 | PASS | Named failure conditions + Retry-After honored + alerting. Strong. |
| Service-topology ambiguity (KG lookups disagree) | Partial — NFR-013 requires engineer confirmation before build | PARTIAL | The risk is *gated*, not *mitigated*. "No build work should proceed against an unconfirmed service map" is a process guard, not a design mitigation. Acceptable, but ensure the pipeline enforces the gate. |
| Hard 5-week deadline | Partial — NFR-014 ties to resolving open decisions + Section 6 phasing | PARTIAL | Mitigation is conditional on 7 unresolved DECISION REQUIRED items being closed "early." No owner or date assigned to those decisions. |
| New UI/UX (on-screen view) | Weak | FAIL | HIGH-risk tier explicitly cites "new UI/UX," but there is no NFR addressing front-end-specific risk (accessibility, loading/error states beyond empty-state, cross-browser). FR-015 covers empty state only. Add a UI-robustness guardrail or explicitly scope it out. |
| Legacy/upstream data-freshness (sam-campaign-data-sync) | No | FAIL | KG cross-check surfaces a campaign-data-sync service as an upstream data-freshness dependency. FR-009 handles missing data, but nothing requires detecting/handling *stale-but-present* data (sync lag producing last-week's numbers labeled as this week's). See Section 4. |

---

## 4. Missed Edge Cases

1. **Stale upstream sync (data-freshness):** Both the Context Seed and the independent KG cross-check name a `sam-campaign-data-sync` / `k4a-sam-campaign-data-sync` service as the upstream that feeds spend data. FR-009/FR-010 handle *missing* data, but nothing handles data that is *present but stale* — e.g., the sync last ran before the covered week closed. The summary would show complete-looking but wrong figures. No requirement asserts a freshness/as-of check against the sync service.
2. **Entitlement revoked between generation and delivery:** Covered in CG-001 — timing window undefined.
3. **Entitlement revoked after on-screen render:** Covered in CG-002.
4. **Partial-week trigger / timezone boundary:** "Covered week" is "system-derived from the generation trigger." The customer's billing/campaign timezone vs. the system timezone is undefined; a Monday-00:00-UTC trigger may slice a channel's spend day differently than marketing expects. No FR pins the week boundary definition.
5. **Recipient list empty or all-invalid:** FR-011 delivers to "configured marketing-team recipients," but DEC-002 leaves recipient resolution open. No FR covers the case where the resolved recipient list is empty or every address bounces hard (vs. a transient failure in FR-012).
6. **On-screen view during an in-progress generation:** FR-014 shows "the latest" summary; FR-015 shows empty state when none exists. The state where a run is *in progress* (FR-006) and a prior week's summary exists is undefined — does the user see last week's, or an "updating" state?
7. **Idempotency across a kill-switch flip mid-run:** NFR-011 says an in-flight run completes under prior behavior, but FR-007 idempotency + a re-trigger after the switch is re-enabled isn't reconciled. Could a completed-under-prior-behavior run plus a re-run double-count?
8. **Audit write failure during the audited action:** NFR-010 says audit writes are async and a failure alerts — but does the generate/deliver action succeed or get held if its audit record never lands? Compliance-relevant; undefined.
9. **Rate-limit (429) with no Retry-After header:** FR-013 honors Retry-After, but the downstream may return 429 *without* that header. Fallback backoff for that sub-case is unspecified.
10. **Channel set changes between weeks:** BR-6/DEC-004 leaves the channel set open. If a channel is added/removed between weeks, week-over-week comparability (and FR-007 idempotency if the set changes on re-run) is undefined.
11. **Multiple on-screen viewers / caching:** No requirement on whether the on-screen view is cached and how cache invalidation interacts with a fresh generation — a viewer could see a stale cached summary just after regeneration.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-016 are all in valid gherkin fenced blocks. No prose FRs. |
| Document contains only Sections 0–6 | FAIL | The document ends with an **"## Appendix: Engineering Reference"** containing a multi-paragraph narrative service-map analysis with its own per-service findings and a "Reconciliation" recommendation paragraph. This is exactly the kind of stray multi-paragraph narrative appendix the format rule is meant to catch — it is **not** a sanctioned Revision Summary nor a short one-line `> **Label:**` metadata trailer. It should be removed from the spec proper (moved to eng-plan or a linked reference doc). |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 contains only bullet-list items with prose descriptions; no attribute tables or type declarations. |

---

## KG Cross-Check Finding (reconciliation against independent lookup)

The spec is commendably honest about the topology ambiguity: Section 1's Reconciliation Note and NFR-013 both flag that two KG lookups disagree and that no service map is settled. My independent fresh lookup re-found the **SAM-oriented cluster** (k4a-sam-backend, sam-report-scheduler, sam-performance-reporter, sam-campaign-data-sync, k4a-sam-campaign-data-sync, sam-api, notifications-api, reporting-subscriptions-worker) and did **not** re-find the reporting/notifications cluster (reporting-api, reporting-dispatcher, accounts-api, k4a-mcp-server, frontend-mos) except for the shared notifications-api.

- **No contradiction of a concrete implementation claim:** The spec makes no claims about specific endpoints, tables, joins, or config that the evidence refutes — it deliberately keeps everything role-level. So there is nothing to flag as a factual conflict.
- **Discrepancy worth surfacing (already surfaced):** The spec attributes entitlement verification to `accounts-api` (Lookup 1 only). My fresh lookup did not re-find `accounts-api`; in the SAM cluster, entitlement verification has no clear owner. The spec already handles this correctly by treating the account source abstractly ("authoritative account source") in FR-001/003, so no FR needs rework — but the engineering gate in NFR-013 must specifically confirm *which* service owns entitlement before FR-001 can be implemented.

This finding is consistent with the spec's own posture; no new Critical/High issue is raised by the cross-check beyond the data-freshness gap already logged as Missed Edge Case #1.

---

## 6. Recommendations

1. **Close the entitlement-timing gaps (CG-001, CG-002, CG-003):** Add FRs for re-verifying entitlement before delivery, suppressing/invalidating the on-screen artifact on revocation, and failing closed on the on-screen read path.
2. **Add a data-freshness requirement (Edge Case #1):** Require an as-of / sync-recency check against the upstream campaign-data-sync service so stale-but-present data cannot masquerade as the covered week's spend.
3. **Move the Engineering Reference appendix out of the spec** to restore Sections 0–6 format compliance (the only FAIL row).
4. **Bind the placeholder NFR targets** (NFR-001 N/X, NFR-002 caps, NFR-003 window W, FR-012 backoff base) to concrete, testable numbers — or explicitly state they are carried to eng-plan with a named owner/date.
5. **Add a UI-robustness guardrail** addressing the HIGH-tier "new UI/UX" risk (loading/in-progress state, error state, accessibility) or explicitly scope it out.
6. **Resolve the remaining edge cases** (timezone/week boundary, empty recipient list, in-progress on-screen state, 429-without-Retry-After, channel-set changes, audit-write-failure disposition, caching/invalidation) before or in parallel with engineering per severity.
7. **Enforce the NFR-013 build gate operationally** — ensure the pipeline actually blocks implementation until the service map (especially the entitlement owner) is confirmed.