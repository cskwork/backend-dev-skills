# Implementation Playbook (Phase 2 Stack-Specific Guardrails)

Quick-reference for `/work`'s TDD implementation phase. Read the section matching the stack you are editing. Deviations require explicit justification in `work.md`.

> **Fork this file.** The examples below are for a Spring Boot + MyBatis backend and a Vue 3 + Vite frontend. If your stack differs (NestJS, FastAPI, Django, Go, Rails, etc.), copy this file to your fork and replace the sections with your framework's equivalents. The **rules** generalize; the **code snippets** do not.

## Symbol Legend (same as explore.md / work.md)

```
→ calls / depends-on     ← called-by     ⇢ async / event
⊕ add   ⊖ remove   ✎ edit   ≡ reuse   ≈ extend
↑ read (DB)   ↓ write (DB)   ⟳ retry / loop
```

---

## 1. Spring Boot + MyBatis Backend (reference stack)

Applies to any Spring Boot service that uses MyBatis as its data layer. Names like `user-service`, `order-api`, etc. are your own.

### Layer discipline

```
Controller  → parses HTTP, validates shape, returns response envelope
   ↓
Service     → owns business logic, transaction boundaries, outbound calls
   ↓
Mapper/Repo → owns SQL, no conditionals beyond dynamic SQL
   ↓
Database / Cache / Message Queue
```

- Do not push business logic into controllers "because it's shorter". Controllers are thin.
- Do not write SQL in services. New query → new mapper method + mapper XML entry.
- DTOs never carry JPA entities across the controller boundary. Convert at the service layer.

### Transaction annotations (critical)

- **Read paths** (GET endpoints, query-only service methods): `@Transactional(readOnly = true)` — enables read-replica routing where configured. Missing `readOnly = true` on a read path means hitting the primary DB unnecessarily.
- **Write paths** (POST/PUT/PATCH/DELETE): `@Transactional` without `readOnly` — goes to primary.
- Never leave the annotation off a service-layer public method that touches the DB. Not annotating breaks the transaction boundary contract.
- If a read path must also write (e.g., audit log), split: keep the read in `@Transactional(readOnly = true)` and publish an event for the write side.

### MyBatis conventions

- New query: add a method signature to the Mapper interface, add a `<select>` / `<insert>` / `<update>` / `<delete>` block with the **same id** to the XML, add parameter/result types. No inline SQL in services.
- Reuse existing `resultMap` definitions when the row shape matches. Do not define a parallel resultMap for the same table columns.
- Bind parameters with `#{param}`, never `${param}`, except for whitelisted dynamic identifiers (column names from a fixed set). Anything else is SQL injection.
- `LIMIT`/`OFFSET` pagination: match the existing pagination helper pattern — grep for `pageNum`, `pageSize`, `PagingDto` or similar in the target service.

### HTTP clients to downstream services (Feign / WebClient / RestTemplate)

- Reuse an existing `@FeignClient` / WebClient bean if it targets the same downstream service. Add a new method to that client; do not create a second client to the same service.
- Method signatures on cross-service clients are a contract. Adding a required field or changing a path requires matching changes in the downstream service within the same PR (or a backwards-compatible overload).
- Timeouts, retries, and fallbacks are configured at the client level. Do not reimplement retry logic inside a new method.

### Message queue (Kafka / RabbitMQ / SQS / etc.)

- Events flow producer ⇢ consumers. Reuse an existing topic/queue if the payload semantics match.
- Payload DTOs for message queues are a wire contract. Adding a field = add as nullable with a default; removing or renaming = versioned topic or versioned payload.
- `@KafkaListener` (or equivalent) methods must be idempotent. If the plan did not call this out in §5B, flag it in Open Questions.

### Encrypted properties (Jasypt / Vault / SOPS / KMS)

- Any secret added to `application*.yml` must be encrypted by whatever scheme the project already uses. Do not commit plaintext.
- Search existing `application-dev.yml` / `application-prod.yml` for the existing pattern (`ENC(...)` for Jasypt, `vault:` for Vault, etc.) and follow it.

### Test scaffolding (TDD Phase 2)

