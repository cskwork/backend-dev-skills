# Reuse & Discipline Re-check (Phase 1)

Run this **after** loading the explore report and **before** writing any code. Its job is to re-validate the reuse decisions captured in `§6 Reuse Inventory` and `§7 Proposed Changes` of `explore.md` against the **current** state of the codebase. The codebase moves between exploration and implementation — your plan must not ship stale assumptions.

## The Core Question

> "Is every `new` item in §7 still genuinely new today, and is every `extend` item still safe?"

If you haven't re-grepped since the explore report was written, the answer is unknown — not "yes".

## The Re-check Procedure

For each row in §7 Proposed Changes, classify it:

- **reuse** (≡) — no action here; reuse is a pure read.
- **extend** (≈) — run Section A below.
- **modify-shared** — run Section B below.
- **new** (⊕) — run Section C below.

Record the outcome of each check in your work notes; it will go into `work.md §Reuse Re-check`.

---

## Section A — `extend` Items

The plan adds an overload, parameter, or field to an existing symbol.

- [ ] **Caller census** — grep every call site of the symbol. List them. If the list is longer than §8 described, **stop** and update the report.
- [ ] **Safety check** — is the existing behavior preserved for every current caller? (New optional param with old default, new overload, new nullable field.) If any caller depends on the old invariant, switch to **add sibling** instead.
- [ ] **Transaction scope** — if extending a service method, does the new path have compatible `@Transactional` semantics (readOnly vs read-write)?
- [ ] **Wire compatibility** — if extending a DTO/request/response type, is the field backwards-compatible (JSON nullable, default value, absent field tolerated)?

If any box is unchecked after honest examination: **stop, note the gap, update the plan or ask the user.** Do not paper over with "probably fine".

## Section B — `modify shared utility` Items

Highest-risk category. Default is **don't**.

- [ ] **Re-grep callers** — do not trust §8's caller list; re-run the grep now. The set can change.
- [ ] **Behavior diff** — for every caller, does the new behavior still satisfy that caller's expectations? If even one does not, do not modify in place.
- [ ] **Safer alternatives considered:**
  - Add a new overload / sibling function, leave the original untouched
  - Parameterize the behavior (new optional argument, old default preserves prior behavior)
  - Extract the new variant into a new utility and route only the new feature through it
- [ ] **Decision:** modify-in-place (only if provably safe for every caller), add-sibling, or parameterize.

Record the chosen alternative and the file:line of each existing caller in your notes.

## Section C — `new` Items (utility / DTO / service method / client / event)

For each proposed new symbol, re-run the core search. The explore report's search was done at time T; you are at T+Δ.

- [ ] **Verb re-search** — the primary verb of the helper (`format`, `normalize`, `validate`, `parse`, `convert`, `build`, `create`). Note current hits across **all services in the repo**, not just the target one.
- [ ] **Noun re-search** — the domain noun (`Jwt`, `Session`, `Order`, `Invoice`, `Customer`, `User`, etc.).
- [ ] **Synonym pass** — at least 2 synonyms (`build`↔`create`, `check`↔`validate`, `toX`↔`fromX`, `get`↔`fetch`, `load`↔`find`).
- [ ] **Cross-service utility folders** — grep `**/util/**`, `**/common/**`, `**/support/**`, `**/helper/**`, `**/service/**` across *every* service module, not just the target service.
- [ ] **Semantic match, not just name match** — a `UserDto` that exists is NOT a match if the new usage has different required fields or different meaning. Reusing a type with wrong semantics is worse than adding a new one.
- [ ] **Decision:**
  - If a match was found that the explore report missed → **stop.** Amend the plan to `reuse` or `extend` this match; inform the user the plan shifted.
  - If no match was found → proceed with `new`, and record the exact search terms you ran so `work.md` can prove the search was performed.

## Cross-Service Reuse Checks

Legacy codebases have strong cross-service reuse patterns. Before assuming "new", verify each of these:

- [ ] **HTTP clients to downstream services.** If the change involves a downstream service, grep `@FeignClient` / `WebClient` / `RestTemplate` / similar annotations under `**/client/**` and sibling folders — a client may already exist with the method you need or close to it.
- [ ] **Message queue producers/consumers.** If the change emits or reads an event, grep `@KafkaListener`, `KafkaTemplate`, `@RabbitListener`, or the equivalent for your queue, plus topic/queue name constants. A channel often already exists.
- [ ] **Cache / Redis keys.** If the change reads/writes a cache (session, data cache, pub/sub), grep for the key prefix. Two services accidentally using the same prefix for different payloads is a classic legacy-codebase bug.
- [ ] **SSE / WebSocket / STOMP channels.** If the change pushes real-time messages, look for existing realtime services in the repo for existing channels — prefer adding to an existing channel over creating a new one.
- [ ] **SSO / JWT handling.** Never roll your own. Use existing filter/interceptor chains; grep `JwtFilter`, `SsoInterceptor`, `TokenValidator`, etc.
- [ ] **Encrypted properties.** If config must be added, check how siblings in the same service encrypt values (Jasypt `ENC(...)`, Vault references, SOPS, or project-specific scheme). Never commit a plaintext credential.
- [ ] **ORM mappers / repositories.** If a new query is needed, first look at the same-domain mapper/repository file. An existing `SELECT` may already return a superset you can filter in the service layer, avoiding a new query.

Each of these checks takes 30 seconds with `grep` and prevents a 2-hour incident.

## High-Cohesion / Low-Coupling Smells (re-check)

Flag any of these and **stop** to redesign before coding:

- New code reaches across module boundaries that existing code does not cross (e.g., service A directly reading from service B's DB — it should go through service B's API).
- Proposed service method now takes 5+ unrelated parameters after the re-check. Split or wrap in a request object.
- New DTO bundles fields from multiple unrelated concerns. Split per concern.
- The change creates a new two-way dependency between modules that previously had a one-way dependency.
- The same logic would exist in two places after the change. Factor once, or reuse the existing copy.

If any smell is present, do not proceed to Phase 2. Go back to the user with the smell and a proposed redesign.

## Surgical-Edit Precondition

Before writing the first line of code:

- [ ] Every line you are about to change traces to a row in §7.
- [ ] There are zero "while I'm here" items mixed in.
- [ ] There are zero style-only reformats mixed in.
- [ ] Any legitimate follow-up you spotted during exploration is captured as a note for `work.md §Follow-ups`, not a change in this session.

If any box is unchecked, the diff is not ready.

## Output for `work.md`

This re-check feeds a dedicated section in `work.md`:

```
§Reuse Re-check
- §7.row_1  act=⊕  verb=<verb>  noun=<noun>  searched={<terms>}  result=no-match → proceed new
- §7.row_2  act=≈  callers-before=3  callers-now=4  new-caller=<file:line>  safe=yes  decision=add-param
- §7.row_3  act=modify-shared  callers-now=<N>  safe=no → switched to add-sibling Util2.newHelper
```

Every row in §7 must have a corresponding line here. If it doesn't, the re-check wasn't complete.
