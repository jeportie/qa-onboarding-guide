

# PART 2 -- TOOLS IN CONTEXT

<div class="chapter-intro">
Part 1 gave you the mental model — what you are testing, how the pieces fit together, and where things live. Now it is time to learn the tools you will use every day. Each chapter in this part explains how a specific tool is <strong>used in the Ledger Live repository</strong>, not just its generic features. You will learn the commands, file paths, configuration, and patterns that matter for your daily E2E work. By the end of Part 2, you will be comfortable with the entire workflow: branching, building, running tests, emulating devices, managing feature flags, and reading reports.
</div>

---

## Git Workflow, Hooks & Changesets

<div class="chapter-intro">
Every contribution you make — whether it is a new E2E test, a bug fix, or a page object refactor — flows through Git. This chapter covers the branching model, naming conventions, commit format, pre-commit hooks, and the changeset system that Ledger Live uses for versioning its 200+ packages. Getting this right from day one prevents friction in code review and CI.
</div>

### 6.1 Branch Strategy

```
develop (main integration branch -- your base for all work)
   |
   +-- release        (created from develop when a release starts)
   +-- hotfix         (created from main for urgent production fixes)
   +-- nightly        (long-lived, standalone — never merged into develop/main)
   |
   |-- feat/add-send-test-cardano      # New feature / test branches
   |-- bugfix/fix-flaky-swap-test      # Bug fix branches
   |-- support/update-e2e-page-objects  # Refactor / maintenance branches
   +-- chore/update-playwright-version  # Tooling changes
```

**Rules**:
- All work branches from `develop`
- PRs target `develop`
- `main` is protected — only `release` merges into `main` for production releases
- **Never work directly on `main` or `develop`** — always use a feature branch
- One branch = one isolated concern

#### The Release Flow (from the wiki)

The full release lifecycle is automated with GitHub Actions:

1. **Create Release** — A `release` branch is created from `develop` head. Changesets enters pre-release mode (`pnpm changeset pre enter next`). Pushing to this branch triggers pre-release builds.
2. **Stabilisation** — Fixes are pushed directly to `release`. Each push triggers new pre-release builds. QA validates the pre-release builds.
3. **Prepare Release** — When QA approves, the `release-prepare` workflow exits pre-release mode, bumps versions, tags apps, and merges `release` into `main` and `develop`.
4. **Release** — The `release-final` workflow builds production binaries, publishes libraries to npm with `pnpm changeset publish`, and pushes tags.

**Hotfix flow**: A `hotfix` branch is created from `main` head. Uses pre-release channel `hotfix`. After QA validation, merged back into `main`, `develop`, and `release` (if present).

**Nightly flow**: The `nightly` branch is long-lived and standalone (never merged anywhere). Every night, `develop` is merged into it, and the standard pre-release cycle publishes `@nightly` tagged packages and builds.

**Pre-release channels**: Any branch can enter pre-release mode with any channel name:
```bash
pnpm changeset pre enter experimental-bitcoin
# Then trigger the Release Prerelease workflow manually
# Builds will be available at: https://download.live.ledger.com/experimental-bitcoin/{os}
```

> **Important**: Pre-release commits should be dropped before merging back to avoid polluting `develop`.

### 6.2 Branch Naming Convention

```bash
# Pattern: <prefix>/qaa-<jira-ticket-number>
git checkout -b feat/qaa-1139
git checkout -b bugfix/qaa-2045
git checkout -b support/qaa-987
git checkout -b chore/qaa-1500
```

| Prefix | Use For | Example |
|--------|---------|---------|
| `feat/` | New features or tests | `feat/qaa-1139` |
| `bugfix/` | Bug fixes | `bugfix/qaa-2045` |
| `support/` | Refactors, test improvements | `support/qaa-987` |
| `chore/` | Tooling, configs, dependencies | `chore/qaa-1500` |

### 6.3 Commit Message Format (Conventional Commits)

```bash
# Format: <type>(<scope>): <description>
git commit -m "test(e2e): add send flow test for Cardano"
git commit -m "fix(e2e): resolve flaky swap confirmation timeout"
git commit -m "refactor(e2e): extract common swap helpers"

# Interactive commit helper:
pnpm commit
```

| Type | Meaning | Example |
|------|---------|---------|
| `feat` | New feature | `feat(desktop): add dark mode toggle` |
| `fix` | Bug fix | `fix(e2e): increase swap timeout` |
| `test` | Adding/updating tests | `test(e2e): add send flow for Solana` |
| `refactor` | Restructure, no behavior change | `refactor(e2e): extract common helpers` |
| `chore` | Maintenance, tooling | `chore(e2e): update Playwright` |
| `docs` | Documentation only | `docs: update E2E README` |
| `ci` | CI/CD changes | `ci: add Stax device to E2E matrix` |
| `perf` | Performance improvement | `perf(e2e): parallelize account setup` |

**Scope** is optional but recommended: `desktop`, `mobile`, `e2e`, `coin`, `common`.

### 6.4 Pre-Commit Hooks

The repo uses **hk** (a modern Husky alternative) configured in `hk.pkl`. A single pre-commit hook runs:

**`gitleaks`** — scans staged files for accidental secrets (API keys, passwords, tokens, private keys).

```bash
# If your commit is blocked by gitleaks:
# 1. It means you're about to commit a secret!
# 2. Find the secret in your staged files
# 3. Remove it (use environment variables instead)
# 4. Try committing again
```

### 6.5 Changesets

Every PR that modifies a **published** package needs a changeset — a small markdown file describing the change and its semver bump type.

```bash
# Add a changeset (interactive prompt):
pnpm changeset
# The wiki also defines this alias:
pnpm changelog

# This creates a file like:
# .changeset/happy-dogs-smile.md
# ---
# "@ledgerhq/live-common": patch
# ---
# Fix Cardano delegation E2E test
```

#### How Changesets Work (from the wiki)

Changesets are designed to make your workflow easier by allowing contributors to make versioning decisions at contribution time. Each changeset holds two key pieces of information:
- A **version type** (following semver: `major`, `minor`, `patch`)
- **Change information** to be added to a changelog

When you run `pnpm changelog`, you will:
1. See a list of all known packages in the monorepo
2. Select all packages **affected** by the code change
3. Choose the type of **bump** for each selected package
4. Enter a **summary** with context (GitHub issue number, Jira ticket for Ledger employees)

Changesets accumulate in the `develop` branch in the `.changeset/` folder until release day, when they are consumed and `CHANGELOG.md` files get populated.

> **If a PR has no changeset**, a custom GitHub Action will comment on the PR to make sure this is not an oversight.

**For E2E-only changes** (files in `e2e/`), you usually **don't need** a changeset because E2E packages are private and not published to npm.

### 6.6 PR Workflow

1. Create a feature branch from `develop`
2. Make your changes, commit with Conventional Commits
3. Push and open a PR targeting `develop`
4. CI runs automatically (commitlint, lint, typecheck, unit tests, smoke E2E)
5. The `@Gate` job must pass — this is the required check that blocks merging
6. Request review from CODEOWNERS team (your team: `@ledgerhq/qaa` for E2E)
7. Merge when approved + CI green

### 6.7 Common Git Issues in the Monorepo

#### pnpm-lock.yaml Conflicts

This is one of the most common issues when rebasing `develop` into your working branch:

```bash
git checkout develop
git pull origin develop --rebase
git checkout my-current-branch
git checkout develop pnpm-lock.yaml
pnpm i
```

This gets the latest lockfile from `develop`, then reinstalls to update it for your branch's changes.

> **Why not just delete pnpm-lock.yaml?** Because it tracks specific transitive dependency versions. Deleting and regenerating can silently update dependencies, causing hard-to-debug breakages.

#### Resolving pnpm-lock.yaml Conflicts During Release

After resolving all other conflicts in a release merge:
```bash
pnpm clean
pnpm i
# If CocoaPods lockfile also conflicts:
pnpm mobile pod
```

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://www.conventionalcommits.org/en/v1.0.0/">Conventional Commits specification</a></li>
<li><a href="https://github.com/changesets/changesets">Changesets documentation</a> — the versioning tool used by the monorepo</li>
<li><a href="https://semver.org">Semantic Versioning (semver)</a> — the versioning convention followed by Ledger packages</li>
<li><a href="https://learngitbranching.js.org/">Learn Git Branching</a> — interactive visual game to master Git branching concepts</li>
<li><a href="https://github.com/gitleaks/gitleaks">Gitleaks</a> — the secret scanner used in pre-commit hooks</li>
<li><a href="https://github.com/nickel-lang/hk">hk</a> — the modern Git hooks manager (Husky alternative)</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> The Git workflow is strict for a reason — 200+ packages, multiple teams, and a release train that ships to millions of users. Conventional commits feed into changelogs, changesets feed into version bumps, and the <code>@Gate</code> job protects <code>develop</code>. Internalize these patterns now and you will never block a PR on process issues.
</div>

### 6.8 Quiz

