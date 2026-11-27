# Spec Critique: Apple Ads v5→v1 API Migration (Search Ads Maven)

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-27

## Summary

- **Critical Issues:** 2
- **High Issues:** 6
- **Medium Issues:** 7
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **Section 7 Appendix contradicts the spec's own affected-services narrative on confidence levels, which could mislead the planning cut.** The spec body (Section 1 / NFR-008) treats `k4a-sam-backend` as the *high-confidence core migration-implementing backend* and `sam-mono`/`sam-api` as ambiguous. The independent KG cross-check reverses this: `sam-api` (high) and `sam-mono` (high) are the high-confidence Apple Ads callers, while `k4a-sam-backend` is only **medium** confidence as an Apple Ads caller. The spec builds its entire "core lift" framing and NFR-008 blast-radius mitigation around a service the evidence ranks below two others it demotes. If planning anchors on `k4a-sam-backend` as "the core," call sites in `sam-api`/`sam-mono` can be missed — directly violating FR-002 ("no residual v5 dependency") and the decommission precondition in NFR-008. | Critical | 1, 7, NFR-008 | Reconcile the confidence ordering with the KG evidence: `sam-api` and `sam-mono` are the highest-confidence Apple Ads callers; `k4a-sam-backend`'s caller status is only medium and unconfirmed. Make "locate all Apple Ads call sites across ALL listed repos" a hard, gating precondition (not merely "flagged for planning") before any v5 decommission. |
| CG-002 | **No defined behavior when the kill-switch is flipped to "prior path" but the prior path (v5) is already decommissioned by Apple.** FR-022 / NFR-011 make rollback depend entirely on reverting to the prior v5 integration. After Dec 28 2026, v5 is *gone* — the rollback target no longer exists. The spec's only incident mitigation becomes a dead end the moment the deadline it is built around passes. This is a hard dead-end in the primary (and only) rollback flow. | Critical | FR-022, NFR-011, NFR-012 | Add an FR defining incident behavior once v5 is decommissioned and rollback is no longer possible (e.g. forward-fix-only mode, degraded-but-safe read behavior, explicit "rollback window closes on date X" with an alternate containment plan). Tie the soak/variance gate (DEC-003/NFR-012) explicitly to "all validation must pass *while v5 still exists* as a fallback." |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "must not **materially degrade** operator-perceived latency… p95 no worse than the current v5 p95" | "Materially" and "representative concurrent load" are untestable until DEC-004 sets a tolerance and the load profile is fixed. No numeric gate. | Given the v5 p95 baseline is recorded per operation class under a fixed load profile L, When the v1 path runs the same operation class under L, Then the v1 p95 is ≤ (v5 p95 + agreed tolerance T), where T is set by DEC-004. |
| NFR-003 | "rate-limit (429) responses exceed **a threshold** sustained over the same window" | The 429 alert threshold is a literal unspecified blank ("a threshold"). Cannot write the alert test. | Given outbound v1 calls are monitored, When 429 responses exceed N% over 5 minutes, Then an alert fires — define N. |
| NFR-003 | "Alerting on a **spike** in field-mapping errors must be included" | "Spike" is not quantified. | Given field-mapping-error records are emitted, When field-mapping errors exceed M occurrences (or M% of maps) over window W, Then an alert fires — define M, W. |
| FR-017 | "a duplicate bulk submission for the same operation… from the same session" | "Same operation" and "same set of entities" are not defined precisely enough to test the dedup key; cross-session duplicates are silently out of scope. | Specify the exact dedup key (entity IDs + operation type + actor). Add scenario for duplicate from a *different* session. |
| FR-018 | "no duplicate record is created and no side effect is double-counted" | "Side effect" is undefined — untestable without naming the concrete side effects (audit rows, metric counters, Apple-side writes). | Enumerate the side effects guarded: Apple-side write, audit-trail row, bulk-outcome counter. |
| NFR-013 | "a v1-driven field change must not **silently** break a downstream consumer" | "Silently" has no detection criterion. | Given a v1 field change alters an internal contract field, When a downstream consumer reads it, Then a contract-validation check fails loudly at build/deploy or the consumer is updated in lockstep — define the detection mechanism. |

