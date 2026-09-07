---
name: work
description: Implement an approved backend exploration plan with scoped code changes, verification evidence, and paired work documentation.
---

# Work

## Overview

`/work` is the main developer skill. It consumes the exploration contract produced by `/explore` and turns it into shipped code — without re-deciding scope, re-discovering the pattern, or widening the blast radius.

**Core principle:** The exploration report is the contract. Implementation follows the contract. If the contract is wrong or insufficient, stop and update the contract — do not improvise in code.

**Design goals every change must optimize for:**
- **Contract adherence** — every edited file and line traces to §7 Proposed Changes in the explore report
- **High cohesion, low coupling** — one concern per change, no cross-module drive-bys
- **Reuse over duplication** — prefer existing utilities/DTOs/services; justify every `new` item inline
- **Minimal blast radius** — touch only what the report listed; surface anything else as a blocker
- **Style match** — follow the patterns the report cited, even if you'd personally do it differently

**Violating the letter of this process is violating the spirit.** A clever refactor that wasn't in the report is not professionalism; it's scope creep.

## The Iron Law

```
NO CODE CHANGES UNTIL THE EXPLORATION REPORT IS LOADED AND INTERNALIZED
NO COMPLETION CLAIMS UNTIL FRESH VERIFICATION EVIDENCE IS IN HAND
NO SKILL EXIT UNTIL <artifact-root>/.backend/.../work.md AND <artifact-root>/docs/features|docs/bugs ENTRIES ARE WRITTEN
```

## When to Use

Invoke `/work` when **all three** hold:

1. `/explore` has been run for this request.
2. An exploration report exists at `<artifact-root>/.backend/<YYYYMM>/<slug>/explore.md`.
3. The user has reviewed and approved the plan — or has explicitly asked you to proceed.

If any of those are missing, do **not** run `/work`:
- No report yet → run `/explore` first.
- Report exists but approval is unclear → ask which slug, then confirm intent to implement.
- User wants a one-line edit / typo fix → do it directly, skip both skills.

## Inputs & Outputs

### Inputs

| Source | What to extract |
|---|---|
| `<artifact-root>/.backend/<YYYYMM>/<slug>/explore.md` | §2 Meta (kind/slug/scope), §5 Root Cause or Pattern & Fit, §6 Reuse Inventory, §7 Proposed Changes, §8 Blast Radius, §9 Test Plan, §10 Open Questions, §11 Rollback |
| The user's latest message | Any scope delta or override since the report was written |
| The service's own `CLAUDE.md` (auto-loaded when you `cd` into it) | Stack-specific conventions, ports, profiles |
| project root `CLAUDE.md` | Cross-service patterns (Feign, Kafka, Redis, SSE, SSO) |

### Outputs (all three are required for completion)

| Artifact | Path | Purpose |
|---|---|---|
| Code changes | The services listed in §7 of the report | The actual fix/feature |
| Work log (dense, AI-optimized) | `<artifact-root>/.backend/<YYYYMM>/<slug>/work.md` | Paired with `explore.md`, reconstructible in a future session |
| Human-facing change note | `<artifact-root>/docs/features/<YYYY-MM-DD>-<slug>.md` (feature/refactor) or `<artifact-root>/docs/bugs/<YYYY-MM-DD>-<slug>.md` (bug fix) | Today's dated record of what shipped, for stakeholders |

**Folder layout (per ticket):**

```
<artifact-root>/.backend/
  <YYYYMM>/                e.g. 202604/
    <slug>/                e.g. fix-login-redirect-loop/
      explore.md           ← written by /explore (input to this skill)
      work.md              ← written by /work (this skill)
      verify.md            ← written by /verify (do NOT create from /work)
      harness/ fixtures/ runs/   ← also written by /verify; do NOT create from /work
      (other artifacts, e.g. benchmark.json, screenshots — only if a report section required them)
```

See `wrap-up-template.md` (this directory) for the exact templates of the two documents.

## Execution Rules

**Language:** Match the user's language for user-facing responses, final summaries, and human-facing docs. Keep code symbols, file paths, SQL, endpoints, commands, and exact error strings unchanged.

