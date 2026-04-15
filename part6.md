


# PART 6 -- E2E DESKTOP IN DETAIL

<div class="chapter-intro">
Parts 2 and 3 gave you the overview — what the tools are and how the architecture fits together. Part 6 is where you become productive. This is a deep-dive masterclass that takes you from zero knowledge in Playwright, Electron, Speculos, and Allure to confidently writing, running, and debugging desktop E2E tests. Every concept is explained with real code from the Ledger Live codebase. By the end, you will have completed your first real Jira ticket.
</div>

---

## Playwright from Zero -- Core Concepts

<div class="chapter-intro">
Playwright is the foundation of everything in desktop E2E. If you have never used it before, this chapter teaches you every concept you need — from what a locator is to how auto-waiting works — using real examples from the Ledger Live codebase. After this chapter, you will be able to read any test file in the suite and understand every line.
</div>

### 43.1 What Is Playwright?

Playwright is an open-source browser automation framework created by Microsoft. It can control Chromium, Firefox, WebKit, and — critically for us — **Electron** applications. Unlike Selenium (which communicates over HTTP), Playwright talks to browsers via the Chrome DevTools Protocol (CDP), making it faster and more reliable.

**Key facts:**
- Language: TypeScript/JavaScript (also supports Python, Java, .NET)
- Created by: Microsoft (the same team that originally built Puppeteer at Google)
- First release: 2020
- Used for: E2E testing, web scraping, automation
- Killer feature: **Auto-waiting** — Playwright automatically waits for elements to be ready before interacting with them

**Why Playwright over alternatives?**

| Feature | Playwright | Cypress | Selenium |
|---------|-----------|---------|----------|
| Multi-browser | Chromium, Firefox, WebKit | Chromium only (limited Firefox) | All browsers |
| Electron support | Native | No | No |
| Auto-waiting | Built-in | Built-in | Manual |
| Parallel execution | Built-in | Via CI splitting | Via Grid |
| Speed | Fast (CDP) | Fast (in-process) | Slower (HTTP) |
| Language | TS/JS, Python, Java, .NET | JS/TS only | All major |

Ledger Live Desktop uses Playwright because it natively supports **Electron testing** — the ability to launch and control the desktop application as if a real user were clicking through it.

### 43.2 The Locator System

A **locator** is how you tell Playwright which element to interact with. Think of it as a CSS selector on steroids — but with built-in waiting and retry logic.

#### The Golden Rule: `getByTestId` first

The most reliable locator strategy is `data-testid` attributes. These are custom attributes added to the source code specifically for testing:

```html
<!-- In the React source code -->
<button data-testid="portfolio-empty-state-add-account-button">Add Account</button>
```

```typescript
// In the test
this.page.getByTestId("portfolio-empty-state-add-account-button");
```

This is the most stable because `data-testid` attributes are never changed by designers or translators. They exist solely for tests.

**Real example from `portfolio.page.ts`:**
```typescript
private addAccountButton = this.page.getByTestId("portfolio-empty-state-add-account-button");
private buySellEntryButton = this.page.getByTestId("buy-sell-entry-button");
private chart = this.page.getByTestId("chart-container");
private totalBalance = this.page.getByTestId("total-balance");
```

#### `getByRole` — Accessible locators

`getByRole` finds elements by their ARIA role, which is what screen readers see. It is the second most reliable strategy:

```typescript
// From add.account.modal.ts — find an option by accessible name
private selectAccountInList = (Currency: Currency) =>
  this.page.getByRole("option", {
    name: `${Currency.name} (${Currency.ticker})`,
    exact: true,
  });
```

The `exact: true` option means the name must match exactly (no partial matches).

**Common roles:** `button`, `link`, `textbox`, `option`, `heading`, `checkbox`, `radio`, `dialog`.

#### `getByText` — Text content locators

When elements have no `data-testid` and no specific ARIA role, you can find them by their visible text:

```typescript
// From portfolio.page.ts
private assetAllocationTitle = this.page.getByText("Asset allocation");
private showAllButton = this.page.getByText("Show all");
private showMoreButton = this.page.getByText("Show more");
```

**Warning:** Text locators break when translations change or text is dynamic. Use `getByTestId` whenever possible.

#### `page.locator()` — CSS and XPath selectors

For complex cases, you can use raw CSS selectors or XPath expressions:

```typescript
// CSS selector — from portfolio.page.ts
private operationList = this.page.locator("#operation-list");
private assetRowElements = this.page.locator("[data-testid^='asset-row-']");
private operationRows = this.page.locator("[data-testid^='operation-row-']");
```

The `^=` means "starts with" — so `[data-testid^='asset-row-']` matches any element whose `data-testid` starts with `asset-row-`.

**XPath — from `abstractClasses.ts`:**
```typescript
// XPath: find any element containing specific text
protected optionWithText = (text: string) =>
  this.page.locator(`//*[contains(text(),"${text}")]`).first();

// XPath with following-sibling: find text then look for nearby text
protected optionWithTextAndFollowingText = (text: string, followingText: string) =>
  this.page.locator(
    `//*[contains(text(),"${text}")]/following::span[contains(text(),"${followingText}")]`,
  ).first();
```

XPath is powerful but fragile — it is used in the codebase only when simpler locators cannot work (typically for dropdown options that have no `data-testid`).

#### Dynamic locators (factory functions)

Sometimes you need a locator that depends on a parameter. The codebase uses arrow functions for this:

```typescript
// From portfolio.page.ts — locator factory for any asset row
private assetRow = (asset: string) =>
  this.page.getByTestId(`asset-row-${asset.toLowerCase()}`);

// From layout.component.ts — locator factory for topbar buttons
private readonly topbarActionButton = (action: string) =>
  this.page.getByTestId(`topbar-action-button-${action}`);
```

Usage:
```typescript
await this.assetRow("bitcoin").click();          // -> getByTestId("asset-row-bitcoin")
await this.topbarActionButton("settings").click(); // -> getByTestId("topbar-action-button-settings")
```

#### The `.or()` combinator

When the same element has different `data-testid` values in different versions of the UI:

```typescript
// From layout.component.ts — handles both old and new testId
readonly topbarSettingsButton = this.topbarActionButton("settings").or(
  this.page.getByTestId("topbar-settings-button"),
);
```

This means: try `topbar-action-button-settings` first, OR fall back to `topbar-settings-button`. This pattern handles UI transitions between versions.

#### Locator Priority Summary

| Priority | Strategy | When to use | Stability |
|----------|----------|-------------|-----------|
| 1 | `getByTestId` | Always when a `data-testid` exists | Highest |
| 2 | `getByRole` | Accessible elements (buttons, links, options) | High |
| 3 | `getByText` | Static, non-translatable text | Medium |
| 4 | `locator` (CSS) | Complex structural selectors | Medium |
| 5 | `locator` (XPath) | Last resort — sibling/ancestor traversal | Lowest |

### 43.3 Actions

Once you have a locator, you perform actions on it. Here are every action used in the codebase:

#### `click()` — The most common

```typescript
await this.addAccountButton.click();
await this.showAllButton.click();
await this.checkbox.click({ force: true }); // force: bypass visibility checks
```

The `{ force: true }` option skips Playwright's actionability checks. Used when elements are technically visible but covered by overlays.

#### `fill()` — Type into an input

```typescript
// From add.account.modal.ts
await this.selectAccountInput.fill(currency.name);
```

`fill()` clears the input first, then types the text. Unlike `type()`, it does not simulate individual keystrokes — it sets the value directly.

#### `press()` — Press a keyboard key

```typescript
await this.selectAccountInput.press("Enter");
```

Common keys: `"Enter"`, `"Escape"`, `"Tab"`, `"ArrowDown"`, `"Backspace"`.

#### `hover()` — Move mouse over element

```typescript
await this.page.mouse.move(0, 0); // Move mouse to top-left corner (dismiss tooltips)
```

#### `waitFor()` — Wait for element state

```typescript
// Wait for element to become visible
await this.addAccountsButton.waitFor({ state: "visible" });

// Wait for element to disappear
await this.stopButton.waitFor({ state: "hidden" });
await this.loadingSpinner.waitFor({ state: "hidden" });
```

States: `"visible"`, `"hidden"`, `"attached"`, `"detached"`.

#### `scrollIntoViewIfNeeded()` — Scroll to element

```typescript
// From portfolio.page.ts
await this.operationList.scrollIntoViewIfNeeded();
```

#### `evaluate()` — Run JavaScript in the browser

```typescript
// From portfolio.page.ts — wait for DOM element count to change
await this.page.waitForFunction(() => {
  return document.querySelectorAll("[data-testid^='asset-row-']").length > 6;
});

// From common.ts — save logs
await page.evaluate(filePath => {
  window.saveLogs(filePath);
}, filePath);
```

`evaluate()` runs the given function inside the browser context. The second argument is passed as a parameter. This is the escape hatch for anything Playwright's API cannot do directly.

#### `isVisible()` — Check without asserting

```typescript
if (await this.deselectAllButton.isVisible()) {
  await this.deselectAllButton.click();
}
```

Unlike `expect().toBeVisible()`, `isVisible()` returns a boolean and does not throw on failure. Used for conditional logic.

### 43.4 Assertions

Assertions verify that the application is in the expected state. Playwright's `expect` function comes with built-in auto-retry — it keeps checking until the assertion passes or times out.

#### `toBeVisible()` — Element is on screen

```typescript
import { expect } from "@playwright/test";

await expect(this.addAccountButton).toBeVisible();
await expect(this.chart).toBeVisible();
```

This is the most used assertion in the codebase. It waits up to the configured timeout (41 seconds in our config).

#### `not.toBeVisible()` — Element is NOT on screen

```typescript
await expect(this.portfolioTotalBalance).not.toBeVisible();
await expect(this.portfolioTrend).not.toBeVisible();
```

#### `toContainText()` — Element contains text

```typescript
await expect(this.totalBalance).toContainText("$");
await expect(this.toaster).toContainText("Transaction sent !");
await expect(this.assetRowValue("bitcoin")).toContainText("0.001");
```

`toContainText` is a substring match. It passes if the text appears anywhere in the element.

#### `toHaveText()` — Exact text match

```typescript
expect(await this.title.textContent()).toBe("Add accounts");
```

Note: `toBe()` is a Jest-style assertion (string equality), while `toHaveText()` is Playwright's locator assertion with auto-retry. Prefer `toHaveText()` for locators.

#### `toHaveCount()` — Number of matching elements

```typescript
await expect(this.assetRowElements).toHaveCount(6);
```

Useful for verifying lists: "I expect exactly 6 asset rows."

#### `toBeEnabled()` / `toBeDisabled()` — Button state

```typescript
await expect(this.quickActionButton("sell")).toBeDisabled();
await expect(this.quickActionButton("send")).toBeEnabled();
```

#### `toHaveAttribute()` — Check HTML attributes

```typescript
await expect(this.drawerBuycryptoButton).toHaveAttribute("data-active", "true");
```

#### `toBeGreaterThan()` — Numeric comparison

```typescript
const numberOfOperationsAfter = await this.operationRows.count();
expect(numberOfOperationsAfter).toBeGreaterThan(numberOfOperationsBefore);
```

Note: this is a plain value assertion (no auto-retry). The `count()` already resolved the value.

### 43.5 Auto-Waiting — Playwright's Killer Feature

This is what makes Playwright different from Selenium. When you call:

```typescript
await this.addAccountButton.click();
```

Playwright does NOT just click immediately. Behind the scenes, it:

1. **Waits** for the element to be attached to the DOM
2. **Waits** for the element to be visible (not `display: none`, not zero-size)
3. **Waits** for the element to be stable (not animating)
4. **Waits** for the element to receive pointer events (not covered by another element)
5. **Waits** for the element to be enabled (not `disabled`)
6. **Clicks** the element

If any step fails, Playwright retries from the beginning. This continues until the `timeout` is reached (41 seconds in our config, 120 seconds for the default page timeout).

**This means you almost never need `sleep()` or `waitForTimeout()`.** Instead of:

```typescript
// BAD — arbitrary wait
await page.waitForTimeout(3000);
await button.click();
```

You write:
```typescript
// GOOD — auto-wait handles everything
await button.click();
```

The same auto-retry applies to assertions:

```typescript
// This will retry for up to 41 seconds until the text appears
await expect(this.totalBalance).toContainText("$1,234.56");
```

**When you DO need explicit waits:**

```typescript
// Wait for a specific element state change
await this.loadingSpinner.waitFor({ state: "hidden" });

// Wait for the page to finish loading
await page.waitForLoadState("domcontentloaded");

// Wait for a specific selector to appear/disappear
await page.waitForSelector("#loader-container", { state: "hidden" });

// Wait for a DOM condition (escape hatch)
await page.waitForFunction(() => {
  return document.querySelectorAll("[data-testid^='asset-row-']").length > 6;
});
```

### 43.6 Configuration — `playwright.config.ts` Line by Line

Open `e2e/desktop/playwright.config.ts`. Here is every property explained:

```typescript
const config: PlaywrightTestConfig = {
  // Where to find test files
  testDir: "./tests/specs",

  // Retry failed tests: 2 times in CI, never locally
  retries: process.env.CI ? 2 : 0,

  // Max time per test: 400s in CI, 1200s (20min) locally
  // Local is higher because your machine may be slower
  timeout: process.env.CI ? 400000 : 1200000,

  // Where to save screenshots, videos, traces
  outputDir: "./tests/artifacts/test-results",

  // Max time for each expect() assertion: 41 seconds
  expect: { timeout: 41000 },

  // No global timeout (tests can run as long as needed)
  globalTimeout: 0,

  // Run this file once before ALL tests
  globalSetup: require.resolve("./tests/utils/global.setup"),

  // Run this file once after ALL tests
  globalTeardown: require.resolve("./tests/utils/global.teardown"),

  use: {
    // Accept self-signed certificates
    ignoreHTTPSErrors: true,
    // We handle screenshots manually in captureArtifacts()
    screenshot: "off",
  },

  // In CI, fail if someone left test.only() in code
  forbidOnly: !!process.env.CI,

  // In CI, only save artifacts for failed tests
  preserveOutput: process.env.CI ? "failures-only" : "always",

  // In CI, stop after 5 failures (don't waste time on cascading failures)
  maxFailures: process.env.CI ? 5 : undefined,

  // Report tests that take longer than 60 seconds
  reportSlowTests: process.env.CI ? { max: 0, threshold: 60000 } : null,

  // Run all tests in parallel (not sequentially)
  fullyParallel: true,

  // Use all available CPU cores
  workers: "100%",

  // Reporters: what format to output results in
  reporter: process.env.CI
    ? [
        ["list"],                          // Console output
        ["allure-playwright", { ... }],    // Allure HTML report
        ["./tests/utils/customJsonReporter.ts"],  // Xray JSON for Jira
      ]
    : [["allure-playwright"]],  // Local: Allure only
};
```

### 43.7 Running Tests from the CLI

```bash
# Run ALL tests
pnpm e2e:desktop test:playwright

# Run a specific file
pnpm e2e:desktop test:playwright add.account.spec.ts

# Run tests matching a name pattern (grep)
pnpm e2e:desktop test:playwright -- --grep "Add account"

# Run tests with a specific tag
pnpm e2e:desktop test:playwright -- --grep "@smoke"
pnpm e2e:desktop test:playwright -- --grep "@bitcoin"

# Run with the Playwright UI (interactive debugger)
pnpm e2e:desktop test:playwright -- --ui

# Run in debug mode (opens inspector, pauses on first action)
PWDEBUG=1 pnpm e2e:desktop test:playwright add.account.spec.ts

# Run with a single worker (no parallelism, easier to debug)
pnpm e2e:desktop test:playwright -- --workers=1

