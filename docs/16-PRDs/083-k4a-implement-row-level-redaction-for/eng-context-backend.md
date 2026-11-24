# Engineering Notes: eventGrainEligibility Row-Level Redaction - Backend

**Engineer:** _(unassigned)_
**Date:** 2026-11-24
**Based on:** product-spec.md (Type A confirmed) + real repo investigation across 7 candidate repos

---

## Initial Thoughts

First reaction: the spec is honest about the big landmine — **we don't actually know where the `eventGrainEligibility` logic lives yet**, and after reading 7 repos myself, I can confirm: *it's not in any of them*. Every single candidate I investigated came back negative on `eventGrainEligibility` / `amazon` / `redact` / `eligib` grep. So the KG "primary vs alternate owner" discrepancy isn't really resolved by picking between two lists — it's that **neither list points at the real owner**. The repos we have are consumers, dependencies, and pattern references, not the home of the core logic.

Complexity feel: **HIGH**, and the spec rates it that way too. Not because any one piece is hard, but because:
- The actual implementation site is an un-checked-out repo (`query.api.kochava.com` is the recurring suspect for the reporting/query chokepoint).
- This is row-level enforcement, and *every* existing pattern I found is account/app-grain, write-side, or entity-grain — never row-grain read-side.
- We have a live, in-the-wild example of the exact "second door" bug the spec warns about (see entitlements-api and audit-api below). So the risk is real, not theoretical.

The good news: the **enforcement shape the spec wants (BR-004/BR-005: authoritative, per-request, fail-closed) is idiomatic across Kochava.** I found clean precedents in at least 4 repos. We know *how* to build this kind of guard; we just have to find where it belongs and generalize it to row-grain.

---

## Concerns & Risks

- **We can't name the owning repo from what we have.** `query.api.kochava.com` is the strongest lead (it's what `k4a-mcp-server` and `entitlements-api` both call for the reporting/query path), but it wasn't in the investigation set. Until someone reads that repo we can't scope the actual delta. This is the #1 blocker.
- **"Second door" is a confirmed real pattern, not hypothetical.** In `entitlements-api` the Spanner query path does per-row filtering (`scanRowIntoValidEntitlement`) but the BigQuery *reporting/summary* path only does coarse app-membership checks on a pre-aggregated SUM — it literally cannot redact a row post-aggregation. In `audit-api`, the transaction read path fetches rows unscoped and redacts in a Go loop while the list path scopes in SQL. **Two paths, two mechanisms, same table — exactly BR-002's nightmare.** Strongly argues for NFR-019's single chokepoint, but a BigQuery aggregate path may not *have* a natural row-grain chokepoint. That tension needs design attention.
- **Row-grain over BigQuery aggregates is genuinely hard.** Several repos enforce tenancy by routing to per-account datasets (`account_<id>.*`) rather than row predicates. A per-row eligibility signal doesn't map cleanly onto per-account-table isolation. If the reporting path is aggregate SUMs, "redact one row" may require re-architecting what gets aggregated.
- **No existing distinct dependency-unavailable error anywhere.** FR-011/FR-012/NFR-014 want named, distinct failure conditions. Every precedent I found collapses "authoritative source unreachable" into a generic 500/ErrInternalError. This is net-new error-type work everywhere it lands.
- **No feature-flag/kill-switch framework in the obvious candidates.** `mos-iam-server` has an *org-level* `FeatureFlags map[string]bool` but it's returned as data for callers to interpret — there's no in-path `IsFeatureEnabled` gate, and it's per-org not global. FR-013/NFR-015 (5-min propagation, no deploy) is net-new wherever the chokepoint ends up.
- **Audit-on-read doesn't exist today.** FR-015/NFR-009 want an append-only audit record per redaction *decision* (a read event). `audit-api` exists and is the obvious writer, but in `entitlements-api` audit middleware is wired only to write handlers — reads emit nothing. So we're adding audit to the read path, which is a new access pattern.
- **The "what do customers see today" baseline is still Unknown.** OQ-008 / the baseline info-requirement. FR-014 "prior behavior" and the whole redaction delta can't be fully specced until someone traces this end-to-end in the real owning repo. Discovery task, blocks scoping.
- **Row-grain = N Keto checks if we route through mos-iam-server.** Its auth API is single-subject/single-object (`CheckPermission`). No batch "filter this list for this subject" primitive. Redacting N rows = N checks or a new bulk primitive. Latency concern vs NFR-001 (≤50ms p95 added).

