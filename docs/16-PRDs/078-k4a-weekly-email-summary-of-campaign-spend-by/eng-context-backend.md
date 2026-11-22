# Engineering Notes: Meridian Weekly Spend Summary - Backend

**Engineer:** _(unassigned)_
**Date:** 2026-11-22
**Based on:** product-spec.md (+ real repo investigation across 10 candidate services)

---

## Initial Thoughts

OK so the good news first: the investigation basically already did our architecture spike for us, and the shape of this feature is way less scary than the candidate-service list makes it look. The spec keeps flagging "ownership of spend-data vs email-sender is unresolved" — after reading the actual repos, the real picture is a **two/three-tier report pipeline**:

- **reporting-api** = stores + validates the schedule/report *definition* (MySQL `kochava_reports.scheduled`/`requested`). Does NOT fire, query, or email.
- **reporting-dispatcher** = polls `kochava_reports.scheduled`, computes the window, bumps `sch_next_schedule_dttm`, inserts a `req_status='queued'` row, then **publishes to Pub/Sub** (`ko-p-restmeas-pst-summary-report-requests-1-3`). Does NOT query spend or email either.
- **The cloud summary worker** (consumes that Pub/Sub topic) = where spend query + channel aggregation + email composition + partial-data flagging + delivery/retry actually live. **This repo was NOT in the investigation checkout.** That's the real gap.

My gut: the scheduling half (FR-001, Feature Area A, BR-2 weekly/Monday) is **mostly configuration, not new code** — the dispatcher already supports `weekly` + `run_on=Monday` + timezone via `CONVERT_TZ`. The hard/unknown half is the summary worker, which we can't see.

Complexity feels **medium**, but with a big asterisk: it's entirely gated on confirming the summary-worker repo and whether its existing spend data even has a "channel" concept. Across every repo we DID read, nobody has email/paid-search/paid-social/display as a first-class dimension — closest things are `traffic` types (reporting-api), `network_name` (k4a-mcp), ASA device-class, MMP partners (sam-api). That channel-mapping question is the thing that could blow up the estimate.

---

## Concerns & Risks

- **The actual implementation repo (cloud summary worker) was never investigated.** Everything downstream of the Pub/Sub publish — spend query, aggregation by channel, email body, partial-data flag, delivery retry — is a black box. Can't size the bulk of the work until we find and read it.
- **"Channel" doesn't exist anywhere we looked.** reporting-api = `traffic` types; sam-api = partner/campaign/keyword, only `revenue`+`event_count`, no "spend"; k4a-mcp cost tools = `network_company_name`. BR-5's four channels (email/paid search/paid social/display) have no off-the-shelf representation. Someone has to define the channel→data mapping, and it may be net-new.
- **"spend" vs "revenue" confusion in sam-api.** sam-api's only money aggregate is conversion `revenue` (money in), NOT campaign spend (money out). If sam-api gets picked as the source, that's a dead end without new fields/tables.
- **Per-advertiser kill-switch doesn't exist in the dispatcher.** CA-7 in dispatcher: only global `maintenance_mode` + per-schedule `sch_status`. FR-019/BR-9 "feature enabled for Meridian?" gate is **new**. Good news: reporting-api already has a `feature_rollout` whitelist/blacklist-by-account_id pattern (CA-5) we can copy.
- **Delivery-failure retry/backoff + ops alert (FR-016/017, flow L→M→N) is NOT in the scheduling/dispatcher tier.** Dispatcher only counts *report-run* failures (max 7 → `sch_status='error'`). And the one email-sending analog we read (sam-report-scheduler) **swallows SendGrid failures silently** (CA-6) — exactly the anti-pattern FR-017 forbids. Whatever sends our email needs real bounded retry + alerting built.
- **Partial-data flagging (BR-8) is new everywhere.** sam-report-scheduler zero-fills missing data silently (indistinguishable from real $0). reporting-subscriptions-worker's only "partial" concept is row-limit truncation. Nothing flags "channel missing/delayed." Net-new logic.
- **DST correctness for "8:00 AM Eastern."** Dispatcher does weekly/timezone via MySQL `CONVERT_TZ` and the code itself comments on DST pain. sam-report-scheduler is worse — pure `getUTCHours()`, no TZ, would drift an hour across DST. Need to verify whichever path we use actually holds 8 AM ET year-round.
- **Several "candidate" repos are empty stubs or red herrings** — `k4a-sam-backend` (2-line README), `sam-performance-reporter` (bare Go scaffold), `kochava-k4a-mcp-server` (metadata-only). `notifications-api` is a name-collision (in-app banners, not email). Don't burn time wiring to these.
- **Hardcoded SendGrid keys in source** in both email-capable repos we read (sam-report-scheduler, reporting-subscriptions-worker). If we reuse either pattern, don't inherit that.
- **Where's the "email-only, inline per-channel body" model?** Every existing report emailer sends a *link/attachment notification* (S3/SFTP + "your report is ready"). The spec wants the per-channel totals **in the email body** (BR-3, FR-011/012). That's a different content model — new template + param shape.
- **Idempotency / exactly-once (FR-002/003/004):** dispatcher is single-replica (CA-9) so the periodic loop can't double-fire across pods — exactly-once rests on `sch_next_schedule_dttm` being committed before re-selection, NOT a distributed lock. Fine as-is, but worth a note that we must NOT add any in-process weekly timer to the multi-replica tiers (reporting-api, k4a-mcp-server would multi-fire).

