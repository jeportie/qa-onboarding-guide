

# PART 4 -- CI/CD & MASTERY

<div class="chapter-intro">
You can write tests. Now you need to understand the system that runs them, the strategies that make them effective, the techniques that debug them when they fail, and the AI tools that accelerate your work. Part 4 covers the CI/CD pipeline, test strategy, debugging methodology, contribution workflow, and the AI agent configurations (Claude Code and Cursor) that shape how development happens in this repository.
</div>

---

## GitHub Actions Workflows

<div class="chapter-intro">
The Ledger Live repository uses <strong>GitHub Actions</strong> as its CI/CD platform. With over <strong>73 workflow files</strong>, this is one of the most complex CI configurations you will encounter. This chapter explains the workflow anatomy, the E2E test pipelines, sharding, composite actions, and how to read CI logs and artifacts when things fail.
</div>

### 17.1 CI/CD in the Ledger Live Monorepo

**Why 73+ workflows?**

- The monorepo contains multiple deployable artifacts (desktop app, mobile app, dozens of npm libraries)
- E2E tests run on multiple platforms (Linux, macOS, Windows for desktop; iOS and Android for mobile)
- Different triggers: pull requests, merges to `develop`, nightly schedules, manual dispatches
- Separation of concerns: linting, building, unit tests, E2E tests, deployments are all independent

**Key directories:**

```
.github/
├── workflows/           # 73+ YAML workflow definitions
│   ├── build-desktop-*.yml
│   ├── build-mobile-*.yml
│   ├── test-desktop-*.yml
│   ├── test-mobile-*.yml
│   ├── bot-*.yml        # Ledger Live Bot workflows
│   ├── release-*.yml    # Release automation
│   └── ...
├── actions/             # Reusable composite actions
│   ├── setup-build/
│   ├── setup-test-desktop/
│   ├── setup-speculos/
│   └── ...
└── scripts/             # Helper scripts for CI
```

### 17.2 Workflow Anatomy

Every workflow file follows the same YAML structure:

```yaml
name: "[Desktop] E2E Tests - Speculos"

on:
  workflow_dispatch:        # Manual trigger with inputs
    inputs:
      branch:
        description: "Branch to test"
        required: false
  pull_request:
    branches: [develop, main]
    paths:
      - "apps/ledger-live-desktop/**"
      - "libs/ledger-live-common/**"
  schedule:
    - cron: "0 3 * * 1-5"  # Weekdays at 3 AM

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  e2e-tests:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shardIndex: [1, 2, 3, 4, 5]
        shardTotal: [5]
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-build
      - name: Run E2E tests
        run: |
          cd apps/ledger-live-desktop
          pnpm e2e:speculos --shard=${{ matrix.shardIndex }}/${{ matrix.shardTotal }}
```

**Key concepts:**

| Concept | Purpose |
|---------|---------|
| `on.workflow_dispatch` | Allows manual runs from the GitHub UI with custom inputs |
| `on.pull_request.paths` | Only triggers when files in specific paths change |
| `on.schedule` | Cron-based triggers for nightly/periodic runs |
| `concurrency` | Prevents duplicate runs; cancels older runs on the same branch |
| `strategy.matrix` | Runs the job in parallel across multiple configurations |
| `fail-fast: false` | Continues all shards even if one fails |
| Composite actions | Reusable step bundles in `.github/actions/` |

### 17.3 E2E Test Workflows in Detail

**Desktop E2E workflows:**

| Workflow | Trigger | What it does |
|----------|---------|-------------|
| `test-desktop-e2e-speculos.yml` | PR, nightly | Runs Speculos-based tests with sharding |
| `test-desktop-e2e-mock.yml` | PR | Runs mocked E2E tests (no device) |
| `test-desktop-e2e-full.yml` | Manual | Full suite with real device firmware |

**Mobile E2E workflows:**

| Workflow | Trigger | What it does |
|----------|---------|-------------|
| `test-mobile-e2e-android.yml` | PR, nightly | Runs Detox tests on Android emulator |
| `test-mobile-e2e-ios.yml` | PR, nightly | Runs Detox tests on iOS simulator |

