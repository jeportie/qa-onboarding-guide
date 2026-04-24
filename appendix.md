
# APPENDIX

<div class="chapter-intro">
<strong>Your daily reference shelf.</strong> This appendix consolidates every command, variable, device model, troubleshooting recipe, and term you'll encounter in the Ledger Live QA ecosystem. Bookmark it -- you'll come back here constantly during your first weeks.
</div>

---

## A: Complete Command Reference

### Setup

```bash
mise install                              # Install all tools (Node, pnpm, gitleaks, etc.)
pnpm i                                   # Install all dependencies
pnpm i --filter="package..."             # Install for specific package (+ all its deps)
```

### Desktop E2E Setup

```bash
# Install deps for desktop + CLI + E2E
pnpm i --filter="ledger-live-desktop..." \
       --filter="live-cli..." \
       --filter="ledger-live" \
       --filter="@ledgerhq/dummy-*-app..." \
       --filter="ledger-live-desktop-e2e-tests" \
       --unsafe-perm

# Build dependencies
pnpm build:lld:deps

# Build the CLI (used for test data setup)
pnpm build:cli

# Build the desktop app in testing mode
pnpm desktop build:testing

# Install Playwright browsers
pnpm e2e:desktop test:playwright:setup
```

### Mobile E2E Setup

```bash
# Install deps for mobile + CLI + E2E
pnpm i --filter="live-mobile..." \
       --filter="ledger-live" \
       --filter="live-cli..." \
       --filter="ledger-live-mobile-e2e-tests"

# Build dependencies
pnpm build:llm:deps

# Build the CLI
pnpm build:cli

# === Android (release -- debug builds broken locally) ===
cd e2e/mobile && pnpm build:android
# or from repo root: nx run live-mobile:e2e:build -- --configuration android.emu.release

# === iOS (debug) ===
pnpm mobile pod                           # Install CocoaPods (still from apps/ledger-live-mobile)
cd e2e/mobile && pnpm build:ios:debug
# or from repo root: pnpm --filter ledger-live-mobile-e2e-tests run build:ios:debug
```

### Development

```bash
pnpm dev:lld                              # Start Desktop dev server
pnpm dev:llm:ios                          # Start Mobile iOS dev server
pnpm dev:llm:android                      # Start Mobile Android dev server
```

> **Pro tip** -- For live reload during library development, run `pnpm --filter @ledgerhq/<lib-name> run watch` in a separate terminal. Changes to the library will propagate to the consuming app in real time (from wiki: Assorted Tips).

### Building

```bash
pnpm build:lld                            # Build Desktop (+ deps via Turbo)
pnpm build:lld:deps                       # Build Desktop dependencies only
pnpm build:llm:deps                       # Build Mobile dependencies only
pnpm build:libs                           # Build ALL libraries
pnpm build:cli                            # Build CLI tool
pnpm desktop build:testing                # Build Desktop in testing mode (E2E)
pnpm turbo build --filter=@ledgerhq/<lib> # Build a specific library
```

### Linting & Type-Checking

```bash
pnpm lint                                 # Lint entire monorepo
pnpm lint:fix                             # Lint + auto-fix
pnpm typecheck                            # Type-check everything
pnpm desktop typecheck                    # Type-check desktop only
pnpm mobile typecheck                     # Type-check mobile only
pnpm --filter ledger-live-desktop-e2e-tests typecheck  # Type-check desktop E2E
pnpm --filter ledger-live-desktop-e2e-tests lint        # Lint desktop E2E
pnpm --filter ledger-live-mobile-e2e-tests typecheck   # Type-check mobile E2E
```

### Unit & Integration Tests

```bash
pnpm desktop test:jest                    # All desktop unit/integration tests
pnpm desktop test:jest "filename"         # Desktop test file matching "filename"
pnpm mobile test:jest                     # All mobile unit/integration tests
pnpm mobile test:jest "filename"          # Mobile test file matching "filename"
```

### Desktop E2E Tests

```bash
pnpm e2e:desktop test:playwright                      # Run all desktop E2E tests
pnpm e2e:desktop test:playwright settings.spec.ts     # Run a single spec file
pnpm e2e:desktop test:playwright --grep "Settings" # Filter by test name pattern (alternative)
pnpm e2e:desktop test:playwright --grep "@smoke"   # Run only smoke tests
pnpm e2e:desktop test:playwright --grep "@NanoSP"  # Run only NanoSP-compatible tests
pnpm e2e:desktop allure:generate                      # Generate Allure report from results
pnpm e2e:desktop allure:open                          # Open Allure report in browser
pnpm e2e:desktop allure                               # Generate + open (combined)

# Debug mode (opens Playwright Inspector):
cd e2e/desktop
PWDEBUG=1 npx playwright test --config=playwright.config.ts settings.spec.ts
```

### Mobile E2E Tests (from e2e/mobile/)

```bash
pnpm test:android                         # All Android tests (release build)
pnpm test:android myTest.spec.ts          # Single file (Android)
pnpm test:ios:debug                       # All iOS tests (requires Metro running)
pnpm test:ios:debug myTest.spec.ts        # Single file (iOS)
pnpm allure                               # Generate + open mobile Allure report
pnpm typecheck                            # Type-check (runs scripts/typecheck.js)
pnpm lint                                 # oxlint over the workspace
```

**From the repo root (equivalent forms):**

```bash
# pnpm filter form
pnpm --filter ledger-live-mobile-e2e-tests run build:ios:debug
pnpm --filter ledger-live-mobile-e2e-tests run test:android

# Nx target form (delegates to the mobile app's e2e:build executor)
nx run live-mobile:e2e:build -- --configuration ios.sim.release
nx run live-mobile:e2e:build -- --configuration android.emu.release
```

> **Note**: iOS debug tests require Metro running in a separate terminal: `pnpm mobile start` (from repo root).
> **Note**: ENVFILE paths remain `apps/ledger-live-mobile/.env.mock` and `apps/ledger-live-mobile/.env.mock.prerelease` (see Appendix B) — they live under the app directory, not the E2E workspace.

### Git & Versioning

```bash
pnpm commit                               # Interactive conventional commit helper
pnpm changeset                            # Add a changeset for versioning
```

### Docker (Speculos)

```bash
docker pull ghcr.io/ledgerhq/speculos:latest          # Pull latest Speculos image
docker images | grep speculos                          # Verify image exists locally
docker ps --filter name=speculos                       # List running Speculos containers
docker rm -f $(docker ps -aq --filter name=speculos)   # Kill ALL leftover containers
docker logs <container-id>                             # View container logs
docker run --rm ghcr.io/ledgerhq/speculos:latest --help  # Verify Docker image works

# Manual Speculos launch (for debugging):
docker run --rm -it -p 5000:5000 \
  -v ~/coin-apps:/speculos/apps \
  ghcr.io/ledgerhq/speculos:latest \
  --model nanosp \
  --sdk 1.0 \
  apps/nanoSP/1.1.1/Ethereum/app_1.13.0.elf
```

### Speculos REST API (Manual Testing)

```bash
# Check Speculos is alive
curl http://127.0.0.1:5000/

# Take a screenshot
curl http://127.0.0.1:5000/screenshot -o screenshot.png

# Get current screen events (text displayed)
curl http://127.0.0.1:5000/events

# Press buttons (non-touch devices)
curl -X POST http://127.0.0.1:5000/button/left
curl -X POST http://127.0.0.1:5000/button/right
curl -X POST http://127.0.0.1:5000/button/both    # Confirm

# Touch screen (touch devices: Stax, Flex, Gen5)
curl -X POST http://127.0.0.1:5000/finger -d '{"action":"press-and-release","x":200,"y":400}'

# Send raw APDU command
curl -X POST http://127.0.0.1:5000/apdu -d '{"data":"e003000000"}'

# Kill the Speculos session
curl -X DELETE http://127.0.0.1:5000/
```

### Ledger Live Bot Commands