<!-- ── Chapter 6 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> Which branch should you base your feature branch on?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>main</code></button>
<button class="quiz-choice" data-value="B">B) <code>develop</code></button>
<button class="quiz-choice" data-value="C">C) <code>release</code></button>
<button class="quiz-choice" data-value="D">D) <code>nightly</code></button>
</div>
<p class="quiz-explanation"><code>develop</code> is the integration branch. All feature branches are created from and merged back into <code>develop</code>. <code>main</code> is only touched by release merges. <code>nightly</code> is standalone and never merged.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> What does the <code>gitleaks</code> pre-commit hook scan for?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) TypeScript type errors</button>
<button class="quiz-choice" data-value="B">B) Linting violations and code style</button>
<button class="quiz-choice" data-value="C">C) Accidental secrets (API keys, passwords, tokens)</button>
<button class="quiz-choice" data-value="D">D) Merge conflicts in staged files</button>
</div>
<p class="quiz-explanation">The <code>gitleaks</code> pre-commit hook prevents accidentally committing secrets like API keys, passwords, or private keys into the repository.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q3.</strong> Which commit message follows Conventional Commits format correctly?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>Added new test for swap</code></button>
<button class="quiz-choice" data-value="B">B) <code>test(e2e): add swap flow test for Bitcoin</code></button>
<button class="quiz-choice" data-value="C">C) <code>TEST - swap bitcoin test added</code></button>
<button class="quiz-choice" data-value="D">D) <code>test: added swap test (Bitcoin).</code></button>
</div>
<p class="quiz-explanation">Correct format: <code>type(scope): imperative description</code>. Lowercase type, optional scope in parentheses, imperative mood ("add" not "added"), no trailing period.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q4.</strong> Do E2E-only changes in <code>e2e/</code> typically need a changeset?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Yes, always — every PR needs one</button>
<button class="quiz-choice" data-value="B">B) No, because E2E packages are private (not published to npm)</button>
<button class="quiz-choice" data-value="C">C) Only for mobile E2E tests</button>
<button class="quiz-choice" data-value="D">D) Only if the tests are failing in CI</button>
</div>
<p class="quiz-explanation">E2E packages are private and not published to npm, so no changeset is needed. Changesets are only required for published packages whose version must be bumped.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q5.</strong> During a release, what happens when the <code>release-prepare</code> workflow runs?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It creates the <code>release</code> branch from <code>develop</code></button>
<button class="quiz-choice" data-value="B">B) It publishes packages to npm with production tags</button>
<button class="quiz-choice" data-value="C">C) It only builds desktop binaries</button>
<button class="quiz-choice" data-value="D">D) It exits pre-release mode, bumps versions, tags apps, and merges <code>release</code> into <code>main</code> and <code>develop</code></button>
</div>
<p class="quiz-explanation">The prepare step runs <code>pnpm changeset pre exit</code> and <code>pnpm changeset version</code>, then merges the release branch into both <code>main</code> and <code>develop</code>. If there are conflicts on <code>develop</code>, a PR titled "Release merge conflicts" is created.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## pnpm, Turbo & Build Pipeline

<div class="chapter-intro">
A monorepo with 200+ packages needs a fast package manager and an intelligent build orchestrator. This chapter covers <strong>pnpm</strong> (the package manager), <strong>Turborepo</strong> (the build system), and the practical commands you will use daily to install, build, and run packages. You will also learn how to deal with dependency duplicates — a recurring challenge in large monorepos.
</div>

### 7.1 pnpm Basics

pnpm is the package manager. Unlike npm, it uses a **content-addressable store** and **symlinks**, making it fast and disk-efficient for monorepos. A single copy of each package version exists on disk, and all projects link to it.

```bash
# Install all dependencies
pnpm i

# Install for a specific package (and its dependencies with ...)
pnpm i --filter="ledger-live-desktop..."

# Run a script in a specific package
pnpm --filter ledger-live-desktop test:jest

# Clean everything (node_modules, build artifacts)
pnpm clean
```

### 7.2 Package Aliases

The root `package.json` defines shortcuts so you don't need to type long filter expressions:

```bash
pnpm desktop ...       # = pnpm --filter ledger-live-desktop ...
pnpm mobile ...        # = pnpm --filter live-mobile ...
pnpm e2e:desktop ...   # = pnpm --filter ledger-live-desktop-e2e-tests ...
pnpm e2e:mobile ...    # = pnpm --filter ledger-live-mobile-e2e-tests ...
```

### 7.3 The Filter System

The `--filter` flag selects which packages to operate on:

```bash
--filter="ledger-live-desktop"        # Exact package name
--filter="ledger-live-desktop..."     # Package + all its dependencies (note: ...)
--filter="@ledgerhq/live-common"      # Scoped package
--filter="[origin/develop]"           # Packages changed since develop
```

### 7.4 Turbo (Build Orchestration)

Turbo manages the **build dependency graph**. When you run `pnpm build:lld`, Turbo knows to build all dependent libraries first, in the correct order, with maximum parallelism.

```bash
# Build a specific library
pnpm turbo build --filter=@ledgerhq/live-common

# Build all libraries
pnpm build:libs

# Build desktop app + all deps
pnpm build:lld

# Build mobile deps
pnpm build:llm:deps
```

### 7.5 When TypeCheck Fails on an Import

If you see:
```
error TS2305: Module '"@ledgerhq/live-countervalues-react"' has no exported member 'useMarketcapIds'.
```

The library needs to be **rebuilt** — its build output is stale:
```bash
pnpm turbo build --filter=@ledgerhq/live-countervalues-react
```

### 7.6 Live Reload During Development

If you are modifying a library package while working on LLD or LLM, you can get on-the-fly updates:

```bash
# Watch a package for changes and rebuild automatically
pnpm --filter="@ledgerhq/hw-transport" run watch
```

> **Tip from the wiki**: For `live-common`, run the watch command **before** starting the application, or the hot reload can destabilize the running app.

### 7.7 Dependency Duplicates Management

The monorepo relies heavily on many npm libraries. Over time, the `pnpm-lock.yaml` can accumulate duplicate dependency versions — different packages locking slightly different versions of the same dependency.

**Automated detection**: CI runs a non-regression check on every PR, comparing against `develop` for possible introduction of duplicates.

#### Recovering a Broken pnpm-lock.yaml

```bash
git checkout origin/develop pnpm-lock.yaml
pnpm i
```

#### Manually Deduplicating a Library

When a dependency is used by multiple packages at slightly different locked versions (e.g., 1.2.3 and 1.2.4):

```bash
pnpm -r up <dep>    # Unify across all packages
```

#### Platform-Specific Duplicate Solutions

| Platform | Problem | Solution |
|----------|---------|----------|
| **LLM (Mobile)** | Metro can't resolve hoisted duplicates | Add to `FORCED_DEPENDENCIES` array in `metro.config.js` |
| **LLD (Desktop)** | Webpack bundles wrong version | Add alias in webpack config |
| **Global** | Peer dependency causes duplication | Add to `readPackage` function in `.pnpmfile.cjs` to force-remove peer deps |

### 7.8 Windows Considerations

On Windows, running binaries by specifying their path in package.json scripts will fail:

```
'XXX' is not recognized as an internal or external command
```

**Solution**: Use the `x-ref` homemade package to run binaries in a cross-platform compatible way.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://pnpm.io/">pnpm documentation</a> — created by Zoltan Kochan</li>
<li><a href="https://turbo.build/repo/docs">Turborepo documentation</a> — by Vercel (Jared Palmer)</li>
<li><a href="https://pnpm.io/filtering">pnpm filtering</a> — complete reference for the <code>--filter</code> flag</li>
<li><a href="https://turbo.build/repo/docs/crafting-your-repository/caching">Turbo caching</a> — how Turbo avoids rebuilding unchanged packages</li>
<li><a href="https://semver.org/">Semantic Versioning</a> — the versioning convention for all packages</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> In a monorepo this size, you will rarely install or build everything. Learn <code>--filter</code>, learn the aliases (<code>pnpm desktop</code>, <code>pnpm e2e:desktop</code>), and know when to rebuild a stale library. These commands will be your most-typed shell inputs.
</div>

### 7.9 Quiz

<!-- ── Chapter 7 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> What does <code>pnpm e2e:desktop test:playwright</code> actually run?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>pnpm --filter ledger-live-desktop test:playwright</code></button>
<button class="quiz-choice" data-value="B">B) <code>pnpm --filter ledger-live-desktop-e2e-tests test:playwright</code></button>
<button class="quiz-choice" data-value="C">C) <code>npx playwright test</code></button>
<button class="quiz-choice" data-value="D">D) <code>pnpm --filter e2e test:playwright</code></button>
</div>
<p class="quiz-explanation">The <code>e2e:desktop</code> alias maps to <code>--filter ledger-live-desktop-e2e-tests</code>, which is the E2E test package — not the desktop app itself.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> A typecheck error says <code>Module '@ledgerhq/coin-evm' has no exported member 'getTransactionStatus'</code>. What should you do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Delete <code>node_modules</code> and reinstall everything</button>
<button class="quiz-choice" data-value="B">B) Run <code>pnpm turbo build --filter=@ledgerhq/coin-evm</code></button>
<button class="quiz-choice" data-value="C">C) Ignore the error — it's a false positive</button>
<button class="quiz-choice" data-value="D">D) Upgrade the package version in package.json</button>
</div>
<p class="quiz-explanation">The library's build output is stale. Rebuilding it with Turbo regenerates the TypeScript declaration files and exports.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q3.</strong> How does pnpm achieve disk efficiency compared to npm?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It compresses node_modules into a ZIP archive</button>
<button class="quiz-choice" data-value="B">B) It only installs production dependencies by default</button>
<button class="quiz-choice" data-value="C">C) It uses a content-addressable store with symlinks — one copy per version on disk</button>
<button class="quiz-choice" data-value="D">D) It removes unused dependencies automatically</button>
</div>
<p class="quiz-explanation">pnpm stores each package version once in a global content-addressable store and creates symlinks from <code>node_modules</code> to the store. This avoids the massive duplication that npm's flat <code>node_modules</code> creates in monorepos.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> Your <code>pnpm-lock.yaml</code> has conflicts after rebasing <code>develop</code>. What is the recommended approach?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>git checkout develop pnpm-lock.yaml</code> then <code>pnpm i</code></button>
<button class="quiz-choice" data-value="B">B) Delete <code>pnpm-lock.yaml</code> and run <code>pnpm i</code> to regenerate</button>
<button class="quiz-choice" data-value="C">C) Manually resolve the conflicts in the lockfile</button>
<button class="quiz-choice" data-value="D">D) Run <code>git merge --abort</code> and start over</button>
</div>
<p class="quiz-explanation">Checkout the lockfile from <code>develop</code>, then reinstall. This preserves locked transitive dependency versions. Never delete the lockfile — regenerating it can silently update transitive dependencies and introduce hard-to-debug breakages.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q5.</strong> What does the <code>...</code> suffix do in <code>--filter="ledger-live-desktop..."</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It's a glob that matches any package starting with "ledger-live-desktop"</button>
<button class="quiz-choice" data-value="B">B) It includes only direct dependencies</button>
<button class="quiz-choice" data-value="C">C) It excludes the package itself and only processes dependencies</button>
<button class="quiz-choice" data-value="D">D) It includes the package AND all its transitive dependencies</button>
</div>
<p class="quiz-explanation">The <code>...</code> suffix means "this package and everything it depends on (transitively)". This is essential for installing or building the complete dependency tree of a specific app.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Playwright in Ledger Live