---

## Questions & Answers

### Q1: Which repo actually owns the `eventGrainEligibility` read/query path? None of the 7 investigated repos contain it — `query.api.kochava.com` is the strongest lead but wasn't checked out.
- **For:** Backend (ownership) / Architect
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Blocks all scoping and estimation. We cannot size the redaction delta, identify the chokepoint, or confirm the two read paths until the owning repo is identified and read. Everything downstream depends on this.

### Q2: What does a customer see today when accessing eventGrainEligibility in K4A (the baseline)? Spec flags this as Unknown (OQ-008) and it blocks defining the redaction delta.
- **For:** Backend (discovery) / PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Blocks finalizing FR-014 "prior behavior" semantics and the overall redaction delta. Needs end-to-end tracing in the real owning repo before rules are finalized.

### Q3: Is the reporting/query path an aggregate (SUM-over-rows) path or a row-returning path? If it aggregates, a single row-level chokepoint (NFR-019) may be structurally impossible — we'd have to change what gets aggregated.
- **For:** Backend / Data
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Determines whether NFR-019's "single chokepoint" is achievable or whether the reporting path needs separate redaction design. Directly affects the "second door" risk mitigation and whether result counts/pagination change (NFR-018).

### Q4: Does the "authoritative eligibility source" for a *row* already exist as an ingested field, or must it be newly defined upstream? None of the investigated ingestion repos carry an eligibility/sensitivity field — the write model is flat JSON rows with no per-field provenance.
- **For:** Backend / Data (ingestion owner)
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** If net-new, adds an upstream ingestion/field-definition workstream before the mechanism can be fully exercised. DEC-001 says redact when requesting account ≠ advertiser owner OR a per-row sensitivity flag is set — the sensitivity flag may not exist yet.

### Q5: For row-level eligibility, do we route per-row checks through mos-iam-server/Keto (single-object API, would be N calls), or is eligibility derivable from a field already on the row (advertiser ownership comparison)?
- **For:** Backend / Architect
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Determines the per-request call pattern and whether NFR-001 (≤50ms p95 added) is achievable. N-checks-per-request against Keto is a latency risk; a row-field comparison is cheap. DEC-001's "account ≠ advertiser owner" leans toward the cheap path but needs confirmation.

---

## Rough Scope

What I think we'll need to do (assuming the real owning repo gets identified):

- **Identify + read the owning repo** (`query.api.kochava.com` suspected). Prerequisite to everything.
- **Trace the baseline** — what rows come back unredacted today across both read paths + UI.
- **Build the row-level eligibility evaluator** — per-row: requesting-account == advertiser-owner check AND per-row sensitivity-flag check (DEC-001). Model this on the fail-closed authoritative-guard pattern I found in `ad-insight-api` (`AccountChecker`→sprinkler-api, CA-1/2/3) and `mos-iam-server` (`/checkpermission`→Keto), but generalize account-grain → row-grain.
- **Place a single enforcement chokepoint** (NFR-019) shared by query + reporting paths. Reference the good pattern in `audit-api`'s `BuildQuery`/`[]Filter` (CA-1) — add eligibility as one more SQL `Filter` so it applies everywhere. Avoid the `audit-api` transaction-path mistake of re-implementing in a Go loop (CA-3).
- **Add distinct named failure conditions** (FR-012/NFR-014): timeout / dependency-unavailable / rate-limited each get their own error status. Net-new — no precedent.
- **Fail-closed posture** (FR-011/NFR-012): deny rows we can't confirm, return distinct dependency-unavailable error. Model the *shape* on existing guards but NOT the failure policy — note `ad-insight-ingestion` uses fail-to-prior-state, which is NOT what we want.
- **Kill-switch / feature flag** (FR-013/FR-014/NFR-015/NFR-016): config-based, no-deploy, 5-min propagation, staged rollout 1%→10%→50%→100%. Net-new; mos-iam-server's org-flag store is a possible carrier but needs an in-path gate added.
- **Audit-on-read** (FR-015/016/017): append-only, immutable, async-write, actor+resource+outcome+timestamp, no raw sensitive data, op-alert on write failure. `audit-api` + `pkg/auditclient` is the obvious writer to POST into. New access pattern (audit on a read decision).
- **Rate cap on shared query resource** (NFR-002) if evaluation adds load to BigQuery/shared query infra.
- **Error-rate alerting** (NFR-003): >1% for 5min on eval + on the eligibility dependency.
- **Consistency verification** across query/reporting/UI for same row+requester (FR-007) — the hard part given the aggregate-path concern.

