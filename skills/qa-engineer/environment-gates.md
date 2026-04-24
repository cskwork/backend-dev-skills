# Environment Gates

> Phase-0 / Phase-1 companion to `SKILL.md`. Defines which environments `/qa-engineer` may target, the safety preflight checks, and the per-env constraints. The symmetric counterpart to `verify/curl-harness.md §Preflight` — where `/verify` refuses any non-localhost URL, `/qa-engineer` refuses **localhost** and has tighter controls for higher environments.

## Environment Matrix (fork to match your env URLs)

| Env | Purpose | Who uses it | Typical URL pattern | Auth | `/qa-engineer` allowed? | Default posture |
|---|---|---|---|---|---|---|
| `local` / `localhost` | Developer loopback | Engineer | `http://localhost:{3000-8080}` | Dev JWT injection | **No — use `/verify`** | Refuse |
| `dev` | Integration, engineer smoke | Engineers, QA | `https://dev-<svc>.example.com` | Dev JWT OR SSO test account | Yes | Default target |
| `stg` | Stakeholder preview, pre-release | QA, PO | `https://stg-<svc>.example.com` | SSO test account | Yes | Read+write OK (test tenant) |
| `audit` | Audit/compliance rehearsal | Auditors, SRE | `https://audit-<svc>.example.com` | SSO test account | Yes (sparingly) | Read-only unless user explicitly authorizes writes |
| `prod` | Real end users | End users | `https://<svc>.example.com` | SSO real or test account | Yes with guardrails | **Read-only by default**, writes require per-step consent |

URLs are placeholders — resolve from the user's answer and/or the service's `CLAUDE.md`. Fork this file to match your org's actual env hostname patterns.

## Preflight Checks (Phase 1)

Run these before any spec executes. If any check fails, STOP and surface the mismatch.

### Check 1 — BASE_URL is not localhost

```bash
# In _env.sh or _preflight.sh
case "$BASE_URL" in
  http://localhost*|http://127.0.0.1*|http://0.0.0.0*|http://[::1]*)
    echo "REFUSED: BASE_URL=$BASE_URL is localhost. Use /verify for localhost probes." >&2
    exit 2
    ;;
esac
```

If the user supplies a tunnel URL (e.g. `https://tunnel.ngrok.io` forwarding to localhost), require explicit user confirmation that the tunnel target has the deployed-env properties being tested. Log the tunnel usage in `qa.md §4`.

### Check 2 — BASE_URL matches declared QA_ENV

```bash
case "$QA_ENV" in
  dev)    echo "$BASE_URL" | grep -qE '(^https://dev-|^https?://.*-dev\.)' || exit 2 ;;
  stg)    echo "$BASE_URL" | grep -qE '(^https://stg-|^https?://.*-stg\.)' || exit 2 ;;
  audit)  echo "$BASE_URL" | grep -qE '^https://audit-' || exit 2 ;;
  prod)   echo "$BASE_URL" | grep -qvE '(-dev|-stg|-audit|localhost)' || exit 2 ;;
  *) echo "REFUSED: unknown QA_ENV=$QA_ENV" >&2; exit 2 ;;
esac
```

Mismatch (e.g. `QA_ENV=dev` but `BASE_URL` points to prod) is a hard failure. Do not allow a dev-labeled run to hit prod by accident.

### Check 3 — deployment is reachable and healthy

```bash
# Basic liveness — follow documented health endpoint per service
curl -fsS --max-time 5 "${BASE_URL}/actuator/health" > /dev/null \
  || { echo "REFUSED: $BASE_URL health check failed" >&2; exit 2; }
```

For frontend-only Vite-served builds, fall back to `GET /` returning 200 with a non-empty body.

### Check 4 — prod consent

```bash
if [[ "$QA_ENV" == "prod" ]]; then
  if [[ -z "${QA_PROD_CONSENT:-}" ]]; then
    echo "REFUSED: prod runs require QA_PROD_CONSENT=yes-<ticket-slug>" >&2
    exit 2
  fi
  if [[ "${QA_PROD_CONSENT}" != "yes-${TICKET_SLUG}" ]]; then
    echo "REFUSED: QA_PROD_CONSENT must match 'yes-<current ticket slug>'" >&2
    exit 2
  fi
fi
```

The slug-scoped consent prevents an accidental prod run from a re-opened terminal with stale env vars.

## Prod Run Guardrails

When `QA_ENV=prod`, additional constraints apply beyond preflight.

### Default to read-only

All specs under prod run with this fixture applied:

