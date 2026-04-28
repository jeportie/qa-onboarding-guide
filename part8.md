## Test Strategy & Best Practices

<div class="chapter-intro">
Knowing how to write a test is not enough — you need to know <strong>which</strong> tests to write, <strong>how</strong> to structure them, and <strong>what</strong> to do when they become flaky. This chapter covers the testing pyramid, naming conventions, page object best practices, data-driven testing, and flaky test management. These are the principles that separate a test suite that helps from one that hinders.
</div>

### 8.1.1 The Testing Pyramid at Ledger

```
            /\
           /  \
          / E2E \          ← Playwright (Desktop), Detox (Mobile)
         / with  \            Real device flows via Speculos
        / Speculos \
       /────────────\
      /  Integration  \    ← Jest + MSW + RTL
     /   Tests (API    \      API mocking, Redux state, component interaction
    /    + Component)    \
   /──────────────────────\
  /      Unit Tests         \ ← Jest
 /   Pure functions, utils,   \   Fast, isolated, no side effects
/   reducers, selectors         \
─────────────────────────────────
```

| Layer | Speed | Confidence | Cost | Quantity |
|-------|-------|-----------|------|----------|
| Unit | ~1ms/test | Low (isolated) | Cheap | Many |
| Integration | ~100ms/test | Medium | Moderate | Some |
| E2E | ~30s-5min/test | High (real flows) | Expensive | Few, critical paths |

**What goes where:**
- **Unit tests**: Currency formatters, fee calculators, transaction builders, Redux reducers, utility functions
- **Integration tests**: Screen rendering with mocked APIs, Redux store interactions, navigation flows, hook behavior
- **E2E tests**: Critical user journeys — onboarding, send/receive, portfolio, settings. Full stack including Speculos.

### 8.1.2 Test Naming Conventions

```typescript
// ✅ Good -- describes user-visible behavior
test("user can send 0.001 BTC from a funded account", async () => { ... });
test("portfolio displays correct total balance across all accounts", async () => { ... });
test("onboarding flow completes successfully with Nano S Plus", async () => { ... });

// ❌ Bad -- describes implementation or is vague
test("should call sendTransaction API", async () => { ... });
test("test portfolio page", async () => { ... });
test("it works", async () => { ... });
```

### 8.1.3 Page Object Best Practices

**Rule 1: One action = one method**
```typescript
// ✅ Granular, composable
@step("Fill recipient address")
async fillRecipient(address: string) { ... }

// ❌ Monolithic, inflexible
async doEverything(address: string, amount: string) { ... }
```

**Rule 2: Assertions belong in tests, not page objects**
```typescript
// ✅ Page object provides data, test asserts
const balance = await portfolioPage.getBalance();
expect(balance).toBe("1.5 BTC");

// ❌ Page object owns the assertion
await portfolioPage.assertBalanceIs("1.5 BTC");
```

**Rule 3: Always use `@step` decorator** for Allure reporting.

### 8.1.4 Data-Driven Testing

```typescript
const currencies = [
  { name: "Bitcoin",  ticker: "BTC", amount: "0.001",  address: "bc1q..." },
  { name: "Ethereum", ticker: "ETH", amount: "0.01",   address: "0x..." },
  { name: "Solana",   ticker: "SOL", amount: "0.1",    address: "..." },
];

for (const currency of currencies) {
  test(`can send ${currency.name}`, async ({ app }) => {
    await app.send.selectCurrency(currency.ticker);
    await app.send.fillRecipient(currency.address);
    await app.send.fillAmount(currency.amount);
    await app.send.confirm();
  });
}
```

### 8.1.5 Flaky Test Management

| Cause | Solution |
|-------|----------|
| Race conditions | Use `await expect(locator).toBeVisible()` instead of `page.waitForTimeout()` |
| Slow CI runners | Increase timeouts for CI; keep them tight locally |
| Speculos startup timing | Use health checks (`/events`) before running tests |
| Network latency | Mock external APIs or use retry logic |
| Shared state between tests | Ensure each test starts from a clean app state |

**When you encounter a flaky test:**
1. Reproduce locally (run 10 times: `--repeat-each=10`)
2. Check if it is environment-specific (CI vs local)
3. Look at the Allure timeline for timing issues
4. Fix the root cause — do not just add retries
5. If a quick fix is not possible, use `test.fixme()` and file an issue

```typescript
test.fixme("send flow times out on CI", async ({ app }) => {
  // TODO: Fix timing issue -- see JIRA LL-12345
});
```

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://martinfowler.com/bliki/TestPyramid.html">Martin Fowler: Test Pyramid</a></li>
<li><a href="https://testing.googleblog.com/2015/04/just-say-no-to-more-end-to-end-tests.html">Google Testing Blog: Just Say No to More End-to-End Tests</a></li>
<li><a href="https://martinfowler.com/bliki/PageObject.html">Martin Fowler: Page Object pattern</a></li>
<li><a href="https://playwright.dev/docs/best-practices">Playwright Best Practices</a></li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> E2E tests are expensive. Every test you write should earn its place by covering a critical path that lower-level tests cannot. When a test becomes flaky, fix the root cause — adding retries or timeouts just delays the problem.
</div>

### 8.1.6 Quiz

<!-- ── Chapter 8.1 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> Why should E2E tests be reserved for critical paths rather than testing everything?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because Playwright and Detox are unreliable tools</button>
<button class="quiz-choice" data-value="B">B) Because E2E tests are slow, expensive, and harder to maintain — they should validate what lower layers cannot</button>
<button class="quiz-choice" data-value="C">C) Because the QA team does not have CI access</button>
<button class="quiz-choice" data-value="D">D) Because unit tests already cover everything</button>
</div>
<p class="quiz-explanation">E2E tests provide the highest confidence but at the highest cost (~30s-5min per test vs ~1ms for unit). They should cover critical user journeys that cannot be adequately validated by unit or integration tests alone.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> Where should test assertions live?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) In page object methods, close to the UI interaction</button>
<button class="quiz-choice" data-value="B">B) In the test file itself — page objects provide actions and data, tests decide what to assert</button>
<button class="quiz-choice" data-value="C">C) In shared assertion utility files</button>
<button class="quiz-choice" data-value="D">D) In the fixture setup</button>
</div>
<p class="quiz-explanation">If a page object asserts <code>balance === "1.5 BTC"</code>, it can't be reused in a test that expects a different balance. Page objects should be assertion-free (except soft checks like <code>isVisible()</code>).</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> Which of the following is NOT a characteristic of a production-ready E2E test?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It has a clear, descriptive test name with tags</button>
<button class="quiz-choice" data-value="B">B) It uses page objects with <code>@step</code> decorators</button>
<button class="quiz-choice" data-value="C">C) It can run independently in any order</button>
<button class="quiz-choice" data-value="D">D) It uses <code>page.waitForTimeout(5000)</code> to ensure stability</button>
</div>
<p class="quiz-explanation"><code>waitForTimeout</code> is a hardcoded wait that makes tests slow and flaky. Production-ready tests use proper locator waits (<code>waitFor({ state: "visible" })</code>) instead of fixed delays.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q4.</strong> A test passes locally but fails 30% of the time on CI. What is the best first step?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Add <code>page.waitForTimeout(10000)</code> before the failing step</button>
<button class="quiz-choice" data-value="B">B) Mark the test as <code>test.skip()</code> and move on</button>
<button class="quiz-choice" data-value="C">C) Run the test 10 times locally with <code>--repeat-each=10</code> and compare CI vs local Allure timelines</button>
<button class="quiz-choice" data-value="D">D) Delete the test and rewrite it from scratch</button>
</div>
<p class="quiz-explanation">Reproducing the flakiness locally (10 repetitions) is the first step. Comparing CI and local timelines in Allure can reveal timing differences, race conditions, or environment-specific issues.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q5.</strong> In the testing pyramid, which layer should have the MOST tests?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Unit tests — fast, cheap, isolated</button>
<button class="quiz-choice" data-value="B">B) Integration tests — medium speed, moderate confidence</button>
<button class="quiz-choice" data-value="C">C) E2E tests — high confidence, real flows</button>
<button class="quiz-choice" data-value="D">D) Manual tests — human judgment</button>
</div>
<p class="quiz-explanation">Unit tests form the wide base of the pyramid. They are the fastest (~1ms each), cheapest to maintain, and should be the most numerous. E2E tests are the narrow tip — few, targeted at critical paths.</p>
</div>

<div class="quiz-score"></div>
</div>

---


## Debugging Failures Like a Pro

<div class="chapter-intro">
When an E2E test fails, resist the urge to immediately re-run it. This chapter teaches a systematic debugging approach: read the error, check the artifacts, reproduce locally, and fix the root cause. You will learn the Playwright Inspector, Trace Viewer, Detox debugging tools, Speculos troubleshooting, and common failure patterns with their solutions.
</div>

### 8.2.1 The Debugging Mindset

```
FAIL → Read the error → Check the screenshot/video → Reproduce locally → Fix
```

**Never:**
- Re-run without reading the error
- Add `waitForTimeout()` as a fix
- Delete a test because it's "flaky"

### 8.2.2 Common Error Patterns

```
# Timeout waiting for element
TimeoutError: locator.getByTestId("send-button").click
  Timeout 30000ms exceeded.

# Assertion failure
expect(received).toHaveText(expected)
  Expected: "1.5 BTC"
  Received: "1.50000000 BTC"

# Speculos not responding
Error: connect ECONNREFUSED 127.0.0.1:5000

# Element detached from DOM
Error: Element is not attached to the DOM
```

### 8.2.3 Playwright Debugging Tools

**Playwright Inspector** (headed debug mode):
```bash
PWDEBUG=1 pnpm e2e:desktop test:playwright your-test.spec.ts
```
Opens a browser with an inspector panel — step through actions, inspect locators live, see resolved elements. See Part 4, Chapter 4.3.8 for a detailed walkthrough with screenshots.

**Playwright Trace Viewer:**
```bash
npx playwright test --trace on
npx playwright show-trace test-results/my-test/trace.zip
```
Shows every action with before/after screenshots, network requests, console logs, and DOM snapshots.

**Playwright UI Mode:**
```bash
npx playwright test --ui
```
Interactive test runner with time-travel debugging.

**Playwright Codegen** (for exploring the app):
```bash
npx playwright codegen http://localhost:3000
```

### 8.2.4 Detox Debugging Tools

**Verbose logging:**
```bash
detox test --loglevel trace -c android.emu.debug
adb logcat | grep "ReactNative"
```

**Synchronization bypass** (for infinite animations):
```typescript
await device.disableSynchronization();
await element(by.id("animated-component")).tap();
await device.enableSynchronization();
```

### 8.2.5 Speculos Debugging

```bash
curl http://localhost:5000/events    # Health check
docker ps | grep speculos            # Container running?
docker logs <container_id>           # Container logs
```

| Symptom | Cause | Fix |
|---------|-------|-----|
| `ECONNREFUSED :5000` | Container not started | Check Docker daemon |
| `404 Not Found` on `/apdu` | Wrong Speculos version | Pull latest image |
| App stuck on home screen | Wrong coin app `.elf` | Verify COINAPPS path |
| Button press does nothing | Wrong interaction model | Check touch vs non-touch device |
| `APDU error 6985` | User rejected on device | Test didn't press "Approve" |

### 8.2.6 Common Failure Patterns

| Pattern | Symptom | Root Cause | Solution |
|---------|---------|-----------|----------|
| Timing | `TimeoutError` on existing element | Element renders late | Wait for prerequisite first |
| Stale data | Test sees old balance | Previous test pollution | Use fresh userdata |
| Layout shift | Click lands wrong | Element moved during click | Wait for animation completion |
| Locale | `"1,5 BTC"` vs `"1.5 BTC"` | System locale differs | Set locale in config |
| Resolution | Element off-screen | Different CI viewport | Set viewport in config |
| Docker | `OCI runtime create failed` | Out of disk space | `docker system prune` |