**Bot workflows:**

| Workflow | Trigger | What it does |
|----------|---------|-------------|
| `bot-nonreg-nitrogen.yml` | Daily | Non-regression with SEED7 on `develop` |
| `bot-nonreg-oxygen.yml` | Weekly | Costly non-regression with SEED5 on `develop` |
| `bot-nonreg-carbone.yml` | Daily (weekdays) | Non-regression on `main` with staging explorers |

**How sharding works in CI:**

```yaml
strategy:
  fail-fast: false
  matrix:
    shardIndex: [1, 2, 3, 4, 5]
    shardTotal: [5]
```

This creates 5 parallel runners. Each runs ~20% of the tests. After all shards complete, a merge job collects and combines the results:

```yaml
merge-reports:
  needs: e2e-tests
  if: always()
  steps:
    - name: Merge Allure results
      uses: ./.github/actions/merge-allure-reports
    - name: Upload merged report
      uses: actions/upload-artifact@v4
```

### 17.4 The @Gate Job

The `@Gate` job is the required CI check that blocks PR merging. It acts as a single aggregation point that verifies all required jobs have passed.

- If `@Gate` passes → the PR can be merged
- If a non-required job fails but `@Gate` passes → the PR can still be merged, but failures should be reported on Slack
- Only `@Gate` is tracked by branch protection rules

### 17.5 Reusable Composite Actions

Instead of duplicating setup steps across 73+ workflows, the repo uses **composite actions**:

```
.github/actions/
├── setup-build/          # Install pnpm, node, restore caches, install deps
├── setup-test-desktop/   # Build desktop app for testing
├── setup-speculos/       # Pull Speculos Docker image, start containers
└── upload-allure-results/ # Upload test artifacts for reporting
```

You reference them in any workflow with:
```yaml
- uses: ./.github/actions/setup-build
```

When the setup process changes (e.g., new Node.js version), you update one `action.yml` file instead of 73+.

### 17.6 Reading CI Logs and Artifacts

When a CI run fails:

1. **Find the failed run**: PR's "Checks" tab → click the failed check
2. **Identify the failed step**: Expand the failed job → look for the red X
3. **Read the logs**: Click the failed step and scan for the error (usually near the bottom)
4. **Download artifacts**: Workflow run summary → scroll to "Artifacts" → download
5. **View Allure report locally**:

```bash
unzip allure-results-shard-3.zip -d allure-results/
npx allure generate allure-results -o allure-report --clean
npx allure open allure-report
```

**Common CI error patterns:**

| Error | Meaning |
|-------|---------|
| `TimeoutError: locator.click` | Element not found in time |
| `expect(received).toHaveText(expected)` | Assertion mismatch |
| `ECONNREFUSED :5000` | Speculos container not started |
| `OCI runtime create failed` | Docker out of disk space |

### 17.7 Common CI Troubleshooting (from the wiki)

**Node Action Cache Error**: If you see cache-related failures in CI, it may be an issue with the GitHub `cache` action, not the code. The team has migrated to their own S3 cache, but legacy workflows may still be affected.

**Solution**: If cache errors persist, temporarily disable pnpm cache in the `node-setup` action by commenting out the `cache` and `cache-dependency-path` lines.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://docs.github.com/en/actions">GitHub Actions documentation</a></li>
<li><a href="https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/using-a-matrix-for-your-jobs">GitHub Actions Matrix Strategy</a></li>
<li><a href="https://docs.github.com/en/actions/sharing-automations/creating-actions/creating-a-composite-action">Creating Composite Actions</a></li>
<li><a href="https://crontab.guru/">Crontab Guru</a> — interactive cron expression editor</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> You will rarely need to create workflows from scratch — most of your CI interaction will be reading logs, downloading artifacts, and understanding why a shard failed. Know where to find the Allure report URL, how to download artifacts, and how to reproduce failures locally. The <code>@Gate</code> job is your single source of truth for whether a PR is ready.
</div>

### 17.8 Quiz

