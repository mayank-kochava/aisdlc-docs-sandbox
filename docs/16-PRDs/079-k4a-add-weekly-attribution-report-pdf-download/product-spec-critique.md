# Spec Critique: Brightway Retail Weekly Attribution PDF Report

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-22

## Summary

- **Critical Issues:** 2
- **High Issues:** 6
- **Medium Issues:** 7
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | FR-005 and FR-004 are referenced by NFR-009, NFR-010, and the "resolves" chain, but **neither FR-004 nor FR-005 is defined anywhere in Section 3.** NFR-009 ("Idempotent weekly generation (FR-005)") and NFR-010 ("No overlapping scheduled runs (FR-004)") point at requirements that do not exist. Engineering cannot implement idempotency/no-overlap behavior against a missing FR, and QA cannot trace the NFR to any testable behavior. This is a dangling-reference data gap that blocks build. | Critical | 3, 5 | Add FR-004 (scheduler must prevent concurrent runs for same customer+window) and FR-005 (generation idempotent on an explicit window key, repeat run overwrites, never double-counts) as full Gherkin FRs, or renumber the existing FRs and fix all cross-references. |
| CG-002 | **"First run / cold start" dead end.** FR-003, FR-006, NFR-016, and the flowchart all depend on "the prior stored report remains the latest available." But before the very first successful Monday run there IS no prior stored report. The flowchart's "no stored report yet" branch shows a message + alert, but no FR defines the cold-start behavior during the 5-week deadline window — i.e., what the entitled finance user sees between go-live and the first successful generation, and whether an initial backfill run is triggered off-schedule. Given the hard year-end deadline, a feature that goes live but has no first report until the next Monday could miss the deadline. | Critical | 2, 3 | Add an FR covering initial/first-run generation (trigger an immediate generation at go-live rather than waiting for the next Monday) and define the user-facing state when zero reports have ever been stored. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "must meet a p95 latency target... Target value: p95 ≤ 3 seconds under expected concurrent download load, pending confirmation" | "Expected concurrent load" is undefined (how many finance users?), and the target is "pending confirmation" — not a committed, testable number. | Given N concurrent entitled finance users (N defined), When each triggers a download of a stored report, Then p95 time-to-first-byte must be ≤ 3 seconds. (Define N explicitly.) |
| FR-013 / NFR-012 | "the change takes effect for new requests within an established propagation window" / "working default: 5 minutes" | "Established propagation window" is not a committed value in the FR itself; "working default" leaves it untestable as a pass/fail gate. | Given the kill-switch is flipped at T0, When a new request arrives at T0+5min, Then the feature is disabled for that request. (Commit the number into the FR.) |
| FR-014 | "the logo renders correctly in the stored and downloaded PDF" | "Renders correctly" is not objectively testable — no criteria for placement, size, resolution, or corruption. | Given a report PDF is generated, When rendered, Then the Brightway logo appears at the defined anchor position, at the defined dimensions, non-corrupted, and matches the approved brand asset checksum. |
| NFR-008 / NFR-006 | "retained for a defined minimum period" / "Link lifetime is an engineering-tunable default" | "Defined minimum period" and "engineering-tunable default" defer the actual testable boundary. Audit retention for finance data is a compliance assertion that must be pinned. | Given an audit entry written at T0, When T0+24 months has not elapsed, Then the entry must be retrievable; link lifetime must be committed (e.g., exactly 15 min, single-use). |
| NFR-011 / FR-009 (implied) | "A reconciling check or shared source-of-truth... must exist so discrepancies are caught before a report is served." | The reconciliation is asserted in an NFR but no FR defines the actual gate behavior (what happens when reconciliation fails — is the report suppressed? alerted?). "Must exist" is not a testable behavior. | Add an FR: Given a report is generated, When its channel totals do not reconcile to the authoritative attribution source within tolerance T, Then the report is not stored/served and an operational alert is raised. |

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Multi-service blast radius | Yes — kill-switch FR-013 / NFR-012 | PASS | Well-specified, including propagation window and in-flight behavior. |
| Finance-close correctness (numbers disagree with source) | Partial — NFR-011/NFR-014 assert a reconciliation check | **FAIL** | The reconciliation is only asserted in NFRs. There is **no Functional Requirement** defining the serve/suppress gate when reconciliation fails. A risk whose only mitigation lives in an NFR with no enforcing FR is effectively unmitigated at the behavior level. Add the FR (see Testability NFR-011 row). |
| Deadline risk (hard 5 weeks) | Partial — NFR-015 points to MVP scoping | **FAIL** | Mitigation is "scope the MVP and defer non-critical" — a planning statement, not a requirement. The cold-start gap (CG-002) directly threatens the deadline and has no mitigating FR. |
| Data-not-ready collision | Yes — FR-003, FR-006, NFR-016 | PASS | Skip/alert/keep-prior behavior is defined. |
| Shared query-resource contention | Yes — NFR-002 | PASS (soft) | Cap is left to engineering, acceptable as a guardrail, but "scheduled off-peak where feasible" is non-binding. |
| Entitlement source unreachable | Yes — fail-closed NFR-005, FR-010 | PASS | Explicitly fails closed. |

