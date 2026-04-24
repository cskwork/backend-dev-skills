# Payload Sourcing

Companion to `SKILL.md` Phase 1. Verification payloads must reflect **what the service will actually see in production**, not what feels reasonable to type. This document defines the priority order, the safety rules, and the redaction contract.

## Core Principle

> Invented payloads verify invented behavior. Real shapes verify real behavior.

The same discipline `database-first.md` applies to schema — cite the source, prefer live over static — applies here to payload bodies.

## Priority Order

Apply in order. Stop at the first source that produces a usable payload; move to the next for variants it cannot produce.

### 1. User-provided payloads (highest)

If the user pasted a payload (in the current conversation, in a Jira ticket, in a Slack excerpt included in the request) — use it verbatim as the **happy** variant and as the **regression** variant for bug fixes.

Save unchanged to `fixtures/<endpoint>.user-<label>.json`, except:
- Replace real auth tokens with `__REDACTED_TOKEN__`.
- Replace real email / phone / national-ID with the placeholders in §Redaction below.
- Keep nulls, empty strings, trailing whitespace, and weird casing exactly as provided — those are the bug signal.

Record the source in `verify.md §4`:
```
fixtures/<endpoint>.user-bug-report.json   origin=user  supplied=<when>  redacted=<fields>
```

### 2. Saved fixtures (from prior runs on this ticket)

If `.backend/<YYYYMM>/<slug>/fixtures/*.json` already exists from a previous `/verify` iteration on the same ticket, prefer reusing.

**Reuse validity check** — reject the saved fixture if any of these changed since it was written:

- The endpoint's request DTO added/removed fields (diff the DTO `file:line` cited in `explore.md §5B.contract`).
- The endpoint's validation annotations changed (`@NotNull`, `@Size`, enum narrowed).
- The ticket's `explore.md` has been replaced by an `explore-v2.md`.

If the check fails, regenerate from source 3 or 4. Do not mutate the old fixture in place — write a new file with a suffix (`.v2.json`) so the history is preserved for the audit trail.

### 3. Live read-only MySQL sampling

Use when sources 1-2 cannot produce a realistic shape. This is the default for the `happy` and `boundary` variants of any endpoint that reads or writes rows.

**Access detection** — same detection order as `database-first.md` §1:

1. MCP tool whose name contains `mysql` / `postgres` / `mariadb` / `sql` / `db`. Confirm it is documented as read-only.
2. Local `mysql` binary + a dev/stg-safe connection string from `application-dev.yml` or `database-dump/`.
3. If neither, stop and ask the user for a payload.

**Safety rules (absolute):**

- Read-only statements only: `SELECT`, `SHOW`, `DESCRIBE`, `EXPLAIN`. Never `INSERT`/`UPDATE`/`DELETE`/`DDL`.
- Non-prod targets only. If the connection string's host resembles `*-prod-*` / `*-audit-*` / a release hostname — **do not connect**. Ask the user for confirmation.
- Every sample query is `LIMIT`-bounded. Default `LIMIT 5`, never exceed `LIMIT 50`.
- Shape sampling, not data extraction. Copy column types, null-ness, value ranges — not the raw values for PII columns.

**Sampling recipe:**

```sql
-- Shape: columns + nullability of a target table
DESCRIBE <table>;

-- Representative rows (use ORDER BY primary key DESC for recent shapes)
SELECT <columns>
FROM <table>
WHERE <partition-or-tenant-filter>
ORDER BY <pk> DESC
LIMIT 5;
```

Pick rows that differ — for example:
- One with every nullable column populated.
- One with a mix of populated and null.
- One with minimum-length / maximum-length values (if `LENGTH(col)` can be sampled quickly).

**From sampled rows, build a payload** by mapping columns → request fields using the mapper XML or JPA entity cited in `explore.md §4 Domain & Data`. Do not invent the mapping — cite it.

Save to `fixtures/<endpoint>.<variant>.json`. Record origin in `verify.md §4`:
```
fixtures/<endpoint>.happy.json   origin=db  query="SELECT ... LIMIT 5"  row_idx=2  redacted=[email,phone]
```

### 4. Synthesized payloads (lowest)

Only for variants where no real row will ever look this way. Examples:

- **Negative variants** — missing required field, wrong enum value, over-max-length string.
- **Boundary variants the DB cannot produce** — empty array where NOT NULL schema blocks empty, a field set to exactly the validation maximum.
- **New endpoints with no existing data** (green-field feature, table has 0 rows).

Every synthesized field is justified in `verify.md §4` with one line:

```
fixtures/<endpoint>.negative.json   origin=synth  field=<name>  why=<"exceeds @Size(max=50)" | "enum 'X' not in allowed set" | ...>
```

**Synthesis is not invention.** The shape still matches `explore.md §5B.contract.req`. Only the values are manufactured.

## Redaction Contract

Apply to every file written under `.backend/<YYYYMM>/<slug>/fixtures/` and every line appended to `runs/run-<N>.log`.

