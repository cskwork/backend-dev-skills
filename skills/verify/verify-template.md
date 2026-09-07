# Verify Report Template

Companion to `SKILL.md` Phase 6. `verify.md` is the session-continuity document for verification — it must be readable alongside `explore.md` and `work.md` without any extra context.

Two audiences:

- §1 only — human prose recap (user-language primary).
- §2 onward — AI-optimized symbolic shorthand using the same legend as `explore.md` + this file's extensions.

Save to `<artifact-root>/.backend/<YYYYMM>/<slug>/verify.md`. For single-service changes, `<artifact-root>` is the service directory, not the monorepo root. The `harness/`, `fixtures/`, and `runs/` subfolders belong to the same ticket folder and are referenced from within `verify.md`.

---

## Symbol Legend (verify.md extensions)

Inherit the legend from `explore.md` (`../explore/report-template.md` §Symbol Legend). Adds:

```
Result   ✓ pass       ✗ fail       ⊘ skip        ⚠ flake/retry
Origin   U: user      S: saved     D: db-sample  X: synthesized
Status   pass | fail-escalated | fail-user-required | partial
Iter     run-1, run-2, ...         (each == one Phase 3 execution)
Cross    FG: Feign    KF: Kafka    SS: SSE       WS: WebSocket   SV: server
Escalate →W /work handoff        →E /explore handoff        →U user-required
```

---

## 1. Recap  *(prose — user-language primary, ≤6 lines, only section with prose allowed)*

> Plain language a non-developer can understand. What was verified, on which local service, which variants were run, what the outcome was, what the user needs to do next. No file paths, no curl snippets here — those belong below.

```
<example: "Verified POST /api/v1/users/login against the local example-api dev instance with happy,
boundary, and negative variants. Fixtures came from two MySQL sample rows and one user-provided
payload. After the second /work iteration, every assertion passed. Only deployment approval remains.">
```

---

## 2. Meta

```
kind: [B|F|R]
slug: <kebab-case-slug>
yyyymm: <YYYYMM>
status: pass | fail-escalated | fail-user-required | partial
iterations: <N>                    # how many Phase 3 runs were executed
done_on: <YYYY-MM-DD>
explore_ref: explore.md
work_ref: work.md
```

---

## 3. Verification Plan & Verifier Work Log

Record the approved Phase 0 plan and the verifier-subagent handoffs. Record which user instruction authorized the target and data scope, including authorization given earlier in the session.

```
plan_status: approved | changed-then-approved | waived
approved_by: user | explicit-waiver
target_env: local | dev | audit | stg
base_url: <scheme://host:port>
approval_note: <short quote or timestamp/order marker>

# endpoint/concern · agent · assigned artifacts · allowed commands · handoff result
POST /api/v1/users/login · verifier-1 · fixtures/post-api-v1-users-login.*.json, harness/post-api-v1-users-login.sh, runs/run-1.log · curl harness only · PASS 4/4 variants
FG ContentClient.getMeta · verifier-2 · harness/fg-content-getmeta.sh, runs/run-1.log · downstream curl only · PASS 1/1 probe
main · coordinator · verify.md + docs note · reviewed subagent logs/matrix · status=pass
```

If no subagent was used, record the execution choice:

```
subagent_waiver: <not needed, unavailable, or user-directed; retained field name for compatibility>
```

---

## 4. Endpoint Target List

Derived in Phase 0 from `explore.md §7` + `work.md §4`. Each target has a harness file.

```
# method path                                      service          auth   harness
POST   /api/v1/users/login                         example-api     none   harness/post-api-v1-users-login.sh
GET    /api/v1/content/{contentId}                 content-was    jwt    harness/get-api-v1-content-id.sh
FG     ContentClient.getMeta(contentId)        content-was    svc    harness/fg-content-getmeta.sh  # Feign downstream
```

If any target in `work.md §4` was deliberately **not** probed, list it here with a reason:

```
skipped: <method> <path>      reason=<"covered by integration test T — no HTTP surface change" | ...>
```

---

## 5. Payload Sources

One row per fixture file. Record origin, variant, and redaction.

```
# fixture path                                  origin  variant     row/source                     redacted=[fields]
fixtures/post-api-v1-users-login.happy.json     D       happy       lms.user WHERE org_id=X LIMIT 5 row_idx=2   [email,phone]
fixtures/post-api-v1-users-login.boundary.json  D       boundary    same query row_idx=4                        [email]
fixtures/post-api-v1-users-login.negative.json  X       negative    synth: missing 'password'                    —
fixtures/post-api-v1-users-login.regression.json U      regression  user-supplied 2026-04-20                    [email,phone]
```

