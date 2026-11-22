# Engineering Notes: Weekly Spend-by-Channel Email - Backend

**Engineer:** _(unassigned)_
**Date:** 2026-11-22
**Based on:** product-spec.md + live-repo investigation findings (10 candidate repos read)

---

## Initial Thoughts

OK, so the big win here is the investigation already did the hard part — it read all ten candidate repos and reconciled the service-name mess the spec flagged. Short version of what I'm walking away with:

- **Scheduler/dispatcher role = `reporting-dispatcher`.** Real, table-driven poll loop. Single-process systemd binary on `reporting02`. Weekly cadence already exists.
- **Schedule config store + query-request definition = `reporting-api`.** Stores schedule rows, computes `next_schedule_dttm`, doesn't fire anything itself.
- **Report builder + delivery = `summary-report-cloud`** (Pub/Sub consumer off `ko-p-restmeas-pst-summary-report-requests-1-3`). Builds the file, uploads, sends a notification email.
- **Spend data = BigQuery `cost_network_ad`** via `summary-report-cloud`, dimensioned by **ad network**, not by the four spec "channels."

The **good news**: weekly Monday cadence, previous-period resolution, account eligibility, SendGrid delivery to a recipient list, and a partial-data marker all already exist somewhere in these pipelines. OQ-007's "reuse the existing pipeline" is genuinely viable — I'm not building a scheduler from scratch.

The **bad news**: two gaps are real and non-trivial, and they're the heart of the feature:
1. **There is no "channel" dimension anywhere.** Not in `reporting-api`, not in `sam-api`, not in `summary-report-cloud`, not in the subscriptions worker. Spend is keyed by ad `network_id`. The spec's four fixed channels (Email / paid search / paid social / display, BR-6) are a **net-new mapping** with no existing pattern.
2. **Every delivery path emails a *link to an uploaded file*, not an in-body table.** BR-4 ("plain table in the email body, no attachment") is unsupported in all three delivery-capable repos. `summary-report-cloud` and the subscriptions worker both render a `report-notification-full-sftp` template whose payload is an `s3link` + `row_count`, and emit CSV/JSON only. No HTML/table body generator exists.

Complexity gut-check: **medium-to-high**, and almost all of it concentrated in those two gaps + the timezone correctness issue below. The orchestration wiring is mostly reuse.

Ruling out the dead ends up front so nobody wastes time: `k4a-mcp-server` (MCP tool gateway, wrong service), `kochava-k4a-mcp-server` (registry manifest, no code), `k4a-sam-backend` (empty README stub), `sam-performance-reporter` (greenfield scaffold, zero source), `sam-api` (SAM event/revenue analytics, no spend, no channel). The KG's medium/high confidence on several of these is wrong — the real repos are `reporting-dispatcher`, `reporting-api`, and `summary-report-cloud`.

---

## Concerns & Risks

