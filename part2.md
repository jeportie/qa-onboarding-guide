## Git Workflow, Hooks & Changesets

<div class="chapter-intro">
Every contribution you make — whether it is a new E2E test, a bug fix, or a page object refactor — flows through Git. This chapter covers the branching model, naming conventions, commit format, pre-commit hooks, and the changeset system that Ledger Live uses for versioning its 200+ packages. Getting this right from day one prevents friction in code review and CI.
</div>

### 2.1.1 Branch Strategy

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

### 2.1.2 Branch Naming Convention

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

### 2.1.3 Commit Message Format (Conventional Commits)

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

### 2.1.4 Pre-Commit Hooks

The repo uses **hk** (a modern Husky alternative) configured in `hk.pkl`. A single pre-commit hook runs:

**`gitleaks`** — scans staged files for accidental secrets (API keys, passwords, tokens, private keys).

```bash
# If your commit is blocked by gitleaks:
# 1. It means you're about to commit a secret!
# 2. Find the secret in your staged files
# 3. Remove it (use environment variables instead)
# 4. Try committing again
```

### 2.1.5 Changesets

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

### 2.1.6 PR Workflow

1. Create a feature branch from `develop`
2. Make your changes, commit with Conventional Commits
3. Push and open a PR targeting `develop`
4. CI runs automatically (commitlint, lint, typecheck, unit tests, smoke E2E)
5. The `@Gate` job must pass — this is the required check that blocks merging
6. Request review from CODEOWNERS team (your team: `@ledgerhq/qaa` for E2E)
7. Merge when approved + CI green

### 2.1.7 Common Git Issues in the Monorepo

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

### 2.1.8 Quiz

<!-- ── Chapter 2.1 Quiz ── -->

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

### 2.2.1 pnpm Basics

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

### 2.2.2 Package Aliases

The root `package.json` defines shortcuts so you don't need to type long filter expressions:

```bash
pnpm desktop ...       # = pnpm --filter ledger-live-desktop ...
pnpm mobile ...        # = pnpm --filter live-mobile ...
pnpm e2e:desktop ...   # = pnpm --filter ledger-live-desktop-e2e-tests ...
pnpm e2e:mobile ...    # = pnpm --filter ledger-live-mobile-e2e-tests ...
```

### 2.2.3 The Filter System

The `--filter` flag selects which packages to operate on:

```bash
--filter="ledger-live-desktop"        # Exact package name
--filter="ledger-live-desktop..."     # Package + all its dependencies (note: ...)
--filter="@ledgerhq/live-common"      # Scoped package
--filter="[origin/develop]"           # Packages changed since develop
```

### 2.2.4 Turbo (Build Orchestration)

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

### 2.2.5 When TypeCheck Fails on an Import

If you see:
```
error TS2305: Module '"@ledgerhq/live-countervalues-react"' has no exported member 'useMarketcapIds'.
```

The library needs to be **rebuilt** — its build output is stale:
```bash
pnpm turbo build --filter=@ledgerhq/live-countervalues-react
```

### 2.2.6 Live Reload During Development

If you are modifying a library package while working on LLD or LLM, you can get on-the-fly updates:

```bash
# Watch a package for changes and rebuild automatically
pnpm --filter="@ledgerhq/hw-transport" run watch
```

> **Tip from the wiki**: For `live-common`, run the watch command **before** starting the application, or the hot reload can destabilize the running app.

### 2.2.7 Dependency Duplicates Management

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

### 2.2.8 Windows Considerations

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

### 2.2.9 Quiz

<!-- ── Chapter 2.2 Quiz ── -->

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


## Speculos — Device Emulation

<div class="chapter-intro">
Speculos is the Ledger device emulator — a QEMU-based program that runs real device firmware and real coin apps on your laptop, exposing a REST API that tests can drive programmatically. It is what makes end-to-end testing possible without shipping a Nano X to every engineer. This chapter teaches Speculos from zero: what it is, how it works, the six device models it supports, the REST endpoints, the coin-apps repository, environment variables, and the Docker lifecycle. It then covers how Ledger Live's E2E framework integrates Speculos into the test fixture, and ends with the Ledger Live Bot — an autonomous transaction system built on top of Speculos that runs real blockchain transactions on seven rotated seeds.

By the end of this chapter you should be able to answer these questions without opening a second tab:

- What does Speculos emulate, and what does it not?
- Which device model should I target by default, and why?
- What are the two display frameworks and how do they change interaction?
- What is the difference between a button press, a touch, and an APDU?
- What must I set up on a fresh machine before running a Speculos test?
- How do I drive Speculos by hand, outside the test framework, when I want to reproduce a device-only bug?
- What is the Ledger Live Bot and where does it fit against Playwright/Detox E2E?
</div>

### 2.3.1 What Is Speculos?

Speculos is a **Ledger device emulator** that runs real firmware inside a Docker container. Under the hood it emulates the ARM processor of a Ledger device using QEMU, loads the actual `.elf` binary of a device app (Bitcoin, Ethereum, Solana, etc.), and runs it in a sandboxed environment. It supports all six device models currently produced by Ledger: Nano S, Nano S Plus, Nano X, Stax, Flex, and Nano Gen 5.

- **Source**: `github.com/LedgerHQ/speculos`
- **Written in**: C (ARM emulation core) + Python (REST API, automation)
- **Image**: `ghcr.io/ledgerhq/speculos:latest`
- **Official docs**: https://speculos.ledger.com/

Speculos exposes three surfaces:

- A **REST API** (default port 5000) for button presses, screenshots, APDU commands, and automation rules
- A **TCP server** (default port 9999) for raw APDU communication
- A **web UI** at `http://localhost:5000` that renders the emulated screen in your browser — invaluable for visual debugging

In E2E tests, Speculos runs inside a **Docker container**. Each test gets its own container on a unique port so that test workers can run in parallel without stepping on each other.

> **Note:** You never "install" Speculos locally in the classic sense. You run the Docker image, which ships with a specific firmware set. The device apps (Bitcoin, Ethereum, …) come from a separate repository: `coin-apps`. Keep both in sync.

#### A tiny mental model

Think of Speculos as three nested layers:

1. A **Docker container** — the unit of isolation. One test worker, one container, one port.
2. Inside the container, a **QEMU process** that emulates the ARM Cortex-M CPU used by Ledger devices. QEMU runs the exact firmware image that ships on real hardware — including the secure element stubs, the BOLOS operating system, and the display framework (BAGL or NBGL).
3. On top of the firmware, a **device app** (`Bitcoin.elf`, `Ethereum.elf`, …) loaded from the `coin-apps` repo. The app is what actually signs transactions.

Around the emulated CPU, a Python layer wraps the button GPIOs and the screen framebuffer, and exposes them over HTTP. That is the REST API — your tests' only access point into the device.

```
+------------------------------------------------------------+
|                    Test process (Node)                     |
|   Playwright / Detox  →  live-common transport             |
+------------------------------------------------------------+
                         |  HTTP (REST + APDU)
                         v
+------------------------------------------------------------+
|                   Docker container                         |
|   +-----------------------------------------------------+  |
|   |         Speculos Python layer (REST/TCP)            |  |
|   +-----------------------------------------------------+  |
|   |         QEMU ARM Cortex-M emulation core            |  |
|   +-----------------------------------------------------+  |
|   |         BOLOS firmware + display framework          |  |
|   +-----------------------------------------------------+  |
|   |         Coin app .elf (Bitcoin, Ethereum, …)        |  |
|   +-----------------------------------------------------+  |
+------------------------------------------------------------+
```

Read that diagram top-to-bottom when a test fails: the higher the problem, the cheaper it is to diagnose. A Playwright selector miss is seconds to fix; a QEMU-level emulation bug is a ticket against the Speculos repo.

### 2.3.2 Why Speculos Exists

Without Speculos, an E2E test that signs a transaction would need:

- A physical Ledger device connected via USB
- A human pressing buttons or tapping the screen at the right moment
- A seed loaded on that device with real or test funds

Running CI that way is impossible. Speculos replaces the hardware with software:

- Real firmware (same `.elf` that ships to production devices, bit-for-bit)
- Real coin apps (same `.elf` a user would install via Ledger Live Manager)
- REST endpoints instead of fingers — tests press buttons through HTTP calls

Because the firmware and apps are the real thing, a transaction signed on Speculos is cryptographically valid. The Ledger Live Bot (covered in 2.3.10) takes advantage of this by broadcasting real signed transactions to real testnet and mainnet blockchains.

#### What Speculos is not

A few limits are worth stating up front so you do not waste time fighting them:

- **Not a secure element.** Real Ledger devices store private keys in a certified secure element chip; Speculos derives keys from a BIP39 seed held in memory. Do not treat Speculos screenshots as a substitute for attestation or hardware-security review.
- **Not a USB device.** Speculos speaks HTTP, not HID/USB. The `live-common` transport layer abstracts that difference, but any test that pokes real USB internals will not work.
- **Not a UI substitute.** Speculos emulates what a device shows — not what the Ledger Live desktop or mobile app shows. For that you need Playwright/Detox on top. The E2E architecture is always "Playwright/Detox drives the app UI; the app drives Speculos; Speculos drives the device app".
- **Not a perfect match for every firmware version.** Speculos lags real-hardware firmware by a few weeks at times. If you see behavior that differs from a physical device, bump `SPECULOS_IMAGE_TAG` first.

### 2.3.3 Device Models in Detail

Speculos supports all six current Ledger device models. The model you emulate affects interaction (buttons vs touch), display framework (BAGL vs NBGL), and which coin-app binary gets loaded.

| Device | Env value | Touch | Display | Screen | Notes |
|--------|-----------|-------|---------|--------|-------|
| Nano S | `nanoS` | No | BAGL | 128x32 | Legacy. Limited memory. Requires a specific Docker SHA. |
| Nano S Plus | `nanoSP` | No | BAGL | 128x64 | Most common test target. Good default. |
| Nano X | `nanoX` | No | BAGL | 128x64 | Has Bluetooth in production (not relevant in Speculos). |
| Stax | `stax` | **Yes** | NBGL | 400x672 | Touchscreen. Tap, swipe, long press. |
| Flex | `flex` | **Yes** | NBGL | 480x600 | Touchscreen. Different form factor than Stax. |
| Nano Gen 5 | `nanoGen5` | **Yes** | NBGL | — | Newest. Internally codenamed "apex" in coin-apps. |

**BAGL vs NBGL.** These are the two display frameworks used by Ledger devices.

- **BAGL** (Bolos Application Graphics Library) is the older system for button-based Nano devices. It is monochrome, character-oriented, and drives pagination through left/right/both button events.
- **NBGL** (New Bolos GL) is the modern framework for touchscreen devices (Stax, Flex, Gen 5). It uses high-level widgets (review pages, long-press confirmations, swipe navigation) rendered on a color display.

The display framework affects how E2E tests interact with the emulated device — buttons vs touch gestures. You cannot share the same page object between a Nano SP and a Stax; the interaction grammar is different.