<!-- ── Chapter 17 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> Why does the Ledger Live repo have 73+ workflow files instead of a single CI pipeline?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because GitHub Actions limits each file to 10 jobs</button>
<button class="quiz-choice" data-value="B">B) Because the monorepo produces multiple artifacts on multiple platforms with different triggers and separation of concerns</button>
<button class="quiz-choice" data-value="C">C) Because each developer has their own workflow file</button>
<button class="quiz-choice" data-value="D">D) Because GitHub requires one workflow per programming language</button>
</div>
<p class="quiz-explanation">The monorepo contains desktop and mobile apps plus dozens of libraries, each needing different build/test/deploy steps across multiple platforms (Linux, macOS, Windows, iOS, Android) and triggers (PR, nightly, manual, release).</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> What does <code>concurrency.cancel-in-progress: true</code> do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It cancels all running workflows in the repository</button>
<button class="quiz-choice" data-value="B">B) It cancels the previous run of the same workflow on the same branch when a new one starts</button>
<button class="quiz-choice" data-value="C">C) It prevents any concurrent jobs from running</button>
<button class="quiz-choice" data-value="D">D) It cancels the current run if a newer one is queued</button>
</div>
<p class="quiz-explanation">When you push a new commit while CI is still running for the previous commit on the same branch, the older run is cancelled to save resources. The concurrency group is typically <code>workflow + ref</code>.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q3.</strong> Why is <code>fail-fast: false</code> used in sharded test workflows?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) To make tests run faster</button>
<button class="quiz-choice" data-value="B">B) To ensure all shards complete and produce results, even if one shard has failures</button>
<button class="quiz-choice" data-value="C">C) To save CI minutes by stopping early</button>
<button class="quiz-choice" data-value="D">D) Because sharded tests cannot fail individually</button>
</div>
<p class="quiz-explanation">Without <code>fail-fast: false</code>, a failure in shard 1 would cancel shards 2-5, and you'd lose visibility into failures in those shards. Full results from all shards are needed for the merged Allure report.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q4.</strong> What is the benefit of composite actions over copying steps into every workflow?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) They run faster than regular steps</button>
<button class="quiz-choice" data-value="B">B) They provide a single source of truth for shared setup logic — one update propagates to all consumers</button>
<button class="quiz-choice" data-value="C">C) GitHub gives a discount for using composite actions</button>
<button class="quiz-choice" data-value="D">D) They are required by the repository's branch protection rules</button>
</div>
<p class="quiz-explanation">When the setup process changes (e.g., a new Node.js version, a new cache strategy), you update one <code>action.yml</code> file instead of modifying 73+ workflow files.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> A workflow uses <code>matrix.shardIndex: [1,2,3,4,5]</code> and <code>matrix.shardTotal: [5]</code>. How many parallel runners does this create?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) 1</button>
<button class="quiz-choice" data-value="B">B) 2</button>
<button class="quiz-choice" data-value="C">C) 5</button>
<button class="quiz-choice" data-value="D">D) 25</button>
</div>
<p class="quiz-explanation">The matrix creates one job per <code>shardIndex</code> value (1, 2, 3, 4, 5), so 5 parallel runners. <code>shardTotal: [5]</code> is a single value passed to each shard so it knows the total count.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Test Strategy & Best Practices

<div class="chapter-intro">
Knowing how to write a test is not enough — you need to know <strong>which</strong> tests to write, <strong>how</strong> to structure them, and <strong>what</strong> to do when they become flaky. This chapter covers the testing pyramid, naming conventions, page object best practices, data-driven testing, and flaky test management. These are the principles that separate a test suite that helps from one that hinders.
</div>

### 18.1 The Testing Pyramid at Ledger

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

### 18.2 Test Naming Conventions

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

### 18.3 Page Object Best Practices

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

### 18.4 Data-Driven Testing

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

### 18.5 Flaky Test Management

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

### 18.6 Quiz

<!-- ── Chapter 18 Quiz ── -->

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

### 19.1 The Debugging Mindset

```
FAIL → Read the error → Check the screenshot/video → Reproduce locally → Fix
```

