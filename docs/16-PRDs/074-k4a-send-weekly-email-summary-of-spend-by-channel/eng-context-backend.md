# Engineering Notes: Weekly Spend-by-Channel Email Summary - Backend

**Engineer:** _(unassigned)_
**Date:** 2026-11-21
**Based on:** product-spec.md (+ live repo investigation across all candidate services)

---

## Initial Thoughts

OK, I went and actually read the candidate repos instead of trusting the KG list. Big picture: the KG cross-check was right to be nervous — the curated affected-services list is **partially wrong**, and the two "strongest reuse candidates" the spec leans on (k4a-sam-backend, sam-performance-reporter) are **empty scaffolds**. So the de-risk story that was supposed to shrink the 5-week estimate is shakier than the spec assumes.

Good news first — the pipeline mostly exists and I can map it:
- **reporting-api** = write/define/validate side of scheduled reports. Weekly-Monday cadence is already first-class (`frequency="weekly"`, `run_on=1`). "Total spend per channel" already exists as a summary report: `traffic=[network_spend]` grouped by `network`. There's even an existing endpoint — `POST /v1.4/schedule/summary`. This is real reuse.
- **reporting-dispatcher** = the scheduler/execution side. Polls `kochava_reports.scheduled`, materializes due runs, and for `summary` reports **publishes to a Pub/Sub topic** (`ko-p-restmeas-pst-summary-report-requests-1-3`). It does NOT render or send email.
- The **actual spend-by-channel aggregation + email render + delivery lives in the downstream consumer of that summary Pub/Sub topic** — and I could not find/verify that consumer in this investigation. That's the real gap.

So the shape is: reporting-api (define schedule) → reporting-dispatcher (fire weekly, publish summary job) → **??? summary consumer** (aggregate, render one-line-per-channel, email). The whole feature's hardest requirements (BR-7 zero-fill, BR-8 retry-once-then-alert, BR-4 fixed recipient) land on that unidentified consumer.

Complexity: **medium-to-high**, but mostly because of *integration-path uncertainty and the BR-8 failure model mismatch*, not because the data/query is hard. The query is basically solved.

---

## Concerns & Risks

- **The two reuse candidates are empty.** `k4a-sam-backend` = one-line README stub, no code. `sam-performance-reporter` = Go CI scaffold, no `cmd/`, no `go.mod`, no source. The spec's NFR-011 de-risk explicitly hangs on "reuse k4a-sam-backend or sam-performance-reporter spend-summary logic if confirmed close enough." **Confirmed against code: there is nothing to reuse in either.** Either the clones were incomplete, or these are reserved-but-unbuilt. This directly threatens the 4–5 week estimate.
- **The summary Pub/Sub *consumer* was never identified.** reporting-dispatcher proves the publish side exists; nobody in this pass is the confirmed consumer that aggregates + renders + emails. Until we find it, we don't know whether BR-2/BR-6/BR-7/BR-8 are "extend existing" or "build new."
- **BR-8 retry/alert model fights the existing failure semantics.** The real failure model everywhere is **count-to-7** (`reportFailureMax=7` in dispatcher, "fresh 7 failures" comment in reporting-api). The spec wants **retry *once*, then alert**. And there's **no ops-owner alert-recipient field anywhere** — reporting-api's schema only has the `Notify` recipient list, no separate marketing-ops owner address. BR-8 is net-new logic + a schema/config addition, not a config tweak.
- **BR-1 exact "8:00 AM Eastern" is not cleanly expressible.** Weekly `run_on` only encodes the *day*, not hour-of-day. Time-of-day for weekly schedules is derived at create-time. AND reporting-dispatcher deliberately **delays summary reports by ~1hr** (`glDelayForSceduled=3600`, mislabeled "3 HRS") "so rollup can happen" — so a Monday-08:00 summary won't actually fire at 08:00. Need to confirm whether that violates NFR-002's 60-min window.
- **Eastern-time reporting week (BR-3).** Dispatcher converts via `CONVERT_TZ` only if the stored schedule carries the ET timezone. `network_spend_restated` *forces UTC + full-day boundaries*. Need to make sure we use the right traffic type and carry ET correctly, incl. DST.
- **"Channel" ≠ "network" exactly.** Everything backend-side models spend by `network` (reporting-api) or `company_network_name` (cost-v2). The spec's four channels (email / paid search / paid social / display) have **no direct column** — cost-v2's `channel` field is sparse/often-NULL. Mapping the 4 named channels → networks is undefined product/data work.
- **BR-7 zero-fill is in the (unknown) render layer.** Row-presence for zero-spend channels is a property of whatever renders the summary, which we haven't located. sam-report-scheduler *does* zero-fill, but it's Apple-Search-Ads-only and a different data model.
- **Two repos have committed secrets** (reporting-dispatcher GCP SA private keys + hardcoded Influx creds; multiple repos hardcode SendGrid keys). Don't propagate; flag if we touch those files.
- **NFR-012 kill-switch** — no config-based kill-switch exists in the dispatcher today. Net-new.

