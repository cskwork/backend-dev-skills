# Implementation Playbook (Phase 2 Stack-Specific Guardrails)

This is the quick-reference for `/work`'s TDD implementation phase, tuned to common backend/fullstack stacks. Read the section matching the service family you are editing. Deviations require explicit justification in `work.md`.

## Symbol Legend (same as explore.md / work.md)

```
→ calls / depends-on     ← called-by     ⇢ async / event
⊕ add   ⊖ remove   ✎ edit   ≡ reuse   ≈ extend
↑ read (DB)   ↓ write (DB)   ⟳ retry / loop
```

---

## 1. Spring Boot + MyBatis Backend (`example-*-api`)

Affected services: `example-api`, `content-api`, `admin-api`, `example-viewer-api`.

### Layer discipline

```
Controller  → parses HTTP, validates shape, returns response envelope
   ↓
Service     → owns business logic, transaction boundaries, Feign calls
   ↓
Mapper/Repo → owns SQL, no conditionals beyond dynamic SQL
   ↓
MySQL / Redis / Kafka
```

- Do not push business logic into controllers "because it's shorter". Controllers are thin.
- Do not write SQL in services. New query → new mapper method + mapper XML entry.
- DTOs never carry JPA entities across the controller boundary. Convert at the service layer.

### Transaction annotations (critical for Spring/JPA/routing setups)

- **Read paths** (GET endpoints, query-only service methods): `@Transactional(readOnly = true)` — this enables read-replica routing when the stack supports it. Missing `readOnly = true` on a read path means hitting the primary DB unnecessarily.
- **Write paths** (POST/PUT/PATCH/DELETE): `@Transactional` without `readOnly` — goes to Master.
- Never leave the annotation off a service-layer public method that touches the DB. Not annotating breaks the transaction boundary contract.
- If a read path must also write (e.g., audit log), split: keep the read in `@Transactional(readOnly = true)` and publish an event for the write side.

### MyBatis conventions

- New query: add a method signature to the Mapper interface, add a `<select>` / `<insert>` / `<update>` / `<delete>` block with the **same id** to the XML, add parameter/result types. No inline SQL in services.
- Reuse existing `resultMap` definitions when the row shape matches. Do not define a parallel resultMap for the same table columns.
- Bind parameters with `#{param}`, never `${param}`, except for whitelisted dynamic identifiers (column names from a fixed set). Anything else is SQL injection.
- `LIMIT`/`OFFSET` pagination: match the existing pagination helper pattern — grep for `pageNum`, `pageSize`, `PagingDto` in the target service.

### Feign clients (service → downstream)

- Reuse an existing `@FeignClient` if it targets the same downstream service. Add a new method to that client; do not create a second client to the same service.
- Method signatures on Feign clients are a cross-service contract. Adding a required field or changing a path requires matching changes in the downstream service within the same PR (or a backwards-compatible overload).
- Timeouts, retries, and fallbacks are configured at the client level. Do not reimplement retry logic inside a new method.

### Kafka

- Events flow producer service ⇢ consumers. Reuse an existing topic if the payload semantics match.
- Payload DTOs for Kafka are a wire contract. Adding a field = add as nullable with a default; removing or renaming = versioned topic or versioned payload.
- `@KafkaListener` methods must be idempotent. If the plan did not call this out in §5B, flag it in Open Questions.

### Jasypt properties

- Any secret added to `application*.yml` must be encrypted with the project's Jasypt master key. Do not commit plaintext.
- Search the service's existing `application-dev.yml` / `application-prod.yml` for the `ENC(...)` pattern; follow it.

### Test scaffolding (TDD Phase 2)

- Unit tests for services: use Mockito for mapper/Feign dependencies. Do **not** spin up Spring context for a pure service unit.
- Integration tests for mappers: use the MyBatis test slice (`@MybatisTest` or the project's existing test base class — grep for one). Real DB (H2 or Testcontainers MySQL — follow whatever the service already uses).
- Do not add @SpringBootTest where a slice test suffices; it slows the suite and masks coupling issues.
- Test file naming: mirror source — `SomeService` → `SomeServiceTest`, same package.

### Running tests (per service)

```bash
# Full test suite
cd <service>-api && ./gradlew test

# Focused
./gradlew test --tests 'com.example.<pkg>.<Class>'

# Build without tests (smoke compile check)
./gradlew clean build -x test
```

---

## 2. WAS Middleware (`admin-was`, `content-was`, `demo-was`)

Affected services: `admin-was`, `content-was`, `demo-was`.

These are proxy layers in front of the corresponding `-api` service, with their own business logic added on top.

- Do **not** duplicate a business rule that already exists in the `-api`. If the rule is needed here too, call the `-api` via Feign or move the rule to a shared module, per the plan.
- Request/response shapes between WAS and API are an internal contract — keep them in sync. If a field is added on one side, the other side either ignores it safely or is updated in the same PR.
- Error handling at the WAS layer should unwrap and re-wrap `-api` errors into the user-facing response envelope. Do not leak raw `-api` stack traces to the browser.

Everything in Section 1 (layer discipline, transactions if the WAS touches its own DB, MyBatis conventions, Jasypt) applies.

---

## 3. Vue 3 + Vite Frontend (`example-*-web`, `chat-widget`)

Affected services: `frontend-web`, `admin-web`, `content-web`, `viewer-web`, `chat-widget`.

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

### Vite proxy (common proxy gotcha)

- Frontend calls are proxied per `vite.config.js`. Before adding a new endpoint, open the target service's `vite.config.js` and confirm the path prefix is routed.
- If a new prefix is needed, the change belongs in `vite.config.js` **and** in the deployed nginx/Helm config — which is a deploy-time change, not a build-time one. Flag this in the report if missed.

### Type safety

- TypeScript-first services (`frontend-web`, `viewer-web`): types go in `src/types/` or colocated `*.types.ts`. Do **not** use `any` to silence the compiler.
- JavaScript services (`admin-web`): use JSDoc `@typedef` on shared shapes. Lint must pass.

### Tests (TDD Phase 2 for frontend)

- Component tests: Vitest + `@vue/test-utils`. Test the component's contract (props → rendered DOM, user interaction → emitted event), not implementation details.
- Composable tests: pure unit tests; mock axios at the API module boundary, not inside the composable.
- Do not test Pinia stores through components; test the store directly.

### Running (per service)

```bash
cd <service>-web
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
- **Ports come from project config.** Derive them from profile files, compose files, service docs, or repo instructions. Never hardcode a port in a new config file without a compelling reason.
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