*Note: all FR-001 through FR-022 are written in valid Gherkin — no prose FRs found. NFRs are intentionally prose per the template and are not required to be Gherkin, but the ones above are flagged because their acceptance criteria are not measurable.*

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard Dec 28 2026 deadline (NFR-007) | Yes — parity-only scope + FR-015 field validation; DEC-005 for safe cutover date | PASS | Mitigation is concrete and tied to requirements. |
| Multi-service blast radius (NFR-008) | Partial — kill-switch (FR-022) given, but "resolve naming/duplication before decommission" is only *flagged*, not a gated requirement | **FAIL** | No FR *requires* a completed call-site inventory as a decommission gate. Combined with CG-001, this is the highest-probability path to a missed v5 call site. Add an explicit gating FR. |
| Authentication / external API surface (NFR-009) | Yes — FR-004/005/006, named failure classification | PASS | Well covered. |
| Silent reporting/optimization corruption (NFR-010) | Yes — FR-010/014/015 forbid silent substitution, require field-level validation | PASS | Strongest-covered risk. Good. |
| Rollback is HIGH-risk (NFR-011) | Yes, but mechanism assumes v5 still exists | **FAIL** | See CG-002 — rollback target vanishes at the deadline; no post-decommission containment defined. |
| Parallel-run / soak (NFR-012, DEC-003) | Deferred to decision; engineering default stated | PASS (conditional) | Acceptable as a flagged decision, but the variance-gate threshold is undefined and must be set before cutover. |
| Transition-period dual auth (note under FR-006) | No specific requirement — deferred to DEC-003/FR-021 | **FAIL** | Whether v5+v1 auth must coexist during soak is unresolved AND has no requirement. If NFR-012's parallel-soak default is adopted, dual auth becomes mandatory but is unspecified. Add a conditional FR. |

---

## 4. Missed Edge Cases