---

## Questions & Answers

### Q1: Where is the summary Pub/Sub consumer that aggregates spend-by-channel, renders the email, and sends it?
- **For:** Backend (whoever owns the summary pipeline) / PM to help point us at the repo
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Blocks almost all of scope B/C/D. reporting-dispatcher publishes to `ko-p-restmeas-pst-summary-report-requests-1-3`; the consumer of that topic is the real home of BR-2/BR-6/BR-7/BR-8 and is unidentified. Can't estimate render/delivery/zero-fill/alert until we find it.

### Q2: The spec's two named reuse candidates (k4a-sam-backend, sam-performance-reporter) are empty in the checkouts I got. Are these actually empty, or was the clone incomplete?
- **For:** PM / Infra
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** NFR-011's entire de-risk rationale assumes reusable spend-summary logic in one of these. If they're genuinely stubs, "reuse not build" is off the table and the 4–5 week estimate needs revisiting.

### Q3: How do we reconcile BR-8 "retry once then alert ops owner" with the existing count-to-7 failure model, and where does the marketing-ops-owner alert address live?
- **For:** PM / Backend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** No ops-owner recipient field exists in any schema I found; retry semantics everywhere are 7-strikes, not 1. This is net-new behavior + a new config/schema field. Need to confirm whether BR-8 overrides the shared 7-failure model only for this report or needs isolated logic in the summary consumer.

### Q4: Does the dispatcher's ~1hr summary rollup delay (and weekly run_on not encoding hour-of-day) satisfy BR-1's "8:00 AM Eastern" + NFR-002's 60-min window?
- **For:** PM / Backend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** If the marketing team expects the email *at* 8:00 AM ET and the rollup delay pushes it to ~9:00 (or wherever hour-of-day resolves), that's a visible SLA/expectation issue. Need the acceptable delivery window confirmed against NFR-002.

### Q5: How do the four marketing channels (email / paid search / paid social / display) map onto the backend `network` / `company_network_name` dimension?
- **For:** PM / Data
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Spend is grouped by network, not by these four channels. cost-v2's `channel` column is sparse/NULL. BR-6/FR-007/FR-010 can't be built until this mapping is defined. May be a Data-team concern.

### Q6: Which traffic type for the reporting-week query — `network_spend` or `network_spend_restated`? The restated variant forces UTC + full-day boundaries, which conflicts with BR-3 Eastern-time week.
- **For:** Backend / Data
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Affects correctness of the ET Mon–Sun window (BR-3) and whether we need custom TZ handling vs. an existing traffic type.

---

## Rough Scope

What I think we'll need to touch (assuming we reuse the existing reporting pipeline):

- **reporting-api** — likely just *configuration*: create the weekly summary schedule row (`POST /v1.4/schedule/summary`, `frequency=weekly, run_on=1, traffic=[network_spend], traffic_grouping=[network], notify=[marketing-team@…]`). Possibly a new field to carry the ET timezone and the BR-8 ops-owner address (no home for it today).
- **reporting-dispatcher** — confirm weekly summary dispatch works for a standard-plan activated account (no plan gate exists, so should be fine). Possibly add the **NFR-012 kill-switch** here (config-based, flips within 5 min). Reconcile the rollup-delay vs. BR-1 timing.
- **The summary Pub/Sub consumer (UNKNOWN — must locate)** — this is where the real build is:
  - BR-7 zero-fill: ensure all 4 channels appear with $0 if no spend.
  - BR-6/FR-010: one line per channel rendering (totals-only, no extra columns, no branding).
  - BR-11: show the reporting week.
  - BR-8/FR-009/FR-013/FR-014: retry-once + distinct failure reasons (data-retrieval vs delivery) + ops-owner alert email.
  - BR-4/NFR-009: fixed recipient list + recipient-integrity scoping.
  - NFR-008: account-scoped query, re-derived server-side.
  - FR-005/NFR-007: idempotency on account+reporting-week key.