<div class="chapter-intro">
Playwright is the E2E testing framework for <strong>Ledger Live Desktop</strong>. This chapter covers the Playwright configuration, the custom fixture system (the most important architectural pattern in the desktop E2E suite), locator strategies, the <code>@step</code> decorator for Allure reporting, test tags, userdata profiles, and the complete test development workflow as documented in the team wiki.
</div>

### 8.1 Configuration

**File**: `e2e/desktop/playwright.config.ts`

Key settings:
- `testDir`: Points to `tests/specs/`
- `timeout`: Test timeout (e.g., 5 minutes per test)
- `retries`: Number of retries on failure (usually 1-2)
- `workers`: Number of parallel workers
- `reporter`: Allure reporter + custom Xray JSON reporter
- `use.screenshot`: Capture on failure
- `use.video`: Record video (deleted on success to save space)
- `use.trace`: Capture Playwright trace on failure

### 8.2 The Custom Fixture System

**File**: `e2e/desktop/tests/fixtures/common.ts` — THE most important file in the E2E suite.

The fixture defines everything a test needs:

```typescript
type TestFixtures = {
  // UI Configuration
  lang: string;                          // Default: "en-US"
  theme: "light" | "dark";              // Default: "dark"

  // Test Data
  userdata?: string;                    // Pre-configured JSON profile name
  cliCommands?: CliCommand[];           // Commands to populate test data via CLI

  // Device Simulator
  speculosApp: AppInfos;               // Which coin app to load on the emulated device

  // Feature Flags
  featureFlags: OptionalFeatureMap;     // Override Firebase feature flags

  // Provided Objects (injected into tests)
  electronApp: ElectronApplication;     // The running Electron app instance
  page: Page;                           // The Playwright page
  app: Application;                     // All page objects (YOUR MAIN TOOL)
  userdataFile: string;                 // Path to the test's app.json

  // Reporting
  teamOwner?: Team;                     // Which team owns this test
};
```

Tests configure fixtures with `test.use()`:

```typescript
test.describe("Settings", () => {
  test.use({
    teamOwner: Team.WALLET_XP,
    userdata: "erc20-0-balance",
    // speculosApp: account.currency.speculosApp,  // Only if device needed
    // cliCommands: [liveDataCommand(account)],      // Only if test data needed
  });

  test("my test", async ({ app }) => {
    // app has ALL page objects ready to use
  });
});
```

### 8.3 Locator Strategy

Locator priority (most stable to least stable):

```typescript
// 1. BEST: by test ID (won't break on text/style changes)
page.getByTestId("send-button");

// 2. GOOD: by role (accessible, semantic)
page.getByRole("button", { name: "Send" });

// 3. OK: by text (breaks if translations change)
page.getByText("Send transaction");

// 4. LAST RESORT: CSS selector
page.locator(".send-button");
```

### 8.4 The @step Decorator

**File**: `e2e/desktop/tests/misc/reporters/step.ts`

Every page object method is decorated with `@step` for Allure reporting:

```typescript
class AccountPage extends AppPage {
  @step("Click Send button")
  async clickSend(): Promise<void> {
    await this.sendButton.click();
  }

  @step("Navigate to token $0")  // $0 replaced with first argument
  async navigateToToken(name: string): Promise<void> {
    await this.page.getByText(name).click();
  }
}
```

In Allure reports, this creates a collapsible step tree:
```
Test: ERC20 token with 0 balance is hidden
  + Step: Open target from main navigation "accounts"    [0.5s] PASSED
  + Step: Show parent account tokens "Ethereum 1"        [0.3s] PASSED
  + Step: Verify token visibility "USDT"                 [0.2s] PASSED
  + Step: Click hide empty token accounts toggle          [0.1s] PASSED
  + Step: Verify children tokens are not visible "USDT"  [0.2s] PASSED
```

### 8.5 Test Tags

Tests are tagged for filtering in CI:

```typescript
test("My test", {
  tag: [
    "@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",  // Device compatibility
    "@ethereum", "@family-evm",                                     // Currency / family
    "@smoke",                                                       // Quick validation
  ],
  annotation: [{ type: "TMS", description: "B2CQA-817" }],         // Jira test case ID
}, async ({ app }) => { ... });
```

| Tag Type | Examples | Used For |
|----------|----------|----------|
| Device | `@NanoSP`, `@Stax`, `@LNS` | Filter by device model in CI |
| Currency | `@ethereum`, `@bitcoin` | Filter by cryptocurrency |
| Family | `@family-evm`, `@family-bitcoin` | Filter by coin family |
| Scope | `@smoke` | Quick PR validation |

### 8.6 Userdata Profiles

Pre-configured JSON files in `e2e/desktop/tests/userdata/`:

| Profile | Purpose |
|---------|---------|
| `skip-onboarding-with-last-seen-device` | Skips onboarding, has a remembered device |
| `erc20-0-balance` | ETH account with zero-balance ERC-20 tokens |
| `1AccountBTC1AccountETH` | One Bitcoin + one Ethereum account |
| (and more...) | Various pre-configured states |

These avoid running through onboarding and account setup in every test, making tests faster and more reliable.

### 8.7 Test Development Workflow (from the wiki)

The wiki documents a 4-step workflow for developing desktop E2E tests:

#### Step 1: Set Up Environment

```bash
export MOCK=0
export SPECULOS_DEVICE=nanoSP
export COINAPPS=~/coin-apps
export SPECULOS_IMAGE_TAG=ghcr.io/ledgerhq/speculos:latest
# Optional: export SEED="your test seed"
```

#### Step 2: Build the Application

```bash
pnpm build:lld:deps
pnpm desktop build:testing
```

#### Step 3: Run Your Test

```bash
pnpm e2e:desktop test:playwright send.spec.ts
# Or filter by test name pattern (alternative):
pnpm e2e:desktop test:playwright --grep "my test name"
```

#### Step 4: Debug Failures

```bash
# Open Playwright UI mode for interactive debugging
pnpm e2e:desktop test:playwright --ui

# Generate and view Allure report
pnpm e2e:desktop allure
```

### 8.8 Common Issues (from the wiki)

| Issue | Cause | Fix |
|-------|-------|-----|
| "Element not found: send-button" + loading spinner in screenshot | Test doesn't wait for loading state | Add `waitFor({ state: "visible" })` before interaction |
| Speculos container timeout | Docker not running or coin-apps outdated | Check Docker, run `cd ~/coin-apps && git pull` |
| Test passes locally but fails in CI | Environment differences (timeouts, race conditions) | Increase timeouts, add explicit waits |
| "No app found for Solana on nanoSP" | Missing .elf binary | Update coin-apps: `cd ~/coin-apps && git pull` |

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://playwright.dev/docs/intro">Playwright documentation</a> — by Microsoft (Andrey Lushnikov, Dmitry Gozman)</li>
<li><a href="https://playwright.dev/docs/test-fixtures">Playwright Fixtures</a> — deep dive into the fixture system</li>
<li><a href="https://playwright.dev/docs/locators">Playwright Locators</a> — the recommended locator strategies</li>
<li><a href="https://playwright.dev/docs/test-annotations">Playwright Annotations & Tags</a></li>
<li><a href="https://playwright.dev/docs/trace-viewer">Playwright Trace Viewer</a> — for post-mortem debugging</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> The fixture system in <code>common.ts</code> is the heart of desktop E2E testing. Every test declares what it needs (userdata, device, feature flags) and the fixture orchestrates the setup and teardown. Master fixtures and you master the test suite.
</div>

### 8.9 Quiz

<!-- ── Chapter 8 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> What does <code>test.use({ userdata: "erc20-0-balance" })</code> do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Creates a new ERC-20 account from scratch</button>
<button class="quiz-choice" data-value="B">B) Loads a pre-configured app state that skips onboarding and has an ETH account with zero-balance ERC-20 tokens</button>
<button class="quiz-choice" data-value="C">C) Deletes all ERC-20 tokens from the test state</button>
<button class="quiz-choice" data-value="D">D) Sets the balance of all tokens to zero via the CLI</button>
</div>
<p class="quiz-explanation">It loads a pre-configured JSON profile that provides a known state, avoiding the need to run through onboarding and account setup. This makes tests faster and more deterministic.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> Which locator strategy should you use as your first choice?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) CSS selector: <code>page.locator(".send-btn")</code></button>
<button class="quiz-choice" data-value="B">B) Text: <code>page.getByText("Send")</code></button>
<button class="quiz-choice" data-value="C">C) Test ID: <code>page.getByTestId("send-button")</code></button>
<button class="quiz-choice" data-value="D">D) XPath: <code>page.locator("//button[@class='send']")</code></button>
</div>
<p class="quiz-explanation"><code>getByTestId</code> is the most stable locator — it won't break when text changes (translations) or when CSS classes are refactored. Test IDs are added specifically for test automation.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q3.</strong> In <code>@step("Navigate to token $0")</code>, what does <code>$0</code> represent?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The step sequence number in the test</button>
<button class="quiz-choice" data-value="B">B) The first argument passed to the method (e.g., "USDT")</button>
<button class="quiz-choice" data-value="C">C) The test case ID</button>
<button class="quiz-choice" data-value="D">D) The return value of the method</button>
</div>
<p class="quiz-explanation"><code>$0</code> is replaced with the first argument passed to the decorated method. If you call <code>navigateToToken("USDT")</code>, the Allure step shows "Navigate to token USDT".</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> A test fails in CI with "Element not found: send-button" and the Allure screenshot shows a loading spinner. What is the most likely fix?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Change the test ID to a different string</button>
<button class="quiz-choice" data-value="B">B) Remove the test and rewrite it from scratch</button>
<button class="quiz-choice" data-value="C">C) Increase the global timeout in <code>playwright.config.ts</code></button>
<button class="quiz-choice" data-value="D">D) Add an explicit wait for the element to be visible before interacting with it</button>
</div>
<p class="quiz-explanation">The test tries to interact with the element before the loading state finishes. Adding <code>await page.getByTestId("send-button").waitFor({ state: "visible", timeout: 30000 })</code> ensures the page has loaded.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q5.</strong> What is the purpose of the <code>annotation: { type: "TMS", description: "B2CQA-817" }</code> in a test?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It links the automated test to a Jira/Xray test case, creating a clickable link in Allure</button>
<button class="quiz-choice" data-value="B">B) It sets the test timeout to 817 milliseconds</button>
<button class="quiz-choice" data-value="C">C) It tags the test for CI sharding</button>
<button class="quiz-choice" data-value="D">D) It marks the test as belonging to a specific team</button>
</div>
<p class="quiz-explanation">TMS = Test Management System. <code>B2CQA-XXX</code> is a Jira/Xray test case ID. The Allure report creates a clickable link to the Jira issue, connecting automated test results to manual test case definitions.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Detox in Ledger Live

