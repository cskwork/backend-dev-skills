# Curl Harness

Companion to `SKILL.md` Phase 2–3. The harness is the executable form of the verification contract. Running it must produce recorded, replayable evidence.

## Core Principle

> A harness is valuable only if a human or a future session can replay it and reach the same conclusion. Anything that drifts between runs (tokens, timestamps, random ids) must be parameterized, not hardcoded.

## Layout

```
.backend/<YYYYMM>/<slug>/harness/
  _env.sh                 ← one per ticket; sourced by every endpoint script
  _assert.sh              ← one per ticket; bash assertion helpers
  _preflight.sh           ← one per ticket; safety checks (local target, service reachable)
  <endpoint-slug>.sh      ← one per endpoint target; runs every variant
```

`<endpoint-slug>` is a kebab-case rendering of METHOD + path, e.g.:

- `POST /api/v1/auth/login` → `post-api-v1-auth-login.sh`
- `GET /api/v1/items/{itemId}` → `get-api-v1-items-id.sh`

One harness file per endpoint, not per variant. Variants are iterated inside the file.

All scripts are `chmod +x`. All scripts assume `set -euo pipefail`.

## `_env.sh` Template

```bash
#!/usr/bin/env bash
# Environment for verify harness. DO NOT commit real tokens.
# Regenerate by /verify Phase 2. Values here are session-scoped.

# ---- Target (default localhost; remote envs require VERIFY_REMOTE_ACK) ----
export BASE_URL="${BASE_URL:-http://localhost:8080}"
export SERVICE_NAME="<your-service-name>"
export SERVICE_PROFILE="dev"

# ---- Remote-env consent (set by caller; preflight reads, never persisted) -
# VERIFY_REMOTE_ACK=<env>   one of: dev | stg | audit | prod   (matches BASE_URL host)
# VERIFY_PROD_ACK=YYYYMMDD-HHMM  required per-request when env=prod
# Both come from environment vars only. Do not hardcode here.

# ---- Auth (populated at runtime by _preflight.sh; never a real prod token)
export TOKEN="${TOKEN:-}"                          # Bearer token if endpoint needs it
export SESSION_COOKIE="${SESSION_COOKIE:-}"        # if endpoint uses session auth

# ---- Common headers -------------------------------------------------------
export COMMON_HEADERS=(
  -H "Content-Type: application/json"
  -H "Accept: application/json"
  -H "X-Request-Id: verify-$(date +%s)-$$"
)

# ---- Timing / limits ------------------------------------------------------
export CURL_MAX_TIME=10                            # seconds; hard cap per request
export RUNS_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/../runs" && pwd)"
export CURRENT_RUN="${CURRENT_RUN:-1}"             # passed by caller; which run-N.log to append to
export RUN_LOG="${RUNS_DIR}/run-${CURRENT_RUN}.log"

mkdir -p "${RUNS_DIR}"
```

Values are emitted by Phase 2 based on:
- `SERVICE_NAME` — derived from `work.md §4` touched services.
- `BASE_URL` — defaults to localhost + port from the service's `CLAUDE.md` / `application-dev.yml` / `.env.dev` (match the project's convention — 8080, 3000, 8000, etc.). For remote envs, the caller exports `BASE_URL` and `VERIFY_REMOTE_ACK=<env>` per the Consent Record in `verify.md §0`.
- `SERVICE_PROFILE` — dev unless the user specifies.
- `TOKEN` — populated by `_preflight.sh` for localhost; for remote envs it must be exported by the caller (env var only) and is never written to the env file.

Never write real tokens into `_env.sh`. Localhost: preflight obtains them at run time. Remote: caller exports per-session.

## `_preflight.sh` Template

Runs before any variant. If any check fails, the whole harness aborts non-zero — no partial evidence.

