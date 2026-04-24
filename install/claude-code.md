# Install on Claude Code

Claude Code has native support for "skills" as markdown files with frontmatter. Each skill becomes a slash command.

## Step 1 — Copy the skill folders

```bash
# Clone this repo (or download the skills/ folder)
git clone https://github.com/cskwork/backend-dev-skills.git
cd backend-dev-skills

# Copy all four skills into your Claude Code skills directory
mkdir -p ~/.claude/skills
cp -r skills/explore     ~/.claude/skills/
cp -r skills/work        ~/.claude/skills/
cp -r skills/verify      ~/.claude/skills/
cp -r skills/qa-engineer ~/.claude/skills/
```

That's the entire install for `/explore`, `/work`, `/verify`. No settings.json edits, no plugins, no restart required.

**For `/qa-engineer` only**, install Playwright once per machine (or per repo if you prefer local installs):

```bash
# Global install (simplest):
npm install -g @playwright/test @playwright/cli
npx playwright install        # downloads browser binaries (~300MB, one-time)

# OR per-project, from the repo where you'll run QA:
npm install --save-dev @playwright/test @playwright/cli
npx playwright install
```

The skill will detect Playwright during Phase 1 and refuse to proceed if it's missing.

## Step 2 — Verify

Open Claude Code in any project and type:

```
/explore
```

Claude will load `~/.claude/skills/explore/SKILL.md` and follow the 5-phase procedure. Same for `/work`, `/verify`, and `/qa-engineer`.

## Step 3 (optional) — Add a company-announcement banner

If you want all sessions on your machine (or across your team) to see the pipeline reminder on startup, add an entry to the `companyAnnouncements` array in `~/.claude/settings.json`:

```json
{
  "companyAnnouncements": [
    "[Backend Pipeline] Default flow for any legacy/multi-service backend task (feature or bug)\n[Backend Pipeline] 1. /explore     -> evidence-based plan @ .backend/<YYYYMM>/<slug>/explore.md  (NO CODE until approved)\n[Backend Pipeline] 2. /work        -> TDD implementation + paired work.md + docs/features|bugs entry\n[Backend Pipeline] 3. /verify      -> curl E2E gate (localhost) + verify.md (status: pass | fail-escalated | fail-user-required)\n[Backend Pipeline] 4. [deploy]     -> ship to dev/stg/audit/prod\n[Backend Pipeline] 5. /qa-engineer -> browser E2E gate (deployed URL) + repeatable Playwright suite under e2e/specs/ + qa.md"
  ]
}
```

Claude Code displays each announcement once per session.

## Step 4 (optional) — Team-wide install

For a shared team install, check this repo into your monorepo under any path, and symlink or reference from each team member's `~/.claude/skills/`. Or use your team's dotfiles pipeline.

## How Claude Code discovers skills

- Each skill is a folder under `~/.claude/skills/<name>/` with a `SKILL.md` file
- `SKILL.md` starts with YAML frontmatter: `name`, `description`
- The `description` field is what tells Claude *when* to use the skill — keep it specific
- Supporting files (`*.md` siblings) are read lazily when the main `SKILL.md` references them

## File layout after install

```
~/.claude/skills/
  explore/
    SKILL.md
    database-first.md
    report-template.md
    reuse-checklist.md
  work/
    SKILL.md
    implementation-playbook.md
    reuse-discipline.md
    wrap-up-template.md
  verify/
    SKILL.md
    curl-harness.md
    jwt-auth-reference.md
    payload-sourcing.md
    verify-template.md
  qa-engineer/
    SKILL.md
    playwright-playbook.md      (playwright-cli commands + cli→spec translation)
    repeatable-suite.md         (update-in-place discipline for accumulated specs)
    environment-gates.md        (dev/stg/prod URL preflight + prod write-block fixture)
    report-template.md          (qa.md 11-section format + docs-entry appendix)
```

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `/explore` not recognized | `SKILL.md` missing or frontmatter `name:` field differs from folder name |
| Claude ignores the skill | `description` field too vague — Claude's routing decides based on it |
| Supporting `.md` files not referenced | Check that `SKILL.md` cross-references them by relative filename |

## Uninstall

```bash
rm -rf ~/.claude/skills/explore ~/.claude/skills/work ~/.claude/skills/verify ~/.claude/skills/qa-engineer
```

And remove the `companyAnnouncements` entry from `settings.json` if you added one.