# Run a specific shard (for CI splitting)
pnpm e2e:desktop test:playwright -- --shard=1/3
```

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://playwright.dev/docs/locators">Playwright: Locators</a> — official locator guide with interactive examples</li>
<li><a href="https://playwright.dev/docs/actionability">Playwright: Auto-waiting</a> — the actionability checks explained</li>
<li><a href="https://playwright.dev/docs/test-assertions">Playwright: Assertions</a> — full assertion reference</li>
<li><a href="https://playwright.dev/docs/test-configuration">Playwright: Configuration</a> — all config options</li>
<li><a href="https://playwright.dev/docs/running-tests">Playwright: Running Tests</a> — CLI options</li>
<li><a href="https://playwright.dev/docs/test-cli">Playwright: Command Line</a> — complete CLI reference</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Playwright gives you locators that find elements, actions that interact with them, and assertions that verify state — all with built-in auto-waiting. You never write <code>sleep()</code>. The configuration in <code>playwright.config.ts</code> controls timeouts, retries, parallelism, and reporting. Master these fundamentals and you can read any test in the codebase.
</div>

### 43.8 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Playwright Core Concepts</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="A">
<p><strong>Q1.</strong> Which locator strategy should you prefer when a <code>data-testid</code> attribute exists on the element?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>getByTestId()</code> — it is the most stable because test IDs are never changed by designers or translators</button>
<button class="quiz-choice" data-value="B">B) <code>getByText()</code> — text is always visible and easy to understand</button>
<button class="quiz-choice" data-value="C">C) <code>locator()</code> with CSS — CSS selectors are more powerful</button>
<button class="quiz-choice" data-value="D">D) <code>getByRole()</code> — ARIA roles are the web standard</button>
</div>
<p class="quiz-explanation"><code>getByTestId()</code> is the #1 priority in the locator strategy. Test IDs exist solely for testing and are never affected by design changes, translations, or accessibility refactors.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> What does Playwright do automatically BEFORE performing a <code>click()</code> action?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Takes a screenshot of the element</button>
<button class="quiz-choice" data-value="B">B) Scrolls to the top of the page</button>
<button class="quiz-choice" data-value="C">C) Waits for the element to be attached, visible, stable, able to receive events, and enabled</button>
<button class="quiz-choice" data-value="D">D) Refreshes the page to ensure latest DOM</button>
</div>
<p class="quiz-explanation">Playwright's auto-waiting performs 5 checks before every action: attached to DOM, visible, stable (not animating), receives pointer events (not covered), and enabled. This is the "killer feature" that eliminates most <code>sleep()</code> calls.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> In our <code>playwright.config.ts</code>, what is the <code>expect.timeout</code> set to and what does it control?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) 120 seconds — how long each test can run</button>
<button class="quiz-choice" data-value="B">B) 400 seconds — the CI test timeout</button>
<button class="quiz-choice" data-value="C">C) 5 seconds — how long Playwright waits before failing</button>
<button class="quiz-choice" data-value="D">D) 41 seconds — how long each <code>expect()</code> assertion retries before failing</button>
</div>
<p class="quiz-explanation"><code>expect.timeout: 41000</code> means every <code>expect(locator).toBeVisible()</code> (and similar assertions) will retry for up to 41 seconds. This is separate from the overall test timeout (400s CI / 1200s local).</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q4.</strong> What is the difference between <code>fill()</code> and <code>press()</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>fill()</code> is for buttons, <code>press()</code> is for inputs</button>
<button class="quiz-choice" data-value="B">B) <code>fill()</code> clears the input and sets a text value directly, while <code>press()</code> simulates pressing a single keyboard key</button>
<button class="quiz-choice" data-value="C">C) They are the same, just different naming conventions</button>
<button class="quiz-choice" data-value="D">D) <code>fill()</code> is synchronous, <code>press()</code> is asynchronous</button>
</div>
<p class="quiz-explanation"><code>fill("Bitcoin")</code> clears the input field and types "Bitcoin" as a whole string. <code>press("Enter")</code> simulates pressing a single key. They serve different purposes and are often used together: <code>fill()</code> to type text, then <code>press("Enter")</code> to submit.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> When should you use <code>page.waitForFunction()</code> instead of regular Playwright locators?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Always — it is more reliable than locators</button>
<button class="quiz-choice" data-value="B">B) When you want to add a sleep delay</button>
<button class="quiz-choice" data-value="C">C) When you need to check a DOM condition that cannot be expressed as a locator assertion (e.g., counting elements after a dynamic change)</button>
<button class="quiz-choice" data-value="D">D) When elements have no <code>data-testid</code></button>
</div>
<p class="quiz-explanation"><code>waitForFunction()</code> is the escape hatch for conditions that Playwright's locator API cannot express. For example, waiting for the count of dynamically-added elements to exceed a threshold, as seen in <code>portfolio.page.ts</code>.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Playwright Advanced -- Fixtures, POM & TypeScript Decorators

<div class="chapter-intro">
Chapter 43 taught you how Playwright finds elements and interacts with them. This chapter goes deeper into the three patterns that make the Ledger E2E suite maintainable at scale: <strong>fixtures</strong> (dependency injection for test setup), the <strong>Page Object Model</strong> (separation of test logic from UI details), and <strong>TypeScript decorators</strong> (the <code>@step</code> annotation for reporting). These are the patterns you will use in every test you write.
</div>

### 44.1 What Are Fixtures?

In Playwright, a **fixture** is a reusable piece of setup that is automatically created before your test and cleaned up after. Think of it as dependency injection for tests.

Without fixtures, you would write setup code in every test:

```typescript
// BAD — setup code repeated in every test
test("test 1", async () => {
  const app = await launchElectron();
  const page = await app.firstWindow();
  // ... test code ...
  await app.close();
});

test("test 2", async () => {
  const app = await launchElectron();  // repeated!
  const page = await app.firstWindow();  // repeated!
  // ... test code ...
  await app.close();  // repeated!
});
```

With fixtures, setup is declared once and injected:

```typescript
// GOOD — fixtures handle setup/teardown automatically
test("test 1", async ({ app, page }) => {
  // app and page are ready to use!
  // cleanup happens automatically after the test
});

test("test 2", async ({ app, page }) => {
  // fresh app and page for each test
});
```

The magic: you just list what you need in the destructured parameter (`{ app, page }`), and Playwright creates it for you. When the test ends, Playwright tears it down.

### 44.2 `base.extend<T>()` — Creating Custom Fixtures

Playwright provides built-in fixtures (`page`, `context`, `browser`). To add your own, you use `base.extend<T>()`:

```typescript
import { test as base } from "@playwright/test";

// Define the type of your custom fixtures
type MyFixtures = {
  greeting: string;
  counter: number;
};

// Extend the base test with your fixtures
const test = base.extend<MyFixtures>({
  greeting: "hello",           // Simple value fixture
  counter: async ({}, use) => {
    // Setup
    const value = await fetchCounterFromAPI();
    // Provide the value to the test
    await use(value);
    // Teardown (runs after the test)
    await resetCounter();
  },
});
```

**The `use()` callback pattern**: The `use` function is the key concept. Everything before `use()` is setup, the argument to `use()` is what the test receives, and everything after `use()` is teardown.

```typescript
myFixture: async ({}, use) => {
  // SETUP: runs before the test
  const resource = await createResource();

  await use(resource);  // TEST RUNS HERE with this value

  // TEARDOWN: runs after the test (even if test fails)
  await resource.cleanup();
},
```

### 44.3 The Ledger Live `TestFixtures` Type

Open `e2e/desktop/tests/fixtures/common.ts`. Here is the `TestFixtures` type with every field explained:

```typescript
type TestFixtures = {
  // ── UI Configuration ──
  lang: string;                  // App language (default: "en-US")
  theme: "light" | "dark" | "no-preference" | undefined;  // Color scheme (default: "dark")
  env: Record<string, string>;   // Extra environment variables

  // ── Test Data ──
  userdata?: string;             // Name of JSON fixture (e.g., "skip-onboarding")
  extraUserdataFiles?: Record<string, string>;  // Additional files to create
  settings: Record<string, unknown>;   // App settings merged into app.json
  userdataDestinationPath: string;     // Generated unique dir per test
  userdataOriginalFile?: string;       // Full path to source JSON
  userdataFile: string;                // Full path to app.json

  // ── Feature Flags ──
  featureFlags: OptionalFeatureMap;    // Override feature flags for this test

  // ── Device Simulation ──
  speculosApp: AppInfos;              // Which Speculos app to launch (e.g., "Bitcoin")
  speculos: SpeculosFixtureHandle;    // Device handle with current + relaunch
  simulateCamera: string;             // Fake video capture file path

  // ── Injected Objects ──
  electronApp: ElectronApplication;   // The launched Electron app
  page: Page;                         // The app's main window
  app: Application;                   // The POM hub (all page objects)

  // ── CLI & Data Population ──
  cliCommands?: CliCommand[];         // Commands to run after Speculos starts
  cliCommandsOnApp?: { app: AppInfos; cmd: CliCommand }[];  // Commands with specific apps

  // ── Reporting ──
  teamOwner?: Team;                   // Allure team ownership
  localManifestOverride?: LiveAppManifest[];  // Override Live App manifests
};
```

**The fixture dependency graph:**

```
speculosApp (input)
    ↓
speculos (launches Docker container)
    ↓
electronApp (launches Electron with SPECULOS_API_PORT)
    ↓
page (gets first window from Electron)
    ↓
app (wraps page in Application POM hub)
```

Each fixture depends on the one above it. Playwright resolves these dependencies automatically — you never have to worry about the order.

### 44.4 Overriding Fixtures with `test.use()`

Tests configure their fixtures using `test.use()` inside a `describe` block:

```typescript
test.describe("Add Accounts", () => {
  test.use({
    teamOwner: Team.WALLET_XP,                       // For Allure reports
    userdata: "skip-onboarding-with-last-seen-device", // Pre-configured state
    speculosApp: Currency.BTC.speculosApp,             // Launch BTC device
    featureFlags: {                                    // Override flags
      currencyAleo: { enabled: true },
    },
  });

  test("Add BTC account", async ({ app }) => {
    // app is ready with BTC Speculos running
  });
});
```

**Rules:**
- `test.use()` applies to ALL tests in the same `describe` block
- Values in `test.use()` override defaults from the fixture definition
- You can nest `describe` blocks with different `test.use()` for different configurations

### 44.5 Page Object Model (POM) — The Architecture

The **Page Object Model** is a design pattern where each screen (or major UI component) of the application gets its own class. This class encapsulates:
1. **Locators** — how to find elements on that screen
2. **Actions** — what a user can do on that screen
3. **Assertions** — what you can verify on that screen

**Why POM?**
- If a `data-testid` changes, you update ONE file (the page object), not 50 tests
- Tests read like user stories: `app.portfolio.clickAddAccountButton()`
- Page objects can be reused across many tests

**The Ledger implementation:**

```
PageHolder (base — holds the Page reference)
    ↓
Component (adds shared UI: spinner, toaster, dropdown options)
    ↓
AppPage (abstract — all concrete pages extend this)
    ↓
PortfolioPage, AccountPage, SettingsPage, ... (concrete pages)
```

And the **Application class** (hub) aggregates all page objects:

```typescript
// From e2e/desktop/tests/page/index.ts
export class Application extends PageHolder {
  public account = new AccountPage(this.page);
  public accounts = new AccountsPage(this.page);
  public portfolio = new PortfolioPage(this.page);
  public addAccount = new AddAccountModal(this.page);
  public send = new SendModal(this.page);
  public receive = new ReceiveModal(this.page);
  public speculos = new SpeculosPage(this.page);
  public mainNavigation = new MainNavigationPage(this.page);
  // ... 30+ more page objects
}
```

In tests, you access everything through `app`:

```typescript
test("navigate and check", async ({ app }) => {
  await app.portfolio.clickAddAccountButton();
  await app.addAccount.selectCurrency(Currency.BTC);
  await app.mainNavigation.openTargetFromMainNavigation("accounts");
  await app.accounts.navigateToAccountByName("Bitcoin 1");
});
```

**Rule: Tests never use locators directly.** Every interaction goes through a named method on a page object. This is what makes the suite maintainable.

### 44.6 TypeScript Decorators — The `@step()` Pattern

If you come from TypeScript without decorator experience, the `@` syntax may look mysterious. Here is what it means.

**What is a decorator?** A decorator is a function that wraps another function, adding behavior before or after it runs. The `@` symbol is syntax sugar for applying that wrapper.

```typescript
// Without decorator:
async clickButton() {
  await test.step("Click button", async () => {
    await this.button.click();
  });
}

