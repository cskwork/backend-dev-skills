---
name: qa-engineer
description: Use when a change has been deployed to a dev/stg/audit/prod project environment and browser-level E2E proof against the deployed URL is needed — typically after /verify passes and the deploy lands, for post-deploy smoke checks, or standalone when a specific deployed flow needs ad-hoc browser verification. Not for localhost — /verify owns that.
---

# QA Engineer (Browser-Level E2E Gate)

## Overview

`/qa-engineer` is the **post-deployment** gate. `/verify` proves that a locally running service responds correctly to real curl payloads. **That is not the same as proving users can actually use the feature.** The deployed environment has a different reverse proxy, a different auth flow, different CORS config, a different static bundle, a different CDN, different session cookies, real browser quirks, and real SSE/WebSocket pipes terminating at real load balancers. Almost every outage has shipped through code that passed unit tests, integration tests, and a local curl check.

`/qa-engineer` closes that gap. It drives a real browser against the deployed URL, walks the user flows described in `explore.md §5B.contract` and `work.md §4 Changes`, and asserts that every assertion in `verify.md` still holds once the frontend, the reverse proxy, and the real session flow are in the picture.

**Core principle:** No "this is live for users" claim is trustworthy until a real browser successfully completes the critical flow against the deployed environment — with a recorded trace, video, and screenshots, replayable on demand.

**Design goals every run must optimize for:**
- **Deployed-only blast radius** — probes hit `dev-` / `stg-` / audit / `prod` hostnames; localhost is refused (opposite of `/verify`).
- **Repeatability is the product** — the artifact is not a single pass/fail. It is a `.spec.ts` suite under the ticket folder that any future operator re-runs on every deploy.
- **Update-in-place discipline** — on re-invocation for the same `<YYYYMM>/<slug>`, specs are diffed against the new contract and *updated*, not rewritten. Previously passing specs stay green unless the contract changed.
- **Evidence, not assertion** — every pass is backed by Playwright trace (`trace.zip`), video (`video.webm`), and per-step screenshots.
- **Prod write gate** — prod runs are **read-only by default**; any step that would mutate prod data requires explicit user consent per step, logged in `qa.md §9`.
- **PII discipline** — no real user credentials, student names, phone numbers, or national ID get written into any file under `.backend/`. Auth goes through the documented test fixtures or redacted `storageState.json`.
- **Flake triage over retry-until-green** — a variant that passes 3 times and fails twice is flaky, not passing. Record flakiness; do not average it away.

**Violating the letter of this process is violating the spirit.** A "QA pass" with no trace, no video, or from localhost, is not a QA pass.

## The Iron Law

```
NO "LIVE FOR USERS" CLAIM UNTIL A REAL BROWSER COMPLETES THE FLOW
  AGAINST THE DEPLOYED URL, WITH TRACE + VIDEO RECORDED
NO QA RUN AGAINST localhost FROM THIS SKILL — THAT IS /verify's JOB
NO PROD WRITES WITHOUT EXPLICIT PER-STEP USER CONSENT
NO SPEC REGENERATION ON RE-INVOCATION — SPECS UPDATE IN PLACE
NO SKILL EXIT UNTIL qa.md IS WRITTEN AND STATUS IS
  pass | pass-with-flake | fail-escalated | fail-user-required | blocked-env
```

## Pipeline Position

```
/explore  →  explore.md       (the contract)
/work     →  work.md          (the code + unit/integration proof)
/verify   →  verify.md        (HTTP-level E2E against localhost)
    │
    ▼  [ user deploys to dev / stg / audit / prod ]
    │
/qa-engineer → qa.md + e2e/   (browser E2E against deployed URL, repeatable)
```

Invoke `/qa-engineer` when **any** of the following holds:

1. `/verify` has exited with `status: pass` AND the change has been deployed to dev (normal path).
2. A post-prod-deploy smoke check is requested — a re-invocation for an existing `<YYYYMM>/<slug>` that already has an `e2e/` suite.
3. A regression report arrived and you want to reproduce it in a browser against the current deployed build, before looping back to `/explore` or `/work`.
4. The user asks for ad-hoc browser QA of a deployed feature without an explore/work/verify trail (**standalone mode** — see below).