> **Note:** Device tags in tests (`@NanoSP`, `@NanoX`, `@Stax`, `@Flex`, `@NanoGen5`) indicate which device models a test supports. In CI, the matrix runs each supported device model for every tagged test.

#### Choosing a default device for local work

Most day-to-day E2E work targets **Nano S Plus** (`nanoSP`). Reasons:

- It is the most common test target — the majority of existing tests assume it.
- It has a BAGL display, which keeps the interaction grammar simple (two buttons).
- The coin-apps repo has the widest coverage for `nanoSP` across currencies.

When you add a test, start with `nanoSP`, then widen coverage only once the behavior is stable. Add `@Stax`/`@Flex` tags (and rework the interaction layer for touch) once the logic is solid on Nano.

### 2.3.4 The REST API

Speculos exposes a comprehensive REST API. Even though tests usually interact with Speculos through the `live-common` library (not direct HTTP calls), understanding the raw API is essential for debugging.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | Health check (alive?) |
| `/events` | GET | Get display events (text currently on screen) |
| `/apdu` | POST | Send raw APDU command |
| `/button/left` | POST | Press left button (non-touch) |
| `/button/right` | POST | Press right button (non-touch) |
| `/button/both` | POST | Press both buttons = confirm (non-touch) |
| `/finger` | POST | Touch event with x,y coordinates (touch devices) |
| `/screenshot` | GET | Take PNG screenshot of the display |
| `/` | DELETE | Kill the session |
| `/automation` | POST | Set automation rules (auto-approve text matches) |

Concrete examples you can copy-paste when a test is running and you want to poke the device directly:

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

The port (`5023` in the examples) is assigned randomly per test. Look for it in the test console output or the Allure description.

**Reference**: full endpoint list at https://speculos.ledger.com/user/launch.html.

#### A short APDU primer

APDU stands for Application Protocol Data Unit. It is the binary protocol every smartcard speaks, and Ledger devices are smartcards at heart. When Ledger Live "asks the device to sign a transaction", what actually crosses the wire is a sequence of APDUs. A command APDU looks like this:

```
| CLA | INS | P1 | P2 | Lc | Data (Lc bytes) | Le |
```