- **Channel mapping is undefined and has no home.** Spend is by ad network. Who decides that network X = "paid social"? Is that mapping static, per-account, or does it live upstream of all these repos? This is the single biggest unknown and it's not just code — it's a data-modeling + product decision. Everything in Feature Area C (FR-006/007/008) depends on it.
- **BR-4 (table-in-body, no attachment) fights the existing architecture everywhere.** Current flow is *always* query → file → S3/SFTP upload → email-with-link. Producing an inline spreadsheet-ready table means either a new email template on notifications-service **plus** a new render path in `summary-report-cloud` that carries spend rows into the body, or an entirely new delivery type. This is the heaviest build item and it crosses a repo boundary (notifications-service-api, which I couldn't read).
- **Timezone correctness.** Spec wants Monday–Sunday **Eastern** anchoring (BR-3/FR-004). Reality:
  - `reporting-api` computes the week edge in **UTC** first, then CONVERT_TZ, and only accepts whole-hour offsets (ET passes, but the boundary math is UTC-first — needs tracing).
  - `reporting-dispatcher`'s named "PreviousWeek" path is **Sunday-anchored** (`time.Weekday()`), and the common scheduled path is pure day-offset math with *no* weekday anchoring at all.
  - `summary-report-cloud` just trusts the passed timestamps.
  So ET-anchored Mon–Sun is **not** free — it's a correction somewhere in dispatcher/reporting-api boundary logic, and it has golden-example tests (`helpers_test.go`) that will need updating.
- **BR-3 vs BR-8 tension.** BR-8 says send Monday 8 AM ET; BR-3 says send ~1 day after the covered Sun. The existing schedule model expresses either one but the investigation flagged they can't both be expressed simultaneously in the current `run_on`/`delay`/`previous_time` shape cleanly. Need PM/arch to confirm which anchor wins — I read it as "Monday 8 AM ET covering the prior Mon–Sun," which is internally consistent, but want it confirmed.
- **FR-002/FR-003 (no-overlap / idempotent-per-week) are not actually implemented.** Dispatcher's overlap guard is in-memory + relies on being single-replica (the hash-based concurrency guard is *commented out*). It always INSERTs a fresh `requested` row and advances the schedule — no per-(account, covered-week) overwrite. FR-003 "overwrite rather than duplicate" is net-new logic. And FR-002 correctness *depends on the dispatcher staying single-replica* — if anyone ever scales it out, this silently breaks.
- **NFR-012 kill-switch** — need to confirm a config-only disable exists that the dispatcher honors within 5 min without a deploy. Didn't see an explicit flag; `sch_status` toggling is the closest lever but that's per-schedule, not an incident kill-switch.
- **Hardcoded SendGrid keys** in `sam-report-scheduler` AND `reporting-subscriptions-worker` AND `summary-report-cloud` email paths. Pre-existing security debt. If we touch email we should move these to secrets, but that's scope creep I want to flag not silently absorb.
- **5-week hard deadline.** The two real gaps (channel mapping + in-body table) are exactly the parts with no existing pattern. If either balloons, the deadline is at risk. Strict MVP scope is doing a lot of load-bearing work here.

---

## Questions & Answers

### Q1: Where does the ad-network → channel mapping live, and who owns it?
- **For:** PM / Architect / Data-eng
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Blocks all of Feature Area C (FR-006/007/008) and the whole BR-6 four-channel output. Spend is only available by `network_id` in BigQuery `cost_network_ad` — there is no channel concept anywhere in code. Need to know: is the Email/paid-search/paid-social/display mapping static, business-defined config, or does it already exist in some upstream system I didn't find? Can't estimate the core feature without this.

### Q2: For BR-4 (table in email body, no attachment) — is a notifications-service template change in scope, and can that template accept tabular spend rows?
- **For:** PM / Architect (+ notifications-service owner)
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Every delivery path today emails an S3 *link*, not data. `notifications-service-api` is template-GUID-driven and I couldn't read it from these checkouts. If it can render an HTML table from passed vars, this is a new template + new render path in `summary-report-cloud`. If it can't, it's bigger. This is the largest build item — need the answer to scope it.

### Q3: BR-8 (Monday 8 AM ET) vs BR-3 (~1 day after covered Sunday) — which is the authoritative send anchor?
- **For:** PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** The existing schedule model can express one anchor cleanly, not both simultaneously. I'm reading it as "send Monday 8 AM ET, covering the preceding Mon–Sun week" (which satisfies both since Monday 8 AM IS ~1 day after Sunday). Want confirmation before I configure the schedule row.

### Q4: Is `reporting-dispatcher` guaranteed to stay single-replica, and is there a real config-only kill-switch?
- **For:** Infra / Backend lead
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** FR-002 (no overlapping runs) currently works *only* because the dispatcher is a single systemd process on `reporting02` — the in-memory dedup and commented-out concurrency guard mean a multi-replica move silently breaks correctness. Also need to confirm NFR-012's 5-minute config kill-switch actually exists as a lever the dispatcher honors (not just per-schedule `sch_status`).

### Q5: FR-003 idempotent-per-week overwrite — acceptable to build new upsert logic, or is "new row per run" tolerable for MVP?
- **For:** PM / Architect
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Dispatcher always INSERTs a fresh `requested` row + advances the schedule; there's no overwrite-by-(account, covered-week). True FR-003 is net-new logic (upsert keyed on account+week). Want to know if this is a must-have for MVP or a fast-follow, given the deadline.

### Q6: Does the whole-hour-offset UTC constraint in `reporting-api` land the week edge exactly on ET midnight after CONVERT_TZ?
- **For:** Backend (self/team — needs tracing into dispatcher date math)
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** FR-004/FR-005 contiguity + non-overlap correctness. `reporting-api` computes the Monday edge in UTC first; America/New_York passes the whole-hour validator but whether the resolved edge is true ET-midnight needs verification against how the dispatcher consumes `sch_next_schedule_dttm`. DST edges are the scary case.

---

## Rough Scope

What I think we'll need to touch (reuse-first, per OQ-007):

- **`reporting-dispatcher`** — configure the weekly Monday schedule row; **fix/add** Mon–Sun **Eastern**-anchored previous-week boundary logic (current paths are Sunday-anchored or offset-only). Update the `helpers_test.go` golden examples. Possibly add per-(account,week) idempotency (FR-003) depending on Q5.
- **`reporting-api`** — register the schedule definition (frequency=weekly, run_on=1); verify ET boundary math; the spend query is expressible as a `summary` report with `traffic=network_spend` grouped by `network`.
- **`summary-report-cloud`** — the big one. **New**: network→channel aggregation/mapping (BR-6), zero-fill for empty channels (FR-008), in-body table rendering (BR-4), partial-data caveat text (extend the existing `PARTIAL` marker for BR-3a). New delivery path or new email template that carries spend rows instead of an S3 link.
- **`notifications-service-api`** (external, couldn't read) — likely a new template for the in-body spreadsheet table. Need owner involvement.
- **Operational config** — recipient list (`marketing-team@meridiantravel.example`), schedule, Eastern TZ, kill-switch wiring.
- **Alerting/failure conditions** — distinct data-not-ready / delivery-failure / missed-send statuses (FR-012/014/015), single-missed-send alert (NFR-004). Some bookkeeping exists (`RecordScheduledFailure`), needs the named conditions.

What I explicitly do **not** need to touch: `k4a-mcp-server`, `kochava-k4a-mcp-server`, `k4a-sam-backend`, `sam-performance-reporter`, `sam-api`, `sam-report-scheduler`, `reporting-subscriptions-worker` (entitlements/subscription domain, wrong data — though its partial-data + Notify patterns are useful references).

---

## Dependencies on Other Roles

- **Data-eng / Architect:** the network→channel mapping definition (Q1). Hard blocker for the core feature.
- **PM:** send-anchor decision (Q3), FR-003 MVP-vs-fast-follow call (Q5), confirmation that BR-4 template work is in scope (Q2).
- **Notifications-service owner:** whether their template can render a tabular body, and ownership of a new template (Q2). Cross-repo, cross-team.
- **Infra:** dispatcher single-replica guarantee + kill-switch confirmation (Q4). Also any deploy of the modified `summary-report-cloud` worker.
- **No frontend/SDK dependency** — spec confirms no UI for Phase 1 (OQ-006, out-of-scope table).

---

## Notes

- Outbound seam to remember: `reporting-dispatcher` → Pub/Sub topic `ko-p-restmeas-pst-summary-report-requests-1-3` (prod project `ko-p-restmeas-98fj`) → `summary-report-cloud`. The `ReportStruct` already carries `Notify`, `DeliveryMethod`, `TrafficFilter`, `Columns`, `TimeStart/TimeEnd` — the fields a channel-spend email would ride on.
- `summary-report-cloud` exposes **no inbound HTTP** — only entry is publishing to its subscription. Keep that in mind for any orchestration assumptions.
- Existing `PARTIAL` filename marker in `summary-report-cloud` (`main.go:573`) is the hook to extend for BR-3a's "note identifying what's missing" — currently only affects naming, not email text.
- BR-5 standard-plan gating: no plan-tier code exists anywhere. Eligibility today = `status==activated` + reporting access. Meridian just needs to be activated; no plan-tier code to add (confirmed in both `reporting-api` and `reporting-dispatcher`).
- KG confidence scores were unreliable here — several medium/high candidates turned out to be stubs or wrong-domain. Trusting the live investigation over the KG.
- The `reporting-api` → `frontend-mos` KG hint was **explicitly ruled out** by grep — not a real connection.

---

*Captured by `/kochava:eng-context backend` on 2026-11-22*