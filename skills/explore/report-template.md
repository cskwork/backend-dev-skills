# Exploration Report Template

Two audiences, two styles:

- **Executive Summary** → human, prose, plain language
- **Everything below** → AI re-read, symbolic shorthand, minimal prose

Save to `.backend/<YYYYMM>/<slug>/explore.md` (see SKILL.md Phase 4 for folder layout and naming rules).
The `<YYYYMM>/<slug>/` folder is the shared workspace for this ticket — `/work` later writes `work.md` alongside this file, and `/verify` writes `verify.md` plus `harness/`, `fixtures/`, and `runs/` subfolders.

---

## Symbol Legend (applies to all AI sections)

Keep this legend in the report file. It is cheap and makes the file self-describing.

```
Arrows    →  calls / depends-on          ⇢  async/event
          ←  called-by                    ⇔  bidirectional
Actions   ⊕  add         ⊖  remove        ✎  edit         ≡  reuse   ≈  extend
Flow      ↑  read (DB)   ↓  write (DB)    ⟳  loop / retry  ∑  aggregate
Risk      ○  low         ◐  med           ●  high
Kind      [B] bug        [F] feature      [R] refactor
Schema    PK=primary key FK=foreign key   UX=unique idx    IX=index    NN=NOT NULL
Sources   L:<env>  live query on env      S:<file:line>    static file
Status    ✓ verified     ?  unverified    ⚠ risk/gap
Citation  path/to/file.ext:LINE           ~File.ext:LINE   approximate
Person    P:<role>                        (e.g. P:PM, P:QA)
```

If you introduce a new symbol, add it to the legend in this file. Do not invent ad-hoc shorthand without legending it.

---

## 1. Executive Summary *(human-readable, prose)*

> Share as-is with non-developers. 5–10 lines. No file paths, no code, no symbols.

- **Goal:** <what we were asked to do, one sentence>
- **Finding:** <what we discovered, one to two sentences, plain language>
- **Change:** <what will change from a user's perspective, one sentence>
- **Risk:** <low/medium/high> — <one-sentence reason>
- **Done when:** <the observable condition that confirms completion, one sentence>
- **Open questions needing input:** <list or "none">

---

## 2. Meta

```
kind: [B|F|R]
slug: <kebab-case-slug>
yyyymm: <YYYYMM>
repo: <repo-name>
scope: <service(s) / module(s)>
one-liner: <verb-first ≤15 words>
```

---

## 3. Evidence  (Phase 1 findings)

Format: `<path>:<line>  ·  <≤12-word note>`

```
services/user-api/.../LoginController.java:42    · POST /login entry
services/user-api/.../LoginService.java:118      · session issue + redirect
services/user-api/.../UserMapper.xml:210         · update last_login_at
```

---

## 4. Domain & Data  (Phase 1.5)

Source-of-truth tag per fact: `L:<env>` or `S:<file:line>`.

```
tables:
  user_login    S: migration/V23__...sql:1-28     L: dev-replica ✓
  user          S: User.java:1-92                 L: dev-replica ✓

cols (scope):
  user_login.id           BIGINT   NN  PK   · [↓ write ⊕]
  user_login.user_id      BIGINT   NN  FK→user.id  IX(idx_ul_user)   · [↑ read]
  user_login.login_at     TS       NN  default NOW() · [↓ write]
  user.last_login_at      TS       NULL                · [↓ write ✎]

rels:
  user_login.user_id → user.id                  FK at migration/V23:18

card:
  user_login ≈ 12M rows  (L: SELECT COUNT(*))
  user       ≈ 1.2M rows (docs)

queries existing:
  UserLoginMapper.xml:42-68   selectLastLoginByUserId   ≡ reusable
  UserMapper.xml:210-232      updateLastLoginAt         ≡ reusable

mismatches:  none | ⚠ <entity vs DDL drift — see §10>
```

---

## 5A. Root Cause  *(remove for [F]/[R])*

```
symptom (verbatim):
  "<error / observed>"

repro:
  step1 → step2 → step3      ✓ reproducible on <env>
  (or:  ? not reproduced — need <X>)

trace (symptom → trigger):
  LoginController.java:42  →  LoginService.java:118  →  TokenUtil.java:57
    →  JwtParser.java:34  ←  original trigger

root cause:
  <one-sentence: X is cause because Y>

rejected hypotheses:
  H1 <...>    ✗  evidence: <file:line>
  H2 <...>    ✗  evidence: <file:line>
```

