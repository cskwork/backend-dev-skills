---
name: verify
description: Use when code changes have been made to backend HTTP endpoints and HTTP-level proof against a running service is needed — typically after /work completes, or standalone when you have an endpoint plus at least one payload to probe ad-hoc
---

# Verify

## Overview

`/verify` is the final gate before a change ships. Unit tests prove the code compiles and the methods return what their authors expected. **They do not prove the running service accepts real payloads, returns the documented envelope, preserves cross-service contracts, or survives the weird shapes production traffic actually produces.** `/verify` does that.

**Core principle:** No completion claim is trustworthy until a curl against a running service produces the right response body, status code, and side effects — with fresh, recorded output. Everything else is paraphrase.

**Design goals every run must optimize for:**
- **Evidence, not assertion** — every "pass" is backed by recorded `curl -i` output stored under the ticket folder
- **Payload realism** — fixtures come from real row shapes (DB sample) or user-provided cases, not invented JSON
- **Variant coverage** — happy path + at least one boundary + at least one negative per endpoint
- **Iteration discipline** — on failure, pass a *specific* delta to `/work`, not "it broke, fix it"
- **Escalation bounds** — unproductive `/work` loops are detected and escalated, not retried forever
- **Environment-bounded blast radius** — local/dev is default; audit/stg payload runs are allowed only when the user explicitly names that target; prod is never allowed

**Violating the letter of this process is violating the spirit.** A "verification" without recorded request/response pairs is not a verification.

## The Iron Law

```
NO "PRODUCTION READY" CLAIM UNTIL EVERY ASSERTION PASSES AGAINST A LIVE TARGET SERVICE
NO LIVE VERIFICATION UNTIL A VERIFICATION PLAN IS PRESENTED TO THE USER AND APPROVED
NO API VERIFICATION WITHOUT A GENERATED CURL HARNESS, RECORDED `curl -i` OUTPUT, AND SAVED REQUEST FIXTURES
NO REACHABLE HTTP ENDPOINT MAY BE VERIFIED BY UNIT TESTS, BYTECODE, MAPPER SQL, OR CODE READING INSTEAD OF CURL
NO ACTUAL VERIFICATION WITHOUT TASK-BOUNDED VERIFIER SUBAGENTS AFTER PLAN APPROVAL
NO PAYLOAD RUN AGAINST prod FROM THIS SKILL
AUDIT/STG PAYLOAD RUNS REQUIRE EXPLICIT USER INSTRUCTION AND RECORDED TARGET ENVIRONMENT
NO SKILL EXIT UNTIL verify.md IS WRITTEN AND ITS STATUS IS  pass | fail-escalated | fail-user-required
```

## When to Use

Normal mode — the pipeline position:

```
/explore  →  explore.md
/work     →  work.md   (code + unit/integration tests + lint/build)
/verify   →  verify.md (HTTP-level E2E proof; re-triggers /work on failure)
```

Invoke `/verify` when **any** holds:

1. `/work` has just completed for `<YYYYMM>/<slug>` and the user wants proof before merging.
2. A paired `work.md` exists but the current session has no fresh HTTP-level evidence.
3. The user supplies an API path + at least one payload and asks for ad-hoc verification (standalone mode — see below).
4. A bug report arrived against an existing endpoint and you want to reproduce + gate the fix before touching code. (In this case `/verify` runs first, captures the failing curl, then hands off to `/explore` → `/work` → `/verify`.)

Skip only for:
- Pure frontend-only diffs with no backend surface touched (run `npm run dev` and use the browser; a separate skill owns frontend QA).
- Config-only changes that do not alter any HTTP contract.
- Doc/comment-only commits.

## Standalone Mode (no ticket folder)

If there is no `<artifact-root>/.backend/<YYYYMM>/<slug>/` folder and the user wants ad-hoc API verification:

1. Ask the user for: endpoint (method + path), target service, at least one sample payload (or explicit permission to pull one from DB).
2. Derive a slug from the endpoint (e.g. `verify-post-users-login`) and today's `<YYYYMM>`.
3. Choose the artifact root:
   - If the endpoint belongs to a single microservice/module, use that service directory. Example: `example-viewer-api/.backend/<YYYYMM>/<slug>/`.
   - If the endpoint spans multiple services or is explicitly repo-wide, use the repo root.
   - If unclear, infer from controller/service paths; ask only when multiple service roots are equally plausible.
4. Create `<artifact-root>/.backend/<YYYYMM>/<slug>/` and proceed from Phase 2 onward. `explore.md` and `work.md` will not exist — record that in Phase 0 instead of loading them.
5. On Phase 6 write only `verify.md` (no docs entry — that is `/work`'s job).

## Inputs & Outputs

### Inputs

| Source | What to extract |
|---|---|
| `<artifact-root>/.backend/<YYYYMM>/<slug>/explore.md` | §5B.contract (req/resp shape), §7 (which endpoints changed), §8 (cross-service contracts that must still hold) |
| `<artifact-root>/.backend/<YYYYMM>/<slug>/work.md` | §4 (files edited → derive touched endpoints), §7 (cross-service impact to re-check), §8 (follow-ups that might be relevant to probe) |
| The service's own `CLAUDE.md` | Run command, local port, profile, auth scheme |
| project root `CLAUDE.md` | "local service ports, profile overrides, SSO callback rules"; Feign/Redis/SSE patterns |
| User messages | Extra payloads, environment overrides, explicit accept/reject on re-work |
| Saved fixtures | `<artifact-root>/.backend/<YYYYMM>/<slug>/fixtures/*.json` from earlier `/verify` runs on this ticket |
| Live read-only MySQL | Payload shapes sampled from real rows — PII redacted (see `payload-sourcing.md`) |

### Outputs (all three are required for completion)

| Artifact | Path | Purpose |
|---|---|---|
| Curl harness | `<artifact-root>/.backend/<YYYYMM>/<slug>/harness/*.sh` | Reusable shell scripts — one file per endpoint, `BASE_URL`/`TOKEN` configurable |
| Fixture payloads | `<artifact-root>/.backend/<YYYYMM>/<slug>/fixtures/*.json` | Per-variant payload bodies, redacted of PII |
| Run log + report | `<artifact-root>/.backend/<YYYYMM>/<slug>/verify.md` + `<artifact-root>/.backend/<YYYYMM>/<slug>/runs/run-<N>.log` | Dense symbolic report + raw curl output per iteration |

**Folder layout (per ticket, post-verify):**

```
<artifact-root>/.backend/
  <YYYYMM>/
    <slug>/
      explore.md          ← from /explore
      work.md             ← from /work
      verify.md           ← this skill
      harness/
        <endpoint>.sh     ← one curl harness per endpoint (executable)
        _env.sh           ← BASE_URL / TOKEN / COMMON_HEADERS
      fixtures/
        <endpoint>.<variant>.json
        ...
      runs/
        run-1.log         ← full curl -i output for iteration 1
        run-2.log         ← (only exists if /work was re-invoked)
        run-3.log
```

See `verify-template.md` for the exact `verify.md` format and `curl-harness.md` for the shell script template.

## Execution Rules

**Language:** Match the user's language for user-facing responses, verification plans, final summaries, and docs notes. Keep code symbols, file paths, SQL, endpoints, commands, and exact error strings unchanged.

**Required:**
- In Phase 0, present and get approval for the verification plan before live work.
- Before approval, do not run curl, sample DB data, start services, generate/execute harnesses, or run payloads against shared envs.
- After approval, at least one verifier subagent performs real verification.
- Verifier subagents write only under the ticket folder: `harness/*.sh`, `fixtures/*.json`, `runs/run-<N>.log`.
- The main agent writes the approved plan, verifier work log, and result matrix to `<artifact-root>/.backend/<YYYYMM>/<slug>/verify.md`.
- If a `/work` docs entry exists, append only a `Verification (/verify)` section to that file. In standalone mode, do not create a new docs entry.

**Plan must include:**
`targets`, `env/base URL`, `payload sources`, `variants`, `assertions`, `artifact paths`, `safety bounds`, `subagent assignments`.

**Forbidden:**
- prod payload run.
- audit/stg run without explicit user instruction.
- Substituting unit tests or code reading for a reachable endpoint.
- Proceeding solo when subagents are unavailable unless the user explicitly waives the requirement.

**Verifier handoff:**
`scope`, `artifacts`, `commands`, `result matrix rows`, `failures/blockers`, `redaction notes`.

**Quality gates:**
- Done = approved plan recorded, fixtures/harness/run logs exist, `verify.md §6` matrix matches latest `runs/run-<N>.log`, status is valid.
- Scope = writes only under ticket `.backend/<YYYYMM>/<slug>/{harness,fixtures,runs,verify.md}` plus optional docs-note append.
- Evidence = HTTP claim cites `runs/run-<N>.log:line`; payload claim cites fixture/source; DB claim records read-only query shape + redaction note.
- Failure = `FAIL-APP` service behavior wrong, `FAIL-CONTRACT` explore/work contract wrong, `FAIL-ENV` target/auth/downstream unavailable, `FAIL-DATA` payload source invalid, `PASS-WITH-NOTES` non-blocking gap.
- Escalate = `FAIL-APP` creates bounded `/work` delta with user approval; `FAIL-CONTRACT` returns to `/explore`; `FAIL-ENV`/`FAIL-DATA` asks user for env/data.

**Final response:**
Report the absolute `verify.md` path, status, endpoint/variant pass-fail, main-agent work, verifier work, and blockers, then STOP.

## The Seven Phases

Complete each phase before the next. Do not interleave. The loop between Phase 4 and Phase 5 is the only legal loop; no ad-hoc retries.

### Phase 0 — Load & Validate the Contract

1. Determine artifact root and ticket folder — ask or infer `<YYYYMM>/<slug>`. Read `explore.md` and `work.md` **in full** if they exist.
   - If `work.md §4` or the user scope points to a single microservice/module, use that service directory as artifact root.
   - If the change spans multiple services or is explicitly repo-wide, use the repo root.
2. Announce the kind (Bug / Feature / Refactor), the scope one-liner from `explore.md §2`, and whether `/work` reported `status: done | partial | blocked` in `work.md §10` re-entry header.
3. From `work.md §4 Changes` and `explore.md §7 Proposed Changes`, derive the **endpoint target list** — the minimal set of HTTP surfaces that must be probed:
   - Controllers / `@RequestMapping` that were edited or added.
   - Feign client methods that were added/extended (these imply an endpoint on the callee service that now has a new contract).
   - Endpoints exercised by any test that was added/edited in `work.md §5`.
4. For each target, record: service, method, path, auth required?, content-type, sample response envelope citation (`file:line`).
5. Validate:
   - At least one endpoint target was derivable. If zero targets, stop — the change had no HTTP surface and `/verify` is the wrong skill.
   - Each target's service exposes a reachable local/dev profile, or the user explicitly asked to verify an audit/stg target. Confirm ports/profile from the service's `CLAUDE.md`, profile config, process command, or target URL.
   - `work.md §10` status is not `blocked`. If blocked, stop and tell the user `/work` must complete before `/verify` runs.
6. Confirm the target environment:
   - Default: `local`/`dev` service on `localhost`, `127.0.0.1`, `0.0.0.0`, or `host.docker.internal`.
   - Allowed by explicit user instruction: `audit` or `stg`, including a locally bound instance running that profile.
   - Never allowed: `prod` or any hostname/base URL that resolves to production.
7. Record the target env, `BASE_URL`, and evidence that the user asked for audit/stg if not local/dev. If the target is ambiguous, stop and ask.

**Output of this phase:** a verification plan checkpoint, printed in chat, with:

- target env and proof it is allowed
- endpoint target list
- payload/variant plan
- harness/run-log artifact plan
- assertion plan
- subagent assignment plan
- explicit "waiting for approval before live verification"

Then **STOP** until the user approves the plan. Do not source DB payloads, start services, generate harness files, run curl, or dispatch verifier subagents before approval.

### Phase 1 — Payload Sourcing

Follow `payload-sourcing.md` in this directory. Priority order, top wins:

1. **User-provided payloads** — explicitly pasted in the conversation or attached to the ticket. Save each under `fixtures/<endpoint>.user-<label>.json` verbatim (redact obvious PII before writing).
2. **Saved fixtures** — any `<artifact-root>/.backend/<YYYYMM>/<slug>/fixtures/*.json` from a prior `/verify` run on this ticket. Reuse when the endpoint contract has not changed since (compare `explore.md §5B contract` with the fixture shape).
3. **Live MySQL sampling** — only if the prior two are insufficient. Read-only, dev-only, `LIMIT`-bounded. Shapes are copied; PII is redacted to placeholders (`__REDACTED_EMAIL__`, `__REDACTED_PHONE__`). This procedure lives in `payload-sourcing.md`.
4. **Synthesized payloads** — last resort. Only for boundary/negative variants where no real row would look that way (e.g. empty array, max-length string, invalid enum value). Every synthesized field must be justified in `verify.md §4`.

For each endpoint target, produce at minimum **three variants**:

| Variant | Purpose | Example |
|---|---|---|
| `happy` | Representative valid request | Real row shape, expected 2xx |
| `boundary` | Edge of what schema allows | Max-length string, min/max numeric, empty-but-valid collection |
| `negative` | Request that must be rejected | Missing required field, wrong enum, unauth if endpoint requires auth |

Bug fixes get one additional mandatory variant:

| Variant | Purpose |
|---|---|
| `regression` | The exact payload that triggered the original bug per `explore.md §5A.repro` |

Write each variant to `fixtures/<endpoint>.<variant>.json`. For GET/DELETE requests without a JSON body, the fixture still records the full request shape: method, path, query parameters, path variables, headers that affect behavior, and expected assertions. Never commit raw PII — apply redaction before write.

This phase is performed by verifier subagents after plan approval. Each subagent records fixture paths, source category, redaction decisions, and any sample-query shape it used.

### Phase 2 — Harness Generation

Follow `curl-harness.md` in this directory. For each endpoint target, generate `harness/<endpoint>.sh` with this shape:

- Loads `_env.sh` which exports `BASE_URL`, `TOKEN` (if needed), and `COMMON_HEADERS`.
- One `run_variant()` invocation per variant fixture.
- `curl -i --max-time 10 -s -S -o >(tee -a ../runs/run-<N>.log)` — always includes `-i` to capture status + headers, always bounded by timeout.
- Captures: status code, response body, elapsed time. Does **not** follow redirects unless the endpoint documents redirect behavior.
- Exits non-zero if any variant failed its declared assertion.

Declare assertions per variant inline in the shell script:

```bash
assert_status 200
assert_json_path '.data.userId' '<expected>'
assert_json_key_present '.data.createdAt'
assert_header 'Content-Type' 'application/json'
```

Assertion helpers are defined in `harness/_assert.sh` (generated once per ticket). Keep them pure bash + `jq` + `grep`. No Python, no Node — the harness must run in any local dev environment with a shell and `jq` installed.

**Auth strategy per service family:**

- Services behind an auth filter → obtain a dev credential from the project's auth authority in the dev profile, such as `GET /test/generate-jwt`, for local/dev runs. For shared environments, use only the user-provided or already available environment token/session for that target; never mint or paste a prod token. The same token is reusable across other services when they share the same issuer or signing key. **Full reference: `jwt-auth-reference.md` in this directory** — contains stack-neutral auth schemes plus ready-to-adapt `_env.sh` and `_preflight.sh` snippets.
- Feign-only internal endpoints (no external auth): still send the standard `X-Request-Id` header.
- Services with no auth in the target profile: skip the token step but assert the profile/target really matches the recorded `TARGET_ENV` when exposed (e.g. `actuator/env`/`actuator/info`, process command, or service banner).
- Services with a different auth scheme must use their own section of `jwt-auth-reference.md`; do not assume JWT applies everywhere.

Cache the token in `_env.sh` for the run; never paste it into fixture files. Tokens have `exp` — re-run `_preflight.sh` on every iteration (see `curl-harness.md §Re-Run Protocol`).

**Do not generate harness files for endpoints you cannot reach in the selected target.** If the target service cannot start or is unreachable (missing downstream, auth unavailable, wrong profile), stop, record it in `verify.md §9` as `[BLOCK]`, and ask the user.

This phase is performed by verifier subagents after plan approval. Each subagent owns only the harness files assigned in the approved plan.

### Phase 3 — Execution

1. Start the target service(s) if not already running:
   - `cd <service> && ./gradlew bootRun --args='--spring.profiles.active=dev'` (in background or a separate terminal the user is already running), unless the user explicitly directed an already-running audit/stg instance.
   - Poll a health endpoint (`/actuator/health` or the service's documented readiness probe) until `200 OK` or until a 60s cap. Do not start timing tests before ready.
2. Execute each `harness/<endpoint>.sh`. Each run appends to `runs/run-<N>.log` where `<N>` is the current iteration (starts at 1; increments each time Phase 5 re-invokes `/work`).
3. **Capture every request/response pair verbatim.** `curl -i` headers + full body. Redact the `Authorization:` header value in the log before committing.
   - If the endpoint is reachable, keep running the curl variants until happy, boundary, and negative payloads have all produced recorded HTTP results or a concrete app/contract failure is found.
   - Unit tests, bytecode inspection, mapper SQL checks, and code reading are supplemental evidence only. They may explain or narrow a failure, but they do not satisfy `/verify` for any reachable HTTP endpoint.
   - If auth/session/downstream state prevents a real happy-path curl, classify it as `FAIL-ENV` or `fail-user-required`; do not mark the endpoint as passing based on non-HTTP evidence.
4. Compute the variant result matrix:

```
endpoint          variant     status  latency(ms)  assertions  result
POST /users/login happy       200     142          3/3         ✓
POST /users/login boundary    400     89           2/2         ✓  (expected 400)
POST /users/login negative    401     91           2/2         ✓
POST /users/login regression  500     154          1/3         ✗
```

5. **Never paraphrase the result matrix.** It is copied verbatim into `verify.md §5`.

If any service fails to start or a probe times out, record the actual error and stop — do not retry silently.

This phase is performed by verifier subagents after plan approval. Each subagent must return the exact command(s), run-log paths, matrix rows, and pass/fail classification for its assigned endpoint or cross-service probe. The main agent reviews and synthesizes; it does not silently replace missing subagent evidence with a passing claim.

### Phase 4 — Triage

Classify the matrix:

| Classification | Condition | Next |
|---|---|---|
| **PASS** | All variants `✓`; cross-service assertions in §6 pass; latency within bounds | Phase 6 wrap-up, `status: pass` |
| **FAIL-APP** | ≥1 variant `✗` with 4xx/5xx or wrong body shape attributable to the service under test | Phase 5 iteration (re-work) |
| **FAIL-CONTRACT** | Request shape rejected but `explore.md §5B.contract` said it should be accepted, OR response shape does not match `§5B.contract.resp` | Phase 5 — contract drift; likely `/explore` re-run |
| **FAIL-ENV** | Port conflict, missing downstream, DB not reachable — not a code issue | Stop. Record in §9 `[BLOCK]`. Ask user. Do not re-invoke `/work`. |
| **PASS-WITH-NOTES** | Happy + negative pass, one boundary fails in a way the contract did not pin | `verify.md §9` `[GAP]`; ask user whether to escalate |

For **FAIL-APP** and **FAIL-CONTRACT**, prepare a **rework delta** — a structured handoff to `/work`:

```
endpoint:       <method> <path>
variant:        <variant-name>
request:        <fixture path>
expected:       <status>, body path <path> == <value>, header <name>=<value>
observed:       <status>, body <snippet>, header <name>=<value>
trace-hint:     <service log excerpt or absence thereof>
likely-source:  <file:line from work.md §4 that owns this behavior, if derivable>
```

This delta is the input to Phase 5. It is also saved into `verify.md §7` for the run log.

### Phase 5 — Iteration (bounded)

**Iteration cap: 3 /work re-invocations per /verify session.**

For each fail classification:

1. Present the rework delta to the user in chat. Ask: "Trigger `/work` with this delta, or stop here?"
2. On **user accept**: invoke `/work` with explicit scope limited to the delta. Pass the delta as the opening message and point `/work` at the same `<YYYYMM>/<slug>` folder. `/work` appends a new entry to its `work.md §2 Contract Adherence` ("rows added mid-flight via /verify rework delta #N"). Do not let `/work` widen scope beyond the delta.
3. On **user reject / hold**: stop. Write `verify.md` with `status: fail-user-required` and the delta as the question.
4. After `/work` returns, **re-run Phase 3 (execution)** with the same harness — do not regenerate fixtures unless the contract itself changed. Append to `runs/run-<N+1>.log`.
5. Compare iteration N+1 to iteration N. **Meaningful progress** = at least one of:
   - A previously failing variant now passes.
   - The failing variant moved to a *different, more specific* failure (e.g. 500 → 400 with a validation message — indicates the service is now routing the request correctly but rejecting its content).
   - The rework delta's `likely-source` line was edited and the corresponding assertion's observed value changed (even if still wrong).

   If none of these hold, the iteration **did not make meaningful progress**.

6. Escalation ladder:

```
iteration 1 fails           → generate new rework delta, ask user, go to Phase 5 step 2
iteration 2 fails AND no progress vs iter 1
                            → STOP the /work loop.
                              Ask user: (a) re-run /explore (contract may be wrong),
                                        (b) accept verified scope + defer, or
                                        (c) provide new evidence / payloads.
iteration 2 fails WITH progress
                            → one more /work attempt allowed (iteration 3).
iteration 3 fails (any)     → STOP. status: fail-escalated.
                              Write verify.md §9 [BLOCK] with the specific decision the user must make.
                              Do NOT re-invoke /work.
```

7. If at any iteration the failure classification flips to **FAIL-CONTRACT**, skip the remaining `/work` budget and escalate to `/explore` immediately — the contract document is the root cause, not the code.

### Phase 6 — Wrap Up (verify.md)

Write `<artifact-root>/.backend/<YYYYMM>/<slug>/verify.md` using `verify-template.md` in this directory. Required sections:

1. §1 One-paragraph recap (prose — the only prose section, plain language primary).
2. §2 Meta (kind/slug/yyyymm/status/iteration-count).
3. §3 Verification Plan & Agent Work Log (approved plan + verifier subagent handoffs).
4. §4 Endpoint Target List (matches Phase 0 output).
5. §5 Payload Sources (origin of each fixture — user / saved / DB-sampled / synthesized, with redaction notes).
6. §6 Results Matrix (verbatim from the last iteration's execution).
7. §7 Cross-Service Checks (Feign consumers, SSE channels, Kafka payloads — what was probed and the result).
8. §8 Iteration Trail (one row per `/work` re-invocation with the delta and the delta's effect).
9. §9 Performance Observations (p95 latency per endpoint, if meaningfully measured).
10. §10 Open Items (`[BLOCK]` / `[INFO]` / `[GAP]`).
11. §11 Re-entry Header (how a future session reloads context).

Also:

- Each `harness/<endpoint>.sh` and each `fixtures/*.json` remains in the ticket folder — they are reusable evidence.
- `runs/run-<N>.log` files stay. They are the audit trail. Do not prune between iterations.
- Append a "Verification (/verify)" section to the existing `<artifact-root>/docs/features/<YYYY-MM-DD>-<slug>.md` or `<artifact-root>/docs/bugs/...md` change note written by `/work`, summarizing the verification in 3-5 lines of plain language. If the docs entry does not exist (standalone mode), **do not** create one.

After writing, print:
- Absolute path to `verify.md`
- Final status: `pass` | `fail-escalated` | `fail-user-required`
- A 3-5 line human summary of: endpoints probed, variants run, pass/fail counts, what (if anything) blocks merging.
- A clear explanation of what the main agent did and what each verifier subagent did.

Then **STOP**. Do not commit, push, or open a PR. `/verify`, like `/work`, ends at the filesystem.

## Scope Boundaries

| In scope for `/verify` | Out of scope |
|---|---|
| HTTP-level probing of endpoints touched in `work.md §4` | Load testing, chaos testing, long-running soak tests |
| Deriving payloads from read-only DB access for the selected non-prod target | Running any write query against dev/audit/stg/prod DB |
| Re-invoking `/work` with a bounded delta after user consent | Re-invoking `/work` with "just fix it" and no delta |
| Re-invoking `/explore` on **FAIL-CONTRACT** after user consent | Silently rewriting `explore.md` or `work.md` |
| Writing `verify.md` and per-iteration run logs | Modifying source code. `/verify` never edits `src/` or `test/` |
| Redacting PII before saving fixtures | Pasting real user data into any file under `.backend/` |
| Appending a "Verification" section to today's docs entry | Creating a brand-new docs entry (that is `/work`'s job) |
| Frontend E2E via browser automation (if a companion skill exists) | Frontend QA without a companion skill — out of scope, say so and stop |
| Dispatching verifier subagents after plan approval | Running actual verification solo without user waiver |

## Red Flags — STOP and Restart Phase

If you catch yourself thinking:

- "I'll skip the happy-path assertion and just check the status code."
- "The response looks right, I don't need to diff it against `§5B.contract`."
- "I'll mock the downstream service — it's faster than starting it."
- "`BASE_URL=https://stg-...` is fine even though the user did not explicitly ask for stg." **(absolute stop)**
- "`BASE_URL=https://prod-...` is fine for one quick check." **(absolute stop)**
- "The DB row has the user's real email; I'll commit the fixture as-is."
- "Iteration 2 failed but I feel iteration 3 will fix it — let me just try once more past the cap."
- "The user already accepted iteration 1, I'll keep re-invoking `/work` without checking in."
- "`/work` finished. The build passed, so I'll skip the curl and just write `verify.md`."
- "One assertion failed but it's a flaky test — I'll mark it pass."
- "The Feign cross-service check failed but my endpoint passed — that's not my problem."
- "The endpoint list is obvious, so I'll skip the plan and start curl now."
- "The plan was approved, so I can do the curl myself without verifier subagents."
- "The verifier said it passed, so I don't need to inspect the run log or matrix."
- "The final answer can just say pass without explaining what each verifier did."

**All of these mean: STOP. Return to the phase that enforces the missing discipline.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Unit tests passed, curl is overkill" | Unit tests do not exercise the HTTP filter chain, deserialization, auth, or the response envelope wrapper. Curl does. |
| "The endpoint is internal, no one calls it externally" | Internal-only Feign endpoints still have a contract. Breaking it crashes the caller service. |
| "DB sampling is slow; I'll invent the payload" | Invented payloads have invented shapes. Real rows have nulls, leading zeros, Korean text, and 6-year-old legacy enum values. Sampling finds these. |
| "The user said 'verify it works' — happy path is enough" | "Works" in legacy code means "doesn't crash on any input type that exists in the DB." One variant is not verification. |
| "I'll re-invoke `/work` without asking; we already discussed it" | Every re-invocation changes code. Every code change needs explicit consent, not assumed consent. |
| "Iteration cap is arbitrary; I can keep going" | The cap exists because unproductive loops burn trust and signal a contract problem. Past the cap, escalate — that is the correct action, not the failure mode. |
| "Performance numbers vary; I won't record them" | You do not need to pass a SLA. You need to record the number so regressions are detectable. Absence of a baseline is itself a regression. |
| "A verification plan slows things down" | The plan is the safety gate: it exposes target env, payload shape, mutation risk, and assertions before live calls run. |
| "Subagents are optional for verification" | Verifier subagents create an auditable execution trace: who ran which endpoint, what files/logs they produced, and what evidence the main agent accepted. |

## Integration with Other Skills

**Sibling skills in this pipeline:**

- **`work`** (typical predecessor) — produces `work.md` whose §4 / §5 / §7 define the endpoint target list and cross-service checks. `/verify` reads but never edits `work.md`.
- **`explore`** (rerun target on FAIL-CONTRACT) — if the contract is wrong, escalate here rather than patching in code.

**Support files in this directory (always used):**

- **`payload-sourcing.md`** — Phase 1 payload sourcing procedure with PII redaction
- **`curl-harness.md`** — Phase 2 harness template (`_env.sh`, `_assert.sh`, per-endpoint scripts) + Re-Run Protocol
- **`jwt-auth-reference.md`** — auth acquisition procedure for Phase 2 (currently written for one specific codebase's JWT scheme — adapt to your stack's auth flow)
- **`verify-template.md`** — Phase 6 `verify.md` output format

**Optional companion skills (if also installed — e.g. superpowers):**

This skill is self-contained — the plan gate, fresh-output rule, and verifier-subagent dispatch are inlined. The following are optional enhancers if present in your agent's skill registry:

- `verification-before-completion` — alternative formulation of the "fresh output only" rule; applies at unit/integration layers where `/verify` does not operate
- `systematic-debugging` — use when a FAIL-APP triage cannot identify a plausible `likely-source`; do not hand `/work` a rework delta without a call-chain hypothesis
- `test-driven-development` — not invoked directly by `/verify`, but any regression variant added here becomes a candidate test for the next `/work` pass
- `dispatching-parallel-agents` — alternative formulation of the parallel-endpoint dispatch technique for Phase 3 with 3+ independent endpoints
- `finishing-a-development-branch` — use AFTER `/verify` exits with `status: pass`, for commit / PR ceremony
- `jira-update` / `jira-ascli` — use AFTER `/verify` exits, for ticket status updates with verification evidence

## Quick Reference

| Phase | Input | Output | Fail state → |
|---|---|---|---|
| 0. Load Contract | `explore.md` + `work.md` (or user spec) | Verification plan + subagent assignment plan; wait for approval | No endpoints / blocked work → stop, ask user |
| 1. Payload Sourcing | Approved plan + DB read-only access (optional) | Verifier-created fixtures per variant, PII-redacted | Can't source real shape → ask user for samples |
| 2. Harness Generation | Fixtures + service CLAUDE.md | Verifier-created `harness/*.sh` + `_env.sh` + `_assert.sh` | Service unreachable locally → `[BLOCK]`, ask user |
| 3. Execution | Harness + running local service | Verifier-created `runs/run-<N>.log` + results matrix | Service won't start → stop, record actual error |
| 4. Triage | Results matrix | Pass \| FailApp \| FailContract \| FailEnv \| PassWithNotes | FailEnv → stop, not a code issue |
| 5. Iteration | Rework delta | Re-run `/work` (up to 3x) or escalate | No progress 2x → escalate to `/explore` or user |
| 6. Wrap Up | All above | `verify.md` + verifier work log + append to docs note | Missing file → not done |

## Citations & Redaction Discipline

- Every claim about the service's behavior must cite the exact `runs/run-<N>.log:line` range.
- Every claim about the contract must cite `explore.md §5B` or the controller/DTO `file:line`.
- PII categories to redact before writing any file under `.backend/`: email addresses (replace with `user+<id>@example.test`), phone numbers, national ID, real student names (replace with `student<id>`), real school names (replace with `school<id>`).
- `Authorization:` header values in run logs → replace with `Bearer __REDACTED__`.
- Cookies / session IDs → replace value with `__REDACTED__`.
- If a field you cannot classify appears in a sampled row, redact it to `__UNKNOWN__` and note the field in `verify.md §4` — do not commit it.

## The Bottom Line

`/work` proves the code compiles, the unit tests pass, and the diff matches the plan.
`/verify` proves the **service**, as assembled, accepts production-shaped payloads, produces documented responses, preserves cross-service contracts, and does so with fresh, recorded, replayable evidence.

Until `verify.md` says `status: pass`, the change is not production-ready — regardless of how green the unit tests are.

If the contract is wrong, `/verify` escalates to `/explore`. If the code is wrong, `/verify` hands a precise delta to `/work` and retries — up to the cap, then escalates. `/verify` never invents a passing result and never silently loops.