**Never:**
- Re-run without reading the error
- Add `waitForTimeout()` as a fix
- Delete a test because it's "flaky"

### 19.2 Common Error Patterns

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

### 19.3 Playwright Debugging Tools

**Playwright Inspector** (headed debug mode):
```bash
PWDEBUG=1 pnpm e2e:desktop test:playwright your-test.spec.ts
```
Opens a browser with an inspector panel — step through actions, inspect locators live, see resolved elements. See Part 5, Chapter 22.8 for a detailed walkthrough with screenshots.

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

### 19.4 Detox Debugging Tools

**Verbose logging:**
```bash
detox test --loglevel trace -c android.debug
adb logcat | grep "ReactNative"
```

**Synchronization bypass** (for infinite animations):
```typescript
await device.disableSynchronization();
await element(by.id("animated-component")).tap();
await device.enableSynchronization();
```

### 19.5 Speculos Debugging

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

### 19.6 Common Failure Patterns

| Pattern | Symptom | Root Cause | Solution |
|---------|---------|-----------|----------|
| Timing | `TimeoutError` on existing element | Element renders late | Wait for prerequisite first |
| Stale data | Test sees old balance | Previous test pollution | Use fresh userdata |
| Layout shift | Click lands wrong | Element moved during click | Wait for animation completion |
| Locale | `"1,5 BTC"` vs `"1.5 BTC"` | System locale differs | Set locale in config |
| Resolution | Element off-screen | Different CI viewport | Set viewport in config |
| Docker | `OCI runtime create failed` | Out of disk space | `docker system prune` |

### 19.7 LLD Debugging Tips (from the wiki)

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

<!-- No quiz for Ch 19 — per Rodin evaluation, debugging is best learned by doing, not by QCM -->

---

## Contributing Back -- Writing Production-Ready Tests

<div class="chapter-intro">
This chapter covers the complete workflow for contributing new E2E tests to the Ledger Live repository: from picking a test case to merging your PR. You will learn the anatomy of a production-ready test, the PR checklist, and what reviewers look for.
</div>

### 20.1 The Test Contribution Workflow

```
1. Pick a user story or test case from Xray
2. Create a feature branch from develop
3. Write the test following patterns from Part 3
4. Run locally with Speculos
5. Add Allure annotations (TMS link, tags, description)
6. Create a PR targeting develop
7. CI runs your test automatically
8. Address review feedback
9. Merge when approved + @Gate green
```

### 20.2 PR Checklist

Before opening your PR, verify:

- [ ] Test has a clear, descriptive name
- [ ] Tags are present (`@smoke`, `@currency`, `@B2CQA-XXX`)
- [ ] Uses page objects (no raw locators in test body)
- [ ] Page object methods have `@step` decorators
- [ ] Runs successfully locally with Speculos
- [ ] No hardcoded waits (`page.waitForTimeout`)
- [ ] Userdata is appropriate for the test scenario
- [ ] Test is independent (can run in isolation or in any order)
- [ ] No sensitive data (seeds, private keys) in the test
- [ ] Feature flags are explicitly overridden (not relying on defaults)

### 20.3 Code Review Expectations

| Area | What reviewers check |
|------|---------------------|
| Naming | Test name describes user journey, not implementation |
| Page objects | Uses existing POs; new methods have `@step` |
| Locators | Uses `data-testid`, roles, text; avoids CSS selectors |
| Assertions | Specific (`toHaveText("1.5 BTC")` not `toBeTruthy()`) |
| Independence | No dependency on other tests running first |
| Flakiness | No `waitForTimeout`, proper waits |
| Tags | Xray TMS link present, device and currency tags |

### 20.4 When You Need a Changeset

- **Need one**: Modified files in `libs/`, shared test utilities, E2E infrastructure (fixtures, base classes)
- **Don't need one**: Test files only, test data (userdata profiles), test configuration

<div class="chapter-outro">
<strong>Key takeaway:</strong> A production-ready test is not just a test that passes. It is readable, independent, properly tagged, non-flaky, and uses the established patterns. Follow the checklist and your PRs will sail through review.
</div>