---

## 4. Missed Edge Cases

1. **First run / cold start (also CG-002):** No report exists before the first Monday; deadline exposure and undefined user state.
2. **Quarter-boundary rollover mid-feature:** BR-07 says the window advances only when a new calendar quarter completes. No FR defines what happens on the first Monday run AFTER a quarter boundary — does the stored report silently switch to the new quarter, and does the finance user get any indication the covered window changed? Risk of finance pulling the wrong quarter during year-end close.
3. **Reconciliation-failure serve gate:** No FR defines behavior when figures fail to reconcile (NFR-011 only asserts a check exists). Is a mismatched-but-stored report still served?
4. **Stale report staleness bound:** If generation skips/fails for several consecutive Mondays (FR-003/FR-006), the user keeps being served an increasingly old report with no staleness warning or max-age alert. Finance could unknowingly close the year on a weeks-old figure set.
5. **Partial audit-write failure during successful download:** FR-012 says audit writes async and a failure raises an alert — but does a successful download with a failed audit write count as compliant for a finance/compliance audit trail? The non-blocking choice may be a compliance gap, undocumented as a trade-off.
6. **Entitlement revoked mid-session:** User is entitled when the page loads (FR-007 shows the button) but entitlement is revoked before they click download. FR-010 re-derives per request so this likely fails closed — but the user-facing experience (button shown, then denied) is undefined.
7. **Secure link reuse / expiry during download:** NFR-006 proposes a single-use or 15-min link. What happens if a large PDF download outlives the link lifetime, or the user retries (FR-011) after the link has expired? Interaction between FR-011 retry and single-use links is unspecified.
8. **Concurrent download of a report being overwritten:** NFR-009 says a repeat run overwrites the stored artifact for a window. If a user is mid-download while the Monday run overwrites the latest-available pointer, the delivered file could be inconsistent. No FR addresses read-during-write.
9. **Zero-data quarter:** If the last-quarter window legitimately has zero attribution rows, FR-002 produces an empty table. Is an empty report served, suppressed, or alerted? Finance may misread "empty" as "no data available = error."
10. **Logo asset missing/unavailable at generation time:** FR-014 requires the logo but no FR covers the case where the brand asset is missing or fails to load — does generation fail, or produce a logo-less PDF?
11. **Timezone of "every Monday" and quarter boundaries:** BR-01 "every Monday" and BR-07 calendar quarters have no stated timezone. For a year-end close this is material (which day/quarter in which TZ).
12. **Browser "Back" / double-click during download:** The "Happy Path" bias — the flowchart assumes a clean click→retrieve→success. No handling for double-submits or navigation away mid-retrieval.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001, 002, 003, 006, 007, 008, 010, 011, 012, 013, 014 are all Gherkin. (Note: FR-004 and FR-005 are referenced but missing entirely — a content gap, see CG-001, not a prose-format violation.) |
| Document contains only Sections 0–6 | **FAIL** | The document contains an `## Appendix: Engineering Reference` with a multi-paragraph "Affected Services" narrative analysis (per-service roles, confidence labels, reconciliation recommendations) BEYOND Sections 0–6. This is exactly the stray multi-paragraph KG cross-check/analysis appendix the format rule is written to catch — it is NOT the sanctioned one-line `> **Label:**` metadata trailer, and NOT the Revision Summary. The trailing `## Revision Summary` section itself is EXPECTED/CORRECT and is NOT flagged. Remove or relocate the Engineering Reference appendix (move to a separate engineering planning doc). |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is bullet-list only; no attribute tables or type declarations. Each item uses prose Source/Matters annotations, which is acceptable. |