```bash
#!/usr/bin/env bash
set -euo pipefail
source "$(dirname "${BASH_SOURCE[0]}")/_env.sh"

# 1. Classify target env. Remote envs require an explicit consent flag.
LOCAL_RE='^https?://(localhost|127\.0\.0\.1|0\.0\.0\.0|host\.docker\.internal)(:[0-9]+)?(/|$)'
if [[ "${BASE_URL}" =~ ${LOCAL_RE} ]]; then
  TARGET_ENV="local"
else
  TARGET_ENV="${VERIFY_REMOTE_ACK:-}"
  if [[ -z "${TARGET_ENV}" ]]; then
    echo "PREFLIGHT FAIL: BASE_URL='${BASE_URL}' is non-local but VERIFY_REMOTE_ACK is unset." >&2
    echo "  Record consent in verify.md §0, then export VERIFY_REMOTE_ACK=<dev|stg|audit|prod>." >&2
    exit 2
  fi
  case "${TARGET_ENV}" in
    dev|stg|audit|prod) : ;;
    *) echo "PREFLIGHT FAIL: VERIFY_REMOTE_ACK='${TARGET_ENV}' invalid. Use dev|stg|audit|prod." >&2; exit 2 ;;
  esac
  if [[ "${TARGET_ENV}" == "prod" ]]; then
    if [[ -z "${VERIFY_PROD_ACK:-}" ]]; then
      echo "PREFLIGHT FAIL: prod requires per-request VERIFY_PROD_ACK=YYYYMMDD-HHMM." >&2
      exit 2
    fi
    EXPECTED_ACK="$(date -u +%Y%m%d-%H%M)"
    if [[ "${VERIFY_PROD_ACK}" != "${EXPECTED_ACK}" && "${VERIFY_PROD_ACK}" != "$(date -u -v-1M +%Y%m%d-%H%M 2>/dev/null || true)" ]]; then
      echo "PREFLIGHT FAIL: VERIFY_PROD_ACK='${VERIFY_PROD_ACK}' is stale (expected ${EXPECTED_ACK})." >&2
      echo "  Re-confirm with the user and re-export with the current minute stamp." >&2
      exit 2
    fi
  fi
fi
export TARGET_ENV

# 2. Service must be reachable. Probe the readiness endpoint.
HEALTH_PATH="${HEALTH_PATH:-/actuator/health}"
if ! curl -sS -o /dev/null -w "%{http_code}" --max-time 5 "${BASE_URL}${HEALTH_PATH}" | grep -q "^200$"; then
  echo "PREFLIGHT FAIL: ${SERVICE_NAME} at ${BASE_URL} is not ready (${HEALTH_PATH} != 200)." >&2
  exit 2
fi

# 3. Profile check — local only. Remote envs are trusted to be the env the user named.
if [[ "${TARGET_ENV}" == "local" ]]; then
  if curl -sS -o /dev/null -w "%{http_code}" --max-time 3 "${BASE_URL}/actuator/info" | grep -q "^200$"; then
    ENV_PROFILES=$(curl -sS --max-time 3 "${BASE_URL}/actuator/info" | jq -r '.activeProfiles // empty' 2>/dev/null || echo "")
    if [[ -n "${ENV_PROFILES}" && "${ENV_PROFILES}" != *"${SERVICE_PROFILE}"* ]]; then
      echo "PREFLIGHT FAIL: active profiles '${ENV_PROFILES}' do not include '${SERVICE_PROFILE}'." >&2
      exit 2
    fi
  fi
fi

# 4. Token acquisition.
#    Local: auto-mint via dev JWT endpoint if DEV_LOGIN_PATH is set.
#    Remote: TOKEN must be exported by caller (env var only). Preflight refuses to mint.
if [[ -z "${TOKEN}" ]]; then
  if [[ "${TARGET_ENV}" == "local" && -n "${DEV_LOGIN_PATH:-}" ]]; then
    TOKEN=$(curl -sS --max-time 5 \
      -X "${DEV_LOGIN_METHOD:-GET}" \
      -H "Accept: application/json" \
      "${JWT_ISSUER_URL:-${BASE_URL}}${DEV_LOGIN_PATH}${DEV_LOGIN_QUERY:+?$DEV_LOGIN_QUERY}" \
      | jq -r '.token // .data.accessToken // .accessToken // empty')
    if [[ -z "${TOKEN}" ]]; then
      echo "PREFLIGHT FAIL: ${DEV_LOGIN_PATH} did not return a token. See jwt-auth-reference.md. Is ${SERVICE_NAME} running on ${JWT_ISSUER_URL:-${BASE_URL}} with the dev profile active?" >&2
      exit 2
    fi
    export TOKEN
  elif [[ "${TARGET_ENV}" != "local" ]]; then
    echo "PREFLIGHT NOTE: TOKEN unset for remote env '${TARGET_ENV}'. Export it (env var) before running variants that need auth." >&2
  fi
fi

echo "PREFLIGHT OK: ${SERVICE_NAME} @ ${BASE_URL}  env=${TARGET_ENV}  profile=${SERVICE_PROFILE}  auth=$([[ -n "${TOKEN}" ]] && echo present || echo none)"
```