Skip only for:
- Pure backend-to-backend Feign-only changes with zero frontend surface and no SSE/WebSocket push to a browser (`/verify` alone is sufficient).
- Config changes that do not affect any user-visible flow.
- Doc-only commits.

## Standalone Mode (no ticket folder)

If the user wants ad-hoc browser QA without an existing ticket folder:

1. Ask for: deployed URL (full, including protocol and host), user flow description (1-3 sentences), test-account credentials or SSO path, environment (dev / stg / audit / prod).
2. Derive a slug (e.g. `qa-login-banner`) and today's `<YYYYMM>`.
3. Create `.backend/<YYYYMM>/<slug>/` and proceed from Phase 2 onward. Record in `qa.md §0` that `explore.md` / `work.md` / `verify.md` do not exist.
4. On Phase 6 write only `qa.md`. No docs entry — standalone QA does not own the public changelog.

## Inputs & Outputs

### Inputs

| Source | What to extract |
|---|---|
| `.backend/<YYYYMM>/<slug>/explore.md` | §2 scope, §5B.contract (req/resp shape → expected UI state), §7 proposed changes |
| `.backend/<YYYYMM>/<slug>/work.md` | §4 files edited (derive affected UI flows), §7 cross-service impact (Feign/SSE/Kafka that terminate in a browser pipe), §8 follow-ups |
| `.backend/<YYYYMM>/<slug>/verify.md` | §5 results matrix (which endpoints are now proven to respond correctly — UI must exercise all of them), §9 open items |
| `.backend/<YYYYMM>/<slug>/e2e/` | **If it exists**: prior specs to UPDATE, `_env.sh` to reuse, `storageState.json` to refresh |
| Frontend service `CLAUDE.md` | Vite dev server URL, proxy target, auth scheme for the UI |
| project root `CLAUDE.md` | SSO flow, JWT reuse across services, ports |
| User messages | Deployed URL per env, test-account handle, specific scenarios to exercise, accept/reject on prod writes |

### Outputs (all required for completion)

| Artifact | Path | Purpose |
|---|---|---|
| Spec suite | `.backend/<YYYYMM>/<slug>/e2e/specs/*.spec.ts` | Playwright Test files, re-runnable via `npx playwright test`. One file per flow. Grows over time. |
| Smoke spec | `.backend/<YYYYMM>/<slug>/e2e/specs/smoke.spec.ts` | 30-60s critical-path subset; run after every deploy. |
| Env config | `.backend/<YYYYMM>/<slug>/e2e/_env.sh` | `BASE_URL`, `QA_ENV`, `TEST_USER_ID`, storage state path — per target env. |
| Auth state | `.backend/<YYYYMM>/<slug>/e2e/storageState.json` | Persisted cookies/localStorage from one login; PII-redacted. Refreshed per run. |
| Fixtures | `.backend/<YYYYMM>/<slug>/e2e/fixtures/*.json` | Deterministic test data, redacted. |
| Scripted flows | `.backend/<YYYYMM>/<slug>/e2e/scripts/*.sh` | playwright-cli shell scripts for ad-hoc/debug use (not the primary artifact, but preserved for record). |
| Run artifacts | `.backend/<YYYYMM>/<slug>/e2e/artifacts/run-<YYYYMMDD-HHMM>-<env>/` | `trace.zip`, `video.webm`, screenshots, `console.log`, `network.har`. One folder per execution. |
| Report | `.backend/<YYYYMM>/<slug>/qa.md` | Dense symbolic report (latest run on top; prior runs rolled into §11 History). |

**Folder layout (post-qa):**

```
.backend/
  <YYYYMM>/
    <slug>/
      explore.md          ← from /explore
      work.md             ← from /work
      verify.md           ← from /verify
      qa.md               ← this skill  (single file; each run appends a History row, replaces §3-9)
      e2e/
        _env.sh           ← BASE_URL / QA_ENV / TEST_USER_ID per environment
        _login.sh         ← obtains auth state, writes storageState.<env>.json
        playwright.config.ts ← per-ticket config (reuses root config if present)
        storageState.dev.json
        storageState.stg.json
        storageState.prod.json    (prod only if user consented)
        specs/
          smoke.spec.ts           ← critical path, runs every deploy
          <flow>.spec.ts          ← full happy / boundary / negative per flow
          regression.spec.ts      ← repro specs for any bug fixed in this ticket
        scripts/
          <flow>.sh               ← playwright-cli scripted flow (for debug / teach)
        fixtures/
          <flow>.json
        artifacts/
          run-20260421-1430-dev/
            trace.zip
            video.webm
            screenshot-<step>.png
            console.log
            network.har
            result.json
          run-20260421-1612-stg/
            ...
```

