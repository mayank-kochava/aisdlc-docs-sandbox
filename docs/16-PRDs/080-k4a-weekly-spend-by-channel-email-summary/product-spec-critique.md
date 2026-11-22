# Spec Critique: K4A — Weekly Spend-by-Channel Email Summary

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-22

## Summary

- **Critical Issues:** 0
- **High Issues:** 5
- **Medium Issues:** 7
- **Blocking:** No

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | The single-source assumption in FR-004/FR-005 ("the authoritative spend data source") is contradicted by the engineering grounding: the two KG lookups disagree on whether `reporting-api` or the SAM-specific `sam-performance-reporter`/`sam-campaign-data-sync` stack is the report-generation path, and confidence on `cost-v2` differs between lookups. "Equals the authoritative spend figure" is untestable until *which* source is authoritative is pinned. | High | §1 caveat, FR-004/005 | Add an acceptance criterion that names the authoritative source once resolved, and make build-start gated on that resolution (already called out as a deadline risk — tie it to a concrete FR/exit criterion, not just prose). |
| CG-002 | FR-007 / FR-012 both cap retries at 3 attempts with backoff + a 5-minute kill-switch propagation (FR-016), but the spec never bounds the *total wall-clock* a run may consume before it must be declared failed. A slow data source that answers just under the timeout on every one of 3 attempts could push dispatch past the (undefined, DEC-006) "Monday morning" window silently — NFR-003 only alerts *after* the window closes. | High | FR-007, NFR-001, NFR-003 | Add a per-run total deadline (e.g. "if a run cannot dispatch by HH:MM in the governing timezone it is recorded as missed-window and alerted"), resolvable with DEC-006. |
| CG-003 | FR-002/FR-003/NFR-008/NFR-009 define idempotency on key (customer + reporting window). The spec never defines behavior for a *manual re-trigger with a corrected format/channel-scope config* after a bad email already went out — idempotency would suppress the corrected send. There is no "supersede/resend" path. | Medium | FR-002/003 | Define whether an operator can force a corrected resend for an already-delivered window, and how that reconciles with "never two emails for the same window." |

No issue rises to the Critical bar (data loss / security breach / hard implementation blocker with no stated owner). The access-control, fail-closed, and block-on-incomplete-data requirements are present and specific. Several High gaps remain and should be closed before engineering starts.

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "generated and dispatched within the 'Monday morning' delivery window" | "Monday morning" and timezone undefined (DEC-006 open) — not testable. | Given an entitled customer, When the Monday run completes, Then the email is dispatched before [HH:MM] in [governing tz], And a dispatch after that boundary is recorded as missed-window. |
| NFR-002 | "p95 latency target measured as a defined number of concurrent summary runs" | Target value and concurrency count both deferred to planning — no number to test against. | Given [N] concurrent summary runs, When per-channel spend is retrieved, Then p95 retrieval latency ≤ [X] ms, And the shared source's rate cap is never exceeded. |
| FR-016 | "new scheduled runs stop starting within 5 minutes" | Testable as written (good) but "completes under the prior behavior" for in-flight runs is unverified — no assertion on what the in-flight run *does* to audit/delivery. | Given a run in flight when the kill-switch flips, When it completes, Then it still writes its terminal audit record and either delivers or records failure per prior rules. |
| FR-008 | "reproduces the format the marketing team currently uses in their manual spreadsheet" | Format (headers, currency, decimals, sort) is DEC-001 / unsupplied — the acceptance criterion references an artifact not in the spec. | Given the agreed format spec [DEC-001 resolved], When the body is composed, Then headers/currency/decimals/sort-order match the supplied reference exactly. |