- **CLA** — class byte (which app's instruction set)
- **INS** — instruction byte (what to do)
- **P1, P2** — parameters
- **Lc** — length of `Data`
- **Le** — expected response length

Speculos's `POST /apdu` accepts the raw hex bytes of a command APDU and returns the device's response APDU. You will rarely hand-craft APDUs in E2E tests — `live-common` does that for you — but knowing the shape helps when you read logs that contain lines like `=> e0c4000000` (command) and `<= 9000` (status word "OK").

#### Automation rules

The `/automation` endpoint lets you pre-program "if the screen ever shows X, press Y". This is how long, deterministic flows stay tractable — instead of scripting every button press, you install a ruleset and let Speculos auto-advance.

```json
{
  "version": 1,
  "rules": [
    { "text": "Review", "actions": [["button", 2, true], ["button", 2, false]] },
    { "text": "Approve", "actions": [["button", 2, true], ["button", 2, false]] }
  ]
}
```

Rule of thumb: if your test repeats the same "press both buttons until we see Approve" dance in multiple places, hoist it into an automation ruleset.

#### A full curl-driven session

To make the REST API click, here is a hand-driven session against a running Speculos on port 5023 (Bitcoin app, Nano SP):

```bash
# 1. Confirm the emulator is alive.
curl http://localhost:5023/

# 2. See what the screen shows at boot.
curl http://localhost:5023/events
# => {"events":[{"text":"Bitcoin","x":41,"y":12}, ...]}

# 3. Ask the app for a receive address (the app will prompt on the screen).
curl -d '{"data":"e04000000d058000002c8000000080000000000000000000000000"}' \
     http://localhost:5023/apdu
# The app now shows "Verify receive address" and blocks.

# 4. Press right to page through the address.
curl -d '{"action":"press-and-release"}' http://localhost:5023/button/right

# 5. Press both to approve.
curl -d '{"action":"press-and-release"}' http://localhost:5023/button/both

# 6. The original /apdu call unblocks and returns the address + status 9000.

# 7. Snapshot for the record.
curl http://localhost:5023/screenshot -o receive.png
```

Running through that flow once, by hand, is worth more than reading ten pages of documentation. Do it early in your onboarding.

### 2.3.5 Non-Touch vs Touch Interaction

The API split above reflects a deeper grammar difference. Your tests should never hard-code interactions — they should call helpers that dispatch based on `SPECULOS_DEVICE`.

**Non-touch devices** (Nano S, Nano S Plus, Nano X) use the two physical buttons:

```typescript
buttons.right();     // Next screen
buttons.right();     // Next screen
buttons.both();      // Confirm
```

**Touch devices** (Stax, Flex, Gen 5) use taps, long presses, and swipes:

```typescript
pressAndRelease("Sign transaction", x, y);   // Tap on a label
longPressAndRelease("Hold to sign", 3);       // Long press for 3 seconds
swipeRight();                                  // Swipe gesture
```

> **Note:** A test written for Nano devices that relies on button sequences will not run unmodified on Stax. Conversely, tests written with touch helpers cannot execute on a Nano. This is why device tags exist — they gate which device models a given test claims to support.

#### A worked walkthrough: sending 0.01 BTC on Nano SP

To make the grammar concrete, here is what happens inside Speculos when Ledger Live asks the user to confirm a 0.01 BTC transaction on a Nano SP:

1. The app sends an APDU: "please display and sign this transaction". Speculos forwards it to the emulated Bitcoin app.
2. The Bitcoin app parses the transaction and draws the first screen: `Review transaction`. A test observer sees `GET /events` return that text.
3. The test (or an automation rule) calls `POST /button/right`. Screen advances to `Amount: 0.01 BTC`.
4. Another `POST /button/right`. Screen: `Fees: 0.00002 BTC`.
5. Another `POST /button/right`. Screen: `Accept and send?`.
6. The test calls `POST /button/both`. The Bitcoin app reads that as "confirm", signs the transaction, and returns the signed payload in a response APDU.
7. Ledger Live broadcasts. The test asserts against the resulting account state.

On Stax the same flow uses three screens of a review widget, a swipe to advance, and a long-press-to-sign at the end. The HTTP grammar is different (`POST /finger` with coordinates) but the overall narrative is identical.

### 2.3.6 The Coin-Apps Repository

Speculos emulates the firmware, but not the device apps. Device apps (`Bitcoin.elf`, `Ethereum.elf`, `Solana.elf`, …) live in a separate repository: `github.com/LedgerHQ/coin-apps`. This repo contains compiled `.elf` binaries organized by device model, firmware version, app name, and app version:

```
coin-apps/
├── nanoS/1.6.2/Bitcoin/app_2.5.0.elf
├── nanoSP/1.8.1/Bitcoin/app_2.7.0.elf
├── nanoSP/1.8.1/Ethereum/app_1.13.0.elf
├── stax/.../
├── flex/.../
└── apex/.../            # Nano Gen 5 (codename)
```

Local setup:

```bash
git clone https://github.com/LedgerHQ/coin-apps.git ~/coin-apps
export COINAPPS=~/coin-apps

# Keep it updated frequently — new apps and versions land constantly:
cd ~/coin-apps && git pull
```

> **Note:** The #2 cause of "App not found" errors in E2E tests (after Docker issues) is a stale `coin-apps` checkout. If a test fails with `Error: no app found for Solana on nanoSP`, your first reflex should be `cd $COINAPPS && git pull`.

#### Pinning app versions

`coin-apps` keeps multiple app versions side-by-side. Tests can pin a specific `(device, firmware, appName, appVersion)` tuple, which is what the `appQuery` field in the bot spec above does. Pinning is important for two reasons:

- **Reproducibility.** A test that passes today on `Bitcoin 2.7.0` should not silently start running against `2.8.0` tomorrow.
- **Bug triage.** When a behavior changes, knowing the exact app version narrows the search to a single commit range.

When adding a test for a new currency, copy the version pin from a similar existing test and only bump it once you have a green baseline.

### 2.3.7 Environment Variables

Speculos and the test framework communicate through environment variables. You must set the required ones before running any Speculos-based test.

```bash
# REQUIRED — set these before running tests
export MOCK=0                                            # Disable mocks, use real Speculos
export SPECULOS_DEVICE=nanoSP                            # Which device to emulate
export SEED="abandon abandon ... about"                  # Recovery phrase (CI secret; ask QAA team)
export COINAPPS=~/coin-apps                              # Path to .elf binaries
export SPECULOS_IMAGE_TAG=ghcr.io/ledgerhq/speculos:latest   # Docker image (use :master in some setups)

# OPTIONAL — usually managed by fixtures, do not set manually unless debugging
# export SPECULOS_API_PORT=5000                          # REST API port (auto-assigned per test)
# export SPECULOS_ADDRESS=http://127.0.0.1               # Speculos host
# export REMOTE_SPECULOS=true                            # Use remote Speculos (CI only)
```

**`MOCK=0` is the switch that matters most.** When `MOCK=1`, the Ledger Live app uses a fake device that returns canned responses — fast, but not representative. When `MOCK=0`, the app talks to the actual Speculos emulator via the port in `SPECULOS_API_PORT`. All Speculos-based E2E tests require `MOCK=0`.

> **Note:** `SEED` is a real BIP39 mnemonic. It controls real addresses on real blockchains (testnet or mainnet). Never commit a seed. The QAA team manages the seven shared bot seeds — use a dedicated seed with minimal funds for local experiments (less than ~$10 per coin).

#### First-day setup: getting a Speculos test to run locally

Assume a fresh macOS or Linux workstation with Docker installed and the Ledger Live monorepo checked out. The path from zero to "a green Speculos test in my terminal" is:

1. **Install Docker** and confirm it is running: `docker info` should print cluster info without errors. On macOS, Docker Desktop must be launched manually before every session.
2. **Pull the Speculos image**: `docker pull ghcr.io/ledgerhq/speculos:latest`. This is a few hundred megabytes the first time.
3. **Clone coin-apps** somewhere outside the monorepo: `git clone https://github.com/LedgerHQ/coin-apps.git ~/coin-apps`. Checking it out inside the monorepo will confuse the pnpm workspace resolver.
4. **Ask QAA for a non-production seed.** Do not reuse a personal seed. Do not commit it. Store it in your shell profile or a `direnv` file that is gitignored.
5. **Export the env vars** described above (`MOCK=0`, `SPECULOS_DEVICE=nanoSP`, `COINAPPS`, `SEED`, `SPECULOS_IMAGE_TAG`).
6. **Run a known-good Speculos test.** Pick the simplest tagged `@Speculos @NanoSP` test in the desktop suite and run it via the standard pnpm command. If that test is green, your setup is correct.

If step 6 fails, walk back through the list. Every failure at this stage has one of three causes: Docker not running, `COINAPPS` wrong or stale, or `SEED` missing.

#### A minimal `.env` for a new contributor

If you have just cloned the monorepo and want a working Speculos setup, start with this and adjust:

```bash
# ~/.config/ledger-live-e2e/env.local (or whatever loader your shell uses)
MOCK=0
SPECULOS_DEVICE=nanoSP
SPECULOS_IMAGE_TAG=ghcr.io/ledgerhq/speculos:latest
COINAPPS=/Users/me/code/coin-apps
SEED="ask QAA for the dev seed, do not paste a real one"
```

Source the file, run `docker pull $SPECULOS_IMAGE_TAG`, `cd $COINAPPS && git pull`, and you are ready to run any Speculos-tagged test.

### 2.3.8 Docker Commands You Will Actually Use

Speculos runs inside Docker. When things go wrong, Docker is the first place to look.

```bash
# Pull the latest image
docker pull ghcr.io/ledgerhq/speculos:latest

# Verify the image is present
docker images | grep speculos

# List running Speculos containers (should match running test workers)
docker ps --filter name=speculos

# Kill leftover containers from crashed or interrupted tests
docker rm -f $(docker ps -aq --filter name=speculos)

# View logs from a specific container
docker logs <container-id>
```

**The single most common failure mode** is a stuck container from a previous run. If you interrupted a test with `Ctrl+C` and the next run fails with port conflicts or "container not ready", the fix is almost always:

```bash
docker rm -f $(docker ps -a -q --filter ancestor=ghcr.io/ledgerhq/speculos:latest)
```

#### One-off Speculos, without the test framework

Sometimes you want to poke Speculos by hand — for example, to reproduce a device-side bug outside the full E2E stack. You can launch a container directly:

```bash
docker run --rm -it \
  -p 5000:5000 -p 9999:9999 \
  -v $COINAPPS/nanoSP/1.8.1:/speculos/apps \
  ghcr.io/ledgerhq/speculos:latest \
  /speculos/apps/Bitcoin/app_2.7.0.elf \
  --display headless --api-port 5000 --model nanosp
```

Then open `http://localhost:5000` in your browser. You will see the emulated Bitcoin app boot, and you can drive it with `curl` against port 5000. This is the purest form of Speculos — no app framework, no Playwright, just you and the device.

### 2.3.9 Speculos in the Ledger Live Test Framework

The pieces above — REST API, coin-apps, env vars, Docker — are raw Speculos. In the Ledger Live repo, an orchestration layer wraps all of this so that individual tests never have to `docker run` anything. The contract for a test writer is simple:

1. Declare which app the device should launch with, via the `speculosApp` fixture value.
2. Optionally declare `cliCommands` to populate accounts or balances before the test starts.
3. Use the `speculos` fixture in the test; it exposes `current` (the device) and `relaunch(appName)` (switch apps mid-test, e.g., for swap flows).

Behind the scenes, a fixture in `common.ts` does the heavy lifting: it launches a container, waits for it to become ready, assigns a random port, registers the transport with `live-common`, runs any CLI commands, hands the device to your test, and — most importantly — cleans up after the test regardless of outcome.

The port is assigned randomly per container precisely to enable parallel execution. With `fullyParallel: true` and a high `workers` count, many tests run at the same time; each needs its own isolated Speculos instance on a unique port.

#### What the test author writes

From the perspective of somebody writing a new test, the Speculos-specific surface area is small:

```typescript
test.use({
  speculosApp: specs.bitcoin,         // which app to boot the device with
  cliCommands: [addAccountCommand],   // optional: populate accounts before the test runs
});

test("send 0.01 BTC", async ({ page, speculos }) => {
  // ... Playwright-driven UI steps ...
  await confirmOnDevice(speculos.current);   // helper that dispatches on SPECULOS_DEVICE
  // ... assertions on the UI and/or on-chain state ...
});
```

Everything else — container lifecycle, port assignment, transport registration, Allure metadata, cleanup — is the fixture's responsibility. If you find yourself reaching for `docker` or `curl` inside a test body, stop and see whether a helper already exists or should exist.

The fixture also exposes a **mutable handle** rather than a bare device. That is because some tests — swaps in particular — need to switch from app A to app B mid-flow. The handle's `relaunch(appName)` stops the current container and starts a new one, preserving the test's view of "the current device" without littering test code with container management.

> **Cross-reference:** The exact internals of the Ledger Live E2E fixture — `speculos` in `common.ts`, the `launchSpeculos` and `cleanSpeculos` helpers in `speculosUtils.ts`, and the `SpeculosDevice` TypeScript type — are covered in **Part 3 Chapter 3.6 (Desktop E2E Fixtures & Speculos Integration)**. This chapter stays at the tool level; Part 3 walks through the code that binds Speculos to Playwright.

#### What the fixture buys you

When you write `test("...", async ({ speculos }) => { … })`, you inherit five things for free:

1. **A running container** on a unique port, reachable via the standard REST API.
2. **Transport registration** — `live-common` already knows how to talk to this specific Speculos instance, so the in-app device picker resolves to it.
3. **Data population** — if you declared `cliCommands`, accounts are already scanned and balances loaded by the time your test body runs.
4. **Allure integration** — the device name, app name, and app version show up in the report description, so later triage does not require guessing what firmware was in play.
5. **Guaranteed cleanup** — even if the test throws, fails, or the worker is killed, the fixture's `finally` stops the container. That is the difference between a clean CI run and a board full of zombie Docker processes.

#### CI matrix: one test, many devices

In CI, a single test tagged `@NanoSP @Stax @Flex` runs three times — once per device model — with `SPECULOS_DEVICE` overridden per matrix entry. The test code does not change; only the env var does. This is the payoff for keeping all device-specific interaction behind helpers that dispatch on `SPECULOS_DEVICE`: one source of truth, N executions.

Practical consequences:

- If you hard-code `nanoSP` anywhere in a helper, you break the matrix for that test.
- If you add a new device tag to a test, make sure every interaction path it touches has a branch for that device.
- If you find yourself adding `if (device === "stax")` in tests, extract a helper instead. Tests should read as intent, not as device switching.

### 2.3.10 The Ledger Live Bot

On top of Speculos, Ledger Live runs an autonomous testing system called **the Ledger Live Bot**. It is fundamentally different from the E2E tests we have discussed so far: the bot does not drive a UI, does not use Playwright or Detox, and does not assert against screenshots. It uses Speculos and `live-common` directly to perform **real transactions on real blockchains**.

#### Philosophy

- **Stateless**: its state is on the blockchain, not on disk
- **Configless**: only needs a seed and a coin-apps folder
- **Autonomous**: restores from existing seed accounts and continues from existing blockchain state
- **Data driven**: actions are described by data specs (mutations) that declare capabilities
- **End to end**: relies on the complete Ledger stack — `live-common` plus Speculos
- **Realistic**: very close to what a real user does — it even presses device buttons via the Speculos API

#### How It Works

The bot scans accounts for a given seed. For each account it randomly selects a possible transaction — a **mutation** — to perform against one of its sibling accounts. This exercises the full transaction flow: scan accounts → build transaction → sign on device → broadcast to the network → verify the on-chain outcome.

A **mutation** is a possible action to perform on an account. It includes:

- The **transaction expression** (what to do)
- The **device action** (how to interact with Speculos — which buttons to press, what text to approve)
- An optional **assertion test** (expected outcome after broadcast)

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

#### Where the bot fits vs Playwright/Detox E2E

It is useful to see the three automated stacks side by side:

| Stack | Drives | Asserts on | Runs on | Good for |
|-------|--------|------------|---------|----------|
| Playwright E2E (desktop) | The Electron UI + Speculos | UI state, device screen, final account state | Every PR, nightly | UI regressions, end-to-end flows |
| Detox E2E (mobile) | The native mobile UI + Speculos | UI state, device screen | Nightly, on demand | Mobile UI flows |
| Ledger Live Bot | `live-common` + Speculos directly (no UI) | On-chain state after broadcast | Rotated every 8h on `develop`, weekly costly on `develop`, daily on `main` | Protocol-level regressions, broad currency coverage |

The bot is the only one of the three that ends in a real broadcast, which is why it uses funded seeds and why its scope is intentionally narrow (happy path only). UI regressions are Playwright/Detox's job; the bot catches things like "Bitcoin app 2.8.0 produces invalid signatures against testnet" long before a human would notice from a screenshot.

#### The Seven Seeds

The bot runs against seven dedicated seeds, each with a codename and a specific role in the CI rotation:

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

- Only covers **successful** transactions (no error-path coverage)
- Not all apps are supported in Speculos
- Cannot assert all possible transactions in a single run (it samples mutations probabilistically)
- Only works with one app at a time (no disconnection or dashboard flows)
- Does **not** run on LLD or LLM — cannot detect UI-specific issues
- Cannot perform swaps

> **Security warning:** The bot uses seeds in clear text. Never use a personal seed. Use a dedicated seed with minimal funds (~$10 per coin) for any manual bot run.

### 2.3.11 Patterns and Anti-Patterns

A handful of habits make Speculos-backed tests dramatically more reliable. A handful of others guarantee flake.

#### Patterns to adopt

- **Drive the device through named helpers, not raw button presses.** A helper like `confirmOnDevice()` that dispatches on `SPECULOS_DEVICE` keeps the test narrative readable and the matrix honest.
- **Wait on the device's screen text before pressing.** Poll `GET /events` or use the live-common wrapper until the expected label appears, then press. Never sleep-then-press.
- **Pin app versions in fixtures.** A test that works today on Bitcoin 2.7.0 should still work tomorrow on Bitcoin 2.7.0 — even if 2.8.0 exists.
- **Keep one Speculos app per test whenever possible.** The `relaunch()` path is powerful but adds a container restart, which slows the test and widens the failure surface. Only use it when the test genuinely needs two apps (swaps, app-to-app flows).
- **Attach a device screenshot to the failure.** The Playwright fixture already does this; just do not disable it.
- **Use automation rules for long, deterministic flows.** If you are writing a for-loop of button presses, you are probably reinventing the automation endpoint.

#### Anti-patterns to avoid

- **Hard-coding `nanoSP` anywhere.** Even in "temporary" debug code, this breaks the device matrix the moment you forget to remove it.
- **Asserting against raw coordinates on touch devices.** Layouts shift between firmware versions. Assert on labels (`pressAndRelease("Confirm")`) and let Speculos resolve the coordinates.
- **Letting the test choose the Speculos port.** The fixture does this; overriding defeats parallelism.
- **Running long sleeps to "let the device settle".** If you need a sleep, you probably need a wait-for-event instead. Sleeps hide real race conditions and reappear as flake in CI.
- **Sharing a single container across tests.** Each test writes state into the emulated device (automation rules, last-screen history). Share nothing — that is what per-test containers buy you.
- **Treating a Speculos-pass as a hardware-pass.** Speculos is a faithful emulator, not a secure element. Hardware-specific certification still needs real devices.

#### Reading a live-common APDU log

When you tail an E2E run with debug logging on, you will see lines like:

```
=> e040000015 058000002c800000008000000000000000 00000000
<= 4104a3...9000
```

Decoded:

- `=>` is a command APDU the app sent to Speculos. `e0` is CLA, `40` is INS (get-public-key on Bitcoin), then P1, P2, Lc, data, Le.
- `<=` is the response. The last two bytes are the status word: `9000` means "OK". Anything else is a failure to investigate — `6a80` for invalid data, `6d00` for instruction not supported, and so on.

You will not normally hand-parse APDUs. But being able to tell "OK vs not OK" from the trailing `9000` is enough to identify whether the device refused a request or Ledger Live misinterpreted the response.

#### A concrete automation ruleset example

The automation endpoint accepts a JSON document. Here is a ruleset that auto-approves a standard Bitcoin send on a Nano SP — useful when you want to isolate a UI-side issue and do not care about the device-side prompts:

```json
{
  "version": 1,
  "rules": [
    { "regexp": "Review.*transaction", "actions": [["button", 2, true], ["button", 2, false]] },
    { "regexp": "Amount",               "actions": [["button", 2, true], ["button", 2, false]] },
    { "regexp": "Address",              "actions": [["button", 2, true], ["button", 2, false]] },
    { "regexp": "Fees",                 "actions": [["button", 2, true], ["button", 2, false]] },
    { "regexp": "Accept and send",      "actions": [["button", 2, true], ["button", 2, false]] }
  ]
}
```

Ship this to `POST /automation` once at test start, and Speculos will fire the matching button sequence whenever any of those texts appears. The test body no longer has to script device presses — it just waits on the UI.

> **Note:** Automation rules bypass the human-in-the-loop guarantee that signing implies an explicit user approval. That is fine for tests, but never copy a ruleset into anything that interacts with a real user's seed.

### 2.3.12 Debugging Speculos Issues

A practical checklist, in the order you should work through it when a Speculos-based test misbehaves.

**"No Speculos device" or container timeout:**

```bash
# Check if Docker is running
docker ps

# Check for lingering containers from failed tests
docker ps -a | grep speculos

# Remove stuck containers
docker rm -f $(docker ps -a -q --filter ancestor=ghcr.io/ledgerhq/speculos:latest)
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

**View the device screen during a test:**

Open `http://localhost:<port>` in your browser while the test is running (replace `<port>` with the Speculos port from the console output or Allure description). You will see the emulated display update in real time. This is the single most useful debugging tool — far more informative than reading logs.

**Audit what the device is doing at a given instant:**

```bash
# What text is currently on the screen?
curl http://localhost:<port>/events

# What does the screen look like right now?
curl http://localhost:<port>/screenshot -o now.png
```

> **Cross-reference:** Broader E2E troubleshooting (Playwright traces, Detox logs, CI artifact download) lives in **Appendix E: Troubleshooting FAQ**.

#### Common failure modes, in one table

| Symptom | Most likely cause | First fix |
|---------|-------------------|-----------|
| `container not ready` after 30s timeout | Stuck container from a previous run | `docker rm -f $(docker ps -aq --filter name=speculos)` |
| `no app found for X on nanoSP` | Stale `coin-apps` checkout | `cd $COINAPPS && git pull` |
| `EADDRINUSE` on port 5000 | Another Speculos still owns the port | `lsof -i :5000`, then kill |
| `SIGNATURE_INVALID` on transaction | Wrong app version pinned vs seed state | Verify `appQuery.appVersion` against coin-apps |
| Test hangs at "waiting for device" | `MOCK=1` still set, app uses mock transport | `export MOCK=0`, relaunch app |
| Screenshots blank / all black | Container started but firmware did not boot | Check Docker logs; may need to repull the image |
| Works locally, fails in CI only | `REMOTE_SPECULOS` / secrets missing | Inspect CI env, confirm `SEED` is injected |
| Screen shows unexpected app | `speculosApp` fixture value stale from previous `relaunch()` | Use `speculos.current.appName` to confirm state |

#### Three scenarios, three fixes

A few concrete examples of the kind of bug reports that land on the QAA team and the five-minute mental path from symptom to resolution.

**Scenario 1 — "My test was green yesterday, now it fails with `no app found for Bitcoin 2.9.0 on nanoSP`."**

Somebody bumped the pinned Bitcoin app version in the test's fixture, but your local `coin-apps` checkout has not seen `2.9.0` yet. `cd $COINAPPS && git pull`, rerun. If the failure persists after pulling, check that the new version actually exists in the upstream repo — the bump might have been wishful.

**Scenario 2 — "The test hangs forever after clicking the Send button. No error, no screenshot."**

Almost always a missing or wrong `MOCK` value. The app is waiting for a device that is not connected, because the transport layer is still pointed at the mock. Verify `MOCK=0`, verify `SPECULOS_API_PORT` is exported for the current worker (or let the fixture manage it by not setting it at all), and verify you are not running two Ledger Live instances against the same port.

**Scenario 3 — "On CI the test fails with `EADDRINUSE`, but locally it is green."**

The CI runner has leftover containers from a previous job. This is a CI-hygiene issue, not a test bug. Open the CI runner logs, confirm that a container with your port is still alive, and either let the next scheduled cleanup step take care of it or trigger one manually. If you can reproduce this locally by running the same test twice in quick succession without `docker rm -f`, you have confirmed the diagnosis.

#### Reading Allure artifacts for a Speculos failure

When a Speculos-based test fails in CI, the Allure report typically carries:

- The **fixture description** — device, app name, app version, Speculos image tag.
- The **Playwright trace** — every action up to the failure, including page HTML.
- A **device screenshot** captured at failure time (the app takes one via `GET /screenshot`).
- The **APDU log** from `live-common`, surfaced as a step attachment.

Your triage reflex should be: open the trace, scroll to the last successful step, look at the device screenshot for that moment, then compare it to the assertion that failed. Most "flaky Speculos" failures turn out to be a race between the app's UI and the device's display — the test asked the next button press too early, or queried `/events` before the new screen rendered.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://github.com/LedgerHQ/speculos">Speculos: GitHub repository</a> — source code, issues, releases</li>
<li><a href="https://speculos.ledger.com/">Speculos: official site</a></li>
<li><a href="https://speculos.ledger.com/user/launch.html">Speculos: launch and usage documentation</a></li>
<li><a href="https://github.com/LedgerHQ/coin-apps">Coin-apps repository</a> — compiled .elf binaries for all devices</li>
<li><a href="https://github.com/LedgerHQ/app-boilerplate">Ledger app boilerplate</a> — reference template for device apps</li>
<li><a href="https://developers.ledger.com/docs/device-interaction/beginner/exchange_data">APDU — how to exchange data with a Ledger device</a> (Ledger developer docs)</li>
<li><a href="https://www.ledger.com/introducing-bolos-blockchain-open-ledger-operating-system">BOLOS / BAGL / NBGL — introducing the Blockchain Open Ledger Operating System</a></li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Speculos is what makes your tests realistic — it runs the real firmware and the real coin apps, and it responds to real APDU commands over a REST API. The Ledger Live framework wraps it so individual tests never touch Docker directly, but when something breaks your mental model should start at the bottom: is Docker running, are there stuck containers, is <code>COINAPPS</code> fresh, is <code>MOCK=0</code>, is the right <code>SPECULOS_DEVICE</code> selected? Above that layer, the Ledger Live Bot performs autonomous blockchain transactions on seven rotated seeds — the most realistic automated coverage we have, and the closest thing to a real user that does not need a human in the loop.
</div>

### 2.3.13 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Chapter 2.3 Quiz</h3>
<p class="quiz-subtitle">9 questions · 80% to pass</p>
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
<button class="quiz-choice" data-value="A">A) Speculos does not support Solana at all</button>
<button class="quiz-choice" data-value="B">B) The <code>COINAPPS</code> path is wrong or coin-apps needs <code>git pull</code></button>
<button class="quiz-choice" data-value="C">C) You must use <code>nanoX</code> for Solana</button>
<button class="quiz-choice" data-value="D">D) The Speculos Docker image is too new</button>
</div>
<p class="quiz-explanation">Either <code>COINAPPS</code> is not set, points to the wrong directory, or the coin-apps repo is outdated and does not have the Solana .elf binary for the requested device/firmware version. Run <code>cd ~/coin-apps && git pull</code>.</p>
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

