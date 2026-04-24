# Playwright Playbook

> Phase-2 companion to `SKILL.md`. Upstream reference: Microsoft's `playwright-cli` skill at https://github.com/microsoft/playwright-cli/tree/main/skills/playwright-cli — this playbook assumes that skill is loaded or its SKILL.md has been read. This file covers **the generic cli → spec translation pattern**, auth bootstrap templates, and realtime (SSE/WebSocket) assertions. Fork it and add your app's specific routes, auth flow, and selectors.

## Two-Tool Split

| Tool | Purpose | When |
|---|---|---|
| `playwright-cli` | **Probe.** Interactive browser driver. One command at a time. Produces page snapshots and scripted shell files. | Phase 2 exploration: walking a flow for the first time, capturing the DOM shape, debugging a failure, teaching a new flow. |
| `@playwright/test` (via `npx playwright test`) | **Harness.** Spec-file-driven test runner with assertions, retries, fixtures, parallelism, HTML reports, trace viewer. | Phase 3 execution: re-running the accumulated spec suite on every deploy. |

**Rule**: never use `playwright-cli` as the Phase-3 execution mechanism. cli commands are not assertions — they are actions. A `click e5` that "didn't throw" is not a pass. Translate cli sequences into `.spec.ts` with real `expect()` calls.

## cli → spec Translation Pattern

Walk the flow once with cli, transcribing every command into `e2e/scripts/<flow>.sh`. Then translate to spec.

**cli probe** (in `scripts/login.sh`):

```bash
#!/usr/bin/env bash
set -euo pipefail
source ../_env.sh

playwright-cli -s=qa-login open "$BASE_URL/login"
playwright-cli -s=qa-login --raw snapshot > ../artifacts/login-before.yml
playwright-cli -s=qa-login fill e3 "$QA_TEST_USER_ID"
playwright-cli -s=qa-login fill e5 "__PASSWORD_FROM_ENV__"
playwright-cli -s=qa-login click e7
playwright-cli -s=qa-login --raw snapshot > ../artifacts/login-after.yml
playwright-cli -s=qa-login close
```

**Translated spec** (in `specs/login.spec.ts`):

```typescript
import { test, expect } from '@playwright/test';

test.describe('login', () => {
  test('happy — valid credentials land on dashboard', async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill(process.env.QA_TEST_USER_ID!);
    await page.getByLabel('Password').fill(process.env.QA_TEST_PASSWORD!);
    await page.getByRole('button', { name: 'Sign in' }).click();

    await expect(page).toHaveURL(/\/dashboard/);
    await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();
  });

  test('negative — wrong password shows error', async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill(process.env.QA_TEST_USER_ID!);
    await page.getByLabel('Password').fill('wrong-password-123!');
    await page.getByRole('button', { name: 'Sign in' }).click();

    await expect(page.getByRole('alert')).toContainText(/invalid|incorrect/i);
    await expect(page).toHaveURL(/\/login/);
  });
});
```

Key differences on translation:

- `eN` numeric refs → **semantic locators** (`getByRole`, `getByLabel`, `getByTestId`). Numeric refs change per snapshot; semantic locators survive UI churn.
- Inline passwords → `process.env` references loaded from `_env.sh`.
- Final `snapshot > after.yml` → `expect(page).toHaveURL(...)` + `toBeVisible()` assertions.
- Error paths get their own `test()` block, not a branch inside the happy path.

## Login Flows — Pick What Matches Your App

Replace the examples below with your app's actual flow. Three common patterns:

### Pattern A — Dev env: test JWT injection (fastest, recommended for dev)

If your backend has a dev-profile-only "test token" endpoint (see `verify/jwt-auth-reference.md`), bypass UI login by injecting the token directly:

```typescript
// e2e/_login.ts — called from playwright.config.ts `globalSetup`
import { chromium } from '@playwright/test';

export default async function globalSetup() {
  const res = await fetch(
    `${process.env.API_BASE_URL}/test/generate-jwt?userId=${process.env.QA_TEST_USER_ID}`
  );
  const { token } = await res.json();

  const browser = await chromium.launch();
  const ctx = await browser.newContext();
  await ctx.addInitScript((t) => {
    localStorage.setItem('accessToken', t);  // confirm key name from the web app's auth utility
  }, token);
  await ctx.storageState({ path: 'storageState.dev.json' });
  await browser.close();
}
```

Before committing `storageState.dev.json`:

- Redact any cookie with a real session ID.
- Keep only the `localStorage.accessToken` entry (it's already a short-lived dev token tied to a test account).
- Add a comment in `_env.sh` noting the regeneration command.

### Pattern B — Stg / audit env: interactive SSO via playwright-cli

For SSO-gated environments, walk the login once, capture storage state, reuse. Re-run when the SSO session expires (typically 30–60 min).

```bash
#!/usr/bin/env bash
# e2e/_login.sh for stg
source ./_env.sh

playwright-cli -s=qa-login open "$BASE_URL"
playwright-cli -s=qa-login snapshot
playwright-cli -s=qa-login click "getByRole('button', { name: 'Sign in with SSO' })"
# User-driven fill — SSO is out-of-band; ask the user to complete login in the browser window
echo "Complete the SSO login in the opened browser window, then press Enter..."
read -r
playwright-cli -s=qa-login state-save "storageState.stg.json"
playwright-cli -s=qa-login close
```

Do NOT automate typing real SSO credentials. Treat the SSO window as a user-driven step and wait for the operator to complete it. Spec files reuse the saved `storageState.stg.json` non-interactively.

### Pattern C — Prod env: test account credentials per session

Same pattern as stg, but:

- Credentials are provided by the user **in-session only**. They do not get written to disk except as part of `storageState.prod.json` (which itself holds a redacted/short-lived session token, not the raw password).
- Every run against prod asks the user at Phase 0 before proceeding.
- The prod write gate in `environment-gates.md` applies.

## Snapshot-Driven Element Targeting

`playwright-cli snapshot` produces YAML listing every interactable element with a numeric ref (`e1`, `e2`, ...). Use refs for the initial probe; **never** for the spec file.

```bash
playwright-cli snapshot > current.yml
grep -E 'testId:|role:|name:' current.yml | head -40
# Use role-based locator in the spec
```

For unlabeled custom components, add a `data-testid` attribute to the frontend code (this is a `/work` change, not a `/qa-engineer` change — escalate via Phase 5 delta) rather than falling back to CSS selectors.

## SSE / WebSocket Assertions

For realtime updates pushed from the backend (Redis Pub/Sub → SSE, STOMP over WebSocket, Server-Sent Events, Phoenix Channels, etc.) use `page.waitForResponse` (SSE) or `page.waitForEvent('websocket')` (WS).

### SSE

```typescript
test('broadcast reaches subscriber browser', async ({ page }) => {
  await page.goto('/dashboard');

  const ssePromise = page.waitForResponse(
    resp => resp.url().includes('/sse/subscribe') && resp.status() === 200
  );
  // Trigger broadcast via a parallel API call (test harness uses curl in a sibling step).
  await ssePromise;

  // Assert the UI reflects the broadcast.
  await expect(page.getByRole('status', { name: /new notification/i })).toBeVisible();
});
```

### WebSocket (STOMP or raw WS)

```typescript
test('realtime counter increments on peer action', async ({ page, context }) => {
  const wsPromise = page.waitForEvent('websocket');
  await page.goto('/room/1001/live');
  const ws = await wsPromise;

  const frameReceived = new Promise<string>(resolve => {
    ws.on('framereceived', f => resolve(f.payload.toString()));
  });

  // Trigger: a peer joins in a second browser context
  const peer = await context.browser()!.newContext({ storageState: 'storageState.dev.peer.json' });
  const peerPage = await peer.newPage();
  await peerPage.goto('/room/1001/join');

  const frame = await frameReceived;
  expect(frame).toContain('PEER_JOINED');
  await expect(page.getByTestId('peer-count')).toContainText('2');
});
```

## Tracing & Video

Configure once in `playwright.config.ts`:

```typescript
export default defineConfig({
  use: {
    trace: 'on-first-retry',  // always traceable on failure
    video: 'retain-on-failure',
    screenshot: 'only-on-failure',
  },
  // Smoke gets always-on video/trace regardless of result
  projects: [
    {
      name: 'smoke',
      testMatch: /smoke\.spec\.ts/,
      use: { trace: 'on', video: 'on' },
    },
    {
      name: 'full',
      testMatch: /.*\.spec\.ts/,
      testIgnore: /smoke\.spec\.ts/,
    },
  ],
});
```

Open a trace:

```bash
npx playwright show-trace artifacts/run-20260421-1430-dev/trace.zip
```

Smoke suite always records — it is the one that runs on every deploy and becomes the reference trace for later comparison.

## Route Mocking (rarely needed here)

`/qa-engineer` tests the deployed system. Mocking a backend call defeats the point. The only legitimate use is:

- Isolating a flaky third-party dependency (e.g. an external SSO endpoint with known instability on the dev env) to reduce false negatives. Record mock usage explicitly in `qa.md §4` with the endpoint and the reason.

If you find yourself mocking the service under test, stop — you're writing an integration test, not a deployed-env QA.

## Debugging a Failing Spec

1. Re-run with `--headed --workers=1 --debug`: `npx playwright test specs/<flow>.spec.ts --headed --debug`.
2. Open the last trace: `npx playwright show-trace artifacts/.../trace.zip`.
3. Use `playwright-cli` in a parallel terminal to walk the same steps live and snapshot the DOM where the spec claimed a locator was missing.
4. If the locator is genuinely absent in the deployed build (not a timing issue), that is a FAIL-UI or FAIL-BACKEND per SKILL.md Phase 4.

## Per-Flow Spec Template

```typescript
import { test, expect } from '@playwright/test';
import fixtures from '../fixtures/<flow>.json';

test.describe('<flow>', () => {
  test.beforeEach(async ({ page }) => {
    // storageState is loaded via playwright.config.ts projects per env
    await page.goto('/');
  });

  test('happy — <one-line outcome>', async ({ page }) => { /* ... */ });
  test('boundary — <one-line outcome>', async ({ page }) => { /* ... */ });
  test('negative — <one-line outcome>', async ({ page }) => { /* ... */ });

  // Bug fixes only:
  test('regression — <original symptom> does not recur', async ({ page }) => { /* ... */ });
});
```

Keep spec file size under 300 lines. Split by flow, not by variant — one file per flow, variants as tests inside.