### 8.2.7 LLD Debugging Tips (from the wiki)

For desktop app debugging beyond E2E tests:

- **Renderer process**: Chrome DevTools Performance tab → record and expand User section to see React re-renders
- **Main process**: `ELECTRON_ARGS=--inspect-brk` then go to `chrome://inspect` (brk mode waits for you)
- **Internal process**: `LEDGER_INTERNAL_ARGS=--inspect`, add port manually in `chrome://inspect`

> **Tip from the wiki**: "Our goal should be to never drop a single frame in renderer thread in production mode."

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://playwright.dev/docs/debug">Playwright Debugging Guide</a></li>
<li><a href="https://playwright.dev/docs/trace-viewer">Playwright Trace Viewer</a></li>
<li><a href="https://wix.github.io/Detox/docs/troubleshooting/running-tests">Detox Troubleshooting</a></li>
<li><a href="https://developer.chrome.com/docs/devtools/">Chrome DevTools documentation</a></li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Debugging is a skill, not a talent. Follow the systematic approach: error → artifacts → reproduce → fix. The tools are excellent (Playwright Inspector, Trace Viewer, Allure) — learn to use them fluently and you will spend less time debugging than most engineers.
</div>

<!-- No quiz for Ch 7.2 — per Rodin evaluation, debugging is best learned by doing, not by QCM -->

---


## CI/CD & Test Sharding

<div class="chapter-intro">
A full Ledger Live E2E pass serialised on one machine would take the better part of an afternoon: roughly 45–60 minutes for mobile iOS alone, another 45–60 for Android, plus 30-odd for the desktop Playwright corpus against Speculos. Running that on every PR would strangle the review loop. GitHub Actions — with about 75 workflow files, a fleet of composite actions, and a greedy bin-packing scheduler — keeps that wall-clock compressed to 10–15 minutes. This chapter is the canonical reference for how CI is wired at Ledger Live: the workflow hierarchy, the <code>@Gate</code> contract that blocks your PR, the composite actions you never write but rely on every run, desktop Playwright sharding, and the full mobile <code>shard-tests.mjs</code> / <code>generate-shards-matrix</code> pipeline — ending with two ASCII sequence diagrams you can read top-to-bottom when a shard goes red.
</div>

### 8.3.1 CI at Ledger Live — GitHub Actions Anatomy

Ledger Live uses **GitHub Actions** as its sole CI/CD platform. The monorepo carries roughly **75 workflow YAMLs** under `.github/workflows/`, because it ships:

- Multiple deployable artifacts (desktop Electron app, mobile app, dozens of npm libraries under `libs/`).
- Across five OSes (Linux, macOS, Windows for desktop builds; iOS and Android for mobile).
- Under four trigger families (pull-request, merge to `develop`, nightly schedule, `workflow_dispatch`).
- With separation of concerns — linting, unit tests, E2E tests, native builds, and releases each live in their own file.

```
.github/
├── workflows/                              # ~75 workflow YAMLs
│   ├── test-ui-e2e-only-desktop.yml        # desktop Playwright (manual/scheduled)
│   ├── test-mobile-e2e-reusable.yml        # mobile Detox core (the big one)
│   ├── test-mobile-e2e-lint-reusable.yml
│   ├── test-desktop-e2e-lint-reusable.yml
│   ├── notify-e2e-required-reusable.yml    # @Gate-style aggregator
│   ├── build-desktop-*.yml
│   ├── build-mobile-*.yml
│   ├── bot-*.yml                            # Ledger Live Bot (non-reg)
│   ├── release-*.yml
│   └── …
tools/actions/composites/                    # reusable composite actions
  ├── setup-caches/
  ├── setup-e2e-test-desktop/
  ├── setup-e2e-env/
  ├── setup-speculos_image/
  ├── generate-shards-matrix/                 # the mobile sharding brain
  ├── aggregate-shard-results/
  ├── run-e2e-playwright-tests/
  ├── upload-allure-report/
  ├── merge-e2e-detox-timings/
  └── …
```

**The hierarchy every workflow uses.** GitHub Actions has a strict three-level model; mastering it is 80% of reading any `.yml` in this repo.

```yaml
name: "[Desktop] - E2E Only - Scheduled/Manual"   # workflow

on:                                                # triggers
  workflow_dispatch:
    inputs:
      ref:
        description: "Branch to test"
      speculos_device:
        type: choice
        options: [nanoS, nanoSP, nanoX, stax, flex, nanoGen5]
  schedule:
    - cron: "0 4 * * 1-5"                          # Mon–Fri 04:00 UTC
  workflow_call:                                   # callable from other workflows

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true                         # newer push cancels older run

jobs:
  prepare-devices:                                 # job #1
    runs-on: ubuntu-latest
    outputs:
      e2e_matrix: ${{ steps.parse.outputs.e2e_matrix }}
    steps:                                         # steps
      - name: Parse speculos device input
        id: parse
        run: echo "..."

  e2e-tests:                                       # job #2 — fanned-out
    needs: [prepare-devices]
    strategy:
      fail-fast: false
      matrix: ${{ fromJson(needs.prepare-devices.outputs.e2e_matrix) }}
    runs-on: [ledger-live-linux-e2e-speculos-32CPU-128RAM]
    steps:
      - uses: actions/checkout@v6.0.1
      - uses: LedgerHQ/ledger-live/tools/actions/composites/setup-caches@develop
      - name: Run Playwright
        run: pnpm desktop e2e:speculos
```

| Layer | What it represents | How you think about it |
|-------|-------------------|--------------------------|
| **Workflow** | One YAML file, one `name:` | A single "button" in the Actions UI |
| **Job** | A node in the DAG, pinned to a runner | One virtual machine's worth of work |
| **Step** | Imperative command or `uses:` call | A single line of a shell script |
| **Composite action** | A reusable bundle of steps under `tools/actions/composites/` | A shell function you can call from any workflow |
| **Reusable workflow** | A YAML with `on.workflow_call` | A whole sub-pipeline you can invoke and pass inputs to |

**Concurrency.** `concurrency.cancel-in-progress: true` means when you push a new commit on the same branch, the older run is cancelled so runners are freed for the fresher commit. Reusable workflows that serve release gating (`test-mobile-e2e-reusable.yml`) deliberately set `cancel-in-progress: false` so two manual runs against the same branch serialise rather than destroy each other's data.

**Matrix strategy.** `strategy.matrix` fans one job out into N parallel jobs. Each instance gets unique values (a device name, a shard index, a platform…). `fail-fast: false` is the almost-universal setting in this repo — it means if shard 3 fails, shards 4–N still run to completion so Allure gets a full picture. Losing half the shards to save a few runner-minutes is not a trade QA accepts.

### 8.3.2 The E2E Workflow Map

Most of CI you will meet is either a **wrapper** (lightweight YAML that fires on PR) or a **reusable core** (the YAML with `workflow_call` that does the heavy lifting). The split exists so the same core logic is invoked identically whether a human presses Run, a cron fires, or a release workflow chains in.

**Desktop E2E — the concrete layout.**

| Workflow | Trigger | What it does |
|----------|---------|--------------|
| `test-ui-e2e-only-desktop.yml` | `workflow_dispatch`, `schedule`, `workflow_call` | The reusable core for desktop E2E. Runs Playwright against Speculos on a Linux self-hosted runner (`ledger-live-linux-e2e-speculos-32CPU-128RAM`). Matrix is **per device**, not per shard index — see 6.3.5 below. |
| `test-desktop-e2e-lint-reusable.yml` | `workflow_call` from PR | Lints the E2E test TypeScript without running any browser. Fast. |
| Build workflows (`build-desktop-*.yml`) | Various | Produce the Electron binaries cached by E2E jobs. |

**Mobile E2E — the concrete layout.**

| Workflow | Trigger | What it does |
|----------|---------|--------------|
| `test-mobile-e2e-reusable.yml` | `workflow_dispatch`, `schedule` (Mon–Fri 04:00 UTC), `workflow_call` | The reusable core for mobile E2E. Runs Detox against Speculos on iOS simulators and Android emulators, with true sharding (1–12 shards per platform) and greedy bin-packing. |
| `test-mobile-e2e-lint-reusable.yml` | `workflow_call` from PR | Lints Detox test TypeScript. |
| `notify-e2e-required-reusable.yml` | `workflow_call` on PR | Analyses diff paths and posts a PR comment listing which E2E pipelines are required for this PR. |

**Bot / non-regression workflows.**

| Workflow | Trigger | What it does |
|----------|---------|--------------|
| `bot-nonreg-nitrogen.yml` | Daily | Non-regression with SEED7 on `develop`. |
| `bot-nonreg-oxygen.yml` | Weekly | Costly non-regression with SEED5 on `develop`. |

**Release workflows.** `release-*.yml` chains into both `test-ui-e2e-only-desktop.yml` and `test-mobile-e2e-reusable.yml` via `workflow_call` with `production_firebase: true` (see 6.3.9).

**Mental model.** Every E2E run in the repo — whether started by a PR, a cron, a release gate, or `gh workflow run` — ultimately calls exactly one of the two reusable cores. The wrappers only differ in which inputs they pass.

### 8.3.3 Required Status Checks and the `notify-e2e-required` Workflow

CI gate logic in this repo is handled by the `notify-e2e-required-reusable.yml` workflow. Rather than relying on a single aggregator job named `@Gate`, the repo uses branch-protection rules on `develop` and `main` that list specific required checks, and the `notify-e2e-required` workflow analyses which E2E pipelines must pass for a given PR's diff.

The key design principle is the same: not every CI job is required for every PR. The `notify-e2e-required` reusable workflow is invoked from `build-and-test-pr.yml` and posts a PR comment listing which E2E pipelines are required based on which paths were touched. This keeps branch-protection stable without hard-coding shard indices.

**`if: always()` on aggregator jobs.** Any job that acts as a merge gate should specify `if: always()`, so it runs even when dependencies fail or are cancelled. Without this, the job is skipped, GitHub treats the status as "pending forever", and the PR becomes unmergeable without a manual override. This pattern appears throughout the repo's reusable workflows.

**Reading required checks in practice.** Go to the repo's Settings → Branches → `develop` rule → "Require status checks to pass before merging" to see which specific checks are required. The list is kept deliberately short to minimise drift as the workflow graph evolves.

### 8.3.4 Composite Actions

Instead of duplicating `actions/checkout`, `actions/setup-node`, cache restore, pnpm install, AWS credential assume-role, and so on across 75 workflow files, the repo centralises them as **composite actions** under `tools/actions/composites/`. Each composite is an `action.yml` plus optional scripts; any workflow references it with `uses: LedgerHQ/ledger-live/tools/actions/composites/<name>@develop`.

| Composite | Used by | What it does |
|-----------|---------|--------------|
| `setup-caches` | every E2E, build, lint workflow | Sets up pnpm, mise, Node, AWS credentials via OIDC, Nx remote cache, Turbo server token. Single source of truth for how the workspace bootstraps. |
| `setup-e2e-test-desktop` | desktop E2E | Installs desktop package, builds LLD in `testing` or `production` mode, seeds Playwright browsers. |
| `setup-e2e-env` | desktop E2E | Writes the `.env` file that configures API endpoints, broadcast flag, and per-build feature flags. |
| `setup-speculos_image` | desktop E2E | Pulls the pinned Speculos Docker image, pre-loads coin-apps. |
| `run-e2e-playwright-tests` | desktop E2E | Wraps the Playwright invocation with exit-code capture, test totals, and artifact naming. |
| `setup-android-env` | mobile E2E | Installs Android SDK, emulator components, and cached AVD. |
| `setup-android-avd` | mobile E2E | Boots one or more AVDs with kvm acceleration. |
| `duplicate-avd` | mobile E2E | Clones the base AVD into N per-emulator copies so N shards can run on one runner without state collision. |
| `generate-shards-matrix` | mobile E2E | The brain of mobile sharding — listing specs, sizing shards, computing timing-aware assignments (see 6.3.6). |
| `aggregate-shard-results` | mobile E2E | Collects `status_1` … `status_N` outputs from the matrix into one Slack-friendly summary. |
| `merge-e2e-detox-timings` | mobile E2E | Merges per-shard Jest timing JSONs into a single cache blob keyed by spec-dir hash. |
| `upload-allure-report` | desktop + mobile E2E | Generates HTML, uploads to the internal Allure server (auth via `ALLURE_USERNAME` / `ALLURE_LEDGER_LIVE_PASSWORD`), returns a `report-url`. |
| `get-allure-summary` | desktop + mobile E2E | Reads the Allure server's JSON summary after upload so Slack can render pass/fail counts. |
| `ci-flake-notifier` | desktop + mobile E2E | Posts flaky-test digest to Slack. |
| `setup-caches`, `nx-affected-packages`, `turbo-step`, `nx-step` | many | Nx/Turbo plumbing. |

