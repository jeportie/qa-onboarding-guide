## Mobile E2E -- Architecture Deep Dive

<div class="chapter-intro">
The mobile E2E suite tests Ledger Live Mobile (React Native) using Detox + Jest. While the high-level patterns mirror the desktop suite — Page Object Model, Speculos emulation, feature flag overrides — the implementation differs significantly. This chapter maps the directory structure, the initialization flow, the WebSocket bridge, and the key architectural differences from desktop.
</div>

### 5.1.1 Directory Structure

Mobile E2E lives in a **peer workspace** at the repo root: `e2e/mobile/` sits alongside `e2e/desktop/`, each with its own `package.json`, Detox/Playwright config, and Nx `project.json`. The mobile workspace is fully self-contained — it is no longer nested under `apps/ledger-live-mobile/e2e/`.

```
e2e/mobile/                        # peer to e2e/desktop/
|-- package.json                   # own workspace package (ledger-live-mobile-e2e-tests)
|-- project.json                   # Nx target definitions
|-- detox.config.js                # Detox device/app configuration
|-- jest.config.js                 # Jest configuration
|-- jest.environment.ts            # Custom Jest environment (per-worker device swap)
|-- jest.globalSetup.ts            # Pre-test setup (delegates to Detox)
|-- jest.globalTeardown.ts         # Post-run cleanup
|-- babel.config.js                # Babel for the test transform
|-- tsconfig.json             # TS config for specs (experimentalDecorators)
|
|-- specs/                         # TEST FILES (~213 specs)
|   |-- account/                   #   Account screen tests
|   |-- addAccount/                #   Add-account flow tests
|   |-- buySell/                   #   Buy / sell flow tests
|   |-- delegate/                  #   Staking tests
|   |-- deleteAccount/             #   Remove account tests
|   |-- deposit/                   #   Receive tests
|   |-- earn/                      #   Earn feature tests
|   |-- ledgerSync/                #   Ledger Sync tests
|   |-- portfolio/                 #   Portfolio tests
|   |-- send/                      #   Send flow tests
|   |-- settings/                  #   Settings tests
|   |-- subAccount/                #   Token sub-account tests
|   |-- swap/                      #   Swap tests (incl. otherTestCases/)
|   |-- verifyAddress/             #   Address verification tests
|   +-- wallet40/                  #   Wallet 4.0 tests
|
|-- page/                          # PAGE OBJECTS
|   |-- index.ts                   #   `Application` class (lazy-init POM hub, 32 getters)
|   |-- common.page.ts             #   Shared actions
|   |-- passwordEntry.page.ts      #   Password entry
|   |-- speculos.page.ts           #   Speculos-device POM
|   |-- error.page.ts              #   Error screen POM
|   |-- accounts/                  #   Account management screens
|   |-- discover/                  #   Discover screens
|   |-- drawer/                    #   Modular drawers
|   |-- liveApps/                  #   Live-app screens (e.g. swap live app)
|   |-- manager/                   #   Manager screens
|   |-- market/                    #   Market screens
|   |-- onboarding/                #   Onboarding flow
|   |-- settings/                  #   Settings screens
|   |-- stax/                      #   Stax-specific screens
|   |-- trade/                     #   Send / receive / swap / stake / earn
|   +-- wallet/                    #   Portfolio and wallet navigators
|
|-- bridge/
|   |-- server.ts                  #   WebSocket bridge server (~307 lines)
|   +-- types.ts                   #   Canonical bridge message types
|
|-- helpers/                       # HELPER FUNCTIONS
|-- models/                        # Test models (transactions, etc.)
|-- userdata/                      # JSON fixtures (accounts, settings)
|-- scripts/                       # Workspace scripts (typecheck, etc.)
|-- types/                         # Shared TS types
+-- utils/                         # Utilities (initUtil, speculos, cli, constants)
```

> **Legacy residue.** A small set of specs still lives under `apps/ledger-live-mobile/e2e/` (roughly 17 files: onboarding, password, deeplinks, language, manager, market, plus a few send/receive/delegate/swap specs). They are being migrated; treat `e2e/mobile/` as the canonical workspace for new work.

### 5.1.2 The Initialization Flow

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

### 5.1.3 The WebSocket Bridge In Depth

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

### 5.1.4 Why `app.init()` is in `beforeAll` (not `beforeEach`)

Desktop uses Playwright fixtures which automatically create fresh state per test (new userdata directory, new Electron process). Mobile cannot afford this — launching a native app, connecting the bridge, and populating data is expensive (10-30 seconds per setup).

So `app.init()` runs once per describe block in `beforeAll`. All tests in that block share the app instance. This is faster but means **tests in the same block are not fully isolated** — one test's side effects can affect the next.

**Mitigation**: Group tests that need the same setup in the same describe block. Tests that need different setups go in separate files.

### 5.1.5 Key Differences from Desktop (Summary)

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

### 5.1.6 Quiz

<!-- ── Chapter 5.1 Quiz ── -->

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
Time to write mobile code. This chapter mirrors Chapter 4.2 but for the mobile platform. You will see real mobile test patterns, the helper functions that simplify Detox interactions, and how to translate between desktop and mobile test styles. By the end, you will have a template for your first mobile E2E test.
</div>

### 5.2.1 Simple Mobile Test (No Device)

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

### 5.2.2 Mobile Test With Speculos

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

### 5.2.3 Desktop to Mobile Translation

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

### 5.2.4 Template: Write Your Own Mobile Test

New specs live under `e2e/mobile/specs/<area>/<name>.spec.ts` (e.g. `e2e/mobile/specs/settings/myFeature.spec.ts`). Shared test enums (`Account`, `Provider`, `Addresses`, `Fee`, …) live in `@ledgerhq/live-common/e2e/enum/...`; local page objects, helpers, and utils use relative imports inside `e2e/mobile/`.

```typescript
// e2e/mobile/specs/<area>/myFeature.spec.ts
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

The `beforeAll(() => app.init({...}))` pattern above is the one most specs use directly (see `e2e/mobile/specs/languageChange.spec.ts` for a canonical example).

**Alternative idiom — thin spec + shared driver.** Several feature areas (notably `specs/swap/`) have moved to a pattern where the `.spec.ts` file is a short data bundle that delegates to a `runXxx(...)` driver in a sibling `*.other.ts` / `*.ts` module. The driver owns `describe()`, `beforeAll()`, and `it()`. This keeps dozens of closely-related scenarios DRY. You'll see this in the QAA-702 walkthrough (Chapter 5.10).

### 5.2.5 Helper Functions Reference

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

### 5.2.6 Quiz

<!-- ── Chapter 5.2 Quiz ── -->

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


## React Native Primer for TypeScript Engineers

<div class="chapter-intro">
If you already know TypeScript and a bit of React, you are 70% of the way to understanding React Native. The remaining 30% is about what happens under the hood — how JavaScript ends up driving a native iOS or Android UI. This chapter explains that gap so that, by the time we introduce Detox, you understand exactly <em>what</em> Detox is automating and <em>why</em> the <code>testID</code> prop is the single most important line of code a QA engineer will ever add to a component.
</div>

### 5.3.1 What Is React Native?

**React Native (RN)** is a framework from Meta that lets you write native mobile apps using TypeScript/JavaScript and React. The "native" part is not marketing — your `<View>` in JSX really does become a `UIView` on iOS and an `android.view.ViewGroup` on Android. It is **not** a web browser, not a webview, not a PWA.

**Key facts:**
- Language: TypeScript/JavaScript + React component model
- Created by: Meta (Facebook), 2015
- What it is: a JavaScript runtime + a bridge to the host platform's native UI toolkit
- Used by: Meta, Shopify, Discord, Microsoft Teams, and **Ledger Live Mobile**
- Not to be confused with: Cordova, Ionic, Capacitor (all of which render inside a `WebView`)

#### The bridge model (Old Architecture) vs the New Architecture

Classic RN apps run on two threads:

- **JS thread** — executes your bundled JavaScript in a JavaScript engine (Hermes by default since RN 0.70). This is where React reconciliation, your `useState`, your business logic, and your Redux store all live.
- **UI thread (main thread)** — the platform's native thread. Draws the actual pixels. On iOS this is driven by UIKit, on Android by the Android View system (or Jetpack Compose for newer widgets).

These two threads communicate through a **bridge** (old arch) or a **JSI + TurboModules** layer (new arch):

```
+---------------------------+          +-------------------------------+
|      JS Thread (Hermes)   |          |   UI / Main Thread (Native)   |
|  ----------------------   |  bridge  |   --------------------------  |
|   React tree              | <======> |   UIView tree (iOS)           |
|   setState / reconciler   |  (JSON   |   ViewGroup tree (Android)    |
|   Your TypeScript code    |  or JSI) |   Native modules (Camera,     |
|   Redux / Zustand state   |          |   USB, Bluetooth, Keychain)   |
+---------------------------+          +-------------------------------+
              ^                                        ^
              |                                        |
              | (in dev) Metro bundler serves JS       | (always) platform SDK
              | over http://localhost:8081             | drives real pixels
```

Every time your JS calls `setState`, RN diffs the virtual tree and sends a batch of "mutate this view, set this prop" instructions across the bridge. The UI thread applies them.

> **Note:** The New Architecture (JSI, Fabric, TurboModules) replaces JSON serialization over the bridge with direct C++ function calls exposed to JS. From a QA perspective the mental model is the same — there is still a JS side and a native side — but it is faster and the error messages look a bit different.

#### React Native vs web React

| Aspect | Web React | React Native |
|--------|-----------|--------------|
| Rendering target | DOM (`<div>`, `<span>`) | Native views (`UIView`, `ViewGroup`) |
| Styling | CSS, class names | `StyleSheet` objects (a JS subset of CSS) |
| Layout engine | Browser's CSS engine | Yoga (Flexbox-only, no block/inline) |
| Routing | `react-router`, `next/router` | `react-navigation` |
| File loading | Webpack / Vite | Metro |
| DOM APIs | `document.querySelector` | None — there is no DOM |
| Events | `onClick`, `onChange` | `onPress`, `onChangeText` |

> **Gotcha:** There is **no DOM**. `document` does not exist. `window` exists but is mostly empty. If a library assumes a browser, it does not work in RN.

#### React Native vs Electron vs Flutter

- **Electron** (what Desktop Ledger Live uses): ships Chromium + Node.js. Every UI element is an HTML element. No native views.
- **Flutter**: ships its own rendering engine (Skia). It does *not* use native views — it paints pixels itself. Fast, but "looks native" rather than "is native".
- **React Native**: uses the host platform's actual native views. When you put a `<Text>` on screen, it is a real `UILabel` / `TextView`. This is why testing frameworks like Detox can find it by the platform's accessibility ID.

### 5.3.2 Core Primitives

Instead of HTML elements, RN exposes a small set of built-in components. These are the only ones you need to read 90% of Ledger Live Mobile source.

#### `View` — the `<div>` of React Native

```tsx
import { View, StyleSheet } from "react-native";

export function Card(): JSX.Element {
  return (
    <View style={styles.card}>
      {/* children */}
    </View>
  );
}

const styles = StyleSheet.create({
  card: {
    padding: 16,
    borderRadius: 8,
    backgroundColor: "#fff",
  },
});
```

A `View` is a flex container. By default `flexDirection` is `column` (not `row` like on the web) and every `View` is itself a flex container (no need to set `display: flex`).

#### `Text` — **the only** way to put text on screen

```tsx
import { Text } from "react-native";

<Text style={{ fontSize: 16 }}>Hello, Ledger</Text>;
```

> **Gotcha:** You cannot put a string directly inside a `View`. `<View>Hello</View>` throws at runtime. Every visible string must be wrapped in `<Text>`. This trips up almost every new RN developer.

#### `Pressable` — the modern tap handler

```tsx
import { Pressable, Text } from "react-native";

<Pressable
  testID="portfolio-add-account-button"
  onPress={() => handleAdd()}
  style={({ pressed }) => [styles.btn, pressed && styles.btnPressed]}
>
  <Text>Add account</Text>
</Pressable>;
```

`Pressable` is the recommended way to make anything tappable. It exposes the `pressed` state and supports `onPressIn`, `onPressOut`, `onLongPress`, and hit-slop configuration. See [`reactnative.dev/docs/pressable`](https://reactnative.dev/docs/pressable).

#### `TouchableOpacity` — the older, still-everywhere alternative

```tsx
import { TouchableOpacity, Text } from "react-native";

<TouchableOpacity testID="continue-button" onPress={onContinue}>
  <Text>Continue</Text>
</TouchableOpacity>;
```

`TouchableOpacity` fades the button when pressed. It predates `Pressable`. Ledger Live Mobile still uses it extensively in legacy screens. Functionally equivalent for QA purposes.

#### `ScrollView` — scrolls any children

```tsx
import { ScrollView } from "react-native";

<ScrollView contentContainerStyle={{ padding: 16 }}>
  <LongForm />
</ScrollView>;
```

Use `ScrollView` for a **small, bounded** list of items. It renders every child immediately — bad for 500 accounts.

#### `FlatList` — virtualized lists

```tsx
import { FlatList } from "react-native";

type Account = { id: string; name: string };

<FlatList<Account>
  data={accounts}
  keyExtractor={a => a.id}
  renderItem={({ item }) => (
    <AccountRow testID={`account-row-${item.id}`} account={item} />
  )}
/>;
```

`FlatList` only renders the rows currently on screen — essential for the accounts list, operations history, market cap list, etc. See [`reactnative.dev/docs/flatlist`](https://reactnative.dev/docs/flatlist).

> **Gotcha:** If you set `testID` on the `FlatList` itself, it lands on the outer scroll container, not the rows. Row `testID`s must be set **inside `renderItem`** — on the row component — and the row component must actually forward the prop to its top-level `View`/`Pressable`. See 32.3 for why this matters.

#### Other primitives you will see

- `TextInput` — the native text field (`onChangeText`, not `onChange`).
- `Image` — renders a `UIImageView` / `ImageView`. Source is `{ uri: "https://..." }` or `require("./logo.png")`.
- `SafeAreaView` — pads children to avoid the notch / home indicator.
- `Modal` — renders a native modal on top of the app window.
- `ActivityIndicator` — the spinning wheel (`UIActivityIndicator` / `ProgressBar`).

> **Further reading:** [`reactnative.dev/docs/components-and-apis`](https://reactnative.dev/docs/components-and-apis) lists every built-in component and API.

### 5.3.3 The `testID` Prop — Your Best Friend

`testID` is the prop you will encounter, add, and debate more than any other. **It is the single most important prop for mobile QA.**

```tsx
<Pressable testID="portfolio-add-account-button" onPress={handleAdd}>
  <Text>Add account</Text>
</Pressable>;
```

#### What `testID` becomes on each platform

React Native translates `testID` into the platform's accessibility identifier:

| Platform | Native attribute | Queried by |
|----------|------------------|------------|
| iOS      | `accessibilityIdentifier` on the backing `UIView` | XCUITest, Detox `by.id()` |
| Android  | `resource-id` (tag on the `View`) | UIAutomator / Espresso, Detox `by.id()` |

Detox's `by.id("portfolio-add-account-button")` resolves to `accessibilityIdentifier == "portfolio-add-account-button"` on iOS and `resource-id == "portfolio-add-account-button"` on Android. See [`reactnative.dev/docs/view#testid`](https://reactnative.dev/docs/view#testid).

#### Why not use text or accessibility labels?

Same reason as desktop: text changes with i18n, layout, and copy reviews. `testID` is added **for tests**, stays stable across translations, and does not affect the rendered UI.

#### Common gotchas

1. **`testID` must be on a real view** — not on a `React.Fragment`, not on a `<>`, not on a pure JS object. Some components (like `react-native-svg`'s `<Svg>`) accept it but do not always propagate it to the backing view.

2. **Row-level `testID` requires forwarding**. If you write a reusable component:

   ```tsx
   type Props = { testID?: string; label: string };
   export function MyRow({ testID, label }: Props) {
     return (
       <View testID={testID}>        {/* <- must forward! */}
         <Text>{label}</Text>
       </View>
     );
   }
   ```

   If you forget `testID={testID}` on the outer `View`, Detox will never find the row even though the prop is set on the parent.

3. **Duplicate `testID`s** break Detox with an ambiguous-match error. Make them unique with a stable suffix: `` `account-row-${accountId}` ``.

4. **`FlatList` cells**. The outer `FlatList` is a scroll container; only rows rendered inside `renderItem` carry row-level `testID`s. To wait for a specific row, scroll the list first with Detox's `scrollTo()`.

5. **Dynamic vs static**. Prefer `testID={currency}` over `testID={\`row-${isSelected ? "sel" : "idle"}\`}` — a `testID` that changes on interaction is a flaky test waiting to happen.

> **Note:** In Ledger Live Mobile, `testID` values are kebab-case and read like a breadcrumb: `settings-general-language-row`, `send-recipient-input`, `modal-confirm-button`. Follow the existing convention when adding new ones.

### 5.3.4 Platform Module & Cross-Platform Branching

A single RN codebase produces two native apps. Sometimes you need to branch per platform:

```tsx
import { Platform } from "react-native";

if (Platform.OS === "ios") {
  // iOS-specific logic
}

// Or, cleaner:
const headerPaddingTop = Platform.select({
  ios: 20,    // under the notch
  android: 0, // status bar is handled by the system
  default: 0,
});
```

`Platform.OS` is `"ios" | "android" | "windows" | "macos" | "web"`. For Ledger Live it is always `"ios"` or `"android"`. See [`reactnative.dev/docs/platform`](https://reactnative.dev/docs/platform).

#### Why E2E tests often branch on platform

Detox exposes `device.getPlatform()` so tests can branch at runtime. Patterns you will see in Ledger Live Mobile E2E:

```ts
export const isAndroid = (): boolean => device.getPlatform() === "android";
export const isIos = (): boolean => device.getPlatform() === "ios";
```

Why? A handful of reasons recur:

- **Keyboard**: Android auto-dismisses the keyboard on `Enter`; iOS does not. Tests may need `device.pressBack()` on Android only.
- **Permissions dialogs**: On iOS, push/biometric dialogs are system alerts handled by the iOS alert API; on Android they are shown by the app itself.
- **Scrolling behavior**: iOS supports bounce/over-scroll; Android does not. Some tests calibrate pixel offsets.
- **File/attachment picker**: completely different implementations.
- **Back button**: only Android has a hardware-equivalent back button.

> **Gotcha:** Avoid scattering `if (isAndroid())` through tests. Put the branching inside a page-object method so tests stay clean.

### 5.3.5 Metro — The JavaScript Bundler

Metro is RN's dedicated bundler. It exists because Webpack was too heavy and too web-centric when RN was created.

**What Metro does:**
- Walks your `import` graph starting from `index.js`
- Transpiles TypeScript/JSX with Babel
- Produces a **single** JavaScript bundle (no code-splitting by default)
- Serves the bundle over HTTP at `http://localhost:8081/index.bundle`
- Hot-reloads modules in dev mode

#### Debug vs release builds

| Mode | Who serves JS? | When used |
|------|----------------|-----------|
| **Debug** | Metro dev server on port 8081 | Local development, Detox `*.debug` configs |
| **Release** | JS is bundled into the `.ipa` / `.apk` at build time | TestFlight, Play Store, Detox `*.release` configs |

A debug build loads the JS over HTTP each time the app starts. A release build has the JS baked in as a static file inside the app bundle. This is why you can edit code and reload without rebuilding in debug, but not in release.

> **Note:** For E2E we use both. Debug builds iterate fast locally. Release builds are what CI runs to catch bundler-only issues (dead-code elimination, Hermes bytecode). See [`reactnative.dev/docs/metro`](https://reactnative.dev/docs/metro).

### 5.3.6 Deep Links & the `Linking` API

A **deep link** is a URL that opens a specific screen inside your app. Ledger Live Mobile uses the `ledgerlive://` scheme.

```
ledgerlive://portfolio
ledgerlive://account?currency=bitcoin
ledgerlive://wc?uri=wc:abc123...
ledgerlive://discover/swap
```

When the OS sees a URL with a registered scheme, it launches the app and hands the URL to it. In RN you subscribe with the `Linking` module:

```tsx
import { Linking } from "react-native";

useEffect(() => {
  const sub = Linking.addEventListener("url", ({ url }) => {
    navigateFromDeeplink(url);
  });
  return () => sub.remove();
}, []);
```

See [`reactnative.dev/docs/linking`](https://reactnative.dev/docs/linking).

#### How deep links are used in E2E

Deep links are the fastest way to jump a test to a specific screen without tapping through navigation. Detox exposes `device.openURL()`, and Ledger Live Mobile wraps it in an `openDeeplink()` helper:

```ts
await openDeeplink("ledgerlive://account?currency=bitcoin");
```

This skips onboarding, the portfolio tab, the account list — straight to the Bitcoin account screen. Huge time saver and fewer flaky steps.

### 5.3.7 Dev Menu & React DevTools

When you run a debug build on a simulator/emulator, you can open the **dev menu**:

- iOS simulator: **Device → Shake** or **⌘D**
- Android emulator: **⌘M** (macOS) / **Ctrl+M**

From here you can reload JS, enable/disable Fast Refresh, toggle the performance overlay, and open the in-app element inspector.

The **element inspector** is the RN equivalent of the web's DOM inspector: tap a component on screen and the menu shows its props, source location, and accessibility info. This is how you discover which component owns which `testID` — critical when designing new E2E tests.

**React DevTools** (a standalone app: `npx react-devtools`) attaches over the network and shows the full component tree. Use it when designing page objects to understand the rendered hierarchy. See [`reactnative.dev/docs/debugging`](https://reactnative.dev/docs/debugging) and [`reactnative.dev/docs/react-devtools`](https://reactnative.dev/docs/react-devtools).

> **Gotcha:** The dev menu is only available in debug builds. Release builds strip it.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://reactnative.dev/">React Native</a> — official landing page</li>
<li><a href="https://reactnative.dev/docs/components-and-apis">Core Components and APIs</a> — every built-in primitive</li>
<li><a href="https://reactnative.dev/docs/view#testid">View: testID prop</a> — how testID maps to native platform attributes</li>
<li><a href="https://reactnative.dev/docs/pressable">Pressable</a> — modern touchable component</li>
<li><a href="https://reactnative.dev/docs/flatlist">FlatList</a> — virtualized list rendering</li>
<li><a href="https://reactnative.dev/docs/platform">Platform</a> — detect iOS vs Android at runtime</li>
<li><a href="https://reactnative.dev/docs/metro">Metro</a> — the RN bundler</li>
<li><a href="https://reactnative.dev/docs/linking">Linking</a> — deep linking API</li>
<li><a href="https://reactnative.dev/docs/debugging">Debugging</a> — dev menu and inspectors</li>
<li><a href="https://reactnative.dev/docs/react-devtools">React DevTools</a> — component tree inspection</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> React Native runs your TypeScript on a JS thread and drives real native views on the UI thread via a bridge. The <code>testID</code> prop becomes <code>accessibilityIdentifier</code> on iOS and <code>resource-id</code> on Android — which is what Detox queries. Master <code>View</code>, <code>Text</code>, <code>Pressable</code>, <code>FlatList</code>, the <code>Platform</code> module, Metro, and deep links, and you can read any screen in Ledger Live Mobile.
</div>

### 5.3.8 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — React Native for TypeScript Engineers</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> What does the <code>testID</code> prop on a <code>&lt;Pressable&gt;</code> become at runtime on the native side?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A CSS class on the Chromium DOM node</button>
<button class="quiz-choice" data-value="B">B) A custom React context consumed by Detox</button>
<button class="quiz-choice" data-value="C">C) <code>accessibilityIdentifier</code> on the backing UIView on iOS, and <code>resource-id</code> on the Android View</button>
<button class="quiz-choice" data-value="D">D) Nothing — it is stripped at build time in release mode</button>
</div>
<p class="quiz-explanation">React Native forwards <code>testID</code> to the platform's native accessibility identifier. Detox's <code>by.id()</code> query resolves to those native attributes, which is why the same test locator works on both platforms.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> Why does <code>&lt;View&gt;Hello&lt;/View&gt;</code> crash at runtime?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>View</code> is self-closing and cannot take children</button>
<button class="quiz-choice" data-value="B">B) Plain strings must be wrapped in <code>&lt;Text&gt;</code> — <code>View</code> is not a text container</button>
<button class="quiz-choice" data-value="C">C) Only <code>ScrollView</code> accepts string children</button>
<button class="quiz-choice" data-value="D">D) It works — the question is wrong</button>
</div>
<p class="quiz-explanation">Unlike a <code>&lt;div&gt;</code> in the DOM, <code>&lt;View&gt;</code> does not render text. Every visible string must be inside <code>&lt;Text&gt;</code>, which maps to <code>UILabel</code> / <code>TextView</code>.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> What happens if you put <code>testID="account-row"</code> directly on a <code>&lt;FlatList&gt;</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Every row receives the same <code>testID</code></button>
<button class="quiz-choice" data-value="B">B) Only the first row receives the <code>testID</code></button>
<button class="quiz-choice" data-value="C">C) It is ignored entirely</button>
<button class="quiz-choice" data-value="D">D) It lands on the outer scroll container, not on rows. Row <code>testID</code>s must be set inside <code>renderItem</code></button>
</div>
<p class="quiz-explanation"><code>FlatList</code> is a virtualized scroll container. Its own <code>testID</code> identifies the list. To address a row, pass a unique <code>testID</code> inside <code>renderItem</code> — and ensure the row component forwards it to its outer <code>View</code>.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> What is the role of the <strong>bridge</strong> (or JSI in the new architecture) in React Native?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It connects the JS thread (Hermes) to the UI / native thread so state changes in JS become native view mutations</button>
<button class="quiz-choice" data-value="B">B) It connects the app to Metro over HTTP</button>
<button class="quiz-choice" data-value="C">C) It is Detox's test runner channel</button>
<button class="quiz-choice" data-value="D">D) It converts JSX to HTML</button>
</div>
<p class="quiz-explanation">The bridge (old arch) serialized calls between the JS thread and the UI thread. The new architecture replaces that with JSI — direct synchronous function calls exposed to JS via C++ — but the role is the same: move data and commands between JS and native.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> How does a debug build differ from a release build with respect to JavaScript?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Debug uses Hermes, release uses JavaScriptCore</button>
<button class="quiz-choice" data-value="B">B) Debug loads the JS bundle from Metro over HTTP at app start; release bakes the JS bundle into the app package</button>
<button class="quiz-choice" data-value="C">C) Debug has a dev menu, release also has a dev menu</button>
<button class="quiz-choice" data-value="D">D) There is no difference — both download from Metro</button>
</div>
<p class="quiz-explanation">In debug the app fetches <code>index.bundle</code> from Metro on launch (which is why you need Metro running). In release the bundle is embedded in the <code>.ipa</code> / <code>.apk</code>, the dev menu is removed, and you cannot hot-reload.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> A test needs to jump directly to the Bitcoin account screen. What is the idiomatic way?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Tap through Portfolio → Accounts → Bitcoin</button>
<button class="quiz-choice" data-value="B">B) Call a native Detox API to push a screen onto the stack</button>
<button class="quiz-choice" data-value="C">C) Use a deep link such as <code>ledgerlive://account?currency=bitcoin</code> via <code>openDeeplink()</code></button>
<button class="quiz-choice" data-value="D">D) Edit the React navigation state from the test runner</button>
</div>
<p class="quiz-explanation">Deep links skip the tap-through navigation and cut flakiness. Detox's <code>device.openURL()</code> is wrapped by an <code>openDeeplink()</code> helper in Ledger Live Mobile E2E. The <code>Linking</code> API on the app side receives and routes the URL.</p>
</div>

<div class="quiz-score"></div>
</div>

---


## Mobile Toolchain & Environment Setup

<div class="chapter-intro">
Unlike desktop E2E — where <code>pnpm i</code> is roughly enough — mobile E2E requires you to install a small operating system's worth of native tooling: Xcode, Android Studio, CocoaPods, Ruby, Bundler, Java, Gradle, adb, simctl, applesimutils, a handful of emulators, and at least one simulator. This chapter is the definitive checklist. If anything breaks during the rest of Part 5, come back here.
</div>

### 5.4.1 The Toolchain at a Glance

Here is every moving part you will touch, grouped by role:

```
                +---------------------------------------------+
                |             YOUR LAPTOP (macOS)             |
                |                                             |
                |   proto / mise  ──>  Node, pnpm, Ruby       |
                |         │                                   |
                |         ├─> pnpm  ──>  monorepo packages    |
                |         │                                   |
                |         └─> Bundler  ──>  CocoaPods (gem)   |
                |                           │                 |
                |                           v                 |
                |                     Podfile / Pods          |
                |                                             |
                |   Xcode ──┬─> iOS Simulator                 |
                |           ├─> simctl (CLI to sim)           |
                |           └─> applesimutils (QA CLI)        |
                |                                             |
                |   Android Studio ──┬─> AVD Manager          |
                |                    ├─> emulator (CLI)       |
                |                    ├─> adb (device bridge)  |
                |                    └─> Gradle / JDK         |
                |                                             |
                |   Metro (localhost:8081)                    |
                |        ▲                                    |
                |        │ http (debug only)                  |
                |        │                                    |
                |   +----+-----------------+                  |
                |   |  Simulator / Emulator                   |
                |   |   runs Ledger Live Mobile .ipa / .apk   |
                |   +------------------------------------+    |
                +---------------------------------------------+
```

Pay attention to the arrows. Metro is a **dev-time** HTTP server that the app connects to **only for debug builds**. On iOS the simulator shares `localhost` with the host automatically. On Android the emulator does *not* — you must use `adb reverse` to forward port 8081 (see 33.5).

### 5.4.2 macOS-Only Reality (for iOS)

Apple legally restricts iOS development tooling to macOS:

- **iOS builds require a Mac** running macOS + Xcode. There is no supported way around this.
- **Android builds** work on macOS, Linux, and Windows.
- Linux contributors to Ledger Live Mobile can work on Android E2E only; the CI runs iOS jobs on macOS runners.

Windows is not supported for iOS. If you are on a Ledger-issued MacBook you are fine.

### 5.4.3 Version Management — `proto` and `mise`

A monorepo this big needs pinned tool versions. Ledger Live uses [`mise`](https://mise.jdx.dev/) (and historically [`proto`](https://moonrepo.dev/proto)) as the single source of truth.

At the repo root:

```
.prototools      (minimal: node, npm, pnpm)
mise.toml        (full: node, pnpm, npm, bun, gitleaks, pkl, plus plugins)
```

Example `mise.toml` entries (paraphrased from the actual repo):

```toml
min_version = '2026.3.1'

[plugins]
bundler   = "jonathanmorley/asdf-bundler"
cocoapods = "mise-plugins/mise-cocoapods"

[tools]
node = '24.14.0'
pnpm = '10.24.0'
npm  = '10.3.0'
```

Install `mise` once (`brew install mise`), then run `mise install` at the repo root. It reads the file and pulls down the exact versions, putting shims on your `PATH`. You never `nvm use` or `rbenv global` again — `mise` does it automatically when you `cd` into the repo.

> **Note:** `mise` can also manage Ruby, Bundler, CocoaPods, and Java via plugins. The current repo comments out Ruby management (install Ruby via Homebrew), but the plugins are declared so the team can flip them on. Check `mise.toml` on the branch you are working on.

> **Further reading:** [`mise.jdx.dev`](https://mise.jdx.dev/) and [`moonrepo.dev/proto`](https://moonrepo.dev/proto).

### 5.4.4 iOS Side — Xcode to CocoaPods

#### Xcode

Install from the Mac App Store. Version should match what the repo expects (check the `ios/` project's build settings). After install, run **once**:

```
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
sudo xcodebuild -license accept
xcodebuild -downloadPlatform iOS
```

This accepts the license and downloads the required iOS simulator platform. See [`developer.apple.com/documentation/xcode`](https://developer.apple.com/documentation/xcode).

#### iOS Simulator

Bundled with Xcode. Run it via **Xcode → Open Developer Tool → Simulator**, or from the CLI through `xcrun simctl`:

```
xcrun simctl list devices         # list all simulators
xcrun simctl boot "iPhone 15"     # boot a specific one
xcrun simctl shutdown all         # stop all
```

Reference: [`developer.apple.com/documentation/xcode/xcrun`](https://developer.apple.com/documentation/xcode/xcrun) and [`developer.apple.com/documentation/xcode/running-your-app-in-simulator-or-on-a-device`](https://developer.apple.com/documentation/xcode/running-your-app-in-simulator-or-on-a-device).

#### applesimutils

The Apple `simctl` CLI is powerful but misses a few operations Detox needs — granting permissions, clearing keychains, resetting push tokens. [`applesimutils`](https://github.com/wix/AppleSimulatorUtils) from the Wix team (who also maintain Detox) fills those gaps:

```
brew tap wix/brew
brew install applesimutils
```

If you skip this, Detox will complain at startup on iOS.

#### Ruby + Bundler + CocoaPods

CocoaPods is the iOS dependency manager. It is a **Ruby gem**. You do not install it globally — you use Bundler to install the exact version pinned by the repo's `Gemfile`:

```ruby
# apps/ledger-live-mobile/Gemfile (excerpt)
source "https://rubygems.org"

gem "fastlane", '~> 2.228.0'
gem 'cocoapods', '>= 1.13', '!= 1.15.0', '!= 1.15.1', '!= 1.16.0'
gem 'xcodeproj', '< 1.26.0'
```

Workflow:

```
cd apps/ledger-live-mobile
bundle install                # installs all gems locally (vendor/bundle)
bundle exec pod install --project-directory=ios
```

- `bundle install` reads the `Gemfile` + `Gemfile.lock` and installs the exact gem versions.
- `bundle exec pod install` runs CocoaPods via Bundler, so you use the pinned version instead of whatever is on your machine.
- `pod install` reads `ios/Podfile`, resolves all native iOS dependencies (React Native itself, Firebase SDK, WalletConnect, etc.), and generates `ios/Pods/` plus the `ios/ledgerlivemobile.xcworkspace`.

> **Gotcha:** After a big dependency bump, you will often need to run `pod install` again. Symptom: the app builds but crashes on launch with a missing class. Running `bundle exec pod install --project-directory=ios` and rebuilding almost always fixes it.

> **Further reading:** [`bundler.io/guides/getting_started.html`](https://bundler.io/guides/getting_started.html), [`cocoapods.org`](https://cocoapods.org/), [`guides.cocoapods.org/using/getting-started.html`](https://guides.cocoapods.org/using/getting-started.html), [`guides.cocoapods.org/using/the-podfile.html`](https://guides.cocoapods.org/using/the-podfile.html).

### 5.4.5 Android Side — Studio to Gradle to adb

#### Android Studio

Install from [`developer.android.com/studio`](https://developer.android.com/studio). The first launch walks you through installing:

- A JDK (Android Studio ships with one; you can use it)
- The Android SDK
- A default system image (pick API 34 or higher — check the repo's `android/build.gradle` for the target SDK)
- At least one AVD (Android Virtual Device)

Set two environment variables in your shell profile (`.zshrc`):

```
export ANDROID_HOME="$HOME/Library/Android/sdk"
export PATH="$PATH:$ANDROID_HOME/platform-tools:$ANDROID_HOME/emulator"
```

`platform-tools` is where `adb` lives. `emulator` is where the `emulator` CLI lives.

#### AVD Manager and the `emulator` CLI

AVDs are created in Android Studio: **Device Manager → Create device**. Name each one consistently, e.g. `Pixel_7_API_34`. List them from the CLI:

```
emulator -list-avds
emulator -avd Pixel_7_API_34                 # boot one
emulator -avd Pixel_7_API_34 -no-snapshot    # cold boot, useful after crashes
```

See [`developer.android.com/studio/run/managing-avds`](https://developer.android.com/studio/run/managing-avds) and [`developer.android.com/studio/run/emulator-commandline`](https://developer.android.com/studio/run/emulator-commandline).

#### `adb` — Android Debug Bridge

`adb` is the CLI that talks to connected devices and running emulators. Core commands:

```
adb devices                          # list attached devices / emulators
adb install app.apk                  # install an APK
adb shell pm clear com.ledger.live   # clear app data
adb logcat                           # stream device logs
adb reverse tcp:8081 tcp:8081        # <-- KEY for Metro
```

Reference: [`developer.android.com/tools/adb`](https://developer.android.com/tools/adb).

##### Why `adb reverse tcp:8081 tcp:8081` is critical

On iOS, `localhost` inside the simulator is the same as `localhost` on your Mac — so a debug build automatically reaches Metro on `http://localhost:8081/index.bundle`.

On Android it is not. The Android emulator runs in its own network namespace and its `localhost` is the emulator itself, not your machine. Without help, the app opens a connection to `localhost:8081` and gets nothing.

`adb reverse tcp:8081 tcp:8081` tells adb: "any TCP connection from the device to `localhost:8081` should be forwarded to `localhost:8081` on the host." Metro then answers and the debug bundle loads.

In Ledger Live Mobile's E2E setup, this call happens automatically in Detox's `setup.ts`:

```ts
await device.reverseTcpPort(8081);   // Metro
await device.reverseTcpPort(wsPort); // Device Bridge (default 8099)
```

See [`developer.android.com/tools/adb#forwardports`](https://developer.android.com/tools/adb#forwardports).

> **Gotcha:** If Android debug builds crash with "Unable to load script" or an endless red screen, your first instinct should be: "did `adb reverse` run? is Metro running on 8081?"

#### Gradle

Gradle is Android's build system. It resolves Maven dependencies, compiles Kotlin/Java, runs annotation processors, and packages the final `.apk`. You rarely invoke it directly — RN's scripts do. When you do:

```
cd apps/ledger-live-mobile/android
./gradlew assembleDebug
./gradlew assembleRelease
```

The Android Gradle Plugin version must match the Gradle version, and both must match the Kotlin version the RN release expects. When in doubt, follow the versions the repo pins. See [`docs.gradle.org/current/userguide/userguide.html`](https://docs.gradle.org/current/userguide/userguide.html) and [`developer.android.com/build/releases/gradle-plugin`](https://developer.android.com/build/releases/gradle-plugin).

#### System images and API levels

An AVD is "a device config + a system image." The system image is a specific Android version (e.g. "Android 14, API 34, x86_64 with Google Play"). Install system images from **SDK Manager → SDK Platforms** in Android Studio. For E2E, pick the API level the Android project targets (check `android/build.gradle`'s `compileSdk`). See [`developer.android.com/tools/releases/platforms`](https://developer.android.com/tools/releases/platforms).

### 5.4.6 The Ledger Live Monorepo Relevance

All of the above feeds into two peer locations in the monorepo:

```
ledger-live/
  apps/
    ledger-live-mobile/           <-- the RN app itself
      android/                    <-- Gradle project, AndroidManifest
      ios/                        <-- Xcode project, Podfile
      src/                        <-- TypeScript source
      .env.mock                   <-- ENVFILE for mock E2E (staging Firebase)
      .env.mock.prerelease        <-- ENVFILE for prerelease E2E (prod Firebase)
      Gemfile                     <-- Bundler-managed gems (CocoaPods)
      scripts/e2e-ci.mjs          <-- zx orchestrator invoked by CI
  e2e/
    mobile/                       <-- the canonical mobile E2E workspace (peer of apps/*)
      detox.config.js             <-- Detox build & device matrix
      jest.config.js              <-- Jest runner + Allure reporter config
      specs/                      <-- ~200 .spec.ts files (covered in Ch 5.5+)
      page/                       <-- Application POM hub (see Ch 5.7)
      bridge/                     <-- mock device + WS server
      package.json                <-- scripts: build:ios(:debug), test:ios(:debug), allure
      project.json                <-- Nx targets (e2e:ci)
```

> **Migration note:** the mobile E2E code used to live inside `apps/ledger-live-mobile/e2e/`. Most of it now lives at the top-level `e2e/mobile/` peer workspace. A handful of legacy specs (about 17 at the time of writing) still sit at the old path during the in-progress migration — if you see `apps/ledger-live-mobile/e2e/...` in the wild, treat it as legacy and prefer `e2e/mobile/` for new work.

Part 1 Chapter 4 explained the monorepo layout at a high level (apps vs libs vs tools). Zoom in here: the app itself (native projects, TypeScript source, env files, `e2e-ci.mjs`) stays under `apps/ledger-live-mobile/`, while the Detox test suite has been lifted into its own Nx/Turbo workspace at `e2e/mobile/`. Shared code (LLM logic, account model, countervalues) lives in `libs/` and is consumed through the monorepo workspace links, exactly like Desktop.

### 5.4.7 `ENVFILE` — Which Firebase Project Do You Mock Against?

Ledger Live Mobile reads its environment from a dotenv file pointed at by the `ENVFILE` variable. Two files matter for QA:

| File | Firebase / feature-flag backend | When to use |
|------|--------------------------------|-------------|
| `.env.mock`             | **Staging** Firebase (feature flags for dev + QA) | Default for local E2E and CI |
| `.env.mock.prerelease`  | **Production** Firebase (what real users see)     | Pre-release smoke tests only |

The `mock` suffix does not mean "fake everything." It means "run the app against **mocked** Ledger Sync, Countervalues, and Speculos-driven device flows" — the rest (Firebase, deep links, navigation) is still real. We mock because tests must be deterministic and must not spend real crypto or depend on external API availability.

Typical usage (canonical command):

```
pnpm e2e:mobile run build:ios:debug
```

The build scripts read `ENVFILE` (set inside `e2e/mobile/detox.config.js` as `apps/ledger-live-mobile/.env.mock` or `.env.mock.prerelease`) and inject it into the native build, so both JS and native code see the same values. The env files themselves still live at the app root (`apps/ledger-live-mobile/.env.mock` / `.env.mock.prerelease`) and are not checked in — see the internal `mobile-env.md` for the list of required keys.

> **Gotcha:** If `ENVFILE` is missing or unreadable, the app crashes on first launch with "Firebase config missing." Running the canonical scripts above is the safe path — they rely on the `detox.config.js` constants so you rarely need to export `ENVFILE` yourself.

### 5.4.8 Build Commands — the `e2e/mobile` workspace scripts

Since the migration, the Detox build and test scripts live in `e2e/mobile/package.json`. There are three equivalent ways to invoke them — pick whichever fits your mental model:

**1. From the repo root via pnpm filter (recommended in docs & CI):**

```
pnpm e2e:mobile run build:ios:debug
pnpm e2e:mobile run build:ios        # iOS release
pnpm e2e:mobile run build:android:debug
pnpm e2e:mobile run build:android    # Android release
```

**2. From inside the `e2e/mobile/` workspace:**

```
cd e2e/mobile
pnpm build:ios:debug
pnpm build:ios
pnpm build:android:debug
pnpm build:android
```

**3. Directly via Nx (what CI ultimately runs):**

```
nx run live-mobile:e2e:build -- --configuration ios.sim.debug
nx run live-mobile:e2e:build -- --configuration ios.sim.release
nx run live-mobile:e2e:build -- --configuration android.emu.debug
nx run live-mobile:e2e:build -- --configuration android.emu.release
```

Breaking those down:

- `pnpm e2e:mobile` is a root-level alias declared in the monorepo `package.json` (`"e2e:mobile": "pnpm --filter ledger-live-mobile-e2e-tests"`). It forwards the rest of the command to the `e2e/mobile/` workspace. If the alias is missing on your branch, substitute `pnpm --filter ledger-live-mobile-e2e-tests`.
- `build:ios:debug` ultimately invokes `pnpm --filter live-mobile run e2e:build --configuration ios.sim.debug`, which triggers the Nx target `live-mobile:e2e:build` and runs `detox build -c ios.sim.debug` inside the mobile app.
- The configuration name (`ios.sim.debug`, `ios.sim.release`, etc.) selects a configuration from `e2e/mobile/detox.config.js`. The full list exposed in the repo is:

| Config                   | Platform | Build type | Firebase / flags source      |
|--------------------------|----------|------------|------------------------------|
| `ios.sim.debug`          | iOS      | Debug      | `.env.mock` (staging)        |
| `ios.sim.staging`        | iOS      | Staging    | Pre-release Firebase             |
| `ios.sim.release`        | iOS      | Release    | Production-like, mock flags  |
| `ios.sim.prerelease`     | iOS      | Prerelease | `.env.mock.prerelease` (prod)|
| `android.emu.debug`      | Android  | Debug      | `.env.mock` (staging)        |
| `android.emu.prerelease`    | Android  | Staging    | Pre-release Firebase             |
| `android.emu.release`    | Android  | Release    | Production-like, mock flags  |

Rules of thumb:
- For iterating on tests locally → `ios.sim.debug` or `android.emu.debug`. Fast, hot-reloadable.
- For reproducing a CI failure → the same `release` config CI uses. Slower but bytecode-accurate.
- For smoke-testing a release candidate before store submission → `prerelease`. Runs against production Firebase, so you see the flags real users will see.

The `test:*` scripts then run the already-built app. From inside `e2e/mobile/`:

```
pnpm test:ios:debug                 # iOS debug
pnpm test:ios                       # iOS release
pnpm test:android:debug             # Android debug
pnpm test:android                   # Android release
```

Or from the repo root:

```
pnpm e2e:mobile run test:ios:debug
```

Each of these ultimately expands to `detox test --configuration <config>` with `detox` resolved from `e2e/mobile/node_modules` and pointing at `e2e/mobile/detox.config.js`.

Chapter 5.5 will dig into the Detox runner side. Here we only care that the **build** step produces an artifact (`.app` or `.apk`) that Detox can install.

### 5.4.9 Verifying Your Setup — A Checklist

Before you ever try to run a real test, go through this list. Each step has an expected output; if yours differs, stop and fix it before moving on.

**1. Tool versions**

```
mise --version            # >= 2026.3.1
node -v                   # 24.14.x
pnpm -v                   # 10.24.x
ruby -v                   # 3.x (via Homebrew or mise)
bundle -v                 # 2.x
pod --version             # 1.13+
```

**2. Xcode & simulators**

```
xcodebuild -version       # Xcode <pinned version>
xcrun simctl list devices | grep Booted    # at least one simulator or nothing yet
applesimutils --list      # prints simulators; means applesimutils works
```

**3. Android tools**

```
adb version               # Android Debug Bridge version
adb devices               # List of devices — empty is fine, not missing
emulator -list-avds       # Prints at least one AVD name
echo $ANDROID_HOME        # Prints /Users/you/Library/Android/sdk
```

**4. Repo-level install**

From the monorepo root:

```
pnpm install              # Installs all workspace packages
cd apps/ledger-live-mobile
bundle install            # Installs Ruby gems (CocoaPods, fastlane)
bundle exec pod install --project-directory=ios
```

Expected: `pnpm install` completes without errors, `bundle install` prints "Bundle complete!", `pod install` prints "Pod installation complete!"

**5. Metro can start**

```
pnpm mobile start         # Starts Metro on http://localhost:8081
```

Visit `http://localhost:8081/status` in a browser — you should see `packager-status:running`.

**6. Build a debug app**

```
pnpm e2e:mobile run build:ios:debug
# or: cd e2e/mobile && pnpm build:ios:debug
```

Expected: `xcodebuild` runs for several minutes, ends with "BUILD SUCCEEDED" and the built `.app` lands inside the Detox build output directory (resolved via `detox.config.js` to `apps/ledger-live-mobile/ios/build`).

If all six pass, you are ready for Chapter 5.5 where we finally introduce Detox and wire the first test.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://developer.apple.com/documentation/xcode">Xcode documentation</a></li>
<li><a href="https://developer.apple.com/documentation/xcode/xcrun">xcrun / simctl</a></li>
<li><a href="https://developer.apple.com/documentation/xcode/running-your-app-in-simulator-or-on-a-device">Running your app in Simulator</a></li>
<li><a href="https://developer.android.com/tools/adb">adb reference</a></li>
<li><a href="https://developer.android.com/tools/adb#forwardports">adb: port forwarding / reverse</a></li>
<li><a href="https://developer.android.com/studio/run/managing-avds">Managing AVDs</a></li>
<li><a href="https://developer.android.com/studio/run/emulator-commandline">emulator CLI</a></li>
<li><a href="https://developer.android.com/tools/releases/platforms">Android platforms &amp; API levels</a></li>
<li><a href="https://cocoapods.org/">CocoaPods</a> and <a href="https://guides.cocoapods.org/using/getting-started.html">Getting Started</a> / <a href="https://guides.cocoapods.org/using/the-podfile.html">The Podfile</a></li>
<li><a href="https://bundler.io/">Bundler</a> and <a href="https://bundler.io/guides/getting_started.html">Getting Started</a></li>
<li><a href="https://docs.gradle.org/current/userguide/userguide.html">Gradle user guide</a> and <a href="https://developer.android.com/build/releases/gradle-plugin">Android Gradle Plugin</a></li>
<li><a href="https://moonrepo.dev/proto">proto</a> and <a href="https://mise.jdx.dev/">mise</a></li>
<li><a href="https://github.com/wix/AppleSimulatorUtils">applesimutils</a></li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Mobile E2E needs an entire platform toolchain on top of Node — Xcode + simulator + applesimutils + CocoaPods/Ruby/Bundler on the iOS side, and Android Studio + AVDs + adb + Gradle on the Android side. <code>mise</code> pins versions; <code>ENVFILE</code> (wired in <code>e2e/mobile/detox.config.js</code>) picks which Firebase backend you test against; <code>pnpm e2e:mobile run build:ios:debug</code> (or the equivalent in-workspace <code>pnpm build:ios:debug</code>, or <code>nx run live-mobile:e2e:build -- --configuration ios.sim.debug</code>) compiles the app for Detox. On Android, <code>adb reverse tcp:8081 tcp:8081</code> is what makes debug builds able to reach Metro — without it, you get a red screen and a very confused developer.
</div>

### 5.4.10 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Mobile Toolchain &amp; Environment Setup</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> Why does <code>adb reverse tcp:8081 tcp:8081</code> exist in the Android E2E setup?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) To expose Detox's runner to the emulator</button>
<button class="quiz-choice" data-value="B">B) To reset the emulator's network stack</button>
<button class="quiz-choice" data-value="C">C) The emulator's <code>localhost</code> is not the host's — this command forwards the device's <code>localhost:8081</code> to the host's Metro on <code>localhost:8081</code></button>
<button class="quiz-choice" data-value="D">D) It is only used in release builds</button>
</div>
<p class="quiz-explanation">Android emulators have their own network namespace. Debug builds need to load the JS bundle from Metro on the host machine. <code>adb reverse</code> forwards the emulator's <code>localhost:8081</code> to the host's <code>localhost:8081</code>. iOS simulators share the host network so this is not needed.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q2.</strong> Why install CocoaPods via <code>bundle install</code> rather than <code>gem install cocoapods</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Bundler installs the exact gem versions pinned in <code>Gemfile</code> / <code>Gemfile.lock</code>, ensuring every engineer uses the same CocoaPods version</button>
<button class="quiz-choice" data-value="B">B) <code>gem install</code> does not work on macOS</button>
<button class="quiz-choice" data-value="C">C) Bundler installs faster</button>
<button class="quiz-choice" data-value="D">D) CocoaPods can only be installed through Bundler</button>
</div>
<p class="quiz-explanation">The repo pins specific versions (e.g. excluding known-broken CocoaPods 1.15.0 / 1.15.1 / 1.16.x). Bundler reads <code>Gemfile.lock</code> and installs those exact versions, so <code>bundle exec pod install</code> uses a reproducible CocoaPods on every machine and in CI.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q3.</strong> What is the practical difference between <code>.env.mock</code> and <code>.env.mock.prerelease</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) One enables mocks, the other disables them</button>
<button class="quiz-choice" data-value="B">B) <code>.env.mock</code> points to the staging Firebase project; <code>.env.mock.prerelease</code> points to the production Firebase project</button>
<button class="quiz-choice" data-value="C">C) <code>.env.mock</code> is for iOS, <code>.env.mock.prerelease</code> is for Android</button>
<button class="quiz-choice" data-value="D">D) They are identical — the suffix is cosmetic</button>
</div>
<p class="quiz-explanation">Both activate the same mocking strategy for Ledger Sync, countervalues, and device flows. They differ in which Firebase project (therefore which feature-flag set) the app reads from. Day-to-day E2E uses staging; pre-release smoke tests validate against production feature flags before shipping to users.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> What does <code>pnpm e2e:mobile run build:ios</code> (equivalent to <code>nx run live-mobile:e2e:build -- --configuration ios.sim.release</code>) produce, and what is it used for?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A debug <code>.app</code> ready for hot reload</button>
<button class="quiz-choice" data-value="B">B) A Firebase release tag</button>
<button class="quiz-choice" data-value="C">C) A signed IPA for TestFlight</button>
<button class="quiz-choice" data-value="D">D) A release-mode iOS <code>.app</code> for the simulator — JS is pre-bundled (no Metro), ideal for reproducing CI failures</button>
</div>
<p class="quiz-explanation">The <code>*.release</code> Detox configs build a release-mode app: Hermes bytecode, bundled JS, no dev menu. This matches what CI runs, so it is the configuration to reach for when reproducing a CI-only bug locally.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> You skipped installing <code>applesimutils</code>. What symptom will you see first?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The iOS build fails in <code>xcodebuild</code></button>
<button class="quiz-choice" data-value="B">B) Detox fails at startup on iOS because it cannot grant permissions / reset the simulator</button>
<button class="quiz-choice" data-value="C">C) Metro refuses to start</button>
<button class="quiz-choice" data-value="D">D) The tests pass but Allure produces no report</button>
</div>
<p class="quiz-explanation"><code>applesimutils</code> supplies operations that plain <code>simctl</code> does not cover — granting permissions, clearing keychains, resetting push tokens. Detox calls it during test setup. Without it installed, the iOS run aborts almost immediately.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q6.</strong> What role does <code>mise</code> (or <code>proto</code>) play in the toolchain?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It pins exact versions of Node, pnpm, and optionally Ruby / CocoaPods / Java, and activates them automatically when you <code>cd</code> into the repo</button>
<button class="quiz-choice" data-value="B">B) It is a replacement for Gradle</button>
<button class="quiz-choice" data-value="C">C) It is the iOS build system</button>
<button class="quiz-choice" data-value="D">D) It manages Android AVDs</button>
</div>
<p class="quiz-explanation"><code>mise</code> reads <code>mise.toml</code> / <code>.prototools</code> at the repo root and shims the right versions of each tool onto your <code>PATH</code>. This removes the class of bugs where "works on my machine" means "works on <em>my</em> Node version."</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Detox from Zero: Core Concepts

<div class="chapter-intro">
Detox is to React Native what Playwright is to the web — a grey-box end-to-end framework that drives a real application running on a real device (or simulator/emulator). If you have never touched it, this chapter takes you from "what is Detox?" to reading and writing tests confidently. Every concept is anchored in Ledger Live Mobile code. After this chapter you will understand the three-verb grammar of Detox (matcher, action, expectation), the synchronization model, and why mobile E2E feels different from web E2E even though the surface APIs look similar.
</div>

### 5.5.1 What Is Detox?

Detox is an open-source **grey-box** end-to-end testing framework for React Native, created and maintained by **Wix** since 2016. Wix built it because, at scale, their React Native app was flaky under every black-box tool available (Appium in particular). Their insight: a test runner that has **privileged, in-process knowledge** of the app under test can detect when the app is busy — and wait deterministically — rather than guessing with arbitrary `sleep()` calls.

**Key facts:**
- Language: TypeScript/JavaScript
- Created by: Wix (the website builder)
- First release: 2016
- Targets: React Native on iOS and Android only
- Runs on top of: **XCUITest** (iOS) and **Espresso** (Android) — both are Google-maintained native UI test drivers
- Killer feature: **synchronization** — Detox knows when the app's JS bridge, dispatch queues, animations, timers, and network calls are idle, and it automatically waits until the app is idle before performing the next action

**Grey-box vs black-box — what the difference actually means:**

| Aspect | Black-box (Appium) | Grey-box (Detox) |
|--------|--------------------|------------------|
| How it drives the app | Talks to an external OS-level agent (WebDriverAgent on iOS, UIAutomator2 on Android) over HTTP | Injects a native test library into the app itself (XCUITest/Espresso) |
| Knowledge of app state | None — sees only what the OS accessibility tree exposes | Full — sees the JS bridge, dispatch queues, animations, timers, network |
| Waiting strategy | Poll with timeouts (`implicitWait`) | Wait for app to be idle, then act |
| Flakiness | High — tests need explicit waits and retries everywhere | Low — idle detection eliminates most race conditions |
| Speed | Slow — every command is an HTTP round-trip to an agent | Fast — commands run inside the app process |
| Cross-platform real devices | Yes, on real devices + simulators | Simulators and emulators primarily (real device support is experimental) |

> **Why this matters for Ledger Live:** our React Native app talks to a Ledger device over BLE/USB, downloads feature flags, imports Redux state, dispatches mock device events from the bridge — a huge amount happens asynchronously on startup. Black-box testing this with arbitrary waits would be miserable. Detox waits for the bridge to be idle before it does anything.

**Detox vs Playwright — same verbs, very different model:**

| | Playwright (web/Electron) | Detox (React Native) |
|---|---|---|
| Driver protocol | Chrome DevTools Protocol (in-process) | Native test library injected into the app |
| Locator primitive | `Locator` — a lazy, retryable query | `Element` — a matcher reference, not a node |
| Auto-waiting | Actionability checks (attached, visible, stable, enabled) | App-wide idle detection (bridge + queues + timers + animations) |
| Assertion syntax | `expect(locator).toBeVisible()` | `expect(element(by.id("x"))).toBeVisible()` |
| Parallelism | Workers share a browser or spawn many | Each worker needs its own device/simulator |
| Cleanup granularity | `context` per test | `device` per worker, state management is manual |

The two tools look deceptively similar on the page. The underlying model is very different: Playwright watches the DOM; Detox watches the whole app runtime.

### 5.5.2 The Matcher–Action–Expectation Triad

Every Detox test boils down to three kinds of calls:

1. **Matcher** — how do I describe the element I want? (`by.id("send-button")`)
2. **Action** — what do I do to it? (`.tap()`)
3. **Expectation** — what should be true after? (`expect(...).toBeVisible()`)

The three are composed through `element()`:

```typescript
import { by, element, expect } from "detox";

await element(by.id("send-button")).tap();                 // matcher + action
await expect(element(by.text("Send"))).toBeVisible();      // matcher + expectation
```

**Side-by-side examples:**

```typescript
// (1) Tap a button identified by testID
await element(by.id("onboarding-skip")).tap();

// (2) Type into a text field then assert visibility of a result
await element(by.id("search-input")).typeText("Bitcoin");
await expect(element(by.text("Bitcoin (BTC)"))).toBeVisible();

// (3) Scroll a list until an element appears
await waitFor(element(by.text("Solana")))
  .toBeVisible()
  .whileElement(by.id("currencies-list"))
  .scroll(200, "down");

// (4) Long-press then assert a context menu opened
await element(by.id("account-row-bitcoin")).longPress();
await expect(element(by.id("account-context-menu"))).toExist();

// (5) Replace text (faster than typeText) and assert value
await element(by.id("amount-input")).replaceText("0.001");
await expect(element(by.id("amount-input"))).toHaveValue("0.001");
```

> **Rule:** every Detox statement is either "find + do" (matcher + action) or "find + assert" (matcher + expectation). If a line isn't one of those, you probably want `device.*` (see 34.8) or a `waitFor(...)` polling construct (see 34.7).

### 5.5.3 Matchers — `by.*`

Matchers are small description objects that tell Detox "the element is the one whose *X* equals *Y*". They do **not** find anything yet. They just describe.

| Matcher | What it matches | Android | iOS |
|---------|-----------------|---------|-----|
| `by.id("send-btn")` | React Native `testID` prop | `android:tag` | `accessibilityIdentifier` |
| `by.text("Send")` | Visible text content | `android:text` | label/text |
| `by.label("Send button")` | Accessibility label | `contentDescription` | `accessibilityLabel` |
| `by.type("RCTTextInput")` | Native view class name | Full Java class name | Full ObjC class name |
| `by.traits(["button"])` | iOS accessibility traits | not supported | `accessibilityTraits` |

**`by.id` is the idiomatic choice**, for the same reason `getByTestId` dominates in Playwright: it is stable, short, and injected by developers specifically for tests. The React Native source looks like this:

```tsx
<Pressable testID="onboarding-skip" onPress={handleSkip}>
  <Text>Skip</Text>
</Pressable>
```

And the test:

```typescript
await element(by.id("onboarding-skip")).tap();
```

**Composition** — when a test ID alone is not unique (e.g. many rows in a list), compose matchers with `.and(...)`, `.withAncestor(...)`, `.withDescendant(...)`:

```typescript
// Combine: the element with testID "row" AND containing text "Bitcoin"
await element(by.id("row").and(by.text("Bitcoin"))).tap();

// Ancestor: the "delete" button whose ancestor has testID "row-bitcoin"
await element(by.id("delete").withAncestor(by.id("row-bitcoin"))).tap();

// Descendant: the row that contains a child with testID "badge-selected"
await element(by.id("row").withDescendant(by.id("badge-selected"))).tap();
```

When multiple elements match, use `atIndex(n)`:

```typescript
await element(by.id("account-row")).atIndex(0).tap();  // first account row
```

This is exactly what Ledger Live does in `e2e/helpers/elementHelpers.ts`:

```typescript
getElementById(id: string | RegExp, index = 0) {
  return element(by.id(id)).atIndex(index);
},
```

> **Reference:** https://wix.github.io/Detox/docs/api/matchers

### 5.5.4 Actions — `element(...).*`

Once you have a matcher, you pick an action. Actions return promises — always `await` them.

| Action | What it does | Typical use |
|--------|--------------|-------------|
| `tap()` | Single tap at the element's centre | Buttons, list rows |
| `longPress(duration?)` | Press-and-hold | Context menus, reordering |
| `multiTap(times)` | Tap N times in quick succession | Double-tap to zoom |
| `tapAtPoint({ x, y })` | Tap at a specific offset inside the element | Click on a map, on a specific part of a custom view |
| `typeText(text)` | Simulate per-character keystrokes | Inputs that react to each character (autocomplete) |
| `replaceText(text)` | Replace the whole value atomically (no keystrokes) | Fast form filling — preferred when you don't need the per-char handlers |
| `clearText()` | Clear an input | Reset before typing |
| `scroll(amount, direction, startX?, startY?)` | Scroll the element by N pixels | Lists, scroll views |
| `scrollTo(edge)` | Scroll to `"top"`, `"bottom"`, `"left"`, `"right"` | Jump to list edges |
| `swipe(direction, speed?, percentage?)` | Swipe gesture on the element | Dismiss rows, pull-to-refresh |
| `pinch(scale, speed?, angle?)` | Pinch zoom gesture | Maps, images |

**Real examples:**

```typescript
// Tap a button
await element(by.id("continue-button")).tap();

// Type a BIP39 word, expect autocomplete to react
await element(by.id("word-input-1")).typeText("abandon");

// Replace text — faster, skips per-character events
await element(by.id("amount-input")).replaceText("0.00123");

// Clear then replace
await element(by.id("amount-input")).clearText();
await element(by.id("amount-input")).replaceText("1.0");

// Scroll a list 200px down, starting from 80% of its height (Android gotcha)
await element(by.id("account-list")).scroll(200, "down", NaN, 0.8);

// Swipe left on a row to reveal delete
await element(by.id("account-row-bitcoin")).swipe("left", "fast", 0.75);

// Pull-to-refresh
await element(by.id("portfolio-scroll")).swipe("down", "slow", 0.9);
```

> **Android gotcha (from `elementHelpers.ts`):** on Android, starting a scroll at `y = 0.5` (middle) often triggers a long-press on a list row. The Ledger Live helpers use `startPositionY = 0.8`:
> ```typescript
> const startPositionY = 0.8; // Needed on Android to scroll views: https://github.com/wix/Detox/issues/3918
> ```

> **Reference:** https://wix.github.io/Detox/docs/api/actions

### 5.5.5 Expectations — `expect(...).*`

Expectations retry-wait for up to a small internal timeout (≈ 1s by default, but synchronization-gated — see 35.3) and then throw if the condition is false.

| Expectation | Meaning |
|-------------|---------|
| `toBeVisible()` | Element is rendered AND visible on screen (intersection > 75% by default) |
| `toExist()` | Element exists in the hierarchy (may be off-screen) |
| `toHaveText(text)` | The element's text content equals `text` |
| `toHaveValue(value)` | The element's value equals `value` (inputs, switches) |
| `toHaveLabel(label)` | Accessibility label equals `label` |
| `toHaveId(id)` | testID equals `id` |
| `toBeFocused()` | Element currently has input focus |
| `toHaveSliderPosition(n)` | Slider is at position `n` |

Every expectation can be negated with `.not`:

```typescript
await expect(element(by.id("loading-spinner"))).not.toBeVisible();
await expect(element(by.id("error-banner"))).not.toExist();
```

**Mental model — `toBeVisible` vs `toExist`:**

- `toExist()` only checks the view hierarchy. A detached `View` deep under a collapsed scroll view counts as existing.
- `toBeVisible()` requires the element's visible area to exceed 75% of its size — it's much stricter, and it is the right default for "the user can see this".

```typescript
// Correct — assert the user can see the button
await expect(element(by.id("send-button"))).toBeVisible();

// Correct — assert the row is in the DOM even if off-screen (common for virtualized lists)
await expect(element(by.id("account-row-42"))).toExist();
```

> **Reference:** https://wix.github.io/Detox/docs/api/expect

### 5.5.6 `element()` Is Not an Element

This is the number-one source of confusion for people coming from Playwright, Selenium, or DOM manipulation.

```typescript
const btn = element(by.id("send-button"));
```

`btn` is **not a reference to a UI node**. It is a reference to a *query* — a matcher wrapped in a proxy — that Detox will resolve **every time** you call an action or expectation on it. There is no `await`, no round-trip to the device here. Nothing has happened yet.

This means:

- You can declare `element()` proxies eagerly at the top of a page object — it is free.
- The same proxy can resolve to different physical views across navigations — you don't need to re-create it.
- If the matcher is ambiguous, `element(...)` itself doesn't throw; the action/expectation does.

**Compare to Playwright's `Locator`:** same idea. A `Locator` is also lazy and re-evaluates on each action. The mental model carries over cleanly.

```typescript
// Declare once, use many times
const amountInput = element(by.id("amount-input"));
await amountInput.clearText();
await amountInput.replaceText("0.001");
await expect(amountInput).toHaveValue("0.001");
```

> **Reference:** https://wix.github.io/Detox/docs/api/the-element-object

### 5.5.7 `waitFor(...)` — Polling With a Timeout

Expectations have a short internal wait. When you need to wait longer — for example, for a screen to finish loading, or for a list to scroll until an item comes into view — use `waitFor(...)`:

```typescript
import { waitFor } from "detox";

// Wait up to 10s for the portfolio screen to appear
await waitFor(element(by.id("portfolio-screen")))
  .toBeVisible()
  .withTimeout(10000);
```

The chain is:
1. `waitFor(element(...))` — what to poll
2. `.toBeVisible()` / `.toExist()` / `.toHaveText(...)` / etc. — the condition
3. `.withTimeout(ms)` — the overall deadline

**`whileElement(...)` — scroll until visible:**

One of Detox's most useful patterns is "keep scrolling this list until the target is visible":

```typescript
await waitFor(element(by.text("Solana")))
  .toBeVisible()
  .whileElement(by.id("currencies-list"))
  .scroll(200, "down");
```

This says: poll for `by.text("Solana")` to be visible; while it isn't, scroll the list 200px down, then re-check. It is the idiomatic way to find items in long lists.

The Ledger Live wrapper is simply a tiny default-timeout helper:

```typescript
// e2e/helpers/elementHelpers.ts
waitForElementById(id: string | RegExp, timeout: number = DEFAULT_TIMEOUT) {
  return waitFor(element(by.id(id)))
    .toBeVisible()
    .withTimeout(timeout);
},
```

With `DEFAULT_TIMEOUT = 60000` — 60 seconds. Long, because mobile startup + Speculos handshake takes time.

> **Reference:** https://wix.github.io/Detox/docs/api/waitfor

### 5.5.8 The `device` Global

`device` is the process-wide handle to the simulator/emulator. It is imported from `detox`:

```typescript
import { device } from "detox";
```

| Method | What it does |
|--------|--------------|
| `device.launchApp(opts?)` | Launch/relaunch the app under test with optional `launchArgs`, `permissions`, `languageAndLocale`, `newInstance`, etc. |
| `device.reloadReactNative()` | Reload the JS bundle without restarting the native side. Fast test isolation for pure-RN state. |
| `device.terminateApp()` | Kill the app process cleanly |
| `device.sendToHome()` | Send the app to background (home screen). Used to test backgrounding |
| `device.takeScreenshot(name)` | Capture a screenshot, attached to the Detox artifacts folder |
| `device.generateViewHierarchyXml()` | Dump the entire native view tree as XML — gold for debugging "why doesn't this match?" |
| `device.openURL({ url })` | Open a deeplink (e.g. `ledgerlive://portfolio`) |
| `device.sendUserNotification(payload)` | Simulate a push notification tap |
| `device.getPlatform()` | `"ios"` or `"android"` — use to branch test logic |
| `device.reverseTcpPort(port)` | Android only: forward `emulator:port → host:port` (see 35.7) |
| `device.disableSynchronization()` | Turn off idle detection for stubborn screens (see 35.4) |

**Real usage from Ledger Live `commonHelpers.ts`:**

```typescript
const BASE_DEEPLINK = "ledgerlive://";

/** @param path the part after "ledgerlive://", e.g. "portfolio" */
export async function openDeeplink(path?: string) {
  await device.openURL({ url: BASE_DEEPLINK + path });
}

export function isAndroid() {
  return device.getPlatform() === "android";
}
```

And the launch call:

```typescript
await device.launchApp({
  launchArgs: {
    wsPort: port,                                  // which port the bridge server is on
    detoxURLBlacklistRegex: '\\(...\\)',           // URLs to NOT wait on (analytics, etc.)
    mock: getEnv("MOCK") ? getEnv("MOCK") : "1",   // mock mode flag
    IS_TEST: true,
  },
  languageAndLocale: { language: "en-US", locale: "en-US" },
  permissions: { camera: "YES" },                  // pre-grant iOS permissions
});
```

The `launchArgs` object is turned into CLI-style flags that the native code reads on boot (via `react-native-launch-arguments`). This is how the bridge client learns which WebSocket port to connect to.

> **Reference:** https://wix.github.io/Detox/docs/api/device

### 5.5.9 Your First Detox Test — Line by Line

Here is the minimal possible Detox spec. Nothing Ledger-specific, just the framework:

```typescript
// firstTest.spec.ts
import { by, device, element, expect } from "detox";

describe("First Detox test", () => {
  beforeAll(async () => {
    await device.launchApp();               // (1)
  });

  beforeEach(async () => {
    await device.reloadReactNative();       // (2)
  });

  it("shows the welcome screen on boot", async () => {
    await expect(element(by.id("welcome"))).toBeVisible();  // (3)
  });

  it("navigates to settings when the gear is tapped", async () => {
    await element(by.id("gear-button")).tap();              // (4)
    await expect(element(by.text("Settings"))).toBeVisible();
  });
});
```

**Walk-through:**

1. `device.launchApp()` — installs (if needed) and launches the app on the configured device. This happens once before all tests in the file.
2. `device.reloadReactNative()` — between every test, reload only the JS bundle. Much faster than a full `launchApp()` and enough to reset RN state for most tests. (When you need a full reset — e.g. the bridge client has to reconnect — use `device.launchApp({ newInstance: true })`.)
3. Matcher + expectation: find the element whose `testID === "welcome"`, assert it is visible. Detox synchronizes internally: if the splash screen is still animating, it waits.
4. Matcher + action: tap the gear button. Detox waits for the app to be idle (no pending animations, no pending JS, no in-flight network in the non-blacklisted set), then taps, then waits again.

There is no `sleep()`, no `waitForTimeout()`, no polling loop. That is the entire point of Detox.

### 5.5.10 The Platform Split — XCUITest vs Espresso

Detox is a thin-ish cross-platform façade over two very different native drivers:

- **iOS:** [XCUITest](https://developer.apple.com/documentation/xctest) — Apple's native iOS UI testing framework. Runs in the same process as the app. Hooks into `CADisplayLink`, `NSURLSession`, `NSOperationQueue`, main dispatch queue, etc., to detect idle.
- **Android:** [Espresso](https://developer.android.com/training/testing/espresso) — Google's Android UI test driver. Runs in an instrumented test process. Hooks into the main looper, `IdlingResource`s, view tree observer.

**Consequences you will hit in practice:**

| Symptom | iOS (XCUITest) | Android (Espresso) |
|---------|----------------|--------------------|
| Test IDs resolve to | `accessibilityIdentifier` | `android:tag` |
| `by.text` matches against | `UILabel`/`UITextView` text | `android:text` (excludes children's text unless the view is a merged node) |
| `scroll()` starting position | Safe at `(NaN, 0.5)` | Often fails — use `(NaN, 0.8)` (see 34.4 gotcha) |
| Long lists | Virtualized `FlatList` rows exist when scrolled near, so `toExist` works after reaching them | Same |
| Networking idle | Watches `NSURLSession` | Watches OkHttp via an IdlingResource |
| Per-worker isolation | One Simulator per worker (simulator shutdown is cheap) | One Emulator per worker (emulator boot is expensive) |

You rarely need to branch by platform in test logic, but you *do* have to remember that:

- **`reverseTcpPort` is Android-only.** On iOS simulator, `localhost` in the app === `localhost` on the host, so no port forwarding is needed (see 35.7).
- **Android APKs ship as a pair**: the app APK plus a `testBinaryPath` (the instrumentation APK that hosts Espresso). The `detox.config.js` declares both. iOS ships a single `.app` bundle.

> **Why this matters for flakiness:** most "my test works on iOS but fails on Android" bugs trace back to one of: (a) wrong `by.text` match because Android excludes nested text, (b) scroll gesture rejected because it started on a pressable, (c) a missing `device.reverseTcpPort()` so the bridge client can't reach the host.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://wix.github.io/Detox/docs/introduction/getting-started">Detox: Getting Started</a> — official onboarding doc</li>
<li><a href="https://wix.github.io/Detox/docs/api/matchers">Detox: Matchers API</a> — full <code>by.*</code> reference</li>
<li><a href="https://wix.github.io/Detox/docs/api/actions">Detox: Actions API</a> — every action with options</li>
<li><a href="https://wix.github.io/Detox/docs/api/expect">Detox: Expectations API</a> — assertion reference</li>
<li><a href="https://wix.github.io/Detox/docs/api/the-element-object">Detox: The element() Object</a> — why it's a proxy, not a node</li>
<li><a href="https://wix.github.io/Detox/docs/api/waitfor">Detox: waitFor API</a> — polling and <code>whileElement</code></li>
<li><a href="https://wix.github.io/Detox/docs/api/device">Detox: device API</a> — the process-wide device handle</li>
<li><a href="https://developer.apple.com/documentation/xctest">XCUITest</a> — the iOS driver Detox wraps</li>
<li><a href="https://developer.android.com/training/testing/espresso">Espresso</a> — the Android driver Detox wraps</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Detox is a grey-box React Native E2E framework. Every test reduces to three verbs — matcher, action, expectation — composed through a lazy <code>element()</code> proxy. It synchronizes automatically with the app's internal queues, so you almost never write <code>sleep()</code>. Under the hood, iOS runs on XCUITest and Android on Espresso, and the small platform seams (scroll start position, <code>reverseTcpPort</code>, APK vs app bundle) are the ones you'll hit in practice.
</div>

### 5.5.11 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Detox Core Concepts</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> What does "grey-box" mean in the context of Detox?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It uses a greyscale screenshot diffing algorithm</button>
<button class="quiz-choice" data-value="B">B) The test driver is injected into the app process itself, giving it privileged knowledge of the app's runtime state (JS bridge, dispatch queues, timers, network) to synchronize automatically</button>
<button class="quiz-choice" data-value="C">C) It is neither open-source nor fully proprietary</button>
<button class="quiz-choice" data-value="D">D) It only runs on physical devices, never on simulators</button>
</div>
<p class="quiz-explanation">Grey-box means the test framework has inside knowledge of the app under test. Appium is black-box — it sees only the OS accessibility tree. Detox ships a native library that is linked into the app, so it can detect when the app is idle and wait deterministically.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> What does <code>element(by.id("send-button"))</code> actually return?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A reference to the native view — fails immediately if no match</button>
<button class="quiz-choice" data-value="B">B) A promise resolving to the native view's coordinates</button>
<button class="quiz-choice" data-value="C">C) A lazy matcher proxy — no lookup happens until an action or expectation is called on it, and the lookup re-runs every time</button>
<button class="quiz-choice" data-value="D">D) An array of all matching elements</button>
</div>
<p class="quiz-explanation">Just like Playwright's <code>Locator</code>, <code>element()</code> is lazy. It stores the matcher and re-resolves against the current UI hierarchy on each action. This is why you can declare element proxies eagerly in page objects without worrying about screen transitions.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> Which statement is TRUE about <code>toBeVisible()</code> vs <code>toExist()</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>toBeVisible()</code> requires the element to be on screen with &gt;75% of its area visible, while <code>toExist()</code> only requires the element to be in the view hierarchy (possibly off-screen)</button>
<button class="quiz-choice" data-value="B">B) They are synonyms</button>
<button class="quiz-choice" data-value="C">C) <code>toBeVisible()</code> is iOS-only</button>
<button class="quiz-choice" data-value="D">D) <code>toExist()</code> asserts the element's parent exists, not the element itself</button>
</div>
<p class="quiz-explanation"><code>toBeVisible()</code> is the stricter, user-facing check — "the user can actually see this". <code>toExist()</code> is a hierarchy check — useful for virtualized lists where rows exist but may be scrolled off. Use <code>toBeVisible</code> by default.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> What is the correct way to "scroll this list until the target row is visible"?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Write a <code>for</code> loop that scrolls 200px and checks <code>isVisible</code></button>
<button class="quiz-choice" data-value="B">B) Use <code>device.scrollUntilVisible()</code></button>
<button class="quiz-choice" data-value="C">C) Add an arbitrary <code>await new Promise(r =&gt; setTimeout(r, 3000))</code> before the assertion</button>
<button class="quiz-choice" data-value="D">D) Use <code>waitFor(target).toBeVisible().whileElement(listId).scroll(200, "down")</code></button>
</div>
<p class="quiz-explanation">The <code>whileElement(...).scroll(...)</code> combinator is Detox's idiomatic polling-scroll. It re-checks the condition between each scroll step and gives up at <code>withTimeout</code>.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> What is the difference between <code>typeText("hello")</code> and <code>replaceText("hello")</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>typeText</code> is iOS-only, <code>replaceText</code> is Android-only</button>
<button class="quiz-choice" data-value="B">B) <code>typeText</code> simulates per-character keystrokes (slower, fires per-char handlers), <code>replaceText</code> sets the value atomically in one shot (faster, skips per-char handlers)</button>
<button class="quiz-choice" data-value="C">C) They are identical — one is just an alias</button>
<button class="quiz-choice" data-value="D">D) <code>replaceText</code> clears the field first, <code>typeText</code> appends</button>
</div>
<p class="quiz-explanation">Use <code>replaceText</code> for fast form filling when the app does not care about keystroke-by-keystroke events. Use <code>typeText</code> when you need autocomplete, validation, or analytics hooks that fire per character to run.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> Which of the following is a platform-specific Detox gotcha you will hit?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) iOS requires <code>reverseTcpPort</code> to reach <code>localhost</code> from the simulator</button>
<button class="quiz-choice" data-value="B">B) Android does not support <code>by.id</code>; you must use <code>by.label</code></button>
<button class="quiz-choice" data-value="C">C) On Android, scrolling from the default vertical start position (0.5) may be interpreted as a long-press on a list row, so Ledger Live's helpers use <code>startPositionY = 0.8</code></button>
<button class="quiz-choice" data-value="D">D) <code>device.takeScreenshot()</code> is unsupported on Android</button>
</div>
<p class="quiz-explanation">This is an infamous upstream issue (wix/Detox#3918). On Android, gestures starting on a pressable row can be eaten as long-presses. Ledger Live's <code>elementHelpers.ts</code> sets <code>startPositionY = 0.8</code> to start scrolls from near the bottom of the container.</p>
</div>

<div class="quiz-score"></div>
</div>

---


## Detox Advanced: Config, Synchronization & Bridge

<div class="chapter-intro">
Chapter 5.5 taught you the grammar. This chapter teaches you the machinery: how Detox is configured, how it stays in sync with the app, how Jest drives it, and — critically — how Ledger Live's custom WebSocket bridge injects Redux state, feature flags, and mock device events into the running mobile app. By the end you will be able to read every line of <code>detox.config.js</code>, <code>jest.config.js</code>, and the bridge server/client, and you will know when to reach for <code>device.disableSynchronization()</code> (spoiler: almost never).
</div>

### 5.6.1 `detox.config.js` — The Top-Level Map

Every Detox project has one configuration file at its root. It has six top-level sections:

- `testRunner` — which runner to use (Jest here) and its options
- `apps` — the *things* to install (builds, binaries)
- `devices` — the *things* to run on (simulators, emulators)
- `configurations` — pairings of an app with a device (`ios.sim.debug`, etc.)
- `behavior` — lifecycle defaults (reinstall? clean up? expose globals?)
- `artifacts`, `session`, `logger` — I/O defaults

Here is the Ledger Live file, annotated section by section.

**File: `e2e/mobile/detox.config.js`** — the config itself lives in the `e2e/mobile/` peer workspace, but the paths it emits (binaries, `ENVFILE`) still point back into `apps/ledger-live-mobile/ios` and `apps/ledger-live-mobile/android` because the *app* still belongs to the app workspace.

```js
// lines 1-12 — architecture constants + cross-workspace paths
const path = require("path");
const iosArch = "arm64";
// NOTE: Pass CI=1 if you want to build locally when you don't have a mac M1.
const androidArch = process.env.CI ? "x86_64" : "arm64-v8a";
const SCHEME = "ledgerlivemobile";

const rootDir = path.resolve(__dirname, "../..");                        // repo root from e2e/mobile/
const iosDir = path.join(rootDir, "apps/ledger-live-mobile/ios");
const iosBuildDir = path.join(iosDir, "build");
const androidDir = path.join(rootDir, "apps/ledger-live-mobile/android");
const ENV_FILE_MOCK            = path.join("apps", "ledger-live-mobile", ".env.mock");
const ENV_FILE_MOCK_PRERELEASE = path.join("apps", "ledger-live-mobile", ".env.mock.prerelease");
```

**Why this matters:** CI runners are x86_64, Apple Silicon dev machines are arm64. Gradle builds an ABI-specific APK, and picking the wrong one causes `INSTALL_FAILED_NO_MATCHING_ABIS`. The `CI=1` escape hatch lets you force x86 on a laptop. The `rootDir` / `iosDir` / `androidDir` constants anchor all binary paths back to the app workspace — the config file lives in `e2e/mobile/` but the `.xcworkspace`, the Gradle project, and the `.env.mock` files stay with the app.

```js
// testRunner
testRunner: {
  $0: "jest",                              // the test runner executable
  args: {
    config: "jest.config.js",              // resolved relative to e2e/mobile/
  },
  jest: {
    setupTimeout: 500000,                  // 500s before giving up on a worker's setup
    teardownTimeout: 120000,
  },
  noRetryArgs: ["json", "outputFile"],
  retries: 0,                              // no Detox-level retries (we rely on CI-level retries)
  forwardEnv: true,                        // forward DETOX_CONFIGURATION (and the rest of env) to Jest workers
},
```

**Why `setupTimeout: 500000`?** Each Jest worker does `device.launchApp()` in `beforeAll` — on Android that can mean booting a cold emulator, installing the APK, and starting Metro. Half a second is fine on CI; half a millennium (500s) is the safety net.

```js
// logger + behavior
logger: {
  level: process.env.DEBUG_DETOX ? "trace" : "info",
},
behavior: {
  init: {
    reinstallApp: true,                    // blow away prior install + data each worker
    exposeGlobals: false,                  // do NOT inject by/element/expect as globals
  },
  launchApp: "auto",                       // auto-launch before first test of each worker
  cleanup: {
    shutdownDevice: false,                 // leave device up; CI shards own their own lifecycle
  },
  extends: "detox-allure2-adapter/preset-detox",  // Allure adapter preset
},
```

- `reinstallApp: true` — a **fresh install** per worker boot. Eliminates state bleed between CI runs.
- `exposeGlobals: false` — Detox can inject `by`, `element`, `expect`, `waitFor` as globals. Ledger Live disables that and imports them explicitly — better TypeScript ergonomics and stricter linting.
- `launchApp: "auto"` — before the first test in each file, call `device.launchApp()`. The suite's `setup.ts` in `beforeAll` then relaunches with extra args + the bridge port.
- `cleanup.shutdownDevice: false` — never kill the simulator/emulator on teardown. Device lifecycle is owned by the CI shard (or your local session), which amortises boot time across reruns.
- `extends: "detox-allure2-adapter/preset-detox"` — composes in the Allure 2 adapter's behavior preset. This is **new** vs the older guide description and is what wires screenshots/attachments into the Allure pipeline.

```js
// apps — builds, each ENVFILE still points into apps/ledger-live-mobile/
apps: {
  "ios.debug": {
    type: "ios.app",
    build: `export ENVFILE=${ENV_FILE_MOCK} && xcodebuild ARCHS=${iosArch} ONLY_ACTIVE_ARCH=YES \
            -workspace ios/ledgerlivemobile.xcworkspace -scheme ledgerlivemobile \
            -configuration Debug -sdk iphonesimulator -derivedDataPath ios/build`,
    binaryPath: getIosBinary("Debug"),     // => apps/ledger-live-mobile/ios/build/Build/Products/Debug-iphonesimulator/ledgerlivemobile.app
  },
  "ios.staging":    { /* Staging    configuration, ENVFILE = .env.mock */ },
  "ios.release":    { /* Release    configuration, ENVFILE = .env.mock */ },
  "ios.prerelease": { /* Release    configuration, ENVFILE = .env.mock.prerelease */ },
  "android.debug": {
    type: "android.apk",
    build: `cd android && ENVFILE=${ENV_FILE_MOCK} SENTRY_DISABLE_AUTO_UPLOAD=true \
            ./gradlew app:assembleDebug app:assembleAndroidTest \
            -DtestBuildType=debug -PreactNativeArchitectures=${androidArch} && cd ..`,
    binaryPath: getAndroidBinary("debug"),          // app-<arch>-debug.apk under apps/ledger-live-mobile/android/
    testBinaryPath: getAndroidTestBinary("debug"),  // app-debug-androidTest.apk
  },
  "android.release":    { /* ./gradlew app:assembleDetox …          ENVFILE = .env.mock */ },
  "android.prerelease": { /* ./gradlew app:assembleDetoxPreRelease  ENVFILE = .env.mock.prerelease */ },
},
```

**Key fields:**

- `type` — `"ios.app"` or `"android.apk"`. Determines the install pipeline.
- `build` — shell command that Detox will run when you pass `detox build`. Here it is `xcodebuild` on iOS and Gradle on Android. `ENVFILE` points at `apps/ledger-live-mobile/.env.mock` or `apps/ledger-live-mobile/.env.mock.prerelease` — the names are unchanged after the workspace migration.
- `binaryPath` — where the artefact lands after build, resolved via the `getIosBinary` / `getAndroidBinary` helpers against `apps/ledger-live-mobile/ios|android`.
- `testBinaryPath` (Android only) — the separate instrumentation APK (Espresso test harness).

There are four iOS variants and three Android variants because each `.env` profile (`.env.mock`, `.env.mock.prerelease`) produces a different bundle, and each Xcode configuration (`Debug`/`Staging`/`Release`) enables different compiler flags. The Android release variants use the `detox` and `detoxPreRelease` build types, matching their Gradle `assembleDetox*` tasks.

```js
// lines 73-116 — devices
devices: {
  simulator: { type: "ios.simulator", device: { name: "iOS Simulator" } },
  simulator2: { type: "ios.simulator", device: { name: "iOS Simulator 2" } },
  simulator3: { type: "ios.simulator", device: { name: "iOS Simulator 3" } },
  emulator: {
    type: "android.emulator",
    device: { avdName: "Android_Emulator" },
    gpuMode: "swiftshader_indirect",
    headless: !!process.env.CI,
  },
  emulator2: { /* same but avdName: "Android_Emulator_2" */ },
  emulator3: { /* same but avdName: "Android_Emulator_3" */ },
},
```

**Why three simulators and three emulators?** The mobile suite can run with `JEST_MAX_WORKERS=3`. Each worker needs its own device — you cannot share a simulator between parallel Jest workers (two tests writing to the same RN state at the same moment would be chaos). The `jest.environment.ts` (see 35.6) looks at `JEST_WORKER_ID` and swaps the device to `simulator${id}` / `emulator${id}`.

- `gpuMode: "swiftshader_indirect"` — software GPU. On headless CI there is no real GPU; SwiftShader gives predictable rendering.
- `headless: !!process.env.CI` — no window in CI.

```js
// lines 117-148 — configurations (the things you actually pass to `detox test`)
configurations: {
  "ios.sim.debug":        { device: "simulator", app: "ios.debug" },
  "ios.sim.staging":      { device: "simulator", app: "ios.staging" },
  "ios.sim.release":      { device: "simulator", app: "ios.release" },
  "ios.sim.prerelease":   { device: "simulator", app: "ios.prerelease" },
  "android.emu.debug":    { device: "emulator",  app: "android.debug" },
  "android.emu.release":  { device: "emulator",  app: "android.release" },
  "android.emu.prerelease": { device: "emulator", app: "android.prerelease" },
},
```

These are the names you pass on the command line: `detox test -c ios.sim.release`. Each one is just a (device, app) pair. The `jest.environment.ts` post-processes the chosen device to swap it for `simulator2`, `simulator3`, etc., when running with multiple workers.

> **What's missing here?** `artifacts` and `session` are both absent, so Detox uses defaults. `artifacts` would control where screenshots/videos/logs go (the Allure adapter in `jest.config.js` handles screenshots instead). `session` would pin WebSocket ports between Detox and the app — we don't pin them because our own bridge uses a dynamic free port (see `commonHelpers.ts`).

> **Reference:** https://wix.github.io/Detox/docs/config/overview

### 5.6.2 The `detox test` CLI

Once configured, you run:

```bash
# Build the iOS debug variant
detox build -c ios.sim.debug

# Run tests on the iOS debug variant
detox test -c ios.sim.debug

# Run one spec only
detox test -c ios.sim.debug e2e/specs/deeplinks.spec.ts

# Keep the simulator booted and the app installed (faster iteration locally)
detox test -c ios.sim.debug --reuse --cleanup=false

# Run with 3 Jest workers (needs simulator2/simulator3 available)
JEST_MAX_WORKERS=3 detox test -c ios.sim.debug

# Write artefacts to a custom folder
detox test -c ios.sim.debug --artifacts-location ./my-artifacts
```

**Important flags:**

| Flag | Purpose |
|------|---------|
| `-c, --configuration <name>` | Which configuration from `detox.config.js` to use |
| `--reuse` | Don't reinstall the app; reuse the simulator's existing install. Huge iteration-time win |
| `--cleanup` | Kill the device after tests. Invert with `--cleanup=false` (default depends on `behavior.cleanup`) |
| `--artifacts-location <path>` | Where screenshots, videos, logs go |
| `--record-videos all`, `--record-logs all` | Force artefact capture regardless of pass/fail |
| `--loglevel <trace\|debug\|info\|warn\|error>` | Detox logger verbosity. Equivalent to `DEBUG_DETOX=1` for the `trace` level in our config |
| `--device-launch-args "..."` | Extra launch args appended to the app |
| `--workers <n>` | Shorthand for `JEST_MAX_WORKERS=n` in Ledger Live's setup |
| `--headless` | Override the config's headless flag (Android) |

> **Local iteration tip:** once you have the app installed once, drop `--cleanup` and pass `--reuse`:
> ```bash
> detox test -c ios.sim.debug --reuse --cleanup=false e2e/specs/send/send.spec.ts
> ```
> This skips the ~30s reinstall and ~10s simulator boot.

> **Reference:** https://wix.github.io/Detox/docs/guide/running-tests

### 5.6.3 How Detox Synchronizes — Idle Detection

The single most important thing to understand about Detox is **what it watches to decide "the app is idle, I can act now"**.

Before any action or expectation, Detox waits for all of the following to reach an idle state:

1. **The JavaScript thread** — the RN JS bundle must be done with all currently queued tasks (the RN JS runtime exposes hooks for this).
2. **The native dispatch queues** — iOS main queue + registered serial queues; Android main looper. No pending tasks.
3. **The native animation frame pump** — `CADisplayLink` on iOS, `Choreographer` on Android. No animations in flight.
4. **Active timers** — `setTimeout`, `setInterval`, `NSTimer`, Android `Handler.postDelayed`. Detox waits for short timers to fire; long ones are ignored (configurable).
5. **In-flight network requests** — any HTTP request not on the URL blacklist must complete.
6. **The React Native bridge** — all bridge-queued calls between JS and native must drain.

When all six queues drain, the app is *idle* and Detox performs the action. If any queue never drains within the test timeout, the action fails with:

> Test Failed: App is busy. Trying to synchronize...

This is informative, not a bug. The error message usually includes which queue is busy:

> App synchronization debug: {dispatch_queue: com.example.long_running_queue has 1 job(s); ui: 2 animations in progress}

**The URL blacklist — the single most important escape hatch.** If your app makes a long-polling or streaming request (e.g., analytics beacon, push notifications, feature flag streaming), Detox will wait forever for it to finish. The fix is to mark those URLs as "don't synchronize on me":

```ts
// Ledger Live — e2e/helpers/commonHelpers.ts (inside launchApp)
detoxURLBlacklistRegex:
  '\\(".*sdk.*.braze.*",".*.googleapis.com/.*",".*clients3.google.com.*",' +
  '".*tron.coin.ledger.com/wallet/getBrokerage.*"\\)',
```

That ugly string is the list of regexes Detox will NOT wait for. The format is a Detox-specific escaped tuple: `\("regex1","regex2",...\)`. Four regexes here:

- `.*sdk.*.braze.*` — Braze analytics SDK endpoints
- `.*.googleapis.com/.*` — Google APIs (push notification token registration, fonts)
- `.*clients3.google.com.*` — Google Play captive-portal detection (Android)
- `.*tron.coin.ledger.com/wallet/getBrokerage.*` — a specific long-polling endpoint

You can also set this at runtime:

```ts
await device.setURLBlacklist([".*analytics.*", ".*telemetry.*"]);
```

> **Reference:** https://wix.github.io/Detox/docs/articles/how-detox-works

### 5.6.4 Synchronization Troubles — When to Give Up

Sometimes a screen has an infinite animation, or a custom native module that Detox can't see, and the app is simply never idle. Symptoms:

- `App is busy. Trying to synchronize...` errors that persist even after blacklisting
- Tests that hang at the same screen every time
- `detox.log` showing the same queue stuck for >30s

**The nuclear option** — turn off synchronization entirely:

```ts
await device.disableSynchronization();
// ... interact with the screen manually using waitFor() ...
await device.enableSynchronization();
```

While disabled, Detox will not wait for anything before an action. You must manually `waitFor(...).toBeVisible().withTimeout(...)` every element. This is strictly worse than letting Detox synchronize — it brings back all the flakiness Detox was designed to eliminate — so do it narrowly, around exactly the offending screen, and re-enable immediately.

**Diagnostic recipe:**

```ts
await device.enableSynchronization();       // ensure on
try {
  await waitFor(element(by.id("infinite-spinner-screen")))
    .toBeVisible().withTimeout(5000);
} catch (e) {
  // capture state for debugging
  await device.takeScreenshot("busy-sync");
  console.log(await device.generateViewHierarchyXml());
  throw e;
}
```

If the view hierarchy shows an animated Lottie loader that never stops: add its URL to the blacklist (if it fetches remote JSON), wrap the interaction in `disableSynchronization`, or fix the animation in the app to terminate.

> **Reference:** https://wix.github.io/Detox/docs/troubleshooting/synchronization

### 5.6.5 Jest as the Runner — `jest.config.js`

Detox does not run tests itself; it delegates to a runner. Ledger Live uses **Jest**, because (a) the React Native ecosystem already ships with it, (b) it handles TypeScript via SWC, and (c) the `jest-allure2-reporter` plugin gives us rich Allure reports for free.

**File: `e2e/mobile/jest.config.js`** — the important bits (> **Verify:** the exact body against the canonical file; `rootDir` and `setupFilesAfterEnv` paths are now workspace-local under `e2e/mobile/`, not `<app>/e2e/`):

```js
// lines 35-80 — the config itself
module.exports = async () => ({
  rootDir: ".",                                                           // (1)
  maxWorkers: process.env.JEST_MAX_WORKERS
    ? parseInt(process.env.JEST_MAX_WORKERS, 10)
    : 1,                                                                   // (2)
  transform: {
    "^.+\\.m?jsx?$": "babel-jest",
    "^.+\\.(ts|tsx)?$": ["@swc/jest", {                                    // (3)
      jsc: {
        target: "es2022",
        parser: {
          syntax: "typescript",
          tsx: true,
          decorators: true,                                                // (4)
          dynamicImport: true,
        },
        transform: { react: { runtime: "automatic" } },
      },
      sourceMaps: "inline",
      module: { type: "commonjs" },
    }],
  },
  setupFilesAfterEnv: ["<rootDir>/setup.ts"],                          // (5)
  testTimeout: 300000,                                                     // (6)
  testMatch: ["<rootDir>/specs/**/*.spec.ts"],                         // (7)
  reporters: [
    "detox/runners/jest/reporter",                                         // (8)
    ["jest-allure2-reporter", jestAllure2ReporterOptions],
  ],
  globalSetup: "<rootDir>/jest.globalSetup.ts",                        // (9)
  testEnvironment: "<rootDir>/jest.environment.ts",                    // (10)
  testEnvironmentOptions: {
    eventListeners: [
      "jest-metadata/environment-listener",
      "jest-allure2-reporter/environment-listener",
      ["detox-allure2-adapter", detoxAllure2AdapterOptions],
    ],
  },
  transformIgnorePatterns: [
    `node_modules/.pnpm/(?!(${transformIncludePatterns.join("|")}))`,       // (11)
  ],
  verbose: true,
  resetModules: true,                                                      // (12)
});
```

Line by line:

1. **`rootDir`** — in the migrated workspace the config is already at the root of `e2e/mobile/`, so `<rootDir>` resolves to `e2e/mobile/`. Any legacy reference like `"<rootDir>/e2e/..."` is retained only for the residual specs still living under `apps/ledger-live-mobile/e2e/`.
2. **`maxWorkers`** — **default is `1`**. Parallel E2E is off by default because each worker needs its own simulator/emulator, and booting three emulators locally is painful. Set `JEST_MAX_WORKERS=3` on CI where three emulators are pre-booted.
3. **`@swc/jest`** — not `ts-jest` (too slow). SWC compiles TypeScript to JS 20× faster. `target: es2022` keeps async/await native.
4. **`decorators: true`** — enables `@Step` (see 35.8). Without this flag, TypeScript decorators are a syntax error.
5. **`setupFilesAfterEnv: ["<rootDir>/setup.ts"]`** — runs `setup.ts` *after* Jest's globals (`describe`, `it`, `expect`) are injected. This is where Ledger Live wires up its globals: `global.app`, `global.Step`, the page-object helpers (see 35.6).
6. **`testTimeout: 300000`** — 300 seconds (5 minutes) per test. Mobile is slow: launching the app, importing accounts over the bridge, mocking device events, rendering onboarding screens — it adds up.
7. **`testMatch`** — any `*.spec.ts` under `e2e/specs/`.
8. **`reporters`** — Detox's Jest reporter logs action traces; `jest-allure2-reporter` generates the Allure XML/JSON that populates the reporting dashboard.
9. **`globalSetup`** — runs *once* per process, before all workers. Here it just delegates to Detox's own setup (boot the device, install the app if needed). See 35.6.
10. **`testEnvironment`** — Ledger Live ships a **custom test environment** that extends Detox's. This is what lets us support multiple workers without hard-coding three copies of the config (see 35.6).
11. **`transformIgnorePatterns`** — by default Jest does NOT transform `node_modules`. This regex carves an exception for packages published as ESM (`ky`, `@mysten+ledgerjs-hw-app-sui`) so Jest can compile them.
12. **`resetModules: true`** — between tests, clear the module registry. This prevents Redux store singletons from leaking state across tests.

> **Reference:** https://jestjs.io/docs/configuration

### 5.6.6 `jest.globalSetup.ts` and `jest.environment.ts`

Jest distinguishes two lifecycle hooks:

- **`globalSetup`** — a single async function, runs **once** before the whole test run begins, in the master process.
- **`testEnvironment`** — a class instantiated **per worker**, `setup()` runs before each test file on that worker, `teardown()` after.

**`jest.globalSetup.ts`** is tiny:

```ts
// e2e/jest.globalSetup.ts
import { globalSetup } from "detox/runners/jest";

export default async () => {
  await globalSetup();
};
```

Detox's built-in `globalSetup` handles the heavy lifting: reading `detox.config.js`, deciding which device to reuse, deciding whether to install the app, etc. We just call it.

**`jest.environment.ts`** is where the interesting Ledger Live customization happens:

```ts
// e2e/jest.environment.ts
import { config as detoxConfig } from "detox/internals";
// @ts-expect-error detox doesn't provide type declarations for this module
import DetoxEnvironment from "detox/runners/jest/testEnvironment";

import { logMemoryUsage } from "./helpers/commonHelpers";

export default class TestEnvironment extends DetoxEnvironment {
  declare global: typeof globalThis;

  async setup() {
    const workerId = Number(process.env.JEST_WORKER_ID ?? 1);            // (1)
    if (workerId > 1) await this.setupDeviceForSecondaryWorker(workerId); // (2)
    await super.setup();                                                  // (3)
  }

  private async setupDeviceForSecondaryWorker(workerId: number) {
    const configName = process.env.DETOX_CONFIGURATION;
    if (!configName) throw new Error("Missing DETOX_CONFIGURATION...");

    const detoxConfigModule = await import("../detox.config.js");
    const detoxConfigFile =
      "default" in detoxConfigModule ? detoxConfigModule.default : detoxConfigModule;
    const targetConfig = detoxConfigFile.configurations[configName];
    if (!targetConfig || !targetConfig.device) {
      throw new Error(`Detox configuration "${configName}" not found...`);
    }

    const deviceName = `${targetConfig.device}${workerId}`;               // (4) "simulator" -> "simulator2"
    const targetDevice = detoxConfigFile.devices[deviceName];
    if (!targetDevice) {
      throw new Error(`Device configuration not found for ${deviceName}...`);
    }

    Object.assign(detoxConfig, { device: targetDevice });                 // (5) monkey-patch Detox's runtime config
  }

  async handleTestEvent(event: any, state: any) {
    await super.handleTestEvent(event, state);

    if (["hook_failure", "test_fn_failure"].includes(event.name)) {
      this.global.IS_FAILED = true;                                       // (6)
    }
    if (event.name === "run_start") {
      await logMemoryUsage();                                             // (7)
    }
  }
}
```

What it does:

1. **`JEST_WORKER_ID`** — Jest sets this env var on each worker subprocess. Worker 1 uses the default device from config; workers 2+ need to swap.
2. For workers 2+, we load the config file, look up the configuration (e.g. `ios.sim.debug`), find its device (`simulator`), and ...
3. ... swap the runtime device to `simulator2` / `simulator3` (line 4) by monkey-patching `detox/internals.config.device` (line 5). Then call `super.setup()` so Detox boots the correct device.
4. **The naming convention is load-bearing:** `simulator + workerId` → `simulator2`, `simulator3`. This is why `detox.config.js` has exactly `simulator`, `simulator2`, `simulator3`. You can extend to 4+ workers by adding devices and AVDs.
5. **`IS_FAILED`** — on any hook failure or test failure, set a global boolean. `setup.ts` reads it in `afterAll` and attaches app logs to Allure only when the test failed (writing logs on every test would drown the report).
6. **`logMemoryUsage()`** — at the start of each test run, `top` is sampled for the current PID and memory is attached to Allure. Debugging memory leaks in a long-running E2E CI run is much easier with this timeline.

> **Why extend vs wrap?** Detox's `testEnvironment` already plugs into Jest's lifecycle, reads `detox.config.js`, and manages the Detox client/connection. Extending via `super.setup()` preserves all that behavior; we only inject the worker-ID device swap and the two observability hooks. Wrapping or re-implementing would fight Detox's internal state machine.

> **Reference:** https://jestjs.io/docs/configuration#testenvironment-string

### 5.6.7 The WebSocket Bridge — Why Detox Alone Isn't Enough

Detox drives the UI. It does not know about Redux, feature flags, or Ledger's hardware device events. For most RN apps, that is fine — you seed state by tapping through the UI. For Ledger Live it isn't, because:

- Onboarding the app from scratch every test would take 2 minutes (and require BLE pairing).
- Feature flags need to be overridden per test without a real Firebase fetch.
- Device events (USB plug, APDU responses) must be **mocked**, not driven by a real Ledger.
- Redux state (accounts, settings) must be pre-populated with test fixtures.

So Ledger Live layers a **custom WebSocket bridge** on top of Detox:

- **Server** — runs in the test process (Jest worker). Started in `commonHelpers.ts#launchApp` on a free port (default 8099). Sends commands.
- **Client** — linked into the RN app under `src/e2e/`. Connects to the server on boot. Receives commands, dispatches into Redux / emits native events / calls the navigation controller.

```
┌──────────────────────┐            ┌───────────────────────┐
│ Jest worker (Node)   │  WebSocket │ RN App on device      │
│                      │ <────────> │                       │
│ bridge/server.ts     │   ws://... │ bridge/client.ts      │
│  - postMessage()     │            │  - dispatch(importX)  │
│  - pendingAcks       │            │  - navigate()         │
│  - ws.Server         │            │  - setOverride(flag)  │
└──────────────────────┘            └───────────────────────┘
         │                                    │
         │          Test code calls:          │
         │     await importAccounts([...])    │
         │     await mockDeviceEvent(...)     │
         │     await setFeatureFlags({...})   │
         ▼                                    ▼
```

#### 35.7.1 Server side

**File: `e2e/mobile/bridge/server.ts`** (excerpt — the canonical server is ~307 lines and differs from the legacy copy still in `apps/ledger-live-mobile/e2e/bridge/`):

```ts
// lines 61-83 — init: open a WebSocket server
export function init(port = 8099, onConnection?: () => void) {
  webSocket.wss = new WebSocket.Server({ port });
  webSocket.messages = {};
  log(`Start listening on localhost:${port}`);

  webSocket.wss.on("connection", (ws: WebSocket) => {
    log(`Client connected`);
    onConnection && onConnection();
    webSocket.ws?.close();           // close any prior client (reconnect case)
    webSocket.ws = ws;
    ws.on("message", onMessage);
    ws.on("close", () => { webSocket.ws = undefined; });
    // Re-send any messages that were queued while disconnected
    if (Object.keys(webSocket.messages).length !== 0) {
      Object.values(webSocket.messages).forEach(m => postMessage(m));
    }
  });
}
```

One bridge server per test process. The `global.webSocket` object (declared in `setup.ts`) holds the singleton so every helper in `bridge/server.ts` can reach it.

**The ACK pattern — every message waits for a reply:**

```ts
// lines 237-251 — postMessageAndWaitForAck
function postMessageAndWaitForAck(message: MessageData, timeout = RESPONSE_TIMEOUT): Promise<void> {
  return new Promise((resolve, reject) => {
    const timeoutId = setTimeout(() => {
      pendingAcks.delete(message.id);
      reject(new Error(`Timeout while waiting for ACK for ${message.type} (${message.id})`));
    }, timeout);

    pendingAcks.set(message.id, { resolve, timeoutId });
    postMessage(message);
  });
}

// lines 253-280 — onMessage handles the ACK reply
function onMessage(messageStr: string) {
  const msg: ServerData = JSON.parse(messageStr);
  switch (msg.type) {
    case "ACK": {
      delete webSocket.messages[msg.id];
      const pendingAck = pendingAcks.get(msg.id);
      if (pendingAck) {
        clearTimeout(pendingAck.timeoutId);
        pendingAcks.delete(msg.id);
        pendingAck.resolve();
      }
      break;
    }
    // ...
  }
}
```

**Why ACKs matter:** without them, `await setFeatureFlag(...)` would resolve as soon as the message left the server's socket — *before* the RN client had actually run the Redux dispatch. The next test line would race against the dispatch. With ACKs, the client only sends `{ type: "ACK", id }` *after* running the handler (see `bridge/client.ts` line 199), so the server's promise only resolves after the state change is live.

**The message catalogue** (`e2e/mobile/bridge/types.ts` — verify exact names against this file; the legacy `apps/ledger-live-mobile/e2e/bridge/types.ts` is being retired):

| Message | Direction | Purpose |
|---------|-----------|---------|
| `open` | server → client | Open a BLE connection to a mock device |
| `acceptTerms` | server → client | Mark T&Cs as accepted in Redux |
| `importSettings` | server → client | Seed settings state |
| `importAccounts` | server → client | Seed accounts state from a fixture |
| `importBle` | server → client | Seed known BLE devices |
| `addUSB` | server → client | Emit a fake USB device-connect native event |
| `addKnownSpeculos` | server → client | Add a Speculos-emulated device to the known list and set DEVICE_PROXY_URL |
| `removeKnownSpeculos` | server → client | Remove a Speculos device, clear DEVICE_PROXY_URL |
| `overrideFeatureFlag` | server → client | Override a single feature flag (merge into existing overrides) |
| `overrideFeatureFlags` | server → client | Replace all feature flag overrides at once |
| `setGlobals` | server → client | Inject globals into the RN runtime (rarely used) |
| `navigate` | server → client | Call the React Navigation navigator programmatically |
| `mockDeviceEvent` | server → client | Push one or more device events into the mock subject (APDU responses, connect events, etc.) |
| `getLogs` / `appLogs` | ↔ | Request log dump; client replies with `appLogs` payload |
| `getFlags` / `appFlags` | ↔ | Request resolved feature flags; client replies |
| `getEnvs` / `appEnvs` | ↔ | Request live-env config; client replies |
| `swapSetup` / `swapSetupDone` | ↔ | Set up Swap mock env (sync handshake) |
| `ACK` | client → server | Acknowledge any server message after handling |

#### 35.7.2 Client side

**File: `e2e/mobile/bridge/client.ts`** (excerpt):

> **Verify:** message types and the exact switch/case against the canonical `e2e/mobile/bridge/types.ts` before changing a handler — the server diverges slightly from the legacy.

```ts
// lines 34-74 — init: connect to the server from inside the app
export function init() {
  const wsPort = LaunchArguments.value()["wsPort"] || "8099";                   // (1)
  const mock = LaunchArguments.value()["mock"];
  const disable_broadcast = LaunchArguments.value()["disable_broadcast"];

  if (mock == "0") {
    setEnv("MOCK", ""); setEnv("MOCK_COUNTERVALUES", ""); Config.MOCK = "";
  }
  setEnv("DISABLE_TRANSACTION_BROADCAST", disable_broadcast != "0");

  if (ws) ws.close();

  const ipAddress = Platform.OS === "ios" ? "localhost" : "10.0.2.2";           // (2)
  const path = `${ipAddress}:${wsPort}`;
  ws = new WebSocket(`ws://${path}`);

  ws.onopen = () => { retryCount = 0; };
  ws.onerror = error => {
    if (retryCount < maxRetries) {
      retryCount++;
      const retryTimeout = retryDelay * Math.pow(2, retryCount);                // (3) exponential backoff
      setTimeout(init, retryTimeout);
    }
  };
  ws.onmessage = onMessage;
}
```

1. **`LaunchArguments.value()["wsPort"]`** — reads the `--wsPort=NNNN` launch argument that `device.launchApp({ launchArgs: { wsPort } })` set from the host side. This is how the RN app learns which port to connect to.
2. **The iOS/Android IP split** — **the single most important line in this file**. On iOS simulator, `localhost` inside the simulated device resolves to the host Mac's loopback, because the simulator runs as a macOS process sharing the same network stack. On Android emulator, the guest has its own kernel and network; `10.0.2.2` is the magic IP that the QEMU NIC maps to the host's loopback. Without this split, the bridge can't connect on Android.
3. **Exponential backoff** — the app might boot before the server is ready (race with `commonHelpers.launchApp`, which starts the server *before* calling `device.launchApp`, but still). Five retries, doubling delay each time.

**The message handler:**

```ts
// lines 76-203 (excerpt) — onMessage dispatches the right side effect
switch (msg.type) {
  case "acceptTerms":
    acceptGeneralTerms(store);
    break;
  case "importAccounts":
    store.dispatch(await importAccountsRaw({ active: msg.payload }));
    break;
  case "mockDeviceEvent":
    msg.payload.forEach(e => mockDeviceEventSubject.next(e));  // push into RxJS subject
    break;
  case "importSettings":
    store.dispatch(importSettings(msg.payload));
    break;
  case "overrideFeatureFlag": {
    const { id, value } = msg.payload as { id: FeatureId; value: Feature | undefined };
    store.dispatch(setOverride({ key: id, value }));
    break;
  }
  case "navigate":
    navigate(msg.payload, {});
    break;
  case "addUSB":
    DeviceEventEmitter.emit("onDeviceConnect", msg.payload);   // native event
    break;
  case "addKnownSpeculos": {
    const { address, model } = JSON.parse(msg.payload);
    await disconnectAllSpeculosSessions();
    // ... update known devices + set DEVICE_PROXY_URL env ...
    setEnv("DEVICE_PROXY_URL", address);
    break;
  }
  // ... many more ...
}
postMessage({ type: "ACK", id: msg.id });     // ALWAYS ACK last
```

Three side-effect shapes:

- **Redux dispatch** — `importAccounts`, `importSettings`, `overrideFeatureFlag`, etc. Modifies app state directly.
- **RxJS subject emission** — `mockDeviceEvent`. The real-app device action pipeline subscribes to `mockDeviceEventSubject`, so pushing events into it is indistinguishable from a real device sending APDU responses.
- **Native event emission** — `addUSB` uses `DeviceEventEmitter.emit("onDeviceConnect", ...)`, the exact same native event a real USB-connect would fire.

Every handler ends by posting an `ACK` back to the server with the original message id.

#### 35.7.3 `device.reverseTcpPort` — the Android-only glue

In `setup.ts`:

```ts
beforeAll(async () => {
  const port = await launchApp();
  await device.reverseTcpPort(8081);      // Metro bundler
  await device.reverseTcpPort(port);      // bridge server
  await device.reverseTcpPort(52619);     // dummy app for webview tests
  // ...
});
```

`device.reverseTcpPort(p)` tells the emulator: "when the guest connects to `localhost:p`, route that connection to the host's `localhost:p`". It is an `adb reverse tcp:p tcp:p` under the hood.

**Why only Android?**

- iOS simulator shares the host's network, so `localhost` already works. `reverseTcpPort` is a no-op on iOS (it's either missing in the iOS driver or silently skipped — we call it regardless and it's harmless).
- Android emulator has an isolated network; without `adb reverse`, `localhost` inside the emulator points at the emulator itself.
- On Android we also call `10.0.2.2` from the client as a second path (see client line 52). With `reverseTcpPort` established, both `10.0.2.2:port` and `localhost:port` work; Ledger Live standardized on `10.0.2.2`.

**Three forwarded ports in Ledger Live:**

| Port | Used for |
|------|----------|
| `8081` | Metro bundler (development-only, for `--devmenu` style reloads) |
| `port` (dynamic) | The bridge WebSocket server — port chosen by `findFreePort()` |
| `52619` | A local "dummy app" used by WebView tests |

### 5.6.8 `@Step` — Allure Step Decorators on POM Methods

Ledger Live uses the [Page Object Model](#pom) pattern. Each page object has methods like `tapSendButton()`, `waitForPortfolioToLoad()`. For Allure reporting, we want each method to show up as a named step with its own duration and sub-assertions.

The `jest-allure2-reporter/api` package exposes a `Step` decorator factory:

```ts
import { Step } from "jest-allure2-reporter/api";

class SendPage {
  @Step("Open the Send flow")
  async openSendFlow() {
    await tapById("portfolio-send-button");
    await waitForElementById("send-flow-screen");
  }

  @Step("Enter amount $0")
  async enterAmount(amount: string) {
    await replaceTextById("amount-input", amount);
  }
}
```

What happens at runtime:

- The decorator wraps the method in `allure.step(name, async () => originalMethod.apply(this, args))`.
- The step name supports positional parameter interpolation — `$0` = first arg, `$1` = second, etc.
- In the Allure report, the test tree shows the step hierarchy with timings and nested assertions.

**Why this needs `experimentalDecorators: true`.** Open `e2e/tsconfig.json`:

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "allowJs": true,
    "experimentalDecorators": true
  }
}
```

TypeScript's "legacy" decorator syntax (`@decorator`) is still behind an experimental flag. The newer stage-3 decorators have a different signature and are not what `jest-allure2-reporter` ships. Mixing the two would be a compile error. Ledger Live extends the base app `tsconfig.json` but layers this flag for the `e2e/` subtree only — tests see decorators, app code does not.

The corresponding SWC flag is in `jest.config.js`:

```js
parser: {
  syntax: "typescript",
  tsx: true,
  decorators: true,           // <- must match tsconfig
  dynamicImport: true,
},
```

Both flags must be on for the transform to emit decorator metadata.

> **In `setup.ts`**, `Step` is also exposed as a **global** so page objects can use it without importing — see `Object.defineProperty(globalThis, "Step", { value: Step, ... })`. This keeps page-object files terse.

### 5.6.9 `$TmsLink("B2CQA-XXX")` — Jira/Xray Linkage

Every spec and every `it` block in the Ledger Live mobile suite has one or more TMS (Test Management System) links. These are the Xray test case IDs in Jira:

```ts
// e2e/specs/deeplinks.spec.ts
import { knownDevices } from "../models/devices";

$TmsLink("B2CQA-1837");                         // (1) test-file-level link
describe("DeepLinks Tests", () => {
  beforeAll(async () => {
    await app.init({ userdata: "...", /* ... */ });
  });

  it("should open My Ledger page", async () => {
    await app.manager.openViaDeeplink();
    await app.manager.expectManagerPage();
  });
  // ...
});
```

1. **`$TmsLink("B2CQA-1837")`** — a global helper (set up in `setup.ts`) that attaches a TMS link to all subsequent tests in the file via Allure's `allure.link()` API. The Jira ticket prefix `B2CQA-` identifies the B2C QA project; the number is the Xray test case ID.

The `jest-allure2-reporter` config in `jest.config.js` (lines 1-23) turns that raw ID into a clickable URL:

```js
testCase: {
  links: {
    issue: "https://ledgerhq.atlassian.net/browse/{{name}}",
    tms:   "https://ledgerhq.atlassian.net/browse/{{name}}",
  },
  // ...
},
```

`{{name}}` is the template placeholder that the reporter replaces with the value passed to `$TmsLink`. So `$TmsLink("B2CQA-1837")` produces a clickable link to `https://ledgerhq.atlassian.net/browse/B2CQA-1837` in the Allure report.

**Why does this matter for QA workflows?**

- The Xray test case in Jira has the manual test steps written by QA.
- When the automated test runs, Xray receives the pass/fail via the Allure-Jira bridge.
- Traceability is complete: a failing test in the suite links directly back to the Xray case, which links to the product requirement.

> **In practice:** every new spec starts with `$TmsLink("B2CQA-NNNN")`, and every Xray case has a matching automation. If you add a test without a TMS link, Xray has no idea the automation exists — the case stays "manual" forever.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://wix.github.io/Detox/docs/config/overview">Detox: Configuration Overview</a> — every field in <code>detox.config.js</code></li>
<li><a href="https://wix.github.io/Detox/docs/guide/running-tests">Detox: Running Tests</a> — the <code>detox test</code> CLI</li>
<li><a href="https://wix.github.io/Detox/docs/articles/how-detox-works">Detox: How Detox Works</a> — idle detection deep dive</li>
<li><a href="https://wix.github.io/Detox/docs/troubleshooting/synchronization">Detox: Troubleshooting Synchronization</a> — <code>disableSynchronization</code> and the URL blacklist</li>
<li><a href="https://jestjs.io/docs/configuration">Jest: Configuration</a> — every Jest config option</li>
<li><a href="https://jestjs.io/docs/configuration#testenvironment-string">Jest: testEnvironment</a> — how to write your own</li>
<li><a href="https://github.com/wix/Detox/blob/master/docs/APIRef.Configuration.md#behavior-configuration">Detox: Behavior Configuration</a> — <code>reinstallApp</code>, <code>cleanup</code>, <code>launchApp</code></li>
<li><a href="https://www.typescriptlang.org/docs/handbook/decorators.html">TypeScript: Decorators</a> — legacy vs stage-3 decorator syntax</li>
<li><a href="https://github.com/wix/Detox/issues/3918">wix/Detox#3918</a> — the Android scroll long-press gotcha</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Detox's configuration lives in three files working together — <code>detox.config.js</code> defines builds and devices, <code>jest.config.js</code> drives the runner, and <code>jest.environment.ts</code> customizes per-worker lifecycle. Detox synchronizes automatically with the JS bridge, dispatch queues, timers, and network — with the URL blacklist as your escape hatch and <code>disableSynchronization()</code> as the nuclear option. On top of Detox, Ledger Live runs a WebSocket bridge that injects Redux state, feature flags, and mock device events into the running app, with every message waiting for an ACK to guarantee state has applied before the test proceeds. <code>reverseTcpPort</code>, the iOS/Android IP split, <code>@Step</code>, and <code>$TmsLink</code> are the seams where cross-platform and reporting concerns live — master them and you can read any spec in the suite.
</div>

### 5.6.10 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Detox Advanced</h3>
<p class="quiz-subtitle">7 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> In <code>detox.config.js</code>, what is the purpose of the <code>configurations</code> section?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It defines build commands for iOS and Android</button>
<button class="quiz-choice" data-value="B">B) It lists the test files Detox should run</button>
<button class="quiz-choice" data-value="C">C) It pairs an <code>app</code> (a build) with a <code>device</code> (a simulator/emulator) into a named target that you pass on the CLI with <code>-c</code></button>
<button class="quiz-choice" data-value="D">D) It stores environment variables for the app</button>
</div>
<p class="quiz-explanation"><code>apps</code> defines what to install, <code>devices</code> defines what to run on, and <code>configurations</code> pairs them. You run <code>detox test -c ios.sim.debug</code>, which resolves to the <code>simulator</code> device plus the <code>ios.debug</code> app.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q2.</strong> Why does Ledger Live's <code>jest.config.js</code> default <code>maxWorkers</code> to <code>1</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Each worker needs its own simulator/emulator, and the config only guarantees one device per platform by default; parallel workers are opt-in via <code>JEST_MAX_WORKERS</code> on CI where additional devices are pre-booted</button>
<button class="quiz-choice" data-value="B">B) Jest does not support parallel execution</button>
<button class="quiz-choice" data-value="C">C) Detox is single-threaded by nature</button>
<button class="quiz-choice" data-value="D">D) Parallel E2E tests are forbidden by the test policy</button>
</div>
<p class="quiz-explanation">Mobile E2E parallelism requires one device per worker. The config exposes <code>simulator</code>, <code>simulator2</code>, <code>simulator3</code> and matching emulators; the custom <code>jest.environment.ts</code> swaps the device based on <code>JEST_WORKER_ID</code>. But booting three emulators locally is expensive, so the default is 1.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> What does Detox wait on before performing each action (idle detection)?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Only network requests</button>
<button class="quiz-choice" data-value="B">B) Only the React Native JS thread</button>
<button class="quiz-choice" data-value="C">C) Only UI animations</button>
<button class="quiz-choice" data-value="D">D) All of: JS thread, native dispatch queues, animation frame pump, active timers, in-flight non-blacklisted network requests, and the RN bridge</button>
</div>
<p class="quiz-explanation">This is why Detox is called grey-box. It synchronizes against the app's whole runtime — not just the DOM-equivalent. When any of these queues is busy, the action waits.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q4.</strong> What is the <code>detoxURLBlacklistRegex</code> and why does Ledger Live set it?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A list of URLs the app is forbidden from calling</button>
<button class="quiz-choice" data-value="B">B) A list of URL regexes that Detox will NOT treat as in-flight network requests for synchronization — used to exclude long-polling or always-on endpoints (analytics, Google APIs) that would otherwise keep Detox waiting forever</button>
<button class="quiz-choice" data-value="C">C) A list of URLs that Jest will intercept and mock</button>
<button class="quiz-choice" data-value="D">D) A CSP (Content Security Policy) for the RN WebView</button>
</div>
<p class="quiz-explanation">Detox's network synchronization waits for all in-flight requests to complete before acting. Analytics beacons and Google connectivity checks never complete in the test's lifetime, so they must be excluded or every test would hang.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> In the WebSocket bridge, why does every server-to-client message wait for an ACK from the client?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) WebSockets don't guarantee delivery without ACKs</button>
<button class="quiz-choice" data-value="B">B) To encrypt the message in transit</button>
<button class="quiz-choice" data-value="C">C) Without an ACK, <code>await setFeatureFlag(...)</code> would resolve as soon as the message was sent — before the client actually ran the Redux dispatch. The ACK is sent only after the client handler completes, so the next test line only runs once the state change is live</button>
<button class="quiz-choice" data-value="D">D) To measure network latency</button>
</div>
<p class="quiz-explanation">The ACK is a handler-completion signal. Without it, tests would race against the async state updates they just requested. See <code>bridge/client.ts</code>: every case in the switch statement ends with <code>postMessage({ type: "ACK", id: msg.id })</code>.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q6.</strong> Why does <code>bridge/client.ts</code> use <code>localhost</code> on iOS but <code>10.0.2.2</code> on Android?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) iOS simulator shares the host macOS network stack so <code>localhost</code> resolves to the host; Android emulator has an isolated guest network and <code>10.0.2.2</code> is the QEMU-defined alias for the host's loopback</button>
<button class="quiz-choice" data-value="B">B) <code>10.0.2.2</code> is Android's public DNS server</button>
<button class="quiz-choice" data-value="C">C) iOS blocks <code>10.0.2.2</code> at the firewall</button>
<button class="quiz-choice" data-value="D">D) Historical reasons; both should resolve identically today</button>
</div>
<p class="quiz-explanation"><code>10.0.2.2</code> is a well-known Android emulator alias documented by Google. The bridge must use the correct IP for each platform; <code>reverseTcpPort</code> additionally enables <code>localhost</code> on Android, but the client is coded defensively to use the canonical alias.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q7.</strong> What does the <code>@Step("Enter amount $0")</code> decorator do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Pauses the test at that line, like a debugger breakpoint</button>
<button class="quiz-choice" data-value="B">B) Wraps the decorated method in an <code>allure.step(name, fn)</code> call so the method appears as a named step in the Allure report, with <code>$0</code>/<code>$1</code> substituted by the method's arguments. Requires <code>experimentalDecorators: true</code> in tsconfig and <code>decorators: true</code> in SWC</button>
<button class="quiz-choice" data-value="C">C) Marks the step as a blocking checkpoint in CI</button>
<button class="quiz-choice" data-value="D">D) Registers the step with Jira/Xray as a new test case</button>
</div>
<p class="quiz-explanation"><code>@Step</code> is a reporting decorator from <code>jest-allure2-reporter/api</code>. It's a zero-runtime-overhead way to make page-object methods show up as hierarchical steps in Allure. The positional placeholders interpolate the method's arguments into the step name.</p>
</div>

<div class="quiz-score"></div>
</div>

## Mobile Codebase Deep Dive: Every File Explained

<div class="chapter-intro">
This is your reference chapter for the mobile side. It catalogues every file in the canonical E2E workspace at <code>e2e/mobile/</code>, shows the important ones verbatim, and explains how each piece connects to the rest of the suite. Keep this chapter open when you land in an unfamiliar mobile test file — it will tell you what the file does, why it exists, and where to look next. It is the mobile counterpart of Part 4 Chapter 4.6.
</div>

### 5.7.1 The Canonical Workspace — `e2e/mobile/`

Until early 2026 the mobile E2E suite lived under `apps/ledger-live-mobile/e2e/`, nested inside the React Native app's own directory. Since then the suite has been promoted to a **peer workspace** at the monorepo root:

```
<repo-root>/
├── apps/
│   └── ledger-live-mobile/          # the RN app itself
│       ├── src/
│       ├── ios/
│       ├── android/
│       ├── .env.mock                # ENVFILE consumed by Detox builds
│       └── .env.mock.prerelease
├── e2e/
│   ├── desktop/                     # peer Playwright workspace
│   └── mobile/                      # THIS chapter's subject — Detox workspace
└── libs/
    └── ledger-live-common/          # shared E2E enums/models live here
        └── src/e2e/
```

`e2e/mobile/` is its own pnpm / Nx package: it has its own `package.json` (name `ledger-live-mobile-e2e-tests`), its own `project.json` (Nx targets), its own `detox.config.js`, its own Jest stack, and its own `node_modules`. That peer-workspace decision mirrors what already happened on desktop (`e2e/desktop/` separated from `apps/ledger-live-desktop/`). The win is the same in both cases: a clean dependency boundary, faster type-checking, and an Nx graph where the test workspace is a first-class participant.

Two consequences you will feel immediately:

1. **Every path in this chapter is relative to `e2e/mobile/`**, not to the app. Old paths like `apps/ledger-live-mobile/e2e/page/index.ts` are stale; the file now lives at `e2e/mobile/page/index.ts`.
2. **The build artifacts still live with the app.** The iOS `.app` bundle is produced under `apps/ledger-live-mobile/ios/build/...` and the Android APKs under `apps/ledger-live-mobile/android/app/build/...`. `detox.config.js` crosses the workspace boundary with `path.resolve(__dirname, "../..")` to reach them. The ENVFILE names — `apps/ledger-live-mobile/.env.mock` and `apps/ledger-live-mobile/.env.mock.prerelease` — are also unchanged, because Detox hands those paths straight to `xcodebuild` and `gradlew`, which run in the app directory.

A small caveat: the legacy `apps/ledger-live-mobile/e2e/` directory still exists and still contains 17 specs that have not yet been ported. You may see both trees when you grep the repo. For everything covered in this guide — and for every new spec — the canonical location is `e2e/mobile/`. Treat the old tree as a migration artefact; do not write new tests there.

### 5.7.2 Directory Tree

Here is the full layout of `e2e/mobile/`:

```
e2e/mobile/
├── babel.config.js                # Babel plugins (decorators + namespace exports)
├── detox.config.js                # Detox: test runner, apps, devices, configurations
├── detox.config.d.ts              # Ambient type for the JS config
├── jest.config.js                 # Jest: transform, path-alias map, reporters, env
├── jest.environment.ts            # Custom Jest env (extends DetoxEnvironment)
├── jest.globalSetup.ts            # One-shot setup (Detox bootstrap + Speculos cleanup)
├── jest.globalTeardown.ts         # One-shot teardown (env.properties + userdata cleanup)
├── tsconfig.json                  # TS config for the workspace (decorators + path aliases)
├── setup.ts                       # Per-file setup wired via setupFilesAfterEnv
├── package.json                   # name: ledger-live-mobile-e2e-tests
├── project.json                   # Nx targets (e2e:ci, e2e:ci:with-cli)
├── xray.formater.sh               # Post-processes Allure results for Xray export
├── README.md
├── CHANGELOG.md
│
├── bridge/                        # WebSocket bridge (test-process side)
│   └── server.ts                  # 307 lines — open/close, loadConfig, feature flags, ACKs
│
├── helpers/                       # Low-level runtime helpers
│   ├── commonHelpers.ts           # launchApp, openDeeplink, setupEnvironment, normalizeText, ...
│   ├── elementHelpers.ts          # NativeElementHelpers + WebElementHelpers (Detox primitives)
│   ├── errorHelpers.ts            # detectErrorModal, checkForErrorElement
│   ├── pageScroller.ts            # PageScroller class (used by NativeElementHelpers.scrollToId)
│   └── allure/
│       └── allure-helper.ts       # setAllureDescription (wallet40 / shard badging)
│
├── models/                        # Test-side domain helpers
│   ├── currencies.ts              # getCurrencyManagerApp
│   ├── send.ts                    # verifyAppValidationSendInfo
│   └── stake.ts                   # verifyAppValidationStakeInfo, verifyStakeOperationDetailsInfo
│
├── page/                          # Page Object Model layer (32 POMs)
│   ├── index.ts                   # Application class — lazy-init hub
│   ├── common.page.ts             # CommonPage (base class for many POMs)
│   ├── error.page.ts              # ErrorPage (generic-error-modal testID holder)
│   ├── passwordEntry.page.ts      # Simplest POM — password lock screen
│   ├── speculos.page.ts           # Speculos device interaction wrapper
│   ├── accounts/                  # account.page, accounts.page, addAccount.drawer, assetAccounts.page
│   ├── discover/                  # discover.page
│   ├── drawer/                    # modular.drawer (asset/network/account picker)
│   ├── liveApps/                  # swapLiveApp.ts (WebView-driven swap UI)
│   ├── manager/                   # manager.page (My Ledger screen)
│   ├── market/                    # market.page
│   ├── onboarding/                # onboardingSteps.page
│   ├── settings/                  # settings, settingsGeneral, settingsHelp, ledgerSync
│   ├── stax/                      # customLockscreen.page
│   ├── trade/                     # swap, send, receive, stake, buySell, earn*, operationDetails, deviceValidation, celoManageAssets
│   └── wallet/                    # portfolio, portfolioEmptyState, walletTabNavigator, mainNavigation, transferMenu.drawer
│
├── specs/                         # 213 .spec.ts files — organised by feature bucket
│   ├── deeplinks.spec.ts
│   ├── languageChange.spec.ts
│   ├── market.spec.ts
│   ├── onboardingReadOnly.spec.ts
│   ├── userOpensApplication.spec.ts
│   ├── account/        addAccount/        buySell/
│   ├── delegate/       deleteAccount/     deposit/
│   ├── earn/           ledgerSync/        portfolio/
│   ├── send/           settings/          subAccount/
│   ├── swap/           verifyAddress/     wallet40/
│   └── swap/otherTestCases/  (and swap/otherTestCases/tooLowAmountForQuoteSwaps/)
│
├── userdata/                      # Redux state snapshots loaded via the bridge
│   ├── skip-onboarding.json                       # Onboarding complete, no accounts (~1 KB)
│   ├── 1AccountBTC1AccountETHReadOnlyFalse.json   # 1 BTC + 1 ETH account (~1 MB)
│   ├── EthAccountXrpAccountReadOnlyFalse.json     # ETH + XRP accounts (~160 KB)
│   ├── speculos-tests-app.json                    # Pre-wired Speculos accounts (~200 KB)
│   ├── speculos-x-other-account.json              # Variant with a second account (~100 KB)
│   └── swap-history.json                          # Rich swap history, used by QAA-604 (~2.3 MB)
│
├── scripts/
│   ├── typecheck.js               # node scripts/typecheck.js — filtered tsc run
│   └── build-ai-artifact.sh       # Post-run bundler for AI-assisted analysis
│
├── types/
│   ├── global.d.ts                # Ambient globals (app, tapById, Currency, ...)
│   └── jest-allure2-reporter.d.ts # Module augmentation for $TmsLink / $Tag
│
├── utils/
│   ├── cliUtils.ts                # CLI class: wraps live-common CLI commands
│   ├── constants.ts               # NANO_APP_CATALOG_PATH, WALLET_40_FEATURE_FLAGS
│   ├── fileUtils.ts               # FileUtils.waitForFileToExist
│   ├── initUtil.ts                # InitializationManager (called by Application.init)
│   ├── loggingUtils.ts            # Console capture + Allure attachment helpers
│   ├── retry.ts                   # retryUntilTimeout
│   ├── speculosUtils.ts           # launchSpeculos, deleteSpeculos, registerSpeculos, ...
│   └── swapUtils.ts               # performSwapUntilQuoteSelectionStep
│
├── artifacts/                     # Test run output (created at runtime, gitignored)
│   ├── allure-report/             # Generated HTML Allure report
│   ├── environment.properties     # Feature-flag + env dump from globalTeardown
│   ├── screenshots/               # PNGs attached to Allure on failure
│   └── nano-app-catalog.json      # Pinned Nano app versions
│
└── node_modules/                  # The workspace's own installed deps
```

The five load-bearing folders are `bridge/`, `helpers/`, `page/`, `specs/`, and `utils/`. Everything else (models, userdata, types, scripts) either feeds them or supports tooling.

### 5.7.3 `package.json` and `project.json` — the Workspace Identity

`package.json` names the workspace and defines the commands that humans and CI invoke. These are the scripts verbatim:

```json
{
  "name": "ledger-live-mobile-e2e-tests",
  "version": "0.20.0",
  "private": true,
  "scripts": {
    "lint": "oxlint .",
    "lint:fix": "oxfmt . && oxlint . --fix",
    "typecheck": "node ./scripts/typecheck.js",
    "test:detox": "pnpm detox test",
    "test:ios": "pnpm test:detox --configuration ios.sim.release",
    "test:android": "pnpm test:detox --configuration android.emu.release",
    "build:ios": "nx run live-mobile:e2e:build -- --configuration ios.sim.release",
    "build:android": "nx run live-mobile:e2e:build -- --configuration android.emu.release",
    "test:ios:debug": "pnpm test:detox --configuration ios.sim.debug",
    "test:android:debug": "pnpm test:detox --configuration android.emu.debug",
    "build:ios:debug": "pnpm --filter live-mobile run e2e:build --configuration ios.sim.debug",
    "build:android:debug": "pnpm --filter live-mobile run e2e:build --configuration android.emu.debug",
    "allure:generate": "allure generate ./artifacts --clean -o ./artifacts/allure-report",
    "allure:open": "allure open ./artifacts/allure-report",
    "allure": "pnpm run allure:generate && pnpm run allure:open"
  }
}
```

A few things worth noting:

- **Build vs. test.** Build scripts (`build:ios`, `build:android`, `build:ios:debug`, `build:android:debug`) always delegate to the **app** project (`live-mobile`) via Nx or pnpm filter, because the native bundle has to be produced where the native project lives. Test scripts run Detox from this workspace because Detox's config file is here.
- **Nx wrapping.** `build:ios` uses `nx run live-mobile:e2e:build -- --configuration ios.sim.release`; the double-dash passes the rest straight to the Detox build command. Nx adds caching, dependency graph awareness, and affected-detection; it does not replace Detox.
- **`test:ios` defaults to release.** The default test target uses the `ios.sim.release` Detox configuration. That is important: release builds are closer to what ships, and Detox explicitly disables dev-menu shortcuts in them. Use `test:ios:debug` when you need Metro hot-reload.
- **Allure is workspace-local.** `pnpm allure` (from within `e2e/mobile/`) expects `./artifacts` to contain the raw `allure-results` output written by `jest-allure2-reporter`. Do not run it from the monorepo root.

On the Nx side, `project.json` is short but carries the CI-relevant targets:

```json
{
  "name": "ledger-live-mobile-e2e-tests",
  "targets": {
    "e2e:ci": {
      "executor": "nx:run-script",
      "dependsOn": ["ledger-live-mobile:prepare-e2e-deps"],
      "options": { "script": "e2e:ci" },
      "outputs": [
        "{workspaceRoot}/apps/ledger-live-mobile/artifacts/**",
        "{projectRoot}/artifacts/**"
      ],
      "inputs": ["default", "mobileEnv", "sharedEnv"],
      "cache": true,
      "parallelism": true
    },
    "e2e:ci:with-cli": {
      "executor": "nx:run-script",
      "dependsOn": ["ledger-live-mobile:prepare-e2e-deps", "@ledgerhq/live-cli:build"],
      "options": { "script": "e2e:ci" },
      "outputs": [
        "{workspaceRoot}/apps/ledger-live-mobile/artifacts/**",
        "{projectRoot}/artifacts/**"
      ],
      "inputs": ["default", "mobileEnv", "sharedEnv"],
      "cache": true,
      "parallelism": true
    }
  }
}
```

The two CI targets differ by one dependency: `e2e:ci:with-cli` also builds `@ledgerhq/live-cli`, which is required when `cliCommands` or `cliCommandsOnApp` are used to pre-seed Speculos accounts. Both targets:

- Depend on `ledger-live-mobile:prepare-e2e-deps`, which makes sure the app is built, the ENV files are present, and Speculos prerequisites are in place before Detox starts.
- Write outputs to both the **app** artifacts directory (where `xcodebuild`/`gradlew` drop the binaries and where Detox writes device-side video/screenshots) and the **workspace** artifacts directory (where Jest/Allure write results).
- Are **cacheable** and **parallelism-enabled**, which lets CI shard across multiple agents.

Key dev-dependencies worth knowing about (trimmed list from `package.json`):

- `detox` 20.48.0 — the runner (see 4.7.4).
- `detox-allure2-adapter` 1.0.0-alpha.42 — hooks Detox events into Allure.
- `jest-allure2-reporter` 2.2.8 — provides the `@Step` decorator, `$TmsLink`, `$Tag`, and the Allure writer.
- `@swc/jest` — the TypeScript transformer used by Jest. Much faster than `ts-jest`.
- `@jest/reporters`, `jest-environment-node`, `jest-metadata`, `jest-docblock` — the Jest ecosystem glue.
- `@ledgerhq/live-common`, `@ledgerhq/live-env`, `@ledgerhq/live-dmk-speculos` — the monorepo libraries the tests consume via `workspace:*` protocol.
- `oxlint` / `oxfmt` — Rust-based lint/format toolchain (much faster than ESLint).
- `ts-node`, `tsconfig-paths` — used only by `jest.globalSetup.ts` and `jest.globalTeardown.ts` to register path aliases before Jest starts.

### 5.7.4 `detox.config.js` — The Runner Configuration

`detox.config.js` is 167 lines and it is the single source of truth for how Detox finds the app binary, which simulator/emulator to spawn, which Jest config to hand the test runner, and how artifacts flow to Allure. Here is the full file:

```javascript
const path = require("path");
const iosArch = "arm64";
// NOTE: Pass CI=1 if you want to build locally when you don't have a mac M1. This works better if you do export CI=1 for the whole session.
const androidArch = process.env.CI ? "x86_64" : "arm64-v8a";
const SCHEME = "ledgerlivemobile";

const rootDir = path.resolve(__dirname, "../..");
const iosDir = path.join(rootDir, "apps/ledger-live-mobile/ios");
const iosBuildDir = path.join(iosDir, "build");
const androidDir = path.join(rootDir, "apps/ledger-live-mobile/android");
const ENV_FILE_MOCK = path.join("apps", "ledger-live-mobile", ".env.mock");
const ENV_FILE_MOCK_PRERELEASE = path.join("apps", "ledger-live-mobile", ".env.mock.prerelease");

const getIosBinary = config =>
  path.join(iosBuildDir, `Build/Products/${config}-iphonesimulator/${SCHEME}.app`);
const getAndroidBinary = type =>
  path.join(androidDir, `app/build/outputs/apk/${type}/app-${androidArch}-${type}.apk`);
const getAndroidTestBinary = type =>
  path.join(androidDir, `app/build/outputs/apk/androidTest/${type}/app-${type}-androidTest.apk`);

/** @type {Detox.DetoxConfig} */
module.exports = {
  testRunner: {
    $0: "jest",
    args: {
      config: "jest.config.js",
    },
    jest: {
      setupTimeout: 500000,
      teardownTimeout: 120000,
    },
    noRetryArgs: ["json", "outputFile"],
    retries: 0,
    forwardEnv: true,
  },
  logger: {
    level: process.env.DEBUG_DETOX ? "trace" : "info",
  },
  behavior: {
    init: {
      reinstallApp: true,
      exposeGlobals: false,
    },
    launchApp: "auto",
    cleanup: {
      shutdownDevice: false,
    },
    extends: "detox-allure2-adapter/preset-detox",
  },
  apps: {
    "ios.debug":  { /* Debug build */ },
    "ios.staging":   { /* Staging build */ },
    "ios.release":   { /* Release build (CI default) */ },
    "ios.prerelease":{ /* Release with .env.mock.prerelease */ },
    "android.debug": { /* assembleDebug */ },
    "android.release": { /* assembleDetox (special detox variant) */ },
    "android.prerelease": { /* assembleDetoxPreRelease */ },
  },
  devices: {
    simulator:  { type: "ios.simulator", device: { name: "iOS Simulator" } },
    simulator2: { type: "ios.simulator", device: { name: "iOS Simulator 2" } },
    simulator3: { type: "ios.simulator", device: { name: "iOS Simulator 3" } },
    emulator:   { type: "android.emulator", device: { avdName: "Android_Emulator" },
                  gpuMode: "swiftshader_indirect", headless: !!process.env.CI },
    emulator2:  { type: "android.emulator", device: { avdName: "Android_Emulator_2" },
                  gpuMode: "swiftshader_indirect", headless: !!process.env.CI },
    emulator3:  { type: "android.emulator", device: { avdName: "Android_Emulator_3" },
                  gpuMode: "swiftshader_indirect", headless: !!process.env.CI },
  },
  configurations: {
    "ios.sim.debug":     { device: "simulator", app: "ios.debug" },
    "ios.sim.staging":   { device: "simulator", app: "ios.staging" },
    "ios.sim.release":   { device: "simulator", app: "ios.release" },
    "ios.sim.prerelease":{ device: "simulator", app: "ios.prerelease" },
    "android.emu.debug":     { device: "emulator", app: "android.debug" },
    "android.emu.release":   { device: "emulator", app: "android.release" },
    "android.emu.prerelease":{ device: "emulator", app: "android.prerelease" },
  },
};
```

Reading it from top to bottom:

- **Cross-workspace paths.** `rootDir = path.resolve(__dirname, "../..")` jumps two levels up — from `e2e/mobile/` to the monorepo root — so every `iosDir` / `androidDir` / `ENV_FILE_*` path resolves to files under `apps/ledger-live-mobile/`. Detox does not care that the config lives in a different workspace from the native project; it only cares that the final binary paths exist at the time the test starts.
- **`iosArch` and `androidArch`.** iOS is pinned to `arm64` (Apple-silicon simulators). Android defaults to `arm64-v8a` locally (faster on M1+ Macs) but flips to `x86_64` in CI, where the emulators run on x86_64 Linux hosts.
- **`SCHEME`.** The Xcode scheme name (`ledgerlivemobile`) is what `xcodebuild -scheme` receives. It is also the name of the `.app` bundle produced in `iosBuildDir`.
- **`testRunner.$0: "jest"`.** Detox forwards to Jest. Every argument under `args` becomes a CLI flag (`--config jest.config.js`). `setupTimeout: 500000` (8m20s) and `teardownTimeout: 120000` (2m) are generous because Speculos launches and Detox cleanups can each burn a minute in CI. `retries: 0` — we do not retry at the Detox layer; Jest handles retries if configured. `forwardEnv: true` ensures `DETOX_CONFIGURATION` is passed to Jest workers, which `jest.environment.ts` reads to pick the right device for a secondary worker.
- **`behavior.init.reinstallApp: true`.** Every session reinstalls the app — guaranteeing a clean slate. `exposeGlobals: false` means Detox does **not** install `device`, `element`, `by`, `waitFor`, etc. on `globalThis`; the test code imports them from `detox`. This is important: the globals you see in specs (`tapById`, `waitForElementById`, `app`) come from our own `setup.ts` / `jest.environment.ts`, not from Detox.
- **`behavior.extends: "detox-allure2-adapter/preset-detox"`.** The adapter preset hooks Detox's artifact pipeline (video, screenshots, device logs, view hierarchy) into Allure's attachment API.
- **`apps`.** Seven app configurations. Each carries a `build` command (the literal shell command Detox runs on `detox build`) and a `binaryPath`. Notable details: iOS `Debug`/`Staging`/`Release` all use `.env.mock`; `prerelease` uses `.env.mock.prerelease`. Android has three separate Gradle variants — `assembleDebug`, `assembleDetox`, `assembleDetoxPreRelease` — because the Android build pipeline customises bundling (Hermes vs JSC, ProGuard, signing).
- **`devices`.** Three iOS simulators (`simulator`, `simulator2`, `simulator3`) and three Android emulators. The numbered suffixes exist so parallel Jest workers (`maxWorkers: 3` in `jest.config.js`) can each own their own simulator/emulator. `jest.environment.ts` reads `JEST_WORKER_ID` and rewrites `detoxConfig.device` to the right entry for workers 2 and 3 (see 4.7.5).
- **`configurations`.** The cartesian product of devices × apps — what the CLI receives as `--configuration <name>`. The ones you will see most often are `ios.sim.debug`, `ios.sim.release`, `android.emu.debug`, `android.emu.release`.

### 5.7.5 The Jest Stack

Detox delegates to Jest. The Jest stack in this workspace is four files: `jest.config.js`, `jest.environment.ts`, `jest.globalSetup.ts`, and `jest.globalTeardown.ts` — plus one Babel file and one setup file.

#### `jest.config.js`

```javascript
const { compilerOptions } = require("./tsconfig.json");
const {
  getDeviceFirmwareVersion,
  getSpeculosModel,
} = require("@ledgerhq/live-common/e2e/speculosAppVersion");

function pathsToModuleNameMapper(paths, { prefix = "<rootDir>/" } = {}) {
  const jestPaths = {};
  if (!paths) return jestPaths;

  Object.keys(paths).forEach(pathKey => {
    if (pathKey === "*") return;
    const pathEntry = paths[pathKey];
    const pathValues = Array.isArray(pathEntry) ? pathEntry : [pathEntry];
    pathValues.forEach(pathValue => {
      const jestKey = pathKey.replace(/\*$/, "(.*)");
      const jestValue = pathValue.replace(/\*/g, "$1");
      jestPaths[jestKey] = `${prefix}${jestValue}`;
    });
  });

  return jestPaths;
}

const jestAllure2ReporterOptions = {
  extends: "detox-allure2-adapter/preset-detox",
  resultsDir: "artifacts",
  testCase: {
    links: {
      issue: "https://ledgerhq.atlassian.net/browse/{{name}}",
      tms: "https://ledgerhq.atlassian.net/browse/{{name}}",
    },
    labels: { host: process.env.RUNNER_NAME },
    status: ({ value }) => (value === "broken" ? "failed" : value),
  },
  overwrite: false,
  environment: async ({ $ }) => ({
    SPECULOS_DEVICE: process.env.SPECULOS_DEVICE,
    SPECULOS_FIRMWARE_VERSION: await getDeviceFirmwareVersion(getSpeculosModel()),
    MOBILE_DEVICE: process.env.DEVICE_INFO || "Unknown device",
    path: process.cwd(),
    "version.node": process.version,
    "version.jest": await $.manifest("jest", ["version"]),
    "package.name": await $.manifest(m => m.name),
    "package.version": await $.manifest(["version"]),
  }),
};

const detoxAllure2AdapterOptions = {
  deviceLogs: true,
  deviceScreenshots: false,
  deviceVideos: false,
  deviceViewHierarchy: false,
  onError: "warn",
};

const ESM_PACKAGES = ["ky", "@polkadot"].join("|");

const config = {
  rootDir: ".",
  modulePaths: [compilerOptions.baseUrl ?? "."],
  maxWorkers: process.env.CI ? 3 : 1,
  transform: {
    "^.+\\.(js|jsx)?$": "babel-jest",
    "^.+\\.(ts|tsx)?$": [
      "@swc/jest",
      {
        jsc: {
          target: "es2022",
          parser: { syntax: "typescript", tsx: true, decorators: true, dynamicImport: true },
          transform: { react: { runtime: "automatic" } },
        },
        sourceMaps: "inline",
        module: { type: "commonjs" },
      },
    ],
  },
  moduleNameMapper: {
    "^@ledgerhq/live-common/e2e/(.*)$": "<rootDir>/../../libs/ledger-live-common/src/e2e/$1",
    ...pathsToModuleNameMapper(compilerOptions.paths, { prefix: "<rootDir>/" }),
  },
  transformIgnorePatterns: [`/node_modules/(?!(${ESM_PACKAGES})/)`],
  setupFilesAfterEach: undefined,
  setupFilesAfterEach: undefined,
  setupFilesAfterEnv: ["<rootDir>/setup.ts"],
  testMatch: ["<rootDir>/specs/**/*.spec.ts"],
  testTimeout: 300_000,
  reporters: [
    "detox/runners/jest/reporter",
    ["jest-allure2-reporter", jestAllure2ReporterOptions],
  ],
  globalSetup: "<rootDir>/jest.globalSetup.ts",
  globalTeardown: "<rootDir>/jest.globalTeardown.ts",
  testEnvironment: "<rootDir>/jest.environment.ts",
  testEnvironmentOptions: {
    customConditions: ["node"],
    eventListeners: [
      "jest-metadata/environment-listener",
      "jest-allure2-reporter/environment-listener",
      ["detox-allure2-adapter", detoxAllure2AdapterOptions],
    ],
  },
  verbose: true,
  resetModules: true,
};

module.exports = config;
```

The key decisions encoded here:

- **`maxWorkers: 3` in CI, `1` locally.** CI can afford three simulators/emulators; your laptop usually cannot. When workers > 1, the three device entries (`simulator`/`simulator2`/`simulator3`) matter.
- **SWC as the TS transformer.** `@swc/jest` is wired with `decorators: true` — non-negotiable, because every POM method uses `@Step(...)`.
- **Path aliases.** The helper `pathsToModuleNameMapper` translates `tsconfig.json`'s `paths` entries into Jest's `moduleNameMapper`. One extra rule is added by hand: `@ledgerhq/live-common/e2e/*` is remapped to `../../libs/ledger-live-common/src/e2e/*` so that enums and models used across desktop and mobile are loaded from the shared source (not the compiled `lib/`).
- **`testMatch: ["<rootDir>/specs/**/*.spec.ts"]`.** Only files ending in `.spec.ts` under `specs/` run. Driver modules like `swap.ts`, `swap.other.ts`, `send.ts` do not match this glob — they are imported by specs but never executed directly.
- **`testTimeout: 300_000`.** 5 minutes per test. Swap flows that require Speculos signing and confirmation can take that long in CI.
- **`reporters`.** Two: Detox's own Jest reporter (required for Detox/Jest integration) and `jest-allure2-reporter` with options tuned for Ledger's Jira instance — links to `ledgerhq.atlassian.net` follow the `$TmsLink("B2CQA-xxx")` decorator in specs.
- **`testEnvironmentOptions.eventListeners`.** Three listeners chain into the env lifecycle: `jest-metadata` (enables metadata APIs), `jest-allure2-reporter` (the writer side), and `detox-allure2-adapter` (device log/video/screenshot/view-hierarchy capture).
- **`resetModules: true`.** Between specs, Jest's module registry is wiped. This makes sure spec-level `setEnv(...)` calls do not leak into the next spec.

#### `jest.environment.ts`

This is the custom Jest environment — the per-worker globals factory. Some key excerpts (the file is 223 lines):

```typescript
import DetoxEnvironment from "detox/runners/jest/testEnvironment";

export default class TestEnvironment extends DetoxEnvironment {
  declare global: typeof globalThis;

  async setup() {
    const workerId = Number(process.env.JEST_WORKER_ID ?? "1");
    if (workerId > 1) this.setupDeviceForSecondaryWorker(workerId);
    await super.setup();

    setupEnvironment();

    const speculosDevicesMap = new Map<string, number>();
    const webSocketObj = {
      wss: undefined,
      ws: undefined,
      messages: {},
      e2eBridgeServer: new Subject<ServerData>(),
    };
    const pendingCallbacksMap = new Map<string, { callback: (data: string) => void }>();
    const appInstance = new Application();

    this.global.app = appInstance;
    this.global.IS_FAILED = false;
    this.global.speculosDevices = speculosDevicesMap;
    this.global.webSocket = webSocketObj;
    this.global.pendingCallbacks = pendingCallbacksMap;

    // identical assignments to globalThis follow, plus:
    Object.assign(this.global, enums);
    Object.assign(this.global, nativeHelpers);
    Object.assign(this.global, webHelpers);
    Object.assign(this.global, cliCommandsUtils);
    // ... and to globalThis as well
  }
  // ...
}
```

What it does:

1. **Selects the right device for the worker.** Worker 1 uses the default device in `detox.config.js`. Workers 2 and 3 call `setupDeviceForSecondaryWorker(workerId)`, which rewrites `detoxConfig.device` to `simulator2`/`emulator2` or `simulator3`/`emulator3` before `super.setup()` boots Detox. This is how parallel Jest workers avoid fighting over the same simulator UDID.
2. **Constructs `app = new Application()` once per worker.** The `Application` class (see 4.7.10) lazy-initialises its 32 POMs; constructing it is cheap.
3. **Installs every global the test suite expects.** Enums (`Account`, `Currency`, `Delegate`, `Transaction`, `Fee`, `AppInfos`, `Swap`, `TokenAccount`, `CLI`), every `NativeElementHelpers.*` method renamed to its top-level alias (`tapById`, `waitForElementById`, etc.), every `WebElementHelpers.*` method, and the `cliCommandsUtils` from live-common. The `types/global.d.ts` file declares the matching ambient types so TypeScript knows about them.
4. **Installs failure hooks.** `handleTestEvent` (not shown above) reacts to Jest's `hook_failure` and `test_fn_failure` events: it flips `IS_FAILED`, takes a Speculos screenshot, an app screenshot, captures the native view hierarchy XML, pulls the bridge log, and attaches everything to Allure. This is why failing tests have rich Allure reports even though no test code mentions Allure.
5. **Tears down carefully.** On teardown, it closes the bridge server/client, disconnects DMK Speculos transports, and triggers GC.

<div class="resource-box" style="border-left: 6px solid #dc2626; background: #fef2f2;">
<h4 style="color: #991b1b; margin-top: 0;">Mobile dual-package hazard — never <code>import</code> what is already a global</h4>

<p>This is the single most expensive mistake a new mobile QA can make. Read this section before you ever write <code>import { liveDataWithAddressCommand } from "@ledgerhq/live-common/...";</code> in a mobile spec.</p>

<p><code>@ledgerhq/live-common</code> ships <strong>two builds</strong> of every module:</p>

<ul>
<li><code>libs/ledger-live-common/lib/e2e/cliCommandsUtils.js</code> — CommonJS</li>
<li><code>libs/ledger-live-common/lib-es/e2e/cliCommandsUtils.js</code> — ES modules</li>
</ul>

<p>The harness (<code>jest.environment.ts</code>) boots, imports <code>cliCommandsUtils</code> through <strong>one</strong> of those paths, and assigns the result to globals (<code>globalThis.liveDataWithAddressCommand</code>, <code>globalThis.CLI</code>, etc.). At the same time, the harness calls <code>CLI.registerSpeculosTransport(port, address)</code> — which writes the port → transport mapping into a <strong>module-private registry</strong> living inside that specific module instance.</p>

<p>If your spec then writes <code>import { liveDataWithAddressCommand } from "@ledgerhq/live-common/...";</code>, Node/Detox can resolve through a <strong>different</strong> path (different <code>package.json</code> entry — <code>main</code> vs <code>module</code> vs conditional exports — different bundler resolution, different <code>node_modules</code> hoist level — any of these). You get a <strong>second module instance</strong> of <code>cliCommandsUtils</code>. Same source code, completely separate in-memory state. The transport registry in your imported instance is <em>empty</em>.</p>

<p><strong>The failure mode.</strong> Your test calls <code>liveDataWithAddressCommand(...)</code>, which spawns the CLI subprocess. The subprocess inherits <code>process.env.SPECULOS_API_PORT</code> correctly, but the CLI then asks live-common's transport registry "give me the transport for port X" — and that registry is also a separate instance, also empty, because the registration happened in the harness's instance, not the import's. CLI falls through to the only transport it can find — <strong>HID</strong> — and fails with <code>NoDevice</code>. Speculos is running. The port env var is set. The CLI binary is fine. The harness is fine. You have two clones of the same module disagreeing about state.</p>

<p><strong>Mental model.</strong> Every time you <code>import</code> something from <code>@ledgerhq/live-common</code> in a mobile spec, you might be spinning up a clone of the module's runtime state — disconnected from whatever the harness already configured. The globals are the harness's configured version. The imports are not.</p>

<p><strong>The rule.</strong> On mobile E2E, <strong>never <code>import</code> a CLI helper, an enum, or anything else the harness has already exposed as a global</strong>. Use <code>liveDataWithAddressCommand</code>, <code>CLI</code>, <code>Account</code>, <code>Currency</code>, etc. directly from <code>globalThis</code> (i.e., as the bare identifier — TypeScript knows about them because of <code>types/global.d.ts</code>). The duplication isn't stylistic — it routes you to a different module instance with different runtime state.</p>

<p><strong>Why desktop doesn't have this problem.</strong> Playwright runs the spec, the POMs, and any imports in a single Node process with a single module graph; the harness's <code>cliCommandsUtils</code> instance and the spec's are the same instance. Mobile's split between the Detox runner, the Jest worker process, and the harness server makes dual-package issues painful and silent. Items 3 above (globals installation) is your contract — respect it.</p>

<p><strong>How to spot it in a PR.</strong> Search the diff for <code>from "@ledgerhq/live-common</code>. If the imported symbol is also a documented global (see <code>types/global.d.ts</code>), reject the import and use the global. The desktop's import-everything style does not transfer to mobile.</p>

<p><strong>How to debug it when it bites.</strong> Symptoms: <code>NoDevice</code> error from the CLI subprocess, but Speculos is healthy and the port env var is correct. Confirm with: <code>console.log((globalThis as any).cliCommandsUtils === require("@ledgerhq/live-common/...").cliCommandsUtils)</code> — if that prints <code>false</code>, you have two instances. Cure: delete the import, use the global.</p>
</div>

#### `jest.globalSetup.ts`

Runs **once** before any Jest workers start:

```typescript
export default async function setup(): Promise<void> {
  const envFileName = process.env.ENV_FILE || ".env.mock";
  const envFile = path.join(__dirname, "../../apps/ledger-live-mobile", envFileName);
  try {
    await fs.access(envFile, fs.constants.R_OK);
  } catch (error) {
    throw new Error(`Mock env file not found or not readable: ${envFile}`);
  }

  setupSpeculosCleanupHandlers();
  await cleanupPreviousNanoAppJsonFile();

  await globalSetup();   // detox/runners/jest

  const testSessionIndex = detoxSession.testSessionIndex ?? 0;
  const maxRetries = detoxConfig.testRunner?.retries ?? 0;
  const isLastRetry = maxRetries > 0 && testSessionIndex >= maxRetries;

  if (isLastRetry) {
    process.env.DETOX_ENABLE_VIDEO = "true";
    process.env.DETOX_VIDEO_OPTIONS = JSON.stringify({
      android: { recording: { bitRate: 1_000_000, maxSize: 720, codec: "h264" },
                 audio: false, window: false },
      ios: { codec: "hevc" },
    });
  }
}
```

Three jobs:

1. **Validate the ENV file exists.** Before any binary is launched, check that `apps/ledger-live-mobile/.env.mock` (or the one named in `ENV_FILE`) is readable. This fails fast with a clear message.
2. **Install SIGINT/SIGTERM/SIGHUP/SIGQUIT handlers.** If you Ctrl-C the test run, these handlers clean up any Speculos containers tracked in `SPECULOS_TRACKING_FILE`. Without this, laptop disks accumulate orphan Docker containers.
3. **Enable video on the last retry only.** If `detox.config.js` had `retries: N`, the final attempt gets video recording enabled (via env vars that `detox-allure2-adapter` reads). Recording every attempt would balloon artifact size.

At the very top of the file — and of `jest.globalTeardown.ts` — you will see:

```typescript
import { register } from "tsconfig-paths";
const tsConfig = require("./tsconfig.json");
register({ baseUrl: __dirname, paths: tsConfig.compilerOptions.paths });
```

This is the TS-path bootstrap. Global setup/teardown are run by Jest **before** its transform layer applies to them in the same way it does to specs, so they need `tsconfig-paths` to resolve `@ledgerhq/live-common/e2e/...` imports.

#### `jest.globalTeardown.ts`

Runs **once** after all workers finish:

```typescript
export default async () => {
  if (process.env.CI && process.env.SHARD_INDEX === "1") {
    try {
      await initDetox();
      await launchApp();
      await loadConfig("1AccountBTC1AccountETHReadOnlyFalse", true);
      await NativeElementHelpers.waitForElementById("topbar-settings", 120_000);

      const flagsData = formatFlagsData(JSON.parse(await getFlags()));
      const envsData = formatEnvData(JSON.parse(await getEnvs()));
      await fs.appendFile(ARTIFACT_ENV_PATH, flagsData + envsData);
    } catch (err) {
      log.error("Error during CI global teardown:", sanitizeError(err));
    } finally {
      try {
        closeBridge();
        await withTimeout(cleanupDetox(), 30_000, "cleanupDetox");
      } catch { /* ... */ }
    }
  } else if (process.env.CI) {
    try { await fs.unlink(ARTIFACT_ENV_PATH); } catch { /* ... */ }
  }

  await withTimeout(globalTeardown(), 60_000, "globalTeardown");
  await Promise.all([cleanupUserdata(), forceGarbageCollection()]);
};
```

What it does:

- **Only shard 1 writes `environment.properties`.** In CI the suite is sharded. Writing the feature-flags and env dump from each shard would clobber the file; only shard 1 does it. The other shards delete any pre-existing file so stale data does not end up in the report.
- **Runs the app one more time to collect state.** It launches the app, loads `1AccountBTC1AccountETHReadOnlyFalse.json`, waits for the portfolio, and calls `getFlags()` / `getEnvs()` over the bridge. These produce the Allure environment table you see in the report header.
- **Cleans up `userdata/temp-userdata-*.json` files.** `Application.init()` creates a temp copy of the userdata JSON before handing it to Speculos; in rare crash scenarios those files are not unlinked. The glob cleanup ensures the workspace does not accumulate cruft.
- **Wraps everything in timeouts.** `proper-lockfile` has been known to deadlock in CI; `withTimeout` ensures the Jest process always exits, even if a cleanup hangs.

#### `babel.config.js`

Three lines of plugins:

```javascript
module.exports = {
  plugins: [
    "@babel/plugin-transform-named-capturing-groups-regex",
    "@babel/plugin-transform-export-namespace-from",
    ["@babel/plugin-proposal-class-properties", { loose: true }],
  ],
};
```

This config is used by `babel-jest` for `.js`/`.jsx` files only (TypeScript goes through SWC). The three plugins are required because some transitive dependencies emit code that uses those syntaxes before reaching Jest.

#### `setup.ts`

Wired via `setupFilesAfterEnv` in `jest.config.js` — so Jest runs it **once per spec file**, after the environment has been set up:

```typescript
import { device } from "detox";
import { launchApp, setupEnvironment } from "./helpers/commonHelpers";
import { close as closeBridge } from "./bridge/server";
import { getEnv, setEnv } from "@ledgerhq/live-env";
import { setAllureDescription } from "./helpers/allure/allure-helper";

const broadcastOriginalValue = getEnv("DISABLE_TRANSACTION_BROADCAST");
setupEnvironment();

beforeAll(
  async () => {
    const port = await launchApp();
    await device.reverseTcpPort(8081);
    await device.reverseTcpPort(port);
    await device.reverseTcpPort(52619); // To allow the android emulator to access the dummy app
    setAllureDescription();
  },
  process.env.CI ? 150000 : 120000,
);

afterAll(async () => {
  setEnv("DISABLE_TRANSACTION_BROADCAST", broadcastOriginalValue);
  closeBridge();
  await app.common.removeSpeculos();
});
```

What it means in practice:

- Every spec file gets an implicit `beforeAll` that launches the app, opens TCP ports 8081 (Metro), the bridge port (random free port), and 52619 (dummy-app test server). A spec file's own `beforeAll(() => app.init(...))` runs **after** this one.
- Every spec file also gets an `afterAll` that restores the broadcast env flag, closes the bridge, and asks `app.common.removeSpeculos()` to tear down any Speculos container registered during the run.
- `setAllureDescription()` adds the "Legacy Wallet" vs "Wallet 4.0" badge and the shard index to the Allure test description.

### 5.7.6 `tsconfig.json` — TypeScript for Tests

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "experimentalDecorators": true,
    "checkJs": false,
    "jsx": "react-native",
    "lib": ["es2017", "DOM"],
    "types": ["node", "jest"],
    "skipLibCheck": true,
    "noEmit": true,
    "typeRoots": ["./types", "./node_modules/@types"],
    "paths": {
      "*": ["./*"],
      "LLM/*": ["../../apps/ledger-live-mobile/src/mvvm/*"],
      "~/*": ["../../apps/ledger-live-mobile/src/*"],
      "@shared/feature-flags": ["../../shared/feature-flags/src"],
      "@ledgerhq/live-common/e2e/*": ["../../libs/ledger-live-common/src/e2e/*"],
      "@ledgerhq/live-common/lib/e2e/*": ["../../libs/ledger-live-common/src/e2e/*"],
      "@ledgerhq/live-common/hw/index": ["../../libs/ledger-live-common/src/hw/index"],
      "@ledgerhq/live-common/hw/deviceAccess": ["../../libs/ledger-live-common/src/hw/deviceAccess"]
    }
  },
  "include": ["types", "**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules", "babel.config.js", "jest.config.js", "**/*.js"]
}
```

What matters for a test author:

- **`experimentalDecorators: true`.** Enables `@Step("...")` on POM methods and `@TmsLink`/`@Tag` ideas used by Allure.
- **`"~/*"` alias.** Lets tests import app source (`~/screens/portfolio/PortfolioScreen`) when they need to reference constants shared with the production code. Rarely needed, but a safety valve.
- **`"@ledgerhq/live-common/e2e/*"` alias.** Resolves directly to the `src/` of the shared library, so the tests use the TypeScript source instead of the compiled output. This is why you can set a breakpoint in `@ledgerhq/live-common/e2e/models/Swap.ts` from a mobile test — they are the same file.
- **`"LLM/*"` alias.** Expands to `apps/ledger-live-mobile/src/mvvm/*`. It is scaffolding for the MVVM migration happening in the app; you will rarely use it, but seeing it in a file tells you that file is bridging to new architecture code.

### 5.7.7 The Bridge Layer — `bridge/server.ts`

The bridge is how the test process (Node) talks to the React Native app (JS/native) at runtime. On the app side, a WebSocket client receives messages and dispatches Redux actions. On the test side, a WebSocket **server** sends messages and optionally awaits a typed reply.

Unlike the legacy tree, the canonical workspace contains only the server (`bridge/server.ts`, 307 lines). The types and the client live under the app itself: `apps/ledger-live-mobile/e2e/bridge/types.ts` exports `MessageData` and `ServerData`, which `server.ts` imports directly:

```typescript
import { NavigatorName } from "../../../apps/ledger-live-mobile/src/const";
import {
  MessageData,
  OverrideFeatureFlagPayload,
  ServerData,
} from "../../../apps/ledger-live-mobile/e2e/bridge/types";
```

This is the one place in the workspace where a test-side file reaches across into the app's `src/` and `e2e/` directories. Conceptually, the types describe a contract: the test emits `MessageData` variants, the app emits `ServerData` variants.

Here is the message-catalog derived from what `server.ts` actually sends and receives:

| Direction | Type | Purpose |
|---|---|---|
| Test → app | `importSettings` | Replace the Redux settings slice |
| Test → app | `importAccounts` | Replace the accounts slice |
| Test → app | `overrideFeatureFlag` | Override a single feature flag at runtime |
| Test → app | `acceptTerms` | Mark terms as accepted |
| Test → app | `navigate` | Imperative navigation to a navigator name |
| Test → app | `addKnownSpeculos` | Register a Speculos device with the app's transport layer |
| Test → app | `removeKnownSpeculos` | Deregister a Speculos device |
| Test → app | `swapSetup` | Initialise the Swap Live App (optional `SWAP_API_BASE`) |
| Test → app | `waitSwapReady` | Block until the swap app is ready |
| Test → app | `waitEarnReady` | Block until the earn app is ready |
| Test → app | `getLogs` / `getFlags` / `getEnvs` | Dump runtime data for Allure env file / debugging |
| App → test | `ACK` | Acknowledges a previously sent message by id |
| App → test | `walletAPIResponse` | Streams back a response to a Wallet API request |
| App → test | `appLogs` / `appFlags` / `appEnvs` | Reply payloads for the corresponding `get*` requests |
| App → test | `swapSetupDone` | Swap Live App is initialised |
| App → test | `swapLiveAppReady` / `earnLiveAppReady` | Live apps signal readiness |
| App → test | `appFile` | Persists a file to `../artifacts/` on the test host |

Key public functions exported by `server.ts` (you call these from POMs, not from specs directly):

```typescript
export async function findFreePort(): Promise<number>
export function init(port = 8099, onConnection?: () => void)
export function close()
export async function loadConfig(fileName: string, agreed: true = true): Promise<void>
export async function setFeatureFlags(flags: PartialFeatures)
export async function setFeatureFlag(flag: OverrideFeatureFlagPayload)
export async function swapSetup()
export async function waitSwapReady()
export async function waitEarnReady()
export async function getLogs()
export async function getFlags()
export async function getEnvs()
export async function addKnownSpeculos(address: string)
export async function removeKnownSpeculos(address: string)
```

A few implementation details worth knowing:

- **Unique ids.** Every outgoing message carries a `uuid()` id so the app can ACK it. `webSocket.messages[id] = message` stores pending messages; when the app ACKs, the entry is deleted.
- **Reconnect buffering.** If the app crashes and reconnects, `server.on("connection")` resends anything still in `webSocket.messages` — tests survive a mid-run reconnection.
- **Pending-callback map.** For request/response patterns (like `getFlags`), the server registers a callback in `global.pendingCallbacks` keyed by type; the matching `onMessage` branch fires the callback when the reply arrives. A 10 s timeout (`RESPONSE_TIMEOUT`) covers most cases; swap/earn readiness extends it to 30 s.
- **`loadConfig(fileName)`.** Reads `userdata/<fileName>.json`, merges the settings with `{ shareAnalytics: false, hasSeenAnalyticsOptInPrompt: true }`, sends `importSettings`, navigates to the base navigator, sends `importAccounts`, then applies any `featureFlags.overrides` declared in the userdata. This one function implements everything that `app.init({ userdata })` ultimately does on the app side.

### 5.7.8 The Helpers Layer

`helpers/` is where low-level Detox wrappers and per-run plumbing live. Four files plus one subfolder:

#### `helpers/elementHelpers.ts` — 18 KB, the primitive catalogue

Defines two large objects exported as named modules:

- **`NativeElementHelpers`** — wraps Detox's native matchers (`by.id`, `by.text`, `element(...).tap()`, `waitFor(...).toBeVisible()`) into friendlier primitives. The methods you see promoted to `globalThis` by `jest.environment.ts` all come from here. A non-exhaustive list: `tapById`, `tapByElement`, `tapByText`, `tapByIdAndExpectToDisappear`, `typeTextById`, `typeTextByElement`, `clearTextByElement`, `waitForElementById`, `waitForElementByText`, `waitForElementNotVisible`, `getElementById`, `getElementByText`, `getElementByIdAndText`, `getElementByIdWithDescendantTexts`, `getElementsById`, `getIdByRegexp`, `getIdOfElement`, `getTextOfElement`, `getAttributesOfElement`, `countElementsById`, `isIdPresent`, `isIdVisible`, `scrollToId`, `scrollToText`, `expect` (aliased to `detoxExpect` on globals).
- **`WebElementHelpers`** — the WebView counterpart. Used when interacting with Live Apps (Swap Live App, Ramp, etc.). Methods include `getWebElementByTestId`, `tapWebElementByTestId`, `typeTextByWebTestId`, `waitWebElement`, `waitWebElementByTestId`, `getCurrentWebviewUrl`, `waitForCurrentWebviewUrlToContain`, `scrollToWebElement`, and more.

One method worth looking at in detail is `waitForElement` (or `waitForElementById`), which accepts an optional `errorElementId`. When provided, it polls both the expected element **and** the error element in parallel; if the error element appears first, it throws immediately with a clear message. This is the backbone of the fail-fast behaviour used across swap POMs.

#### `helpers/commonHelpers.ts`

Cross-cutting runtime helpers. The most important exports:

- `launchApp()` — allocates a free port, closes any existing bridge, inits a new bridge on that port, and calls `device.launchApp({ launchArgs: { wsPort: port, ... } })`. The app sees the port in its launch args and connects its WS client to it.
- `openDeeplink(path)` — deeplink the app: `ledgerlive://<path>`.
- `setupEnvironment()` — sets `DISABLE_APP_VERSION_REQUIREMENTS`, `MOCK`, `DETOX`, `E2E_NANO_APP_VERSION_PATH`, and reads `DISABLE_TRANSACTION_BROADCAST` from env.
- `isAndroid()` / `isIos()` / `isSpeculosRemote()` / `isRemoteIos()` — platform guards used throughout POMs.
- `describeIfNotNanoS(...)` — skip blocks on Nano S (firmware too old for some flows).
- `takeAppScreenshot(name)` — screenshots the device and attaches to Allure.
- `captureNativeViewHierarchy(label)` — dumps the native view tree XML to Allure.
- `normalizeText(text)` — replaces multiple whitespace with a single space and normalises non-breaking spaces (` `). Used by swap assertions that compare RN-rendered text.
- `logMemoryUsage()` — appends process memory usage to Allure.
- `isWallet40` — top-level boolean (`process.env.E2E_ENABLE_WALLET40 !== "0"`) used to fork behaviour for Wallet 4.0-specific POM code.

#### `helpers/errorHelpers.ts`

Tiny but important. Defines `ERROR_MODAL_SELECTORS = ["generic-error-modal"]`, then `detectErrorModal(timeout)` and `checkForErrorModals(timeout, customMessage)`. Used by POMs (and automatically by `waitForElement`) to fail fast on generic error dialogs.

#### `helpers/pageScroller.ts`

A small class that encapsulates "keep scrolling until this element appears, switch direction at the edge, give up after N attempts". Used internally by `NativeElementHelpers.scrollToId` and `.scrollToText`. Constants: 10 attempts per direction, 7 stall cycles before switching, 500 ms delay between Android scrolls.

#### `helpers/allure/allure-helper.ts`

A single function `setAllureDescription()` that reads the current test file path and composes the Allure description with `📄 Test file: <name>` and `🏷️ Test run on: <mode>` (where mode is Wallet 4.0 or Legacy Wallet), plus the shard index. Called once per spec from `setup.ts`.

### 5.7.9 The Models Layer

Mobile's `models/` is deliberately thin (three files). The heavy models — `Swap`, `Transaction`, `Delegate`, `Account`, `Currency`, `AppInfos`, `Fee`, `Provider` — live in `@ledgerhq/live-common/e2e/` and are shared with desktop. The three files here are **cross-POM composers**:

- `models/currencies.ts` — a single helper: `getCurrencyManagerApp(currencyId)` returns the lowercase first word of the Manager app name for a given currency. Used by `manager.page` to find the right install card.
- `models/send.ts` — `verifyAppValidationSendInfo(transaction, amount)`. Orchestrates `app.deviceValidation.expect*` calls for currencies that show amount/recipient/sender/ENS on the device screen. The function knows which assertions to make per currency (ETH amounts shown, ATOM sender shown, POL recipient shown, etc.).
- `models/stake.ts` — `verifyAppValidationStakeInfo(delegation, amount, fees?)` and `verifyStakeOperationDetailsInfo(...)`. Same shape as `send.ts` but for delegation flows.

These are not POMs (no `@Step`, no page state), and not models either in the data-class sense — they are assertion compositors. Use them from a spec's `it()` body when you want "verify the device screen matches a transaction" without rewriting the per-currency logic every time.

### 5.7.10 The Page Layer — The Centrepiece

The page layer is the largest and most-touched part of the workspace. It is organised into a single hub (`page/index.ts`) and eleven thematic subdirectories.

#### `page/index.ts` — the `Application` class

This is the file every spec reaches into via `app.*`. It is 234 lines, imports 32 POMs, and exposes each one behind a lazy-initialised getter. Here it is in full:

```typescript
import { Step } from "jest-allure2-reporter/api";
import AssetAccountsPage from "./accounts/assetAccounts.page";
import AccountPage from "./accounts/account.page";
import AccountsPage from "./accounts/accounts.page";
import AddAccountDrawer from "./accounts/addAccount.drawer";
import CommonPage from "./common.page";
import CustomLockscreenPage from "./stax/customLockscreen.page";
import DeviceValidationPage from "./trade/deviceValidation.page";
import DiscoverPage from "./discover/discover.page";
import LedgerSyncPage from "./settings/ledgerSync.page";
import ManagerPage from "./manager/manager.page";
import MarketPage from "./market/market.page";
import OnboardingStepsPage from "./onboarding/onboardingSteps.page";
import OperationDetailsPage from "./trade/operationDetails.page";
import PasswordEntryPage from "./passwordEntry.page";
import PortfolioEmptyStatePage from "./wallet/portfolioEmptyState.page";
import PortfolioPage from "./wallet/portfolio.page";
import ReceivePage from "./trade/receive.page";
import SendPage from "./trade/send.page";
import SettingsGeneralPage from "./settings/settingsGeneral.page";
import SettingsHelpPage from "./settings/settingsHelp.page";
import SettingsPage from "./settings/settings.page";
import SpeculosPage from "./speculos.page";
import StakePage from "./trade/stake.page";
import SwapPage from "./trade/swap.page";
import SwapLiveAppPage from "./liveApps/swapLiveApp";
import WalletTabNavigatorPage from "./wallet/walletTabNavigator.page";
import MainNavigationPage from "./wallet/mainNavigation.page";
import CeloManageAssetsPage from "./trade/celoManageAssets.page";
import TransferMenuDrawer from "./wallet/transferMenu.drawer";
import BuySellPage from "./trade/buySell.page";
import EarnDashboardPage from "./trade/earnDasboard.page";
import EarnV2DashboardPage from "./trade/earnV2Dashboard.page";
import ModularDrawer from "./drawer/modular.drawer";

import path from "path";
import fs from "fs";
import { InitializationManager, InitOptions } from "../utils/initUtil";
import { randomUUID } from "crypto";

export type ApplicationOptions = InitOptions;

export const getUserdataPath = (userdata: string) => {
  return path.resolve("userdata", `${userdata}.json`);
};

const lazyInit = <T>(PageClass: new () => T) => {
  let instance: T | null = null;
  return () => {
    if (!instance) instance = new PageClass();
    return instance;
  };
};

export class Application {
  private assetAccountsPageInstance = lazyInit(AssetAccountsPage);
  private accountPageInstance = lazyInit(AccountPage);
  private accountsPageInstance = lazyInit(AccountsPage);
  private addAccountDrawerInstance = lazyInit(AddAccountDrawer);
  private commonPageInstance = lazyInit(CommonPage);
  private customLockscreenPageInstance = lazyInit(CustomLockscreenPage);
  private deviceValidationPageInstance = lazyInit(DeviceValidationPage);
  private discoverPageInstance = lazyInit(DiscoverPage);
  private ledgerSyncPageInstance = lazyInit(LedgerSyncPage);
  private managerPageInstance = lazyInit(ManagerPage);
  private marketPageInstance = lazyInit(MarketPage);
  private onboardingPageInstance = lazyInit(OnboardingStepsPage);
  private operationDetailsPageInstance = lazyInit(OperationDetailsPage);
  private passwordEntryPageInstance = lazyInit(PasswordEntryPage);
  private portfolioEmptyStatePageInstance = lazyInit(PortfolioEmptyStatePage);
  private portfolioPageInstance = lazyInit(PortfolioPage);
  private receivePageInstance = lazyInit(ReceivePage);
  private sendPageInstance = lazyInit(SendPage);
  private settingsPageInstance = lazyInit(SettingsPage);
  private settingsGeneralPageInstance = lazyInit(SettingsGeneralPage);
  private speculosPageInstance = lazyInit(SpeculosPage);
  private stakePageInstance = lazyInit(StakePage);
  private swapLiveAppInstance = lazyInit(SwapLiveAppPage);
  private swapPageInstance = lazyInit(SwapPage);
  private walletTabNavigatorPageInstance = lazyInit(WalletTabNavigatorPage);
  private mainNavigationPageInstance = lazyInit(MainNavigationPage);
  private celoManageAssetsPageInstance = lazyInit(CeloManageAssetsPage);
  private TransferMenuDrawerInstance = lazyInit(TransferMenuDrawer);
  private buySellPageInstance = lazyInit(BuySellPage);
  private settingsHelpPageInstance = lazyInit(SettingsHelpPage);
  private earnDashboardPageInstance = lazyInit(EarnDashboardPage);
  private readonly earnV2DashboardPageInstance = lazyInit(EarnV2DashboardPage);
  private modularDrawerPageInstance = lazyInit(ModularDrawer);

  @Step("Account initialization")
  public async init(options: ApplicationOptions) {
    this.modularDrawer.resetFlags();
    const userdataSpeculos = `temp-userdata-${randomUUID()}`;
    const userdataPath = getUserdataPath(userdataSpeculos);
    fs.copyFileSync(getUserdataPath(options.userdata || "skip-onboarding"), userdataPath);
    try {
      await InitializationManager.initialize(options, userdataPath, userdataSpeculos);
    } finally {
      fs.unlinkSync(userdataPath);
    }
  }

  public get assetAccountsPage()    { return this.assetAccountsPageInstance(); }
  public get account()              { return this.accountPageInstance(); }
  public get accounts()             { return this.accountsPageInstance(); }
  public get addAccount()           { return this.addAccountDrawerInstance(); }
  public get common()               { return this.commonPageInstance(); }
  public get customLockscreen()     { return this.customLockscreenPageInstance(); }
  public get deviceValidation()     { return this.deviceValidationPageInstance(); }
  public get discover()             { return this.discoverPageInstance(); }
  public get ledgerSync()           { return this.ledgerSyncPageInstance(); }
  public get manager()              { return this.managerPageInstance(); }
  public get market()               { return this.marketPageInstance(); }
  public get onboarding()           { return this.onboardingPageInstance(); }
  public get operationDetails()     { return this.operationDetailsPageInstance(); }
  public get passwordEntry()        { return this.passwordEntryPageInstance(); }
  public get portfolio()            { return this.portfolioPageInstance(); }
  public get portfolioEmptyState()  { return this.portfolioEmptyStatePageInstance(); }
  public get receive()              { return this.receivePageInstance(); }
  public get send()                 { return this.sendPageInstance(); }
  public get settings()             { return this.settingsPageInstance(); }
  public get settingsGeneral()      { return this.settingsGeneralPageInstance(); }
  public get speculos()             { return this.speculosPageInstance(); }
  public get stake()                { return this.stakePageInstance(); }
  public get swap()                 { return this.swapPageInstance(); }
  public get swapLiveApp()          { return this.swapLiveAppInstance(); }
  public get walletTabNavigator()   { return this.walletTabNavigatorPageInstance(); }
  public get mainNavigation()       { return this.mainNavigationPageInstance(); }
  public get celoManageAssets()     { return this.celoManageAssetsPageInstance(); }
  public get transferMenuDrawer()   { return this.TransferMenuDrawerInstance(); }
  public get buySell()              { return this.buySellPageInstance(); }
  public get settingsHelp()         { return this.settingsHelpPageInstance(); }
  public get earnDashboard()        { return this.earnDashboardPageInstance(); }
  public get earnV2Dashboard()      { return this.earnV2DashboardPageInstance(); }
  public get modularDrawer()        { return this.modularDrawerPageInstance(); }
}
```

Three patterns to internalise:

1. **`lazyInit<T>()`.** Each POM is wrapped in a thunk that creates the instance on first read and returns the cached instance on subsequent reads. This keeps `new Application()` cheap (nothing is constructed up front) and ensures every test call to `app.portfolio.xxx` hits the same `PortfolioPage` object — useful when a POM holds cached state like flags.
2. **`init(options)` decorated with `@Step("Account initialization")`.** Every spec starts with `await app.init({ userdata, speculosApp, cliCommands, featureFlags })`. The method creates a **temporary copy** of the requested userdata JSON (`temp-userdata-<uuid>.json`) so that Speculos CLI commands that mutate the userdata file do not corrupt the canonical snapshot. It then delegates to `InitializationManager.initialize(...)` (see 4.7.14) and unlinks the temp file in `finally`. The `@Step` decorator wraps the whole call in an Allure step — you will see it as the first step in every test's Allure report.
3. **Naming is consistent.** Every getter returns an instance of the `*Page` or `*Drawer` class whose default export matches the file name. When you need to find a POM's source, the mapping is mechanical: `app.swap` → `page/trade/swap.page.ts`, `app.modularDrawer` → `page/drawer/modular.drawer.ts`, `app.swapLiveApp` → `page/liveApps/swapLiveApp.ts` (note: no `.page` suffix).

#### The subdirectory taxonomy

Eleven subdirectories, each with a purpose:

- **`accounts/`** — Everything to do with the account list and individual accounts: `accounts.page` (list screen), `account.page` (single account details), `addAccount.drawer` (the bottom sheet that opens when adding a new account), `assetAccounts.page` (the screen that shows accounts filtered by asset).
- **`common.page.ts`** — Shared UI elements at the workspace root, not in a subfolder. `CommonPage` is the base class that `SwapPage`, `AccountsPage`, and several others extend. It centralises search bar, close/back buttons, account item regex, `selectAccount(account)`, `selectKnownDevice(index)`, and `removeSpeculos()`.
- **`discover/`** — The Discover tab (`discover.page`). Holds a static list of Live Apps (`MoonPay`, `Ramp`, `Kiln`, `Lido`, `1inch`, `Zerion`, `Transak`).
- **`drawer/`** — Currently one file: `modular.drawer.ts`. This is the reusable asset/network/account picker that shows up across send, receive, swap, earn, and buySell since Wallet 4.0. The POM exposes `performSearchByTicker`, `selectCurrencyByTicker`, `selectNetworkIfAsked`, `selectFirstAccount`, `tapAddNewOrExistingAccountButtonMAD`, and reads the `llmModularDrawer` flag to decide which flow applies.
- **`liveApps/`** — WebView-driven Live Apps. Currently: `swapLiveApp.ts` (the Swap Live App that renders inside Ledger Live). Uses `WebElementHelpers` instead of `NativeElementHelpers` for all interactions.
- **`manager/`** — My Ledger / Manager screen (`manager.page`), two lines deep: deeplink, expect.
- **`market/`** — Market tab (`market.page`): search asset, star, filter starred, open asset, quick actions.
- **`onboarding/`** — `onboardingSteps.page`. Handles the full first-run flow: language, Get Started, accept analytics, explore without device, recovery phrase / setup Ledger / access wallet forks.
- **`passwordEntry.page.ts`** — The password lock screen (at the root of `page/`, not in a folder). Minimal: enter password, login, expect lock / no lock.
- **`settings/`** — `settings.page` (card list), `settingsGeneral.page` (password, theme, language), `settingsHelp.page` (links), `ledgerSync.page` (cloud sync flow).
- **`speculos.page.ts`** — Wrapper around live-common's Speculos helpers: `signSendTransaction(tx)`, `signDelegationTransaction(delegation)`, `verifyAmountsAndAcceptSwap(swap, amount)`, `goToSettings()`, `activateContractData()`, `activateExpertMode()`. Also holds `setExchangeDependencies(accounts)` — required before any swap to tell the Speculos backend which nano apps must be available.
- **`stax/`** — Stax-specific screens. Currently one file: `customLockscreen.page`.
- **`trade/`** — The densest folder. All the transactional flows and their side-effects: `swap.page` (10 KB — swap confirmation UI, history, operation details, export CSV), `send.page`, `receive.page`, `stake.page`, `buySell.page`, `earnDasboard.page` and `earnV2Dashboard.page` (two generations of the earn UI), `operationDetails.page`, `deviceValidation.page` (the "Verify on device" full-screen wait), `celoManageAssets.page`.
- **`wallet/`** — Portfolio and top-level navigation: `portfolio.page` (17 KB — huge, covers the portfolio screen in both Legacy and Wallet 4.0 layouts), `portfolioEmptyState.page`, `walletTabNavigator.page` (Portfolio/Market tab switching), `mainNavigation.page` (Wallet 4.0 main nav), `transferMenu.drawer` (the transfer bottom sheet).

#### Sample POM — `page/trade/swap.page.ts` (annotated excerpt)

```typescript
export default class SwapPage extends CommonPage {
  baseLink = "swap";
  confirmSwapOnDeviceDrawerId = "confirm-swap-on-device";
  swapSuccessTitleId = "swap-success-title";
  swapOperationDetailsScrollViewId = "swap-operation-details-scroll-view";
  // ... more testIDs

  operationDetails = {
    fromAccount: "swap-operation-details-fromAccount",
    toAccount: "swap-operation-details-toAccount",
    fromAmount: "swap-operation-details-fromAmount",
    toAmount: "swap-operation-details-toAmount",
    provider: "swap-operation-details-provider",
    providerLink: "swap-operation-details-provider-link",
    swapId: "swap-operation-details-swapId",
    date: "swap-operation-details-date",
    viewInExplorerButton: "operation-detail-view-in-explorer-button",
  };

  @Step("Open swap via deeplink")
  async openViaDeeplink() {
    await openDeeplink(this.baseLink);
    await waitForElementById(app.common.walletApiWebview);
  }

  @Step("Expect swap page")
  async expectSwapPage() {
    if (await IsIdVisible(this.swapFormTabId, 5000)) {
      await detoxExpect(this.swapFormTab()).toBeVisible();
    } else {
      // Wallet 4.0 swap screen does not expose `swap-form-tab`
      await detoxExpect(getElementById(app.common.walletApiWebview)).toBeVisible();
    }
  }

  @Step("Check swap operation row details")
  async checkSwapOperation(swapId: string, swap: SwapType) {
    await detoxExpect(this.operationRows()).toBeVisible();
    await detoxExpect(this.getSpecificOperation(swapId)).toBeVisible();
    jestExpect(await getTextOfElement(this.specificOperationAccountFromId(swapId))).toEqual(
      swap.accountToDebit.accountName,
    );
    // ...
  }

  @Step("Click on export operations")
  async clickExportOperations() {
    await tapById(this.exportOperationsButton);
    const filePath = path.resolve(__dirname, "../../artifacts/ledgerwallet-swap-history.csv");
    const fileExists = await FileUtils.waitForFileToExist(filePath, 5000);
    jestExpect(fileExists).toBeTruthy();
  }
}
```

Notice four things:

1. **`extends CommonPage`.** Several trade POMs extend `CommonPage` to inherit the search bar, account item regex, close/back buttons, `selectAccount`, `selectKnownDevice`, etc.
2. **testIDs are fields, not magic strings.** Every testID used by the POM is declared at the top. If the app renames one, only the POM changes — specs stay untouched.
3. **Every user-facing method is decorated with `@Step(...)`.** These become nested Allure steps. When a test fails, the Allure timeline tells you exactly which step broke.
4. **Wallet 4.0 conditionals are inline.** `expectSwapPage()` checks `IsIdVisible(this.swapFormTabId, 5000)` — if the legacy testID is visible, verify it; otherwise fall back to the WebView check. This is the dominant pattern for handling the in-flight Wallet 4.0 migration across POMs.

#### Sample POM — `page/common.page.ts`

The shared base class. A few representative methods:

```typescript
export default class CommonPage {
  assetScreenFlatlistId = "asset-screen-flatlist";
  searchBarId = "common-search-field";
  accountCardPrefix = "account-card-";
  accountItemId = "account-item-";
  accountItemNameRegExp = new RegExp(`${this.accountItemId}.*-name`);
  deviceItem = (deviceId: string): string => `device-item-${deviceId}`;
  deviceItemRegex = /device-item-.*/;
  walletApiWebview = "wallet-api-webview";
  errorPage = new ErrorPage();

  accountId = (account: Account) =>
    `test-id-account-${getParentAccountName(account)}${account.tokenType !== undefined ? ` (${account.currency.ticker})` : ""}`;

  @Step("Select currency to debit")
  async selectAccount(account: Account) {
    const accountId = this.accountId(account);
    await waitForElementById(accountId);
    await tapByIdAndExpectToDisappear(accountId);
  }

  @Step("Select a known device")
  async selectKnownDevice(index = 0) {
    const speculosAddress = process.env.DEVICE_PROXY_URL;
    const elementId = speculosAddress
      ? this.deviceItem(`speculos|${speculosAddress}`)
      : this.deviceItemRegex;
    await waitForElementById(elementId);
    await tapById(elementId, speculosAddress ? undefined : index);
  }

  @Step("Remove Speculos")
  async removeSpeculos(deviceId?: string) {
    await removeSpeculosAndDeregisterKnownSpeculos(deviceId);
  }
}
```

The `accountId(account: Account)` function translates a shared `Account` enum value into the testID the mobile app renders. This is the reason tests can write `await app.common.selectAccount(Account.BTC_1)` — the enum carries everything the POM needs to build a matcher. The same `Account` enum is consumed by desktop; the two suites thus agree on the fixture's semantic identity while each translates it to its own locator grammar.

### 5.7.11 The Specs Layer — 213 files across 15+ buckets

`specs/` holds the actual Jest test files. There are 213 `.spec.ts` files in the canonical workspace today. They are organised into top-level buckets under `specs/`:

```
specs/
├── account/                 # account rename
├── addAccount/              # 17 per-currency specs + addAccount.ts driver
├── buySell/                 # Buy/Sell flows
├── delegate/                # Staking delegation per currency
├── deleteAccount/           # Account deletion flows
├── deposit/                 # Deposit entry flows
├── earn/                    # Earn dashboard
├── ledgerSync/              # Cross-device sync
├── portfolio/               # Portfolio screen checks
├── send/                    # 17 per-currency send specs + send.ts driver + sendInvalid/, sendValidAddress/
├── settings/                # Settings flows
├── subAccount/              # Token/sub-account flows
├── swap/                    # 9 currency-pair specs + swap.ts driver + swap.setup.ts + otherTestCases/
│   └── otherTestCases/      # Edge cases + swap.other.ts driver + tooLowAmountForQuoteSwaps/
├── verifyAddress/           # Receive address verification per currency
├── wallet40/                # Wallet 4.0-only specs
├── deeplinks.spec.ts
├── languageChange.spec.ts
├── market.spec.ts
├── onboardingReadOnly.spec.ts
└── userOpensApplication.spec.ts
```

#### The thin-spec + shared-driver pattern

Almost every bucket follows the same convention: a driver module (`swap.ts`, `swap.other.ts`, `send.ts`, `addAccount.ts`, etc.) declares a family of `export function runXxxTest(...)` helpers; the individual `.spec.ts` files are **thin config bundles** that import the driver and call it with concrete data.

A representative pair:

`specs/swap/swapBTC_NATIVE_SEGWIT_LTC.spec.ts` (full file):

```typescript
import { Account } from "@ledgerhq/live-common/e2e/enum/Account";
import { runSwapTest } from "./swap";

const swap = new Swap(Account.BTC_NATIVE_SEGWIT_1, Account.LTC_1, "0.0006", undefined, Fee.MEDIUM);
runSwapTest(
  swap,
  ["B2CQA-3078"],
  [
    "@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
    "@bitcoin", "@family-bitcoin",
    "@litecoin", "@family-litecoin",
  ],
);
```

`specs/swap/swap.ts` — the driver (abridged):

```typescript
export function runSwapTest(swap: SwapType, tmsLinks: string[], tags: string[]) {
  describe("Swap - Accepted (without tx broadcast)", () => {
    beforeAll(async () => {
      await beforeAllFunction(swap);   // app.init + swap setup
    });

    tmsLinks.forEach(tmsLink => $TmsLink(tmsLink));
    tags.forEach(tag => $Tag(tag));
    it(`Swap ${swap.accountToDebit.currency.name} to ${swap.accountToCredit.currency.name}`, async () => {
      const minAmount = await app.swapLiveApp.getMinimumAmount(...);
      const swapAmount = /* adjusted for XRP precision */;
      await performSwapUntilQuoteSelectionStep(...);
      const provider = await app.swapLiveApp.selectExchange();
      await app.swapLiveApp.checkExchangeButtonHasProviderName(provider.uiName);
      await app.common.disableSynchronizationForiOS();
      await app.swapLiveApp.tapExecuteSwap();
      await app.swap.verifyAmountsAndAcceptSwap(swap, swapAmount);
      await app.swap.waitForSuccessAndContinue();
    });
  });
}
```

Two observations:

1. **The spec is data.** `swapBTC_NATIVE_SEGWIT_LTC.spec.ts` never calls `app.*` directly. It declares a `Swap` instance, a TMS link, and a list of tags, then delegates to the driver. Adding a new currency pair is a one-file change — no test logic to copy.
2. **The driver wraps `describe`/`beforeAll`/`it` itself.** Because Jest `testMatch` only runs files ending in `.spec.ts`, drivers are safe to co-locate: they are only loaded when a spec imports them.

`specs/swap/otherTestCases/swapExportHistoryOperations.spec.ts` (QAA-604):

```typescript
import { Account } from "@ledgerhq/live-common/e2e/enum/Account";
import { Provider } from "@ledgerhq/live-common/e2e/enum/Provider";
import { Addresses } from "@ledgerhq/live-common/e2e/enum/Addresses";
import { runExportSwapHistoryOperationsTest } from "./swap.other";

const swapHistoryTestConfig = {
  swap: new Swap(Account.SOL_1, Account.ETH_1, "0.07"),
  provider: Provider.EXODUS,
  swapId: "wQ90NrWdvJz5dA4",
  addressFrom: Addresses.SWAP_HISTORY_SOL_FROM,
  addressTo: Addresses.SWAP_HISTORY_ETH_TO,
  tmsLinks: ["B2CQA-604"],
  tags: [
    "@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
    "@solana", "@family-solana", "@ethereum", "@family-evm",
  ],
};

runExportSwapHistoryOperationsTest(
  swapHistoryTestConfig.swap,
  swapHistoryTestConfig.provider,
  swapHistoryTestConfig.swapId,
  swapHistoryTestConfig.addressFrom,
  swapHistoryTestConfig.addressTo,
  swapHistoryTestConfig.tmsLinks,
  swapHistoryTestConfig.tags,
);
```

Note:

- **Shared enums from `@ledgerhq/live-common/e2e/`.** `Account.SOL_1`, `Provider.EXODUS`, `Addresses.SWAP_HISTORY_SOL_FROM` — exactly the same references desktop uses.
- **No `describe` or `it` in the spec.** The entire test lives in the driver (`swap.other.ts`'s `runExportSwapHistoryOperationsTest(...)`).
- **Tags drive CI filters.** The `test_filter` input on the reusable CI workflow accepts tag expressions — running only `@family-solana` or `@NanoSP` is a one-flag operation at the PR dispatch.

Inside `specs/swap/otherTestCases/swap.other.ts` (538 lines) you will find every driver for swap edge cases: `runSwapWithoutAccountTest`, `runSwapCheckProviderTest`, `runExportSwapHistoryOperationsTest`, `runSwapUserRefusesTransactionTest`, `runSwapSwitchSendAndReceiveCurrenciesTest`, `runSwapWithDifferentSeedTest`, `runSwapWithSendMaxTest`, `runSwapTooLowAmountForQuoteTest`, `runSwapNetworkFeesAboveAccountBalanceTest`, etc. Each spec in `otherTestCases/` matches exactly one of these.

Some specs at the top of `specs/` are not drivered — they are one-off scenarios:

- `market.spec.ts` — a single `describe` block that initialises with ETH, navigates to Market, stars ETH, filters starred. Tags `@family-evm`, `@ethereum`.
- `onboardingReadOnly.spec.ts`, `userOpensApplication.spec.ts`, `languageChange.spec.ts`, `deeplinks.spec.ts` — small, standalone.

### 5.7.12 Userdata JSON Files

`userdata/` holds Redux state snapshots the bridge uses to hydrate the app. Every file has the same top-level shape:

```
{
  "data": {
    "SPECTRON_RUN": { "localStorage": { ... } },
    "settings": { ... Redux settings slice ... },
    "user": { "id": "<uuid>" },
    "accounts": [ ... list of accounts, possibly huge ... ],
    "countervalues": { ... },
    "featureFlags": { "overrides": { ... optional ... } }
  }
}
```

The simplest one is `skip-onboarding.json` (~1 KB):

```json
{
  "data": {
    "SPECTRON_RUN": { "localStorage": { "acceptedTermsVersion": "2019-12-04" } },
    "settings": {
      "hasCompletedOnboarding": true,
      "counterValue": "USD",
      "language": "en",
      "theme": "light",
      "locale": "en-US",
      "hasSeenAnalyticsOptInPrompt": true,
      "preferredDeviceModel": "nanoS",
      "hasInstalledApps": true,
      "starredAccountIds": [],
      "hasPassword": false
    },
    "user": { "id": "08cf3393-c5eb-4ea7-92de-0deea22e3971" },
    "accounts": [],
    "countervalues": {}
  }
}
```

This is the default userdata that `Application.init()` falls back to when the caller omits `userdata`. "Onboarded, no accounts" is the most common starting state for feature tests.

The heaviest is `swap-history.json` (~2.3 MB). It contains the Redux state that powers QAA-604 (Swap Export History Operations) — historical swap operations, pre-provisioned accounts for the relevant currencies, etc. The pattern generalises: the larger the test surface, the more state the userdata file encodes.

#### How userdata is resolved

`page/index.ts` exports the resolver:

```typescript
export const getUserdataPath = (userdata: string) => {
  return path.resolve("userdata", `${userdata}.json`);
};
```

Relative to `process.cwd()`, which Detox sets to `e2e/mobile/`. `Application.init()` calls this twice: once to resolve the source file (the canonical snapshot, read-only), and once to compute the destination for the per-test copy (`temp-userdata-<uuid>.json`). Only the temp copy is ever passed to Speculos CLI commands or loaded over the bridge; the canonical file is never mutated.

### 5.7.13 Scripts and Types

#### `scripts/typecheck.js`

```javascript
const ts = require("typescript");
const rootDirectory = path.resolve(__dirname, "..", "..", "..");
const e2eDirectory = path.resolve(__dirname, "..");

function compile() {
  const config = ts.parseJsonConfigFileContent(require("../tsconfig.json"), ts.sys, e2eDirectory);
  const program = ts.createProgram(config.fileNames, { ...config.options, noEmit: true });

  const allDiagnostics = ts
    .getPreEmitDiagnostics(program)
    .filter(
      diag =>
        diag.file &&
        diag.file.fileName.startsWith(e2eDirectory) &&
        /\.tsx?/.test(diag.file.fileName),
    );
  // ... print diagnostics, exit 1 on errors
}
compile();
```

Why a custom script? Because the TypeScript program has to include cross-workspace references (`apps/ledger-live-mobile/src/const`, `libs/ledger-live-common/src/e2e/*`), so naive `tsc` would report errors in those trees — which are not this workspace's responsibility. The filter `diag.file.fileName.startsWith(e2eDirectory)` scopes the output to errors that originate **inside** `e2e/mobile/`. Run it with `pnpm --filter e2e-mobile run typecheck` or `pnpm typecheck` from within `e2e/mobile/`.

#### `scripts/build-ai-artifact.sh`

A shell script that post-processes Allure results and collects them into an archive for AI-assisted failure analysis. Enabled by the `generate_ai_artifacts` input on the reusable CI workflow; see Part 8 Chapter 8.3.

#### `types/global.d.ts`

The single source of truth for the ambient globals that `jest.environment.ts` installs. Every `tapById`, `waitForElementById`, `app`, `Currency`, etc. that you use in a spec file without importing is declared here. The file is about 110 lines and maps one-to-one with the assignments inside `jest.environment.ts`'s `setup()`.

A small excerpt:

```typescript
declare global {
  var IS_FAILED: boolean;
  var speculosDevices: Map<string, number>;
  var webSocket: {
    wss: Server | undefined;
    ws: WebSocket | undefined;
    messages: { [id: string]: MessageData };
    e2eBridgeServer: Subject<ServerData>;
  };
  var app: Application;
  var Step: typeof StepType;
  var $TmsLink: typeof $TmsLinkType;
  var $Tag: typeof $TagType;
  var Currency: typeof CurrencyType;
  var Account: typeof AccountType;
  var Swap: typeof SwapType;
  var tapById: typeof NativeElementHelpers.tapById;
  var waitForElementById: typeof NativeElementHelpers.waitForElementById;
  // ... and many more
}
```

When you add a helper to `jest.environment.ts`, you must add its ambient type here. Otherwise the spec compiles locally but CI typecheck fails.

#### `types/jest-allure2-reporter.d.ts`

A small module augmentation that declares `$TmsLink(name: string)` and `$Tag(name: string)` as top-level calls (they come from `jest-allure2-reporter/api`, but typing them as globals makes them ergonomic in specs).

### 5.7.14 `utils/initUtil.ts` — the Real Engine Behind `app.init()`

`Application.init()` is a three-liner; the real work happens in `utils/initUtil.ts` inside the `InitializationManager.initialize(...)` static method. Understanding this file is the difference between "I can write a spec that calls `app.init({})`" and "I know why my test deadlocked during initialisation".

The exported types:

```typescript
export type InitOptions = {
  speculosApp?: SpeculosAppType;
  cliCommands?: CliCommand[];
  cliCommandsOnApp?: {
    app: SpeculosAppType;
    cmd: CliCommand;
  }[];
  userdata?: string;
  testedCurrencies?: string[];
  featureFlags?: PartialFeatures;
};
```

`InitOptions` is re-exported from `page/index.ts` as `ApplicationOptions`. Every argument passed to `app.init(...)` must match this shape.

The static entry point:

```typescript
export class InitializationManager {
  static async initialize(
    options: InitOptions,
    userdataPath: string,
    userdataSpeculos: string,
  ): Promise<void> {
    const { speculosApp, cliCommands = [], cliCommandsOnApp = [], featureFlags } = options;

    // 1. Group commands by app name
    const commandsByAppMap = new Map<string, { app: SpeculosAppType; cmds: CliCommand[] }>();
    for (const { app, cmd } of cliCommandsOnApp) {
      const existing = commandsByAppMap.get(app.name);
      if (existing) { existing.cmds.push(cmd); }
      else { commandsByAppMap.set(app.name, { app, cmds: [cmd] }); }
    }
    const commandsByApp = Array.from(commandsByAppMap.values());

    // 2. Launch all needed Speculos devices in parallel
    const appsToLaunch = [
      ...new Map(
        commandsByApp.map(x => x.app)
          .concat(speculosApp ? [speculosApp] : [])
          .map(app => [app.name, app]),
      ).values(),
    ];
    const speculosDevices = await launchSpeculosDevices(appsToLaunch);

    // 3. Execute app-specific commands (with retry & instance recreation)
    await executeCliCommandsOnApp(commandsByApp, speculosDevices, userdataPath, speculosApp);

    // 4. Set up the main Speculos app (if declared)
    if (speculosApp) {
      await setupMainSpeculosApp(speculosApp, speculosDevices);
    }

    // 5. Execute global commands (with retry & full-run recovery)
    await executeCliCommands(cliCommands, userdataPath, speculosApp, speculosDevices);

    // 6. Load userdata over the bridge
    await loadConfig(userdataSpeculos, true);

    // 7. Apply Wallet 4.0 default flags + user-supplied overrides
    const defaultFlags = {
      lwmWallet40: { enabled: isWallet40, params: { /* ... */ } },
      llmModularDrawer: { enabled: true, params: { /* ... */ } },
    };
    await setFeatureFlags({ ...defaultFlags, ...featureFlags });
  }
}
```

Seven phases, three retry loops, one load, one flag merge. Let me unpack the important ones:

- **Phase 1: Group commands by app.** `cliCommandsOnApp` is a flat array — two commands for the same Speculos app share one launch. The grouping halves the Speculos surface when your test needs "live-data for BTC, live-data for ETH, token-allowance for ETH" (the ETH ops run against a single ETH Speculos, not two).
- **Phase 2: Parallel launch.** All Speculos containers boot concurrently via `Promise.all`. Each entry records `name`, `speculosPort`, `deviceId`. The test does not wait for Speculos N+1 before asking for Speculos N's data.
- **Phase 3: Per-app commands, with retry.** For each grouping, try the commands up to three times. On failure, delete the Speculos container, re-launch a fresh one, re-register, and retry. Between groups that are **not** the main `speculosApp`, the container is deleted — the live-data fetch is done, we do not need that container any more.
- **Phase 4: Main Speculos setup.** If `options.speculosApp` is set, that container becomes "the device the app will talk to": `registerSpeculos(port)` plus `registerKnownSpeculos(port)` (which sends `addKnownSpeculos` over the bridge so the app knows about it). Three-attempt retry here too, with full instance recreation on failure.
- **Phase 5: Global commands, with full-run recovery.** `cliCommands` (without `app:`) run against the main Speculos. On failure, the main Speculos is recreated **and the entire command list restarts**. This protects against partial-mutation states (e.g. a CLI command half-applied a token allowance and then failed).
- **Phase 6: Load userdata.** `loadConfig(userdataSpeculos, true)` reads the temp userdata file, sends it over the bridge. Under the hood this translates to `importSettings`, `navigate`, `importAccounts`, and optionally `setFeatureFlags({ overrides })`.
- **Phase 7: Feature-flag merge.** Default flags for the Wallet 4.0 rollout and the modular drawer get applied, then any caller-supplied `featureFlags` override on top. This is where `beforeAllFunctionSwap` (in `specs/swap/swap.setup.ts`) enables `ptxSwapLiveAppMobile` and `llmAnalyticsOptInPrompt`.

Each retry loop starts with a call to `checkTestFailed()`:

```typescript
function checkTestFailed(): void {
  if (globalThis.IS_FAILED) {
    throw new Error("Test failed - aborting initialization to prevent orphaned Speculos instances");
  }
}
```

`IS_FAILED` is flipped by `jest.environment.ts`'s `handleTestEvent` on hook or test failure. Without this guard, a fail-fast in the middle of initialisation could leave Speculos containers running forever; the guard forces initialisation to abort cleanly and propagate the failure.

### 5.7.15 How a Spec Actually Boots — end-to-end trace

Here is the sequence when you run `pnpm test:ios` from `e2e/mobile/`:

```
pnpm test:ios
 │
 └─ pnpm detox test --configuration ios.sim.release
     │
     ├─ Reads detox.config.js
     │   • Picks app: ios.release, device: simulator
     │   • testRunner.$0: jest  -->  spawns Jest with --config jest.config.js
     │
     ├─ Jest starts
     │   │
     │   ├─ [1] Runs globalSetup: jest.globalSetup.ts (once)
     │   │       • tsconfig-paths.register(…)
     │   │       • Verify apps/ledger-live-mobile/.env.mock exists
     │   │       • setupSpeculosCleanupHandlers()
     │   │       • cleanupPreviousNanoAppJsonFile()
     │   │       • await detox/runners/jest.globalSetup()
     │   │       • Enable video if on last retry
     │   │
     │   ├─ Spawns N workers (N = 3 in CI, 1 locally)
     │   │
     │   └─ For each worker:
     │       │
     │       ├─ [2] Constructs TestEnvironment (jest.environment.ts)
     │       │       • If workerId > 1: rewrite detoxConfig.device
     │       │       • super.setup() boots Detox for this worker
     │       │         (installs app, launches simulator, opens WS, etc.)
     │       │       • setupEnvironment()
     │       │       • new Application()  ← the `app` global
     │       │       • Install globals (enums, helpers, cliCommandsUtils)
     │       │       • Install handleTestEvent for failure capture
     │       │
     │       └─ For each spec file matching specs/**/*.spec.ts:
     │           │
     │           ├─ [3] Runs setupFilesAfterEnv: setup.ts (per file)
     │           │       • const broadcastOriginalValue = getEnv("DISABLE_TRANSACTION_BROADCAST")
     │           │       • setupEnvironment()
     │           │       • Registers beforeAll():
     │           │           launchApp()  ← finds free port, inits bridge, device.launchApp({ wsPort })
     │           │           device.reverseTcpPort(8081) + port + 52619
     │           │           setAllureDescription()
     │           │       • Registers afterAll():
     │           │           restore broadcast, closeBridge, app.common.removeSpeculos()
     │           │
     │           ├─ [4] Spec body runs — typically:
     │           │       describe("...", () => {
     │           │         beforeAll(async () => {
     │           │           await app.init({ userdata, speculosApp, cliCommands, featureFlags })
     │           │              ↓
     │           │              @Step("Account initialization")
     │           │              ├─ copy userdata → temp-userdata-<uuid>.json
     │           │              └─ InitializationManager.initialize(...)
     │           │                  ├─ group commandsByApp
     │           │                  ├─ launchSpeculosDevices in parallel
     │           │                  ├─ executeCliCommandsOnApp (retry)
     │           │                  ├─ setupMainSpeculosApp (retry)
     │           │                  ├─ executeCliCommands (retry + full-run recovery)
     │           │                  ├─ loadConfig(userdataSpeculos) over bridge
     │           │                  └─ setFeatureFlags(defaultFlags ⊕ options.featureFlags)
     │           │         });
     │           │         it("…", async () => {
     │           │           await app.portfolio.waitForPortfolioPageToLoad()
     │           │           await app.swap.openViaDeeplink()       ← each @Step becomes an Allure step
     │           │           await app.swap.verifyAmountsAndAcceptSwap(swap, amount)
     │           │           // ...
     │           │         });
     │           │       });
     │           │
     │           ├─ [5] On failure: handleTestEvent fires
     │           │       • IS_FAILED = true
     │           │       • takeSpeculosScreenshot()
     │           │       • takeAppScreenshot("Test Failure")
     │           │       • attachTestExecutionConsoleToAllure()
     │           │       • attachSpeculosStartupErrorToAllure()
     │           │       • getLogs() over the bridge → attachFailureLogsToAllure()
     │           │       • captureNativeViewHierarchy() → Allure XML attachment
     │           │
     │           └─ [6] afterAll (from setup.ts) runs
     │
     ├─ [7] Runs globalTeardown: jest.globalTeardown.ts (once)
     │       • If CI and SHARD_INDEX == 1: relaunch app, load userdata, write environment.properties
     │       • globalTeardown() (detox, 60 s timeout)
     │       • cleanupUserdata() (glob temp-userdata-*.json)
     │       • forceGarbageCollection()
     │
     └─ Jest writes allure-results under ./artifacts
         • pnpm allure picks them up on demand
```

Read this once end-to-end. Every time you see a weird behaviour at runtime — "why did my Speculos container leak?" (phase 1 handler), "why are there two `beforeAll` hooks firing before my spec runs?" (phases 3 and 4), "why does `app` exist even though I never imported anything?" (phase 2) — the answer sits in one of these numbered steps.

### 5.7.16 Cross-Reference with Part 4 Chapter 4.6

If you have read Part 4 Chapter 4.6 (Desktop Codebase Deep Dive), many shapes here will feel familiar. The two suites converge where the domain matches and diverge where the platform forces the hand:

| Concern | Desktop (`e2e/desktop/`) | Mobile (`e2e/mobile/`) |
|---|---|---|
| Workspace location | Top-level peer workspace | Top-level peer workspace |
| Runner | Playwright Test | Detox on Jest |
| Entry config | `playwright.config.ts` | `detox.config.js` + `jest.config.js` |
| Per-run bootstrap | Playwright `globalSetup` | `jest.globalSetup.ts` + `detox/runners/jest` |
| Per-worker env | `TestFixtures` | `jest.environment.ts` (extends `DetoxEnvironment`) |
| POM hub | `Application` class injected via fixture | `Application` class instantiated per worker, exposed as `app` global |
| POM instantiation | Constructed on fixture access | `lazyInit<T>()` thunks inside the class |
| POM decorator | `@Step` from `allure-playwright` | `@Step` from `jest-allure2-reporter/api` |
| Globals | None — everything passed as fixture parameters | Many — `tapById`, `app`, enums, etc. on `globalThis` |
| Device mocking | Hook directly into the Electron main process (no bridge) | WebSocket bridge (`bridge/server.ts`) |
| Userdata | `userdata/*.json` replayed via app bridge | `userdata/*.json` replayed via bridge `loadConfig` |
| Shared enums | `@ledgerhq/live-common/e2e/enum/*` | Same package, same path alias |
| Spec shape | Imperative `test("…", async ({ page, app }) => …)` | Thin config → `runXxxTest(...)` driver that calls `describe`/`it` itself |
| Driver pattern | Occasional (e.g. `addAccount.shared.ts`) | Dominant (one driver per bucket: `swap.ts`, `swap.other.ts`, `send.ts`, `addAccount.ts`, ...) |
| CI workflow | `test-desktop-e2e-reusable.yml` | `test-mobile-e2e-reusable.yml` (peer) |

The most consequential difference is the bridge. Desktop can poke at the Electron main process in-process; mobile cannot — the test runs in Node on your host, but the app runs on a device/simulator with its own JS runtime, on the other side of the OS sandbox. The WS bridge is how the test process says "import these accounts" or "enable this flag" without rebuilding the app. Every architectural decision downstream from that — why `app.init()` is so complex, why there is a separate `speculos.page.ts`, why `userdata/` is shipped as Redux-shaped JSON — follows from the bridge being the only control plane.

<div class="chapter-outro">
You now have a map of <code>e2e/mobile/</code>. Every <code>.ts</code> file has a purpose, every subdirectory has a role, every global has a declaration and an installer. The next chapter (4.8) turns this map into muscle memory: every flag on every command, every environment file toggle, every way a mobile test can go wrong — and exactly how to recover.
</div>

<div class="quiz-container" data-pass-threshold="80">
<h3 class="quiz-title">Chapter 5.7 Quiz</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> Where does the canonical mobile E2E suite live in the monorepo, and where do the native build artifacts end up?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Tests and artifacts both live under <code>apps/ledger-live-mobile/e2e/</code></button>
<button class="quiz-choice" data-value="B">B) Tests live under <code>e2e/mobile/</code> (peer workspace); native binaries are built under <code>apps/ledger-live-mobile/ios/build/</code> and <code>apps/ledger-live-mobile/android/app/build/</code>, and <code>detox.config.js</code> reaches across with <code>path.resolve(__dirname, "../..")</code></button>
<button class="quiz-choice" data-value="C">C) Tests live under <code>e2e/mobile/</code> and the binaries are also produced under <code>e2e/mobile/artifacts/</code></button>
<button class="quiz-choice" data-value="D">D) Both tests and binaries live under <code>libs/ledger-live-common/src/e2e/</code></button>
</div>
<p class="quiz-explanation">The workspace is a top-level peer (like <code>e2e/desktop/</code>), but the binaries still belong to the app project because <code>xcodebuild</code> and <code>gradlew</code> run from the app directory. The detox config bridges the gap with a relative <code>../..</code> jump.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> How many POMs does the <code>Application</code> class in <code>page/index.ts</code> expose, and what mechanism delays their construction?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) 20 POMs, constructed eagerly in the constructor</button>
<button class="quiz-choice" data-value="B">B) 32 POMs, all constructed eagerly in the constructor</button>
<button class="quiz-choice" data-value="C">C) 32 POMs, each wrapped in a <code>lazyInit&lt;T&gt;()</code> thunk that constructs the instance on first getter access and caches it</button>
<button class="quiz-choice" data-value="D">D) A variable number of POMs loaded via dynamic <code>import()</code> based on the current spec's tags</button>
</div>
<p class="quiz-explanation">32 POMs total (including <code>swap</code>, <code>swapLiveApp</code>, <code>earnV2Dashboard</code>, <code>modularDrawer</code>). Each getter calls a thunk produced by <code>lazyInit</code>, so constructing <code>new Application()</code> is cheap and instances are created on demand and cached.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> A spec calls <code>await app.init({ userdata: "1AccountBTC1AccountETHReadOnlyFalse", speculosApp: AppInfos.EXCHANGE, cliCommandsOnApp: [...] })</code>. What happens between the call and the first <code>it()</code> body?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Jest reinstalls the app from scratch, then the bridge handshakes</button>
<button class="quiz-choice" data-value="B">B) The userdata JSON is written to the device filesystem</button>
<button class="quiz-choice" data-value="C">C) Detox directly calls the Exchange app over USB</button>
<button class="quiz-choice" data-value="D">D) <code>Application.init</code> copies the userdata to a temp file, then <code>InitializationManager.initialize</code> launches all required Speculos devices in parallel, runs per-app then global CLI commands with retry, sets up the main Speculos device, sends <code>loadConfig</code> over the bridge, and applies default + caller-supplied feature flags</button>
</div>
<p class="quiz-explanation">The <code>@Step("Account initialization")</code> on <code>Application.init</code> shows up as the first Allure step. Everything described in D happens inside <code>InitializationManager.initialize</code> in <code>utils/initUtil.ts</code>.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> Why does <code>jest.environment.ts</code> read <code>JEST_WORKER_ID</code> and selectively rewrite <code>detoxConfig.device</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because <code>jest.config.js</code> sets <code>maxWorkers: 3</code> in CI, and the three <code>simulator</code>/<code>simulator2</code>/<code>simulator3</code> (or <code>emulator*</code>) entries in <code>detox.config.js</code> exist so each parallel worker can own a distinct device</button>
<button class="quiz-choice" data-value="B">B) Because Detox requires each test to pick its device at runtime</button>
<button class="quiz-choice" data-value="C">C) Because Jest does not support parallel execution natively</button>
<button class="quiz-choice" data-value="D">D) It is a dead-code path; <code>maxWorkers</code> is always 1</button>
</div>
<p class="quiz-explanation">Workers 2 and 3 would fight over the same simulator UDID otherwise. The rewrite happens <em>before</em> <code>super.setup()</code> so Detox's own bootstrap sees the right device.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> What is the relationship between <code>specs/swap/swapBTC_NATIVE_SEGWIT_LTC.spec.ts</code>, <code>specs/swap/swap.ts</code>, and <code>specs/swap/swap.setup.ts</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Only the <code>.spec.ts</code> file runs; the others are dead code kept for historical reasons</button>
<button class="quiz-choice" data-value="B">B) The <code>.spec.ts</code> file is a thin config bundle that calls <code>runSwapTest</code> exported by <code>swap.ts</code> (the driver). <code>swap.ts</code> calls <code>beforeAllFunctionSwap</code> from <code>swap.setup.ts</code> for shared initialisation. Only files matching <code>specs/**/*.spec.ts</code> are executed by Jest; <code>swap.ts</code> and <code>swap.setup.ts</code> are plain modules imported by specs</button>
<button class="quiz-choice" data-value="C">C) <code>swap.ts</code> is the spec; <code>swap.setup.ts</code> configures Detox</button>
<button class="quiz-choice" data-value="D">D) All three files must be listed in <code>jest.config.js</code>'s <code>testMatch</code> to run</button>
</div>
<p class="quiz-explanation">This is the thin-spec + shared-driver pattern. A new currency pair is a ~20-line <code>.spec.ts</code> file; all the test logic lives in the driver. Jest's <code>testMatch</code> glob (<code>specs/**/*.spec.ts</code>) keeps the drivers out of the run list.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q6.</strong> Which structural difference with the desktop workspace most directly explains the existence of <code>bridge/server.ts</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Mobile uses Jest, desktop uses Playwright Test</button>
<button class="quiz-choice" data-value="B">B) Mobile writes POMs in TypeScript, desktop in JavaScript</button>
<button class="quiz-choice" data-value="C">C) Mobile has more POMs (32 vs fewer on desktop)</button>
<button class="quiz-choice" data-value="D">D) On desktop, the test process can poke at the Electron main process in-process (shared memory, one Node runtime). On mobile, the test process runs on the host in Node, but the app runs on a simulator/emulator with its own JS runtime and an OS sandbox between them. A WebSocket bridge is the only control plane for importing accounts, overriding feature flags, and dispatching programmatic navigation</button>
</div>
<p class="quiz-explanation">Every architectural decision downstream (separate <code>speculos.page.ts</code>, Redux-shaped <code>userdata/*.json</code>, the request/response <code>pendingCallbacks</code> map, the <code>ACK</code> protocol) follows from the bridge being the only way across the runtime boundary.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Running & Debugging Mobile E2E Tests

<div class="chapter-intro">
Chapters 34 and 35 taught you how Detox finds elements, how the bridge mocks a device, and how a spec file is structured. Chapter 5.7 grounded you in the actual repository layout. This chapter turns that knowledge into muscle memory: every flag on every command, every environment file toggle, every way a mobile test can go wrong — and exactly how to recover. By the end you should be able to boot a simulator, reproduce a failing scheduled CI run on your laptop, attach a debugger, and read the artifacts like a native speaker.
</div>

### 5.8.1 Prerequisites Recap

Before running a single `detox test`, confirm the toolchain from Chapter 5.4 is in place. Unlike desktop Playwright — where launching the Electron binary is essentially self-contained — mobile E2E depends on **three** external processes that must all be alive simultaneously: the simulator/emulator, the Metro bundler (for debug builds), and the Detox test-host process.

**iOS checklist (on macOS only):**
- Xcode 15+ installed (`xcode-select -p` should point to `/Applications/Xcode.app/Contents/Developer`)
- Command-line tools: `xcode-select --install`
- A booted simulator matching the Detox config: typically `iPhone 15` on iOS 17.x. List them with `xcrun simctl list devices`.
- `applesimutils` installed (`brew tap wix/brew && brew install applesimutils`) — Detox uses this to set permissions.
- CocoaPods synced (Chapter 5.4 `pod install` in `apps/ledger-live-mobile/ios`).

**Android checklist (macOS or Linux):**
- Android Studio installed, or at least the `cmdline-tools` package via `sdkmanager`.
- An AVD created matching the config name — typically `Android_Emulator` API 35 `pixel_7_pro`.
- `ANDROID_HOME` exported and `$ANDROID_HOME/platform-tools` on `PATH` (so `adb` resolves).
- Emulator booted and visible: `adb devices` should list exactly one device in `device` state.

**Universal checklist:**
- `pnpm` installed and `pnpm i` run at the monorepo root.
- `pnpm e2e:mobile i` (Chapter 5.7 explained why the `e2e/mobile/` workspace pulls its own Detox + Jest deps).
- Speculos coin apps accessible — either a local `COINAPPS` directory or the `selectMockDevice` bridge helper (see 37.10).

> **Tip:** Run `pnpm e2e:mobile exec detox --version` from the repo root before anything else (or `cd e2e/mobile && pnpm exec detox --version`). If that command fails, nothing else will work. A non-zero exit here almost always means `pnpm i` was never run inside the `e2e/mobile/` workspace.

A running simulator or emulator is **not optional**. Detox does not spin one up for you the way Playwright spins up Chromium. If you forget this step you will see `DetoxRuntimeError: Failed to run application on the device` within the first ten seconds.

### 5.8.2 Build Commands in Depth

Detox separates **build** from **test**. The build step compiles a binary wired for Detox (`detox.frameworkBundle` injected, testability enabled). The test step installs and drives that binary. You almost never want to rebuild on every run — it takes 5–15 minutes for iOS, 3–8 minutes for Android.

#### iOS debug build

```bash
# From the repo root (recommended):
pnpm e2e:mobile run build:ios:debug

# Or, from inside the E2E workspace:
cd e2e/mobile && pnpm build:ios:debug

# Or, directly via Nx (what CI ultimately runs):
nx run live-mobile:e2e:build -- --configuration ios.sim.debug
```

**What each piece does:**
- `pnpm e2e:mobile` — root-level alias defined in the monorepo `package.json` (`"e2e:mobile": "pnpm --filter ledger-live-mobile-e2e-tests"`). It targets the `e2e/mobile/` workspace.
- `build:ios:debug` — script in `e2e/mobile/package.json`. It delegates to `pnpm --filter live-mobile run e2e:build --configuration ios.sim.debug`, which ultimately runs `detox build -c ios.sim.debug` with `e2e/mobile/detox.config.js` as the config.
- `ios.sim.debug` — a configuration defined in `e2e/mobile/detox.config.js`. The name encodes platform (`ios`), target (`sim` = simulator), and variant (`debug`).

A **debug** build is compiled with Metro serving JS at runtime. Reloads are fast (edit code → `r` in Metro → hot reload), but you pay the cost of a live Metro server on port 8081 during the test run.

#### iOS release build

```bash
pnpm e2e:mobile run build:ios
# equivalent to: nx run live-mobile:e2e:build -- --configuration ios.sim.release
```

A **release** build bundles JS into `main.jsbundle` at build time, inlines it into the `.app`, and runs Hermes bytecode. No Metro needed. This is what CI uses because it is faster per-test (no Metro round-trips) and more representative of production.

Use `release` when:
- You are debugging a flaky test that you suspect is timing-related.
- You want to reproduce a CI failure exactly.
- You have finished iterating and want a final validation pass.

Use `debug` when:
- You are authoring a new test and want to tweak the spec and re-run without rebuilding.
- You want to use React DevTools or the Metro logs to inspect app state.

#### Android builds

```bash
pnpm e2e:mobile run build:android:debug
pnpm e2e:mobile run build:android         # release
```

Same split: `debug` uses Metro, `release` uses a bundled APK. The emulator config expects an AVD named exactly `Android_Emulator` — look at `e2e/mobile/detox.config.js` for the `avdName` field.

#### The prerelease variant

```bash
nx run live-mobile:e2e:build -- --configuration ios.sim.prerelease
nx run live-mobile:e2e:build -- --configuration android.emu.prerelease
```

`prerelease` is a third variant used for pre-release validation against production services. It compiles the app with `ENVFILE=apps/ledger-live-mobile/.env.mock.prerelease` (see below), which points Firebase Remote Config at the **production** project instead of staging. You rarely need this locally; CI uses it when `production_firebase: true` is set on the workflow (Chapter 7.3.12).

#### The `ENVFILE` toggle

Every React Native build reads an `.env` file at compile time via `react-native-config`. Detox variants pick different env files:

| Variant | ENVFILE | Firebase | Intended use |
|---------|---------|----------|--------------|
| `*.debug`, `*.release` | `.env.mock` | Staging project | Normal dev & scheduled CI |
| `*.prerelease` | `.env.mock.prerelease` | Production project | Release gating, dogfood validation |

**What changes downstream when you flip the file:**
- `FIREBASE_*` keys (API key, project ID, app ID) swap to production.
- Remote Config feature flags are read from the production namespace — a flag disabled in staging may be enabled in prod or vice versa.
- Analytics endpoints change (Segment write keys, etc.), though the E2E harness typically stubs these.
- The bundle ID of the installed `.app` / `.apk` changes (e.g., `com.ledger.live.dev.detox` vs `com.ledger.live.detox.prerelease`). This matters because the test can only drive the binary whose bundle ID matches the Detox config.

> **Warning:** If you rebuild with a different `ENVFILE` without uninstalling the previous binary, the simulator may still launch the old one — Detox resolves apps by bundle ID. Run `xcrun simctl uninstall booted <bundleId>` between variant switches, or simply `device.uninstallApp()` at the start of a spec.

#### Build caching

The native build artifact is large (several hundred MB) and its content is deterministic per the combination of:
- The `ios/` or `android/` source tree (hashed).
- The selected variant.
- The `ENVFILE` content.

CI keys its S3 cache off `hashFiles('apps/ledger-live-mobile/ios')` — locally, you get the same effect by not touching native code between runs. If you only edit TypeScript specs, the previously-built `.app` or `.apk` is reused and you just re-run tests.

### 5.8.3 Test Execution

Once a build exists, test execution is invoked from inside the `e2e/mobile/` workspace (or via `pnpm e2e:mobile run ...` from the repo root):

```bash
cd e2e/mobile
pnpm test:ios:debug
pnpm test:android:debug
# equivalently from the repo root:
pnpm e2e:mobile run test:ios:debug
```

The script expands to `pnpm detox test --configuration <config> [...]` with `detox` resolved against `e2e/mobile/` (so the workspace's `detox.config.js` is picked up). Anything after the script passes through to Detox and thence to Jest.

#### Filtering which tests run

Jest supplies the selection primitives; Detox just forwards them. The examples below assume you are inside `e2e/mobile/` — prepend `pnpm e2e:mobile run` if running from the repo root.

**By test name (`-t` or `--testNamePattern`):**

```bash
pnpm test:ios:debug -- -t "Portfolio"
pnpm test:ios:debug -- --testNamePattern "B2CQA-604"
```

This matches anything in the concatenation of `describe` + `it` titles. Since every `it()` in the suite embeds a `$TmsLink("B2CQA-604")` tag in its title (Chapter 5.7.5), `-t "B2CQA-604"` is the single-ticket filter you will use most often.

**By file path (`--testPathPattern`):**

```bash
pnpm test:ios:debug -- --testPathPattern specs/portfolio
pnpm test:ios:debug -- --testPathPattern "addAccount|send"
```

This is a regex against the absolute path of each spec file. It runs before the test files are loaded, so it is cheaper than `-t` when you know you only want a subset.

**Combining them:**

```bash
pnpm test:ios:debug -- --testPathPattern specs/swap/otherTestCases -t "swapExportHistoryOperations"
```

Runs only spec files under `specs/swap/otherTestCases/`, and within those only `it()` names matching `/swapExportHistoryOperations/`.

**Excluding paths (`--testPathIgnorePatterns`):**

```bash
pnpm test:ios:debug -- --testPathIgnorePatterns specs/experimental
```

Handy when a directory is known-broken and you want to skip it wholesale without editing `jest.config.js`.

#### Worker and lifecycle flags

```bash
pnpm test:ios:debug -- --maxWorkers=1 --cleanup --reuse
```

- `--maxWorkers=1` — run tests serially in a single process. This is actually the **default** (and only supported) configuration for mobile E2E; the codebase sets `maxWorkers: 1` in `jest.config.js` because Detox can only drive one device per worker and the bridge port would collide with parallel workers. CI achieves parallelism across **shards**, not workers (Chapter 8.3).
- `--cleanup` — uninstall the app at the end of the run. Without this, the app stays on the simulator and subsequent runs start faster (but may see stale Redux persist state).
- `--reuse` — skip the install step and reuse the binary already on the device. Fastest iteration mode when nothing has been rebuilt.
- `--forceExit` — kill Jest's process even if there are unhandled handles. Used in CI to avoid timing out on lingering WebSocket connections.
- `--retries 2` — retry a failed `it()` up to 2 times. The CI orchestrator sets this; locally you usually want `0` so you see the true failure.
- `--record-logs failing`, `--take-screenshots failing` — Detox artifact policies. Keep logs and screenshots only for failing tests (see 37.6).
- `--headless` — boot the simulator/emulator without a visible window. CI uses this; locally you probably want the window so you can watch.
- `--loglevel warn` — Detox log verbosity. Values: `trace`, `debug`, `info`, `warn`, `error`. For a frustrating failure, bump to `trace` and grep the output.

#### Putting it together

A realistic local iteration loop on a single ticket:

```bash
# One-time: build (or rebuild if native code changed)
pnpm e2e:mobile run build:ios:debug

# Many times: run just the failing test with verbose logging and no retries
cd e2e/mobile
pnpm test:ios:debug \
  -- -t "B2CQA-604" \
  --reuse \
  --retries 0 \
  --loglevel trace \
  --record-logs all \
  --take-screenshots all
```

### 5.8.4 The `scripts/e2e-ci.mjs` Orchestrator

Opening the workflow YAML you saw in Chapter 5.7, the actual command every shard runs is not `detox test` directly — it is a `zx` script at `apps/ledger-live-mobile/scripts/e2e-ci.mjs` (the script still lives under the mobile app, not in the E2E workspace, because it orchestrates native-build caching that is app-local). Read it verbatim; it is small (~210 lines) and explains itself:

```js
#!/usr/bin/env zx
// apps/ledger-live-mobile/scripts/e2e-ci.mjs (excerpt)
let platform, test, build, bundle, bundleSize;
let testType = "mock";
let cache = true;
let shard = "";
let target = "release";
let filter = "";
let outputFile = "";
```

**Flags it accepts:**

| Flag | Meaning |
|------|---------|
| `-p <ios\|android>` | Which platform to target. Required. |
| `-t` | Run the `test` step. |
| `-b` | Run the `build` step (native compile). |
| `--bundle` | Run the JS bundling step (produces `main.jsbundle`). |
| `--bundle-size` | Produce a minified bundle for size reporting only (not used for tests). |
| `--cache / --no-cache` | On iOS, whether to copy the JS bundle into the cached `Release-iphonesimulator` directory so subsequent cache hits are valid. |
| `--e2e` | Switch `testType` from `mock` (default) to `e2e` — flips between the mock-bridge test script and the real-Speculos test script. |
| `--shard N/M` | Forwarded to Detox/Jest as `--shard N/M`. |
| `--production` | Switch the Detox target from `release` to `prerelease`. |
| `--filter <pattern>` | Not forwarded directly; present so the orchestrator can strip it from trailing args. |
| `-o / --outputFile <path>` | Appends `--json --outputFile=<path>` to the Jest command for timing-data collection. |

**How it invokes Detox.** The production-test call shells out to the `e2e/mobile/` workspace scripts — conceptually:

```js
await $`pnpm e2e:mobile run ${testType === "e2e" ? "test" : "mock:test"}:ios:${target}\
    --loglevel warn \
    --record-logs failing \
    --take-screenshots failing \
    --forceExit \
    --headless \
    --retries 2 \
    --cleanup \
    ${filteredArgs}`.nothrow();
```

> **Verify:** the exact shape of the command and the `mock:test` / `test:ios` script split can drift between refactors — when you need the literal command line a shard runs, open `apps/ledger-live-mobile/scripts/e2e-ci.mjs` on the branch you care about.

`filteredArgs` is every trailing positional argument that isn't one of the recognized flags — in practice, the list of spec file paths computed by the sharding step (Chapter 8.3.5).

**The mock/e2e split.** The `mock` path uses the bridge mock device (no Speculos required — see Chapter 5.6 and 37.10). The `e2e` path uses a real Speculos container. CI almost always uses `--e2e` because scheduled runs exercise real device flows; `mock` is kept as an option for speed-critical smoke passes.

**The bundle-then-build dance.** Notice `bundle_ios_with_cache` copies `main.jsbundle` into the already-built `Release-iphonesimulator/` directory. This lets CI cache the native build (which is expensive and rarely changes) and only re-bundle JS (which is cheap and changes every commit).

**Metro and the bridge.** For `release`/`prerelease` builds, there is no Metro — the JS is already inside the `.app`/`.apk`. The Detox bridge still runs (that is the WebSocket that ships mock device commands from spec to app), on port 8099 by default. For `debug` builds, Metro runs on 8081 and the bridge on 8099 — two distinct processes with two distinct ports.

### 5.8.5 Debugging Cookbook

This section is a field guide. Each entry is a symptom, a diagnosis, and the exact command or code change that resolves it.

#### Symptom: "App is busy"

```
DetoxRuntimeError: Timed out while waiting for the app to become idle
```

**Diagnosis:** Detox synchronizes with the app — it won't execute the next action until the JS thread, animations, pending network requests, and timers have all settled. If your app has a never-closing WebSocket or a repeating poller, Detox waits forever.

**Fix:** Blacklist the URL or disable synchronization locally.

```typescript
// Globally blacklist a URL from the sync check
await device.setURLBlacklist([".*/polling/.*", ".*/sse/.*"]);

// Or disable synchronization for a block of actions
await device.disableSynchronization();
try {
  await Tap(button).click();
  await AnimationThatTakesForever.waitFor({ timeout: 5000 });
} finally {
  await device.enableSynchronization();
}
```

See Detox's [synchronization troubleshooting guide](https://wix.github.io/Detox/docs/troubleshooting/synchronization) — the full list of strategies includes disabling specific modules (`device.setSyncSettings`) and whitelisting vs blacklisting.

#### Symptom: "Element not found"

```
Error: Test Failed: No elements found for “MATCHER(id == 'send-button')”
```

**Diagnosis:** The `testID` you expect isn't on the screen. Either (a) the screen is not rendered yet, (b) the element uses a different `testID`, or (c) it's inside a `FlatList` that hasn't virtualized it into view yet.

**Fix:** Dump the view hierarchy at the moment of failure.

```typescript
// In the failing test, just before the action:
const xml = await device.generateViewHierarchyXml();
console.log(xml);
// Or to a file:
await device.generateViewHierarchyXml({ path: "/tmp/hierarchy.xml" });
```

The XML contains every `testID`, `accessibilityLabel`, and `text` value currently on screen. Grep it for a partial match. If the element is genuinely missing, the preceding action must not have completed — walk back through the spec.

If the element is offscreen in a list:

```typescript
await waitFor(element(by.id("item-42")))
  .toBeVisible()
  .whileElement(by.id("scrollable-list"))
  .scroll(200, "down");
```

#### Symptom: Metro port collision

```
error Metro is already running on port 8081
```

**Diagnosis:** A previous run left Metro alive, or another project is using 8081.

**Fix:**

```bash
# Find the process
lsof -i :8081
# Kill it
kill -9 <pid>
# Or kill every node process listening on 8081
lsof -t -i :8081 | xargs kill -9
```

For Android specifically, even if Metro is running on the host, the emulator must be told to forward its own `localhost:8081` to the host's. This happens automatically when Detox installs a debug APK, but if you sideload manually:

```bash
adb reverse tcp:8081 tcp:8081
```

See Android's [adb forwardports docs](https://developer.android.com/tools/adb#forwardports) — `adb reverse` is the opposite direction (device → host) and is what React Native's Metro bridge depends on.

#### Symptom: Bridge WebSocket not connecting

```
DetoxRuntimeError: The app has not started communicating with the test runner
```

**Diagnosis:** The Detox bridge runs on port 8099. Either the port is taken, the app build didn't include the Detox framework, or the device cannot reach the host.

**Fix checklist:**
1. `lsof -i :8099` — if something else is there, kill it.
2. Rebuild with `-c <config>` matching the one you're testing against. A common mistake is building `ios.sim.debug` and trying to test with `ios.sim.release` — the binary on the simulator is the old one.
3. Check `e2e/mobile/detox.config.js` and the app's `AppDelegate.mm` — confirm `wsPort: 8099` matches on both sides. The bridge host and port are injected at launch via `launchArgs`. The canonical bridge server implementation lives at `e2e/mobile/bridge/server.ts`.
4. On Android, verify `adb reverse tcp:8099 tcp:8099` is active.

#### Symptom: Timeouts

```
thrown: "Exceeded timeout of 120000 ms for a test"
```

**Diagnosis:** Jest's per-test timeout is exhausted. Mobile tests are slow — a default 30s is far too short.

**Fix:** `e2e/mobile/jest.config.js` sets the default.

```js
// Typical config
module.exports = {
  testTimeout: 300000,    // 5 minutes per it()
  maxWorkers: 1,          // Detox constraint
  resetModules: true,
  setupFilesAfterEnv: ["<rootDir>/setup.ts"],
};
```

For a specific slow action, use `waitFor(...).withTimeout()`:

```typescript
await waitFor(element(by.id("sync-complete-banner")))
  .toBeVisible()
  .withTimeout(180000); // 3 minutes
```

#### Symptom: Flaky test ordering

Test A passes alone but fails when run after Test B.

**Diagnosis:** State leaking between tests — usually Redux state persisted across `it()` blocks.

**Fix:** The harness already sets `resetModules: true`. What you likely need is `device.reloadReactNative()` in a `beforeEach` — it reboots the JS runtime without reinstalling the native side. If that's not enough, `device.launchApp({ delete: true })` reinstalls fresh.

Do not attempt to run tests in parallel by bumping `maxWorkers`. The bridge port alone makes that impossible; the codebase pins `maxWorkers: 1` on purpose.

#### Symptom: `NoDevice` from a CLI helper, but Speculos is running

`liveDataWithAddressCommand`, `tokenAllowance`, or any other CLI helper rejects with `NoDevice` / `transport not found`, even though Speculos is up, `SPECULOS_API_PORT` is set in the env, and the CLI binary works when you run it by hand.

**Diagnosis:** Dual-package hazard. Your spec is `import`-ing the helper from `@ledgerhq/live-common` instead of using the global the harness installed. You now have two module instances — the harness registered the Speculos transport against one, and your CLI call is reading the registry of the other (which is empty), so the CLI falls through to HID and finds nothing. Full mechanism in §5.7 (the dual-package hazard callout).

**Fix:** Delete the `import` line. Call the helper as a bare global identifier (it is declared in `types/global.d.ts`). Confirm with `console.log((globalThis as any).cliCommandsUtils === require("@ledgerhq/live-common/...").cliCommandsUtils)` — if that prints `false`, you have the bug; deleting the import will make it `true` and the test will pass.

### 5.8.6 Screenshots, Videos, View Hierarchy Dumps

Detox's `artifacts` section in `e2e/mobile/detox.config.js` controls automated captures.

```js
artifacts: {
  rootDir: "./artifacts",
  plugins: {
    log: "failing",
    screenshot: {
      enabled: true,
      shouldTakeAutomaticSnapshots: true,
      keepOnlyFailedTestsArtifacts: true,
      takeWhen: { testStart: true, testDone: true, testFailure: true },
    },
    video: "failing",
    instruments: "failing",
    uiHierarchy: "enabled",
  },
}
```

**What lands where:**
- Screenshots: `./artifacts/<run-id>/<spec-name>/<test-name>/<step>.png`
- Videos: `./artifacts/<run-id>/<spec-name>/<test-name>/test.mp4` (iOS only; Android uses screen recording when the emulator supports it)
- View hierarchy: `.viewhierarchy` files on iOS, XML dumps on Android
- Logs: `device.log`, `app.log`, `jest.log`

The `failing` policy means artifacts are only retained for failed tests — passing tests' captures are deleted at the end of the run to save disk.

**Allure pickup.** The Jest Allure reporter (configured in `jest.config.js`) writes one JSON per `it()` to `./allure-results/`. When the Detox run emits an artifact for a failing test, the reporter attaches the file path as an Allure attachment so the eventual HTML report embeds the screenshot and video inline.

### 5.8.7 Allure Output Locally

After a run, the raw Allure JSON and attachments sit under:

```
e2e/mobile/artifacts/
```

Each `it()` produces one `<uuid>-result.json` plus an `-attachment.json` file for each attached artifact.

The workspace ships an `allure` script that combines generate + open:

```bash
cd e2e/mobile
pnpm allure
# runs: allure generate ./artifacts --clean -o ./artifacts/allure-report
#   and: allure open ./artifacts/allure-report
```

You can also run the two halves separately — `pnpm allure:generate` and `pnpm allure:open`. They call `allure-commandline` from the workspace's own `node_modules`, so no system-wide install is required. If you prefer the live-reload experience of `allure serve`, install the CLI globally (`brew install allure`) and point it at the same folder:

```bash
allure serve e2e/mobile/artifacts
```

Cross-reference Part 3 Chapter 3.5 for the full Allure internals — the mobile setup reuses the same reporter conventions as desktop, so the mental model is identical. For the official specification, see the [Allure documentation](https://allurereport.org/docs/).

### 5.8.8 Logs on Failure

The test-harness setup (wired from `e2e/mobile/setup.ts` and `e2e/mobile/bridge/server.ts`) installs an `afterAll` hook that, on failure, pulls three diagnostic bundles and attaches them to Allure:

- **App logs** — the output of the React Native app's `console.log` and native logs, captured by Detox's `device.log` plugin.
- **Bridge logs** — the WebSocket traffic between the spec runner and the bridged app (the bridge WebSocket still runs on port 8099). Indispensable for "why did `selectMockDevice` fail".
- **Memory snapshots** — on iOS only, a compact memory graph dumped via `instruments` when the app crashes.

They land under `e2e/mobile/artifacts/<run-id>/`. Open the Allure report and click into the failing test — the attachments tab lists each one by name.

When a CI failure is reported in Slack, your first move should be: download the artifact zip, open `app.log`, then open `bridge.log`. Three out of four failures become obvious from those two files alone.

### 5.8.9 Running a Single `it()` by TmsLink

Every `it()` in the Ledger suite is tagged with a Jira ticket via `$TmsLink`:

```typescript
it(
  `@B2CQA-604 • Send BTC - valid address, broadcast disabled`,
  async () => {
    // ...
  },
);
```

To run only that one test (from inside `e2e/mobile/`):

```bash
pnpm test:ios:debug -- --testNamePattern "B2CQA-604"
# or shorthand:
pnpm test:ios:debug -- -t "B2CQA-604"
```

The `-t` pattern is a regex (case-insensitive by default for Jest), so `B2CQA-604` will match exactly one test unless there are duplicates — in which case `--testPathPattern` narrows it further:

```bash
pnpm test:ios:debug -- --testPathPattern swapExportHistoryOperations
```

**Performance tip:** combine with `--testPathPattern` to skip loading spec files that can't contain the test. Jest loads every matching file and then filters by name; if you know the ticket is under `specs/swap/otherTestCases/`, adding `--testPathPattern specs/swap/otherTestCases` saves 2–5 seconds of file loading.

### 5.8.10 Speculos on Mobile

Chapter 4.5 (desktop) and Chapter 5.6 (mobile) both introduced Speculos. Mobile E2E has two modes:

**Mode A: `selectMockDevice` (bridge mock — no Speculos needed).**

```typescript
import { selectMockDevice } from "@bridge/client";

beforeAll(async () => {
  await selectMockDevice({ nano: "nanoX" });
});
```

The bridge intercepts all DMK calls from the app and answers them with canned signed payloads. This is the fastest mode and is used for flows where you don't care about on-device confirmation screens (e.g., portfolio display, settings).

**Mode B: Real Speculos container.**

A Speculos container (the same Docker image from Part 5) runs on the host exposing HTTP/WS on a port, typically `52619`. The app's DMK transport must be redirected to that host/port.

- **iOS simulator** — shares the host's network, so `http://localhost:52619` works from inside the simulator without any port-forwarding dance.
- **Android emulator** — has its own network namespace. `localhost` from the emulator is the emulator itself, not the host. You must forward:

```typescript
await device.reverseTcpPort(52619);
// or from a shell: adb reverse tcp:52619 tcp:52619
```

`device.reverseTcpPort()` is Detox's wrapper around `adb reverse`. After this call, the emulator's `localhost:52619` points at the host's Speculos container.

**When to pick which:**
- Use **`selectMockDevice`** when you're testing app-side logic and any valid-looking response is sufficient. 80% of specs are this.
- Use **real Speculos** when the test asserts against on-device screens (confirming an address matches what the device shows), when you need signature randomness to match a specific seed, or when you're validating a transaction-signing flow end-to-end.

The Chapter 8.3 CI workflow has a `speculos_device` input to pick which device firmware Speculos emulates — `nanoX`, `stax`, `flex`, etc.

### 5.8.11 Chapter 5.8 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Running & Debugging Mobile E2E Tests</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> You change a single line in a spec file and want to re-run it against the simulator as fast as possible. Which command best minimizes iteration time?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>pnpm e2e:mobile run build:ios:debug && pnpm e2e:mobile run test:ios:debug</code></button>
<button class="quiz-choice" data-value="B">B) <code>cd e2e/mobile && pnpm test:ios:debug -- -t "B2CQA-604" --reuse --retries 0</code></button>
<button class="quiz-choice" data-value="C">C) <code>cd e2e/mobile && pnpm test:ios -- --maxWorkers=4</code></button>
<button class="quiz-choice" data-value="D">D) <code>nx run live-mobile:e2e:build -- --configuration ios.sim.prerelease</code></button>
</div>
<p class="quiz-explanation">When only TypeScript changes, the native binary does not need to be rebuilt. <code>--reuse</code> skips re-installing the app, <code>-t</code> runs just the target test, and <code>--retries 0</code> surfaces the true failure instead of silently retrying. Option C is invalid because <code>maxWorkers</code> is pinned to 1 by the bridge constraint.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> What is the difference between <code>.env.mock</code> and <code>.env.mock.prerelease</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>.env.mock.prerelease</code> uses Metro bundling while <code>.env.mock</code> uses Hermes</button>
<button class="quiz-choice" data-value="B">B) Only the bundle ID changes; Firebase keys are identical</button>
<button class="quiz-choice" data-value="C">C) <code>.env.mock</code> points Firebase at the staging project; <code>.env.mock.prerelease</code> points it at the production project — Remote Config flags, analytics endpoints, and bundle IDs all change</button>
<button class="quiz-choice" data-value="D">D) They are the same file under two names, kept for legacy reasons</button>
</div>
<p class="quiz-explanation">The ENVFILE toggle is the mechanism by which CI validates against production Firebase during release gating (<code>production_firebase: true</code> in the workflow). Everything keyed off <code>FIREBASE_*</code> and the bundle ID flips at compile time.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> A test fails with “App is busy” even though nothing is visibly animating. What is the most likely cause and fix?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A long-poll or SSE connection keeps the network idle-check from ever settling — blacklist the URL with <code>device.setURLBlacklist([...])</code></button>
<button class="quiz-choice" data-value="B">B) The simulator is out of memory — reboot it</button>
<button class="quiz-choice" data-value="C">C) The spec is using <code>getByText</code> — switch to <code>by.id</code></button>
<button class="quiz-choice" data-value="D">D) Metro is not running — start it with <code>pnpm mobile start</code></button>
</div>
<p class="quiz-explanation">Detox waits for the app to be idle before each action. Persistent network sockets never become idle. <code>setURLBlacklist</code> tells the synchronization engine to ignore traffic matching the given patterns.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> You need to drive the app against a real Speculos Nano X on an Android emulator. Which step is required but easy to forget?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Rebuild the app with <code>-c android.emu.prerelease</code></button>
<button class="quiz-choice" data-value="B">B) Set <code>maxWorkers: 2</code> so Speculos and Detox don't share a worker</button>
<button class="quiz-choice" data-value="C">C) Call <code>selectMockDevice({ nano: "nanoX" })</code> before the test</button>
<button class="quiz-choice" data-value="D">D) Reverse the Speculos port to the emulator with <code>device.reverseTcpPort(52619)</code> (or <code>adb reverse tcp:52619 tcp:52619</code>)</button>
</div>
<p class="quiz-explanation">Android emulators have their own network namespace — <code>localhost</code> is the emulator itself. <code>adb reverse</code> (wrapped by Detox's <code>device.reverseTcpPort</code>) forwards the emulator's <code>localhost:52619</code> to the host, where Speculos is listening. iOS simulators share the host network, so no forwarding is needed there. Option C is the <em>opposite</em> of using real Speculos — it is the mock-bridge mode.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> Where do Allure results from a local mobile run land, and how do you view them?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>apps/ledger-live-mobile/coverage/</code> — open <code>index.html</code> directly</button>
<button class="quiz-choice" data-value="B">B) <code>e2e/mobile/artifacts/</code> — run <code>cd e2e/mobile && pnpm allure</code> (or <code>allure serve e2e/mobile/artifacts</code>) to view</button>
<button class="quiz-choice" data-value="C">C) They are only produced on CI; local runs don't generate Allure data</button>
<button class="quiz-choice" data-value="D">D) In <code>~/.allure/</code>, indexed by run timestamp</button>
</div>
<p class="quiz-explanation">The Jest Allure reporter writes per-test JSON plus attachment metadata into <code>e2e/mobile/artifacts/</code>. The workspace's <code>pnpm allure</code> script runs <code>allure generate</code> then <code>allure open</code> against that folder; <code>allure serve</code> is an equivalent live-render alternative if you have the CLI installed globally.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> Why does <code>jest.config.js</code> pin <code>maxWorkers: 1</code> for mobile E2E, and what is the CI's parallelism strategy instead?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Jest does not support multi-worker runs — it is a framework limitation</button>
<button class="quiz-choice" data-value="B">B) Detox can use multiple workers, but TypeScript compilation is single-threaded</button>
<button class="quiz-choice" data-value="C">C) Each Detox worker needs its own device and bridge port; parallel workers would collide on port 8099 and on the single booted simulator. CI parallelizes across <em>shards</em> (separate GitHub runners), not workers</button>
<button class="quiz-choice" data-value="D">D) It is a historical artifact — the value could be raised but nobody has tested it</button>
</div>
<p class="quiz-explanation">Mobile E2E is constrained at the device level: one simulator per process, one bridge WebSocket per simulator. Horizontal scaling therefore happens at the runner level — each CI shard is its own runner with its own simulator/emulator(s) (Chapter 8.3).</p>
</div>

<div class="quiz-score"></div>
</div>

---


## Your Daily Mobile Workflow: From Ticket to PR

<div class="chapter-intro">
You have learned the React Native primitives, the Detox toolchain, the matcher DSL, the bridge protocol, the singleton page-object pattern, the sharding algorithm, the Allure pipeline, and the CI graph. This chapter stitches all of that into the <strong>workflow</strong> — the exact, repeatable sequence you will execute every single day as an E2E Mobile engineer at Ledger. It mirrors Chapter 4.7 (desktop) but tracks the mobile realities: a Jira QAA ticket, a B2CQA Xray test, a Detox build, a Metro bundle, an Allure upload, and an iOS/Android CI matrix. Follow it every time. The steps do not change just because the ticket seems small.
</div>

### 5.9.1 The Ticket Lifecycle at Ledger Live

Every piece of work you do on the mobile E2E suite starts in Jira and ends in an Xray traceability update. The lifecycle has **three ticket shapes** — learn to distinguish them on sight.

| Ticket type | Project | Example | Purpose |
|---|---|---|---|
| **QAA ticket** | `QAA` | `QAA-702` | A work item you pick up in your sprint — "automate X", "cover regression Y", "fix flake in Z". |
| **Xray test case** | `B2CQA` | `B2CQA-604` | A test scenario in Xray — reusable across platforms (LLD, LLM, LLC). |
| **Bug** | `LIVE` (tagged with `LLM` / `LLD` labels) | `LIVE-12345` | A product defect. Rarely your starting point, but you link it in `$BugLink(...)` when reproducing. |

The **QAA ticket tells you what to do**. The **B2CQA ticket tells you what to verify**. These are never the same ticket. A single QAA-XXX can cover one B2CQA, or it can bundle five. Read carefully.

```
QAA-702  ─────linked to─────►  B2CQA-604  (Swap history export — ERC20 receive)
 "meta: automate this test"      "scenario: after a swap, user clicks 'Export
                                  history' and CSV contains ERC20 row"
```

The traceability chain below is the reason this distinction exists:

```
QAA-702 (sprint ticket)
   │
   │ links to
   ▼
B2CQA-604 (Xray test case)          ◄── PM and QA leads see coverage here
   │
   │ automated in
   ▼
e2e/specs/swap/swapHistoryExport…spec.ts
   │
   │ runs in CI, uploads results
   ▼
Allure report + Xray execution update
   │
   │ stakeholders trace
   ▼
"Is B2CQA-604 passing on main?"  ◄── single source of truth
```

If you skip the Xray link (`$TmsLink("B2CQA-604")`), your test runs but Xray sees nothing. It is **invisible coverage** — a cardinal sin of the onboarding guide's Part 3 (Allure & Xray) and still a cardinal sin here. The B2CQA ID must appear in the spec.

### 5.9.2 Picking Up a Ticket

You have been assigned `QAA-702`. Before touching code, do the **read-only pass**:

1. **Open the ticket in Jira.** Read the description end-to-end. Twice.
2. **Acceptance Criteria (AC).** If the ticket has an AC block, copy it into your branch notes. If it does not, go back to your lead and ask — a ticket without AC is a ticket you cannot prove done.
3. **Linked Xray tests.** Look at the "Tests" panel on the right. Click through each B2CQA. Note the `preconditions`, `steps`, and `expected result`. These become the `@Step` descriptions in your POM.
4. **Feature area.** Map the ticket to a source tree:
   - Portfolio, Market, Accounts → `apps/ledger-live-mobile/src/screens/Portfolio`, `.../Market`, `.../Accounts`
   - Trade: Swap / Buy / Sell → `.../Swap`, `.../Exchange`
   - Onboarding → `.../Onboarding`
   - Live Apps (DApps, discover) → `.../Platform` and `live-apps` package
   - Settings / Manager / Sync → `.../Settings`, `.../Manager`
5. **CODEOWNERS.** Before you write a line, look up who must review your PR:

   ```bash
   cat .github/CODEOWNERS | grep -E "e2e/mobile|src/screens/Swap"
   # e2e/mobile/                                @ledgerhq/wallet-xp
   # apps/ledger-live-mobile/src/screens/Swap/  @ledgerhq/ptx
   ```

   For a swap-area test you will end up with `@ledgerhq/wallet-xp` (owns the `e2e/mobile/` workspace) plus the relevant feature-team owner (here `@ledgerhq/ptx` for Swap) if you touch any React code.

6. **Estimate scope.** Most QAA tickets fit in 0.5–2 days of focused work. If your first estimate exceeds 3 days, the ticket is probably two tickets in a trench coat. Ask your lead to split it.

### 5.9.3 Branching

Branch names must describe the scope and reference the Jira ID. The global `git-workflow.md` rules apply verbatim:

| Prefix | Use for |
|---|---|
| `feat/` | New feature coverage, new POM, new CI job. |
| `bugfix/` | Fixing a real product-triggered flake, fixing a broken spec. |
| `support/` | Refactor, renames, test-data updates, config tweaks. |
| `chore/` | Dependency bumps, tooling config, lint rules. |

Mobile-scope conventions:

```bash
# New coverage
git checkout -b feat/llm-qaa-702-swap-history-erc20-export

# Bug fix in an existing spec
git checkout -b bugfix/llm-qaa-814-fix-send-ton-flake

# Refactor shared POM
git checkout -b support/llm-extract-modal-base-page
```

The `llm-` segment stands for **Ledger Live Mobile** — it tells the reviewer the platform without opening the diff. Desktop branches use `lld-`, live-common uses `llc-`.

**Conventional Commits** for mobile:

```
test(mobile): add swap history ERC20 export test (QAA-702)
feat(mobile): add SwapHistoryPage POM with @Step decorators
fix(mobile): debounce swap history refresh to stabilize e2e
refactor(mobile): extract bridge CSV listener into helper
chore(mobile): bump detox to 20.x
```

One logical change per commit. A diff that adds a POM, adds userdata, adds a spec, and bumps Detox is four commits, not one. Rebase and polish before you open the PR — the global rules say "rebase before PR to keep history clean."

### 5.9.4 Orientation Pass — Mapping the Terrain

You now open your editor and **`cd e2e/mobile`** — that's where the vast majority of your day-to-day work happens. Before writing, you **recon the repo**. This pass alone catches 80 % of the "I wrote it twice" mistakes.

```bash
# 0. Land in the mobile E2E workspace
cd e2e/mobile

# 1. Is there already a spec for this feature?
ls specs/swap/

# 2. Is there an existing POM?
ls page/trade/

# 3. Which testIDs already exist on screen? (source still lives in the app)
rg "testID=" ../../apps/ledger-live-mobile/src/screens/Swap/History/

# 4. Which userdata fixtures look similar?
ls userdata/ | rg -i "swap|exchange"

# 5. Which Xray IDs already reference this feature?
rg "B2CQA-6(0[0-9]|1[0-9])" .
```

The answers shape the rest of the day:

- **Spec exists** → add an `it()` inside it, do not create a new file.
- **POM exists** → add one or two `@Step` methods, do not create a parallel POM.
- **testID missing on screen** → stop. Open a product ticket, tag `@ledgerhq/ptx`, ask them to add it. **Never** select by text or accessibility label as a substitute — it will break on the next i18n change.
- **Userdata exists** → reuse it. Duplicated fixtures are a maintenance tax.

Write the outcome of the recon in 5 bullets in the Jira ticket. Your reviewer will read those bullets and immediately know why your diff looks the way it does.

### 5.9.5 Designing the Flow

Before writing `it()`, sketch the user flow **top to bottom** as a sequence of POM method calls. Use a scratch file, a whiteboard, or the comment header of your future spec:

```
// QAA-702: swap history export with ERC20 receive
// 1. Launch app with userdata = swapHistoryWithERC20
// 2. PortfolioPage.openSwapTab()
// 3. SwapPage.openHistoryTab()
// 4. SwapHistoryPage.expectOperationRowVisible(swapId)
// 5. SwapHistoryPage.expectERC20Received(swapId, "USDT")
// 6. SwapHistoryPage.tapExportButton()
// 7. Bridge: await sendFile event
// 8. Assert CSV contains USDT row with correct "To account address"
```

Each bullet is one `@Step`. If you cannot express a bullet with an existing method, you have two choices:

1. **Extend the existing POM** — preferred. Adds a `@Step` method to `swap.page.ts`.
2. **Create a new POM** — only when the feature is a new screen with no POM yet (our case for `SwapHistoryPage`).

**Rule of thumb.** If the screen has its own route and its own set of testIDs, it deserves its own POM. If it is a modal over an existing screen, extend the parent POM.

### 5.9.6 Implementation Order

Do not write the spec first. You will re-edit it six times. Instead:

1. **Fixtures (`userdata/*.json`)** — what does the app state need to be at `beforeAll`?
2. **POM methods (`page/*.page.ts`)** — `@Step` decorators, return types, no assertions on the caller's behalf unless the step name says "Expect".
3. **Spec (`specs/**/*.spec.ts`)** — stitch the POM calls into an `it()`.
4. **Dry-run locally** — `pnpm test:ios:debug -- -t "<name>"`.
5. **Fix** — the first run fails. That is normal. Read the failure, open Allure, patch the POM or spec.
6. **Push** to a branch.
7. **CI green** — wait for the full matrix. If flaky, rerun once; if flaky twice, treat as failure and investigate.
8. **Allure check** — open the uploaded report, confirm `@Step` hierarchy and `$TmsLink`.
9. **PR** — follow 39.10.

Do not skip step 8. A green CI with a broken Allure report is a hidden regression — Xray will see "passed", reviewers will see no steps, and debugging when it breaks next month will be miserable.

### 5.9.7 Running Locally

The canonical scripts live in `e2e/mobile/package.json`. Run them from the repo root via `pnpm --filter e2e-mobile run <script>`, or `cd e2e/mobile` and run them directly.

```bash
# ONCE per working session — build the app bundle
#   from repo root:
pnpm --filter e2e-mobile run build:ios:debug
#   or (release bundle, via Nx — same target as CI):
nx run live-mobile:e2e:build -- --configuration ios.sim.release

# TIGHT LOOP — run tests (from e2e/mobile/)
cd e2e/mobile
pnpm test:ios:debug        # ios.sim.debug
pnpm test:ios              # ios.sim.release
pnpm test:android:debug    # android.emu.debug
pnpm test:android          # android.emu.release

# Filter by test name (same -t as Jest)
pnpm test:ios:debug -- -t "swap history containing an ERC20"
```

The Detox configuration is chosen by the script name (`test:ios` → `ios.sim.release`, etc.). See Chapter 5.6 on `detox.config.js` for the full list. Common values:

| Configuration | Target |
|---|---|
| `ios.sim.debug` | iOS simulator with Metro debug bundle — fast reload. |
| `ios.sim.release` | iOS simulator with production bundle — slower, closer to CI. |
| `android.emu.debug` | Android emulator with Metro debug bundle. |
| `android.emu.release` | Android emulator with production bundle — what CI runs. |

`-t "<pattern>"` filters by the `it()` description. Detox passes it straight to Jest's `--testNamePattern`, so **use the exact string** that will appear in your `it()` title.

Local Allure report: `cd e2e/mobile && pnpm allure` (runs `allure generate ./artifacts` then opens the report).

If you work on Android, keep an emulator warm:

```bash
emulator -avd Pixel_7_API_34 -no-snapshot-save -no-audio &
```

iOS simulators auto-boot via `applesimutils` — you do not need to pre-launch them.

### 5.9.8 Adding Allure Steps

Every public POM method should be decorated:

```typescript
@Step("Tap the export history button")
async tapExportButton() {
  await this.exportButton().tap();
}

@Step("Verify ERC20 receive for swap $0 with ticker $1")
async verifyERC20Received(swapId: string, tokenTicker: string) {
  await expect(this.toAmount(swapId)).toHaveText(new RegExp(tokenTicker));
}
```

The `@Step` decorator is imported from `jest-allure2-reporter/api`:

```typescript
import { Step } from "jest-allure2-reporter/api";
```

Behavior:

- `@Step("text")` wraps the method in an Allure step with that name.
- `$0`, `$1`, … are substituted with the method's arguments at call time — same semantics as Chapter 3.5.2's desktop `@step("Click on asset $0")`.
- If the method throws, the step is marked `failed` and the screenshot is attached automatically by the global Jest retry handler (Chapter 5.4.6).

**Do not** nest two `@Step` methods silently. If `verifyERC20Received()` internally calls `tapExportButton()`, you will get a two-level tree. That is usually fine, but if it produces a 40-line single-step trace in the report, flatten it.

### 5.9.9 Xray Linkage

The `$TmsLink` wrapper is how a mobile spec announces its B2CQA coverage. Two equivalent forms depending on your Jest-Allure version:

```typescript
// form 1 — title token
it($TmsLink("B2CQA-604"), "exports swap history containing an ERC20 receive", async () => { … });

// form 2 — programmatic API
it("exports swap history containing an ERC20 receive", async () => {
  allure.tms("B2CQA-604", "https://ledgerhq.atlassian.net/browse/B2CQA-604");
  …
});
```

The title-token form is preferred because Xray's importer reads the Allure test name and automatically extracts `B2CQA-XXX`. It also keeps the spec top clean.

What `$TmsLink` emits in Allure:

- A clickable link labelled `B2CQA-604` in the test's sidebar.
- A `tms` entry in the JSON for the Xray custom reporter to consume.
- Nothing in the spec runtime — the wrapper is a no-op at runtime, it only annotates.

**Multiple Xray IDs per spec.** Wrap more than one:

```typescript
it($TmsLink("B2CQA-604", "B2CQA-605"), "exports swap history …", async () => { … });
```

### 5.9.10 PR Checklist

The PR template lives in `.github/PULL_REQUEST_TEMPLATE.md` and the `/create-pr` Claude Code skill fills most of it in. But the human-readable checklist below is what your reviewer actually looks for:

1. **Title** — `test(mobile): add swap history ERC20 export test (QAA-702)`. Conventional Commits, one line, references the Jira ticket in parens.
2. **Jira link** — the QAA ticket URL at the top of the description.
3. **Description** — acceptance criteria restated, a one-paragraph summary of the approach, a "testing notes" section listing the Detox configs you ran.
4. **Test evidence** — a screenshot or screencast of the test passing locally (Allure step list or Detox console output), plus the CI run link once green.
5. **CODEOWNERS** — GitHub auto-requests, but do a sanity check: swap-area touches should have `@ledgerhq/wallet-xp` and `@ledgerhq/ptx`.
6. **Reviewers** — your lead plus one peer familiar with the area.
7. **Semantic-release impact** — for a test-only PR, the `test(mobile): …` prefix means **no version bump**. For a `fix(mobile): …` inside `e2e/mobile/` only, still no bump — the publishable package is the app, not the e2e harness. For `fix(mobile-app): …` inside `apps/ledger-live-mobile/src/**`, a patch bump will trigger on `main`.
8. **Changeset** — the monorepo uses Changesets. If your diff only touches `e2e/mobile/`, you do not need one. If you modify `apps/ledger-live-mobile/src/`, run `pnpm changeset` and pick "patch" or "none" as appropriate.

Open the PR as **draft** first. Mark it ready only after CI is green.

### 5.9.11 Review Etiquette

The golden rules, applied to mobile:

- **Small diffs per ticket.** A 1,500-line PR does not get a careful review — it gets a rubber stamp or a rejection. Split.
- **Rebase, do not merge.** `git pull --rebase origin dev` before pushing. A spaghetti merge graph makes `git bisect` useless.
- **Squash only for trivial branches.** `support/cleanup` branches squash cleanly. Feature branches with a thoughtful commit history should preserve that history on merge.
- **Address review comments in new commits, not amends.** The reviewer wants to see *what changed since I last looked* — a force-push nukes that. Only squash at the very end, and only if the reviewer agrees.
- **Block your own PR until CI is green.** A reviewer should never be the one who tells you CI is red.

Reviewer-facing signals that you did the work:

- The recon bullets from 39.4 in the PR description.
- The Allure link in the PR comments (or a screenshot if the report has rolled off).
- A note if any product change (testID addition, bridge extension) was required, with the linked product ticket.

<div class="chapter-outro">
<strong>Key takeaway.</strong> The workflow is: QAA ticket → B2CQA scenario → branch with <code>llm-</code> prefix → recon the repo → sketch the flow → write fixtures → write POM → write spec → dry-run loop → push → wait for CI → inspect Allure → open a small, well-scoped PR with CODEOWNERS auto-requested. The same dance, every ticket. Predictable is professional.
</div>

### 5.9.12 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Daily Mobile Workflow</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> What is the difference between <code>QAA-702</code> and <code>B2CQA-604</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) They are aliases for the same ticket</button>
<button class="quiz-choice" data-value="B">B) <code>QAA-702</code> is the sprint work item; <code>B2CQA-604</code> is the Xray test scenario it is expected to automate</button>
<button class="quiz-choice" data-value="C">C) <code>QAA-702</code> is mobile-only; <code>B2CQA-604</code> is desktop-only</button>
<button class="quiz-choice" data-value="D">D) <code>B2CQA-604</code> is a bug report</button>
</div>
<p class="quiz-explanation">QAA tickets are sprint work items ("automate this"). B2CQA tickets live in Xray and describe a reusable test scenario that may be covered by LLD, LLM, or LLC.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> A testID you need is missing from a screen. What do you do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Select by visible text until the testID is added</button>
<button class="quiz-choice" data-value="B">B) Select by accessibility label</button>
<button class="quiz-choice" data-value="C">C) Stop, open a product ticket with <code>@ledgerhq/ptx</code>, and wait for the testID to land before writing the test</button>
<button class="quiz-choice" data-value="D">D) Ship the test with an XPath query</button>
</div>
<p class="quiz-explanation">Text and accessibility labels are i18n-fragile. The repo rule is: testIDs are the contract. If one is missing, product adds it first. Writing a test against a text selector is technical debt the next translator will silently break.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> Which branch name correctly follows the mobile conventions?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>QAA-702-swap-export</code></button>
<button class="quiz-choice" data-value="B">B) <code>swap-history-erc20</code></button>
<button class="quiz-choice" data-value="C">C) <code>feature/swap-history</code></button>
<button class="quiz-choice" data-value="D">D) <code>feat/llm-qaa-702-swap-history-erc20-export</code></button>
</div>
<p class="quiz-explanation">Prefix (<code>feat/</code>) + platform (<code>llm-</code>) + Jira ID (<code>qaa-702</code>) + short scope. The kebab-case format is required by <code>git-workflow.md</code>.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> What is the correct implementation order for a new mobile spec?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Userdata fixtures → POM methods → spec → dry-run → push</button>
<button class="quiz-choice" data-value="B">B) Spec → POM methods → userdata</button>
<button class="quiz-choice" data-value="C">C) POM methods → spec → userdata</button>
<button class="quiz-choice" data-value="D">D) Dry-run first, then write</button>
</div>
<p class="quiz-explanation">You cannot write a useful spec until the POM methods exist, and POM methods are meaningless without a known starting state (userdata). Bottom-up is the only sane order.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> What does <code>$TmsLink("B2CQA-604")</code> at the start of an <code>it()</code> do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Causes the test to fail if Xray is unreachable</button>
<button class="quiz-choice" data-value="B">B) Annotates the Allure result with a clickable TMS link so Xray can match the spec to the scenario</button>
<button class="quiz-choice" data-value="C">C) Fetches the ticket description at runtime</button>
<button class="quiz-choice" data-value="D">D) Triggers a network request to Jira</button>
</div>
<p class="quiz-explanation">The wrapper is a no-op at runtime. It adds a <code>tms</code> annotation to the Allure result, which the Xray importer reads to update B2CQA-604's execution status.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q6.</strong> Why do you open the PR as draft first?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Draft PRs run faster in CI</button>
<button class="quiz-choice" data-value="B">B) GitHub requires it</button>
<button class="quiz-choice" data-value="C">C) It exempts the PR from CODEOWNERS</button>
<button class="quiz-choice" data-value="D">D) It lets you validate CI and Allure before burning reviewer attention — you mark ready only once the PR is actually reviewable</button>
</div>
<p class="quiz-explanation">A reviewer's time is the scarce resource. A red CI or a broken Allure report is something you find — not something your reviewer finds.</p>
</div>

<div class="quiz-score"></div>
</div>

---


## Real Ticket Walkthrough: QAA-702 — Swap History ERC20 Export

<div class="chapter-intro">
Your mobile capstone. <strong>QAA-702</strong> — <em>[SWAP] [LLM] Improve 'History' test</em> — is a completed sprint ticket (retrospective walkthrough). The parent scenario <strong>B2CQA-604</strong> already has step 1 automated (native → native swap export); QAA-702 asks you to add step 2: the same export flow, but with an <strong>ERC20 token on the Receive side</strong>. This is not a recap. By the end of this chapter you will have a branch, a new spec file, an extended userdata fixture, a green Detox run, and an Allure report ready to paste into a PR.
</div>

### 5.10.1 Understanding the Ticket

**Jira ticket:** QAA-702 · **Parent epic:** QAA-919 *"SWAP regression coverage automation"*
**Xray test case:** B2CQA-604 *"[SWAP] User should be able to export all history operations"*
**Linked bug:** LIVE-19533 *"[SWAP][LLM][iOS] Can't export Swap history"* (closed)
**Status:** Done · not yet automated · labels `LLM`, `UI`

The ticket text, verbatim:

> Only LLM related.
>
> Base test B2CQA-604 should be improved if possible, as we faced the bug of exporting History fail.
>
> So, need to add the step of History export, when there is a tx with ERC20 token in Receive field.

**What "step 2" means.** `B2CQA-604` is not a single-step test case. Inside Xray it contains multiple numbered steps; each step is a distinct export scenario that the automation suite must cover. Step 1 — a SOL → ETH swap where the Receive side is **native ETH** — is already automated (you will see it in a moment). Step 2 — adding coverage where the Receive side is an **ERC20 token on Ethereum** (e.g., USDT) — is what `QAA-702` asks for.

**Why it matters historically.** Ledger's own swap team filed `LIVE-19533` last cycle: on iOS, exporting the swap history failed whenever the history contained an ERC20 receive. The bug was fixed, but no regression test was added for that specific shape. `QAA-702` closes that regression hole.

**Why the test case ID does not change.** Pavlo OKHONKO left a comment on B2CQA-604 on 2025-07-21:

> "Keep this test in current suite till step2 to be automated: QAA-702"

So our new spec will reuse `tmsLinks: ["B2CQA-604"]`. In Xray, both steps map to the same B2CQA card; in the repo, they live in two sibling spec files sharing the same driver.

### 5.10.2 Why This Matters

ERC20 receive is by far the most common swap shape on Ethereum. A user who swaps BTC or SOL into "a dollar on Ethereum" almost always lands in USDT or USDC, not native ETH. Without this coverage:

- **Export regressions are invisible.** A change in how the CSV serialises a `TokenAccount` row would pass CI even if it corrupted every non-native receive.
- **Bug LIVE-19533 can resurface.** The original failure was specific to how ERC20 operations were iterated on iOS. Nothing prevents a similar regression tomorrow.
- **Xray dashboards lie.** Coverage looks green at the B2CQA level even though only half the scenario is exercised.

Adding step 2 is a small, high-leverage fix: one spec file, one fixture append, reuse of an already-battle-tested driver.

### 5.10.3 Analyzing Existing Coverage

Before writing anything, read what is already there.

**The step 1 spec** — `e2e/mobile/specs/swap/otherTestCases/swapExportHistoryOperations.spec.ts`, verbatim:

```typescript
import { Account } from "@ledgerhq/live-common/e2e/enum/Account";
import { Provider } from "@ledgerhq/live-common/e2e/enum/Provider";
import { Addresses } from "@ledgerhq/live-common/e2e/enum/Addresses";
import { runExportSwapHistoryOperationsTest } from "./swap.other";

const swapHistoryTestConfig = {
  swap: new Swap(Account.SOL_1, Account.ETH_1, "0.07"),
  provider: Provider.EXODUS,
  swapId: "wQ90NrWdvJz5dA4",
  addressFrom: Addresses.SWAP_HISTORY_SOL_FROM,
  addressTo: Addresses.SWAP_HISTORY_ETH_TO,
  tmsLinks: ["B2CQA-604"],
  tags: [
    "@NanoSP",
    "@LNS",
    "@NanoX",
    "@Stax",
    "@Flex",
    "@NanoGen5",
    "@solana",
    "@family-solana",
    "@ethereum",
    "@family-evm",
  ],
};

runExportSwapHistoryOperationsTest(
  swapHistoryTestConfig.swap,
  swapHistoryTestConfig.provider,
  swapHistoryTestConfig.swapId,
  swapHistoryTestConfig.addressFrom,
  swapHistoryTestConfig.addressTo,
  swapHistoryTestConfig.tmsLinks,
  swapHistoryTestConfig.tags,
);
```

**The driver** — `e2e/mobile/specs/swap/otherTestCases/swap.other.ts`, the function we will reuse:

```typescript
export function runExportSwapHistoryOperationsTest(
  swap: SwapType,
  provider: Provider,
  swapId: string,
  addressFrom: string,
  addressTo: string,
  tmsLinks: string[],
  tags: string[],
) {
  describe("Swap history", () => {
    beforeAll(async () => {
      await app.speculos.setExchangeDependencies(swap);
      await beforeAllFunctionSwap({
        userdata: "swap-history",
        speculosApp: AppInfos.EXCHANGE,
      });
    });

    tmsLinks.forEach(tmsLink => $TmsLink(tmsLink));
    tags.forEach(tag => $Tag(tag));
    it(`Export swap history operations - ${swap.accountToDebit.currency.name} to ${swap.accountToCredit.currency.name}`, async () => {
      swap.accountToDebit.address = addressFrom;
      swap.accountToCredit.address = addressTo;
      await app.swap.goToSwapHistory();
      await app.swap.clickExportOperations();
      await app.swap.checkExportedFileContents(swap, provider, swapId);
    });
  });
}
```

**The assertion** — `e2e/mobile/page/trade/swap.page.ts`, the method the driver calls at the end:

```typescript
@Step("Check contents of exported operations file")
async checkExportedFileContents(swap: SwapType, provider: Provider, id: string) {
  const targetFilePath = path.resolve(__dirname, "../../artifacts/ledgerwallet-swap-history.csv");
  const fileContents = await fs.readFile(targetFilePath, "utf-8");

  jestExpect(fileContents).toContain(provider.name);
  jestExpect(fileContents).toContain(id);
  jestExpect(fileContents).toContain(swap.accountToDebit.currency.ticker);
  jestExpect(fileContents).toContain(swap.accountToCredit.currency.ticker);
  jestExpect(fileContents).toContain(swap.amount);
  jestExpect(fileContents).toContain(swap.accountToDebit.accountName);
  jestExpect(fileContents).toContain(swap.accountToDebit.address);
  jestExpect(fileContents).toContain(swap.accountToCredit.accountName);
  jestExpect(fileContents).toContain(swap.accountToCredit.address);
}
```

**Conclusion.** The driver is fully parameter-safe. It takes any `SwapType`, any `Provider`, any `swapId`, and any two addresses. Nothing about it is hard-wired to native-receive. The assertion only reads `currency.ticker`, `accountName`, and `address` from each side of the swap — all of which a `TokenAccount` exposes correctly (more on that in 4.10.4). **The gap is pure data coverage: we need a second spec, targeting the same driver, with an ERC20 receive account and a matching fixture entry.**

### 5.10.4 Picking the Representative Case

**Choice: SOL → USDT (ERC-20 on Ethereum).** Reasons:

- **Matches the bug.** LIVE-19533 was triggered by an ERC20 receive; USDT is the archetype.
- **USDT is the highest-volume ERC20 Ledger Live users receive from swaps.** USDC is a close second, covered in 5.10.15 as the obvious follow-up.
- **The enum entry already exists.** `TokenAccount.ETH_USDT_1` is defined at `libs/ledger-live-common/src/e2e/enum/Account.ts:319` with `accountName: "Tether USD 1"`, `Currency.ETH_USDT` (ticker `"USDT"`), and `parentAccount: Account.ETH_1`.
- **Reuses the step-1 spec shape exactly.** Only three fields change in the spec: the receive-side account (`TokenAccount.ETH_USDT_1` instead of `Account.ETH_1`), the `swapId`, and the two address constants (because the swap we capture in Phase 2 will be executed on a different SOL/ETH holder pair from step 1 — we will add two new entries to `Addresses.ts`).

**Alternative considered: SOL → USDC.** Equally valid; `TokenAccount.ETH_USDC_1` exists at `Account.ts:310`. If the Swap team prefers USDC as the first ERC20 scenario, the only changes are the enum name and the captured `swapId`. Either choice discharges the ticket. We pick USDT and mention USDC in the reference section.

**Why not ETH → USDT?** A native-to-ERC20 swap would cover the scenario too, but the step-1 precedent (`SOL_1` as debit) and the fixture shape (SOL history on the sending side) make **keeping the debit side as SOL** the lowest-churn change. One data axis moves — the Receive side — which is what the ticket asks for.

### 5.10.5 Create a Branch

Following the repo convention for new E2E coverage — `support/` prefix, ticket slug:

```bash
cd /path/to/ledger-live
git checkout develop
git pull
pnpm i

git checkout -b support/qaa-702-swap-history-erc20-export
```

Why `support/` and not `feat/`: the change lives entirely under `e2e/mobile/`, `e2e/mobile/userdata/`, and `libs/ledger-live-common/src/e2e/enum/`. We are shipping test coverage, not product behaviour. This matches the commit type we will use below (`test(mobile): ...`) — check recent PRs landing swap specs to confirm the team's current preference; `feat(mobile):` is also tolerated.

### 5.10.6 Run the Baseline in Isolation

Never judge your own change against a broken baseline. Confirm step 1 is green **before** touching anything.

**One-time build** (required after a fresh clone or any native-code change):

```bash
# iOS — needs macOS + Xcode
pnpm --filter ledger-live-mobile-e2e-tests run build:ios:debug

# Android — needs Android SDK + a running emulator
pnpm --filter ledger-live-mobile-e2e-tests run build:android:debug
```

**Run step 1 alone:**

```bash
cd e2e/mobile
pnpm test:ios:debug -- --testPathPattern "swapExportHistoryOperations"
```

Expected output: **1 test passing**, the SOL → native-ETH export. The test takes about 45–90 seconds end-to-end (Speculos boot, app launch, userdata hydration, navigation, CSV write, CSV read, teardown).

If this run is red, **stop**. It means either your environment is broken (Speculos not running, wrong Detox build, stale metro cache) or the step 1 spec has started failing for unrelated reasons. Chapters 5.4 (Toolchain) and 5.8 (Running & Debugging) walk through the recovery paths. Fix the environment first; resume QAA-702 once the baseline is green.

### 5.10.7 Phase 1 — Check if the Pair Exists in the Current Fixture

The driver hardcodes `userdata: "swap-history"` → `e2e/mobile/userdata/swap-history.json`. We need to know whether it already contains a SOL → USDT swap operation. Rather than grepping through 5,000 lines of dense JSON, load the fixture into Ledger Live Desktop and browse the swap history in the product UI — faster, and it lets you see the entries as a user would.

**On macOS the LLD state file is** `~/Library/Application Support/Ledger Live/app.json` (Linux: `~/.config/Ledger Live/app.json`; Windows: `%AppData%/Ledger Live/app.json`). The fixture file has the same shape as that app state, by design.

Steps:

1. **Quit Ledger Live Desktop** completely (Cmd-Q on macOS). A running instance will overwrite any change you make on disk when it exits.
2. **Back up your personal state** so you can restore it after:

   ```bash
   cd "$HOME/Library/Application Support/Ledger Live"
   mv app.json app2.json
   ```
3. **Copy the fixture in:**

   ```bash
   cp /path/to/ledger-live/e2e/mobile/userdata/swap-history.json \
      "$HOME/Library/Application Support/Ledger Live/app.json"
   ```
4. **Relaunch Ledger Live Desktop.** It will hydrate with the fixture's accounts and swap history.
5. **Navigate to Swap → History.** Scan for any SOL → USDT entries.

You will find **two SOL → native-ETH** operations (both `wQ90NrWdvJz5dA4`) and **no ERC20 receive entry**. That is the gap QAA-702 asks us to close — so we need to produce a real ERC20-receive swap and fold it into the fixture.

> **Restore your personal state when you're done experimenting:** quit LLD, rename `app2.json` back to `app.json`. Don't forget this — running your personal account list through a dev build of LLD is not the mistake we want to debug.

### 5.10.8 Phase 2 — Create the Pair on a Real Device

To produce a genuine swap operation we execute it for real, then scrape the result off the exported CSV.

1. **Start from the fixture state** (Phase 1 step 3 already put the fixture in `app.json`). Launch LLD.
2. **Plug in a Ledger device** loaded with the **QAA test seed**. That seed owns the Solana account (`Solana 1`) already present in the fixture — otherwise the swap won't sign.
3. **Open Swap.** Choose **SOL** as the *from* asset (pick your existing Solana 1 account), **USDT** on Ethereum as the *to* asset.
4. **Enter the minimum amount — `0.07` SOL** (on French locale, LLD renders this as `0.07` — the comma form is what the Swap class stores, and what you'll pass back in the spec below).
5. **Select a quote.** Today only **NEAR Intents** quotes this pair; take it. Quote availability drifts over time, so note whatever you use.
6. **Confirm the swap on the device.** Let it execute to completion.
7. **Go to Swap → History.** Confirm the new entry shows up.
8. **Click "Export operations"** to generate the CSV, then open the file.
9. **Capture three values from the CSV** — you will need them verbatim for Phase 3 and Phase 4:
   - The **swap ID** (UUID, e.g., `1172570f-5a02-43b9-83fc-cad47bfd12f3`).
   - The **Solana sender address** — the SOL holder that paid for the swap.
   - The **Ethereum recipient address** — the ETH holder that received the USDT.

Because the swap was executed on the QAA seed (not the synthetic fixture seed from step 1), both addresses will be new — they do NOT match `SWAP_HISTORY_SOL_FROM` or `SWAP_HISTORY_ETH_TO`. That's why we will add two new `Addresses.ts` constants in Phase 4.

### 5.10.9 Phase 3 — Import the New Operation into the Test Fixture

After Phase 2 the new swap lives in your local `app.json` at `~/Library/Application Support/Ledger Live/app.json`. It needs to be copied into the test fixture `e2e/mobile/userdata/swap-history.json` so the spec has something to assert against.

The scale mismatch: `app.json` is roughly **25,000 lines** (full LLD state — every account, every operation, every cached balance, every feature-flag override), while `swap-history.json` is closer to **5,000 lines** (trimmed to what the swap suite actually needs). You do **not** want to wholesale replace the fixture with your personal dump. Instead, extract just the Solana account block containing the new swap and graft it in.

Steps:

1. **Open `app.json` in an editor** that can handle a 25k-line file (VS Code is fine). Format it (`jq '.' app.json > app.formatted.json` or run your editor's Prettier-on-save) so the indentation matches the fixture.
2. **Search for the swap ID** you captured in Phase 2 — e.g., `1172570f-5a02-43b9-83fc-cad47bfd12f3`. You will land inside the Solana account's `data.swapHistory` array. Scroll out to the enclosing *accounts[i]* wrapper — the `{ "data": { ... }, "version": 1 }` object.
3. **Copy the entire accounts[i] wrapper** — from the opening `{` (immediately before `"data": {`) through the matching closing `}` after `"version": 1`. Example shape (abbreviated):

   ```json
   {
     "data": {
       "id": "js:2:solana:FDaDTiZbkXh5H5mdfsW3UHVME15qgMPAcKE7JowzU61Z:solanaSub",
       "freshAddress": "FDaDTiZbkXh5H5mdfsW3UHVME15qgMPAcKE7JowzU61Z",
       "operationsCount": 3,
       "operations": [ /* … */ ],
       "currencyId": "solana",
       "balance": "77232851",
       "swapHistory": [
         {
           "status": "finished",
           "provider": "nearintents",
           "operationId": "js:2:solana:…-OUT",
           "swapId": "1172570f-5a02-43b9-83fc-cad47bfd12f3",
           "receiverAccountId": "js:2:ethereum:0x70AAEEe70118a065ddF84dF6669b496A447C8CcC:",
           "tokenId": "ethereum/erc20/usd_tether__erc20_",
           "fromAmount": "69637880",
           "toAmount": "5623953.000000000000000000000000000000000106324",
           "finalAmount": "5.611443"
         },
         {
           "status": "finished",
           "provider": "nearintents",
           "operationId": "js:2:solana:…-OUT",
           "swapId": "1172570f-5a02-43b9-83fc-cad47bfd12f3",
           "receiverAccountId": "js:2:ethereum:0x70AAEEe70118a065ddF84dF6669b496A447C8CcC:+ethereum%2Ferc20%2Fusd~!underscore!~tether~!underscore!~~!underscore!~erc20~!underscore!~",
           "tokenId": "js:2:ethereum:0x70AAEEe70118a065ddF84dF6669b496A447C8CcC:+ethereum%2Ferc20%2Fusd~!underscore!~tether~!underscore!~~!underscore!~erc20~!underscore!~",
           "fromAmount": "69637880",
           "toAmount": "5622578",
           "finalAmount": "5.611443"
         }
       ],
       "name": "Solana 1",
       "starred": false
     },
     "version": 1
   }
   ```

   Notice the pattern: LLD stores **two** entries per ERC20 swap — one addressed to the parent ETH account (native form) and one to the ETH+token sub-account (the `+ethereum%2Ferc20%2F...` suffix). Keep both; the driver's CSV export assertion reads whichever the UI hands it.
4. **Open `e2e/mobile/userdata/swap-history.json`.** Search for the existing step-1 swap ID, `wQ90NrWdvJz5dA4`. You'll find it inside the existing SOL account block.
5. **Paste your copied accounts[i] wrapper immediately after** the existing SOL account block's closing `}`. Fix the commas: the previous block must end with `,`, and yours is followed by whatever separator (`,` or `]`) was there already.
6. **Validate the JSON:**

   ```bash
   jq '.' e2e/mobile/userdata/swap-history.json > /dev/null
   ```

   A syntax error here will kill every spec in `specs/swap/otherTestCases/`, so validate before moving on.
7. **(Optional) Trim noise.** LLD may have filled your account block with additional fields the fixture does not use (per-session feature-flag overrides, analytics breadcrumbs). Nothing forces you to trim them — but if the fixture grows past what feels reasonable, compare against neighbouring accounts already in the file and delete the fields they don't include.

### 5.10.10 Phase 4 — Extend the Addresses Enum, Teach the Driver about ERC20 Names, and Implement the Spec

Three files change: `Addresses.ts` gets two new constants, `swap.page.ts`'s CSV assertion gains an ERC20 parent-name fallback, and the new spec file wires everything together.

**Step 1 — Extend `libs/ledger-live-common/src/e2e/enum/Addresses.ts`:**

```ts
export enum Addresses {
  BTC_NATIVE_SEGWIT_1 = "bc1q2dh3p38d3pavjkvvk7wr25dcdnclvjnll9kta9",
  SANCTIONED_ETHEREUM = "0x04DBA1194ee10112fE6C3207C0687DEf0e78baCf",
  ETH_2 = "0xc79c7a29c40Ce8F5746af2c956F93F27e2820307",
  ETH_OTHER_SEED = "0xce12D0A5cFf4A88ECab96ff8923215Dff366127b",
  SOL_OTHER_SEED = "DLVArScX1BQEr7gZSSpHMGMq7HKKFKdFF82Cs6PvEKVC",
  SOL_GIGA_1_ATA_ADDRESS = "4h9zusLsPZsZVajWH1Dtgd4ccopL4fvNAE7T8LxX4H1g",
  SOL_GIGA_2_ATA_ADDRESS = "FWS3ZK9E4vc6Wz29vWzPpY7CLNebBRYe1FaxY7EdQDGf",
  SOL_WIF_2_ATA_ADDRESS = "FDFqWGGg9mP1JBVLS4UXDhs4y5mHnHNDXQxCGUHr81Vz",
  SWAP_HISTORY_SOL_FROM = "21kh76PRK8k6UFgd7uwpmkCq1V5q9B8WNKHvwYNgTNub",
  SWAP_HISTORY_ETH_TO = "0x8526F50A2FA870B1B7b91cc054aa06799dAc0110",
  SWAP_HISTORY_ERC20_SOL_FROM = "FDaDTiZbkXh5H5mdfsW3UHVME15qgMPAcKE7JowzU61Z",
  SWAP_HISTORY_ERC20_ETH_USDT_TO = "0x70AAEEe70118a065ddF84dF6669b496A447C8CcC",
}
```

The last two entries are the `from` and `to` addresses you captured from the CSV in Phase 2. Keep the existing `SWAP_HISTORY_*` constants untouched — step 1 still consumes them.

**Step 2 — Teach `swap.page.ts` that ERC20 rows export the parent account's name, not the token account's name.** Ledger Live's CSV exporter writes the *parent* Ethereum account's `accountName` (e.g., `"Ethereum 1"`) when the row involves an ERC20 TokenAccount — not the token-side name (`"Tether USD 1"`). The existing `checkExportedFileContents` asserted on `swap.accountToCredit.accountName`, which works for native-ETH step 1 but breaks on any ERC20 pair. Update the driver helper to fall back to the parent when the account is an ERC20 TokenAccount.

Open `e2e/mobile/page/trade/swap.page.ts` and replace the existing `checkExportedFileContents` method with:

```typescript
@Step("Check contents of exported operations file")
async checkExportedFileContents(swap: SwapType, provider: Provider, id: string) {
  const targetFilePath = path.resolve(__dirname, "../../artifacts/ledgerwallet-swap-history.csv");
  const fileContents = await fs.readFile(targetFilePath, "utf-8");

  // If sender or receiver is a ERC20 token, the csv export takes the parent name
  let accountToDebitNameInCsv: string;
  let accountToCreditNameInCsv: string;

  if (swap.accountToDebit.tokenType === TokenType.ERC20 && swap.accountToDebit.parentAccount) {
    accountToDebitNameInCsv = swap.accountToDebit.parentAccount.accountName;
  } else {
    accountToDebitNameInCsv = swap.accountToDebit.accountName;
  }
  if (swap.accountToCredit.tokenType === TokenType.ERC20 && swap.accountToCredit.parentAccount) {
    accountToCreditNameInCsv = swap.accountToCredit.parentAccount.accountName;
  } else {
    accountToCreditNameInCsv = swap.accountToCredit.accountName;
  }

  jestExpect(fileContents).toContain(provider.name);
  jestExpect(fileContents).toContain(id);
  jestExpect(fileContents).toContain(swap.accountToDebit.currency.ticker);
  jestExpect(fileContents).toContain(swap.accountToCredit.currency.ticker);
  jestExpect(fileContents).toContain(swap.amount);
  jestExpect(fileContents).toContain(accountToDebitNameInCsv);
  jestExpect(fileContents).toContain(swap.accountToDebit.address);
  jestExpect(fileContents).toContain(accountToCreditNameInCsv);
  jestExpect(fileContents).toContain(swap.accountToCredit.address);
}
```

What changed, line-by-line:

- Two new locals `accountToDebitNameInCsv` / `accountToCreditNameInCsv` resolve the correct account name to assert against.
- Each local is set to the **parent** `accountName` when the side is an ERC20 TokenAccount (guarded on both `tokenType === TokenType.ERC20` and the presence of `parentAccount`, so a malformed enum entry can't trigger a null dereference), and falls back to the account's own `accountName` otherwise.
- The `toContain(swap.accountToDebit.accountName)` and `toContain(swap.accountToCredit.accountName)` assertions are replaced with the resolved locals.
- Ticker, address, amount, provider, and `swapId` assertions are unchanged — those already read the right source.
- `TokenType` must be in scope. If it is not already imported at the top of the file, add it alongside the existing enum imports, e.g. `import { TokenType } from "@ledgerhq/live-common/e2e/enum/TokenType"` (verify the exact export path against the file — it lives alongside `TokenAccount`).

This change is backward-compatible: step 1 (SOL → native ETH) takes neither `if` branch, so `accountToCreditNameInCsv` still resolves to `"Ethereum 1"` (the step-1 expectation). No adjustment to step 1 is required — but your rerun matrix must still include it, because a driver change always has to clear the existing green before you trust it on the new case.

**Step 3 — Create the spec** at `e2e/mobile/specs/swap/otherTestCases/swapExportHistoryOperationsERC20.spec.ts`:

```ts
import { Account, TokenAccount } from "@ledgerhq/live-common/e2e/enum/Account";
import { Provider } from "@ledgerhq/live-common/e2e/enum/Provider";
import { Addresses } from "@ledgerhq/live-common/e2e/enum/Addresses";
import { runExportSwapHistoryOperationsTest } from "./swap.other";

const solMinAmount = "0.07";
const swapId = "1172570f-5a02-43b9-83fc-cad47bfd12f3";

const swapHistoryERC20TestConfig = {
  swap: new Swap(Account.SOL_1, TokenAccount.ETH_USDT_1, solMinAmount),
  provider: Provider.NEAR_INTENTS,
  swapId,
  addressFrom: Addresses.SWAP_HISTORY_ERC20_SOL_FROM,
  addressTo: Addresses.SWAP_HISTORY_ERC20_ETH_USDT_TO,
  tmsLinks: ["B2CQA-604"],
  tags: [
    "@NanoSP",
    "@LNS",
    "@NanoX",
    "@Stax",
    "@Flex",
    "@NanoGen5",
    "@solana",
    "@family-solana",
    "@ethereum",
    "@family-evm",
  ],
};

runExportSwapHistoryOperationsTest(
  swapHistoryERC20TestConfig.swap,
  swapHistoryERC20TestConfig.provider,
  swapHistoryERC20TestConfig.swapId,
  swapHistoryERC20TestConfig.addressFrom,
  swapHistoryERC20TestConfig.addressTo,
  swapHistoryERC20TestConfig.tmsLinks,
  swapHistoryERC20TestConfig.tags,
);
```

Field-by-field justification:

- **`swap: new Swap(Account.SOL_1, TokenAccount.ETH_USDT_1, solMinAmount)`** — debit is `Account.SOL_1`, credit is the ERC20 token account `TokenAccount.ETH_USDT_1`. Note the `TokenAccount` import is added to the same `@ledgerhq/live-common/e2e/enum/Account` module as the base `Account` enum. The amount is passed as the string `"0.07"` (dot form used throughout the codebase). `Swap` is a global injected by the Jest environment — do not `import` it.
- **`solMinAmount = "0.07"`** hoisted to a named constant so the magic number doesn't get lost in the object.
- **`provider: Provider.NEAR_INTENTS`** — must match the `"provider": "nearintents"` field in the fixture entry you just pasted.
- **`swapId`** hoisted and set to the UUID you copied from the CSV; the driver uses it verbatim to locate the CSV row.
- **`addressFrom: Addresses.SWAP_HISTORY_ERC20_SOL_FROM`** — the Solana holder the swap was executed from (captured in Phase 2, added to the enum in Step 1 above).
- **`addressTo: Addresses.SWAP_HISTORY_ERC20_ETH_USDT_TO`** — the Ethereum holder that received the USDT. Two new constants, not a reuse of step 1's — different seed, different addresses.
- **`tmsLinks: ["B2CQA-604"]`** — reuse of the same Xray case ID per Pavlo's step-2 comment on the ticket. In the Allure output this spec will display a `B2CQA-604` badge alongside step 1's identical badge; Xray is fine with two automated executions mapping to one test case.
- **`tags`** — same set as step 1. The pair is cross-family (SOL → USDT on Ethereum) so both `@family-solana` and `@family-evm` apply; a CI filter on either will pick this spec up.

### 5.10.11 Rerun the Tests

With the spec and fixture in place, run only the new file:

```bash
cd e2e/mobile
pnpm test:ios:debug -- --testPathPattern "swapExportHistoryOperationsERC20"
```

**On the first run, expect RED.** Typical first-fail causes, in order of likelihood:

1. **Fixture JSON is malformed** — most common. `jq .` on the file catches this in one command. Fix the comma/brace that went missing when you pasted the accounts block in Phase 3.
2. **`swapId` mismatch** between spec and fixture. The UUID you captured from the CSV (e.g., `1172570f-5a02-43b9-83fc-cad47bfd12f3`) must appear verbatim in both files; the driver compares byte-for-byte.
3. **`toContain` miss on `swap.amount`.** The amount string you pass to the `Swap` constructor must appear *as written* in the CSV. Here it is `"0.07"` (dot form — the amount string as used to the CSV). If you see a `toContain("0.07")` failure, open the CSV and confirm what character the app actually used; on an English-locale build it may write `0.07` instead, in which case align the spec's `solMinAmount` with the CSV.
4. **`provider` mismatch.** `Provider.NEAR_INTENTS` resolves to the string `"nearintents"` in the enum; the fixture entry must use the same key. If you took the swap through a different provider, update both.
5. **`toContain` miss on `swap.accountToCredit.accountName`** — the driver asserts that `"Tether USD 1"` appears in the exported CSV. If the product has recently renamed the default USDT account label, you will need to align `TokenAccount.ETH_USDT_1.accountName` with reality (a live-common change — separate concern, cover it in a follow-up PR).

Walk failures by opening the Detox console output, then the Allure report (next section), then iterating. Do **not** change the driver to "work around" a fixture problem. The driver is shared; it is correct for step 1 and for every sibling in `swap.other.ts`.

Once green, run three times in a row for stability (mobile E2E is flakier than desktop by an order of magnitude):

```bash
for i in 1 2 3; do
  pnpm test:ios:debug -- --testPathPattern "swapExportHistoryOperationsERC20" || break
done
```

All three passes is the acceptance bar.

### 5.10.12 Verify with Allure

From `e2e/mobile/`:

```bash
pnpm allure
```

This runs `allure generate ./artifacts` then `allure open`. In the report:

- Navigate **Suites → Swap history → Export swap history operations - Solana to Tether USD**
- Check the **Links** panel — a `B2CQA-604` badge must be present (injected by `tmsLinks.forEach($TmsLink)` in the driver)
- Expand **Steps** — you should see:
  1. *Go to swap history* (from `goToSwapHistory()`)
  2. *Click on export operations* (from `clickExportOperations()`)
  3. *Check contents of exported operations file* (from `checkExportedFileContents()`)
- The step list must match step 1's shape exactly. If your tree diverges (extra step, missing step), the driver contract has broken.
- **Tags** — confirm the 10 tags you set appear as Allure labels. CI's `test_filter` matches against these.

Paste a screenshot of the step tree into the PR body when you open the review.

### 5.10.13 Commit the Change

Four concerns, four commits is cleanest:

```bash
git add libs/ledger-live-common/src/e2e/enum/Addresses.ts
git commit -m "test(mobile): add SWAP_HISTORY_ERC20_* addresses to Addresses enum"

git add e2e/mobile/page/trade/swap.page.ts
git commit -m "test(mobile): handle ERC20 parent-name fallback in swap CSV assertion"

git add e2e/mobile/userdata/swap-history.json
git commit -m "test(mobile): add ERC20 (USDT) swap history entry to swap-history userdata"

git add e2e/mobile/specs/swap/otherTestCases/swapExportHistoryOperationsERC20.spec.ts
git commit -m "test(mobile): add ERC20 receive scenario to swap history export (QAA-702)"
```

Why four commits: each concern is independently reversible and reviewable. The `Addresses` enum change touches live-common and may want isolated review by that module's owners; the `swap.page.ts` change is a driver-level contract that affects every swap spec that ever runs `checkExportedFileContents`, so reviewers will want to inspect it alone; the fixture change is reusable for future SPL/BSC scenarios; the spec is the coverage delivery. If the team's norm is one commit per PR (check recent merged PRs under `e2e/mobile/`), squash on merge.

Conventional-Commit shape is per `.claude/rules/git-workflow.md`: `<type>(scope): <imperative description>`. Scope is `mobile` (matching the `e2e/mobile/` root). Type is `test` — we are strictly adding coverage; no product behaviour moves.

### 5.10.14 Open the PR with `/create-pr`

From the repo root:

```
/create-pr
```

Answer the prompts:

1. **Ticket URL** — `https://ledgerhq.atlassian.net/browse/QAA-702`
2. **Ticket description** — "Add step 2 of B2CQA-604: swap history export with ERC20 (USDT) on Receive side. Closes LIVE-19533 regression hole."
3. **Change type** — `test`
4. **Change scope** — `e2e/mobile`
5. **Test coverage** — `yes`
6. **QA focus areas** — "Swap history export, ERC20 receive, CSV artifact assertion on `TokenAccount.accountName` / `currency.ticker`"
7. **UI changes** — `no`

Reviewers to request (CODEOWNERS will add `@ledgerhq/wallet-xp` automatically; add the PTX/Swap owner by hand):

- `@ledgerhq/wallet-xp` — `e2e/mobile/` owners
- `@ledgerhq/ptx` or the Swap automation sub-team — fixture owners (surface the `swap-history.json` diff in the PR description so they review it deliberately)

After CI is green and approvals land:

1. Click **Ready for review** on the draft
2. Merge into `develop`
3. Move `QAA-702` to **Done** and comment "Step 2 of B2CQA-604 automated in PR #XXXX, spec `swapExportHistoryOperationsERC20.spec.ts`"
4. In Xray, **B2CQA-604 already reads as Automated** (step 1 did that). If there is a "scenario steps covered" tracking field, mark step 2 as covered. Otherwise no Xray field changes — both steps share the same card.

### 5.10.15 Reference — Extending to Other ERC20 Receive Currencies

Once this template is green, adding USDC, DAI, or LINK is mechanical. Each new scenario requires **the same four-phase loop**: check fixture (Phase 1) → execute real swap on device (Phase 2) → import JSON block (Phase 3) → spec + `Addresses` entry (Phase 4).

- **USDC.** Replace `TokenAccount.ETH_USDT_1` with `TokenAccount.ETH_USDC_1` (Account.ts:310, accountName `"USD Coin 1"`, `Currency.ETH_USDC`). Execute a fresh SOL → USDC swap on the QAA seed, capture the new `swapId` + addresses, append two new `Addresses` entries (`SWAP_HISTORY_ERC20_USDC_SOL_FROM`, `SWAP_HISTORY_ERC20_ETH_USDC_TO`), add the account block to `swap-history.json`, and name the spec `swapExportHistoryOperationsERC20_USDC.spec.ts`.
- **DAI, LINK, MATIC-on-mainnet.** Same pattern. If the `TokenAccount` enum does not have the entry you need, add it to `libs/ledger-live-common/src/e2e/enum/Account.ts` in a **separate PR** first — the enum file is shared with desktop and wants to be reviewed in isolation.
- **SPL / Solana tokens (e.g. ETH → GIGA).** Different family, but the driver does not care. The constraint is fixture content: SPL ops have `tokenId` in a different namespace and the `receiverAccountId` suffix is Solana-specific. Phases 1-3 stay identical; Phase 4 uses `TokenAccount.SOL_GIGA_1` (or equivalent) and Solana-side `Addresses` constants.

If the team later asks for a dedicated B2CQA per currency (e.g. `B2CQA-9999 — Swap history export, USDC receive`), flip `tmsLinks` to the new ID. Until then, reuse `B2CQA-604` — that is the commitment recorded in Pavlo's comment.

### 5.10.16 Reference — Why `checkExportedFileContents` Needed the ERC20 Branch

Phase 4 Step 2 changed a driver shared by every swap export spec. The change deserves a short memo so future readers understand why step 1 did not need it.

**What Ledger Live's CSV exporter actually writes.** When you export swap history, the CSV's "From account" / "To account" columns render the **account display label as shown in the wallet UI**. For a native account (ETH, SOL, BTC), that label is the account's own `accountName` — `"Ethereum 1"`, `"Solana 1"`. For an ERC20 TokenAccount (`Account.ETH_USDT_1`, `Account.ETH_USDC_1`), the wallet UI shows the **parent** ETH account's name because the token sub-account is mounted under it — "Ethereum 1" holds both the native ETH balance and every ERC20 balance at that address. The CSV preserves that hierarchy: ERC20 rows export `"Ethereum 1"`, not `"Tether USD 1"`.

**Why step 1 passed without the branch.** Step 1 swaps SOL → native ETH. `swap.accountToCredit` is `Account.ETH_1`, not a TokenAccount. Its `accountName` is already `"Ethereum 1"`. `tokenType` is absent (falsy), so the new `if` branch is skipped and the assertion runs unchanged on `swap.accountToCredit.accountName`. Backward compatibility is free.

**Why QAA-702 would fail without the branch.** In the ERC20 case, `swap.accountToCredit` is `TokenAccount.ETH_USDT_1` with `accountName = "Tether USD 1"`. The CSV never contains that string — it contains `"Ethereum 1"`. Without the parent-name fallback, `toContain("Tether USD 1")` fails on the very first assertion; the driver cannot even describe what the CSV actually says.

**Two guards, not one.** The condition is `tokenType === TokenType.ERC20 && parentAccount`. Both checks matter:

- `tokenType === TokenType.ERC20` narrows to the CSV-formatting case. If Ledger Live ever exports SPL (Solana) or other token families with a different CSV rule, this branch must be extended, not reused.
- `parentAccount` is a defensive null-check. A malformed enum entry (`tokenType` set, `parentAccount` missing) would otherwise throw on `parentAccount.accountName`.

**What it does not cover.** The symmetric case — a swap where the DEBIT side is an ERC20 — is also handled by Step 2's code (the `accountToDebit` branch). No ERC20-debit spec exists yet in the mobile suite; when one lands (e.g., QAA-715: "USDT → BTC from-ERC20 swap history"), this branch is the reason it will not need a second driver change.

**What it does not patch.** `currency.ticker`, `currency.address`, `provider.name`, `swapId`, and `swap.amount` were already correct for every account type. Those assertions are untouched.

> **If a future CSV format change breaks this assumption,** revisit the branch. The truth is what the CSV writes, not what our enum holds — always read the artifact first (`e2e/mobile/artifacts/ledgerwallet-swap-history.csv`) before blaming the spec.

### 5.10.17 PR Checklist

- [ ] New spec file compiles and runs in isolation
- [ ] `swap-history.json` validates under `jq .`
- [ ] Three consecutive green runs on iOS (and Android if your matrix covers it)
- [ ] Allure report shows `B2CQA-604` link and three `@Step` entries
- [ ] Commit messages follow `test(mobile): ...` convention
- [ ] PR body includes the Allure step-tree screenshot and the fixture-diff highlight
- [ ] Reviewers: `@ledgerhq/wallet-xp` + Swap-automation owner
- [ ] QAA-702 linked in the PR description
- [ ] `libs/ledger-live-common/src/e2e/enum/Addresses.ts` has the two new `SWAP_HISTORY_ERC20_*` constants and nothing else was moved
- [ ] `e2e/mobile/page/trade/swap.page.ts` `checkExportedFileContents` has the ERC20 parent-name fallback (Phase 4 Step 2) and the step-1 spec still passes after the driver change
- [ ] No changes to `swap.other.ts`

<div class="chapter-outro">
<strong>Key takeaway.</strong> QAA-702 is a surgical coverage addition, not a rewrite. The driver is already ERC20-safe; the page object's assertions already read ticker/accountName/address from any <code>Account | TokenAccount</code>; only two things were genuinely missing — a spec file declaring the USDT receive scenario, and a matching entry in the <code>swap-history</code> userdata fixture. The work that carries weight is the <em>diligence</em>: reading the driver first, inspecting the fixture to know whether you are adding or reusing, matching the <code>swapId</code> and <code>provider</code> fields byte-for-byte, and running three times before opening the PR. That is the loop every real QAA ticket rewards.
</div>

### 5.10.18 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — QAA-702 Walkthrough</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> Why does the new spec for QAA-702 reuse <code>tmsLinks: ["B2CQA-604"]</code> instead of opening a new B2CQA case?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Xray forbids creating new B2CQA cases from the mobile repo</button>
<button class="quiz-choice" data-value="B">B) The driver hard-codes that ID</button>
<button class="quiz-choice" data-value="C">C) Pavlo OKHONKO's comment on B2CQA-604 explicitly says step 2 (the ERC20 receive) remains inside the same test case until QAA-702 automates it; both steps map to one card</button>
<button class="quiz-choice" data-value="D">D) Because step 1 failed and is being retired</button>
</div>
<p class="quiz-explanation">The test case ID is the contract between Xray and the suite. Pavlo's comment is the single source of truth that the ERC20 scenario belongs under <code>B2CQA-604</code> as step 2 — both specs will carry the same <code>tmsLinks</code> entry.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> Why do we add <code>SWAP_HISTORY_ERC20_SOL_FROM</code> and <code>SWAP_HISTORY_ERC20_ETH_USDT_TO</code> to <code>Addresses.ts</code> instead of reusing the existing <code>SWAP_HISTORY_SOL_FROM</code> / <code>SWAP_HISTORY_ETH_TO</code> constants?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The existing constants are frozen</button>
<button class="quiz-choice" data-value="B">B) ERC20 tokens require a dedicated non-parent address</button>
<button class="quiz-choice" data-value="C">C) The Phase-2 swap was executed on the QAA device seed, which has different Solana and Ethereum holders than the step-1 synthetic fixture seed. The new spec's <code>addressFrom</code> / <code>addressTo</code> must match the holders that actually produced the CSV row the spec asserts against</button>
<button class="quiz-choice" data-value="D">D) Detox forbids reusing enum members across specs</button>
</div>
<p class="quiz-explanation">The driver writes <code>swap.accountToDebit.address = addressFrom</code> / <code>accountToCredit.address = addressTo</code> before calling <code>checkExportedFileContents</code>, which asserts those addresses appear in the exported CSV. If we reused step 1's constants the assertion would fail — the CSV was produced by a different seed and therefore contains different holder addresses. New capture → new constants.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q3.</strong> You load <code>swap-history.json</code> into Ledger Live Desktop (Phase 1) and the Swap → History screen shows only two SOL → native-ETH entries, no ERC20 receive. What is the correct next step?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Hand-craft a synthetic ERC20 entry in the JSON using dummy values</button>
<button class="quiz-choice" data-value="B">B) Execute a real SOL → USDT swap on the QAA device (Phase 2), export the CSV, extract <code>swapId</code>/<code>addressFrom</code>/<code>addressTo</code>, then graft the new account block from your local <code>app.json</code> into the fixture (Phase 3)</button>
<button class="quiz-choice" data-value="C">C) Write the spec anyway; the fixture will catch up</button>
<button class="quiz-choice" data-value="D">D) Skip the ticket — the fixture is not ready</button>
</div>
<p class="quiz-explanation">Fixture entries must describe real, verifiable swaps. Synthesising fake <code>swapId</code>/<code>operationId</code> values leads to drift between the fixture and live-common's account-ID encoding. The 4-phase workflow produces an authentic entry and guarantees all three identifiers (swap ID, from address, to address) come from the same observation.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> The driver calls <code>jestExpect(fileContents).toContain(swap.amount)</code>. You pass <code>"0.07"</code> (comma) to the <code>Swap</code> constructor via <code>solMinAmount</code>. The fixture's <code>fromAmount</code> is <code>"69637880"</code>. Why is the assertion still correct?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The CSV export is human-readable in display units (SOL) using the running app's locale — the French locale renders <code>0,07 SOL</code>. <code>fromAmount</code>'s integer lamports are live-common's internal storage. The assertion reads the CSV, not the fixture</button>
<button class="quiz-choice" data-value="B">B) <code>toContain</code> silently coerces numeric types</button>
<button class="quiz-choice" data-value="C">C) The driver divides by 10^9 before asserting</button>
<button class="quiz-choice" data-value="D">D) The assertion is known-broken but nobody has fixed it</button>
</div>
<p class="quiz-explanation">Internal units (lamports) and display units (SOL) are different representations of the same amount. The CSV shows display units with the locale's decimal separator; that is what <code>"0.07"</code> matches on a French-locale build. The fixture's <code>fromAmount: "69637880"</code> is 0.069637880 SOL at 9 decimals — within rounding of the 0,07 the user typed, and stored separately from what the CSV shows.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> Your first run is red: <code>toContain("Tether USD 1")</code> fails on the exported CSV. What is the correct first investigation?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Patch the driver to use <code>currency.name</code> instead of <code>accountName</code></button>
<button class="quiz-choice" data-value="B">B) Open the CSV in <code>e2e/mobile/artifacts/ledgerwallet-swap-history.csv</code> and read what the app actually wrote; reconcile with <code>TokenAccount.ETH_USDT_1.accountName</code> — the mismatch is data, not code</button>
<button class="quiz-choice" data-value="C">C) Delete the fixture entry</button>
<button class="quiz-choice" data-value="D">D) Change <code>Swap</code>'s constructor</button>
</div>
<p class="quiz-explanation">A red assertion is evidence, not a verdict. Read the artifact; compare it to the expected substring. In QAA-702 the CSV turns out to contain <code>"Ethereum 1"</code> (the parent account's name) rather than <code>"Tether USD 1"</code> — a CSV-formatting behaviour the driver did not yet account for, which is exactly why Phase 4 Step 2 adds the ERC20 parent-name fallback to <code>checkExportedFileContents</code>. Sometimes the mismatch is a fixture field, sometimes it is a driver contract the product has changed under you. The investigation rule is the same either way: read the artifact first, decide second.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> A reviewer asks: "why not modify step 1's spec to iterate over both native and ERC20 receive, instead of creating a second file?" What is the best defense?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Jest forbids parameterised <code>it</code> blocks</button>
<button class="quiz-choice" data-value="B">B) The driver is read-only</button>
<button class="quiz-choice" data-value="C">C) The thin-spec + shared-driver pattern keeps each spec a flat data bundle; one spec = one scenario = one Allure test case. Folding two receive types into one file would hide failures behind a shared <code>beforeAll</code>, collapse two Allure cases into one, and break the one-file-per-pair convention established by the existing sibling specs</button>
<button class="quiz-choice" data-value="D">D) The driver's <code>describe</code> block does not accept loops</button>
</div>
<p class="quiz-explanation">The repo's swap specs are deliberately one-scenario-per-file. Two separate specs means two independent Detox runs, two Allure cases, two CI artifacts, two failure signals. Iteration inside the spec would save two lines of code and cost every one of those benefits.</p>
</div>

<div class="quiz-score"></div>
</div>

---


## Mobile Exercises & Challenges

<div class="chapter-intro">
Reading and watching are not enough. The exercises below move from minor POM additions to CI-level changes and end with a simulator-warming wrapper that will speed up your daily loop. Do them in order — each one builds a skill you will need in the next. Every exercise has a verification step; if you cannot verify it yourself, grab a reviewer before moving on.
</div>

### 5.11.1 Exercise 1: Add a `@Step` for the Empty State (15 min)

**Objective.** Practice adding a decorated assertion to an existing POM.

**Instructions.**
1. Open `e2e/mobile/page/trade/swap.page.ts` (the POM you wrote in Chapter 5.10 — or a draft of it).
2. Add a selector for the history empty-state title: `swap-history-empty-title`.
3. Add a method `expectEmptyState()` decorated with `@Step("Expect swap history to be empty")` that asserts the empty-state title is visible.
4. Make sure the method is `public async`.

**Verification.** Run `pnpm mobile typecheck`. No type errors. Open the POM side by side with `swap.page.ts`; the signature should match the other `expect*` methods.

**Hints.**
- Copy the shape of `expectOperationRowVisible(swapId)`, drop the parameter.
- The empty-state subcomponent is `apps/ledger-live-mobile/src/screens/Swap/History/EmptyState.tsx`; open it to confirm the testID before adding it to the POM.

**Stretch goal.** Add a second `@Step("Expect export button to be hidden")` that uses `toBeNotVisible()` on `export-swap-operations-link`.

### 5.11.2 Exercise 2: A Modal Close Spec (30 min)

**Objective.** Write a minimal end-to-end spec that re-uses an existing POM.

**Instructions.**
1. Locate `e2e/mobile/page/passwordEntry.page.ts` (or its equivalent — search with `rg "passwordEntry"`).
2. Write a spec `e2e/specs/common/passwordEntryClose.spec.ts` with a single `it()`:
   - `beforeAll` → `app.init({ userdata: "skipOnboardingWithPasswordLock" })` — if this fixture does not exist, use the closest one that boots the app at the lock screen.
   - Assert the password-entry modal is visible.
   - Tap the existing Close button via the POM's `tapCloseButton()`.
   - Assert the modal is gone.
3. Tag the spec with an arbitrary `$TmsLink("B2CQA-DUMMY")` — the point is the wiring, not the Xray link.

**Verification.** Run `pnpm test:ios:debug -- -t "password entry close"`. Three runs, all green.

**Hints.**
- If `skipOnboardingWithPasswordLock` does not exist, check `userdata/` for a "locked" variant; if none, discuss with your lead before inventing one.
- The Close button may raise a confirmation. Use the POM's confirm method if so.

**Stretch goal.** Add a second `it()` that, instead of tapping Close, types the wrong password three times and asserts the lockout message appears.

### 5.11.3 Exercise 3: Android API-Level Matrix Entry (45 min)

**Objective.** Add a second Android configuration to `detox.config.js` and invoke it locally.

**Instructions.**
1. Open `apps/ledger-live-mobile/detox.config.js`.
2. Duplicate the existing `android.emu.debug` block into a new config `android.emu.api34.debug` that targets an AVD named `Pixel_7_API_34`.
3. Create the AVD if it does not exist (`avdmanager create avd …` or via Android Studio).
4. Build with `pnpm mobile e2e:build -c android.emu.api34.debug`.
5. Run one of the small specs you wrote in exercises 1–2 under the new config.

**Verification.** The spec boots on an API-34 emulator. Visible in `emulator -list-avds`.

**Hints.**
- The existing `android.emu.debug` block is your template; change only `device.avdName`.
- API 34 introduces stricter foreground-service policy — if the app fails to boot, you likely need to adjust `device.headless` or `device.utilBinaryPaths`.

**Stretch goal.** Add the new config to a nightly CI job by appending it to the matrix in `.github/workflows/test-mobile-e2e-reusable.yml`. Do not merge without your lead's sign-off — nightly minutes are not free.

### 5.11.4 Exercise 4: Multi-Account Userdata + Accounts List Spec (60 min)

**Objective.** Build a non-trivial userdata fixture and assert that the app renders it correctly.

**Instructions.**
1. Create `e2e/userdata/twoEthTwoUsdt.json` — two Ethereum parent accounts (`Ethereum 1`, `Ethereum 2`), each with a USDT TokenAccount child (balances: `100 USDT`, `250 USDT`).
2. Write `e2e/specs/accounts/listTwoEthTwoUsdt.spec.ts` that:
   - Boots with the new userdata.
   - Opens the Accounts tab via `app.accounts.openViaDeeplink()`.
   - Asserts four rows are visible: `Ethereum 1`, `Ethereum 2`, and the two USDT child rows.
3. Tag with `$TmsLink("B2CQA-DUMMY2")`.

**Verification.** Three green runs. Open Allure and confirm each assertion is a separate `@Step`.

**Hints.**
- Use distinct, collision-free account IDs — copy the ID pattern from an existing userdata fixture.
- The Accounts list uses `account-row-${id}` as its testID pattern (confirm by grepping `src/screens/Accounts/`).

**Stretch goal.** Extend with a third Ethereum account that has an ERC721 NFT sub-account, and assert the NFT row renders distinctly.

### 5.11.5 Exercise 5: Feature-Flag Override via Bridge (90 min)

**Objective.** Use the bridge's `overrideFeatureFlag` (or equivalent) to flip a flag in `beforeAll` and assert the UI changes.

**Instructions.**
1. Locate the bridge message responsible for setting a feature flag in `e2e/bridge/types.ts`.
2. In a new spec `swapLiveAppFlagOn.spec.ts`, in `beforeAll`, send `overrideFeatureFlag("ptxSwapLiveAppMobile", { enabled: true })` to the app before it navigates.
3. In the `it()`, open the swap tab and assert the Live App route is mounted (look for a testID like `swap-live-app-root` — if it does not exist, add the product ticket and stop; do not hack around it).

**Verification.** Three green runs. Flip the flag to `false` and rerun; the test should fail as expected.

**Hints.**
- The bridge `overrideFeatureFlag` contract is documented in Chapter 5.6. If the payload shape is wrong, the app ignores the message silently — attach a `onerror` logger while iterating.
- Not every flag propagates at runtime without an app restart. Check `FeatureFlagsProvider` in `apps/ledger-live-mobile/src/components/FirebaseFeatureFlags.tsx`.

**Stretch goal.** Write a helper `withFeatureFlags(flags) { ... }` that accepts an object and sends one bridge message per flag. Use it in your `beforeAll`.

### 5.11.6 Exercise 6: Nightly Swap-Only CI Matrix (90 min)

**Objective.** Add a scheduled CI entry that runs only swap-tagged specs.

**Instructions.**
1. Open `.github/workflows/test-mobile-e2e-reusable.yml` and its caller (likely `test-mobile-e2e-reusable.yml`).
2. Add a `schedule` trigger firing at `0 2 * * *` (02:00 UTC).
3. Add a matrix entry that passes `-t "SWAP|Swap|swap"` to `pnpm test:ios:debug` — this filters by test-name pattern.
4. Limit to one shard (swap-only run — no need to parallelize across 10 shards for a dozen tests).

**Verification.** Run `act` locally or push to a throwaway branch and wait for the scheduled run. Confirm only swap specs execute.

**Hints.**
- Use a `if: github.event_name == 'schedule'` guard so the matrix entry does not run on every PR.
- The sharding algorithm (Chapter 8.3) is irrelevant here — you are opting out of sharding by setting shard count to 1.

**Stretch goal.** Add a Slack notification step that posts the Allure link to `#qa-mobile` on failure.

### 5.11.7 Exercise 7: Extend QAA-702 with an Empty-State Path (120 min)

**Objective.** Add a second `it()` to `swapHistoryExportERC20.spec.ts` that covers the no-history case.

**Instructions.**
1. Create a userdata fixture `swapHistoryEmpty.json` with at least one Ethereum account but **no** entries in `swap.history`.
2. Add a new `describe("Swap History — Empty")` block with a single `it($TmsLink("B2CQA-604-empty"), "hides the export button when no swaps exist")`.
3. Use the `expectEmptyState()` method from Exercise 1.
4. Assert the export button is not visible.

**Verification.** Both specs — the ERC20 export spec and the empty-state spec — pass on three consecutive runs.

**Hints.**
- `beforeAll` in the new `describe` re-inits the app with the new userdata.
- Detox's `toBeNotVisible()` is different from `toBeHidden()`. Check which one exists in your Detox major version.

**Stretch goal.** Write a helper `seededSwapHistory(opts: { count: number; erc20: boolean })` that generates the JSON in-memory and hands it to `app.init()`. This removes two userdata files in favor of one generator.

### 5.11.8 Challenge: Warm-Simulator Detox Wrapper (120 min, stretch)

**Objective.** Cut local iteration time in half by keeping the simulator booted between runs.

**Task.** Write a small Node or shell wrapper — `scripts/e2e-warm.sh` or `scripts/e2e-warm.ts` — that:

1. Checks whether the iOS simulator UUID is already booted (`xcrun simctl list | rg Booted`).
2. If not, boots it.
3. Runs `detox test --reuse -c ios.sim.debug -t "$@"` — the `--reuse` flag reuses an already-installed app binary.
4. Does **not** shut down the simulator on exit.
5. Accepts the same `-t` pattern and `-c` config as `pnpm test:ios:debug`.

The existing pnpm command rebuilds and rebooks every time; `--reuse` is up to 30 s faster per run.

**Verification.** Three consecutive runs of a small spec (Exercise 2's password-entry close) should each start the test in under 10 s of wall time after the first.

**Hints.**
- `detox test --reuse` is documented in the Detox CLI reference. If the binary isn't installed, `--reuse` falls back gracefully.
- `xcrun simctl boot <UDID>` is the lowest-level way to boot a sim; `applesimutils --booted` lets you query the currently booted sim without parsing.
- Print the elapsed time at the end of the wrapper so you can track your speedup.

**Stretch.** Add an Android branch that checks `adb devices` and only boots an emulator if none is running.

<div class="chapter-outro">
<strong>Key takeaway.</strong> The exercises ramp from touching a POM to rewriting your local loop. Exercises 1–2 are warm-ups; 3–4 build the fixture and config intuition; 5–6 move into infrastructure; 7 extends your capstone; 8 is the quality-of-life payoff. If you finish all eight without peeking at solutions, you are past the onboarding bar and into steady-state contribution territory.
</div>

---


---

## Part 5 Final Assessment

You have walked the full mobile E2E stack: React Native primitives, the iOS/Android toolchain, Detox's matcher-action-expectation DSL, the WebSocket bridge that layers Redux and device-event control on top of Detox, the POM hub, the sharding algorithm, and one real ticket (QAA-702) shipped end to end. This assessment samples the load-bearing ideas across Chapters 5.1-5.11. Eighty percent to pass. If you miss a question, jump back to the referenced chapter — the answer is always grounded in a concrete file or command, not general knowledge.

<a id="part-4-final-assessment"></a>

<div class="quiz-container" data-pass-threshold="80">
<h3>Part 5 Final Assessment</h3>
<p class="quiz-subtitle">10 questions · 80% to pass · Covers Chapters 5.1-5.11</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> In React Native, what does the <code>testID</code> prop become at runtime on each native platform?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A CSS class on iOS and an XML id on Android</button>
<button class="quiz-choice" data-value="B">B) The visible text label on both platforms</button>
<button class="quiz-choice" data-value="C">C) <code>accessibilityIdentifier</code> on the backing <code>UIView</code> on iOS, and <code>resource-id</code> on the <code>View</code> on Android — which is what <code>by.id()</code> resolves against</button>
<button class="quiz-choice" data-value="D">D) A React key used only by the reconciler</button>
</div>
<p class="quiz-explanation">RN translates <code>testID</code> to the platform's accessibility identifier. Detox's <code>by.id("export-swap-operations-link")</code> matches <code>accessibilityIdentifier</code> on iOS and <code>resource-id</code> on Android. This is why <code>testID</code> is stable across i18n and copy changes — the value never appears on screen.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> Why do E2E tests in Ledger Live Mobile often branch on <code>device.getPlatform()</code> (exposed via <code>isIos()</code>/<code>isAndroid()</code> helpers) rather than on <code>Platform.OS</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>Platform.OS</code> is deprecated</button>
<button class="quiz-choice" data-value="B">B) <code>Platform.OS</code> runs inside the app bundle, but the Detox test code runs in the Jest worker (Node) — it has no <code>Platform</code>, so it asks the device driver instead</button>
<button class="quiz-choice" data-value="C">C) Detox forbids importing from <code>react-native</code></button>
<button class="quiz-choice" data-value="D">D) Both helpers are aliases for the same thing</button>
</div>
<p class="quiz-explanation"><code>Platform</code> is a React Native module — only defined inside the RN runtime. Test code runs in Node, so it queries the Detox <code>device</code> global. <code>device.getPlatform()</code> returns <code>"ios"</code> or <code>"android"</code> and is the only reliable way to branch from the test side.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> The repo pins Node, pnpm, npm, bun, and CocoaPods via <code>mise.toml</code> (and a minimal <code>.prototools</code>). What does running <code>mise install</code> at the repo root accomplish?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Reads the pinned versions, pulls them down, and places shims on <code>PATH</code> so every tool auto-switches when you <code>cd</code> into the repo</button>
<button class="quiz-choice" data-value="B">B) Installs the Ledger Live mobile APK onto an attached Android device</button>
<button class="quiz-choice" data-value="C">C) Runs <code>pod install</code> for the iOS project</button>
<button class="quiz-choice" data-value="D">D) Boots an iOS simulator and an Android emulator</button>
</div>
<p class="quiz-explanation"><code>mise</code> (successor to <code>proto</code>) is the single source of truth for pinned tool versions. You never touch <code>nvm use</code> or <code>rbenv global</code> — shims on <code>PATH</code> resolve to the exact versions declared in <code>mise.toml</code> as soon as you enter the directory.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> You run an E2E build and the app crashes on launch with "Firebase config missing." What is the most likely cause?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) CocoaPods is out of date — run <code>bundle exec pod install</code></button>
<button class="quiz-choice" data-value="B">B) Metro is not running on port 8081</button>
<button class="quiz-choice" data-value="C">C) The iOS simulator was cold-booted without <code>-no-snapshot</code></button>
<button class="quiz-choice" data-value="D">D) <code>ENVFILE</code> was not set, so the build did not inject <code>.env.mock</code> (staging Firebase) or <code>.env.mock.prerelease</code> (production Firebase)</button>
</div>
<p class="quiz-explanation">The build scripts read <code>ENVFILE</code> and inject it into the native build so JS and native code see the same values. Without it, Firebase keys are absent and the app crashes immediately. Default for local E2E and CI is <code>ENVFILE=.env.mock pnpm mobile e2e:build -c ios.sim.debug</code>.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> Why does Detox's <code>setup.ts</code> call <code>await device.reverseTcpPort(8081)</code> and <code>await device.reverseTcpPort(wsPort)</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) To harden the Metro bundler against MITM attacks</button>
<button class="quiz-choice" data-value="B">B) Because Android emulators run in an isolated network namespace — without <code>adb reverse tcp:p tcp:p</code>, <code>localhost:p</code> inside the emulator does not reach the host Mac where Metro and the bridge server run</button>
<button class="quiz-choice" data-value="C">C) Detox requires reverse-port-forwarding on both iOS and Android to boot the simulator</button>
<button class="quiz-choice" data-value="D">D) It is a Jest configuration step unrelated to the app</button>
</div>
<p class="quiz-explanation">iOS simulator shares the Mac's network stack, so <code>localhost</code> already works and <code>reverseTcpPort</code> is a harmless no-op. Android emulator has its own kernel and network — <code>adb reverse</code> is the magic that routes <code>localhost:8081</code> (Metro) and the dynamic bridge port to the host.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> In Detox, which line correctly expresses the matcher-action-expectation triad for "tap the Send button by testID, then assert the amount field is visible"?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>element.click("send-button"); expect.visible("amount-input")</code></button>
<button class="quiz-choice" data-value="B">B) <code>await by.id("send-button").tap(); await by.id("amount-input").toBeVisible()</code></button>
<button class="quiz-choice" data-value="C">C) <code>await element(by.id("send-button")).tap(); await expect(element(by.id("amount-input"))).toBeVisible()</code></button>
<button class="quiz-choice" data-value="D">D) <code>device.tap("send-button"); device.assertVisible("amount-input")</code></button>
</div>
<p class="quiz-explanation">Every Detox statement composes a matcher (<code>by.*</code>) through <code>element()</code> with either an action (<code>.tap()</code>) or an expectation (<code>expect(...).toBeVisible()</code>). Matchers are pure descriptions — they do not find anything until <code>element()</code> resolves them.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q7.</strong> The Ledger Live Mobile WebSocket bridge server (Node, Jest worker) and client (inside the RN app) every message round-trips with an <code>ACK</code>. Why?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Without ACKs, <code>await setFeatureFlag(...)</code> would resolve as soon as the bytes left the socket — before the RN client ran the Redux dispatch. The next test line would race the state change. The client only sends <code>{ type: "ACK", id }</code> after running the handler, so the server's promise resolves only once the side effect is live</button>
<button class="quiz-choice" data-value="B">B) WebSockets require application-level ACKs by the protocol specification</button>
<button class="quiz-choice" data-value="C">C) ACKs are used to measure round-trip latency for Allure reports</button>
<button class="quiz-choice" data-value="D">D) They allow Detox to synchronize with Metro during reloads</button>
</div>
<p class="quiz-explanation">The server's <code>postMessageAndWaitForAck</code> stashes the promise in <code>pendingAcks</code> keyed by message id; the client's <code>onMessage</code> always ends with <code>postMessage({ type: "ACK", id: msg.id })</code> after dispatching the Redux action, emitting the native event, or calling <code>navigate()</code>. This is what makes the bridge safely awaitable.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q8.</strong> What role does the WebSocket bridge play that Detox alone cannot cover?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It replaces the Metro bundler during E2E</button>
<button class="quiz-choice" data-value="B">B) It is the transport Detox uses to drive UI matchers</button>
<button class="quiz-choice" data-value="C">C) It replaces the React Native <code>Bridge</code> at runtime</button>
<button class="quiz-choice" data-value="D">D) It exchanges typed messages (<code>importAccounts</code>, <code>overrideFeatureFlag</code>, <code>mockDeviceEvent</code>, <code>sendFile</code>, <code>addKnownSpeculos</code>, etc.) so tests can seed Redux, override feature flags, emit native device events, and receive exported files — things Detox, which only drives UI, cannot do</button>
</div>
<p class="quiz-explanation">Detox drives the UI; the bridge gives the test state-level control. Onboarding from scratch every run would take two minutes plus BLE pairing. Feature flags must be overridden without a real Firebase fetch. Device events (USB, APDU) must be mocked. The bridge handles all of that via a small typed protocol declared in <code>bridge/types.ts</code>.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q9.</strong> In QAA-702, the spec registers the <code>sendFile</code> bridge listener <em>before</em> tapping the Export button, and the app's <code>exportSwapHistory()</code> branches on <code>getEnv("DETOX")</code>. Why both details?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because Detox disables the share sheet globally, and bridge listeners are rate-limited</button>
<button class="quiz-choice" data-value="B">B) Because the underlying <code>EventEmitter</code> emits <code>sendFile</code> synchronously when the button is tapped — a late listener would miss the event — and because Detox cannot dismiss the native share sheet, the app intentionally skips <code>Share.open</code> and calls <code>sendFile({ fileName: "ledgerwallet-swap-history.csv", fileContent })</code> when <code>DETOX=true</code></button>
<button class="quiz-choice" data-value="C">C) So the CSV is written to the simulator's filesystem and read back by Allure</button>
<button class="quiz-choice" data-value="D">D) Because <code>toAccount.type === "TokenAccount"</code> blocks the listener on Android</button>
</div>
<p class="quiz-explanation">Two cooperating decisions: the app bypasses <code>Share.open</code> in Detox mode and emits a bridge event instead; the bridge client uses a plain <code>EventEmitter</code>, so listeners attached after the synchronous emit see nothing. The test assembles the CSV promise (via <code>awaitExportedCSV</code>) before <code>tapExportButton()</code> and then awaits it. CSV structural checks then assert that "To account" and "To account address" hold the parent Ethereum account name and fresh address, because for ERC20 receives <code>getMainAccount</code> returns the parent.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q10.</strong> Mobile CI sharding differs from desktop in two specific ways. Which pair of statements is correct?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Mobile shards round-robin specs by filename; desktop greedily bin-packs by duration</button>
<button class="quiz-choice" data-value="B">B) Mobile uses a single 30-shard cap across both platforms; desktop uses no cap</button>
<button class="quiz-choice" data-value="C">C) Mobile splits specs at the runner (job) level — each shard has its own OS, simulator/emulator, and bridge port — and applies asymmetric caps (up to 3 iOS / 12 Android on regular branches; 12/12 on scheduled or release/hotfix refs) because macOS runners are scarce while Linux runners are cheap. The <code>shard-tests.mjs</code> algorithm is greedy bin-packing by historical duration, with zero-timing specs round-robin'd</button>
<button class="quiz-choice" data-value="D">D) Mobile runs all shards sequentially to avoid simulator contention</button>
</div>
<p class="quiz-explanation">Two distinctive mobile patterns: (1) sharding is at the runner level, not the Jest-worker level — <code>maxWorkers: 1</code> stays in force per shard — and (2) the iOS/Android cap asymmetry reflects runner economics. The greedy longest-first bin-packing minimizes max-shard wall time; naive round-robin would leave one shard running 4x longer than its siblings. New (zero-timing) specs are round-robin'd on first appearance and picked up by the algorithm on the next run.</p>
</div>

<div class="quiz-score"></div>
</div>

<div class="chapter-outro">
<strong>Part 5 complete.</strong> You own the mobile E2E stack end to end: React Native primitives (<code>View</code>, <code>testID</code>, <code>Platform</code>, <code>Linking</code>), the <code>mise</code>/CocoaPods/<code>adb</code>/<code>ENVFILE</code> toolchain, the Detox matcher-action-expectation DSL, the WebSocket bridge with its ACK protocol, <code>device.reverseTcpPort</code> as the Android glue, the POM hub and <code>@Step</code> decorators, the sharding algorithm with its asymmetric iOS/Android caps, the daily workflow, and one ticket shipped (QAA-702) with a bridge-delivered CSV assertion. <strong>Next: Part 6</strong> steps off the GUI altogether and onto the command line — the <strong>wallet-cli</strong> workspace that powers test-data hooks like the QAA-615 token-approval revoke we'll need before QAA-613's nightly tests can stay deterministic. Same Bun runtime, same DMK transport, same Bunli command shape — but no UI to drive. Then Part 7 picks up the Swap Live App that exercises everything you've just learned.
</div>