---

## 6. Recommendations

1. **(Critical, CG-001)** Define the missing **FR-004** (no overlapping runs) and **FR-005** (idempotent generation) as full Gherkin FRs, or renumber and fix every dangling reference in NFR-009/NFR-010. Build is blocked until the NFRs trace to real FRs.
2. **(Critical, CG-002)** Add a first-run / cold-start FR that triggers an immediate generation at go-live and defines the user-facing state when zero reports exist — this directly protects the hard year-end deadline.
3. **(High)** Add a Functional Requirement for the **reconciliation serve-gate**: suppress + alert when the PDF figures fail to reconcile to the authoritative source. The finance-correctness risk currently has no enforcing FR (Risk Audit FAIL).
4. **(High)** Add a **max-staleness bound / alert** for consecutive skipped or failed runs so finance is never silently served a weeks-old report (Edge Case 4).
5. **(High)** Pin all "pending confirmation / engineering-tunable / working default" values that are compliance- or acceptance-critical: NFR-001 concurrency N and p95, NFR-006 link lifetime, NFR-008 retention, FR-013 propagation window.
6. **(High)** Define **quarter-boundary rollover** behavior (Edge Case 2) and the **timezone** for "every Monday" and calendar quarters (Edge Case 11) — material for year-end close correctness.
7. **(High, Format)** Remove the `## Appendix: Engineering Reference` multi-paragraph narrative from the spec and relocate it to the engineering planning doc the spec already references; keep the document to Sections 0–6 plus the sanctioned Revision Summary.
8. **(Medium)** Resolve the **KG cross-check discrepancy** flagged below in the spec body (Section 1) rather than leaving it to planning, since entitlements-api gating is a hard FR-010 requirement.
9. **(Medium)** Address remaining edge cases: audit-write-failure compliance stance (5), entitlement revoked mid-session (6), link expiry vs. retry interaction (7), read-during-overwrite (8), zero-data quarter (9), missing logo asset (10), and double-click/back navigation (12).

---

### Note on KG Cross-Check (feeds Recommendation 8)

The spec is **internally honest** about the KG discrepancy — Section 1's "Knowledge-Graph Cross-Check Note" and the appendix both acknowledge the original three-service analysis (reporting-api, reporting-dispatcher, frontend-mos) vs. the four additional services from the fresh lookup (reporting-subscriptions-worker, summary-report-cloud, koch-query-api, entitlements-api). My independent cross-check **agrees with the spec's reconciled seven-service list** — no service is claimed by the spec that the fresh lookup fails to re-find, and no service in the fresh lookup is omitted. The confidence labels also align (reporting-api high; dispatcher/subscriptions-worker/summary-report-cloud/koch-query-api/frontend-mos medium; entitlements-api low). There is **no contradiction** between the spec's implementation-reality claims and the KG evidence. The only residual issue is that the spec leaves the reconciliation "to planning" for a service (entitlements-api) that underpins a hard requirement (FR-010) — hence Recommendation 8. Note this is distinct from the Format Compliance failure, which concerns the *appendix format*, not the KG content's accuracy.