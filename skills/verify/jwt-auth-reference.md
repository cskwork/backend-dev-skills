# Auth Reference (Adapt to Your Stack)

Companion to `SKILL.md` Phase 2 and `curl-harness.md` preflight. Defines **how a local curl harness obtains a valid auth credential** for any endpoint that sits behind authentication.

> **Fork this file.** The core rules (no real tokens on disk, use a dev-only path, reuse the same token across same-signing-key services) are universal. The specific endpoints, claim names, and config keys are per-project — adapt them to your codebase before running `/verify`.

## Core Principle

> One authority mints credentials. Every service that trusts that authority (same signing key, same cookie domain, same OAuth issuer) accepts the same credential. Obtain it once; reuse.

The skill's job is to fetch a credential cheaply and deterministically for local verification. It is not to simulate production auth flows.

---

## Common Auth Schemes — Quick Reference

Pick the section that matches your stack. Fork this file to narrow it to just yours and remove the rest.

### Scheme A — JWT via a dev-profile "test-token" endpoint (recommended when it exists)

Many backends expose a permit-all, dev-profile-only endpoint for local testing:

```
GET /test/generate-jwt   (or /dev/token, /internal/test-jwt, etc.)
```

- Auth: none (permit-all, dev profile only)
- Response: JSON with a `token` (or `accessToken`) field
- Extract with: `jq -r '.token // .accessToken // .data.accessToken'`

If such an endpoint does not exist, consider adding one guarded by profile (see §Adding a Dev Token Endpoint at the end of this file). It pays for itself after the first `/verify` run.

### Scheme B — OAuth2 client_credentials

For service-to-service auth:

```bash
TOKEN=$(curl -sS --max-time 5 \
  -X POST \
  -u "${CLIENT_ID}:${CLIENT_SECRET}" \
  -d 'grant_type=client_credentials' \
  "${OAUTH_TOKEN_URL}" \
  | jq -r '.access_token')
```

`CLIENT_ID` / `CLIENT_SECRET` / `OAUTH_TOKEN_URL` come from env vars or the project's `application-dev.yml`. Dev credentials are committed; production credentials are not.

### Scheme C — Session cookie from a dev login endpoint

For session-authenticated apps:

```bash
curl -sS -c cookies.txt \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{"username":"dev-user","password":"dev-pass"}' \
  "${BASE_URL}/auth/login"

# Subsequent requests use the cookie jar
curl -b cookies.txt "${BASE_URL}/api/..."
```

Delete `cookies.txt` at end of run. Never commit.

### Scheme D — API key header

For API-key-authenticated services:

```bash
export API_KEY="${API_KEY:-}"   # from env var, never hardcoded
if [[ -z "${API_KEY}" ]]; then
  echo "PREFLIGHT FAIL: API_KEY env var is required" >&2
  exit 2
fi
COMMON_HEADERS+=(-H "X-API-Key: ${API_KEY}")
```

Dev-profile API keys may be committed as fixtures if the service treats them as public test keys; production keys never touch the repo.

### Scheme E — SSO (Keycloak / Auth0 / Okta / Azure AD / in-house)

For SSO-gated services, prefer one of:

- **Dev-profile bypass** (Scheme A) if your service offers it
- **OAuth2 resource-owner-password flow** (Scheme B variant) against a dev tenant with a test user
- **Interactive login once, save state** — for `/qa-engineer` (browser), never for `/verify` (curl)

`/verify` is a fast-feedback loop; if SSO's interactive path is the only way in, add a dev-only bypass (Scheme A) — see §Adding a Dev Token Endpoint.

### Scheme F — mTLS client certificate

```bash
curl --cert "${CLIENT_CERT_PATH}" --key "${CLIENT_KEY_PATH}" \
  "${BASE_URL}/api/..."
```

Cert/key paths come from env vars. Never commit keys; `.gitignore` a local `certs/` directory.

### Scheme G — No auth (dev profile)

Some services disable auth entirely in the `dev` profile. If so, assert the active profile is `dev` in `_preflight.sh` and skip token acquisition.

---

## Harness Integration Snippet (Scheme A — JWT dev endpoint)

Paste into `harness/_env.sh` (adapt token path, claims, port to your service):