Using composites: `- uses: LedgerHQ/ledger-live/tools/actions/composites/setup-caches@develop`. When the setup process changes (a new Node version, a cache-bucket migration), you update **one** `action.yml` and all 75 workflows get the new behaviour.

**Anatomy of a composite.** A composite is itself a small YAML with inputs, outputs, and steps. Example from `generate-shards-matrix`:

```yaml
name: "Generate Shards Matrix"
description: "Generates iOS and Android test file lists and sharding matrices …"
inputs:
  test_directory:  { default: "e2e/mobile" }
  test_filter:     { default: "" }
  event_name:      { required: true }
  ref:             { required: true }
  # … more inputs …
outputs:
  matrix_ios:      { value: ${{ steps.matrix.outputs.matrix_ios }} }
  matrix_android:  { value: ${{ steps.matrix.outputs.matrix_android }} }
runs:
  using: "composite"
  steps:
    - name: Get test files
      id: test-files
      shell: bash
      run: |
        node apps/ledger-live-mobile/scripts/shard-tests.mjs \
             "${{ inputs.test_filter }}" "${{ inputs.test_directory }}"
    - …
```

Key differences from a reusable workflow: composites **inline** into the calling job (they do not consume a new runner), they have no `jobs:` block (only steps), and they inherit the caller's working directory and secrets. Reusable workflows, by contrast, spawn their own jobs and must be passed secrets explicitly via `secrets: inherit` or named secret inputs.

**Pinning.** The `@develop` suffix pins the composite to the `develop` branch of the same repo. Release workflows sometimes pin `@v1.2.3` tags for reproducibility. Avoid `@main` on anything security-sensitive — composites can execute arbitrary code in your runner context.

### 8.3.5 Desktop Sharding

Two very different notions of "sharding" coexist on the desktop side. Pick the right one for your situation.

**Device sharding (what CI actually does).** `test-ui-e2e-only-desktop.yml` runs its matrix over **devices**, not over shard indices:

```yaml
prepare-devices:
  steps:
    - id: parse
      run: |
        if [ "${{ inputs.speculos_device }}" = "nanoSP and stax" ]; then
          echo 'e2e_matrix={"include":[
            {"device":"nanoSP","is_primary":true, "artifact_suffix":""     },
            {"device":"stax",  "is_primary":false,"artifact_suffix":"-stax"}
          ]}' >> $GITHUB_OUTPUT
        else
          DEVICE="${{ inputs.speculos_device || 'nanoSP' }}"
          printf 'e2e_matrix={"include":[{"device":"%s","is_primary":true,"artifact_suffix":""}]}\n' "$DEVICE" >> $GITHUB_OUTPUT
        fi

e2e-tests:
  needs: [prepare-devices]
  runs-on: [ledger-live-linux-e2e-speculos-32CPU-128RAM]
  strategy:
    fail-fast: false
    matrix: ${{ fromJson(needs.prepare-devices.outputs.e2e_matrix) }}
  timeout-minutes: 40
  env:
    SPECULOS_IMAGE_TAG: ghcr.io/ledgerhq/speculos:${{ matrix.device == 'nanoS' && '1be65b91a8e0691866f880fd437ac05fce78af9d' || 'latest' }}
    SPECULOS_DEVICE: ${{ matrix.device }}
```

The runner tag is specific — 32 vCPU / 128 GB RAM, Linux, Speculos-capable — and every spec in the Playwright corpus runs on it against one Speculos-emulated device firmware. The matrix exists so QA can demand "run the suite on nanoSP **and** stax" in one button press; those fan out into two parallel jobs.

**Playwright test-level sharding (what you run locally or in ad-hoc workflows).** Playwright has a built-in `--shard=I/N` flag that splits spec files hash-deterministically:

```bash
# Terminal 1
pnpm e2e:desktop test:playwright -- --shard=1/4
# Terminal 2
pnpm e2e:desktop test:playwright -- --shard=2/4
# Terminal 3
pnpm e2e:desktop test:playwright -- --shard=3/4
# Terminal 4
pnpm e2e:desktop test:playwright -- --shard=4/4
```

Each shard runs independently, spawns its own Speculos containers on random ports (see Part 4 Ch 4.1), and writes its own Allure JSONs. The results can then be merged with `allure generate --clean allure-results-*`. Ad-hoc shard-based workflows occasionally appear for one-off bulk reruns, but the standard PR path is the single-job-per-device model above.

**Why the difference from mobile?** Desktop runners are beefy (32 vCPU), Playwright already parallelises at the worker level inside one job, and Speculos containers are cheap to spin up per-worker. So one runner churns through the whole corpus with internal parallelism. Mobile, in contrast, is constrained by `maxWorkers: 1` per emulator (see Part 8 Ch 8.2), making runner-level sharding the only scalable axis.

Cross-reference Part 4 Ch 4.1.5 for the Playwright-sharding primer and Part 3 Ch 3.3 for the Speculos setup that runs inside both flows.

### 8.3.6 Mobile Sharding Deep Dive

Mobile needs genuine parallelism because Detox pins Jest to one worker per device. The answer is to parallelise at the **runner** level: split N test files across K GitHub runners, run them truly in parallel, merge Allure at the end. Wall-clock becomes `ceil(N/K) × per-test-time` instead of `N × per-test-time`.

Two components cooperate to make it work:

1. `tools/actions/composites/generate-shards-matrix/action.yml` — decides how many shards and emits the matrix JSON.
2. `apps/ledger-live-mobile/scripts/shard-tests.mjs` — decides which specs go into which shard.

#### The `generate-shards-matrix` composite — walkthrough

```yaml
name: "Generate Shards Matrix"
description: "Generates iOS and Android test file lists and sharding matrices …"
inputs:
  test_directory:        { default: "e2e/mobile" }
  test_filter:           { default: "" }
  event_name:            { required: true }
  max_shards:            { default: "0" }
  ref:                   { required: true }
  ios_timing_cache_key:  { default: "" }
  android_timing_cache_key: { default: "" }
  aws_access_key_id: {} ; aws_secret_access_key: {} ; aws_session_token: {}
  cache_bucket: {} ; aws_region: {}

runs:
  using: "composite"
  steps:
    - name: Get test files
      id: test-files
      shell: bash
      run: |
        TEST_FILES=$(node apps/ledger-live-mobile/scripts/shard-tests.mjs \
          "${{ inputs.test_filter }}" "${{ inputs.test_directory }}")
        echo "test_files_for_sharding<<EOF" >> $GITHUB_OUTPUT
        echo "$TEST_FILES"                  >> $GITHUB_OUTPUT
        echo "EOF"                          >> $GITHUB_OUTPUT

    - name: Calculate shard counts
      id: shard-counts
      shell: bash
      run: |
        TEST_COUNT=$(echo "${{ steps.test-files.outputs.test_files_for_sharding }}" | wc -w)

        # Target one shard per group of 3 tests — matches 3 emulators/simulators
        # per runner. Keeps all devices busy without provisioning extras.
        EMULATORS_PER_RUNNER=3
        BASE_SHARD=$(( TEST_COUNT > 0
                       ? (TEST_COUNT + EMULATORS_PER_RUNNER - 1) / EMULATORS_PER_RUNNER
                       : 1 ))

        if [ "${{ inputs.event_name }}" = "schedule" ]; then
          SHARD_COUNT_IOS=$(( BASE_SHARD < 12 ? BASE_SHARD : 12 ))
          SHARD_COUNT_ANDROID=$(( BASE_SHARD < 12 ? BASE_SHARD : 12 ))
        elif [[ "${{ inputs.ref }}" == *"release"* || "${{ inputs.ref }}" == *"hotfix"* ]]; then
          SHARD_COUNT_IOS=$(( BASE_SHARD < 12 ? BASE_SHARD : 12 ))
          SHARD_COUNT_ANDROID=$(( BASE_SHARD < 12 ? BASE_SHARD : 12 ))
        else
          SHARD_COUNT_IOS=$(( BASE_SHARD < 3 ? BASE_SHARD : 3 ))
          SHARD_COUNT_ANDROID=$(( BASE_SHARD < 12 ? BASE_SHARD : 12 ))
        fi

        echo "shard_count_ios=$SHARD_COUNT_IOS"         >> $GITHUB_OUTPUT
        echo "shard_count_android=$SHARD_COUNT_ANDROID" >> $GITHUB_OUTPUT

    # Subsequent steps (abbreviated): restore timing data from S3 with AWS creds,
    # call shard-tests.mjs once per shard to emit the file assignment, and build
    # matrix_ios / matrix_android JSON arrays for consumption by the two test jobs.
```

**Reading the math.** `BASE_SHARD = ceil(TEST_COUNT / 3)`. The `3` is the number of simulators or emulators each runner boots in parallel. Three devices per runner means one shard's workload is roughly three specs' worth, so when the worker-per-emulator restriction eventually lifts, the infrastructure is already sized.

**The two-tier caps.**

| Event | iOS cap | Android cap | Rationale |
|-------|---------|-------------|-----------|
| `schedule` | 12 | 12 | Nightly full pass; wall-clock matters most. |
| Manual on `release/*` or `hotfix/*` | 12 | 12 | Release gating; PR-review urgency. |
| Manual on any other ref | 3 | 12 | macOS runners are scarce and expensive; Linux runners are abundant. |

**Timing-data restore.** Before sharding, the composite restores per-spec duration data from an S3 bucket keyed by the spec-directory hash. Keys come in via `ios_timing_cache_key` / `android_timing_cache_key` inputs. Missing data is not fatal — the bin-packer handles a zero-history run gracefully, as described below.

#### The `shard-tests.mjs` algorithm — greedy bin-packing

The file is ~200 lines at `apps/ledger-live-mobile/scripts/shard-tests.mjs`. Core loop:

```js
// 1. Enumerate spec files under the test directory, excluding *.skip.spec.ts
//    and applying the user-supplied filter (a regex over file path + contents).

// 2. Join each file with its historical duration (from merged timing JSON);
//    0 if no history.

const filesWithTiming = collected.map(file => ({
  file,
  duration: history[file] ?? 0,
}));

// 3. Sort DESCENDING by duration; tie-break alphabetically for determinism.
filesWithTiming.sort((a, b) => {
  if (b.duration !== a.duration) return b.duration - a.duration;
  return compareStrings(a.file, b.file);
});

const testsWithTiming     = filesWithTiming.filter(f => f.duration >  0);
const testsWithZeroTiming = filesWithTiming.filter(f => f.duration === 0);

// 4. Empty shards; greedy-assign each timed test to the currently-lightest one.
const shards = Array.from({ length: shardTotal },
                         () => ({ files: [], totalDuration: 0 }));

for (const { file, duration } of testsWithTiming) {
  let minIdx = 0, minTotal = Infinity;
  for (let i = 0; i < shardTotal; i++) {
    if (shards[i].totalDuration < minTotal) {
      minTotal = shards[i].totalDuration;
      minIdx   = i;
    }
  }
  shards[minIdx].files.push(file);
  shards[minIdx].totalDuration += duration;
}

// 5. Round-robin the zero-timing tests. They're new specs — no history yet,
//    so we spread them uniformly and let next run measure them.
testsWithZeroTiming.forEach((t, i) => {
  shards[i % shardTotal].files.push(t.file);
});

return shards[shardIndex - 1]?.files ?? [];
```

