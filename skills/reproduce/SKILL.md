---
name: reproduce
description: Use when a user-reported or ticketed browser issue must be reproduced in an actual Playwright browser before coding, especially when exact URL, account/role, data state, screenshots, network calls, and as-is/to-be comparison evidence are needed.
---

# Reproduce

## Overview

`/reproduce` is the browser-first gate before fixing UI-visible issues. It captures the current **as-is** behavior with a runnable Playwright script or spec, raw evidence, and optional read-only data proof so `/explore`, `/work`, and later to-be checks can compare against the same user path.

Core principle: reproduce the user's exact issue before explaining or fixing it. If the browser path, account, URL, data row, or expected/actual behavior is unclear, surface that gap instead of inventing a flow.

## Iron Law

```text
NO CODE FIX BEFORE THE AS-IS REPRODUCTION STATUS IS RECORDED
NO BROWSER RUN OR SCRIPT CREATION BEFORE A REPRODUCTION PLAN IS PRESENTED
NO SYNTHETIC USER STEPS WHEN THE TICKET OR USER PROVIDED EXACT STEPS
NO PASS CLAIM WITHOUT PLAYWRIGHT COMMAND OUTPUT AND ARTIFACT PATHS
NO DATA MUTATION FROM THIS SKILL; READ-ONLY DATA CHECKS ONLY
NO PROD DATA CHANGE OR PROD-ONLY EXPERIMENT
```

## When to Use

Use when the user says `/reproduce`, asks to reproduce a browser issue, asks for as-is/to-be comparison setup, or provides a URL/account/screenshot and wants the current issue captured before a fix.

Use especially for:

- Browser symptoms where API-only curl is not enough.
- Ticketed bugs where expected behavior and current rendered behavior must be compared.
- Issues needing real login, role/account choice, network capture, screenshot, trace, console log, or DB-backed data confirmation.
- Pre-fix regression setup: as-is proves the bug exists; to-be later proves the fix.

Skip for:

- Backend API-only contracts with no browser surface: use `/verify`.
- Pure code investigation with no live UI reproduction need: use `/explore`.
- Already-fixed flows where the user only asks for deployed smoke testing: use `/qa-engineer`.

## Required Start

Before any file edit or browser run, state:

1. Problem: exact symptom and ticket/user source.
2. Goal: what must be reproduced and what evidence will prove it.
3. Affected area: URL, service/UI, account/role, data identifiers if known.
4. Expected outcome: `reproduced`, `not-reproduced`, `blocked-env`, `blocked-data`, or `inconclusive`.
5. The smallest suitable reproduction approach; compare alternatives only when they change a material tradeoff.

Default recommendation:

- Use the project's existing Playwright setup when present.
- Write one focused as-is spec under the existing e2e test tree, or `.backend/<YYYYMM>/<slug>/e2e/specs/` if no stronger convention exists.
- Save the persistent report and raw evidence under `.backend/<YYYYMM>/<slug>/`.

Ask only for missing inputs that cannot be safely inferred, such as account, target environment, exact URL, or permission for any shared-env run.

## Inputs

| Source | What to extract |
|---|---|
| Ticket key/URL | Summary, description, comments, attachments, exact steps, expected/actual. |
| User message | Target URL, role/account, data identifiers, exact click path, expected/actual wording. |
| Existing E2E setup | Playwright config, page objects, nearby `*-as-is.spec.ts` and `*-to-be.spec.ts`. |
| Browser runtime | Screenshots, trace/video, console, page errors, request failures, relevant API responses. |
| Data state | Optional read-only SQL/API checks for data existence and shape. |

## Safety Bounds

| Target | Allowed | Guardrail |
|---|---|---|
| local/dev | Browser reproduction, read-only DB/API checks | Bound samples, redact PII |
| stg/audit/shared test | Browser reproduction with user-approved account/session | No destructive setup; record target URL |
| prod | Read-only browser observation only | No DB checks, no data mutation, no destructive/user-visible experiment |

If the target environment is ambiguous, ask before running. Production observation requires explicit user instruction and every action remains read-only; do not request the same confirmation again.

## Artifact Layout

Default to repo root unless the issue clearly belongs to a service with its own `.backend` convention:

```text
.backend/<YYYYMM>/<slug>/
  reproduce.md
  runs/run-1.log
  network/*.json
  screenshots/*.png
  db/*.md
  e2e/
    specs/<ticket-or-slug>-as-is.spec.ts
    specs/<ticket-or-slug>-to-be.spec.ts   # only if useful or explicitly requested
    artifacts/run-<YYYYMMDD-HHMM>-<env>/
```

If the repo already has a Playwright test tree, keep the spec there and store the raw evidence/report under `.backend/<YYYYMM>/<slug>/`.

## Status Semantics

| Status | Meaning |
|---|---|
| `reproduced` | Browser evidence shows the reported bug on the selected target. |
| `not-reproduced` | The exact path ran, but the reported bug did not appear. Record differences; do not call it fixed. |
| `blocked-env` | Login, URL, service, browser, auth, or downstream environment prevented the run. |
| `blocked-data` | Required account/content/database state is missing or ambiguous. |
| `inconclusive` | Evidence conflicts or the issue lacks enough specificity to decide. |

For as-is specs, it is valid for the test to **pass when the bug is observed**. Name that assertion clearly as reproduction proof. A separate to-be spec should fail before the fix and pass after the fix when the corrected behavior is known.

