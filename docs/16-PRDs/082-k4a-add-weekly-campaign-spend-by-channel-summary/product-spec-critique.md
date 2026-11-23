# Spec Critique: Meridian Travel Weekly Campaign Spend-by-Channel Summary

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-23

## Summary

- **Critical Issues:** 2
- **High Issues:** 6
- **Medium Issues:** 5
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **FR-010 has no body.** The FR is titled "Eligibility re-derived from authoritative source" but contains no Gherkin block — there is literally nothing under the title. NFR-004/NFR-005 reference FR-010 as the authoritative, fail-closed re-derivation requirement, so the single most security-critical behavior in the spec is undefined. Engineering cannot implement an empty requirement. | Critical | 3 (FR-010) | Author the full Gherkin for FR-010, including the fail-closed path on unreachable auth source (align with NFR-005): Given a request for the summary, When eligibility is evaluated, Then it is re-derived from the authoritative account/auth source on every request, And if that source is unreachable the request is denied (403). |
| CG-002 | **Service ownership is genuinely unresolved and contradicts between two KG lookups AND a third independent cross-check.** The spec's own Section 1 / NFR-010 admit the owning backend stack is unknown (`reporting-api` cluster vs. `sam-*` cluster vs. `k4a-sam-backend`). An independent cross-check re-confirms the `sam-*` / `k4a-sam-backend` cluster and does NOT re-surface `accounts-api` or `reporting-subscriptions-worker` at all. With the owning stack, the scheduler, the cost source, and the plan-check service all unconfirmed, no FR can be sized or built. This is flagged in Section 1 per instructions rather than silently resolved. | Critical | 1, NFR-010, Appendix | Engineering must reconcile ownership BEFORE scope lock. In particular: the plan-check path (`accounts-api`) that FR-009/FR-010/NFR-004 depend on appears in only one lookup and is not re-found by the cross-check — the eligibility enforcement point may not exist where assumed. Confirm the authoritative scheduler, cost-aggregation, and plan-check services and record the single resolved owner. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-002 | "the scheduled generation time for the week is reached" | "Scheduled generation time" is undefined — DEC-004 (refresh timing + timezone) is open, so there is no testable trigger moment or timezone. A test cannot assert *when* generation should fire. | Given the configured weekly generation time of {TIME} in timezone {TZ}, When the system clock reaches that time, Then a new summary for the covered week is generated. (Blocked on DEC-004 — cannot write a concrete test until resolved.) |
| FR-004 | "the second run does not start concurrently / the in-progress run is allowed to complete" | Says what must NOT happen but not the observable outcome of the rejected run — is it dropped, queued, or errored? Untestable as to the second run's fate. | Given a generation run for period P is in progress, When a second run for P is triggered, Then the second run is rejected/no-ops without starting, And exactly one run completes for P, And the rejection is recorded (not silently dropped). |
| FR-008 | "the underlying spend data source is unavailable at generation time" | "Unavailable" is one lumped condition, but NFR-008 distinguishes unavailable vs. timeout vs. rate-limited as distinct statuses. FR-008 does not enumerate them, so the error-state tests don't map to the three statuses the NFR mandates. | Split into scenarios: Given the source returns {unavailable|timeout|429}, When generation runs, Then status {SOURCE_UNAVAILABLE|TIMEOUT|RATE_LIMITED} is recorded, And the FR-008 error state is shown, And the last-known summary remains viewable. |
| NFR-001 | "measured under expected concurrent K4A load" / "the view's expected peak concurrency" | "Expected" concurrency is never quantified. p95 ≤ 2000 ms is only testable against a stated concurrency number. | Given {N} concurrent summary-view requests, When the view is loaded, Then p95 response time ≤ 2000 ms. (Author must supply N.) |
| NFR-002 | "state an explicit rate cap on the aggregation query path" | The NFR instructs that a rate cap be stated but never states the number. A guardrail with no value is not enforceable or testable. | Given the aggregation query path, When generation runs, Then query rate must not exceed {X} queries/sec against the shared cost source. (Author must supply X.) |

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Multi-service blast radius / unclear ownership (NFR-010) | Partial — reconciliation named but deferred | FAIL | "Mitigated only when engineering confirms…" is a dependency, not a mitigation. No fallback if reconciliation slips past scope lock. Combined with the empty FR-010 (CG-001) and the vanished `accounts-api` (CG-002), the eligibility path is unmitigated. |
| Channel may not be first-class in source data (NFR-011) | Partial — confirmation named, no fallback | FAIL | NFR-011 says scope "expands materially" if channel isn't first-class, but defines no contingency FR or de-scope plan. No requirement covers the case where some campaigns have NO channel attribution (null channel). |
| ~5-week fixed deadline (NFR-012) | Yes — scope locking + out-of-scope list | PASS | Reasonable; depends on open decisions resolving early. |
| Net-new UI with no anchor component (NFR-013) | Yes — early UX delivery | PASS (as a plan) | Not a testable requirement, but an acceptable schedule mitigation. |
| Spend-data sensitivity (NFR-006) | Partial — no-log rule stated; viewer-restriction unresolved (DEC-006) | FAIL | "Who may view spend by channel" is undecided AND out-of-scope list says build no gate — these two must be reconciled. If DEC-006 lands on "restricted," Phase 1 ships without the required gate. |
| Shared cost-data query degradation (NFR-002) | Partial — cap mandated but unquantified | FAIL | No actual rate cap value (see Testability). Guardrail is not enforceable as written. |
| Downstream dependency failure (NFR-008) | Yes — bounded retry, backoff, distinct statuses | PASS | Well specified. |

