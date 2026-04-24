---
name: qa-engineer
description: Use when a change has been deployed to a dev/stg/audit/prod environment and browser-level E2E proof against the deployed URL is needed — typically after /verify passes and the deploy lands, for post-deploy smoke checks, or standalone when a specific deployed flow needs ad-hoc browser verification. Not for localhost — /verify owns that.
---

# QA Engineer (Browser-Level E2E Gate)

## Overview

`/qa-engineer` is the **post-deployment** gate. `/verify` proves a locally running service responds correctly to curl; that is not the same as proving users can actually use the feature. The deployed environment has a different reverse proxy, auth flow, CORS, static bundle, CDN, session cookies, browser quirks, and real SSE/WebSocket pipes. Most outages ship through code that passed units, integration, and local curl.

`/qa-engineer` drives a real browser against the deployed URL, walks flows from `explore.md §5B.contract` and `work.md §4`, and asserts `verify.md` assertions still hold with frontend + proxy + real session in the picture.

**Core principle:** No "live for users" claim is trustworthy until a real browser completes the critical flow against the deployed env — with trace, video, and screenshots, replayable on demand.

**Design goals:**
- Deployed-only — `dev-` / `stg-` / `audit-` / `prod-` hostnames; localhost refused (opposite of `/verify`)
- Repeatability is the product — artifact is a `.spec.ts` suite under the ticket folder, re-run on every deploy
- Update-in-place — re-invocation diffs specs against new contract, *updates* not rewrites; passing specs stay green unless contract changed
- Evidence — every pass backed by Playwright trace + video + screenshots
- Prod write gate — prod is **read-only by default**; mutations require per-step user consent
- PII discipline — no real credentials, names, phone numbers, or national-ID numbers in `.backend/` files
- Flake triage — 3-attempt rule; pass,fail,pass = `pass-with-flake`, not `pass`

## The Iron Law

```
NO "LIVE FOR USERS" CLAIM UNTIL BROWSER COMPLETES FLOW ON DEPLOYED URL WITH TRACE+VIDEO
NO QA RUN AGAINST localhost — THAT IS /verify's JOB
NO PROD WRITES WITHOUT EXPLICIT PER-STEP USER CONSENT
NO SPEC REGENERATION ON RE-INVOCATION — SPECS UPDATE IN PLACE
NO EXIT UNTIL qa.md STATUS IS  pass | pass-with-flake | fail-escalated | fail-user-required | blocked-env
```

## Pipeline Position

```
/explore → explore.md  →  /work → work.md  →  /verify → verify.md  (localhost)
                                                 │
                                                 ▼  [ deploy ]
                                  /qa-engineer → qa.md + e2e/  (deployed)
```

Invoke when **any** holds:
1. `/verify` exited `status: pass` AND change deployed to dev (normal path).
2. Post-prod-deploy smoke — re-invocation on existing ticket with `e2e/` suite.
3. Regression report — reproduce in browser against current deploy before looping back.
4. Ad-hoc browser QA without ticket trail (**standalone mode**).

Skip for: pure backend-to-backend internal HTTP-client-only changes (no frontend, no SSE to browser), config with no user-visible flow, doc-only commits.

## Standalone Mode (no ticket folder)

1. Ask: deployed URL, flow description (1-3 sentences), test-account creds or SSO path, env (dev/stg/audit/prod).
2. Derive slug (e.g. `qa-lms-login-banner`) + `<YYYYMM>`.
3. Create `.backend/<YYYYMM>/<slug>/`, start from Phase 2. Record "no explore/work/verify" in `qa.md §0`.
4. Phase 6 writes only `qa.md`. No docs entry.

## Inputs & Outputs

### Inputs

| Source | Extract |
|---|---|
| `explore.md` | §2 scope, §5B contract → expected UI state, §7 changes |
| `work.md` | §4 files edited → UI flows, §7 cross-service (HTTP clients / SSE / queues to browser), §8 follow-ups |
| `verify.md` | §5 results (UI must exercise all proven endpoints), §9 open items |
| `e2e/` (if exists) | Prior specs to UPDATE, `_env.sh`, `storageState.json` to refresh |
| Frontend `CLAUDE.md` | Dev server URL, proxy target, UI auth scheme |
| Repo root `CLAUDE.md` | SSO flow, JWT / session reuse, ports |
| User messages | Deployed URL per env, creds, prod-write consent |

### Outputs (all required)