**Why greedy, longest-first is near-optimal.** The goal is to minimise **the slowest shard's duration**, because that is the critical-path wall time. A classic result in bin-packing: sorting items longest-first and placing each on the currently-lightest bin (Longest-Processing-Time, LPT) is guaranteed to be within 4/3 of optimal, and in practice much closer. The alternative — naive round-robin (`file_i → shard_{i mod K}`) — can be catastrophically unbalanced if durations are skewed.

**A concrete worked example.** 27 specs, scheduled run, 9 shards. Assume:

- 3 specs × 8 minutes each (the long ones),
- 6 specs × 4 minutes,
- 18 specs × 2 minutes.

LPT walk:

| Step | What happens | Shard totals (min) |
|------|--------------|--------------------|
| Sort DESC | 8,8,8,4,4,4,4,4,4,2×18 | — |
| Place 8,8,8 | Each goes to a fresh empty shard | `[8,8,8,0,0,0,0,0,0]` |
| Place 4,4,4,4,4,4 | Each picks the lightest (i.e. the zeros) | `[8,8,8,4,4,4,4,4,4]` |
| Place eighteen 2-min specs | First six go to shards 4–9 (lightest at 4) → `[8,8,8,6,6,6,6,6,6]`. Next six again → `[8,8,8,8,8,8,8,8,8]`. Next three go to shards 1–3 → `[10,10,10,8,8,8,8,8,8]`. Final three go to shards 4–9 (tie-break on index). | `[10,10,10,10,10,10,8,8,8]` finally `[10,10,10,10,10,10,10,10,10]` |
| Result | All nine shards equal at 10 minutes | — |

Critical-path wall time: **10 minutes**. A naive round-robin over 27 specs into 9 shards would give each shard 3 random specs; in the worst case one shard gets all three 8-minute specs = **24 minutes**, and another gets three 2-min specs = **6 minutes**. LPT beats that by more than 2×.

**Determinism.** Same spec set + same timing data → same assignments, because ties break on file path. That is essential for artifact names (`ios-test-artifacts-3`) to be stable across reruns, which in turn lets the timing cache update work correctly.

**The first-run problem.** When you add a new spec, it has zero timing history. Round-robin over the zero-timing bucket distributes new specs evenly across shards. On the next run, the new specs' durations are in the cache and LPT treats them as first-class citizens.

### 8.3.7 Environment Variable Flow

Every interesting variable that flows from `workflow_dispatch` inputs down to the Detox child process:

| Variable | Source | Consumed by | Effect |
|----------|--------|-------------|--------|
| `SEED` | Repo secret `SEED_QAA_B2C` | App (via bridge) | The 24-word mnemonic the emulated Speculos uses. Determines every derived address. |
| `MOCK` | Set on the test step (desktop) | App runtime | Enables mock mode — no real network calls. Desktop concept; mobile uses `selectMockDevice` in specs. |
| `PRODUCTION` | Input `production_firebase` | `e2e-ci.mjs` | When `true`, orchestrator flips target from `release` to `prerelease`. |
| `SHARD_INDEX` | `matrix.shard` | Detox / Jest | 1-based index of the current shard — used for artifact names and, when enabled, Jest's `--shard N/M`. |
| `SHARD_TOTAL` | `matrix.total` | Detox / Jest | Total number of shards in the matrix. |
| `SHARD_TEST_FILES` | `matrix.files` written to `$GITHUB_ENV` | Detox `--e2e` argument | Space-separated list of specs for this shard. |
| `SPECULOS_DEVICE` | Input `speculos_device` | Speculos container | Which device firmware image Speculos loads. |
| `SPECULOS_API_PORT` | Computed per-shard | Speculos container + bridge | Default `52619`; per-shard variants avoid collisions across three simulators on one runner. |
| `SPECULOS_IMAGE_TAG` | Computed from `speculos_device` | `docker pull` | `latest` for most devices; pinned legacy SHA (`1be65b91…`) for Nano S. |
| `E2E_ENABLE_WALLET40` | Input `enable_wallet40` | App feature-flag loader | `"1"` enables Wallet 4.0 UI; `"0"` keeps the legacy flow. |
| `E2E_ENABLE_BROADCAST` | Derived from `enable_broadcast` (inverse of `DISABLE_TRANSACTION_BROADCAST`) | App transaction layer | When disabled, `sendTransaction` no-ops rather than broadcasting. |
| `ENVFILE` | Computed in Gradle/Xcode build from `production_firebase` | Native build | Picks `.env.mock` vs `.env.mock.prerelease`. |
| `ANDROID_APK_PATH` | Computed from `production_firebase` | Detox | Points at `app-x86_64-detoxPreRelease.apk` when production, otherwise `app-x86_64-detox.apk`. |
| `INPUTS_TEST_FILTER` | `steps.test-filter.outputs.filter` | Spec runner | The resolved filter after optional `@smoke` prefix. |
| `SWAP_API_BASE` | Repo env | App runtime | Points the swap feature at a stubbed backend. |
| `DEVICE_INFO` | Computed per-platform | Allure metadata | Human-readable device string (e.g. `iPhone 15 (iOS 17.4)`), attached as an Allure parameter. |
| `E2E_FEATURE_FLAGS_JSON` | Input `feature_flags_json` (desktop) | App runtime | Extra feature-flag overrides for the run. |

**The chain — inputs become process.env.** GitHub Actions does not auto-expose workflow inputs as environment variables. Each layer is explicit:

```
workflow_dispatch / workflow_call inputs
        │
        ▼
env: block at workflow level     (uses ${{ inputs.* }})
        │
        ▼
env: block at job level          (uses ${{ inputs.* }} and top-level env)
        │
        ▼
env: block at step level         (uses ${{ matrix.* }}, secrets.*, above env)
        │
        ▼
$GITHUB_ENV writes for values that must cross multiple steps in the same job
        │
        ▼
child process inherits environment
        │
        ▼
pnpm mobile e2e:ci  /  pnpm desktop e2e:speculos
        │
        ▼
detox test  /  playwright test
        │
        ▼
Jest  /  Playwright worker
        │
        ▼
process.env.<VAR> accessed in setup.ts, bridge, and specs
```

Each `${{ … }}` interpolation is evaluated when the runner parses the step. A subtle trap: `env:` blocks at the step level **override** job-level ones, so late-stage customisation is safe.

### 8.3.8 Artifact Flow

Artifacts fan out from shards, get merged, then get published.

**Per-shard outputs (mobile).**

- `ios-test-artifacts-N` — a zip of `e2e/mobile/artifacts/` containing Allure JSON, screenshots, videos (on failure), and the Detox log bundle.
- A timing JSON uploaded separately as `<ios_timing_cache_key>-N` — consumed by the next run's sharding step via the `merge-e2e-detox-timings` composite.
- A status output `status_<N>` exposed from the job via `outputs:`, collected later by `aggregate-shard-results`.

**Per-device outputs (desktop).**

- `allure-results${{ matrix.artifact_suffix }}` — one per device in the matrix. Suffix is empty for the primary device and `-stax` (etc.) for secondary devices.

**Merge step.** Both desktop and mobile use the same pattern in their report jobs:

```yaml
- name: Download all shard artifacts
  uses: actions/download-artifact@...
  with:
    path: ios-test-artifacts
    pattern: ios-test-artifacts*
    merge-multiple: true                        # <── the glue

- name: Upload Allure report
  uses: LedgerHQ/ledger-live/tools/actions/composites/upload-allure-report@develop
  with:
    platform: ios-e2e
    path: ios-test-artifacts
```

`merge-multiple: true` extracts all 12 per-shard zips into the same directory, effectively concatenating their Allure result JSONs. The `upload-allure-report` composite then:

1. Generates the HTML report with `allure generate`.
2. Uploads to the internal Allure server (auth via `ALLURE_USERNAME` / `ALLURE_LEDGER_LIVE_PASSWORD`).
3. Emits `report-url` — posted to Slack by the downstream `report-on-slack` job.

**Xray sync (optional).** If `export_to_xray: true`, the `upload-to-xray` job reformats the same Allure JSONs with `e2e/mobile/xray.formater.sh`, authenticates against Xray Cloud, and POSTs to `/api/v2/import/execution`. The resulting test-execution key is written to the job summary. Cross-reference Part 3 Ch 3.5 for the full Allure collection pipeline and how `CODE scopes` propagate from specs to the final report.

**Timing-cache update.** After the test step, each shard uploads its `e2e-test-results-<platform>-shard-N.json` (Jest `--json --outputFile`). A dedicated job merges those N files, S3-caches the union under the spec-dir-hash key, and the next run's `generate-shards-matrix` step restores it. Closes the feedback loop on the greedy bin-packer.

### 8.3.9 Smoke vs Full, Production vs Staging Firebase

**Smoke runs.** PRs run `smoke_tests: true` by default. The reusable workflow prepends `@smoke` to the filter:

```yaml
- name: Prepare test filter
  id: test-filter
  run: |
    BASE_FILTER="${{ inputs.test_filter }}"
    if [ -n "$BASE_FILTER" ]; then
      BASE_FILTER="${BASE_FILTER//,/|}"            # commas → pipes (regex alternation)
    fi
    if [ "${{ inputs.smoke_tests }}" = "true" ]; then
      if [ -n "$BASE_FILTER" ]; then
        echo "filter=@smoke $BASE_FILTER" >> $GITHUB_OUTPUT
      else
        echo "filter=@smoke" >> $GITHUB_OUTPUT
      fi
    else
      echo "filter=$BASE_FILTER" >> $GITHUB_OUTPUT
    fi
```

Any `it()` title containing `@smoke` matches; everything else is skipped. The tag is maintained in source:

```typescript
it(
  `@smoke @B2CQA-604 • Send BTC - valid address, broadcast disabled`,
  async () => { /* … */ },
);
```

Smoke corpora are ~10–15% of the full suite, chosen for **breadth** (one test per coin family, one per major feature) rather than depth. Wall time drops from ~45 min/platform to ~10 min/platform. Scheduled runs always set `smoke_tests: false`.

**`describe.skip` vs tags.** A few legacy files still use `describe.skip(…)` to disable an entire block. Prefer tags in new work — they compose (`@smoke @bitcoin`) while `describe.skip` is binary.

**Production vs staging Firebase.** `production_firebase: true` flips four things in lockstep:

1. **Build config:** `ENVFILE=.env.mock.prerelease` → production Firebase keys baked into the binary.
2. **Binary path:** Gradle outputs `app-x86_64-detoxPreRelease.apk` instead of `app-x86_64-detox.apk`; the workflow env uses a ternary to pick the right one.
3. **Cache-key suffix:** `prod-2` vs `3` — separate native-build caches so production and staging don't overwrite each other.
4. **Orchestrator:** `--production` flag to `e2e-ci.mjs` → Detox uses the `*.prerelease` config.

**When you need it.**

- **Release / hotfix gating.** Remote Config flag values differ between projects; staging-green is insufficient for release sign-off.
- **Production-only feature validation.** Features gated by production Remote Config rollouts only light up here.
- **Incident reproduction.** When a user report cannot be reproduced against staging, switch to production to see if it's Remote-Config-driven.

**When you do not need it.**

- **Every PR.** The CI cost is 2× (separate native build, separate cache line), and staging is representative for 95% of changes.
- **Pure UI work.** If the change doesn't touch anything Firebase-driven, staging is enough.

### 8.3.10 Kicking Off a Custom Run with `gh workflow run`

