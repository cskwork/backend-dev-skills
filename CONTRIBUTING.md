# Contributing

Thanks for considering a contribution. This project is pure markdown — no binaries, no runtime — which means the only thing that can "break" is the **procedure** the skills encode. PRs that strengthen the procedure or widen the stack coverage are the most valuable.

## What we're looking for

**Highest value**

- **Alternate stack variants** of the two fork-expected files — if you use Node/TypeScript, Python, Go, Rust, Kotlin, .NET, Rails, Phoenix, etc., a sibling variant is immediately useful:
  - `skills/work/implementation-playbook.md` → `implementation-playbook.<stack>.md`
  - `skills/verify/jwt-auth-reference.md` → `jwt-auth-reference.<auth-scheme>.md`
- **Alternate env conventions** for `skills/qa-engineer/environment-gates.md` (Kubernetes-routed, Vercel/Netlify preview URLs, feature-branch staging, tunnel-based dev, etc.)
- **CI recipes** that re-run `e2e/specs/smoke.spec.ts` on every deploy (GitHub Actions, GitLab CI, CircleCI, Buildkite, Jenkins)
- **Bug reports** — if an agent skipped a phase, fabricated a citation, or ignored an iron law, file it. Include which agent, which skill, which phase, and what the output was.

**Medium value**

- Clarifying a red flag or common rationalization with a concrete example
- Tightening a symbol legend or template to remove ambiguity
- Fixing typos / cross-reference links

**Out of scope**

- Adding runtime code (binaries, scripts that must be installed) — the value of this repo is pure-markdown portability
- Vendor-specific integrations that only work with one commercial agent (the skills must remain agent-agnostic in spirit)
- Adding a build system, package manager, or test harness to the repo itself

## Workflow

1. **Open an issue first** for non-trivial changes (new stack variant, procedure change). A 2-line issue ("I'd like to add a Node.js implementation-playbook — anything I should know?") saves rework.
2. **Fork and branch** from `main`.
3. **Dogfood the pipeline.** Run `/explore` → `/work` on your own change. The `explore.md` and `work.md` are not committed, but the discipline catches scope creep before review does.
4. **Match the existing tone.** Terse, opinionated, citation-demanding. No marketing fluff in the skill bodies.
5. **Open a PR** with a short summary — why, what changed, which agent you tested it on.

## Style

- Follow the existing file's structure (headers, symbol legends, red flags). Variants should be drop-in.
- Code examples in the stack's idiomatic style (Prettier-default for JS, `gofmt` for Go, etc.) — no opinion wars.
- If you invent a new symbol for a symbol legend, **add it to the legend** in the same file. No ad-hoc shorthand.

## Licensing

By contributing you agree your contribution is MIT-licensed under this repo's terms. No CLA, no extra paperwork.

## Questions

Open an issue with the label `question` or start a discussion. Short questions get short answers; long ones get long ones.
