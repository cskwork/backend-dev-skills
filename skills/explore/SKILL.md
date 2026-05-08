---
name: explore
description: Use when starting a non-trivial backend task in a legacy or multi-service codebase — especially when you've never touched the affected area, the blast radius is unclear, or a quick fix would be tempting but risky
---

# Explore

## Overview

Before changing legacy code, you must understand it. Guessing creates duplication, breaks hidden coupling, and ships the wrong fix.

**Core principle:** Produce a codebase-grounded, evidence-based exploration report BEFORE writing any implementation code. Every claim in the report must be backed by a concrete `file_path:line_number` citation.

**Design goals the report must optimize for:**
- **High cohesion, low coupling** — scope changes to a single concern
- **Reuse over duplication** — find and prefer existing utilities, DTOs, services
- **Minimal blast radius** — touch only code that traces directly to the request
- **Style match** — follow existing patterns even if you'd do it differently

**Violating the letter of this process is violating the spirit.** A half-evidenced report is not a report.

## The Iron Law

```
NO EXPLORATION WITHOUT TASK-BOUNDED SUBAGENTS AFTER REQUEST CLASSIFICATION
NO CODE CHANGES UNTIL THE EXPLORATION REPORT IS DELIVERED AND APPROVED
NO FINAL CONCLUSION WITHOUT EXPLAINING THE EXPLORATION STEPS AND REASONING PATH
```

The report is the deliverable. Implementation is a separate step that happens only after the user reviews and confirms the plan.

## Execution Rules

**Language:** Match the user's language for user-facing summaries and human prose sections. Keep code symbols, file paths, SQL, endpoints, commands, and exact error strings unchanged.

**Required:**
- After Phase 0 classifies Bug / Feature / Refactor, use at least one exploration subagent for non-trivial investigation.
- Assign each subagent a read-only scope: one service, layer, DB/schema area, or hypothesis.
- The main agent reviews subagent evidence, drops uncited claims, and writes `<artifact-root>/.backend/<YYYYMM>/<slug>/explore.md`.
- For single-service/module scope, use that service directory as artifact root; for multi-service/repo-wide scope, use repo root.
- If `explore.md` already exists, do not overwrite it; create `explore-v2.md`, `explore-v3.md`, and so on.
- `explore.md §12` records each subagent's search steps and reasoning log.
- Explain difficult background terms at first use in user-facing summaries. Example: `identifier` = a stable value used to find the same record, screen, or user again.

**Forbidden:**
- Editing code before `explore.md` is approved.
- Asking a subagent to edit files, revert changes, or implement code.
- Returning conclusions without search steps and reasoning.
- Proceeding solo when subagents are unavailable unless the user explicitly waives the requirement.

**Subagent handoff:**
`scope`, `steps`, `evidence(file:line)`, `hypotheses kept/rejected`, `conclusion`, `uncertainty`, `next search`.

**Quality gates:**
- Done = `explore.md` exists, §1 explains jargon, §3/§4/§5/§8/§9 cite evidence, §12 shows agent steps/reasoning, and no `[BLOCK]` is hidden.
- Scope = read-only. Allowed outputs are `.backend/<YYYYMM>/<slug>/explore*.md` only; source/test/config files are forbidden.
- Evidence = code claim uses `file:line`; DB/schema claim uses `live:` read-only result or static DDL/mapper/entity citation.
- Failure = `[BLOCK]` missing input/env access, `[GAP]` unverified but non-blocking evidence, `[INFO]` assumption that can proceed.
- Escalate = if conclusion needs plan-outside evidence or runtime proof, stop and ask; do not turn exploration into `/work` or `/verify`.

**Final response:**
Use this response order, then STOP:
1. `Executive Summary:`
2. `Recommended Change:`
3. `Rationale:`
4. `Terms:`
5. `Agent Findings:`
6. `Search Order:`

`Executive Summary:` includes the full `explore.md §1 Executive Summary` as Markdown. Put the absolute `explore.md` path in the first or last sentence of the same section. Do not only state the conclusion; explain which hypotheses the evidence accepted or rejected in `Rationale:`.