```bash
# Run the bot (from live-common or CLI)
pnpm build:cli && pnpm run:cli bot --seed "your 24 word seed" -f bitcoin

# Available flags:
#   --seed <mnemonic>       BIP39 mnemonic to use
#   -f, --family <family>   Coin family to test (bitcoin, ethereum, solana, etc.)
#   --device <model>        Device model for Speculos (nanoSP, stax, etc.)
#   --mutations <count>     Max number of mutations per account
```

### Performance Profiling

```bash
# Renderer process profiling (from wiki: Performance Tips)
# 1. Open DevTools in Ledger Live Desktop (Ctrl+Shift+I)
# 2. Go to Performance tab > Record
# 3. Reproduce the slow action
# 4. Stop recording > Analyze flame chart

# Main process profiling
# Start desktop with --inspect flag:
pnpm dev:lld -- --inspect
# Then open chrome://inspect in Chrome to connect to the Node.js debugger
```

---

## B: Environment Variables Reference

### Required for E2E Testing

| Variable | Example | Description |
|---|---|---|
| `MOCK` | `0` | `0` = use real Speculos (Docker). `1` = mocked device (no Docker, uses stub responses). Always `0` for real E2E tests. |
| `SPECULOS_DEVICE` | `nanoSP` | Device model to emulate. Values: `nanoS`, `nanoSP`, `nanoX`, `stax`, `flex`, `nanoGen5` |
| `SPECULOS_IMAGE_TAG` | `ghcr.io/ledgerhq/speculos:latest` | Docker image for Speculos. Use `latest` unless debugging a specific version. |
| `COINAPPS` | `~/coin-apps` | Path to the cloned coin-apps repository. Required for local runs. Auto-set in CI. |

### Optional / Advanced

| Variable | Example | Description |
|---|---|---|
| `SEED` | `abandon abandon ... about` | BIP39 mnemonic (24 words) for deterministic account generation. Stored in CI secrets. Ask your team lead for the local value. |
| `E2E_ENABLE_WALLET40` | `1` | Enable Wallet 4.0 UI during tests. Changes navigation and layout significantly. |
| `E2E_FEATURE_FLAGS_JSON` | `{"myFlag":{"enabled":true}}` | Override feature flags as a JSON string. Merged with defaults. |
| `SPECULOS_API_PORT` | `5000` | REST API port for Speculos. Auto-assigned by the fixture system (you rarely set this manually). |
| `SPECULOS_ADDRESS` | `http://127.0.0.1` | Base URL for Speculos REST API. Auto-set by the fixture system. |
| `DISABLE_TRANSACTION_BROADCAST` | `1` | Prevent transactions from being broadcast to the real blockchain. Always `1` in tests. |
| `ENVFILE` | `apps/ledger-live-mobile/.env.mock` | Detox-time env file selection. Two values for E2E mocks: `.env.mock` (staging Firebase) and `.env.mock.prerelease` (prod Firebase). **File location unchanged** — they still live under `apps/ledger-live-mobile/` even though the E2E workspace moved to `e2e/mobile/`. Resolved in `e2e/mobile/detox.config.js` via `ENV_FILE_MOCK` / `ENV_FILE_MOCK_PRERELEASE` constants. |
| `NODE_OPTIONS` | `--max-old-space-size=14336` | Node.js memory limit. CI uses 14GB. Increase locally if you get OOM errors. |

### CI-Specific

| Variable | Description |
|---|---|
| `GITHUB_TOKEN` | GitHub API token for authenticated Docker pulls from `ghcr.io` and CI operations |
| `ALLURE_SERVER_URL` | URL of the Allure reporting server for uploading results |
| `XRAY_CLIENT_ID` / `XRAY_CLIENT_SECRET` | Credentials for Jira/Xray test result import |

### Development

| Variable | Description |
|---|---|
| `DEBUG` | Enable debug logging. Formats: `speculos:*`, `live-common:*`, `hw-transport:*` |
| `VERBOSE` | Enable verbose output in CLI commands |
| `PLAYWRIGHT_TRACE` | Set to `on` to capture Playwright traces for debugging |

---

## C: Ledger Live Stack

Quick references for the 21 technologies in the Ledger Live stack. Each entry covers what the technology is, its core concepts, how it is used in Ledger Live, and links to documentation. Use these as a reference when you encounter a technology in the codebase.

### TypeScript

TypeScript is a **statically-typed superset of JavaScript** developed by **Microsoft** (Anders Hejlsberg). Every valid JS file is also valid TS. In Ledger Live, TypeScript is used everywhere — desktop, mobile, libraries, and E2E tests.

**Core concepts**: Type annotations, interfaces, type aliases, generics, enums, union/intersection types, type guards, utility types (`Partial<T>`, `Pick<T,K>`, `Omit<T,K>`, `Record<K,V>`).

**In Ledger Live**: All packages use strict TypeScript. The `Account` interface, `CryptoCurrency` type, and `Transaction` type are foundational types defined in `libs/ledger-live-common/`.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://www.typescriptlang.org/docs/">TypeScript Handbook</a> — official documentation by Microsoft</li>
<li><a href="https://www.typescriptlang.org/play">TypeScript Playground</a> — interactive browser-based editor</li>
<li><a href="https://github.com/type-challenges/type-challenges">Type Challenges</a> — interactive type-level programming exercises</li>
<li><a href="https://www.totaltypescript.com/">Total TypeScript</a> — Matt Pocock's advanced TypeScript tutorials</li>
</ul>
</div>

### React

React is a **UI library** for building component-based interfaces, created by **Meta** (Jordan Walke). It uses a virtual DOM for efficient rendering.

**Core concepts**: Components (function), JSX, props, state (`useState`), effects (`useEffect`), context (`useContext`), custom hooks, memoization (`useMemo`, `useCallback`, `React.memo`).

**In Ledger Live Desktop**: The entire UI is React components. State is managed with Redux Toolkit. Navigation uses React Router. Styling uses styled-components.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://react.dev/">React documentation</a> — official docs (new site)</li>
<li><a href="https://react.dev/learn">React Learn</a> — interactive tutorial</li>
<li><a href="https://www.debugbear.com/blog/measuring-react-app-performance">Measuring React Performance</a></li>
</ul>
</div>

### React Native

React Native is a framework for building **native mobile apps** using React, created by **Meta**. It renders to native iOS/Android components, not a WebView.

**Core concepts**: `View`, `Text`, `ScrollView`, `FlatList`, `TouchableOpacity`, `StyleSheet`, platform-specific code (`.ios.ts`/`.android.ts`), native modules, Metro bundler.

**In Ledger Live Mobile**: The entire LLM app. Uses React Navigation for screen transitions, styled-components for styling, and Redux for state.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://reactnative.dev/">React Native documentation</a></li>
<li><a href="https://reactnative.dev/docs/debugging">React Native Debugging</a></li>
<li><a href="https://expo.dev/snacks">Expo Snack</a> — browser-based React Native playground</li>
</ul>
</div>

### Electron

Electron is a framework for building **cross-platform desktop apps** with web technologies, created by **GitHub** (Cheng Zhao). It combines Chromium (renderer) + Node.js (main process).

**Core concepts**: Main process, renderer process, preload scripts, IPC (inter-process communication), `BrowserWindow`, context isolation, `webPreferences`.

**In Ledger Live Desktop**: The entire LLD app. Main process handles native features (file system, USB, tray icon). Renderer process runs the React UI. Preload scripts bridge the two with IPC.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://www.electronjs.org/docs/latest/">Electron documentation</a></li>
<li><a href="https://www.electronjs.org/docs/latest/tutorial/quick-start">Electron Quick Start</a></li>
<li><a href="https://playwright.dev/docs/api/class-electron">Playwright Electron API</a> — how Playwright tests Electron apps</li>
</ul>
</div>

### Redux Toolkit

Redux Toolkit (RTK) is the official **state management** library for Redux, created by **Mark Erikson**. It simplifies Redux with `createSlice`, `createAsyncThunk`, and `configureStore`.

**Core concepts**: Store, slices, reducers, actions, selectors, `createSlice`, `createAsyncThunk`, middleware, RTK Query.

**In Ledger Live**: Global app state — accounts, settings, feature flags, countervalues. Each domain has its own slice in `src/renderer/reducers/` (desktop) or `src/reducers/` (mobile).

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://redux-toolkit.js.org/">Redux Toolkit documentation</a></li>
<li><a href="https://redux.js.org/tutorials/essentials/part-1-overview-concepts">Redux Essentials Tutorial</a></li>
</ul>
</div>

