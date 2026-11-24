# Spec Critique: Amazon Ads eventGrainEligibility Row-Level Redaction

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-24

## Summary

- **Critical Issues:** 0
- **High Issues:** 6
- **Medium Issues:** 8
- **Blocking:** No

This spec is unusually disciplined for the amount of genuine uncertainty it carries. It openly flags its open decisions ([DECISION REQUIRED]), writes its FRs to verify the *mechanism* independently of the undefined rule *values*, and explicitly adopts a fail-closed posture — which pre-empts the single most dangerous edge case (leaking restricted rows on dependency failure). There are no Critical gaps that block implementation of the mechanism. However, several High-severity testability and consistency gaps remain, driven mostly by the three deferred product decisions and by thresholds left as "engineering determination." None of these cause data loss or a security hole given the fail-closed default, so none are Critical.

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | FR-010 (access-request workflow) is "conditional on decision" and BR-007 is undecided, but the spec never defines the *terminal* state if a customer initiates a request and no approval path exists yet. The Gherkin says "the request follows the agreed approval or escalation path" — if that path is undefined, the flow dead-ends with no defined customer-facing outcome. | High | §3 FR-010 | Either keep FR-010 fully out of Phase 1 (consistent with §6 OOS) or specify the no-workflow fallback: "access-request UI is not rendered while BR-007 is undecided." Avoid a half-enabled dead end. |
| CG-002 | FR-011 fails closed on *unreachable* source, but the flow never defines behavior when the authoritative source is **reachable but returns a malformed/ambiguous eligibility signal** (e.g. null, unparseable, or a value not in the known set). Fail-closed is stated for unreachability only. | High | §2 flow / FR-011 | Add an FR: "Given the authoritative source returns an unrecognized or null eligibility value for a row, Then that row is treated as redacted (fail-closed) and a distinct 'indeterminate-eligibility' status is recorded." |
| CG-003 | The kill-switch flip (FR-013/NFR-015) defines in-flight request behavior but not the window where the flag is *mid-propagation* across multiple services/paths — the query path may see "enabled" while the reporting path still sees "disabled," violating BR-002 consistency during the 5-minute window. | High | FR-013 / NFR-019 | Add a requirement that during flag propagation the consistency guarantee (BR-002) is explicitly best-effort/eventually-consistent with a bounded window, OR that all paths read the flag from a single shared source evaluated at the chokepoint. |

No issue rises to Critical: the fail-closed default (NFR-012) means the worst realistic failure is over-redaction (availability), not a leak (security/data loss). CG-002/CG-003 are genuine logic gaps but each resolves to "redact" / "withhold," not "expose."

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "must not **materially degrade**… target p95 added latency ≤ X ms… working default ≤ 50 ms" | "Materially degrade" is unmeasurable; X is a placeholder. The 50ms default is testable but "representative concurrent load" / "existing peak concurrency" (N) is unbound. | Given N concurrent eventGrainEligibility read calls at the service's documented peak concurrency, When redaction evaluation is enabled, Then p95 added latency attributable to redaction is ≤ 50 ms measured against the traced no-redaction baseline. (Pin N before QA sign-off.) |
| FR-013 / NFR-015 | "takes effect… within the stated propagation time" / "within 5 minutes" | FR-013 references a "stated propagation time" that FR-013 itself does not state; only NFR-015 pins 5 min. An FR should be self-contained/testable. | Replace "within the stated propagation time" with "within 5 minutes (per NFR-015)" so the FR is independently testable. |
| FR-009 | "presented in exactly **one agreed manner**" | Not yet testable because the agreed manner (BR-006) is undefined — acceptable as a deferred decision, but flag that the test cannot be authored until BR-006 lands. | No rewrite needed; add explicit note: "QA test for FR-009 is blocked on BR-006 and must not be marked covered until the presentation rule is chosen." |
| NFR-011 | "A longer retention period **may be required**" | "May be required" is not testable. [DECISION REQUIRED] correctly tags it, but the baseline 12-month floor is the only testable clause today. | Keep 12-month floor as the testable requirement; gate the extended-retention test on the §NFR-011 decision. |

