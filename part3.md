

# PART 3 -- E2E TESTING IN PRACTICE

<div class="chapter-intro">
Parts 1 and 2 gave you the knowledge — what you are testing, how the repo is organized, and how each tool works. Now you put it all together. This part takes you through the actual architecture of the Desktop and Mobile E2E suites, and then guides you through writing your first test on each platform. By the end, you will have written and run a real E2E test against a Speculos-emulated device.
</div>

---

## Desktop E2E -- Architecture Deep Dive

<div class="chapter-intro">
Before writing a test, you need to understand the architecture that supports it. The desktop E2E suite is a carefully layered system: spec files call page objects, page objects use the Playwright Page, and the fixture system orchestrates the entire lifecycle from Speculos launch to screenshot capture on failure. This chapter maps the directory structure, the class hierarchy, the Application hub, and the fixture lifecycle in detail.
</div>

### 13.1 Directory Structure

```
e2e/desktop/
|-- playwright.config.ts              # Test configuration
|-- package.json                      # Dependencies & scripts
|-- tsconfig.json                     # TypeScript config
|-- README.md                         # Quick start guide
|
|-- tests/
|   |-- specs/                        # TEST FILES (26 spec files)
|   |   |-- settings.spec.ts          # Settings tests
|   |   |-- send.tx.spec.ts           # Send transaction tests
|   |   |-- newSendFlow.tx.spec.ts    # New send flow tests
|   |   |-- add.account.spec.ts       # Add account tests
|   |   |-- receive.address.spec.ts   # Receive address tests
|   |   |-- delegate.spec.ts          # Staking/delegation tests
|   |   |-- send.swap.spec.ts         # Swap tests
|   |   |-- portfolio.spec.ts         # Portfolio tests
|   |   |-- market.spec.ts            # Market discovery tests
|   |   |-- onboarding.spec.ts        # First-launch tests
|   |   +-- ... (16 more)
|   |
|   |-- page/                         # PAGE OBJECTS
|   |   |-- abstractClasses.ts        # Base classes (PageHolder, Component, AppPage)
|   |   |-- index.ts                  # Application class (aggregates ALL pages)
|   |   |-- account.page.ts           # Single account operations
|   |   |-- accounts.page.ts          # Accounts list
|   |   |-- settings.page.ts          # Settings
|   |   |-- portfolio.page.ts         # Home/dashboard
|   |   |-- speculos.page.ts          # Device simulator controls
|   |   |-- mainNavigation.page.ts    # Top-level navigation
|   |   |-- market.page.ts            # Market discovery
|   |   |-- onboarding.page.ts        # First launch flow
|   |   |-- drawer/                   # Side panel UIs (8 files)
|   |   |-- modal/                    # Modal dialogs (8 files)
|   |   +-- dialog/                   # Modular dialogs
|   |
|   |-- fixtures/
|   |   +-- common.ts                 # THE MOST IMPORTANT FILE — fixture definitions
|   |
|   |-- utils/                        # Utilities
|   |   |-- global.setup.ts           # Runs once before all tests
|   |   |-- global.teardown.ts        # Runs once after all tests
|   |   |-- speculosUtils.ts          # Speculos Docker management
|   |   |-- electronUtils.ts          # Electron app launcher
|   |   |-- allureUtils.ts            # Allure reporting helpers
|   |   |-- featureFlagUtils.ts       # Feature flag management
|   |   |-- cliUtils.ts               # CLI data population
|   |   +-- customJsonReporter.ts     # Xray JSON export
|   |
|   |-- misc/reporters/
|   |   +-- step.ts                   # @step decorator for Allure
|   |
|   +-- userdata/                     # Pre-configured test profiles (JSON)
|       |-- erc20-0-balance.json
|       |-- skip-onboarding-with-last-seen-device.json
|       |-- 1AccountBTC1AccountETH.json
|       +-- ...
|
+-- allure-results/                   # Generated test results (gitignored)
```

### 13.2 The Class Hierarchy

```
PageHolder (abstract)
  |  constructor(page: Page)
  |  getPage(): Page
  |
  +-- Component (abstract)
        |  loadingSpinner, toaster, dropdownOptions
        |  waitForPageDomContentLoadedState()
        |  expectUrlToContainAll()
        |
        +-- AppPage (abstract)
              |
              |-- AccountPage         # send, receive, navigate to token
              |-- AccountsPage        # list accounts, count, show tokens
              |-- SettingsPage         # tabs, toggles, language, counter value
              |-- PortfolioPage        # balance, assets, operations
              |-- SpeculosPage         # sign transactions, verify addresses
              |-- MainNavigationPage   # navigate between sections
              |-- MarketPage           # market discovery
              |-- OnboardingPage       # first launch flow
              +-- ... (20+ more)
```

- **PageHolder**: Base. Holds the `Page` reference. Nothing else.
- **Component**: Adds shared UI elements (`loadingSpinner`, `toaster`, `dropdownOptions`) and common utility methods like `waitForPageDomContentLoadedState()`.
- **AppPage**: Abstract page class that all concrete page objects extend.
- **Concrete pages**: Each encapsulates one screen's locators and actions, decorated with `@step` for Allure.

### 13.3 The Application Class (Hub)

```typescript
// File: e2e/desktop/tests/page/index.ts
export class Application extends PageHolder {
  public account = new AccountPage(this.page);
  public accounts = new AccountsPage(this.page);
  public settings = new SettingsPage(this.page);
  public portfolio = new PortfolioPage(this.page);
  public speculos = new SpeculosPage(this.page);
  public mainNavigation = new MainNavigationPage(this.page);
  public addAccount = new AddAccountModal(this.page);
  public send = new SendModal(this.page);
  public newSendFlow = new NewSendModal(this.page);
  public receive = new ReceiveModal(this.page);
  // ... 30+ more page objects
}
```