| Artifact | Path | Purpose |
|---|---|---|
| Spec suite | `e2e/specs/*.spec.ts` | Playwright Test files, re-runnable. One per flow. Grows over time. |
| Smoke spec | `e2e/specs/smoke.spec.ts` | 30-60s critical-path subset, every deploy |
| Env config | `e2e/_env.sh` | `BASE_URL`, `QA_ENV`, `TEST_USER_ID`, storage path per env |
| Auth state | `e2e/storageState.<env>.json` | Redacted cookies/localStorage, refreshed per run |
| Fixtures | `e2e/fixtures/*.json` | Deterministic, redacted |
| Scripted flows | `e2e/scripts/*.sh` | playwright-cli scripts for ad-hoc/debug |
| Run artifacts | `e2e/artifacts/run-<YYYYMMDD-HHMM>-<env>/` | `trace.zip`, `video.webm`, screenshots, `console.log`, `network.har` |
| Report | `qa.md` | Dense symbolic; latest on top; prior runs rolled into §11 History |

**Folder layout:**
```
.backend/<YYYYMM>/<slug>/
  explore.md, work.md, verify.md, qa.md
  e2e/
    _env.sh, _login.sh, playwright.config.ts
    storageState.<env>.json          (prod only with consent)
    specs/   smoke.spec.ts, <flow>.spec.ts, regression.spec.ts
    scripts/ <flow>.sh
    fixtures/<flow>.json
    artifacts/run-<YYYYMMDD-HHMM>-<env>/
      trace.zip, video.webm, screenshot-*.png, console.log, network.har, result.json
```

See `report-template.md` (qa.md format), `playwright-playbook.md` (command patterns), `repeatable-suite.md` (update discipline), `environment-gates.md` (env safety).

## The Seven Phases

Complete each before the next. Only legal loop: Phase 4 → 5 iteration.

### Phase 0 — Load & Validate the Contract

1. Determine ticket folder. Read `explore.md`, `work.md`, `verify.md` in full if they exist.
2. Announce kind + scope + `verify.md` status. If `verify.md` missing or not `status: pass`, STOP unless standalone mode.
3. **Detect re-invocation:** if `e2e/specs/` already has files, this is an **update run**, not fresh generation. Load them; record filenames in TaskCreate.
4. From `work.md §4` + `verify.md §5`, derive **UI flow target list**:
   - Every endpoint proven by `/verify` → one UI flow (unless explicitly internal-only, called by another service not a browser)
   - SSE/WebSocket channels added/changed → one "realtime" flow per channel
   - New/changed pages in web services → one flow per page
5. Per flow record: frontend service, route, entry action (click path), expected end-state (from §5B.contract.resp as UI state).
6. Validate: every `verify.md §5` endpoint has a UI caller in scope OR is flagged `[GAP]` as internal-only. Ask user explicitly: "Run against dev, stg, audit, or prod?" Don't infer.
7. **Env safety preflight** (see `environment-gates.md`) — refuse if:
   - `BASE_URL` resolves to `localhost`/`127.0.0.1`/internal without tunnel consent
   - Target is `prod` and user hasn't consented this invocation

**Output:** TaskCreate + env choice + run-kind + plan restatement.

### Phase 1 — Environment & Auth Bootstrap

See `environment-gates.md` + `playwright-playbook.md §Auth`.

1. Resolve `BASE_URL` per env. Write/update `e2e/_env.sh` with `QA_ENV`, `BASE_URL`, storage path.
2. Obtain auth state:
   - **dev:** prefer a dev-profile test-token path per `verify/jwt-auth-reference.md` (e.g. `GET /test/generate-jwt`), inject into localStorage / cookie per the web app's key. Save to `storageState.dev.json`.
   - **stg / audit:** drive the real SSO login once via `playwright-cli` and save `storageState.<env>.json` (`playwright-playbook.md §Login Flows`).
   - **prod:** user provides test creds in session only (never persist outside redacted storage). Confirm prod-read-only.
3. Check Playwright: `npx --no-install playwright-cli --version` + `playwright --version` must succeed. If missing, ask before installing; decline → `status: blocked-env`.
4. Generate `e2e/playwright.config.ts` if absent. Min: trace on first retry, video retain-on-failure, screenshot only-on-failure, per-test timeout 30s, global 10m, reporter `html`+`json`, `baseURL` + `storageState` from env.
5. Redact storage: emails → `user+<id>@example.test`, names → `user<id>`, national-ID numbers → `__REDACTED_ID__`. Redact before writing.

### Phase 2 — Spec Synthesis

`playwright-cli` is the **probe**, `.spec.ts` is the **artifact**. See `playwright-playbook.md`.

**First-run (no existing specs):**