**`test-mobile-e2e-reusable.yml` — canonical inputs.** (Verify against `.github/workflows/test-mobile-e2e-reusable.yml`; the list below is current as of the `e2e/mobile/` migration.)

| Input | Type | Purpose |
|-------|------|---------|
| `ref` | string | Branch/sha to test |
| `test_filter` | string | Filter by test name / path / tag (comma or pipe separated) |
| `tests_type` | choice | `Android Only` \| `iOS Only` \| `iOS & Android` |
| `speculos_device` | choice | `nanoS` \| `nanoSP` \| `nanoX` \| `stax` \| `flex` \| `nanoGen5` |
| `production_firebase` | boolean | Switch to prerelease Firebase + `.env.mock.prerelease` |
| `enable_broadcast` | boolean | Actually broadcast signed transactions on-chain |
| `export_to_xray` | boolean | **(NEW)** Reformat Allure JSONs and POST to Xray Cloud (`upload-to-xray` job) |
| `test_execution_android` | string | **(NEW)** Optional pre-existing Xray Test Execution key for Android results |
| `test_execution_ios` | string | **(NEW)** Optional pre-existing Xray Test Execution key for iOS results |
| `smoke_tests` | boolean | Prepend `@smoke` to the filter for a fast subset |
| `enable_wallet40` | boolean | Default `true`; toggles the Wallet 4.0 UI (`E2E_ENABLE_WALLET40`) |
| `generate_ai_artifacts` | boolean | **(NEW)** Produce AI-triage artifacts alongside Allure |

Three paths of increasing speed.

**1. GitHub Actions UI.** Browse to Actions → "[Mobile] - E2E Only - Scheduled/Manual" → "Run workflow". Pick inputs from the dropdowns. Obvious but slow — form-filling adds minutes.

**2. The `gh` CLI.** Fastest for repeat runs:

```bash
gh workflow run test-mobile-e2e-reusable.yml \
  -f ref=feat/qaa-702-add-cosmos-staking \
  -f tests_type="iOS & Android" \
  -f test_filter='@bitcoin Cosmos' \
  -f speculos_device=nanoX \
  -f smoke_tests=false
```

Follow along live in your terminal:

```bash
gh run watch
```

**3. Common recipes.**

Run only one ticket's tests on iOS against a PR branch:

```bash
gh workflow run test-mobile-e2e-reusable.yml \
  -f ref=feat/qaa-702 \
  -f tests_type="iOS Only" \
  -f test_filter='-t "B2CQA-604"'
```

> Note: `test_filter` is passed verbatim to `shard-tests.mjs`, which treats it as a regex over both file path and file content. Jest-specific flags like `-t` survive because the script greps file contents for the pattern as well.

Validate a release candidate against production Firebase:

```bash
gh workflow run test-mobile-e2e-reusable.yml \
  -f ref=release/3.50.0 \
  -f tests_type="iOS & Android" \
  -f production_firebase=true \
  -f enable_broadcast=false
```

Smoke-only run after a dependency bump:

```bash
gh workflow run test-mobile-e2e-reusable.yml \
  -f ref=chore/bump-dmk \
  -f tests_type="iOS & Android" \
  -f smoke_tests=true
```

Desktop equivalent:

```bash
gh workflow run test-ui-e2e-only-desktop.yml \
  -f ref=feat/qaa-812-swap-history-erc20-export \
  -f speculos_device=nanoSP \
  -f test_filter='@swap' \
  -f smoke_tests=false \
  -f enable_broadcast=false \
  -f build_type=testing
```

Cross-reference the daily loops in Part 4 Ch 4.8 (desktop) and Part 5 Ch 5.9 (mobile) for when and why you fire these from your laptop vs wait for the PR check.

**Watching smartly.** `gh run watch` refreshes every few seconds and surfaces the full job tree. Useful additions:

```bash
# Pick a specific run to watch (if multiple are in flight)
gh run list --workflow=test-mobile-e2e-reusable.yml --limit 5
gh run watch <run-id>

# Tail logs of a specific job once the run is underway
gh run view <run-id> --log --job=<job-id>

# Only download a single shard's artifact for triage
gh run download <run-id> --name ios-test-artifacts-4
```

**Skipping the UI altogether.** Once you know the canonical inputs for your flow, save them as a shell function in your dotfiles:

```bash
# ~/.zshrc
ll-e2e-mobile-smoke() {
  gh workflow run test-mobile-e2e-reusable.yml \
     -f ref="$(git rev-parse --abbrev-ref HEAD)" \
     -f tests_type="iOS & Android" \
     -f smoke_tests=true
  gh run watch
}
```

Then one shell invocation triggers the run and watches it for you. This is how the experienced QA engineers at Ledger spend zero seconds in the Actions UI.

### 8.3.11 Reading a Failed Run

The recipe for triaging a red E2E CI:

1. **Click the failing run.** In the PR's Checks tab, open the red workflow. Find the red shard — mobile looks like "iOS E2E Tests (4, 12)"; desktop looks like "Desktop E2E (stax)".
2. **Scroll to the failing step.** On mobile, it will be `Run iOS Detox shard 4/12`; on desktop, `Run Playwright`. The exit code separates test failures (`1`) from setup errors (`127`, `137`, etc.). Note the `DEVICE_INFO` line so you know which simulator/emulator you're reproducing.
3. **Look for the Allure URL.** The `allure-report-<platform>` job posts the URL to the GitHub job summary on success. If it's there, click through — you skip the local-replay dance entirely.
4. **In Allure, find the failed `it()`.** Navigate to "Failed" and open the test. The Attachments panel usually contains:
   - `app.log` — the JS console from the app.
   - `bridge.log` — the Speculos bridge traffic.
   - `device.log` — Detox or Playwright device output.
   - Failure screenshot.
   - Video or screen recording.
5. **If Allure did not render** (the shard timed out entirely, or the report upload step was skipped), fall back to raw artifacts: run summary → Artifacts → download `ios-test-artifacts-4.zip` (or `allure-results-stax.zip` for desktop) and unzip locally. You get the same Allure inputs plus raw Detox/Playwright artifacts.
6. **Reproduce locally.** For mobile:

```bash
cd e2e/mobile
pnpm build:ios                              # nx run live-mobile:e2e:build -- --configuration ios.sim.release
pnpm test:ios -- -t "<exact it() name from Allure>" --loglevel trace
```

For desktop:

```bash
cd apps/ledger-live-desktop
SPECULOS_DEVICE=stax pnpm playwright test tests/specs/swap/history.spec.ts \
     --grep "<exact it() name>" --headed
```

7. **If it passes locally but fails in the shard**, suspect state bleed from an earlier spec in the same shard. Look at the shard's step log — the file order is printed at the start of the Detox/Playwright run — and re-run the preceding specs locally in the same order. This is the most common "green locally, red in shard N" root cause.

**Common CI error patterns.**

| Error | Most likely cause |
|-------|-------------------|
| `TimeoutError: locator.click` | Element never appeared; feature flag mismatch, navigation change, or sync race. |
| `expect(received).toHaveText(expected)` | Assertion mismatch — real difference or i18n drift. |
| `ECONNREFUSED :5000` / `ECONNREFUSED :52619` | Speculos container failed to start or died mid-run. Check the `Pull Speculos docker image` step for image-pull issues. |
| `OCI runtime create failed` | Docker out of disk on the runner; transient, re-run usually fixes it. |
| `Exit code 137` | OOM kill — `NODE_OPTIONS=--max-old-space-size=14336` is already set on desktop; on mobile, usually means an emulator crashed. |
| `detox test failed with exit code null` | Bridge disconnected unexpectedly. Almost always an app crash — check `app.log`. |

**Node Action Cache error (legacy).** An older pattern: intermittent cache failures from GitHub's `actions/cache`. The team migrated to an S3 cache, but if you see it on an older branch, the tactical fix is to comment out the `cache` and `cache-dependency-path` lines in the `node-setup` composite. Better to rebase onto fresh `develop`.

**Flake vs. real failure.** A single shard red across a run of 12 is not automatically a flake — the greedy bin-packer is deterministic, and so are your specs. Before re-running, check:

1. **Does the same `it()` fail on the second run?** If yes, treat as a real failure.
2. **Does the app log show a crash?** `app.log` → search for `FATAL` or stack traces. A crash is a code bug, not a test bug.
3. **Does the `bridge.log` show the device hanging?** Speculos occasionally freezes on the second-to-last transaction — the CI-flake-notifier composite tracks these.
4. **Is the failed test tagged with `@flaky` in source?** Some specs are explicitly marked; they should fail softly and be triaged weekly, not re-run blindly.

If none of the above apply, a plain re-run burns runner time you could spend on the root-cause investigation. When in doubt, reproduce locally (step 6 above) before hitting "Re-run failed jobs".

### 8.3.12 ASCII Sequence Diagrams

#### Mobile sharding pipeline

```
  DEVELOPER            GITHUB ACTIONS                          DETOX                ALLURE / XRAY
  ─────────            ──────────────                          ─────                ─────────────
     │                         │                                  │                       │
     │  gh workflow run -f …   │                                  │                       │
     ├────────────────────────►│                                  │                       │
     │                         │                                  │                       │
     │                 ┌───────┴───────────┐                      │                       │
     │                 │ determine-builds  │                      │                       │
     │                 │ (ledger-live-med) │                      │                       │
     │                 │                   │                      │                       │
     │                 │  checkout ref     │                      │                       │
     │                 │  compute filter   │                      │                       │
     │                 │  shard-tests.mjs  │──► read test dir     │                       │
     │                 │                   │◄── N spec files      │                       │
     │                 │  restore timing   │──► S3 cache          │                       │
     │                 │  cache (AWS)      │◄── per-spec durations│                       │
     │                 │  greedy bin-pack  │                      │                       │
     │                 │  emit matrix_ios  │                      │                       │
     │                 │  emit matrix_and  │                      │                       │
     │                 └───────┬───────────┘                      │                       │
     │                         │                                  │                       │
     │                 ┌───────┴──────────────┐                   │                       │
     │                 │ build-ios (macOS)    │                   │                       │
     │                 │ build-android (linux)│                   │                       │
     │                 │                      │                   │                       │
     │                 │  compile + bundle    │                   │                       │
     │                 │  upload to S3 cache  │                   │                       │
     │                 └───────┬──────────────┘                   │                       │
     │                         │                                  │                       │
     │                 ┌───────┴──────────────────────┐           │                       │
     │                 │ detox-tests-ios matrix       │           │                       │
     │                 │ [shard 1/9][shard 2/9] … [9] │           │                       │
     │                 │ fail-fast: false             │           │                       │
     │                 │ (9 parallel macOS runners)   │           │                       │
     │                 │                              │           │                       │
     │                 │  per shard:                  │           │                       │
     │                 │   ├─ checkout + install      │           │                       │
     │                 │   ├─ download binary from S3 │           │                       │
     │                 │   ├─ boot 3 simulators       │           │                       │
     │                 │   ├─ pull speculos image     │           │                       │
     │                 │   ├─ run e2e-ci.mjs          │──────────►│ detox test            │
     │                 │   │                          │           │  run subset           │
     │                 │   │                          │           │  capture artifacts    │
     │                 │   │                          │◄──────────│ exit code N           │
     │                 │   ├─ upload ios-test-arts-K  │           │                       │
     │                 │   ├─ upload timing JSON      │           │                       │
     │                 │   ├─ set status_<K> output   │           │                       │
     │                 │   └─ delete simulators       │           │                       │
     │                 └───────┬──────────────────────┘           │                       │
     │                         │                                  │                       │
     │                 ┌───────┴────────────────┐                 │                       │
     │                 │ allure-report-ios      │                 │                       │
     │                 │ allure-report-android  │                 │                       │
     │                 │                        │                 │                       │
     │                 │  download all shard    │                 │                       │
     │                 │  artifacts, merge      │                 │                       │
     │                 │  generate HTML         │                 │                       │
     │                 │  upload to server      │────────────────────────────────────────►│ Allure web UI
     │                 │  aggregate status_1…N  │                 │                       │
     │                 └───────┬────────────────┘                 │                       │
     │                         │                                  │                       │
     │                 ┌───────┴────────────────┐                 │                       │
     │                 │ upload-to-xray         │                 │                       │
     │                 │ (if export_to_xray)    │                 │                       │
     │                 │                        │                 │                       │
     │                 │  format allure→xray    │                 │                       │
     │                 │  POST /import/execution│────────────────────────────────────────►│ Xray test exec
     │                 └───────┬────────────────┘                 │                       │
     │                         │                                  │                       │
     │                 ┌───────┴────────────────┐                 │                       │
     │                 │ @Gate                  │                 │                       │
     │                 │  aggregate required    │                 │                       │
     │                 │  emit single pass/fail │                 │                       │
     │                 └───────┬────────────────┘                 │                       │
     │                         │                                  │                       │
     │                 ┌───────┴────────────────┐                 │                       │
     │                 │ report-on-slack        │                 │                       │
     │                 │  post summary + URLs   │                 │                       │
     │                 └───────┬────────────────┘                 │                       │
     │                         │                                  │                       │
     │ ◄───────────────────────┤  notification in #qa-results     │                       │
     │                         │                                  │                       │
```

