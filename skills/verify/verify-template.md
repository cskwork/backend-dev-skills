# Verify Report Template

Companion to `SKILL.md` Phase 6. `verify.md` is the session-continuity document for verification — it must be readable alongside `explore.md` and `work.md` without any extra context.

Two audiences:

- §1 only — human prose recap (Korean primary).
- §2 onward — AI-optimized symbolic shorthand using the same legend as `explore.md` + this file's extensions.

Save to `.backend/<YYYYMM>/<slug>/verify.md`. The `harness/`, `fixtures/`, and `runs/` subfolders belong to the same ticket folder and are referenced from within `verify.md`.

---

## Symbol Legend (verify.md extensions)

Inherit the legend from `explore.md` (`../explore/report-template.md` §Symbol Legend). Adds:

```
Result   ✓ pass       ✗ fail       ⊘ skip        ⚠ flake/retry
Origin   U: user      S: saved     D: db-sample  X: synthesized
Status   pass | fail-escalated | fail-user-required | partial
Iter     run-1, run-2, ...         (each == one Phase 3 execution)
Cross    HC: http-client   MQ: message-queue   SS: SSE       WS: WebSocket   SV: server
Escalate →W /work handoff        →E /explore handoff        →U user-required
```

---

## 1. Recap  *(prose — Korean primary, ≤6 lines, only section with prose allowed)*

> Plain language, 비개발자도 이해할 수 있게. What was verified, on which local service, which variants were run, what the outcome was, what the user needs to do next. No file paths, no curl snippets here — those belong below.

```
<예: "user-api 로컬 인스턴스(dev 프로파일, 포트 8080)에 대해 POST /api/v1/auth/login
엔드포인트를 happy · boundary · negative 3개 변형으로 검증했습니다. DB 샘플 행 2건과
사용자 제공 페이로드 1건으로 픽스처를 구성했고, 2차 /work 반복 후 모든 assertion이 통과했습니다.
운영 배포 전 결재만 남았습니다.">
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

## 3. Endpoint Target List

Derived in Phase 0 from `explore.md §7` + `work.md §4`. Each target has a harness file.

```
# method path                                      service          auth   harness
POST   /api/v1/auth/login                          user-api         none   harness/post-api-v1-auth-login.sh
GET    /api/v1/items/{itemId}                      item-api         jwt    harness/get-api-v1-items-id.sh
HC     ItemClient.getMeta(itemId)                  item-api         svc    harness/hc-item-getmeta.sh  # HTTP-client downstream
```

If any target in `work.md §4` was deliberately **not** probed, list it here with a reason:

```
skipped: <method> <path>      reason=<"covered by integration test T — no HTTP surface change" | ...>
```

---

## 4. Payload Sources

One row per fixture file. Record origin, variant, and redaction.

```
# fixture path                                  origin  variant     row/source                     redacted=[fields]
fixtures/post-api-v1-auth-login.happy.json      D       happy       app_user WHERE tenant_id=X LIMIT 5 row_idx=2   [email,phone]
fixtures/post-api-v1-auth-login.boundary.json   D       boundary    same query row_idx=4                           [email]
fixtures/post-api-v1-auth-login.negative.json   X       negative    synth: missing 'password'                       —
fixtures/post-api-v1-auth-login.regression.json U      regression  user-supplied 2026-04-20                        [email,phone]
```

Source codes: `U`=user, `S`=saved (prior /verify run), `D`=DB sample, `X`=synthesized.

For **D** (DB sample), cite the query shape and row index(es). Do not paste real values.
For **X**, cite the justification (`why=<rule violated>`).

---

## 5. Results Matrix

Verbatim copy of the final iteration's execution output. No summarization.

```
# endpoint                                variant     status  latency(ms)  assertions  result  log-ref
POST /api/v1/auth/login                   happy       200     142          4/4         ✓       runs/run-2.log:14-58
POST /api/v1/auth/login                   boundary    200     198          2/2         ✓       runs/run-2.log:60-104
POST /api/v1/auth/login                   negative    400     89           2/2         ✓       runs/run-2.log:106-148
POST /api/v1/auth/login                   regression  200     154          3/3         ✓       runs/run-2.log:150-194
HC   ItemClient.getMeta                   downstream  200     71           2/2         ✓       runs/run-2.log:196-228
```

Any `⊘` row must include a reason in §9 `[GAP]`.
Any `⚠` row (flake/retry) must be explained in §7 Iteration Trail, not silently retried.

---

## 6. Cross-Service Checks

Match each item against `explore.md §8` and `work.md §7`. One row per cross-service edge.

```
# kind endpoint-or-channel                          probe-result                                            ref
HC   ItemClient.getMeta(itemId)                     request shape accepted, response shape matches DTO      §5 row 5
HC   SettingsClient.getOrgSettings(orgId)           not probed — unchanged in work.md §7                    skip-reason
MQ   topic=audit.user.login                         message published, consumer observed within 3s          runs/run-2.log:230-248
SS   channel=/topic/notify/{userId}                 not probed — no SSE change in this ticket               skip-reason
SV   Redis key 'session:<userId>'                   set on happy, ttl=1800s (expected)                      runs/run-2.log:250-262
```

Unprobed edges must say why — either "unchanged in work.md §7" or "`[GAP]` — see §9".

---

## 7. Iteration Trail

One row per `/work` re-invocation triggered from Phase 5. If no re-invocation (pass on iteration 1), write `—`.

```
# iter  trigger                                              delta                                                        outcome
1       first run                                            n/a                                                          FAIL-APP: regression variant 500 at LoginService.authenticate → verifyPassword null
→W      /work rework #1 with delta (see §7.1 below)          scope: services/user-api/src/.../LoginService.java:88-120    work.md §2 updated mid-flight=1
2       re-run after /work #1                                n/a                                                          PASS