The FRs are generally strong on Given/When/Then discipline. The untestable language is concentrated in the NFRs that depend on open DECISIONs, which is acceptable *as long as* those decisions gate build-start.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard external deadline (5 wks, summer) | Partial | PASS (weak) | Scope held to MVP; but "mitigated only if the contested data-path is resolved" is a precondition, not a mitigation. No concrete go/no-go exit criterion for the resolution. |
| Contested data-source / report-gen path (§1 caveat) | Named, deferred | FAIL | Acknowledged and assigned to engineering planning, but NO functional requirement forces resolution-before-build or defines a fallback if the two paths can't be reconciled in time. This is the deadline's biggest single threat and has no in-spec gate. |
| Multi-service blast radius | Yes | PASS | Kill-switch (NFR-013), per-dependency failure handling (NFR-010), monitoring (NFR-003). |
| Billing/spend data handling (wrong numbers) | Yes | PASS | FR-005/006, NFR-006/011, server-side access (NFR-004), immutable audit (NFR-012), block-on-incomplete. Strong coverage. |
| Shared spend source degradation for other consumers | Yes | PASS | NFR-002 rate-cap respect + 429 Retry-After honoring (FR-007). |
| Entitlement source unreachable | Yes | PASS | Fail-closed (FR-014, NFR-005). |
| Backward compat (reuse of existing report path) | Partial | PASS (weak) | NFR-015 states intent but defers "which existing consumers are at risk" to planning with no regression-test requirement attached. |

---

## 4. Missed Edge Cases

