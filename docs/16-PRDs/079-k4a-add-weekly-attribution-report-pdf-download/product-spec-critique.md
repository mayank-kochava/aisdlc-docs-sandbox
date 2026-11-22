# Spec Critique: Brightway Retail Weekly Attribution PDF Report

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-22

## Summary

- **Critical Issues:** 2
- **High Issues:** 6
- **Medium Issues:** 5
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **Report contents are entirely undefined.** The single most important thing the finance team consumes — the actual metrics, dimensions, and data points — is flagged `[DECISION REQUIRED]` with no placeholder set. FR-001/FR-002 generate and scope "a report" with zero definition of what is *in* it. Engineering cannot build, and QA cannot validate finance-grade correctness (BR-09/NFR-011), against an unknown payload. This blocks implementation. | Critical | 1, 3, 4 | Resolve report contents before build. At minimum define a frozen MVP metric/dimension set and add an FR asserting the PDF contains exactly that set. |
| CG-002 | **"Last quarter" is undefined (fixed vs. rolling) yet drives data scope.** FR-002 excludes/includes data by "the resolved 'last quarter' window," but BR-07 leaves resolution open. A rolling window changes the finance team's figures weekly; a fixed completed-quarter window does not. For a year-end close this is a data-correctness fork, not a nicety — the wrong choice silently produces audit-wrong numbers. | Critical | 1, 3 | Resolve BR-07 to a concrete, dated window rule (recommend most-recent-*completed* calendar quarter held fixed for close) and encode it in FR-002. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "p95 ≤ 3 seconds under expected concurrent download load, pending confirmation" | "expected concurrent load" is undefined (how many finance users? what file size?); "pending confirmation" leaves the target non-final, so no pass/fail test can be authored. | Given 20 concurrent entitled finance users, When each requests a stored ≤10MB report, Then p95 delivery-start latency ≤ 3s and no request exceeds 8s. |
| NFR-002 | "scheduled off-peak where feasible" and "concrete cap is an engineering decision" | "where feasible" and an unspecified cap are untestable — there is no threshold to assert against. | Given generation is running, When it queries the shared attribution source, Then concurrent generation queries ≤ N and interactive query p95 degrades by ≤ X%. |
| FR-013 / NFR-012 | "takes effect for new requests within an established propagation window" (working default 5 min) | "established propagation window" is a placeholder; 5 min is only a "working default," so the acceptance bound is soft. | Given the kill-switch is flipped at T, When a new request arrives at T+5min, Then it is rejected/not honored. |
| NFR-006 | "short-lived (e.g. 15-minute) link" / "single-use or short-lived" | "single-use OR short-lived" are two different testable behaviors left as an either/or; "e.g." makes 15 min non-binding. | Given a download link is issued, When it is reused OR accessed after 15 min, Then access is rejected. (Pick single-use vs. TTL, not both.) |
| NFR-008 | "retained for a defined minimum period. Working default: 24 months" | "defined minimum period" is deferred; confirm-or-default leaves no firm assertion. | Given an audit entry written at T, When queried at T+24 months, Then it is still retrievable and unaltered. |

**Note:** All FR-001 through FR-013 are correctly written in Gherkin — good. The testability issues above are concentrated in the NFR section's soft targets.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Multi-service blast radius (HIGH) | Yes — kill-switch FR-013 / NFR-012 | PASS | Propagation window still soft (see NFR-012 above). |
| Finance-close correctness / wrong figures | Yes — NFR-011 / NFR-014 reconciliation check | PARTIAL | Reconciliation *concept* exists but no FR defines what "reconcile" compares against or the tolerance (exact match? rounding?). Add an FR with an explicit equality/tolerance rule. |
| 5-week hard deadline | Partial — NFR-015 names MVP scoping | FAIL | Mitigation is "depends on scoping" + an unresolved `[DECISION REQUIRED]` plan-capacity question. A risk whose mitigation is gated on an open decision is not mitigated. Resolve BR-06/NFR-015 or the deadline is uncontrolled. |
| Data-not-ready collision | Yes — FR-003 / FR-006 / NFR-016 | PASS | Well handled. |
| Standard-plan capacity/cost for weekly gen+storage | No — flagged open (BR-06, NFR-015) | FAIL | No requirement handles the "upgrade required / phased delivery" branch. This can invalidate the entire deliverable. |
| Authorization source unreachable | Yes — FR-010 / NFR-005 fail-closed | PASS | Good. |

---

## 4. Missed Edge Cases