<div class="chapter-intro">
Detox is the E2E testing framework for <strong>Ledger Live Mobile</strong>. While the high-level testing philosophy (Page Object Model, Speculos emulation, feature flag overrides) mirrors the desktop suite, the implementation is fundamentally different. This chapter covers Detox configuration, the global app singleton, the WebSocket bridge, helper functions, the multi-phase initialization, and the complete mobile E2E project structure and debugging techniques as documented in the team wiki.
</div>

### 9.1 Configuration

**File**: `e2e/mobile/detox.config.js`

Defines device and app configurations:
- `ios.sim.debug`: iOS Simulator, debug build (needs Metro)
- `android.emu.release`: Android Emulator, release build (recommended for CI)
- App binary paths, launch arguments, test runner config (Jest)

### 9.2 The Global App Singleton

**File**: `e2e/mobile/page/index.ts`

Unlike Desktop's fixture injection, Mobile uses a global `app` object:

```typescript
// Accessible globally in all tests:
await app.portfolio.waitForPortfolioPageToLoad();
await app.portfolio.navigateToSettings();
await app.settings.openGeneralSettings();
```

### 9.3 The WebSocket Bridge

**File**: `e2e/mobile/bridge/server.ts`

Enables communication between Jest/Detox and the React Native app:

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

This bridge is necessary because Detox cannot directly manipulate React Native app state (unlike Playwright which has access to the Electron process). The WebSocket bridge is the channel through which the test harness configures the running app.

### 9.4 Helper Functions

**File**: `e2e/mobile/helpers/elementHelpers.ts`

```typescript
// Wrappers around verbose Detox API:
await tapById("send-button");           // element(by.id("send-button")).tap()
await tapByText("Confirm");             // element(by.text("Confirm")).tap()
await typeTextById("amount-input", "0.001");
await waitForElementById("success-message", 30000);
await scrollToId("account-row-bitcoin");
const balance = await getTextOfElement("balance-text");
```

### 9.5 Multi-Phase Initialization

The `app.init()` method orchestrates everything:

```typescript
await app.init({
  speculosApp: account.currency.speculosApp,  // Start Speculos with coin app
  cliCommands: [liveDataCommand(account)],    // Populate accounts via CLI
  featureFlags: { myFlag: { enabled: true } }, // Override feature flags
});
```

Internally this: launches Speculos Docker containers in parallel, executes CLI commands with retry logic, registers Speculos ports via the bridge, loads feature flags, and imports accounts.

### 9.6 Mobile E2E Project Structure (from the wiki)

```
e2e/mobile/
├── bridge/              # WebSocket bridge (server + client)
│   ├── server.ts        # Bridge server (Jest side)
│   └── client.ts        # Bridge client (React Native side)
├── helpers/             # Helper functions (element helpers, CLI helpers)
├── models/              # Data models (Account, Currency, etc.)
├── page/                # Page Object Model classes
│   ├── index.ts         # Global app singleton exporting all pages
│   ├── portfolio.page.ts
│   ├── account.page.ts
│   ├── send.page.ts
│   └── ...
├── specs/               # Test specifications
│   ├── send/
│   ├── receive/
│   ├── swap/
│   └── ...
├── detox.config.js      # Detox configuration
└── jest.config.ts       # Jest runner configuration
```

### 9.7 Building and Running Mobile E2E Tests (from the wiki)

#### Prerequisites

- Node.js, pnpm, Docker (for Speculos)
- **iOS**: Xcode, CocoaPods, iOS Simulator
- **Android**: Android Studio, Android SDK, an AVD (Android Virtual Device)

#### Building

```bash
# Install all dependencies
pnpm i

# Build mobile dependencies
pnpm build:llm:deps

# iOS: Install CocoaPods
pnpm mobile pod

# Build the app for testing
# iOS:
pnpm e2e:mobile build:ios.sim.debug
# Android:
pnpm e2e:mobile build:android.emu.release
```

#### Running

```bash
# Set environment variables
export MOCK=0
export SPECULOS_DEVICE=nanoSP
export COINAPPS=~/coin-apps

# Run all tests (iOS):
pnpm e2e:mobile test:ios.sim.debug

# Run specific test:
pnpm e2e:mobile test:ios.sim.debug -- --testNamePattern "send BTC"

# Run with specific device:
pnpm e2e:mobile test:android.emu.release
```

### 9.8 Debugging Mobile E2E Tests (from the wiki)

| Technique | How | When |
|-----------|-----|------|
| **Detox logs** | `pnpm e2e:mobile test:ios.sim.debug -- --loglevel trace` | Detox framework issues |
| **App logs via bridge** | `const logs = await app.bridge.getLogs()` | React Native app issues |
| **Metro console** | Watch Metro bundler terminal output | JavaScript errors, red screen |
| **Xcode console** | Open Xcode → Window → Devices and Simulators | iOS native crashes |
| **Android logcat** | `adb logcat *:E` | Android native crashes |
| **Allure report** | `pnpm e2e:mobile allure` | Test step trace, screenshots |

### 9.9 Key Differences from Desktop

| Aspect | Desktop (Playwright) | Mobile (Detox + Jest) |
|--------|---------------------|----------------------|
| Framework | Playwright | Detox + Jest |
| App launch | `electron.launch()` | `device.launchApp()` |
| Fixture system | Playwright fixtures (`test.use()`) | Global `app.init()` |
| Page access | `({ app }) =>` destructured | `app.` global variable |
| Element find | `page.getByTestId()` | `element(by.id())` |
| Wait strategy | Auto-wait built-in | Manual `waitFor().withTimeout()` |
| Text input | `.fill("text")` | `.typeText("text")` |
| Bridge | Not needed | WebSocket bridge (test <-> RN app) |
| Parallelism | Full parallel | 1-3 workers (device limitation) |
| Port forwarding | Not needed | `device.reverseTcpPort()` (Android) |
| Build | Rspack bundle | Xcode / Gradle native build |

### 9.10 CI Workflow Inputs (from the wiki)

The mobile E2E CI workflow accepts several inputs that control test execution:

| Input | Description | Default |
|-------|-------------|---------|
| `platform` | `ios` or `android` | `ios` |
| `configuration` | Detox configuration name | `ios.sim.release` |
| `test_filter` | Jest `--testNamePattern` filter | (none) |
| `shard_index` / `shard_total` | Sharding for parallel CI | `1/1` |
| `speculos_device` | Device model for Speculos | `nanoSP` |
| `retry_count` | Number of retries on failure | `1` |

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://wix.github.io/Detox/">Detox documentation</a> — by Wix Engineering</li>
<li><a href="https://wix.github.io/Detox/docs/api/matchers">Detox Matchers</a> — element matching API</li>
<li><a href="https://wix.github.io/Detox/docs/api/actions-on-element">Detox Actions</a> — tap, type, scroll, swipe</li>
<li><a href="https://wix.github.io/Detox/docs/troubleshooting/running-tests">Detox Troubleshooting</a></li>
<li><a href="https://reactnative.dev/docs/debugging">React Native Debugging</a></li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Mobile E2E has more moving parts than desktop — native builds, the WebSocket bridge, device simulators, port forwarding. The <code>app.init()</code> method abstracts most of this complexity. When debugging failures, know which layer to look at: Detox framework, bridge communication, React Native app, or native platform.
</div>

### 9.11 Quiz

