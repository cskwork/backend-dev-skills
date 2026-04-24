# Install guides

Pick your agent and follow the guide. All skills are pure markdown — no binaries, no runtime.

| Agent | Guide | Install effort |
|---|---|---|
| **Claude Code** | [claude-code.md](./claude-code.md) | 3 `cp` commands. Native slash-command support. |
| **OpenAI Codex / CLI** | [codex.md](./codex.md) | Append to `AGENTS.md`, or paste SKILL.md per conversation |
| **Gemini CLI** | [gemini.md](./gemini.md) | Append to `GEMINI.md`, or paste per conversation |
| **Anything else** | [generic.md](./generic.md) | Paste system-prompt prefix once + SKILL.md per invocation |

## Which agent should I use?

Not our business — the skills are designed to be portable. A few practical notes:

- **Best native integration:** Claude Code (slash commands, automatic supporting-file discovery)
- **Best for team-wide install:** Codex or Gemini via repo-level `AGENTS.md` / `GEMINI.md` (version-pinned with the repo)
- **Most flexible:** Generic prompt (works on Cursor, Aider, Continue.dev, self-hosted, etc.)

## Update policy

These skills evolve. When upgrading:

1. `git pull` this repo
2. Re-run the `cp` commands (Claude Code) or re-reference the path (others)
3. Check `CHANGELOG.md` for breaking changes to the folder contract

The `.backend/<YYYYMM>/<slug>/` folder contract is stable — your existing tickets keep working across skill upgrades.
