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
| CG-001 | **Contested data-source/report-generation path.** Two KG lookups disagree on which services own this feature; the spec defers resolution to engineering planning. An independent fresh KG cross-check (see below) **re-confirms Lookup B (the SAM-specific stack) and does NOT re-find `reporting-api` at all** — meaning the Lookup-A generic path is the weaker of the two. The spec still presents both as equally plausible. This is the single highest-leverage unknown and it sits against a hard 5-week deadline. Not a document defect per se (it is honestly flagged), but it must be resolved before build, and the spec should note the fresh cross-check tilts the evidence. | High | §1 caveat / Appendix | Record that an independent third lookup re-confirmed the SAM stack and failed to re-find `reporting-api`; narrow the "two competing paths" framing toward the SAM path pending engineering confirmation. |
| CG-002 | **Partial channel data within an otherwise-successful fetch.** FR-004 requires each channel "represented exactly once" and FR-006 blocks on unavailable/incomplete data, but neither defines behavior when the source returns 3 of 4 channels with no incompleteness flag (silent partial). "Each channel represented exactly once" is an assertion, not a defined failure handler. | High | §3 FR-004/FR-006 | Add an FR: if the retrieved set does not contain all four channels in the agreed scope, treat as data-incomplete (block + alert), regardless of whether the source flagged it. |
| CG-003 | **DST ambiguity on the delivery window.** NFR-001 fixes send at 08:00 US Eastern observing DST, but BR-3's "7 days ending the day before send" is calendar-week based. On a DST-transition weekend the reporting window's day boundaries vs. the Eastern send clock are undefined (23h/25h day). No FR pins the window's timezone. | Medium | BR-3 / NFR-001 | State the reporting window's timezone explicitly and define how a DST-transition week's boundaries are computed. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-002 | "p95 latency target measured as a defined number of concurrent summary runs… engineering sets the concrete p95 target and concurrency count during planning" | No number = not testable. Deferral is acknowledged but leaves no acceptance threshold. | Given N concurrent weekly runs, When per-channel spend is retrieved, Then p95 retrieval latency ≤ X ms (define N and X before build). |
| FR-001 | "fires on Monday morning" / "any day other than Monday morning" | "Morning" is not a testable boundary at the FR level (only NFR-001 pins 08:00). | Given the scheduler fires, When the local Eastern time is Monday 08:00, Then generation begins; And a fire at any other datetime does not generate or send. |
| FR-009a | "renders correctly in the recipients' email clients" | "Correctly" across unspecified clients is untestable. | Given the email is rendered, When opened in the supported client matrix [list], Then the logo displays (or shows alt text) and the two-column layout is unbroken. |
| NFR-006 | "must never be written to application logs" | Testable in principle but no verification requirement exists. | Given a run executes, When logs are scanned, Then no raw spend total, recipient address, or token appears (automated log-scrub assertion in CI). |

All Section 3 FRs are in Gherkin — no prose FRs found. The above are mostly NFRs and client-dependent assertions.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard external deadline (5 weeks) | Yes — MVP scope hold + resolve data path first | PASS | Mitigation is contingent on CG-001 resolving quickly; risk is real but acknowledged. |
| Multi-service blast radius | Yes — kill-switch (NFR-013), per-dependency failure handling (NFR-010), monitoring (NFR-003) | PASS | Adequately covered. |
| Billing/spend data handling | Yes — NFR-011, FR-006, NFR-006, NFR-004, NFR-012 | PASS | Strong coverage. |
| Contested data-source/report-gen path | Partial — deferred to engineering, no fallback if neither path is authoritative | FAIL | No requirement covers what happens if the chosen path can't deliver all four channels or disagrees with the customer's other spend views (ties to CG-002). Add a correctness gate independent of which path wins. |
| Backward-compat with existing scheduled reports | Yes — NFR-015 | PASS | But "engineering confirms which consumers are at risk" leaves no testable regression list yet. |
| Spend data freshness vs. completeness tension | Partial | FAIL | NFR-016 says use latest aggregate "not held back to prior full week," while BR-3 scopes to the completed preceding Mon–Sun week. These two can conflict: if the window is a completed past week, "latest aggregate at run time" and "prior full week totals" should be identical — but NFR-016's wording invites shipping an unsettled figure. Reconcile the freshness-vs-finalized tension explicitly. |

---

## 4. Missed Edge Cases