<div class="quiz-question" data-correct="B">
<p><strong>Q4.</strong> Why does each Speculos container get a random port number?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) For security — random ports are harder to attack</button>
<button class="quiz-choice" data-value="B">B) To enable parallel execution — multiple test workers run simultaneously, each needing its own isolated Speculos instance</button>
<button class="quiz-choice" data-value="C">C) Speculos requires random ports by design</button>
<button class="quiz-choice" data-value="D">D) To avoid conflicts with the Ledger Live app's default port</button>
</div>
<p class="quiz-explanation">With <code>fullyParallel: true</code> and a high worker count, many tests run at the same time. Each gets its own Docker container on a unique port, preventing port conflicts.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> What does <code>MOCK=0</code> do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Enables mock data for faster tests</button>
<button class="quiz-choice" data-value="B">B) Disables all network requests</button>
<button class="quiz-choice" data-value="C">C) Disables mocked device responses so the app talks to a real Speculos emulator instead</button>
<button class="quiz-choice" data-value="D">D) Turns off test assertions</button>
</div>
<p class="quiz-explanation"><code>MOCK=0</code> tells the Ledger Live app to NOT use mock device responses. Instead, it communicates with the actual Speculos emulator via the port specified in <code>SPECULOS_API_PORT</code>. This is required for Speculos-based E2E tests.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q6.</strong> What should you check FIRST when Speculos fails to start?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The Playwright configuration file</button>
<button class="quiz-choice" data-value="B">B) The test source code</button>
<button class="quiz-choice" data-value="C">C) The Allure report</button>
<button class="quiz-choice" data-value="D">D) Docker — is it running? Are there stuck containers from previous test runs?</button>
</div>
<p class="quiz-explanation">The #1 cause of Speculos failures is Docker issues: Docker Desktop not running, stuck containers from interrupted tests, or port conflicts. Always check <code>docker ps -a</code> first.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q7.</strong> What is the <code>SpeculosFixtureHandle</code> and why does it have a <code>relaunch()</code> method?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It is a mutable wrapper around the current device. <code>relaunch()</code> lets a test switch to a different Speculos app mid-test (e.g., for swap tests that need two different currency apps)</button>
<button class="quiz-choice" data-value="B">B) It is a Playwright page object for device interaction</button>
<button class="quiz-choice" data-value="C">C) It is a Docker container management utility exposed to test authors</button>
<button class="quiz-choice" data-value="D">D) It is used only for retry logic when Speculos crashes</button>
</div>
<p class="quiz-explanation">The handle wraps the current device with a <code>current</code> getter and a <code>relaunch()</code> method. <code>relaunch()</code> stops the current container and starts a new one with a different app — essential for tests that need multiple device apps (like swaps).</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q8.</strong> In the Ledger Live Bot, what is a "mutation"?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A change to the Speculos Docker image configuration</button>
<button class="quiz-choice" data-value="B">B) A modification to the source code of a coin integration</button>
<button class="quiz-choice" data-value="C">C) A React state update in the Ledger Live UI</button>
<button class="quiz-choice" data-value="D">D) A possible transaction action to perform on an account (e.g., "send max", "move 50%") that mutates the on-chain state</button>
</div>
<p class="quiz-explanation">In the bot's vocabulary, a mutation is a transaction scenario that changes (mutates) account state on the blockchain. Each mutation includes the transaction definition, device action (how to interact with Speculos), and optional assertions.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q9.</strong> Which bot seed runs daily non-regression on <code>develop</code>?</p>
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