Follow the arrows top-to-bottom: trigger → determine matrix → build → sharded test → merge → report → gate. The critical insight: **all shards run truly in parallel** — wall time of the `detox-tests-ios` job is the wall time of the slowest shard, not the sum. Hence the greedy bin-packer's job: minimise the maximum.

#### Desktop device-matrix pipeline

```
  DEVELOPER            GITHUB ACTIONS                          PLAYWRIGHT           ALLURE
  ─────────            ──────────────                          ──────────           ──────
     │                         │                                  │                    │
     │  gh workflow run -f …   │                                  │                    │
     ├────────────────────────►│                                  │                    │
     │                         │                                  │                    │
     │                 ┌───────┴───────────┐                      │                    │
     │                 │ prepare-devices   │                      │                    │
     │                 │                   │                      │                    │
     │                 │  parse input      │                      │                    │
     │                 │  emit e2e_matrix  │                      │                    │
     │                 │   {include:[nanoSP│                      │                    │
     │                 │            ,stax]}│                      │                    │
     │                 └───────┬───────────┘                      │                    │
     │                         │                                  │                    │
     │                 ┌───────┴──────────────────────┐           │                    │
     │                 │ e2e-tests matrix             │           │                    │
     │                 │ [device=nanoSP][device=stax] │           │                    │
     │                 │ fail-fast: false             │           │                    │
     │                 │ runs-on: self-hosted linux   │           │                    │
     │                 │   32CPU / 128GB RAM          │           │                    │
     │                 │                              │           │                    │
     │                 │  per device:                 │           │                    │
     │                 │   ├─ checkout + coin-apps    │           │                    │
     │                 │   ├─ setup-caches            │           │                    │
     │                 │   ├─ pull speculos image     │           │                    │
     │                 │   ├─ build LLD (testing)     │           │                    │
     │                 │   ├─ setup-e2e-env           │           │                    │
     │                 │   ├─ compute filter (@smoke) │           │                    │
     │                 │   ├─ run-e2e-playwright-tests│──────────►│ playwright test    │
     │                 │   │                          │           │  workers × N       │
     │                 │   │                          │           │  spawn Speculos    │
     │                 │   │                          │           │  on random ports   │
     │                 │   │                          │◄──────────│ exit code N        │
     │                 │   ├─ upload allure-results   │           │                    │
     │                 │   │   {suffix}               │           │                    │
     │                 │   └─ pass/fail outputs       │           │                    │
     │                 └───────┬──────────────────────┘           │                    │
     │                         │                                  │                    │
     │                 ┌───────┴────────────────┐                 │                    │
     │                 │ allure-report (merge)  │                 │                    │
     │                 │  download all device   │                 │                    │
     │                 │  artifacts             │                 │                    │
     │                 │  merge-multiple:true   │                 │                    │
     │                 │  generate HTML         │                 │                    │
     │                 │  upload to server      │──────────────────────────────────────►│ Allure web UI
     │                 └───────┬────────────────┘                 │                    │
     │                         │                                  │                    │
     │                 ┌───────┴────────────────┐                 │                    │
     │                 │ @Gate + Slack notify   │                 │                    │
     │                 └───────┬────────────────┘                 │                    │
     │                         │                                  │                    │
     │ ◄───────────────────────┤  notification in #qa-results     │                    │
     │                         │                                  │                    │
```

Note the shape difference: desktop fans out over **devices** (usually one, occasionally two for nanoSP + stax), each device does its own internal Playwright-worker parallelism. Mobile fans out over **shards** (up to 12), each shard does sequential specs. Same tools, inverted strategies, because the cost model of the underlying device is inverted.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://docs.github.com/en/actions">GitHub Actions documentation</a></li>
<li><a href="https://docs.github.com/en/actions/using-workflows/reusing-workflows">Reusable workflows</a></li>
<li><a href="https://docs.github.com/en/actions/sharing-automations/creating-actions/creating-a-composite-action">Creating composite actions</a></li>
<li><a href="https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/using-a-matrix-for-your-jobs">Matrix strategy</a></li>
<li><a href="https://playwright.dev/docs/test-sharding">Playwright sharding</a></li>
<li><a href="https://en.wikipedia.org/wiki/Longest-processing-time-first_scheduling">Longest-Processing-Time scheduling (LPT)</a></li>
<li><a href="https://crontab.guru/">Crontab Guru</a> — interactive cron editor</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> You will rarely write workflow YAML from scratch — the composites and the reusable cores are mature. Most of your CI interaction is reading runs, downloading artifacts, understanding <em>why</em> a shard failed, and kicking off a <code>gh workflow run</code> to validate a fix. Know the <code>@Gate</code> contract, know that mobile's shard count is driven by <code>ceil(N/3)</code> capped at 3/12 or 12/12, know that desktop's matrix is per-device not per-shard, and know that <code>shard-tests.mjs</code> is a greedy bin-packer that minimises the slowest shard. The rest is mechanics.
</div>

### 8.3.13 Chapter 8.3 Quiz

<!-- ── Chapter 8.3 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — CI/CD &amp; Test Sharding</h3>
<p class="quiz-subtitle">8 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> Why does the Ledger Live repo have ~75 workflow files instead of a single CI pipeline?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because GitHub Actions limits each file to 10 jobs</button>
<button class="quiz-choice" data-value="B">B) Because the monorepo produces multiple artifacts on multiple platforms, under multiple triggers, and separates concerns (lint / unit / E2E / build / release)</button>
<button class="quiz-choice" data-value="C">C) Because each team member owns their own workflow file</button>
<button class="quiz-choice" data-value="D">D) Because GitHub requires one workflow per programming language</button>
</div>
<p class="quiz-explanation">Multiple deployable artifacts (desktop + mobile + dozens of libs), multiple OSes (Linux/macOS/Windows/iOS/Android), multiple triggers (PR / merge / schedule / dispatch / release), and clean separation of concerns — each dimension multiplies workflow count.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> <code>fail-fast: false</code> is set on nearly every E2E matrix in this repo. Why?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It makes the matrix run faster</button>
<button class="quiz-choice" data-value="B">B) GitHub requires it for reusable workflows</button>
<button class="quiz-choice" data-value="C">C) Shards are independent; letting all shards complete produces the full Allure report, which is more valuable than saving runner time</button>
<button class="quiz-choice" data-value="D">D) <code>fail-fast</code> is incompatible with <code>matrix.include</code></button>
</div>
<p class="quiz-explanation">Without <code>fail-fast: false</code>, a shard-3 failure cancels shards 4–12 mid-run and QA loses visibility into what else was broken. Full reports across the whole corpus are worth the extra runner minutes.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q3.</strong> The repo has 27 mobile specs. A user fires a manual run on <code>feat/qaa-702</code> (not a release/hotfix branch). How many iOS shards and Android shards does the matrix contain?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) iOS 27, Android 27 — one shard per spec</button>
<button class="quiz-choice" data-value="B">B) iOS 3, Android 9 — iOS is capped at 3 for non-release manual runs; Android uses BASE_SHARD = ceil(27/3) = 9 under the 12 cap</button>
<button class="quiz-choice" data-value="C">C) iOS 9, Android 9 — BASE_SHARD applies equally to both</button>
<button class="quiz-choice" data-value="D">D) iOS 12, Android 12 — the upper cap is always used</button>
</div>
<p class="quiz-explanation">BASE_SHARD = ceil(27/3) = 9. On manual runs on non-release branches, iOS is capped at 3 (scarce macOS runners), Android at 12. So iOS = min(9, 3) = 3, Android = min(9, 12) = 9.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> Why does <code>shard-tests.mjs</code> sort specs by duration descending before placing them into shards?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Longest-Processing-Time greedy bin-packing minimises the maximum shard duration, which is the wall-clock critical path</button>
<button class="quiz-choice" data-value="B">B) Jest requires specs to be provided in descending duration order</button>
<button class="quiz-choice" data-value="C">C) It optimises cache locality on the runner</button>
<button class="quiz-choice" data-value="D">D) For alphabetical stability in CI logs</button>
</div>
<p class="quiz-explanation">Classic LPT scheduling: place the largest items first and let smaller items fill the gaps. This is provably within 4/3 of optimal and dramatically beats round-robin when durations are skewed.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> How does a value from a <code>workflow_dispatch</code> input end up inside the Detox child process as <code>process.env</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) GitHub Actions injects inputs as env vars automatically</button>
<button class="quiz-choice" data-value="B">B) Through an explicit chain: <code>inputs.*</code> → workflow-level <code>env:</code> → job-level <code>env:</code> → step-level <code>env:</code> (plus <code>$GITHUB_ENV</code> for cross-step values) → child-process inheritance</button>
<button class="quiz-choice" data-value="C">C) Only via repo secrets that Detox reads at runtime</button>
<button class="quiz-choice" data-value="D">D) Via the <code>with:</code> block on <code>actions/checkout</code></button>
</div>
<p class="quiz-explanation">Nothing is automatic. Each layer is explicit and uses <code>${{ inputs.* }}</code> or prior env to propagate values. <code>$GITHUB_ENV</code> is needed only when a value must survive across steps in the same job.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q6.</strong> Which combination validates a release candidate against production Firebase, full suite, both platforms?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>ref=release/3.50.0, tests_type="iOS Only", smoke_tests=true</code></button>
<button class="quiz-choice" data-value="B">B) <code>ref=develop, production_firebase=true</code></button>
<button class="quiz-choice" data-value="C">C) <code>ref=release/3.50.0, tests_type="iOS &amp; Android"</code> with no other changes</button>
<button class="quiz-choice" data-value="D">D) <code>ref=release/3.50.0, tests_type="iOS &amp; Android", production_firebase=true, smoke_tests=false</code></button>
</div>
<p class="quiz-explanation">Release gating needs the full suite (no smoke), both platforms, and production Firebase. Because the ref contains <code>release</code>, both platforms also get the 12-shard cap automatically.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q7.</strong> A test passes locally but fails consistently in CI shard 7. What is the most likely cause, and what is the first thing to check?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) GitHub runners are slower; bump the test timeout and retry</button>
<button class="quiz-choice" data-value="B">B) Allure is dropping events; re-run the workflow</button>
<button class="quiz-choice" data-value="C">C) Another spec earlier in shard 7 leaves state that breaks your test; look at the shard's step log to see which specs ran before yours, then reproduce them locally in the same order</button>
<button class="quiz-choice" data-value="D">D) The Speculos firmware differs on CI; you cannot reproduce locally</button>
</div>
<p class="quiz-explanation">Shard membership is determined by LPT-greedy packing of durations — which specs share a shard with yours is data-dependent and non-alphabetical. State bleed (Redux residue, undelivered async actions, device state) is the #1 cause of "green locally, red in shard N". Replay the shard's spec order locally.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q8.</strong> What role does the <code>@Gate</code> job play in the PR-merge contract?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It throttles CI to one run per PR at a time</button>
<button class="quiz-choice" data-value="B">B) It is the single aggregator check named in branch-protection; required deps must all pass for it to go green, but non-required deps can fail without blocking merge (they only affect Slack reporting)</button>
<button class="quiz-choice" data-value="C">C) It is GitHub's built-in merge queue for this repo</button>
<button class="quiz-choice" data-value="D">D) It replaces the unit-test job on release branches</button>
</div>
<p class="quiz-explanation">Branch-protection pins exactly one required check (<code>@Gate</code>), and the aggregation logic — which deps are required vs advisory — lives in code inside the job. This keeps branch-protection stable as the workflow graph evolves and separates hard-fail checks from flaky-but-informational ones.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## AI Agents & Automation Rules