- Unit tests for services: use Mockito for mapper / HTTP-client dependencies. Do **not** spin up Spring context for a pure service unit.
- Integration tests for mappers: use the MyBatis test slice (`@MybatisTest` or the project's existing test base class — grep for one). Real DB (H2, Testcontainers Postgres/MySQL — follow whatever the service already uses).
- Do not add `@SpringBootTest` where a slice test suffices; it slows the suite and masks coupling issues.
- Test file naming: mirror source — `SomeService` → `SomeServiceTest`, same package.

### Running tests (per service)

```bash
# Full test suite
./gradlew test

# Focused
./gradlew test --tests 'com.example.<pkg>.<Class>'

# Build without tests (smoke compile check)
./gradlew clean build -x test
```

---

## 2. Middleware / BFF Layer (proxy services with added logic)

When a service sits in front of another service and adds business logic on top (backend-for-frontend, gateway, orchestrator):

- Do **not** duplicate a business rule that already exists downstream. If the rule is needed here too, call downstream via the shared client, or move the rule to a shared module, per the plan.
- Request/response shapes between the layer and its downstream are an internal contract — keep them in sync. If a field is added on one side, the other side either ignores it safely or is updated in the same PR.
- Error handling at this layer should unwrap and re-wrap downstream errors into the user-facing response envelope. Do not leak raw downstream stack traces to the browser.

Everything in Section 1 (layer discipline, transactions if the layer touches its own DB, MyBatis/JPA conventions, secret handling) applies.

---

## 3. Vue 3 + Vite Frontend (reference stack)

Applies to any Vue 3 + Vite SPA. React/Svelte/Angular equivalents follow the same layer rules — only the idioms differ.

### Layer discipline

```
Page (.vue)         → layout, route-level state
  ↓
Composable (useX)   → reactive business logic, API orchestration
  ↓
API module (axios)  → typed request/response, path and query construction
  ↓
Backend
```

- Keep HTTP calls in the API module, not inline in components. Grep for the existing API module (`src/api/**`, `src/services/**`) and add a sibling method.
- Composables return `{ state, actions }`; components bind them. Don't put business logic in components.
- Pinia stores: one store per domain, not per component. Grep existing stores before creating a new one.

### Vite proxy (common gotcha)

- Frontend calls are proxied per `vite.config.js`. Before adding a new endpoint, open the target service's `vite.config.js` and confirm the path prefix is routed.
- If a new prefix is needed, the change belongs in `vite.config.js` **and** in the deployed nginx/Helm/Ingress config — which is a deploy-time change, not a build-time one. Flag this in the report if missed.

### Type safety

- TypeScript-first projects: types go in `src/types/` or colocated `*.types.ts`. Do **not** use `any` to silence the compiler.
- JavaScript projects: use JSDoc `@typedef` on shared shapes. Lint must pass.

### Tests (TDD Phase 2 for frontend)

- Component tests: Vitest + `@vue/test-utils`. Test the component's contract (props → rendered DOM, user interaction → emitted event), not implementation details.
- Composable tests: pure unit tests; mock axios at the API module boundary, not inside the composable.
- Do not test Pinia stores through components; test the store directly.

### Running (per service)

```bash
npm run dev         # dev server
npm run lint        # eslint
npm run type-check  # tsc --noEmit or vue-tsc
npm run test        # vitest
npm run build       # production build smoke-check
```

Type-check and lint MUST both be clean before Phase 4.

---

## 4. Cross-Cutting Guardrails (applies to every service)

- **No new `@SuppressWarnings`, `@ts-ignore`, `any`, empty `catch`, or swallowed errors.** If you reach for these to make a test pass, the design is wrong — fix it or raise a blocker.
- **No `System.out.println`, `console.log`, `println`, `printStackTrace`.** Use the project's logger (SLF4J in Java, the existing logger util in the frontend if present — grep for it).
- **Structured logging.** Java: `log.info("event_name", kv("userId", id), kv("reason", reason));` (match whatever the service already does). Frontend: the existing logger, not raw `console.log`.
- **No hardcoded URLs, ports, tenant IDs.** Use `application-<profile>.yml` for backend, `import.meta.env` or the proxy table for frontend.
- **Ports belong in config, not source.** Never hardcode a port in a new config file without a compelling reason — match whatever the existing services use.
- **`@Transactional(readOnly = true)`** on reads — say it once more, because it's the most common regression.

---

## 5. Bug Fix Specifics (when §2 kind = Bug)

- Write the regression test at the level the bug manifests (unit if pure, integration if it needs the DB, controller slice if it's the request path).
- Red-Green-Refactor is non-negotiable:
  1. Write the test; run it; **see it fail in the way the bug report describes.**
  2. Apply the fix from §7; re-run; see it pass.
  3. Revert only the fix; re-run; confirm the test fails again with the bug's symptom.
  4. Re-apply the fix; final green.
- If the test does not fail for the right reason in step 1, the test is wrong. Fix the test before touching production code.
- Fix at the **root cause**, not the symptom (§5A of the report). If the root cause is untouchable (vendor lib, upstream service), a symptom-level fix is acceptable only if §5A explicitly justified it.

---

## 6. Feature / Refactor Specifics (when §2 kind = Feature or Refactor)

- Write the test that pins the new contract from §9 first (input → expected output), then implement just enough to pass.
- If the feature spans multiple layers (controller, service, mapper), write the test at the outermost layer the report said to test — usually a controller slice or a service-level integration test. Do not write a test for every line.
- For refactors with no behavior change, the test suite **must already cover the behavior**. If it doesn't, pause and add the missing tests before refactoring — otherwise you can't prove the refactor preserved behavior.

---

## 7. When You Hit a Blocker Mid-Implementation

| Situation | Action |
|---|---|
| Test you wrote won't fail correctly | Design smell. Revisit the test design. Ask if stuck 10 min. |
| Report §7 is missing a file you now realize you need | Stop. Update the plan with the user. Do not silently add it. |
| Cross-service impact you didn't see before | Stop. Re-run the blast-radius check. Update §8 or ask for approval. |
| Existing test breaks from your change | That's a test catching the regression. Understand why before "fixing" it. |
| CI linter/typechecker fails on touched code | Fix the root cause (proper types, proper imports). Do not `@ts-ignore` / `@SuppressWarnings`. |
| You find dead code nearby | Note it in `work.md §Follow-ups`. Do not delete in this PR unless §7 asked for it. |

Each of these is a signal to **stop and write in work.md**, not a signal to power through.