// With decorator — exactly the same behavior, cleaner syntax:
@step("Click button")
async clickButton() {
  await this.button.click();
}
```

**How `step.ts` works (the source code):**

```typescript
// From e2e/desktop/tests/misc/reporters/step.ts
export function step<This extends HasConstructor, Args extends unknown[], Return>(
  message?: string,
) {
  return function actualDecorator(
    target: (this: This, ...args: Args) => Promise<Return>,
    context: ClassMethodDecoratorContext,
  ) {
    async function replacementMethod(this: This, ...args: Args): Promise<Return> {
      const ctorName = this.constructor.name;  // e.g., "PortfolioPage"

      // Build the step name: custom message or "ClassName.methodName"
      const name = message
        ? message.replace(/\$(\d+)/g, (_, idx) => String(args[Number(idx)]))
        : `${ctorName}.${String(context.name)}`;

      // Wrap the method call in a Playwright test step
      return await test.step(name, () => target.call(this, ...args), { box: true });
    }
    return replacementMethod;
  };
}
```

**Key details:**

1. **`$N` placeholders** — The message can include `$0`, `$1`, etc. which get replaced with the function arguments:
   ```typescript
   @step("Click on asset row $0")
   async clickOnSelectedAssetRow(asset: string) { ... }
   // When called with "bitcoin", step name becomes: "Click on asset row bitcoin"
   ```

2. **`{ box: true }`** — This tells Playwright to "box" the step, meaning the step's internal details are collapsed in the report. The error points to the test line that called the method, not the internals.

3. **Fallback for non-test contexts** — If `@step` is called outside a test (e.g., during fixture setup), it catches the error and runs the method without wrapping.

4. **Auto-naming** — If no message is provided, the step name is `ClassName.methodName` (e.g., `PortfolioPage.clickAddAccountButton`).

### 44.7 Tags and Annotations

Tags and annotations are metadata attached to tests for filtering and reporting.

**Tags — for filtering which tests run:**

```typescript
test("Add BTC account", {
  tag: [
    "@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",  // Device compatibility
    "@bitcoin",                                                     // Currency
    "@family-bitcoin",                                              // Blockchain family
    "@smoke",                                                       // Test importance
  ],
}, async ({ app }) => { ... });
```

Run only tests with a specific tag:
```bash
pnpm e2e:desktop test:playwright -- --grep "@smoke"
pnpm e2e:desktop test:playwright -- --grep "@bitcoin"
pnpm e2e:desktop test:playwright -- --grep "@NanoSP"
```

**Annotations — for linking to external systems:**

```typescript
test("Add BTC account", {
  annotation: {
    type: "TMS",                                    // Test Management System
    description: "B2CQA-2499, B2CQA-2644",         // Jira/Xray ticket IDs
  },
}, async ({ app }) => {
  // Inside the test, link to Allure
  await addTmsLink(getDescription(test.info().annotations, "TMS").split(", "));
});
```

The annotation links the test to Xray test cases in Jira. The `addTmsLink()` call creates clickable links in the Allure report.

### 44.8 Parallel Execution and Workers

```typescript
// In playwright.config.ts
fullyParallel: true,  // All tests can run at the same time
workers: "100%",      // Use all CPU cores
```

Each worker gets its own:
- Speculos Docker container (on a random port)
- Electron app instance
- Userdata directory (unique UUID)
- Isolated test state

This is why the fixture system assigns random ports to Speculos — to prevent port conflicts between parallel workers.

**In CI**, the suite is also **sharded** across multiple runners:
```bash
# Runner 1:
pnpm e2e:desktop test:playwright -- --shard=1/3
# Runner 2:
pnpm e2e:desktop test:playwright -- --shard=2/3
# Runner 3:
pnpm e2e:desktop test:playwright -- --shard=3/3
```

### 44.9 Retry Strategy

```typescript
retries: process.env.CI ? 2 : 0,
```

- **Locally**: No retries. You see failures immediately and fix them.
- **CI**: Up to 2 retries. A test must fail 3 times to be reported as failed.

The `isLastRetry()` utility checks if the current run is the last attempt. Expensive operations like video capture only happen on the last retry:

```typescript
recordVideo: isLastRetry(testInfo),  // Only record video on the final attempt
```

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://playwright.dev/docs/test-fixtures">Playwright: Test Fixtures</a> — the official fixture guide</li>
<li><a href="https://playwright.dev/docs/pom">Playwright: Page Object Models</a> — official POM guide</li>
<li><a href="https://playwright.dev/docs/test-parameterize">Playwright: Parameterize Tests</a> — data-driven testing patterns</li>
<li><a href="https://playwright.dev/docs/test-parallel">Playwright: Parallelism</a> — workers and sharding</li>
<li><a href="https://playwright.dev/docs/test-retries">Playwright: Retries</a> — retry configuration</li>
<li><a href="https://www.typescriptlang.org/docs/handbook/decorators.html">TypeScript: Decorators</a> — the language feature behind <code>@step</code></li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Fixtures are dependency injection for tests — declare what you need, Playwright creates it. The Page Object Model separates locators from test logic so changes propagate from a single place. The <code>@step</code> decorator wraps methods in Allure reporting steps with zero effort. Together, these patterns make a 26-file test suite manageable and a 100+ page object codebase navigable.
</div>

### 44.10 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Fixtures, POM & Decorators</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> In the <code>use()</code> callback pattern, what runs AFTER the test finishes?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Everything before <code>use()</code></button>
<button class="quiz-choice" data-value="B">B) The argument passed to <code>use()</code></button>
<button class="quiz-choice" data-value="C">C) Everything after <code>use()</code> — this is the teardown phase</button>
<button class="quiz-choice" data-value="D">D) Nothing — teardown must be done manually</button>
</div>
<p class="quiz-explanation">The <code>use()</code> callback splits the fixture into three phases: before <code>use()</code> = setup, the argument to <code>use()</code> = what the test gets, after <code>use()</code> = teardown. The teardown runs even if the test fails.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q2.</strong> Why does the Ledger E2E suite import <code>test</code> from <code>"tests/fixtures/common"</code> instead of <code>"@playwright/test"</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The custom <code>test</code> extends Playwright's base with fixtures for Speculos, Electron, feature flags, page objects, and userdata</button>
<button class="quiz-choice" data-value="B">B) It is a naming convention with no functional difference</button>
<button class="quiz-choice" data-value="C">C) The standard Playwright import does not work with TypeScript</button>
<button class="quiz-choice" data-value="D">D) To avoid a dependency on the Playwright package</button>
</div>
<p class="quiz-explanation">The custom <code>test</code> from <code>common.ts</code> uses <code>base.extend&lt;TestFixtures&gt;()</code> to add 20+ custom fixtures (app, electronApp, speculos, featureFlags, userdata, etc.). Using the standard import would miss all of these.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> What does <code>@step("Select currency $0")</code> produce when called with <code>selectCurrency("Bitcoin")</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) An error — <code>$0</code> is not valid syntax</button>
<button class="quiz-choice" data-value="B">B) A step named "Select currency $0" (literal)</button>
<button class="quiz-choice" data-value="C">C) A step named "AddAccountModal.selectCurrency"</button>
<button class="quiz-choice" data-value="D">D) A step named "Select currency Bitcoin" — <code>$0</code> is replaced with the first argument</button>
</div>
<p class="quiz-explanation">The <code>@step</code> decorator's implementation uses <code>message.replace(/\$(\d+)/g, (_, idx) => String(args[Number(idx)]))</code> to replace <code>$N</code> placeholders with the corresponding function arguments. <code>$0</code> becomes the first argument.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q4.</strong> In the fixture dependency graph, which fixture must be created BEFORE <code>electronApp</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>page</code> — the window must exist before the app</button>
<button class="quiz-choice" data-value="B">B) <code>speculos</code> — the Electron app needs the Speculos port number to connect to the device</button>
<button class="quiz-choice" data-value="C">C) <code>app</code> — the Application must be ready first</button>
<button class="quiz-choice" data-value="D">D) No dependencies — <code>electronApp</code> is created independently</button>
</div>
<p class="quiz-explanation">The <code>electronApp</code> fixture sets <code>SPECULOS_API_PORT</code> as an environment variable from <code>speculos.current.port</code>. Without the Speculos container running first, the app would not know how to reach the emulated device.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> Why should tests NEVER use locators directly (e.g., <code>page.getByTestId("button").click()</code>) and instead use page object methods?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It is slower to use locators directly</button>
<button class="quiz-choice" data-value="B">B) Playwright does not support direct locator usage in tests</button>
<button class="quiz-choice" data-value="C">C) If a <code>data-testid</code> changes, you update one page object method instead of every test that uses it</button>
<button class="quiz-choice" data-value="D">D) Direct locators cannot be used with <code>expect()</code></button>
</div>
<p class="quiz-explanation">The POM pattern's main benefit is maintainability. When a locator changes (e.g., a <code>data-testid</code> is renamed), you update the page object once. All tests that use that method automatically get the fix. Without POM, you would update every test file.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Electron -- Desktop App Testing

<div class="chapter-intro">
Ledger Live Desktop is an <strong>Electron</strong> application — a web app packaged as a native desktop program. This chapter explains what Electron is, how Playwright launches and controls it, and how the E2E fixture system bridges the two. Understanding this layer is essential because Electron introduces concepts (main process, renderer, webviews, environment variables) that do not exist in regular web testing.
</div>

### 45.1 What Is Electron?

Electron is a framework that lets you build desktop applications using web technologies (HTML, CSS, JavaScript). It bundles:

- **Chromium** — the same engine that powers Chrome (renders the UI)
- **Node.js** — server-side JavaScript (for file system access, USB communication, etc.)

The result: a cross-platform desktop app (macOS, Windows, Linux) built with web technologies. Slack, VS Code, Discord, and Ledger Live are all Electron apps.

**Two processes:**
- **Main process** — Node.js. Handles native things: file system, USB, system tray, window management
- **Renderer process** — Chromium. Displays the UI. This is what Playwright controls

When Playwright launches an Electron app, it gets access to the renderer process (the `Page` object) and can also interact with the main process via `electronApp.evaluate()`.

### 45.2 How Playwright Launches the Electron App

The `electronUtils.ts` file contains the `launchApp()` function:

```typescript
import { _electron as electron } from "@playwright/test";

export async function launchApp({ env, lang, theme, userdataDestinationPath, ... }) {
  return await electron.launch({
    args: [
      // The compiled app entry point
      `${path.join(__dirname, "../../../../apps/ledger-live-desktop/.webpack/main.bundle.js")}`,
      // Isolated userdata directory (unique per test)
      `--user-data-dir=${userdataDestinationPath}`,
      // Display consistency
      "--force-device-scale-factor=1",
      // Prevent shared memory issues in Docker
      "--disable-dev-shm-usage",
      // Required for CI environments
      "--no-sandbox",
      // Enable Chromium logging
      "--enable-logging",
    ],
    env,                              // Environment variables (FEATURE_FLAGS, SPECULOS_API_PORT, etc.)
    colorScheme: theme,               // "dark" or "light"
    locale: lang,                     // "en-US", "fr-FR", etc.
    executablePath: require("electron/index.js"),  // Path to the Electron binary
    timeout: 120000,                  // 2 minutes to launch
  });
}
```

**Key points:**
- The first `args` entry is the **webpack bundle** — the compiled Ledger Live Desktop code
- `--user-data-dir` isolates each test's data (accounts, settings) in a unique directory
- `env` is where feature flags, Speculos connection, and test configuration are passed
- `require("electron/index.js")` finds the Electron binary installed by npm

### 45.3 The Environment Variables

When the `electronApp` fixture launches, it sets these environment variables:

```typescript
env = {
  VERBOSE: true,                    // Enable detailed logging
  MOCK: undefined,                  // Disable mocks (use real Speculos)
  HIDE_DEBUG_MOCK: true,            // Hide mock indicators in UI
  CI: process.env.CI,               // Are we in CI?
  PLAYWRIGHT_RUN: true,             // Tell the app it is running in Playwright
  CRASH_ON_INTERNAL_CRASH: true,    // Fail loudly on crashes
  LEDGER_MIN_HEIGHT: 768,           // Minimum window height
  FEATURE_FLAGS: JSON.stringify(mergedFeatureFlags),  // Feature flags JSON
  MANAGER_DEV_MODE: true,           // Enable developer features
  SPECULOS_API_PORT: String(speculos.current.port),   // Speculos connection
  SPECULOS_ADDRESS: getSpeculosAddress(),              // Speculos host
  DISABLE_TRANSACTION_BROADCAST: "1",  // Don't send real transactions
};
```

The most important ones:
- **`FEATURE_FLAGS`** — controls which UI features are enabled (modular drawer, Wallet 4.0, etc.)
- **`SPECULOS_API_PORT`** — tells the app where to find the emulated device
- **`DISABLE_TRANSACTION_BROADCAST`** — prevents tests from spending real crypto

### 45.4 `ElectronApplication` vs `Page`

After launch, you have two objects:

```typescript
const electronApp: ElectronApplication = await launchApp({ ... });
const page: Page = await electronApp.firstWindow();
```

- **`electronApp`** — represents the whole application. Used for:
  - Getting windows: `electronApp.firstWindow()`, `electronApp.windows()`
  - Evaluating code in the main process: `electronApp.evaluate()`
  - Closing the app: `electronApp.close()`

- **`page`** — represents the main window. This is what you interact with 99% of the time:
  - Finding elements: `page.getByTestId()`
  - Clicking, typing, asserting: all the locator/action/assertion APIs
  - Console logging: `page.on("console", msg => ...)`

Some page objects need both — for example, `SwapPage` and `BuyAndSellPage` access webview windows through `electronApp`:

```typescript
export class SwapPage extends WebViewAppPage {
  constructor(page: Page, electronApp?: ElectronApplication) {
    super(page, electronApp);
  }
  // Access the webview window
  const [, webview] = this.electronApp.windows();
}
```

### 45.5 The Application Startup Sequence

After the Electron app launches, the fixture waits for it to be ready:

```typescript
// Get the first window
const page = await electronApp.firstWindow();

// Set generous timeout (CI can be slow)
page.setDefaultTimeout(120000);

// Attach network logging
attachNetworkLogging(page, testInfo);

// Record all console messages to a log file
page.on("console", msg => {
  safeAppendFile(logFile, `${txt}\n`);
});

// Start collecting webview logs
const webviewCollector = new WebviewLogCollector();
webviewCollector.start(electronApp);

// Wait for the page HTML to load
await page.waitForLoadState("domcontentloaded");

// Wait for the loading screen to disappear
await page.waitForSelector("#loader-container", { state: "hidden" });

// NOW the test can start
```

The `#loader-container` is Ledger Live's splash screen. Once it disappears, the portfolio or onboarding screen is visible and the test can begin.

### 45.6 Webviews

A **webview** is an embedded browser window inside the Electron app. Ledger Live uses webviews for:
- **Swap** — the exchange interface is a separate web app loaded in a webview
- **Earn** — staking providers run in webviews
- **Buy/Sell** — external payment providers
- **Discover** — Live Apps marketplace

When testing webview content, you access the webview window through `electronApp.windows()`:

```typescript
// The first window is the main app, subsequent ones are webviews
const windows = electronApp.windows();
const mainPage = windows[0];
const webview = windows[1];  // The webview window

// Now interact with the webview
await webview.getByTestId("swap-amount-input").fill("0.01");
```

The `WebviewLogCollector` automatically captures console and network logs from all webviews for debugging.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://playwright.dev/docs/api/class-electron">Playwright: Electron API</a> — full Electron testing reference</li>
<li><a href="https://www.electronjs.org/docs/latest/">Electron: Official Documentation</a> — understanding the framework</li>
<li><a href="https://www.electronjs.org/docs/latest/tutorial/process-model">Electron: Process Model</a> — main vs renderer process</li>
<li><a href="https://playwright.dev/docs/api/class-electronapplication">Playwright: ElectronApplication class</a> — methods and properties</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Electron bundles Chromium + Node.js into a desktop app. Playwright controls it by launching the compiled webpack bundle with isolated userdata and environment variables. The <code>page</code> object represents the main window, <code>electronApp</code> represents the whole app. Feature flags, Speculos ports, and test configuration all flow through environment variables at launch time.
</div>

### 45.7 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Electron Testing</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> What does the <code>--user-data-dir</code> flag do when launching the Electron app?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Sets the directory where the app binary is located</button>
<button class="quiz-choice" data-value="B">B) Isolates each test's persistent data (accounts, settings, app.json) in a unique directory</button>
<button class="quiz-choice" data-value="C">C) Specifies where to download updates</button>
<button class="quiz-choice" data-value="D">D) Sets the working directory for the test runner</button>
</div>
<p class="quiz-explanation">Each test gets a unique UUID directory under <code>tests/artifacts/userdata/</code>. The <code>--user-data-dir</code> flag tells Electron to read/write its persistent state (like <code>app.json</code>) from that directory, ensuring complete test isolation.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> What is the purpose of <code>DISABLE_TRANSACTION_BROADCAST: "1"</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Disables network access entirely</button>
<button class="quiz-choice" data-value="B">B) Prevents the app from connecting to Speculos</button>
<button class="quiz-choice" data-value="C">C) Prevents the app from broadcasting real cryptocurrency transactions to the blockchain</button>
<button class="quiz-choice" data-value="D">D) Disables test result reporting</button>
</div>
<p class="quiz-explanation">Without this flag, send transaction tests would actually broadcast transactions and spend real cryptocurrency (from the test seed). This flag ensures the app goes through the entire transaction flow but stops before the final broadcast step.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> How does Playwright know the app is ready for testing?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It waits for <code>domcontentloaded</code> and then waits for <code>#loader-container</code> to be hidden</button>
<button class="quiz-choice" data-value="B">B) It uses a fixed 5-second delay</button>
<button class="quiz-choice" data-value="C">C) It checks if the Electron process is running</button>
<button class="quiz-choice" data-value="D">D) It sends a health check HTTP request to the app</button>
</div>
<p class="quiz-explanation">The fixture first waits for the HTML to load (<code>domcontentloaded</code>), then waits for the loading splash screen (<code>#loader-container</code>) to disappear. Only after both conditions are met does the test begin.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> When do you need the <code>electronApp</code> object instead of just <code>page</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Always — <code>page</code> alone is never enough</button>
<button class="quiz-choice" data-value="B">B) Only when clicking buttons</button>
<button class="quiz-choice" data-value="C">C) Only for taking screenshots</button>
<button class="quiz-choice" data-value="D">D) When you need to access webview windows (Swap, Earn, Buy) or close the app</button>
</div>
<p class="quiz-explanation"><code>page</code> is the main window and handles 99% of interactions. <code>electronApp</code> is needed to: access additional windows (webviews), evaluate code in the main Node.js process, and close the application.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> What is the <code>FEATURE_FLAGS</code> environment variable?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A list of Playwright test tags</button>
<button class="quiz-choice" data-value="B">B) A JSON string controlling which app features are enabled — merged from defaults, env overrides, and test-specific overrides</button>
<button class="quiz-choice" data-value="C">C) The Firebase project configuration</button>
<button class="quiz-choice" data-value="D">D) A comma-separated list of test file names to run</button>
</div>
<p class="quiz-explanation">Feature flags are JSON objects like <code>{"lldModularDrawer": {"enabled": true, "params": {...}}}</code>. They are merged from three sources: ENV (<code>E2E_FEATURE_FLAGS_JSON</code>), default flags in <code>common.ts</code>, and test-specific overrides via <code>test.use({ featureFlags: ... })</code>.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Speculos -- Device Emulation Mastery