In tests, you access everything through the single `app` object:
```typescript
test("example", async ({ app }) => {
  await app.mainNavigation.openTargetFromMainNavigation("accounts");
  await app.accounts.navigateToAccountByName("Bitcoin 1");
  await app.account.clickSend();
});
```

### 13.4 The Fixture Lifecycle

The fixture system runs this lifecycle automatically for each test:

```
1. CREATE USERDATA DIR
   |-- tests/artifacts/userdata/{uuid}/
   |-- Copy userdata JSON profile -> app.json
   +-- Merge default settings + test overrides

2. LAUNCH SPECULOS (if speculosApp specified)
   |-- Start Docker container with coin app .elf binary
   |-- Assign random port (5000+)
   |-- Execute CLI commands (populate accounts/transactions)
   +-- Return device handle

3. LAUNCH ELECTRON
   |-- Set env vars (SPECULOS_API_PORT, FEATURE_FLAGS, etc.)
   |-- Open app with --user-data-dir pointing to test userdata
   |-- Wait for #loader-container to hide
   +-- Return ElectronApplication + Page

4. CREATE APPLICATION
   +-- Instantiate all page objects with the live Page

5. YOUR TEST RUNS

6. TEARDOWN
   |-- FAILURE: capture screenshot, video, logs, Speculos screenshot
   |-- SUCCESS: delete video (save disk space)
   |-- Stop Speculos container
   +-- Clean up userdata directory
```

**Why random ports for Speculos?** Each test worker gets its own Speculos container on a unique port. This enables full parallel execution — multiple tests running simultaneously without port conflicts.

### 13.5 Sharding in CI (from the wiki)

For large test suites, CI uses **Playwright sharding** to split tests across multiple runners:

```bash
# Shard 1 of 4:
pnpm e2e:desktop test:playwright -- --shard=1/4
# Shard 2 of 4:
pnpm e2e:desktop test:playwright -- --shard=2/4
```

Each shard runs independently, with its own Speculos containers, and produces its own Allure results. Results from all shards are merged for the final report.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://playwright.dev/docs/test-fixtures">Playwright Fixtures deep dive</a></li>
<li><a href="https://playwright.dev/docs/test-sharding">Playwright Sharding</a> — how to split tests across CI runners</li>
<li><a href="https://playwright.dev/docs/pom">Page Object Model with Playwright</a></li>
<li><a href="https://martinfowler.com/bliki/PageObject.html">Martin Fowler on Page Object pattern</a></li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> The architecture is layered by design — specs → page objects → fixtures → utilities. Each layer has a single responsibility. When you write a test, you only interact with the <code>app</code> object. When you maintain the suite, you modify one layer without breaking the others. This separation is what makes a 26-file test suite manageable.
</div>

### 13.6 Quiz

<!-- ── Chapter 13 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> Which file is considered the "most important" in the desktop E2E suite?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>playwright.config.ts</code></button>
<button class="quiz-choice" data-value="B">B) <code>tests/fixtures/common.ts</code></button>
<button class="quiz-choice" data-value="C">C) <code>tests/page/index.ts</code></button>
<button class="quiz-choice" data-value="D">D) <code>tests/specs/settings.spec.ts</code></button>
</div>
<p class="quiz-explanation"><code>common.ts</code> defines all fixture types, handles Speculos lifecycle, Electron launch, page object creation, and teardown. Every test in the suite flows through it.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> What does the <code>Component</code> abstract class provide that <code>PageHolder</code> doesn't?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The Playwright Page reference</button>
<button class="quiz-choice" data-value="B">B) Shared UI elements like <code>loadingSpinner</code>, <code>toaster</code>, and utility methods like <code>waitForPageDomContentLoadedState()</code></button>
<button class="quiz-choice" data-value="C">C) Device interaction methods for Speculos</button>
<button class="quiz-choice" data-value="D">D) Test configuration and fixture types</button>
</div>
<p class="quiz-explanation"><code>PageHolder</code> only holds the Page reference. <code>Component</code> extends it with shared UI elements and common methods that all pages need, like spinner detection and URL checks.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q3.</strong> What happens to the test video on success vs failure?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Videos are always kept in both cases</button>
<button class="quiz-choice" data-value="B">B) Videos are always deleted in both cases</button>
<button class="quiz-choice" data-value="C">C) Videos are kept on failure (as Allure attachments) and deleted on success to save disk space</button>
<button class="quiz-choice" data-value="D">D) Videos are only recorded when a failure is detected</button>
</div>
<p class="quiz-explanation">Recording is always active, but on success the video is deleted to save disk space. On failure, the video is preserved and attached to the Allure report for debugging.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> Why does the fixture system assign random ports to Speculos containers?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) For security — random ports prevent unauthorized access</button>
<button class="quiz-choice" data-value="B">B) Because Speculos doesn't support port configuration</button>
<button class="quiz-choice" data-value="C">C) To match the behavior of real Ledger devices</button>
<button class="quiz-choice" data-value="D">D) To enable full parallel execution — each test worker gets its own container on a unique port, preventing conflicts</button>
</div>
<p class="quiz-explanation">Parallel test execution requires each test to have its own isolated Speculos instance. Random ports prevent port conflicts between concurrent tests.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q5.</strong> Trace the class hierarchy for <code>AccountPage</code> from bottom to top.</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>AccountPage</code> → <code>AppPage</code> → <code>Component</code> → <code>PageHolder</code></button>
<button class="quiz-choice" data-value="B">B) <code>AccountPage</code> → <code>PageHolder</code> → <code>Component</code> → <code>AppPage</code></button>
<button class="quiz-choice" data-value="C">C) <code>AccountPage</code> → <code>Component</code> → <code>AppPage</code></button>
<button class="quiz-choice" data-value="D">D) <code>AccountPage</code> → <code>Application</code> → <code>PageHolder</code></button>
</div>
<p class="quiz-explanation">The hierarchy is: <code>PageHolder</code> (holds Page) → <code>Component</code> (adds shared UI) → <code>AppPage</code> (abstract page) → <code>AccountPage</code> (concrete). The <code>Application</code> class is a separate aggregator, not part of the inheritance chain.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Writing Your First Desktop E2E Test