```typescript
// e2e/fixtures/prod-guard.ts
import { test as base, expect } from '@playwright/test';

export const test = base.extend({
  page: async ({ page }, use) => {
    if (process.env.QA_ENV === 'prod') {
      await page.route('**/*', async (route, request) => {
        const method = request.method();
        if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(method) &&
            !process.env.QA_PROD_WRITE_OK) {
          console.warn(`[prod-guard] blocked ${method} ${request.url()}`);
          await route.abort('blockedbyclient');
          return;
        }
        await route.continue();
      });
    }
    await use(page);
  },
});
```

Specs that require writes on prod must:
1. Set `QA_PROD_WRITE_OK=<write-reason>` per step, explicitly.
2. Use a dedicated `prod-write` test tenant account (no real student/teacher data).
3. Clean up after themselves in `afterEach` — delete what they created.
4. Be reviewed line-by-line with the user before the run.
5. Be logged in `qa.md §9 [INFO]` with the exact write operations performed.

### Forbidden on prod — always

- Any destructive operation (DELETE on user data, drop resource, delete item).
- Any action that sends notifications to real end users (mass email, SMS, push).
- Any action that triggers a billable external call.
- Any action that modifies admin-level config (role assignments, feature flags).
- Taking a screenshot that includes PII from a real end user — redact immediately or discard.

If a flow legitimately requires one of these on prod, it is out of scope for `/qa-engineer` — escalate to the engineering lead for a purpose-built procedure.

## Cross-Env Parity Checks

A useful post-deploy technique — run the same smoke suite across dev, stg, prod in sequence and diff the results:

```bash
for env in dev stg; do
  QA_ENV=$env BASE_URL=<resolved> npx playwright test specs/smoke.spec.ts \
    --output=artifacts/run-$(date +%Y%m%d-%H%M)-$env
done

# If prod authorized:
QA_PROD_CONSENT=yes-$TICKET_SLUG QA_ENV=prod BASE_URL=<prod> \
  npx playwright test specs/smoke.spec.ts --project=smoke \
  --output=artifacts/run-$(date +%Y%m%d-%H%M)-prod
```

Record parity outcomes in `qa.md §5` side-by-side. A spec that passes dev + stg but fails prod is usually a config/secrets issue (different encrypted property value, different cache cluster, different SSO endpoint per env).

## Per-Env Fixture Strategy

Different envs have different tenant data shapes. A fixture hardcoded to a dev tenant ID breaks against stg.

```typescript
// e2e/fixtures/env-tenants.ts
const tenants = {
  dev:   { orgId: 1001, adminId: 'test-admin-001', memberId: 'test-member-001' },
  stg:   { orgId: 5001, adminId: 'stg-admin-001',  memberId: 'stg-member-001' },
  audit: { orgId: 7001, adminId: 'aud-admin-001',  memberId: 'aud-member-001' },
  prod:  { orgId: 9001, adminId: 'qa-admin-001',   memberId: 'qa-member-001' },  // QA-dedicated accounts
};
export const currentTenant = tenants[process.env.QA_ENV as keyof typeof tenants];
```

Tenant IDs per env are provided by the user once, persisted in `e2e/fixtures/env-tenants.ts`, and NEVER contain real user IDs.

## SSO Session Lifetime

Captured `storageState.<env>.json` has a finite lifetime. Typical:
- dev JWT: 2 hours (see `verify/jwt-auth-reference.md`).
- stg / audit / prod SSO session: 30–60 min depending on the SSO provider's config.

On spec run start, check session freshness. If older than 15 min, re-run `_login.sh` to refresh. Otherwise the first spec will fail with a 401/302 to the SSO page and the whole run wastes time.

```bash
# _preflight.sh
STATE_FILE="storageState.${QA_ENV}.json"
if [[ ! -f "$STATE_FILE" ]] || \
   [[ $(($(date +%s) - $(stat -f %m "$STATE_FILE"))) -gt 900 ]]; then
  echo "[preflight] storageState stale or missing — refreshing"
  ./_login.sh
fi
```

## Summary — the Opposite of /verify

| Concern | `/verify` | `/qa-engineer` |
|---|---|---|
| Allowed `BASE_URL` | localhost only | dev / stg / audit / prod only |
| Health check | Local process health | Deployed LB + SSO reachability |
| Write allowed | Yes (it's all local) | Dev yes; stg/audit yes on test tenant; prod gated per-step |
| Auth | Test JWT via `/test/generate-jwt` | Env-appropriate: JWT (dev), SSO storageState (stg/audit/prod) |
| Failure on env mismatch | Refuse non-localhost | Refuse localhost + refuse env/URL mismatch |
| Prod posture | N/A | Read-only by default; explicit per-step consent for writes |

The two skills are complementary gates. Both must pass for a change to be considered fully shipped.