---

## Questions & Answers

### Q1: Where is the cloud summary worker that consumes `ko-p-restmeas-pst-summary-report-requests-1-3`? That's where the spend query, channel aggregation, email composition, partial-data flag, and delivery retry actually live — and it was never in the investigation checkout.
- **For:** Backend (architect) / PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Blocks sizing the majority of the feature. We can scope the scheduling/config side now, but FR-005–011, FR-016/017, BR-8 all land in this unseen repo.

### Q2: Which system owns authoritative **spend-by-channel** data, and does a "channel" (email/paid search/paid social/display) dimension exist there — or does it need to be derived/built? Nothing we read (reporting-api `traffic`, sam-api revenue-only, k4a-mcp `network_name`) has it off-the-shelf.
- **For:** Backend (architect) / PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Core data retrieval (FR-005/006/008) is unestimatable. If channel is net-new mapping, this is a meaningful chunk of work and possibly a schema change.
- **Note:** The spend data is almost certainly in **BigQuery** (reporting-api CA-6 points downstream workers at `account_<AccountId>` datasets via the Shaman/analytics_query_creator translator). Need the worker repo to confirm the actual channel grain.

### Q3: Confirm spend = campaign **spend** (money out), not conversion **revenue**. sam-api only exposes `revenue`/`event_count`, no spend field. If sam-api is on the table as a source, this matters.
- **For:** PM / Backend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Picking the wrong source gives semantically wrong numbers to the VP. Must be nailed before build.

### Q4: For the per-advertiser kill-switch (FR-019/BR-9) — OK to reuse reporting-api's existing `feature_rollout` whitelist/blacklist-by-account_id pattern, enforced at dispatcher-poll time? Dispatcher has no per-advertiser gate today, only global `maintenance_mode`.
- **For:** Backend (architect)
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Shapes the kill-switch design. Reuse keeps it small; a new mechanism is more work. Also FR-019's "propagate within 5 min" needs the dispatcher poll interval (~10s loop) to actually read the flag each cycle.

### Q5: Does `CONVERT_TZ`'s named-zone handling in the dispatcher reliably give us 8:00 AM **ET across DST** transitions? Code comments explicitly call out DST risk.
- **For:** Backend (architect) / Infra (TZ tables)
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** BR-2 correctness. If the dispatcher path is solid, scheduling is ~config only. If not, we need TZ handling work.

### Q6: Is the once-per-week cron that fires `rsGenerator`/the dispatcher owned here or in a separate IaC/Cloud Scheduler repo? sam-report-scheduler's actual Monday-8am trigger (Cloud Scheduler → Pub/Sub) was not in its repo, and reporting-dispatcher is a single-host systemd poller.
- **For:** Infra / Backend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Need to know which scheduling tier we're actually extending before committing to reporting-dispatcher vs sam-report-scheduler.