<div class="chapter-intro">
The Ledger Live repository uses two AI-assisted development tools: <strong>Claude Code</strong> (Anthropic's CLI) and <strong>Cursor</strong> (AI-powered IDE). Both are configured with project-specific rules, agents, commands, and skills that enforce the team's coding standards and accelerate development. As a QA engineer, understanding these configurations helps you use AI tools effectively and understand the guardrails they enforce.
</div>

### 8.4.1 Claude Code in the Ledger Live Repository

**What is Claude Code?**

Claude Code is Anthropic's CLI tool for Claude. It operates directly in your terminal, reads and writes files, runs commands, and assists with software engineering tasks. In Ledger Live, it is configured with specific rules to enforce coding standards, testing practices, and architectural decisions.

**Configuration structure:**

```
CLAUDE.md                        # Root-level project instructions
.claude/
├── rules/                       # Granular rules loaded by context
│   ├── react-mvvm.md           # MVVM architecture enforcement
│   ├── testing.md              # Test conventions (Jest, MSW, etc.)
│   ├── coin-families-contract.md  # Coin family abstraction rules
│   ├── git-workflow.md         # Branch naming, commit format
│   ├── jest-mocks.md           # Mock patterns to avoid
│   ├── redux-slice.md          # RTK slice conventions
│   └── ...
└── settings.json               # Claude Code settings
```

### 8.4.2 Key Claude Rules for QA

| Rule File | What It Enforces |
|-----------|-----------------|
| `testing.md` | Test structure, naming, MSW usage, query priority |
| `jest-mocks.md` | No duplicate mocks, no `restoreAllMocks`, proper lifecycle |
| `react-mvvm.md` | MVVM folder structure, integration test requirements |
| `coin-families-contract.md` | No `if (family === "evm")` in generic code |
| `git-workflow.md` | Conventional commits, branch prefixes |

**Critical rule: `jest-mocks.md`** — `jest.restoreAllMocks()` restores ALL mocks including global ones from `jest-setup.js`. This breaks unrelated tests. Use `jest.clearAllMocks()` (clears call history only) or `mySpy.mockRestore()` (specific spy only).

### 8.4.3 The Socratic Decision Rule (Rodin)

The Ledger Live Claude Code configuration includes a mandatory reasoning framework called "Rodin" that prevents echo-chamber planning:

- Never validates a proposal just because the user proposed it
- Presents the strongest opposing position before agreeing
- Classifies assumptions: `✓ Justified`, `~ Contestable`, `⚡ Simplification`, `◐ Blind spot`, `✗ Unjustified`

**Why this matters for QA**: When using Claude Code to plan test strategy, it will challenge whether a test is necessary, whether the scope is right, and whether the effort is justified. This prevents feature creep in test suites.

### 8.4.4 Cursor IDE in the Ledger Live Repository

**What is Cursor?**

Cursor is an AI-powered IDE (fork of VS Code) that integrates AI assistance directly into the coding experience. The Ledger Live repository has a comprehensive `.cursor/` configuration directory.

**Configuration structure:**

```
.cursor/
├── agents/                      # 5 specialized AI agents
│   ├── ci-watcher.md           # Monitors CI pipelines
│   ├── code-architect.md       # Designs feature architectures
│   ├── code-explorer.md        # Explores and understands codebase
│   ├── code-reviewer.md        # Reviews code changes
│   └── test-runner.md          # Runs and debugs tests
│
├── rules/                       # 14 context-specific rules
│   ├── coin-families-contract.mdc
│   ├── common-commands.mdc
│   ├── cursor-rules.mdc
│   ├── domain-packages.mdc
│   ├── git-workflow.mdc
│   ├── jest-mocks.mdc
│   ├── react-general.mdc
│   ├── react-mvvm.mdc
│   ├── testing.mdc
│   ├── typescript.mdc
│   └── ... (4 more)
```

### 8.4.5 Cursor Agents for QA

The most relevant Cursor agents for QA work:

**`test-runner`**: Runs and debugs tests. Knows how to execute Jest unit tests, Playwright desktop E2E, and Detox mobile E2E. Understands MSW mocking patterns and can fix test failures.

**`ci-watcher`**: Monitors GitHub CI for the current branch. Reports pass/fail with relevant failure logs. Use after pushing to track CI status.

**`code-reviewer`**: Reviews code changes for compliance with MVVM patterns, testing requirements, and project conventions. Use before submitting a PR.

### 8.4.6 Cursor Commands for QA

**`e2e-desktop-onboard`**: A 6-phase guided workflow for onboarding to desktop E2E testing:
1. Explore the E2E architecture
2. Understand fixtures and page objects
3. Read existing tests
4. Write your first test
5. Run with Speculos
6. Debug and iterate

**`e2e-mobile-onboard`**: Similar guided workflow for mobile E2E testing.

**`pre-review`**: Runs 4 parallel review checks before you submit your PR: MVVM compliance, test coverage, code quality, and security.

### 8.4.7 Cursor Skills for QA

**`e2e`** — The skill for adding new coin support to E2E tests. An 8-step process:
1. Identify the coin family
2. Add the coin to the Account enum
3. Create the Speculos app mapping
4. Write the send test
5. Write the receive test
6. Add to CI sharding
7. Tag appropriately
8. Create the changeset

**`e2e-swap`** — Specific skill for swap E2E testing.

**`fix-ci`** — Diagnoses CI failures by reading workflow logs, identifying the root cause, and suggesting fixes.

### 8.4.8 Using AI Tools for QA Tasks

**Common tasks and which tool to use:**

| Task | Tool | Why |
|------|------|-----|
| Write a new E2E test | Claude Code (terminal) or Cursor | Both have project rules; Claude Code starts in plan mode |
| Debug a CI failure | Cursor `ci-watcher` agent or `fix-ci` skill | Integrated log reading |
| Onboard to E2E testing | Cursor `e2e-desktop-onboard` command | Guided 6-phase workflow |
| Review your code before PR | Cursor `pre-review` command | 4 parallel checks |
| Add a new coin to E2E | Cursor `e2e` skill | 8-step guided process |
| Batch create tests | Claude Code Orchestrator mode | Parallel sub-task dispatch |

**Tips for effective AI-assisted QA work:**
- Always review the AI's plan before letting it implement
- Verify generated tests run locally before pushing
- Check that the AI follows existing patterns (page objects, fixtures, decorators)
- Do not blindly trust AI-generated selectors — verify they match the actual DOM
- Use AI for repetitive tasks (generating similar tests for multiple currencies)

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://docs.anthropic.com/en/docs/claude-code">Claude Code documentation</a></li>
<li><a href="https://docs.cursor.com/">Cursor IDE documentation</a></li>
<li><a href="https://docs.cursor.com/context/rules-for-ai">Cursor Rules for AI</a></li>
<li><a href="https://docs.cursor.com/chat/agents">Cursor Agents</a></li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> AI tools are configured with project-specific rules that enforce Ledger Live's standards. They are powerful accelerators for repetitive QA tasks (writing similar tests, debugging failures, adding coin support). Use them — but always verify the output. The <code>~/.claude/rules/</code> and <code>.cursor/rules/</code> directories are the source of truth for how the AI should behave.
</div>

### 8.4.9 Quiz

<!-- ── Chapter 8.4 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> What is the purpose of <code>~/.claude/rules/</code> and <code>.cursor/rules/</code> files?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) They are documentation files for new developers</button>
<button class="quiz-choice" data-value="B">B) They are configuration rules that shape how AI agents behave and enforce project-specific coding standards</button>
<button class="quiz-choice" data-value="C">C) They are test configuration files for CI</button>
<button class="quiz-choice" data-value="D">D) They are Git hooks that run on commit</button>
</div>
<p class="quiz-explanation">These rule files instruct AI tools on project-specific conventions, patterns, and constraints. When the AI assists with development, it follows these rules to produce code that matches the team's standards.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> What does the Socratic Decision Rule (Rodin) prevent?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It prevents the AI from writing code</button>
<button class="quiz-choice" data-value="B">B) It prevents echo-chamber planning by requiring independent reasoning, counterarguments, and assumption classification</button>
<button class="quiz-choice" data-value="C">C) It prevents the AI from reading files</button>
<button class="quiz-choice" data-value="D">D) It prevents the AI from using Git</button>
</div>
<p class="quiz-explanation">Rodin ensures the AI provides independent reasoning, presents the strongest opposing position, and classifies assumptions rather than blindly agreeing with proposals.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> Which Cursor skill would you use to add a new cryptocurrency to the E2E test suite?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>fix-ci</code></button>
<button class="quiz-choice" data-value="B">B) <code>create-changeset</code></button>
<button class="quiz-choice" data-value="C">C) <code>e2e-swap</code></button>
<button class="quiz-choice" data-value="D">D) <code>e2e</code> — an 8-step guided process for adding new coin support</button>
</div>
<p class="quiz-explanation">The <code>e2e</code> skill guides you through 8 steps: identify coin family, add to Account enum, create Speculos mapping, write send/receive tests, add to CI sharding, tag, and create changeset.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q4.</strong> Why is <code>jest.restoreAllMocks()</code> dangerous according to the <code>jest-mocks.md</code> rule?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It is deprecated in recent Jest versions</button>
<button class="quiz-choice" data-value="B">B) It only works with TypeScript, not JavaScript</button>
<button class="quiz-choice" data-value="C">C) It restores ALL mocks including global ones from <code>jest-setup.js</code>, breaking unrelated tests that depend on those mocks</button>
<button class="quiz-choice" data-value="D">D) It increases test execution time significantly</button>
</div>
<p class="quiz-explanation"><code>restoreAllMocks()</code> removes all mocks including global setup for navigation, i18n, and native modules. Other tests that depend on these global mocks will break. Use <code>clearAllMocks()</code> or restore only your specific spies.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q5.</strong> You want to use AI to write E2E tests for 10 different currencies. Which approach is most efficient?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Claude Code Orchestrator mode — breaks work into sub-tasks (one per currency) and dispatches them in parallel</button>
<button class="quiz-choice" data-value="B">B) Write each test manually and use AI only for review</button>
<button class="quiz-choice" data-value="C">C) Use Cursor's <code>fix-ci</code> skill</button>
<button class="quiz-choice" data-value="D">D) Copy-paste one test 10 times and change the currency name</button>
</div>
<p class="quiz-explanation">The Orchestrator mode can break the work into parallel sub-tasks, using existing page objects and the parameterized test pattern. This is much more efficient than writing 10 tests sequentially.</p>
</div>

<div class="quiz-score"></div>
</div>

---


## Part 8 Final Assessment

