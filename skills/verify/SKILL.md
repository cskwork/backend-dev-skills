---
name: verify
description: Use when code changes have been made to backend HTTP endpoints and HTTP-level proof against a running service is needed — typically after /work completes, or standalone when you have an endpoint plus at least one payload to probe ad-hoc
---

# Verify

## Overview

`/verify` is the final gate before a change ships. Unit tests prove the code compiles and returns what authors expected. **They do not prove the running service accepts real payloads, returns the documented envelope, preserves cross-service contracts, or survives production traffic shapes.** `/verify` does.

**Core principle:** No completion claim is trustworthy until a curl against a running service produces the right body, status, and side effects — with fresh, recorded output.

**Design goals:**
- Evidence, not assertion — every "pass" backed by recorded `curl -i` output
- Payload realism — fixtures from real row shapes or user-provided cases, not invented JSON
- Variant coverage — happy + ≥1 boundary + ≥1 negative per endpoint
- Iteration discipline — on failure, pass a *specific* delta to `/work`, not "it broke"
- Localhost-default blast radius — remote environments (dev/stg/audit/prod) require explicit, recorded user consent

## The Iron Law

```
NO "PRODUCTION READY" CLAIM UNTIL EVERY ASSERTION PASSES AGAINST A LIVE SERVICE
DEFAULT TARGET IS LOCALHOST — REMOTE ENV REQUIRES CONSENT RECORDED IN verify.md §0
NO PROD MUTATION WITHOUT PER-REQUEST CONFIRMATION + REVIEWED PAYLOAD
NO EXIT UNTIL verify.md STATUS IS  pass | fail-escalated | fail-user-required
```

## Target Environment Policy

`/verify` defaults to localhost. Remote environments are *opt-in* and must be requested by the user explicitly (URL or env name in their message). Once consent is granted, record it in `verify.md §0` before generating any harness.

| Env | Read (GET/HEAD) | Mutating (POST/PUT/PATCH/DELETE) | Token storage |
|---|---|---|---|
| localhost | default-allowed | default-allowed | `_env.sh` (cached) |
| dev (deployed) | per-session consent | per-session consent | env var only, never written to disk |
| stg / audit | per-session consent | per-session consent **+ reaffirm before run** | env var only, never written to disk |
| prod | **per-request** consent | **per-request** explicit confirmation **+ payload review** | env var only, never written to disk |

**Consent record format** (`verify.md §0` — required for any non-localhost target):
```
target-env:    stg
base-url:      https://stg-api.example.com
consent-by:    user (verbatim quote: "yes verify against stg")
consent-scope: GET /api/x, GET /api/y    # exact methods+paths covered
consent-ttl:   per-session                # or per-request for prod / mutations
```

**Remote-env guard rails (always apply):**
- **PII redaction at capture time** — raw bodies with real PII never written to `runs/*.log` or `fixtures/*.json`. Save redacted form + summary metrics; if a checksum is needed for replay, hash the raw body, do not store it.
- **Tokens** for remote envs come from environment variables only. Never persist to `_env.sh`. Re-export per session.
- **DB writes** against remote envs remain banned by default; require per-statement consent.
- **Iteration cap** stays at 3, but each iteration on stg/audit/prod must re-confirm scope hasn't widened.
- On **prod**: re-prompt user before EACH variant run, even within a session. Read-only assertions only unless the user explicitly green-lit a write.

## When to Use

Pipeline: `/explore → explore.md` → `/work → work.md` → `/verify → verify.md`

Invoke when **any** holds:
1. `/work` completed for `<YYYYMM>/<slug>` and user wants proof before merging.
2. A paired `work.md` exists but no fresh HTTP-level evidence this session.
3. User supplies endpoint + payload for ad-hoc verification (standalone mode).
4. Bug report against an existing endpoint — reproduce first, then hand off to `/explore → /work → /verify`.

Skip for frontend-only diffs, config-only changes that don't alter HTTP contracts, doc/comment commits.

## Standalone Mode (no ticket folder)

1. Ask user for: endpoint (method + path), target service, ≥1 sample payload (or permission to pull from DB).
2. Derive slug (e.g. `verify-post-users-login`) + today's `<YYYYMM>`.
3. Create `.backend/<YYYYMM>/<slug>/` and start from Phase 2. Record "no explore.md/work.md" in Phase 0.
4. Phase 6 writes only `verify.md` — no docs entry (that's `/work`'s job).

## Inputs & Outputs

