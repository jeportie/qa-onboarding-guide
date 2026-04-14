

# PART 5 -- CRASH COURSES

<div class="chapter-intro">
These crash courses provide a quick introduction to every technology in the Ledger Live stack. They are designed as <strong>standalone references</strong> you can come back to. Each chapter covers: what the tool is, core concepts, essential syntax, and how it is used in Ledger Live. Each includes resource links with official documentation, the technology's author, and interactive learning resources where available.
<br><br>
<strong>How to use Part 5:</strong> You do not need to read these sequentially. Use them as a reference when you encounter a technology in the codebase. The crash courses complement — not replace — the official documentation linked in each chapter.
</div>

---

## Chapter 22: TypeScript

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

---

## Chapter 23: React

React is a **UI library** for building component-based interfaces, created by **Meta** (Jordan Walke). It uses a virtual DOM for efficient rendering.

**Core concepts**: Components (function), JSX, props, state (`useState`), effects (`useEffect`), context (`useContext`), custom hooks, memoization (`useMemo`, `useCallback`, `React.memo`).

**In Ledger Live Desktop**: The entire UI is React components. State is managed with Redux Toolkit (Ch 26). Navigation uses React Router (Ch 27). Styling uses styled-components (Ch 30).

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://react.dev/">React documentation</a> — official docs (new site)</li>
<li><a href="https://react.dev/learn">React Learn</a> — interactive tutorial</li>
<li><a href="https://www.debugbear.com/blog/measuring-react-app-performance">Measuring React Performance</a></li>
</ul>
</div>

---

## Chapter 24: React Native

React Native is a framework for building **native mobile apps** using React, created by **Meta**. It renders to native iOS/Android components, not a WebView.

**Core concepts**: `View`, `Text`, `ScrollView`, `FlatList`, `TouchableOpacity`, `StyleSheet`, platform-specific code (`.ios.ts`/`.android.ts`), native modules, Metro bundler.

**In Ledger Live Mobile**: The entire LLM app. Uses React Navigation (Ch 28) for screen transitions, styled-components for styling, and Redux for state.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://reactnative.dev/">React Native documentation</a></li>
<li><a href="https://reactnative.dev/docs/debugging">React Native Debugging</a></li>
<li><a href="https://expo.dev/snacks">Expo Snack</a> — browser-based React Native playground</li>
</ul>
</div>

---

## Chapter 25: Electron

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

---

## Chapter 26: Redux Toolkit

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

---

## Chapter 27: React Router

React Router is a **routing library** for React web apps, created by **Remix** (Ryan Florence, Michael Jackson).

**Core concepts**: `<BrowserRouter>`, `<Routes>`, `<Route>`, `<Link>`, `useNavigate`, `useParams`, `useLocation`, nested routes, route guards.

**In Ledger Live Desktop**: URL-based navigation between screens (portfolio, accounts, settings, etc.). Deep links use React Router paths.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://reactrouter.com/en/main">React Router documentation</a></li>
</ul>
</div>

---

## Chapter 28: React Navigation

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

---

## Chapter 29: i18next

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

---

## Chapter 30: Styled-Components

Styled-components is a **CSS-in-JS library** that uses tagged template literals to style React components, created by **Max Stoiber** and **Glen Maddern**.

**Core concepts**: `styled.div`, template literals, props-based styling, themes, `ThemeProvider`, `css` helper, extending styles.

**In Ledger Live**: Both LLD and LLM use styled-components for all UI styling. The design system uses a shared theme with colors, fonts, and spacing.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://styled-components.com/docs">Styled-components documentation</a></li>
</ul>
</div>

---

## Chapter 31: Tailwind CSS

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

---

## Chapter 32: Playwright

Playwright is a **browser automation framework** for E2E testing, created by **Microsoft** (Andrey Lushnikov, Dmitry Gozman — former Puppeteer team).

