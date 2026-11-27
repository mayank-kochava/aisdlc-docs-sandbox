# Spec Critique: Apple Search Ads v5→v1 Platform API Migration

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-27

## Summary

- **Critical Issues:** 2
- **High Issues:** 7
- **Medium Issues:** 6
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **In-flight / partial-sync state at cutover is undefined as a hard requirement.** Section 6 lists "campaigns/keywords existing only in v5 or partially synced at cutover" as an *engineering discovery item*, not a functional requirement. This is a data-correctness dead end: if cutover happens with entities half-synced, the customer could see missing or orphaned campaigns, and no FR governs reconciliation. For a "hard Dec 28 deadline" migration, this is a data-loss/data-integrity risk, not a planning footnote. | Critical | 1, 6 | Add an FR defining the reconciliation/one-time-sync behavior and the acceptance criterion for cutover readiness (e.g. "zero entities in v5-only state at cutover, verified by a reconciliation report"). Do not leave it as discovery-only. | 
| CG-002 | **"Dependency-unavailable" (5xx / total v1 outage) has no defined customer-facing outcome.** NFR-005 *names* dependency-unavailable as a distinct condition, and flow 2.3 covers only "timeout or rate limit." No FR or flow defines what the customer sees when v1 returns a 500/503 or is fully down during a campaign action, bulk upload, or report refresh. The retry bound (NFR-004, 3 attempts) exhausts — then what? This is an unhandled dead end on the primary write path. | Critical | 2.3, 3, NFR-004/005 | Add an FR and a flow branch for "v1 returns 5xx / unavailable after bounded retries": define the customer-facing message, whether the operation is queued/rolled back, and whether the kill-switch (NFR-016) is the intended escape. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-001–FR-006, FR-013, FR-014, FR-022 | "...the same outcome the customer experiences under v5" / "no change in behavior is visible" | "Same as v5" is not independently testable unless a v5 baseline/golden-master is captured. A tester cannot assert equivalence against an API Apple is retiring. | `Given a captured v5 golden-master output for operation X with inputs I, When the same operation runs against v1 with inputs I, Then the customer-facing response (fields, status, latency class) equals the golden-master within defined tolerance T.` Requires an explicit requirement to capture the v5 baseline before cutover. |
| FR-015 / NFR-004 | "SAM handles the condition exactly as it does today" | "Exactly as today" is undefined — today's retry count, backoff, and surfaced message are not documented in-spec. NFR-004 gives a *default* of 3 attempts, which may contradict "exactly as today." | `Given a v1 rate-limit (429) response with Retry-After = R, When SAM retries, Then SAM waits R, retries at most 3 times with exponential backoff, and on exhaustion surfaces <specific, enumerated customer message>.` Pin the actual current values. |
| FR-021 | "...surfaced using the presentation approach proposed by engineering" | Deliberately deferred — acceptable per OOS-4, but still untestable as written. The testable core (distinguish unavailable from zero) is present in FR-020; FR-021 adds nothing testable. | Keep FR-021's hard clause only: `Then the rendered output for an unavailable metric is not equal to any rendering of the numeric value 0.` |
| NFR-003 | "perform no worse than today... p95 ≤ current v5 baseline" | Measurable in principle, but "comparable production load" and the baseline itself are undefined. Not testable until the baseline exists. | `Given the v5 p95 baseline B(op) captured under load profile L, When op runs on v1 under L, Then p95(op) ≤ B(op) × 1.0.` Require B and L to be artifacts. |

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| False-zero defect (NFR-011) | Yes — FR-019, FR-020, FR-023 | **PASS** | Well-covered; hard constraint ("never 0") is testable. Mitigation depends on the renamed/removed-metric inventory, correctly flagged as a blocking discovery item. |
| Multi-service blast radius (NFR-012) | Partial | **FAIL** | Risk is acknowledged and the authoritative service set is deferred to engineering, but **no FR requires each consuming service to be verified against v1 field shapes before cutover.** NFR-012 states the mitigation condition but there is no functional/acceptance requirement enforcing it. See also KG discrepancy in Edge Cases. |
| External API contract change (NFR-013) | Yes — FR-022, NFR-016 | **PASS** | Validation + rollback referenced. |
| Hard Dec 28 deadline vs. unresolved unknowns (NFR-014) | Partial | **FAIL** | Mitigation is "resolve early, else the rollout strategy accommodates it" — but the rollout strategy is itself an *open [DECISION REQUIRED]*. The risk's mitigation depends on an undecided decision. No contingency requirement exists if unknowns are NOT resolved by Dec 28. This is an unmitigated schedule/correctness risk. |
| Dependency-unavailable / v1 outage | No | **FAIL** | Named in NFR-005 but no FR defines the outcome (see CG-002). Listing the condition is not mitigating it. |
| In-flight entity state at cutover | No | **FAIL** | See CG-001. Risk is implied but only recorded as a discovery item, not mitigated by any requirement. |

---

## 4. Missed Edge Cases