### Inputs

| Source | What to extract |
|---|---|
| `explore.md` | §5B.contract (req/resp shape), §7 (changed endpoints), §8 (cross-service contracts) |
| `work.md` | §4 (edited files → touched endpoints), §7 (cross-service impact), §8 (follow-ups) |
| Service `CLAUDE.md` | Run command, local port, profile, auth scheme |
| User messages | Payloads, env overrides, rework consent |
| Saved fixtures | `fixtures/*.json` from prior runs on this ticket |
| Live read-only MySQL | Payload shapes from real rows — PII redacted (see `payload-sourcing.md`) |

### Outputs (all three required)

| Artifact | Path | Purpose |
|---|---|---|
| Curl harness | `harness/*.sh` | Reusable shell scripts — one per endpoint, `BASE_URL`/`TOKEN` configurable |
| Fixtures | `fixtures/*.json` | Per-variant bodies, PII-redacted |
| Run log + report | `verify.md` + `runs/run-<N>.log` | Dense symbolic report + raw curl output per iteration |

**Folder layout:**
```
.backend/<YYYYMM>/<slug>/
  explore.md, work.md, verify.md
  harness/<endpoint>.sh, _env.sh
  fixtures/<endpoint>.<variant>.json
  runs/run-<N>.log
```

See `verify-template.md` for `verify.md` format, `curl-harness.md` for shell script template.

## Fast Path — Bug Reproduction First

**If invoked to reproduce a reported bug** (user pasted a failing request/response, or intent = "버그 재현"), preempt standard phase order:

1. Before Phase 1 sourcing completes, assemble a minimal curl for the reported case. Save as `harness/repro.sh` — single-variant with user-reported params verbatim (PII redacted in storage; running request uses real values).
2. Tee `curl -i` output to `runs/run-1.log` **before** deriving other variants.
3. Classify immediately:
   - **Reproduced** → expand to Phase 2 with confidence; regression variant is grounded.
   - **Not reproduced locally** → try unit-test repro (mapper/service isolated) before blaming env; record in `verify.md §8`. If nothing, escalate — do not fabricate a repro.
4. Only after repro signal (positive or negative) expand to full 3-variant matrix.

**Rationale:** with a bug report, fastest signal is "can we see it here?" — every downstream judgment is gated on that answer. Fast Path applies **in addition to** Phase 0 safety checks — for non-local targets, the consent gate (see *Target Environment Policy*) runs before the repro curl is even assembled.

## DB Hypothesis — Execute, Don't Just Propose

When a bug hypothesis depends on **what's in the database** (e.g. "row X missing", "duplicates cause wrong record picked"), don't stop at proposing SQL.

1. If a MySQL tool is available (`mysql-intelligence`, read-only client), **run the query immediately** against dev profile.
2. Record observed result beside SQL in `verify.md §8`. Hypothesis without observation = assertion.
3. Chain queries — if first rules out a hypothesis, run next in the same turn.
4. Redact PII before writing row values (see §Citations & Redaction).
5. Rules:
   - **Read-only by default.** Writes require per-statement user authorization.
   - **Dev/local only without confirmation.** stg/audit/prod needs the same recorded consent as HTTP probes (see *Target Environment Policy*); on prod, each statement re-confirms.
   - Connect/permission failures → `§9 [BLOCK]`, escalate; don't fall back to "please run this yourself".

Rationale: an unexecuted SQL proposal adds a user round-trip. When the tool exists, using it compresses multi-turn debug into single-turn.

## The Seven Phases

Complete each before the next. Do not interleave. Phase 4↔5 loop is the only legal loop.

### Phase 0 — Load & Validate the Contract

1. Determine ticket folder — read `explore.md` and `work.md` **in full** if they exist.
2. Announce kind (Bug/Feature/Refactor), scope one-liner, and `/work` status (`done|partial|blocked`).
3. Derive **endpoint target list** from `work.md §4` and `explore.md §7`:
   - Controllers / `@RequestMapping` edited or added
   - Feign client methods added/extended (implies new callee contract)
   - Endpoints exercised by added/edited tests in `work.md §5`