**Required:**
- Phase 0 validates `explore.md`. Implement directly unless a bounded independent task justifies available, authorized delegation.
- The main agent splits §7/§9 into non-overlapping work units and states file/module ownership.
- Use an independent reviewer when the risk warrants one and delegation is available and authorized.
- The main agent reviews subagent diffs and fresh verification evidence, then writes `<artifact-root>/.backend/<YYYYMM>/<slug>/work.md`.
- Features/refactors write `<artifact-root>/docs/features/<YYYY-MM-DD>-<slug>.md`; bug fixes write `<artifact-root>/docs/bugs/<YYYY-MM-DD>-<slug>.md`.
- If a docs entry already exists for the same date, do not overwrite it; use `-part2`, `-part3`, and so on.
- `work.md` includes the agent work log. If the main agent made glue edits, record them separately from subagent work.

**Forbidden:**
- Editing code before `explore.md` has been validated.
- Assigning overlapping write sets to two subagents at the same time.
- Claiming completion without reviewing subagent handoffs.
- Delegating without a bounded task, available tools, and authorization.

**Subagent handoff:**
`scope`, `files changed`, `tests/commands`, `result`, `blockers`, `scope drift`.

**Quality gates:**
- Done = every `explore.md §7` row is implemented or explicitly skipped with reason, tests/checks from §9 have fresh output, `work.md` + docs entry exist.
- Scope = edits limited to §7 files/work units plus approved glue; any new file/symbol outside §7 is `scope drift` and needs user/plan approval.
- Evidence = code claim uses `file:line`; test/build claim records command + exit/result; subagent claim requires reviewed diff/handoff.
- Failure = `FAIL-SCOPE` plan needs new files/behavior, `FAIL-TEST` check fails, `FAIL-ENV` tooling/env blocks verification, `FAIL-CONTRACT` explore report is wrong.
- Escalate = `FAIL-SCOPE`/`FAIL-CONTRACT` stops coding and returns to `/explore` or user approval; do not silently widen implementation.

**Final response:**
Report the absolute `work.md`/docs paths, work performed and, if delegated, agent ownership, verification result, and remaining blockers, then STOP.

## The Six Phases

Complete each phase before the next. Do not interleave.

### Phase 0 — Load & Validate the Contract

1. Ask or infer the artifact root and `<YYYYMM>/<slug>` ticket folder. Read `<artifact-root>/.backend/<YYYYMM>/<slug>/explore.md` **in full**.
   - If the report scope is a single microservice/module, the artifact root is that service directory.
   - If the report scope spans multiple services or is explicitly repo-wide, the artifact root is the repo root.
2. Announce the kind (Bug / Feature / Refactor) and the scope one-liner from §2.
3. Validate the contract is workable:
   - §7 Proposed Changes lists concrete `file_path · act · why` rows
   - §9 Test Plan names specific files or test classes (not "write tests")
   - §10 Open Questions has no `[BLOCK]` items still unresolved
4. If any validation fails → **STOP**. Tell the user what's missing, suggest returning to `/explore` or resolving the blocker. Do not compensate by guessing.

**Output of this phase:** a one-paragraph restatement of the plan in your own words, proving you read it. Then a TaskCreate list mirroring §7 + §9 one task per change unit, with owner, write scope, expected tests, and handoff requirements for any delegated units.

### Phase 1 — Reuse & Discipline Re-check

The reuse decisions in §6 were made during exploration, possibly weeks ago. Before writing code, re-validate them. See `reuse-discipline.md` in this directory.

Minimum re-check:

- For every `new` utility/DTO/service in §7: re-grep the current codebase for the same verb/noun. If a match you didn't see before exists, **stop and update the plan** — do not ship duplicate logic.
- For every `extend` item: re-list the callers of the symbol you're extending. If any caller relies on the old behavior, switch to `add sibling` instead of modifying in place.
- For every `modify shared utility` item: require an explicit caller census in §8. If it's missing, produce one now before touching the code.

Record the outcome of this re-check in your work-log notes (used later in `work.md`).

If a subagent performed any part of the re-check, record which agent checked which symbols and which search terms or caller census it used.

### Phase 2 — TDD Implementation

**Test-Driven Development discipline (inline — no external skill required):**

For a behavior change, follow Red → Green → Verify → Refactor: demonstrate that the test fails for the intended behavior before implementing. For a behavior-preserving refactor, establish passing characterization coverage instead; do not manufacture a failing test for unchanged behavior. Documentation-only changes use the relevant content checks.

For each behavior-changing unit (in the order listed in §7):

```
RED  →  Write ONE failing test that pins the behavior from §9 Test Plan
         for this change unit. Run it. Confirm it fails for the RIGHT reason
         (missing code, not typos).

GREEN →  Write the minimum code in the file §7 named. No speculative
         branches. No unrelated parameters or refactors.

VERIFY→  Re-run the one test. Then run the service's focused test
         command (e.g. `./gradlew :example-api:test --tests '<class>'`).
         Exit 0 + output pristine, or the unit is not done.

REFACTOR→ Only if the change unit introduced duplication with itself.
         Do NOT refactor adjacent untouched code — that's scope creep.
```

