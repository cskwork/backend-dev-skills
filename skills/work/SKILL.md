---
name: work
description: Use when an approved exploration report exists at .backend/<YYYYMM>/<slug>/explore.md and it is time to implement the plan
---

# Work

## Overview

`/work` consumes the exploration contract from `/explore` and ships code — without re-deciding scope, re-discovering the pattern, or widening the blast radius.

**Core principle:** The exploration report is the contract. Implementation follows it. If the contract is wrong or insufficient, stop and update the contract — do not improvise.

**Design goals:**
- Contract adherence — every edited line traces to §7 Proposed Changes
- High cohesion, low coupling — one concern per change
- Reuse over duplication — justify every `new` item inline
- Minimal blast radius — surface anything outside §7 as a blocker
- Style match — follow patterns the report cited

## The Iron Law

```
NO CODE CHANGES UNTIL THE EXPLORATION REPORT IS LOADED AND INTERNALIZED
NO COMPLETION CLAIMS UNTIL FRESH VERIFICATION EVIDENCE IS IN HAND
NO SKILL EXIT UNTIL work.md AND docs/features|docs/bugs ENTRIES ARE WRITTEN
```

## When to Use

All three must hold:
1. `/explore` has been run for this request
2. Report exists at `.backend/<YYYYMM>/<slug>/explore.md`
3. User approved the plan or explicitly asked to proceed

Otherwise:
- No report → run `/explore` first
- Approval unclear → ask which slug, confirm intent
- One-line edit / typo → do it directly, skip both skills

## Inputs & Outputs

### Inputs

| Source | What to extract |
|---|---|
| `explore.md` | §2 Meta, §5 Root Cause/Pattern, §6 Reuse, §7 Changes, §8 Blast Radius, §9 Tests, §10 Open Qs, §11 Rollback |
| User's latest message | Scope delta since report |
| Service `CLAUDE.md` | Stack conventions, ports, profiles |
| Repo root `CLAUDE.md` | Cross-service patterns (HTTP clients, Kafka, Redis, SSE, SSO) |

### Outputs (all three required)

| Artifact | Path | Purpose |
|---|---|---|
| Code changes | Services in §7 | The fix/feature |
| Work log | `.backend/<YYYYMM>/<slug>/work.md` | Dense, AI-optimized; reconstructible |
| Change note | `docs/features/<YYYY-MM-DD>-<slug>.md` or `docs/bugs/<YYYY-MM-DD>-<slug>.md` | Human-readable, dated |

**Folder layout:**
```
.backend/<YYYYMM>/<slug>/
  explore.md   ← input
  work.md      ← this skill
  verify.md, harness/, fixtures/, runs/   ← /verify's; do NOT create here
```

See `wrap-up-template.md` for both document templates.

## The Six Phases

Complete each before the next.

### Phase 0 — Load & Validate the Contract

1. Ask/infer `<YYYYMM>/<slug>`. Read `explore.md` **in full**.
2. Announce kind (Bug/Feature/Refactor) + scope one-liner from §2.
3. Validate:
   - §7 lists concrete `file · act · why` rows
   - §9 names specific files/test classes (not "write tests")
   - §10 has no unresolved `[BLOCK]`
4. Any validation fail → **STOP**. Tell user what's missing; suggest `/explore` re-run. Do not guess.

**Output:** one-paragraph restatement + TaskCreate list mirroring §7 + §9.

### Phase 1 — Reuse & Discipline Re-check

§6 decisions may be stale (weeks old). See `reuse-discipline.md`. Minimum:

- Every `new` utility/DTO/service → re-grep verb/noun. New match → **stop + update plan**.
- Every `extend` → re-list callers. Any relies on old behavior → switch to `add sibling`.
- Every `modify shared utility` → require explicit caller census in §8. Missing → produce it now before touching code.

Record outcome in work-log notes.

### Phase 2 — TDD Implementation

Every change unit: **Red → Green → Verify → Refactor**. Skipping RED is how this phase fails — the test must be **seen** failing for the right reason.

For each change unit (in §7 order):

```
RED    → ONE failing test pinning behavior from §9 for this unit.
         Run it. Confirm it fails for the RIGHT reason.
GREEN  → Minimum code in the file §7 named. No speculative branches.
         No parameters §7 didn't ask for. No refactors.
VERIFY → Re-run the test + focused command
         (e.g. ./gradlew test --tests '<class>' or npm test -- <file>).
         Exit 0 + pristine output, or the unit isn't done.
REFACTOR → Only if this change unit introduced duplication with itself.
           Never refactor adjacent untouched code.
```

Overriding constraints:
- **Layer discipline.** Controllers parse HTTP, services own logic, mappers/repos own SQL, DTOs carry data. No logic-shifting across layers.
- **Transaction annotation matches intent.** Read paths → `@Transactional(readOnly = true)` (enables read-replica routing where configured). Writes → `@Transactional`. Never leave off on service methods.
- **Don't touch encrypted properties, SSO flows, or port defaults** unless §7 says to.
- **HTTP clients / Kafka / SSE / WebSocket = cross-service contracts.** Any signature or topic/payload/channel edit requires §8 already documents downstream impact. Silent §8 → stop, update report.

See `implementation-playbook.md` for stack-specific guardrails.

### Phase 3 — Surgical Review

Before claiming completion:

- Run `git diff` / `git status` on every touched service.
- For every hunk: **"Does this line trace to §7?"** No → revert.
- Remove imports/variables/helpers that only support something you removed.
- Confirm no `// TODO`, `console.log`, `System.out.println`, commented-out code, debug breakpoints.
- No new `@SuppressWarnings`, `any`, `Object`, or empty catch blocks added to make compiler happy.

