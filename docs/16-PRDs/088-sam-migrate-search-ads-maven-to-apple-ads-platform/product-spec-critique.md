# Spec Critique: SAM — Migrate Search Ads Maven to Apple Ads Platform API v1

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
| CG-001 | The spec's whole safety thesis — "never show 0, always flag missing" — depends on reliably distinguishing a *genuinely missing* metric from a *legitimate zero*. A brand-new campaign, a paused campaign, or a zero-spend day legitimately returns 0 for spend/taps/installs. Nothing in the FRs defines how the system tells a legitimate 0 apart from a missing field. FR-009 keys off "missing one of the seven metrics" but never defines "missing" at the wire level (field absent? null? empty string? 0? error subcode?). If "missing" is mis-detected, the system will either flood users with false "metric missing" alerts on healthy zero-spend campaigns (alert fatigue → people ignore the real spend/ROAS alert) or fail to flag a true missing metric. This is the exact failure the project exists to prevent. | Critical | §1 Business Rules, FR-009, §4 Per-metric presence state | Define "missing" precisely at the v1 contract level (field absent/null vs. present-and-zero) in a dedicated FR, with explicit Given/When/Then for the legitimate-zero case. This is a prerequisite, not a planning detail. |
| CG-002 | Partial-response / malformed-response dead end. FR-008 handles "all seven present," FR-009 handles "one or more missing," and the Mermaid flow handles "call failed." But a v1 response that arrives HTTP 200 with a *structurally malformed* body, an unparseable number, a negative spend, a field present but typed differently (string "N/A" where a number is expected), or a partial payload truncated mid-stream is undefined. Does a garbage value count as "present" (and get displayed as corrupt data — the 0-masquerade in a new costume) or as "missing"? No FR covers it. | Critical | §3 Feature Area B, §2 flow node F/G | Add an FR: a metric present but failing validation (wrong type, un-parseable, out-of-range/negative where impossible) is treated as MISSING, routed to blank/flag + the severity ladder, and never displayed as-is. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-001–FR-006 | "...identical in outcome and user-visible behavior to the current v5 ... behavior" | "Identical to v5" is not independently testable — there is no captured v5 baseline, oracle, or golden-output set referenced. A tester cannot assert "identical" without a defined reference. | `Given a recorded v5 golden-response fixture for operation X, When the same operation runs against v1, Then the v1 result record matches the v5 golden fixture on all defined fields [enumerate fields], excluding only the API-version marker.` Requires the spec to name/define the golden-fixture set. |
| FR-010 / FR-011 | "louder / more prominent alert" vs "standard alert" | "Louder / more prominent" is subjective and untestable. No channel, no payload, no severity enum. | `Given spend is missing in cycle C, When missing-metric handling triggers, Then an alert of severity=CRITICAL is emitted to channel [X] within [N] seconds; Given avg CPT is missing, Then an alert of severity=WARNING is emitted to channel [X].` Define the two severities as concrete enum values + channels. |
| FR-013 | "displays a warning" | "A warning" is undefined — visual treatment, placement, and dismissibility are unspecified, so no UI assertion can be written. Compounded by the unresolved UI-impact [DECISION REQUIRED]. | `Given metric M is blank/n-a on dashboard D, When D renders, Then a warning indicator [defined visual token] is shown adjacent to M AND all other tiles render their present values.` Requires the UI decision to be resolved. |
| NFR-001 | "within parity of the current v5 baseline (p95 over a representative reporting cycle)" | "Parity" has no tolerance band and "representative cycle" is undefined. Pass/fail is not computable. | `Given the v5 p95 report-latency baseline B, When v1 is live, Then v1 p95 latency ≤ B × 1.10 measured over [defined dataset/window].` Fill the tolerance and dataset. |
| NFR-009 | "exceeds a sustained threshold — e.g. sync error rate > 1% for 5 minutes" | "e.g." makes the only concrete number an example, not a committed requirement. | Promote to a committed threshold: `Given sync error rate > 1% sustained for ≥ 5 minutes, Then a systemic-failure alert of severity=CRITICAL fires.` Remove "e.g." |
| NFR-002 | "detected and classified within the same reporting cycle ... exact maximum acceptable latency is not specified" | Explicitly unspecified by the author — untestable until a number exists. | `Given a v1 response received at T, Then present/missing classification and alert surfacing complete by T + [N minutes].` Supply N. |