## `_assert.sh` Template

Pure bash + jq. Each helper logs the assertion's outcome to `${RUN_LOG}`.

```bash
#!/usr/bin/env bash
# Assertion helpers. Keep deterministic and side-effect-free except for appending to ${RUN_LOG}.

_log() {
  printf '[%s] %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$*" >> "${RUN_LOG}"
}

# Globals used by assertions — populated by run_variant()
: "${LAST_STATUS:=}"
: "${LAST_BODY_FILE:=}"
: "${LAST_HEADERS_FILE:=}"
: "${LAST_ELAPSED_MS:=}"

assert_status() {
  local expected="$1"
  if [[ "${LAST_STATUS}" == "${expected}" ]]; then
    _log "PASS  assert_status ${expected}"
    return 0
  fi
  _log "FAIL  assert_status expected=${expected} observed=${LAST_STATUS}"
  return 1
}

assert_json_path() {
  local path="$1" expected="$2"
  local observed
  observed=$(jq -r "${path}" < "${LAST_BODY_FILE}" 2>/dev/null || echo "__JQ_ERR__")
  if [[ "${observed}" == "${expected}" ]]; then
    _log "PASS  assert_json_path ${path} = ${expected}"
    return 0
  fi
  _log "FAIL  assert_json_path ${path} expected='${expected}' observed='${observed}'"
  return 1
}

assert_json_key_present() {
  local path="$1"
  if jq -e "${path}" < "${LAST_BODY_FILE}" > /dev/null 2>&1; then
    _log "PASS  assert_json_key_present ${path}"
    return 0
  fi
  _log "FAIL  assert_json_key_present ${path} missing or null"
  return 1
}

assert_header() {
  local name="$1" expected="$2"
  local observed
  # grep -i for case-insensitive header name; strip CR; trim whitespace
  observed=$(grep -i "^${name}:" "${LAST_HEADERS_FILE}" | head -n1 | sed -E "s/^[^:]*:[[:space:]]*//; s/\r$//")
  if [[ "${observed}" == "${expected}" ]] || [[ "${observed}" == "${expected}"* ]]; then
    _log "PASS  assert_header ${name} ~ ${expected}"
    return 0
  fi
  _log "FAIL  assert_header ${name} expected='${expected}' observed='${observed}'"
  return 1
}

assert_latency_under() {
  local max_ms="$1"
  if [[ "${LAST_ELAPSED_MS}" -lt "${max_ms}" ]]; then
    _log "PASS  assert_latency_under ${max_ms}ms (observed ${LAST_ELAPSED_MS}ms)"
    return 0
  fi
  _log "FAIL  assert_latency_under ${max_ms}ms (observed ${LAST_ELAPSED_MS}ms)"
  return 1
}
```

**Do not add a "soft assert" mode.** Every assertion is hard — a failure means the variant fails. Soft modes silently accumulate debt.

## `<endpoint>.sh` Template