## When to Use

Use at the **start** of any non-trivial backend task:
- Feature request ("add X endpoint", "support Y case", "integrate with Z")
- Bug report ("A fails when B", "C returns wrong value", "intermittent D")
- Refactor scoping ("we want to change how E works")
- Cross-service impact questions ("what breaks if we change F?")

**Skip only for:** typo fixes, pure config changes, or one-line edits with zero behavioral ambiguity.

**Use especially when:**
- The codebase has multiple services/modules (multi-microservice, monorepo)
- You have never touched the affected area before
- The user mentioned "legacy" or asked how to avoid breaking things
- A quick fix would be tempting but the blast radius is unclear

## The Five Phases

Complete each phase before the next. Record evidence as you go — the report is assembled from these notes.

### Phase 0: Classify the Request

Decide ONE: **Bug** or **Feature** (refactor counts as feature). Write it down. The investigation path differs.

If ambiguous, ask the user. Do not guess.

### Phase 1: Locate (both tracks)

Find where the request lives in the codebase.

**Search order (cheapest first):**
1. Domain terms from the user's request — grep exact nouns/verbs
2. Existing endpoints/controllers matching the surface area
3. Service layer methods called by those controllers
4. Data access (mappers/repositories) and DB column names
5. Configuration, feature flags, profiles

**Record:** every relevant `file_path:line_number`, what the symbol does in one sentence.

**Subagent dispatch:** dispatch exploration subagents in parallel when possible — one agent per service, layer, concern, or hypothesis. Each agent returns `file:line` citations plus steps taken and reasoning. Synthesize in the main thread; do not paste uncited subagent claims into the report.

### Phase 1.5: Domain & Database (Database-First) — both tracks

Backend work is data work. Before proposing any change, ground the investigation in the schema. See `database-first.md` in this directory for the full procedure.

Minimum you must produce:

1. **Tables involved** — list every table touched by the affected code paths, with the source of truth (DDL file, MyBatis mapper, JPA entity, migration) cited as `file:line`.
2. **Columns involved** — for each table, the specific columns read/written by the change. Include type, nullability, default, PK/FK, indexes where relevant.
3. **Relationships** — FK edges between the listed tables, or the join keys used in existing queries.
4. **Domain meaning** — one sentence per table describing what the table represents in the business domain (not a restatement of the column names).
5. **Cardinality & volume (when it matters)** — is this table 1k or 100M rows? Affects index strategy, migration safety, N+1 risk.

**Use live DB read-only access when available.** Many environments ship a MySQL/Postgres MCP tool, a sandboxed read-only CLI, or a dev-replica connection string. Check in this order:

1. Look for an MCP tool whose name contains `mysql`, `postgres`, `sql`, or `db` in the available tool list. If present and read-only, use it for `DESCRIBE`, `SHOW CREATE TABLE`, `SELECT … LIMIT` against a non-prod DB.
2. Look for a repo-local helper (e.g. `database-dump/`, `scripts/db-*`, a connection string in `*.yml` under a `dev` profile). Use only if clearly read-only and non-prod.
3. If no live access, fall back to static sources in this order: migration files, JPA `@Entity` / MyBatis mapper XML, DDL dumps, schema docs.

**Safety rules when running queries:**
- Read-only only: `SELECT`, `SHOW`, `DESCRIBE`, `EXPLAIN`. No `INSERT`/`UPDATE`/`DELETE`/`DDL`.
- Non-prod only. If you can't confirm the target is non-prod, do not connect.
- Always bound with `LIMIT` on sample queries.
- Do not copy PII / real user data into the report. Redact or use placeholder values.
- If sampling data for the report, show shape and types, not raw values.

**Record in the report:** every claim about schema must cite either a DDL/mapper/entity `file:line` OR a live-query result labeled "live: DESCRIBE <table>" (with the server/database name redacted if sensitive).