### React Router

React Router is a **routing library** for React web apps, created by **Remix** (Ryan Florence, Michael Jackson).

**Core concepts**: `<BrowserRouter>`, `<Routes>`, `<Route>`, `<Link>`, `useNavigate`, `useParams`, `useLocation`, nested routes, route guards.

**In Ledger Live Desktop**: URL-based navigation between screens (portfolio, accounts, settings, etc.). Deep links use React Router paths.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://reactrouter.com/en/main">React Router documentation</a></li>
</ul>
</div>

### React Navigation

React Navigation is the standard **navigation library** for React Native, created by the **React Navigation team** (Satyajit Sahoo, Michał Osadnik).

**Core concepts**: Stack navigator, tab navigator, drawer navigator, `navigation.navigate()`, `useNavigation`, screen options, deep linking, nested navigators.

**In Ledger Live Mobile**: All screen transitions. Tab bar (Portfolio, Market, Transfer, Discover), stack-based flows (send, receive, swap), and modal sheets.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://reactnavigation.org/">React Navigation documentation</a></li>
<li><a href="https://reactnavigation.org/docs/getting-started">Getting Started guide</a></li>
</ul>
</div>

### i18next

i18next is an **internationalization framework** for JavaScript, created by **Jan Mühlemann**. It provides translation management with interpolation, plurals, and context.

**Core concepts**: Translation keys, namespaces, interpolation (`{{name}}`), plurals, `useTranslation` hook, language detection, fallback languages.

**In Ledger Live**: All user-facing strings are translated. Translation files live in `apps/ledger-live-desktop/src/renderer/i18n/` and `apps/ledger-live-mobile/src/locales/`. The `t()` function is used everywhere.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://www.i18next.com/">i18next documentation</a></li>
<li><a href="https://react.i18next.com/">react-i18next documentation</a></li>
</ul>
</div>

### Styled-Components

Styled-components is a **CSS-in-JS library** that uses tagged template literals to style React components, created by **Max Stoiber** and **Glen Maddern**.

**Core concepts**: `styled.div`, template literals, props-based styling, themes, `ThemeProvider`, `css` helper, extending styles.

**In Ledger Live**: Both LLD and LLM use styled-components for all UI styling. The design system uses a shared theme with colors, fonts, and spacing.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://styled-components.com/docs">Styled-components documentation</a></li>
</ul>
</div>

### Tailwind CSS

Tailwind CSS is a **utility-first CSS framework** created by **Adam Wathan**. It provides pre-built utility classes instead of writing custom CSS.

**Core concepts**: Utility classes (`flex`, `p-4`, `text-lg`, `bg-blue-500`), responsive design (`md:`, `lg:`), dark mode, `@apply`, configuration (`tailwind.config.js`).

**In Ledger Live**: Used in newer parts of the codebase (web tools, some shared components). Coexists with styled-components.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://tailwindcss.com/docs">Tailwind CSS documentation</a></li>
<li><a href="https://play.tailwindcss.com/">Tailwind Play</a> — interactive browser playground</li>
</ul>
</div>

### Playwright

Playwright is a **browser automation framework** for E2E testing, created by **Microsoft** (Andrey Lushnikov, Dmitry Gozman — former Puppeteer team).

**Core concepts**: `test`, `expect`, locators (`getByTestId`, `getByRole`, `getByText`), fixtures, auto-wait, screenshots, video, traces, parallel execution, sharding, Electron support.

**In Ledger Live**: Desktop E2E suite. Custom fixtures extend Playwright with Speculos, Electron lifecycle, page objects, and feature flag management.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://playwright.dev/docs/intro">Playwright documentation</a></li>
<li><a href="https://playwright.dev/docs/codegen">Playwright Codegen</a> — record interactions to generate test code</li>
<li><a href="https://try.playwright.tech/">Try Playwright</a> — interactive browser playground</li>
<li><a href="https://playwright.dev/docs/api/class-electron">Playwright Electron API</a></li>
</ul>
</div>

### Detox

Detox is a **gray-box E2E testing framework** for React Native, created by **Wix Engineering** (Rotem Mizrachi-Meidan). It provides native-level interaction with automatic synchronization.

**Core concepts**: `element(by.id())`, `by.text()`, `.tap()`, `.typeText()`, `.scroll()`, `waitFor().withTimeout()`, `device.launchApp()`, `device.reloadReactNative()`, synchronization.

**In Ledger Live**: Mobile E2E suite. Uses Jest as test runner. Global `app` singleton, WebSocket bridge, helper wrappers.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://wix.github.io/Detox/">Detox documentation</a></li>
<li><a href="https://wix.github.io/Detox/docs/api/matchers">Detox Matchers API</a></li>
<li><a href="https://wix.github.io/Detox/docs/api/actions-on-element">Detox Actions API</a></li>
</ul>
</div>

### Jest

Jest is a **JavaScript testing framework** created by **Meta** (Christoph Nakazawa). It includes test runner, assertion library, mocking, and code coverage.

**Core concepts**: `describe`, `it`/`test`, `expect`, matchers (`.toBe`, `.toEqual`, `.toContain`), `beforeEach`/`afterEach`, mocking (`jest.fn()`, `jest.mock()`, `jest.spyOn()`), snapshots, `--coverage`.

**In Ledger Live**: Unit and integration tests throughout. Mobile E2E uses Jest as the Detox test runner.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://jestjs.io/docs/getting-started">Jest documentation</a></li>
<li><a href="https://jestjs.io/docs/mock-functions">Jest Mock Functions</a></li>
</ul>
</div>

### MSW

MSW (Mock Service Worker) is an **API mocking library** that intercepts HTTP requests at the network level, created by **Artem Zakharchenko**.

**Core concepts**: Request handlers (`http.get`, `http.post`), `setupServer` (Node), `setupWorker` (browser), response resolvers, request matching, runtime request handlers.

**In Ledger Live**: Integration tests mock API calls (countervalues, explorers, feature flags) without changing application code. MSW intercepts at the network level, so the app code doesn't know it's being mocked.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://mswjs.io/docs/">MSW documentation</a></li>
<li><a href="https://mswjs.io/docs/getting-started">MSW Getting Started</a></li>
</ul>
</div>

### Allure

Allure is a **test reporting framework** created by **Qameta Software**. It generates interactive HTML reports from test results.

**Core concepts**: Steps, attachments (screenshots, videos), annotations (severity, feature, story, owner), TMS links, history, categories, environment info.

**In Ledger Live**: Both desktop and mobile E2E suites use Allure. The `@step` decorator creates the step trace. TMS annotations link to Jira/Xray.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://allurereport.org/docs/">Allure Report documentation</a></li>
<li><a href="https://allurereport.org/docs/playwright/">Allure Playwright integration</a></li>
<li><a href="https://allurereport.org/docs/jest/">Allure Jest integration</a></li>
</ul>
</div>

### Testing Library

Testing Library is a family of **testing utilities** that encourage testing from the user's perspective, created by **Kent C. Dodds**.

**Core concepts**: `render`, `screen`, queries (`getByRole`, `getByText`, `getByTestId`), `userEvent`, `waitFor`, query priority (role → label → text → testId).

**In Ledger Live**: Integration tests for React components. Used alongside Jest and MSW. Query priority matches Playwright's locator strategy.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://testing-library.com/docs/">Testing Library documentation</a></li>
<li><a href="https://testing-library.com/docs/react-testing-library/intro">React Testing Library</a></li>
<li><a href="https://testing-playground.com/">Testing Playground</a> — interactive query helper</li>
</ul>
</div>

### pnpm

pnpm is a **fast, disk-space efficient package manager** created by **Zoltan Kochan**. It uses a content-addressable store with symlinks.

**Core concepts**: Content-addressable store, symlinks, `--filter`, workspaces, `.pnpmfile.cjs`, `pnpm-workspace.yaml`, `--frozen-lockfile`, aliases.