```bash
# ---- JWT — Scheme A (dev /test/generate-jwt or equivalent) -----------------
export JWT_ISSUER_URL="${JWT_ISSUER_URL:-http://localhost:8080}"

# Optional test-user claims; adjust to your service's dev-token controller
export JWT_USER_ID="${JWT_USER_ID:-}"
export JWT_USER_ROLE="${JWT_USER_ROLE:-}"
export JWT_TENANT_ID="${JWT_TENANT_ID:-}"

# DEV_LOGIN_PATH / DEV_LOGIN_METHOD read by _preflight.sh to obtain TOKEN
export DEV_LOGIN_METHOD="GET"
export DEV_LOGIN_PATH="/test/generate-jwt"

# Query-string builder (only includes set params; avoids '&&' and trailing '&')
_qs=""
_add() { [[ -n "$2" ]] && _qs="${_qs:+${_qs}&}$1=$2"; }
_add userId   "${JWT_USER_ID}"
_add userRole "${JWT_USER_ROLE}"
_add tenantId "${JWT_TENANT_ID}"
export DEV_LOGIN_QUERY="${_qs}"
```

Update `harness/_preflight.sh` token acquisition block:

```bash
if [[ -z "${TOKEN}" && -n "${DEV_LOGIN_PATH:-}" ]]; then
  TOKEN=$(curl -sS --max-time 5 \
    -X "${DEV_LOGIN_METHOD:-GET}" \
    -H "Accept: application/json" \
    "${JWT_ISSUER_URL}${DEV_LOGIN_PATH}${DEV_LOGIN_QUERY:+?$DEV_LOGIN_QUERY}" \
    | jq -r '.token // .accessToken // .data.accessToken // empty')
  if [[ -z "${TOKEN}" ]]; then
    echo "PREFLIGHT FAIL: ${DEV_LOGIN_PATH} did not return a token. Is the service running on ${JWT_ISSUER_URL} with the dev profile active?" >&2
    exit 2
  fi
  export TOKEN
fi
```

Then any endpoint curl reuses the same `Authorization: Bearer ${TOKEN}` header (see `curl-harness.md` `<endpoint>.sh` template).

---

## Cross-Service Reuse Note

If multiple services validate tokens signed with the **same** key (common in a monorepo where one service is the "auth authority"):

- Mint once at the authority (e.g. `auth-service:8080/test/generate-jwt`)
- Reuse the token across any downstream service that trusts the same signing key
- If a downstream rejects a token the authority accepts, the first thing to check is the `secret-key` / JWKS URL in each service's `application-dev.yml` — divergence is almost always the cause

Record the cross-service token reuse pattern in your forked version of this file so operators don't re-mint per service.

---

## Anti-Patterns

- **Hardcoding a token in a fixture file.** Tokens include `exp`. They go stale. Always derive them fresh via `_preflight.sh`.
- **Minting a token offline when the service is running.** Offline minting requires matching the service's current secret exactly; if `application-dev.yml` is overridden, your offline token will be rejected. Prefer a live dev endpoint.
- **Using production credentials for local verification.** The whole point of a dev token scheme is to keep prod credentials out of `/verify`'s blast radius. If you need prod creds to test locally, you need a dev-profile bypass — see below.
- **Committing real secrets to `_env.sh`.** Dev-only, well-known secrets may be committed if the project treats them as public; production secrets never are.

---

## Adding a Dev Token Endpoint (when none exists)

If `/verify` against your service is blocked by an interactive SSO flow, the right fix is a dev-profile-only permit-all token endpoint — not bypassing auth in the verify skill. Rough template (Spring Boot):

```java
@RestController
@Profile("dev")  // present only in dev profile
@RequestMapping("/test")
public class DevTokenController {

  private final TokenIssuer tokens;

  @GetMapping("/generate-jwt")
  public Map<String, Object> generate(
      @RequestParam(required = false) String userId,
      @RequestParam(required = false) String userRole,
      @RequestParam(required = false) String tenantId) {

    String token = tokens.issue(Map.of(
        "sub", Optional.ofNullable(userId).orElse("dev-user-001"),
        "role", Optional.ofNullable(userRole).orElse("USER"),
        "tenantId", Optional.ofNullable(tenantId).orElse("t-dev"),
        "iat", Instant.now().getEpochSecond(),
        "exp", Instant.now().plusSeconds(3600).getEpochSecond()));

    return Map.of("token", token, "userId", userId, "userRole", userRole);
  }
}
```

Guarded by `@Profile("dev")`, the controller is absent in staging/prod builds. Make sure the dev profile's security config has an `/test/**` permit-all rule. Document the endpoint (method, path, query params, response shape) in this file's forked version.

---

## When to Update This File

- Dev token endpoint signature or path changes → update §Scheme A table.
- Signing algorithm changes (e.g. HS256 → RS256) → update any offline-minting notes.
- A new service joins with a *different* secret key → add to §Cross-Service Reuse.
- A staging/audit profile gets a test endpoint added → flag it here so no one accidentally points the harness at it.

## Bottom Line

The cheapest auth is a dev-only permit-all endpoint that hands back a valid credential. One curl, one `jq`, one exported `${TOKEN}` — and every protected endpoint in `/verify`'s local probe becomes callable. Fork this file, trim it to your scheme, and commit the forked version with your skills.