| Category | Detection | Replacement |
|---|---|---|
| Email | `[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}` | `user+<rowIdx>@example.test` |
| Phone | `01[016-9]-?\d{3,4}-?\d{4}` | `010-0000-<seq>` (seq = 0001, 0002, …) |
| National-ID (e.g. SSN, RRN, NIN) | pattern-specific | `__REDACTED_ID__` |
| Passport / alien-registration / other national ID | pattern-specific | `__REDACTED_ID__` |
| Personal name | mapped via `explore.md §4 cols`  | `user<rowIdx>` |
| Organization name | mapped via `explore.md §4 cols` | `org<rowIdx>` |
| Authorization header value | exact match `Bearer <token>` | `Bearer __REDACTED__` |
| Session cookie | cookie name from `application-dev.yml` | `__REDACTED__` |
| API keys / secrets | Jasypt `ENC(...)`, `sk_*`, `AKIA*` | `__REDACTED_SECRET__` |
| Unknown sensitive-looking field | anything in the mapping you cannot classify | `__UNKNOWN__` + note in `§4` |

**Redaction is a pre-write transform.** Redact the payload *before* calling Write. Never write a PII-containing fixture with "will fix later" intent.

**Row index mapping** — if the fixture references a real `user.id = 12345`, keep the id only if the `id` column is a non-sensitive surrogate key. Resource ids are usually fine; national identifiers are not. If in doubt, redact and note.

## Variant Coverage Matrix

Minimum per endpoint target:

| Kind | Variants required |
|---|---|
| Read (`GET`) | happy, boundary (empty result / filter hits single row / large-page), negative (non-existent id / forbidden id) |
| Create (`POST`) | happy, boundary (min/max field lengths), negative (missing required, dup unique) |
| Update (`PUT` / `PATCH`) | happy, boundary (no-op update, full-replacement), negative (stale version / forbidden) |
| Delete (`DELETE`) | happy (soft-delete path), negative (non-existent / already deleted) — **never** run destructive DELETE against real dev data without explicit user approval; default to a dry-run mode or a seeded throwaway row |
| Bug fix | everything above PLUS a `regression` variant matching `explore.md §5A.repro` verbatim |

If the endpoint behavior depends on tenant / role / partition, add one variant per distinct class actually used in the codebase (e.g. `happy.teacher`, `happy.student`, `negative.wrong-tenant`).

## Cross-Service Payload Considerations

Endpoints that fan out to other services (Feign clients called inside the handler) need payloads that exercise the downstream path:

- If the handler calls a downstream service for metadata, include an id in the payload that **actually exists** in the downstream's local DB sample. A happy-path payload with a nonexistent downstream id tests a 404 path, not the real behavior.
- Record in `verify.md §6 Cross-Service Checks` which downstream ids were used and how they were sourced.

## DB-Write Safety

`/verify` will generate payloads for write endpoints. **Running** those payloads may mutate local dev data. Rules:

- Default: run writes only against the local dev DB. Confirm the target service's `spring.datasource.url` is pointing at `localhost` / `host.docker.internal` / a dev container.
- Before running a `DELETE` or any irreversible operation, ask the user once per session even if they previously approved.
- Prefer **cleanup-in-the-harness**: if a write creates a row, the same harness includes a teardown curl (also asserted) to remove or revert it. Record teardowns in `verify.md §5` as their own rows.
- If the user explicitly requests "leave the side effect for inspection", note it in `§9 [INFO]` so the next session does not mistake leftover rows for pollution.

## Anti-Patterns

- **Payload drift.** Copy-pasting last week's payload for a new endpoint without re-mapping to this endpoint's request DTO. Re-derive every time; cite the DTO file:line.
- **Faking locale.** Using `"홍길동"` as a placeholder Korean name while the real column contains `"김*훈"`-style masked audit values. Mirror the real shape including already-masked forms.
- **Invented enums.** The DTO says `enum {A,B,C}` but live data uses `A`, `A_LEGACY`, and `UNKNOWN`. Sample the DB for actual values; don't trust the DTO alone.
- **Committing PII** because "the folder is `.backend/` so it's local". `.backend/` files can be pushed, emailed, screenshared. Redaction is unconditional.
- **`SELECT *`** into a fixture file. Read only the columns the endpoint consumes. `SELECT *` leaks PII fields you had no intent to use.

## When to Escalate to the User

Ask before proceeding if:

- The only DB access available points at stg/audit/prod.
- A mandatory variant (e.g. `happy` for a read endpoint) cannot be produced from any source — the DB has zero rows and the user has not provided a payload.
- A field in the sampled row is sensitive and does not match any row in the redaction contract (e.g. a custom biometric field). Ask how to treat it before writing the fixture.
- A write-variant's teardown path is unclear (e.g. the delete endpoint itself is under test, so you can't use it to clean up).

## Bottom Line

Payloads are evidence. Saved fixtures are the audit trail. Neither is complete without a citation back to the source — user message, prior run, DB sample, or a justified synthesis. Commit redacted, cite everything, never widen the access you have.
