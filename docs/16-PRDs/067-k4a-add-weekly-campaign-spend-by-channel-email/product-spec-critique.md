# Spec Critique: Meridian Travel Weekly Spend-by-Channel Summary

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-21

## Summary

- **Critical Issues:** 2
- **High Issues:** 6
- **Medium Issues:** 7
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | The spec mandates building against a confirmed service topology (NFR-013) and states "No build work should proceed against an unconfirmed service map" — but the entire spec is written atop an *unresolved* two-cluster ambiguity. Every FR depends on services (aggregation, scheduler, notification, entitlement) whose real repositories are unknown. The fresh independent KG cross-check re-confirms ONLY the SAM-oriented cluster (k4a-sam-backend, sam-report-scheduler, sam-performance-reporter, sam-api, sam-campaign-data-sync/k4a-sam-campaign-data-sync, reporting-subscriptions-worker, notifications-api) — it did NOT re-find the reporting-api / reporting-dispatcher / accounts-api / frontend-mos / k4a-mcp-server cluster at all. This widens, rather than resolves, the Section 1 ambiguity: the only service appearing in all three lookups is notifications-api. Engineering cannot start; FR-001 (entitlement via "authoritative account source") in particular has NO confirmed backing service — accounts-api was named by only one of three lookups. | Critical | 1 / NFR-013 / Appendix | Resolve the topology per NFR-013 before engineering sign-off. Explicitly note in Section 1 that a third independent lookup failed to re-find the reporting/accounts cluster and re-confirmed only the SAM cluster + notifications-api, so the entitlement-source service (FR-001/FR-003) is currently ungrounded. |
| CG-002 | FR-004 and FR-015 enforce authorization "when the dashboard link is followed" and assume the K4A dashboard already implements server-side viewer-authorization re-derivation. The spec asserts "no new UI build" and treats the dashboard as a pre-existing, already-secured surface — but provides no requirement confirming the existing dashboard actually enforces per-request entitlement today. If it does not, FR-004/FR-015 are net-new work falsely scoped as free, and a 403 path that doesn't exist is a security gap. BR-3 further states "no extra access controls beyond membership of that distribution list," which directly contradicts FR-004/FR-015/NFR-008 requiring per-request 403 authorization. | Critical | 3 (FR-004/FR-015) / 1 (BR-3) / 5 (NFR-008) | Add an FR asserting/verifying the existing dashboard's current authorization behavior. Reconcile BR-3 ("no extra access controls") against FR-004/FR-015 ("reject ineligible viewer with 403") — these cannot both be true as written. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-008a | "a prior-week comparison can be derived **at low implementation cost**" / "if the prior-week comparison is **not low-effort** to add, it is omitted" | "Low-effort" / "low implementation cost" is a developer-time judgment, not a runtime condition. It cannot be tested by QA; the same input can pass or fail depending on a human's estimate. A conditional-on-effort requirement is untestable and should be a planning decision, not an FR branch. | `Given the prior week's summary exists for the customer` `When the current week's summary is generated` `Then the summary includes, per channel, the prior-week spend value and the current-vs-prior delta` — and move the "include it or not" decision to eng-plan as a binary scope decision, removing the effort-based branch from the FR. |
| FR-009 | "handled **per the engineering-chosen approach (deferred to engineering discretion)**" | A functional requirement that defers the actual observable behavior to engineering is not a requirement and not testable. QA cannot write an assertion against "engineering's choice." | `Given a channel has missing or incomplete spend data for the covered week` `When the summary is generated` `Then that channel is displayed with an explicit "data unavailable" marker distinct from a zero-spend value` (pick a concrete, assertable behavior). |
| NFR-001 | "must return within a **p95 latency target** ... (e.g. N concurrent requests, p95 ≤ X ms)" | Placeholder symbols N and X. Nothing testable until values are set. | `Given 50 concurrent dashboard requests` `When the spend-by-channel view is requested` `Then p95 response time ≤ 800ms` (fill real numbers in eng-plan and treat as blocking for NFR sign-off). |
| NFR-003 | "generated and delivered within a **bounded window**" / "exact bounded delivery-window target ... is an engineering decision" (DEC-006) | "Bounded window" is undefined; the core value prop ("predictable cadence") has no measurable SLA. | `Given the Monday 08:00 trigger fires for an entitled customer` `When generation and delivery run` `Then the email is delivered by 08:30 local time` (set the concrete minute target). |
| NFR-010 | "Minimum retention: 12 months. [DECISION REQUIRED: DEC-007]" | Retention stated but flagged as still an open decision — contradictory (is it 12 months or TBD?). | Resolve DEC-007; if 12 months is final, remove the DECISION REQUIRED tag. |