1. **Report never generated before the first scheduled run, but deadline arrives.** FR-009 handles the empty state, but nothing guarantees at least one successful report exists before year-end close. The whole point is the deadline — add a "first run must succeed by T-minus-X" operational requirement/alert.
2. **Stale latest-available report served silently.** FR-003/FR-006 keep the prior report as "latest available" when runs skip/fail. A finance user could download a weeks-old report believing it is current. No requirement surfaces the *as-of / generated-on* date to the user in the UI or PDF.
3. **PDF size / pagination / large-dataset generation.** No upper bound on report size or behavior when last-quarter data is unusually large. Generation timeout and partial-PDF behavior under volume is undefined.
4. **Quarter boundary / timezone ambiguity.** "Last quarter" window start/end has no stated timezone. For a global attribution dataset this shifts which events fall in-window — a correctness risk for close.
5. **Entitlement revoked mid-session.** FR-010 re-derives entitlement per request (good), but no case covers a user with an active download link (NFR-006) whose entitlement is revoked before the link expires. Does the signed link still work?
6. **Audit write fails permanently.** FR-012 writes the audit asynchronously and alerts on failure, but does not define whether a persistent audit-write outage should eventually *block* downloads of finance-close data (compliance posture). State the policy.
7. **Kill-switch flipped mid-download.** NFR-012 says in-flight runs complete and already-stored reports are unaffected, but a download in progress when the switch flips is not addressed.
8. **Data correction after a report is generated.** If upstream attribution data is restated after the weekly PDF is stored, nothing forces regeneration or flags the served PDF as superseded — a real finance-close hazard.
9. **Concurrent download of a report being overwritten (FR-005).** Idempotent regeneration overwrites the stored artifact; no requirement covers a user downloading at the exact moment of overwrite (read-during-write).
10. **"Weekly" schedule misses / clock drift / DST.** No requirement defines catch-up behavior if a scheduled run is skipped by the scheduler itself (distinct from data-not-ready).

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-013 are all proper Gherkin blocks. |
| Document contains only Sections 0–6 | **FAIL** | The document contains an **"## Appendix: Engineering Reference"** after Section 6 — a multi-subsection narrative with an "Affected Services" analysis (per-service roles, confidence labels, reconciliation prose). This is exactly the kind of stray multi-paragraph KG-analysis appendix the format rule is meant to catch, not a sanctioned one-line metadata trailer or a Revision Summary. Additionally, Section 1 contains a **"### Knowledge-Graph Cross-Check Note"** narrative subsection that duplicates/previews the appendix and belongs in engineering planning, not the spec body. Remove both to engineering planning docs. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is bullet-list prose only; types explicitly deferred to engineering. Good. |

---

## 6. Recommendations

1. **Resolve CG-001 (report contents).** Freeze an MVP metric/dimension set and add an FR asserting the PDF contains exactly it. Nothing is buildable or testable without this.
2. **Resolve CG-002 / BR-07 ("last quarter" fixed vs. rolling)** with a concrete dated, timezone-qualified window rule, and encode it in FR-002.
3. **Resolve the deadline/plan risk (NFR-015, BR-06).** The standard-plan capacity question gates the entire commitment; an unmitigated hard deadline is a program risk, not a footnote.
4. **Remove the "## Appendix: Engineering Reference" and the Section 1 "Knowledge-Graph Cross-Check Note"** from the spec body to restore Section 0–6 compliance; move KG/service reconciliation to an engineering planning artifact.
5. **Reconcile the affected-services discrepancy (now in the body) with an independent KG lookup** — see note below — but do it in engineering docs, not the spec.
6. **Firm up the soft NFR targets** (NFR-001 load definition, NFR-002 cap, NFR-006 single-use vs. TTL, NFR-008 retention, NFR-012 propagation window) into binding, assertable values.
7. **Add requirements for the stale-report "as-of" indicator, data-restatement regeneration, and first-run-before-deadline guarantee** (Edge cases 1, 2, 8).

---

**KG cross-check finding (for the author, re: Recommendation 5):** My own independent KG lookup matches the spec's *expanded* seven-service list (reporting-api high; reporting-dispatcher, reporting-subscriptions-worker, summary-report-cloud, koch-query-api, frontend-mos medium; entitlements-api low). The spec correctly surfaces the discrepancy between the original three-service analysis and the seven-service cross-check rather than silently picking one — that handling is sound. One nuance: the spec labels frontend-mos as "agreed by both lookups" at high confidence, but my fresh lookup rates frontend-mos only *medium*. Minor — flag for engineering reconciliation, no product-behavior impact.