<div class="chapter-intro">
You have seen the architecture — now it is time to write code. This chapter walks through real tests from the codebase, explains each section line by line, shows data-driven patterns, and provides a template you can copy for your first test. By the end, you will have the muscle memory for the standard test structure.
</div>

### 14.1 Anatomy of a Real Test

Study this real test from `e2e/desktop/tests/specs/settings.spec.ts`:

```typescript
// 1. IMPORTS
import { test } from "tests/fixtures/common";           // Custom fixture (NOT @playwright/test)
import { Team } from "@ledgerhq/live-common/e2e/enum/Team";
import { addTmsLink } from "tests/utils/allureUtils";
import { getDescription } from "tests/utils/customJsonReporter";
import { Account } from "@ledgerhq/live-common/e2e/enum/Account";

// 2. DESCRIBE BLOCK
test.describe("Settings", () => {

  // 3. FIXTURE CONFIGURATION
  test.use({
    teamOwner: Team.WALLET_XP,
    userdata: "erc20-0-balance",
  });

  // 4. THE TEST
  test(
    "ERC20 token with 0 balance is hidden if 'hide empty token accounts' is ON",
    {
      tag: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
            "@ethereum", "@family-evm"],
      annotation: [{ type: "TMS", description: "B2CQA-817" }],
    },
    async ({ app }) => {
      await addTmsLink(getDescription(test.info().annotations, "TMS").split(", "));

      await app.mainNavigation.openTargetFromMainNavigation("accounts");
      await app.accounts.showParentAccountTokens(Account.ETH_1.accountName);
      await app.accounts.verifyTokenVisibility(
        Account.ETH_1.accountName,
        "Tether USD",
      );

      await app.mainNavigation.openSettings();
      await app.settings.goToAccountsTab();
      await app.settings.clickHideEmptyTokenAccountsToggle();

      await app.mainNavigation.openTargetFromMainNavigation("accounts");
      await app.accounts.verifyChildrenTokensAreNotVisible(
        Account.ETH_1.accountName,
        "Tether USD",
      );
    },
  );
});
```

**Key observations**:
1. **Import `test` from custom fixtures**, NOT from `@playwright/test` — this is critical
2. `test.use()` configures fixtures for all tests in the describe block
3. Tags indicate device compatibility and currency (used for CI filtering)
4. `addTmsLink` links the test to Jira/Xray
5. All interactions go through the `app` object and page object methods
6. No raw locators — every action is a named method on a page object

### 14.2 A Test With Speculos (Device Interaction)

When your test needs to interact with the Ledger device (signing transactions, verifying addresses), add `speculosApp` and `cliCommands` to the fixtures:

```typescript
test.describe("Password", () => {
  const account = Account.ETH_1;

  test.use({
    teamOwner: Team.WALLET_XP,
    userdata: "skip-onboarding-with-last-seen-device",
    cliCommands: [liveDataCommand(account)],          // Populate test data via CLI
    speculosApp: account.currency.speculosApp,         // Start Speculos with Ethereum app
  });

  test("The user enters password to access the app", {
    tag: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
          "@ethereum", "@family-evm"],
    annotation: { type: "TMS", description: "B2CQA-2343, B2CQA-1763, B2CQA-826" },
  }, async ({ app }) => {
    await addTmsLink(getDescription(test.info().annotations, "TMS").split(", "));

    await app.mainNavigation.openTargetFromMainNavigation("accounts");
    const countBeforeLock = await app.accounts.countAccounts();

    await app.mainNavigation.openSettings();
    await app.password.toggle();
    await app.password.enablePassword("SpeculosPassword", "SpeculosPassword");

    await app.settings.goToHelpTab();
    await app.settings.clearCache();

    await app.LockscreenPage.login("bad password");
    await app.LockscreenPage.checkInputErrorVisibility("visible");

    await app.LockscreenPage.login("SpeculosPassword");

    await app.mainNavigation.openTargetFromMainNavigation("accounts");
    const countAfterLock = await app.accounts.countAccounts();
    await app.accounts.compareAccountsCountFromJson(countBeforeLock, countAfterLock);
  });
});
```

Note: `speculosApp` and `cliCommands` trigger Speculos Docker start and CLI data population **automatically** before the test runs — the fixture system handles all of this.

### 14.3 Data-Driven Tests

Use JavaScript loops to generate multiple tests from data:

```typescript
const languageTestData = [
  { lang: "Francais", generalTabLabel: "General" },
  { lang: "Русский", generalTabLabel: "Общие" },
  { lang: "日本語", generalTabLabel: "一般" },
];

for (const l10n of languageTestData) {
  test(`Change language to ${l10n.lang}`, async ({ app }) => {
    await app.settings.changeLanguage(l10n.lang);
    await app.settings.expectGeneralTabLabel(l10n.generalTabLabel);
  });
}
// This generates 3 separate tests in the test runner!
```

This pattern is used extensively for testing multiple currencies, device models, and locales.

### 14.4 Template: Write Your Own Test

Copy this template as your starting point:

```typescript
import { test } from "tests/fixtures/common";
import { Team } from "@ledgerhq/live-common/e2e/enum/Team";
import { addTmsLink } from "tests/utils/allureUtils";
import { getDescription } from "tests/utils/customJsonReporter";

test.describe("YOUR FEATURE NAME", () => {
  test.use({
    teamOwner: Team.QAA,
    userdata: "skip-onboarding-with-last-seen-device",
    // speculosApp: account.currency.speculosApp,    // Uncomment if device needed
    // cliCommands: [liveDataCommand(account)],        // Uncomment for test data
    // featureFlags: { myFlag: { enabled: true } },    // Uncomment for FF overrides
  });

  test("YOUR TEST DESCRIPTION", {
    tag: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5"],
    annotation: { type: "TMS", description: "B2CQA-XXXX" },
  }, async ({ app }) => {
    await addTmsLink(getDescription(test.info().annotations, "TMS").split(", "));
    // YOUR TEST STEPS HERE
  });
});
```

### 14.5 The 4-Step Development Workflow

1. **Set up environment**: Export `MOCK=0`, `SPECULOS_DEVICE`, `COINAPPS`
2. **Build the app**: `pnpm build:lld:deps && pnpm desktop build:testing`
3. **Write and run your test**: `pnpm e2e:desktop test:playwright your-test.spec.ts`
4. **Debug and iterate**: Use `--ui` mode, check Allure report, fix and re-run

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://playwright.dev/docs/writing-tests">Playwright: Writing Tests</a></li>
<li><a href="https://playwright.dev/docs/test-parameterize">Playwright: Parameterizing Tests</a></li>
<li><a href="https://playwright.dev/docs/debug">Playwright: Debugging Tests</a></li>
<li><a href="https://playwright.dev/docs/test-ui-mode">Playwright: UI Mode</a> — interactive test debugging</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Every desktop E2E test follows the same pattern: import from custom fixtures, configure with <code>test.use()</code>, tag and annotate, interact through <code>app</code>. Once you have written one test, you have written them all — the variation is in the page objects you call, not the structure.
</div>

### 14.6 Quiz

<!-- ── Chapter 14 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">4 questions · 75% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> Why do we import <code>test</code> from <code>"tests/fixtures/common"</code> instead of <code>"@playwright/test"</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The Playwright package is not installed in the E2E project</button>
<button class="quiz-choice" data-value="B">B) The custom fixture extends Playwright's test with Speculos, Electron, page objects, and feature flag support</button>
<button class="quiz-choice" data-value="C">C) It is just a coding convention with no functional difference</button>
<button class="quiz-choice" data-value="D">D) The standard Playwright test doesn't support TypeScript</button>
</div>
<p class="quiz-explanation">The custom <code>test</code> from <code>fixtures/common.ts</code> extends Playwright's base test with additional fixtures: <code>app</code> (page objects), <code>electronApp</code>, <code>speculosApp</code>, <code>featureFlags</code>, <code>userdata</code>, and more. Using the standard import would miss all of this.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> What does <code>liveDataCommand(account)</code> do in the <code>cliCommands</code> array?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Downloads live price data from CoinGecko</button>
<button class="quiz-choice" data-value="B">B) Connects to a live blockchain node for real-time data</button>
<button class="quiz-choice" data-value="C">C) Uses the Ledger Live CLI to create the specified account and populate it with test data using the deterministic SEED</button>
<button class="quiz-choice" data-value="D">D) Starts a live recording of the test execution</button>
</div>
<p class="quiz-explanation"><code>liveDataCommand</code> generates a CLI command that derives the account from the test seed and populates it with balance and transaction history. This runs before the test starts, as part of the fixture setup.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> In a data-driven test that loops over 3 languages, how many separate tests are generated?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) 3 separate tests, each appearing independently in the test runner and report</button>
<button class="quiz-choice" data-value="B">B) 1 test that internally loops 3 times</button>
<button class="quiz-choice" data-value="C">C) 3 tests that share the same test result</button>
<button class="quiz-choice" data-value="D">D) It depends on the number of workers</button>
</div>
<p class="quiz-explanation">The <code>for</code> loop runs at test file parse time, calling <code>test()</code> once per iteration. Playwright registers each as an independent test with its own name, result, and retry behavior.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> What is the recommended way to debug a failing desktop E2E test locally?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Add <code>console.log</code> statements and re-run</button>
<button class="quiz-choice" data-value="B">B) Read the raw test output in the terminal</button>
<button class="quiz-choice" data-value="C">C) Use <code>test.only()</code> and increase all timeouts</button>
<button class="quiz-choice" data-value="D">D) Use Playwright UI mode (<code>--ui</code>) for interactive step-through, then check the Allure report for step traces and screenshots</button>
</div>
<p class="quiz-explanation">Playwright's UI mode lets you step through test execution interactively, inspect the DOM, and see network requests. Combined with Allure's step trace and failure screenshots, this is the most efficient debugging workflow.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Mobile E2E -- Architecture Deep Dive

<div class="chapter-intro">
The mobile E2E suite tests Ledger Live Mobile (React Native) using Detox + Jest. While the high-level patterns mirror the desktop suite — Page Object Model, Speculos emulation, feature flag overrides — the implementation differs significantly. This chapter maps the directory structure, the initialization flow, the WebSocket bridge, and the key architectural differences from desktop.
</div>

### 15.1 Directory Structure

