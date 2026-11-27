# Spec Critique: AIM Onboarding Decision Tool

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-27

## Summary

- **Critical Issues:** 2
- **High Issues:** 7
- **Medium Issues:** 9
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **"Book my onboarding meeting" notification has no failure path.** FR-094 / FR-081 state an email is sent to the onboarding team when the client books. There is NO defined behavior if the email send fails. NFR-016 mentions bounded retry/backoff for outbound calls but no FR defines what the client sees, whether the booking is marked pending, or whether the offline-meeting gate can ever unlock if the notification silently failed. A lost notification stalls the entire approval chain with no recovery. | Critical | §3 FR-094, §5 NFR-016 | Add an FR: Given the onboarding notification send fails, When retries are exhausted, Then the client sees a confirmed/deferred booking state and an operational alert is raised so the meeting can still be scheduled. |
| CG-002 | **Audit/approval persistence failure posture is still an open DECISION, not a requirement.** NFR-009 carries `[DECISION REQUIRED: DEC-002]` for what happens when a change-log/approval record fails to persist. Approval formalizes a commercial scope signoff; losing it silently is a data-integrity/traceability risk. An unresolved decision on a legally-flavored signoff record blocks implementation — engineering cannot build the approval write path without knowing the failure contract. | Critical | §5 NFR-009 | Resolve DEC-002 before build. Define: approval write is atomic; on persistence failure the approval is NOT recorded as successful and the user is told to retry, with an operational alert. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "...must render without perceptible delay for a single session's client-side interaction." | "Perceptible delay" is not measurable; no threshold. | Given a user changes an answer on any step, When the routing flags and model-count recompute, Then the UI reflects the new value within 100ms at p95. |
| NFR-002 | "...reflect real processing time rather than a fixed artificial delay once a backend generation step exists." | "Once a backend generation step exists" makes the clause untestable in Phase 1 (no backend). Ambiguous conditionality. | Given generation runs client-side in Phase 1, When the user clicks Generate, Then the indicator is shown for the actual compute duration and the output renders within 500ms p95; mark the backend-timing clause as a Phase-2 testable requirement. |
| FR-096 | "approximately 7 weeks minimum" / "approximately 9 or more weeks" | "Approximately" is not testable; cascade math needs exact anchors. | Given an AIM X plan, When the Timeline renders, Then exactly 7 milestones are shown and the final milestone start date is ≥ 7 calendar weeks after the kickoff anchor (next-Monday rule applied). |
| FR-081 | "a brief generation indicator is shown" | "Brief" is subjective. | Given generation completes in < 500ms, When the output is ready, Then the indicator is shown for the real processing time and dismissed on render. |
| NFR-007 | "mask to the last few characters only" | "A few" is not a fixed count; inconsistent masking is untestable. | Given a value must be shown for identification, When it is rendered, Then all but the last 4 characters are masked. |

*Note: all FRs are in Gherkin — this section flags vagueness within otherwise well-formed FRs, not prose violations.*

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Scope creep at source | Yes — NFR-013, FR-095 lock + CSM re-approval | PASS | Deterministic SoW before engineering, post-approval lock. Well covered. |
| Zero-touch integrity breaking the AIM X scaling model | Yes — NFR-012 (hard constraint, must be verified) | PASS | Explicit gate. |
| Brand ad-stock / mixed-network caveat | Yes — NFR-014 → FR-092 | PASS | Caveat surfaced wherever brand present. |
| Non-blocking validation letting incomplete config reach approval | Yes — NFR-015 → FR-004 | PASS | Output-screen full validation with jump-back. |
| Identity source unreachable (role/mode resolution) | Yes — NFR-016 fail-closed, NFR-005 | PASS | Fail-closed posture is explicit. |
| Legacy/unresolved onboarding-flow ownership (which backend owns the trigger) | Partial — Appendix flags it, but DEC-001 (multi-region) and the flow-ownership question are unresolved | **FAIL** | The spec acknowledges the ownership overlap (`aim-customers-onboarding-initialization` vs `aim-workflow-manager`) but defines NO requirement or fallback for how the tool behaves if the downstream owner is undetermined at build time. An acknowledged risk with no mitigating requirement = fail. |
| Audit record loss (approval/change-log) | No — gated behind DEC-002 | **FAIL** | NFR-009 names the risk but defers the mitigation to an unresolved decision. See CG-002. |
| Notification/booking delivery failure | Partial — NFR-016 says bounded retry, no FR for user-facing recovery | **FAIL** | See CG-001. The risk posture exists at NFR level but no functional requirement operationalizes the recovery/visible state. |

