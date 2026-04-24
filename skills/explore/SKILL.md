---
name: explore
description: Use when starting a non-trivial backend task in a legacy or multi-service codebase — especially when you've never touched the affected area, the blast radius is unclear, or a quick fix would be tempting but risky
---

# Explore

## Overview

Before changing legacy code, understand it. Guessing creates duplication, breaks hidden coupling, and ships the wrong fix.

**Core principle:** Produce a codebase-grounded, evidence-based report BEFORE writing any implementation code. Every claim cites `file:line`.

**Design goals:**
- High cohesion, low coupling — scope to a single concern
- Reuse over duplication — prefer existing utilities, DTOs, services
- Minimal blast radius — touch only what traces to the request
- Style match — follow existing patterns

## The Iron Law

```
NO CODE CHANGES UNTIL THE EXPLORATION REPORT IS DELIVERED AND APPROVED
```

The report is the deliverable. Implementation happens only after user reviews and confirms.

## When to Use

At the **start** of any non-trivial backend task: feature, bug, refactor scoping, cross-service impact question.

**Skip only for:** typo fixes, pure config, one-line edits with zero behavioral ambiguity.

**Use especially when:** codebase has multiple services; you've never touched the area; user mentioned "legacy"; a quick fix would be tempting but blast radius unclear.

## The Five Phases

Complete each before the next. Record evidence as you go.

### Phase 0: Classify the Request

Decide ONE: **Bug** or **Feature** (refactor counts as feature). If ambiguous, ask the user.

### Phase 1: Locate

Find where the request lives.

**Search order (cheapest first):**
1. Domain terms from user's request — grep exact nouns/verbs
2. Existing endpoints/controllers matching the surface
3. Service methods called by those controllers
4. Data access (mappers/repositories), DB column names
5. Config, feature flags, profiles

**Record:** every relevant `file:line` with one-sentence purpose.

**Parallel dispatch:** if 3+ services to inspect, dispatch `Explore` subagent per service. Each returns `file:line` citations only; synthesize in main thread.

### Phase 1.5: Domain & Database (Database-First)

Backend work is data work. See `database-first.md` for full procedure.

Minimum output:
1. **Tables involved** — each with DDL/mapper/entity `file:line`
2. **Columns involved** — type, nullability, default, PK/FK, indexes
3. **Relationships** — FK edges or join keys used in existing queries
4. **Domain meaning** — one sentence per table (business, not column restatement)
5. **Cardinality** when it matters — affects index, migration safety, N+1

**Prefer live DB read-only access** (MCP mysql/postgres tool, repo helper, dev-profile connection). Fall back to static sources: migrations, JPA/MyBatis mappers, DDL dumps.

**Safety rules:** read-only only (`SELECT`/`SHOW`/`DESCRIBE`/`EXPLAIN`); non-prod only; `LIMIT` bounded samples; no PII in report (shape + type only, redacted placeholders).

Every schema claim cites DDL/mapper `file:line` OR a `live: DESCRIBE <table>` label.

### Phase 2A: Bug Track — Root Cause

Bugs are found by tracing **backward** from symptom to trigger, not patching at symptom site.

1. **Symptom verbatim** — exact stack trace, message, observed output.
2. **Symptom site** — `file:line` where failure manifests.
3. **Walk up the call stack** — caller that produced the bad input. Cite `file:line`. Repeat until the **original trigger** — earliest site where the bad input/state entered.
4. **Alternative hypotheses** — for each rejected cause, one sentence of evidence.
5. **Root cause statement** — "X is the root cause because Y." One sentence. If longer, chain is incomplete.

**Red flag:** if the fix is at symptom site not original trigger, ask why. Symptom-only fixes leave the trigger free to re-emerge.

Report minimum: symptom / reproduction / call chain / root cause / rejected alternatives.

### Phase 2B: Feature Track — Pattern & Fit

Answer all five:

1. **Where does it belong?** Service, layer, package. Justify with an analogous feature (`file:line`).
2. **What existing pattern applies?** Find ≥1 similar feature. Read it fully. Note shape: controller → service → mapper → DTO → envelope.
3. **What can be reused?** See `reuse-checklist.md`. List every candidate with `file:line`. Default is reuse; new code needs justification.
4. **Interface/contract?** Req/resp shape, columns, events, downstream calls.
5. **Blast radius?** Grep every caller of every symbol you'd change. Long list = wrong plan.

### Phase 3: Impact & Risk

For both tracks, before writing the report:

- **Callers/dependents** per touched file with `file:line`
- **Shared state** — DB tables, Redis keys, Kafka topics, feature flags, cache namespaces
- **Cross-service** — Feign, SSE/WebSocket, SSO/session assumptions
- **Tests to change/add** — list paths, no counts without paths
- **Rollback story** — config toggle / single commit / migration?

Unknown items → "unknown — need to verify X" explicitly. Never fabricate confidence.

### Phase 4: Report

Produce using `report-template.md`. Format non-negotiable:

1. **Executive Summary** — human-readable prose for non-devs. 5-10 lines. No jargon, no paths, no symbols. What / why / risk / when done.
2. **AI-Optimized Body** — dense symbolic shorthand. Every fact cites `file:line`. Use symbol legend. Prose banned; use arrows, bullets, tables.

**Persistence:**
- Path: `.backend/<YYYYMM>/<slug>/explore.md` (repo-root relative)
- `<YYYYMM>`: year+month (e.g. `202604`); `<slug>`: kebab-case, ≤50 chars
- Create parent folders if absent
- Existing `explore.md` → append as `explore-v2.md`, `explore-v3.md`… never overwrite
- Folder is the persistent ticket workspace (holds `work.md`, `verify.md` from sibling skills — do not create those here)
- After writing, print absolute path + one-paragraph verbal summary

Then **STOP**. Wait for user confirmation.

## Output Format (Strict)

See `report-template.md` for full template + symbol legend. Required sections:

1. §1 Executive Summary — prose, human, non-dev
2. §2 Meta — kind/slug/yyyymm/scope/one-liner
3. §3 Evidence — `file:line · note`
4. §4 Domain & Data — tables, cols (type/NN/PK/FK/IX), rels, cardinality, source tags
5. §5A Root Cause (bug) OR §5B Pattern & Fit (feature)
6. §6 Reuse Inventory — utils/dtos/svc/clients with `≡ ≈ ⊕`
7. §7 Proposed Changes — `# file · act · why · reuse/new`
8. §8 Blast Radius — callers / shared state / cross-service / config / migration
9. §9 Test Plan — add/edit/manual/regression-focus
10. §10 Open Questions — `[BLOCK] [INFO] [GAP]`
11. §11 Rollback — code/db/data/flag
12. §12 Search Trail *(optional)*
13. §13 Repro Header — 3 lines to reload context

Prose banned from §2 onward.

## Quick Reference

| Phase | Bug | Feature |
|---|---|---|
| 0. Classify | ✓ | ✓ |
| 1. Locate | grep symptom, trace callers | grep domain, find analogue |
| 1.5. Domain & DB | tables/cols in broken path | tables/cols touched by new logic |
| 2. Analyze | root-cause chain (file:line) | pattern fit + reuse candidates |
| 3. Impact | who hits this path/schema | who imports/calls/reads the table |
| 4. Report | executive summary + detailed plan |

| Signal | Action |
|---|---|
| 3+ services to search | Dispatch `Explore` subagent per service |
| "Just fix it quickly" | Still produce report — short if bug is small |
| Cannot reproduce | Say so in Executive Summary + Open Questions |
| "New utility needed" feels right | Re-run reuse checklist first |
| Blast radius has 10+ items | Scope wrong — narrow or split |

## Red Flags — STOP

- "Seen this pattern, just write the fix."
- "Evidence obvious, skip citations."
- "User in a hurry, skip executive summary."
- "New helper — probably nothing like it exists."
- "Blast radius looks fine, won't list callers."
- "Similar enough to X, don't need to read X fully."
- "I'll figure out tests when I implement."
- "Table name obvious from entity — skip schema check."
- "Columns obvious from DTO — skipping DESCRIBE."
- "Just adding a column, indexes/FKs don't matter."

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Simple change, no report" | Simple changes in legacy code still have non-obvious dependents. 10min of exploration prevents 2-hour incidents. |
| "Executive summary is fluff" | Non-devs approve/route this work. Without summary, report is unusable. |
| "I'll cite files later" | "Later" erases audit trail and invites fabrication. Cite as you find. |
| "New utility cleaner than adapting" | Cleanness is local optimum. Codebase optimum is one way per thing. |
| "Blast radius intuitive" | Intuition misses Feign clients, Kafka consumers, SSE channels, cache keys. |
| "Root cause and fix are same" | No. Root cause is about code; fix is a proposal. Separate them. |

## Reuse Discipline

See `reuse-checklist.md`. Summary:

- Before new utility: grep verb + noun + 2 synonyms. Prefer match if found.
- Before new DTO: check existing req/resp in same module. Extend only if semantics match.
- Before new cross-service call: verify no existing Feign/event channel carries this data.
- Before modifying shared utility: list every caller. Not safe for all → add sibling or parameterize.
- **Surgical edit rule:** every changed line traces to the request. No drive-by reformat/rename/cleanup.

## Citations

- Every factual claim → `path/from/repo/root.ext:LINE`
- Ranges OK: `Foo.java:120-145`
- Quote ≤3 lines when showing code; link with `file:line`
- Approximate (searched, didn't open) → mark `~`, e.g. `Foo.java:~120`

## Cross-References

**Support files (always used):**
- `database-first.md` — Phase 1.5 schema/data procedure
- `reuse-checklist.md` — Phase 2B reuse discipline
- `report-template.md` — Phase 4 output format + symbol legend

**Pipeline siblings:**
- `work` — consumes `explore.md`, implements the plan
- `verify` — consumes both, proves implementation with live curl; may re-invoke `/work`

## The Bottom Line

Replace guessing with evidence. Produce a document a non-developer can read at the top and a developer can implement from the bottom. The report is the contract. No code before the contract.