## Phases

Complete each phase before the next.

### Phase 0: Plan

1. Classify the issue as browser-visible bug, browser-visible feature gap, or mixed browser/API issue.
2. Restate the exact reported symptom and expected behavior.
3. Identify target env and safety bound:
   - Default: local/dev/shared-test browser run.
   - Prod browsing is read-only only and requires explicit user instruction.
   - Data checks are read-only and non-prod only.
4. Choose artifact paths:
   - `<YYYYMM>` from current date.
   - `<slug>` from ticket key + short issue name, kebab-case.
5. Present a reproduction plan with:
   - target URL/env
   - account/role
   - data identifiers
   - browser steps
   - Playwright files to create/edit
   - network/console/screenshot artifacts
   - DB/API checks if needed
   - expected status criteria

Then proceed when the user requested reproduction. If the plan reveals risky ambiguity, ask the shortest blocking question.

### Phase 1: Load Issue and Existing Harness

1. If a ticket key or URL is present, read the ticket body before searching code.
2. Inspect existing Playwright config, specs, page objects, and test helpers.
3. Prefer existing login/navigation helpers instead of reimplementing them.
4. If no Playwright setup exists, create a one-off Playwright Test harness under `.backend/<YYYYMM>/<slug>/e2e/` after recording that no harness exists.

Record the issue source and harness choice in `reproduce.md`.

### Phase 2: Write the As-Is Spec

Create the smallest spec or script that follows the exact reported path.

Minimum capture:

- console messages
- page errors
- request failures
- relevant API responses, redacted
- screenshot before/after target action
- Playwright trace/video when available

Spec pattern:

```typescript
import { test, expect } from '@playwright/test';
import * as fs from 'fs';
import * as path from 'path';

const SLUG = 'ticket-or-short-slug';
const ARTIFACT_DIR = process.env.REPRO_ARTIFACT_DIR || path.resolve('.backend', 'YYYYMM', SLUG);
const SCREENSHOT_DIR = path.join(ARTIFACT_DIR, 'screenshots');
const NETWORK_DIR = path.join(ARTIFACT_DIR, 'network');

test.beforeEach(async ({ page }) => {
  fs.mkdirSync(SCREENSHOT_DIR, { recursive: true });
  fs.mkdirSync(NETWORK_DIR, { recursive: true });

  page.on('pageerror', error => {
    fs.appendFileSync(path.join(NETWORK_DIR, 'page-errors.log'), `${new Date().toISOString()} ${error.message}\n`);
  });
});

test('AS-IS: reported symptom is visible', async ({ page }) => {
  await page.goto(process.env.REPRO_URL!);
  // Follow the exact user/ticket steps here.
  await page.screenshot({ path: path.join(SCREENSHOT_DIR, 'as-is-target.png'), fullPage: true });

  // Assert the current bug condition as reproduction proof.
  await expect(page.getByText(/reported wrong text|error message/i)).toBeVisible();
});
```

Adjust only slug, URL/env, auth/bootstrap, navigation steps, and the final as-is assertion.

### Phase 3: Execute and Capture Evidence

Run the focused spec with the selected target URL and env:

```bash
REPRO_URL='<target-url>' npx playwright test <spec-path> --reporter=list
```

For headed debugging when needed:

```bash
REPRO_URL='<target-url>' npx playwright test <spec-path> --headed
```

Save command output to `.backend/<YYYYMM>/<slug>/runs/run-1.log`. Preserve Playwright trace, video, and screenshot paths when produced.

### Phase 4: Optional Read-Only Data Check

Use only when the browser result depends on a specific row/account/content state.

Rules:

- Non-prod only.
- Read-only only: `SELECT`, `SHOW`, `DESCRIBE`, `EXPLAIN`, or safe read-only project APIs.
- Bound samples with `LIMIT`.
- Redact PII before saving evidence.

Save findings under `.backend/<YYYYMM>/<slug>/db/*.md` and cite them in `reproduce.md`.

### Phase 5: Report

Write `.backend/<YYYYMM>/<slug>/reproduce.md` with:

1. Summary: status, target, one-line conclusion.
2. Source: ticket/user message, exact symptom, expected behavior.
3. Plan: env, account/role, data identifiers, browser steps.
4. Evidence: run command, exit code, screenshots, trace/video, console/network files.
5. Result: `reproduced` / `not-reproduced` / `blocked-env` / `blocked-data` / `inconclusive`.
6. As-is assertion: why the spec proves or fails to prove the symptom.
7. Next step: `/explore`, `/work`, `/verify`, `/qa-engineer`, or ask user for missing input.

Final response:

- Absolute path to `reproduce.md`.
- Final status.
- Exact command that was run.
- Key artifact paths.
- Short next step.

## Red Flags

- Inventing a click path because the ticket is vague.
- Calling `not-reproduced` a fix.
- Running prod actions that mutate data.
- Saving real names, emails, tokens, cookies, or IDs in `.backend/`.
- Using screenshots alone when console/network evidence is available.
- Treating a passing as-is reproduction spec as corrected behavior.

## Cross-References

- `/explore` consumes `reproduce.md` as concrete symptom evidence.
- `/work` may add a to-be spec or convert the as-is assertion into a regression test.
- `/verify` proves backend HTTP contracts after the fix.
- `/qa-engineer` proves the fixed deployed browser flow.
