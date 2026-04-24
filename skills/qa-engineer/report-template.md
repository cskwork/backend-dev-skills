# qa.md Template

> Phase-6 companion to `SKILL.md`. Defines the exact output format for `.backend/<YYYYMM>/<slug>/qa.md`. Mirrors the symbolic density of `explore.md` and `verify.md` — §1 is the only prose section (Korean plain-language), §2-11 are AI-optimized symbolic.

## Legend

Reuse `explore.md` / `verify.md` legend for consistency, with QA-specific additions.

| Symbol | Meaning |
|---|---|
| `✓` | Variant passed (3-attempt clean) |
| `⚠` | Variant passed but flaky (mixed outcomes within 3 attempts) |
| `✗` | Variant failed all 3 attempts |
| `⊘` | Variant skipped (marked `.skip` in spec — reason in §9) |
| `◐` | Variant partial (some assertions pass, some fail within the same test) |
| `[BLOCK]` | Cannot proceed — env / deploy / credentials issue |
| `[INFO]` | Noteworthy observation, not blocking |
| `[GAP]` | Coverage gap: endpoint in `verify.md` has no UI exercise |
| `[FLAKE]` | Intermittent failure, needs root-cause |
| `[PERF]` | Performance observation (slow smoke, slow LB, slow first-load) |
| `[REMOVAL]` | Spec marked `.skip` or deleted per user consent |
| `[PROD-WRITE]` | A write to prod was performed, logged with reason + reviewer |
| `U/S/D/X` | Fixture origin: User / Saved / DB / Synthesized |
| `→W/→V/→E/→U` | Escalation target: /work / /verify / /explore / User |

## Template

```markdown
---
kind: qa
slug: <slug>
yyyymm: <YYYYMM>
env: <dev|stg|audit|prod>
run_kind: <first-run | update-run>
status: <pass | pass-with-flake | fail-escalated | fail-user-required | blocked-env>
iteration_count: <N>
run_timestamp: <YYYY-MM-DD HH:MM KST>
explore_ref: .backend/<YYYYMM>/<slug>/explore.md
work_ref: .backend/<YYYYMM>/<slug>/work.md
verify_ref: .backend/<YYYYMM>/<slug>/verify.md
smoke_rerun: cd .backend/<YYYYMM>/<slug>/e2e && QA_ENV=<env> npx playwright test specs/smoke.spec.ts
---

## §1 Recap (Korean prose)

<2-5 문장의 한국어 서술. 배포된 환경(<env>)에서 어떤 기능을 실제 브라우저로 검증했는지,
어떤 변형(variants)이 통과했고 실패했는지, 후속 조치가 무엇인지, 재실행 명령은 무엇인지.
스테이크홀더(PM/QA 리드/릴리스 매니저)가 이 단락만 읽고 의사결정할 수 있도록 작성.>

## §2 Meta

| field | value |
|---|---|
| kind | Feature \| Bug \| Refactor |
| slug | <slug> |
| yyyymm | <YYYYMM> |
| env | <env> |
| run-kind | first-run \| update-run (Nth) |
| status | <status> |
| upstream reinvocations | <count> of 2 max |
| run-timestamp | <YYYY-MM-DD HH:MM KST> |
| suite size | <N specs>, <M variants total>, smoke <K variants> |
| smoke duration | <seconds> |

## §3 UI Flow Target List

| # | Flow | Frontend service | Route | Entry action | Expected end-state | Spec file |
|---|---|---|---|---|---|---|
| 1 | login-sso | web-app | `/login` | SSO click → provider | dashboard heading "Dashboard" visible | `specs/login-sso.spec.ts` |
| 2 | create-item | web-app | `/projects/:id` | click "Save" button | item count +1 | `specs/create-item.spec.ts` |
| 3 | realtime-room-ws | web-app | `/rooms/:id/live` | peer-join via 2nd context | peer-count = 2 | `specs/realtime-room.spec.ts` |
| 4 | broadcast-sse | web-app | `/dashboard` | API-triggered broadcast | toast "New notification" visible | `specs/broadcast.spec.ts` |

Endpoints from `verify.md §5` without a UI caller (gap):
- `POST /api/internal/sync` — internal HTTP-client-only, no UI → [INFO] (not a gap)
- `GET /api/users/{id}/history` — has UI caller in admin-web but not in scope this ticket → [GAP]

## §4 Environment & Auth

| field | value |
|---|---|
| BASE_URL | `https://dev-app.example.com` |
| QA_ENV | dev |
| storageState | `e2e/storageState.dev.json` (redacted, refreshed at <HH:MM>) |
| auth path | dev-profile test-JWT via `/test/generate-jwt?userId=<id>` (see verify/jwt-auth-reference.md) |
| prod-write flag | N/A |
| tenant IDs | `fixtures/env-tenants.ts §dev` (orgId 1001, adminId test-admin-001) |
| PII redactions | email→user+<id>@example.test, names→user<id>, Authorization→Bearer __REDACTED__ |
| playwright version | <output of `npx playwright --version`> |
| playwright-cli version | <output of `npx playwright-cli --version`> |