**Core concepts**: `test`, `expect`, locators (`getByTestId`, `getByRole`, `getByText`), fixtures, auto-wait, screenshots, video, traces, parallel execution, sharding, Electron support.

**In Ledger Live**: Desktop E2E suite. Custom fixtures extend Playwright with Speculos, Electron lifecycle, page objects, and feature flag management. See Chapters 8, 13-14 for deep coverage.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://playwright.dev/docs/intro">Playwright documentation</a></li>
<li><a href="https://playwright.dev/docs/codegen">Playwright Codegen</a> — record interactions to generate test code</li>
<li><a href="https://try.playwright.tech/">Try Playwright</a> — interactive browser playground</li>
<li><a href="https://playwright.dev/docs/api/class-electron">Playwright Electron API</a></li>
</ul>
</div>

---

## Chapter 33: Detox

Detox is a **gray-box E2E testing framework** for React Native, created by **Wix Engineering** (Rotem Mizrachi-Meidan). It provides native-level interaction with automatic synchronization.

**Core concepts**: `element(by.id())`, `by.text()`, `.tap()`, `.typeText()`, `.scroll()`, `waitFor().withTimeout()`, `device.launchApp()`, `device.reloadReactNative()`, synchronization.

**In Ledger Live**: Mobile E2E suite. Uses Jest as test runner. Global `app` singleton, WebSocket bridge, helper wrappers. See Chapters 9, 15-16 for deep coverage.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://wix.github.io/Detox/">Detox documentation</a></li>
<li><a href="https://wix.github.io/Detox/docs/api/matchers">Detox Matchers API</a></li>
<li><a href="https://wix.github.io/Detox/docs/api/actions-on-element">Detox Actions API</a></li>
</ul>
</div>

---

## Chapter 34: Jest

Jest is a **JavaScript testing framework** created by **Meta** (Christoph Nakazawa). It includes test runner, assertion library, mocking, and code coverage.

**Core concepts**: `describe`, `it`/`test`, `expect`, matchers (`.toBe`, `.toEqual`, `.toContain`), `beforeEach`/`afterEach`, mocking (`jest.fn()`, `jest.mock()`, `jest.spyOn()`), snapshots, `--coverage`.

**In Ledger Live**: Unit and integration tests throughout. Mobile E2E uses Jest as the Detox test runner. Key rule: never use `jest.restoreAllMocks()` (see Ch 21).

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://jestjs.io/docs/getting-started">Jest documentation</a></li>
<li><a href="https://jestjs.io/docs/mock-functions">Jest Mock Functions</a></li>
</ul>
</div>

---

## Chapter 35: MSW

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

---

## Chapter 36: Allure

Allure is a **test reporting framework** created by **Qameta Software**. It generates interactive HTML reports from test results.

**Core concepts**: Steps, attachments (screenshots, videos), annotations (severity, feature, story, owner), TMS links, history, categories, environment info.

**In Ledger Live**: Both desktop and mobile E2E suites use Allure. The `@step` decorator creates the step trace. TMS annotations link to Jira/Xray. See Chapter 12 for deep coverage.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://allurereport.org/docs/">Allure Report documentation</a></li>
<li><a href="https://allurereport.org/docs/playwright/">Allure Playwright integration</a></li>
<li><a href="https://allurereport.org/docs/jest/">Allure Jest integration</a></li>
</ul>
</div>

---

## Chapter 37: Testing Library

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

---

## Chapter 38: pnpm

pnpm is a **fast, disk-space efficient package manager** created by **Zoltan Kochan**. It uses a content-addressable store with symlinks.

**Core concepts**: Content-addressable store, symlinks, `--filter`, workspaces, `.pnpmfile.cjs`, `pnpm-workspace.yaml`, `--frozen-lockfile`, aliases.

**In Ledger Live**: The package manager for the entire monorepo. See Chapter 7 for deep coverage.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://pnpm.io/">pnpm documentation</a></li>
<li><a href="https://pnpm.io/filtering">pnpm filtering</a></li>
<li><a href="https://pnpm.io/workspaces">pnpm workspaces</a></li>
</ul>
</div>