- **Channel→network mapping config** — wherever that lands (may be data config, not code).

What I think we DON'T need:
- A new scheduler (reuse dispatcher — aligns with requester guidance).
- A new report-query engine (network_spend summary already exists).
- sam-api, k4a-mcp-server, notifications-api, cost-v2, reporting-subscriptions-worker, sam-report-scheduler — **ruled out** as owners (see Notes).

---

## Dependencies on Other Roles

- **PM** — resolve OQ-001 for real (which service consumes the summary topic / owns render+send), confirm BR-8 reconciliation, confirm acceptable delivery window vs. 8 AM ET, confirm whether the empty reuse repos are real stubs.
- **Data** — channel→network mapping for the four marketing channels; which spend metric/traffic type gives the correct ET-bounded weekly totals.
- **Infra** — confirm whether k4a-sam-backend / sam-performance-reporter checkouts were incomplete or genuinely empty; deploy topology for any new/changed summary-consumer service; where the kill-switch config lives.
- **Whoever owns the summary Pub/Sub consumer** — biggest dependency; most of the net-new build lands on them, and we don't even know who they are yet.

---

## Notes

**Services ruled OUT (verified against code, not inferred):**
- `sam-api` — SAM read API over Spanner for Apple-Search-Ads *postback/conversion* data. No spend, no channel, no scheduler, no email. "sam" name overlap only.
- `k4a-mcp-server` — stateless FastMCP tool proxy for AI agents. No scheduler, no mail transport. Only delegates email to backends via params. Read-side spend-by-channel data (`network_name` split + `Total_Cost`) is reachable but as interactive per-caller tools, not a job.
- `notifications-api` — misleadingly named; it's an *in-app UI banner* selection API. No email/SMTP, no scheduler. Zero link to any reporting service.
- `reporting-subscriptions-worker` — per-report worker for subscription/entitlement *revenue* (MRR), not ad spend. No channel concept. Good *pattern* reference (SendGrid email, queue/status model) but not a reuse target for spend.
- `cost-v2` — upstream *ingestion* pipeline; the origin of `total_spend`/`calc_total_spend_USD`/`channel` in BigQuery. Schedules *daily data pulls*, not weekly reports. Not a reporting/email service. Its `notifications-service-api` failure-email integration is a useful pattern for BR-8 though.
- `sam-report-scheduler` — Node GCF, strongest *structural* match (native weekly cadence, zero-fill, one-line-per-entity render, SendGrid email) but **Apple-Search-Ads-only** data, **all UTC** (moment-timezone declared but never imported — no ET support), no test files, hardcoded SendGrid key. Strong mechanism / wrong data model.

**Key reuse facts:**
- reporting-api `validTrafficSum` includes `network_spend` / `network_spend_restated`; requesting it forces grouping by `network` (api.go:70, :735-803).
- reporting-dispatcher weekly cadence (root.go:2045-2074), summary→Pub/Sub publish (pushSummaryToPubSub, root.go:1520-1623), 7-failure model (RecordScheduledFailure, root.go:2488-2506), summary rollup delay (root.go:84).
- No plan-tier gate in dispatcher — standard-plan K4A account (BR-5) flows through fine (dispatch gates on `status='activated'` only).

**Open integration-path summary:** the pipeline up to "publish summary job" is solved and reusable. Everything after the Pub/Sub publish — the consumer that does aggregate/render/deliver/retry/alert — is the unknown that OQ-001 needs to close, and it's where the real work and the BR-8 mismatch live. Do NOT commit to the 4–5 week estimate until Q1/Q2 are answered.

---

*Captured by `/kochava:eng-context backend` on 2026-11-21*