Note: All FRs are in Gherkin form (FR-009 has a trailing prose note, which is acceptable as an annotation, not the requirement body). The untestability above is about *content* (vague/deferred conditions), not format.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Multi-service blast radius | Yes — NFR-011 kill-switch (config-disable, 5-min propagation) | PASS | Well-specified, including in-flight-run behavior. |
| Customer-facing email reliability | Yes — FR-011–FR-013, NFR-004, NFR-005 | PASS | Bounded retry, named failure statuses, 1%/5min alerting. Solid. |
| Service-topology ambiguity / unclear technical design | Partially — NFR-013 says "confirm before building" | FAIL | A "don't build until someone confirms" mitigation is a gate, not a mitigation of the underlying design risk. The fresh KG cross-check makes this worse (only notifications-api common to all three lookups). No fallback or decision owner/date is named. This is the CG-001 blocker. |
| Hard 5-week deadline | Partially — NFR-014 defers scope | FAIL | NFR-014 claims deadline is mitigated "only if remaining open scope decisions (recipient resolution, cadence) are resolved early" — yet BR-1 fixes cadence (Monday 8am) and BR-3 resolves recipient. So NFR-014 references decisions already resolved elsewhere, while NFR-003/DEC-006 (delivery window) and DEC-007 (retention) remain OPEN. The mitigation points at the wrong open items. Reconcile. |
| Rollback safety | Yes — NFR-015 | PASS | Clear: disables future runs, does not revert delivered emails. |
| Entitlement source unreachable (fail-closed) | Yes — FR-003, NFR-008 | PASS | Fail-closed + alert is specified. |
| Backward compatibility with existing reporting/notification consumers | Partially — NFR-016 | FAIL (weak) | NFR-016 says existing consumers "must remain unchanged" but given the unconfirmed topology (CG-001) there is no way to enumerate existing consumers, so the requirement is unverifiable today. Depends on NFR-013 resolution. |

---

## 4. Missed Edge Cases

