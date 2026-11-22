# Spec Critique

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-22

## Summary

- **Critical Issues:** 2
- **High Issues:** 5
- **Medium Issues:** 6
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **Entitlement for the sole target customer is unresolved (BR-1).** Brightway is a standard-plan customer, and whether the feature is plan-gated is flagged [DECISION REQUIRED]. If gated-by-default, the entire deliverable is unusable by the one customer it is being built for — the happy path in FR-006/FR-008 (entitled user downloads) may never trigger for the actual target. This is not a mere ambiguity; it can nullify the MVP against a hard 5-week deadline. | Critical | 1 (BR-1), 3 (FR-006/009) | Resolve entitlement before build starts. If gated, add an explicit requirement for the entitlement grant/change as an in-scope prerequisite, not a "later decision" (which contradicts Out-of-Scope row 6). | 
| CG-002 | **Availability gate (BR-5) has no known deadline, yet it is the core success criterion.** The entire Value proposition is "available before year-end close," but the close date is not supplied [DECISION REQUIRED]. There is no FR or NFR that asserts/verifies the report is ready before that date. The success condition is literally untestable until this input exists, and the 5-week risk cannot be managed against an unknown target. | Critical | 1 (BR-5), 5 (Risk table) | Obtain the exact year-end close date and add a dated acceptance criterion: "Given the close date D, the first valid weekly report must be available no later than D minus [buffer]." |

---

## 2. Testability Review

### Vague Requirements Found

The FRs themselves are in solid Gherkin. The vagueness lives mostly in NFRs and in Gherkin steps that assert undefined concepts.

| FR/NFR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-002 | "operate under an explicit rate cap... The specific cap is an engineering default to be set during planning." | No numeric cap = not testable. "explicit rate cap" with an unset value cannot be asserted. | Given the weekly generation job queries the attribution source, When it issues read requests, Then the request rate must not exceed [N] req/s, And requests beyond [N] are queued or throttled, not dropped. (N must be set.) |
| NFR-009 | "generation time of day is an engineering/operational default, chosen to complete before the finance team's typical workday." | "typical workday" is unmeasurable; no timezone, no completion deadline. | Given a weekly cycle, When generation runs, Then the available report must exist by [HH:MM TZ], And a cycle not complete by [HH:MM TZ] raises an alert. |
| NFR-001 | "measured under expected concurrent finance-user load" | "expected...load" is undefined — p95 ≤ 5s is meaningless without a concurrency figure. | Given [C] concurrent download requests for an available report, When authorized, Then p95 latency ≤ 5s. (C must be set.) |
| NFR-005 | "link validity window is an engineering default to be set during planning." | No validity window value = cannot test link expiry. | Given a delivered secure link, When [T] minutes have elapsed, Then the link returns an access-denied/expired response. (T must be set.) |
| FR-010 | "the report content is unavailable or the retrieval fails" | Two distinct failure modes collapsed; "unavailable" overlaps with FR-007 empty state. Ambiguous which state to assert. | Split: one scenario for content-exists-but-retrieval-fails, one for content-purged/unavailable, each with its own named error state. |

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard external deadline (5 weeks) | Partial | **FAIL** | Mitigation leans on resolving BR-5 (close date) and BR-6 "early" — but both remain [DECISION REQUIRED] in the spec itself. A mitigation that depends on an unresolved input is not yet a mitigation. No dated acceptance criterion exists. |
| Multi-service blast radius (5+ services) | Yes | PASS | NFR-004, NFR-010, NFR-008, NFR-009 concretely contain cross-service failure. Strong. |
| Unconfirmed UX scope | Yes | PASS | MVP UX bounded to FR-006–010; customization explicitly out of scope (Section 6). |
| Unclear recurring vs. one-time | Yes | PASS | Resolved to weekly (BR-2); no one-time path built. |
| Legacy/shared attribution data source dependency | Yes | PASS | NFR-002 rate cap + NFR-008 bounded retry/backoff + Retry-After honoring. Adequate (pending numeric cap, see Testability). |
| Service-ownership ambiguity for report-content data | Flagged, not resolved | **FAIL** (as a mitigation) | Section 1 and the Appendix correctly refuse to pick a side, which is honest — but no FR/NFR makes the report robust to the ambiguity (e.g. an interface-contract requirement independent of which backend wins). Add a requirement that report content is consumed via a defined contract regardless of owning service. |
| Audit/regulatory retention mandate (NFR-007) | No | **FAIL** | 12-month minimum is asserted but the actual regulatory obligation is [DECISION REQUIRED]. If a longer finance/audit mandate applies, 12 months is non-compliant. No mitigation beyond flagging. |

---

## 4. Missed Edge Cases