1. **Silent partial channel set:** Source returns < 4 channels with no incompleteness flag (see CG-002).
2. **Extra/unexpected channel:** Source returns a 5th channel not in the agreed scope — spec says "represented exactly once" per scope channel but doesn't say to drop/alert on unknown channels.
3. **Negative or credit spend:** A channel with a refund/credit yielding negative total — two-decimal USD formatting of negatives is undefined.
4. **Very large spend values:** No bound on figure magnitude/formatting (thousands separators? currency symbol placement?). BR-2 says "US dollars to two decimal places" but not grouping.
5. **Freshness vs. finalized conflict:** NFR-016 vs. BR-3 (see Risk audit).
6. **Distribution-list bounce / partial delivery:** FR-011/FR-012 treat delivery as a single accept/fail. A distro list where the SMTP relay accepts but some members bounce later is not covered — "accepted for delivery" ≠ delivered.
7. **Kill-switch flipped mid-compose:** FR-016 says an in-flight run completes under prior behavior — but "in-flight" is undefined (pre-fetch? post-compose? post-handoff?). Ambiguous completion boundary.
8. **Entitlement flips to false between runs while a retry is pending:** FR-014 checks at run start; a retry (FR-003/FR-012) crossing an entitlement change is undefined.
9. **Audit write fails AND send succeeds:** NFR-012 raises an alert but does not define whether the run is considered "delivered" or "indeterminate" for reporting purposes.
10. **Logo image host unreachable at send time:** FR-009a assumes a hosted image — if the host 404s, does the HTML degrade gracefully (alt text) without blocking the send? Not stated.
11. **Multiple spend sources (foreshadowed OOS) but real today:** If the authoritative source itself internally aggregates multiple feeds and one is stale, FR-005 "matches authoritative source" could still be wrong vs. the customer's billing view.
12. **DST-transition week boundary** (see CG-003).

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-017 all use `gherkin` blocks. Stray sentence "Now reporting the decisions I flagged in Sections 0–3." at the end of Section 3 is narrative noise but not an FR-format violation. |
| Document contains only Sections 0–6 | FAIL | The document contains an **"Appendix: Engineering Reference"** — a multi-paragraph narrative analysis section (Affected Services, Lookup A/B breakdowns, Discrepancy to resolve) placed between Section 6 and the Revision Summary. This is the multi-paragraph KG-analysis appendix the base rule is written to catch, not a sanctioned one-line metadata trailer. The trailing **"## Revision Summary"** is EXPECTED and correctly not flagged. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is bullet-list only; no attribute tables or type declarations. (Section 1 §1 references an "engineering appendix" — the appendix itself is the compliance problem, not Section 4.) |

Note on the Section 1 engineering-grounding caveat: the body text (§1) correctly keeps service names OUT and defers to an appendix — good — but the appendix then violates the Sections 0–6 constraint by existing as prose analysis.

---

## 6. Recommendations

1. **Remove or externalize the Engineering Reference appendix** (Format FAIL). Move the service-ownership analysis to a separate engineering planning doc; the spec body already references it correctly in §1. Keep the document to Sections 0–6 + Revision Summary.
2. **Resolve CG-001 before build.** Note in §1 that an independent fresh KG cross-check re-confirmed the SAM-specific stack (`k4a-sam-backend`, `sam-report-scheduler`, `sam-performance-reporter`) and did **not** re-find `reporting-api`, tilting the evidence toward Lookup B. Still require engineering confirmation, but stop presenting the two paths as equally weighted.
3. **Close CG-002 / Edge cases 1–2:** Add an FR making "fewer than four scoped channels" or "an unknown channel" a data-incomplete condition (block + alert), independent of any source-supplied completeness flag.
4. **Reconcile NFR-016 freshness vs. BR-3 finalized-week** (Risk FAIL / Edge 5). State whether a completed past week's "latest aggregate" can still shift, and if so whether the email waits.
5. **Pin the reporting-window timezone and DST handling** (CG-003 / Edge 12).
6. **Define delivery semantics for a distribution list** (Edge 6): distinguish relay-accepted from member-delivered, and state whether post-accept bounces trigger alerting.
7. **Specify logo-host failure degradation** (FR-009a / Edge 10) and a supported email-client matrix so FR-009a/FR-009b become testable.
8. **Give NFR-002 a concrete p95 and concurrency number** (or an explicit placeholder marked as a build-blocking TODO).
9. **Define "in-flight" for the kill-switch** (FR-016 / Edge 7) and entitlement-flip-during-retry behavior (Edge 8).