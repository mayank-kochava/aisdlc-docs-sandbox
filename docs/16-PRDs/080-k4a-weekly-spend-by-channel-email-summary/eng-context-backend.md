# Engineering Notes: K4A Weekly Spend-by-Channel Email Summary - Backend

**Engineer:** (unassigned — backend)
**Date:** 2026-11-22
**Based on:** product-spec.md (Type B confirmed) + Read-verified repo investigation across 8 candidate services

---

## Initial Thoughts

OK, I read the spec AND I actually went and looked at all the candidate repos. The spec's "engineering grounding caveat" is the whole ballgame here and it's worse than the spec makes it sound. Two KG lookups disagreed on ownership; I investigated all 8 named services and the honest summary is: **nothing is a clean fit, and the two "product-specific" repos that were supposed to own the new logic are literally empty stubs.**

What's real and confirmed from the investigation:
- `sam-report-scheduler` — this IS a working scheduled-email-delivery service (generator → Cloud Tasks → runner → SendGrid). GCP Cloud Function, Node 18. Weekly cadence already exists (`repeatType===7` + `days:[2]` = Tuesday). This is the most reusable thing we have.
- `cost-v2` — real spend/cost ingest pipeline (Go, 40+ ad networks → BigQuery). But it's write-side only, no read/query path, and already delegates async state to `netwrkcost-api`.
- `sam-campaign-data-sync` — real ASA spend data lives here (Mongo `localSpend` + MySQL mirror), has a spend-summing pattern, has an entitlement-ish check. But ASA-only = one channel, not four.
- `k4a-sam-backend` — **empty stub.** Two-line README, one commit. No code.
- `k4a-sam-campaign-data-sync` — **empty stub.** Same.
- `sam-performance-reporter` — **greenfield Go skeleton.** CI + Dockerfile only, zero app code.
- `reporting-subscriptions-worker` — wrong domain (subscriptions/MRR, not marketing spend). Ruled out. Good reference template only.
- `notifications-api` — in-product UI banners. Totally unrelated. Ruled out.

Gut feeling on complexity: **HIGH, and the uncertainty is the hard part, not the code.** The actual feature (sum 4 numbers, format 2 columns, email it weekly) is trivial. The problem is we have NO service that can produce a 4-channel (Email/Paid Search/Paid Social/Display) spend breakdown today, and we have a 5-week hard deadline. That math worries me.

---

## Concerns & Risks

- **The four-channel breakdown has no home.** This is the biggest one. BR-4 wants Email, Paid Search, Paid Social, Display. Every spend source I found is single-platform or has a *different* meaning of "channel": `cost-v2` has a free-form per-row `channel` string (ad-network dimension, not the 4 marketing categories); `sam-campaign-data-sync` is Apple Search Ads only (one slice). **Nobody holds a confirmed cross-channel spend-by-marketing-channel view.** BR-8 says these numbers must match what the customer sees elsewhere — but I can't even confirm where "elsewhere" is.
- **Two of the named owner repos are empty stubs, one is a greenfield skeleton.** If the plan is "build it in `k4a-sam-backend`," that's net-new from literally nothing in 5 weeks, including scaffolding + tests from scratch. No existing pattern to inherit in those repos.
- **Timezone gap in the one working scheduler.** `sam-report-scheduler` is UTC-only — `moment-timezone` is a declared dep but unused; day match is `getUTCDay()`. BR-1 / NFR-001 want "Tuesday 9am US Eastern, observing DST." As-is that's not satisfiable; either add TZ handling or encode a UTC hour that tracks EST/EDT (DST risk, ugly).
- **No idempotency / duplicate-send guard anywhere.** `sam-report-scheduler` has NO lock/dedup key (`uuid` dep declared but unused), and Cloud Tasks is at-least-once with retries → amplifies duplicate risk. FR-002/FR-003/NFR-008/NFR-009 all need a run-ledger keyed on (customer + reporting window). That's net-new.
- **No entitlement check exists in the delivery path.** `sam-report-scheduler` has zero plan/entitlement logic. `sam-campaign-data-sync` has a subscription-cancelled check but no plan-tier/feature-entitlement field. FR-014/BR-6/NFR-005 (fail-closed) are net-new and need an authoritative entitlement source (accounts-api? not confirmed).
- **Weak failure handling vs. the spec flow.** `sam-report-scheduler` logs errors and often swallows them; there's a latent `send[0]` crash when SendGrid rejects. No bounded app-level retry (relies on Cloud Tasks config I couldn't read). No alerting (`grep alert|pagerduty|opsgenie` → nothing). FR-006/FR-007/FR-012/NFR-003/NFR-010 need real work: distinct failure conditions, 429 Retry-After honoring, operational alerts.
- **No audit trail primitive.** Nobody has the append-only immutable run audit (FR-017/NFR-012). `reporting-subscriptions-worker` has a mutable DB status column, not append-only. Net-new.
- **Kill-switch (FR-016/NFR-013) doesn't exist** in any delivery path. Needs config-driven disable, <5 min to stop new runs. Net-new.
- **Security landmines in the reusable repos.** Hardcoded SendGrid API key in BOTH `sam-report-scheduler` (`email-service.js:5`) and `reporting-subscriptions-worker` (`email.go:54`). If we reuse these, do NOT propagate that. NFR-006 (no spend figures / recipients / credentials in logs) also needs attention.
- **BR-8 integrity: don't read the float mirror.** In `sam-campaign-data-sync` the MySQL `spend` column is `REAL` (float) and built via non-parameterized SQL string interpolation. For customer-facing figures the Mongo `localSpend` string is the source of truth. Reading the wrong one is a correctness defect, not cosmetic.
- **Currency assumption.** Spend in the ASA data carries a currency and amount-as-string. BR-2 assumes USD to 2dp. If any org data is non-USD we have a problem — need to confirm Meridian's data is USD.
- **5-week deadline + unresolved ownership = real schedule risk.** The spec itself says the deadline is only mitigated if ownership is resolved BEFORE build. It isn't resolved. This is the critical path.

