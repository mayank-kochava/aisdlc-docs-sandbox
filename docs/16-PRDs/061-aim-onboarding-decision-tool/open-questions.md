---
id: open-questions
title: "Open Questions — to revisit"
---

## Open Questions — to revisit

**Date:** 2026-06-12
Pending external answers. Each row notes the owner, what it gates, and where to fold the answer when it lands. Nothing here blocks starting execution except the two flagged "BLOCKS."

### Awaiting Jaya (Data Engineering) — message sent 2026-06-12

| # | Question | Gates | Fold answer into |
|---|----------|-------|------------------|
| J1 | **Collection model:** is event/cohort data always client-provided CSV, or does AppsFlyer Direct API mean we pull it and the client uploads nothing? | **BLOCKS O6** (download state vs "we collect it" state) | design §8c §2, O6, F6 |
| J2 | Confirm the loop: download tailored template → fill → upload to existing Validate Onboarding tab to check format (Gary B8). | O6 | O6 |
| J3 | Column set for a subscription + LTV app correct? (in: skad_installs, daus, retentions, revenue/registrations/retentions cohorts; out: views, ad_revenue) | F6/O6 content | F6, worked-example §3 |
| J4 | **Funnel mapping:** Trial Start + Subscription Start → which system columns? (proposed new_orders / total_orders) | **BLOCKS F6** (whether new_orders/total_orders columns join) | F6, worked-example §4 |
| J5 | Cohort days 0/1/7/30/90/360, and exclude `_incr_`/`_lift_` (incremental-only) for normal onboarding — confirm. | F6 | F6 |
| J6 | OK to hand each client a tailored (trimmed) template vs the full master sheet? | product/F6 | design §8c |
| J7 | Template format: CSV (current) vs xlsx with group headers (DIMENSIONS / NON-COHORT / COHORTED) like Anupam's sheet? | O6 (download artifact) | O6 |

### Awaiting Satish (backend / identity)

| # | Question | Gates | Fold into |
|---|----------|-------|-----------|
| S1 | Confirm CSM signal = `is_as_admin` resolved via mos-iam `GetUserByEmail` (mechanism decided; needs sign-off + the exact endpoint). | B5 (implementable on the assumption; confirm before merge) | B5, design §8c |

### Awaiting Gary

| # | Question | Gates | Fold into |
|---|----------|-------|-----------|
| G1 | Confirm "Book my meeting" email copy (subject/body/from/reply-to drafted in §8c). | B9/X2 (copy drafted; implementable) | B9, X2 |

### Execution impact

- **Truly blocked until answered:** only **F6** (buildProvisionPlan columns — needs J4) and **O6** (Data Schema tab state — needs J1). Defer these two.
- **Implementable now on current assumptions** (answers only confirm, not reshape): B5 (is_as_admin via GetUserByEmail), B9/X2 (drafted copy), everything else.
- So the rest of the plan set (vertical slice + backend + foundation + components + wizard + SoW/Timeline outputs) can start without waiting.