<!-- ── Chapter 9 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> How do you access page objects in mobile E2E tests?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Through Playwright fixtures: <code>async ({ app }) =></code></button>
<button class="quiz-choice" data-value="B">B) Through a global <code>app</code> singleton variable</button>
<button class="quiz-choice" data-value="C">C) By importing each page object individually</button>
<button class="quiz-choice" data-value="D">D) Through dependency injection with InversifyJS</button>
</div>
<p class="quiz-explanation">Mobile tests use a global <code>app</code> singleton (defined in <code>e2e/mobile/page/index.ts</code>), unlike Desktop which uses Playwright's fixture injection pattern.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q2.</strong> Why does Detox need <code>device.reverseTcpPort()</code> for Android but not iOS?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Android emulators run in a sandboxed network and need port forwarding to reach the host</button>
<button class="quiz-choice" data-value="B">B) iOS is faster and doesn't need port configuration</button>
<button class="quiz-choice" data-value="C">C) Android doesn't support WebSocket natively</button>
<button class="quiz-choice" data-value="D">D) Port reverse is only needed for Bluetooth connections</button>
</div>
<p class="quiz-explanation">Android emulators run with their own network stack where <code>localhost</code> refers to the emulator, not the host machine. Port reverse mapping connects the emulator's localhost to the host's localhost. iOS simulators share the host's network directly.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q3.</strong> What is the purpose of the WebSocket bridge in mobile E2E tests?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It connects the mobile app to Firebase for feature flags</button>
<button class="quiz-choice" data-value="B">B) It streams video recordings from the device to the test runner</button>
<button class="quiz-choice" data-value="C">C) It enables the test harness (Jest/Detox) to send commands to the React Native app (import accounts, override flags, navigate)</button>
<button class="quiz-choice" data-value="D">D) It provides a REST API for Speculos device emulation</button>
</div>
<p class="quiz-explanation">Detox can interact with native UI elements but cannot directly manipulate React Native app state. The WebSocket bridge is the channel through which the test harness sends commands like <code>importAccounts</code>, <code>overrideFeatureFlag</code>, and <code>navigate</code> to the running app.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> Translate this Playwright code to Detox: <code>await page.getByTestId("amount").fill("0.5");</code></p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>await element(by.text("amount")).fill("0.5");</code></button>
<button class="quiz-choice" data-value="B">B) <code>await device.typeText("amount", "0.5");</code></button>
<button class="quiz-choice" data-value="C">C) <code>await element(by.id("amount")).setText("0.5");</code></button>
<button class="quiz-choice" data-value="D">D) <code>await element(by.id("amount")).typeText("0.5");</code></button>
</div>
<p class="quiz-explanation">In Detox: <code>getByTestId</code> → <code>by.id()</code>, <code>.fill()</code> → <code>.typeText()</code>. Detox uses <code>typeText</code> to simulate keyboard input, while Playwright's <code>fill</code> directly sets the input value.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> Your mobile E2E test fails with a red screen in the React Native app. Which debugging technique should you try first?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Run <code>adb logcat</code> for Android native crash logs</button>
<button class="quiz-choice" data-value="B">B) Check the Metro bundler terminal output for JavaScript errors</button>
<button class="quiz-choice" data-value="C">C) Increase the Detox timeout to 5 minutes</button>
<button class="quiz-choice" data-value="D">D) Rebuild the app from scratch</button>
</div>
<p class="quiz-explanation">A red screen in React Native indicates a JavaScript error. The Metro bundler terminal will show the exact error and stack trace. Only check native logs (<code>adb logcat</code>, Xcode console) if the issue is at the native layer.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Speculos Deep Dive

<div class="chapter-intro">
Speculos is the Ledger device emulator — the component that makes E2E testing possible without physical hardware. It runs real firmware inside a Docker container, exposing a REST API for interaction. This chapter covers all six device models, the REST API, touch vs non-touch interaction, the coin-apps repository, Docker commands, and the Ledger Live Bot — an autonomous testing system built on top of Speculos.
</div>

### 10.1 What Is Speculos?

Speculos is a **Ledger device emulator** that runs real firmware inside a Docker container. It supports all device models: Nano S, Nano S Plus, Nano X, Stax, Flex, and Nano Gen 5.

- **Source**: `github.com/LedgerHQ/speculos`
- **Written in**: C (ARM emulation core) + Python (REST API, automation)
- **Image**: `ghcr.io/ledgerhq/speculos:latest`

### 10.2 Device Models in Detail

| Device | Env | Touch | Display | Screen | Notes |
|--------|-----|-------|---------|--------|-------|
| Nano S | `nanoS` | No | BAGL | 128x32 | Legacy. Limited memory. Requires specific Docker SHA. |
| Nano S Plus | `nanoSP` | No | BAGL | 128x64 | Most common test target. Good default. |
| Nano X | `nanoX` | No | BAGL | 128x64 | Has Bluetooth (not relevant in Speculos). |
| Stax | `stax` | **Yes** | NBGL | 400x672 | Touchscreen. Tap, swipe, long press interactions. |
| Flex | `flex` | **Yes** | NBGL | 480x600 | Touchscreen. Different form factor than Stax. |
| Nano Gen 5 | `nanoGen5` | **Yes** | NBGL | — | Newest. Internally codenamed "apex" in coin-apps. |

**BAGL vs NBGL**: These are the two display frameworks used by Ledger devices. BAGL (Bolos Application Graphics Library) is the older system for button-based Nano devices. NBGL (New Bolos GL) is the modern framework for touchscreen devices (Stax, Flex, Gen 5). The display framework affects how E2E tests interact with the emulated device — buttons vs touch gestures.

### 10.3 The REST API

Speculos exposes a comprehensive REST API:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | Health check (alive?) |
| `/events` | GET | Get display events (text on screen) |
| `/apdu` | POST | Send raw APDU command |
| `/button/left` | POST | Press left button (non-touch) |
| `/button/right` | POST | Press right button (non-touch) |
| `/button/both` | POST | Press both buttons = confirm (non-touch) |
| `/finger` | POST | Touch event with x,y coordinates (touch devices) |
| `/screenshot` | GET | Take PNG screenshot |
| `/` | DELETE | Kill the session |
| `/automation` | POST | Set automation rules (auto-approve) |

### 10.4 Non-Touch vs Touch Interaction

**Non-touch devices** (Nano S, S Plus, X):
```typescript
buttons.right();     // Next screen
buttons.right();     // Next screen
buttons.both();      // Confirm
```

**Touch devices** (Stax, Flex, Gen 5):
```typescript
pressAndRelease("Sign transaction", x, y);     // Tap on label
longPressAndRelease("Hold to sign", 3);         // Long press 3 seconds
swipeRight();                                    // Swipe gesture
```

### 10.5 The Coin-Apps Repository

`github.com/LedgerHQ/coin-apps` contains compiled `.elf` binaries:

```
coin-apps/
|-- nanoS/1.6.2/Bitcoin/app_2.5.0.elf
|-- nanoSP/1.8.1/Bitcoin/app_2.7.0.elf
|-- nanoSP/1.8.1/Ethereum/app_1.13.0.elf
|-- stax/.../
|-- flex/.../
+-- apex/.../                # Nano Gen 5 (codename)
```

```bash
# Local setup:
git clone https://github.com/LedgerHQ/coin-apps.git ~/coin-apps
export COINAPPS=~/coin-apps
# Keep it updated frequently:
cd ~/coin-apps && git pull
```

### 10.6 Environment Variables

```bash
# Required
export MOCK=0                                          # Real Speculos, not mocked
export SPECULOS_DEVICE=nanoSP                          # Device model
export SPECULOS_IMAGE_TAG=ghcr.io/ledgerhq/speculos:latest
export COINAPPS=~/coin-apps                            # Path to .elf binaries

# Auto-managed by the test framework (don't set manually):
# SPECULOS_API_PORT=5000        # REST API port
# SPECULOS_ADDRESS=http://127.0.0.1
# SEED=abandon abandon ... about   # BIP39 mnemonic (CI secret)
```

### 10.7 Docker Commands

```bash
docker pull ghcr.io/ledgerhq/speculos:latest          # Pull latest
docker images | grep speculos                          # Verify
docker ps --filter name=speculos                       # Running containers
docker rm -f $(docker ps -aq --filter name=speculos)   # Kill leftovers
docker logs <container-id>                             # View logs
```

### 10.8 The Ledger Live Bot (from the wiki)

The Ledger Live Bot is an autonomous testing system that performs real transactions using Speculos. It is fundamentally different from E2E tests:

#### Philosophy

- **Stateless**: Its state is on the blockchain
- **Configless**: Only needs a seed and a coin-apps folder
- **Autonomous**: Restores from existing seed accounts and continues from existing blockchain state
- **Data driven**: Actions are based on data specs (mutations) that drive capabilities
- **End to End**: Relies on the complete Ledger stack: live-common + Speculos
- **Realistic**: Very close to what Ledger Live users would do — it even presses device buttons

#### How It Works

The bot scans accounts for a given seed and, for each account, randomly selects a possible transaction ("mutation") to perform against one of its sibling accounts. This exercises the full transaction flow: scan → build transaction → sign on device → broadcast → verify.

#### Mutations

A **mutation** is a possible action to perform on an account. It includes:
- The **transaction expression** (what to do)
- The **device action** (how to interact with Speculos)
- An optional **assertion test** (expected outcome)

```javascript
const dogecoinSpec = {
  name: "DogeCoin",
  currency: getCryptoCurrencyById("dogecoin"),
  dependency: "Bitcoin",
  appQuery: { model: "nanoS", appName: "Dogecoin", firmware: "1.6.0" },
  mutations: [
    {
      name: "send max",
      transaction: ({ account, siblings, bridge }) => {
        invariant(account.balance.gt(100000), "balance is too low");
        let t = bridge.createTransaction(account);
        const sibling = pickSiblings(siblings);
        t = bridge.updateTransaction(t, { useAllAmount: true, recipient: sibling.freshAddress });
        return t;
      },
      deviceAction: deviceActionAcceptBitcoin,
      test: ({ account }) => {
        expect(account.balance.toString()).toBe("0");
      },
    },
  ],
};
```

#### The Seven Seeds

| Secret ID | Codename | Purpose |
|-----------|----------|---------|
| `SEED1` | **Mère Denis** | Stability tests (decommissioned, kept for large accounts) |
| `SEED2` | **Carbone** | Non-regression on `main`, staging explorers (daily on Bitcoin/Ethereum) |
| `SEED3` | **Silicium** | Available for manual runs |
| `SEED4` | **Mooncake** | Available for manual runs |
| `SEED5` | **Oxygen** | Costly non-regression on `develop` (weekly) |
| `SEED6` | **Phosphore** | Available for manual runs |
| `SEED7` | **Nitrogen** | Daily non-regression on `develop` |

All seeds are rotated for non-regression against `develop` every 8 hours.

#### Bot Limitations