You have reached the end of the guide. Part 7 covered the disciplines that separate a competent E2E contributor from a senior one. Chapter 8.1 taught test strategy — the pyramid, page object discipline, data-driven patterns, and flaky test management. Chapter 8.2 walked you through debugging failures across desktop (Playwright Inspector, Trace Viewer, Allure) and mobile (Detox verbose logs, `adb logcat`, synchronization bypass), plus Speculos troubleshooting. Chapter 8.3 unpacked CI/CD anatomy, the `@Gate` job, composite actions, and the shard matrix algorithm. Chapter 8.4 introduced the AI tooling around the repo — Claude Code, Cursor agents, and the Socratic Decision Rule (Rodin) that keeps planning honest.

The ten questions below cross-cut all four chapters. Treat this assessment as a self-audit: if you can answer every question with confidence, you are ready to own E2E work end-to-end — from ticket intake, through local Speculos reproduction, through shard-aware CI, to a green `@Gate` ready to merge. Passing threshold is 80%.

<div class="quiz-container" data-pass-threshold="80">
<h3>Part 8 Final Assessment</h3>
<p class="quiz-subtitle">10 questions · 80% to pass · Covers Ch 7.1 through 6.4</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="A">
<p><strong>Q1.</strong> In the Ledger testing pyramid, which layer should contain the fewest tests and why?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) E2E with Speculos — highest confidence but slowest (~30s-5min per test) and most expensive to maintain, so reserved for critical user journeys</button>
<button class="quiz-choice" data-value="B">B) Unit tests — they are too isolated to catch real bugs</button>
<button class="quiz-choice" data-value="C">C) Integration tests — they duplicate unit coverage</button>
<button class="quiz-choice" data-value="D">D) All layers should have the same number of tests to guarantee uniform coverage</button>
</div>
<p class="quiz-explanation">E2E sits at the narrow tip of the pyramid. Each test takes 30s-5min, involves Speculos and a full Electron or React Native stack, and is the most expensive to debug when flaky. Unit tests (~1ms) form the wide base, integration tests (~100ms) sit in the middle. E2E should cover only critical paths that lower layers cannot validate — onboarding, send, receive, portfolio aggregation.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> A reviewer flags this line in your new page object: <code>await portfolioPage.assertBalanceIs("1.5 BTC")</code>. Why is this a page object discipline violation?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The method name does not start with a verb</button>
<button class="quiz-choice" data-value="B">B) It should have used <code>data-testid</code> instead of role locators</button>
<button class="quiz-choice" data-value="C">C) Assertions belong in tests, not page objects — a PO that owns an assertion cannot be reused by a test expecting a different balance</button>
<button class="quiz-choice" data-value="D">D) <code>assertBalanceIs</code> is missing the <code>@step</code> decorator</button>
</div>
<p class="quiz-explanation">Page objects provide actions and data (e.g. <code>getBalance()</code>); tests decide what to assert. An assertion baked into the PO ties it to one expected value and destroys reuse. The rule is: PO returns, test asserts. Soft visibility checks like <code>isVisible()</code> inside a PO are acceptable; hard equality assertions are not.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q3.</strong> You need to cover send flows for Bitcoin, Ethereum, and Solana. What is the idiomatic way to express this in the Ledger E2E suite?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Copy one test three times and change the currency name in each file</button>
<button class="quiz-choice" data-value="B">B) Define a parameterized array of <code>{ name, ticker, amount, address }</code> and loop <code>for (const currency of currencies) { test(\`can send ${currency.name}\`, ...) }</code></button>
<button class="quiz-choice" data-value="C">C) Write one monolithic test that sends all three currencies sequentially</button>
<button class="quiz-choice" data-value="D">D) Use a manual test plan — data-driven E2E is not supported by Playwright</button>
</div>
<p class="quiz-explanation">Data-driven testing keeps one test body and iterates over a parameter array, generating a distinct test per currency with a unique name. It avoids copy-paste drift, makes the coverage matrix visible in one place, and lets each generated test fail or shard independently. Playwright registers each loop iteration as a separate test case.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> A CI test fails with <code>Error: connect ECONNREFUSED 127.0.0.1:5000</code>. Which diagnosis and fix path is correct?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The test assertion is wrong — update the expected value</button>
<button class="quiz-choice" data-value="B">B) Playwright locator is stale — switch to <code>getByTestId</code></button>
<button class="quiz-choice" data-value="C">C) The Allure reporter crashed — rerun the workflow</button>
<button class="quiz-choice" data-value="D">D) The Speculos container is not reachable on port 5000 — check the Docker daemon, verify <code>docker ps</code> shows the container, and curl <code>/events</code> for the health check</button>
</div>
<p class="quiz-explanation"><code>ECONNREFUSED :5000</code> is a Speculos connectivity issue, not a test logic issue. Verify the container is running (<code>docker ps | grep speculos</code>), inspect logs (<code>docker logs &lt;id&gt;</code>), and confirm the health endpoint responds (<code>curl http://localhost:5000/events</code>) before touching test code. A 404 on <code>/apdu</code> usually means wrong Speculos version; a stuck home screen means wrong coin app <code>.elf</code>.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> You are debugging a Detox test on Android that fails because a loading spinner animates forever. Which tool or technique is the right first move?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Increase the global Jest timeout to 5 minutes</button>
<button class="quiz-choice" data-value="B">B) Wrap the interaction in <code>device.disableSynchronization()</code> / <code>enableSynchronization()</code> so Detox stops waiting for idle, then use <code>--loglevel trace</code> and <code>adb logcat | grep ReactNative</code> to confirm behavior</button>
<button class="quiz-choice" data-value="C">C) Use the Playwright Trace Viewer to inspect the animation</button>
<button class="quiz-choice" data-value="D">D) Delete the test — infinite animations cannot be tested</button>
</div>
<p class="quiz-explanation">Detox synchronizes on the app's idle state; an infinite animation blocks that forever. The documented escape hatch is <code>disableSynchronization()</code> around the offending interaction. Trace logging plus <code>adb logcat</code> confirms what the native layer is doing. Playwright's Trace Viewer is desktop-only and does not apply to the Detox / React Native runtime.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> In GitHub Actions terminology, which relationship is correct?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A step contains jobs, and jobs contain workflows</button>
<button class="quiz-choice" data-value="B">B) A workflow is a single step with multiple jobs inside</button>
<button class="quiz-choice" data-value="C">C) A workflow contains one or more jobs; each job runs on a runner and executes an ordered list of steps</button>
<button class="quiz-choice" data-value="D">D) Workflows, jobs, and steps are synonyms for the same YAML block</button>
</div>
<p class="quiz-explanation">The hierarchy is workflow → jobs → steps. A workflow is the top-level YAML file triggered by events (<code>pull_request</code>, <code>schedule</code>, <code>workflow_dispatch</code>). It defines one or more jobs, which each run on a runner (e.g. <code>ubuntu-latest</code>). Each job executes steps in order. A matrix strategy fans a single job definition out into multiple parallel copies.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q7.</strong> Your PR shows 23 CI checks. One non-required job is red, but <code>@Gate</code> is green. What is the correct interpretation?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The PR is mergeable because <code>@Gate</code> is the single aggregation point tracked by branch protection; the failing non-required job should still be reported on Slack for follow-up</button>
<button class="quiz-choice" data-value="B">B) The PR is blocked until every single check is green</button>
<button class="quiz-choice" data-value="C">C) <code>@Gate</code> is irrelevant — only individual job statuses matter</button>
<button class="quiz-choice" data-value="D">D) The PR must be closed and reopened to re-trigger CI</button>
</div>
<p class="quiz-explanation">Only <code>@Gate</code> is tracked by branch protection rules. A green <code>@Gate</code> means all required jobs passed and the PR can merge. Non-required failures do not block merging but must be surfaced so the team can investigate. That design keeps the merge signal simple while preserving visibility into secondary checks.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q8.</strong> A matrix with <code>shardIndex: [1, 2, 3, 4, 5]</code>, <code>shardTotal: [5]</code>, and <code>fail-fast: false</code> is used for E2E. Which statement describes its behavior most accurately?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It runs the full suite five times in sequence on one runner</button>
<button class="quiz-choice" data-value="B">B) It fans out into 5 parallel runners, each executing ~20% of tests via <code>--shard=i/5</code>; if one shard fails the others still complete, and a downstream merge job combines Allure results</button>
<button class="quiz-choice" data-value="C">C) It runs 5 shards and stops all of them the moment one fails</button>
<button class="quiz-choice" data-value="D">D) It creates 25 runners (5 × 5) due to the two matrix axes</button>
</div>
<p class="quiz-explanation">Playwright's <code>--shard=i/n</code> flag deterministically partitions the test list into <code>n</code> slices; each runner executes slice <code>i</code>. The matrix expands to 5 parallel jobs (one per <code>shardIndex</code>, with <code>shardTotal</code> fixed at 5). <code>fail-fast: false</code> ensures all shards finish so you get a complete merged Allure report — critical when diagnosing broad regressions that affect multiple shards.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q9.</strong> You want to understand a legacy E2E fixture, study its patterns, and get exercises that let you rebuild it yourself. Which agent mode is the best fit, and why?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>/pair-programmer</code> — it is the only agent that can read files</button>
<button class="quiz-choice" data-value="B">B) <code>/docker</code> — fixtures run inside containers</button>
<button class="quiz-choice" data-value="C">C) No agent — always work directly on <code>dev</code></button>
<button class="quiz-choice" data-value="D">D) <code>/pedagogy</code> — it is the teaching-focused mode that explains decisions, walks through concepts, and proposes exercises; <code>/pair-programmer</code> is for TDD Red-Green-Refactor delivery, not learning</button>
</div>
<p class="quiz-explanation">The two modes have distinct purposes. <code>/pedagogy</code> is for understanding and learning: explanations, trade-offs, exercises you complete yourself. <code>/pair-programmer</code> is for building features under strict TDD discipline (Red → Green → Refactor). Picking the wrong one gives you tests when you wanted a lesson, or a lecture when you wanted a feature. Small single-file changes need no agent at all — work directly on <code>dev</code>.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q10.</strong> Under the Socratic Decision Rule (Rodin), you propose adding a new flaky-test retry mechanism. The agent agrees three times in a row without pushing back. What should happen next?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Nothing — three agreements confirm the plan is correct</button>
<button class="quiz-choice" data-value="B">B) The agent must stop and run a contradiction pass: steelman the strongest opposing position (e.g. "fix the root cause instead of adding retries"), classify assumptions with tags like <code>~ Contestable</code> or <code>◐ Blind spot</code>, and keep only <code>✓ Justified</code> items in the plan</button>
<button class="quiz-choice" data-value="C">C) The user must rephrase the proposal until the agent disagrees</button>
<button class="quiz-choice" data-value="D">D) Rodin only applies to code review, not to planning</button>
</div>
<p class="quiz-explanation">Rodin is explicit: three validations in a row triggers a mandatory contradiction pass. The agent must steelman the counter-position, classify each assumption with the Rodin tags (<code>✓ Justified</code>, <code>~ Contestable</code>, <code>⚡ Simplification</code>, <code>◐ Blind spot</code>, <code>✗ Unjustified</code>), and default to implementing only <code>✓ Justified</code> items. This prevents echo-chamber planning and protects the suite from feature creep — exactly the risk when adding retries instead of fixing the flaky root cause.</p>
</div>

<div class="quiz-score"></div>
</div>

Congratulations — you have completed the QA Onboarding Guide. You now understand the full path from a Xray ticket to a green `@Gate`: where a test belongs in the pyramid, how to keep page objects disciplined, how to parameterize coverage, how to read desktop and mobile failures systematically, how CI workflows fan out across shards, and how to work with AI agents under the Socratic Decision Rule without ceding judgment to them.

For day-to-day reference, keep the [Appendix](appendix.md) close: it collects the command cheat sheets, locator patterns, Speculos recipes, fixture anatomy, and CI artifact recovery steps you will reach for most often. From here, pick up a ticket, branch from `dev` with a `feat/`, `bugfix/`, or `support/` prefix, write the test, reproduce the failure locally before pushing, and let `@Gate` be your final check. Welcome to the team.