<div class="chapter-intro">
As you learned in Chapter 10, Speculos is a QEMU-based emulator that runs Ledger device apps on your computer. In this chapter, we go inside the code that manages Speculos during test execution: how the fixture launches Docker containers, how the REST API is used programmatically, how the <code>SpeculosFixtureHandle</code> pattern works, and how to debug common issues. After this chapter, you will understand exactly what happens between "test starts" and "device is ready."
</div>

### 46.1 How Speculos Works (Quick Recap)

Speculos emulates the ARM processor of a Ledger device using QEMU. It loads the actual `.elf` binary of a Nano app (Bitcoin, Ethereum, etc.) and runs it in a sandboxed environment. It exposes:

- A **REST API** (default port 5000) for button presses, screenshots, APDU commands
- A **TCP server** (default port 9999) for raw APDU communication
- A **web UI** at `http://localhost:5000` for visual debugging

In E2E tests, Speculos runs inside a **Docker container**. Each test gets its own container on a unique port.

### 46.2 The Speculos Fixture in `common.ts`

The `speculos` fixture in `common.ts` orchestrates the entire device lifecycle:

```typescript
speculos: async ({ speculosApp, cliCommands, userdataDestinationPath, cliCommandsOnApp }, use, testInfo) => {
  let currentDevice: SpeculosDevice | undefined;

  const handle: SpeculosFixtureHandle = {
    // Getter that throws if no device exists
    get current(): SpeculosDevice {
      if (!currentDevice) throw new Error("No device (missing speculosApp?)");
      return currentDevice;
    },
    // Method to stop current device and start a new one
    relaunch: async (appName: string) => {
      currentDevice = await launchSpeculos(appName, testInfo.title, currentDevice);
      return currentDevice;
    },
  };

  try {
    // 1. If cliCommandsOnApp exists, launch temporary Speculos for CLI commands
    if (cliCommandsOnApp?.length) {
      for (const { app, cmd } of cliCommandsOnApp) {
        currentDevice = await launchSpeculos(app.name, testInfo.title);
        await executeCliCommand(cmd, userdataDestinationPath);
        await cleanSpeculos(currentDevice);
      }
    }

    // 2. Launch the main Speculos device
    if (speculosApp) {
      currentDevice = await launchSpeculos(speculosApp.name, testInfo.title);

      // 3. Execute CLI commands to populate test data
      if (cliCommands?.length) {
        for (const cmd of cliCommands) {
          await executeCliCommand(cmd, userdataDestinationPath);
        }
      }
    }

    // 4. Provide the handle to the test
    await use(handle);

  } finally {
    // 5. Cleanup: stop the Docker container
    if (currentDevice) await cleanSpeculos(currentDevice);
  }
},
```

**The flow:**
1. If `cliCommandsOnApp` is set — launch temporary Speculos instances for data preparation, run CLI commands, then stop them
2. Launch the main Speculos instance with the app specified in `speculosApp`
3. Run any `cliCommands` to populate test data (accounts, balances)
4. Hand the device handle to the test
5. After the test, clean up the Docker container

### 46.3 `speculosUtils.ts` — Launch and Cleanup

```typescript
export async function launchSpeculos(appName, testTitle?, previousDevice?) {
  // Stop any previous device
  if (previousDevice) await cleanSpeculos(previousDevice);

  // Start a new Speculos container
  const device = await startSpeculos(
    testTitle ?? "cli_speculos",
    specs[appName.replace(/ /g, "_")],
    previousDevice?.port,
  );

  // For remote Speculos (CI): wait for the container to be ready
  if (process.env.REMOTE_SPECULOS === "true") {
    await waitForSpeculosReady(device.id);
  }

  // Register the Speculos port so the app can find the device
  setEnv("SPECULOS_API_PORT", device.port);
  CLI.registerSpeculosTransport(device.port.toString(), getSpeculosAddress());

  // Add device info to Allure report
  await allure.description("SPECULOS\nApp: " + device.appName + " (" + device.appVersion + ")");

  return device;
}

export async function cleanSpeculos(speculos) {
  // Stop the Docker container
  await stopSpeculos(speculos.id);
  // Unregister the transport module
  unregisterTransportModule("speculos-http-" + String(speculos.port));
}
```

### 46.4 The `SpeculosDevice` Object

When a Speculos container starts, you get a `SpeculosDevice`:

```typescript
type SpeculosDevice = {
  id: string;              // Docker container ID
  port: number;            // REST API port (random, e.g., 5023)
  appName: string;         // "Bitcoin", "Ethereum", etc.
  appVersion: string;      // "2.1.0"
  dependencies?: Array<{   // Side-loaded apps (e.g., for swaps)
    name: string;
    appVersion: string;
  }>;
};
```

The `port` is randomly assigned to enable parallel execution. No two tests share the same port.

### 46.5 Supported Devices

The `SPECULOS_DEVICE` environment variable selects which device model to emulate:

| Value | Device Model | Screen Type |
|-------|-------------|-------------|
| `nanoSP` | Nano S Plus | Small OLED, 2 buttons |
| `nanoX` | Nano X | Small OLED, 2 buttons |
| `stax` | Stax | Touch screen |
| `flex` | Flex | Touch screen |
| `nanoGen5` | Nano Gen 5 | Next-gen Nano |

Device tags in tests (`@NanoSP`, `@NanoX`, `@Stax`, etc.) indicate which devices a test supports. In CI, the matrix runs each device model.

### 46.6 The Speculos REST API

Even though the E2E tests interact with Speculos through the live-common library (not direct HTTP calls), understanding the REST API helps with debugging:

```bash
# Take a screenshot of the device display
curl http://localhost:5023/screenshot -o device.png

# Press the right button (navigate right on Nano)
curl -d '{"action":"press-and-release"}' http://localhost:5023/button/right

# Press both buttons (confirm on Nano)
curl -d '{"action":"press-and-release"}' http://localhost:5023/button/both

# Touch the screen at (x, y) — for Stax/Flex
curl -d '{"action":"press-and-release","x":200,"y":300}' http://localhost:5023/finger

# Send an APDU command (raw device protocol)
curl -d '{"data":"e0c4000000"}' http://localhost:5023/apdu

# Get the current screen text (for debugging)
curl http://localhost:5023/events
```

### 46.7 Environment Variables for Speculos

```bash
# REQUIRED — set these before running tests
export MOCK=0                    # Disable mocks, use real Speculos
export SPECULOS_DEVICE=nanoSP    # Which device to emulate
export SEED="your test seed"     # Recovery phrase (get from QAA team)
export COINAPPS="/path/to/coin-apps"   # Repository with .elf binaries
export SPECULOS_IMAGE_TAG=ghcr.io/ledgerhq/speculos:master  # Docker image

# OPTIONAL — usually managed by fixtures
export SPECULOS_API_PORT=5023    # Auto-assigned by fixtures
export SPECULOS_ADDRESS=http://localhost  # Speculos host
export REMOTE_SPECULOS=true      # Use remote Speculos (CI)
```

### 46.8 Debugging Speculos Issues

**"No Speculos device" or container timeout:**
```bash
# Check if Docker is running
docker ps

# Check for lingering containers from failed tests
docker ps -a | grep speculos

# Remove stuck containers
docker rm -f $(docker ps -a -q --filter ancestor=ghcr.io/ledgerhq/speculos:master)
```

**"App not found" error:**
```bash
# Update your coin-apps repository
cd $COINAPPS
git pull origin develop
```

**Port conflicts:**
```bash
# Check what is using a port
lsof -i :5000
# Kill the process
kill -9 <PID>
```

**View device screen during test:**
Open `http://localhost:<port>` in your browser while the test is running (replace `<port>` with the Speculos port from the console output).

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://github.com/LedgerHQ/speculos">Speculos: GitHub Repository</a> — source code and documentation</li>
<li><a href="https://github.com/LedgerHQ/speculos/blob/master/docs/user/api.md">Speculos: REST API Reference</a> — all endpoints</li>
<li><a href="https://github.com/LedgerHQ/speculos/blob/master/docs/user/docker.md">Speculos: Docker Guide</a> — container setup</li>
<li><a href="https://github.com/LedgerHQ/speculos/blob/master/docs/user/usage.md">Speculos: Command-Line Usage</a> — CLI options and models</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Speculos runs Ledger device apps in Docker containers, each on a unique port for parallel execution. The fixture system orchestrates the entire lifecycle: launch container, populate data via CLI, hand the device to the test, clean up after. When debugging, check Docker first (<code>docker ps</code>), then environment variables, then coin-apps freshness.
</div>

### 46.9 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Speculos Mastery</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> Why does each Speculos container get a random port number?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) For security — random ports are harder to attack</button>
<button class="quiz-choice" data-value="B">B) To enable parallel execution — multiple test workers run simultaneously, each needing its own isolated Speculos instance</button>
<button class="quiz-choice" data-value="C">C) Speculos requires random ports by design</button>
<button class="quiz-choice" data-value="D">D) To avoid conflicts with the Ledger Live app's default port</button>
</div>
<p class="quiz-explanation">With <code>fullyParallel: true</code> and <code>workers: "100%"</code>, many tests run at the same time. Each gets its own Docker container on a unique port, preventing port conflicts.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q2.</strong> What is the purpose of <code>cliCommandsOnApp</code> in the fixture?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It launches temporary Speculos instances to run CLI commands (like data population) before the main test device starts</button>
<button class="quiz-choice" data-value="B">B) It runs Playwright commands before the Electron app launches</button>
<button class="quiz-choice" data-value="C">C) It installs npm packages needed by the test</button>
<button class="quiz-choice" data-value="D">D) It configures feature flags via the command line</button>
</div>
<p class="quiz-explanation"><code>cliCommandsOnApp</code> is used when you need to run CLI commands against a different Speculos app than the test's main app. For example, a swap test might need to populate data for both the source and destination currencies using different Speculos instances.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> What should you check FIRST when Speculos fails to start?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The Playwright configuration file</button>
<button class="quiz-choice" data-value="B">B) The test source code</button>
<button class="quiz-choice" data-value="C">C) The Allure report</button>
<button class="quiz-choice" data-value="D">D) Docker — is it running? Are there stuck containers from previous test runs?</button>
</div>
<p class="quiz-explanation">The #1 cause of Speculos failures is Docker issues: Docker Desktop not running, stuck containers from interrupted tests, or port conflicts. Always check <code>docker ps -a</code> first.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q4.</strong> What does <code>MOCK=0</code> do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Enables mock data for faster tests</button>
<button class="quiz-choice" data-value="B">B) Disables all network requests</button>
<button class="quiz-choice" data-value="C">C) Disables mocked device responses so the app talks to a real Speculos emulator instead</button>
<button class="quiz-choice" data-value="D">D) Turns off test assertions</button>
</div>
<p class="quiz-explanation"><code>MOCK=0</code> tells the Ledger Live app to NOT use mock device responses. Instead, it communicates with the actual Speculos emulator via the port specified in <code>SPECULOS_API_PORT</code>. This is required for Speculos-based E2E tests.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q5.</strong> What is the <code>SpeculosFixtureHandle</code> and why does it have a <code>relaunch()</code> method?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It is a mutable wrapper around the current device. <code>relaunch()</code> allows a test to switch to a different Speculos app mid-test (e.g., for swap tests that need two different currency apps)</button>
<button class="quiz-choice" data-value="B">B) It is a Playwright page object for device interaction</button>
<button class="quiz-choice" data-value="C">C) It is a Docker container management utility</button>
<button class="quiz-choice" data-value="D">D) It is used only for retry logic when Speculos crashes</button>
</div>
<p class="quiz-explanation">The <code>SpeculosFixtureHandle</code> wraps the current device with a <code>current</code> getter and a <code>relaunch()</code> method. The <code>relaunch()</code> method stops the current container and starts a new one with a different app — essential for tests that need multiple device apps (like swaps).</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Allure & Xray -- Test Reporting

<div class="chapter-intro">
Tests are only useful if their results are readable. Allure is the reporting framework that turns raw test output into interactive HTML reports with steps, screenshots, videos, and links to Jira. This chapter traces the entire pipeline: from <code>@step</code> decorators in code to the final Allure report you read in the browser, including the custom Xray JSON reporter that syncs results to Jira.
</div>

### 47.1 The Reporting Pipeline

```
Your Code                  Playwright                   Files                      Reports
─────────                  ──────────                   ─────                      ───────
@step decorators    →  test.step() calls          →  allure-results/*.json    →  allure-report/index.html
addTmsLink()        →  allure.tms() metadata      →  allure-results/*.json    →  Clickable Jira links
captureArtifacts()  →  testInfo.attach()          →  allure-results/*.png     →  Embedded screenshots
                                                                                 
customJsonReporter  →  onTestEnd() / onExit()     →  xray-report.json        →  Jira/Xray import
```

### 47.2 The `@step` Chain in Practice

When you write:
```typescript
@step("Click add account button")
async clickAddAccountButton() {
  const selector = (await isWallet40Enabled(this.page))
    ? this.portfolioAddAccountButton
    : this.addAccountButton;
  await selector.click();
}
```

The execution chain is:
1. The decorator wraps the method in `test.step("Click add account button", () => ...)`
2. Playwright records the step start time and name
3. The method body executes
4. Playwright records the step end time and status (pass/fail)
5. The `allure-playwright` reporter writes this as a step in the Allure JSON
6. When you generate the report, this step appears in the test's step tree

In the Allure report, you see a collapsible hierarchy:
```
✓ [Bitcoin] Add account (12.3s)
  ├── Click add account button (0.8s)
  ├── Select asset by ticker (1.2s)
  ├── Select network (0.5s)
  ├── Select first account (3.1s)
  ├── Click close button (0.3s)
  ├── Wait for balance to be visible (2.4s)
  └── Expect accounts persisted in app.json (4.0s)
```

### 47.3 Artifact Capture on Failure

When a test fails, `captureArtifacts()` collects debugging evidence:

```typescript
export async function captureArtifacts(page, testInfo, electronApp, takeSpeculosScreenshot, webviewCollector) {
  // 1. Desktop screenshot
  const screenshot = await page.screenshot();
  await testInfo.attach("Screenshot", { body: screenshot, contentType: "image/png" });

  // 2. Speculos device screenshots (what the emulated device screen shows)
  if (takeSpeculosScreenshot) {
    await attachSpeculosScreenshots(testInfo);  // Single image or HTML gallery
  }

  // 3. Test execution logs (on last retry only)
  if (isLastRetry(testInfo)) {
    await page.evaluate(filePath => window.saveLogs(filePath), filePath);
    await testInfo.attach("Test logs", { path: filePath, contentType: "application/json" });
    await attachIfExists(testInfo, "Network failures", "network.log", "text/plain");
  }

  // 4. Webview logs (for Swap/Earn/Buy debugging)
  if (webviewCollector) {
    await testInfo.attach("Webview Console Logs", { body: webviewCollector.getFormattedConsoleLogs(), ... });
    await testInfo.attach("Webview Network Logs", { body: webviewCollector.getFormattedNetworkLogs(), ... });
  }

  // 5. Video recording (on last retry only)
  const video = page.video();
  if (video) {
    await electronApp.close();
    const videoData = await readFileAsync(await video.path());
    await testInfo.attach("Test Video", { body: videoData, contentType: "video/webm" });
  }
}
```

### 47.4 TMS Links and Bug Links

**Linking tests to Jira/Xray:**

```typescript
// In the test definition
annotation: { type: "TMS", description: "B2CQA-2499, B2CQA-2644" },

// Inside the test body
await addTmsLink(getDescription(test.info().annotations, "TMS").split(", "));
```

This creates clickable links in the Allure report. The URL template in `playwright.config.ts` turns `B2CQA-2499` into `https://ledgerhq.atlassian.net/browse/B2CQA-2499`.

**Bug links work the same way:**
```typescript
await addBugLink(["LIVE-12345"]);  // Links to a known bug ticket
```

### 47.5 Team Ownership

```typescript
test.use({ teamOwner: Team.WALLET_XP });

// This calls:
export async function addTeamOwner(team: Team) {
  await allure.owner(team.toString());       // Sets the owner in report
  await allure.parentSuite(team.toString()); // Groups tests by team
  await allure.feature(team.toString());     // Categorizes in features view
}
```