**In Ledger Live**: The package manager for the entire monorepo.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://pnpm.io/">pnpm documentation</a></li>
<li><a href="https://pnpm.io/filtering">pnpm filtering</a></li>
<li><a href="https://pnpm.io/workspaces">pnpm workspaces</a></li>
</ul>
</div>

### Turborepo

Turborepo is a **build system for monorepos** created by **Jared Palmer** (now maintained by **Vercel**). It orchestrates task execution with caching and parallel builds.

**Core concepts**: Task graph, `turbo.json`, pipeline definition, caching (local + remote), `--filter`, task dependencies (`dependsOn`), dry runs.

**In Ledger Live**: Orchestrates all build tasks. When you run `pnpm build:lld`, Turbo resolves the dependency graph, builds libraries in the correct order, and caches results.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://turbo.build/repo/docs">Turborepo documentation</a></li>
<li><a href="https://turbo.build/repo/docs/crafting-your-repository/caching">Turbo caching</a></li>
</ul>
</div>

### Rspack

Rspack is a **high-performance JavaScript bundler** written in Rust, created by **ByteDance**. It is webpack-compatible but significantly faster.

**Core concepts**: Entry points, loaders, plugins, code splitting, tree shaking, hot module replacement (HMR), webpack compatibility layer.

**In Ledger Live Desktop**: Replaced webpack as the bundler. Configuration is in `apps/ledger-live-desktop/rspack.config.ts`. The webpack-compatible API means most existing loaders and plugins work without changes.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://rspack.dev/">Rspack documentation</a></li>
<li><a href="https://rspack.dev/guide/start/quick-start">Rspack Quick Start</a></li>
</ul>
</div>

### Metro

Metro is the **JavaScript bundler for React Native**, created by **Meta**. It bundles JS code and assets for iOS and Android.

**Core concepts**: `metro.config.js`, resolver, transformer, serializer, file map, symlink resolution, cache.

**In Ledger Live Mobile**: Bundles the LLM app. Custom configuration handles pnpm workspace symlinks (using `@rnx-kit/metro-resolver-symlinks`) and forced dependency resolution for deduplication.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://metrobundler.dev/">Metro documentation</a></li>
<li><a href="https://github.com/microsoft/rnx-kit/tree/main/packages/metro-resolver-symlinks">@rnx-kit/metro-resolver-symlinks</a> — the symlink resolver used by LLM</li>
</ul>
</div>

### ESLint & Oxlint

ESLint is a **JavaScript/TypeScript linter** created by **Nicholas Zakas**. Oxlint is a newer, faster alternative written in Rust by the **oxc** project.

**Core concepts**: Rules, plugins, configurations, `--fix`, `.eslintrc`, flat config, custom rules.

**In Ledger Live**: ESLint enforces code quality across the monorepo. Oxlint is being evaluated/adopted for faster linting. Both catch bugs, enforce conventions, and maintain consistency.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://eslint.org/docs/latest/">ESLint documentation</a></li>
<li><a href="https://oxc.rs/docs/guide/usage/linter.html">Oxlint documentation</a></li>
</ul>
</div>

---

## D: Supported Devices & Coins

### Device Models

| Device | Env Value | Touch | Display | Screen Size | Tag | Form Factor | Notes |
|--------|-----------|-------|---------|-------------|-----|-------------|-------|
| **Nano S** | `nanoS` | No | BAGL | 128x32 | `@LNS` | USB stick | Legacy. Limited memory. Requires specific Docker image SHA. |
| **Nano S Plus** | `nanoSP` | No | BAGL | 128x64 | `@NanoSP` | USB stick | Most common test device. Good default for development. |
| **Nano X** | `nanoX` | No | BAGL | 128x64 | `@NanoX` | USB stick + BLE | Bluetooth not relevant in Speculos. |
| **Stax** | `stax` | **Yes** | NBGL | 400x672 | `@Stax` | Card/touchscreen | Touch interactions: tap, swipe, long press. |
| **Flex** | `flex` | **Yes** | NBGL | 480x600 | `@Flex` | Card/touchscreen | Similar to Stax, different form factor. |
| **Nano Gen 5** | `nanoGen5` | **Yes** | NBGL | -- | `@NanoGen5` | USB + touchscreen | Newest device. Internally codenamed "apex". |

### Device Interaction Comparison

| Action | Non-Touch (Nano S/SP/X) | Touch (Stax/Flex/Gen5) |
|--------|-------------------------|------------------------|
| Navigate screens | `pressRight()` / `pressLeft()` | `swipe("left")` / `swipe("right")` |
| Confirm/select | `pressBoth()` | `tap(x, y)` on "Approve" label |
| Cancel | Navigate to "Reject" + `pressBoth()` | `tap(x, y)` on "Reject" label |
| Long confirm | N/A | `longPress(x, y, 3)` (hold for 3 seconds) |

### Coin Families & Modules

| Family | Package | Representative Chains | E2E Coverage |
|--------|---------|----------------------|--------------|
| **EVM** | `@ledgerhq/coin-evm` | Ethereum, Polygon, BSC, Arbitrum, Optimism, Avalanche, Base, Fantom, Cronos, Moonbeam, Blast, Linea, Scroll, zkSync Era | Extensive |
| **Bitcoin** | `@ledgerhq/coin-bitcoin` | Bitcoin, Litecoin, Dogecoin, Bitcoin Cash, Zcash, Dash, Decred, Qtum, Komodo, Peercoin | Extensive |
| **Solana** | `@ledgerhq/coin-solana` | Solana | Yes |
| **Cosmos** | `@ledgerhq/coin-cosmos` | Cosmos (ATOM), Osmosis, dYdX, Injective, Sei, Celestia, Stargaze, Persistence, Quicksilver, Onomy | Yes |
| **Cardano** | `@ledgerhq/coin-cardano` | Cardano (ADA) | Yes |
| **Polkadot** | `@ledgerhq/coin-polkadot` | Polkadot (DOT) | Yes |
| **Tezos** | `@ledgerhq/coin-tezos` | Tezos (XTZ) | Yes |
| **Algorand** | `@ledgerhq/coin-algorand` | Algorand (ALGO) | Yes |
| **Stellar** | `@ledgerhq/coin-stellar` | Stellar (XLM) | Yes |
| **XRP** | `@ledgerhq/coin-xrp` | XRP | Yes |
| **Tron** | `@ledgerhq/coin-tron` | Tron (TRX) | Yes |
| **NEAR** | `@ledgerhq/coin-near` | NEAR Protocol | Yes |
| **MultiversX** | `@ledgerhq/coin-elrond` | MultiversX (EGLD) | Yes |
| **Aptos** | `@ledgerhq/coin-aptos` | Aptos (APT) | Partial |
| **Sui** | `@ledgerhq/coin-sui` | Sui | Partial |
| **Hedera** | `@ledgerhq/coin-hedera` | Hedera (HBAR) | Partial |
| **Vechain** | `@ledgerhq/coin-vechain` | Vechain (VET) | Partial |
| **Internet Computer** | `@ledgerhq/coin-internet-computer` | ICP | Partial |
| **Kaspa** | `@ledgerhq/coin-kaspa` | Kaspa | Partial |
| **Celo** | `@ledgerhq/coin-celo` | Celo | Partial |
| **Filecoin** | `@ledgerhq/coin-filecoin` | Filecoin (FIL) | Partial |
| **Stacks** | `@ledgerhq/coin-stacks` | Stacks (STX) | Partial |
| **Mina** | `@ledgerhq/coin-mina` | Mina Protocol | Partial |
| **Aleo** | `@ledgerhq/coin-aleo` | Aleo | Partial |

> **80+ total supported cryptocurrencies** across these families, including ERC-20 tokens (USDT, USDC, DAI, LINK, UNI, etc.) and other chain-specific tokens.

---

## E: Troubleshooting FAQ

### Docker & Speculos