---

## Rough Scope

Assuming we go the **reporting-api + reporting-dispatcher + (existing) summary worker** route:

- **reporting-api:** create the weekly summary report *definition* row via `PostSchedSummary`→`PostSchedGeneric`→`insertScheduledReportQuery`. `frequency='weekly'`, `run_on=1` (Mon), time-of-day offset, Eastern `time_zone`, `notify=[marketing-team@meridiantravel.example]`. Mostly data/config.
- **reporting-api:** add a `feature_rollout` row + enable check for the Meridian weekly-summary kill-switch (copy `checkMetaAMMFeatureAccess` pattern, keyed on account_id). (FR-019/BR-9)
- **reporting-dispatcher:** likely config-only to pick up the new weekly row — generic path already dispatches `summary` type over Pub/Sub. Verify weekly window = previous Mon–Sun (CA-3 `getFromToForTimeRange`/`PreviousWeek`). Add per-advertiser enable gate at poll time (NEW).
- **summary worker (UNSEEN — biggest chunk):**
  - spend-by-channel BigQuery query, advertiser-scoped server-side (NFR-004, fail-closed)
  - aggregate → one total per channel (FR-006), all 4 channels (FR-008)
  - **inline per-channel email body** (new template/content model, not link-notification)
  - partial-data detection + "incomplete" flag per channel (BR-8 — NEW)
  - bounded exponential backoff 3 attempts + honor Retry-After + ops alert + distinct failure reasons (FR-016/017 — NEW, since existing emailers swallow failures)
  - audit record per run (NFR-005, append-only, 12mo retention, masked recipient addrs per NFR-006)
- **Idempotency check:** confirm exactly-once via `sch_next_schedule_dttm` advance (dispatcher single-replica) — likely no new work, just verification.

Explicitly NOT touching: k4a-mcp-server (wrong tier, multi-replica), sam-api (no spend/channel), notifications-api (in-app banners), the empty stubs.

---

## Dependencies on Other Roles

- **Architect / PM:** resolve Q1 (find the summary worker repo) and Q2/Q3 (spend-by-channel data owner + channel definition) — **hard blockers** for sizing most of the feature.
- **Infra:** Q5 (DST-correct `CONVERT_TZ`) and Q6 (who owns the Cloud Scheduler/cron that fires the dispatcher). Also deploy topology for any new worker code.
- **Data:** if channel is net-new, need the BigQuery channel→spend mapping defined (which datasets/fields roll up to email/paid search/paid social/display).
- **Nobody on frontend** — spec confirms no UI (BR-9/NFR-007), and investigation ruled out `frontend-mos` (reporting-api has zero outbound calls to it). Keep it off the list.

---

## Notes

- Pub/Sub seam confirmed from dispatcher side: topic `ko-p-restmeas-pst-summary-report-requests-1-3`, payload = marshalled `ReportStruct` JSON (carries `Notify`, `DeliveryMethod`, `TimeZone`, `TimeStart`/`TimeEnd`). **The summary worker that consumes this is the #1 next investigation target** — it answers the two questions the spec flagged as unresolved.
- Topology reality check: reporting-dispatcher = single-host systemd poller (`reporting-02.aerospike.internal`), reporting-api = multi-replica Marathon, k4a-mcp-server = multi-replica stateless uvicorn. The once-per-week guarantee must NOT come from any in-process timer on a multi-replica tier.
- Good reusable patterns to copy: reporting-api `feature_rollout` (kill-switch), reporting-subscriptions-worker (closest executor shape: BQ→compose→SendGrid, recipients from request `Notify`) — but it emails a *link*, zero-fills silently, and has no delivery retry, so treat it as a skeleton, not a drop-in.
- Pre-existing hygiene flag for whoever owns the worker: hardcoded SendGrid keys appeared in both email-capable repos — move to Secret Manager if we reuse.

---

*Captured by `/kochava:eng-context backend` on 2026-11-22*