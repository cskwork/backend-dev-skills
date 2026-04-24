---
name: Stack variant proposal
about: You want to contribute an implementation-playbook or jwt-auth-reference for a new stack
labels: enhancement, stack-variant
---

## Which file

- [ ] `skills/work/implementation-playbook.<stack>.md`
- [ ] `skills/verify/jwt-auth-reference.<scheme>.md`
- [ ] `skills/qa-engineer/environment-gates.<convention>.md`
- [ ] `skills/qa-engineer/playwright-playbook.<framework>.md`

## Stack / scheme / convention

E.g. "NestJS + TypeORM", "FastAPI + SQLAlchemy", "Go + gorm", "Kotlin + Spring WebFlux", "OAuth2 device-code flow", "Kubernetes env-per-PR".

## Why it's useful

A sentence or two — what's different about this stack that the current file doesn't cover? Who would use the variant?

## Scope

What you plan to cover (layer discipline, transaction pattern, test scaffolding, test commands, cross-cutting guardrails, etc.). Aim for the same depth as the reference file.

## Anything weird I should know before PR'ing

Unclear naming convention? Preferred directory layout? Ping here and I'll reply before you sink time into it.