```
e2e/mobile/
|-- jest.config.js                # Jest configuration
|-- detox.config.js               # Detox device/app configuration
|-- jest.environment.ts           # Custom Jest environment
|-- jest.globalSetup.ts           # Pre-test cleanup
|-- setup.ts                      # Runs before each test suite
|-- package.json
|
|-- specs/                        # TEST FILES (184 specs!)
|   |-- send/                     #   68 send flow tests
|   |-- swap/                     #   23 swap tests
|   |-- addAccount/               #   17 add account tests
|   |-- delegate/                 #   10 staking tests
|   |-- deleteAccount/            #   13 remove account tests
|   |-- deposit/                  #   3 receive tests
|   |-- earn/                     #   9 earn feature tests
|   |-- settings/                 #   6 settings tests
|   |-- portfolio/                #   4 portfolio tests
|   |-- verifyAddress/            #   12 address verification tests
|   +-- subAccount/               #   13 token sub-account tests
|
|-- page/                         # PAGE OBJECTS (34 files)
|   |-- index.ts                  #   Application singleton (global `app`)
|   |-- common.page.ts            #   Shared actions
|   |-- wallet/                   #   Wallet screens
|   |-- trade/                    #   Transaction screens
|   |-- settings/                 #   Settings screens
|   |-- accounts/                 #   Account management
|   +-- onboarding/               #   Onboarding flow
|
|-- helpers/                      # HELPER FUNCTIONS
|   |-- commonHelpers.ts          #   App launch, deeplinks
|   |-- elementHelpers.ts         #   Detox element wrappers (tapById, typeTextById, etc.)
|   +-- allure/allure-helper.ts   #   Allure metadata
|
|-- utils/                        # UTILITIES
|   |-- speculosUtils.ts          #   Speculos Docker management
|   |-- cliUtils.ts               #   CLI command wrappers
|   |-- initUtil.ts               #   Multi-phase initialization
|   +-- constants.ts              #   Shared constants
|
|-- bridge/
|   +-- server.ts                 #   WebSocket bridge
|
+-- models/
    +-- send.ts                   #   Transaction validation
```

### 15.2 The Initialization Flow

The mobile E2E suite has a 7-phase initialization:

```
1. GLOBAL SETUP
   +-- Validate .env.mock, install signal handlers, configure video recording

2. ENVIRONMENT SETUP
   +-- Initialize Detox environment, configure multi-worker devices, set globals

3. SUITE SETUP (beforeAll in jest.environment.ts)
   +-- Launch app with bridge, reverse TCP ports (Android), set Allure metadata

4. TEST INIT (app.init())
   |-- Launch Speculos Docker containers in parallel
   |-- Execute CLI commands with retry logic
   |-- Register Speculos ports via bridge
   |-- Load feature flag overrides via bridge
   +-- Import accounts via bridge

5. TEST RUNS (your test code)

6. SUITE TEARDOWN (afterAll)
   +-- Close bridge, stop Speculos containers

7. GLOBAL TEARDOWN
   +-- Docker cleanup, artifact collection
```

### 15.3 The WebSocket Bridge In Depth

The bridge is a WebSocket server running on the Jest side that the React Native app connects to:

```
[Jest + Detox]  <--- WebSocket --->  [React Native App]
     |                                      |
     |  importAccounts                      |  Receives accounts JSON
     |  overrideFeatureFlag                 |  Toggles feature flag
     |  navigate                            |  Changes screen
     |  getLogs                             |  Returns app logs
     |  addKnownSpeculos                    |  Registers Speculos device
     +  acceptTerms                         +  Skips onboarding
```

**Why is a bridge needed?** Detox can interact with native UI elements (tap, scroll, type), but it cannot directly access or manipulate React Native app state. The bridge is the channel for state manipulation — injecting accounts, overriding flags, navigating programmatically.

### 15.4 Why `app.init()` is in `beforeAll` (not `beforeEach`)

Desktop uses Playwright fixtures which automatically create fresh state per test (new userdata directory, new Electron process). Mobile cannot afford this — launching a native app, connecting the bridge, and populating data is expensive (10-30 seconds per setup).

So `app.init()` runs once per describe block in `beforeAll`. All tests in that block share the app instance. This is faster but means **tests in the same block are not fully isolated** — one test's side effects can affect the next.

**Mitigation**: Group tests that need the same setup in the same describe block. Tests that need different setups go in separate files.

### 15.5 Key Differences from Desktop (Summary)

| Aspect | Desktop | Mobile |
|--------|---------|--------|
| Framework | Playwright | Detox + Jest |
| App launch | `electron.launch()` | `device.launchApp()` |
| Fixture system | Per-test Playwright fixtures | Global `app.init()` in `beforeAll` |
| Page access | `({ app }) =>` destructured | `app.` global variable |
| Bridge | Not needed (Electron access) | WebSocket bridge required |
| Parallelism | Full parallel (random ports) | 1-3 workers (device limitation) |
| Build | Rspack bundle (~2 min) | Xcode/Gradle native (~10-30 min) |
| Test isolation | Full (fresh app per test) | Partial (shared app per describe) |

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://wix.github.io/Detox/docs/introduction/project-setup">Detox Project Setup</a></li>
<li><a href="https://wix.github.io/Detox/docs/guide/testing-in-ci">Detox: Testing in CI</a></li>
<li><a href="https://jestjs.io/docs/configuration">Jest Configuration</a></li>
<li><a href="https://reactnative.dev/docs/testing-overview">React Native Testing Overview</a></li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Mobile E2E testing has more constraints than desktop: slower builds, limited parallelism, and the WebSocket bridge as an indirection layer. But the patterns are similar — Page Object Model, Speculos for device emulation, feature flag overrides. Understanding the constraints helps you design better tests.
</div>

### 15.6 Quiz

