---
id: product-spec-critique
title: "Spec Critique: AIM Onboarding Decision Tool"
---

## Spec Critique: AIM Onboarding Decision Tool

**Reviewer:** QA / Red Team
**Source:** product-spec.md (v3, 3 June 2026)
**Additional context:** Gary's PR #1237 comment answers (A1–A4, B1–B10, 28 May 2026),
meeting notes 2 June 2026, Gary Slack corrections 3 June 2026
**Date:** 2026-06-04
**Verdict:** BLOCKED — 5 critical gaps must be resolved before engineering begins

---

## 1. Critical Logic Gaps

### CG-001 — §3.9 Campaign Grouping Question missing from spec [CRITICAL]

**Context:** Mayank raised §3.9 as a spec-vs-prototype discrepancy. Gary answered in PR comment A3:

> *"Yes — this was a deliberate addition, keep it in. The campaign grouping question belongs in Step 3 and should be implemented as per the prototype. The spec omitted it but the prototype is the source of truth here."*

Gary also confirmed: *"the underlying state keys `campaignMerge` / `campaignMergeChoice` are already present in the state shape — they just lacked a UI collection step. This resolves that open question from §11."*

**Gap:** The spec (§11 Open Questions, §7 State Shape) still lists `campaignMerge` and `campaignMergeChoice` as "missing wizard fields — decision needed: collected here or pulled from CRM?" Gary explicitly resolved this: **collect in the wizard, in Step 3, as per the prototype.** The open question is closed. The spec is not updated.

**Dead end:** Engineer reads §11, sees "decision needed", stops to ask. Wastes a sprint. The answer exists — it's just not in the spec.

---

### CG-002 — Executive Summary still says "four output documents" [CRITICAL]

**Spec says (§1):** *"auto-generates four output documents: a draft Scope of Work, a delivery Timeline, a filtered Data Schema, and an internal Model Config."*

**Gap:** Tab 4 (Model Config) is explicitly out of scope for Phase 1 (confirmed in June 2 meeting, correctly reflected in §4.1 Tab 4). The executive summary contradicts the feature requirements. An engineer reading top-to-bottom gets conflicting signals in the first paragraph.

**Cascading contradiction:** User Story 1 (CSM, §3) states: *"Model Config tab (Tab 4) is visible and populated in CSM mode."* This is a Phase 1 acceptance criterion that directly contradicts the Phase 1 scope decision. An engineer implementing CSM user story acceptance criteria would build Tab 4 against explicit scope.

---

### CG-003 — onboarding_status flag not specified [CRITICAL]

**Context:** Gary answered B3 explicitly:

> *"For the K4A portal to know whether to show the onboarding wizard, we need a server-side flag per advertiser account. My recommendation: introduce an `onboarding_status` field (e.g., not_started / in_progress / complete) against the advertiser record."*
>
> *"Post-completion behaviour: my preference is that the wizard tab does not disappear — it becomes a read-only record of what was agreed. What should change is the CTA: 'Continue onboarding' → 'View your onboarding record.'"*

**Gap:** None of this is in the spec. The spec has no mention of:

- How K4A determines whether to show the wizard to an advertiser
- The `onboarding_status` field or its possible values
- The CTA change from "Continue onboarding" to "View your onboarding record"
- What the wizard tab looks like post-completion (read-only vs hidden)

**Dead end:** Engineer cannot implement the portal entry point without this. K4A currently has no concept of "this advertiser has been onboarded." Without a spec for the status flag, engineers will either skip it (wizard always shows) or invent their own solution.

---

### CG-004 — Tier override not logged to audit [CRITICAL]

**Context:** Gary answered B10 explicitly:

> *"Override logging: Yes — tier overrides should be logged. Any manual change to the tier recommendation should write an entry to the Timeline audit log with: timestamp, CSM user (from session), original recommendation, and override value."*

**Gap:** The spec (§4.1 Tier Recommendation Algorithm) describes the asymmetric override logic but says nothing about logging. The Timeline audit log (§4.1 Tab 2) is described for milestone status changes and notes — tier overrides are not mentioned. An engineer implementing the tier override UI will not add audit logging because the spec doesn't require it.

---

### CG-005 — Phase 9 loader contradicts itself [CRITICAL]

**Spec says (§4.1 Step 8):** *"'Generate my AIM setup' triggers loader animation only. No backend call. No email. No notification."*

**Spec also says (§9 Technical Constraints):** *"Phase 9 loader: Currently a hardcoded 1.6-second timeout. In production, replace with a real API call to generate/save the configuration — loader duration should reflect real processing time."*

**Gap:** These two requirements directly contradict each other. §4.1 says no backend call; §9 says replace with a real API call. This is not a Phase 1 vs Phase 2 distinction — both statements are in the same spec with no phase label on §9's claim.

**Origin:** Gary's original PR comment B5 said "an automated notification must be sent on Generate." The June 2 meeting corrected this to notification-on-"Book my meeting" only. The §4.1 text was updated with the correction. The §9 text was not.

---

### CG-006 — Timeline milestone names still unspecified [CRITICAL]