### 2.4.1 What Are Feature Flags?

Feature flags let you enable/disable features remotely without deploying new code. They are used for:
- **Gradual rollouts** (enable for 10% of users, then 50%, then 100%)
- **A/B testing** (test variant A vs B)
- **Kill switches** (instantly disable a broken feature)
- **Development** (hide WIP features behind a flag)

### 2.4.2 Firebase Remote Config Environments

Ledger Live uses four Firebase environments:

| Environment | Purpose |
|---|---|
| **Ledger Wallet** | Production flags for main app features |
| **Swap** | Flags for swap/exchange functionality |
| **Earn** | Flags for staking/earning features |
| **Buy Sell** | Flags for fiat on/off ramp |

### 2.4.3 Feature Flags in E2E Tests

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

### 2.4.4 The Wallet 4.0 Toggle

```bash
export E2E_ENABLE_WALLET40=1
```

This enables the Wallet 4.0 UI, which significantly changes navigation and layout. Some tests need this flag, others expect the classic UI. Always be aware of which UI version your test targets.

### 2.4.5 Feature Flag Anti-Patterns

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

### 2.4.6 Monitoring Feature Flag Impact

The QA team monitors feature flag impact through the Slack channel **#qa-b2c-releases-bugs-tracking**. When a bug is suspected to be flag-related:

1. Check the flag state in the relevant Firebase environment
2. Try to reproduce with the flag explicitly toggled in both states
3. Report whether the bug is flag-dependent in the Jira ticket

### 2.4.7 Key Takeaways for Feature Flag Hygiene

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

### 2.4.8 Quiz

<!-- ── Chapter 2.4 Quiz ── -->

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


## Allure Reporting & Xray

<div class="chapter-intro">
Running tests without reading reports is like writing code without running it. <strong>Allure</strong> is the test reporting framework used by every Ledger Live E2E and unit pipeline. It turns raw test output into interactive HTML reports with hierarchical step traces, screenshots, device captures, videos, and environment metadata. <strong>Xray</strong> is the Jira plugin that stores our manual test-case catalogue on the <code>B2CQA</code> project; a small custom reporter wires the Playwright and Detox results back into Xray so every execution updates the corresponding <code>B2CQA-*</code> ticket. This chapter teaches Allure from zero, then walks through the exact Ledger Live wiring — per-platform reporter config, the <code>$TmsLink()</code> binding to B2CQA, the artifact pipeline, and the ticket transition that happens when a test lands.
</div>

### 2.5.1 What Is Allure?

Allure Report (by Qameta Software) is an open-source test reporting framework. It consumes a directory of JSON/attachment files emitted by any supported test runner and produces a static HTML site that engineers, QA, and PMs can all read.

At a minimum a report shows:

- Test outcomes grouped by status: **passed**, **failed**, **broken**, **skipped**.
- A collapsible **step tree** for each test (populated by `@step` decorators, see §2.5.4).
- **Attachments** per test: screenshots, videos, Playwright traces, Speculos device screens, console logs, network logs.
- **Metadata**: severity, feature, story, owner, parent suite, TMS links, bug links, environment.
- **Trends**: pass/fail history across the last N runs (when CI keeps a history folder).

Two things Allure is *not*:

- Not a runner. Allure does not execute tests; it consumes results.
- Not a TMS. Allure does not store test-case definitions or track manual execution — that is Xray's job on `B2CQA`.

<div style="border-left:4px solid #3498db; padding:0.5em 1em; background:#f6fafd; margin:1em 0;">
<strong>Mental model.</strong> Your runner (Playwright, Detox, Jest, Vitest) writes raw result files into <code>allure-results/</code>. The <code>allure</code> CLI reads that folder and renders <code>allure-report/</code> — a static site you can open locally or upload to the Ledger Allure server. Xray sync is a <em>separate</em> output file produced by a custom reporter in the same run.
</div>

### 2.5.2 The Reporting Pipeline

```
Your code                     Runner hook                    Artefact                         Consumer
---------                     -----------                    --------                         --------
@step("...") decorator    ->  test.step()               ->   allure-results/*-result.json  -> Allure HTML step tree
addTmsLink([ids])         ->  allure.tms(id)            ->   links[] in result JSON        -> Clickable Jira link
addBugLink([ids])         ->  allure.issue(id)          ->   links[] in result JSON        -> Clickable Jira bug
captureArtifacts()        ->  testInfo.attach(...)      ->   allure-results/*-attachment   -> Embedded png/webm/json
test.use({ teamOwner })   ->  allure.owner/feature(...) ->   labels[] in result JSON       -> Team filter
customJsonReporter        ->  onTestEnd / onExit        ->   xray-report.json              -> Xray REST import
```

One Playwright/Detox run thus fans out into two output channels:

1. The `allure-results/` folder -> `allure generate` -> `allure-report/` -> uploaded to `https://ledger-live.allure.green.ledgerlabs.net`.
2. A single `xray-report.json` file -> Xray REST v2 import -> status written on every `B2CQA-*` ticket listed in the file.

### 2.5.3 Viewing Reports Locally

Desktop (Playwright + Electron):

```bash
pnpm e2e:desktop allure:generate   # reads apps/ledger-live-desktop/allure-results
pnpm e2e:desktop allure:open       # serves apps/ledger-live-desktop/allure-report
pnpm e2e:desktop allure            # convenience: generate + open in one shot
```

Mobile (Detox + Jest), from `apps/ledger-live-mobile/e2e`:

```bash
pnpm allure                        # same generate+open, but on e2e/allure-results
```

Generic fallback — useful when debugging a third-party package:

```bash
npx allure serve ./allure-results  # one-shot local web server
```

### 2.5.4 Reports in CI

Every `e2e-*` GitHub Actions job uploads its report to:

```
https://ledger-live.allure.green.ledgerlabs.net
```

The final URL is printed in the job summary — look for lines starting with **"Desktop Allure report URL"** or **"Mobile Allure report URL"**. The upload also seeds the `history/` folder so the next run shows trends (pass-rate graph, previously-failed tests).

History retention is currently 20 runs per branch; beyond that Allure drops the oldest. Do not rely on Allure for long-term analytics — use the Xray test execution records on Jira for historical reporting (§2.5.10).

### 2.5.5 How `@step` Creates the Step Tree

The step tree is the single most useful part of an Allure report. It is produced by the `@step` decorator applied on every page-object method in both desktop and mobile codebases.

```typescript
// e2e/desktop/tests/models/portfolioPage.ts (shape)
@step("Click Send button")
async clickSend() {
  await this.sendButton.click();
}

@step("Navigate to token $0")
async navigateToToken(name: string) {
  await this.page.getByText(name).click();
}
```

The execution chain on a single call:

1. The decorator wraps the method body in `test.step("Click Send button", () => body())`.
2. Playwright (or Detox) records the step start timestamp and label.
3. The original method body runs.
4. The runner records the end timestamp and outcome.
5. `allure-playwright` / `allure-jest` writes the step into the test's result JSON.
6. `allure generate` renders it as a collapsible row in the report.

The `$0`, `$1` placeholders in the label are interpolated from the call arguments, which is how you get human-readable labels like `Navigate to token "USDT"` instead of the bare method name.

Rendered output:

```
[Bitcoin] Add account (12.3s) PASSED
  Click add account button (0.8s)
  Select asset by ticker (1.2s)
  Select network (0.5s)
  Select first account (3.1s)
  Click close button (0.3s)
  Wait for balance to be visible (2.4s)
  Expect accounts persisted in app.json (4.0s)
```

<div style="border-left:4px solid #e67e22; padding:0.5em 1em; background:#fdf6ef; margin:1em 0;">
<strong>Cross-reference.</strong> The implementation of the <code>@step</code> decorator (TypeScript 5 stage-3 decorators, <code>Symbol.metadata</code>, closure capture of test context) is covered in depth in Part 3 §3.4 "Playwright Advanced — Fixtures, POM &amp; TypeScript Decorators". This chapter assumes the decorator exists and focuses on what it produces, not how it is built.
</div>

### 2.5.6 Artifact Capture on Failure

When a test fails, `captureArtifacts()` (shared helper in `e2e/<platform>/tests/utils/`) collects every piece of forensic evidence an engineer might need. The function is called from the shared `afterEach` fixture with access to `testInfo`, the Playwright `page`, the Electron app, and the Speculos controller.

```typescript
export async function captureArtifacts(
  page,
  testInfo,
  electronApp,
  takeSpeculosScreenshot,
  webviewCollector,
) {
  // 1. Desktop screenshot
  const screenshot = await page.screenshot();
  await testInfo.attach("Screenshot", { body: screenshot, contentType: "image/png" });

  // 2. Speculos device screenshots (what the emulated Nano S/X/SP screen shows)
  if (takeSpeculosScreenshot) {
    await attachSpeculosScreenshots(testInfo); // single PNG or HTML gallery
  }

  // 3. Test execution logs (last retry only to keep the report compact)
  if (isLastRetry(testInfo)) {
    await page.evaluate(filePath => window.saveLogs(filePath), filePath);
    await testInfo.attach("Test logs", { path: filePath, contentType: "application/json" });
    await attachIfExists(testInfo, "Network failures", "network.log", "text/plain");
  }

  // 4. Webview logs (Swap, Earn, Buy all use embedded webviews)
  if (webviewCollector) {
    await testInfo.attach("Webview Console Logs", {
      body: webviewCollector.getFormattedConsoleLogs(),
      contentType: "text/plain",
    });
    await testInfo.attach("Webview Network Logs", {
      body: webviewCollector.getFormattedNetworkLogs(),
      contentType: "text/plain",
    });
  }

  // 5. Video recording (last retry only — large files)
  const video = page.video();
  if (video) {
    await electronApp.close(); // flush video buffer
    const videoData = await readFileAsync(await video.path());
    await testInfo.attach("Test Video", { body: videoData, contentType: "video/webm" });
  }
}
```

Key design choices:

- **Last retry only** for video and log dumps. Earlier retries that fail will be re-attempted; keeping their videos would multiply artefact size for no added value.
- `electronApp.close()` before reading the video file — Playwright only flushes the webm container when the browser context is closed.
- Speculos screenshots come from the emulator's HTTP screenshot endpoint, not from Playwright. See Part 2 §2.4 Speculos for details.