### Phase 2A: Bug Track — Root Cause

**Backward-trace discipline (inline — no external skill required):**

Bugs are found by tracing *backward* from the symptom to the original trigger, not patching at the symptom site. Apply this procedure:

1. **Capture the symptom verbatim** — exact stack trace, error message, observed output. Do not paraphrase.
2. **Identify the symptom site** — the `file:line` where the visible failure manifests (where the exception is thrown, where the wrong value is returned).
3. **Walk one level up the call stack** — find the caller that produced the input leading to the symptom. Cite its `file:line`.
4. **Repeat step 3** until you reach the *original trigger* — the earliest `file:line` where the wrong input/state/assumption entered the system.
5. **Enumerate alternative hypotheses** — for each plausible-but-rejected cause along the chain, write one sentence explaining why it is not the root cause (state evidence, not intuition).
6. **State the root cause in one sentence** — "X is the root cause because Y." If you cannot fit it in one sentence, the chain is incomplete.

**Red flag:** if your proposed fix is at the symptom site rather than the original trigger, ask why. Fixing the symptom without fixing the trigger leaves the trigger free to produce the same bug elsewhere.

Minimum you must produce for the report:
1. **Symptom**: exact error message / observed behavior (copy verbatim)
2. **Reproduction**: steps or input that trigger it (or "not yet reproduced — need X")
3. **Call chain**: symptom site → immediate caller → … → original trigger, each with `file:line`
4. **Root cause statement**: "X is the root cause because Y" (one sentence)
5. **Alternative hypotheses considered and rejected**, each with why

**Fix-at-source rule:** propose the fix at the original trigger, not the symptom. If the source is untouchable, say so explicitly and justify the symptom-level fix.

### Phase 2B: Feature Track — Pattern & Fit

Answer all five before drafting a plan:

1. **Where does this belong?** Which service, which layer, which package. Justify with an existing analogous feature (`file:line`).
2. **What existing pattern applies?** Find ≥1 similar feature already in the codebase. Read it completely. Note its shape: controller → service → mapper/repo → DTO → response envelope.
3. **What can be reused?** See `reuse-checklist.md`. List every candidate utility/DTO/service with `file:line`. Default is reuse — new code requires justification.
4. **What is the interface/contract?** Request/response shape, DB columns touched, events emitted, downstream calls added.
5. **What is the blast radius?** Grep every caller of every symbol you plan to change. List them. If the list is long, the plan is wrong.

### Phase 3: Impact & Risk

For both tracks, before writing the report:

- **Callers and dependents** of each file you'll touch — list them with `file:line`
- **Shared state**: DB tables, Redis keys, Kafka topics, feature flags, cache namespaces
- **Cross-service effects**: Feign clients, SSE/WebSocket channels, SSO/session assumptions
- **Tests that will need to change or be added** — list paths; no counts without paths
- **Rollback story**: how to revert if this is wrong (config toggle? single commit? DB migration?)

If any of these are unknown, say "unknown — need to verify X" explicitly. Do not fabricate confidence.

### Phase 4: Report

Produce the report using `report-template.md` in this directory. The format is non-negotiable:

1. **Executive Summary** at the top — **human-readable prose** for non-developers. 5–10 lines. No unexplained jargon, no file paths, no symbols. Answer: what's the problem/goal, what will change, what's the risk, when it's done. If a technical term is necessary, add a short "Terms" line.
2. **AI-Optimized Body** below — **dense symbolic shorthand** for later re-read by AI. Every fact cites `file:line`. Use the symbol legend in `report-template.md`. Prose is banned in this section; prefer arrows, bullets, tables, and inline citations.

**Persistence — save the report to disk:**

- Artifact root:
  - If scope is a single microservice/module, use that service directory. Example: `example-viewer-api/.backend/<YYYYMM>/<slug>/explore.md`.
  - If scope spans multiple services or is explicitly repo-wide, use the repo root. Example: `.backend/<YYYYMM>/<slug>/explore.md`.
  - If unclear, infer from the endpoint/controller/service path; ask only when multiple service roots are equally plausible.