---

## Dependencies on Other Roles

- **Architect / Backend ownership:** must resolve which repo owns the logic (Q1) before anything else moves. Hard blocker.
- **Data / Ingestion owner:** need to know if the per-row eligibility signal is an existing ingested field or net-new (Q4). The "field-definition layer" the spec references is NOT any ingestion repo I checked.
- **Frontend (K4A UI):** owns the redacted-row presentation (DEC-002: labeled "redacted" + short explanation). Backend provides the redaction decision + presentation metadata; UI renders it. They need our output shape locked (FR-008/FR-009).
- **Infra / Platform:** kill-switch config propagation (5-min), staged rollout mechanism, k8s deploy topology. Note: deploy topology for *every* repo I read lives in external `ko-k8s-apps`/`ko-k8s-minikube` — not answerable from any app checkout. Infra needs to weigh in on replica counts, especially for any ticker-based/scheduler components.
- **Support/Ops:** DEC-003 says no self-service access request in Phase 1 — access changes go through existing manual support/escalation. They should know redacted rows will generate inbound requests.
- **Security/Compliance:** NFR-011/DEC-004 — 12-month audit retention minimum, extend if a third-party data-sharing agreement applies (DEC-005 open-ish). Need confirmation whether Amazon Ads data is under such an agreement.

---

## Notes

- **Rule OUT as logic owner (verified by grep, confident negatives):** `ad-insight-api` (Meta-only, networkcompanies/53), `ad-insight-ingestion` (Facebook-only write pipeline), `k4a-mcp-server` (thin proxy to query.api), `kochava-k4a-mcp-server` (docs/registry manifest, 3 files, no code), `entitlements-api` (billing/subscriptions — KG name-collision on "entitlements"), `mos-iam-server` (IAM authority, account-grain not row), `audit-api` (audit log service).
- **Best pattern references to cite for eng-plan:**
  - Authoritative per-request fail-closed guard shape → `ad-insight-api` CA-1/2/3/4 (`AccountChecker`, fail-closed on error, boot-time hard dependency). Caveat: account-grain, write-side.
  - Single SQL-level chokepoint via filter list → `audit-api` CA-1 (`BuildQuery` + `[]Filter`). The "do this" example.
  - The "don't do this" second-door example → `audit-api` CA-3 (transaction path Go-loop) and `entitlements-api` CA-3 (BQ aggregate path with only coarse app check).
  - Per-request authority re-derivation → `mos-iam-server` `/checkpermission`→Keto (CA-1/2/3), `entitlements-api` `GetLegitObjects`→Sprinkler (CA-4), `k4a-mcp-server` `mmp_entitlement` (CA-3). All fail-closed-ish, none row-grain.
  - Kill-switch carrier candidate → `mos-iam-server` org-level `FeatureFlags map[string]bool` (CA-6), but per-org + no in-path gate.
  - Audit writer → `audit-api` + `pkg/auditclient` (POST-in library other services already use).
- **Watch-out:** `entitlements-api` builds queries by `fmt.Sprintf`/`ReplaceAll` string concat, not parameterized — SQL-injection-adjacent. Do NOT copy that style; use `audit-api`'s parameterized `@account_id` binding instead.
- **Watch-out:** multiple repos use `SAFE_CAST` so a malformed value excludes just that row (`audit-api` CA-5) — a decent "exclude-on-ambiguity" fail-closed precedent for a per-row predicate.
- **"event" trap:** in `k4a-mcp-server`, `event_tools.py` = Kochava conversion events, unrelated to Amazon event-grain. Don't let anyone grep "event" and chase the wrong subsystem.
- All 5 DECs (DEC-001..005) and 8 OQs are marked engineering-decides/resolved in the register — so the product side considers these settled enough to proceed on the *mechanism*; but DEC-001's "per-row sensitivity flag" existence (Q4) and the owning-repo question (Q1) are the things that still gate real work.

---

*Captured by `/kochava:eng-context backend` on 2026-11-24*