### 2.5.7 Per-Platform Reporter Configuration

The reporter configuration lives in two files, one per platform. These are canonical — if you change the reporter list, change it here.

**Desktop — `apps/ledger-live-desktop/playwright.config.ts`:**

```typescript
import { defineConfig } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/specs",
  reporter: [
    ["list"],                                          // terminal output
    ["allure-playwright", {
      detail: true,
      outputFolder: "allure-results",
      links: {
        // Transforms the plain id "B2CQA-817" into a full Jira URL in the report
        tms: { urlTemplate: "https://ledgerhq.atlassian.net/browse/%s" },
        issue: { urlTemplate: "https://ledgerhq.atlassian.net/browse/%s" },
      },
    }],
    ["./tests/utils/customJsonReporter.ts"],           // writes xray-report.json
    ["html", { open: "never", outputFolder: "playwright-report" }],
  ],
  use: {
    video: "retain-on-failure",
    trace: "retain-on-failure",
    screenshot: "only-on-failure",
  },
});
```

**Mobile — `apps/ledger-live-mobile/e2e/jest.config.js` + `detox.config.js`:**

```javascript
// jest.config.js
module.exports = {
  rootDir: "..",
  testMatch: ["<rootDir>/e2e/specs/**/*.spec.ts"],
  reporters: [
    "default",
    ["jest-allure2-reporter", {
      resultsDir: "e2e/allure-results",
      links: {
        tms: "https://ledgerhq.atlassian.net/browse/{}",
        issue: "https://ledgerhq.atlassian.net/browse/{}",
      },
    }],
    ["<rootDir>/e2e/utils/customJsonReporter.js", {}],
  ],
  // ...
};
```

Notes on the two packages:

- Desktop uses [`allure-playwright`](https://allurereport.org/docs/playwright/) — direct integration with Playwright's reporter interface.
- Mobile uses [`jest-allure2-reporter`](https://github.com/zaqqaz/jest-allure2-reporter) — the community reporter that supports Jest + Detox and mirrors the Allure 2 metadata API (`tms`, `issue`, `owner`, `severity`, `feature`, `story`, attachments).
- Both write into their own `allure-results/` folders. CI uploads them as two separate reports.
- The `urlTemplate` (desktop) and `{}` placeholder (mobile) are what turns a raw id like `B2CQA-2499` into the Jira URL inside the HTML. If this template is missing, the link renders as plain text.

### 2.5.8 TMS Links — Binding a Test to a B2CQA Ticket

Every automated test that implements a manual B2CQA test case must declare the binding. This is what makes Xray sync possible and is what a QA engineer looks for when deciding whether to move the `B2CQA-*` ticket from **Manual Test** to **Automated** (see §2.5.10).

```typescript
// Desktop — apps/ledger-live-desktop/tests/specs/wallet/send.spec.ts (shape)
test(
  "Send 0.0001 BTC to external address",
  {
    annotation: { type: "TMS", description: "B2CQA-2499, B2CQA-2644" },
  },
  async ({ app }) => {
    await addTmsLink(getDescription(test.info().annotations, "TMS").split(", "));
    await allure.severity("critical");
    await allure.feature("Send");
    await allure.story("External address");
    // ...test body
  },
);
```

What `$TmsLink()` actually is: it is the **Xray-side JQL and annotation name** for the same binding. In the Ledger monorepo you never call a function literally named `$TmsLink` from TypeScript — Xray parses the list of TMS ids collected in `allure-results` (and in `xray-report.json`) and treats each id as if the test source file had declared `$TmsLink("B2CQA-2499")`. The result is the same: a bi-directional link between the automation spec and the ticket.

**Valid TMS targets — R4 ground truth:**

| Project | Xray issue type id | What lives there |
|---------|--------------------|------------------|
| `B2CQA` | **10756** — Test  | Manual test cases. This is the canonical TMS target. |
| `B2CQA` | **10759** — Test Plan | Grouping of tests for a release or non-reg campaign. |
| `B2CQA` | **10760** — Test Execution | One concrete run (e.g. `Nano Gen5 1.1.0-rc5 non-reg`). |
| `LIVE`  | 10510 — Test      | Legacy product-side Xray tests; do not create new ones here. |

New automations must target `B2CQA-*` Test tickets only. `LIVE-*` is for product bugs and stories; `QAA-*` is for the automation engineering backlog (the ticket that tracks "automate this B2CQA case"). See Part 2 §2.9 Jira & Xray for the full project matrix.

**Bug links** work exactly the same way but use `allure.issue()`:

```typescript
await addBugLink(["LIVE-12345"]); // known open defect blocking this flow
```

This surfaces the ticket next to the test in the Allure report and makes it obvious when a test is failing because of a tracked bug, not a new regression.

### 2.5.9 Team Ownership

```typescript
test.use({ teamOwner: Team.WALLET_XP });

// Implementation — e2e/<platform>/tests/utils/teamOwner.ts
export async function addTeamOwner(team: Team) {
  await allure.owner(team.toString());        // owner in the report header
  await allure.parentSuite(team.toString());  // top-level grouping in Suites view
  await allure.feature(team.toString());      // Features view grouping
}
```

Supported teams (2026-04): `WALLET_XP`, `QAA`, `SWAP`, `EARN`, `MARKET`, `LIVE_DEVICE`, `LIVE_CORE`, `GROWTH`, `REWARDS`. New teams are added to `e2e/<platform>/tests/utils/teams.ts` and should mirror the Jira `team` custom field on `LIVE-*` tickets.

Filtering by team is how each squad finds only its own tests in a shared nightly report. Without this tag, a 1,200-test report is unusable.

### 2.5.10 The Xray Custom Reporter

`customJsonReporter.ts` is the bridge between Playwright/Detox and Jira. It implements the runner's `Reporter` interface and writes a single `xray-report.json` file at the end of the run.

**File**: `apps/ledger-live-desktop/tests/utils/customJsonReporter.ts` (mirror at `apps/ledger-live-mobile/e2e/utils/customJsonReporter.js`).

```typescript
import { Reporter, TestCase, TestResult } from "@playwright/test/reporter";
import { writeFileSync } from "node:fs";

type XrayResult = { testKey: string; status: "PASSED" | "FAILED" };

class JsonReporter implements Reporter {
  private results: XrayResult[] = [];

  onTestEnd(test: TestCase, result: TestResult) {
    const tms = test.annotations.find(a => a.type === "TMS")?.description ?? "";
    const tmsIds = tms.split(",").map(s => s.trim()).filter(Boolean);

    for (const id of tmsIds) {
      this.results.push({
        testKey: id,
        status: result.status === "passed" ? "PASSED" : "FAILED",
      });
    }
  }

  onExit() {
    writeFileSync(
      "xray-report.json",
      JSON.stringify({
        testExecutionKey: process.env.TEST_EXECUTION, // e.g. "B2CQA-5449"
        tests: this.results,
      }),
    );
  }
}

export default JsonReporter;
```

The emitted JSON:

```json
{
  "testExecutionKey": "B2CQA-5449",
  "tests": [
    { "testKey": "B2CQA-817",  "status": "PASSED" },
    { "testKey": "B2CQA-804",  "status": "PASSED" },
    { "testKey": "B2CQA-2574", "status": "FAILED" }
  ]
}
```

CI then calls the Xray REST v2 import endpoint:

```
POST https://xray.cloud.getxray.app/api/v2/import/execution
Content-Type: application/json
Authorization: Bearer $XRAY_TOKEN

<xray-report.json body>
```

Effects on Jira:

- Every `testKey` listed updates its Xray execution status on the B2CQA Test ticket.
- If `testExecutionKey` is provided, the statuses are attached to that existing Test Execution; otherwise Xray creates a new Execution and links each Test to it.
- The B2CQA Test ticket's own workflow status (`To Do` / `In Test Review` / `Manual Test` / `Automated` / `Obsolete`) is **not** changed by an execution — only the execution status changes.

**The one manual step that remains.** When the automation spec lands on `develop` and the CI run is green, the QA Automation owner manually moves the B2CQA Test ticket from **Manual Test** to **Automated** (R4 §2.2). That transition is what tells the QA team "this test case is now covered by the nightly run and no longer needs manual execution". Forgetting this step is the most common source of manual/automated coverage drift.

### 2.5.11 Failures That Are Product Bugs — Filing on LIVE

When a nightly failure reveals a real product defect, the on-call QA Automation engineer files a **LIVE bug** (not a B2CQA bug — B2CQA bugs are for test-infrastructure defects only):

1. Issue type: **Bug** on the **LIVE** project.
2. Add label `qaa` — this is the label LL devs filter on to see "something the automation caught".
3. If the build is release-critical: also add `deploymentPROD` and `blocker` (see R4 §5). The release manager uses these two labels to freeze a hotfix branch.
4. Link the failing B2CQA Test ticket with a **"blocks"** or **"tests"** link type, and paste the Allure report URL in the description.
5. In the next green run, once the fix is merged, add `addBugLink(["LIVE-XXXX"])` inside the test body so the Allure report surfaces the bug next to the test until the LIVE ticket is **Released**.

<div style="border-left:4px solid #c0392b; padding:0.5em 1em; background:#fdf2f1; margin:1em 0;">
<strong>Do not create "LLM", "LLD", or "PTX" tickets.</strong> These are <em>labels</em> on LIVE tickets, not projects. <code>LLM</code> = mobile-only, <code>LLD</code> = desktop-only, <code>PTX</code> = Protect/Recover (filed under LIVE with the <code>Recover</code> / <code>Apex</code> labels). A Jira search for <code>project = LLM</code> will return zero rows.
</div>

### 2.5.12 Reading a Report Effectively

When you open an Allure report, use the five top-level views in this order:

| View | What it shows | When to use it |
|------|---------------|----------------|
| **Overview** | Pass/fail summary, duration, severity counts | 10-second health check |
| **Suites** | Tests grouped by describe block | Find a specific test by name |
| **Graphs** | Status / severity / duration charts | Spot a systemic regression |
| **Timeline** | When each test ran in parallel workers | Diagnose slow / serialized tests |
| **Categories** | Failures bucketed by error type | Tell product defects from test defects |

For a **failed test**, drill in this order. Each step narrows the search space faster than the next; go deeper only when the current layer is inconclusive.

1. **Step trace** — which `@step` failed, and how long did it take? Length anomalies often point to timing / auto-wait issues.
2. **Desktop screenshot** — what did the app UI show at the moment of failure?
3. **Speculos screenshot** — what did the emulated device screen show? Critical when a signing / approval step hangs.
4. **Video** — full execution, only present on last retry. Use for complex multi-click flows where the screenshot is ambiguous.
5. **Playwright trace** — opens in `trace.playwright.dev` with DOM snapshots per action. Heavyweight; use when screenshots and video disagree.
6. **Network failures** attachment — any 4xx/5xx the app made during the test? Common cause of flaky Swap/Earn failures.
7. **Webview Console Logs** — JS errors inside the embedded webview (Swap/Earn/Buy all run in webviews).
8. **TMS link** — jump to the B2CQA ticket to re-read the expected behaviour the test is encoding.

### 2.5.13 Additional Allure Annotations

Everything in the Allure 2 metadata API is available. The annotations most used in Ledger Live:

```typescript
import { allure } from "allure-playwright"; // desktop
// or: import { allure } from "jest-allure2-reporter/api"; // mobile

test("ERC20 token with 0 balance is hidden", async ({ app }) => {
  await allure.severity("critical");            // blocker / critical / normal / minor / trivial
  await allure.feature("Accounts");              // groups in Features view
  await allure.story("Hide empty token accounts");
  await allure.owner("qaa-team");
  await allure.tag("erc20");
  await allure.tag("regression");
  await allure.parameter("coin", "USDT");        // shown in test params table
  await allure.description("Verifies the 'Hide empty tokens' toggle...");
  await allure.link("Spec", "https://ledger.atlassian.net/wiki/...");
  // ...
});
```

Severity guidance:

- `blocker` — test covers a path whose break would block a release. Pair with `deploymentPROD` if the underlying feature is release-gated.
- `critical` — main user flow (send, receive, swap, add account).
- `normal` — secondary flow or less-used asset.
- `minor` — UI polish, localisation, edge case.
- `trivial` — smoke / meta check.

### 2.5.14 Environment Metadata

Allure surfaces a "Environment" panel on the Overview page — useful for knowing which build and which backend a run was executed against. It is populated from a flat file written by CI before `allure generate`.

**File**: `allure-results/environment.properties` (plain Java-properties format, one `key=value` per line).

```properties
ledger-live-desktop.version=2.103.0-nightly.20260422
ledger-live-mobile.version=4.1.0-nightly.20260422
nightly.commit=7f3a9b2
speculos.firmware=nano-sp-1.1.1
node.env=staging
test.execution=B2CQA-5449
runner=github-actions
runner.os=ubuntu-22.04
```

The CI step that writes this file in the Desktop E2E workflow:

```yaml
- name: Write Allure environment
  run: |
    mkdir -p apps/ledger-live-desktop/allure-results
    cat > apps/ledger-live-desktop/allure-results/environment.properties <<EOF
    ledger-live-desktop.version=${{ steps.build.outputs.version }}
    nightly.commit=${{ github.sha }}
    node.env=${{ inputs.environment }}
    test.execution=${{ inputs.test_execution_key }}
    runner=github-actions
    runner.os=${{ runner.os }}
    EOF
```

Locally you can drop the same file into `allure-results/` by hand if you want the report to show which branch / commit you were on.

### 2.5.15 History, Trends, and Categories

**History.** Allure persists a history folder across runs. To enable trends locally:

```bash
# First run
pnpm e2e:desktop allure:generate          # creates allure-report/
cp -r allure-report/history allure-results/history   # seed for next run

# Second run
pnpm e2e:desktop allure:generate          # trend graph now populated
```

CI does the same thing using the GitHub Actions `actions/cache` action, keyed on the branch name.

**Categories.** A `categories.json` file at the root of `allure-results/` tells Allure how to bucket failures on the Categories view. The default at Ledger Live is:

```json
[
  {
    "name": "Product defects",
    "matchedStatuses": ["failed"],
    "messageRegex": ".*Expect(ed)?.*|.*AssertionError.*"
  },
  {
    "name": "Infrastructure / flaky",
    "matchedStatuses": ["broken"],
    "messageRegex": ".*Timeout.*|.*ECONNREFUSED.*|.*ETIMEDOUT.*"
  },
  {
    "name": "Speculos crashes",
    "matchedStatuses": ["broken", "failed"],
    "messageRegex": ".*speculos.*|.*emulator.*"
  },
  {
    "name": "Webview / third-party",
    "matchedStatuses": ["failed"],
    "messageRegex": ".*webview.*|.*Swap.*provider.*"
  }
]
```

The difference between **failed** and **broken** matters:

- **failed** — an assertion returned false. A test wrote `expect(X).toBe(Y)` and `X !== Y`. Usually a product defect.
- **broken** — the test threw before it could assert (timeout, network error, TypeError). Usually an infrastructure problem or a test bug.

Use the Categories view to answer "did today's red run fail because of the product or because of the pipeline?" in five seconds instead of opening twenty tests.

### 2.5.16 A Worked Local Run — Desktop

The fastest way to internalise the pipeline is to run it once end-to-end on your own machine.

```bash
# 1. Make sure the desktop app is built (cached after first run)
pnpm desktop build

# 2. Run a single spec — no retries, visible output
pnpm e2e:desktop test tests/specs/wallet/send.spec.ts --headed

# 3. Inspect what the runner wrote
ls apps/ledger-live-desktop/allure-results
# -> 3f8e...-result.json   <- one per test
# -> 3f8e...-attachment.png
# -> xray-report.json        <- custom reporter output

cat apps/ledger-live-desktop/xray-report.json
# { "testExecutionKey": undefined,
#   "tests": [ { "testKey": "B2CQA-2499", "status": "PASSED" } ] }

# 4. Render the HTML and open it
pnpm e2e:desktop allure:generate
pnpm e2e:desktop allure:open
# Browser opens allure-report/index.html
```

In the browser, click the failing/passing test, expand the step tree, then click Attachments. You should see the screenshot, the video (if last retry), the Speculos gallery, and the TMS link in the sidebar.

To force a failure and see the artefact pipeline in action:

```bash
# Temporarily break an assertion in the spec, then re-run
pnpm e2e:desktop test tests/specs/wallet/send.spec.ts --retries=0
```

Observe in the report:

- `status=failed` (not broken) because the assertion fired.
- Screenshot, Speculos screenshot, and test logs attached.
- The step that failed is highlighted red; its parent `@step` contains the stack trace.

This five-minute exercise is mandatory onboarding for anyone touching E2E tests.

### 2.5.17 Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Report opens but has no steps | `@step` decorator not applied to page-object methods, or TypeScript decorators disabled | Check `tsconfig.json` has `experimentalDecorators: true` or stage-3 config; confirm the class is the one imported by the test. |
| TMS link renders as plain text | `links.tms.urlTemplate` missing from reporter config | Add `urlTemplate` block in `playwright.config.ts` (desktop) or `links.tms` string in Jest reporter config (mobile). |
| Video attachment is missing | `electronApp.close()` not called before reading the video path | `captureArtifacts()` must close the Electron app to flush the webm file before attaching. |
| Xray import returns 400 | `testKey` points to a `LIVE-*` or `QAA-*` id instead of `B2CQA-*`, or Test id does not exist | Verify every id in `xray-report.json` is a B2CQA Test (issue type 10756). |
| Two runs overwrite each other's trend | CI forgot to copy the previous `history/` folder into `allure-results/` before generate | Job must `cp -r previous-report/history allure-results/history` before `allure generate`. |
| Allure HTML is empty in CI artefact | CI uploaded `allure-results/` (raw JSON) instead of `allure-report/` (rendered HTML) | Run `allure generate` before the upload step. |

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://allurereport.org/docs/">Allure Report — official documentation</a> (Qameta Software)</li>
<li><a href="https://docs.qameta.io/allure/">Allure 2 — legacy reference</a> (older API, still maps 1:1 to the current reporter output)</li>
<li><a href="https://allurereport.org/docs/playwright/">Allure Playwright integration guide</a></li>
<li><a href="https://github.com/zaqqaz/jest-allure2-reporter">jest-allure2-reporter on GitHub</a> — the mobile/Detox reporter package</li>
<li><a href="https://marketplace.atlassian.com/apps/1211769/xray-test-management-for-jira">Xray on the Atlassian Marketplace</a></li>
<li><a href="https://docs.getxray.app/display/XRAYCLOUD/Import+Execution+Results+-+REST+v2">Xray — Import Execution Results REST v2</a></li>
<li><a href="https://playwright.dev/docs/test-reporters">Playwright — Custom Reporters API</a></li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway.</strong> Allure turns <code>@step</code> decorators and <code>testInfo.attach()</code> calls into rich HTML reports; the custom Xray reporter turns the same run's annotations into a JSON file that updates every <code>B2CQA-*</code> Test ticket in Jira. The report is your forensic toolkit when a test fails — step trace, screenshot, device screenshot, video, logs, TMS link, in that order. The Xray sync is what ties the automation back to the QA team's single source of truth. Fluency with both is the difference between debugging in minutes and debugging in hours.
</div>

### 2.5.18 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Chapter 2.5 Quiz</h3>
<p class="quiz-subtitle">9 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> What does a TMS annotation like <code>B2CQA-817</code> refer to?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A Git branch name</button>
<button class="quiz-choice" data-value="B">B) A Jira/Xray Test case ticket on the B2CQA project</button>
<button class="quiz-choice" data-value="C">C) A Firebase feature flag</button>
<button class="quiz-choice" data-value="D">D) A Speculos device identifier</button>
</div>
<p class="quiz-explanation">TMS = Test Management System. <code>B2CQA-XXX</code> is a Jira/Xray Test (issue type 10756) on the <code>B2CQA</code> project — Ledger's QA catalogue. Allure renders it as a clickable link via the <code>urlTemplate</code> in <code>playwright.config.ts</code>.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> How do you generate and open an Allure report for desktop E2E tests?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>npx allure serve</code> from the monorepo root</button>
<button class="quiz-choice" data-value="B">B) <code>pnpm desktop allure</code></button>
<button class="quiz-choice" data-value="C">C) <code>pnpm e2e:desktop allure</code></button>
<button class="quiz-choice" data-value="D">D) <code>pnpm test:allure</code></button>
</div>
<p class="quiz-explanation"><code>pnpm e2e:desktop allure</code> is the combined script that runs <code>allure generate</code> on <code>allure-results/</code> then opens the rendered report. Mobile uses <code>pnpm allure</code> from <code>apps/ledger-live-mobile/e2e</code>.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> What creates the collapsible step tree visible in every Allure test?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The <code>@step</code> decorator applied to page-object methods</button>
<button class="quiz-choice" data-value="B">B) Playwright's built-in trace recorder</button>
<button class="quiz-choice" data-value="C">C) Jest's <code>beforeEach</code>/<code>afterEach</code> hooks</button>
<button class="quiz-choice" data-value="D">D) <code>console.log</code> statements in the test body</button>
</div>
<p class="quiz-explanation">The <code>@step</code> decorator wraps each method call in <code>test.step()</code>, which the Allure reporter writes into the result JSON and the HTML renders as a collapsible row with timing.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> A nightly test fails. What is the most efficient order to investigate in Allure?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Video → Screenshot → Step trace → TMS link</button>
<button class="quiz-choice" data-value="B">B) TMS link → Video → Screenshot → Step trace</button>
<button class="quiz-choice" data-value="C">C) Re-run locally, then read the code, then check logs</button>
<button class="quiz-choice" data-value="D">D) Step trace → Desktop screenshot → Speculos screenshot → Video → Network/Webview logs → TMS link</button>
</div>
<p class="quiz-explanation">Start narrow and widen: the step trace tells you <em>where</em> the failure happened, screenshots show <em>what</em> the user/device saw, video reconstructs the flow, logs surface API/JS errors, and the TMS link shows the <em>expected</em> behaviour.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> What does the custom JSON reporter (<code>customJsonReporter.ts</code>) produce, and what consumes it?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It generates the Allure HTML report from <code>allure-results</code></button>
<button class="quiz-choice" data-value="B">B) It writes <code>xray-report.json</code> mapping each TMS id to PASSED/FAILED, which CI posts to Xray's REST v2 import endpoint</button>
<button class="quiz-choice" data-value="C">C) It posts Slack notifications on failures</button>
<button class="quiz-choice" data-value="D">D) It uploads screenshots to S3</button>
</div>
<p class="quiz-explanation">The reporter implements Playwright's <code>Reporter</code> interface. <code>onTestEnd</code> collects TMS ids + status, <code>onExit</code> writes <code>xray-report.json</code>. CI POSTs it to <code>xray.cloud.getxray.app/api/v2/import/execution</code>, which updates every listed <code>B2CQA-*</code> Test's execution status.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> Why are video recordings only attached on the last retry?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Playwright cannot record video during retries</button>
<button class="quiz-choice" data-value="B">B) Videos can only be rendered in CI</button>
<button class="quiz-choice" data-value="C">C) Video files are large; earlier retries will be re-attempted anyway, so their videos add weight without value</button>
<button class="quiz-choice" data-value="D">D) The first retry is always too fast for video</button>
</div>
<p class="quiz-explanation"><code>captureArtifacts()</code> gates video (and large log dumps) on <code>isLastRetry(testInfo)</code>. Only the final attempt's outcome is reported, so only its video is useful.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q7.</strong> Which Jira/Xray combination is the correct canonical TMS target for a new automated test?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A <code>LIVE-*</code> ticket with issue type 10510 (legacy Xray Test)</button>
<button class="quiz-choice" data-value="B">B) A <code>B2CQA-*</code> Test ticket (issue type 10756), optionally attached to a B2CQA Test Plan (10759)</button>
<button class="quiz-choice" data-value="C">C) A <code>QAA-*</code> ticket in the automation backlog</button>
<button class="quiz-choice" data-value="D">D) An <code>LLM-*</code> or <code>LLD-*</code> ticket depending on the platform</button>
</div>
<p class="quiz-explanation">The QA catalogue lives on <code>B2CQA</code>. Test = 10756, Test Plan = 10759, Test Execution = 10760. <code>LIVE-*</code> is for product work, <code>QAA-*</code> tracks the automation task, and <code>LLM</code>/<code>LLD</code> are <em>labels</em> on LIVE, not projects.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q8.</strong> A new automation lands, CI is green. What still needs to happen on the corresponding B2CQA ticket, and who does it?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Nothing — Xray moves the ticket automatically</button>
<button class="quiz-choice" data-value="B">B) The ticket is auto-closed by the CI bot</button>
<button class="quiz-choice" data-value="C">C) The release manager moves it to <strong>Released</strong></button>
<button class="quiz-choice" data-value="D">D) The QA Automation owner manually transitions the B2CQA Test from <strong>Manual Test</strong> to <strong>Automated</strong></button>
</div>
<p class="quiz-explanation">Xray execution status changes are automatic, but the B2CQA Test's <em>workflow</em> status is not. The QA Automation engineer manually moves it from <strong>Manual Test</strong> to <strong>Automated</strong> so the QA team knows this case is now covered by the nightly run.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q9.</strong> A nightly E2E run reveals a real product regression. Where and how should the bug be filed?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A <strong>LIVE Bug</strong> with label <code>qaa</code> (plus <code>deploymentPROD</code> + <code>blocker</code> if release-critical), linked to the failing B2CQA Test</button>
<button class="quiz-choice" data-value="B">B) A B2CQA Bug in the QA catalogue</button>
<button class="quiz-choice" data-value="C">C) A QAA ticket for the automation engineer to fix the test</button>
<button class="quiz-choice" data-value="D">D) A new <code>LLM-*</code> or <code>LLD-*</code> ticket depending on platform</button>
</div>
<p class="quiz-explanation">Product defects live on <code>LIVE</code>. The <code>qaa</code> label is how LL devs filter for "caught by E2E automation". B2CQA bugs are reserved for test-infrastructure issues. <code>LLM</code> / <code>LLD</code> are labels, not projects.</p>
</div>