**Q: `docker pull ghcr.io/ledgerhq/speculos:latest` fails with "unauthorized"**
A: The image is on GitHub Container Registry. Authenticate first:
```bash
echo $GITHUB_TOKEN | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

**Q: Speculos container won't start / immediately exits**
A: Kill all leftover containers and pull a fresh image:
```bash
docker rm -f $(docker ps -aq --filter name=speculos)
docker pull ghcr.io/ledgerhq/speculos:latest
```

**Q: "Port 5000 already in use" error**
A: A previous Speculos container didn't shut down cleanly:
```bash
# Kill all Speculos containers
docker rm -f $(docker ps -aq --filter name=speculos)
# Or kill any process on port 5000
lsof -ti:5000 | xargs kill -9
```

**Q: Docker Desktop is not running**
A: Start Docker Desktop from Applications. Verify with `docker ps`. On macOS, you can also run `open -a Docker`.

**Q: "no app found for Solana on nanoSP"**
A: The coin-apps repo is missing or outdated:
```bash
# Clone if not already done
git clone https://github.com/LedgerHQ/coin-apps.git ~/coin-apps
export COINAPPS=~/coin-apps

# Or pull latest if already cloned
cd ~/coin-apps && git pull
```

**Q: `OCI runtime create failed` — Docker out of disk space**
A: Docker images, containers, and volumes accumulate over time. Free disk space:
```bash
docker system prune -a
```
This removes unused images, stopped containers, and dangling volumes. You will need to re-pull the Speculos image afterwards: `docker pull ghcr.io/ledgerhq/speculos:latest`.

**Q: `APDU error 6985` — user rejected on device**
A: The test sent a transaction but didn't simulate pressing "Approve" on the Speculos device. Make sure your test includes the navigation step to approve the operation on the emulated device screen before expecting a result.

### Build Issues

**Q: `pnpm desktop build:testing` fails**
A: Make sure all dependencies are built first (order matters):
```bash
pnpm build:lld:deps    # Build library dependencies
pnpm build:cli          # Build CLI tool
pnpm desktop build:testing  # Then build the desktop app
```

**Q: TypeScript error: Module '@ledgerhq/xxx' has no exported member 'yyy'**
A: The library needs to be rebuilt after recent changes:
```bash
pnpm turbo build --filter=@ledgerhq/xxx
```

**Q: `pnpm i` fails or hangs**
A: Try cleaning the cache and retrying:
```bash
rm -rf node_modules
pnpm store prune
pnpm i
```

**Q: Out of memory during build (`JavaScript heap out of memory`)**
A: Increase the Node.js memory limit:
```bash
export NODE_OPTIONS="--max-old-space-size=8192"  # 8 GB
```

**Q: iOS pod install fails**
A: Make sure you have the right Xcode version, then clean and retry:
```bash
cd apps/ledger-live-mobile/ios
rm -rf Pods Podfile.lock
cd ../../..
pnpm mobile pod
```

**Q: `pnpm-lock.yaml` merge conflict during rebase**
A: This is the most common merge conflict in the monorepo (from wiki: Issues & Workarounds):
```bash
# Do NOT manually resolve pnpm-lock.yaml conflicts
# Instead, accept either version and regenerate:
git checkout --theirs pnpm-lock.yaml
pnpm i
git add pnpm-lock.yaml
git rebase --continue
```

**Q: Node Action Cache Error in CI**
A: Known CI issue (from wiki: Common CI Troubleshooting). The node_modules cache may be corrupted. Re-run the CI job -- it usually resolves on retry. If persistent, ask a maintainer to purge the GitHub Actions cache.

**Q: CocoaPods lockfile out of sync (cow ASCII art error during `pnpm i`)**
A: This means the iOS CocoaPods cache is stale. Clear it and regenerate:
```bash
rm -rf ~/.cocoapods/
cd apps/ledger-live-mobile/ios/
bundle install
cd ../../..
pnpm mobile pod
```
Then retry `pnpm i`. See also Chapter 5.6 and Chapter 3.7.3.

**Q: `TypeError: Invalid Version: DS_Store` in coin-apps directory**
A: macOS `.DS_Store` files in the coin-apps folder confuse version parsing. Remove them:
```bash
find $COINAPPS -name ".DS_Store" -type f -delete
```

**Q: Node version mismatch — wrong Node.js version**
A: The monorepo requires a specific Node.js version managed by `proto`. Run this in the repo root:
```bash
proto use
```
This reads `.prototools` and switches to the correct Node.js version automatically.

### Test Issues

**Q: All tests timeout immediately**
A: Check these in order:
1. Docker is running: `docker ps`
2. Env vars are set: `echo $MOCK $SPECULOS_DEVICE`
3. App is built: `ls apps/ledger-live-desktop/.webpack/main.bundle.js`
4. Playwright browsers installed: `pnpm e2e:desktop test:playwright:setup`

**Q: Tests pass locally but fail in CI**
A: Common causes:
- Different device model (CI may test `nanoS`, locally you test `nanoSP`)
- Feature flag differences (CI uses production defaults, you may have overrides)
- Timing issues (CI runners may be slower -- increase timeouts)
- Missing coin-app binary for the CI device model in coin-apps repo

**Q: "Error: SEED not set"**
A: The `SEED` environment variable contains the BIP39 mnemonic used to generate deterministic test accounts. In CI it comes from GitHub Secrets. Locally, ask your team lead for the test SEED value and add it to your `~/.zshrc`.

**Q: Flaky test -- intermittent "Element not found" errors**
A: Add explicit waits before interacting with the element:
```typescript
// BAD
await page.getByTestId("send-button").click();

// GOOD
await page.getByTestId("send-button").waitFor({ state: "visible", timeout: 30000 });
await page.getByTestId("send-button").click();
```

**Q: Allure report is empty or missing tests**
A: Check that `allure-results/` directory exists and contains JSON files:
```bash
ls e2e/desktop/allure-results/
# Should contain .json files from the test run
```
If empty, the tests may not have run or the reporter config is wrong.

**Q: "Element not found" after a UI update**
A: The test ID in the source code likely changed. Find the new test ID:
```bash
# Search for the old test ID in the app source
grep -r "old-test-id" apps/ledger-live-desktop/src/
# Then update the page object with the new ID
```

**Q: `Error: Element is not attached to the DOM` (stale element)**
A: The element was removed and re-rendered between the time Playwright found it and tried to interact with it. This happens in dynamic UIs where React re-renders components. Use a fresh locator and wait for stability:
```typescript
await page.getByTestId("my-element").waitFor({ state: "visible" });
await page.getByTestId("my-element").click();
```

**Q: Locale mismatch — assertion fails with `"1,5 BTC"` vs `"1.5 BTC"`**
A: The test machine has a different locale than CI. Decimal separators and number formatting depend on the system locale. Set the locale explicitly in your Playwright config:
```typescript
use: {
  locale: "en-US",
}
```

**Q: Click lands on the wrong element (layout shift)**
A: An animation or lazy-loaded element shifted the layout between the time Playwright found the target and clicked. Wait for the page to stabilize before interacting:
```typescript
// Wait for the animation/transition to finish
await page.getByTestId("container").waitFor({ state: "visible" });
await page.waitForTimeout(300); // only if animation has no stable signal
await page.getByTestId("target-button").click();
```
Better: wait for the specific element that triggers the shift to finish rendering.

**Q: Test passes but Allure report shows wrong steps or missing TMS links**
A: Check these common causes:
1. **Missing `@step()` decorator** — page object methods need the `@step` annotation to appear in Allure
2. **Wrong `xrayTicket` string** — verify the comma-separated B2CQA ticket IDs in `addTmsLink()` match the expected Xray test cases
3. **Missing team tag** — ensure `test.info().annotations` includes the correct team tag
4. Run the test and open the report with `allure serve allure-results` to inspect step traces and links.

### Mobile-Specific Issues

**Q: Metro bundler won't start**
A: Port 8081 may be occupied. Kill the existing process:
```bash
lsof -ti:8081 | xargs kill -9
pnpm mobile start
```

**Q: iOS simulator not detected by Detox**
A: Verify the simulator is available:
```bash
xcrun simctl list devices available | grep -i iphone
# If no devices, create one in Xcode > Window > Devices and Simulators
```

**Q: Android emulator not detected**
A: Check AVD and adb:
```bash
# List available emulators
emulator -list-avds
# Start one
emulator -avd Pixel_7_Pro_API_35
# Check adb sees it
adb devices
```

**Q: WebSocket bridge connection timeout (mobile E2E)**
A: The bridge port may not be forwarded. For Android:
```bash
# Check that adb reverse is set
adb reverse tcp:8099 tcp:8099
```
For iOS, the bridge connects directly -- check that the simulator is running and the app launched.

**Q: Android E2E tests crash immediately in debug mode**
A: Known issue -- Android debug builds are broken locally due to a Detox/Espresso bug. Always use release builds:
```bash
cd e2e/mobile && pnpm build:android
# or: pnpm --filter ledger-live-mobile-e2e-tests run build:android
```

**Q: Android emulator not found — name mismatch**
A: Detox expects the AVD to be named exactly `Android_Emulator`. Check your AVD name:
```bash
emulator -list-avds
```
If the name doesn't match, rename it in Android Studio (Device Manager → Edit → rename) or create a new AVD with the exact name `Android_Emulator`.

---

## F: Git Cheat Sheet

<div class="chapter-intro">
<strong>Why this matters.</strong> Ledger Live is a large monorepo with hundreds of PRs a week. Commits go through <code>commitlint</code>, <code>gitleaks</code>, GPG signing, and pre-commit hooks. A few git missteps can corrupt your local <code>develop</code>, lose a merge commit, or force-push over a teammate's work. This section collects the commands you'll use every day, plus the ones you'll reach for when things go sideways.
</div>

### Everyday inspection

```bash
git status                              # working tree + staged state
git status --short                      # compact one-letter status
git diff                                # unstaged changes
git diff --cached                       # staged changes (what will commit)
git log --oneline -10                   # last 10 commits, compact
git log --oneline develop..HEAD         # commits on current branch not on develop
git log --format=%B -1                  # full message of last commit
git log -1 --format=%B | awk '{print length, $0}'  # line lengths of last commit msg
git blame <file>                        # who last changed each line
git show <sha>                          # full diff of one commit
```

### Branching

```bash
git switch <branch>                     # modern replacement for checkout
git switch -c feat/my-feature develop   # create new branch from develop
git checkout <branch>                   # older equivalent (still works)
git branch                              # list local branches
git branch -a                           # + remote tracking branches
git branch --show-current               # current branch name
git branch --contains <sha>             # which branches contain this commit
git branch -d <branch>                  # delete merged branch
git branch -D <branch>                  # force delete (unmerged) - careful
```

### Staging & committing

```bash
git add <file>                          # stage a specific file (preferred)
git add -p                              # stage by hunks (interactive)
git restore --staged <file>             # unstage without losing changes
git commit -m "subject"                 # single-line message
```

**Multi-line commit with heredoc** (preserves line breaks exactly — use this whenever you need a body):

```bash
git commit -m "$(cat <<'EOF'
type(scope): short imperative subject

