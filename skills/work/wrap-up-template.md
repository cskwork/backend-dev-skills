# Wrap-Up Templates (Phase 5)

Two artifacts are written at the end of every `/work` run. They have different audiences and different styles — do not merge them.

| Artifact | Audience | Style | Path |
|---|---|---|---|
| `work.md` | Future AI session / future dev reloading context | Dense symbolic shorthand, same legend as `explore.md` | `.backend/<YYYYMM>/<slug>/work.md` |
| Change note | Stakeholders, non-developers, future human readers | Korean-first prose, plain language | `docs/features/<YYYY-MM-DD>-<slug>.md` (feature/refactor) **or** `docs/bugs/<YYYY-MM-DD>-<slug>.md` (bug fix) |

Both files must exist for `/work` to complete. Create parent folders if missing. Use `currentDate` from the session context for `<YYYY-MM-DD>` — do not guess.

---

## Template A — `work.md` (AI-optimized, symbolic)

Mirror the `explore.md` symbol legend. Prose banned outside §1. Every code claim cites `file:line`.

```markdown
---
name: <one-line title, same human title used in explore.md>
kind: <B|F|R>                 # Bug / Feature / Refactor
slug: <kebab-case-slug>
yyyymm: <YYYYMM>
explore_ref: explore.md
status: <done|partial|blocked>
done_on: <YYYY-MM-DD>
---

# §1 One-paragraph recap (prose OK — only section where prose is allowed)

<4-6 lines, plain language: what was implemented, which services were touched, what the verification confirmed, any follow-ups carried forward. No file paths here; save those for the sections below.>

# §2 Contract Adherence
plan_source: explore.md §7
rows_in_plan: <N>
rows_implemented: <N>
rows_skipped: <N>  reason=<...>  (empty if zero)
rows_added_mid_flight: <N>  reason=<...>  (should be zero — each entry is a deviation that needs a why)

# §3 Reuse Re-check Result
# one line per row in explore.md §7, format:
#   §7.rowN  act=<≡|≈|⊕|modify-shared>  verb/noun=<terms>  decision=<kept|switched-to-X>  why=<short>
- §7.row_1  act=⊕  verb=format  noun=JwtClaims  searched={formatJwt,claimsToString}  decision=kept  why=no-match
- §7.row_2  act=≈  callers-now=4  safe=yes  decision=kept  why=added-optional-param
- §7.row_3  act=modify-shared  callers-now=11  safe=no  decision=switched-to-add-sibling  why=caller #7 depends on old behavior

# §4 Changes (file-level)
# one row per edited file, format: file_path · act · why · reuse-or-new
<path/to/File.java:120-145> · ✎ · implements §7.row_1 · reuse JwtUtil.format()
<path/to/NewController.java> · ⊕ · implements §7.row_2 · new (no analog in service)
<path/to/Mapper.xml:88-97> · ✎ · implements §7.row_3 · extend existing resultMap
...

# §5 Tests Written / Updated
# test path · kind (unit/integration/controller-slice/component/e2e) · what it pins · red→green verified? (Y/N)
<test/path/FooServiceTest.java:40-88> · unit · retries exhaust after 3 attempts · Y
<test/path/BarMapperIT.java:22-60> · integration · column mapping for is_deleted · Y
...

# §6 Verification Evidence (fresh output only)
# command · exit · summary line · timestamp-or-ordering
cd services/user-api && ./gradlew test --tests 'com.x.FooServiceTest'  · 0 · 4/4 pass · run 1
cd apps/web && npm run lint                                             · 0 · 0 errors / 0 warnings · run 2
cd apps/web && npm run type-check                                       · 0 · 0 errors · run 3
# for bug fixes, include the red-green revert cycle:
revert fix + re-run FooServiceTest                                 · 1 · 1/4 fail (expected) · run 4
re-apply fix + re-run FooServiceTest                               · 0 · 4/4 pass · run 5

# §7 Cross-Service Impact Realized
# If §8 of explore.md predicted impact, report actual outcome
http-client: <client.method sig unchanged | new method added | existing extended with optional field>
queue:       <no topic/queue touched | new consumer registered | payload field added backwards-compat>
cache:       <no keyspace touched | new key prefix <name> added | ttl set to <N>>
sse/ws:      <no channel touched | <channel> added>
config:      <no property added | encrypted prop added to application-<profile>.yml under <key>>
migration:   <no schema change | migration <id> added | data-backfill script <path>>

# §8 Scope Integrity
diff_scope_matches_plan: <yes|no>
drive_bys_removed: <list or "none">
follow_ups_deferred: <list of file:line · note — these go into a later ticket, not this PR>

# §9 Open Items / Carry-forward
[INFO]  <non-blocking observation worth recording>
[GAP]   <gap in tests or docs accepted for now, with justification>
[BLOCK] <only if status=blocked — what the user must decide>

# §10 Re-entry Header (how a future session reloads context)
read: .backend/<YYYYMM>/<slug>/explore.md → .backend/<YYYYMM>/<slug>/work.md
kind=<B|F|R>   slug=<slug>   yyyymm=<YYYYMM>   status=<status>
services=<user-api, web-app, ...>
```

