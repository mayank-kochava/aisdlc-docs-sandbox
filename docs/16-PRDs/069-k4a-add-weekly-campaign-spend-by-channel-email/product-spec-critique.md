# Spec Critique: Meridian Travel Weekly Spend Summary

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
| CG-001 | The spec contains **two separate "Appendix: Engineering Reference" / KG content locations** that disagree about ownership: Section 1's "Known Scope Ambiguity" paragraph and the trailing Engineering Reference appendix both describe unresolved service ownership, but neither commits to a single owner for scheduling, email send, or authoritative spend source. Because FR-001/FR-012/FR-004 all depend on exactly one owner, engineering cannot start the data-retrieval or dispatch work without a dead end. However, the spec correctly defers this to a discovery spike and flags it, so it is a High (not Critical) planning blocker rather than a logic dead-end in the defined behavior. | High | 1 / Appendix | Resolve the discovery spike (single owner for scheduling, send, and spend source) before FR-004/FR-012 implementation; record the decision in the spec. |
| CG-002 | FR-006 (reconciliation) references "an agreed tolerance" and NFR-010 references "per-channel variance … as the explicit exit gate" but neither the tolerance value nor the soak duration is defined (DEC-006 open). Launch readiness is therefore ungated by any concrete number — the "business-critical accuracy" rule (BR-3) has no pass/fail threshold. | High | 3 / 5 | Define the numeric reconciliation tolerance (e.g. 0% exact, or ±$X / ±Y%) and the soak-period length as part of DEC-006 before launch gating. |
| CG-003 | The flow and FRs cover delivery retry (FR-013) and partial data (FR-008–010), but there is **no defined behavior for a run that passes reconciliation offline yet produces figures that later prove wrong in production** (e.g. upstream recalculates spend retroactively after the email is sent). NFR-011 states already-delivered emails are not recalled — but no requirement defines detecting or correcting a *silently-wrong* delivered number, which is exactly the failure BR-3 says is "worse than the manual process." | High | 2 / 3 | Add an FR covering post-delivery discrepancy detection / correction-notice behavior, or explicitly accept the risk in Out of Scope. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "completes well within its delivery window" | "well within" is not measurable; the concrete 15-min p95 target follows but the prose lead-in is vague | Given a weekly run under normal data volume, When the scheduled trigger fires, Then the end-to-end run completes within 15 minutes (p95 across runs). |
| FR-006 | "match the manual figures within an agreed tolerance" | "agreed tolerance" is undefined — not testable | Given the manual report for week W produces authoritative per-channel totals, When the automated summary for week W is generated, Then each automated per-channel total differs from the manual figure by no more than [DEC-006: define tolerance], And any larger discrepancy is surfaced and blocks delivery. |
| FR-003 | "produces the same end result" | "same end result" is not precisely defined (same figures? same email? same message-id?) | Given a summary for week W has been generated, When the job re-runs for week W, Then the per-channel totals and reporting period are byte-identical to the first run, And no second email is dispatched for week W. |
| FR-007 | "not silently omitted in a way that could be mistaken for missing data" | "mistaken for" is subjective — no observable assertion | Given a channel in the confirmed set has zero recorded spend, When the email is composed, Then that channel appears as a row with value 0, visually distinct from any channel marked "data unavailable". |
| FR-015 | "across multiple backend services" is prose context inside Gherkin | Mixing narrative into Given; "within 5 minutes of the configuration change" is testable (good) but the propagation across the unknown service set is untestable until ownership is resolved | Given the kill-switch config is set to disabled, When the change propagates, Then no new run starts after the change, And new runs stop within 5 minutes, measured across all services that initiate runs. |

Note: the stray sentence **"Now I'll report the decisions I flagged in Sections 0–3."** appears at the end of Section 3 — this is leaked authoring/agent narration, not a requirement, and must be removed. See Section 5.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard 5-week deadline before live campaign (HIGH) | Yes — discovery spike + reconciliation buffer | PASS | Mitigation is a plan, not a requirement, but acceptable; deadline achievability hinges on DEC-006 timing. |
| Affects 5 backend services (HIGH) | Yes — single-owner choice + kill-switch (FR-015) | PASS (conditional) | Mitigation depends on the unresolved ownership spike; until resolved the "single owner" is aspirational. |
| Data accuracy business-critical (HIGH) | Partial — FR-006/BR-3 + partial-data policy | FAIL | Reconciliation tolerance is undefined (DEC-006) and post-delivery wrong-number detection is absent (CG-003). The risk is acknowledged but the mitigation has no testable threshold. |
| Shared upstream query load (NFR-002) | Yes — explicit rate cap | PASS | Exact cap deferred to eng-plan; acceptable for weekly single-tenant batch. |
| Legacy/multi-consumer dependency regression (NFR-012) | Yes — additive, no-regression requirement | PASS | Needs a regression-test assertion in eng-plan but requirement is present. |
| Unresolved service ownership / possible reuse path | Documented, not mitigated | FAIL | Reuse-vs-build (reporting-subscriptions-worker / sam-performance-reporter) is flagged but no requirement forces the spike to conclude before build commit. |

---

## 4. Missed Edge Cases