Teams: `WALLET_XP`, `QAA`, `SWAP`, `EARN`, `MARKET`, etc. In the Allure report, you can filter by team to see only your team's tests.

### 47.6 Reading an Allure Report

Generate and open the report:
```bash
cd e2e/desktop
pnpm run allure:generate
pnpm run allure:open
# Or simply:
npx allure serve allure-results
```

**The five views:**

| View | What it shows | When to use it |
|------|--------------|----------------|
| **Overview** | Pass/fail summary, duration, severity | Quick health check |
| **Suites** | Tests grouped by describe blocks | Find your test |
| **Graphs** | Charts of pass/fail/broken/skipped | Trend analysis |
| **Timeline** | When each test ran (parallel view) | Identify bottlenecks |
| **Categories** | Failures grouped by error type | Find systematic issues |

**Drilling into a failure:**
1. Click the failed test in Suites view
2. Read the **step trace** — which step failed?
3. Check the **Screenshot** — what was on screen?
4. Check the **Speculos Screenshot** — what was on the device?
5. Watch the **Video** — replay the entire test
6. Read **Network failures** — any 4xx/5xx errors?
7. Read **Webview Console Logs** — any JavaScript errors in webviews?

### 47.7 The Xray Custom Reporter

The `customJsonReporter.ts` generates a JSON file that maps test results to Xray test cases:

```typescript
class JsonReporter implements Reporter {
  onTestEnd(test, result) {
    // Parse TMS annotations to get Xray ticket IDs
    const tmsIds = getDescription(test.annotations, "TMS").split(", ");
    // Map pass/fail to Xray status
    for (const id of tmsIds) {
      this.results.push({
        testKey: id,
        status: result.status === "passed" ? "PASSED" : "FAILED",
      });
    }
  }

  onExit() {
    // Write xray-report.json for Jira import
    writeFileSync("xray-report.json", JSON.stringify({
      testExecutionKey: process.env.TEST_EXECUTION,
      tests: this.results,
    }));
  }
}
```

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://allurereport.org/docs/playwright/">Allure: Playwright Integration</a> — official setup guide</li>
<li><a href="https://allurereport.org/docs/how-it-works/">Allure: How It Works</a> — architecture overview</li>
<li><a href="https://docs.getxray.app/display/XRAY/Import+Execution+Results">Xray: Import Results</a> — Jira integration</li>
<li><a href="https://playwright.dev/docs/test-reporters">Playwright: Custom Reporters</a> — Reporter interface</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Allure turns <code>@step</code> decorators and <code>testInfo.attach()</code> calls into rich HTML reports with hierarchical steps, screenshots, videos, and Jira links. When a test fails, the Allure report is your forensic toolkit: step trace → screenshot → device screenshot → video → logs. The Xray reporter syncs results to Jira automatically.
</div>

### 47.8 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Allure & Xray Reporting</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> What does <code>addTmsLink(["B2CQA-2499"])</code> do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Sends the test result to Jira immediately</button>
<button class="quiz-choice" data-value="B">B) Creates a new Jira ticket</button>
<button class="quiz-choice" data-value="C">C) Creates a clickable link in the Allure report that points to the Jira ticket <code>B2CQA-2499</code></button>
<button class="quiz-choice" data-value="D">D) Tags the test with a device model</button>
</div>
<p class="quiz-explanation"><code>addTmsLink()</code> calls <code>allure.tms(id)</code> which creates a TMS (Test Management System) link in the report. The URL template in <code>playwright.config.ts</code> generates the full Jira URL.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> When investigating a test failure in Allure, what is the recommended debugging order?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Video → Screenshot → Code</button>
<button class="quiz-choice" data-value="B">B) Step trace (which step failed?) → Screenshot → Speculos screenshot → Video → Network logs</button>
<button class="quiz-choice" data-value="C">C) Network logs → Video → Screenshot</button>
<button class="quiz-choice" data-value="D">D) Re-run the test immediately without reading the report</button>
</div>
<p class="quiz-explanation">Start with the step trace to identify WHERE the failure occurred. Then check screenshots (desktop and device) to see what was visible. Use video for complex interactions. Check network logs for API failures. This order goes from fastest to most detailed.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> Why is video only captured on the last retry?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) To save disk space — video files are large, and failed tests that pass on retry don't need video from earlier attempts</button>
<button class="quiz-choice" data-value="B">B) Because Playwright cannot record video during retries</button>
<button class="quiz-choice" data-value="C">C) Video is only available in CI environments</button>
<button class="quiz-choice" data-value="D">D) The first retry is always too fast for video</button>
</div>
<p class="quiz-explanation">Recording produces large files. The <code>isLastRetry()</code> check ensures video is only captured on the final attempt — the one whose result will actually be reported. Earlier retries that fail don't need video since the test will be re-attempted.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> What does <code>teamOwner: Team.WALLET_XP</code> affect in the report?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Which Speculos device is used</button>
<button class="quiz-choice" data-value="B">B) The test's retry count</button>
<button class="quiz-choice" data-value="C">C) Which feature flags are enabled</button>
<button class="quiz-choice" data-value="D">D) The test's owner, parent suite, and feature in Allure — allowing filtering by team</button>
</div>
<p class="quiz-explanation"><code>addTeamOwner()</code> sets three Allure metadata fields: <code>owner</code>, <code>parentSuite</code>, and <code>feature</code>. This allows filtering the report to show only one team's tests.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> What is the purpose of <code>customJsonReporter.ts</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It generates the Allure HTML report</button>
<button class="quiz-choice" data-value="B">B) It generates a <code>xray-report.json</code> file that maps test results to Xray/Jira ticket IDs for automated import</button>
<button class="quiz-choice" data-value="C">C) It sends Slack notifications</button>
<button class="quiz-choice" data-value="D">D) It captures screenshots on failure</button>
</div>
<p class="quiz-explanation">The custom reporter implements Playwright's <code>Reporter</code> interface. In <code>onTestEnd()</code>, it reads TMS annotations and maps them to pass/fail. In <code>onExit()</code>, it writes the JSON file that Xray can import to update test execution results in Jira.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Codebase Deep Dive -- Every File Explained

<div class="chapter-intro">
This is your reference chapter. It catalogs every file in <code>e2e/desktop/tests/</code> with its purpose and how it connects to the rest of the suite. Keep this chapter open when you encounter an unfamiliar file — it will tell you what it does, why it exists, and where to look next.
</div>

### 48.1 Configuration Files

| File | Purpose |
|------|---------|
| `playwright.config.ts` | Test runner configuration (covered in Ch 43.6) |
| `tsconfig.json` | TypeScript config. Maps `~/*` to the desktop app source for type-checking |
| `package.json` | Dependencies and scripts: `test:playwright`, `allure:generate`, `lint`, `typecheck` |
| `.oxlintrc.json` | Linting rules for code quality |

### 48.2 The Fixture File

| File | Purpose |
|------|---------|
| `tests/fixtures/common.ts` | **The most important file.** Defines all test fixtures (covered in Ch 44.2-44.4) |

### 48.3 Page Objects — The Class Hierarchy

| File | Class | Extends | Purpose |
|------|-------|---------|---------|
| `page/abstractClasses.ts` | `PageHolder` | — | Base: holds `page` and optional `electronApp` |
| | `Component` | `PageHolder` | Adds shared UI: `loadingSpinner`, `toaster`, `dropdownOptions` |
| | `AppPage` | `Component` | Abstract base for all concrete page objects |
| `page/index.ts` | `Application` | `PageHolder` | Hub: aggregates all 30+ page objects into `app.*` |

### 48.4 Main Page Objects

| File | Class | Purpose | Key Methods |
|------|-------|---------|------------|
| `page/portfolio.page.ts` | `PortfolioPage` | Dashboard screen | `clickAddAccountButton()`, `expectBalanceVisibility()`, `checkOperationHistory()`, `expectAccountsPersistedInAppJson()` |
| `page/account.page.ts` | `AccountPage` | Single account detail | `clickSend()`, `clickReceive()`, `expectAccountBalance()`, `expectLastOperationsVisibility()`, `clickOnLastOperationAndReturnStatus()` |
| `page/accounts.page.ts` | `AccountsPage` | Accounts list | `navigateToAccountByName()`, `countAccounts()`, `showParentAccountTokens()` |
| `page/settings.page.ts` | `SettingsPage` | Settings screen | `goToAccountsTab()`, `clickHideEmptyTokenAccountsToggle()`, `changeLanguage()`, `clearCache()` |
| `page/market.page.ts` | `MarketPage` | Market/prices | Market data and navigation |
| `page/onboarding.page.ts` | `OnboardingPage` | First launch | `waitForLaunch()` |
| `page/mainNavigation.page.ts` | `MainNavigationPage` | Top-level nav | `openTargetFromMainNavigation()`, `openSettings()` |
| `page/speculos.page.ts` | `SpeculosPage` | Device control | `signSendTransaction()`, `expectValidAddressDevice()`, `shareViewKey()` |
| `page/swap.page.ts` | `SwapPage` | Swap/exchange | Webview-based, complex flow orchestration |
| `page/earn.dashboard.page.ts` | `EarnPage` | Earn/staking v1 | Provider navigation |
| `page/earn.v2.dashboard.page.ts` | `EarnV2Page` | Earn/staking v2 | New earn UI |
| `page/buyAndSell.page.ts` | `BuyAndSellPage` | Buy/Sell | External provider integration |
| `page/liveApp.page.ts` | `LiveApp` | Discover/marketplace | App launching |
| `page/lockscreen.page.ts` | `LockscreenPage` | Lock screen | `login()`, `checkInputErrorVisibility()` |

### 48.5 Component Classes

| File | Class | Purpose |
|------|-------|---------|
| `component/layout.component.ts` | `Layout` | Sidebar buttons (Portfolio, Market, Accounts, etc.) and topbar buttons (Sync, Settings, Notifications) |
| `component/drawer.component.ts` | `Drawer` | Generic side panel: `waitForDrawerToBeVisible()`, `closeDrawer()`, `selectAccountByName()` |
| `component/modal.component.ts` | `Modal` | Generic modal: `continue()`, `close()`, `toggleMaxAmount()`, `scrollUntilOptionIsDisplayed()` |
| `component/dialog.component.ts` | `Dialog` | Generic modular dialog |

### 48.6 Modal, Drawer, and Dialog Objects

**Modals** (overlay windows):

| File | Class | Purpose |
|------|-------|---------|
| `modal/add.account.modal.ts` | `AddAccountModal` | Account creation (legacy flow) |
| `modal/send.modal.ts` | `SendModal` | Send transaction |
| `modal/new.send.modal.ts` | `NewSendModal` | Send transaction (Wallet 4.0) |
| `modal/receive.modal.ts` | `ReceiveModal` | Receive address display |
| `modal/delegate.modal.ts` | `DelegateModal` | Staking delegation |
| `modal/settings.modal.ts` | `SettingsModal` | App settings |
| `modal/passwordlock.modal.ts` | `PasswordlockModal` | Password setup/login |
| `modal/private.balance.modal.ts` | `PrivateBalanceModal` | Privacy feature |

**Drawers** (side panels):

| File | Class | Purpose |
|------|-------|---------|
| `drawer/send.drawer.ts` | `SendDrawer` | Send confirmation details |
| `drawer/operation.drawer.ts` | `OperationDrawer` | Transaction details |
| `drawer/asset.drawer.ts` | `AssetDrawer` | Asset information |
| `drawer/delegate.drawer.ts` | `DelegateDrawer` | Delegation details |
| `drawer/swap.confirmation.drawer.ts` | `SwapConfirmationDrawer` | Swap verification |
| `drawer/ledger.sync.drawer.ts` | `LedgerSyncDrawer` | Cloud sync |
| `drawer/modular.scan.accounts.drawer.ts` | `ModularScanAccountsDrawer` | Device account scanning |
| `drawer/choose.asset.drawer.ts` | `ChooseAssetDrawer` | Multi-asset selection |

**Dialogs** (new modular UI):

| File | Class | Purpose |
|------|-------|---------|
| `dialog/modular.dialog.ts` | `ModularDialog` | Routes to asset/network/account dialogs |
| `dialog/modular.asset.dialog.ts` | `ModularAssetDialog` | Asset selection in new UI |
| `dialog/modular.network.dialog.ts` | `ModularNetworkDialog` | Network selection in new UI |
| `dialog/modular.account.dialog.ts` | `ModularAccountDialog` | Account selection in new UI |
| `dialog/fearGreed.dialog.ts` | `FearAndGreedDialog` | Market sentiment |

### 48.7 Utility Files

| File | Purpose |
|------|---------|
| `utils/allureUtils.ts` | Allure helpers: `addTmsLink()`, `captureArtifacts()`, `addTeamOwner()` |
| `utils/speculosUtils.ts` | `launchSpeculos()`, `cleanSpeculos()` |
| `utils/electronUtils.ts` | `launchApp()` — Electron launch configuration |
| `utils/featureFlagUtils.ts` | `isWallet40Enabled()`, flag presets |
| `utils/featureFlagsJsonUtils.ts` | `parseExtraFeatureFlags()` — parse `E2E_FEATURE_FLAGS_JSON` |
| `utils/cliUtils.ts` | `CLI.getAddress()`, `CLI.liveData()`, `CLI.tokenApproval()` |
| `utils/customJsonReporter.ts` | Xray JSON report generation |
| `utils/global.setup.ts` | Runs once before all tests: clean artifacts, get firmware version |
| `utils/global.teardown.ts` | Runs once after all tests (CI): extract feature flags to env properties |
| `utils/redux.ts` | `listenToReduxActions()`, `waitForReduxAction()` — state tracking |
| `utils/userdata.ts` | `getUserdata()`, `waitForAccountsPersisted()` — app.json polling |
| `utils/fileUtils.ts` | `safeAppendFile()`, `waitForFileToExist()`, file operations |
| `utils/modularSelectorUtils.ts` | `getModularSelector()` — detects modular vs legacy UI |
| `utils/networkLogging.ts` | `attachNetworkLogging()` — captures request failures |
| `utils/webviewLogCollector.ts` | Collects console and network logs from webviews |
| `utils/swapUtils.ts` | Swap test orchestration helpers |
| `utils/waitFor.ts` | Custom wait helpers |
| `utils/testInfoUtils.ts` | `isLastRetry()` and TestInfo helpers |

### 48.8 Userdata JSON Files

These are pre-configured app states that bypass setup steps:

| File | State | Use When |
|------|-------|----------|
| `skip-onboarding.json` | Onboarding complete, no accounts | Testing account creation flows |
| `skip-onboarding-with-last-seen-device.json` | Onboarding complete, device remembered | Most tests — skips device pairing |
| `1AccountBTC1AccountETH.json` | Has 1 BTC + 1 ETH account | Testing with existing portfolio |
| `1AccountDOT.json` | Has 1 Polkadot account | Polkadot-specific tests |
| `speculos-tests-app.json` | Full Speculos test suite data | Complex multi-account tests |
| `speculos-subAccount.json` | Parent + token accounts | Token/sub-account tests |
| `erc20-0-balance.json` | ERC20 tokens with 0 balance | Token visibility tests |
| `portfolioWithManyStablecoins.json` | Many stablecoin accounts | Performance/pagination tests |
| `swap-history.json` | Pre-populated swap history | Swap history display tests |

### 48.9 Test Data Enums (from `@ledgerhq/live-common`)

| Enum | Location | Purpose |
|------|----------|---------|
| `Currency` | `live-common/e2e/enum/Currency` | Cryptocurrency definitions with `speculosApp`, `ticker`, `id`, `name` |
| `Account` | `live-common/e2e/enum/Account` | Pre-defined accounts with addresses and currency references |
| `AppInfos` | `live-common/e2e/enum/AppInfos` | Speculos app names and versions |
| `Team` | `live-common/e2e/enum/Team` | Team names for Allure ownership |
| `Device` | `live-common/e2e/enum/Device` | Device model identifiers |
| `Fee` | `live-common/e2e/enum/Fee` | Fee strategies: SLOW, MEDIUM, FAST |