See `report-template.md` for the `qa.md` format, `playwright-playbook.md` for command patterns, `repeatable-suite.md` for the update discipline, and `environment-gates.md` for per-env safety rules.

## The Seven Phases

Complete each phase before the next. The only legal loop is Phase 4 → Phase 5 iteration.

### Phase 0 — Load & Validate the Contract

1. Determine ticket folder — ask or infer `<YYYYMM>/<slug>`. Read `explore.md`, `work.md`, and `verify.md` in full if they exist.
2. Announce kind (Bug / Feature / Refactor), scope one-liner from `explore.md §2`, and `verify.md` status. If `verify.md` is missing or not `status: pass`, STOP and tell the user `/verify` must pass before `/qa-engineer` runs. The only exception is standalone mode.
3. **Detect re-invocation**: if `.backend/<YYYYMM>/<slug>/e2e/specs/` already contains spec files, this is an **update run**, not a fresh generation. Load them into memory. Record spec filenames in TaskCreate.
4. From `work.md §4` + `verify.md §5`, derive the **UI flow target list** — user-visible flows that must be exercised:
   - Every endpoint proven by `/verify` must have a UI caller in the relevant frontend service; each such caller is one flow.
   - SSE/WebSocket channels added or changed → one "realtime" flow per channel.
   - New/changed pages in `frontend-web`, `admin-web`, `content-web`, `viewer-web`, or `chat-widget` → one flow per page.
5. For each flow, record: frontend service, route (e.g. `/class/:id/assign`), entry action (click path from landing page), expected end-state (from `explore.md §5B.contract.resp` interpreted as UI state).
6. Validate:
   - Every endpoint in `verify.md §5` has a UI caller in scope OR is explicitly documented as internal-only Feign. Flag any endpoint-to-UI gap in `qa.md §9 [GAP]`.
   - Environment target — ask the user explicitly: "Run against dev, stg, audit, or prod?" Do not infer.
7. **Environment safety preflight** — see `environment-gates.md`. Refuse to proceed if:
   - User-provided `BASE_URL` resolves to `localhost` / `127.0.0.1` / internal-only hostname without tunnel consent. (`/verify` handles localhost.)
   - Target env is `prod` and the user has not explicitly consented to a prod run for this invocation.

**Output of this phase:** a TaskCreate list mirroring the UI flow target list, the environment choice, whether this is a first-run or update-run, and the plan restatement ("I will exercise flows A, B, C on <env> (base: <BASE_URL>). This is an update-run over <N> existing specs.").

### Phase 1 — Environment & Auth Bootstrap

Follow `environment-gates.md` for the env safety contract and `playwright-playbook.md §Auth` for Project login patterns.

1. Resolve `BASE_URL` per env from the user's answer OR from the service's `CLAUDE.md`. Write (or update) `e2e/_env.sh` with `QA_ENV`, `BASE_URL`, and the path to the env-specific `storageState.<env>.json`.
2. Obtain auth state:
   - **dev env, project backend**: prefer the test JWT path per `verify/jwt-auth-reference.md` — fetch a token via `GET /test/generate-jwt`, inject into localStorage or as a cookie per the web app's expected key. Save to `storageState.dev.json`.
   - **stg / audit env**: use the SSO provider test account the user has authorized, driven interactively with `playwright-cli` (see `playwright-playbook.md §Project Login Flows`). Save result to `storageState.<env>.json`.
   - **prod env**: require the user to provide the test-account credentials in this session only (never persist outside the redacted storageState). Confirm the prod-read-only contract (see Phase 4 gate).
3. Install/check Playwright:
   - Confirm `npx --no-install playwright-cli --version` and `npx --no-install playwright --version` both succeed.
   - If missing, ask the user before running `npm install @playwright/test playwright-cli @playwright/cli` (do not auto-install into their project root without consent).
   - If missing and the user declines, STOP with `status: blocked-env`.