For a delegated unit, the agent follows the same verification requirements and returns:

- assigned scope and files touched
- test added/edited
- RED command + failing reason for behavior changes, or passing baseline for pure refactors
- GREEN/VERIFY command + result
- any follow-up or blocker

The main agent reviews the returned diff before accepting the unit. If the handoff lacks evidence, send the subagent back for the missing proof or redo the verification locally and record why.

Constraints that override "my instinct says X":

- **Layer discipline.** Controllers parse HTTP, services own business logic, mappers/repositories own SQL, DTOs carry data. Do not move logic across layers even if the destination file "would be more convenient".
- **Transaction annotation matches intent.** Read paths get `@Transactional(readOnly = true)` (enables read-replica routing when the stack supports it — see project CLAUDE.md). Write paths get `@Transactional` without `readOnly`. Never leave the annotation off in a service-layer method.
- **Don't change Jasypt-encrypted properties, SSO flows, or port defaults** unless §7 says to. Those are cross-service contracts.
- **Feign / Kafka / SSE / WebSocket changes are cross-service contracts.** Any edit to a `@FeignClient` method signature, a Kafka topic/payload, or an SSE/STOMP channel requires that the report already documented the downstream impact in §8. If §8 is silent and you want to make such an edit, stop and update the report.

See `implementation-playbook.md` in this directory for stack-specific guardrails per service family (Spring/MyBatis backend, Vue frontend).

### Phase 3 — Surgical Review

Before claiming completion, run your own diff review:

- Run `git diff` (or `git status` + targeted diffs) on every service touched.
- For every hunk, ask: **"Does this line trace to §7 of the report?"** If no, revert it.
- Remove imports / variables / helpers that only exist to support something you removed.
- Confirm no `// TODO`, no `console.log`, no `System.out.println`, no commented-out code, no debug breakpoints.
- Confirm no new `@SuppressWarnings`, `any`, `Object`, or empty catch blocks were introduced to make the compiler happy.

If you find anything "while I'm here"-style, revert it. Capture genuine findings as a note in `work.md §Follow-ups`, not as a change in this PR.

### Phase 4 — Verification (Evidence-First)

Record the command, actual outcome, and evidence location for checks covering the final state. Reuse current results when inputs have not changed; rerun after relevant edits, failures, or unresolved concerns. Do not claim an unrun or blocked check passed.

Applicable verifications — run those required by §9 and the touched surface, and record the command + result in `work.md`:

| Target | Command (examples — adapt to the service) |
|---|---|
| Unit + integration tests on touched services | `cd <service> && ./gradlew test` (Java) or `npm test` / `vitest` (frontend) |
| Lint / type-check on touched frontends | `npm run lint && npm run type-check` |
| Backend build | `./gradlew clean build -x test` then `./gradlew test` |
| Regression focus from §9 | The specific test names the report called out |
| Manual smoke (if §9 requires it) | The exact curl / browser steps the report listed, with observed result |

For **bug fixes**, retain evidence that the regression test fails for the original symptom and passes after the fix. Reuse the Phase 2 red/green evidence when it covers the final test and code. If the negative control is missing, run it in an isolated fixture or worktree rather than temporarily undoing shared working-tree edits.

If a verification step fails, **stop claiming progress**. Fix or revert, re-run, record the actual output. Never paraphrase test results — paste the relevant line.

Review the final diff against `explore.md §7` and the required evidence. When an independent review is warranted, give the reviewer a bounded read-only assignment and record residual risk; agent agreement alone is not proof.

### Phase 5 — Wrap Up (Paired Documentation)

Write both artifacts using `wrap-up-template.md` in this directory.

1. **`<artifact-root>/.backend/<YYYYMM>/<slug>/work.md`** — dense, symbolic, same legend as `explore.md`. This is the session-continuity document: any future run of `/work` (or a human returning in two weeks) can reload context from `explore.md` + `work.md` alone.
2. **`<artifact-root>/docs/features/<YYYY-MM-DD>-<slug>.md`** for features/refactors **or** `<artifact-root>/docs/bugs/<YYYY-MM-DD>-<slug>.md` for bug fixes — human-readable plain-language prose. This is the stakeholder record of what shipped today.

Both files are required. Do not skip either.

