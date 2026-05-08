# Changelog 2026-05-08

## Summary

- Updated the public backend-dev-skills pipeline from four skills to five by adding `/reproduce`.
- Refreshed current `/explore`, `/work`, `/verify`, and `/qa-engineer` skill content from the active Codex versions, then removed project-specific service names, auth details, URLs, and Korean-only output requirements.
- Updated README and install guides so Claude Code, Codex, Gemini, and generic agents know the full `/reproduce -> /explore -> /work -> /verify -> /qa-engineer` flow.

## Rationale

The public repository must stay stack-agnostic. Any internal project defaults, real URLs, credential patterns, service names, and organization-specific wording were generalized to examples or placeholders.

## Verification

- Searched for internal project identifiers, URLs, secrets, and Korean-only content before commit.
- Checked generated Markdown structure with repository-local grep checks.