- body bullet one, wrapped at 100 chars max
- body bullet two
  continuation line, 2-space indented to read as a continuation
EOF
)"
```

**Amending the last commit:**

```bash
git commit --amend -m "new subject"     # change message only
git commit --amend --no-edit            # keep message, add newly staged changes
git commit --amend                      # opens $EDITOR to edit the message
```

⚠️ **Never amend a commit that has already been pulled by someone else** — you'll rewrite shared history. Amending is safe on your own feature branch that only you push to.

### Pulling & fetching

```bash
git fetch origin                        # update all remote refs, no merge
git fetch origin develop                # fetch only develop
git pull                                # fetch + merge (or rebase if configured)
git pull --rebase                       # fetch + rebase local commits on top
git pull --ff-only                      # refuse if not a fast-forward
```

### Merging

```bash
git merge <branch>                      # merge branch into current
git merge --no-ff <branch>              # force a merge commit even if FF possible
git merge --abort                       # bail out of a conflicted merge
git commit                              # finalize merge after resolving conflicts
```

### Rebasing

```bash
git rebase <base>                       # replay current branch commits onto <base>
git rebase origin/develop               # rebase feature branch onto latest develop
git rebase --continue                   # after resolving conflicts
git rebase --skip                       # drop current conflicted commit
git rebase --abort                      # return to pre-rebase state
git rebase -i <base>                    # interactive: reword, squash, fixup, drop
```

**Interactive rebase actions** (change the leading `pick` keyword in the editor):

| Action | Effect |
|---|---|
| `pick` | keep the commit as-is |
| `reword` (`r`) | keep the diff, rewrite the message |
| `edit` (`e`) | pause here so you can `--amend` content |
| `squash` (`s`) | merge into previous commit, combine messages |
| `fixup` (`f`) | merge into previous commit, **discard** this message |
| `drop` (`d`) | delete the commit entirely |

### Pushing (safely)

```bash
git push                                # push current branch to its tracked remote
git push -u origin feat/my-feature      # set upstream on first push
git push --force-with-lease             # force push, refuses if remote moved (SAFE)
git push --force                        # force push, clobbers remote (DANGEROUS)
```

🚫 **Never** `--force` push to `develop` or `main`. Use `--force-with-lease` on your feature branches only.

### Resetting & undoing

```bash
git reset --soft <ref>                  # move HEAD, keep staging + working tree
git reset --mixed <ref>                 # move HEAD, keep working tree only (default)
git reset --hard <ref>                  # move HEAD, wipe staging + working tree
git restore <file>                      # discard working-tree changes to one file
git restore --source=HEAD~1 <file>      # restore file to its version 1 commit back
git revert <sha>                        # create a NEW commit that undoes <sha>
```

**Common scenarios:**

| Want to… | Command |
|---|---|
| Undo last commit, keep changes staged | `git reset --soft HEAD~1` |
| Undo last commit, keep changes unstaged | `git reset HEAD~1` |
| Nuke last commit entirely | `git reset --hard HEAD~1` |
| Realign local `develop` with `origin/develop` | `git fetch && git reset --hard origin/develop` |
| Squash last 4 commits into 1 | `git reset --soft HEAD~4 && git commit -m "..."` |

### Recovery — `reflog` is your safety net

Every HEAD move is logged. Even after a bad `reset --hard`, the old commit is still in the reflog for ~30 days.

```bash
git reflog                              # every HEAD move with timestamps
git reflog --date=iso -n 20             # with absolute dates
git reset --hard HEAD@{5}               # reset to HEAD state 5 moves ago
git checkout HEAD@{yesterday}           # check out yesterday's HEAD
```

### Stashing

```bash
git stash                               # save uncommitted changes, clean WD
git stash -u                            # include untracked files
git stash push -m "WIP swap fix"        # named stash
git stash list                          # show all stashes
git stash pop                           # restore most recent + remove from stack
git stash apply stash@{2}               # restore a specific stash, keep in stack
git stash drop stash@{0}                # delete a stash
```

### Cherry-picking

```bash
git cherry-pick <sha>                   # copy one commit to current branch
git cherry-pick <sha1>..<sha2>          # range (exclusive of sha1)
git cherry-pick --continue              # after resolving conflicts
git cherry-pick --abort                 # bail out
```

### Resolving merge conflicts

**With `git-conflict.nvim`** (install it in your nvim config — the only decent conflict plugin in this setup):

1. Open the file in nvim
2. `]x` / `[x` — jump to next / previous conflict
3. `co` — choose **ours** (current branch / HEAD)
4. `ct` — choose **theirs** (incoming branch)
5. `cb` — keep **both** (⚠️ only safe when no duplicate keys / invalid syntax result)
6. `c0` — discard the entire conflict block
7. `:GitConflictListQf` — populate quickfix with all conflicts in the repo

**Manual (plain vim), if no plugin:**

```
/<<<<<<<       then n to cycle markers
dd             on each marker line, or
V}d            to delete one whole side in visual-block mode
```

**After resolving:**
```bash
git add <file>
git commit                              # for plain merge
git rebase --continue                   # during a rebase
git cherry-pick --continue              # during a cherry-pick
```

### Comparing branches

```bash
git log --oneline A..B                  # commits on B but not on A
git log --oneline --left-right A...B    # both sides of divergence, prefixed < / >
git rev-list --left-right --count A...B # "X  Y" = X on A only, Y on B only
git diff A...B                          # diff from merge-base of A and B to B
```

### Diagnosing a diverged local branch

If `git rev-list --left-right --count develop...origin/develop` returns anything other than `0 0`, your local `develop` has drifted — usually from an accidental rebase or a pull that rewrote history. Safe recovery:

```bash
git fetch origin
git switch develop
git reset --hard origin/develop
```

Do **not** try to merge the mess back — you'll duplicate commits.

---

### Ledger Live conventions

**Target branch.** Feature PRs target `develop`, never `main`. `main` is protected and only receives merges from `develop` at release time.

**Branch naming:**

| Prefix | Use for |
|---|---|
| `feat/` | New features |
| `fix/` or `bugfix/` | Bug fixes |
| `support/` | Refactor, tests, CI improvements |
| `chore/` | Maintenance, tooling, configs |
| `test/` | Test-only additions (common for QA tickets) |
| `docs/` | Documentation only |

Use kebab-case, keep names short, one branch = one concern. Include the Jira ticket when relevant: `test/qaa-1136-desktop-swap`.

**Conventional Commits** (enforced by `commitlint`):

```
type(scope): short imperative subject