1. **Partial-channel data (some channels present, one channel's source silently empty):** FR-006 covers "data flagged as incomplete," but if the source returns a *successful* response that simply omits a channel (vs. flagging incompleteness), FR-010's zero-spend rule may render a real-but-missing channel as "£0" — indistinguishable from true zero. No reconciliation of channel scope (DEC-003) against returned channels.
2. **Timezone / DST boundary:** With DEC-006 open, a "Monday morning" run straddling a DST change, or a customer whose business timezone differs from the scheduler's, is undefined.
3. **Recipient list empty or all-invalid at send time:** FR-011 assumes a configured list exists. No behavior for an empty, null, or all-bounced recipient list — does the run count as delivered, failed, or entitlement-blocked?
4. **Recipient receives spend but is not entitled to view it:** BR-5/DEC-004 flags per-recipient permission tiers as unsettled, deferred to Future. But FR-015/NFR-004 enforce access at the *service* level by requesting identity — the *email recipient* is a different trust boundary. A recipient on the list who shouldn't see spend is an open confidentiality gap even in MVP.
5. **Partial email delivery (some recipients accepted, some bounced):** FR-012 treats delivery as binary accept/fail. Multi-recipient partial delivery (3 of 5 accepted) has no defined outcome state.
6. **Currency / multi-currency spend:** DEC-001 mentions "currency symbol" but nothing addresses whether channels could report spend in different currencies, or how mixed currencies are totalled/displayed.
7. **Audit write fails AND send fails simultaneously:** FR-017 writes audit async and alerts on audit-write failure; FR-006 alerts on send failure. Two alerts for one run, or a dropped audit for a failed run, is undefined ordering.
8. **Reporting window with zero activity across ALL channels:** Is an all-zero email still sent, or suppressed? FR-010 covers per-channel zero; whole-report-empty is not addressed.
9. **Scheduler fires late / skipped Monday (scheduler outage):** If the Monday trigger never fires (infra down), NFR-003's "fails to produce a delivered email by end of window" alert is the only backstop — but no catch-up/replay requirement defines whether a missed Monday is ever re-run.
10. **Mid-week config change to channel scope or format:** If DEC-001/DEC-003 config changes after a window closes but before send, which version applies? No config-snapshot-at-window-close requirement.
11. **`notifications-api` vs `reporting-subscriptions-worker` as delivery path (KG low-confidence):** The two possible delivery services have different retry/bounce semantics; FR-012's "3 attempts + alert" may not map cleanly to whichever is chosen.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-017 are all well-formed Gherkin blocks. |
| Document contains only Sections 0–6 | FAIL | The document ends with an **"Appendix: Engineering Reference"** containing a multi-paragraph narrative "Affected Services" analysis with Lookup A / Lookup B prose findings and a "Discrepancy to resolve" conclusion. This is not a sanctioned Revision Summary, nor a one-line `> **Label:**` metadata blockquote trailer — it is exactly the kind of multi-paragraph narrative analysis appendix with its own findings that the format rule is written to catch. Move its engineering-planning content out of the product spec (or into a linked engineering doc). |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is bullet list only; no attribute tables or type declarations. (Section 6's out-of-scope table is in Section 6, not Section 4, so it does not violate this check.) |

Note: The spec references "Sections 0–3" and "reporting the decisions I flagged in Sections 0–3" but no Section 0 is present in the document; numbering starts at Section 1. Minor inconsistency (Medium), not itself a compliance failure under the 0–6 rule.

---

## 6. KG Cross-Check Findings

The independent fresh KG lookup **matches Lookup B** of the spec's own appendix (the SAM-specific stack: `k4a-sam-backend`, `sam-report-scheduler`, `sam-performance-reporter`, `reporting-subscriptions-worker`, `notifications-api`, `sam-campaign-data-sync`, `cost-v2`, plus low-confidence `frontend-mos`). It did **not** re-find `reporting-api`, `accounts-api`, or the generic `reporting-api`-centric Lookup A stack.

- **Discrepancy 1 — `reporting-api` / `accounts-api` not corroborated:** The fresh lookup does not surface `reporting-api` (Lookup A's report-generation path) or `accounts-api` (Lookup A's entitlement source). The spec's FR-014 entitlement check and FR-004 spend retrieval therefore rest on a source only *one* of three lookups identified. This strengthens CG-001: engineering must not assume `reporting-api`/`accounts-api` are the paths.
- **Discrepancy 2 — fresh lookup leans SAM-stack:** Two of three lookups now point at the SAM-specific stack, shifting the balance of evidence. The spec correctly refuses to pick a side and flags resolution-before-build — this is the right call and is noted, not over-ruled.
- **Note — `frontend-mos`:** Re-surfaced at low confidence as "possible UI to configure weekly summary." The spec drops it on PM-confirmed no-UI (BR-7). Consistent; the PM decision correctly overrides the low-confidence KG signal, but worth a one-line note that a config UI service exists and is deliberately unused in Phase 1.

No claim in the spec about data flow, joins, or existing endpoints *contradicts* the evidence — the spec's own appendix already honestly records the disagreement. The finding is that the balance of fresh evidence now favors Lookup B.

---

## 7. Recommendations

1. **Resolve the contested data-source / report-generation path before build (CG-001, Risk FAIL).** Add an explicit build-start gate / exit criterion naming the authoritative spend source and report-gen path; the fresh KG lookup now favors the SAM stack (Lookup B) — treat that as the leading candidate but confirm.
2. **Move the Engineering Reference appendix out of the spec (Format FAIL).** It is a multi-paragraph narrative analysis appendix beyond Sections 0–6; relocate to a linked engineering-planning doc.
3. **Resolve DEC-006 (timezone + Monday-morning window boundary)** and add a per-run total wall-clock deadline (CG-002) so slow runs are caught *before* the window closes, not after.
4. **Close the recipient trust-boundary gap (edge case 4):** service-level access control (FR-015) does not govern who is on the email recipient list; define MVP behavior for recipients not entitled to view spend.
5. **Define empty/partial recipient-list and partial-delivery outcomes (edge cases 3, 5).**
6. **Define channel-scope reconciliation (edge case 1):** distinguish a source-omitted channel from a true zero-spend channel so FR-010 cannot mask missing data.
7. **Define config-snapshot-at-window-close and missed-Monday catch-up behavior (edge cases 9, 10).**
8. Resolve remaining open DECISIONs (DEC-001 format, DEC-003 channels, DEC-007 retention) and fix the Section-0 numbering reference.