- Only covers successful transactions (no error cases)
- Not all apps are supported in Speculos
- Cannot assert all possible transactions in one run
- Only works with one app at a time (no disconnection/dashboard flows)
- Does **not** run on LLD or LLM — cannot detect UI-specific issues
- Cannot perform swaps

> **Security warning**: The bot uses seeds in clear text. Never use an existing seed. Use a dedicated seed with minimal funds (~$10 per coin).

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://github.com/LedgerHQ/speculos">Speculos source code</a> — the device emulator</li>
<li><a href="https://github.com/LedgerHQ/coin-apps">Coin-apps repository</a> — compiled .elf binaries for all devices</li>
<li><a href="https://developers.ledger.com/docs/device-interaction/beginner/exchange_data">APDU — How to exchange data with a Ledger device</a> (Ledger developer docs)</li>
<li><a href="https://docs.zondax.ch/ledger-apps/starkware/APDU">APDU — Protocol specification reference</a> (Zondax docs)</li>
<li><a href="https://www.ledger.com/introducing-bolos-blockchain-open-ledger-operating-system">BOLOS / BAGL / NBGL — Introducing the Blockchain Open Ledger Operating System</a></li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Speculos is what makes your tests realistic — it runs real firmware, real coin apps, and responds to real APDU commands. The bot extends this further by performing autonomous transactions on the blockchain. When a test involves device interaction, understanding Speculos is non-negotiable.
</div>

### 10.9 Quiz

<!-- ── Chapter 10 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> How do you confirm a transaction on a non-touch device (Nano SP) via the Speculos API?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>POST /finger</code> with x,y coordinates on the "Approve" label</button>
<button class="quiz-choice" data-value="B">B) <code>POST /button/both</code></button>
<button class="quiz-choice" data-value="C">C) <code>POST /confirm</code></button>
<button class="quiz-choice" data-value="D">D) <code>POST /apdu</code> with a confirm APDU command</button>
</div>
<p class="quiz-explanation">On non-touch devices, pressing both buttons simultaneously means "confirm". Touch devices (Stax, Flex, Gen 5) use <code>POST /finger</code> with coordinates instead.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> Your tests fail with "Error: no app found for Solana on nanoSP". What is the most likely cause?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Speculos doesn't support Solana at all</button>
<button class="quiz-choice" data-value="B">B) The <code>COINAPPS</code> path is wrong or coin-apps needs <code>git pull</code></button>
<button class="quiz-choice" data-value="C">C) You must use <code>nanoX</code> for Solana</button>
<button class="quiz-choice" data-value="D">D) The Speculos Docker image is too new</button>
</div>
<p class="quiz-explanation">Either <code>COINAPPS</code> is not set, points to the wrong directory, or the coin-apps repo is outdated and doesn't have the Solana .elf binary for the requested device/firmware version. Run <code>cd ~/coin-apps && git pull</code>.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q3.</strong> What is the difference between BAGL and NBGL devices?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) BAGL is for production devices, NBGL is for development devices</button>
<button class="quiz-choice" data-value="B">B) BAGL supports color displays, NBGL only supports monochrome</button>
<button class="quiz-choice" data-value="C">C) BAGL is for button-based Nano devices, NBGL is for touchscreen devices (Stax, Flex, Gen 5)</button>
<button class="quiz-choice" data-value="D">D) They are the same framework with different version numbers</button>
</div>
<p class="quiz-explanation">BAGL (Bolos Application Graphics Library) is the display framework for button-based Nanos. NBGL (New Bolos GL) is the modern framework for touchscreen devices. This distinction affects how E2E tests interact with the emulated device.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> In the Ledger Live Bot, what is a "mutation"?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A change to the Speculos Docker image configuration</button>
<button class="quiz-choice" data-value="B">B) A modification to the source code of a coin integration</button>
<button class="quiz-choice" data-value="C">C) A React state update in the Ledger Live UI</button>
<button class="quiz-choice" data-value="D">D) A possible transaction action to perform on an account (e.g., "send max", "move 50%"), which mutates the blockchain state</button>
</div>
<p class="quiz-explanation">In the bot's vocabulary, a mutation is a transaction scenario that changes (mutates) account state on the blockchain. Each mutation includes the transaction definition, device action (how to interact with Speculos), and optional assertions.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q5.</strong> Which bot seed (Nitrogen) runs daily non-regression on <code>develop</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) SEED7 (Nitrogen)</button>
<button class="quiz-choice" data-value="B">B) SEED1 (Mère Denis)</button>
<button class="quiz-choice" data-value="C">C) SEED2 (Carbone)</button>
<button class="quiz-choice" data-value="D">D) SEED5 (Oxygen)</button>
</div>
<p class="quiz-explanation">Nitrogen (SEED7) runs daily non-regression on <code>develop</code>. Carbone (SEED2) runs on <code>main</code> with staging explorers. Oxygen (SEED5) runs weekly costly non-regression on <code>develop</code>.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Firebase & Feature Flags

<div class="chapter-intro">
Feature flags are a critical part of modern software delivery. They let teams decouple deployment from release, run experiments, and kill broken features instantly. This chapter covers how Ledger Live uses Firebase Remote Config, the four environments, how to override flags in E2E tests, and — crucially — the <strong>anti-patterns</strong> that have caused real problems in the codebase. Understanding what NOT to do with feature flags is as important as knowing how to use them.
</div>

### 11.1 What Are Feature Flags?

Feature flags let you enable/disable features remotely without deploying new code. They are used for:
- **Gradual rollouts** (enable for 10% of users, then 50%, then 100%)
- **A/B testing** (test variant A vs B)
- **Kill switches** (instantly disable a broken feature)
- **Development** (hide WIP features behind a flag)

### 11.2 Firebase Remote Config Environments

Ledger Live uses four Firebase environments:

| Environment | Purpose |
|---|---|
| **Ledger Wallet** | Production flags for main app features |
| **Swap** | Flags for swap/exchange functionality |
| **Earn** | Flags for staking/earning features |
| **Buy Sell** | Flags for fiat on/off ramp |

### 11.3 Feature Flags in E2E Tests

You can override flags in tests using three methods:

**Desktop** (via fixture):
```typescript
test.use({
  featureFlags: {
    llmNewTransferDrawer: { enabled: true },
    myExperimentalFeature: { enabled: true, params: { variant: "B" } },
  },
});
```

**Mobile** (via app.init):
```typescript
await app.init({
  featureFlags: {
    llmNewTransferDrawer: { enabled: true },
  },
});
```

**Via environment variable**:
```bash
export E2E_FEATURE_FLAGS_JSON='{"myFlag":{"enabled":true}}'
```

### 11.4 The Wallet 4.0 Toggle

```bash
export E2E_ENABLE_WALLET40=1
```

This enables the Wallet 4.0 UI, which significantly changes navigation and layout. Some tests need this flag, others expect the classic UI. Always be aware of which UI version your test targets.

### 11.5 Feature Flag Anti-Patterns

The following anti-patterns have been identified from real issues in the Ledger Live codebase. Understanding them will help you write more robust E2E tests and contribute to better feature flag hygiene.

#### Anti-Pattern 1: Inconsistent Feature Control Capabilities

**Problem**: Different feature flags have wildly different control capabilities. Some only offer an on/off toggle. Others support per-coin toggling, versioned parameters, or platform-specific overrides. There is no standard interface.

**Impact on QA**: You cannot assume a flag behaves the same as another. A flag might be a simple boolean in one context but carry complex parameters in another. Your tests must account for the specific flag structure.

**Example**:
```typescript
// Simple flag — just enabled/disabled
featureFlags: { llmNewTransferDrawer: { enabled: true } }

// Complex flag with params, per-coin control, and versioning
featureFlags: {
  stakingProviders: {
    enabled: true,
    params: {
      listProvider: [
        { id: "lido", coins: ["ethereum"], minVersion: "3.12.0" }
      ]
    }
  }
}
```

**Best practice**: Always read the flag's type definition before writing a test that overrides it. Check `shared/feature-flags/` for the schema.

#### Anti-Pattern 2: Cross-Layer or "God" Configuration

**Problem**: Some feature flags control behavior across multiple layers simultaneously — UI rendering, business logic, API endpoints, and coin support. A single flag change can have cascading effects that are hard to predict and test.

**Impact on QA**: When testing a flag, you may need to verify its effect at multiple layers. Toggling a "God flag" might break unrelated features.

**Best practice**: When writing E2E tests for a feature behind a flag, test the feature both with the flag enabled AND disabled to catch regressions in both states.

#### Anti-Pattern 3: Configuration Discoverability Issues

**Problem**: Feature flags are spread across four Firebase environments (Ledger Wallet, Swap, Earn, Buy Sell) with no central registry. Finding which flags exist, what they do, and who owns them requires tribal knowledge.

**Impact on QA**: You may not know which flags affect the flow you are testing. A test that worked yesterday might fail because someone changed a flag in a Firebase environment you didn't know about.

**Best practice**: Always explicitly override all flags that your test depends on via the fixture system. Never rely on Firebase defaults — they can change at any time.

#### Anti-Pattern 4: Unclear Ownership or Control Scope

**Problem**: Some flags lack clear ownership. When a flag causes an issue, it's not obvious who should fix it — the team that created it, the team whose feature it affects, or the platform team.

**Impact on QA**: When a test fails due to a flag change, escalation path is unclear.

**Best practice**: Use the `teamOwner` fixture property to tag your tests, and document which flags your test depends on in the test description or annotation.

#### Anti-Pattern 5: Implicit Dependencies Between Feature Flags

**Problem**: Some features only work when multiple flags are enabled together, but these dependencies are not declared anywhere. Enabling flag A without flag B might result in a broken state.

**Impact on QA**: Your test might fail not because of the feature under test, but because a dependency flag is not enabled. These implicit dependencies are the #1 source of flaky tests related to feature flags.