<!-- ── Chapter 15 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> How many spec files does the mobile E2E suite have?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) 26 (same as desktop)</button>
<button class="quiz-choice" data-value="B">B) 73</button>
<button class="quiz-choice" data-value="C">C) 184</button>
<button class="quiz-choice" data-value="D">D) 250</button>
</div>
<p class="quiz-explanation">The mobile suite has 184 spec files covering send (68), swap (23), addAccount (17), delete (13), verify address (12), delegate (10), earn (9), settings (6), and more.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q2.</strong> Why is <code>app.init()</code> called in <code>beforeAll</code> instead of <code>beforeEach</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because Detox doesn't support <code>beforeEach</code></button>
<button class="quiz-choice" data-value="B">B) Because the tests don't need any setup</button>
<button class="quiz-choice" data-value="C">C) Because Jest requires it for parallel execution</button>
<button class="quiz-choice" data-value="D">D) Because launching a native app, connecting the bridge, and populating data is expensive (10-30s) — doing it per test would be prohibitively slow</button>
</div>
<p class="quiz-explanation">Unlike desktop (where Playwright fixtures quickly create fresh state), mobile app launch and bridge connection take 10-30 seconds. Running this per test would make the suite impractically slow.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q3.</strong> What is the consequence of mobile tests sharing the app instance within a <code>describe</code> block?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Tests run faster but can't use Speculos</button>
<button class="quiz-choice" data-value="B">B) Tests are not fully isolated — one test's side effects can affect the next</button>
<button class="quiz-choice" data-value="C">C) Tests must be run sequentially and can never be parallelized</button>
<button class="quiz-choice" data-value="D">D) Test results are merged into a single pass/fail outcome</button>
</div>
<p class="quiz-explanation">Shared state means a test that navigates to a different screen or changes a setting can affect subsequent tests. This is why test grouping (same setup = same describe) is critical in mobile E2E.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> What does the WebSocket bridge do that Detox cannot?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Manipulate React Native app state (inject accounts, override feature flags, navigate programmatically)</button>
<button class="quiz-choice" data-value="B">B) Tap on UI elements and scroll through lists</button>
<button class="quiz-choice" data-value="C">C) Take screenshots and record video</button>
<button class="quiz-choice" data-value="D">D) Launch the app and manage the device lifecycle</button>
</div>
<p class="quiz-explanation">Detox handles UI interaction (tap, type, scroll) and device management. The bridge handles state injection — things like loading accounts, setting feature flags, and navigating to specific screens that would be slow or impossible via UI interaction alone.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> Mobile E2E build times are significantly longer than desktop. What is the approximate range?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) ~30 seconds (Rspack bundle)</button>
<button class="quiz-choice" data-value="B">B) ~2 minutes (same as desktop)</button>
<button class="quiz-choice" data-value="C">C) ~10-30 minutes (Xcode/Gradle native build)</button>
<button class="quiz-choice" data-value="D">D) ~1 hour (full monorepo rebuild)</button>
</div>
<p class="quiz-explanation">Mobile builds require native compilation through Xcode (iOS) or Gradle (Android), which is significantly slower than desktop's Rspack JavaScript bundle. This is why mobile CI caches build artifacts aggressively.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Writing Your First Mobile E2E Test

<div class="chapter-intro">
Time to write mobile code. This chapter mirrors Chapter 14 but for the mobile platform. You will see real mobile test patterns, the helper functions that simplify Detox interactions, and how to translate between desktop and mobile test styles. By the end, you will have a template for your first mobile E2E test.
</div>

### 16.1 Simple Mobile Test (No Device)

```typescript
describe("Language Change", () => {
  beforeAll(async () => {
    await app.init({
      speculosApp: undefined,  // No device needed
      cliCommands: [],
      featureFlags: {},
    });
  });

  it("should change app language to French", async () => {
    await app.portfolio.navigateToSettings();
    await app.settings.openGeneralSettings();
    await app.settingsGeneral.changeLanguage("Francais");
    await expect(element(by.text("General"))).toBeVisible();
  });
});
```

### 16.2 Mobile Test With Speculos

```typescript
const transaction = {
  accountToDebit: Account.ETH_1,
  accountToCredit: Account.ETH_2,
  amount: "0.00001",
  speed: Fee.SLOW,
};

describe("Send ETH", () => {
  beforeAll(async () => {
    await app.init({
      speculosApp: transaction.accountToDebit.currency.speculosApp,
      cliCommands: [
        liveDataCommand(transaction.accountToDebit),
        getAddressCommand(transaction.accountToCredit),
      ],
      featureFlags: {},
    });
  });

  afterAll(async () => {
    await app.common.removeSpeculos();
  });

  it("should send ETH successfully", async () => {
    await runSendTest(transaction);
  });
});
```

### 16.3 Desktop to Mobile Translation

Here's a side-by-side comparison:

| Desktop (Playwright) | Mobile (Detox) |
|---------------------|----------------|
| `import { test } from "tests/fixtures/common"` | `import { app } from "../page"` (global) |
| `test.describe("...", () => {` | `describe("...", () => {` |
| `test.use({ userdata: "..." })` | `beforeAll(() => app.init({ ... }))` |
| `test("...", async ({ app }) => {` | `it("...", async () => {` |
| `await app.mainNavigation.openSettings()` | `await app.portfolio.navigateToSettings()` |
| `page.getByTestId("btn")` | `element(by.id("btn"))` |
| `page.getByTestId("btn").click()` | `element(by.id("btn")).tap()` |
| `page.getByTestId("input").fill("text")` | `element(by.id("input")).typeText("text")` |
| `expect(locator).toBeVisible()` | `await waitFor(element(by.id("x"))).toBeVisible().withTimeout(10000)` |

### 16.4 Template: Write Your Own Mobile Test

```typescript
import { Account } from "@ledgerhq/live-common/e2e/enum/Account";

describe("MY FEATURE NAME", () => {
  beforeAll(async () => {
    await app.init({
      speculosApp: undefined,       // or account.currency.speculosApp
      cliCommands: [],               // or [liveDataCommand(account)]
      featureFlags: {},              // or { myFlag: { enabled: true } }
    });
  });

  afterAll(async () => {
    await app.common.removeSpeculos();
  });

  it("should do something", async () => {
    await app.portfolio.waitForPortfolioPageToLoad();
    await app.portfolio.navigateToSettings();
    await waitForElementById("settings-screen", 10000);
  });
});
```