Source codes: `U`=user, `S`=saved (prior /verify run), `D`=DB sample, `X`=synthesized.

For **D** (DB sample), cite the query shape and row index(es). Do not paste real values.
For **X**, cite the justification (`why=<rule violated>`).

---

## 6. Results Matrix

Verbatim copy of the final iteration's execution output. No summarization.

```
# endpoint                                variant     status  latency(ms)  assertions  result  log-ref
POST /api/v1/users/login                  happy       200     142          4/4         ✓       runs/run-2.log:14-58
POST /api/v1/users/login                  boundary    200     198          2/2         ✓       runs/run-2.log:60-104
POST /api/v1/users/login                  negative    400     89           2/2         ✓       runs/run-2.log:106-148
POST /api/v1/users/login                  regression  200     154          3/3         ✓       runs/run-2.log:150-194
FG   ContentClient.getMeta            downstream  200     71           2/2         ✓       runs/run-2.log:196-228
```

Any `⊘` row must include a reason in §10 `[GAP]`.
Any `⚠` row (flake/retry) must be explained in §8 Iteration Trail, not silently retried.

---

## 7. Cross-Service Checks

Match each item against `explore.md §8` and `work.md §7`. One row per cross-service edge.

```
# kind endpoint-or-channel                          probe-result                                            ref
FG   ContentClient.getMeta(contentId)           request shape accepted, response shape matches DTO      §5 row 5
FG   BoWasClient.getOrgSettings(orgId)              not probed — unchanged in work.md §7                    skip-reason
KF   topic=audit.user.login                         message published, consumer observed within 3s          runs/run-2.log:230-248
SS   channel=/topic/notify/{userId}                 not probed — no SSE change in this ticket               skip-reason
SV   Redis key 'session:<userId>'                   set on happy, ttl=1800s (expected)                      runs/run-2.log:250-262
```

Unprobed edges must say why — either "unchanged in work.md §7" or "`[GAP]` — see §10".

---

## 8. Iteration Trail

One row per `/work` re-invocation triggered from Phase 5. If no re-invocation (pass on iteration 1), write `—`.

```
# iter  trigger                                              delta                                                        outcome
1       first run                                            n/a                                                          FAIL-APP: regression variant 500 at LoginService.authenticate → verifyPassword null
→W      /work rework #1 with delta (see §7.1 below)          scope: example-api/src/.../LoginService.java:88-120         work.md §2 updated mid-flight=1
2       re-run after /work #1                                n/a                                                          PASS

# §7.1 delta-to-work  (iter 1 → iter 2)
endpoint:       POST /api/v1/users/login
variant:        regression
request:        fixtures/post-api-v1-users-login.regression.json
expected:       status=200, body.data.userId present, header X-Request-Id echoed
observed:       status=500, body.error.message="NullPointerException at LoginService:101"
trace-hint:     stderr "NPE at LoginService.verifyPassword(LoginService.java:101)"
likely-source:  example-api/src/main/java/.../LoginService.java:95-110  (password null-guard removed in /work commit abc123)
meaningful-progress: yes → iter 2 status changed 500→200
```

For each delta, record: endpoint, variant, fixture, expected, observed, trace hint, likely-source, and whether the next iteration made meaningful progress (per `SKILL.md §Phase 5 step 5`).

If escalated, show the escalation target:

```
# iter  trigger                                               outcome
3       re-run after /work #2                                 FAIL-APP same variant; no progress
→E      /explore rerun requested; user approval: pending      status: fail-user-required
```

---

## 9. Performance Observations

Record p95 (or max when N<5) latency per endpoint, per iteration. Compare against a baseline if one exists in an earlier `verify.md` or if `explore.md §5B.contract` declared a budget.

```
# endpoint                                baseline  iter-1 p95  iter-2 p95  delta   budget  result
POST /api/v1/users/login                  —         198ms       201ms       +3ms    <500ms  ✓
GET  /api/v1/content/{contentId}          71ms      89ms        81ms        +10ms   <200ms  ✓
```

If no budget exists, record the observation anyway — `verify.md` becomes the new baseline for the next run.

Never claim "performance unchanged" without a number. Either record it or declare `[GAP]`.

---

## 10. Open Items

```
[BLOCK]  <specific decision the user must make before status can move to pass>
                                             · needed-from: <P:PM | P:QA | infra>
[INFO]   <non-blocking observation worth recording>
[GAP]    <probe that was skipped with justification, or assertion that could not be written>
```