4. Generate `e2e/playwright.config.ts` if absent. Minimum config: trace on first retry, video retain-on-failure, screenshot only-on-failure, per-test timeout 30s, global timeout 10m, reporter `html` + `json`, `baseURL` from env, `storageState` from env-specific file.
5. Redact any captured storage state: replace real user emails with `user+<id>@example.test`, names with `student<id>`/`school<id>`, any national ID with `__REDACTED_ID__`. Run the redaction before writing the file; never commit raw PII.

### Phase 2 — Spec Synthesis (via playwright-cli interactive exploration)

This is where `/qa-engineer` consumes Microsoft's `playwright-cli` skill. The `playwright-cli` is the **probe**; the `.spec.ts` is the **artifact**. Follow `playwright-playbook.md` for command patterns.

**For first-run** (no existing specs):

1. Start a named cli session: `playwright-cli -s=qa-<slug> open "$BASE_URL"`.
2. For each flow in the UI flow target list:
   a. Walk the flow manually via `playwright-cli`: `goto`, `snapshot`, `click eN`, `fill eN`, until the expected end-state is reached. Record every command verbatim into `e2e/scripts/<flow>.sh` as it happens.
   b. After the flow completes, use `playwright-cli --raw snapshot` to capture the final DOM state as a plain-text assertion target.
   c. Translate the scripted flow into a Playwright Test spec file at `e2e/specs/<flow>.spec.ts`, using `page.getByRole`, `page.getByTestId`, `page.getByLabel` locators in preference to CSS selectors. Add assertions for the key intermediate and final states.
   d. Add at least three variants per flow:
      - `happy` — canonical success path, assert end-state.
      - `boundary` — max-length input / min/max numeric / empty-but-valid collection; assert UI remains in valid state.
      - `negative` — invalid input or unauth; assert UI shows the expected error UI (toast, inline message, modal).
   e. If this ticket is a bug fix, add a `regression` variant with the exact repro from `explore.md §5A.repro` expressed as a browser action sequence; assert the user-visible symptom does NOT occur.
3. Populate `e2e/specs/smoke.spec.ts` with the happy-path variant from each flow — this is the post-deploy gate for future runs.
4. Close the cli session: `playwright-cli -s=qa-<slug> close`.

**For update-run** (specs already exist):

1. Load each existing `e2e/specs/*.spec.ts` into memory.
2. Diff the UI flow target list against the existing specs. Three cases:
   - **Unchanged flows** — spec exists and the underlying endpoint/contract/UI route is unchanged per `explore.md §5B` and `work.md §4`. **Do not touch the spec.** This is the most important rule: update discipline protects the accumulated test suite.
   - **Changed flows** — spec exists, but the contract or the UI path changed. Use `playwright-cli` to re-walk the updated flow, then apply a **minimum-delta edit** to the existing spec: change only the assertions or selectors that must change. Preserve test names, preserve variant structure. Record the diff in `qa.md §6`.
   - **New flows** — no spec exists. Generate per the first-run procedure above.
3. If any previously passing spec is now outdated because the feature was removed, mark it `.skip` in place with a `// [REMOVED-FEATURE qa:<date>]` breadcrumb comment. Never delete a spec silently — deletion requires user consent and is recorded in `qa.md §9 [REMOVAL]`.

**Every spec file must:**
- Have `test.describe('<flow>', () => { ... })` wrapping its variants.
- Use `test.beforeEach` to restore storage state for the target env.
- End with `await expect(page).toHaveURL(...)` or `toHaveText(...)` or a `toHaveScreenshot()` visual assertion, not just a passing `click`.
- NOT embed secrets or real PII. Reference `process.env.QA_USER_ID` / fixtures instead.

### Phase 3 — Execution

1. Run the full suite against the chosen env:
   ```bash
   cd .backend/<YYYYMM>/<slug>/e2e
   QA_ENV=dev npx playwright test --config=playwright.config.ts \
     --reporter=html,json \
     --output=artifacts/run-<YYYYMMDD-HHMM>-<env>
   ```