### 48.10 The Modular Dialog System

The codebase supports both **legacy** and **new modular** UI flows. The `getModularSelector()` function detects which is active:

```typescript
// From modularSelectorUtils.ts
export async function getModularSelector(app, type: "ASSET" | "ACCOUNT") {
  try {
    // Wait up to 5 seconds for the modular dialog to appear
    await app.modularDialog.waitForDialogToBeVisible(type, 5000);
    return app.modularDialog;  // New UI is active
  } catch {
    return null;  // Fall back to legacy UI
  }
}
```

In tests, this creates a dual-path pattern:

```typescript
const selector = await getModularSelector(app, "ASSET");
if (selector) {
  // New modular flow
  await selector.selectAssetByTicker(Currency.BTC);
  await selector.selectNetwork(Currency.BTC);
} else {
  // Legacy modal flow
  await app.addAccount.selectCurrency(Currency.BTC);
}
```

<div class="chapter-outro">
<strong>Key takeaway:</strong> This chapter is your map. When you encounter an unfamiliar file, check the tables above to understand its role. The codebase has three layers: <strong>page objects</strong> (50+ files that encapsulate UI interactions), <strong>test specs</strong> (26 files that define test scenarios), and <strong>utilities</strong> (20+ files that handle infrastructure). The modular dialog system adds a dual-path pattern you will see in many tests.
</div>

### 48.11 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Codebase Deep Dive</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="A">
<p><strong>Q1.</strong> What is the difference between <code>Modal</code>, <code>Drawer</code>, and <code>Dialog</code> in the codebase?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>Modal</code> is a centered overlay window, <code>Drawer</code> is a side panel that slides in, <code>Dialog</code> is the new modular UI system</button>
<button class="quiz-choice" data-value="B">B) They are all the same thing with different names</button>
<button class="quiz-choice" data-value="C">C) <code>Modal</code> is for errors, <code>Drawer</code> is for forms, <code>Dialog</code> is for confirmations</button>
<button class="quiz-choice" data-value="D">D) <code>Modal</code> is legacy, <code>Drawer</code> and <code>Dialog</code> are the same new UI</button>
</div>
<p class="quiz-explanation">The three types represent different UI patterns: Modals are centered overlays (e.g., Add Account), Drawers slide in from the side (e.g., Operation details), and Dialogs are the newest modular UI system that replaces some legacy flows.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q2.</strong> What does <code>getModularSelector()</code> return when the new modular UI is NOT active?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) An error is thrown</button>
<button class="quiz-choice" data-value="B">B) An empty object</button>
<button class="quiz-choice" data-value="C">C) The legacy modal instance</button>
<button class="quiz-choice" data-value="D">D) <code>null</code> — the test then uses the legacy flow</button>
</div>
<p class="quiz-explanation"><code>getModularSelector()</code> waits up to 5 seconds for the modular dialog. If it does not appear, it returns <code>null</code>, signaling the test to use the legacy modal flow. This creates the <code>if (selector) { ... } else { ... }</code> pattern.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q3.</strong> What is the purpose of the <code>skip-onboarding-with-last-seen-device.json</code> userdata file?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It creates a test account with Bitcoin</button>
<button class="quiz-choice" data-value="B">B) It installs the Speculos app</button>
<button class="quiz-choice" data-value="C">C) It provides pre-configured app state where onboarding is complete and the app remembers the last connected device</button>
<button class="quiz-choice" data-value="D">D) It sets feature flags for the test</button>
</div>
<p class="quiz-explanation">Userdata files are Redux state snapshots. <code>skip-onboarding-with-last-seen-device</code> has <code>hasCompletedOnboarding: true</code> and remembers the last device. This is the most commonly used userdata because it skips the onboarding flow and device pairing, going straight to the portfolio.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q4.</strong> Which page objects receive <code>electronApp</code> in addition to <code>page</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) All page objects receive both</button>
<button class="quiz-choice" data-value="B">B) Only those that need webview access: <code>SwapPage</code>, <code>BuyAndSellPage</code>, <code>EarnPage</code>, <code>EarnV2Page</code></button>
<button class="quiz-choice" data-value="C">C) Only <code>SpeculosPage</code></button>
<button class="quiz-choice" data-value="D">D) None — all use only <code>page</code></button>
</div>
<p class="quiz-explanation">In <code>page/index.ts</code>, most page objects are created with <code>new XxxPage(this.page)</code>. But Swap, Buy/Sell, and Earn need <code>new XxxPage(this.page, this.electronApp)</code> because they interact with webview windows, which require access to the ElectronApplication to get additional windows.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q5.</strong> Where do the <code>Currency</code>, <code>Account</code>, and <code>Team</code> enums live?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) In <code>@ledgerhq/live-common/e2e/enum/</code> — shared between desktop and mobile E2E suites</button>
<button class="quiz-choice" data-value="B">B) In <code>e2e/desktop/tests/testdata/</code> — desktop-only</button>
<button class="quiz-choice" data-value="C">C) In <code>playwright.config.ts</code></button>
<button class="quiz-choice" data-value="D">D) In each individual test file</button>
</div>
<p class="quiz-explanation">Test data enums are defined in <code>@ledgerhq/live-common/e2e/enum/</code>, a shared package in the monorepo. This means both desktop and mobile E2E suites use the same currency, account, and team definitions, ensuring consistency.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Your Daily Workflow -- From Ticket to PR

<div class="chapter-intro">
You know the concepts, the architecture, and the codebase. Now it is time to learn the <strong>workflow</strong> — the exact sequence of steps from receiving a Jira ticket to having a merged PR. This chapter is procedural: follow these steps every time you work on an E2E test ticket.
</div>

### 49.1 Step 1: Read the Jira Ticket

Every E2E task starts with a Jira ticket (e.g., `QAA-1139`). Read it carefully:

- **What does it ask?** "Automate test X on desktop"
- **Which Xray test cases?** Look for `B2CQA-XXXX` IDs — these are the test cases to link
- **Is there a mobile reference?** Check if the test already exists in `e2e/mobile/`
- **Which coins/devices?** The ticket or Xray case specifies supported platforms

### 49.2 Step 2: Check What Already Exists

Before writing anything, search the codebase:

```bash
# Search for similar test names
grep -r "Add account" e2e/desktop/tests/specs/

# Search for the currency
grep -r "Currency.BTC" e2e/desktop/tests/specs/

# Search for the Xray ticket
grep -r "B2CQA-786" e2e/desktop/tests/

# Check existing page objects for the feature
ls e2e/desktop/tests/page/modal/
ls e2e/desktop/tests/page/drawer/
```

You may find the test already partially exists, or that page objects already cover the flow.

### 49.3 Step 3: Set Up Your Environment

Run through this checklist before writing code:

```bash
# 1. Environment variables
export MOCK=0
export SPECULOS_DEVICE=nanoSP
export SEED="your test seed phrase"
export COINAPPS="/path/to/coin-apps"
export SPECULOS_IMAGE_TAG=ghcr.io/ledgerhq/speculos:master

# 2. Docker is running
docker info  # Should not error

# 3. Update coin-apps
cd $COINAPPS && git pull origin develop && cd -

# 4. Install dependencies
pnpm i

# 5. Build the app for testing
pnpm build:lld:deps
pnpm desktop build:testing

# 6. Install Playwright dependencies
pnpm e2e:desktop test:playwright:setup
```

### 49.4 Step 4: Explore the App Manually

Launch the app and manually perform the flow you are automating:

```bash
pnpm desktop dev  # Launch in development mode
```

- Navigate to the feature
- Open DevTools (`Cmd+Shift+I` on macOS, `Ctrl+Shift+I` on Linux/Windows)
- Use the Elements panel to find `data-testid` attributes
- Note which UI pattern is used: modal, drawer, or modular dialog
- Write down the sequence of user actions

### 49.5 Step 5: Plan Your Test Structure

Before writing code, decide:

| Decision | Options | How to Decide |
|----------|---------|---------------|
| **Userdata** | `skip-onboarding`, `skip-onboarding-with-last-seen-device`, etc. | Does your test need existing accounts? |
| **speculosApp** | `Currency.BTC.speculosApp`, etc. | Which blockchain does the test interact with? |
| **cliCommands** | `liveDataCommand(account)` or empty | Does the test need pre-populated account data? |
| **featureFlags** | Specific overrides or none | Does the test need non-default features? |
| **teamOwner** | `Team.WALLET_XP`, `Team.QAA`, etc. | Which team owns this feature? |
| **Tags** | Device + currency + family + scope | Which devices and currencies does this test support? |

### 49.6 Step 6: Write the Test

Use this template:

```typescript
import { test } from "tests/fixtures/common";
import { Team } from "@ledgerhq/live-common/e2e/enum/Team";
import { Currency } from "@ledgerhq/live-common/e2e/enum/Currency";
import { addTmsLink } from "tests/utils/allureUtils";
import { getDescription } from "tests/utils/customJsonReporter";

test.describe("YOUR FEATURE", () => {
  test.use({
    teamOwner: Team.QAA,
    userdata: "skip-onboarding-with-last-seen-device",
    speculosApp: Currency.BTC.speculosApp,
  });

  test("YOUR TEST DESCRIPTION", {
    tag: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
          "@bitcoin", "@family-bitcoin"],
    annotation: { type: "TMS", description: "B2CQA-XXXX" },
  }, async ({ app, userdataFile }) => {
    await addTmsLink(getDescription(test.info().annotations, "TMS").split(", "));

    // YOUR TEST STEPS HERE
    await app.portfolio.clickAddAccountButton();
    // ... more steps ...
  });
});
```

### 49.7 Step 7: Run Locally

```bash
# Run your specific test
pnpm e2e:desktop test:playwright -- --grep "YOUR TEST DESCRIPTION"

# Run with debug mode (step-by-step)
PWDEBUG=1 pnpm e2e:desktop test:playwright -- --grep "YOUR TEST"

# Run with single worker (easier to debug)
pnpm e2e:desktop test:playwright -- --grep "YOUR TEST" --workers=1
```

### 49.8 Step 8: Verify Stability

**Run the test at least 3 times.** A test that passes once but fails on subsequent runs is **flaky** — and flaky tests are worse than no tests.

```bash
# Run 3 times
for i in 1 2 3; do
  echo "--- Run $i ---"
  pnpm e2e:desktop test:playwright -- --grep "YOUR TEST"
done
```

Common flakiness causes:
- Race conditions (element not ready)
- Dynamic data (amounts change)
- Network timing (API responses vary)
- Selector instability (using text instead of testId)

### 49.9 Step 9: Check the Allure Report

```bash
cd e2e/desktop
npx allure serve allure-results
```

Verify:
- Steps appear correctly in the report
- TMS links are clickable and correct
- Team ownership is set
- Tags are correct
- No unexpected warnings or errors

### 49.10 Step 10: Commit, Push, Create PR

```bash
# Create a feature branch
git checkout -b feat/add-first-time-btc-account-test

# Stage your changes
git add e2e/desktop/tests/specs/your.spec.ts

# Commit with conventional commit format
git commit -m "test(e2e): add first-time BTC account test for desktop"

# Push to remote
git push -u origin feat/add-first-time-btc-account-test

# Create a PR targeting develop
gh pr create --base develop --title "test(e2e): add first-time BTC account test" \
  --body "## Summary
- Automates QAA-1139: Add Account (first time, BTC) on Desktop
- Links to Xray: B2CQA-786

## Test plan
- [x] Test passes locally 3/3 runs
- [x] Allure report verified
- [x] Tags and TMS links correct"
```

After the PR is merged, update the Jira ticket to "Done" and add a comment linking the PR.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://playwright.dev/docs/debug">Playwright: Debugging</a> — PWDEBUG mode and trace viewer</li>
<li><a href="https://playwright.dev/docs/test-ui-mode">Playwright: UI Mode</a> — interactive test debugging</li>
<li><a href="https://www.conventionalcommits.org/">Conventional Commits</a> — commit message format</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> The workflow is always the same: Read ticket → Check existing code → Set up environment → Explore manually → Plan fixtures → Write test → Run locally → Verify stability (3x) → Check Allure → Commit and PR. Following this sequence every time prevents common mistakes like missing TMS links, wrong userdata, or flaky tests.
</div>

### 49.11 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Daily Workflow</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> How many times should you run a new test locally before considering it stable?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) 1 time — if it passes, it is stable</button>
<button class="quiz-choice" data-value="B">B) At least 3 times — to catch intermittent failures</button>
<button class="quiz-choice" data-value="C">C) 10 times — the more the better</button>
<button class="quiz-choice" data-value="D">D) 0 times — CI will catch issues</button>
</div>
<p class="quiz-explanation">The wiki explicitly states: "Run test at least 3 times to ensure it is not flaky." A test that passes once but fails intermittently is worse than no test — it creates false alarms in CI and erodes trust in the suite.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q2.</strong> Before writing a new test, what should you check first?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Whether the test or similar coverage already exists in the codebase</button>
<button class="quiz-choice" data-value="B">B) Whether Playwright is installed</button>
<button class="quiz-choice" data-value="C">C) Whether the CI pipeline is running</button>
<button class="quiz-choice" data-value="D">D) Whether the Jira ticket has been assigned to you</button>
</div>
<p class="quiz-explanation">Always search the existing specs first. The test may already exist (just needs a new TMS link), or the page objects may already cover the flow (you just need to compose them). Duplicate coverage wastes maintenance effort.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> What is the correct commit message format for a new E2E test?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>Added new test for BTC account</code></button>
<button class="quiz-choice" data-value="B">B) <code>fix: add BTC account test</code></button>
<button class="quiz-choice" data-value="C">C) <code>WIP: BTC test</code></button>
<button class="quiz-choice" data-value="D">D) <code>test(e2e): add first-time BTC account test for desktop</code></button>
</div>
<p class="quiz-explanation">The repo uses Conventional Commits: <code>&lt;type&gt;(&lt;scope&gt;): &lt;description&gt;</code>. For new tests, use <code>test(e2e):</code>. This format is required for consistent changelogs and CI automation.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q4.</strong> Which tool do you use to find <code>data-testid</code> attributes on a screen?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The Allure report</button>
<button class="quiz-choice" data-value="B">B) The Playwright CLI</button>
<button class="quiz-choice" data-value="C">C) The browser DevTools (Elements panel) in the running Electron app</button>
<button class="quiz-choice" data-value="D">D) The Speculos REST API</button>
</div>
<p class="quiz-explanation">Launch the Ledger Live Desktop app (<code>pnpm desktop dev</code>), open DevTools with <code>Cmd+Shift+I</code>, and use the Elements panel to inspect DOM elements and find their <code>data-testid</code> attributes.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> What is the purpose of <code>PWDEBUG=1</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It enables verbose Speculos logging</button>
<button class="quiz-choice" data-value="B">B) It opens the Playwright Inspector, pausing before each action so you can step through the test interactively</button>
<button class="quiz-choice" data-value="C">C) It writes debug information to a file</button>
<button class="quiz-choice" data-value="D">D) It disables timeouts</button>
</div>
<p class="quiz-explanation"><code>PWDEBUG=1</code> opens the Playwright Inspector — a GUI tool that shows the test's actions step by step. You can pause, resume, inspect locators, and see what the test sees. This is the most powerful debugging tool for E2E tests.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Real Ticket Walkthrough -- QAA-1139 (Add Account BTC)

<div class="chapter-intro">
This is your capstone chapter. Everything you learned in Chapters 43-49 converges here on a real Jira ticket: <strong>QAA-1139 — Automate Add Account (first time, BTC) test on Desktop</strong>. We will walk through the entire process: understanding the ticket, analyzing existing code, writing the test, running it, and preparing the PR.
</div>

### 50.1 Understanding the Ticket

**Jira ticket:** QAA-1139
**Xray test case:** B2CQA-786
**Description:** "Add account when no account exists before (BTC)" — a first-time add account flow for Bitcoin on Desktop

