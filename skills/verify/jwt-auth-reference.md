# Auth Reference (Adapt to Your Stack)

Companion to `SKILL.md` Phase 2 and `curl-harness.md` preflight. Defines **how a local curl harness obtains a valid auth credential** for endpoints behind authentication.

> **Fork this file.** The core rules are universal: no real tokens on disk, prefer a dev-only credential path, and reuse one credential across services that trust the same authority. Endpoints, claim names, and config keys are project-specific.

## Core Principle

One authority mints credentials. Every service that trusts that authority, such as a shared signing key, cookie domain, or OAuth issuer, should accept the same credential. Obtain it once and reuse it.

The skill fetches credentials cheaply and deterministically for local verification. It does not simulate production login flows.

## Common Auth Schemes

Pick the section that matches your stack. In a real project, trim this file to the schemes that apply.

### Scheme A: JWT via Dev Test-Token Endpoint

Many backends expose a permit-all, dev-profile-only endpoint for local testing:

```text
GET /test/generate-jwt
GET /dev/token
POST /internal/test-token
```

- Auth: none, dev profile only
- Response: JSON with `token`, `accessToken`, or `data.accessToken`
- Extract with: `jq -r '.token // .accessToken // .data.accessToken // empty'`

If the service has no equivalent, consider adding one behind a dev-only profile. That usually pays for itself after the first `/verify` run.

### Scheme B: OAuth2 Client Credentials

```bash
TOKEN=$(curl -sS --max-time 5 \
  -X POST \
  -u "${CLIENT_ID}:${CLIENT_SECRET}" \
  -d 'grant_type=client_credentials' \
  "${OAUTH_TOKEN_URL}" \
  | jq -r '.access_token')
```

`CLIENT_ID`, `CLIENT_SECRET`, and `OAUTH_TOKEN_URL` come from environment variables or non-prod config. Production credentials never go into harness files.

### Scheme C: Session Cookie from Dev Login

```bash
curl -sS -c cookies.txt \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{"username":"dev-user","password":"dev-pass"}' \
  "${BASE_URL}/auth/login"

curl -b cookies.txt "${BASE_URL}/api/..."
```

Delete `cookies.txt` at the end of the run. Never commit it.

### Scheme D: API Key Header

```bash
export API_KEY="${API_KEY:-}"
if [[ -z "${API_KEY}" ]]; then
  echo "PREFLIGHT FAIL: API_KEY env var is required" >&2
  exit 2
fi
COMMON_HEADERS+=(-H "X-API-Key: ${API_KEY}")
```

Dev fixture keys may be committed only when the project explicitly treats them as public test keys.

### Scheme E: SSO

For SSO-gated services, prefer one of:

- Dev-profile token bypass, as in Scheme A
- OAuth2 resource-owner or client-credentials flow against a dev tenant
- Interactive login once and save browser state, for `/qa-engineer` only

`/verify` is a fast curl loop. If interactive SSO is the only path, add a dev-only credential endpoint instead of teaching the harness to drive a browser.

### Scheme F: mTLS Client Certificate

```bash
curl --cert "${CLIENT_CERT_PATH}" --key "${CLIENT_KEY_PATH}" \
  "${BASE_URL}/api/..."
```

Certificate paths come from env vars. Never commit keys.

### Scheme G: No Auth in Dev Profile

Some services disable auth entirely in `dev`. If so, assert the active profile in `_preflight.sh` and skip token acquisition.

## Harness Integration: Scheme A

Add this to `harness/_env.sh`, adapting names and paths:

```bash
export JWT_ISSUER_URL="${JWT_ISSUER_URL:-http://localhost:8080}"

export JWT_USER_ID="${JWT_USER_ID:-}"
export JWT_USER_ROLE="${JWT_USER_ROLE:-}"
export JWT_TENANT_ID="${JWT_TENANT_ID:-}"

export DEV_LOGIN_METHOD="GET"
export DEV_LOGIN_PATH="/test/generate-jwt"

_qs=""
_add() { [[ -n "$2" ]] && _qs="${_qs:+${_qs}&}$1=$2"; }
_add userId   "${JWT_USER_ID}"
_add userRole "${JWT_USER_ROLE}"
_add tenantId "${JWT_TENANT_ID}"
export DEV_LOGIN_QUERY="${_qs}"
```

Add this to `harness/_preflight.sh`:

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

Endpoint scripts then reuse `Authorization: Bearer ${TOKEN}`.

## Cross-Service Reuse

If multiple services validate tokens signed by the same authority:

- Mint once at the authority, such as `auth-service:8080/test/generate-jwt`
- Reuse that token across downstream services that trust the same key or issuer
- If a downstream rejects a token the authority accepts, first compare the signing key, JWKS URL, issuer, and audience config in each service's non-prod profile

Record the project-specific pattern in your fork so operators do not re-mint per service.

## Anti-Patterns

- Hardcoding a token in a fixture file. Tokens expire and may leak claims.
- Minting offline when the service is running. Prefer the live dev endpoint so the signing config matches runtime.
- Using production credentials for local verification.
- Committing real secrets to `_env.sh`.
- Saving interactive SSO cookies for `/verify`; that belongs in browser QA only.

## Adding a Dev Token Endpoint

When `/verify` is blocked by interactive auth, add a dev-profile-only token endpoint rather than bypassing auth in the skill. Spring Boot sketch:

```java
@RestController
@Profile("dev")
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
        "tenantId", Optional.ofNullable(tenantId).orElse("tenant-dev"),
        "iat", Instant.now().getEpochSecond(),
        "exp", Instant.now().plusSeconds(3600).getEpochSecond()));

    return Map.of("token", token);
  }
}
```

Guard it with a dev-only profile and a permit-all route in dev security config. Document method, path, params, and response shape in this file.

## When to Update This File

- Dev token endpoint path or response shape changes
- Signing algorithm, issuer, audience, or secret source changes
- A new service joins with a different auth authority
- A staging/test profile gets a special token endpoint and needs guardrails

## Bottom Line

The cheapest local auth is a dev-only endpoint that returns a valid credential. One curl, one `jq`, one exported `${TOKEN}`, then every protected endpoint in `/verify` can be called with the same harness.