2. Run the smoke suite in isolation too (separate run folder). Smoke alone should complete in under 90 seconds. If it exceeds that, flag `qa.md §9 [PERF]`.
3. Capture per run:
   - `artifacts/run-<YYYYMMDD-HHMM>-<env>/trace.zip` (Playwright trace, viewable via `npx playwright show-trace`).
   - `artifacts/run-<YYYYMMDD-HHMM>-<env>/video.webm` (only on failure by default; always-on for the critical-path smoke).
   - Per-step screenshots under `artifacts/.../screenshots/`.
   - `console.log` — all browser console messages at WARN or ERROR, timestamped.
   - `network.har` — HAR capture of every XHR/fetch. Redact `Authorization` header values after capture.
   - `result.json` — Playwright JSON reporter output.
4. Compute the variant result matrix — verbatim into `qa.md §5`:

```
flow                variant     env  status  duration(ms)  assertions  notes
login-sso           happy       dev  pass    3242          5/5         —
login-sso           negative    dev  pass    1410          2/2         expected toast shown
assign-class        happy       dev  fail    8003          3/5         locator not found: getByRole('button',{name:'Assign'})
assign-class        boundary    dev  pass    4112          3/3         —
assign-class        regression  dev  pass    5220          4/4         bug did not recur
```

5. **Flake detection**: immediately re-run any failed spec up to 2 more times (total 3 attempts). Record all three outcomes. A variant that is `pass,fail,pass` is `pass-with-flake`, not `pass`. Flakes are real — do not re-run past 3.

6. **Never paraphrase the result matrix.** It is copied verbatim into `qa.md §5`.

### Phase 4 — Triage

Classify each failure:

| Classification | Condition | Next |
|---|---|---|
| **PASS** | All variants `pass`; no flake; smoke < 90s | Phase 6 wrap-up, `status: pass` |
| **PASS-WITH-FLAKE** | All variants eventually pass but at least one showed `pass,fail,pass`-type pattern | Phase 6 wrap-up, `status: pass-with-flake`; flake logged in §9 `[FLAKE]` |
| **FAIL-UI** | Variant fails due to missing element, wrong text, broken navigation — attributable to the frontend build | Phase 5 iteration: handoff to `/work` or frontend owner |
| **FAIL-BACKEND** | UI shows an error toast / 5xx because the deployed backend rejects a request that `/verify` accepted locally | Phase 5: dev-env parity issue OR deployment delta; escalate with details to user / `/verify` re-run on deployed |
| **FAIL-CONTRACT** | UI works but the user-visible outcome contradicts `explore.md §5B.contract` as written | Phase 5: escalate to `/explore` — the contract is wrong, not the code |
| **FAIL-ENV** | Deployment not healthy — 502 from LB, SSO redirect loop, static asset 404 | STOP. Record in §9 `[BLOCK]`. Not a code issue. Ask user. |
| **FAIL-TEST** | The spec itself is wrong — stale selector, bad assertion, timing race | Update spec in place (Phase 2 update-run logic), re-run. Do not count as an app failure. |

For **FAIL-UI**, **FAIL-BACKEND**, and **FAIL-CONTRACT**, prepare a **rework delta**:

```
flow:               <flow-name>
variant:            <variant-name>
env:                <dev|stg|audit|prod>
spec:               e2e/specs/<flow>.spec.ts:<line>
expected:           <UI state description, cite explore.md §5B>
observed:           <trace step + screenshot filename + console excerpt>
network-evidence:   <HAR entry — request + response status + body-snippet (redacted)>
likely-source:      <frontend file:line OR backend controller:method from work.md §4>
classification:     FAIL-UI | FAIL-BACKEND | FAIL-CONTRACT
```

This delta is the input to Phase 5 and is saved verbatim into `qa.md §7`.

### Phase 5 — Iteration (bounded)

**Iteration cap: 2 re-invocations of an upstream skill (`/work` OR `/verify` OR `/explore`) per `/qa-engineer` session.** Lower than `/verify`'s cap because browser-level regressions that persist past one round of rework usually indicate a contract or env problem, not a code-tweak issue.

For each fail classification:

1. Present the rework delta to the user. Ask: "Re-invoke `<upstream skill>` with this delta, or stop here?"
2. On **user accept**:
   - **FAIL-UI / FAIL-BACKEND** → `/work` with the delta scoped to the specific spec line and `likely-source`. `/work` may only edit the files referenced in the delta. On return, user must redeploy before re-run.
   - **FAIL-BACKEND that survives dev parity check** → `/verify` re-run on localhost to confirm parity with deployed; if `/verify` still passes, the delta is a deployment/env issue → STOP, escalate to user.
   - **FAIL-CONTRACT** → `/explore` with the delta. The contract is wrong. Do not bypass this into `/work`.
3. On **user reject / hold** → stop. Write `qa.md` with `status: fail-user-required` and the delta as the question.
4. After the upstream skill returns AND the user confirms redeployment, re-run **only the affected specs** (not the full suite) with a new run folder. Record both runs — the failing one and the re-verified one.
5. **Meaningful progress** check (same rules as `/verify §Phase 5`):
   - A previously failing variant now passes.
   - The failure moved to a different, more specific classification.
   - The `likely-source` was edited and the observed value changed.

   If none hold, the iteration did not make meaningful progress.
6. Escalation ladder:

```
iter 1 fails                  → generate new delta, ask, re-run upstream
iter 2 fails AND no progress  → STOP. Ask user:
                                 (a) re-run /explore (contract likely wrong),
                                 (b) accept verified scope and defer rest,
                                 (c) abandon QA for this deploy and revert.
iter 2 fails WITH progress    → STOP anyway. Record state. User decides next step.
```

7. If any iteration flips classification to **FAIL-CONTRACT**, abandon the remaining upstream budget and escalate to `/explore` immediately.

### Phase 6 — Wrap Up (qa.md)

Write `.backend/<YYYYMM>/<slug>/qa.md` using `report-template.md` in this directory. Required sections:

