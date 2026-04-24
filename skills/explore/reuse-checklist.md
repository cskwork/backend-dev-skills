# Reuse & Low-Coupling Checklist

Run this **before** proposing any new code in the exploration report. Every item on this list maps to a rationalization that leads to duplicate logic, hidden coupling, or broken scope. The goal is to prove reuse is impossible before adding new code.

## The Core Question

> "Does something already exist that does this, or 80% of this?"

If you haven't grepped for it, the answer is unknown — not "no".

## 1. Before Adding a Utility / Helper

For each proposed helper, record:

- [ ] **Verb search** — grep the primary verb of the helper (`format`, `normalize`, `validate`, `parse`, `convert`). Note hits.
- [ ] **Noun search** — grep the domain noun (`Jwt`, `Session`, `Chapter`, `Worksheet`). Note hits.
- [ ] **Synonym pass** — at least 2 synonyms (`build`↔`create`, `check`↔`validate`, `toX`↔`fromX`).
- [ ] **Cross-service pass** — search utility modules in every service, not only the current one (`**/util/**`, `**/common/**`, `**/support/**`).
- [ ] **Decision:**
  - Reuse: cite `file:line`, describe adaptation if any
  - Extend: cite `file:line`, state what parameter/overload you'll add, confirm no caller breakage
  - New: state what the search found and why none fit (one sentence)

**Rule:** never skip the search just because the helper "feels unique".

## 2. Before Adding a DTO / Request / Response Type

- [ ] **Same-module candidates** — list DTOs already in the target package. Do any match the semantics?
- [ ] **Shared DTO modules** — some codebases have a shared `*-common` or `dto/` module. Grep there too.
- [ ] **Extend vs new** — if extending, will the added field be backwards-compatible on the wire (JSON nullable, default value)?
- [ ] **Semantics check** — a DTO named `UserDto` that already exists is NOT a match if the new usage has different invariants (different required fields, different meaning of "user"). Reusing a type with wrong semantics is worse than adding a new one.
- [ ] **Decision:** reuse / extend / new, each with `file:line`.

## 3. Before Adding a Service / Repository Method

- [ ] **Existing methods on the same class** — list them. Does one already do this or nearly so?
- [ ] **Existing queries in the mapper/repository** — any existing query that returns a superset you can filter?
- [ ] **"Same data, different shape" smell** — if the data is already being fetched for another flow, prefer calling the existing service and shaping the result at the caller.
- [ ] **Transaction scope** — will this method be called from inside an existing `@Transactional`? Match the read-only vs read-write expectation.
- [ ] **Decision:** reuse / new, each with `file:line`.

## 4. Before Adding a Cross-Service Call

- [ ] **Existing Feign/HTTP client** — grep for clients targeting the same downstream service.
- [ ] **Existing event channel** — is there a Kafka topic / SSE channel already carrying relevant data?
- [ ] **Protocol fit** — sync (Feign) vs async (Kafka): does the new call actually need a response, or can it be an event?
- [ ] **Failure mode** — timeouts, retries, circuit breakers already configured on the existing client?
- [ ] **Decision:** reuse existing client with new method / add method to existing client / add new client (last resort, justify).

## 5. Before Modifying a Shared Utility

Highest-risk change category. Default is **don't**.

- [ ] **Caller census** — grep every call site. List each `file:line`.
- [ ] **Safety check** — is the change safe for every listed caller? If even one caller needs the old behavior, you may not modify in place.
- [ ] **Safer alternatives:**
  - Add a new overload / sibling function, leave the original untouched
  - Parameterize the behavior (new optional argument with old default)
  - Extract the new variant into a new utility and call the new one only from the new feature
- [ ] **Decision:** modify (only if safe for every caller) / add sibling / parameterize.

## 6. Surgical-Edit Rule

Each line you plan to change must trace to the request. Before finalizing Proposed Changes in the report:

- [ ] Are there any "while I'm here" edits? Remove them.
- [ ] Any reformatting / rename that isn't required by the change? Remove it.
- [ ] Any refactor that goes beyond the scope of this request? Capture as a separate follow-up, do not bundle.
- [ ] Every deletion is justified by an explicit blast-radius check.

## 7. High-Cohesion / Low-Coupling Smells

Flag any of these in the report under Open Questions:

- New code reaches across module boundaries that existing code does not cross
- Proposed service method takes 5+ unrelated parameters
- New DTO bundles fields from multiple unrelated concerns
- The change creates a new two-way dependency between modules that previously had a one-way dependency
- The same logic appears in two places after the change

If any of these show up, redesign before proposing. Do not ship the smell.

## Quick Grep Cheatsheet

Fill in terms from the request; adapt patterns to the repo's language/layout.

```bash
# Verbs
grep -r "fun <verb>\|def <verb>\|public .* <verb>(" <repo>

# Nouns / domain types
grep -r "class <Noun>\|interface <Noun>\|record <Noun>" <repo>

# Existing utilities by folder convention
grep -r "<term>" <repo>/**/util/ <repo>/**/common/ <repo>/**/support/

# Cross-service clients (adapt to framework)
grep -r "@FeignClient\|RestTemplate\|WebClient\|HttpClient" <repo>

# Event producers/consumers
grep -r "@KafkaListener\|KafkaTemplate\|@EventListener\|publish(" <repo>
```

## Output for the Report

The outcome of running this checklist feeds two report sections:

- **Reuse Inventory** — every reuse-or-extend decision with `file:line`
- **Proposed Changes** — each "new" item must state that the checklist was run and what was searched

If the checklist was not run, the report is not ready.
