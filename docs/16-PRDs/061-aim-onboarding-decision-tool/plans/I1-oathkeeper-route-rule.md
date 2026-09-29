---
id: plan-i1
title: "I1 — Oathkeeper Route Rule"
---

## I1 — Oathkeeper Route Rule

**Workstream:** Infra — `ko-k8s-apps`
**Depends on:** none
**Asana parent:** T201

---

## Goal

Add one Oathkeeper access rule that exposes `…/onboarding-sessions<.*>` (GET / POST / PUT)
to the portal, in both prod and qa. The rule mirrors `mmm-portal-advertiser-saved-views`
exactly — same `nng-sprinkler-auth` api-key authorizer, `noop` mutators. Role enforcement
(CSM-only 403 gate) lives entirely in `mmm-portal-api` (plan B5) and is not expressed here;
Oathkeeper carries no role claims on api-key routes.

---

## Files

| Env | Absolute path |
|-----|---------------|
| prod | `ko-k8s-apps/oathkeeper/proxy/prod/oathkeeper-rules-access/mmm.kochava.com/portal/portal-mmm-kochava-com-api.yaml` |
| qa | `ko-k8s-apps/oathkeeper/proxy/qa/oathkeeper-rules-access/mmm.kochava.com/portal/portal-mmm-kochava-com-api.yaml` |

Both files receive one appended rule block. No other files change.

---

## Dependencies

None. This rule can be merged independently of all backend and frontend work; the
upstream service (`mmm-portal-api-service`) will return 404 until B4 lands, which is
the expected behavior while the feature is in flight.

---

## Tasks

### T1 — Read the prod YAML and understand the saved-views rule

Open the prod file and locate the `mmm-portal-advertiser-saved-views` block (near the
bottom). Confirm the upstream URL, authorizer remote, and payload shape. No edits yet.

Reference rule (prod):

```yaml
# Saved Views API - all methods on base and sub-paths
- id: mmm-portal-advertiser-saved-views
  upstream:
    url: http://mmm-portal-api-service.mmm.svc.clusterset.local
    preserve_host: true
  match:
    url: http://portal.mmm.kochava.com/mmm/advertisers/<[a-zA-Z0-9\-]+>/saved-views<.*>
    methods:
      - GET
      - POST
      - PUT
      - DELETE
  authenticators:
  - handler: noop
  authorizer:
    handler: remote_json
    config:
      remote: http://nng-sprinkler-auth.auth.svc.cluster.local/auth/z/account
      payload: |
        {
          "api_key": "{{ .MatchContext.Header.Get "Authentication-Key" }}",
          "account_id": "{{ printIndex .MatchContext.RegexpCaptureGroups 0 }}"
        }
  mutators:
  - handler: noop
  errors:
  - handler: json
```

### T2 — Append the onboarding-sessions rule to prod

Append the following block at the end of the prod YAML, after the last existing rule.
Copy saved-views exactly; change only `id` and the `url` match. Drop `DELETE` — the API
contract (design-consolidated.md §6) has no DELETE endpoint.

```yaml
# AIM Onboarding Sessions API - all methods on base and sub-paths
- id: mmm-portal-advertiser-onboarding-sessions
  upstream:
    url: http://mmm-portal-api-service.mmm.svc.clusterset.local
    preserve_host: true
  match:
    url: http://portal.mmm.kochava.com/mmm/advertisers/<[a-zA-Z0-9\-]+>/onboarding-sessions<.*>
    methods:
      - GET
      - POST
      - PUT
  authenticators:
  - handler: noop
  authorizer:
    handler: remote_json
    config:
      remote: http://nng-sprinkler-auth.auth.svc.cluster.local/auth/z/account
      payload: |
        {
          "api_key": "{{ .MatchContext.Header.Get "Authentication-Key" }}",
          "account_id": "{{ printIndex .MatchContext.RegexpCaptureGroups 0 }}"
        }
  mutators:
  - handler: noop
  errors:
  - handler: json
```

Diff check: the only differences from saved-views are:

