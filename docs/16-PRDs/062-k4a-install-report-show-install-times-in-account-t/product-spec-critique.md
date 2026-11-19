# Spec Critique: K4A Install Report — Account-Timezone Install Times (CSV Export)

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-19

## Summary

- **Critical Issues:** 1
- **High Issues:** 6
- **Medium Issues:** 5
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **FR-008 is an empty stub.** The "On-screen install time display (scope pending)" FR has a title and no Gherkin body at all — no Given/When/Then. Even for a deferred/pending item, there is zero testable content. An engineer reading FR-008 has nothing to implement, test, or even reason about. This blocks the FR set from being complete. | Critical | 3 (FR-008) | Either remove FR-008 entirely (it is already covered as Out of Scope in Section 6 and flagged DEC-001) or give it a real placeholder Gherkin scenario that is explicitly `@pending`/`@blocked-on-DEC-001`, so the FR set contains no empty requirements. |
| CG-002 | **FR-007 defines no concrete behavior.** The entire missing-timezone path resolves to "applies the agreed missing-timezone behavior" where the behavior is an open decision (BR-4/DEC-002). This is a dead end: an account with no configured timezone requesting an export has no defined outcome. The spec is honest that it's a decision, but the branch cannot be built or tested until DEC-002 is resolved. | High | 3 (FR-007) | Treat DEC-002 as a hard blocker for Phase 1 if accounts-without-timezone can exist. Force a default (recommend: block generation with named error, consistent with FR-006's anti-silent-UTC stance) before engineering starts. |
| CG-003 | **"Distinct, named error" (FR-006) is never defined.** FR-006 requires a "distinct, named failure condition" but never names it, specifies how it surfaces to the user (HTTP status? empty file? error row? UI toast?), or whether a partial CSV is emitted. The user flow box J→K says "surfaces a distinct, named error" but defines no surface. | High | 3 (FR-006), 2 (flow J/K) | Add a scenario specifying the exact error surface: e.g. export fails atomically (no CSV produced), user receives a named error code/message, and the failure is distinguishable from a successful empty report. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "must not materially degrade install report export generation time" / target "not specified" | "Materially degrade" is unmeasurable; target explicitly deferred (DEC-003). Cannot write a perf test. | `Given a representative install report of N rows, When the user exports the CSV, Then p95 generation time is within X% of the UTC-only baseline` — N and X MUST be filled via DEC-003 before this is testable. |
| NFR-002 | "without disproportionate slowdown... conversion cost must stay linear" | "Disproportionate" is subjective; "linear" has no bound constant to assert against. | `Given exports of 1k, 10k, and 100k rows, When generation time is measured, Then per-row conversion cost does not increase as row count grows (slope within tolerance T)` — define T. |
| FR-002 | "shows that instant expressed in Berlin local time / matches Berlin local time" | No concrete UTC instant → expected string pair given. A test needs a literal fixture. | `Given an install recorded at 2026-03-29T00:30:00Z (just before Berlin DST spring-forward), When Globex Media exports the CSV, Then the install time column shows "2026-03-29 01:30:00 +01:00"` (supply real pre/post-DST fixtures). |
| FR-006 | "surfaces a distinct, named failure condition" | "Distinct, named" is not asserted against any specific identifier or surface. | `Given the account timezone lookup fails, When the user exports, Then no CSV is produced And the user receives error code TIMEZONE_UNAVAILABLE` (name the condition). |
| FR-007 | "applies the agreed missing-timezone behavior" | Behavior is undefined (open decision), so no concrete Then exists. | Blocked on DEC-002; cannot be made testable until the default is chosen. |
| NFR-008 | "must be communicated to report consumers who may have assumed UTC" | "Communicated" is not a testable system behavior — it's a comms/release task with no acceptance criterion. | Move to release/comms checklist, OR make testable: `Given the CSV header/metadata, Then the install time column header indicates the timezone basis (e.g. "Install Time (account tz)")`. |

**Note:** All FRs that do have bodies (FR-001–FR-007) are correctly written in Gherkin. FR-008 has no body (see CG-001). No prose-style FRs were found among the populated requirements.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Delivery-surface ambiguity (CSV-only vs on-screen) | Partial — gated behind DEC-001/FR-008, but FR-008 is an empty stub | **FAIL** | The mitigation says "gate on-screen work behind the decision," but the gating FR (FR-008) has no content. The decision itself (DEC-001) is also unresolved. |
| Multi-service blast radius / unresolved service ownership | Partial — "engineering confirms the actual report-generation path" | **FAIL** | This defers the core question to engineering rather than resolving it. No requirement pins WHERE the conversion lands. The KG discrepancy (see §4/§5) is a real contradiction, not a to-be-confirmed detail. |
| Timezone/DST conversion correctness | Yes — NFR-006, FR-002, FR-003 | **PASS** | Well covered: tz database mandated, fixed-offset explicitly disallowed, DST-transition test cases specified. |
| Silent incorrect output on missing/unavailable timezone | Partial — FR-006 (retrieval failure) covered; FR-007 (missing) undefined | **FAIL** | The retrieval-failure half is handled; the missing-timezone half resolves to an undecided behavior (DEC-002). Half-mitigated. |
| Hard month-end deadline | Yes — Phase 1 scoping in Section 6 | **PASS** | Reasonable phasing; unconfirmed scope explicitly deferred. |

---

## 4. Missed Edge Cases