### 16.5 Helper Functions Reference

The most commonly used helpers from `e2e/mobile/helpers/elementHelpers.ts`:

```typescript
await tapById("send-button");                    // Tap by test ID
await tapByText("Confirm");                      // Tap by visible text
await typeTextById("amount-input", "0.001");     // Type into a text field
await waitForElementById("success-msg", 30000);  // Wait for element
await scrollToId("account-row-bitcoin");         // Scroll until element visible
const text = await getTextOfElement("balance");  // Read text content
```

These wrappers reduce the verbosity of raw Detox API calls and provide consistent timeout handling.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://wix.github.io/Detox/docs/api/actions-on-element">Detox Actions API</a></li>
<li><a href="https://wix.github.io/Detox/docs/api/expect">Detox Expect API</a></li>
<li><a href="https://wix.github.io/Detox/docs/api/matchers">Detox Matchers</a></li>
<li><a href="https://wix.github.io/Detox/docs/troubleshooting/running-tests">Detox Troubleshooting</a></li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Mobile tests follow the same logical structure as desktop — describe block, setup, test steps, teardown. The differences are in the APIs (Detox vs Playwright) and the lifecycle (<code>app.init()</code> in <code>beforeAll</code> vs per-test fixtures). Use the helper functions to keep your tests concise and readable.
</div>

### 16.6 Quiz

<!-- ── Chapter 16 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">4 questions · 75% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> Translate this Playwright code to Detox: <code>await page.getByRole("button", { name: "Send" }).click();</code></p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>await element(by.id("Send")).click();</code></button>
<button class="quiz-choice" data-value="B">B) <code>await device.tap("Send");</code></button>
<button class="quiz-choice" data-value="C">C) <code>await element(by.text("Send")).tap();</code></button>
<button class="quiz-choice" data-value="D">D) <code>await tapByRole("button", "Send");</code></button>
</div>
<p class="quiz-explanation">Detox doesn't have role-based matchers like Playwright. The closest equivalent for a button with text "Send" is <code>element(by.text("Send")).tap()</code>. In Detox, <code>click()</code> is called <code>tap()</code>.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> What does <code>await app.common.removeSpeculos()</code> do in <code>afterAll</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Uninstalls the Speculos npm package</button>
<button class="quiz-choice" data-value="B">B) Stops and removes the Speculos Docker container used by the test</button>
<button class="quiz-choice" data-value="C">C) Deletes the Speculos configuration file</button>
<button class="quiz-choice" data-value="D">D) Disconnects the WebSocket bridge</button>
</div>
<p class="quiz-explanation">After all tests in a describe block finish, <code>removeSpeculos()</code> stops the Docker container that was started in <code>app.init()</code>. This prevents Docker container leaks between test suites.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> Why do mobile E2E tests use explicit timeouts like <code>waitFor(...).withTimeout(10000)</code> while desktop tests usually don't?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Mobile tests are written in a different language</button>
<button class="quiz-choice" data-value="B">B) Detox doesn't support any form of auto-waiting</button>
<button class="quiz-choice" data-value="C">C) Mobile apps are always faster than desktop apps</button>
<button class="quiz-choice" data-value="D">D) Playwright has built-in auto-wait (retries until element appears), while Detox requires explicit <code>waitFor().withTimeout()</code> for reliable waiting</button>
</div>
<p class="quiz-explanation">Playwright auto-waits by default — actions like <code>click()</code> automatically wait for the element to be visible, stable, and enabled. Detox does not have this built-in behavior, so you must use <code>waitFor</code> with explicit timeouts.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> What is the Detox equivalent of Playwright's <code>page.getByTestId("amount").fill("0.5")</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>await element(by.id("amount")).typeText("0.5");</code></button>
<button class="quiz-choice" data-value="B">B) <code>await element(by.id("amount")).fill("0.5");</code></button>
<button class="quiz-choice" data-value="C">C) <code>await typeTextById("amount", "0.5");</code> (either answer is correct)</button>
<button class="quiz-choice" data-value="D">D) Both A and C are correct</button>
</div>
<p class="quiz-explanation">Both <code>element(by.id("amount")).typeText("0.5")</code> (raw Detox) and <code>typeTextById("amount", "0.5")</code> (helper wrapper) are correct. The helper is preferred for consistency and built-in error handling.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Part 3 Final Assessment