1. **Post-decommission incident with no rollback target:** (see CG-002) v5 gone → kill-switch "prior path" points at nothing.
2. **Kill-switch flipped mid-bulk-operation:** FR-022 says in-flight requests complete under original behavior, but a *bulk* operation is a multi-item batch spanning time. Is a half-processed batch "in flight" as one unit or per item? Undefined — risks partial batches with mixed v5/v1 behavior.
3. **Idempotency across a kill-switch flip:** FR-018 idempotency assumes one integration path. A retry that first hit v1, then (post-flip) hits the prior path — is the end state still idempotent across *different* backends? Not addressed.
4. **Apple v1 returns a field that is *present but semantically redefined* (same name, different meaning/units):** FR-010/FR-011 catch missing/renamed/enum changes, but a same-name field whose unit or semantics changed (e.g. micros vs. currency, UTC vs. local) passes every existing check and silently corrupts reports — the exact NFR-010 risk, uncaught.
5. **Cross-session duplicate bulk submission:** FR-017 only guards "same session." Two operators (or two sessions) submitting the same bulk op concurrently is uncovered.
6. **Pagination cursor expiry/invalidation mid-retrieval:** FR-020 handles multi-page retrieval but not a cursor that expires or a data set that changes between pages (records added/removed → drops or dupes despite "no records dropped" claim).
7. **429 on the *final* retry of a bulk item vs. the batch:** FR-019 bounded retry and FR-016 per-item tracking don't specify whether retry budget is per-item or per-batch; a large batch could exhaust global retry budget on early items.
8. **Auth token refresh failure mid-operation (not initial auth):** FR-004 covers refresh behavior and FR-005 covers auth failure, but a token expiring *during* a long paginated/bulk operation (refresh needed mid-flight) is not explicitly handled.
9. **Audit-write failure during rollback incident:** NFR-004 says audit-write failure fires an alert, but during an active incident (the time rollback happens) audit infrastructure may also be degraded — no fallback for audit durability.
10. **Partial field availability — field exists in v1 for some entity types but not others:** FR-015 validates "each currently-consumed metric field" but not conditional availability (present on campaigns, absent on keywords).
11. **Clock/timestamp differences between v5 and v1 reporting windows:** Reporting parity (FR-014) assumes equivalent time semantics; v1 may use different reporting-window boundaries or timezones, producing legitimately different numbers that FR-014 would flag as "silently wrong" (false positive) or miss (false negative).
12. **Service-naming duplication resolves to *two real* services, not one:** If `sam-campaign-data-sync` and `k4a-sam-campaign-data-sync` are genuinely two services (not an artifact), the migration must cover both; spec treats it as "unclear" without a requirement to cover both if confirmed distinct.
13. **`sam-api` or `sam-bulk-actions` is a *direct* Apple Ads caller:** The KG evidence lists `sam-api` as high-confidence Apple Ads caller. If it calls Apple directly (not via `k4a-sam-backend`), the migration scope and FR-002 coverage expand — no requirement forces verifying direct-vs-indirect caller status before decommission.
14. **Kill-switch propagation window (5 min) vs. incident severity:** NFR-011 targets a 5-minute revert, but a silent-data-corruption incident may run for the full 5 minutes producing bad data at every operation — no "fast-path" for the highest-severity corruption case.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-022 are all valid Gherkin. No prose FRs found. |
| Document contains only Sections 0–6 | **FAIL** | The document contains a **"## Appendix: Engineering Reference"** after Section 6. This is a multi-paragraph narrative appendix with its own per-service findings and planning recommendations ("should be settled early in planning…") — exactly the kind of stray narrative-analysis appendix the format rule is meant to exclude, not a sanctioned trailing Revision Summary and not a one-line `> **Label:**` metadata blockquote. It duplicates the Section 1 reconciliation content as prose analysis and belongs in a separate engineering doc. (Note: no Revision Summary or labeled-metadata trailer is present, so no exception applies here.) |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is entirely bullet-list items with prose descriptions and Source/Matters notes. No attribute tables or type declarations. |

---

## 6. Recommendations

1. **(Critical — CG-002)** Define post-decommission incident/rollback behavior. The kill-switch rollback target (v5) ceases to exist at the Dec 28 deadline; add an FR for forward-fix/containment mode once v5 is gone, and gate all soak/variance validation to complete *while v5 still exists*.
2. **(Critical — CG-001)** Reconcile affected-services confidence with the KG evidence: `sam-api` and `sam-mono` are the highest-confidence Apple Ads callers, not `k4a-sam-backend` (medium). Convert "locate all Apple Ads call sites across all repos" and "resolve naming/duplication" from *flagged* notes into a **hard gating requirement** that must pass before any v5 decommission (closes the NFR-008 FAIL).
3. **(High)** Add an FR for same-name/semantically-redefined v1 fields (units, timezone, reporting-window changes) — the biggest uncaught silent-corruption vector (Edge Cases 4 & 11), since FR-010/011 only catch missing/renamed/enum changes.
4. **(High)** Remove or relocate the Engineering Reference Appendix to fix the Format Compliance FAIL; fold any essential call-site facts into Section 1.
5. **(High)** Resolve the transition-period dual-auth gap (Risk Audit FAIL): add a conditional FR for simultaneous v5+v1 authentication if the parallel-soak default (NFR-012) is adopted.
6. **(High)** Quantify all unspecified alert thresholds in NFR-003 (429 threshold "N", field-mapping spike "M/W") and the NFR-012 variance gate; none are currently testable.
7. **(Medium)** Specify kill-switch behavior for in-flight *bulk* operations and idempotency across a path flip (Edge Cases 2 & 3).
8. **(Medium)** Broaden FR-017 to cover cross-session/concurrent duplicate bulk submissions.
9. **(Medium)** Address pagination cursor expiry and inter-page data mutation in FR-020 (Edge Case 6).
10. **(Medium)** Set the NFR-001 latency tolerance (DEC-004) and fix the load profile so the parity gate becomes measurable.