---

## Questions & Answers

### Q1: Where does the authoritative four-channel (Email / Paid Search / Paid Social / Display) spend-by-marketing-channel data actually live?
- **For:** PM / Architect / Data
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Blocks everything. No investigated repo produces this breakdown — `cost-v2`'s "channel" is an ad-network field, `sam-campaign-data-sync` is ASA-only (one channel). FR-004/FR-005/BR-4/BR-8 cannot be built until we know the source of truth that matches "the customer's other spend views."

### Q2: Is `k4a-sam-backend` the intended greenfield home for the new report-generation + orchestration logic, or a mis-identified repo?
- **For:** PM / Architect
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** It's an empty stub (two-line README, one commit). If yes, we're scaffolding a Go service from zero including tests. Materially changes the estimate and the deadline risk. Same question applies to `k4a-sam-campaign-data-sync` and `sam-performance-reporter` (greenfield skeleton).

### Q3: Do we reuse `sam-report-scheduler` for delivery, or stand up a new worker?
- **For:** Backend / Architect
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** It's the only working scheduled-email path (generator→Cloud Tasks→runner→SendGrid, weekly cadence already present). But it's ASA-coupled, UTC-only, no idempotency, no entitlement, no alerting, no tests, hardcoded SendGrid key. Reuse saves the delivery skeleton but we inherit real debt; green-build is a clean slate but slower. Need a call to scope the estimate.

### Q4: What is the authoritative entitlement/subscription source we re-derive against (FR-014), and how do we represent "standard plan includes weekly email"?
- **For:** Backend / PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** No investigated delivery service has plan-tier/feature entitlement. KG Lookup A named `accounts-api` but I didn't get to investigate it. Need to confirm the source, the identity context, and fail-closed behavior (NFR-005) before FR-014 can be built.

### Q5: What timezone approach for "Tuesday 9am US Eastern observing DST" (BR-1/NFR-001)?
- **For:** Backend / Architect
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** `sam-report-scheduler` is UTC-only. Either add real TZ support (`moment-timezone` is present but unused) or encode a DST-tracking UTC hour. Affects scheduler config and whatever Cloud Scheduler cron lives in the external `ko-k8s-sam` repo (not in any checkout I had).

### Q6: NFR-015 — which existing scheduled-report consumers are at risk if we reuse an existing path?
- **For:** Backend / PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** If we add onto `sam-report-scheduler`, its existing SAM report recipients/cadence/output must stay unchanged. Need to confirm blast radius before touching shared code.

### Q7: What are the Cloud Tasks queue retry settings, and the Cloud Scheduler cron that fires the generator?
- **For:** Infra / Backend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Both live outside the repos I could read (GCP infra / `ko-k8s-sam`). FR-007/FR-012/NFR-010 "bounded backoff, max 3, honor 429 Retry-After" partly depend on queue config. Can't confirm it's compliant from source.

### Q8: Is Meridian's spend data guaranteed USD? (BR-2 assumes USD to 2dp)
- **For:** PM / Data
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Spend data in the ASA source carries currency + string amount. If non-USD ever appears, the 2-column "$X.XX" format and BR-8 correctness are at risk.

### Q9: DEC-008 — VP CC address + exact definition of "first send"?
- **For:** PM (Product owner)
- **Status:** 🔴 Open
- **Answer:** _Pending response_ (provisional: CC VP on first delivery with no prior delivered-outcome audit record; drop thereafter)
- **Impact:** FR-011a. Also note: "first send" is defined by absence of a prior delivered audit record — which depends on the audit trail (FR-017) existing, which doesn't yet. Slight circular dependency to design around. Can build on provisional value for now.