1. Start session: `playwright-cli -s=qa-<slug> open "$BASE_URL"`.
2. For each flow:
   a. Walk flow via cli (`goto`, `snapshot`, `click eN`, `fill eN`) until end-state. Record commands verbatim to `e2e/scripts/<flow>.sh`.
   b. `playwright-cli --raw snapshot` to capture final DOM as assertion target.
   c. Translate to `e2e/specs/<flow>.spec.ts` — use `getByRole`/`getByTestId`/`getByLabel` over CSS selectors. Assert key intermediate + final states.
   d. ≥3 variants per flow: `happy`, `boundary` (max-length, edges, empty-but-valid), `negative` (invalid/unauth → expected error UI).
   e. Bug fixes add `regression` — exact repro from `explore.md §5A.repro` as browser actions; assert symptom does NOT occur.
3. `e2e/specs/smoke.spec.ts` = happy-path variant from each flow (post-deploy gate).
4. Close cli: `playwright-cli -s=qa-<slug> close`.

**Update-run (specs exist):**

1. Load each `e2e/specs/*.spec.ts`. Diff flow list against specs. Three cases:
   - **Unchanged flow, unchanged contract** — **do not touch the spec.** Most important rule; protects accumulated suite.
   - **Changed contract/path** — re-walk via cli, apply **minimum-delta edit**: only change assertions/selectors that must change. Preserve test names and variant structure. Diff goes to `qa.md §6`.
   - **New flow** — generate per first-run.
2. Removed features → mark spec `.skip` with `// [REMOVED-FEATURE qa:<date>]`. Never delete silently — requires user consent + `§9 [REMOVAL]`.

**Every spec must:**
- Wrap variants in `test.describe('<flow>', () => {...})`
- Use `test.beforeEach` to restore env storage state
- End with `toHaveURL` / `toHaveText` / `toHaveScreenshot` — not just a passing `click`
- No secrets or real PII embedded; use `process.env` / fixtures

### Phase 3 — Execution

1. Run full suite:
   ```bash
   cd .backend/<YYYYMM>/<slug>/e2e
   QA_ENV=dev npx playwright test --config=playwright.config.ts \
     --reporter=html,json --output=artifacts/run-<YYYYMMDD-HHMM>-<env>
   ```
2. Run smoke in isolation (separate run folder). Smoke > 90s → `§9 [PERF]`.
3. Per run capture: `trace.zip`, `video.webm` (always on smoke; fail-only elsewhere), per-step screenshots, `console.log` (WARN/ERROR timestamped), `network.har` (redact `Authorization`), `result.json`.
4. Compute variant matrix (verbatim to `qa.md §5`):

```
flow            variant     env  status  duration(ms)  assertions  notes
login-sso       happy       dev  pass    3242          5/5         —
create-item     happy       dev  fail    8003          3/5         locator not found: getByRole('button',{name:'Save'})
create-item     regression  dev  pass    5220          4/4         bug did not recur
```

5. **Flake detection:** re-run any failed spec up to 2 more times (3 total). `pass,fail,pass` = `pass-with-flake`, not `pass`. No re-runs past 3.

### Phase 4 — Triage

| Classification | Condition | Next |
|---|---|---|
| **PASS** | All `pass`, no flake, smoke < 90s | Phase 6, `status: pass` |
| **PASS-WITH-FLAKE** | Eventually pass but one showed `pass,fail,pass` | Phase 6, `status: pass-with-flake`, `§9 [FLAKE]` |
| **FAIL-UI** | Missing element/wrong text/broken nav — frontend | Phase 5 → `/work` or frontend owner |
| **FAIL-BACKEND** | UI error because deployed backend rejects request `/verify` accepted locally | Phase 5 — dev parity or deployment delta |
| **FAIL-CONTRACT** | UI works but outcome contradicts `§5B.contract` | Phase 5 → `/explore` (contract is wrong) |
| **FAIL-ENV** | 502 from LB, SSO redirect loop, static 404 | STOP. `§9 [BLOCK]`. Not a code issue. |
| **FAIL-TEST** | Spec itself wrong — stale selector, bad assertion, timing race | Update in place (Phase 2 update-run), re-run. Not an app failure. |

For FAIL-UI/BACKEND/CONTRACT prepare **rework delta** (→ `qa.md §7`):
```
flow:               <flow>
variant:            <variant>
env:                <dev|stg|audit|prod>
spec:               e2e/specs/<flow>.spec.ts:<line>
expected:           <UI state, cite explore.md §5B>
observed:           <trace step + screenshot + console excerpt>
network-evidence:   <HAR: request + status + body-snippet (redacted)>
likely-source:      <frontend file:line OR backend controller:method>
classification:     FAIL-UI | FAIL-BACKEND | FAIL-CONTRACT
```