---

## Chapter 39: Turborepo

Turborepo is a **build system for monorepos** created by **Jared Palmer** (now maintained by **Vercel**). It orchestrates task execution with caching and parallel builds.

**Core concepts**: Task graph, `turbo.json`, pipeline definition, caching (local + remote), `--filter`, task dependencies (`dependsOn`), dry runs.

**In Ledger Live**: Orchestrates all build tasks. When you run `pnpm build:lld`, Turbo resolves the dependency graph, builds libraries in the correct order, and caches results. See Chapter 7.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://turbo.build/repo/docs">Turborepo documentation</a></li>
<li><a href="https://turbo.build/repo/docs/crafting-your-repository/caching">Turbo caching</a></li>
</ul>
</div>

---

## Chapter 40: Rspack

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

---

## Chapter 41: Metro

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

---

## Chapter 42: ESLint & Oxlint

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

## Part 5 Final Assessment

<div class="quiz-container" data-pass-threshold="80">
<h3>Part 5 Final Assessment — Crash Courses</h3>
<p class="quiz-subtitle">10 questions across all technologies · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> What makes pnpm more disk-efficient than npm in a monorepo?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It compresses all packages into a single archive</button>
<button class="quiz-choice" data-value="B">B) It only installs production dependencies by default</button>
<button class="quiz-choice" data-value="C">C) It uses a content-addressable store with symlinks — one copy per version on disk</button>
<button class="quiz-choice" data-value="D">D) It removes unused packages automatically</button>
</div>
<p class="quiz-explanation">pnpm's content-addressable store keeps exactly one copy of each package version globally. All projects link to the store via symlinks, avoiding the massive duplication that npm creates in monorepos with 200+ packages.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> In Electron, what is the purpose of the preload script?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It loads CSS before the page renders</button>
<button class="quiz-choice" data-value="B">B) It bridges the main process and renderer process via IPC while maintaining context isolation</button>
<button class="quiz-choice" data-value="C">C) It preloads images for faster rendering</button>
<button class="quiz-choice" data-value="D">D) It installs npm dependencies at startup</button>
</div>
<p class="quiz-explanation">The preload script runs in the renderer process but has access to Node.js APIs. It exposes a controlled API to the renderer via <code>contextBridge.exposeInMainWorld()</code>, bridging main and renderer processes securely.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> What is the key difference between Playwright's auto-wait and Detox's synchronization?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) They are exactly the same mechanism</button>
<button class="quiz-choice" data-value="B">B) Playwright doesn't have any waiting mechanism</button>
<button class="quiz-choice" data-value="C">C) Detox auto-waits for elements, Playwright requires manual waits</button>
<button class="quiz-choice" data-value="D">D) Playwright auto-waits for elements to be actionable before each action; Detox waits for the app to be idle (no animations, no network, no timers) before each action</button>
</div>
<p class="quiz-explanation">Playwright waits per-element (retries until the specific element is visible, stable, and enabled). Detox waits for global app idleness (no pending animations, network requests, or timers). Both reduce flakiness but through different mechanisms.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> In Redux Toolkit, what does <code>createSlice</code> automatically generate?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Action creators and action types from the reducer functions you define</button>
<button class="quiz-choice" data-value="B">B) React components for each state field</button>
<button class="quiz-choice" data-value="C">C) API endpoints for each reducer</button>
<button class="quiz-choice" data-value="D">D) Database tables for persistence</button>
</div>
<p class="quiz-explanation"><code>createSlice</code> takes a name, initial state, and reducer functions, and automatically generates action creators and action types. This eliminates the boilerplate of defining separate action types and creators.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> How does MSW (Mock Service Worker) intercept API calls?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) By modifying the application's HTTP client code</button>
<button class="quiz-choice" data-value="B">B) At the network level — the application code doesn't know it's being mocked</button>
<button class="quiz-choice" data-value="C">C) By replacing the global <code>fetch</code> function with a custom implementation</button>
<button class="quiz-choice" data-value="D">D) By injecting a proxy server between the app and the API</button>
</div>
<p class="quiz-explanation">MSW intercepts requests at the network level using Service Workers (browser) or Node.js request interception. The application code makes real HTTP calls that are intercepted transparently — no code changes needed.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> What problem does <code>@rnx-kit/metro-resolver-symlinks</code> solve in Ledger Live Mobile?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It speeds up Metro's file watching</button>
<button class="quiz-choice" data-value="B">B) It enables TypeScript support in Metro</button>
<button class="quiz-choice" data-value="C">C) It resolves pnpm workspace symlinks that Metro's default resolver can't follow</button>
<button class="quiz-choice" data-value="D">D) It compresses JavaScript bundles for smaller app size</button>
</div>
<p class="quiz-explanation">pnpm uses symlinks to link packages, but Metro's default resolver doesn't follow symlinks properly. <code>@rnx-kit/metro-resolver-symlinks</code> patches this so Metro can resolve packages across the monorepo workspace.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q7.</strong> What is the Testing Library query priority (most preferred to least preferred)?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>getByRole</code> → <code>getByLabelText</code> → <code>getByText</code> → <code>getByTestId</code></button>
<button class="quiz-choice" data-value="B">B) <code>getByTestId</code> → <code>getByText</code> → <code>getByRole</code> → <code>getByLabelText</code></button>
<button class="quiz-choice" data-value="C">C) <code>getByText</code> → <code>getByRole</code> → <code>getByTestId</code> → <code>getByLabelText</code></button>
<button class="quiz-choice" data-value="D">D) All queries are equally preferred</button>
</div>
<p class="quiz-explanation">Testing Library prioritizes queries that reflect how users interact with the UI: role (accessible) → label (form elements) → text (visible content) → testId (fallback). This encourages accessible, user-centric testing.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q8.</strong> What is Rspack and why did Ledger Live Desktop adopt it?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A CSS preprocessor that replaces Sass</button>
<button class="quiz-choice" data-value="B">B) A test framework that replaces Playwright</button>
<button class="quiz-choice" data-value="C">C) A TypeScript compiler that replaces tsc</button>
<button class="quiz-choice" data-value="D">D) A Rust-based JavaScript bundler that is webpack-compatible but significantly faster</button>
</div>
<p class="quiz-explanation">Rspack (by ByteDance) provides the same API as webpack but is written in Rust for much faster build times. It was adopted to speed up desktop app builds while keeping existing webpack configuration compatibility.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q9.</strong> In Allure reports, what creates the step-by-step execution trace?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Console.log statements in test code</button>
<button class="quiz-choice" data-value="B">B) The <code>@step</code> decorator on page object methods</button>
<button class="quiz-choice" data-value="C">C) Playwright's built-in trace recorder</button>
<button class="quiz-choice" data-value="D">D) Manual Allure API calls in each test</button>
</div>
<p class="quiz-explanation">The <code>@step</code> decorator wraps each page object method and reports its execution (name, duration, pass/fail) to the Allure reporter, creating the collapsible step tree visible in the report.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q10.</strong> What does Turborepo's caching achieve?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It caches npm packages to avoid re-downloading</button>
<button class="quiz-choice" data-value="B">B) It caches test results to skip re-running passing tests</button>
<button class="quiz-choice" data-value="C">C) It caches build outputs — if inputs haven't changed, the build is skipped and the cached output is restored</button>
<button class="quiz-choice" data-value="D">D) It caches Docker images for faster container startup</button>
</div>
<p class="quiz-explanation">Turbo hashes each task's inputs (source files, dependencies, environment). If the hash matches a previous build, it restores the cached output instead of rebuilding. This dramatically speeds up incremental builds in a 200+ package monorepo.</p>
</div>

<div class="quiz-score"></div>
</div>

---