1. **Data freshness / upstream sync lag:** The SAM-cluster KG evidence names `sam-campaign-data-sync` / `k4a-sam-campaign-data-sync` as "a potential upstream data-freshness dependency." No FR addresses the case where the weekly aggregation runs *before* the upstream spend sync has completed for the covered week — this would produce a plausible-looking but silently stale/incomplete summary that FR-009/FR-010 would not catch (data looks "present," just old). This is a correctness risk, not just completeness.
2. **Partial-week / late-arriving spend after summary sent:** Spend data for the covered week arriving after Monday 8am generation. Idempotency (FR-007) overwrites to the same end state, but there's no rule for whether a corrected restated summary is re-sent or the stale email stands.
3. **Distribution-list membership changes mid-run:** FR-011 resolves the recipient to a list controlled by the marketing ops lead. If membership changes between generation and delivery, which snapshot governs? Also: empty distribution list (zero recipients) — is that a delivery success, a skip, or an alert?
4. **Partial email delivery (some recipients bounce, some succeed):** FR-012/FR-013 treat delivery as a single success/fail. A distribution list where 3 of 10 addresses hard-bounce is undefined — is the run "delivered" or "failed"? NFR-004 alerts on "customer's summary fails to deliver after all retries" but doesn't define per-recipient failure.
5. **Timezone of "Monday 8am":** BR-1/NFR-003 say "Monday at 8am" with no timezone. Meridian Travel's local time vs. system UTC changes the covered-week boundary and delivery expectation. DST transitions unhandled.
6. **"Last completed week" boundary ambiguity:** "Covered week" = "last completed week" but no definition of week start (Sun vs Mon) or how it interacts with the Monday 8am trigger (does Monday's run cover the prior Mon–Sun, or Sun–Sat?).
7. **Kill-switch flipped mid-generation but pre-delivery:** NFR-011 says an in-flight run "completes under prior behavior." Does "complete" include *delivering* the email? A disabled feature still sending an email during an incident may be exactly what the on-call wanted to stop.
8. **Entitlement changes between generation and delivery:** FR-001 checks entitlement at generation; the customer could be downgraded before the email sends. No re-check at delivery time is specified.
9. **Currency / multi-currency spend:** "Per-channel spend figure" has no currency handling. If channels report in different currencies, summing/comparing is wrong. (Attribute types deferred to eng-plan, but the *existence* of a currency dimension is a business question.)
10. **Prior-week comparison when prior week had missing data:** FR-008a assumes a clean prior-week figure. If the prior week's channel data was incomplete (FR-009), the comparison baseline is distorted — undefined behavior.
11. **Idempotency key collision across customers:** FR-007/NFR-006 key on (customer + covered week). No handling specified for a customer whose account identity itself changes/merges.
12. **Audit write failure during a successful delivery:** NFR-010 says audit writes are async and a failure fires an alert — but does a persistent audit-write outage block/halt generation, or do emails keep sending unaudited indefinitely? For a HIGH-risk, compliance-relevant feature this needs a decision.
13. **Dashboard link longevity / staleness:** The email links to the existing K4A dashboard showing "spend-by-channel data." If a recipient opens last month's email, does the link show *that* week's data or current data? No deep-link-to-covered-week requirement.
14. **Retry-After honored but exceeds delivery window:** FR-013/NFR-005 honor Retry-After on a 429; if the provider's Retry-After pushes delivery past NFR-003's bounded window, which requirement wins? Conflict undefined.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-015 (incl. FR-008a) are all in Gherkin. FR-009's trailing prose is an annotation note, not the requirement body — acceptable. |
| Document contains only Sections 0–6 | FAIL | The document contains a full "## Appendix: Engineering Reference" — a multi-paragraph narrative service-map analysis with two lookup clusters, a Reconciliation prose paragraph, and implementation-direction findings. This is exactly the stray multi-paragraph KG cross-check analysis appendix that the format rule is written to catch; it is NOT a one-line labeled metadata trailer and NOT the sanctioned Revision Summary. The trailing "## Revision Summary" section itself is EXPECTED and correctly present (not flagged). The Appendix preceding it is the violation. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is entirely bullet-list prose with no attribute tables or type declarations; types are explicitly deferred to eng-plan. |

---

## 6. Recommendations

1. **Resolve the service-topology blocker (CG-001 / NFR-013) before any engineering start.** A third independent KG lookup re-confirmed only the SAM cluster + notifications-api and did not re-find the reporting/accounts cluster at all. Have an engineer who knows K4A/SAM confirm the real repositories, *and critically* name the actual entitlement-source service backing FR-001/FR-003, which is currently ungrounded. Update Section 1 to record the three-way discrepancy.
2. **Reconcile the BR-3 vs FR-004/FR-015/NFR-008 contradiction (CG-002).** Decide whether per-request 403 viewer-authorization is in scope. If yes, add an FR confirming the existing dashboard already enforces it (or scope the new enforcement work honestly). If BR-3's "no extra access controls" is authoritative, remove the 403 viewer-authorization requirements.
3. **Remove effort-conditional and discretion-deferred logic from FRs.** Rewrite FR-008a (drop the "low-effort" runtime branch) and FR-009 (specify the concrete missing-data display behavior) so QA can write deterministic tests.
4. **Set concrete values for NFR-001 (N, X), NFR-003/DEC-006 (delivery window), and resolve DEC-007 (retention).** Treat these as blocking for NFR sign-off, not post-hoc eng decisions, since they define the feature's core SLA.
5. **Fix NFR-014's deadline mitigation** to reference the *actually* open decisions (DEC-006, DEC-007) rather than cadence/recipient, which are already resolved by BR-1/BR-3.
6. **Add FRs for the high-severity missed edge cases**, prioritizing: upstream data-freshness/staleness (Edge 1), partial/per-recipient email delivery (Edge 4), timezone + covered-week boundary definition (Edges 5–6), and entitlement re-check at delivery (Edge 8).
7. **Move the Engineering Reference appendix** out of the product-spec body into a separate engineering planning artifact to satisfy the Sections 0–6 structure, keeping only the Revision Summary as the trailing section.