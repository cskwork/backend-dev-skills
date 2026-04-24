---
name: Bug report — agent skipped a phase or broke an iron law
about: The agent violated the procedure and the skill did not stop it
labels: bug
---

## Which skill

- [ ] `/explore`
- [ ] `/work`
- [ ] `/verify`
- [ ] `/qa-engineer`

## Which agent

- [ ] Claude Code
- [ ] OpenAI Codex / CLI
- [ ] Gemini CLI
- [ ] Cursor
- [ ] Aider
- [ ] Continue.dev
- [ ] Other: _________

## What the agent did wrong

A one-sentence summary of the violation (e.g. "agent skipped Phase 1.5 database-first investigation and wrote code anyway").

## Expected behavior per the skill

Cite the section of `SKILL.md` the agent was supposed to follow (e.g. "SKILL.md §Phase 1.5 requires DESCRIBE before proposing changes").

## Reproduction

1. Invoked the skill with: `...`
2. Agent responded with: `...` (paste the relevant output excerpt)
3. Expected: `...`

## Your fix idea (optional)

Sometimes the fix is to tighten the skill wording; sometimes it's an agent-specific quirk worth documenting in the install guide. What do you think?