<div class="quiz-container" data-pass-threshold="80">
<h3>Part 3 Final Assessment</h3>
<p class="quiz-subtitle">10 questions across all chapters · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> The desktop E2E class hierarchy is:</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>AppPage</code> → <code>PageHolder</code> → <code>Component</code></button>
<button class="quiz-choice" data-value="B">B) <code>PageHolder</code> → <code>Component</code> → <code>AppPage</code> → SpecificPage</button>
<button class="quiz-choice" data-value="C">C) <code>Component</code> → <code>AppPage</code> → <code>PageHolder</code></button>
<button class="quiz-choice" data-value="D">D) SpecificPage → <code>Component</code> → <code>PageHolder</code></button>
</div>
<p class="quiz-explanation"><code>PageHolder</code> (holds the Page reference) → <code>Component</code> (adds shared UI elements) → <code>AppPage</code> (abstract page class) → SpecificPage (your concrete page objects like AccountPage, SettingsPage, etc.).</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> What is the purpose of userdata JSON profiles?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Store user credentials for authentication</button>
<button class="quiz-choice" data-value="B">B) Configure CI workflow parameters</button>
<button class="quiz-choice" data-value="C">C) Provide pre-configured app state (accounts, settings, onboarding) so tests start from a known state without manual setup</button>
<button class="quiz-choice" data-value="D">D) Define feature flag defaults</button>
</div>
<p class="quiz-explanation">Userdata profiles contain serialized app state so tests don't need to go through onboarding, account creation, or settings configuration. This makes tests faster and more deterministic.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q3.</strong> How does the mobile E2E bridge communicate?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) HTTP REST API</button>
<button class="quiz-choice" data-value="B">B) WebSocket</button>
<button class="quiz-choice" data-value="C">C) gRPC</button>
<button class="quiz-choice" data-value="D">D) Shared file system</button>
</div>
<p class="quiz-explanation">The bridge runs a WebSocket server (in <code>e2e/mobile/bridge/server.ts</code>) that the React Native app connects to for receiving test commands like <code>importAccounts</code> and <code>overrideFeatureFlag</code>.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> Which of these is NOT captured on test failure in the desktop suite?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) UI screenshot</button>
<button class="quiz-choice" data-value="B">B) Video recording</button>
<button class="quiz-choice" data-value="C">C) Speculos device screenshot</button>
<button class="quiz-choice" data-value="D">D) Database dump</button>
</div>
<p class="quiz-explanation">There is no database — Ledger Live uses JSON files for local state. On failure, the suite captures: UI screenshot, video recording, Electron console logs, and Speculos device screenshot.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q5.</strong> Why do we need 30+ page objects in the Application class?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Each page object encapsulates one screen's locators and actions — this keeps tests readable, centralizes locator changes, and enables the <code>@step</code> decorator for Allure</button>
<button class="quiz-choice" data-value="B">B) Playwright requires one page object per test file</button>
<button class="quiz-choice" data-value="C">C) Each page object corresponds to one development team</button>
<button class="quiz-choice" data-value="D">D) It's an over-engineered design that should be simplified</button>
</div>
<p class="quiz-explanation">Page objects enable: readable tests (call <code>app.settings.changeLanguage()</code> instead of raw locators), single-point-of-change (update a test ID in one place), and automatic Allure step traces via <code>@step</code>. The Application class aggregates them for convenient <code>app.</code> access.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> What does <code>pnpm e2e:desktop test:playwright -- --shard=2/4</code> do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Runs only the second test in the suite</button>
<button class="quiz-choice" data-value="B">B) Runs tests at 2x speed with 4 workers</button>
<button class="quiz-choice" data-value="C">C) Splits the test suite into 4 parts and runs the 2nd part — used for parallel CI execution</button>
<button class="quiz-choice" data-value="D">D) Runs the test 4 times and keeps the 2nd result</button>
</div>
<p class="quiz-explanation">Playwright sharding splits the entire test suite across N runners. <code>--shard=2/4</code> means "I am runner 2 of 4, give me my portion of the tests." Each shard runs independently and produces its own results, which are merged later.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q7.</strong> In the mobile test template, why is <code>speculosApp: undefined</code> valid?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It means "use the default Speculos app"</button>
<button class="quiz-choice" data-value="B">B) It means "this test doesn't need a Speculos device" — no Docker container will be started</button>
<button class="quiz-choice" data-value="C">C) It's a TypeScript error that should be fixed</button>
<button class="quiz-choice" data-value="D">D) It means "use a mock device instead of Speculos"</button>
</div>
<p class="quiz-explanation">Setting <code>speculosApp</code> to <code>undefined</code> tells the initialization to skip Speculos setup entirely. Tests that don't involve device interaction (settings, UI navigation) don't need it.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q8.</strong> What is the key difference in test isolation between desktop and mobile?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Desktop tests share state, mobile tests are isolated</button>
<button class="quiz-choice" data-value="B">B) Both platforms provide full test isolation</button>
<button class="quiz-choice" data-value="C">C) Neither platform provides test isolation</button>
<button class="quiz-choice" data-value="D">D) Desktop has full isolation (fresh app per test via fixtures), mobile has partial isolation (shared app per describe block via <code>beforeAll</code>)</button>
</div>
<p class="quiz-explanation">Playwright fixtures create a fresh Electron instance, userdata directory, and page for every test. Detox shares the app instance within a describe block because native app launch is too slow to repeat per test.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q9.</strong> You want to write a desktop E2E test for a new staking feature behind a feature flag. Which fixture properties do you need?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>userdata</code>, <code>speculosApp</code>, <code>cliCommands</code>, <code>featureFlags</code>, <code>teamOwner</code></button>
<button class="quiz-choice" data-value="B">B) Only <code>userdata</code> — everything else is auto-detected</button>
<button class="quiz-choice" data-value="C">C) <code>speculosApp</code> and <code>featureFlags</code> — the rest is optional</button>
<button class="quiz-choice" data-value="D">D) You don't use fixtures for staking tests</button>
</div>
<p class="quiz-explanation">A staking test needs: <code>userdata</code> (pre-configured state with accounts), <code>speculosApp</code> (device for signing), <code>cliCommands</code> (populate staking account data), <code>featureFlags</code> (enable the new staking flag), and <code>teamOwner</code> (for reporting).</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q10.</strong> A mobile test that worked yesterday now fails with "Element not found: staking-button" after another team's PR was merged. What should you investigate first?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Check if the Detox version was updated</button>
<button class="quiz-choice" data-value="B">B) Rebuild the app from scratch</button>
<button class="quiz-choice" data-value="C">C) Check if a feature flag or test ID was changed in the merged PR, and verify the element's <code>testID</code> prop still matches</button>
<button class="quiz-choice" data-value="D">D) Increase the test timeout</button>
</div>
<p class="quiz-explanation">Another team's PR could have: changed the <code>testID</code> prop on the element, modified a feature flag that controls the staking UI visibility, or restructured the component tree. Check the PR diff for changes to the staking screen and its feature flags.</p>
</div>

<div class="quiz-score"></div>
</div>

---