## §5 Results Matrix (verbatim from last execution)

```
flow                 variant      env   status  duration(ms)  assertions  attempts  notes
login-sso            happy        dev   ✓       3242          5/5         1/3       —
login-sso            negative     dev   ✓       1410          2/2         1/3       toast shown
create-item          happy        dev   ✓       4180          4/4         1/3       —
create-item          boundary     dev   ✓       4112          3/3         1/3       100-char name
create-item          negative     dev   ✓       1230          2/2         1/3       403 shown correctly
realtime-room-ws     happy        dev   ⚠       7820          4/4         2/3       flake: WS frame arrived 1200ms late on attempt 1
broadcast-sse        happy        dev   ✓       2940          3/3         1/3       —
regression-bug-4321  repro        dev   ✓       5220          4/4         1/3       bug did not recur

SMOKE SUBSET (always trace/video):
login-sso            happy-smoke  dev   ✓       3210          5/5         1/3       smoke
create-item          happy-smoke  dev   ✓       4050          4/4         1/3       smoke
broadcast-sse        happy-smoke  dev   ✓       2880          3/3         1/3       smoke
total smoke:                                     10140 (10.1s)
```

## §6 Spec Suite Changes

| Spec file | Change | Reason | Lines touched |
|---|---|---|---|
| `specs/login-sso.spec.ts` | preserved | contract unchanged | 0 |
| `specs/create-item.spec.ts` | modified | button label changed in work.md §4 row 2 | L23 (locator text) |
| `specs/realtime-room.spec.ts` | added | new WS flow in work.md §4 row 3 | +78 lines |
| `specs/broadcast.spec.ts` | preserved | contract unchanged | 0 |
| `specs/smoke.spec.ts` | modified | added happy-smoke for realtime-room | +12 lines |
| `specs/regression-bug-4321.spec.ts` | added | bug fix in this ticket | +45 lines |

`git diff --stat e2e/specs/` (paste verbatim if available):
```
<diffstat output>
```

## §7 Iteration Trail

One row per upstream re-invocation. Empty if no rework occurred.

| # | Failing variant | Classification | Delta target | Delta summary | Outcome |
|---|---|---|---|---|---|
| — | none | — | — | — | — |

Example if rework occurred:
| 1 | create-item/happy | FAIL-UI | →W (work) | locator getByRole('button',{name:'Add member'}) not found; likely-source apps/web/src/components/ProjectDetail.vue:142 | work.md §4 row "2026-05-03 btn label" added; redeploy; re-run passed |
| 2 | broadcast-sse/happy | FAIL-CONTRACT | →E (explore) | response envelope `.data.topic` was expected per explore.md §5B but deployed sends `.data.channel` | escalated; /explore to update contract |

## §8 Evidence Index

| Run folder | Env | Purpose | Artifacts |
|---|---|---|---|
| `artifacts/run-20260421-1430-dev/` | dev | full suite first pass | trace.zip, video.webm (realtime-room only), 14 screenshots, console.log, network.har |
| `artifacts/run-20260421-1445-dev/` | dev | realtime-room re-run post-fix | trace.zip, video.webm, console.log |
| `artifacts/run-20260421-1612-stg/` | stg | smoke on stg | trace.zip, video.webm |