4. For each target record: service, method, path, auth?, content-type, response envelope citation (`file:line`).
5. Validate: ≥1 target derivable; each service has a reachable port (local or remote per consent); `work.md §10` is not `blocked`.
6. **Environment gate:**
   - If `BASE_URL` resolves to `localhost` / `127.0.0.1` / `0.0.0.0` / `host.docker.internal` → proceed.
   - If `BASE_URL` points to a remote env (dev/stg/audit/prod) → check for explicit user consent in this session (URL or env name in user message). If no consent, **stop and ask** before generating any harness. If consent exists, **write a Consent Record into `verify.md §0`** in the format defined under *Target Environment Policy*. Re-read the policy table for verb/env-specific gates (especially prod).
   - For prod: never auto-proceed even with prior consent — re-prompt before each variant run.

**Output:** TaskCreate list (one per target) + plan restatement + Consent Record draft if remote.

### Phase 1 — Payload Sourcing

Follow `payload-sourcing.md`. Priority (top wins):

1. **User-provided** — save under `fixtures/<endpoint>.user-<label>.json` verbatim (PII redacted).
2. **Saved fixtures** — reuse when contract unchanged since last run.
3. **Live MySQL sampling** — read-only, dev-only, `LIMIT`-bounded. Shapes copied, PII redacted to placeholders.
4. **Synthesized** — last resort for boundary/negative variants. Each synthesized field justified in `verify.md §4`.

Produce **three variants per endpoint**:

| Variant | Purpose | Example |
|---|---|---|
| `happy` | Representative valid | Real row, expected 2xx |
| `boundary` | Edge of schema | Max-length, min/max, empty-but-valid collection |
| `negative` | Must be rejected | Missing required, wrong enum, unauth |

Bug fixes add: `regression` — exact payload from `explore.md §5A.repro`.

Write to `fixtures/<endpoint>.<variant>.json`. Never commit raw PII.

### Phase 2 — Harness Generation

Follow `curl-harness.md`. Generate `harness/<endpoint>.sh`:

- Loads `_env.sh` (`BASE_URL`, `TOKEN`, `COMMON_HEADERS`)
- One `run_variant()` per fixture
- `curl -i --max-time 10 -s -S -o >(tee -a ../runs/run-<N>.log)` — always `-i`, always bounded
- Captures: status, body, elapsed. No redirect-follow unless documented.
- Exits non-zero if any variant fails.

Inline assertions per variant:
```bash
assert_status 200
assert_json_path '.data.userId' '<expected>'
assert_json_key_present '.data.createdAt'
assert_header 'Content-Type' 'application/json'
```

Helpers in `harness/_assert.sh` — pure bash + `jq` + `grep`. No Python/Node.

**Auth strategy:** pick whichever scheme matches the service under test and adapt as per `jwt-auth-reference.md` (fork it to your stack):

- JWT via dev-profile test endpoint (preferred when available): mint once, reuse across services signed with the same key.
- OAuth2 client_credentials: mint via token endpoint using committed dev credentials.
- Session cookie: POST to dev login endpoint, save cookie jar, reuse.
- API key header: load from env var; dev-only keys may be committed.
- Internal service-to-service (no end-user auth): send the usual correlation headers (`X-Request-Id` etc).
- No-auth dev profile: assert `SPRING_PROFILES_ACTIVE=dev` (or equivalent) before skipping the token step.

Cache **localhost** tokens in `_env.sh`. **Remote-env tokens** (dev/stg/audit/prod): pass via env var only (`TOKEN=$(...) ./harness/x.sh`); never write to `_env.sh` or commit. Re-run `_preflight.sh` each iteration (tokens expire).

**Don't generate harness for unreachable endpoints.** If service can't start in dev, record `[BLOCK]` in `§9` and ask user.

### Phase 3 — Execution

1. Start target service(s) with the dev profile (commands match your stack):
   - Spring Boot: `./gradlew bootRun --args='--spring.profiles.active=dev'`
   - Node.js: `npm run dev` / `npm start` / `node --env-file=.env.dev server.js`
   - Python: `uvicorn main:app --reload` / `python manage.py runserver`
   - Poll the health endpoint until `200` or 60s cap. Don't time tests before ready.
2. Execute each `harness/<endpoint>.sh`. Append to `runs/run-<N>.log` (N = iteration, starts 1).
3. **Capture every request/response verbatim.** `curl -i` headers + full body. Redact `Authorization:` in logs.
4. Compute variant matrix:
```
endpoint          variant     status  latency(ms)  assertions  result
POST /auth/login  happy       200     142          3/3         ✓
POST /auth/login  regression  500     154          1/3         ✗
```
5. **Never paraphrase the matrix.** Copy verbatim into `verify.md §5`.

On service start failure / probe timeout: record actual error and stop — no silent retry.

