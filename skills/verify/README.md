# `/verify` — live HTTP-level proof, not unit-test theater

## TL;DR

Final gate before a change ships. Runs real `curl` against a live service (localhost by default; remote envs allowed via an explicit, recorded consent gate), captures every request/response verbatim, and refuses to claim "production-ready" without recorded evidence. On failure, hands `/work` a *specific delta* — not "fix it" — and caps the re-work loop at 3 iterations before escalating.

Outputs:
- `.backend/<YYYYMM>/<slug>/verify.md` — dense symbolic report with status `pass | fail-escalated | fail-user-required`
- `.backend/<YYYYMM>/<slug>/harness/*.sh` — reusable curl scripts per endpoint
- `.backend/<YYYYMM>/<slug>/fixtures/*.json` — per-variant payloads, PII redacted
- `.backend/<YYYYMM>/<slug>/runs/run-<N>.log` — raw `curl -i` output per iteration

## What's in this directory

| File | Purpose |
|---|---|
| [`SKILL.md`](./SKILL.md) | The skill definition — 7 phases, iron law, iteration cap, red flags |
| [`payload-sourcing.md`](./payload-sourcing.md) | How to source realistic payloads (user → saved → DB sample → synthesized) with PII redaction |
| [`curl-harness.md`](./curl-harness.md) | Template for the reusable shell harness (`_env.sh`, `_assert.sh`, per-endpoint scripts) |
| [`jwt-auth-reference.md`](./jwt-auth-reference.md) | **Stack-specific auth reference** — adapt to your JWT/session scheme |
| [`verify-template.md`](./verify-template.md) | `verify.md` format with symbol legend extensions |

## The 7 Phases

```
Phase 0 — Load Contract      Derive endpoint target list from work.md §4 / explore.md §7.
Phase 1 — Payload Sourcing   user-provided → saved fixtures → live DB sample → synthesized.
Phase 2 — Harness Generation curl + jq + assertions. Auth token cached in _env.sh.
Phase 3 — Execution          Fire curl. Capture every request/response pair verbatim.
Phase 4 — Triage             PASS | FAIL-APP | FAIL-CONTRACT | FAIL-ENV | PASS-WITH-NOTES
Phase 5 — Iteration (bounded) Hand /work a SPECIFIC delta. Cap: 3 re-invocations.
Phase 6 — Wrap Up            verify.md + runs/ + harness/ + fixtures/ persisted.
```

## What makes this different from "just run the tests"

Unit tests prove *the code compiles and the methods return what their author expected*. They do not prove:

- The HTTP filter chain accepts the payload shape
- Deserialization handles the real enum values, nullables, and 6-year-old legacy data
- The response envelope wrapper preserves the documented contract
- Downstream Feign / Kafka / SSE / WebSocket consumers still work
- The endpoint routes the request correctly under the actual auth flow

`/verify` proves those. With recorded, replayable evidence.

Key disciplines enforced:

- **Real payload shapes** — fixtures come from real DB rows (redacted) or user-provided cases, not invented JSON
- **Three variants minimum** — `happy` + `boundary` + `negative` per endpoint (plus `regression` for bug fixes)
- **Never paraphrase results** — the results matrix is copied verbatim into `verify.md`
- **Localhost-default blast radius** — `BASE_URL` defaults to localhost. Remote envs (dev/stg/audit/prod) are *opt-in*: the user must request them explicitly, the consent is recorded in `verify.md §0`, and `VERIFY_REMOTE_ACK`/`VERIFY_PROD_ACK` flags must be set per the policy table in `SKILL.md`. Remote-env tokens never touch disk.
- **Bounded re-work** — 3-iteration cap prevents unproductive `/work` loops; failure after cap escalates to the user or `/explore`

## Iteration protocol

On FAIL-APP or FAIL-CONTRACT, the skill does **not** just say "it broke, fix it." It constructs a structured delta:

```
endpoint:       POST /bookmarks
variant:        regression
request:        fixtures/bookmarks.regression.json
expected:       200, $.data.bookmarkId present, header X-Request-Id echoed
observed:       500, body {"error":"NPE at BookmarkService.java:47"}, header missing
trace-hint:     BookmarkService.java:45-50 — new nullable param not checked
likely-source:  BookmarkService.java:47 (from work.md §4)
```

This delta — with user consent — is passed to `/work` as a scope-limited re-invocation. `/work` cannot widen scope past the delta.

**Escalation ladder:**

| State | Action |
|---|---|
| Iteration 1 fails | New delta, ask user, re-invoke /work |
| Iteration 2 fails **with progress** | One more attempt (iter 3) allowed |
| Iteration 2 fails **no progress** | STOP. Ask user: re-run /explore, accept partial, or provide new evidence |
| Iteration 3 fails (any) | STOP. `status: fail-escalated`. No more /work. |
| Any iter flips to FAIL-CONTRACT | Skip remaining budget; escalate to /explore |

## Stack-specific adaptation

[`jwt-auth-reference.md`](./jwt-auth-reference.md) is written for **one specific codebase's JWT scheme** (an internal `/test/generate-jwt` dev endpoint). Fork and replace with your stack's auth acquisition:

- OAuth2 client credentials flow
- API key header injection
- Session cookie from dev login endpoint
- mTLS client cert setup
- No-auth dev profile (skip the token step)

The rest of the skill is auth-agnostic.

## Standalone mode

No `/explore`/`/work` artifacts required. If the user supplies an endpoint path + at least one sample payload, `/verify` can run ad-hoc — it skips Phase 0 artifact loading, derives a slug from the endpoint, and proceeds from Phase 1.

## Typical invocation

```
/verify  (after /work has written work.md)
```

Or standalone:

```
/verify POST /api/v1/bookmarks with this payload: { "chapterId": "...", ... }
```

Time budget: 5–10 min for the first run. Re-runs are near-instant since harness/fixtures are cached per ticket.

## When to use

**Use when any holds:**
- `/work` has just completed for a ticket
- You want ad-hoc HTTP verification of any endpoint (standalone mode)
- A bug report arrived — you want to reproduce + gate the fix

**Skip for:**
- Pure frontend-only diffs (run `npm run dev` and use the browser; separate skill owns FE QA)
- Config-only changes with no HTTP contract impact
- Doc/comment-only commits

## Dependencies

**None.** This skill is fully self-contained — the fresh-output rule, the bounded iteration loop, and the parallel-endpoint dispatch technique are all inlined in `SKILL.md`. Install the folder and it works.

**Optional companion skills** (if you also use the superpowers skill pack or similar): `verification-before-completion`, `systematic-debugging`, `dispatching-parallel-agents` — these overlap with the inline procedures and can be used if preferred. They are not required.

## Sibling skills in the pipeline

- [`/explore`](../explore/) — contract drift triggers escalation back here
- [`/work`](../work/) — bounded re-invocation target on FAIL-APP/FAIL-CONTRACT

## The one-sentence summary

> **Until verify.md says `status: pass`, the change is not production-ready — regardless of how green the unit tests are.**