Trace viewer command:
```
npx playwright show-trace .backend/<YYYYMM>/<slug>/e2e/artifacts/run-20260421-1430-dev/trace.zip
```

## §9 Open Items

| Tag | Spec | Description | Owner / Next |
|---|---|---|---|
| [FLAKE] | realtime-room-ws/happy | WS frame arrives 1200ms late on ~30% of dev runs | Observed under dev-only. Retest on stg. Do not mark pass without 3 clean consecutive dev runs. |
| [GAP] | history-view | `GET /api/users/{id}/history` has UI caller in admin-web, not exercised this ticket | Track as future ticket; not blocking. |
| [PERF] | smoke suite | 10.1s total, well under 90s target | [INFO] only. |
| [INFO] | prod run | not attempted this run; user has not requested prod QA for this ticket yet | — |

## §10 Re-entry Header

How a future session (or the same session after compaction) re-enters this QA:

1. Read this file (`qa.md`) — §1 recap + §2 meta for status.
2. Read `explore.md`, `work.md`, `verify.md` in the same ticket folder for context.
3. Re-run smoke: `cd <ticket>/e2e && QA_ENV=<env> npx playwright test specs/smoke.spec.ts`.
4. If the smoke passes, the feature is still verified-as-deployed. If it fails, the result goes to §11 History and this file is re-opened for update-run.
5. Before running in a new env, confirm env choice with user and update `_env.sh` + `storageState.<env>.json`.
6. For prod, require explicit consent per `environment-gates.md §Prod Run Guardrails`.

## §11 History

One line per prior `/qa-engineer` run on this ticket. Grows with each invocation.

```
2026-04-21 14:30 dev  pass             — first-run, all 8 variants pass, 1 flake on realtime-room
2026-04-21 16:12 stg  pass             — smoke only, 3 variants pass on stg
2026-05-03 10:05 dev  pass-with-flake  — update-run after work.md button label change, realtime-room flake recurs
2026-05-03 14:20 stg  pass             — smoke post-redeploy
2026-05-05 09:00 prod pass             — smoke read-only, 3 variants pass on prod
```

Detail for historical runs lives in `artifacts/run-*` folders; §11 is the index.
```

## Writing Discipline

- **§1 Recap**: Korean prose, 2-5 sentences, no symbols, readable by a non-engineer stakeholder. This is the only section rendered to humans before they dive into the suite.
- **§2-11**: Symbolic, dense, AI-optimized. Tables use the legend above. No prose padding between table rows.
- **Every claim cites**: spec line OR run artifact path OR `verify.md §5` row OR `work.md §4` row. A §5 matrix row with no notes and no traceable context is a violation.
- **On update-run**: §11 History grows. §3-10 describe the **latest** state; prior state is in prior run artifacts. Don't maintain per-run §5 inside this file — that's what `artifacts/result.json` is for.
- **Never paraphrase the result matrix.** Copy/paste the table verbatim from Phase 3. If the Playwright JSON reporter shape differs, transform once with a script, not by retyping.

## Appendix: docs Entry Append

When the ticket has a `docs/features/<YYYY-MM-DD>-<slug>.md` or `docs/bugs/...md` from `/work`, append a "배포 QA(/qa-engineer)" section:

```markdown
## 배포 QA(/qa-engineer)

- **검증 환경**: <env> (`<BASE_URL>`)
- **검증 일시**: <YYYY-MM-DD HH:MM KST>
- **상태**: <pass | pass-with-flake | fail-escalated>
- **테스트 범위**: <N개 플로우, M개 변형, 스모크 K개>
- **증거**: `.backend/<YYYYMM>/<slug>/qa.md`, 트레이스 `.backend/.../e2e/artifacts/run-<ts>-<env>/trace.zip`
- **재실행**: `cd .backend/<YYYYMM>/<slug>/e2e && QA_ENV=<env> npx playwright test specs/smoke.spec.ts`
- **미해결 항목**: <§9 [BLOCK]/[FLAKE] 요약; 없으면 "없음">
```

Do not create a new docs entry if one does not already exist — that is `/work`'s responsibility. Standalone-mode `/qa-engineer` runs do not append to docs.