### Phase 4 — Triage

| Classification | Condition | Next |
|---|---|---|
| **PASS** | All `✓`, cross-service §6 passes, latency OK | Phase 6, `status: pass` |
| **FAIL-APP** | ≥1 `✗` with wrong status/body attributable to service under test | Phase 5 (re-work) |
| **FAIL-CONTRACT** | Shape rejected but `§5B.contract` said accept, OR resp mismatches `§5B.contract.resp` | Phase 5 — escalate to `/explore` |
| **FAIL-ENV** | Port conflict, missing downstream, DB unreachable | Stop. `§9 [BLOCK]`. Ask user. Don't re-invoke `/work`. |
| **PASS-WITH-NOTES** | Happy + negative pass, one boundary fails in unpinned way | `§9 [GAP]`; ask user |

For FAIL-APP / FAIL-CONTRACT, prepare **rework delta** (input to Phase 5 + saved to `§7`):
```
endpoint:       <method> <path>
variant:        <variant-name>
request:        <fixture path>
expected:       <status>, body path == <value>
observed:       <status>, body <snippet>
trace-hint:     <log excerpt or absence>
likely-source:  <file:line from work.md §4>
```

### Phase 5 — Iteration (bounded)

**Cap: 3 `/work` re-invocations per `/verify` session.**

1. Present rework delta to user. Ask: "Trigger `/work` with this delta, or stop?"
2. **Accept**: invoke `/work` with scope = delta. Opens `work.md §2 Contract Adherence` with "rework delta #N via /verify". Don't let `/work` widen.
3. **Reject/hold**: write `status: fail-user-required` with delta as the open question. Stop.
4. After `/work` returns, **re-run Phase 3** with same harness (don't regen fixtures unless contract changed). Append `run-<N+1>.log`.
5. Compare iteration N+1 to N. **Meaningful progress** = at least one:
   - Previously failing variant now passes
   - Failure moved to different, more specific one (e.g. 500 → 400 with validation message)
   - `likely-source` was edited and observed value changed (even if still wrong)

6. Escalation ladder:
```
iter 1 fails                    → new delta, ask user, loop
iter 2 fails + no progress       → STOP loop. Ask user: (a) re-run /explore,
                                   (b) accept verified scope + defer, or
                                   (c) provide new evidence.
iter 2 fails WITH progress       → one more attempt allowed (iter 3).
iter 3 fails (any)               → STOP. status: fail-escalated.
                                   §9 [BLOCK] with the specific user decision needed.
                                   Do NOT re-invoke /work.
```

7. If classification flips to **FAIL-CONTRACT** mid-loop, skip remaining `/work` budget and escalate to `/explore` — contract is root cause.

### Phase 6 — Wrap Up

Write `verify.md` using `verify-template.md`. Required sections:

1. One-paragraph recap (Korean prose, only prose section)
2. Meta (kind/slug/yyyymm/status/iteration-count)
3. Endpoint Target List (from Phase 0)
4. Payload Sources (origin per fixture, redaction notes)
5. Results Matrix (verbatim from last iteration)
6. Cross-Service Checks (Feign/SSE/Kafka)
7. Iteration Trail (one row per re-work with delta + effect)
8. Performance Observations (p95 per endpoint if measured)
9. Open Items (`[BLOCK]` / `[INFO]` / `[GAP]`)
10. Re-entry Header

Preserve `harness/*.sh`, `fixtures/*.json`, `runs/run-<N>.log` — audit trail.

Append "검증(/verify)" section to existing `docs/features/...md` or `docs/bugs/...md` (3-5 Korean lines). Standalone mode: **no** docs entry.

Print: `verify.md` absolute path, final status, 3-5 line human summary.

Then **STOP.** `/verify` ends at the filesystem — no commit, push, or PR.

## Scope Boundaries

| In scope | Out of scope |
|---|---|
| HTTP probing of endpoints in `work.md §4` | Load / chaos / soak testing |
| Read-only DB sampling (local default; remote only with consent) | Any DB write on dev/stg/audit/prod without per-statement consent |
| Probing remote envs (dev/stg/audit/prod) **after** Consent Record written to `verify.md §0` | Remote probing without consent record; prod mutating runs without per-request reaffirmation |
| Re-invoking `/work` with bounded delta after consent | Re-invoking with "just fix it" or no delta |
| Re-invoking `/explore` on FAIL-CONTRACT after consent | Silently rewriting `explore.md`/`work.md` |
| Writing `verify.md` + run logs | Modifying `src/` or `test/` |
| PII-redacted fixtures (PII redaction is mandatory for remote-env captures) | Raw user data anywhere under `.backend/`; persisting remote-env tokens to disk |
| Appending "검증" to today's docs entry | Creating brand-new docs entry (`/work`'s job) |