*Note: All FR-001–FR-015 are correctly formatted in Gherkin — the testability problem is the vague predicates inside them ("identical to v5," "louder"), not the format.*

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Report quietly shows 0 for a silently-missing metric | Yes — FR-009, NFR-010, NFR-012 | **PASS (conditional)** | Mitigation exists but is undermined by CG-001: without a defined legitimate-zero vs. missing distinction, "never show 0" is not actually implementable. Mitigation is only as good as the undefined "missing" definition. |
| Spend/ROAS silently wrong → overspend, wrong pauses | Yes — FR-010, NFR-009, NFR-012 | **PASS (conditional)** | Covers *missing* spend/ROAS. Does NOT cover spend/ROAS that arrives *present but wrong* (bad mapping, unit mismatch — e.g. micros vs. dollars, currency). A silently-wrong-but-present value bypasses every missing-metric guard. See EC-7. |
| v5 retired mid-migration with no fallback | Partial — NFR-011 flags it | **FAIL** | NFR-011 *conditions the entire rollback strategy* on v5 still being live but marks that fact [DECISION REQUIRED]. If v5 is already retired, the primary mitigation (kill-switch to v5) evaporates and NO alternative fallback exists. A risk whose sole mitigation is "confirm later" is unmitigated today. Needs a Plan B that does not assume v5 survives. |
| 10-service blast radius, limited course-correct room before target | Partial — NFR-010/011/013 | **PASS (weak)** | Consistency, kill-switch, contract compat are defined. But the affected-service list itself is unresolved (see §1/§5), so "blast radius" is unquantified — you cannot bound a radius you haven't enumerated. |
| v1 drops/renames fields unsupported in v5 | Partial — NFR-013 + §6 | **FAIL** | NFR-013 says changes "must be surfaced," and §6 defers the actual unsupported-field handling to planning. There is no *requirement* defining the behavior when a field v5 had simply does not exist in v1 — only a promise to figure it out. A deferred mitigation is not a mitigation. |
| Credentials/tokens leaking in logs | Yes — NFR-007 | **PASS** | Concrete (mask to last 4–8). Adequate. |
| Date mismatch — spec target dates conflict | N/A (meta) | **FAIL** | §1 states a "fixed Dec 28 2026 target" while §1 Risk & Scope says "fixed Dec 28 2026 target"; the Problem section references v5 retirement timing as unknown ([DECISION REQUIRED] in NFR-011). The migration deadline and the v5-retirement date are treated as independent but the rollback plan silently assumes retirement > deadline. Flag the dependency explicitly. |

---

## 4. Missed Edge Cases

1. **Legitimate zero vs. missing (the core ambiguity):** Zero-spend campaign, brand-new campaign, paused campaign — all legitimately return 0. Undefined whether these trigger false "missing" alerts. (See CG-001.)
2. **Present-but-corrupt value:** HTTP 200 with malformed/wrong-type/negative value — undefined whether treated as present or missing. (See CG-002.)
3. **Spend/ROAS present but *wrong* (unit/currency/mapping error):** micros-vs-dollars, currency conversion, ROAS as ratio vs. percentage. Every guard here targets *missing*; a silently-mis-mapped-but-present spend sails straight through and corrupts financial decisions — the exact harm described in §1. No parity/validation FR covers value correctness beyond "matches v5" (itself untested — FR-008 bullet 2).
4. **Flapping metric:** A metric that is present in cycle N, missing in N+1, present in N+2 — does the paused automation rule (FR-014) resume, re-pause, re-resume on every flap? Auto-resume is explicitly deferred (§6), so flapping behavior is undefined and could thrash campaigns.
5. **Multiple simultaneous missing metrics including BOTH a louder and a standard one:** FR-010/FR-011 treat severities independently; no rule for de-duplicating/escalating when, say, spend (louder) and taps (standard) are both missing in the same cycle. Alert storm vs. single escalated alert is undefined.
6. **Partial automation dependency:** FR-014 pauses the rule depending on a blank metric. What about a rule depending on BOTH a present and a blank metric, or a composite metric (ROAS derived from spend ÷ revenue) where an *input* is missing but ROAS itself is "present"? Derived-metric dependency chains are uncovered.
7. **Operation submitted during the kill-switch flip:** NFR-011 says in-flight requests complete under prior behavior, but an operation submitted to v1, half-written, then rolled back to v5 mid-transaction has undefined end state. Possible split-brain write across two APIs.
8. **Idempotency key collision across v5/v1:** NFR-004 keys on "reporting period + identity." During parallel-run (NFR-012), both v5 and v1 ingest the same window — does the upsert dedupe across sources, or does v1 overwrite v5's verified row (destroying the comparison baseline)?
9. **Rate-limit during bulk + retry interaction:** FR-006/NFR-005/NFR-003 — a bulk op hitting 429 mid-batch with bounded retry (3 attempts) exhausted: is the bulk reported as partial success, fully failed, or retried-later? Interaction of bulk partial-failure semantics with the retry cap is unspecified.
10. **Audit-write failure during an incident storm:** NFR-008 fires an operational alert on audit-write failure. During a systemic v1 outage, audit failures could themselves storm. No throttling/aggregation defined; risk of alert-on-alert cascade.
11. **Credential/token expiry mid-cycle:** Apple API token expires between fetch calls in one 4-hourly cycle → partial cycle fetched. Undefined whether partially-fetched cycle is "missing" for the un-fetched entities or retried.
12. **Timezone/timestamp cadence drift between v5 and v1:** §4 notes v1 timestamp cadence "may differ" from v5. If v1 reports on a shifted boundary, the parallel-run comparison (NFR-012) could compare mismatched windows and produce false variance failures — blocking cutover on a non-bug.
13. **Dashboard with zero present metrics:** FR-013 assumes "all other data" exists to keep showing. If every metric is missing (total v1 outage), the dashboard is all-warnings — undefined whether that's a different (louder) state than per-metric warnings.
14. **Resume-ordering after rollback:** After a kill-switch revert to v5, paused automation rules (FR-014) — do they auto-resume under v5? Interaction between rollback (NFR-011) and paused-rule state (§4) is undefined.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-015 are all in Gherkin fenced blocks. Note: the sentence *"I've written Sections 0–3. Reporting the two business trade-offs I tagged."* after FR-015 is stray authoring chatter that should be removed, but it is not an FR. |
| Document contains only Sections 0–6 | **FAIL** | The document contains a trailing **"## Appendix: Engineering Reference"** with multi-paragraph narrative "Affected Services" analysis, confidence ratings, and an independent cross-check narrative with its own findings. This is exactly the kind of multi-paragraph narrative analysis appendix (not a Revision Summary and not a short `> **Label:**` metadata line) that the format rule prohibits. Fold the affected-service reconciliation into the §1 [DECISION REQUIRED] note (already referenced there) and delete the appendix, or move it to a separate engineering planning doc. |
| Section 4 is bullet-list only (no tables/types) | PASS | §4 is bullet-list only; each item carries prose Source/Matters annotations, no attribute tables or type declarations. Compliant. |