### Phase 5 — Iteration (bounded)

**Cap: 2 upstream re-invocations per `/qa-engineer` session.** Lower than `/verify`'s cap — browser regressions past one round usually mean contract/env, not a tweak.

1. Present delta. Ask: "Re-invoke `<upstream>` with this delta, or stop?"
2. On **accept**:
   - **FAIL-UI / FAIL-BACKEND** → `/work` scoped to spec line + `likely-source`. May only edit referenced files. User must redeploy before re-run.
   - **FAIL-BACKEND surviving dev parity** → `/verify` on localhost; if still passes, it's env → STOP, escalate.
   - **FAIL-CONTRACT** → `/explore` with delta. Do not bypass to `/work`.
3. **Reject/hold** → `status: fail-user-required`, delta as open question.
4. After upstream returns + user confirms redeploy, re-run **only affected specs** (new run folder). Record both runs.
5. **Meaningful progress** (same rules as `/verify`): previously failing passes, or failure moved to more specific, or `likely-source` edited + observed changed.
6. Ladder:
```
iter 1 fails                 → new delta, ask, re-run upstream
iter 2 fails + no progress   → STOP. Ask user: (a) re-run /explore,
                                (b) accept verified scope + defer,
                                (c) abandon QA + revert.
iter 2 fails + progress      → STOP anyway. User decides.
```
7. Flip to FAIL-CONTRACT mid-loop → skip remaining budget, escalate to `/explore`.

### Phase 6 — Wrap Up

Write `qa.md` using `report-template.md`. Sections:

1. One-paragraph recap (Korean prose, only prose section)
2. Meta (kind/slug/yyyymm/env/status/iteration-count/first-run-or-update-run)
3. UI Flow Target List (from Phase 0 + spec file per flow)
4. Env & Auth (URL, auth path, storageState path, redaction notes)
5. Results Matrix (verbatim from Phase 3)
6. Spec Suite Changes (vs last run — new/modified/skipped/preserved)
7. Iteration Trail (one row per upstream re-invocation with delta + effect)
8. Evidence Index (one row per `artifacts/run-.../` with paths)
9. Open Items (`[BLOCK]` / `[INFO]` / `[GAP]` / `[FLAKE]` / `[PERF]` / `[REMOVAL]`)
10. Re-entry Header (future session: what to read, what to re-run, which env)
11. History — one line per prior run: `<YYYYMMDD-HHMM> <env> <status> — <summary>`

Preserve `e2e/specs/*`, `scripts/*`, `fixtures/*`, `artifacts/**` — reusable regression harness.

Append "배포 QA(/qa-engineer)" section to `docs/features/...md` or `docs/bugs/...md` (3-5 Korean lines). Standalone → **no** docs entry.

Print: `qa.md` path, final status, post-deploy re-run command (`cd <ticket>/e2e && QA_ENV=<env> npx playwright test specs/smoke.spec.ts`), 3-5 line summary.

Then **STOP.** No commit/push/PR/Jira transition.

## Scope Boundaries

| In scope | Out of scope |
|---|---|
| Browser E2E of UI flows from `/verify`-proven endpoints | Load/perf/chaos/A-B experiments |
| SSE/WebSocket delivery to real browser | Non-browser clients (mobile, embedded viewers) |
| Visual regression via `toHaveScreenshot` on stable flows | Pixel-perfect cross-browser diffs |
| Re-invoking upstream skills with bounded delta + consent | "Just fix it" re-invocations |
| Updating `e2e/specs/` in place | Silent deletion, regeneration, name mutation |
| Writing `qa.md`, appending docs entry | Modifying `src/` |
| PII-redacted fixtures + storage | Committing real user data |
| Read-only prod + per-step consented writes | Prod writes without explicit consent |
| dev/stg/audit/prod | localhost (that's `/verify`'s job) |

## Red Flags — STOP

