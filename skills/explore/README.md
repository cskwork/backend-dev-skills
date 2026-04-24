# `/explore` — evidence-based plan before any code

## TL;DR

Before touching a legacy / multi-service codebase, this skill forces the agent to produce an **evidence-based exploration report** where every claim cites `file:line`. No code changes happen until the report is written and approved.

Output: `.backend/<YYYYMM>/<slug>/explore.md`

## What's in this directory

| File | Purpose |
|---|---|
| [`SKILL.md`](./SKILL.md) | The skill definition — loaded by the agent, contains the 5 phases, iron law, red flags, citations discipline |
| [`database-first.md`](./database-first.md) | Phase 1.5 — schema/data verification procedure; uses live `DESCRIBE` when available |
| [`reuse-checklist.md`](./reuse-checklist.md) | Phase 2B reuse-over-duplication checklist |
| [`report-template.md`](./report-template.md) | Two-tier output template (prose §1 + symbolic §2+) with symbol legend |

## The 5 Phases

```
Phase 0 — Classify        Bug or Feature? (refactor counts as feature)
Phase 1 — Locate          Grep domain terms → controllers → services → data access
Phase 1.5 — Database      Tables, columns, relationships, cardinality. Live DB when safe.
Phase 2A — Root Cause     (bug) Call chain from symptom → original trigger
Phase 2B — Pattern & Fit  (feature) Analogous feature + reuse inventory + contract
Phase 3 — Impact          Callers, shared state, cross-service, tests, rollback
Phase 4 — Report          Executive summary + symbolic body → save → STOP
```

## What makes this different from "just ask the AI to make a plan"

A typical "make a plan" prompt produces plausible-sounding prose. This skill demands **evidence**:

- Every factual claim → `path/from/repo/root.ext:LINE` citation
- §1 (Executive Summary) is **prose for non-developers** — routable to PM/QA
- §2 onward is **symbolic shorthand** — dense, AI-optimized re-read format
- Prose is **banned** from §2 onward; no paragraphs, only arrows/bullets/tables
- Red flags ("I'll cite files later", "blast radius is intuitive") → mandatory STOP

The report is a contract, not a sketch. `/work` implements against it; `/verify` probes against it. The triad only works because the contract is machine-reliable.

## Typical invocation

```
/explore we need to add a bookmark endpoint so students can pin a chapter
```

Agent responds by:
1. Asking "Bug or Feature?" (answer: Feature)
2. Running Phase 1 (locate) — greps for existing bookmark-ish things
3. Running Phase 1.5 (DB) — `DESCRIBE` the candidate tables
4. Running Phase 2B (pattern) — finds an analogous feature to copy
5. Writing `explore.md` and stopping with a one-paragraph verbal summary

Time budget: 5–15 minutes for most requests. Longer if the codebase has 3+ services to probe in parallel (skill dispatches subagents for this).

## When to use

**Use for:**
- Feature request in a multi-service / legacy codebase
- Bug report where the call chain is not obvious
- Refactor scoping — "what breaks if we change X?"
- Any time you'd tell a junior engineer "go read the code first"

**Skip for:**
- Typo fixes
- Pure config changes
- One-line edits with zero behavioral ambiguity

## Dependencies

**None.** This skill is fully self-contained — the 5 phases, the backward-trace procedure for bug root-causing, and the parallel-subagent technique for multi-service searches are all inlined in `SKILL.md`. Install the folder and it works.

**Optional companion skills** (if you also use the superpowers skill pack or similar): `systematic-debugging`, `dispatching-parallel-agents`, `writing-plans`, `test-driven-development` — these overlap with the inline procedures and can be used if preferred. They are not required.

## Sibling skills in the pipeline

- [`/work`](../work/) — consumes `explore.md` and ships code
- [`/verify`](../verify/) — consumes `work.md` and proves the change end-to-end

## The one-sentence summary

> **Replace guessing with evidence. No code before the contract.**