- Folder layout: `<artifact-root>/.backend/<YYYYMM>/<slug>/` (month-bucketed)
- File: `explore.md` inside that folder — i.e. full path `<artifact-root>/.backend/<YYYYMM>/<slug>/explore.md`
- `<YYYYMM>`: 6-digit year+month of the exploration (e.g. `202604`)
- `<slug>`: kebab-case, ≤50 chars, derived from the request (e.g. `fix-login-redirect-loop`, `add-chapter-bookmark-api`)
- Create `<artifact-root>/.backend/`, `<artifact-root>/.backend/<YYYYMM>/`, and the `<slug>/` subfolder if any of them do not exist
- If an `explore.md` already exists in that folder, append a new version as `explore-v2.md`, `explore-v3.md`, … do not overwrite
- The folder is the persistent workspace for this ticket and will also hold sibling artifacts produced by other skills — in particular `work.md` (written by `/work`) and `verify.md` (written by `/verify`). Do not create `work.md` or `verify.md` from the `/explore` skill.
- After writing, respond in the fixed final-response order: `Executive Summary:` → `Recommended Change:` → `Rationale:` → `Terms:` → `Agent Findings:` → `Search Order:`. In `Executive Summary:`, include the full `explore.md §1 Executive Summary` as Markdown, not only the artifact path.

Then **STOP**. Wait for the user to confirm before any implementation.

## Output Format (Strict)

See `report-template.md` for the full template with the symbol legend. Required structure:

1. **§1 Executive Summary** — prose, human, non-dev (plain language, no symbols, no paths)
2. **§2 Meta** — kind/slug/yyyymm/scope/one-liner
3. **§3 Evidence** — `file:line · note`
4. **§4 Domain & Data** — tables, cols (with type/NN/PK/FK/IX), rels, cardinality, source tags (`L:` / `S:`)
5. **§5A Root Cause** (bug) OR **§5B Pattern & Fit** (feature)
6. **§6 Reuse Inventory** — utils/dtos/svc/clients/config with `≡ ≈ ⊕`
7. **§7 Proposed Changes** — `# file · act · why · reuse/new`
8. **§8 Blast Radius** — callers/shared state/cross-service/config/migration
9. **§9 Test Plan** — add/edit/manual/regression-focus
10. **§10 Open Questions** — `[BLOCK] [INFO] [GAP]`
11. **§11 Rollback** — code/db/data/flag
12. **§12 Exploration Steps & Reasoning** — subagent work log, search trail, hypotheses kept/rejected
13. **§13 Repro Header** — 3 lines to reload context in later sessions

**Prose is banned from §2 onward.** If you're writing sentences, convert to symbolic form per the legend.

## Quick Reference

| Phase | Bug | Feature |
|-------|-----|---------|
| 0. Classify | ✓ | ✓ |
| 1. Locate | grep symptom, trace callers | grep domain, find analogue |
| 1.5. Domain & DB | tables/columns in the broken path | tables/columns touched by new logic |
| 2. Analyze | root-cause chain (file:line) | pattern fit + reuse candidates |
| 3. Impact | who else hits this path/schema | who else imports/calls/reads the table |
| 4. Report | executive summary + detailed plan |

| Signal | Action |
|--------|--------|
| Any non-trivial explore task | Dispatch at least one exploration subagent after Phase 0 |
| Codebase has 3+ services to search | Dispatch one exploration subagent per service/concern in parallel |
| User says "just fix it quickly" | Still produce the report — it will be short if the bug is small |
| You cannot reproduce the bug | Say so in Executive Summary and Open Questions |
| "New utility needed" feels right | Re-run the reuse checklist before proposing it |
| Blast radius list has 10+ items | Scope is wrong — narrow the change or split the task |

## Red Flags — STOP and Restart Phase