1. §1 One-paragraph recap (user-language prose, plain-language; the only prose section).
2. §2 Meta (kind / slug / yyyymm / env / status / iteration-count / first-run-or-update-run).
3. §3 UI Flow Target List (matches Phase 0 output, with the spec file per flow).
4. §4 Environment & Auth (env URL, auth path chosen, storageState path, redaction notes).
5. §5 Results Matrix (verbatim from Phase 3).
6. §6 Spec Suite Changes (diff summary vs last run — new specs, modified specs, skipped specs, preserved specs).
7. §7 Iteration Trail (one row per upstream re-invocation with the delta, target skill, and delta's effect).
8. §8 Evidence Index (one row per `artifacts/run-.../` folder with trace/video/screenshot paths).
9. §9 Open Items (`[BLOCK]` / `[INFO]` / `[GAP]` / `[FLAKE]` / `[PERF]` / `[REMOVAL]`).
10. §10 Re-entry Header (how a future session reloads context: what to read, what to re-run, which env to target).
11. §11 History — one line per prior `/qa-engineer` run on this ticket: `<YYYYMMDD-HHMM> <env> <status> — <summary>`. Grows with every re-invocation. Prior run detail lives in `artifacts/`; the History is the index.

Also:
- Every `e2e/specs/*.spec.ts`, `scripts/*.sh`, `fixtures/*.json`, and `artifacts/**` stays in the ticket folder. They are reusable evidence AND the future operator's regression harness.
- Append a "Deployment QA (/qa-engineer)" section to the existing `docs/features/<YYYY-MM-DD>-<slug>.md` or `docs/bugs/...md` change note summarizing in 3-5 lines of plain language. If no docs entry exists (standalone mode), do not create one.

After writing, print:
- Absolute path to `qa.md`.
- Final status: `pass` | `pass-with-flake` | `fail-escalated` | `fail-user-required` | `blocked-env`.
- Post-deploy re-run command: `cd <ticket>/e2e && QA_ENV=<env> npx playwright test specs/smoke.spec.ts` — so the user (or CI) can re-verify this flow on every subsequent deploy.
- A 3-5 line human summary.

Then **STOP**. Do not commit, push, open a PR, or transition a Jira ticket. `/qa-engineer`, like `/work` and `/verify`, ends at the filesystem.

## Scope Boundaries

| In scope for `/qa-engineer` | Out of scope |
|---|---|
| Browser-level E2E of UI flows that call endpoints proven by `/verify` | Load/perf testing, chaos testing, A/B experiment measurement |
| SSE / WebSocket message delivery to a real browser page | Non-browser clients (mobile apps, embedded viewers) — separate skill |
| Visual regression via `toHaveScreenshot()` on stable flows | Pixel-perfect cross-browser diffs — out of scope, note in §9 if relevant |
| Re-invoking `/work` / `/verify` / `/explore` with a bounded delta after consent | Re-invoking any skill with "just fix it" and no delta |
| Creating or updating `e2e/specs/` spec files in place | Deleting specs silently, regenerating from scratch on re-invocation, mutating spec names to hide history |
| Writing `qa.md`, appending to docs entry | Modifying source code. `/qa-engineer` never edits `src/` |
| Redacting PII before saving fixtures or storage state | Committing any real user data to `.backend/` |
| Read-only assertions against prod with per-step consent | Prod writes without explicit per-step user consent |
| Running against dev / stg / audit / prod | Running against localhost — that is `/verify`'s job |

## Red Flags — STOP and Restart Phase

If you catch yourself thinking:

- "I'll skip the trace capture; the screenshot is enough." **(Iron Law)**
- "`BASE_URL=http://localhost:5173` is faster to test." **(wrong skill — use `/verify`)**
- "The spec already exists for this flow but my new flow is different, I'll overwrite it." **(update-in-place rule — add a variant or a new file, do not overwrite)**
- "Failed once, passed twice — it's fine, mark pass." **(3 attempts; mixed results means `pass-with-flake`, not `pass`)**
- "The prod run needs to POST to confirm — I'll just do it." **(per-step user consent for any prod write)**
- "The user gave me their real creds; I'll inline them in `_env.sh`." **(PII/secret discipline; credentials go in user's env or session-only)**
- "`/verify` passed on localhost, so the dev deploy must be fine." **(that is exactly the gap `/qa-engineer` exists to close)**
- "The flow is too complex for a spec; I'll just manually walk it." **(a manual walk is not repeatable; the whole point is the spec file)**
- "This spec failed because the selector moved; I'll delete it." **(deletion requires user consent AND a `[REMOVAL]` entry)**
- "Iteration 2 failed but let me try one more `/work` pass." **(cap is 2; past the cap, escalate)**
- "Prior run failed for contract; I'll run `/work` instead of `/explore` to save a round-trip." **(FAIL-CONTRACT always goes to `/explore`)**

**All of these mean: STOP. Return to the phase that enforces the missing discipline.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Playwright is heavyweight; playwright-cli alone proves it works" | cli is a probe, not a regression harness. The user wants the script re-runnable after every deploy. That requires spec files + `npx playwright test`. |
| "Unit tests + `/verify` + manual smoke covered it" | Manual smoke is not replayable on next deploy. Most regressions are silent until the next deploy surfaces them; only a spec file catches them. |
| "The deployed env behaves like dev; one run is enough" | LB, CDN, reverse proxy, and static bundle all differ per env. That difference is exactly the failure surface `/qa-engineer` gates. |
| "Prod reads are identical to stg reads; I'll skip prod" | Prod session cookies, prod CORS, prod rate-limits, and prod SSO behave differently. Smoke-on-prod is cheap and catches real incidents. |
| "This feature will be removed soon; skip the spec" | Removal is a future PR. Until then, the flow exists and needs a spec. |
| "The spec file is ugly; let me regenerate it" | Regeneration loses accumulated corrections (waits, retry loops, boundary tweaks). Edit in place. |
| "I'll use CSS selectors — they're simpler" | CSS selectors break on every frontend refactor. Use role / testid / label locators so specs survive UI churn. |
| "The test took 95s; 90s is arbitrary" | Slow smoke suites get skipped. A 60s smoke is a smoke suite people actually run. |
| "Flakes are a test-infra problem, not a QA problem" | Flake is user-visible — a user who hits the flaky state sees a broken app. Record it; do not average it away. |

## Integration with Other Skills

- **`verify`** (typical predecessor) — produces `verify.md §5` whose result matrix defines the endpoint set that the UI must exercise. `/qa-engineer` reads but never edits `verify.md`.
- **`work`** (re-invoked on FAIL-UI / FAIL-BACKEND) — receives a tight delta scoped to the spec line and likely source. User must redeploy after `/work` returns before re-run.
- **`explore`** (re-invoked on FAIL-CONTRACT) — the contract is wrong, not the code; escalate here.
- **`playwright-cli`** (Microsoft's skill, at `skills/playwright-cli/` if installed locally; or documented in `playwright-playbook.md` in this dir) — the interactive probe used during Phase 2 to walk the flow and generate the spec.
- **`verification-before-completion`** — governs the "fresh output only" rule. `/qa-engineer` is a superset of this rule for the browser layer.
- **`systematic-debugging`** — use when a FAIL-BACKEND triage cannot identify a plausible `likely-source`. Do not hand `/work` a rework delta without a call-chain hypothesis.
- **`jwt-auth-reference.md`** (in `~/.claude/skills/verify/`) — project-specific JWT acquisition procedure for dev-env auth bootstrap in Phase 1.
- **`finishing-a-development-branch`** — use AFTER `/qa-engineer` exits with `status: pass`, if the user wants to merge / promote across envs.
- **`jira-update` / `jira-ascli`** — use AFTER `/qa-engineer` exits, if the user wants the ticket updated with QA evidence (qa.md link + smoke re-run command).
- **`dispatching-parallel-agents`** — use when Phase 3 must exercise 3+ independent flows across 2+ envs; dispatch one agent per (flow, env) pair.

## Quick Reference

| Phase | Input | Output | Fail state → |
|---|---|---|---|
| 0. Load Contract | explore/work/verify + optional existing `e2e/specs/` | UI flow target list + TaskCreate + env choice + run-kind (first/update) | `verify.md` missing or not pass → stop, ask user |
| 1. Env & Auth Bootstrap | env choice + user credentials OR test JWT path | `_env.sh` + `storageState.<env>.json` (redacted) + Playwright installed | Playwright unavailable and user declines install → `status: blocked-env` |
| 2. Spec Synthesis | flow targets + playwright-cli + existing specs | `specs/*.spec.ts` (created or minimum-delta updated) + `specs/smoke.spec.ts` + `scripts/*.sh` | Flow cannot be driven (auth loop, etc.) → `[BLOCK]`, ask user |
| 3. Execution | spec suite + env + 3-attempt flake protocol | `artifacts/run-<ts>-<env>/` + result matrix + console/HAR | Deploy not healthy → `FAIL-ENV`, stop |
| 4. Triage | result matrix | Pass \| PassWithFlake \| FailUI \| FailBackend \| FailContract \| FailEnv \| FailTest | FAIL-ENV → stop, not a code issue |
| 5. Iteration | rework delta | Re-run upstream (up to 2x) or escalate | No progress after 2 → escalate to explore/user |
| 6. Wrap Up | all above | `qa.md` + smoke re-run command + docs note append | Missing qa.md → not done |

## Citations & Redaction Discipline

- Every claim about browser behavior must cite a specific artifact: `artifacts/run-<ts>-<env>/trace.zip@step<N>` or `screenshot-<step>.png` or `console.log:<line-range>`.
- Every claim about the contract must cite `explore.md §5B` or the verified `verify.md §5` row.
- PII categories to redact before writing any file under `.backend/<YYYYMM>/<slug>/`:
  - Email addresses → `user+<id>@example.test`
  - Phone numbers → `010-0000-0000`
  - national ID → `__REDACTED_ID__`
  - Student names → `student<id>`
  - School names → `school<id>`
  - Cookies / session IDs in `storageState.json` → leave structurally but replace any `Authorization` token value with a regenerated short-lived test token OR `__REDACTED__` with a note on how to re-obtain.
- `Authorization:` header values in `network.har` → replace with `Bearer __REDACTED__` after capture.
- Never commit real user credentials, even base64-encoded, even in `.sh` files.

## The Bottom Line

`/verify` proves the service accepts payloads locally.
`/qa-engineer` proves the **deployed system** — frontend + reverse proxy + CDN + real session flow + real SSE pipe — delivers the feature to a user's browser, and provides a **reusable spec suite that re-proves it on every future deploy**.

Until `qa.md` says `status: pass` (or `pass-with-flake` with an explicit accepted flake list), the deploy is not user-verified — regardless of how green `/verify` is.

The spec suite under `e2e/specs/` is the durable product. `qa.md` is the audit log. Together they turn "I tested it in my browser and it worked" — the most dangerous sentence in software — into a replayable artifact any operator can re-run after the next deploy.