```bash
#!/usr/bin/env bash
set -euo pipefail

HERE="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${HERE}/_env.sh"
source "${HERE}/_assert.sh"
"${HERE}/_preflight.sh"

FIXTURES="$(cd "${HERE}/../fixtures" && pwd)"
ENDPOINT_METHOD="POST"                       # filled by Phase 2 generator
ENDPOINT_PATH="/api/v1/auth/login"           # filled by Phase 2 generator
ENDPOINT_LABEL="post-api-v1-auth-login"      # filled by Phase 2 generator

# Accumulate results; exit non-zero if any variant fails
FAILED_VARIANTS=()

run_variant() {
  local variant="$1"
  local fixture="${FIXTURES}/${ENDPOINT_LABEL}.${variant}.json"
  local body_file headers_file tmp_status tmp_elapsed

  if [[ ! -f "${fixture}" ]]; then
    _log "SKIP  ${ENDPOINT_LABEL} ${variant} (fixture missing: ${fixture})"
    FAILED_VARIANTS+=("${variant}:missing-fixture")
    return
  fi

  body_file=$(mktemp)
  headers_file=$(mktemp)

  _log "RUN   ${ENDPOINT_METHOD} ${ENDPOINT_PATH}  variant=${variant}"
  _log "REQ   $(jq -c . < "${fixture}" | head -c 1000)"          # truncated for log sanity

  # -w emits status + elapsed to stdout on a known line; headers → -D; body → -o
  local curl_output
  curl_output=$(curl -sS -i \
    --max-time "${CURL_MAX_TIME}" \
    -X "${ENDPOINT_METHOD}" \
    "${COMMON_HEADERS[@]}" \
    ${TOKEN:+-H "Authorization: Bearer ${TOKEN}"} \
    --data-binary @"${fixture}" \
    -D "${headers_file}" \
    -o "${body_file}" \
    -w "\n__VERIFY_META__status=%{http_code}\telapsed_ms=%{time_total}\n" \
    "${BASE_URL}${ENDPOINT_PATH}" || echo "__CURL_EXIT__=$?")

  LAST_STATUS=$(echo "${curl_output}" | awk -F'\t' '/^__VERIFY_META__/{sub(/^__VERIFY_META__status=/,"",$1); print $1}')
  local elapsed_s
  elapsed_s=$(echo "${curl_output}" | awk -F'\t' '/^__VERIFY_META__/{sub(/^elapsed_ms=/,"",$2); print $2}')
  LAST_ELAPSED_MS=$(awk "BEGIN{printf \"%d\", ${elapsed_s:-0} * 1000}")
  LAST_BODY_FILE="${body_file}"
  LAST_HEADERS_FILE="${headers_file}"

  # Redact Authorization in the logged headers before appending
  sed -E 's/^(Authorization:).*/\1 Bearer __REDACTED__/i' "${headers_file}" >> "${RUN_LOG}"
  echo "--- body ---" >> "${RUN_LOG}"
  head -c 4000 "${body_file}" >> "${RUN_LOG}"
  echo >> "${RUN_LOG}"
  echo "--- meta --- status=${LAST_STATUS} elapsed_ms=${LAST_ELAPSED_MS}" >> "${RUN_LOG}"

  # Variant-specific assertions — filled in by Phase 2 generator per variant.
  # The harness exits with non-zero if assertions array contains any failure.
  local -a assertion_failures=()
  case "${variant}" in
    happy)
      assert_status 200        || assertion_failures+=("status")
      assert_header 'Content-Type' 'application/json' || assertion_failures+=("ct")
      assert_json_key_present '.data.userId' || assertion_failures+=("userId")
      assert_latency_under 2000 || assertion_failures+=("latency")
      ;;
    boundary)
      assert_status 200 || assertion_failures+=("status")
      # add boundary-specific field checks here
      ;;
    negative)
      assert_status 400 || assertion_failures+=("status")
      assert_json_key_present '.error.code' || assertion_failures+=("errorCode")
      ;;
    regression)
      # REPLACE with the exact expected outcome for this ticket's bug
      assert_status 200 || assertion_failures+=("status")
      ;;
  esac

  if (( ${#assertion_failures[@]} > 0 )); then
    FAILED_VARIANTS+=("${variant}:${assertion_failures[*]}")
  fi

  rm -f "${body_file}" "${headers_file}"
}

# ---- Variants (ordered: happy → boundary → negative → regression if any) --
run_variant happy
run_variant boundary
run_variant negative
# run_variant regression    # uncomment for bug-fix tickets

if (( ${#FAILED_VARIANTS[@]} > 0 )); then
  printf '%s\n' "${FAILED_VARIANTS[@]}" >&2
  exit 1
fi
```

