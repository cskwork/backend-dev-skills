# Wrap-Up Templates (Phase 5)

Two artifacts are written at the end of every `/work` run. They have different audiences and different styles — do not merge them.

| Artifact | Audience | Style | Path |
|---|---|---|---|
| `work.md` | Future AI session / future dev reloading context | Dense symbolic shorthand, same legend as `explore.md` | `.backend/<YYYYMM>/<slug>/work.md` |
| Change note | Stakeholders, non-developers, future human readers | plain-language prose, plain language | `docs/features/<YYYY-MM-DD>-<slug>.md` (feature/refactor) **or** `docs/bugs/<YYYY-MM-DD>-<slug>.md` (bug fix) |

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

# §4 Agent Work Log
# one row per agent, including main-agent integration edits:
# agent · role · assigned_scope · files_changed · verification · handoff_summary
main · coordinator/integrator · contract+review+docs · <files or none> · <commands or reviewed agent output> · <summary>
agent-1 · implementation · §7.row_1 · <path/to/File.java:120-145> · <command exit/result> · <summary>
agent-2 · verifier · final diff + §9 checks · <none or files> · <command exit/result> · <risk summary>

# §5 Changes (file-level)
# one row per edited file, format: file_path · act · why · reuse-or-new
<path/to/File.java:120-145> · ✎ · implements §7.row_1 · reuse JwtUtil.format()
<path/to/NewController.java> · ⊕ · implements §7.row_2 · new (no analog in service)
<path/to/Mapper.xml:88-97> · ✎ · implements §7.row_3 · extend existing resultMap
...

# §6 Tests Written / Updated
# test path · kind (unit/integration/controller-slice/component/e2e) · what it pins · red→green verified? (Y/N)
<test/path/FooServiceTest.java:40-88> · unit · retries exhaust after 3 attempts · Y
<test/path/BarMapperIT.java:22-60> · integration · column mapping for is_deleted · Y
...

# §7 Verification Evidence (fresh output only)
# command · exit · summary line · timestamp-or-ordering
cd example-api && ./gradlew test --tests 'com.x.FooServiceTest'  · 0 · 4/4 pass · run 1
cd frontend-web && npm run lint                                    · 0 · 0 errors / 0 warnings · run 2
cd frontend-web && npm run type-check                              · 0 · 0 errors · run 3
# for bug fixes, include the red-green revert cycle:
revert fix + re-run FooServiceTest                                 · 1 · 1/4 fail (expected) · run 4
re-apply fix + re-run FooServiceTest                               · 0 · 4/4 pass · run 5

# §8 Cross-Service Impact Realized
# If §8 of explore.md predicted impact, report actual outcome
feign: <client.method sig unchanged | new method added | existing extended with optional field>
kafka: <no topic touched | new consumer registered | payload field added backwards-compat>
redis: <no keyspace touched | new key prefix <name> added | ttl set to <N>>
sse/ws: <no channel touched | <channel> added>
config: <no property added | ENC(...) added to application-<profile>.yml under <key>>
migration: <no schema change | migration <id> added | data-backfill script <path>>

# §9 Scope Integrity
diff_scope_matches_plan: <yes|no>
drive_bys_removed: <list or "none">
follow_ups_deferred: <list of file:line · note — these go into a later ticket, not this PR>

# §10 Open Items / Carry-forward
[INFO]  <non-blocking observation worth recording>
[GAP]   <gap in tests or docs accepted for now, with justification>
[BLOCK] <only if status=blocked — what the user must decide>

# §11 Re-entry Header (how a future session reloads context)
read: .backend/<YYYYMM>/<slug>/explore.md → .backend/<YYYYMM>/<slug>/work.md
kind=<B|F|R>   slug=<slug>   yyyymm=<YYYYMM>   status=<status>
services=<example-api, frontend-web, ...>
```

Ordering inside each section: follow the order of `explore.md §7` so a diff between `explore.md` and `work.md` is trivial.

---

## Template B — Change note under `docs/features/` or `docs/bugs/`

Choose the folder by kind:

- `docs/features/<YYYY-MM-DD>-<slug>.md` → Feature or Refactor
- `docs/bugs/<YYYY-MM-DD>-<slug>.md` → Bug fix

If either folder does not exist, create it before writing. If a file with the same name exists (same slug shipped earlier today), append `-part2`, `-part3`, … — do not overwrite.

Prose-first, user-language primary, plain language. Cite code only where the reader would realistically need to jump (endpoint path, config key, migration id). No `file:line` spam — that's what `work.md` is for.

```markdown
---
title: <short one-line title a non-developer can understand>
kind: <feature|refactor|bug>
slug: <kebab-case-slug>
date: <YYYY-MM-DD>
services: [<example-api>, <frontend-web>, ...]
explore_ref: .backend/<YYYYMM>/<slug>/explore.md
work_ref: .backend/<YYYYMM>/<slug>/work.md
---

# <same one-line title>

## One-Line Summary

<One sentence: what changed and who is affected.>

## Background

<2-4 lines explaining why this work was needed. Mention user request, incident, legal constraint, or business constraint. Minimize jargon.>

## What Changed

<3-6 lines in natural language. Describe user-visible change or internal behavior change. Inline only paths that readers may realistically inspect, such as endpoint paths, screen locations, or config keys.>

- Affected services: <example-api>, <frontend-web>
- External interface change: <none | added POST /api/v1/... | added nullable `xxx` response field, backward-compatible>
- Config/deploy change: <none | added `xxx.yyy` in `application-<profile>.yml` | Helm values update required>
- Data change: <none | added migration `V2026_04_21__add_xxx.sql` | no semantic change to existing columns>

## Bug Fix Only: Cause and Fix

(Write only when kind=bug.)

- **Symptom:** <one-line observed behavior>
- **Root cause:** <one sentence: which rule failed under which condition>
- **Fix:** <one sentence: what condition or logic changed, and whether it was fixed at the root cause rather than the symptom site>
- **Regression guard:** <one line explaining what the added regression test pins>

## Verification

<3-5 lines or bullets. State which tests/build/manual checks ran and what the results were. Include numbers where useful.>

- Unit/integration tests: <pass count / total>
- Lint/typecheck: <pass>
- Manual verification: <scenario in 1-2 lines, or "not applicable">
- Regression cycle for bugs: <baseline failure and final pass evidence; reuse Phase 2 receipts or an isolated negative control>

## Known Limits / Follow-Ups

<Bullets for anything readers should know that was not included in this change. Use "none" if empty.>

## Related Documents

- Exploration report: `.backend/<YYYYMM>/<slug>/explore.md`
- Work log for AI reload: `.backend/<YYYYMM>/<slug>/work.md`
- <related ticket key if any; avoid external links>
```

---

## Quality Bar for Both Artifacts

Before `/work` exits:

- [ ] `work.md` exists and every `explore.md §7` row has a matching row in `work.md §4` **or** is listed in `§2 Contract Adherence` as skipped with a reason.
- [ ] `work.md §6` has actual command output — not paraphrased "looks good".
- [ ] Change note exists under the correct folder (`features` vs `bugs`) with today's date.
- [ ] Change note's "One-Line Summary" is readable by a non-developer — run it past your internal "non-dev reader" filter.
- [ ] Both files reference each other (`work_ref` in change note, `explore_ref` in `work.md`).
- [ ] No PR/commit has been created unless the user explicitly asked — `/work` stops at the file system.
- [ ] `/verify` is noted as the next step in the final chat summary (for example, "Next step: run `/verify` for HTTP-level proof"), unless the change has no backend HTTP surface.

If any box is unchecked, the skill is not done.