If you catch yourself thinking:
- "I've seen this pattern before, I'll just write the fix"
- "Evidence is obvious, skipping citations"
- "The user is in a hurry, skip the executive summary"
- "I'll add a new helper — probably nothing like it exists"
- "Blast radius looks fine, I won't list the callers"
- "This is similar enough to X, I don't need to read X fully"
- "I'll figure out tests when I implement"
- "The table name is obvious from the entity, no need to check the schema"
- "I know the columns from the DTO — skipping DESCRIBE"
- "I'll just add a column, indexes and FKs don't matter here"
- "I'll explore solo because I already know where the issue is"
- "The subagent concluded X, so I don't need to explain the reasoning"
- "The final response can just summarize the conclusion without the steps"

**All of these mean: STOP. Return to the phase that produces the missing evidence.**

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "Simple change, no report needed" | Simple changes in legacy code still have non-obvious dependents. 10 minutes of exploration prevents a 2-hour incident. |
| "Executive summary is fluff" | Non-devs approve/route this work. Without the summary, the report is unusable for reporting. |
| "I'll cite files later when I write code" | "Later" erases the audit trail and invites fabrication. Cite as you find. |
| "New utility is cleaner than adapting the existing one" | Cleanness is a local optimum. The codebase optimum is one way to do each thing. |
| "Blast radius is intuitive, no need to list" | Intuition misses Feign clients, Kafka consumers, SSE channels, cache keys. List them. |
| "Root cause and fix are the same thing" | No. Root cause is a statement about the code. Fix is a proposal. The report must separate them. |
| "Subagents are optional during exploration" | The point is to make discovery auditable: who checked which path, what evidence they found, and why the conclusion follows. |
| "Reasoning belongs only in my head" | The user needs to monitor the exploration path. Report the steps and evidence-based reasoning, not just the final answer. |

## Reuse and Low-Coupling Discipline

See `reuse-checklist.md` for the full checklist. Summary:

- **Before proposing a new utility**: grep for the verb, the noun, and 2 synonyms across the codebase. If a match exists, prefer it.
- **Before adding a new DTO**: check existing request/response classes in the same module. Extend only if the semantics match; otherwise justify a new type.
- **Before adding a cross-service call**: verify no existing Feign client or event channel already carries this data.
- **Before modifying a shared utility**: list every caller. If the change is not safe for all callers, do not modify — add a sibling or parameterize.
- **Surgical edit rule**: every changed line must trace directly to the request. No drive-by reformatting, renames, or "while I'm here" cleanups.

## Citations Discipline

- Every factual claim about the codebase → `path/from/repo/root.ext:LINE`
- Ranges are OK: `Foo.java:120-145`
- Quote at most 3 lines when showing code; link with `file:line` instead of pasting large blocks
- If a citation is approximate (you searched but didn't open the file), mark it `~` e.g. `Foo.java:~120`

## Cross-References

**Support files in this directory (always used):**

- **`database-first.md`** — procedure for Phase 1.5 schema/data verification
- **`reuse-checklist.md`** — checklist for Phase 2B reuse discipline
- **`report-template.md`** — Phase 4 output format with symbol legend

**Sibling skills in this pipeline:**

- **`work`** — consumes `explore.md` and implements the plan
- **`verify`** — consumes `explore.md` + `work.md` and proves the implementation with live curl probes; may re-invoke `/work` on failure

**Optional companion skills (if also installed — the superpowers ecosystem and others):**

This skill is self-contained. The following are optional enhancers if present in your agent's skill registry, but their absence does not degrade `/explore`:

- `systematic-debugging` — alternative formulation of Phase 2A's backward-trace discipline (already inlined above)
- `dispatching-parallel-agents` — alternative formulation of the required exploration-subagent dispatch (already inlined above)
- `writing-plans` — follow-on: turn the approved §7 into a structured executable plan doc
- `test-driven-development` — applies when §9 test plan items are implemented in `/work` Phase 2

## The Bottom Line

The explore skill's job is to replace guessing with evidence and to produce a document that a non-developer can read at the top and a developer can implement from the bottom. The report is the contract. No code before the contract.