1. **v1 total outage / 5xx on a write operation:** No defined behavior after retries exhaust (CG-002). What does the customer see; is the campaign action lost, queued, or rolled back?
2. **Entities v5-only or partially synced at cutover:** No reconciliation requirement (CG-001). Risk of orphaned or disappearing campaigns.
3. **Bulk upload where v1 rejects the *entire* batch (not per-keyword):** FR-008 covers individual rejections. What if v1 rejects the whole request (auth failure, malformed batch, quota exceeded mid-batch)? Partial-success semantics don't apply; undefined.
4. **Bulk upload partial completion due to timeout mid-batch:** If v1 times out after processing 300 of 1000 keywords, is the result a partial success, a retry of the whole batch, or a duplicate upload of the first 300? Interacts with idempotency — not addressed for uploads (only NFR-007 covers report refresh).
5. **Bulk upload idempotency:** NFR-007 guarantees idempotency only for report refresh. A retried bulk upload could double-create keywords. No requirement.
6. **v1 renames a metric but with a changed semantic/unit** (e.g. spend in micro-units, or ROAS as ratio vs. percent): FR-019 maps the *name* back but assumes value semantics are identical. A renamed-and-redefined field would display a correct-looking but wrong value — a silent correctness bug worse than a false zero.
7. **v1 *adds* a new metric with the same name as an existing v5 field but different meaning:** Mapping table assumes v1→v5 is a subset/rename; collision is not considered.
8. **Rate-limit during the 4-hour refresh causing a missed window:** NFR-002 says "stagger pulls," but what if a refresh cannot complete within its window? Does the report show stale data, partial data, or skip? Interacts with NFR-001 "completeness of a refresh cycle" but no FR defines the degraded outcome.
9. **Concurrent kill-switch flip mid-operation:** NFR-016 reverts "new operations going forward," but a bulk upload or refresh in flight when the switch flips has undefined behavior.
10. **Clock/`Retry-After` absent on a 429:** NFR-004 says honor Retry-After "where v1 returns" it — the absent case falls back to "fixed schedule"? Not specified; the exponential-backoff default may or may not apply.
11. **Auth token expiry / refresh failure for v1:** NFR-008/009 cover secret handling and access control, but token-refresh failure as an operational condition (distinct from rate-limit/timeout) is not a defined failure mode.
12. **Automated bid/budget adjustment firing during a v1 rate-limit storm:** FR-014 + FR-015 interact, but whether a dropped automated adjustment is retried later or silently skipped (with customer-visible budget impact) is undefined.
13. **Rejection reason string containing PII/internal identifiers:** NFR-010 requires sanitization, but no FR defines the sanitization rule or what happens if a reason cannot be sanitized to a presentable form (shown blank? generic?).

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-023 are all in Gherkin blocks. Interstitial prose notes (the `[DECISION REQUIRED]` note before FR-012, the note after FR-021, and the trailing "Now reporting the single business trade-off…" line) are annotations, not FRs stated as prose. |
| Document contains only Sections 0–6 | PASS | Document runs Sections 1–6 (no Section 0 present, which is a content gap, not an extra-section violation). No appendices or extra narrative sections beyond the template. Section 1's "Known Scope Note" is a subsection of Section 1, not a stray appendix. No Revision Summary or metadata trailer present. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is entirely a bullet list; each item carries a source/rationale narrative but no attribute tables or type declarations. |

> Note (not a format FAIL): Section 0 (Overview/Metadata) appears to be absent — the document starts at Section 1. If the template mandates a Section 0, flag as a content completeness item, but it is not an *extra*-section violation.

---

## 6. Recommendations

1. **Resolve CG-001 (cutover reconciliation):** Add a functional requirement and a go/no-go acceptance criterion for v5-only / partially-synced entities before Dec 28. This is a data-integrity blocker.
2. **Resolve CG-002 (dependency-unavailable path):** Add an FR and flow branch defining the customer-facing outcome when v1 returns 5xx / is unavailable after bounded retries, and tie it to the kill-switch.
3. **Capture a v5 golden-master baseline:** Make "record the v5 baseline (behavior + p95 latency) before cutover" an explicit requirement. Without it, every "same as v5" FR and NFR-003 is untestable.
4. **Close the NFR-014 contingency gap:** Define what happens if the field-mapping / error-format / rate-limit unknowns are NOT resolved by Dec 28. The mitigation currently depends on an undecided rollout decision — force the rollout-strategy [DECISION REQUIRED] to be made, with a documented fallback.
5. **Add bulk-upload idempotency and whole-batch/mid-batch-failure requirements** (Edge Cases 3–5) to match the idempotency guarantee NFR-007 gives reporting.
6. **Strengthen FR-019 to cover semantic/unit changes, not just renames** (Edge Case 6) — a renamed-and-redefined metric is a silent correctness bug the spec's own "no false zero" philosophy should extend to.
7. **Make NFR-012 enforceable:** Add an acceptance requirement that every confirmed consuming service is verified against v1 field shapes before cutover. **Reconcile the service list with the KG evidence first (see below).**

### KG cross-check note (grounding for Recommendation 7)

The independent KG lookup used to ground this review **contradicts the spec's Section 1 scope note in a material way** and the author should reconcile before planning:

- The spec's Section 1 states the independent lookup did **not** re-find "the dedicated consumer/agent service" (sam-agent) or "the frontend display layer." The KG evidence provided here **does** list `sam-agent` (role: reads avgCPT/avgCPM, verify field shapes, no code change) — so the spec's claim that the fresh lookup dropped sam-agent is inconsistent with the evidence. NFR-015 explicitly names sam-agent as a backward-compat consumer, so it should not be described as "not re-found."
- The KG evidence surfaces **ten** backend services touching Apple Ads data (`sam-mono`, `sam-api`, `sam-campaign-data-sync`, `k4a-sam-campaign-data-sync`, `k4a-sam-backend`, `sam-performance-reporter`, `sam-mmp-ingestion`, `sam-bulk-actions`, `sam-keyword-search-optimization`, `sam-agent`), versus the spec's informal "6+ services" (NFR-012). The blast radius is likely **wider** than the spec's framing; confirm the authoritative set and update NFR-012's count.
- No KG evidence corroborates a **frontend display layer** in scope — the evidence is entirely backend. The spec's claim that the original analysis identified a frontend layer should be verified, since BR-2/FR-022 assume customer-facing screens are affected yet no frontend service appears in the KG.

These are reconciliation items, not format failures, but they bear directly on whether NFR-012's mitigation is complete.