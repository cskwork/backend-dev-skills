# Install on any other agent (generic prompt)

If your agent isn't Claude Code, Codex, or Gemini — this is the universal install. Any LLM that accepts a system prompt (or long user message) will work: Cursor, Aider, Continue.dev, custom API integrations, self-hosted models via Ollama, etc.

## The universal invocation pattern

Every skill is just markdown. Two components:

1. **System-prompt prefix** (paste once per conversation or bake into your agent's system prompt)
2. **Skill body** (copy-paste the specific skill's `SKILL.md` body when you need it)

## Step 1 — The system-prompt prefix

Paste this at the top of your conversation (or into your agent's system prompt):

```
You will assist with backend/fullstack development tasks using a disciplined
4-skill pipeline. The skills are `explore`, `work`, `verify`, and `qa-engineer`.

When the user asks for a feature, bug fix, verification, or post-deploy QA:

1. Identify which skill applies:
   - "explore <thing>" / new feature / bug report       → /explore
   - "work on <thing>" / approved plan exists           → /work
   - "verify <thing>" / code is written (localhost)     → /verify
   - "qa <thing>" / deployed to dev/stg/prod            → /qa-engineer

2. When a skill is invoked, follow its procedure exactly as written. Treat
   every phase boundary as a hard stop. Do not skip phases. Do not improvise.

3. Every factual claim about the codebase must cite `file_path:LINE`. If you
   cannot cite, you do not know — say so explicitly.

4. The output artifacts are files on disk:
   - /explore      writes `.backend/<YYYYMM>/<slug>/explore.md`
   - /work         writes `work.md` + `docs/features|bugs/<date>-<slug>.md`
   - /verify       writes `verify.md` + `harness/*.sh` + `fixtures/*.json` + `runs/run-<N>.log`
   - /qa-engineer  writes `qa.md` + `e2e/specs/*.spec.ts` + `e2e/artifacts/run-<ts>-<env>/*`

   Use your file-write capability to persist them. Do not output the report
   only into chat — it must land on disk under the correct path.

5. Iron laws (violating these is a skill failure, not a stylistic choice):
   - No code changes until /explore has produced an approved explore.md
   - No "done" claim until fresh verification output is in hand
   - No "production-ready" claim until /verify says `status: pass`
   - No "live for users" claim until /qa-engineer says `status: pass` with trace+video against the deployed URL
   - /verify is localhost-only; /qa-engineer is deployed-env-only (they refuse each other's targets)

6. /qa-engineer has two extra rules:
   - Prereq: Node + `@playwright/test` + `playwright-cli` must be installable; the skill will check Phase 1
   - On re-invocation for the same ticket, UPDATE existing specs in place — never regenerate from scratch

When I invoke a skill, I will paste its full SKILL.md body. Load it and follow
it. If you need a supporting file (referenced within SKILL.md), ask me to paste
it, or open it yourself if you have filesystem access.
```

## Step 2 — Invoke a skill

When you want to run one of the skills:

```
Run the /explore skill. Here is its SKILL.md:

---

<paste the full contents of skills/explore/SKILL.md>

---

My task: add a bookmark endpoint so students can pin a chapter
```

The agent follows the 5 phases, stops at phase 4, and waits for your approval before `/work`.

## Step 3 — Supporting files

When a phase references a supporting file (e.g. `database-first.md`, `report-template.md`), either:

1. Paste it when the agent asks, or
2. If your agent has filesystem access, point it: "open `skills/explore/report-template.md` and follow its format for §4"

## The pattern, visualized

```
System prompt:                  one-time, establishes iron laws + skill routing
     │
     ▼
User: "/explore <task>"         you invoke the skill
     + paste SKILL.md body      provides the procedure in-context
     │
     ▼
Agent follows phases 0-4        evidence-based plan, citations, etc.
     │
     ▼
Agent writes explore.md to disk artifact persists for future sessions
Agent STOPS                     waits for approval
     │
     ▼
User: "approved, /work now"     next skill
     + paste work/SKILL.md      ...
```

## What "works on any agent" really means

- **Anything with a long context window** (Claude, GPT-4 class, Gemini 1.5+) handles a pasted SKILL.md comfortably. The largest one (`verify/SKILL.md`) is ~25k characters.
- **Agents with filesystem tools** (Cursor, Aider, Claude Code, Codex CLI, Gemini CLI) can read supporting files directly — you only paste SKILL.md.
- **Chat-only agents** (plain ChatGPT, Claude.ai web) require you to paste each supporting file when the phase needs it. Slower but works.
- **Local models via Ollama / LM Studio**: use a 70B+ instruct model; smaller models tend to skip phases or invent citations. Qwen2.5-Coder 32B+ is the small-model floor where the pipeline still works reliably.

## Minimal self-test

To confirm the install works on your agent:

1. Paste the system-prompt prefix from step 1
2. Paste `skills/explore/SKILL.md` body
3. Say "explore adding a 'last-login-at' column to the users table"

A compliant agent will:
- Classify as Feature
- Ask about your stack / ORM / existing users table
- Propose the 5 phases, not dive into code
- Stop after phase 4

If it dives into code generation without asking for codebase access or running grep — the agent is not following the skill. Reinforce: "Do not skip phases. Do not write code. Produce the evidence-based report first."

## Customization

- Replace every `.backend/` path with your preferred workspace convention
- Replace the iron laws with your team's ship gates (they are just a phrasing)
- Fork `skills/verify/jwt-auth-reference.md` to match your auth scheme
- Fork `skills/work/implementation-playbook.md` to match your stack (layer rules, transaction patterns, etc.)
- Fork `skills/qa-engineer/environment-gates.md` to match your env URL conventions (`dev-*`, `stg-*`, `prod.*`, internal hostnames)
- Fork `skills/qa-engineer/playwright-playbook.md §Auth` with your app's login pattern (SSO provider, JWT injection path, session cookie names)

The 5/6/7/7 phase structure is the load-bearing part. Everything else is adjustable.
