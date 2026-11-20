# Spec Critique: K4A Weekly Attribution Report PDF Download

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-20

## Summary

- **Critical Issues:** 0
- **High Issues:** 5
- **Medium Issues:** 7
- **Blocking:** No

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | The "no report available" state (FR-007) and the "generation failed/not-ready" state (FR-015) can both present to the user, but FR-007 only covers "not yet generated." A period that *attempted* generation, exhausted retries, and is marked not-ready is a distinct state from "never ran." The UI contract for distinguishing "not yet run" vs. "ran and failed" at the *view* level (not the download level, which FR-014 covers) is undefined. A finance user staring at an empty view near year-end cannot tell "wait, it's coming" from "it failed, escalate." | High | 2, 3 | Add an explicit view-level requirement distinguishing not-yet-run, in-progress, and failed/not-ready, with guidance on whether the failed state prompts the user to contact support vs. wait. |
| CG-002 | Secure-link expiry (FR-006) has no defined behavior when the link expires *mid-download* or when the user clicks an already-expired link from a stale tab. FR-005/FR-006 define issuance and expiry but not the re-request path. Dead end: user clicks expired link, gets a raw storage 403/XML error instead of a re-auth/re-issue flow. | High | 3 | Add an FR: given an expired secure link is used, when the user retries, then the system re-authorizes and re-issues a fresh link rather than surfacing a storage-layer error. |
| CG-003 | FR-011 defaults plan-entitlement to "included for standard plan" to hit the deadline, but if DEC-001 resolves to "gated," every Brightway download is rejected server-side (403) by the very requirement meant to protect data. There is no defined interim/manual-grant path. This is a self-acknowledged deadline-blocker with no mitigation requirement, only a flag. | High | 1, 3 | Add a requirement for a manual/temporary entitlement grant path for the initial customer pending DEC-001 resolution, OR escalate DEC-001 to a pre-build blocker with an owner and date. |
| CG-004 | Idempotent generation "overwrites/upserts" the period record (FR-012). If a user has already downloaded week N's PDF and a re-run overwrites it with corrected data, the audit trail (FR-018) records a download of a report whose content no longer matches what was served. No versioning or content-hash is captured. For finance/reconciliation data this is a reconciliation-integrity gap. | High | 3, 4 | Capture a content version/hash in the artifact and audit record so a recorded download is traceable to the exact bytes served, even after an overwrite. |

*No issues rose to Critical (implementation-blocking / data-loss / security-breach) severity; the data-sensitivity and fail-closed posture are well covered. The highest-impact gaps are self-acknowledged deadline and state-machine risks rated High.*

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "...should respond within a p95 latency target... Engineering may refine the exact threshold against real artifact sizes." | Two defects: "should" (not "must") and an open-ended "may refine" clause that makes the stated p95 ≤ 2s non-binding and therefore untestable as an acceptance gate. | Given a generated report is available, When ≥ N concurrent finance users request download-initiation, Then p95 download-initiation response ≤ 2s; threshold is fixed for acceptance and changed only by spec revision, not at implementation discretion. |
| NFR-004 | "...complete all enrolled accounts' reports within the weekly window with margin..." | "with margin" is not a measurable quantity. No concrete window or margin %. | Given the weekly generation run starts, When all enrolled accounts are processed, Then all reports reach ready state no later than [X hours] before the covered week's first consumer need, with ≥ [Y%] headroom against the weekly window. |
| FR-017 | "...within an established propagation window." (body); default 5-minute in note | The normative body references an unquantified "established propagation window"; the number lives only in a non-normative working-default note. A tester cannot bind to a note. | Given the kill-switch is flipped to disabled, When a new request arrives, Then within ≤ 5 minutes the request reflects the disabled state. (Promote the 5-min bound into the normative Then.) |
| NFR-002 | "...must enforce its own rate limit..." | No threshold given for the read-path limit — "its own rate limit" is directional, not testable. | Given a finance user exceeds [N requests / window] on the read/download path, When the next request arrives, Then it is rejected with a 429, independently of generation-path limits. |
| NFR-003 | "...within an explicit rate cap against that source..." | "explicit rate cap" with no number. Not testable. | Given report generation reads the shared attribution source, When the generation read rate would exceed [R req/s], Then the generator throttles to ≤ R req/s. |

All FRs (FR-001 through FR-019) are correctly written in Gherkin — the testability defects above are concentrated in the NFR section and in FR-017's reliance on a non-normative note for its one measurable value.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard year-end deadline (~5 weeks) — NFR-017 | Partial | FAIL | Mitigation is stated as "resolve DEC-001 early" and "budget discovery time," but neither is a *requirement* with an owner or date — they are conditions/hopes. The entitlement default (FR-011) actively risks blocking Brightway if DEC-001 goes the other way, and no interim-grant requirement exists (see CG-003). |
| Multi-service blast radius / unsettled scope — NFR-018 | Partial | PASS (with reservation) | Kill-switch (FR-017) is a concrete mitigation. However "engineering discovery confirms affected services" is deferred work, not a spec-satisfiable requirement; acceptable since the spec correctly refuses to pick a service list. |
| Finance/billing data sensitivity — NFR-019 | Yes | PASS | Well mitigated: NFR-008/009/010/011/012/013 plus FR-009/010/016/018/019 form a complete, verifiable control set. Strongest area of the spec. |
| Shared attribution source contention (NFR-003/NFR-021) | Yes (directional) | PASS | Rate cap + backward-compat requirement present, though unquantified (see NFR-003 testability). |
| Artifact overwrite vs. already-downloaded reconciliation data | No | FAIL | FR-012 idempotent overwrite is defined with no content-versioning safeguard; reconciliation integrity risk (CG-004) is unmitigated. |