**What this means:**
1. The app starts with NO existing accounts (empty portfolio)
2. The user adds a BTC account for the first time
3. The test verifies the account appears in the portfolio and account list

**Context:** This test already exists in mobile (`e2e/mobile/specs/addAccount/addAccountBTC.spec.ts`). We need to create the desktop version.

### 50.2 Analyzing Existing Desktop Coverage

The existing `add.account.spec.ts` already tests adding BTC:

```typescript
const currencies = [
  { currency: Currency.BTC, xrayTicket: "B2CQA-2499, B2CQA-2644, B2CQA-2672, B2CQA-2073" },
  // ... 17 more currencies
];

for (const currency of currencies) {
  test.describe("Add Accounts", () => {
    test.use({
      userdata: "skip-onboarding-with-last-seen-device",
      speculosApp: currency.currency.speculosApp,
    });
    test(`[${currency.currency.name}] Add account`, ...);
  });
}
```

**But notice:** The existing test uses `userdata: "skip-onboarding-with-last-seen-device"` and already has BTC tickets (`B2CQA-2499`). The ticket `B2CQA-786` ("Add account when no account exists before") is specifically about the **first-time** experience — when the empty state UI is shown.

**Your decision:** Should you add a new test to the existing file, or is the existing BTC test sufficient?

Let's check: the existing test starts from the portfolio (which shows the empty state when no accounts exist) and adds a BTC account. The `skip-onboarding-with-last-seen-device` userdata has zero accounts, so the portfolio IS in the empty state. The existing test does cover the first-time flow — but `B2CQA-786` is not yet linked.

**Two possible approaches:**

1. **Add the B2CQA-786 ticket ID to the existing BTC entry** — if the existing test fully covers the first-time flow
2. **Create a separate test** that explicitly validates first-time-specific UI elements (empty state message, onboarding prompts)

Let's check what `B2CQA-786` specifically requires. The mobile test does:
- Start with no accounts
- Navigate to add account
- Select BTC
- Complete the add account flow
- Verify the account appears

The existing desktop test does exactly this. **The simplest and correct approach is to add `B2CQA-786` to the existing BTC currency entry's xrayTicket field.**

### 50.2b Hands-On: Run the Desktop Test BEFORE the Fix

Before touching any code, run the existing desktop BTC add account test to see how it behaves today. This builds your mental model and gives you a baseline to compare after the fix.

**Prerequisites** (one-time setup — skip if already done):
```bash
# Ensure Docker is running (Speculos needs it)
docker info

# Set environment variables in your shell (e.g. ~/.zshrc)
export MOCK=0
export COINAPPS="/path/to/coin-apps"        # your local coin-apps clone
export SEED="your 24-word test seed"
export SPECULOS_IMAGE_TAG="ghcr.io/ledgerhq/speculos:master"
export SPECULOS_DEVICE="nanoSP"

# Build the desktop app for testing
pnpm i
pnpm build:lld:deps
pnpm build:cli
pnpm desktop build:testing
pnpm e2e:desktop test:playwright:setup      # Install Playwright browsers
```

**Run the BTC test:**
```bash
cd e2e/desktop
pnpm test:playwright -- --grep "Bitcoin.*Add account"
```

The test name is `[Bitcoin] Add account` inside the `Add Accounts` describe block. The `--grep` flag matches against the full test title.

**What you should observe:**
1. Playwright launches Electron (Ledger Live Desktop)
2. The app starts with `skip-onboarding-with-last-seen-device` userdata — empty portfolio, no accounts
3. The test clicks "Add Account" → modular drawer opens
4. BTC is selected → Speculos scans and finds "Bitcoin 1"
5. Account is added → portfolio shows balance and operations
6. Navigation to Accounts → Bitcoin 1 → verifies details
7. Test passes

**Now check the Allure report to see what metadata is reported:**
```bash
# Generate and open the Allure report
npx allure serve allure-results
```

This opens the report in your browser. Find the `[Bitcoin] Add account` test and look at:
- **TMS Links section** — you will see `B2CQA-2499`, `B2CQA-2644`, `B2CQA-2672`, `B2CQA-2073` listed
- **Notice:** `B2CQA-786` is **not** there — this is the gap we need to fix
- **Tags** — `@NanoSP`, `@bitcoin`, `@family-bitcoin`, etc.
- **Team** — `WALLET_XP`
- **Steps** — Each `@step`-decorated page object method appears as a named step with timing

> **Tip:** You can also use Playwright's built-in debug mode to step through the test interactively:
> ```bash
> PWDEBUG=1 pnpm test:playwright -- --grep "Bitcoin.*Add account"
> ```
> This opens the Playwright Inspector where you can pause, step, and inspect locators in real time.

Press `Ctrl+C` in the terminal to stop the Allure server when you are done reviewing.

### 50.2c Hands-On: Study the Mobile Reference Implementation

The ticket says this test "currently exists only in `e2e/mobile/specs/addAccount/addAccountBTC.spec.ts`". Let's look at how the mobile version works to understand the reference implementation.

**Read the mobile test:**
```bash
cat e2e/mobile/specs/addAccount/addAccountBTC.spec.ts
```

You will see:
```typescript
import { Currency } from "@ledgerhq/live-common/e2e/enum/Currency";
import { runAddAccountTest } from "./addAccount";

runAddAccountTest(
  Currency.BTC,
  ["B2CQA-2499", "B2CQA-2644", "B2CQA-2672", "B2CQA-786"],
  //                                            ^^^^^^^^^^^
  // B2CQA-786 is here! This is the ticket we need to link on desktop too.
  [
    "@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
    "@smoke", "@bitcoin", "@family-bitcoin",
  ],
);
```

**Key observation:** The mobile test passes `B2CQA-786` in its `tmsLinks` array. Now read the helper to understand the flow:
```bash
cat e2e/mobile/specs/addAccount/addAccount.ts
```

The helper `runAddAccountTest()` does:
1. Initializes the app with `userdata: "skip-onboarding"` — empty portfolio, no accounts
2. Waits for the portfolio page to load
3. Taps "Add Account" → "Import with your Ledger"
4. Selects BTC via modular drawer (or legacy flow as fallback)
5. Adds the account at index 0
6. Verifies the account appears with balance, operations, and correct address index

**Compare mobile vs desktop:**

| Aspect | Mobile (`addAccount.ts`) | Desktop (`add.account.spec.ts`) |
|---|---|---|
| Framework | Detox + Jest | Playwright |
| Userdata | `skip-onboarding` | `skip-onboarding-with-last-seen-device` |
| Starting state | Empty portfolio | Empty portfolio |
| Flow | Add account → verify | Add account → verify |
| `B2CQA-786` linked? | Yes | **No** — this is the gap |

Both tests cover the same scenario (first-time BTC add account from empty state). The only difference is the framework and the missing Xray link on desktop.

**(Optional) Run the mobile test** if you have the mobile build environment set up:
```bash
# Terminal 1: Metro bundler
pnpm mobile start

# Terminal 2: Run the test (iOS example)
pnpm mobile e2e:test -c ios.sim.debug -- --testNamePattern="add account.*BTC"
```

> **Note:** Mobile builds take 10-30 minutes and require Xcode/Android Studio. If you don't have the mobile environment ready, reading the code above is sufficient — the important takeaway is that `B2CQA-786` is in the mobile test's `tmsLinks` but missing from the desktop test's `xrayTicket`.

### 50.3 The Implementation

Open `e2e/desktop/tests/specs/add.account.spec.ts`:

```typescript
const currencies = [
  {
    currency: Currency.BTC,
    xrayTicket: "B2CQA-2499, B2CQA-2644, B2CQA-2672, B2CQA-2073",
    //          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    // ADD B2CQA-786 here:
    // xrayTicket: "B2CQA-2499, B2CQA-2644, B2CQA-2672, B2CQA-2073, B2CQA-786",
  },
```

**That is it for linking the ticket.** But wait — let's verify the existing test actually covers the "first time" aspects. Reading the test flow:

```typescript
async ({ app, userdataFile }) => {
  // 1. Starts from portfolio (empty state, no accounts)
  await app.portfolio.clickAddAccountButton();       // ✓ First time: empty state button

  // 2. Modular dialog or legacy
  const selector = await getModularSelector(app, "ASSET");
  if (selector) {
    await selector.validateItems();
    await selector.selectAssetByTicker(currency.currency);
    await selector.selectNetwork(currency.currency);
    await app.scanAccountsDrawer.selectFirstAccount(); // ✓ Selects BTC account from device
    await app.scanAccountsDrawer.clickCloseButton();
  } else {
    await app.addAccount.expectModalVisibility();
    await app.addAccount.selectCurrency(currency.currency);
    await app.addAccount.addAccounts();
    await app.addAccount.done();
  }

  // 3. Verify first account appears
  await app.portfolio.expectBalanceVisibility();      // ✓ Balance now visible
  await app.portfolio.checkOperationHistory();        // ✓ Operations visible
  await app.portfolio.expectAccountsPersistedInAppJson(userdataFile, 1, 5000); // ✓ Persisted

  // 4. Navigate to account
  await app.mainNavigation.openTargetFromMainNavigation("accounts");
  await app.accounts.navigateToAccountByName(firstAccountName);
  await app.account.expectAccountVisibility(firstAccountName);
  await app.account.expectAccountBalance();           // ✓ Account has balance
  await app.account.expectLastOperationsVisibility(); // ✓ Operations visible
```

**Conclusion:** The existing test already covers the full "first time add account BTC" flow. We just need to add the missing Xray ticket ID.

### 50.4 But What If the Ticket Required a NEW Test?

For learning purposes, here is how you would write it from scratch if the scenario were different enough:

```typescript
import { test } from "tests/fixtures/common";
import { Team } from "@ledgerhq/live-common/e2e/enum/Team";
import { Currency } from "@ledgerhq/live-common/e2e/enum/Currency";
import { addTmsLink } from "tests/utils/allureUtils";
import { getDescription } from "tests/utils/customJsonReporter";
import { getModularSelector } from "tests/utils/modularSelectorUtils";

test.describe("Add Account - First Time BTC", () => {
  test.use({
    teamOwner: Team.WALLET_XP,
    userdata: "skip-onboarding-with-last-seen-device",  // No existing accounts
    speculosApp: Currency.BTC.speculosApp,               // Bitcoin device app
  });

  test(
    "Add BTC account when no account exists (first time)",
    {
      tag: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
            "@bitcoin", "@family-bitcoin"],
      annotation: {
        type: "TMS",
        description: "B2CQA-786",
      },
    },
    async ({ app, userdataFile }) => {
      // Link to Xray
      await addTmsLink(getDescription(test.info().annotations, "TMS").split(", "));

      // Step 1: From empty portfolio, click Add Account
      await app.portfolio.clickAddAccountButton();

      // Step 2: Select BTC (modular dialog or legacy)
      const selector = await getModularSelector(app, "ASSET");
      if (selector) {
        await selector.selectAssetByTicker(Currency.BTC);
        await selector.selectNetwork(Currency.BTC);
        await app.scanAccountsDrawer.selectFirstAccount();
        await app.scanAccountsDrawer.clickCloseButton();
      } else {
        await app.addAccount.expectModalVisibility();
        await app.addAccount.selectCurrency(Currency.BTC);
        await app.addAccount.addAccounts();
        await app.addAccount.done();
      }

      // Step 3: Verify account was created
      await app.portfolio.expectBalanceVisibility();
      await app.portfolio.expectAccountsPersistedInAppJson(userdataFile, 1, 5000);

      // Step 4: Navigate to the new account
      await app.mainNavigation.openTargetFromMainNavigation("accounts");
      await app.accounts.navigateToAccountByName("Bitcoin 1");
      await app.account.expectAccountVisibility("Bitcoin 1");
      await app.account.expectAccountBalance();
    },
  );
});
```

### 50.5 Hands-On: Run the Fixed Test and Generate the Allure Report

Now that you have made the one-line change (adding `B2CQA-786` to the BTC `xrayTicket`), run the test again to confirm it still passes and verify that the Allure report now includes the new ticket.

#### Step 1: Run the test

```bash
cd e2e/desktop
pnpm test:playwright -- --grep "Bitcoin.*Add account"
```

The test should pass exactly as before — you only changed metadata, not behavior.

#### Step 2: Run multiple times for stability

Before opening a PR, run the test at least 3 times to make sure it is stable:
```bash
pnpm test:playwright -- --grep "Bitcoin.*Add account" --repeat-each=3
```

All 3 runs should pass. If any run fails, investigate — it may be a flaky test or an environment issue (Docker not running, Speculos timeout, etc.).

#### Step 3: Generate and open the Allure report

Allure is the test reporting framework used across Ledger Live E2E tests. It produces an interactive HTML report from the `allure-results/` directory that Playwright populates after each run.

**Option A — Serve directly (recommended for quick checks):**
```bash
npx allure serve allure-results
```
This generates a temporary report and opens it in your browser. Press `Ctrl+C` to stop the server when done.

**Option B — Generate a persistent report:**
```bash
# Generate the report into ./allure-report/
pnpm allure:generate

# Open it manually
open allure-report/index.html        # macOS
# or: xdg-open allure-report/index.html  # Linux
```

#### Step 4: Navigate the Allure report

Once the report is open in your browser:

1. **Find the test:** Click on **Suites** in the left sidebar → expand **Add Accounts** → click **[Bitcoin] Add account**

2. **Check the TMS Links:** In the test detail panel, look for the **Links** section. You should now see:
   - `B2CQA-2499`
   - `B2CQA-2644`
   - `B2CQA-2672`
   - `B2CQA-2073`
   - `B2CQA-786` — **this is the new one you just added**

   Each link is clickable and points to the Xray test case in Jira. Compare this with the report you generated in Section 50.2b — `B2CQA-786` was missing before your change, now it is present.

3. **Check the test metadata:**
   - **Tags:** `@NanoSP`, `@bitcoin`, `@family-bitcoin`, etc.
   - **Owner/Team:** `WALLET_XP`
   - **Duration:** How long the test took (typically 30-90 seconds)

4. **Explore the test steps:** Click on the test to expand its step trace. Each `@step`-decorated page object method appears as a named step:
   - `portfolio.clickAddAccountButton()`
   - `modularSelector.selectAssetByTicker(BTC)`
   - `scanAccountsDrawer.selectFirstAccount()`
   - `portfolio.expectBalanceVisibility()`
   - etc.

   Each step shows its duration. If a step fails, the report shows the failure message, a screenshot, and (if configured) a video recording.

5. **Explore the report tabs:**
   - **Overview** — Summary with pass/fail counts, duration, and environment info
   - **Suites** — Tests grouped by `test.describe()` blocks
   - **Graphs** — Duration trends, status distribution
   - **Timeline** — Gantt chart of parallel test execution

#### Step 5: Typecheck

Run the typecheck to make sure the metadata change did not introduce any issues:
```bash
cd ../..   # back to monorepo root
pnpm e2e:desktop typecheck
```

This should pass cleanly — the `xrayTicket` field is a plain string, so adding a ticket ID cannot break types.

### 50.6 Git Workflow for This Ticket

```bash
# 1. Create branch from develop
git checkout develop
git pull
git checkout -b feat/link-btc-first-time-add-account-xray

# 2. Make the change (add B2CQA-786 to the xrayTicket)
# Edit e2e/desktop/tests/specs/add.account.spec.ts

# 3. Commit
git add e2e/desktop/tests/specs/add.account.spec.ts
git commit -m "test(e2e): link B2CQA-786 to BTC add account test on desktop"

# 4. Push and create PR
git push -u origin feat/link-btc-first-time-add-account-xray
gh pr create --base develop \
  --title "test(e2e): link B2CQA-786 to BTC add account test" \
  --body "Closes QAA-1139. Links Xray B2CQA-786 to the existing BTC add account test."
```

### 50.7 Updating Jira

After the PR is merged:
1. Move QAA-1139 to **Done**
2. Add a comment: "Linked B2CQA-786 to existing desktop BTC add account test in PR #XXXX"
3. Verify that the next CI run includes B2CQA-786 in the Xray report