# §7.1 delta-to-work  (iter 1 → iter 2)
endpoint:       POST /api/v1/auth/login
variant:        regression
request:        fixtures/post-api-v1-auth-login.regression.json
expected:       status=200, body.data.userId present, header X-Request-Id echoed
observed:       status=500, body.error.message="NullPointerException at LoginService:101"
trace-hint:     stderr "NPE at LoginService.verifyPassword(LoginService.java:101)"
likely-source:  services/user-api/src/main/java/.../LoginService.java:95-110  (password null-guard removed in /work commit abc123)
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

## 8. Performance Observations

Record p95 (or max when N<5) latency per endpoint, per iteration. Compare against a baseline if one exists in an earlier `verify.md` or if `explore.md §5B.contract` declared a budget.

```
# endpoint                                baseline  iter-1 p95  iter-2 p95  delta   budget  result
POST /api/v1/auth/login                   —         198ms       201ms       +3ms    <500ms  ✓
GET  /api/v1/items/{itemId}               71ms      89ms        81ms        +10ms   <200ms  ✓
```

If no budget exists, record the observation anyway — `verify.md` becomes the new baseline for the next run.

Never claim "performance unchanged" without a number. Either record it or declare `[GAP]`.

---

## 9. Open Items

```
[BLOCK]  <specific decision the user must make before status can move to pass>
                                             · needed-from: <P:PM | P:QA | infra>
[INFO]   <non-blocking observation worth recording>
[GAP]    <probe that was skipped with justification, or assertion that could not be written>
```

If empty: `—`

For **fail-escalated** or **fail-user-required**, there MUST be at least one `[BLOCK]`.

---

## 10. Re-entry Header

Three lines a future session needs to reload the verification state.

```
read: .backend/<YYYYMM>/<slug>/explore.md → work.md → verify.md
kind=<B|F|R>   slug=<slug>   yyyymm=<YYYYMM>   status=<status>   iterations=<N>
last-run=runs/run-<N>.log   next=<"ship" | "await user on §9 [BLOCK]" | "rerun /verify after /work iter N+1">
```

---

## Quality Bar

Before `/verify` exits, the following must all hold:

- [ ] `verify.md` exists at `.backend/<YYYYMM>/<slug>/verify.md`.
- [ ] §2 `status` is one of `pass | fail-escalated | fail-user-required | partial`.
- [ ] §5 matches `runs/run-<iterations>.log` byte-for-byte on the columns it cites.
- [ ] §6 covers every cross-service edge from `explore.md §8` — either probed, or skipped with a cited reason.
- [ ] §7 records every `/work` re-invocation, including the delta. Zero silent re-invocations.
- [ ] §8 has a number per endpoint per iteration — no "looks fine" entries.
- [ ] If status ≠ `pass`, §9 has at least one `[BLOCK]`.
- [ ] Every fixture committed under `fixtures/` is PII-redacted (spot-check 2-3 files).
- [ ] Every `runs/run-<N>.log` has `Authorization:` header values replaced with `__REDACTED__`.
- [ ] The docs note (`docs/features/<date>-<slug>.md` or `docs/bugs/...`) has a "검증(/verify)" section appended — unless this was standalone mode.

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

user-api 로컬 인스턴스(dev 프로파일, 포트 8080)에서 POST /api/v1/auth/login 의 happy · boundary · negative · regression 4개 변형을
총 2회 반복 실행하여 모두 통과했습니다. 1차 반복에서 regression 변형이 500으로 실패하여 /work 를 1회 재실행했고, 2차 반복에서
응답이 200으로 회복되어 모든 assertion 을 충족했습니다. 하위 HTTP 호출 ItemClient.getMeta 도 동일 반복 내에서 함께 통과했습니다.
배포 결재 외에 남은 블로커는 없습니다.

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
POST /api/v1/auth/login           user-api       none  harness/post-api-v1-auth-login.sh
HC   ItemClient.getMeta           item-api       svc   harness/hc-item-getmeta.sh

...
```

## Bottom Line

`verify.md` is the receipt. It proves the change was exercised, under which payloads, against which local service, with which outcome, across which iterations. If any section is vague, the receipt is forgeable — rewrite it with the actual evidence.