**Context:** Gary answered B7:

> *"The milestone names and week durations in the spec are the default template — hardcoded as the starting point for every engagement. AIM X: 7 milestones (~7 weeks). AIM Pro: 8 milestones (~9+ weeks). These reflect Robert's actual delivery model."*

**Gap:** Gary confirms the milestone list exists and is hardcoded from Robert's delivery model — but neither the spec nor Gary's answer provides the actual names or durations. "These reflect Robert's actual delivery model" is not an engineering requirement. Tab 2 (Timeline) cannot be built without the milestone list.

**Dead end:** Frontend cannot render the timeline. Cascade delay logic cannot be implemented without knowing milestone sequence. This has been flagged in two previous critique cycles and remains unresolved.

---

### CG-007 — History threshold bug fix not reflected in spec

**Context:** Gary answered A1:

> *"The spec is correct — use 12 months. Both '12–24 months' and '24+ months' qualify for the spend path. The prototype's 24-month requirement is a bug."*

**Gap:** The spec (§4.1 Tier Recommendation Algorithm) says "monthly spend ≥ $250K AND history is 12–24 or 24+ months" — this is correct. However the spec does not call out that the prototype has this wrong and must be fixed. An engineer porting the prototype's `recommendTier()` logic directly will carry the bug forward. The fix must be explicitly stated: **do not copy the prototype's history condition — it is wrong.**

---

### CG-008 — Budget period conversion still absent from tier algorithm

**Spec says:** Tier condition 1 = "monthly spend ≥ $250K AND history..."

**Gap:** The budget slider has a Monthly/Annual toggle. If user selects Annual and enters $3,000,000, the algorithm must divide by 12 before comparing to $250K. The spec does not specify this conversion. An engineer reading only the spec will compare `budgetAnnual` directly to `250000` — which is always true for any annual budget over $250K (i.e., anything over $20.8K/month), producing rampant false Pro recommendations.

```gherkin
Scenario: Annual budget correctly converts for tier check
  Given user selects Annual budget toggle
  And enters $2,400,000 annual (= $200K/month)
  And no other Pro conditions are met
  When output screen loads
  Then tier recommendation shows "AIM X"
  And effectiveMonthlyBudget = 200000
```

---

### CG-009 — "Book my onboarding meeting" button has no failure state

**Spec says:** Clicking sends email to CSM team.

**Gap:** Zero failure handling specified. No confirmation state. No error state. No retry. Recipient list strategy still unresolved (hardcoded emails vs configurable). If email service is down, user gets no feedback. CSM never gets notified. Onboarding stalls silently.

---

### CG-010 — PDF export scope not defined

**Spec says:** "PDF download is a real Phase 1 implementation."

**Gap:** No spec on: which sections are included, page layout, approach (print stylesheet vs headless Chrome), how inline notes are handled, whether empty optional sections appear, what happens with very long free-text entries. "Real implementation" is not a requirement.

---

## 2. Testability Review

### TR-001 — "Wizard steps load instantly"

**Original (§10):** *"Wizard steps load instantly (client-side only in Phase 1)."*

**Verdict:** FAIL — "instantly" is not measurable.

**Rewrite:**

```gherkin
Given user is on any wizard step
When user clicks Next or navigates via stepper
Then the target step renders within 100ms
```

---

### TR-002 — "under 10 minutes" for wizard completion

**Original (§3 Client User Story):** *"complete an onboarding questionnaire in under 10 minutes"*

**Verdict:** FAIL — not verifiable in automated tests. Remove from acceptance criteria or convert to a UX research benchmark, not a functional requirement.

---

### TR-003 — "near-zero" CSM effort

**Original (§2 Success Metrics):** *"AIM X onboarding touches (CSM + Engineering) — Target: Near-zero"*

**Verdict:** FAIL — not measurable.

**Rewrite:** Target: ≤ 1 CSM action per AIM X onboarding (CSM Approval button only). No other CSM input required.

---

### TR-004 — CSM User Story Tab 4 acceptance criterion

**Original (§3):** *"Model Config tab (Tab 4) is visible and populated in CSM mode"*

**Verdict:** FAIL — directly contradicts Phase 1 scope. Tab 4 is Phase 2. This criterion cannot be tested in Phase 1 because Tab 4 will not be built.

**Action:** Remove from Phase 1 acceptance criteria. Move to Phase 2 user story.

---

### TR-005 — "Output screen generation < 500ms"

**Original (§10):** *"Output screen generation < 500ms (all client-side computation)."*

**Verdict:** PASS as written but needs Gherkin context:

```gherkin
Given a fully-completed wizard session (all fields populated, all steps visited)
When Phase 10 initialises after the 1.6s animation completes
Then Tab 1 (Scope of Work) is fully rendered within 500ms
```

---

## 3. Risk Mitigation Audit