<div class="chapter-outro">
<strong>Key takeaway:</strong> Real E2E work is not always writing new tests. Sometimes the right answer is linking an Xray ticket to existing coverage. Always check what exists before writing new code. When you do write new tests, follow the template from Section 50.4 — it includes every pattern you need: imports, fixtures, modular dialog detection, assertions, and TMS linking.
</div>

### 50.8 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Real Ticket Walkthrough</h3>
<p class="quiz-subtitle">4 questions · 75% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> For QAA-1139, why was the correct approach to add the ticket ID to existing code instead of writing a new test?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because writing new tests is not allowed</button>
<button class="quiz-choice" data-value="B">B) Because the existing <code>add.account.spec.ts</code> already covers the "first time BTC add account" flow — the <code>skip-onboarding-with-last-seen-device</code> userdata starts with zero accounts</button>
<button class="quiz-choice" data-value="C">C) Because BTC is not supported by Speculos</button>
<button class="quiz-choice" data-value="D">D) Because the mobile test already provides coverage</button>
</div>
<p class="quiz-explanation">The existing test already starts with no accounts (empty portfolio), adds BTC, and verifies the result. Adding a duplicate test would waste maintenance effort. The correct action is to link the Xray ticket (<code>B2CQA-786</code>) to the existing test so it appears in coverage reports.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> In the test template, why is <code>getModularSelector()</code> used instead of directly calling <code>app.addAccount.selectCurrency()</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>selectCurrency()</code> does not exist</button>
<button class="quiz-choice" data-value="B">B) The modular selector is faster</button>
<button class="quiz-choice" data-value="C">C) The app has two UI paths (modular dialog and legacy modal) controlled by feature flags, and the test must handle both</button>
<button class="quiz-choice" data-value="D">D) The modular selector is required for Speculos</button>
</div>
<p class="quiz-explanation">The <code>lldModularDrawer</code> feature flag controls whether the new modular dialog or the legacy modal is shown. Since feature flags can change between environments, the test detects which is active and follows the appropriate path.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> What userdata file would you use for a test that requires an EXISTING BTC account with balance?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>1AccountBTC1AccountETH.json</code> — it has pre-populated BTC and ETH accounts</button>
<button class="quiz-choice" data-value="B">B) <code>skip-onboarding.json</code> — it has onboarding complete</button>
<button class="quiz-choice" data-value="C">C) <code>skip-onboarding-with-last-seen-device.json</code> — it has no accounts</button>
<button class="quiz-choice" data-value="D">D) No userdata — the test creates accounts from scratch</button>
</div>
<p class="quiz-explanation"><code>1AccountBTC1AccountETH.json</code> contains pre-configured state with 1 BTC account and 1 ETH account with balances. Use this when your test starts from a portfolio that already has accounts (e.g., send, receive, account details tests).</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> After merging your PR, what should you do in Jira?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Delete the ticket</button>
<button class="quiz-choice" data-value="B">B) Reassign it to someone else</button>
<button class="quiz-choice" data-value="C">C) Leave it as-is — CI will update it</button>
<button class="quiz-choice" data-value="D">D) Move the ticket to Done and add a comment linking the PR</button>
</div>
<p class="quiz-explanation">After merge, move the ticket to Done and add a comment with the PR link. This closes the loop and provides traceability: anyone looking at the ticket can find the implementation.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Exercises & Challenges

<div class="chapter-intro">
Reading is not enough — you need to practice. These exercises progress from read-only analysis to hands-on code writing. Each has a clear objective, estimated time, and verification method. Complete them in order.
</div>

### 51.1 Exercise 1: Trace the Fixture Lifecycle (30 min)

**Objective:** Understand how fixtures wire together.

**Instructions:**
1. Open `e2e/desktop/tests/fixtures/common.ts`
2. For each fixture (`userdataDestinationPath`, `speculos`, `electronApp`, `page`, `app`), identify:
   - What other fixtures it depends on (appears in its parameter list)
   - What setup it performs (before `use()`)
   - What it provides to the test (argument to `use()`)
   - What teardown it performs (after `use()`)
3. Draw the dependency graph on paper or in a text file

**Expected output:** A diagram like:
```
userdata → userdataOriginalFile → userdataDestinationPath
                                          ↓
speculosApp → speculos ←── cliCommands
                  ↓
              electronApp ←── featureFlags, env, lang, theme
                  ↓
                page ←── teamOwner
                  ↓
                 app
```

**Verification:** Compare your diagram against the fixture code. Every arrow should correspond to a fixture name appearing in another fixture's parameter list.

### 51.2 Exercise 2: Add an Assertion to a Page Object (30 min)

**Objective:** Practice the `@step` decorator and page object pattern.

**Instructions:**
1. Open `e2e/desktop/tests/page/portfolio.page.ts`
2. Add a new method `expectNoAccountsMessage()` that:
   - Uses the `@step` decorator with a descriptive message
   - Checks that the "empty state" UI is visible (use `addAccountButton` as a proxy)
   - Uses `expect().toBeVisible()`
3. Save the file (do NOT commit yet — this is practice)

**Template:**
```typescript
@step("Expect empty portfolio state")
async expectNoAccountsMessage() {
  // YOUR CODE HERE
}
```

**Verification:** The method should follow the exact same pattern as `checkAddAccountButtonVisibility()` in the same file.

### 51.3 Exercise 3: Read and Explain a Test (45 min)

**Objective:** Prove you can read any test in the codebase.

**Instructions:**
1. Open `e2e/desktop/tests/specs/settings.spec.ts`
2. Pick the first test in the file
3. Write a line-by-line explanation of:
   - What each import does
   - What `test.use()` configures and why
   - What each `tag` means
   - What the annotation does
   - What each `await app.*` call does in the user flow
   - What the test verifies

**Verification:** Your explanation should match the test's actual behavior. Run the test to confirm: `pnpm e2e:desktop test:playwright -- --grep "Settings"`

### 51.4 Exercise 4: Debug a Broken Test (45 min)

**Objective:** Practice identifying and fixing common test bugs.

**Instructions:** Copy this test into a scratch file and find ALL the bugs:

```typescript
import { test } from "@playwright/test";  // BUG 1: wrong import
import { Team } from "@ledgerhq/live-common/e2e/enum/Team";
import { Currency } from "@ledgerhq/live-common/e2e/enum/Currency";

test.describe("Broken Test", () => {
  test.use({
    teamOwner: Team.WALLET_XP,
    userdata: "skip-onboarding",  // BUG 2: might need device-aware userdata
    speculosApp: Currency.BTC.speculosApp,
  });

  test("Add BTC account", {
    tag: ["@NanoSP"],
    annotation: { type: "TMS", description: "B2CQA-XXXX" },
  }, async ({ app }) => {
    // BUG 3: missing addTmsLink call
    app.portfolio.clickAddAccountButton();  // BUG 4: missing await
    await app.addAccount.selectCurrency(Currency.BTC);
    await app.addAccount.addAccounts();
    await app.addAccount.done();
    // BUG 5: no assertions — test never verifies the account was created
  });
});
```

**Bugs to find:**
1. Wrong import source
2. Userdata might not have last-seen device
3. Missing `addTmsLink()` call
4. Missing `await` on an async call
5. No assertions after the flow

**Verification:** Fix all 5 bugs and compare against the template in Chapter 49.6.

### 51.5 Exercise 5: Write a Test from Scratch (60 min)

**Objective:** Write a complete test for an existing flow.

**Instructions:**
1. Open `e2e/desktop/tests/specs/rename.account.spec.ts` and read it
2. Write a NEW test in the same file (or a scratch file) that:
   - Renames a BTC account (instead of the currency used in the existing test)
   - Uses the correct userdata (needs an existing account)
   - Has proper tags, annotations, and TMS links
   - Verifies the rename persisted
3. Run your test locally

**Hints:**
- You will need `cliCommands` to populate BTC account data
- Look at how the existing rename test sets up its userdata
- Check `app.account` for rename-related methods

### 51.6 Exercise 6: Extend the Add Account Test (45 min)

**Objective:** Add new assertions to an existing test.

**Instructions:**
1. After successfully understanding the add account test (Ch 50), add these verifications:
   - After adding the account, check that the operation history shows at least one operation
   - Verify the account name matches "Bitcoin 1"
   - Click on the last operation and verify the operation drawer opens with correct info

**Hint:** The existing test already does most of this. Study the assertions at the bottom of the `add.account.spec.ts` test and understand what each one verifies.

### 51.7 Challenge: Write a Complete Send BTC Test (90 min)

**Objective:** Write a full E2E test with no guidance.

**Task:** Write a test that:
1. Starts with an existing BTC account (pre-populated via CLI)
2. Opens the Send flow from the account page
3. Fills in a recipient address and amount
4. Signs the transaction on Speculos
5. Verifies the "Transaction sent" toaster appears
6. Verifies the transaction appears in the operation list

**No hints.** Use everything you learned in Chapters 43-50. Refer to:
- `send.tx.spec.ts` for existing send test patterns
- `send.modal.ts` for page object methods
- `speculos.page.ts` for device signing
- The daily workflow checklist (Ch 49)

<div class="chapter-outro">
<strong>Key takeaway:</strong> These exercises progress from reading to writing. Complete Exercises 1-4 first — they build the analytical skills you need. Then tackle 5-6, which require code changes. The Challenge is your graduation test: if you can write a complete Send BTC test from zero, you are ready to work independently on E2E tickets.
</div>

---

## Part 6 Final Assessment

<div class="quiz-container" data-pass-threshold="80">
<h3>Part 6 Final Assessment</h3>
<p class="quiz-subtitle">10 questions · 80% to pass · Covers all Part 6 chapters</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> What does Playwright's auto-waiting mechanism do before a <code>click()</code> action?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Waits a fixed 1 second</button>
<button class="quiz-choice" data-value="B">B) Checks that the element is attached, visible, stable, able to receive events, and enabled — retrying until timeout</button>
<button class="quiz-choice" data-value="C">C) Refreshes the page</button>
<button class="quiz-choice" data-value="D">D) Takes a screenshot</button>
</div>
<p class="quiz-explanation">Playwright's auto-waiting performs 5 actionability checks before every action, retrying the entire check until the timeout is reached. This eliminates manual sleep calls.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q2.</strong> What is the fixture dependency order from first to last?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) app → page → electronApp → speculos</button>
<button class="quiz-choice" data-value="B">B) page → electronApp → app → speculos</button>
<button class="quiz-choice" data-value="C">C) electronApp → speculos → page → app</button>
<button class="quiz-choice" data-value="D">D) speculos → electronApp → page → app</button>
</div>
<p class="quiz-explanation">Speculos must start first (Docker container). Then Electron launches with the Speculos port. Then the page is obtained from Electron. Finally, the Application hub wraps the page. Each depends on the previous.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> What is the purpose of Electron's <code>--user-data-dir</code> flag in the E2E context?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It isolates each test's persistent data in a unique directory, enabling parallel execution without data conflicts</button>
<button class="quiz-choice" data-value="B">B) It sets the download directory for the app</button>
<button class="quiz-choice" data-value="C">C) It specifies the Speculos connection</button>
<button class="quiz-choice" data-value="D">D) It configures the test reporter</button>
</div>
<p class="quiz-explanation">Each test gets a UUID-named directory under <code>tests/artifacts/userdata/</code>. The <code>--user-data-dir</code> flag points Electron to this directory so each parallel test has isolated state (accounts, settings, etc.).</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q4.</strong> How does the <code>@step("Click on asset $0")</code> decorator handle the <code>$0</code> placeholder?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It is ignored — <code>$0</code> appears literally in the step name</button>
<button class="quiz-choice" data-value="B">B) It is replaced with the test name</button>
<button class="quiz-choice" data-value="C">C) It is replaced with the first argument of the decorated method at runtime</button>
<button class="quiz-choice" data-value="D">D) It triggers a special Allure report section</button>
</div>
<p class="quiz-explanation">The step decorator uses <code>message.replace(/\$(\d+)/g, (_, idx) => String(args[Number(idx)]))</code> to substitute placeholders with function arguments. <code>$0</code> = first arg, <code>$1</code> = second arg, etc.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> What does <code>DISABLE_TRANSACTION_BROADCAST: "1"</code> prevent?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Network access during tests</button>
<button class="quiz-choice" data-value="B">B) Real cryptocurrency transactions from being sent to the blockchain</button>
<button class="quiz-choice" data-value="C">C) Speculos from starting</button>
<button class="quiz-choice" data-value="D">D) Test results from being reported</button>
</div>
<p class="quiz-explanation">Without this flag, send tests would broadcast real transactions using the test seed's funds. This flag ensures the app completes the entire send flow but stops before the final broadcast step.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q6.</strong> What is the recommended order for debugging a test failure in Allure?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Step trace → Screenshot → Speculos screenshot → Video → Network logs</button>
<button class="quiz-choice" data-value="B">B) Video → Screenshot → Logs</button>
<button class="quiz-choice" data-value="C">C) Re-run immediately without investigation</button>
<button class="quiz-choice" data-value="D">D) Read the source code first</button>
</div>
<p class="quiz-explanation">Start with the step trace (fastest — which step failed?), then check screenshots (what was visible?), then Speculos screenshot (what was on the device?), then video (for complex flows), then network logs (for API issues).</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q7.</strong> What does <code>getModularSelector(app, "ASSET")</code> return when the legacy UI is active?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) An error</button>
<button class="quiz-choice" data-value="B">B) The legacy modal object</button>
<button class="quiz-choice" data-value="C">C) An empty object</button>
<button class="quiz-choice" data-value="D">D) <code>null</code> — signaling the test to use the legacy flow path</button>
</div>
<p class="quiz-explanation"><code>getModularSelector()</code> waits 5 seconds for the modular dialog. If it doesn't appear, it catches the timeout and returns <code>null</code>. The test uses <code>if (selector) { ... } else { ... }</code> to handle both UI paths.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q8.</strong> Why should you run a new test at least 3 times before submitting it?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) To generate more Allure data</button>
<button class="quiz-choice" data-value="B">B) Because Playwright requires multiple runs</button>
<button class="quiz-choice" data-value="C">C) To catch flaky behavior — a test that passes once but fails intermittently is worse than no test</button>
<button class="quiz-choice" data-value="D">D) To warm up the Docker cache</button>
</div>
<p class="quiz-explanation">Flaky tests erode trust in the test suite. A test that passes 1/3 times will cause false alarms in CI. Running 3 times catches race conditions, timing issues, and dynamic data problems before they reach the main branch.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q9.</strong> Where do the <code>Currency</code> and <code>Account</code> enums live, and why?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) In each test file — for test isolation</button>
<button class="quiz-choice" data-value="B">B) In <code>@ledgerhq/live-common/e2e/enum/</code> — shared between desktop and mobile so both suites use identical test data definitions</button>
<button class="quiz-choice" data-value="C">C) In <code>playwright.config.ts</code> — for centralized configuration</button>
<button class="quiz-choice" data-value="D">D) In the Speculos repository — because they represent device apps</button>
</div>
<p class="quiz-explanation">Test data enums are in the shared <code>live-common</code> package so both desktop (Playwright) and mobile (Detox/Jest) E2E suites use the same currency, account, and team definitions. This prevents inconsistencies between platforms.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q10.</strong> What is the correct commit message format for adding a new desktop E2E test?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>test(e2e): add first-time BTC account test for desktop</code></button>
<button class="quiz-choice" data-value="B">B) <code>Added BTC test</code></button>
<button class="quiz-choice" data-value="C">C) <code>feat: new test for BTC</code></button>
<button class="quiz-choice" data-value="D">D) <code>WIP test BTC</code></button>
</div>
<p class="quiz-explanation">The repo uses Conventional Commits: <code>&lt;type&gt;(&lt;scope&gt;): &lt;description&gt;</code>. For test additions, use <code>test(e2e):</code> as the type and scope. This ensures consistent changelogs and follows the repository's contribution guidelines.</p>
</div>

<div class="quiz-score"></div>
</div>