- "Skip trace; screenshot enough." **(Iron Law)**
- "`BASE_URL=http://localhost:5173` is faster." **(wrong skill — use `/verify`)**
- "Spec exists but my new flow differs — overwrite." **(update-in-place; add variant or new file)**
- "Failed once, passed twice — mark pass." **(3 attempts; mixed = `pass-with-flake`)**
- "Prod run needs POST to confirm — I'll just do it." **(per-step consent required)**
- "Real creds inline in `_env.sh`." **(PII/secret discipline — session-only)**
- "`/verify` passed locally, deploy must be fine." **(that's exactly this skill's gap)**
- "Flow too complex — I'll manually walk it." **(manual walks aren't repeatable)**
- "Spec failed on selector, delete it." **(deletion requires consent + `[REMOVAL]`)**
- "Iter 2 failed, one more `/work` pass." **(cap is 2)**
- "FAIL-CONTRACT, `/work` saves a round-trip vs `/explore`." **(always `/explore`)**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "playwright-cli alone proves it works" | cli is a probe, not a regression harness. Re-runnable after deploy needs spec files + `npx playwright test`. |
| "Units + `/verify` + manual smoke covered it" | Manual smoke isn't replayable. Next-deploy regressions surface only with a spec file. |
| "Deployed env behaves like dev" | LB, CDN, proxy, bundle all differ per env — that's exactly the failure surface. |
| "Prod reads = stg reads" | Prod session/CORS/rate-limits/SSO differ. Prod smoke is cheap and catches real incidents. |
| "Feature removed soon, skip spec" | Removal is a future PR; until then, flow exists → spec needed. |
| "Spec ugly, regenerate" | Regeneration loses accumulated corrections (waits, retries, boundary tweaks). Edit in place. |
| "CSS selectors simpler" | CSS breaks on refactor. Use role/testid/label so specs survive UI churn. |
| "95s smoke; 90s is arbitrary" | Slow smoke gets skipped. A 60s smoke people actually run. |
| "Flakes are test-infra's problem" | Flake is user-visible — the user who hits it sees a broken app. Record; don't average. |

## Integration with Other Skills

- **`verify`** (predecessor) — §5 matrix defines endpoints UI must exercise. Read, never edit.
- **`work`** (re-invoked on FAIL-UI/BACKEND) — delta scoped to spec line + likely-source. User must redeploy before re-run.
- **`explore`** (re-invoked on FAIL-CONTRACT) — contract is wrong; escalate here.
- **`playwright-cli`** (Microsoft's, or inline `playwright-playbook.md`) — the probe for Phase 2.
- **`jwt-auth-reference.md`** (in `~/.claude/skills/verify/`, forked to your auth scheme) — dev-env auth bootstrap in Phase 1.

## Quick Reference

| Phase | Input | Output | Fail → |
|---|---|---|---|
| 0. Load | explore/work/verify + optional existing specs | flow list + TaskCreate + env + run-kind | `verify.md` not pass → stop |
| 1. Bootstrap | env + creds/JWT | `_env.sh` + `storageState.<env>.json` + Playwright installed | install declined → `blocked-env` |
| 2. Spec | flows + cli + existing specs | `specs/*.spec.ts` + `smoke.spec.ts` + `scripts/*.sh` | flow blocked (auth loop) → `[BLOCK]` |
| 3. Execute | specs + env + 3-attempt flake | `artifacts/run-<ts>-<env>/` + matrix | deploy unhealthy → FAIL-ENV, stop |
| 4. Triage | matrix | Pass\|PassWithFlake\|FailUI\|FailBackend\|FailContract\|FailEnv\|FailTest | FAIL-ENV → stop |
| 5. Iterate | delta | Upstream re-run (≤2x) or escalate | no progress 2x → escalate |
| 6. Wrap | all above | `qa.md` + smoke re-run cmd + docs append | missing qa.md → not done |

## Citations & Redaction

- Browser behavior claims → cite `artifacts/run-<ts>-<env>/trace.zip@step<N>` or `screenshot-<step>.png` or `console.log:<line-range>`.
- Contract claims → cite `explore.md §5B` or `verify.md §5` row.
- PII redaction before writing any `.backend/` file:
  - Email → `user+<id>@example.test`; phone → `000-0000-0000`; national-ID → `__REDACTED_ID__`
  - Personal name → `user<id>`; organization name → `org<id>`
  - Cookies/session → keep structure; `Authorization` value → `Bearer __REDACTED__` with note on re-obtaining
- `network.har` `Authorization` values → `Bearer __REDACTED__` after capture.
- Never commit real creds, even base64, even in `.sh`.

## The Bottom Line

`/verify` proves the service accepts payloads locally.
`/qa-engineer` proves the **deployed system** — frontend + proxy + CDN + real session + real SSE — delivers the feature to a user's browser, with a **reusable spec suite that re-proves it on every future deploy**.

Until `qa.md` says `pass` (or `pass-with-flake` with accepted list), the deploy isn't user-verified. The spec suite under `e2e/specs/` is the durable product; `qa.md` is the audit log. Together they turn "I tested it in my browser and it worked" into a replayable artifact.
