# `/work` — TDD implementation against the plan

## TL;DR

Consumes the exploration report from `/explore` as a **contract** and implements it with subagent-driven TDD discipline, surgical diffs, and paired documentation. Refuses to improvise scope.

Outputs (all three required):
- Code changes in the services the report named
- `.backend/<YYYYMM>/<slug>/work.md` — AI-optimized paired log
- `docs/features/<YYYY-MM-DD>-<slug>.md` **or** `docs/bugs/<YYYY-MM-DD>-<slug>.md` — human-prose stakeholder note

After the contract is loaded, implementation can run directly or through useful bounded delegation. The main agent validates the contract, assigns bounded work units, reviews the handoffs, writes the `.md` artifacts, and explains what each agent did.

User-facing responses are Korean-first for Korean users. Code identifiers, paths, commands, SQL, endpoints, and exact error strings stay unchanged.

## What's in this directory

| File | Purpose |
|---|---|
| [`SKILL.md`](./SKILL.md) | The skill definition — 6 phases, iron law, scope boundaries, red flags |
| [`reuse-discipline.md`](./reuse-discipline.md) | Phase 1 re-check: re-grep before accepting stale `new` decisions from the plan |
| [`implementation-playbook.md`](./implementation-playbook.md) | Stack-specific guardrails (Spring/MyBatis backend, Vue frontend) — adapt to your stack |
| [`wrap-up-template.md`](./wrap-up-template.md) | Phase 5 templates for both `work.md` (AI-optimized) and the human-prose docs entry |

## The 6 Phases

```
Phase 0 — Load Contract       Read explore.md fully. Restate it. Validate it's workable.
Phase 1 — Reuse Re-check      Re-grep §6/§7 decisions against current code. Fix drift.
Phase 2 — TDD Implementation  Subagents run Red → Green → Verify → Refactor per unit.
Phase 3 — Surgical Review     git diff: every line must trace to §7. Revert drive-bys.
Phase 4 — Verification        Fresh output only — tests/lint/build/regression cycle.
Phase 5 — Wrap Up             Paired docs + agent work log + final agent summary.
```

## What makes this different from "just ask the AI to implement"

Typical "implement this" prompts yield:
- Scope creep ("while I'm in this file…")
- Invented utilities that duplicate existing ones
- Green tests that don't exercise the failure path
- "Done" claims based on *remembered* test runs, not fresh output

This skill refuses each of those:

- **Contract adherence** — every edited file/line must trace to §7 of `explore.md`. Anything else gets reverted.
- **Reuse re-check** — the plan's reuse decisions are re-validated against the *current* codebase before code is written.
- **Red-green regression cycle (bug fixes only)** — the regression test must fail without the fix and pass with it. Proven both ways.
- **Fresh output only** — "the build passes" is not a claim; pasted output from the last 30 seconds is.
- **Paired documentation** — the skill does not exit without `work.md` AND the human-facing docs entry.
- **Agent traceability** — `work.md` and the final response must say what the main agent did and what each subagent did.

## Stack-specific adaptation

[`implementation-playbook.md`](./implementation-playbook.md) contains guardrails specific to the original codebase (Spring Boot 3.3 + MyBatis + Vue 3). If your stack differs, fork this file and adapt:

- Transaction annotations (Spring `@Transactional(readOnly = true)` pattern)
- Layer discipline (controller / service / mapper rules)
- Cross-service contracts (Feign, Kafka, SSE, WebSocket)
- Frontend conventions (Vue 3 + Pinia + Axios)

The core 6-phase structure and iron laws are stack-independent.

## Typical invocation

```
/work  (after /explore has written explore.md and user approved)
```

Agent responds by:
1. Reading `explore.md` in full; announcing kind + scope
2. Creating a task list mirroring §7 Proposed Changes, then assigning subagents with disjoint ownership
3. Re-validating reuse decisions
4. Having subagents implement one change unit at a time with TDD
5. Reviewing subagent handoffs → verification → paired wrap-up
6. Responding with artifact paths and a clear explanation of what each agent did

Time budget: 30 min – 2 hr depending on scope. The skill will stop and ask if any assumption breaks.

## When to use

**Use when all three hold:**
- `/explore` has run for this ticket
- `explore.md` exists at `.backend/<YYYYMM>/<slug>/explore.md`
- User approved the plan (or said "proceed")

**Skip for:**
- One-line typo fixes (do directly)
- Config-only changes with zero code impact

## Dependencies

Delegation is optional: use bounded independent tasks when useful, available, and authorized. Direct execution follows the same evidence requirements.

**Optional companion skills** (if you also use the superpowers skill pack or similar): `test-driven-development`, `verification-before-completion`, `systematic-debugging` — these overlap with the inline procedures and can be used if preferred. They are not required.

## Sibling skills in the pipeline

- [`/explore`](../explore/) — predecessor; produces the contract
- [`/verify`](../verify/) — typical successor; probes the running service with curl

## The one-sentence summary

> **Ship exactly what the contract specified through bounded subagents, then leave proof and a clear agent-by-agent explanation.**
