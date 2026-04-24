# Repeatable Suite Discipline

> Phase-2 / Phase-6 companion to `SKILL.md`. This is the file that enforces the core user requirement: **"repeatable script that can be used again and again for testing, always updated."**

The spec suite under `.backend/<YYYYMM>/<slug>/e2e/specs/` is not a throwaway artifact of one `/qa-engineer` invocation. It is the **durable regression harness** for this feature. Every subsequent deploy re-runs it. Every subsequent `/qa-engineer` invocation for the same ticket updates it in place.

## The Four Cardinal Rules

### Rule 1 — Preserve, don't regenerate

On re-invocation for an existing `<YYYYMM>/<slug>`, **load every existing `specs/*.spec.ts` into memory and preserve anything unchanged.**

- A spec whose flow is unchanged per `explore.md §5B` and `work.md §4` must not be touched. Not rewritten, not re-formatted, not re-commented.
- A spec that passed in the last run and whose underlying contract has not changed is load-bearing history. Accumulated tweaks (timing waits, retry loops, boundary adjustments) represent real debugging work. Throwing them away to regenerate "cleaner" code silently regresses the suite.

Test: after running `/qa-engineer` on an unchanged ticket, `git diff e2e/specs/` must be empty for untouched flows. If the diff shows cosmetic-only changes on a flow whose contract didn't move, that is a discipline violation — roll back and retry.

### Rule 2 — Minimum-delta edit

When a flow's contract changes (endpoint shape changed in `/verify`, UI route moved, button text updated):

- Identify **exactly** which assertions or locators need to change.
- Edit **only those lines.** Preserve test names. Preserve variant ordering. Preserve comments explaining prior workarounds.
- Never reorder tests, never rename `test.describe` blocks, never change formatting.
- Record the edit summary in `qa.md §6 Spec Suite Changes`: which spec, which lines, what changed, why.

Example — OK:
```diff
  test('happy — add member to project', async ({ page }) => {
    await page.goto('/projects/1001');
-   await page.getByRole('button', { name: 'Add member' }).click();
+   await page.getByRole('button', { name: 'Invite member' }).click();  // qa:2026-05-03 button label change
    await page.getByTestId('member-picker').selectOption('user-42');
    await expect(page.getByTestId('member-count')).toContainText('1');
  });
```

Example — NOT OK (reformat + unrelated refactor sneaked in):
```diff
- test('happy — add member to project', async ({ page }) => {
-   await page.goto('/projects/1001');
-   await page.getByRole('button', { name: 'Add member' }).click();
-   await page.getByTestId('member-picker').selectOption('user-42');
-   await expect(page.getByTestId('member-count')).toContainText('1');
- });
+ test('happy path — member add', async ({ page }) => {  // renamed — bad
+   await page.goto('/projects/1001');
+   const btn = page.getByRole('button', { name: 'Invite member' });
+   await btn.click();  // extracted to var — unrelated — bad
+   await page.locator('[data-testid="member-picker"]').selectOption('user-42');  // locator style change — bad
+   expect(await page.getByTestId('member-count').textContent()).toContain('1');  // assertion pattern change — bad
+ });
```

### Rule 3 — Never delete silently

A spec whose feature was removed from the product goes through this path:

1. Verify the removal is intentional by reading `/work`'s most recent `work.md §4` entry — the feature removal must be explicitly logged there.
2. Mark the spec `.skip` in place with a breadcrumb comment:
   ```typescript
   test.skip('happy — legacy activity view', async ({ page }) => {
     // [REMOVED-FEATURE qa:2026-08-14]
     // Flow removed per work.md 2026-08-14 §4 row 3 — legacy activity view deprecated for new timeline widget.
     // Kept as .skip for 2 release cycles in case of rollback. Delete after 2026-10-14 if not needed.
   });
   ```
3. Log an entry in `qa.md §9 [REMOVAL]` with: spec file, variant name, removal-approved-by (the `work.md` reference), and the date to re-evaluate.
4. Only delete the spec after the grace period has passed AND user explicitly consents in a later invocation.

Silent deletion erases audit trail. Future operators lose the evidence that a flow used to exist.

### Rule 4 — Smoke is sacred

`e2e/specs/smoke.spec.ts` is the post-deploy gate. It must:
- Complete in **under 90 seconds** (excluding the `_login.sh` bootstrap).
- Contain **only happy-path variants**, one per critical flow.
- Have **no external dependencies** beyond services the deployed env itself owns (mock or stub any uncontrolled third-party).
- Always record `trace: on` and `video: on` (not just on failure).

When the smoke suite grows beyond 90s, split: move lower-criticality flows out to a separate `specs/regression-<date>.spec.ts` and keep smoke lean. Do not relax the 90s target — a smoke suite people don't run is worse than no smoke suite.

The user (or CI) re-runs smoke after every deploy:
```bash
cd .backend/<YYYYMM>/<slug>/e2e
QA_ENV=dev  npx playwright test specs/smoke.spec.ts
QA_ENV=stg  npx playwright test specs/smoke.spec.ts
QA_ENV=prod npx playwright test specs/smoke.spec.ts  # only if prod QA is authorized
```