**Best practice**: When you discover a flag dependency, document it in the test file:
```typescript
test.describe("New staking flow", () => {
  test.use({
    featureFlags: {
      // NOTE: llmNewStakingFlow requires llmStakingProviders to also be enabled
      llmNewStakingFlow: { enabled: true },
      llmStakingProviders: { enabled: true },
    },
  });
});
```

### 11.6 Monitoring Feature Flag Impact

The QA team monitors feature flag impact through the Slack channel **#qa-b2c-releases-bugs-tracking**. When a bug is suspected to be flag-related:

1. Check the flag state in the relevant Firebase environment
2. Try to reproduce with the flag explicitly toggled in both states
3. Report whether the bug is flag-dependent in the Jira ticket

### 11.7 Key Takeaways for Feature Flag Hygiene

| Principle | Description |
|-----------|-------------|
| **Scoped** | A flag should control ONE feature at ONE layer |
| **Consistent** | All flags should have the same interface capabilities |
| **Discoverable** | Any engineer should be able to find what a flag does in under 2 minutes |
| **Safe to modify** | Changing a flag should not have unpredictable cascading effects |

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://firebase.google.com/docs/remote-config">Firebase Remote Config documentation</a></li>
<li><a href="https://martinfowler.com/articles/feature-toggles.html">Feature Toggles (Feature Flags)</a> — Martin Fowler's comprehensive article on patterns and practices</li>
<li><a href="https://launchdarkly.com/blog/feature-flag-best-practices/">Feature Flag Best Practices</a> — LaunchDarkly (industry leader in feature flagging)</li>
<li><a href="https://www.devcycle.com/blog/feature-flag-anti-patterns">Feature Flag Anti-Patterns</a> — common mistakes with feature flags</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> In your E2E tests, <strong>always explicitly override every feature flag your test depends on</strong>. Never rely on Firebase defaults — they can change without notice. Document flag dependencies in your test code. And when debugging a flaky test, check whether a flag change is the root cause before investigating other hypotheses.
</div>

### 11.8 Quiz

<!-- ── Chapter 11 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> How do you override a feature flag in a desktop E2E test?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Edit the Firebase console directly</button>
<button class="quiz-choice" data-value="B">B) Use <code>test.use({ featureFlags: { flagName: { enabled: true } } })</code></button>
<button class="quiz-choice" data-value="C">C) Set <code>FIREBASE_CONFIG</code> environment variable</button>
<button class="quiz-choice" data-value="D">D) Modify the <code>shared/feature-flags/</code> source code</button>
</div>
<p class="quiz-explanation">The Playwright fixture system accepts <code>featureFlags</code> which override Firebase defaults for all tests in that describe block. This is the standard way to control flags in desktop E2E tests.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q2.</strong> What is the "implicit dependencies" anti-pattern?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) When a feature flag depends on a specific version of Node.js</button>
<button class="quiz-choice" data-value="B">B) When a feature flag is only documented in Confluence</button>
<button class="quiz-choice" data-value="C">C) When a feature flag has too many parameters</button>
<button class="quiz-choice" data-value="D">D) When a feature only works when multiple flags are enabled together, but these dependencies are not declared anywhere</button>
</div>
<p class="quiz-explanation">Implicit flag dependencies are undeclared relationships between flags. Enabling flag A without flag B might result in a broken state. This is the #1 source of flaky tests related to feature flags.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q3.</strong> What are the four Firebase Remote Config environments used by Ledger Live?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Dev, Staging, Production, Canary</button>
<button class="quiz-choice" data-value="B">B) Ledger Wallet, Swap, Earn, Buy Sell</button>
<button class="quiz-choice" data-value="C">C) Desktop, Mobile, CLI, Web</button>
<button class="quiz-choice" data-value="D">D) Alpha, Beta, RC, Stable</button>
</div>
<p class="quiz-explanation">Each environment manages feature flags for a different product area: Ledger Wallet (main features), Swap (exchange), Earn (staking), and Buy Sell (fiat on/off ramp).</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q4.</strong> Why should you always explicitly override feature flags in E2E tests rather than relying on Firebase defaults?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because Firebase is too slow for E2E tests</button>
<button class="quiz-choice" data-value="B">B) Because Firebase requires network access which CI doesn't have</button>
<button class="quiz-choice" data-value="C">C) Because Firebase defaults can change at any time without notice, making tests non-deterministic</button>
<button class="quiz-choice" data-value="D">D) Because Firebase only works with mobile apps</button>
</div>
<p class="quiz-explanation">If your test depends on a flag being in a certain state but doesn't explicitly set it, someone changing the Firebase default will break your test. Explicit overrides make tests deterministic and immune to remote changes.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q5.</strong> A new feature requires <code>llmNewStakingFlow</code> to be enabled. How would you test it in mobile E2E?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>await app.init({ featureFlags: { llmNewStakingFlow: { enabled: true } } })</code></button>
<button class="quiz-choice" data-value="B">B) Edit the Firebase console to enable the flag globally</button>
<button class="quiz-choice" data-value="C">C) Set <code>STAKING_FLOW=new</code> environment variable</button>
<button class="quiz-choice" data-value="D">D) Modify the feature flag source code to hardcode <code>true</code></button>
</div>
<p class="quiz-explanation">Mobile E2E tests override feature flags through the <code>app.init()</code> method, which sends the overrides to the React Native app via the WebSocket bridge.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Allure Reporting & Xray Integration

<div class="chapter-intro">
Running tests without reading reports is like writing code without running it. <strong>Allure</strong> is the test reporting framework used by Ledger Live E2E tests. It transforms raw test results into interactive HTML reports with step-by-step traces, screenshots, videos, and environment data. Combined with <strong>Xray</strong> integration, it connects automated test results to Jira test case definitions. This chapter shows you how to generate, read, and interpret these reports.
</div>

### 12.1 What Is Allure?

Allure is a **test reporting framework** that generates interactive HTML reports with:
- Test results (passed, failed, broken, skipped)
- Step-by-step execution trace (from `@step` decorators)
- Screenshots on failure
- Video recordings
- Environment information
- Links to Jira/Xray test cases

### 12.2 Viewing Reports Locally

```bash
# Desktop:
pnpm e2e:desktop allure:generate   # Generate from ./allure-results
pnpm e2e:desktop allure:open       # Open in browser
pnpm e2e:desktop allure             # Combined (generate + open)

# Mobile (from e2e/mobile/):
pnpm allure
```

### 12.3 Reports in CI

Reports are automatically uploaded to: `https://ledger-live.allure.green.ledgerlabs.net`

The URL is printed in the GitHub Actions job summary. Look for "Desktop Allure report URL" or "Mobile Allure report URL".

### 12.4 How @step Creates the Report

The `@step` decorator on page object methods automatically populates the Allure report with a step-by-step trace:

```typescript
@step("Click Send button")
async clickSend() { await this.sendButton.click(); }

@step("Navigate to token $0")
async navigateToToken(name: string) { await this.page.getByText(name).click(); }
```

In Allure, this produces:
```
Test: ERC20 token with 0 balance is hidden
  + Step: Open target from main navigation "accounts"    [0.5s] PASSED
  + Step: Show parent account tokens "Ethereum 1"        [0.3s] PASSED
  + Step: Verify token visibility "USDT"                 [0.2s] PASSED
  + Step: Click hide empty token accounts toggle          [0.1s] PASSED
  + Step: Verify children tokens are not visible "USDT"  [0.2s] PASSED
```

### 12.5 TMS Links (Jira/Xray)

```typescript
test("My test", {
  annotation: { type: "TMS", description: "B2CQA-817" },
}, async ({ app }) => {
  await addTmsLink(getDescription(test.info().annotations, "TMS").split(", "));
  // ...
});
```

This creates clickable links in Allure to: `https://ledgerhq.atlassian.net/browse/B2CQA-817`

### 12.6 Xray JSON Export

Test results are exported for Jira/Xray import via a custom reporter:

**File**: `e2e/desktop/tests/utils/customJsonReporter.ts`

```json
{
  "tests": [
    { "testKey": "B2CQA-817", "status": "PASSED" },
    { "testKey": "B2CQA-804", "status": "PASSED" },
    { "testKey": "B2CQA-2574", "status": "FAILED" }
  ]
}
```

This import syncs automated test results with Xray test executions in Jira, giving the QA team a unified view of test coverage across manual and automated tests.

### 12.7 Reading an Allure Report Effectively

When you open an Allure report, focus on these sections:

1. **Overview** — Pass/fail ratio, test count, duration
2. **Suites** — Tests grouped by describe block (your spec files)
3. **Graphs** — Visual breakdown of results by status, severity, duration
4. **Timeline** — When tests ran, useful for detecting parallelism issues
5. **Categories** — Failure categories (product defect vs test defect)

For a **failed test**, drill into:
1. The **step trace** — which step failed and its duration
2. The **screenshot** — what the UI looked like at failure time
3. The **video** — the full test execution recording
4. The **trace** — Playwright trace file for detailed debugging
5. The **TMS link** — jump to the Jira test case for expected behavior

### 12.8 Allure Annotations in Practice

Beyond `@step` and TMS links, you can add more metadata to your tests:

```typescript
import { allure } from "allure-playwright";

test("My test", async ({ app }) => {
  await allure.severity("critical");
  await allure.feature("Send");
  await allure.story("Send Bitcoin to external address");
  await allure.owner("qaa-team");
  // ...
});
```

These annotations feed into Allure's filtering and grouping capabilities, making it easier to find and analyze specific test results.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://allurereport.org/docs/">Allure Report documentation</a> — by Qameta Software</li>
<li><a href="https://allurereport.org/docs/playwright/">Allure Playwright integration</a></li>
<li><a href="https://docs.getxray.app/display/XRAY/About+Xray">Xray documentation</a> — the Jira test management plugin</li>
<li><a href="https://docs.getxray.app/display/XRAY/Import+Execution+Results+-+REST+v2">Xray REST API for importing results</a></li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> The Allure report is your first stop when a test fails. The step trace tells you <em>what</em> went wrong, the screenshot tells you <em>what the user would see</em>, and the TMS link tells you <em>what should have happened</em>. Learn to read reports fluently — it will save you hours of debugging.
</div>