<div class="quiz-score"></div>
</div>

---

---

## Part 2 Final Assessment

<div class="quiz-container" data-pass-threshold="80">
<h3>Part 2 Final Assessment</h3>
<p class="quiz-subtitle">10 questions across Git, pnpm/Turbo, Speculos, Firebase, Allure · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> Which branch naming prefix should you use for adding a new E2E test?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>bugfix/</code></button>
<button class="quiz-choice" data-value="B">B) <code>feat/</code></button>
<button class="quiz-choice" data-value="C">C) <code>chore/</code></button>
<button class="quiz-choice" data-value="D">D) <code>support/</code></button>
</div>
<p class="quiz-explanation"><code>feat/</code> — new tests are new features. Use <code>test(e2e): ...</code> for the commit type within the Conventional Commits format.</p>
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
<p><strong>Q3.</strong> You need to run a test against a Stax device. Which environment variable tells Speculos which device model to emulate?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>DEVICE_TYPE</code></button>
<button class="quiz-choice" data-value="B">B) <code>LEDGER_DEVICE</code></button>
<button class="quiz-choice" data-value="C">C) <code>SPECULOS_MODEL</code> (or <code>--model</code> flag)</button>
<button class="quiz-choice" data-value="D">D) <code>EMULATOR_TARGET</code></button>
</div>
<p class="quiz-explanation">Speculos selects its device model via the <code>SPECULOS_MODEL</code> env var or the <code>--model</code> CLI flag. Valid values include <code>nanos</code>, <code>nanosp</code>, <code>nanox</code>, <code>stax</code>, <code>flex</code>.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> What does the <code>@Step(...)</code> decorator do in an Allure-instrumented Page Object method?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Wraps the method so Allure records its entry, exit, and screenshots as a nested step in the report</button>
<button class="quiz-choice" data-value="B">B) Forces the method to run synchronously</button>
<button class="quiz-choice" data-value="C">C) Registers the method as a Jest-only hook</button>
<button class="quiz-choice" data-value="D">D) Disables retries for the wrapped call</button>
</div>
<p class="quiz-explanation"><code>@Step</code> is an Allure reporter decorator: it intercepts the call, emits a named step event (with arg interpolation via <code>$0</code>, <code>$1</code>, etc.), and nests sub-steps inside it. Failures are attached to the most-specific step.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> Which Speculos endpoint takes a screenshot?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>POST /screenshot</code></button>
<button class="quiz-choice" data-value="B">B) <code>GET /screenshot</code></button>
<button class="quiz-choice" data-value="C">C) <code>GET /screen</code></button>
<button class="quiz-choice" data-value="D">D) <code>POST /capture</code></button>
</div>
<p class="quiz-explanation"><code>GET /screenshot</code> returns a PNG image of the device screen. GET because you are retrieving data, not modifying state.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q6.</strong> Ledger Live ships four Firebase environments. Which list names them correctly?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Alpha, Beta, RC, Stable</button>
<button class="quiz-choice" data-value="B">B) Dev, Staging, Production, Canary</button>
<button class="quiz-choice" data-value="C">C) Ledger Wallet, Swap, Earn, Buy Sell</button>
<button class="quiz-choice" data-value="D">D) <code>ledger-live-development</code>, <code>ledger-live-testing</code>, <code>ledger-live-staging</code>, <code>ledger-live-production</code></button>
</div>
<p class="quiz-explanation">The four Firebase Remote Config projects are <code>development</code> (dev sandbox), <code>testing</code> (QA testing), <code>staging</code> (prerelease QA), and <code>production</code> (prod). Swap, Earn, etc. are Live Apps with per-env manifests — not Firebase environments.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q7.</strong> Every PR that changes published behavior ships with a file in <code>.changeset/</code>. What does that file declare?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The list of reviewers required by CODEOWNERS</button>
<button class="quiz-choice" data-value="B">B) The CI workflow to run</button>
<button class="quiz-choice" data-value="C">C) Which packages are affected and whether this is a <code>patch</code>, <code>minor</code>, or <code>major</code> bump</button>
<button class="quiz-choice" data-value="D">D) The Jira ticket to close when the PR merges</button>
</div>
<p class="quiz-explanation">Changesets use per-PR markdown files listing affected packages + semver bump type. At release, <code>changeset version</code> aggregates them, bumps versions, and generates CHANGELOGs. No semantic-release involved.</p>
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

<div class="quiz-question" data-correct="B">
<p><strong>Q9.</strong> What is the role of <code>$TmsLink("B2CQA-604")</code> in a spec title?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It creates a new Xray test case at run time</button>
<button class="quiz-choice" data-value="B">B) It binds the automated test to the existing B2CQA test case so Xray can transition it from "Manual" to "Automated" and aggregate runs</button>
<button class="quiz-choice" data-value="C">C) It tags the test as skipped on CI</button>
<button class="quiz-choice" data-value="D">D) It marks the test as a flake-quarantine candidate</button>
</div>
<p class="quiz-explanation"><code>$TmsLink</code> is a Jest / Playwright reporter pattern: the Allure reporter parses the placeholder and emits a Jira/Xray link in the report. A downstream sync job transitions the B2CQA ticket and attaches the run.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q10.</strong> A flaky test sometimes passes and sometimes fails in CI. The failure is always "Element not found" for a button that should appear after a feature flag is enabled. What is the most likely root cause?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The CI runner is too slow</button>
<button class="quiz-choice" data-value="B">B) The Playwright version is outdated</button>
<button class="quiz-choice" data-value="C">C) The test ID was changed in a recent commit</button>
<button class="quiz-choice" data-value="D">D) The test relies on a Firebase flag default instead of explicitly overriding it, and the default has been changed intermittently</button>
</div>
<p class="quiz-explanation">This is the classic "implicit flag dependency" anti-pattern. The test doesn't explicitly set the feature flag, so it depends on the Firebase default. If someone changes the default (even temporarily), the test fails. The fix: always explicitly override every flag your test depends on.</p>
</div>

<div class="quiz-score"></div>
</div>

---

<div class="chapter-outro">

Part 2 complete. You now share the same vocabulary — Git, pnpm/Turbo, Speculos, Firebase, Allure — as every other engineer on the team. Part 3 takes you deep into the Desktop E2E stack: Playwright from zero, Electron, the fixture hub, the codebase catalog, and two real ticket walkthroughs.

</div>