This command goes verbatim into `qa.md §10 Re-entry Header` so the operator doesn't have to re-read the whole file to re-verify.

## Suite Growth Over Time

A ticket that gets re-QA'd across 3 deploys (dev → stg → prod) typically ends with:

```
e2e/specs/
  smoke.spec.ts                ← 4-6 happy paths, <90s
  login.spec.ts                ← happy + negative
  create-item.spec.ts          ← happy + boundary + negative + regression
  realtime-room.spec.ts        ← happy + WebSocket assertion
  broadcast.spec.ts            ← happy + SSE assertion
```

After one prod hotfix:
```
e2e/specs/
  smoke.spec.ts
  login.spec.ts
  create-item.spec.ts
  realtime-room.spec.ts
  broadcast.spec.ts
  hotfix-2026-05-12.spec.ts    ← regression variant covering the prod-only symptom
```

The hotfix spec does not merge into the parent flow file — it stays separate so the audit trail is clear. After 2 release cycles with no recurrence, it may be merged into the parent flow as a `regression` variant, at which point the standalone file is deleted with a `[REMOVAL]` entry.

## Cross-Ticket: Building the Global Suite

Each ticket's `e2e/specs/` is self-contained. But a deployed env has many tickets' worth of features in production at once. A CI job (or a manual post-deploy pass) should run **every ticket's smoke suite** for the current active quarter:

```bash
# Pseudo-CI — not part of this skill, but the target usage pattern
for ticket_dir in .backend/$(date +%Y%m)/*/; do
  [ -f "${ticket_dir}e2e/specs/smoke.spec.ts" ] || continue
  (cd "${ticket_dir}e2e" && QA_ENV=$QA_ENV npx playwright test specs/smoke.spec.ts)
done
```

When a ticket's feature is fully cut over and no longer needs individual monitoring, the user may **retire** the ticket's spec suite:
- Move `e2e/specs/smoke.spec.ts` flows into a shared `e2e-global/smoke/` directory (out of scope for this skill, but documented here).
- Archive the ticket's `e2e/specs/` as read-only evidence.
- Log retirement in `qa.md §11 History` with the final date and the destination.

`/qa-engineer` does not perform this retirement automatically — it is an explicit operator decision. But the skill should surface the opportunity in `qa.md §9 [INFO]` when it detects a ticket is 90+ days old and has had no contract drift in the last 2 runs.

## Handling a Flaky Suite

A flaky spec in the suite is worse than a missing spec — it trains operators to ignore failures.

When a variant shows `pass,fail,pass` under the 3-attempt protocol (SKILL.md Phase 3 step 5):

1. Record the flake in `qa.md §9 [FLAKE]` with: spec file, variant, attempt outcomes, environmental factors (time of day, which env, concurrent load).
2. **Do not** mark the spec `.skip`. Skipping hides flakes.
3. **Do not** add arbitrary `waitForTimeout` calls. Those mask flakes rather than fixing them.
4. **Do** investigate:
   - Is the underlying flow itself flaky (a real race condition users can hit)? → Escalate to `/work` as FAIL-UI / FAIL-BACKEND.
   - Is the assertion timing-sensitive (the UI takes a variable time to settle)? → Switch from instantaneous assertion to `expect(locator).toBeVisible({ timeout: 10_000 })` — scoped timeout, not global sleep.
   - Is the env itself flaky (a given deploy-env has known instability)? → Record in `[FLAKE]` with env as root cause; prod run still must pass without flake.
5. A flake that persists 3 runs in a row across different days is not a flake — it is a failure mode. Re-classify as FAIL-UI or FAIL-BACKEND and escalate.

## Never

- **Never** auto-generate specs via Playwright codegen (`npx playwright codegen`) for a ticket that already has hand-maintained specs. Codegen produces verbose, fragile specs with CSS selectors. Codegen is a Phase-2 scratchpad, not a product.
- **Never** commit `video.webm` for passing specs. Passing videos are noise. Only failure videos + always-on smoke videos go into `artifacts/`.
- **Never** delete `artifacts/run-*` folders to save space from this skill. Old run artifacts are the audit trail. Pruning is a separate operator decision.
- **Never** write a spec that depends on another spec having run first. Each test is independent — use `test.beforeEach` for setup, not a sequence.
- **Never** write `test.serial` unless the underlying business flow is genuinely sequential (e.g. first create, then verify). And in that case, extract to a single `test()` with steps, not separate tests in serial mode.

## The Test

After every `/qa-engineer` run, ask: **"If the user re-ran this exact smoke suite on the next deploy without me present, would the result be trustworthy?"**

- If the spec uses a timestamp-based locator that only works today → no, rewrite.
- If the spec relies on seeded data that gets cleaned up nightly → no, the fixture needs a durable anchor.
- If the spec logs an ignored console warning that turns into an error next month → no, assert on console errors now.
- If the spec passes only when run alone, not when parallel with siblings → no, isolate the shared state.

If the answer to all above is "yes, it would still pass meaningfully on next deploy," the spec is durable and belongs in the suite. Otherwise, iterate before marking Phase 3 complete.
