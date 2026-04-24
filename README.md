# Backend Dev Skills

> Turn any AI coding assistant into a **disciplined backend engineer**. Four opinionated, evidence-first skills (`/explore` → `/work` → `/verify` → `/qa-engineer`) that refuse to ship guesses — they demand citations, fresh test output, real `curl` proof, and a real browser trace before calling anything "done".

<p align="center">
  <a href="https://github.com/cskwork/backend-dev-skills/stargazers"><img alt="GitHub stars" src="https://img.shields.io/github/stars/cskwork/backend-dev-skills?style=social"></a>
  <a href="https://github.com/cskwork/backend-dev-skills/network/members"><img alt="GitHub forks" src="https://img.shields.io/github/forks/cskwork/backend-dev-skills?style=social"></a>
  <a href="https://github.com/cskwork/backend-dev-skills/issues"><img alt="GitHub issues" src="https://img.shields.io/github/issues/cskwork/backend-dev-skills"></a>
  <a href="https://github.com/cskwork/backend-dev-skills/commits/main"><img alt="Last commit" src="https://img.shields.io/github/last-commit/cskwork/backend-dev-skills"></a>
  <a href="LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/License-MIT-green.svg"></a>
</p>

<p align="center">
  <img alt="Pure markdown" src="https://img.shields.io/badge/install-pure%20markdown-blue">
  <img alt="No runtime" src="https://img.shields.io/badge/runtime-none-lightgrey">
  <img alt="Works anywhere" src="https://img.shields.io/badge/agents-Claude%20Code%20%7C%20Codex%20%7C%20Gemini%20%7C%20Cursor-purple">
  <img alt="Stack agnostic" src="https://img.shields.io/badge/stack-agnostic-brightgreen">
</p>

**Works with:** Claude Code · OpenAI Codex · Gemini CLI · Cursor · Aider · Continue.dev · any agent that reads a prompt.

## Table of contents