## Result Matrix Emission

After executing all endpoint scripts in Phase 3, build the result matrix by scanning `${RUN_LOG}`:

```
endpoint                            variant     status  latency(ms)  assertions  result
POST /api/v1/auth/login             happy       200     142          4/4         ✓
POST /api/v1/auth/login             boundary    200     198          2/2         ✓
POST /api/v1/auth/login             negative    400     89           2/2         ✓
POST /api/v1/auth/login             regression  500     154          1/3         ✗
```

Copy this block verbatim into `verify.md §5 Results Matrix`. Do not summarize. Do not reorder.

## Cross-Service Probe Pattern

If `work.md §7` declared a cross-service change (HTTP client / queue / SSE / WebSocket), add a **cross-service endpoint** entry to the harness that probes the downstream expectation:

- **HTTP client (synchronous):** direct curl to the downstream service's corresponding endpoint with the payload the caller service would build. Confirms the downstream accepts the request the caller produces.
- **Message queue (asynchronous):** start a consumer in a parallel process that tails the topic/queue with a timeout; publish via the producer endpoint; assert the message arrived with the expected shape. If no console consumer is available, document this as a `[GAP]` in `verify.md §9`.
- **SSE / WebSocket:** use `curl -N` (SSE) or a tiny `websocat` probe (WS) with a 5s timeout. Assert the first event's shape. Kill after the assertion.

Do not try to replicate every downstream — probe only the edges `work.md §7` lists as changed.

## Idempotence & Cleanup

- Every variant that creates state must have a paired teardown variant with its own assertions (e.g. `cleanup` that issues `DELETE` + asserts 204).
- If the endpoint itself is `DELETE` or destructive, require a seed step: the harness first creates a throwaway resource via an adjacent endpoint (cited from the service's test fixtures), then deletes it.
- Never leave the DB in a polluted state after a run unless the user explicitly asks to inspect leftovers (record in `verify.md §9 [INFO]`).

## Re-Run Protocol

After Phase 5 re-invokes `/work`:

1. **Do not regenerate fixtures** unless the contract itself changed. The whole point of the iteration is isolating what changed in code vs. what was observed.
2. **Do regenerate `_env.sh` TOKEN** — dev tokens often expire across `/work` runs because the service restarts.
3. `CURRENT_RUN` is incremented by the caller; the harness appends to a new `runs/run-<N+1>.log`.
4. Diff `run-<N>.log` vs `run-<N+1>.log` at the assertion-outcome level — copy the diff into `verify.md §7 Iteration Trail`.

## What the Harness Never Does

- Never runs against stg/audit/prod **without `VERIFY_REMOTE_ACK` set per the Consent Record** in `verify.md §0`. Preflight enforces. Prod additionally requires per-request `VERIFY_PROD_ACK=YYYYMMDD-HHMM`.
- Never persists remote-env tokens to `_env.sh` or any file. Caller exports them as env vars per session.
- Never writes production-like data to dev DB at scale (bulk inserts, load tests). Scope is functional verification, not load.
- Never stores real tokens or real PII. Redaction is enforced at write time — and is **mandatory** for any capture from a remote env.
- Never retries a failed variant "just in case". A failure is a signal for Phase 4 triage, not a transient to hide.
- Never modifies source files. `/verify` is read-only on `src/` and `test/`.
- Never opens a PR, pushes a branch, or updates a ticket. Those are post-skill user actions.

## Bottom Line

The harness is the executable contract. It is the difference between "I think it works" and "run this file and see for yourself." Keep it reproducible, scoped (localhost by default; remote envs only via the consent gate), redacted, and bounded. When it exits zero, you have earned the right to claim the change is production-ready — and not a line sooner.