- `id`: `mmm-portal-advertiser-onboarding-sessions`
- `match.url`: `…/onboarding-sessions<.*>` (was `…/saved-views<.*>`)
- `methods`: GET / POST / PUT only (no DELETE)

Everything else — upstream service URL, `preserve_host`, authenticator, authorizer
remote + payload, mutators, errors — is identical.

### T3 — Append the onboarding-sessions rule to qa

Repeat T2 for the qa file. The only structural difference between prod and qa rules
is the cluster DNS suffix and the hostname:

- Upstream: `…mmm.svc.cluster.local` (qa) vs `…mmm.svc.clusterset.local` (prod)
- Hostname: `portal.mmm.qa.kochava.com` (qa) vs `portal.mmm.kochava.com` (prod)

Rule to append at the end of the qa YAML:

```yaml
# AIM Onboarding Sessions API - all methods on base and sub-paths
- id: mmm-portal-advertiser-onboarding-sessions
  upstream:
    url: http://mmm-portal-api-service.mmm.svc.cluster.local
    preserve_host: true
  match:
    url: http://portal.mmm.qa.kochava.com/mmm/advertisers/<[a-zA-Z0-9\-]+>/onboarding-sessions<.*>
    methods:
      - GET
      - POST
      - PUT
  authenticators:
  - handler: noop
  authorizer:
    handler: remote_json
    config:
      remote: http://nng-sprinkler-auth.auth.svc.cluster.local/auth/z/account
      payload: |
        {
          "api_key": "{{ .MatchContext.Header.Get "Authentication-Key" }}",
          "account_id": "{{ printIndex .MatchContext.RegexpCaptureGroups 0 }}"
        }
  mutators:
  - handler: noop
  errors:
  - handler: json
```

### T4 — Lint and diff review

Run a YAML lint pass on both modified files to confirm the appended blocks parse cleanly:

```bash
# from ko-k8s-apps root
yamllint oathkeeper/proxy/prod/oathkeeper-rules-access/mmm.kochava.com/portal/portal-mmm-kochava-com-api.yaml
yamllint oathkeeper/proxy/qa/oathkeeper-rules-access/mmm.kochava.com/portal/portal-mmm-kochava-com-api.yaml
```

Then review the diff — expected output: two new rule blocks added, zero deletions,
no whitespace changes to existing rules.

```bash
git diff oathkeeper/
```

### T5 — Commit and open PR

Commit message:

```text
feat(infra): add Oathkeeper route rule for onboarding-sessions (prod + qa)

Exposes /mmm/advertisers/{id}/onboarding-sessions<.*> (GET/POST/PUT) via
nng-sprinkler-auth api-key authorizer, mirroring mmm-portal-advertiser-saved-views.
Role enforcement is in mmm-portal-api (B5), not the gateway.
```

Open a PR against `main`. No additional reviewers required beyond the standard infra
review — this is a mechanical copy of the saved-views rule.

### T6 — Verify after merge (qa first)

Once the PR is merged and the Oathkeeper pods have reloaded (ConfigMap reconcile,
typically < 2 min), verify the rule is active in qa:

```bash
# Should return 401 (rule matched, api-key missing) — NOT 404
curl -v -X GET \
  "https://portal.mmm.qa.kochava.com/mmm/advertisers/test-advertiser-id/onboarding-sessions" \
  | head -5
```

Expected: `HTTP/2 401` with a JSON error body from Oathkeeper.
A `404` means the rule is not yet loaded or the URL match is wrong.
A `200` with an empty response means B4 is already deployed (fine).

Then repeat for prod after the qa signal is clean.

---

## Auth rationale (do not revisit here)

From `design-consolidated.md §7`: `nng-sprinkler-auth` is status-only and `remote_json`
cannot forward a role claim; api-key routes carry no Subject. CSM-only enforcement
(`approve`, `tier-override`) is implemented in `mmm-portal-api` by resolving
`is_as_admin` from mos-iam and returning 403 via `HandleDataUpdateResult`. Oathkeeper
carries no role logic for this feature.