1. **Retroactive upstream spend correction:** Upstream recalculates a prior week's spend after the email is sent — no requirement for detection or a correction notice (see CG-003).
2. **Timezone / DST ambiguity on the delivery schedule:** "fixed recurring day and time" (BR-1, FR-001) does not specify a timezone. A DST shift could move the Monday send by an hour or across a day boundary.
3. **Partial reconciliation failure during soak:** NFR-010 gates cutover on variance but does not define behavior if variance is *intermittent* (passes some weeks, fails others) during the soak.
4. **Empty confirmed channel set:** FR-005/FR-007 assume a confirmed channel set exists; no behavior defined if the set is empty or misconfigured at run time.
5. **Recipient list partially invalid:** NFR-008 halts on *empty/unresolved* list, but not on a *partially* invalid list (e.g. 3 of 5 addresses bounce). FR-013 handles delivery failure generically but not per-recipient partial bounce.
6. **New channel appears in data but not in confirmed set:** FR-005 groups by the *confirmed* set; spend attributed to an unknown/new channel has no defined home — silent drop risk.
7. **Email size / rendering limits:** If a future content profile (BR-5) adds per-campaign breakdowns, no requirement bounds email size or defines truncation/attachment behavior.
8. **Clock skew / scheduler double-fire across services:** FR-002 prevents overlapping runs "for the same period" but the dedup key depends on the yet-unresolved scheduling owner; two scheduling candidates (sam-report-scheduler vs reporting-dispatcher) could each fire.
9. **Audit write succeeds but email figures were wrong:** FR-016 records *delivery status* but not a *content hash* of what was delivered, limiting post-incident traceability of exactly which numbers went out.
10. **Entitlement change mid-cycle:** Meridian Travel downgrades/loses entitlement (BR-7) between trigger and dispatch — no defined behavior for in-flight runs when eligibility is revoked.
11. **Daylight of a skipped week:** If a run is skipped (holiday, maintenance), no requirement defines whether the next run covers the gap week or whether the missed week is simply lost.

### KG Cross-Check Findings (spec claims vs. independent KG lookup)

- **Spend source of truth contradiction:** Section 4 ("Campaign spend by channel … Source: SAM-side campaign spend sources") and the appendix name **sam-api** as the candidate spend source (confidence medium in the appendix). The independent KG lookup, however, rates **sam-api** only *low* confidence and rates **k4a-sam-backend** / **sam-performance-reporter** *medium*. The spec's confidence ranking does not match the fresh lookup — flag for reconciliation rather than adopting the spec's `sam-api`-centric framing. [Section 1 / 4]
- **Possible existing report path treated as "possible" when KG also finds it:** The appendix flags **reporting-subscriptions-worker** and **sam-performance-reporter** as potential reuse; the independent lookup independently re-finds both. This strengthens the reuse-vs-build decision — the spec should elevate this from "investigate" to a required pre-build decision, since both sources agree the path may already exist.
- **No contradiction found** on email-send ambiguity: both the spec appendix and the KG lookup list `notifications-api` vs `reporting-dispatcher` as overlapping send candidates — consistent, but unresolved.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-016 are all in Gherkin. Note: stray narration sentence "Now I'll report the decisions I flagged in Sections 0–3." is embedded at the end of Section 3 — it is not an FR, so this check passes, but the stray sentence must be deleted (flagged below). |
| Document contains only Sections 0–6 | FAIL | The trailing **"Appendix: Engineering Reference"** is a multi-paragraph narrative analysis section with its own findings and a "highest-leverage de-risking step" recommendation. This is exactly the kind of stray multi-paragraph KG cross-check appendix that is NOT exempt (it is not a Revision Summary and not a short one-line `> **Label:**` metadata blockquote). It belongs in eng-plan, not the product spec. Additionally, the stray authoring sentence inside Section 3 ("Now I'll report the decisions I flagged in Sections 0–3.") is leaked content that must be removed. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is entirely bullet-list items describing what/why with no attribute tables or type declarations. |

---

## 6. Recommendations

1. **Define the reconciliation tolerance and soak exit gate (DEC-006)** with concrete numbers — this is the gate for the "business-critical accuracy" risk and currently has no testable threshold (CG-002, Risk Audit FAIL).
2. **Resolve the service-ownership discovery spike** (scheduling owner, email-send owner, authoritative spend source) before FR-004/FR-012 build; reconcile against the KG lookup, which disagrees with the spec on `sam-api` confidence and strengthens the reuse case for `reporting-subscriptions-worker` / `sam-performance-reporter` (CG-001, KG findings).
3. **Add a post-delivery wrong-number detection/correction requirement** (CG-003) or explicitly accept the risk in Out of Scope.
4. **Remove the trailing "Appendix: Engineering Reference" section** from the product spec (move to eng-plan) and delete the stray narration sentence in Section 3 to restore Sections 0–6 compliance.
5. **Tighten vague prose** in NFR-001, FR-003, FR-006, FR-007 per the Gherkin rewrites in Section 2.
6. **Add FRs / clarifications for the missed edge cases**, prioritizing timezone/DST (#2), unknown-channel spend (#6), partial recipient bounce (#5), and mid-cycle entitlement revocation (#10).