---

## 4. Missed Edge Cases

1. **Booking notification failure (no recovery path):** Covered in CG-001 — the approval chain can permanently stall.
2. **"Unlock to edit" after downstream ETL has begun:** FR-095 lets a CSM clear approval and reopen scope. The flow (Schema → ETL) shows data already sent / ETL configuration started. Nothing defines what happens to in-flight ETL or an already-anchored Timeline if scope is reopened and changed. Silent scope drift against live engineering work — the exact failure the tool exists to prevent.
3. **Tier downgrade/upgrade race with auto-recomputation:** FR-091 says AIM X can only upgrade when the auto recommendation is Pro, and Pro can be manually downgraded. But answers remain editable until approval. If a CSM manually downgrades to X, then the client edits spend upward (auto → Pro), is the manual downgrade preserved or overwritten? Undefined.
4. **Conversion share auto-adjust below zero / rounding:** FR-021 three-platform proportional redistribution can drive a share negative or produce rounding that doesn't total exactly 100%. No clamp or rounding rule defined.
5. **Campaign budget split proportional redistribution with a zeroed type:** FR-035 "redistributes proportionally" — if the other active types are at 0%, proportional redistribution is a divide-by-zero. No rule.
6. **"Same as project lead" sync then project lead removed:** FR-010 syncs data leads from project leads in real time. If a synced project lead row is deleted, behavior of the dependent data-lead row is undefined.
7. **Primary KPI event deleted/renamed to empty:** FR-041 requires exactly one KPI. If the user renames the KPI event's name field to blank, or toggles off the event holding the KPI, the "exactly one KPI" invariant may break with no defined recovery.
8. **Business model change wiping a set KPI silently:** FR-040 resets the funnel on business-model change. A user who carefully set a KPI loses it with (per spec) no confirmation. Data-loss UX gap.
9. **Auto-save scope is same-browser-session only:** FR-005 restores "later in the same browser session." Phase 1 persists client-side (§6). Clearing cookies / switching device loses everything with no warning — directly undercuts the save-and-resume value prop for a multi-day onboarding.
10. **Concurrent CSM + client editing the same session:** NFR-003 isolates sessions across users, but CSM mode "completes or reviews on behalf of a client" on the same session. No last-write-wins / locking rule for simultaneous CSM and client edits.
11. **Validation banner jump-back after approval lock:** FR-004 jump-back links navigate to steps, but FR-095 removes the questionnaire after approval. Edge: can a validation banner ever appear once the form is hidden? Interaction undefined.
12. **Region switcher mid-approval integrity:** FR-103 allows switching between region sessions at different approval stages. No rule preventing an approval action being applied to the wrong region's session when switching rapidly.
13. **Annual/Monthly toggle and the tier threshold:** Tier logic (FR-091) checks "monthly spend ≥ $250K." If the user entered an annual budget, the spec never defines the monthly-equivalent conversion used for the threshold. Ambiguous computation.
14. **"Other/none" MMP with zero MMP data at all:** FR-051 injects a "to be confirmed" milestone, but if no MMP exists, much of the attribution-dependent schema/funnel logic may have no source. No defined degraded schema.
15. **WCAG focus management on validation banner + jump-back:** NFR-010 requires focus management on step transitions; the banner→step jump is a transition not explicitly covered.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-103 are all in fenced `gherkin` blocks. |
| Document contains only Sections 0–6 | PASS | Sections 1–6 present. The trailing **Appendix: Engineering Reference** is a multi-section narrative appendix with its own prose and a table — this is NOT a sanctioned Revision Summary or a one-line labeled metadata blockquote. However, it is explicitly fenced as auto-derived KG reference "not required reading for product/requirement review" and the document under review formally terminates at the `=== END OF DOCUMENT ===` marker; treating it as the base-skill "stray KG cross-check appendix" negative example, it is borderline. Given it is clearly delimited, auto-generated, and labeled non-normative, it is recorded here as an observation but does not read FAIL — it is the grounding appendix the spec's Section 1 cross-check note explicitly references ("Per-service grounding evidence is captured in a separate appendix"). |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is bullet lists with bolded labels and provenance tags (User-Entered / System-Derived / External); no attribute tables or field types. Types are explicitly deferred to engineering, consistent with the rule. |

