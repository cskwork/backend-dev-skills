# Database-First Investigation

Companion to `SKILL.md` Phase 1.5. Backend features and bugs live in data. Ground the investigation in the schema before proposing code.

## Core Principle

> The schema is the source of truth. ORM entities, DTOs, and mappers can drift; migrations and `DESCRIBE` don't.

Read the schema first. Then the mapping. Then the code.

## 1. Detect Available DB Access

Run these checks in order and pick the first one that succeeds.

### 1a. Live read-only MCP tool

Scan the tool list for any of: `mysql`, `postgres`, `mariadb`, `sql`, `db`, `database`. If present and documented as read-only, prefer it — it guarantees the schema matches production-like reality, not stale repo files.

Typical safe operations via such a tool:
- `SHOW TABLES LIKE '<pattern>'`
- `DESCRIBE <table>` / `SHOW CREATE TABLE <table>`
- `SHOW INDEX FROM <table>`
- `SELECT COUNT(*) FROM <table>` (for cardinality only — guard with `LIMIT` on row queries)
- `SELECT <cols> FROM <table> WHERE <pk> = <id> LIMIT 5` (shape sampling only)
- `EXPLAIN <query>`

### 1b. Local read-only CLI

Look for a `mysql`/`psql` binary on `$PATH` AND a known read-only connection string in one of:
- `*/src/main/resources/application-dev.yml` (or `-local.yml`)
- `database-dump/`, `scripts/`, `docs/`
- A repo-local `.env.dev`

Confirm the user it's a dev/stg host before connecting. If in doubt, ask.

### 1c. Static fallback

No live access. Use, in this priority:
1. Migration files (`db/migration/**`, `src/main/resources/db/**`, Flyway/Liquibase directories) — authoritative for the intended schema.
2. MyBatis mapper XML (`**/mapper/**/*.xml`) — shows the actual SQL being executed, including joins and dynamic conditions.
3. JPA `@Entity` classes — reveal the ORM's view of the schema; may lag reality.
4. DDL dumps / schema docs (`database-dump/`, `docs/db/**`).

Flag that you used a static source in the report — it may be stale.

## 2. Schema Triage Checklist

For every table the request touches, record:

- [ ] **Name & source** — `mydb.user_login` — DDL at `migration/V23__add_user_login.sql:1-28`
- [ ] **Business meaning** — one sentence. ("Tracks per-user login events for audit.")
- [ ] **Primary key** — composite? auto-increment? natural?
- [ ] **Columns you will read/write** — name, type, nullability, default, whether indexed
- [ ] **Foreign keys in/out** — which other tables this joins to
- [ ] **Indexes relevant to the change** — composite order, prefix length, uniqueness
- [ ] **Row count magnitude** (if migration or large read involved) — `live: SELECT COUNT(*) …` or from the user/docs
- [ ] **Soft-delete / audit columns** (e.g. `deleted_at`, `created_at`, `modified_by`) — existing queries usually filter on these; new queries must too
- [ ] **Multi-tenancy / partition key** (e.g. `org_id`, `school_id`) — most queries in the codebase will already filter by it; new queries must match

## 3. Map Schema → Code

Once the tables are known, produce a table → code map with citations:

| Table | Mapper / Repository (`file:line`) | Entity / DTO (`file:line`) | Service methods (`file:line`) |
|-------|-----------------------------------|----------------------------|--------------------------------|
| `user_login` | `UserLoginMapper.xml:42-68` | `UserLoginVO.java:1-40` | `LoginService.findLastLogin:112` |

This map is what makes the exploration cheap for everyone who reads the report.

## 4. Query Patterns Already in Use

Before proposing a new query, grep the existing ones for the same table:

```bash
grep -rE "FROM\s+<table>|JOIN\s+<table>|INTO\s+<table>" <repo>
```

Check:
- Do existing queries already return what you need, possibly as a superset?
- Are there established join patterns you should match (same aliases, same order, same filter columns)?
- Do the existing queries use hints (`/*+ INDEX(...) */`), pagination, or known-slow full scans you should avoid?

**Rule:** reuse a mapper method before writing a new one. A new method requires justification in the report.

## 5. Migration & Rollback Considerations

If the change touches schema (`ADD COLUMN`, new index, new table, type change):

- [ ] Framework used — Flyway? Liquibase? Raw SQL in CI?
- [ ] File naming convention (`V<version>__<desc>.sql`) and next free version
- [ ] Online vs offline migration — large tables need `ALGORITHM=INPLACE, LOCK=NONE` or pt-online-schema-change
- [ ] Backfill plan for new NOT NULL columns on populated tables
- [ ] Rollback: separate down migration vs application-level feature flag
- [ ] Cross-service readers of the table — do they tolerate the change during rollout?

Call these out under **Proposed Changes** and **Rollback** in the report.

## 6. Query the DB Safely (when available)

Allowed statements for exploration:

```sql
SHOW TABLES LIKE 'pattern%';
SHOW CREATE TABLE table_name;
DESCRIBE table_name;
SHOW INDEX FROM table_name;
SELECT COUNT(*) FROM table_name;                  -- cardinality
SELECT * FROM table_name WHERE pk = ? LIMIT 5;    -- shape sampling
EXPLAIN SELECT ...;                               -- index usage check
```

Forbidden during exploration (require separate user approval even then):

```sql
INSERT, UPDATE, DELETE, REPLACE, TRUNCATE
CREATE, ALTER, DROP, RENAME
GRANT, REVOKE
```

If the available MCP/CLI does not enforce read-only, self-enforce. When in doubt, do not run the query.

**PII:** do not paste row values into the report. Describe shapes, nullability, and types.

## 7. Output for the Report

Phase 1.5 fills the **Domain Model & Data** section of the report. Minimum content:

```markdown
### Domain Model & Data

**Tables involved:**
- `user_login` — tracks login events. DDL `migration/V23__add_user_login.sql:1-28`. Live DESCRIBE run: yes (dev-replica).
- `user` — user master. JPA entity `User.java:1-92`, DDL in `docs/db/schema.sql:134-160`.

**Columns in scope:**
- `user_login.login_at` TIMESTAMP NOT NULL — read by <service:line>
- `user_login.result_code` VARCHAR(16) NOT NULL — written by <service:line>
- `user.last_login_at` TIMESTAMP NULL — proposed to be updated (see Proposed Changes #3)

**Relationships:**
- `user_login.user_id` → `user.id` (FK defined at `V23…:18`)

**Cardinality:**
- `user_login`: ~12M rows (dev replica). `user`: ~1.2M rows. Affects indexing strategy in Proposed Changes #4.

**Existing queries on these tables:**
- `UserLoginMapper.xml:42-68` — selectLastLoginByUserId (reusable for this feature)
- `UserMapper.xml:210-232` — updateLastLoginAt (reusable; no new query needed)

**Source of schema claims:** live DESCRIBE on dev-replica + DDL files cited above.
```

## 8. When to Escalate to the User

Ask before proceeding if:
- The only DB access you can find appears to point at prod/audit — do not connect
- Required DDL is not in the repo and no live access is available
- You find a schema-code mismatch (entity says one thing, DB says another) — this is a risk that must be resolved before any change

## Bottom Line

No Phase 2 / Phase 3 / Proposed Changes until the Domain Model & Data section is filled with cited, verifiable schema facts. That section is how the report proves it is grounded in the system's actual state, not an assumption tree.