"While I'm here"-style edits → revert. Real findings → `work.md §Follow-ups`, not this PR.

### Phase 4 — Verification (Evidence-First)

"Tests passed" isn't a claim. "Here's the exact command I just ran + its last 10 lines" is. Every verification below runs in the current session with command + real output recorded in `work.md`. Paraphrased or remembered results are forbidden.

Minimum (each recorded with command + result):

| Target | Command |
|---|---|
| Unit/integration on touched services | `cd <service> && ./gradlew test` or `npm test` / `vitest` |
| Lint/type-check on touched frontends | `npm run lint && npm run type-check` |
| Backend build | `./gradlew clean build -x test` then `./gradlew test` |
| Regression focus from §9 | Specific tests report called out |
| Manual smoke if §9 requires | Exact curl/browser steps + observed result |

For **bug fixes** run the red-green regression cycle:
1. Fix applied → new regression test passes
2. Revert fix → test **must fail** (record failing output)
3. Re-apply → passes again

Any verification fails → stop claiming progress. Fix/revert, re-run, paste actual output.

### Phase 5 — Wrap Up (Paired Documentation)

Write both artifacts using `wrap-up-template.md`:

1. **`.backend/<YYYYMM>/<slug>/work.md`** — dense symbolic, same legend as `explore.md`. Future `/work` or human can reload context from `explore.md` + `work.md` alone.
2. **`docs/features/<YYYY-MM-DD>-<slug>.md`** (feature/refactor) **or** `docs/bugs/<YYYY-MM-DD>-<slug>.md` (bug) — human-readable Korean-first prose. Stakeholder record.

Both required. Use today's `currentDate` (not a guess). Same-day re-ship → `-part2`, `-part3`; never overwrite.

Print: absolute paths + 3-5 line human summary.

Then **STOP.** `/work` ends at the filesystem; git is the user's call.

## Scope Boundaries

| In scope | Out of scope |
|---|---|
| Code changes in §7 | Refactors outside §7 |
| Adding tests named in §9 | Coverage raises on unrelated files |
| Renaming if §7 mandates | Style-only renames/reformat |
| Creating `docs/features\|bugs` entry for THIS change | Reorganizing existing docs |
| Updating `work.md` | Editing `explore.md` (amend via new explore run) |
| Running tests, lint, build | Migrations, prod DB, deploys |
| Asking user to re-run `/explore` if plan wrong | Silently diverging from plan |

## Red Flags — STOP

- "Report missed this file but I'll edit it."
- "Test in §9 is overkill, I'll write simpler."
- "Skip red-green, confident in fix."
- "`extend` close enough to `new` — just add a helper."
- "While in this file I'll clean up an unrelated method."
- "Build passes, skip focused tests."
- "Write `work.md` later; code is the real deliverable."
- "Human doc is formality; one sentence is enough."
- "Open question §10 isn't blocking — I'll decide."
- "Feign signature change small; downstream probably fine."

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Report slightly wrong; I'll quietly fix in code" | Quiet plan/code divergence rots legacy codebases. Update plan or ask. |
| "Reuse re-check is duplicate work" | 2-week-old grep is stale. Re-check is 5min; duplicate logic is forever. |
| "TDD is ceremony for simple change" | Simple changes break in legacy with hidden coupling. Failing test proves the symptom was real. |
| "I know the test passes, don't need rerun" | No fresh output = no claim. Re-run now, paste output. |
| "Docs for end-of-sprint, not per change" | Per-change docs are how reviewers track shipped work. Paired explore/work is how *you* reload context. |
| "Cross-service impact is DevOps' problem" | HTTP clients / Kafka / SSE cross service boundaries by construction. You edited it, you own the downstream statement. |

## Integration with Other Skills

**Pipeline siblings:**
- `explore` (required predecessor) — produces the report `/work` reads
- `verify` (typical successor) — HTTP-level curl probes against local service; may re-invoke `/work` with bounded rework delta

**Support files (always used):**
- `reuse-discipline.md` — Phase 1 procedure
- `implementation-playbook.md` — stack-specific guardrails (Phase 2)
- `wrap-up-template.md` — Phase 5 templates

## Quick Reference

| Phase | Input | Output | Fail → |
|---|---|---|---|
| 0. Load | `explore.md` | Plan restatement + TaskCreate | Missing/blocked → ask user |
| 1. Reuse | §6/§7 | Decisions confirmed or updated | New duplicate → amend plan |
| 2. TDD | §7/§9 | Code + passing tests | Wrong red → restart for unit |
| 3. Review | `git diff` | Diff matches §7 | Drive-by → revert |
| 4. Verification | Test/build | Fresh pass output | Fail → fix/revert, re-run |
| 5. Wrap | All above | `work.md` + docs entry | Missing file → not done |

## Citations

In `work.md`, same style as `explore.md`:
- `path/from/repo/root.ext:LINE` per code claim; ranges OK (`Foo.java:120-145`)
- Commands: quote command + last 10 lines of relevant output (redact sensitive paths)
- Approximate → mark `~`; never fabricate line numbers

In `docs/features`/`docs/bugs`: plain Korean prose; cite only when reader needs to jump (endpoints, config keys, migration ids).

## The Bottom Line

`/explore` produces the contract. `/work` fulfills it, verifies the fulfillment, and leaves a paired paper trail.

The job isn't "best code from scratch." It's **ship what the contract specified, cleanly, reusing what exists, and prove it works.** If that reveals the contract was wrong, stop and fix the contract — don't fix it in silence.