If empty: `—`

For **fail-escalated** or **fail-user-required**, there MUST be at least one `[BLOCK]`.

---

## 11. Re-entry Header

Three lines a future session needs to reload the verification state.

```
read: <artifact-root>/.backend/<YYYYMM>/<slug>/explore.md → work.md → verify.md
kind=<B|F|R>   slug=<slug>   yyyymm=<YYYYMM>   status=<status>   iterations=<N>
last-run=runs/run-<N>.log   next=<"ship" | "await user on §9 [BLOCK]" | "rerun /verify after /work iter N+1">
```

---

## Quality Bar

Before `/verify` exits, the following must all hold:

- [ ] `verify.md` exists at `<artifact-root>/.backend/<YYYYMM>/<slug>/verify.md`.
- [ ] §2 `status` is one of `pass | fail-escalated | fail-user-required | partial`.
- [ ] §3 records the approved verification plan and each verifier subagent handoff, or the recorded direct-execution choice.
- [ ] §6 matches `runs/run-<iterations>.log` byte-for-byte on the columns it cites.
- [ ] §7 covers every cross-service edge from `explore.md §8` — either probed, or skipped with a cited reason.
- [ ] §8 records every `/work` re-invocation, including the delta. Zero silent re-invocations.
- [ ] §9 has a number per endpoint per iteration — no "looks fine" entries.
- [ ] If status ≠ `pass`, §10 has at least one `[BLOCK]`.
- [ ] Every fixture committed under `fixtures/` is PII-redacted (spot-check 2-3 files).
- [ ] Every `runs/run-<N>.log` has `Authorization:` header values replaced with `__REDACTED__`.
- [ ] The docs note (`<artifact-root>/docs/features/<date>-<slug>.md` or `<artifact-root>/docs/bugs/...`) has a "Verification (/verify)" section appended — unless this was standalone mode.

If any box is unchecked, the skill is not done.

---

## Status State Machine

The skill can exit via exactly one of four states. No custom values.

```
                  ┌─────────────────────┐
                  │  Phase 3: Execute   │
                  └──────────┬──────────┘
                             ▼
                  ┌─────────────────────┐
                  │  Phase 4: Triage    │
                  └──────────┬──────────┘
              ┌──────────────┼──────────────┬───────────────────┐
              ▼              ▼              ▼                   ▼
         ┌────────┐    ┌─────────┐    ┌──────────┐        ┌──────────┐
         │  PASS  │    │ FAIL-APP│    │ FAIL-CTR │        │ FAIL-ENV │
         └───┬────┘    └────┬────┘    └────┬─────┘        └────┬─────┘
             │              ▼              ▼                   ▼
             │        iter < cap ?    immediate           status:
             │        user accept?    escalate             fail-user-required
             │              │              │
             │      yes┌────┴────┐no       ▼
             │         ▼         ▼   status:
             │    →W /work   status:  fail-escalated
             │    then       fail-user-
             │    Phase 3    required
             │         │
             │   progress? (Phase 5.5)
             │   ┌─────┴─────┐
             │   yes         no
             │   │           ▼
             │   ▼      after 2nd no-progress →
             │ next iter  status: fail-escalated
             ▼
        status: pass
             │
             ▼
     Phase 6: write verify.md
```

Reading this diagram is part of `SKILL.md §Phase 5`. The harness never invents a fifth status or partial-success branch.

---

## Worked Example (skeleton)

```markdown
---
status: pass
iterations: 2
done_on: 2026-04-21
explore_ref: explore.md
work_ref: work.md
---

# §1 Recap

Verified POST /api/v1/users/login against the local example-api dev instance with happy, boundary,
negative, and regression variants. All passed after two iterations. The first run exposed a 500 in
the regression variant, so /work ran once more; the second run restored the 200 response and met all
assertions. The downstream ContentClient.getMeta call passed in the same iteration. No blocker remains
except deployment approval.

# §2 Meta
kind: B
slug: fix-login-regression-after-pw-guard
yyyymm: 202604
status: pass
iterations: 2
done_on: 2026-04-21
explore_ref: explore.md
work_ref: work.md

# §3 Endpoint Target List
POST /api/v1/users/login          example-api   none  harness/post-api-v1-users-login.sh
FG   ContentClient.getMeta    content-was  svc   harness/fg-content-getmeta.sh

...
```

## Bottom Line

`verify.md` is the receipt. It proves the change was exercised, under which payloads, against which local service, with which outcome, across which iterations. If any section is vague, the receipt is forgeable — rewrite it with the actual evidence.