<!-- No quiz for Ch 20 — per Rodin evaluation, contribution workflow is process-oriented -->

---

## AI Agents & Automation Rules

<div class="chapter-intro">
The Ledger Live repository uses two AI-assisted development tools: <strong>Claude Code</strong> (Anthropic's CLI) and <strong>Cursor</strong> (AI-powered IDE). Both are configured with project-specific rules, agents, commands, and skills that enforce the team's coding standards and accelerate development. As a QA engineer, understanding these configurations helps you use AI tools effectively and understand the guardrails they enforce.
</div>

### 21.1 Claude Code in the Ledger Live Repository

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

### 21.2 Key Claude Rules for QA

| Rule File | What It Enforces |
|-----------|-----------------|
| `testing.md` | Test structure, naming, MSW usage, query priority |
| `jest-mocks.md` | No duplicate mocks, no `restoreAllMocks`, proper lifecycle |
| `react-mvvm.md` | MVVM folder structure, integration test requirements |
| `coin-families-contract.md` | No `if (family === "evm")` in generic code |
| `git-workflow.md` | Conventional commits, branch prefixes |

**Critical rule: `jest-mocks.md`** — `jest.restoreAllMocks()` restores ALL mocks including global ones from `jest-setup.js`. This breaks unrelated tests. Use `jest.clearAllMocks()` (clears call history only) or `mySpy.mockRestore()` (specific spy only).

### 21.3 The Socratic Decision Rule (Rodin)

The Ledger Live Claude Code configuration includes a mandatory reasoning framework called "Rodin" that prevents echo-chamber planning:

- Never validates a proposal just because the user proposed it
- Presents the strongest opposing position before agreeing
- Classifies assumptions: `✓ Justified`, `~ Contestable`, `⚡ Simplification`, `◐ Blind spot`, `✗ Unjustified`

**Why this matters for QA**: When using Claude Code to plan test strategy, it will challenge whether a test is necessary, whether the scope is right, and whether the effort is justified. This prevents feature creep in test suites.

### 21.4 Cursor IDE in the Ledger Live Repository

**What is Cursor?**

Cursor is an AI-powered IDE (fork of VS Code) that integrates AI assistance directly into the coding experience. The Ledger Live repository has a comprehensive `.cursor/` configuration directory.

**Configuration structure:**

```
.cursor/
├── agents/                      # 5 specialized AI agents
│   ├── ci-watcher.mdc          # Monitors CI pipelines
│   ├── code-architect.mdc      # Designs feature architectures
│   ├── code-explorer.mdc       # Explores and understands codebase
│   ├── code-reviewer.mdc       # Reviews code changes
│   └── test-runner.mdc         # Runs and debugs tests
│
├── commands/                    # 7 automation commands
│   ├── cleanup.mdc             # Code cleanup
│   ├── create-pr.mdc           # PR creation workflow
│   ├── e2e-desktop-onboard.mdc # 6-phase desktop E2E onboarding
│   ├── e2e-mobile-onboard.mdc  # Mobile E2E onboarding
│   ├── feature-dev.mdc         # 7-phase feature development
│   ├── pre-review.mdc          # 4 parallel review checks
│   └── test-coverage.mdc       # Coverage analysis
│
├── rules/                       # 25 context-specific rules
│   ├── client-ids.md
│   ├── coin-families-contract.md
│   ├── cursor-rules.md
│   ├── dialogs-slice.md
│   ├── git-workflow.md
│   ├── jest-mocks.md
│   ├── react-mvvm.md
│   ├── testing.md
│   ├── typescript.md
│   ├── zod-schemas.md
│   └── ... (15 more)
│
└── skills/                      # 8 reusable skills
    ├── create-changeset.mdc    # Changeset creation
    ├── e2e-swap.mdc            # Swap E2E testing
    ├── e2e.mdc                 # Adding new coins to E2E (8 steps)
    ├── fix-ci.mdc              # CI failure diagnosis
    ├── get-pr-comments.mdc     # PR comment retrieval
    ├── run-tests.mdc           # Test execution
    ├── slack-pr-message.mdc    # PR notification to Slack
    └── testing.mdc             # General testing skill
```

### 21.5 Cursor Agents for QA

The most relevant Cursor agents for QA work:

**`test-runner`**: Runs and debugs tests. Knows how to execute Jest unit tests, Playwright desktop E2E, and Detox mobile E2E. Understands MSW mocking patterns and can fix test failures.

**`ci-watcher`**: Monitors GitHub CI for the current branch. Reports pass/fail with relevant failure logs. Use after pushing to track CI status.

**`code-reviewer`**: Reviews code changes for compliance with MVVM patterns, testing requirements, and project conventions. Use before submitting a PR.

### 21.6 Cursor Commands for QA

**`e2e-desktop-onboard`**: A 6-phase guided workflow for onboarding to desktop E2E testing:
1. Explore the E2E architecture
2. Understand fixtures and page objects
3. Read existing tests
4. Write your first test
5. Run with Speculos
6. Debug and iterate

**`e2e-mobile-onboard`**: Similar guided workflow for mobile E2E testing.

**`pre-review`**: Runs 4 parallel review checks before you submit your PR: MVVM compliance, test coverage, code quality, and security.

### 21.7 Cursor Skills for QA

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

### 21.8 Using AI Tools for QA Tasks

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
<strong>Key takeaway:</strong> AI tools are configured with project-specific rules that enforce Ledger Live's standards. They are powerful accelerators for repetitive QA tasks (writing similar tests, debugging failures, adding coin support). Use them — but always verify the output. The <code>.claude/rules/</code> and <code>.cursor/rules/</code> directories are the source of truth for how the AI should behave.
</div>

### 21.9 Quiz

<!-- ── Chapter 21 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> What is the purpose of <code>.claude/rules/</code> and <code>.cursor/rules/</code> files?</p>
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

## Part 4 Final Assessment

<div class="quiz-container" data-pass-threshold="80">
<h3>Part 4 Final Assessment</h3>
<p class="quiz-subtitle">10 questions across all chapters · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> What does <code>concurrency.cancel-in-progress: true</code> do in a GitHub Actions workflow?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Runs all jobs concurrently without limits</button>
<button class="quiz-choice" data-value="B">B) Cancels the previous run of the same workflow on the same branch when a new one starts</button>
<button class="quiz-choice" data-value="C">C) Limits total concurrent jobs across all workflows</button>
<button class="quiz-choice" data-value="D">D) Prevents workflows from running on weekends</button>
</div>
<p class="quiz-explanation">It groups runs by workflow + ref and cancels older in-progress runs when a new one is queued for the same group, saving CI resources.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> In the testing pyramid, which tests should be the MOST numerous?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) E2E tests with Speculos</button>
<button class="quiz-choice" data-value="B">B) Integration tests with Jest + MSW</button>
<button class="quiz-choice" data-value="C">C) Unit tests for pure functions and utilities</button>
<button class="quiz-choice" data-value="D">D) Manual QA tests</button>
</div>
<p class="quiz-explanation">Unit tests are fast, cheap, and isolated. They form the wide base of the pyramid.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q3.</strong> What is the correct debugging order when an E2E test fails on CI?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Re-run → if it passes, ignore → if it fails, escalate</button>
<button class="quiz-choice" data-value="B">B) Read the error → check Allure screenshots/video → reproduce locally → fix</button>
<button class="quiz-choice" data-value="C">C) Delete the test → write a new one → push</button>
<button class="quiz-choice" data-value="D">D) Add <code>waitForTimeout(10000)</code> → push → check if it passes</button>
</div>
<p class="quiz-explanation">Systematic debugging: understand the failure through artifacts, reproduce locally, then fix the root cause.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q4.</strong> A CI workflow uses <code>matrix.shardIndex: [1,2,3,4]</code>. How many parallel runners?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) 1</button>
<button class="quiz-choice" data-value="B">B) 2</button>
<button class="quiz-choice" data-value="C">C) 4</button>
<button class="quiz-choice" data-value="D">D) 8</button>
</div>
<p class="quiz-explanation">One runner per shard index value: 4 parallel runners.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q5.</strong> A test fails with <code>ECONNREFUSED 127.0.0.1:5000</code>. What does port 5000 indicate?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Speculos REST API — the container is not running or not ready</button>
<button class="quiz-choice" data-value="B">B) The Electron dev server — the app failed to start</button>
<button class="quiz-choice" data-value="C">C) The WebSocket bridge — it crashed during setup</button>
<button class="quiz-choice" data-value="D">D) The Allure report server</button>
</div>
<p class="quiz-explanation">Port 5000 is the default Speculos REST API port. <code>ECONNREFUSED</code> means the Docker container didn't start, crashed, or isn't ready. Check Docker daemon, image pull, and container logs.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q6.</strong> A test fails with <code>APDU error 6985</code> from Speculos. What happened?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The Speculos Docker image is corrupted</button>
<button class="quiz-choice" data-value="B">B) The coin app .elf binary is missing</button>
<button class="quiz-choice" data-value="C">C) The network connection to the blockchain was lost</button>
<button class="quiz-choice" data-value="D">D) The user rejected/cancelled the action on the device — the test didn't press "Approve"</button>
</div>
<p class="quiz-explanation">APDU status <code>6985</code> means "conditions of use not satisfied" — typically the user rejected the transaction on the device. The test needs to simulate pressing "Approve" via button press or touch gesture.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q7.</strong> What is the <code>@Gate</code> job in CI?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A security scan that blocks malicious code</button>
<button class="quiz-choice" data-value="B">B) The required check that aggregates all required job statuses — if it passes, the PR can merge</button>
<button class="quiz-choice" data-value="C">C) A manual approval step by the QA team</button>
<button class="quiz-choice" data-value="D">D) A code coverage threshold check</button>
</div>
<p class="quiz-explanation"><code>@Gate</code> is the single aggregation point for branch protection. It checks that all required jobs passed. Non-required job failures don't block merging but should still be investigated.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q8.</strong> Which Cursor command provides a guided 6-phase onboarding to desktop E2E testing?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>feature-dev</code></button>
<button class="quiz-choice" data-value="B">B) <code>pre-review</code></button>
<button class="quiz-choice" data-value="C">C) <code>e2e-desktop-onboard</code></button>
<button class="quiz-choice" data-value="D">D) <code>test-coverage</code></button>
</div>
<p class="quiz-explanation"><code>e2e-desktop-onboard</code> is a 6-phase guided workflow: explore architecture → understand fixtures → read tests → write your first test → run with Speculos → debug and iterate.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q9.</strong> You are tasked with adding E2E tests for a new "Staking" feature. What is the correct first step?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Review the staking feature specs, identify critical user paths, and discuss test strategy with the team</button>
<button class="quiz-choice" data-value="B">B) Immediately create a test file and start writing test code</button>
<button class="quiz-choice" data-value="C">C) Create 20 test files covering every possible scenario</button>
<button class="quiz-choice" data-value="D">D) Ask the AI to generate all tests automatically</button>
</div>
<p class="quiz-explanation">Planning first: understand what needs testing, identify critical paths (stake, unstake, view rewards), and align with the team. Then create a branch and start implementing.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q10.</strong> Why should you always verify AI-generated test selectors against the actual DOM?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because AI tools cannot read code</button>
<button class="quiz-choice" data-value="B">B) Because test IDs change every deployment</button>
<button class="quiz-choice" data-value="C">C) Because the AI might use deprecated selectors</button>
<button class="quiz-choice" data-value="D">D) Because the AI generates selectors based on patterns and training data, which may not match the actual current state of the DOM elements and their test IDs</button>
</div>
<p class="quiz-explanation">AI tools generate code based on patterns. Even with project rules, the AI might guess a test ID that doesn't exist or has been renamed. Always run the test locally and verify selectors match the actual DOM.</p>
</div>

<div class="quiz-score"></div>
</div>

---