---

## 4. Missed Edge Cases

1. **Null / unattributed channel:** FR-001 assumes every campaign has a channel. No FR defines behavior when a campaign's channel is null, empty, or unknown. Does its spend get dropped (undercount), bucketed to "Unknown," or error the run? This is a correctness risk for the headline number.
2. **Negative / refunded / adjusted spend:** Credits, refunds, or negative cost adjustments are not addressed. Can a per-channel total go negative? Is that displayed or clamped to zero?
3. **Currency:** Spend figures have no stated currency or multi-currency handling. If campaigns span currencies, summing them is meaningless. No requirement covers currency normalization or display.
4. **Partial-week / week-boundary spend:** How is spend assigned when a campaign straddles the week boundary, especially given the unresolved DEC-004 timezone? Spend near midnight on the cutoff could land in either week.
5. **New channel appearing mid-week:** FR-001 produces "one total per channel," but behavior for a channel with campaigns but zero spend, or a channel appearing for the first time, is undefined (shown with zero, or omitted?).
6. **Stale last-known summary indistinguishable from fresh:** FR-008 keeps the prior week's summary viewable on failure, but no FR requires the UI to label it as stale or show which week it covers. A user could mistake last week's numbers for this week's — a decision-quality/data-integrity risk during a high-stakes campaign.
7. **Eligibility flips mid-session:** FR-009 enforces per request, but no scenario covers an account that becomes ineligible between page load and a subsequent fetch. Does the already-rendered summary persist on screen?
8. **Kill-switch flipped while a user is viewing:** NFR-009/NFR-014 cover new requests and in-flight generation, but not a user already on the summary screen when the feature is disabled. Does their next refresh show "unavailable"?
9. **First-ever week (no prior summary) + generation failure:** FR-008's graceful degradation relies on a prior summary existing. The very first run failing has no last-known fallback — the error-state-only path is implied but not explicitly specified.
10. **Empty vs. error ambiguity when source returns zero rows:** Flow chart distinguishes "source unavailable" from "no spend," but no FR defines how the system tells a legitimate zero-spend week apart from a silently empty/partial source response. A degraded source returning an empty payload could be misrendered as a valid empty state.
11. **Timezone of the dd-mm-yyyy date:** FR-006 enforces the format but not the timezone the week start/end dates are computed in — tightly coupled to the unresolved DEC-004. Two users in different timezones could see different week boundaries.
12. **Concurrent read during regeneration:** FR-003 overwrites "in place." No FR guarantees a reader during an in-place overwrite sees either the old or new summary atomically, not a half-written mix.
13. **Retry exhaustion outcome:** NFR-008 caps at 3 attempts but no FR states the terminal behavior after all retries fail beyond "error state" — e.g., does it alert (NFR-003) and stop, or keep retrying next cycle?

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | FAIL | **FR-010 has no Gherkin body at all** — only a title. It is neither Gherkin nor prose; it is empty. (See CG-001.) FR-001–FR-009 are correctly in Gherkin. |
| Document contains only Sections 0–6 | FAIL | The document ends with an "**Appendix: Engineering Reference**" containing multi-paragraph narrative "Affected Services" analysis and a "Discrepancy to reconcile" prose block with its own findings. This is exactly the kind of stray multi-paragraph narrative appendix the format rule is written to catch — it is NOT a sanctioned Revision Summary nor a short one-line `> **Label:**` metadata trailer. Remove or relocate the appendix out of the spec; its ownership discrepancy belongs in NFR-010 / Section 1 (where it is already referenced). |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a pure bullet list; each item is prose describing the datum, source, and why it matters, with no attribute tables or type declarations. |

---

## 6. Recommendations

1. **Write FR-010.** The spec's most security-critical requirement is an empty title. Author its full Gherkin, including the fail-closed branch, before any eligibility work begins. (CG-001)
2. **Reconcile service ownership before scope lock.** Resolve the three-way lookup disagreement — in particular confirm whether the plan-check service (`accounts-api`) that FR-009/FR-010 depend on actually exists in the owning stack, since the independent cross-check does not re-surface it. (CG-002)
3. **Resolve DEC-004 (refresh timing + timezone).** It blocks testable triggers for FR-002, the week-boundary edge cases (#4, #11), and the dd-mm-yyyy date computation.
4. **Quantify all numeric guardrails.** Supply concrete values for NFR-001 (expected concurrency N) and NFR-002 (aggregation rate cap X); both are currently unenforceable.
5. **Define null/unknown-channel and negative-spend behavior.** These are correctness risks for the headline number (edge cases #1, #2) and intersect with the NFR-011 channel-availability risk.
6. **Address stale-summary labeling.** Require FR-008's retained last-known summary to be visibly marked as stale with its covered week (edge case #6) to prevent wrong decisions during the campaign.
7. **Remove the Engineering Reference appendix from the spec body** and fold its reconciliation note into Section 1 / NFR-010 to satisfy the Sections 0–6 constraint.
8. **Specify currency handling** (edge case #3) before aggregation is built.
9. **Reconcile DEC-006 vs. the "build no gate" out-of-scope item** so Phase 1 does not ship without a viewer-restriction that data-sensitivity turns out to require.