All FR-001 through FR-017 are written in valid Gherkin with concrete, mechanism-level assertions. The primary testability risk is placeholder thresholds and decision-dependent values, not prose FRs.

---

## 3. Risk Mitigation Audit

| Risk (from spec §1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------|---------------------|---------|-------|
| "Second door" leak — one path redacts, another doesn't | Yes — FR-007, FR-008, NFR-019 single-chokepoint | PASS | Mitigation is a real requirement, not just a stated risk. Residual gap during flag propagation (CG-003). |
| Authoritative eligibility source unreachable | Yes — FR-011, NFR-012 fail-closed | PASS | Strong. Does not cover malformed-response case (CG-002). |
| Client-supplied eligibility spoofing | Yes — FR-003, NFR-005 | PASS | Explicit per-request re-derivation; client claims never trusted. |
| Restricted/PII data leaking into logs | Yes — FR-017, NFR-007, NFR-008 | PASS | Covers logs, caches, debug output, masks identifiers. |
| Read-path performance degradation under new eval load | Partial — NFR-001/002/003 | PARTIAL | Alerting (NFR-003) and rate cap (NFR-002) are concrete; the latency *target* is a placeholder (see T-review). Mechanism present, threshold deferred. |
| Audit write failure dropping compliance records | Yes — FR-015, NFR-010 (alert, never silent) | PASS | |
| Dependency flapping / retry storms | Yes — NFR-013 bounded retry + backoff + Retry-After | PASS | |
| HIGH-risk multi-service rollout causing broad breakage | Yes — NFR-016 staged rollout, NFR-017 rollback | PASS | |
| Unknown current customer-visible baseline undermining "prior behavior" | Acknowledged, deferred to discovery | PARTIAL | NFR-014/FR-014 reference "prior behavior" that is itself undefined (see Edge Cases). Honestly flagged, but FR-014 is untestable until baseline is traced. |
| Service-ownership ambiguity (KG discrepancy) | Acknowledged, routed to eng | PASS (as product doc) | Correctly scoped as an engineering ownership question; see §KG reconciliation below. |

Overall verdict: **PASS with two PARTIALs.** The spec consistently converts stated risks into actual requirements rather than just listing them — the rare and correct pattern. The two PARTIALs are both threshold/baseline placeholders, not missing mitigations.

---

## 4. Missed Edge Cases

1. **Malformed/null eligibility signal (not just unreachable):** FR-011 covers unreachability; nothing covers a reachable source returning an unparseable or out-of-domain value. (See CG-002.)
2. **Flag mid-propagation inconsistency across paths:** Query path "enabled" while reporting path still "disabled" during the 5-min window violates BR-002. (See CG-003.)
3. **Pagination / result-count shift from omitted rows:** NFR-018 flags this for *design verification* but no FR defines the contract. If BR-006 resolves to "omit entirely," result counts and page boundaries shift for programmatic consumers — a behavior change beyond "rows become redacted." Needs an explicit FR once BR-006 lands.
4. **Eligibility changes mid-session / stale UI:** A row eligible at page load becomes ineligible before the user scrolls/refreshes. §6 defers real-time re-evaluation cadence, but the UI consistency expectation (eligible-then-redacted within one session) is undefined.
5. **Partial result-set failure:** Authoritative source succeeds for some rows and times out for others within one request. Fail-closed implies redact-the-unknowns, but FR-011 reads as all-or-nothing ("does not return rows it cannot confirm") — the mixed case should be stated explicitly.
6. **Audit write for the fail-closed/error path:** FR-015 logs "redact or expose" decisions. When FR-011 fails closed and returns a dependency error, is an audit record written? A denied-everything response is itself an auditable outcome but isn't clearly in FR-015's trigger.
7. **Empty / zero-eligible result:** Requester eligible for no rows — does the UI render an empty state vs. an error vs. an all-redacted view? Interacts with BR-006 but is a distinct case.
8. **Audit log volume / self-throttling:** FR-015 writes a record per redaction decision; a large multi-row query could generate a high-cardinality audit burst. No batching or volume guardrail is specified, and it could itself stress the shared resource NFR-002 protects.
9. **Kill-switch disabled + partial deploy:** FR-014 says "no partial or inconsistent redaction while disabled," but during the staged rollout (NFR-016) different percentage buckets are simultaneously enabled/disabled by design — the interaction of per-request bucketing with FR-014's "no inconsistent redaction" guarantee should be reconciled.
10. **Back / cancel mid-request:** Classic happy-path gap — customer navigates away mid-read. Low impact given server-side enforcement, but the audit outcome for an abandoned request is undefined.
11. **Baseline undefined undermines FR-014 testing:** "Prior behavior" is explicitly unknown (§1, §4 baseline item). FR-014 cannot be verified until the baseline is traced — flagged honestly but worth restating as a hard test-blocker.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-017 are all in fenced Gherkin blocks with valid Given/When/Then structure. The inter-FR explanatory notes are annotations, not FRs. |
| Document contains only Sections 0–6 | PASS | Document runs §1 through §6. No stray appendices or extra narrative sections. The "Note on affected services (KG reconciliation)" and "Note on dependencies" are subsections within §1 and §6 respectively, not top-level sections. No trailing Revision Summary or metadata blockquotes present (none required). |
| Section 4 is bullet-list only (no tables/types) | PASS | §4 is entirely bullet items (Source / Why-it-matters prose). No attribute tables, no type declarations, no schema constraints — consistent with its own stated deferral of types to eng-plan. |

Note: §1 is labeled "# 1. Business Logic & Context" — there is no explicit Section 0. This is acceptable (0–6 is a range, not a mandate that a 0 exists), and not flagged as a failure.

---

### KG Cross-Check (reconciliation against independent evidence)

The spec does **not** name specific services; it deliberately describes "two independent backend read paths plus a UI" abstractly and routes service-ownership to engineering (§1 KG reconciliation note, §6 OOS). The independent KG lookup surfaces `k4a-sam-backend` (medium confidence) and `sam-api`, `k4a-mcp-server`, `kochava-k4a-mcp-server`, `k4a-database-images` (all low). **No contradiction exists** — the spec makes no implementation-reality claim (no named endpoint, table, or join) that the evidence could refute. The spec's own disclosure that the primary and cross-check lists "do not fully agree" is consistent with this fresh lookup. One observation: the KG surfaces two near-duplicate MCP-server repo names (`k4a-mcp-server` / `kochava-k4a-mcp-server`), reinforcing that the "which service owns the logic" discrepancy the spec defers to design is real and should be resolved before the single-chokepoint decision (NFR-019) is finalized. This is correctly scoped as engineering ownership, not a spec defect.

---

## 6. Recommendations

1. **Close CG-002 (malformed eligibility signal):** Add an FR extending fail-closed to reachable-but-unparseable responses with a distinct "indeterminate-eligibility" status. Highest-value gap because it's the one realistic path to accidental exposure not yet covered.
2. **Close CG-003 (flag propagation consistency):** Specify that all read paths resolve the kill-switch from a shared source at the chokepoint, or explicitly bound and document the eventual-consistency window during propagation.
3. **Resolve the three open product decisions (BR-003, BR-006, BR-007):** These block end-to-end verification of FR-002, FR-009, FR-010, and the pagination contract. The mechanism can build in parallel, but QA cannot sign off the rule-value tests until these land.
4. **Add explicit FRs for mixed/partial result-set failure (Edge 5) and empty-result/all-redacted states (Edge 7):** Make the fail-closed behavior row-granular in the FR text, not all-or-nothing.
5. **Pin NFR-001's N and X against the traced baseline** and self-contain FR-013's propagation time. Convert the 50ms default from "working default" to a committed, baselined threshold before QA authoring.
6. **Define the audit contract for error/fail-closed paths (Edge 6) and audit-burst volume (Edge 8):** Clarify whether denied-everything responses are audited and add a volume guardrail that doesn't undermine NFR-002.
7. **Flag FR-014 as test-blocked on the baseline-discovery task** so it isn't marked "covered" prematurely, and reconcile FR-014's "no inconsistent redaction" with the by-design per-bucket behavior of NFR-016 staged rollout (Edge 9).