---

## 4. Missed Edge Cases

1. **Expired secure link re-request:** No flow for a user clicking a stale/expired download link (see CG-002) — likely surfaces a raw storage error.
2. **Mid-download link expiry:** Link valid at request time but expires during a slow/large transfer — resume/renew behavior undefined.
3. **Overwrite-after-download drift:** A re-generated report changes content after a user already downloaded and filed the earlier version; no notification or version marker (see CG-004).
4. **Account deactivation / entitlement revoked between generation and download:** User was authorized at generation but account is suspended before download. FR-009 re-derives auth per request (good), but the user-facing message for "was available, now you're not entitled" is undefined.
5. **Empty attribution period:** Attribution source is reachable but returns zero attribution rows for the week. Does the system produce an empty PDF, a "no data" PDF, or mark not-ready? FR-003/FR-007 don't cover reachable-but-empty.
6. **Partial attribution data:** Source reachable but returns incomplete/partial data (not a clean failure). No requirement covers correctness-vs-completeness of a generated report under partial source responses.
7. **Clock/period-boundary ambiguity:** "Most recent completed week" with no timezone definition — week boundaries relative to account TZ vs. UTC undefined. Two users could disagree on which week a report covers.
8. **Concurrent download by multiple finance users:** FR-005 scopes to account; nothing addresses N finance users of the same account requesting simultaneously (rate-limit interaction with NFR-002 per-user vs. per-account).
9. **Kill-switch flipped mid-generation:** FR-017 note says in-flight runs complete under prior behavior — but a generation that *completes* while the feature is disabled may produce a downloadable artifact the user can't retrieve; the resulting state is undefined.
10. **Browser "Back" / double-submit during download:** Classic happy-path bias — user double-clicks download or navigates Back mid-request. Audit (FR-018) could record duplicate or orphaned access events.
11. **Audit-write failure after successful download:** FR-018 writes audit async and alerts on failure, but the download already succeeded — so a sensitive-data access can occur with no durable audit record. Is that an acceptable security posture for finance data? Not addressed.
12. **Retry-After with an absurd interval:** FR-013/NFR-014 honor Retry-After, but a malicious/misconfigured upstream returning a very large Retry-After could stall generation past the weekly window. No cap on honored interval.
13. **Report for an account enrolled mid-week:** Enrollment (FR-001) added partway through a period — does the first report cover a partial week or wait for the next full week? Undefined.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001–FR-019 are all in Gherkin. The working-default `>` notes are non-normative annotations, not FRs, and do not violate the check. |
| Document contains only Sections 0–6 | FAIL | The document contains an **"Appendix: Engineering Reference"** after Section 6 — a multi-paragraph narrative section with its own "Affected Services" prose list and a "Discrepancy from the independent cross-check" findings paragraph. This is exactly the sanctioned-exception counter-example (a stray multi-paragraph KG cross-check analysis appendix with its own findings), **not** a Revision Summary and **not** a short one-line `> **Label:**` metadata blockquote. It must be removed or relocated out of the requirement document. (Note: Sections 1–6 are present but there is no Section 0 header; the document begins at Section 1.) |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a bullet list of business-information items with prose descriptions; no attribute tables or type declarations. |

---

## 6. Recommendations

1. **Resolve DEC-001 (plan entitlement) as a pre-build blocker with an owner and date, and add an interim manual-grant requirement** so FR-011's default cannot silently block Brightway at the deadline (CG-003 / NFR-017).
2. **Remove or relocate the "Appendix: Engineering Reference"** out of the requirement document to restore Section 0–6 format compliance (Section 5 FAIL). Fold the confirmed blast-radius note (already summarized in Section 1) into engineering discovery artifacts, not the spec body.
3. **Add a content version/hash to the artifact and audit record** so idempotent overwrites (FR-012) remain reconcilable against already-downloaded reports (CG-004).
4. **Define the expired-link re-request flow** (CG-002) and mid-transfer expiry behavior (edge cases 1–2).
5. **Add a view-level report-state requirement** distinguishing not-yet-run / in-progress / failed-not-ready so finance users can act (CG-001, edge case 9).
6. **Quantify all NFR thresholds** (NFR-001 "may refine," NFR-002/003 rate caps, NFR-004 "with margin") and promote FR-017's 5-minute propagation into the normative Then clause.
7. **Specify empty-period and partial-data generation behavior** (edge cases 5–6) and the period-boundary timezone (edge case 7).
8. **Address audit-after-successful-download integrity** — decide whether a sensitive download with a failed audit write is acceptable, or gate completion on a durable write (edge case 11).

**Note on KG cross-check:** The spec's own Section 1 and appendix correctly and transparently disclose the service-list discrepancy against the independent cross-check — it names `k4a-mcp-server` as present in the primary list but not re-found by the cross-check, and surfaces `koch-query-api` and `summary-report-cloud` as low-confidence additions. This matches the independent evidence provided. No contradiction between the spec's implementation claims and the KG evidence was found; the spec appropriately declines to pick a side and defers to discovery. The disclosure itself is good practice — the only issue is its *placement* in a trailing appendix (Section 5 FAIL above).