- [The problem](#the-problem)
- [The Iron Laws](#the-iron-laws)
- [What each skill delivers](#what-each-skill-delivers)
- [Quick start (90 seconds)](#quick-start)
- [Folder contract (per ticket)](#folder-contract-per-ticket)
- [Install — per-agent](#install--per-agent)
- [Who this is for](#who-this-is-for)
- [FAQ](#faq)
- [Roadmap & contributing](#roadmap)
- [License](#license)

---

## The problem

Generic AI coding agents ship greenfield code well. They fall apart on **a 200k-line service you've never opened, with three databases, two message buses, and an HTTP client nobody has touched since 2022**.

They will:

- Invent a utility you already have five of
- "Fix" the symptom instead of the root cause
- Write a passing unit test while the real endpoint still 500s
- Drive-by refactor five files that weren't in scope
- Declare "done" before ever running a real `curl`
- Mark a deploy "live" before opening a browser

These four skills encode the pipeline a senior backend engineer runs in their head — forced, step-by-step, into the agent.

```
┌──────────┐   ┌───────┐   ┌──────────┐   [deploy]   ┌──────────────┐
│ /explore │─▶─│ /work │─▶─│ /verify  │────────────▶─│ /qa-engineer │
└──────────┘   └───────┘   └──────────┘              └──────────────┘
   evidence       code         local HTTP               deployed browser
   before         with         proof                    E2E + repeatable
   code           tests        (localhost)              Playwright suite
```

Each skill produces a **file-system artifact** (`explore.md`, `work.md`, `verify.md`, `qa.md` + `e2e/specs/*.spec.ts`) under a per-ticket folder, so future sessions — human or AI — can reload context without re-investigating, and the Playwright suite re-runs on every subsequent deploy.

---

## The Iron Laws

Every skill enforces hard stops. These are not suggestions.

| Skill | Iron Law |
|---|---|
| `/explore` | **No code changes until an evidence-based plan (with `file:line` citations) exists and is approved.** |
| `/work` | **No "done" claim until fresh verification output is in hand AND paired docs are written.** |
| `/verify` | **No "production-ready" claim until every assertion passes against a live local service with recorded `curl -i` evidence.** |
| `/qa-engineer` | **No "live for users" claim until a real browser completes the flow against the deployed URL, with trace + video recorded, and the result is re-runnable from `e2e/specs/` on the next deploy.** |

When the agent is tempted to skip a phase, the skill refuses. That refusal is the point.

---

## What each skill delivers

### [`/explore`](./skills/explore/) — evidence-based plan before any code

5 phases. Produces `.backend/<YYYYMM>/<slug>/explore.md` with:

- §1 Executive summary — **plain-language prose for non-developers**
- §2+ AI-optimized symbolic body — every claim cites `file:line`
- Database-first investigation (live `DESCRIBE` / schema grounding)
- Reuse inventory (forces the agent to find existing utilities before writing new ones)
- Blast-radius analysis (callers, shared state, cross-service effects)
- Proposed changes as a `file · act · reuse/new` table

### [`/work`](./skills/work/) — TDD implementation against the plan

6 phases. Reads `explore.md` as a **contract** and implements it:

- Red → Green → Refactor per change unit
- Surgical diff review: every hunk must trace to §7 of the plan
- Fresh verification evidence per command (no paraphrased results)
- Paired outputs: `work.md` (AI-optimized) + `docs/features|bugs/<date>-<slug>.md` (human prose)

### [`/verify`](./skills/verify/) — live HTTP-level proof, not unit-test theater

7 phases. Gates merge with real `curl` evidence:

- Payload sourcing: user → saved fixtures → live DB sample → synthesized (in that order)
- Reusable curl harness per endpoint, with `happy / boundary / negative / regression` variants
- Bounded re-work loop: on failure, hands `/work` a **specific delta** (not "fix it"); caps at 3 iterations then escalates
- Records every request/response pair verbatim to `runs/run-<N>.log`
- **Localhost by default** — non-localhost targets are opt-in behind a recorded consent gate (per-request for prod)

### [`/qa-engineer`](./skills/qa-engineer/) — post-deploy browser E2E with a repeatable spec suite

7 phases. Runs *after* `/verify` passes and the change has been deployed:

- **Deployed-env only** — refuses `localhost` (opposite of `/verify`); enforces env-URL matching and slug-scoped prod consent
- Uses Microsoft's [`playwright-cli`](https://github.com/microsoft/playwright-cli) as an interactive **probe**, then translates each walked flow into a `.spec.ts` file — the cli is how you generate, the spec file is what you re-run
- Produces a durable Playwright suite under `.backend/<YYYYMM>/<slug>/e2e/specs/` that CI (or a human) re-runs on every subsequent deploy via `npx playwright test specs/smoke.spec.ts`
- **Update-in-place discipline** on re-invocation: unchanged flows keep their spec verbatim (`git diff e2e/specs/` must be empty for untouched flows); only contract-drifted flows get minimum-delta edits; accumulated debugging survives
- 3-attempt flake protocol — a variant that passes 3× and fails 2× is `pass-with-flake`, not `pass`
- Evidence per run: Playwright trace (`trace.zip`), video, screenshots, console log, HAR — all redacted of PII
- Prod runs default to **read-only** via a route-level write-block fixture; writes require explicit per-step consent
- Bounded rework loop: FAIL-UI/BACKEND → `/work` delta; FAIL-CONTRACT → `/explore`; cap 2 upstream re-invocations

---

## Quick start

<a name="quick-start"></a>

Copy the skills, then invoke by name.

```bash
git clone https://github.com/cskwork/backend-dev-skills.git
cd backend-dev-skills

# Claude Code — native slash commands
mkdir -p ~/.claude/skills
cp -r skills/explore skills/work skills/verify skills/qa-engineer ~/.claude/skills/
```

Open any project, type `/explore`, and start:

```
/explore add a bookmark endpoint so users can pin a document
```

The agent asks "Bug or Feature?", runs grep/DB investigation, and writes `.backend/<YYYYMM>/<slug>/explore.md`. It then **stops** and waits for your approval before `/work` begins.

For other agents (Codex, Gemini, Cursor, Aider, …) see [the install guides](./install/).

**Prerequisite for `/qa-engineer` only:** Node.js + `@playwright/test` + `playwright-cli`. The skill detects missing binaries in Phase 1 and offers the install command.

---

## Folder contract (per ticket)

All four skills share a single workspace per ticket:

```
.backend/
  202604/                               ← YYYYMM month bucket
    add-bookmark-api/                   ← kebab-case slug (≤50 chars)
      explore.md                        ← /explore
      work.md                           ← /work
      verify.md                         ← /verify
      qa.md                             ← /qa-engineer
      harness/                          ← /verify — reusable curl scripts
        _env.sh
        _assert.sh
        POST-bookmarks.sh
      fixtures/                         ← /verify — per-variant payloads (PII-redacted)
        POST-bookmarks.happy.json
        POST-bookmarks.boundary.json
        POST-bookmarks.negative.json
      runs/                             ← /verify — raw curl -i logs per iteration
        run-1.log
        run-2.log
      e2e/                              ← /qa-engineer — repeatable Playwright suite
        _env.sh
        _login.sh
        playwright.config.ts
        storageState.dev.json           ← redacted auth state per env
        specs/
          smoke.spec.ts                 ← <90s critical-path, re-run on every deploy
          <flow>.spec.ts                ← full happy/boundary/negative per flow
        scripts/<flow>.sh               ← playwright-cli probes (debug/teach)
        fixtures/<flow>.json
        artifacts/
          run-20260421-1430-dev/        ← one folder per execution
            trace.zip
            video.webm
            console.log
            network.har
```

This layout is **the contract** between the four skills. Any AI session (or human) opening the folder gets full context in under a minute, and `e2e/specs/` gives CI (or a human) a one-command post-deploy gate for every subsequent release.

---

## Install — per-agent

Skills are **just markdown prompts** — no binaries, no runtime. Copy them to the right location for your agent and invoke by name.

| Agent | Guide | Invocation |
|---|---|---|
| **Claude Code** | [install/claude-code.md](./install/claude-code.md) | `/explore`, `/work`, `/verify`, `/qa-engineer` |
| **OpenAI Codex / CLI** | [install/codex.md](./install/codex.md) | Append to `AGENTS.md` or paste per conversation |
| **Gemini CLI** | [install/gemini.md](./install/gemini.md) | Append to `GEMINI.md` or paste per conversation |
| **Cursor / Aider / other** | [install/generic.md](./install/generic.md) | Paste the system-prompt prefix, then each SKILL.md body |

**90 seconds to install.** See the per-agent guides for exact paths.

---

## Who this is for

These skills were forged in a real 10-service backend monorepo with multiple databases, message queues, caches, SSE, WebSockets, and an external SSO vendor. They are opinionated exactly because legacy codebases punish under-opinionated agents.

They earn their keep the moment your codebase has:

- **2+ services** with shared state
- **Any SQL database** older than six months
- **Any** developer who is not the person who originally wrote the module
- **Any** compliance/audit requirement on what shipped and why

If those sound familiar — this repo is for you.

**Stack-agnostic by design.** The pipeline works for:

- Spring Boot / Java (MyBatis, JPA) — implementation-playbook included
- Node.js / TypeScript (Express, NestJS, Fastify) — adapt the playbook
- Python (FastAPI, Django, Flask)
- Go, Rust, Kotlin, C# — anywhere HTTP endpoints and a database exist
- Any ORM, any SQL dialect, any JWT / OAuth / session auth scheme
- Any frontend (Vue, React, Svelte, Angular) — `/qa-engineer` is framework-agnostic

Opinion lives in the **process** (cite `file:line`, re-grep before adding new code, prove with fresh output). Stack specifics live in two files you fork: `skills/work/implementation-playbook.md` and `skills/verify/jwt-auth-reference.md`.

---

## FAQ

**Q: Isn't this overkill for a one-line fix?**
A: Yes. All four skills have explicit "skip for typo fixes / config tweaks / doc changes" clauses. Use judgment. The skills exist for the *other* 95% of work where shortcuts cost incidents.

**Q: I already have Cursor / Copilot — why these?**
A: Those tools optimize for code-completion speed. These skills optimize for **not breaking production in a codebase the author has never seen before**. Different problem.

**Q: Is this Claude-only?**
A: No. The skills are pure markdown; they work wherever an agent can load a system prompt. Claude Code gets native slash-command support; other agents invoke by name.

**Q: Will this slow me down?**
A: `/explore` takes 5–15 min. `/verify` takes 5–10 min. You buy back hours the first time either one catches a contract drift before prod.

**Q: Can I modify the skills for my stack?**
A: Yes — that's the intended use. Fork, localize service names, ports, auth patterns, ship internally. MIT-licensed. The **5/6/7/7 phase structure** is load-bearing; everything else is adjustable.

**Q: Does this work with local / self-hosted models?**
A: Yes, on instruct-tuned 70B+ class models (Qwen2.5-Coder 32B is the practical floor). Smaller models tend to skip phases or fabricate citations. See [install/generic.md](./install/generic.md).

**Q: What about frontend-only changes?**
A: `/qa-engineer` covers deployed browser QA for any frontend. `/verify` is backend-only; skip it for pure frontend diffs. `/explore` and `/work` work fine for frontend too — fork `implementation-playbook.md` to your framework's conventions.

---

## Roadmap

<a name="roadmap"></a>

Contributions welcome — the pipeline generalizes further than one stack. Especially appreciated:

- Alternate `implementation-playbook.md` variants: Node/TypeScript, Python (FastAPI/Django), Go, Rust, Kotlin
- Alternate `jwt-auth-reference.md` variants: OAuth2 client-credentials, session-cookie, API-key, mTLS
- Alternate `environment-gates.md` variants: Kubernetes-routed envs, Vercel/Netlify preview URLs, feature-branch staging
- CI recipes: GitHub Actions / GitLab CI / CircleCI snippets that re-run `e2e/specs/smoke.spec.ts` on every deploy
- Translations of the human-prose `docs/features|bugs/*.md` templates to other languages

Open an issue or PR — happy to review and merge.

### How to contribute

1. Fork the repo
2. For a new stack variant, copy the relevant reference file (e.g. `skills/work/implementation-playbook.md`) to a sibling with a stack suffix (`implementation-playbook.node.md`), edit, and open a PR
3. For a bug in the core procedure, open an issue describing the failure mode and the agent that hit it
4. Run `/explore` → `/work` → `/verify` on your own change (dogfood the pipeline)

No CLA, no contributor-license boilerplate. MIT in, MIT out.

---

## License

MIT — fork, adapt, ship internally, build a startup on top of it. See [LICENSE](./LICENSE).

---

## Star history

If these skills save you one incident, consider starring the repo so other backend engineers can find them.

---

**Built by a backend engineer who got tired of AI agents confidently shipping broken code into legacy systems.** If you've felt that pain — this is for you.