---

## Rough Scope

Assuming we build on `sam-report-scheduler` for delivery + resolve a real spend source (big assumptions):

- **Spend retrieval (FR-004/005):** new read path that aggregates spend into the 4 fixed marketing channels for a Mon–Sun window. Source TBD (Q1). Reuse `sam-campaign-data-sync`'s summing pattern as reference, read from Mongo `localSpend` source-of-truth not the float mirror.
- **Reporting window derivation (FR-009):** compute Mon–Sun ending the Sunday before Tuesday send; state it in the email.
- **Scheduler / Tuesday 9am ET (FR-001, BR-1):** weekly cadence exists (`repeatType===7`); add proper TZ handling for Eastern + DST.
- **Idempotency + overlap guard (FR-002/003, NFR-008/009):** net-new run-ledger keyed (customer + reporting window); dedup key on Cloud Task; "already ran this window" check.
- **Email composition (FR-008/009/009a/009b/010):** NEW simplified 2-column template (channel, $X.XX), multipart HTML + plain-text, Meridian logo (hosted image + alt), reporting-window line, zero-spend channels shown as $0.00. Reuse the `.toFixed(2)` formatting helper.
- **Delivery + retry + recipients (FR-011/011a/012):** `marketing-team@meridiantravel.example` → `report.emails`; VP CC first-send only; bounded backoff / 429 Retry-After (confirm queue config does the backoff or add app-level); fix the latent `send[0]` crash.
- **Entitlement (FR-014/NFR-004/005):** net-new, fail-closed, re-derived per run against authoritative source (Q4).
- **Block-on-incomplete-data (FR-006/007, NFR-011):** no partial/placeholder send; distinct failure conditions recorded (data-source-unavailable vs data-incomplete).
- **Audit trail (FR-017/NFR-012):** net-new append-only immutable record; async write; alert on write failure; 12-month retention.
- **Kill-switch (FR-016/NFR-013):** net-new config-driven disable, <5 min to stop new runs.
- **Monitoring/alerting (NFR-003):** net-new — >1% failure for 5 min, and per-customer missed delivery by end of Tuesday window. Reuse `sam-campaign-data-sync`'s Slack-alert pattern as reference.
- **Log hygiene (NFR-006):** scrub spend figures, recipient addresses, credentials/tokens from logs; mask identifiers to last 4–8.
- **Tests:** `sam-report-scheduler` has ZERO test scaffolding. Build a test strategy from nothing — especially important for a BR-8 spend-integrity feature.

If instead this lands in a greenfield repo (`k4a-sam-backend` / `sam-performance-reporter`): add full Go service scaffolding (`cmd/`, vendored modules, CI already present) + ALL of the above from scratch.

---

## Dependencies on Other Roles

- **PM / Architect:** resolve service ownership (Q1, Q2, Q3) — this is the critical-path blocker. Spec itself says the 5-week deadline only holds if this is resolved before build.
- **Data:** confirm the authoritative four-channel spend source and that figures match the customer's other spend views (BR-8); confirm USD (Q8).
- **PM (Product owner):** DEC-008 VP CC address + first-send definition (Q9).
- **Infra:** Cloud Tasks retry config + Cloud Scheduler cron (lives in `ko-k8s-sam`, outside any checkout I had) (Q7); deploy topology for whichever service we pick (Cloud Function vs VM vs k8s — the candidates differ wildly).
- **Backend (accounts-api owner):** entitlement source contract (Q4) — `accounts-api` was named in KG Lookup A but I didn't investigate it.
- **Frontend:** none — PM confirmed email-only, no UI (BR-7). `frontend-mos` was correctly dropped.

---

## Notes

- Investigation evidence here is Read-verified (real repo checkouts), not just KG prose — treat the "empty stub" findings as hard facts: `k4a-sam-backend` and `k4a-sam-campaign-data-sync` have exactly one two-line README and one commit each; `sam-performance-reporter` is CI/Dockerfile only.
- `reporting-subscriptions-worker` ruled out as owner (subscriptions/MRR domain, zero matches for spend/channel/marketing). Keep as a *reference template* for a Go BigQuery→file→notify worker and a per-row status state-machine if we ever build a new reporting worker.
- `notifications-api` ruled out entirely — in-product UI banners, unrelated.
- `cost-v2` → `netwrkcost-api` is a confirmed outbound seam; `netwrkcost-api` is the strongest unconfirmed candidate for report-gen/entitlement/idempotency but I had no checkout of it. Worth investigating next.
- The "FR flow is trivial, the ownership/data-source question is the whole risk" framing should be front-and-center in planning. We cannot responsibly estimate until Q1/Q2/Q3 land.

---

*Captured by `/kochava:eng-context backend` on 2026-11-22*