*Note on the Appendix:* It is the one piece of trailing content that is neither a Revision Summary nor a one-line `> **Label:**` metadata blockquote. It is a genuine multi-section prose+table appendix. Per the base skill this is exactly the "stray KG cross-check appendix with its own findings/recommendations" shape that should normally be flagged. It is held at PASS-with-observation only because (a) it is explicitly declared non-normative engineering reference, (b) Section 1 forward-references it by design, and (c) it carries no product requirements or findings/recommendations prose that could leak into scope. Recommend the author confirm this appendix is intended to ship in the doc; if product-review cleanliness is required, relocate it to a linked engineering doc. This is a process recommendation, not a format FAIL.

---

## 6. KG / Implementation-Reality Cross-Check

Reconciling the spec's claimed affected services (Appendix + Section 1 note) against the independent fresh KG lookup:

- **No contradiction found.** The fresh lookup re-finds the same six services (`aim-portal-ui`, `aim-portal-api` high; `aim-customers-onboarding-initialization` medium; `aim-reference-data-api`, `aim-workflow-manager`, `frontend-mos` low) at the same confidence levels the spec's Appendix claims. The spec honestly represents the KG evidence.
- **Unresolved ownership (flagged, not resolved):** Both the spec and the fresh lookup leave the onboarding-flow ownership overlap between `aim-customers-onboarding-initialization` and `aim-workflow-manager` unresolved. The spec correctly surfaces this as an open grounding question. This is appropriately NOT silently resolved — but see Risk Mitigation FAIL: no requirement defines tool behavior given the ambiguity.
- **Existing PRD alignment:** PRD-061 (§9 Technical Constraints) is the established precedent and is consistent with the spec's framing. No contradiction in data flow, joins, config, or claimed endpoints — the spec makes no symbol-level claims that the KG contradicts (all attributions are role-level and hedged).

---

## 7. Recommendations

1. **Resolve DEC-002 (CG-002):** Define the atomic write + failure contract for approval and change-log persistence before build. This is a data-integrity blocker on a commercial signoff record.
2. **Add a booking-notification failure/recovery FR (CG-001):** Define the client-visible booking state and operational alert when the onboarding-team notification fails, so the approval chain cannot permanently stall.
3. **Define the "Unlock to edit" interaction with downstream ETL (Edge #2):** Specify what happens to in-flight ETL and the anchored Timeline when an approved scope is reopened and changed — this protects the core anti-scope-creep value.
4. **Specify the annual→monthly conversion for the tier threshold (Edge #13):** FR-091 checks monthly spend but FR-030 allows annual entry; define the exact conversion.
5. **Add clamp/rounding rules for proportional share redistribution (Edges #4, #5):** Prevent negative shares, divide-by-zero on zeroed types, and non-100% totals in FR-021 and FR-035.
6. **Quantify all vague NFR/FR thresholds (Section 2):** Replace "perceptible delay," "approximately," "brief," and "last few characters" with measurable values.
7. **Define concurrent CSM+client edit behavior on a shared session (Edge #10):** Add a locking or last-write-wins rule; NFR-003 only covers cross-user isolation, not same-session collaboration.
8. **Add confirmation on destructive resets (Edges #7, #8):** Warn before business-model change wipes a configured funnel/KPI.
9. **Clarify auto-save durability expectations (Edge #9):** Make explicit that Phase 1 save/resume is browser-session-local and warn the user, given multi-day onboarding.
10. **Confirm whether the Engineering Reference appendix ships in the review doc** or should be relocated to a linked engineering-only document.