---

## 5B. Pattern & Fit  *(remove for [B])*

```
belongs in:  <service>/<layer>/<pkg>   ← analogous: <file:line>

model after:
  ctrl  <file:line>
  svc   <file:line>
  data  <file:line>
  dto   <file:line>

contract:
  req   { <field>:<type>, ... }        shape ≡ <ExistingDto file:line> | ⊕ new
  resp  { <field>:<type>, ... }        envelope ≡ <file:line>
  db    ↑ <table.col>, ↓ <table.col>
  out   <Feign client file:line> | <Kafka topic: name>

deviation from analogue:
  - <what differs, why, risk ○/◐/●>
```

---

## 6. Reuse Inventory

Default is reuse. "new" requires justification.

```
utils:
  ≡ <path>:<line>   <what/how used>
  ≈ <path>:<line>   extend with <param/overload>

dtos:
  ≡ <path>:<line>
  ⊕ <path>:<line>   new  · searched: [<terms>]  · no fit because <why>

svc/repo:
  ≡ <path>:<line>.<method>
  ⊕ <path>:<line>.<method>  new  · why not reuse: <file:line one-liner>

clients (feign/http/kafka):
  ≡ <path>:<line>   add method to existing client
  ⊕ <path>:<line>   new  · justified: <why>

config/consts:
  ≡ <path>:<line>
```

---

## 7. Proposed Changes

Numbered, dependency-ordered. Every item traces to the request.

```
# file                                 act  · why                  · reuse/new
1 <path>                               ✎    · wire new validator   · ≡ <path>:<line>
2 <path>                               ⊕    · new service method   · ⊕ new (searched: <terms>)
3 <path>                               ✎    · update query         · ≡ <mapper:line>
4 <path>                               ⊖    · dead after #2        · blast: see §8
5 migration/V<next>__<desc>.sql        ⊕    · add col user.x       · rollback: down-mig
```

If list > 10: ⚠ scope too broad — narrow or split.

---

## 8. Blast Radius

```
callers of changed symbols:
  <path>:<line>   · <what breaks if contract shifts>
  <path>:<line>   · <...>

shared state:
  db tables:     <list>
  redis keys:    <pattern>
  kafka topics:  <list>
  feature flags: <list>
  cache ns:      <list>

cross-service:
  feign consumers: <service> via <client file:line>
  sse/ws channels: <name>
  sso/session assumptions: <one-liner>

config/migration:
  new prop: <key>=<default>   online? yes/no
  ddl:      <safe? ALGORITHM=INPLACE? backfill?>
```

---

## 9. Test Plan

```
add:
  <path>     · verifies <behavior>          · covers §8 item(s)
  <path>     · verifies <regression case>

edit:
  <file:line> · adjust <what>

manual:
  1. <step>
  2. <step>

regression focus:  <symbol 1> <symbol 2>  (from §8)
```

---

## 10. Open Questions

```
[BLOCK]  <question>                           · need from: <P:role | doc | team>
[INFO]   <question>                           · can proceed w/ assumption: <X>
[GAP]    <codebase gap — e.g. schema drift>   · see §4 mismatches
```

If none: `—`

---

## 11. Rollback

```
code:       revert <commit-ref> | toggle flag <name> = off
db:         down-migration <file> | none (non-ddl)   · online? y/n  · backfill-blocked? y/n
data:       irreversible? <list or none>
flag:       recommended? y/n  · name: <flag>         · default: off
```

---

## 12. Search Trail  *(optional, recommended for future reuse)*

```
grep "<term>" services/user-api/   N hits   · rel: <path>:<line>
grep "<term>" services/admin-api/  N hits   · rel: <path>:<line>
similar-feature scan "<term>"               · hit: <path>:<line>
live: DESCRIBE user_login          ✓ dev-replica
live: EXPLAIN <query>              ✓ uses idx_ul_user
```

---

## 13. Repro Header for Follow-up Sessions

When a later session needs to continue, these three lines + the legend + §2 Meta are usually enough to reload the context:

```
kind=<B|F|R>   slug=<slug>   yyyymm=<YYYYMM>
scope=<services>   touches=<tables>   risk=<○|◐|●>
next=<single pending action or "awaiting-user-approval">
```