## Red Flags — STOP

- "Skip the happy-path assertion, just check status."
- "Response looks right, I don't need to diff against `§5B.contract`."
- "I'll mock the downstream — faster than starting it."
- "`BASE_URL=https://stg-...` is fine without a Consent Record." **(absolute stop — record consent in `verify.md §0` first)**
- "User mentioned prod once, I'll just run all variants." **(absolute stop — prod is per-request, not per-session)**
- "I'll cache the prod token in `_env.sh` to skip re-export." **(absolute stop — remote tokens never touch disk)**
- "DB row has real email, commit fixture as-is."
- "Iter 2 failed but iter 3 will fix it — one more past the cap."
- "User accepted iter 1, I'll keep re-invoking without checking in."
- "Build passed, skip curl and write `verify.md`."
- "Flaky assertion — mark pass."
- "Feign check failed but my endpoint passed — not my problem."

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Unit tests passed, curl overkill" | Units don't exercise filter chain, deserialization, auth, envelope wrapper. Curl does. |
| "Internal endpoint, no external callers" | Feign-internal still has a contract. Breaking crashes the caller. |
| "DB sampling slow, I'll invent payload" | Real rows have nulls, leading zeros, Korean text, 6-year-old legacy enums. Invented shapes miss these. |
| "User said 'verify it works' — happy is enough" | "Works" in legacy code = "doesn't crash on any DB input type." One variant ≠ verification. |
| "Iter cap arbitrary, keep going" | Cap exists because unproductive loops burn trust and signal a contract problem. Escalate is the correct action. |

## Integration with Other Skills

- **`work`** (predecessor) — produces `work.md`. `/verify` reads but never edits.
- **`explore`** (FAIL-CONTRACT rerun) — if contract is wrong, escalate here not in code.

**Support files (always used):**
- `payload-sourcing.md` — Phase 1 sourcing + PII redaction
- `curl-harness.md` — Phase 2 template + Re-Run Protocol
- `jwt-auth-reference.md` — Phase 2 auth acquisition
- `verify-template.md` — Phase 6 output format

## Quick Reference

| Phase | Input | Output | Fail → |
|---|---|---|---|
| 0. Load | `explore.md` + `work.md` | Target list + TaskCreate | No targets / blocked → stop |
| 1. Sourcing | Targets + DB read-only | Fixtures per variant, redacted | No real shape → ask user |
| 2. Harness | Fixtures + service CLAUDE.md | `harness/*.sh` + `_env.sh` + `_assert.sh` | Unreachable → `[BLOCK]` |
| 3. Execution | Harness + running service | `runs/run-<N>.log` + matrix | Won't start → stop |
| 4. Triage | Matrix | Pass \| FailApp \| FailContract \| FailEnv | FailEnv → stop |
| 5. Iteration | Rework delta | `/work` up to 3x or escalate | No progress 2x → escalate |
| 6. Wrap | All above | `verify.md` + docs append | — |

## Citations & Redaction

- Service behavior claims → cite `runs/run-<N>.log:line`.
- Contract claims → cite `explore.md §5B` or controller/DTO `file:line`.
- PII redaction: emails → `user+<id>@example.test`; phone, national-ID, real names → `user<id>`; organization names → `org<id>`.
- `Authorization:` → `Bearer __REDACTED__`. Cookies/session → `__REDACTED__`.
- Unclassifiable fields → `__UNKNOWN__` + note in `§4`.

## The Bottom Line

`/work` proves the code compiles and unit tests pass.
`/verify` proves the **service**, as assembled, accepts production-shaped payloads, produces documented responses, preserves cross-service contracts — with fresh, recorded, replayable evidence.

Default target is **localhost**. Remote envs (dev/stg/audit/prod) are gated behind a Consent Record and verb/env-specific guard rails — not behind a hard refusal. Use the gate, don't bypass it.

Until `verify.md` says `status: pass`, the change is not production-ready. If contract is wrong, `/verify` escalates to `/explore`. If code is wrong, it hands `/work` a precise delta — up to the cap, then escalates. `/verify` never invents a pass and never silently loops.
