# Playwright Conventions

Cross-project conventions for Playwright E2E suites at Selise. Distilled from Playwright's [official best practices](https://playwright.dev/docs/best-practices) and proven patterns from `sbh-e2e`. Adopt when starting a new suite; reach for it when reviewing PRs.

> **Tooling baseline:** `@playwright/test` ≥ 1.58, Node 20+, yarn 4 (corepack-managed), TypeScript strict mode.

---

## Reference project structure

Every Playwright suite should follow this layout — same folder names, same nesting depth, same purpose per folder. Placeholders in angle brackets (`<project>`, `<Module>`, `<feature>`) are filled in per project.

```
apps/<project>-e2e/
├── eslint.config.mjs               # ESLint config (extends repo base + playwright plugin)
├── playwright.config.ts            # base config; usually exports a createConfig() factory
├── playwright.config.<env>.ts      # per-env thin re-exports (e.g. dev, staging, prod)
├── playwright.env.json             # secrets blob (gitignored, materialized in CI)
├── project.json                    # Nx project descriptor (if using Nx)
├── tsconfig.json                   # TS config (extends repo base)
└── src/
    ├── auth.setup.ts               # `setup` project test — writes .auth/auth.json
    ├── auth.setup.<env>.ts         # optional per-env auth setup variants
    ├── global-setup.ts             # one-time pre-suite cleanup (clears generated dirs, etc.)
    │
    ├── fixtures/
    │   ├── TestData/               # static test data (PDFs, images) + generated data factories
    │   │   ├── testData.ts         # randomised test data via @ngneat/falso
    │   │   └── <file>.<ext>        # static binaries (PDFs, images, etc.)
    │   └── <generated-dir>/        # any runtime-generated artifacts (gitignored)
    │
    ├── helpers/                    # cross-cutting utilities (no per-page logic)
    │   ├── utils.ts                # login(), navigation, listenForApiResponse(), ...
    │   ├── api_simulation.ts       # backend simulation endpoints (optional)
    │   └── <domain>_utils.ts       # other shared utilities (PDF assertions, etc.)
    │
    ├── pages/                      # page objects, one folder per FE module
    │   ├── <Module>/
    │   │   ├── <feature>.ts        # page object for a single feature flow
    │   │   └── <module>_utils.ts   # optional shared helpers within the module
    │   └── <Module>/
    │       └── <feature>.ts
    │
    ├── reporters/                  # custom Playwright reporters (optional)
    │   └── <name>-reporter.ts
    │
    └── tests/                      # spec files — folder names mirror pages/ 1:1
        ├── <Module>/
        │   └── <feature>.spec.ts
        └── <Module>/
            ├── <sub-grouping>/     # extra nesting allowed when a module has many sub-flows
            │   └── <feature>.spec.ts
            └── <feature>.spec.ts
```

**What goes where**

| Folder | Purpose |
|---|---|
| `src/auth.setup.ts` | One-time login; writes `storageState` to `.auth/auth.json` (see §5) |
| `src/global-setup.ts` | Idempotent pre-suite cleanup — wipe generated dirs, reset shared state |
| `src/fixtures/TestData/` | Static binaries (PDFs, images) + generated test data factories |
| `src/helpers/` | Cross-cutting utilities used by many page objects |
| `src/pages/<Module>/` | Page objects for one FE module — actions, no test logic |
| `src/tests/<Module>/` | Spec files — folder name mirrors `pages/<Module>/` |
| `src/reporters/` | Custom Playwright reporters |

**Rules**
- `tests/<Module>/` and `pages/<Module>/` folder names **must match 1:1**. If you add `tests/Foo/`, also add `pages/Foo/`.
- Spec files: `snake_case.spec.ts`. Page object files: `snake_case.ts`. Module folder names: `PascalCase` (or however the FE module is named).
- One feature flow per spec file. Don't dump unrelated tests into one file just because they share a module.
- **Never** put selectors or test logic in a spec file. Specs orchestrate; page objects act. Specs should read like a checklist.

---

## Table of contents