optional body, wrapped at 100 chars per line
bullet continuations indented 2 spaces

optional footer (BREAKING CHANGE: ..., Refs: LIVE-1234)
```

| Type | Use for |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `test` | Add/update tests |
| `docs` | Documentation only |
| `style` | Formatting, no code change |
| `refactor` | Restructure without behavior change |
| `perf` | Performance improvement |
| `chore` | Maintenance, tooling, configs |
| `ci` | CI/CD changes |

Common scopes: `desktop`, `mobile`, `common`, `coin`, `<family>` (e.g. `ethereum`, `solana`), or a package name.

**Commitlint rules that bite:**

- `body-max-line-length`: body lines must be **≤ 100 chars**
- `body-leading-blank`: a blank line **must** separate the subject from the body
- `subject-case`: subject must be lowercase after the colon (`: add`, not `: Add`)
- `type-enum`: type must be one of the allowed list above

**Test your commit message locally before pushing:**
```bash
git log -1 --format=%B | pnpm exec commitlint
```

**Changesets.** Every behavior-changing PR on a published package ships a `.changeset/*.md` file:
```bash
pnpm changeset                          # interactive: pick packages + bump level
```
Commit the generated file alongside your code. CI will fail without it when required.

**GPG signing.** Ledger requires GPG-signed commits on protected branches. If you see `gpg: signing failed: No secret key`, ensure `user.signingkey` is configured and `gpg-agent` is running:
```bash
git config --global user.signingkey <YOUR_KEY_ID>
gpg --list-secret-keys --keyid-format=long
```

---

### GitHub CLI (`gh`)

```bash
gh auth login                           # one-time setup
gh pr create --title "..." --body "..."
gh pr view                              # details of PR for current branch
gh pr view 16716 --json state,mergedAt
gh pr list --author "@me" --state open
gh pr checks                            # CI status for current PR
gh pr checkout 16716                    # check out a PR locally
gh pr review --approve                  # approve a PR
gh issue view QAA-702                   # view a GitHub issue
gh api repos/LedgerHQ/ledger-live/pulls/16716/comments  # raw API call
gh run list --branch $(git branch --show-current)       # recent CI runs for branch
gh run watch                            # live-tail the most recent run
gh run rerun <run-id>                   # re-run a failed workflow
```

---

### Common gotchas at Ledger

1. **Local `develop` diverging from origin.** Caused by accidental `pull --rebase` or someone running a rebase on the wrong branch. Fix with `git fetch && git reset --hard origin/develop` — never try to merge the mess back.
2. **`gpg: signing failed`** after switching shells or entering a sandbox. Verify the key (`gpg --list-secret-keys`), restart `gpg-agent`.
3. **Pre-commit hook rewriting staged files.** If a linter auto-fixes, the first commit may fail. Re-run `git add` on the affected files and `git commit` again.
4. **`pnpm-lock.yaml` conflicts.** Do **not** hand-edit. Accept either side, then `pnpm install` to regenerate.
5. **Force-pushing after a teammate rebased.** `--force-with-lease` refuses and saves you. Always prefer it over `--force`.
6. **Amending a pushed commit.** Safe only on your own feature branch. Follow with `git push --force-with-lease`.
7. **Uncommitted work blocking a pull.** `git stash` → `git pull --rebase` → `git stash pop`.
8. **"Detached HEAD" after `git checkout <sha>`.** Use `git switch -c new-branch` to save your position as a real branch before making changes.
9. **Missing Jira ticket in the PR title.** Ledger Live PR titles typically follow `[LWDM|LWM|...] type(scope): description` — check your team's convention.
10. **Changeset file missing.** CI fails on PRs that modify published packages without a `.changeset/*.md`. Run `pnpm changeset` before opening the PR.

---

## G: Glossary

| Term | Definition |
|------|------------|
| **APDU** | Application Protocol Data Unit. The low-level binary protocol used for all communication between a host application and a smartcard/secure element (Ledger device). Commands are sent as byte arrays. |
| **Allure** | An open-source test reporting framework that generates interactive HTML reports with step traces, screenshots, videos, and environment information. |
| **Alpaca** | Generic adapter layer for coin integrations. 4 of 29 coin families are "alpacaized" as of 2026-04 (evm, stellar, tezos, xrp). Only xrp is fully extracted into `alpaca-coin-module`. |
| **B2CQA** | Jira project holding Xray test case definitions. Test types: 10756 Test Plan, 10759 Test Case, 10760 Precondition. |
| **BAGL** | Ledger's display framework for non-touch devices (Nano S, Nano S Plus, Nano X). Renders UI on small monochrome screens (128x32 or 128x64 pixels). |
| **BIP39** | Bitcoin Improvement Proposal 39. Defines the standard for mnemonic seed phrases (12 or 24 words) used to derive cryptographic keys deterministically. |
| **BOLOS** | Blockchain Open Ledger Operating System. The SE OS running on Ledger hardware devices. |
| **Bot (Ledger Live Bot)** | A stateless, autonomous test engine that crawls all accounts in a seed, discovers possible operations, and executes randomized mutations. Uses 7 named seeds (Mere Denis through Nitrogen). |
| **Changeset** | A small markdown file (in `.changeset/`) describing changes to a published package. Used by the changesets tool to generate changelogs and version bumps. |
| **Changesets** | Ledger Live's release-versioning tooling (`@changesets/cli`). Every behavior-changing PR ships a `.changeset/*.md` file. NOT semantic-release. |
| **CI/CD** | Continuous Integration / Continuous Deployment. Automated pipeline that builds, tests, and deploys code on every change. Ledger Live uses GitHub Actions. |
| **CLI** | Command-Line Interface. Ledger Live's CLI tool (`apps/cli/`) is used in E2E tests to populate test data (accounts, transactions). |
| **CODEOWNERS** | `.github/CODEOWNERS` file mapping path patterns to GitHub teams that auto-review PRs touching those paths. |
| **Coin App** | A compiled firmware application (`.elf` binary) that runs on a Ledger device to support a specific cryptocurrency (e.g., the Ethereum app, Bitcoin app). |
| **Coin Family** | A group of related blockchains sharing the same transaction model and coin module. E.g., the EVM family includes Ethereum, Polygon, BSC, Arbitrum, etc. |
| **Content-Addressable Store** | pnpm's storage mechanism where every package version is stored exactly once, identified by its content hash. Saves disk space in monorepos. |
| **Conventional Commits** | A commit message standard: `type(scope): description`. Enables automated changelog generation and semantic versioning. |
| **Detox** | A grey-box E2E testing framework for React Native by Wix. "Grey-box" means it has visibility into app internals for synchronization. |
| **Donjon** | Ledger's security research team. Operates the public bug bounty at donjon.ledger.com. |
| **E2E** | End-to-End testing. Tests that simulate real user interactions across the full application stack, from UI to backend to device. |
| **E2E workspace** | Top-level peer workspaces under `e2e/` — `e2e/desktop/` (Playwright) and `e2e/mobile/` (Detox), each an independent Nx/Turbo project with its own `package.json`, `project.json`, Jest/Playwright config, and specs. Mobile migrated from `apps/ledger-live-mobile/e2e/` to `e2e/mobile/` in 2026; a small legacy set of ~17 specs still lives under `apps/ledger-live-mobile/e2e/` during the transition. |
| **EAL** | Evaluation Assurance Level (Common Criteria). Ledger Stax/Flex/Nano S+ are EAL6+; Nano X is EAL5+. |
| **ELF** | Executable and Linkable Format. The binary format used for compiled Ledger coin applications that run inside Speculos. |
| **Electron** | A framework for building cross-platform desktop applications using web technologies (Chromium + Node.js). Ledger Live Desktop is an Electron app. |
| **ENVFILE** | Environment file passed to Detox / native builds. Two namespaces, both physically under `apps/ledger-live-mobile/` (path unchanged by the E2E migration): Detox mocks — `.env.mock` (staging Firebase) and `.env.mock.prerelease` (prod Firebase), referenced from `e2e/mobile/detox.config.js` via `ENV_FILE_MOCK` constants. Release builds — `.env`, `.env.testing`, `.env.staging`, `.env.production`. |
| **Feature Flag** | A configuration toggle stored in Firebase Remote Config that enables or disables features without deploying new code. Supports gradual rollouts and kill switches. |
| **Firebase Remote Config** | Google's cloud service used by Ledger Live to store and serve feature flags. Four environments: Ledger Wallet, Swap, Earn, Buy Sell. |
| **Fixture** | In Playwright, a mechanism for declaring reusable test setup and teardown logic. Fixtures are injected into tests via parameter destructuring. The Ledger Live fixture system handles Speculos, Electron, page objects, and feature flags. |
| **Flaky Test** | A test that intermittently passes or fails without any code changes. Usually caused by timing issues, race conditions, or shared state. |
| **Flex** | Ledger hardware device, flat E-Ink, EAL6+, internal codename "Europa". |
| **Grey-Box Testing** | Testing approach where the test framework has some visibility into the application's internals (e.g., synchronization with React Native bridge). Detox uses this approach. |
| **hk** | A modern pre-commit hook manager (alternative to Husky) used in the Ledger Live repo. Configured via `hk.pkl`. Runs `gitleaks` to scan for secrets. |
| **IPC** | Inter-Process Communication. In Electron, the mechanism for communication between the main process (Node.js) and renderer process (Chromium). Uses `ipcMain` and `ipcRenderer`. |
| **Jest** | A JavaScript testing framework by Meta (Facebook). Used for unit and integration tests in Ledger Live. Also serves as the test runner for Detox mobile E2E tests. |
| **Ledger Live** | Ledger's official companion application for managing crypto assets. Available as Desktop (Electron), Mobile (React Native), and CLI. |
| **LIVE** | Jira project key for Ledger Wallet product work and bugs. LLM and LLD are LABELS on LIVE tickets, not separate projects. |
| **LWD** | Shorthand for Ledger Live Desktop (also sometimes LLD). |
| **LWM** | Shorthand for Ledger Live Mobile (also sometimes LLM). |
| **Manifest API** | GitHub repo hosting per-environment Live App manifests. |
| **Manifest V2** | Current Live App manifest schema. Fields include `type` (`dapp | walletApp | webBrowser`) and `visibility`. |
| **MCU** | Microcontroller Unit. The non-secure chip handling screen, USB, buttons. STM32WB35/55 on most current devices. Distinct from the SE. |
| **Mise** | A development tool version manager (alternative to nvm, asdf). Used in Ledger Live to manage Node.js, pnpm, and other tool versions consistently. |
| **Mnemonic Seed** | A set of words (usually 24) that encodes a master cryptographic key per BIP39. The same seed always generates the same accounts, enabling deterministic test data. |
| **Monorepo** | A single Git repository containing multiple related projects and packages. Ledger Live is a monorepo with 2 apps, 40+ libraries, 2 E2E suites, and CI tools. |
| **MSW** | Mock Service Worker. A library for intercepting and mocking HTTP requests in tests. Used in Jest integration tests to mock API calls without a real server. |
| **Mutation (Bot)** | A single operation the Ledger Live Bot attempts on an account (e.g., send max, send sub-max, delegate, swap). Each mutation has preconditions, a transaction builder, and assertion checks. |
| **MVVM** | Model-View-ViewModel. An architecture pattern used in newer Ledger Live code. Components are split into Container (wiring), ViewModel (logic), and View (pure UI). |
| **NBGL** | New BAGL. Ledger's modern display framework for touch-screen devices (Stax, Flex, Nano Gen 5). Supports full-color touchscreen interactions. |
| **OSU** | Operating System Update. The SE OS update flow. |
| **Page Object Model (POM)** | A design pattern where each UI page or component is represented as a class with locators (element selectors) and methods (user actions). Reduces test code duplication. |
| **Part 0 Welcome** | Orientation part of this guide (ecosystem, hardware, teams, release cycle, ticket flow, security). |
| **Playwright** | A browser automation framework by Microsoft. Used for Ledger Live Desktop E2E testing. Supports Chromium, Firefox, WebKit, and Electron applications. |
| **pnpm** | A fast, disk-efficient package manager optimized for monorepos. Uses a content-addressable store and symlinks instead of copying files. |
| **PTX** | Post-Transaction eXperience tribe at Ledger. Owns Buy/Sell, Swap, Earn, Recover. NOT a Jira project key. |
| **QAA** | Quality Assurance Automation. The team writing this guide. Also the Jira project key for automation work items. |
| **Recover** | Ledger's opt-in self-custody seed-recovery service. SE-side code + Trust orchestrator + 3 backup providers. |
| **RTK Query** | A data fetching and caching tool built on Redux Toolkit. Provides auto-generated React hooks for API calls with built-in caching and invalidation. |
| **Rspack** | A Rust-based web bundler that's a drop-in replacement for Webpack, offering 5-10x faster build times. Used for the Ledger Live Desktop build. |
| **SE** | Secure Element. The certified chip holding cryptographic secrets. ST33 on Stax/Flex/Nano S+; STM32WB55 on Nano X. |
| **SEED** | In the context of E2E tests, the BIP39 mnemonic phrase used to generate deterministic test accounts and addresses in Speculos. Stored as a CI secret. |
| **Sharding** | Splitting a test suite into multiple parallel groups (shards) to reduce total execution time. Ledger Live CI shards desktop E2E tests across multiple runners by device model. |
| **Speculos** | Ledger's open-source device emulator. Runs real device firmware inside a Docker container, exposing a REST API for automated testing without physical hardware. |
| **Team-Split Convention** | A repo convention where multi-team folders are split into `team-<slug>/` subfolders, each owned by a specific team in CODEOWNERS. |
| **TMS** | Test Management System. In Ledger Live, refers to Jira/Xray test case IDs (e.g., `B2CQA-817`). Tests link to TMS IDs via annotations for traceability. |
| **Train QA / Train Engineer / Train Delivery** | The three roles per release train (one Desktop train, one Mobile train). |
| **Turbo / Turborepo** | A high-performance monorepo build orchestration tool by Vercel. Manages task dependencies, caching, and parallel execution across packages. |
| **Userdata** | Pre-configured JSON files (in `e2e/desktop/tests/userdata/`) containing serialized app state. Used to start tests from a known state without going through onboarding. |
| **Wallet XP** | Tribe at Ledger covering Live Hub and Devices Experience teams. |
| **WebSocket Bridge** | A communication channel between the mobile E2E test harness (Jest/Detox) and the React Native app. Enables commands like `importAccounts`, `overrideFeatureFlag`, and `navigate`. |
| **Xray** | A Jira plugin for test management. Tracks test cases, test execution, and links automated test results back to Jira tickets. Results exported as JSON. |

---

<div class="chapter-outro">
<strong>You've reached the end of the Appendix -- and the end of this guide.</strong><br>
Use these references whenever you need to look up a command, variable, device, or term. Keep this guide bookmarked -- you'll refer to it daily during your first weeks.<br><br>
Welcome to the team. Ship quality. Break nothing. Test everything.
</div>