Ordering inside each section: follow the order of `explore.md §7` so a diff between `explore.md` and `work.md` is trivial.

---

## Template B — Change note under `docs/features/` or `docs/bugs/`

Choose the folder by kind:

- `docs/features/<YYYY-MM-DD>-<slug>.md` → Feature or Refactor
- `docs/bugs/<YYYY-MM-DD>-<slug>.md` → Bug fix

If either folder does not exist, create it before writing. If a file with the same name exists (same slug shipped earlier today), append `-part2`, `-part3`, … — do not overwrite.

Prose-first, Korean primary, plain language. Cite code only where the reader would realistically need to jump (endpoint path, config key, migration id). No `file:line` spam — that's what `work.md` is for.

```markdown
---
title: <짧은 한 줄 제목 — 비개발자도 이해할 수 있게>
kind: <feature|refactor|bug>
slug: <kebab-case-slug>
date: <YYYY-MM-DD>
services: [<user-api>, <web-app>, ...]
explore_ref: .backend/<YYYYMM>/<slug>/explore.md
work_ref: .backend/<YYYYMM>/<slug>/work.md
---

# <동일한 한 줄 제목>

## 한 줄 요약

<한 문장. 무엇을 했고, 누가 영향을 받는지.>

## 배경

<2-4줄. 이 작업이 왜 필요했는지. 사용자 요청, 장애, 법적/업무적 제약 중 어떤 것이었는지. 기술 용어 최소화.>

## 무엇이 바뀌었나

<3-6줄. 사용자가 체감하는 변화 또는 내부 동작의 변화를 자연어로 설명. 엔드포인트 경로·화면 위치·설정 키 같은 "독자가 실제로 확인하러 갈 곳"만 인라인으로 언급.>

- 영향 서비스: <user-api>, <web-app>
- 외부 인터페이스 변화: <없음 | POST /api/v1/... 추가 | 기존 응답에 nullable 필드 `xxx` 추가(하위호환)>
- 설정/배포 변화: <없음 | `application-<profile>.yml` 에 `xxx.yyy` 키 추가(암호화 필요) | Helm values 수정 필요>
- 데이터 변화: <없음 | 마이그레이션 `V2026_04_21__add_xxx.sql` 추가 | 기존 컬럼 의미 변경 없음>

## 버그 수정의 경우 — 원인과 해결

(kind=bug 일 때만 작성)

- **증상:** <관찰된 현상 한 줄>
- **근본 원인:** <한 문장. 어디의 어떤 규칙이 어떤 조건에서 잘못 동작했는지.>
- **해결 방식:** <한 문장. 어떤 지점에 어떤 조건을 추가/수정했는지. 증상 지점이 아닌 근본 원인 지점에서 고쳤는지.>
- **재발 방지:** <추가된 회귀 테스트가 무엇을 고정하는지 한 줄.>

## 검증

<3-5줄 또는 불릿. 어떤 테스트·빌드·수동 검증을 돌렸고 결과가 무엇이었는지. 숫자와 결과를 단문으로.>

- 단위/통합 테스트: <pass 수 / total>
- 린트·타입체크: <pass>
- 수동 검증: <있었으면 시나리오 1-2줄, 없었으면 "해당 없음">
- 회귀 사이클(버그일 때): <revert → fail 확인 → re-apply → pass 확인 완료>

## 알려진 한계 / 후속 과제

<bullet. 이번 변경에 포함되지 않은 것 중 독자가 알아야 할 것. 미정이면 "없음".>

## 관련 문서

- 탐색 리포트: `.backend/<YYYYMM>/<slug>/explore.md`
- 작업 로그(AI용): `.backend/<YYYYMM>/<slug>/work.md`
- <관련 Jira 티켓이 있으면 키만 기록, 외부 링크 금지>
```

---

## Quality Bar for Both Artifacts

Before `/work` exits:

- [ ] `work.md` exists and every `explore.md §7` row has a matching row in `work.md §4` **or** is listed in `§2 Contract Adherence` as skipped with a reason.
- [ ] `work.md §6` has actual command output — not paraphrased "looks good".
- [ ] Change note exists under the correct folder (`features` vs `bugs`) with today's date.
- [ ] Change note's "한 줄 요약" is readable by a non-developer — run it past your internal "non-dev reader" filter.
- [ ] Both files reference each other (`work_ref` in change note, `explore_ref` in `work.md`).
- [ ] No PR/commit has been created unless the user explicitly asked — `/work` stops at the file system.
- [ ] `/verify` is noted as the next step in the final chat summary (e.g. "다음 단계: /verify 로 HTTP-레벨 검증 실행"), unless the change has no backend HTTP surface.

If any box is unchecked, the skill is not done.