1. **Q4 boundary / which year's Q4:** BR-3 says "last quarter (Q4) only," but "last quarter" and "Q4" are not the same thing — trailing-quarter rolls forward each week, yet Q4 is fixed. Across the year-end boundary, does a weekly run in January 2027 still report Q4 2026, or roll to Q1? FR-002 conflates "last quarter" and "Q4." Undefined and likely to produce wrong data at the exact moment (year-end close) the report matters most.
2. **No data for the quarter:** FR-001/FR-002 assume attribution data exists. What is produced if the Q4 window has zero attribution rows — an empty-but-valid PDF, or a generation failure? Undefined.
3. **Partial data at generation time:** If the attribution source returns a success but with incomplete/late-arriving data, the report renders successfully but is wrong. No freshness/completeness check requirement.
4. **Stale "prior report remains available" (FR-003):** When every cycle fails for weeks, the user keeps downloading an increasingly stale report with no indication of staleness. No requirement to surface the report's as-of/generated-at date or a staleness warning.
5. **Concurrent kill-switch flip vs. in-flight download:** NFR-010 covers in-flight *generation*, but not an in-flight *download* whose secure link was issued just before the flip. Does the link still resolve? Undefined.
6. **Secure link after rollback (NFR-005 + NFR-011):** Rollback does not withdraw already-produced reports, but does a previously issued, still-valid time-limited link to a report with *incorrect content* remain downloadable? Potential delivery of known-bad data.
7. **Entitlement revoked between generation and download:** User was entitled at generation, loses entitlement, then downloads. FR-009 re-checks server-side (good), but the audit/empty-state transition for a mid-cycle revocation isn't specified.
8. **Timezone of "weekly" and "Q4":** No timezone is defined for the weekly cadence or the quarter boundary. Finance close is date-sensitive; an off-by-one-day due to TZ could miss the close.
9. **Download of a cycle that was superseded mid-download:** A new weekly report becomes available while the user is mid-download of the prior one. Which artifact is served; is the link to "latest" or to a specific cycle? FR-006 says "the available report" (singular) but FR-003 implies multiple can exist.
10. **Audit-write failure during kill-switch incident:** NFR-007 raises an alert on audit-write failure, but during an incident the feature may be disabled — does a disabled feature still attempt/alert on audit writes? Interaction unspecified.
11. **reporting-dispatcher vs. reporting-subscriptions-worker ownership of cadence:** The spec itself admits these two overlap on who owns the weekly cadence. If both fire, FR-004's "no overlapping generation" guard must span *two* services, not just one scheduler. NFR-009 says "disabled at the scheduler level" (singular) — insufficient if two schedulers exist.
12. **PDF size / timeout on large quarter:** NFR-001 bounds serving an already-generated report but not generation duration for a large Q4 dataset; no upper bound or timeout defined for the async job itself.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-010 are all well-formed Gherkin. |
| Document contains only Sections 0–6 | **FAIL** | The document ends with an **"## Appendix: Engineering Reference"** containing an *Affected Services* list and a multi-paragraph narrative *"Cross-check discrepancy (for planning)"* analysis with its own findings. This is a stray multi-paragraph KG cross-check appendix beyond Sections 0–6 — not a sanctioned trailing Revision Summary, not a one-line `> **Label:**` metadata blockquote. It is exactly the kind of extra narrative appendix that must be flagged. Move the affected-services grounding into planning collateral or fold the ownership ambiguity (already present) into Section 1; remove the appendix from the spec. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is entirely bullet items; each tags *System-derived/External* in prose and defers field names/types. No attribute tables present. |

---

## KG Cross-Check (grounding vs. spec claims)

- The spec's handling of the service-ownership discrepancy is **accurate and honest**: it refuses to pick between pubsuite-attribution, summary-report-cloud, and koch-query-api, matching the independent lookup's low/medium confidence spread. No contradiction to flag here.
- **Discrepancy worth noting:** The independent cross-check surfaced **entitlements-api** (gates access to the report feature) and did **not** re-find pubsuite-attribution. The spec's Appendix lists pubsuite-attribution in the *primary* list and relegates entitlements-api to a low-confidence footnote — yet BR-1/NFR-004 make entitlement enforcement central (CG-001). The spec under-weights entitlements-api relative to how load-bearing entitlement is to this feature's viability. Recommend elevating entitlements-api from footnote to a named dependency for planning. (Grounding note only — this does not change the Format verdict above.)

---

## 6. Recommendations

1. **Resolve CG-001 (entitlement) and CG-002 (year-end close date) before any engineering starts.** Both are Critical and either can nullify the MVP. Add a dated availability acceptance criterion once the close date is known.
2. **Remove the Engineering Reference Appendix** (Section 5 format FAIL) — fold its one genuinely spec-relevant point (service-ownership ambiguity, already in Section 1) in and move the rest to planning collateral.
3. **Fix the "last quarter" vs. "Q4" conflation** (Edge Case #1) in BR-3 and FR-002 — define exactly which quarter is reported across the year-end boundary and the controlling timezone. This is the highest-value data-correctness fix given the close-date context.
4. **Set the deferred numeric NFR values** (NFR-001 concurrency, NFR-002 rate cap, NFR-005 link TTL, NFR-009 completion time) so they become testable.
5. **Reconcile dual schedulers** (reporting-dispatcher vs. reporting-subscriptions-worker) — FR-004/NFR-009 must guarantee non-overlap across both services, not one scheduler.
6. **Add staleness surfacing** (Edge Cases #4) and define empty/partial-data generation behavior (#2, #3).
7. **Confirm the audit retention mandate** (NFR-007 [DECISION REQUIRED]) to avoid building a non-compliant 12-month floor.
8. **Specify secure-link behavior across kill-switch/rollback** (Edge Cases #5, #6) to prevent serving known-bad content post-rollback.