1. [Code style & naming](#1-code-style--naming)
2. [Installing & upgrading](#2-installing--upgrading)
3. [Locator strategy](#3-locator-strategy)
4. [Test ID attribute (`data-testid` vs `data-cy`)](#4-test-id-attribute-data-testid-vs-data-cy)
5. [Authentication](#5-authentication)
6. [`playwright.config.ts`](#6-playwrightconfigts)
7. [Page Object Model](#7-page-object-model)
8. [Writing tests](#8-writing-tests)
9. [Assertions](#9-assertions)
10. [Waits & synchronisation](#10-waits--synchronisation)
11. [Parallelism, retries, sharding](#11-parallelism-retries-sharding)
12. [Network & API interactions](#12-network--api-interactions)
13. [Test data](#13-test-data)
14. [Selecting dates](#14-selecting-dates)
15. [Mailosaur — email testing](#15-mailosaur--email-testing)
16. [Debugging](#16-debugging)
17. [Reporting](#17-reporting)
18. [CI — GitHub Actions](#18-ci--github-actions)
19. [Linting](#19-linting)
20. [rsync deployment](#20-rsync-deployment)
21. [Pull request convention](#21-pull-request-convention)
22. [Quick reference](#22-quick-reference)

---

## 1. Code style & naming

Use **camelCase** for variables, functions, methods and class members.

```ts
let fooVar;
function barFunc() {}
```

Use **PascalCase** for classes.

```ts
class Foo {}
```

Use **PascalCase** for interfaces — **do not** prefix with `I`.

```ts
interface User {}    // ✅
interface IUser {}   // ❌
```

Use **namespaces / modules** to group reusable utilities whose instance creation isn't necessary.

```ts
namespace Foo {}
```

Use **PascalCase** for enum names and members.

```ts
enum Color { Red, Blue }
```

### TODO comments

Always include the *why*.

```ts
// TODO: Reason for revisit here.
```

A bare `// TODO` is noise; future-you won't remember.

---

## 2. Installing & upgrading

Install:

```bash
yarn add -D @playwright/test
npx playwright install              # one-time browser download per machine
```

Upgrade:

```bash
yarn upgrade @playwright/test@<version>
npx playwright install              # always re-install browsers after upgrade
```

Pin the version. Drift between local and CI causes "works on my machine" reports.

---

## 3. Locator strategy

Playwright recommends [user-facing locators](https://playwright.dev/docs/best-practices#use-locators) — they auto-wait, auto-retry, and survive DOM churn. Pick the most stable available; stop at the first one that works.

| Priority | Locator | When to use |
|---|---|---|
| 1 | `page.getByTestId('...')` | The FE owns a stable hook for this element. **Always prefer.** Pair with the `testIdAttribute` config in §4. |
| 2 | `page.getByRole(role, { name })` | Stable, translation-fixed accessible name. |
| 3 | `page.getByLabel('...')` | Form fields with visible label text. |
| 4 | `page.getByPlaceholder('...')` | Inputs whose only identifier is the placeholder. |
| 5 | `page.getByText('...')` | Toasts, success messages, content assertions — never for interactive elements. |
| 6 | `page.locator('component-tag #id')` | Component host + framework-controlled `#id`. Last resort. |

**Never use**
- Brittle CSS chains (`.p-ripple > div > span:nth-child(2)`). Request a `data-cy`.
- XPath. Period.
- Class-based selectors from third-party libs (PrimeNG, Material, Tailwind). They change without notice.
- Text matching on user-visible copy when translation keys exist — other locales break it.

**Multi-element selection**
- Prefer `.filter({ hasText: '...' })` over index access.
- `.first()` / `.last()` / `.nth(n)` are escape hatches. If you reach for them more than once, the FE needs a unique `data-cy` per row.

**Chaining** — locators compose. Build them up from a stable parent:

```ts
const row = page.locator('tr').filter({ hasText: caseId });
await row.getByTestId('actionMenu').click();
await row.getByRole('button', { name: 'Delete' }).click();
```

**Strict mode** — Playwright fails if a locator resolves to more than one element. **Don't disable it.** If your locator is ambiguous, narrow it.

---

## 4. Test ID attribute (`data-testid` vs `data-cy`)

Playwright exposes `page.getByTestId('...')` as a first-class locator. By default it reads `data-testid`, but it can be pointed at any attribute — including legacy `data-cy` from a Cypress era. Choose based on the project's history.

### New project (greenfield Playwright)

**Use `data-testid`.** It's Playwright's default; no config knob needed.

```html
<button type="submit" data-testid="submitButton">Submit</button>
```

```ts
await page.getByTestId('submitButton').click();
```

### Existing project (already uses `data-cy`)

Don't migrate the FE. Point Playwright's `getByTestId` at the existing attribute via `testIdAttribute`:

```ts
// playwright.config.ts
export default defineConfig({
  use: {
    testIdAttribute: 'data-cy',
  },
});
```

Now `getByTestId` resolves `data-cy`. Tests stay consistent across the suite:

```html
<button type="submit" data-cy="submitButton">Submit</button>
```

```ts
await page.getByTestId('submitButton').click();   // resolves [data-cy="submitButton"]
```

### Rules (apply to whichever attribute you chose)

- **One project, one attribute.** Don't mix `data-cy` and `data-testid` in the same suite.
- Values are **camelCase**, action- or noun-oriented:
  - Action: `createCommission`, `deleteOpportunityButton`, `openContactSalesModal`, `saveChanges`, `cancel`.
  - Field: `firstName`, `lastName`, `email`, `phone`, `currentStatus`.
  - List / grid: `actionMenu`, `filter`, `tutorialMenu`, `selectBanner`.
- One element, one test ID. Don't reuse the value on unrelated elements.
- Add the attribute to **every** interactive element a test could plausibly need — buttons, form fields, list rows, modal action rows, status badges. Cheaper to add upfront than to chase a flake later.

---

## 5. Authentication

Playwright's idiomatic pattern: a one-time `setup` project performs login, writes `storageState` to disk; every test project reuses it via `dependencies: ['setup']`. **No login from individual specs — ever.**

```ts
// src/auth.setup.ts
import { test as setup } from '@playwright/test';
import * as path from 'path';
import * as fs from 'fs';
import { login } from './helpers/utils';

const authFile = path.join(__dirname, '../.auth/auth.json');

setup('Authenticate', async ({ page }) => {
  // CI: always re-auth to avoid stale session failures.
  // Local: reuse cached file to avoid burning TOTP codes during dev.
  if (!process.env['CI'] && fs.existsSync(authFile)) {
    return;
  }
  if (fs.existsSync(authFile)) {
    fs.rmSync(authFile);
  }
  await login(page, 'employee');
  await page.context().storageState({ path: authFile });
});
```

**Rules**
- Credentials live in `playwright.env.json` (gitignored). In CI, materialise from a single secret blob.
- Never log, echo, or check in credentials.
- If you need to *test* the login flow itself, create a separate setup project — don't pollute the default auth.
- The `setup` project is the **only** project allowed to retry (see §11).

---

## 6. `playwright.config.ts`

Minimal-but-correct config:

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './src',
  timeout: 120_000,
  fullyParallel: true,
  retries: 0,
  workers: process.env['CI'] ? 4 : undefined,

  reporter: [
    ['html', { outputFolder: './playwright-report', open: 'never' }],
    ['list'],
  ],

  use: {
    baseURL: process.env['BASE_URL'],
    trace: 'retain-on-failure',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    testIdAttribute: 'data-cy',     // remove if your project uses data-testid (the default)
  },

  projects: [
    {
      name: 'setup',
      testMatch: /auth\.setup\.ts/,
      retries: 3,
    },
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'], storageState: '.auth/auth.json' },
      dependencies: ['setup'],
    },
  ],
});
```

**Why these defaults**

| Setting | Rationale |
|---|---|
| `retries: 0` (global) | Feature tests must fail loudly. Retries hide real bugs. |
| `retries: 3` on `setup` | SSO / TOTP / federation can transiently flake. Tolerate there, nowhere else. |
| `workers: 4` in CI | Sweet spot for most apps. Tune up only if you have shard-safe test data. |
| `fullyParallel: true` | Tests within a file also parallelise. |
| `trace: 'retain-on-failure'` | Without retries, this is the only way to capture traces of failures. |
| `screenshot`, `video` on failure | Cheap to capture; invaluable for triage. |

For multi-env projects, factor with a `createConfig(env, authFileName, setupMatch)` helper so `playwright.config.dev.ts` and `playwright.config.tech.ts` are 3-liners.

---

## 7. Page Object Model

Both **functional** and **class-based** POMs are accepted. Playwright's own docs accept either; pick what fits the project and the team. Be consistent within a suite — mixing the two is the worst outcome.

### Functional (default recommendation)

```ts
// pages/Accounts/accounts.ts
import { Page, expect } from '@playwright/test';
import { listenForApiResponse } from '../../helpers/utils';

const contactDefaults = {
  salutation: 'Mr',
  language: 'EN',
  phone: '+41829883883',
} as const;

export async function addContact(page: Page, accountName: string): Promise<void> {
  await openAccount(page, accountName);
  await page.getByRole('button', { name: 'Create Contact' }).click();
  await fillContactDetails(page);

  const submit = page.getByRole('button', { name: 'Add Contact' });
  await Promise.all([
    listenForApiResponse(page, '/graphql', 'POST', '"operationName":"createAccountContact"', 200),
    submit.click(),
  ]);

  await expect(page.getByText('Contact created successfully')).toBeVisible();
}

async function openAccount(page: Page, accountName: string): Promise<void> {
  await page.getByPlaceholder('Search').fill(accountName);
  await page.getByText(accountName).click();
}

async function fillContactDetails(page: Page): Promise<void> {
  // ...
}
```

In a spec:

```ts
// tests/Accounts/customer_accounts.spec.ts
import { test } from '@playwright/test';
import { addContact } from '../../pages/Accounts/accounts';

test('Add contact to customer account', async ({ page }) => {
  await addContact(page, 'DummyTest Company');
});
```

**Why functional**
- **Composition** — flows compose across modules without `new SomePage(page)` ceremony.
- **Stateless** — most page actions start and end in known states. Nothing benefits from `this.x`.
- **Matches Playwright's API** — Playwright is functional. POMs read as a natural extension.

**Rules**
- Exported functions = public API. Helpers stay un-exported.
- Type parameters and return values (`Promise<void>` or `Promise<T>`). No implicit `any`.
- `page: Page` is always the first arg. Never store `page` in module state.
- For default form values, define a `camelCase` `as const` config at the top of the file.
- One feature flow per file. Don't dump unrelated flows into one page object.

### Class-based (also valid)

Class-based POMs are equally acceptable and shown in Playwright's official docs. Use them when:
- The team prefers OOP and is consistent about it across the suite.
- A flow is genuinely stateful (e.g. a wizard tracking `currentStep`, `selectedRow`, validation state).
- You want pre-computed locators encapsulated in the constructor.

```ts
// pages/Accounts/accounts-page.ts
import { Page, Locator, expect } from '@playwright/test';

export class AccountsPage {
  readonly page: Page;
  readonly searchInput: Locator;
  readonly createContactButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.searchInput = page.getByPlaceholder('Search');
    this.createContactButton = page.getByRole('button', { name: 'Create Contact' });
  }

  async open(accountName: string): Promise<void> {
    await this.searchInput.fill(accountName);
    await this.page.getByText(accountName).click();
  }

  async addContact(accountName: string): Promise<void> {
    await this.open(accountName);
    await this.createContactButton.click();
    // ...
    await expect(this.page.getByText('Contact created successfully')).toBeVisible();
  }
}
```

In a spec:

```ts
import { test } from '@playwright/test';
import { AccountsPage } from '../../pages/Accounts/accounts-page';

test('Add contact to customer account', async ({ page }) => {
  const accounts = new AccountsPage(page);
  await accounts.addContact('DummyTest Company');
});
```

**Rules for class-based POMs**
- File name in `kebab-case.ts`; class name in `PascalCase`. Suffix with `Page` (`AccountsPage`, `OpportunityPage`).
- Constructor takes `page: Page` and assigns pre-computed locators to `readonly` fields.
- Methods are typed (`Promise<void>` / `Promise<T>`). Public methods describe user actions; private helpers stay private.
- One class per file. Don't bundle `AccountsPage` and `OpportunityPage` together.

### Pick one style per project

**Don't mix.** A suite where half the pages are functional and half are class-based is the worst outcome — readers context-switch on every file. Decide up front, document it in the project's README, and migrate any holdovers when you touch them.

---

## 8. Writing tests

### Structure

```ts
import { test, expect } from '@playwright/test';
import { navigateToModule } from '../../helpers/utils';
import { addContact } from '../../pages/Accounts/accounts';

test.describe('Accounts - Customer Accounts', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/dashboard');
    await navigateToModule(page, 'Customers', 'Accounts');
    await expect(page.getByText('Here you will find all the customers')).toBeVisible();
  });

  test('Add contact', async ({ page }) => {
    await addContact(page, 'DummyTest Company');
  });
});
```

### Test isolation

Per Playwright's docs: **each test should be runnable independently.** Don't chain tests where `test B` depends on side effects from `test A`. Either:
- Make each test self-contained (create + clean up its own data), or
- Mark the file serial: `test.describe.configure({ mode: 'serial' })` (use sparingly).

### Hooks

| Hook | Use for |
|---|---|
| `test.beforeEach` | Navigate to a starting page; common pre-conditions. |
| `test.afterEach` | Clean up test data the test created (when feasible). |
| `test.beforeAll` / `afterAll` | Heavyweight setup that's idempotent across tests in the file. |

Don't put login in any hook — that's the `setup` project's job (see §5).

### Test titles

Describe the user goal, not the implementation.

```ts
test('Add billing account to customer', ...);  // ✅
test('Click Add Billing Account button', ...); // ❌
```

### Tags (optional)

Use Playwright's [`@tag` annotations](https://playwright.dev/docs/test-annotations#tag-tests) to filter at run time:

```ts
test('critical smoke flow', { tag: '@smoke' }, async ({ page }) => { ... });
```

Run only smoke: `npx playwright test --grep @smoke`.

---

## 9. Assertions

**Use web-first assertions.** They auto-retry until the timeout, eliminating an entire class of race conditions.

```ts
// ✅ Auto-retries until visible or timeout
await expect(page.getByText('Saved')).toBeVisible();

// ❌ One-shot — races with rendering
expect(await page.getByText('Saved').isVisible()).toBe(true);
```

Common assertions:

```ts
await expect(locator).toBeVisible();
await expect(locator).toBeHidden();
await expect(locator).toHaveText('exact text');
await expect(locator).toContainText('partial');
await expect(locator).toHaveCount(3);
await expect(locator).toHaveValue('input value');
await expect(locator).toBeEnabled();
await expect(locator).toBeDisabled();
await expect(page).toHaveURL(/\/dashboard/);
await expect(page).toHaveTitle('Sunrise Business Hub');
```

### Soft assertions

For non-fatal checks where the test should continue even if this one fails:

```ts
await expect.soft(page.getByText('Optional banner')).toBeVisible();
// Test continues; soft failure reported at end.
```

### Custom timeouts

Use sparingly — needing a longer timeout often means the test is racing on the wrong signal.

```ts
await expect(page.getByText('Slow toast')).toBeVisible({ timeout: 30_000 });
```

### One terminal assertion per test

End every flow with an assertion that verifies the user outcome — not "the button got clicked."

---

## 10. Waits & synchronisation

The #1 cause of flake. Get it right.

### Forbidden

```ts
await page.waitForTimeout(5000);             // ❌ never
await new Promise(r => setTimeout(r, 3000)); // ❌ never
```

### Preferred

```ts
// Wait for a UI signal
await expect(page.getByText('Saved')).toBeVisible();
await locator.waitFor({ state: 'visible' });

// Wait for an API response — pair with the trigger using Promise.all
await Promise.all([
  page.waitForResponse(r => r.url().includes('/graphql') && r.status() === 200),
  page.getByRole('button', { name: 'Submit' }).click(),
]);

// Wait for a specific GraphQL operation
await Promise.all([
  listenForApiResponse(page, '/graphql', 'POST', '"operationName":"createBillingAccounts"', 200),
  submit.click(),
]);

// Last resort, when nothing more specific works
await page.waitForLoadState('networkidle');
```

**Rule:** the response listener must be set up **before** the action that triggers the request. `Promise.all` makes this safe — the listener is armed in parallel with the click, so the response can't slip through before you're listening.

### Reusable helper

```ts
// helpers/utils.ts
export function listenForApiResponse(
  page: Page,
  url: string,
  method: string,
  postDataFragment: string,
  status: number,
  timeout = 60_000,
) {
  return page.waitForResponse(
    r =>
      r.url().includes(url) &&
      r.request().method() === method &&
      (r.request().postData()?.includes(postDataFragment) ?? false) &&
      r.status() === status,
    { timeout },
  );
}
```

### Backend simulation

For flows that wait on backend state changes (approvals, status transitions), call simulation endpoints rather than polling the UI:

```ts
await simulationSolutionReady(opportunityId);
await simulationBcApproval(opportunityId);
await page.reload();
await expect(page.getByText('BC Approved')).toBeVisible();
```

---

## 11. Parallelism, retries, sharding

### Parallelism

- `fullyParallel: true` — tests within a file run in parallel.
- `workers: 4` in CI — default ceiling for most apps.
- If tests in a file share state, mark it serial:

  ```ts
  test.describe.configure({ mode: 'serial' });
  ```

### Retries

- Global `retries: 0`. Feature tests fail loudly.
- Only `setup` retries (`retries: 3`) because auth has external dependencies.
- **Never** `test.retry()` from a spec.

### Sharding

For very large suites, split across machines using `--shard`:

```bash
npx playwright test --shard=1/4
npx playwright test --shard=2/4
# ... up to 4/4
```

In GitHub Actions, use a `matrix.containers: [1, 2, 3, 4]` strategy and pass each shard to its own job.

### Running only the failed specs

Playwright tracks last-run state:

```bash
npx playwright test --last-failed
```

Programmatic retry (when `--last-failed` isn't enough):

```ts
// scripts/playwright-retry.ts
import { spawnSync } from 'child_process';
import * as fs from 'fs';

spawnSync('npx', ['playwright', 'test', '--reporter=json'], { stdio: 'inherit' });

const resultPath = 'test-results/.last-run.json';
if (!fs.existsSync(resultPath)) process.exit(0);

const result = JSON.parse(fs.readFileSync(resultPath, 'utf8'));
const failed = result.failedTests?.map((t: any) => t.location.file) ?? [];

if (failed.length === 0) {
  console.log('No failed tests');
  process.exit(0);
}

spawnSync('npx', ['playwright', 'test', ...failed], { stdio: 'inherit' });
```

---

## 12. Network & API interactions

### Waiting on a response

See §10.

### Mocking a response

```ts
await page.route('**/api/products', route =>
  route.fulfill({
    status: 200,
    contentType: 'application/json',
    body: JSON.stringify({ products: [] }),
  }),
);
```

Use mocking when:
- You need a deterministic edge-case payload (empty list, error states).
- The real API is unstable / unavailable.

Don't mock everything — the value of E2E is hitting real surfaces.

### Capturing and asserting a response

```ts
const [response] = await Promise.all([
  page.waitForResponse(r => r.url().includes('/graphql') && r.request().postData()?.includes('createProduct')),
  page.getByRole('button', { name: 'Save' }).click(),
]);

expect(response.status()).toBe(200);
const body = await response.json();
expect(body.data.createProduct.id).toBeDefined();
```

### Type the response body

```ts
interface ProductsResponse {
  data: { products: { nodes: { id: string; name: string }[] } };
}

const body: ProductsResponse = await response.json();
```

---

## 13. Test data

Centralise all generated data in a single file:

```ts
// src/fixtures/TestData/testData.ts
import { randEmail, randFirstName, randLastName, randText } from '@ngneat/falso';

export class TestData {
  firstName = randFirstName();
  lastName = randLastName();
  email = randEmail();
  description = randText();
}

export const testData = new TestData();
export const testDataForEdit = new TestData();
export const testDataForDetails = new TestData();
```

**Rules**
- Random values come from `@ngneat/falso`. Don't import `faker` or roll your own.
- Fixed business values (specific account name, real partner email) → named constant in the page object, not a buried string literal.
- Never commit production data. Use seeded shared test accounts.
- Binary fixtures (PDFs, images) live under `src/fixtures/TestData/`. Load with `path.join(__dirname, '../../fixtures/TestData/<file>')`.

---

## 14. Selecting dates

Use `dayjs` for date math:

```ts
import dayjs from 'dayjs';

export function milestoneDate(daysFromToday: number, format: string): string {
  return dayjs().add(daysFromToday, 'day').format(format);
}
```

Pick a date in a calendar widget (adapt selectors to your UI library):

```ts
export async function selectDate(page: Page, daysFromToday: number): Promise<void> {
  await page.getByTestId('datePicker').click();

  const year = milestoneDate(daysFromToday, 'YYYY');
  const month = milestoneDate(daysFromToday, 'MMM').toUpperCase();
  const day = milestoneDate(daysFromToday, 'D');

  await page.getByRole('button', { name: year }).click();
  await page.getByRole('button', { name: month }).click();
  await page.getByRole('gridcell', { name: day, exact: true }).click();
}
```

Range assertion in a table:

```ts
const rows = await page.getByTestId('dateCell').allTextContents();
const startDate = new Date(milestoneDate(0, 'YYYY-MM-DD'));
const endDate = new Date(milestoneDate(7, 'YYYY-MM-DD'));

for (const cell of rows) {
  const [day, month, year] = cell.split('-');
  const rowDate = new Date(+year, +month - 1, +day);
  expect(rowDate >= startDate && rowDate <= endDate).toBe(true);
}
```

---

## 15. Mailosaur — email testing

Install:

```bash
yarn add -D mailosaur
```

`playwright.env.json`:

```json
{
  "MAILOSAUR_SERVER_ID": "<server-id>",
  "MAILOSAUR_API_KEY": "<api-key>"
}
```

Helper:

```ts
// helpers/mailosaur.ts
import MailosaurClient from 'mailosaur';
import { expect } from '@playwright/test';

const client = new MailosaurClient(process.env['MAILOSAUR_API_KEY']!);
const serverId = process.env['MAILOSAUR_SERVER_ID']!;

export function generateEmail(prefix = 'test'): string {
  return `${prefix}-${Date.now()}@${serverId}.mailosaur.net`;
}

export async function waitForEmail(to: string, expectedSubject: string): Promise<{ link: string }> {
  const message = await client.messages.get(serverId, { sentTo: to });
  expect(message.subject).toBe(expectedSubject);
  const link = message.html?.links?.[0]?.href;
  if (!link) throw new Error('No link in email');
  return { link };
}
```

Usage:

```ts
test('user invitation flow', async ({ page }) => {
  const email = generateEmail('invitee');
  await invite(page, email);

  const { link } = await waitForEmail(email, 'You have been invited');
  await page.goto(link);
  await setPasswordAndLogin(page, 'NewPass123!');
});
```

---

## 16. Debugging

### Codegen — generate locators by clicking

```bash
npx playwright codegen <url>
```

Records actions to TypeScript with suggested locators. Use it to discover stable selectors quickly.

### UI mode — interactive runner with time-travel

```bash
npx playwright test --ui
```

Best DX for iterating on a single test.

### Headed mode

```bash
npx playwright test --headed
```

Watch the browser in real time.

### Debug a single test

```bash
npx playwright test tests/login.spec.ts --debug
```

Opens Playwright Inspector — step through actions, pause on breakpoints.

### Trace viewer

```bash
npx playwright show-trace path/to/trace.zip
```

Time-travel snapshots, network log, console, source. The first thing to check on a CI failure.

### VS Code extension

Install **"Playwright Test for VSCode"**. Run / debug individual tests from the gutter. Locator picker built in.

---

## 17. Reporting

Built-in:

```ts
reporter: [
  ['html', { outputFolder: './playwright-report', open: 'never' }],
  ['list'],
],
```

Open the report:

```bash
npx playwright show-report
```

### Custom reporter

For client-friendly artifacts (CSV, summary HTML), implement a custom reporter under `src/reporters/`. Pattern:

1. Aggregate per-test data in `onTestEnd` (push to an array).
2. Write files in `onEnd` (single fs flush).
3. **Sort rows by `(module, test)`** before writing — keeps the report diff-stable across nightly runs (otherwise rows shuffle by completion order under parallel workers).

---

## 18. CI — GitHub Actions

Two-trigger model (cron + manual). The workflow file must live on the **default branch** for `schedule:` to fire.

```yaml
# .github/workflows/playwright.yaml
name: Playwright E2E

on:
  schedule:
    - cron: '00 22 * * 0-4'    # Sun–Thu 22:00 UTC
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: e2e-nightly-${{ github.ref }}
  cancel-in-progress: false

jobs:
  test:
    name: Run Playwright Tests
    runs-on: [ self-hosted, do ]
    timeout-minutes: 60
    container:
      image: mcr.microsoft.com/playwright:v1.58.2-jammy
      options: --user root
    steps:
      - name: Checkout
        uses: actions/checkout@v6
        with:
          ref: qa/development        # tests come from this branch

      - name: Enable yarn
        run: corepack enable && corepack prepare yarn@4.12.0 --activate

      - name: Install dependencies
        run: yarn install --frozen-lockfile

      - name: Materialize playwright.env.json
        env:
          PLAYWRIGHT_ENV_JSON: ${{ secrets.PLAYWRIGHT_ENV_JSON }}
        run: |
          [ -z "$PLAYWRIGHT_ENV_JSON" ] && { echo "Missing PLAYWRIGHT_ENV_JSON" >&2; exit 1; }
          printf '%s' "$PLAYWRIGHT_ENV_JSON" > apps/<project>/playwright.env.json

      - name: Run tests
        working-directory: apps/<project>
        run: npx playwright test --project chromium
        env:
          CI: 'true'

      - name: Upload HTML report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report-${{ github.run_id }}
          path: apps/<project>/playwright-report
          retention-days: 30

      - name: Upload traces on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-traces-${{ github.run_id }}
          path: apps/<project>/test-results
          retention-days: 14
```

**Sharded variant** (for large suites — `matrix.containers` runs jobs in parallel across machines):

```yaml
strategy:
  fail-fast: false
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - run: npx playwright test --shard=${{ matrix.shard }}/4
```

**Caveats**
- `schedule:` triggers fire only from the default branch. If you need the run attributed to a different branch, use a tiny scheduler workflow on `main` that calls `gh workflow run` with `--ref`.
- The container image version must match the `@playwright/test` version (`playwright:v1.58.2-jammy` ↔ `@playwright/test@1.58.2`).
- Replace `<project>` with your suite's directory (e.g. `sbh-e2e`).

---

## 19. Linting

Use Playwright's official ESLint plugin:

```bash
yarn add -D eslint-plugin-playwright
```

```js
// eslint.config.mjs
import playwright from 'eslint-plugin-playwright';

export default [
  playwright.configs['flat/recommended'],
  // ... your other configs
];
```

The plugin catches:
- `expect().toBe(true)` instead of `toBeVisible()` (race-y).
- `waitForTimeout` usage.
- Conditional `test()` calls.
- Missing `await` on Playwright APIs.

Run lint in CI **before** tests — catches issues without paying for a full E2E run.

---

## 20. rsync deployment

Office QA server (intermediate hop → internal):

```bash
rsync -a --progress <folder> qa:/home/qa/int/ && \
  ssh qa "rsync -a /home/qa/int/<folder> qa:/home/ubuntu/<project>/"
```

Tech server (DigitalOcean):

```bash
rsync -a --progress <project-directory> root@<server-ip>:/root/<project>/
```

Access:

```bash
# Office
ssh qa
cd <project>

# Tech
ssh root@<server-ip> -v
```

**Never** rsync secrets. `playwright.env.json` belongs on the server already (or sourced from a secret manager), not pushed from a laptop.

---

## 21. Pull request convention

### Branch name

Must start with `qa/`. Examples:
- `qa/release/user-add`
- `qa/feature/billing-flow`
- `qa/fix/login-otp-race`

### PR title

```
[<ProjectShort> <Type> <Date if RUSH/URGENT>] - <Brief Message>
```

Examples:
- `[SBH NORMAL] - Add User E2E`
- `[SBH RUSH - 12-06-2026] - Fix login OTP race`
- `[OH URGENT - 25.05.2026] - Playwright workflow fixes`

Types: `NORMAL`, `RUSH`, `URGENT`.

### PR description must contain

1. Brief description of the PR.
2. Merge date — reviewer must complete review before this date.
3. Passed-test screenshots, if relevant.

### Size limit

**Max 20 files changed** per PR. Exception: migrations or codemod sweeps.

### Sample

```
Title: [SBH NORMAL] - Add User E2E

Description:
This PR contains:
 - User Add E2E flow under tests/Users/
 - New custom helper for login in helpers/utils.ts
 - Updated CONVENTIONS.md with helper usage

Merge Date: 15-06-2026
```

---

## 22. Quick reference

| Task | Command |
|---|---|
| Run all tests | `npx playwright test` |
| Run one project | `npx playwright test --project chromium` |
| Run one spec | `npx playwright test tests/login.spec.ts` |
| Run only failed | `npx playwright test --last-failed` |
| Filter by tag | `npx playwright test --grep @smoke` |
| Run in UI mode | `npx playwright test --ui` |
| Run headed | `npx playwright test --headed` |
| Debug one test | `npx playwright test path/to.spec.ts --debug` |
| Generate locators | `npx playwright codegen <url>` |
| Open last report | `npx playwright show-report` |
| Open a trace | `npx playwright show-trace path/to/trace.zip` |
| Update browsers | `npx playwright install` |
| Shard across N | `npx playwright test --shard=1/N` |

---

## References

- [Playwright Best Practices](https://playwright.dev/docs/best-practices)
- [Playwright Locators](https://playwright.dev/docs/locators)
- [Playwright Test Configuration](https://playwright.dev/docs/test-configuration)
- [Playwright Trace Viewer](https://playwright.dev/docs/trace-viewer)
- [Playwright ESLint Plugin](https://github.com/playwright-community/eslint-plugin-playwright)
