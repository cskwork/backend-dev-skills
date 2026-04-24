# Install on OpenAI Codex / Codex CLI

Codex doesn't have native "skills" — but it does read repo-level instruction files. You install the skills by appending their prompts to `AGENTS.md` (Codex's canonical project-instruction file) or by pasting the generic prompt prefix at the start of a conversation.

## Option A — Repo-level install (persistent, recommended)

Codex auto-loads `AGENTS.md` from the repo root. Append the skill references to it:

### Step 1 — Clone this repo somewhere portable

```bash
git clone https://github.com/cskwork/backend-dev-skills.git ~/backend-dev-skills
```

### Step 2 — In your project's `AGENTS.md`, append

```markdown
# Backend Development Pipeline

This project uses a 4-skill pipeline for non-trivial backend/fullstack work. When the
user mentions "explore", "work", "verify", or "qa" (or asks for a feature / bug
fix / post-deploy QA in a legacy / multi-service area), load and follow the
corresponding skill from `~/backend-dev-skills/skills/<name>/SKILL.md`.

Skills:
- `explore`     — evidence-based plan before any code. Output: `.backend/<YYYYMM>/<slug>/explore.md`
- `work`        — TDD implementation against the plan. Output: `work.md` + `docs/features|bugs/<date>-<slug>.md`
- `verify`      — live HTTP-level proof (localhost). Output: `verify.md` + `harness/` + `fixtures/` + `runs/`
- `qa-engineer` — browser E2E against deployed env (dev/stg/prod) using playwright-cli as probe and Playwright Test as harness. Output: `qa.md` + `e2e/specs/*.spec.ts` (repeatable, updated in-place on re-invocation)

Iron laws:
- No code changes until `/explore` has produced an approved `explore.md`
- No "done" claim until fresh verification output is in hand
- No "production-ready" claim until `/verify` says `status: pass`
- No "live for users" claim until `/qa-engineer` says `status: pass` with trace+video recorded against the deployed URL
- `/verify` is localhost-only; `/qa-engineer` is deployed-env-only (they refuse each other's targets)

Before running any of these, read the corresponding `SKILL.md` in full.
```

Now in any Codex conversation: "run the explore skill for the bookmark feature" → Codex loads `explore/SKILL.md` and follows the 5 phases.

### Step 3 — Make the skills reachable

Either:
1. Symlink into the project: `ln -s ~/backend-dev-skills/skills skills-ref` (referenced by `AGENTS.md` above)
2. Or copy the skills/ folder into the repo if you want to version-pin them

## Option B — Per-conversation paste (ad hoc)

For one-off use without modifying `AGENTS.md`:

1. Open the skill's `SKILL.md` (e.g. `skills/explore/SKILL.md`)
2. Copy it verbatim (the entire file — frontmatter is ignored by Codex, the body is what matters)
3. Paste into the Codex conversation as the opening message:

```
You are about to help me with a backend task. Before doing anything else,
read and follow the procedure below. Treat it as a hard specification — do
not skip phases, do not improvise citations, do not start coding until the
procedure tells you to.

---PASTE SKILL.md BODY HERE---

Now, my task is: <your task>
```

Codex will follow the procedure for that conversation. This works but burns context — prefer Option A for recurring use.

## What to expect

Codex-class models handle the symbolic shorthand (`→`, `⊕`, `file:line`, etc.) well when the legend is in-context. The skill files include the legend.

Quality-of-life caveats vs Claude Code:
- No slash-command syntax — you invoke by name ("run the explore skill", "do the work skill now")
- No automatic supporting-file discovery — if the skill references `database-first.md`, tell Codex to read it: "open skills/explore/database-first.md and apply the phase-1.5 procedure"
- The skill's "STOP" instructions (wait for user approval before code) work as long as the model respects the instruction — Codex generally does, but confirm after phase 4

## Supporting-file reference

When asking Codex to run a phase that references a supporting file, point it explicitly:

| Skill | Phase | Supporting file |
|---|---|---|
| explore | 1.5 (Database) | `skills/explore/database-first.md` |
| explore | 2B (Reuse) | `skills/explore/reuse-checklist.md` |
| explore | 4 (Report) | `skills/explore/report-template.md` |
| work | 1 (Reuse Re-check) | `skills/work/reuse-discipline.md` |
| work | 2 (Implementation) | `skills/work/implementation-playbook.md` |
| work | 5 (Wrap-Up) | `skills/work/wrap-up-template.md` |
| verify | 1 (Payload) | `skills/verify/payload-sourcing.md` |
| verify | 2 (Harness) | `skills/verify/curl-harness.md` |
| verify | 2 (Auth) | `skills/verify/jwt-auth-reference.md` (adapt to your stack) |
| verify | 6 (Wrap-Up) | `skills/verify/verify-template.md` |
| qa-engineer | 0-1 (Env gates) | `skills/qa-engineer/environment-gates.md` |
| qa-engineer | 2 (Spec synthesis) | `skills/qa-engineer/playwright-playbook.md` |
| qa-engineer | 2/6 (Update discipline) | `skills/qa-engineer/repeatable-suite.md` |
| qa-engineer | 6 (Wrap-Up) | `skills/qa-engineer/report-template.md` |

## Troubleshooting

| Symptom | Fix |
|---|---|
| Codex skips phases | Add the iron-law reminder at the top of your conversation: "Do not skip any phase. Stop and ask if a phase is unclear." |
| Output mixes prose into §2+ | Remind it: "Per the symbol legend, §2 onward is symbolic shorthand. Convert paragraphs to bullets + citations." |
| Citations missing | "Every factual claim requires a `file:line` citation. Re-check each statement." |