`work.md` must include the work log, naming delegated ownership when applicable. If the main agent made final glue edits after subagent handoff, list those separately so future readers can distinguish subagent work from integration work.

Create parent folders (`<artifact-root>/docs/features/`, `<artifact-root>/docs/bugs/`) if they do not yet exist. Use today's date (from the injected `currentDate`, not a guess) for the filename.

If an `<YYYY-MM-DD>-<slug>.md` already exists for today (e.g. same ticket shipped in two passes), append `-part2`, `-part3`, … do not overwrite.

After writing, print:
- Absolute path to `work.md`
- Absolute path to the docs entry
- A 3-5 line human summary of what shipped and what the verification showed
- A clear explanation of what the main agent did and what each subagent did

Then **STOP**. Do not commit, push, or open a PR unless the user explicitly asks. `/work` ends at the file system; git decisions are the user's.

## Scope Boundaries

| In scope for `/work` | Out of scope |
|---|---|
| Code changes listed in §7 of the explore report | Refactors not in §7 |
| Adding tests named in §9 | Raising test coverage on unrelated files |
| Renaming a symbol if §7 mandates it | Style-only renames / reformat |
| Creating `<artifact-root>/docs/features` or `<artifact-root>/docs/bugs` entry for THIS change | Reorganizing existing docs |
| Updating `work.md` inside the ticket folder | Editing `explore.md` (that's `/explore`'s contract — amend via a new explore run) |
| Running tests, lint, build | Running migrations, touching prod DB, deploying |
| Dispatching subagents with disjoint ownership | Letting subagents broaden scope or overwrite each other |
| Asking the user to re-run `/explore` if the plan is wrong | Silently diverging from the plan |

## Integration with Other Skills

**Sibling skills in this pipeline:**

- **`explore`** (required predecessor) — produces the report at `<artifact-root>/.backend/<YYYYMM>/<slug>/explore.md` that `/work` reads.
- **`verify`** (typical successor) — consumes `work.md` and runs HTTP-level curl probes against a locally running service. On failure it may re-invoke `/work` with a bounded rework delta. Run before claiming production-ready.

**Support files in this directory (always used):**

- **`reuse-discipline.md`** — Phase 1 re-check procedure
- **`implementation-playbook.md`** — stack-specific guardrails (Phase 2)
- **`wrap-up-template.md`** — Phase 5 paired-documentation templates

**Optional companion skills (if also installed — e.g. superpowers):**

This skill is self-contained — TDD and fresh-output discipline are inlined above. The following are optional enhancers if present in your agent's skill registry:

- `test-driven-development` — alternative formulation of Phase 2's Red → Green → Verify → Refactor loop
- `verification-before-completion` — alternative formulation of Phase 4's fresh-output rule
- `systematic-debugging` — use if during implementation you discover the root-cause statement in `explore.md §5A` doesn't match reality
- `requesting-code-review` — use AFTER `/work` exits, if you want a review pass before merging
- `finishing-a-development-branch` — use AFTER `/work` (typically after `/verify` passes), for commit / PR ceremony
- `jira-update` / `jira-ascli` — use AFTER `/work` exits, if you want the ticket status updated

## Quick Reference

| Phase | Input | Output | Fail state → |
|---|---|---|---|
| 0. Load Contract | `explore.md` | Plan restatement + TaskCreate | Missing/blocked report → ask user |
| 1. Reuse Re-check | §6 / §7 of report | Decisions confirmed or updated, agent owner recorded | New duplicate found → amend plan |
| 2. TDD Impl | §7 / §9 of report | Subagent code + passing tests | Wrong red → restart Phase 2 for unit |
| 3. Surgical Review | `git diff` | Diff matches §7 | Drive-by found → revert it |
| 4. Verification | Test/build commands | Fresh pass output + verifier handoff when required | Failure → fix or revert, re-run |
| 5. Wrap Up | All above | `<artifact-root>/.backend/.../work.md` + `<artifact-root>/docs/features\|bugs/…` + agent summary | Missing file → not done |

## Citations Discipline

In `work.md`, cite everything the same way `explore.md` does:

- `path/from/repo/root.ext:LINE` for every claim about code
- Ranges OK: `Foo.java:120-145`
- For commands, quote the command + the last 10 lines of relevant output (redact paths if sensitive)
- If a claim is approximate, mark it with `~` — do not fabricate precise line numbers

In `<artifact-root>/docs/features` / `<artifact-root>/docs/bugs`, prefer plain language prose; cite only when the reader would need to jump to the code (API endpoint paths, config keys, migration ids).