1. **Account timezone changes mid-export / between row reads:** FR-005 says the timezone is read as "the single reference," but doesn't specify snapshotting. If the account tz is reconfigured while a large export is generating, is the timezone read once at start or per-row? Risk of mixed-timezone output in one file.
2. **Ambiguous/nonexistent local times during DST transitions:** FR-003 covers installs on "opposite sides" of a transition but not an install whose UTC instant maps to the *skipped* hour (spring-forward gap) or the *repeated* hour (fall-back overlap). The wall-clock rendering of these is genuinely ambiguous and must be specified.
3. **Invalid/stale timezone identifier on the account:** The account has a tz value, but it's a deprecated/unknown IANA zone the tz database can't resolve. This is distinct from "no timezone" (FR-007) and "retrieval failed" (FR-006) — it's "retrieved but unresolvable." No FR covers it.
4. **Empty install report:** Account has a timezone but zero install rows. Does the export still succeed with just the (relabeled) column? No FR confirms the zero-row path.
5. **User clicks Back / cancels / network drops mid-download:** The "Happy Path" assumes the download completes. No requirement covers an interrupted/partial CSV download — is a partial file possible, and is it distinguishable from a complete one?
6. **CSV column header timezone labeling:** FR-004 mandates columns/order are unchanged. But if the install time column header still reads "UTC" (or is unlabeled), the CSV is now actively misleading. NFR-008 gestures at "communicating" but no FR requires the header to reflect the new basis. This is a correctness gap, not just comms.
7. **Fractional/non-hour-offset and historical-offset zones:** Berlin is clean, but the "reusable behavior" goal implies other zones. Zones with :30/:45 offsets or pre-1970 historical offset changes aren't addressed. If Phase 1 is Berlin-only, say so explicitly in an FR.
8. **Concurrent exports under retrieval failure:** FR-006 surfaces a named error, but under a transient accounts-api outage, retry/idempotency behavior is unspecified — does the user retry, or is there backoff?
9. **Timezone presence check vs retrieval failure race:** The flow branches on "has a configured timezone?" (B) then later "lookup fails" (C→J). What if presence check passes but the subsequent value fetch fails? The two-step read's failure modes aren't fully enumerated.
10. **Downstream parsers expecting UTC format:** NFR-005 preserves UTC in storage, but existing automated consumers parsing the CSV may hard-code UTC assumptions (e.g. appending "Z"). FR-004 guarantees structure, not semantics — a consumer could silently misinterpret local times as UTC. This is the backward-compat risk NFR-008 names but no FR enforces.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | **FAIL** | FR-008 ("On-screen install time display") has a title but **no Gherkin body at all** — it is an empty requirement. All other FRs (FR-001–FR-007) are correctly in Gherkin. |
| Document contains only Sections 0–6 | **FAIL** | The document ends with **"## Appendix: Engineering Reference"**, a multi-paragraph narrative appendix with its own sub-sections ("Affected Services") presenting prose findings derived from the KG. This is NOT a sanctioned trailer: it is not a `## Revision Summary` and not a run of short one-line `> **Label:**` blockquote metadata lines — it is exactly the multi-paragraph KG-analysis appendix the format rule is written to catch. Fold the essential grounding into Section 1 (where the discrepancy is already noted) and remove the appendix, or relocate it outside the spec. |
| Section 4 is bullet-list only (no tables/types) | **PASS** | Section 4 is a pure bullet list. Each item carries a sourcing label (System-Derived / External) in prose, but there are no attribute tables, field types, or schemas. Compliant. |

---

## 6. Recommendations

1. **Resolve FR-008 (CG-001, Critical).** Remove the empty FR-008 or replace it with an explicitly `@pending`/`@blocked-on-DEC-001` Gherkin placeholder. An empty requirement cannot ship.
2. **Remove the "Appendix: Engineering Reference" (Format FAIL).** Fold its load-bearing content (the service-ownership discrepancy) into Section 1 — which already discusses it — and delete the appendix so the document holds only Sections 0–6.
3. **Force DEC-002 before build (CG-002).** Decide the missing-timezone behavior (recommend: block with a named error, aligning with FR-006's anti-silent-UTC principle) so FR-007 becomes implementable and testable.
4. **Define the FR-006 error surface (CG-003).** Name the failure condition, specify atomic failure (no partial CSV), and distinguish it from an empty-but-successful report.
5. **Resolve the service-ownership discrepancy (Risk FAIL).** The independent KG cross-check contradicts the Context Seed: it surfaces **reporting-api** (serves report data/timezone formatting) and **frontend-mos**, while the Context Seed's intake names **k4a-mcp-server** for CSV generation. Both agree only on **accounts-api**. The spec defers this to "engineering confirms" — but this must be resolved before sizing, because it determines whether the conversion lands in k4a-mcp-server or reporting-api (or is duplicated/missed). The appearance of frontend-mos in the independent lookup is also direct evidence bearing on DEC-001 (on-screen scope). Do not pick a side silently; the spec correctly flags it, but it must be closed before implementation.
6. **Specify DST gap/overlap handling (Edge Case 2) and unresolvable-tz handling (Edge Case 3).** Add explicit FRs for the skipped/repeated wall-clock hours and for a retrieved-but-unresolvable timezone identifier.
7. **Add an FR requiring the install-time column header to reflect the new timezone basis (Edge Case 6).** NFR-008's "communicate to consumers" is not testable as written; a labeled header is.
8. **Make NFR-001/NFR-002 measurable (DEC-003).** Supply concrete row counts, a p95 target, and a baseline-delta tolerance so performance is assertable.