---

## Cross-check against independent KG evidence

The spec's own §1 note and Appendix already acknowledge the v5→v1 affected-service discrepancy, which is good. Reconciling against the independent KG cross-check provided for this review:

- **sam-mono (high confidence in BOTH the spec's cross-check note and the independent lookup) is still only in the Appendix, not the primary list.** A high-confidence core monorepo that calls the Apple Ads API should be treated as in-scope primary blast radius, not a "reconcile later" footnote. This under-states the §1 HIGH-risk blast radius.
- **sam-report-scheduler:** spec's primary list asserts it consumes synced performance data (medium/implied confidence); independent lookup rates it **low confidence**. Scope claim is stronger than the evidence supports — flag for confirmation.
- **sam-keyword-search-optimization:** spec calls this its "lowest-confidence" entry; independent lookup rates it **medium**. Minor directional disagreement; worth confirming it IS in scope rather than treating as marginal.
- **k4a-sam-campaign-data-sync vs sam-campaign-data-sync:** both sources flag a possible duplicate/renamed pair. Correctly surfaced as unresolved — no contradiction, but engineering must confirm these are not two live write paths to the same data (which would threaten NFR-004 idempotency and NFR-010 cross-surface consistency).

No hard contradiction between the spec's implementation claims and the KG evidence — the issue is understated scope confidence, not a false claim. This belongs in the §1 [DECISION REQUIRED], which the spec should strengthen rather than relegate to the (to-be-deleted) appendix.

---

## 6. Recommendations

1. **Define "missing" precisely (CG-001).** Add an FR distinguishing field-absent/null (missing → flag) from present-and-zero (legitimate → display as 0). The entire project thesis is non-implementable until this exists. Blocking.
2. **Add a present-but-invalid FR (CG-002).** Wrong-type, un-parseable, negative, or out-of-range values must route to the missing/flag path and never display as-is. Blocking.
3. **Add a value-correctness/parity guard for spend & ROAS (EC-3).** A present-but-mis-mapped spend (units/currency) bypasses every current guard and causes the exact financial harm the project targets. Define unit/currency contract validation or a v5/v1 value-match assertion.
4. **Resolve the v5-retirement dependency (Risk FAIL).** The rollback strategy (NFR-011) is unmitigated if v5 is already retired. Confirm Apple's v5 retirement date vs. the migration target, and define a Plan B fallback that does not assume v5 survives.
5. **Replace subjective alert language with concrete severities/channels (FR-010/FR-011/FR-013).** Define a severity enum (e.g. CRITICAL/WARNING), channels, and payloads; supply the NFR-002 detection-latency number.
6. **Establish the v5 golden-output oracle (FR-001–FR-008).** "Identical to v5" is untestable without a captured baseline fixture set — name it and make the parity assertion concrete.
7. **Specify unsupported-field handling as a requirement, not a deferral (Risk FAIL).** NFR-013 + §6 currently only promise to figure it out; define the behavior when a v5 field has no v1 equivalent.
8. **Delete/relocate the Engineering Reference appendix (Format FAIL).** Fold the service-list reconciliation into the existing §1 [DECISION REQUIRED]; promote high-confidence sam-mono into the recognized blast radius.
9. **Define flapping, multi-missing, and derived-metric dependency behavior (EC-4/5/6).** Resolve auto-resume and alert de-duplication before cutover to prevent campaign thrash and alert storms.
10. **Define parallel-run window alignment and idempotency-source dedup (EC-8/EC-12).** Ensure v5/v1 comparison windows align and that v1 ingestion does not overwrite the verified v5 baseline during soak.