### 12.9 Quiz

<!-- ── Chapter 12 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> What does a TMS annotation like <code>B2CQA-817</code> refer to?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A Git branch name</button>
<button class="quiz-choice" data-value="B">B) A Jira/Xray test case ID</button>
<button class="quiz-choice" data-value="C">C) A Firebase feature flag</button>
<button class="quiz-choice" data-value="D">D) A Speculos device identifier</button>
</div>
<p class="quiz-explanation">TMS = Test Management System. <code>B2CQA-XXX</code> is a Jira/Xray test case ID that links the automated test to its manual test case definition. Allure creates a clickable link to the Jira issue.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> How do you generate and view an Allure report for desktop E2E tests?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>npx allure serve</code></button>
<button class="quiz-choice" data-value="B">B) <code>pnpm desktop allure</code></button>
<button class="quiz-choice" data-value="C">C) <code>pnpm e2e:desktop allure</code></button>
<button class="quiz-choice" data-value="D">D) <code>pnpm test:allure</code></button>
</div>
<p class="quiz-explanation"><code>pnpm e2e:desktop allure</code> is the combined command that generates the report from <code>./allure-results</code> and opens it in a browser. Note the <code>e2e:desktop</code> prefix (not <code>desktop</code>).</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> What creates the step-by-step trace visible in the Allure report?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The <code>@step</code> decorator on page object methods</button>
<button class="quiz-choice" data-value="B">B) Playwright's built-in trace recorder</button>
<button class="quiz-choice" data-value="C">C) Jest's test lifecycle hooks</button>
<button class="quiz-choice" data-value="D">D) Console.log statements in the test code</button>
</div>
<p class="quiz-explanation">The <code>@step</code> decorator (from <code>e2e/desktop/tests/misc/reporters/step.ts</code>) wraps each page object method and reports its execution to Allure, creating the collapsible step tree with timing information.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> A test fails in CI. What is the most efficient order to investigate using the Allure report?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Video → Screenshot → Step trace → TMS link</button>
<button class="quiz-choice" data-value="B">B) TMS link → Video → Screenshot → Step trace</button>
<button class="quiz-choice" data-value="C">C) Re-run the test → Read the code → Check logs</button>
<button class="quiz-choice" data-value="D">D) Step trace (which step failed) → Screenshot (UI state at failure) → Video (full execution) → TMS link (expected behavior)</button>
</div>
<p class="quiz-explanation">Start with the step trace to identify which step failed and how long it took. Then check the screenshot to see the UI state. The video shows the full execution if context is needed. The TMS link shows what the test should have done.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> What does the Xray JSON export enable?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It publishes test results to the Allure dashboard</button>
<button class="quiz-choice" data-value="B">B) It syncs automated test results with Xray test executions in Jira, giving a unified view of manual and automated coverage</button>
<button class="quiz-choice" data-value="C">C) It exports test code to a JSON format for backup</button>
<button class="quiz-choice" data-value="D">D) It sends Slack notifications when tests fail</button>
</div>
<p class="quiz-explanation">The custom JSON reporter (<code>customJsonReporter.ts</code>) exports results in Xray format, which are imported into Jira to sync automated test statuses with their corresponding manual test case definitions.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Part 2 Final Assessment

<div class="quiz-container" data-pass-threshold="80">
<h3>Part 2 Final Assessment</h3>
<p class="quiz-subtitle">10 questions across all chapters · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> Which branch naming prefix should you use for adding a new E2E test?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>bugfix/</code></button>
<button class="quiz-choice" data-value="B">B) <code>feat/</code></button>
<button class="quiz-choice" data-value="C">C) <code>chore/</code></button>
<button class="quiz-choice" data-value="D">D) <code>support/</code></button>
</div>
<p class="quiz-explanation"><code>feat/</code> — New tests are new features. Use <code>test(e2e): ...</code> for the commit type within the Conventional Commits format.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> What does <code>pnpm i --filter="ledger-live-desktop..."</code> do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Installs only the desktop app's direct dependencies</button>
<button class="quiz-choice" data-value="B">B) Installs the desktop app AND all its transitive dependencies</button>
<button class="quiz-choice" data-value="C">C) Filters out the desktop app from installation</button>
<button class="quiz-choice" data-value="D">D) Installs only dev dependencies for the desktop app</button>
</div>
<p class="quiz-explanation">The <code>...</code> suffix means "this package and all packages it depends on (transitively)". This installs the complete dependency tree needed to build and run the desktop app.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q3.</strong> In Playwright's locator priority, which should you use first?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) CSS selector</button>
<button class="quiz-choice" data-value="B">B) <code>getByText</code></button>
<button class="quiz-choice" data-value="C">C) <code>getByTestId</code></button>
<button class="quiz-choice" data-value="D">D) XPath</button>
</div>
<p class="quiz-explanation"><code>getByTestId</code> is the most stable locator — it won't break on text changes (translations) or CSS refactors. Test IDs are added specifically for test automation.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q4.</strong> How does the mobile E2E suite communicate with the React Native app?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Direct DOM access</button>
<button class="quiz-choice" data-value="B">B) WebSocket bridge</button>
<button class="quiz-choice" data-value="C">C) HTTP REST API</button>
<button class="quiz-choice" data-value="D">D) Shared memory</button>
</div>
<p class="quiz-explanation">The WebSocket bridge (<code>e2e/mobile/bridge/server.ts</code>) sends commands like <code>importAccounts</code>, <code>overrideFeatureFlag</code>, and <code>navigate</code> from Jest/Detox to the running React Native app.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> Which Speculos endpoint takes a screenshot?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>POST /screenshot</code></button>
<button class="quiz-choice" data-value="B">B) <code>GET /screenshot</code></button>
<button class="quiz-choice" data-value="C">C) <code>GET /screen</code></button>
<button class="quiz-choice" data-value="D">D) <code>POST /capture</code></button>
</div>
<p class="quiz-explanation"><code>GET /screenshot</code> returns a PNG image of the device screen. It's a GET because you are retrieving data, not modifying state.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q6.</strong> What are the four Firebase environments used by Ledger Live?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Dev, Staging, Production, Canary</button>
<button class="quiz-choice" data-value="B">B) Ledger Wallet, Swap, Earn, Buy Sell</button>
<button class="quiz-choice" data-value="C">C) Desktop, Mobile, CLI, Web</button>
<button class="quiz-choice" data-value="D">D) Alpha, Beta, RC, Stable</button>
</div>
<p class="quiz-explanation">Each environment manages feature flags for a different product area: Ledger Wallet (main features), Swap (exchange), Earn (staking), and Buy Sell (fiat on/off ramp).</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q7.</strong> During the release process, what is the purpose of the <code>nightly</code> branch?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It is a staging branch that merges into <code>main</code> weekly</button>
<button class="quiz-choice" data-value="B">B) It replaces <code>develop</code> during release freezes</button>
<button class="quiz-choice" data-value="C">C) It is used for hotfix deployments</button>
<button class="quiz-choice" data-value="D">D) It is a standalone long-lived branch that receives <code>develop</code> merges nightly and publishes <code>@nightly</code> tagged packages — it never merges anywhere</button>
</div>
<p class="quiz-explanation">The <code>nightly</code> branch is standalone and permanent. Every night, <code>develop</code> is merged into it, and the standard pre-release cycle publishes libraries under the <code>@nightly</code> tag and builds desktop/mobile apps through nightly channels.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q8.</strong> The Ledger Live Bot is described as "stateless." What does this mean?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It doesn't use any databases</button>
<button class="quiz-choice" data-value="B">B) It doesn't remember previous test runs in a local file</button>
<button class="quiz-choice" data-value="C">C) Its state is on the blockchain — it restores from existing seed accounts and continues from the current blockchain state</button>
<button class="quiz-choice" data-value="D">D) It uses a fresh seed for every run</button>
</div>
<p class="quiz-explanation">The bot doesn't maintain local state. It derives account state from the blockchain itself by scanning accounts for its seed. Each run resumes from whatever the current blockchain state is.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q9.</strong> What is the most important file in the desktop E2E suite and why?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>e2e/desktop/tests/fixtures/common.ts</code> — it defines the fixture system that configures and injects everything a test needs</button>
<button class="quiz-choice" data-value="B">B) <code>e2e/desktop/playwright.config.ts</code> — it defines all test timeouts</button>
<button class="quiz-choice" data-value="C">C) <code>e2e/desktop/tests/specs/index.ts</code> — it is the test entry point</button>
<button class="quiz-choice" data-value="D">D) <code>e2e/desktop/package.json</code> — it lists all dependencies</button>
</div>
<p class="quiz-explanation"><code>common.ts</code> defines the fixture system that orchestrates the entire test lifecycle: launching Electron, setting up userdata, starting Speculos, injecting page objects, overriding feature flags, and tearing down. Every test depends on it.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q10.</strong> A flaky test sometimes passes and sometimes fails in CI. The failure is always "Element not found" for a button that should appear after a feature flag is enabled. What is the most likely root cause?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The CI runner is too slow</button>
<button class="quiz-choice" data-value="B">B) The Playwright version is outdated</button>
<button class="quiz-choice" data-value="C">C) The test ID was changed in a recent commit</button>
<button class="quiz-choice" data-value="D">D) The test relies on a Firebase flag default instead of explicitly overriding it, and the default has been changed intermittently</button>
</div>
<p class="quiz-explanation">This is the classic "implicit flag dependency" anti-pattern. The test doesn't explicitly set the feature flag, so it depends on the Firebase default. If someone changes the default (even temporarily), the test fails. The fix: always explicitly override all flags your test depends on.</p>
</div>

<div class="quiz-score"></div>
</div>

---

