# Install on Gemini CLI

Gemini CLI reads `GEMINI.md` from the repo root as its primary instruction file. Install by copying the skill pointers into it.

## Step 1 — Clone this repo

```bash
git clone https://github.com/cskwork/backend-dev-skills.git ~/backend-dev-skills
```

## Step 2 — Append to your project's `GEMINI.md`

```markdown
# Backend Development Skills

This project uses four skills for non-trivial backend/fullstack work. When the
user asks for a feature, bug fix, verification, or post-deploy QA, load the
corresponding skill from `~/backend-dev-skills/skills/<name>/SKILL.md` and
follow it phase-by-phase.

## Skills

- **explore** — evidence-based plan before any code. 5 phases.
  Path: `~/backend-dev-skills/skills/explore/SKILL.md`
  Output: `.backend/<YYYYMM>/<slug>/explore.md`

- **work** — TDD implementation against the plan. 6 phases.
  Path: `~/backend-dev-skills/skills/work/SKILL.md`
  Output: `work.md` + `docs/features|bugs/<date>-<slug>.md`

- **verify** — live HTTP-level proof (localhost only). 7 phases.
  Path: `~/backend-dev-skills/skills/verify/SKILL.md`
  Output: `verify.md` + `harness/` + `fixtures/` + `runs/`

- **qa-engineer** — post-deploy browser E2E (dev/stg/prod, NOT localhost). 7 phases.
  Path: `~/backend-dev-skills/skills/qa-engineer/SKILL.md`
  Output: `qa.md` + `e2e/specs/*.spec.ts` (repeatable Playwright suite, updated in-place)
  Prereq: `@playwright/test` + `playwright-cli` installed (`npm i -g @playwright/test @playwright/cli && npx playwright install`)

## Iron Laws (all four skills)

1. No code changes until `/explore` has produced an approved `explore.md`.
2. No "done" claim until fresh verification output is in hand.
3. No "production-ready" claim until `/verify` says `status: pass`.
4. No "live for users" claim until `/qa-engineer` says `status: pass` with trace+video recorded against the deployed URL.
5. `/verify` refuses non-localhost `BASE_URL`; `/qa-engineer` refuses localhost. They are complementary gates.

## How to invoke

User mentions "explore", "work", "verify" OR asks for backend task → load the
skill file in full, follow its phases, do not skip any step, do not improvise
citations. Every factual claim about the codebase must cite `file:line`.
```

## Step 3 — Invoke

In the Gemini CLI conversation:

```
explore the bookmark-API feature
```

or

```
run the work skill for ticket 202604/add-bookmark-api
```

(Example task names — replace with whatever you're actually working on.)

Gemini reads `GEMINI.md`, sees the skill mapping, loads the skill file, and follows it.

## Option B — Per-conversation paste

For ad-hoc use without modifying `GEMINI.md`:

1. Open `skills/<name>/SKILL.md`
2. Paste the body as the opening user message with a wrapper:

```
Follow the procedure below as a hard specification. Do not skip phases.
Do not start coding until the procedure tells you to.

<paste SKILL.md body>

My task: <your task>
```

## Supporting files

Gemini will not automatically open the `*.md` siblings of `SKILL.md`. When a phase references one, tell Gemini explicitly:

```
For phase 1.5 (Database), open and apply:
  ~/backend-dev-skills/skills/explore/database-first.md
```

Or include relative imports in your `GEMINI.md`:

```markdown
## Skill Support Files

When a skill references a sibling file, read it from:
  ~/backend-dev-skills/skills/<skill>/<file>.md

Common ones:
- explore/database-first.md          (phase 1.5)
- explore/reuse-checklist.md         (phase 2B)
- explore/report-template.md         (phase 4 output format)
- work/reuse-discipline.md           (phase 1)
- work/implementation-playbook.md    (phase 2)
- work/wrap-up-template.md           (phase 5)
- verify/payload-sourcing.md         (phase 1)
- verify/curl-harness.md             (phase 2)
- verify/jwt-auth-reference.md       (phase 2 — adapt to your stack)
- verify/verify-template.md          (phase 6)
- qa-engineer/environment-gates.md   (phase 0-1 — env preflight, prod guardrails)
- qa-engineer/playwright-playbook.md (phase 2 — cli→spec translation)
- qa-engineer/repeatable-suite.md    (phase 2/6 — update-in-place discipline)
- qa-engineer/report-template.md     (phase 6 — qa.md format)
```

## Gemini-specific quality notes

- Gemini tends to be less strict about "stop here and wait" instructions than Claude. Reinforce at phase boundaries: "You are at end of phase 4. Stop. Summarize. Wait for user approval before phase 5."
- Gemini's code-citation discipline is improving but occasionally drifts. If you see a claim without `file:line`, say "add citations" — it will.
- `curl` execution in verify/phase 3 works well in Gemini CLI since it has shell access.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Gemini skips phases | Add at top of conversation: "Execute phase N. Stop. Wait for my go-ahead before phase N+1." |
| Gemini invents file paths | "Grep first. Only cite `file:line` after you've opened the file." |
| Gemini won't stop after phase 4 | Reinforce iron law: "No code until explore.md is written and I approve. Do not start phase 5 without my explicit 'proceed'." |