| Risk | Acknowledged in spec? | Mitigation specified? | Verdict |
|---|---|---|---|
| Budget algorithm reads wrong field (`state.spend`) | ✅ §12 | Fix noted, no Gherkin, no enforcement | ⚠️ PARTIAL |
| Prototype history threshold bug (24 months not 12) | ❌ Not in spec | None | ❌ FAIL |
| Budget period conversion (annual ÷ 12) | ❌ Not in spec | None | ❌ FAIL |
| Organic/paid split always 50/50 | ✅ §12 | Fix noted, no test | ⚠️ PARTIAL |
| Seasonality always shows "Included" | ✅ §12 | Fix noted, no test | ⚠️ PARTIAL |
| Tier eligibility always AIM X | ✅ §12 | Fix noted, no test | ⚠️ PARTIAL |
| Email delivery failure (CSM notification) | ❌ Not mentioned | None | ❌ FAIL |
| localStorage quota exceeded | ❌ Not mentioned | None | ❌ FAIL |
| Super Admin role client-side only | ❌ Not mentioned | None | ❌ FAIL |
| Funnel reset → stale LTV state | ❌ Not mentioned | None | ❌ FAIL |
| Tier override not logged | ❌ Not mentioned (Gary confirmed it must be) | None | ❌ FAIL |
| §3.9 campaign grouping state keys orphaned | ❌ Listed as open question (already resolved by Gary) | None | ❌ FAIL |
| onboarding_status flag missing | ❌ Not in spec | None | ❌ FAIL |
| Phase 9 loader contradiction | ❌ Not mentioned | None | ❌ FAIL |

---

## 4. Missed Edge Cases

### EC-001 — Campaign grouping state never collected, always orphaned in output

Gary confirmed `campaignMerge` / `campaignMergeChoice` belong in Step 3 §3.9. If not added, Model Config (Tab 4, Phase 2) and any campaign merge output in Scope of Work will always be empty — silently, with no error. The prototype populates these only in demo mode (`BRIGHTFIT_DEMO`). Real clients will always get blank campaign merge fields in their output.

---

### EC-002 — Advertiser with no `onboarding_status` flag sees wizard on every login

Without Gary's recommended `onboarding_status` field, K4A has no way to know whether an advertiser has completed onboarding. Every login shows the wizard. A client who completed onboarding last month logs in to check something and is presented with a blank Step 1 form. "Continue saved progress" only works if localStorage is intact — cleared browser, new device, or incognito session = wizard resets silently.

---

### EC-003 — Tier override logged where? Timeline may not exist yet

Gary said tier overrides write to the Timeline audit log. But the timeline is only anchored after SoW approval (`approvedAt`). What if a CSM overrides the tier recommendation on the output screen **before** the client approves the SoW? The Timeline tab exists but has no anchor date. Does the audit log entry still write? To what? Phase 1 is localStorage-only — if the user refreshes before approving, is the override log lost?

---

### EC-004 — Campaign grouping question (§3.9) resets on business model change

Step 4 resets the funnel when business model changes. §3.9 is in Step 3 (Marketing Setup). However `campaignMerge` / `campaignMergeChoice` describe how campaigns are grouped in the model — this may need to reset or be re-evaluated when the business model changes (e.g., Gaming vs Subscription may have different campaign merge implications). The spec does not specify whether §3.9 answers persist or reset when the user changes business model in Step 4.

---

### EC-005 — "View your onboarding record" CTA never specified

Gary said (B3): post-completion, the CTA changes from "Continue onboarding" to "View your onboarding record." This implies the wizard transitions to a read-only state. But:

- What exactly is read-only? Just the output tabs? The wizard steps too?
- Can the CSM still update timeline statuses in read-only mode?
- If the client clicks "View your onboarding record" and the SoW was never approved, is it still editable?
- Does "View your onboarding record" even exist as a UI element anywhere in the spec? No.

---

## 5. Summary

| Category | Count | Blocking? |
|---|---|---|
| Critical Logic Gaps | 10 | 5 are full blockers (CG-001, CG-003, CG-004, CG-005, CG-006) |
| Testability failures | 5 | TR-004 must be removed before engineering starts |
| Risk mitigations missing | 9 of 14 | 7 new risks not acknowledged |
| Missed edge cases | 5 | EC-001 is a silent data bug in every real client session |

**Must resolve before engineering begins:**

1. **CG-001** — Add §3.9 campaign grouping to Step 3 spec. Remove `campaignMerge`/`campaignMergeChoice` from "open questions" — Gary resolved this in PR comment A3.
2. **CG-002** — Fix executive summary (three Phase 1 outputs, not four). Remove Tab 4 from CSM User Story Phase 1 acceptance criteria.
3. **CG-003** — Specify `onboarding_status` field, portal entry point logic, and post-completion CTA.
4. **CG-004** — Add tier override logging requirement to Tab 2 and Tier Algorithm sections.
5. **CG-005** — Remove "replace with a real API call" from §9 Technical Constraints. Phase 9 loader = marketing animation only (June 2 meeting decision).
6. **CG-006** — Provide actual milestone names and week offsets for AIM X (7) and AIM Pro (8). Tab 2 is unimplementable without this.
7. **CG-008** — Add budget period conversion formula to tier algorithm.
