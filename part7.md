## Live Apps in Ledger Wallet

<div class="chapter-intro">
A Live App is not a React component. It is not a screen. It is an embedded webview, loaded from a remote URL, sandboxed behind a strict API, and versioned independently from the Wallet shell itself. Understanding Live Apps is a prerequisite to understanding Swap — and several other high-value surfaces in Ledger Wallet.
</div>

> **If you arrived here from Part 3 Chapter 3.4** (the Swap product overview), this is the deep dive that fills in the architecture you saw at a high level there. Part 3 told you "Swap is a Live App that aggregates providers"; this chapter and the next three explain *how* that Live App is loaded, deployed, configured, and tested. Cross-links back to Part 3 Ch 3.4 are scattered throughout — use them when you need to re-anchor in user-visible behavior.

### 7.1.1 What Is a Live App?

A **Live App** is a web application that renders inside Ledger Wallet (desktop and mobile) in a sandboxed webview and communicates with the Wallet through a well-defined JSON-RPC bridge called **Wallet API**.

In practice, a Live App is:

- A **separate repo** (not part of `ledger-live`) owned by the feature team
- A **separate deployment** — the Swap Live App is containerized and shipped to AWS EKS via ArgoCD; other Live Apps may run on their own infrastructure, but the pattern is always "independent of the Wallet release train"
- A **manifest entry** declared inside `ledger-live` that tells the Wallet shell: "load this URL, expose these Wallet-API permissions, show this icon, put it in that tab"
- A **sandboxed runtime** — the Live App cannot touch the device, blockchain, or accounts directly; every privileged action must go through the Wallet API

This split matters because it decouples the Live App team's velocity from the Wallet shell's velocity. Swap can release multiple times per week without waiting for a desktop release; the market team can iterate on discover surfaces without touching native code.

### 7.1.2 Why Live Apps Exist

Before Live Apps, every new feature had to ship inside `ledger-live-desktop` and `ledger-live-mobile`. That model broke down quickly:

| Constraint | Consequence |
|---|---|
| One release cadence for the whole Wallet | Feature teams blocked by Wallet release train |
| Native code changes for every feature | High risk, long review, long QA cycle |
| Shared bundle size | Every feature grows the installer |
| Shared codebase ownership | Feature teams needed desktop/mobile expertise |
| Single regulatory posture | One non-compliant feature risked the whole app |

Live Apps solve all five:

- Each app ships on its own cadence (Swap ships independently of the Wallet)
- No native code — features are standard web apps
- Bundle size in the Wallet is zero (the webview loads the app remotely)
- The feature team owns the repo, the CI/CD, and the ops
- Legal / compliance boundaries can be drawn **per Live App**, not per Wallet version (critical for MiCA)

### 7.1.3 Architecture -- How a Live App Loads

Here is the data flow when a user opens Swap from inside Ledger Wallet:

```
User taps "Swap" tab
       │
       ▼
Wallet shell reads the manifest
  for "swap" (manifest-api)
       │
       ▼
Wallet mounts a webview/iframe
  pointing at the Live App URL
  (e.g. swap-v3.apps.ledger.com)
       │
       ▼
Live App boots and calls
  window.ledgerLive.accounts.list()
       │
       ▼
Wallet API bridge intercepts the
  call, checks manifest permissions,
  returns account list
       │
       ▼
Live App shows quotes. User picks
  one. App calls
  walletApi.exchange.start()
       │
       ▼
Wallet shell drives the device
  signing flow through Speculos /
  a real device
       │
       ▼
Live App shows the "transaction
  sent" state
```

The three layers to remember:

1. **Manifest** — configuration in `ledger-live`, declares where to load the app from and what it is allowed to do
2. **Live App** — the web app itself, loaded in a webview, owns its UX
3. **Wallet API** — the JSON-RPC bridge between the two; the **only** way a Live App touches accounts, signing, or the network on behalf of the user

**What the user actually sees, in order.** The four screenshots below trace the full boot path described above — from the Discover catalog down to the moment the Live App is fully interactive. Each one is annotated with the layer (manifest, webview, Wallet API, Live App) doing the work behind the pixels.

<photo of: Discover screen showing the Swap tile>

The Discover screen is rendered by the **Wallet shell itself** — the tiles are read from the manifest service. When you see the Swap tile here, it means `ledger-live` has resolved the `swap-live-app-aws` manifest, validated it against `@ledgerhq/wallet-api-manifest-validator`, and decided this build/jurisdiction/feature-flag combination should expose Swap. Hiding the tile is *not* a Live App concern — it is a Wallet-side decision driven by Firebase `ledger-live-production` flags (see Ch 7.3.3) and the manifest's `platforms` field. If a tester reports "I don't see Swap on my build", the bug is almost always in the manifest or in the Wallet's flag layer, not in the Live App.

<photo of: Swap Live App opening — splash / loader>

The instant after the user taps the tile, the Wallet mounts a webview pointing at the manifest's `url` (e.g., `swap-live-app.ledger.com`). What you see here is **not** the Live App's own splash — it is the webview's bootstrap state. Underneath, three things happen in parallel: the webview HTTP-GETs the Next.js HTML shell from the JFrog-hosted EKS deployment, the Wallet API bridge handshakes with the page (announcing which permissions are granted), and Next.js hydrates React 19 client-side. A long pause here usually means a CDN or DNS issue at the Live App's hostname, not a bug in the swap logic.

<photo of: Swap Live App quote selector>

Once the Live App boots, it calls `walletApi.account.list()` and `walletApi.currency.list()` over the JSON-RPC bridge to populate the account picker. The user picks a from-account and an amount; the Live App then calls the **swap backend** (`swap.ledger.com` in prod — see Ch 7.3.2) to fan out a quote request to every enabled provider. The screen you are looking at is the **aggregator output** — multiple quotes, ranked, with the best-rate one preselected. The "best rate" logic lives in the backend, not the Live App; the Live App just renders what came back.

<photo of: Swap Live App provider list>

Tapping the quote header opens the full provider list. Each row is one provider's quote: rate, fees, ETA, KYC requirement, and provider type (CEX or DEX). The provider metadata (logo, name, trust badge) is fetched from `ledger-live-assets` over a CDN; the quote payload itself is from the swap backend. This is the screen where the **CEX/DEX boundary becomes visible to the user** — and the screen where the next two subsections start to matter, because picking a DEX provider triggers the token approval gating step that picking a CEX one would not.

### 7.1.4 The Manifest API

Manifests live inside `ledger-live`. They declare how each Live App is exposed inside the Wallet — which URL to load per environment, which Wallet-API permissions to grant, which icon to show, which platforms it supports.

The manifest service is consumed by both desktop and mobile. For each Live App there is typically one manifest per environment.

**Manifest schema (from the Wallet API spec).** Every manifest validated by `@ledgerhq/wallet-api-manifest-validator` carries this shape:

| Field | Type | Purpose |
|---|---|---|
| `id` | `string` | Manifest unique id (e.g., `swap-live-app-aws`) |
| `name` | `string` | Display name shown in the Wallet (e.g., `"Swap"`) |
| `url` | `string` | The URL the Wallet loads in the webview |
| `homepageUrl` | `string` | Canonical homepage for the app |
| `platform` / `platforms` | `"desktop" \| "mobile" \| "all"` or array | Which Wallet shells load this app |
| `apiVersion` | `string` | Wallet-API semver range the app expects (e.g., `^2.0.0`) |
| `manifestVersion` | `string` | Manifest schema version |
| `branch` | `"stable" \| "experimental" \| "soon" \| "debug"` | Stability channel exposed to users |
| `categories` | `string[]` | Catalog tabs (e.g., `["exchange"]`) |
| `currencies` | `string[]` or `"*"` | Which currencies the app supports |
| `content.shortDescription` / `content.description` | `TranslatableString` | Copy shown in the catalog |
| `permissions` | `AppPermission[]` | The **capability allow-list** — see below |
| `domains` | `string[]` | Allowed navigation domains inside the webview |
| `private` | `boolean` (optional) | Hides the app from the public catalog |

**Real permission list (extract from the Swap dev manifest** — `apps/live-app/manifests/manifest.dev.json`**):**

```json
{
  "id": "swap-live-app-aws",
  "name": "Swap DEV",
  "url": "http://localhost:3000",
  "platforms": ["android", "ios", "desktop"],
  "apiVersion": "^2.0.0",
  "branch": "experimental",
  "categories": ["exchange"],
  "currencies": "*",
  "permissions": [
    "account.list",
    "account.request",
    "currency.list",
    "custom.exchange.start",
    "custom.exchange.complete",
    "custom.exchange.swap",
    "device.exchange",
    "device.transport",
    "exchange.start",
    "exchange.complete",
    "transaction.sign",
    "transaction.signAndBroadcast",
    "wallet.capabilities",
    "wallet.userId"
  ]
}
```

Every permission maps to one or more JSON-RPC methods the Live App can invoke. Anything not whitelisted is denied at the bridge layer, even if the Live App tries to invoke it — which is the Wallet's defence against a compromised or malicious app iframe.

**One manifest per environment.** The URL field differs by environment; everything else stays structurally similar:

| Environment | Manifest | Live App URL (public) |
|---|---|---|
| **Local dev** | local override (`manifest.dev.json` in the repo) | `http://localhost:3000` |
| **Staging** | `manifest-api` staging | `swap-live-app-stg.ledger-test.com` |
| **Pre-prod** | `manifest-api` pre-prod | `swap-live-app-ppr.ledger-test.com` |
| **Production** | `manifest-api` prod | `swap-live-app.ledger.com` |

### 7.1.5 The Wallet API

Wallet API is the [**JSON-RPC 2.0**](https://www.jsonrpc.org/specification) contract between the Wallet shell and any embedded Live App. The transport is **`postMessage`** between the parent window (the Wallet) and the iframe (the Live App) — not HTTP, not WebSocket. This matters: there is no network round-trip for a Wallet-API call, but the Wallet still enforces every permission inside the shell.

Wallet API is split across a **Client** (bundled inside the Live App) and a **Server** (hosted by the Wallet shell):

```
  ┌─────────────────┐      postMessage      ┌──────────────────┐
  │   Live App      │ ◄──── JSON-RPC ────►  │  Wallet shell    │
  │  (webview)      │         2.0           │   (Ledger Live)  │
  │                 │                       │                  │
  │ wallet-api-     │                       │ wallet-api-      │
  │ client          │                       │ server           │
  └─────────────────┘                       └──────────────────┘
@ledgerhq/wallet-api-client    │          @ledgerhq/wallet-api-server
                               │
                   @ledgerhq/wallet-api-core
                   (shared types and errors)
```

**Canonical JSON-RPC methods (from `/spec/rpc/README.md` in the Wallet API repo).** These are the calls a Swap Live App relies on:

| Method | Purpose |
|---|---|
| `transaction.sign` | User signs a transaction on the device — returns the raw signed tx |
| `transaction.broadcast` | Broadcasts a previously signed transaction — returns the tx hash |
| `message.sign` | User signs an EIP-191 / EIP-712 message on the device |
| `account.list` | List accounts the user has in the Wallet |
| `account.request` | Prompt the user to pick (or create) an account for the Live App |
| `account.receive` | Show a "Verify on device" address for an account |
| `currency.list` | List currencies supported in the Wallet |
| `exchange.start` | Start an exchange (swap/buy/sell) flow — returns a `deviceTransactionId` |
| `exchange.complete` | Finish the exchange after backend payload is retrieved — device signs |
| `wallet.capabilities` | Query which Wallet-API features the host supports |
| `wallet.userId` | Get an opaque, stable per-Wallet user identifier |

A minimal request / response pair, verbatim to the spec:

```json
// request
{ "jsonrpc": "2.0", "id": 1, "method": "transaction.sign",
  "params": { "accountId": "...", "transaction": { }, "params": { "useApp": "Ethereum" } } }
// response
{ "jsonrpc": "2.0", "id": 1, "result": { } }
```

**Canonical error types (from `/spec/core/errors.md`).** Every Wallet-API error is a `PlatformError` (`title`, `description`, `errorCode`) wrapped in the JSON-RPC 2.0 error envelope. Four error codes matter for swap QA:

| Code | Title | When QA sees it |
|---|---|---|
| `100` | `AccountNotFound` | The Live App passed an `accountId` the Wallet cannot match — usually a stale fixture or a currency the Wallet has not added |
| `101` | `AccountNotMain` | A sub-account was used where a main account was expected — common mistake when a test targets a token account by mistake |
| `102` | `AccountAndTransactionNotLinked` | The transaction object is for a different currency family than the account — the single most frequent "I swear I typed the right thing" error in fixture-driven tests |
| `103` | `TransactionNotProvided` | The transaction object is missing or malformed — often after a build that dropped a required field |

**Two things to internalize as a QA:**

1. **Everything is audited at the bridge** — if a Wallet-API call is rejected, the Wallet shell returns a `PlatformError` with a clear code. Grep for the error code and title before assuming a code bug; the manifest `permissions` list is usually where the truth lives.
2. **The Wallet API is versioned** — manifests declare `apiVersion` (e.g., `^2.0.0`). The Wallet shell and the Live App can evolve independently but the version range must overlap, so cross-version testing matters when either side bumps.

### 7.1.6 Live Apps in the Ledger Wallet Catalog

The Wallet ships several production Live Apps. Each has its own repo, team, Firebase, and release cycle. The most important ones for QA:

| Live App | What it does | Team | Where it lives |
|---|---|---|---|
| **Swap** | Crypto-to-crypto swap across multiple providers (CEX + DEX) | Swap | Swap tab / CTA from account |
| **PTX (Buy/Sell)** | Buy and sell crypto with fiat through MoonPay / Coinify / Banxa | PTX | Buy / Sell buttons |
| **Earn** | Staking and yield across supported chains | Earn | Earn tab |
| **Discover** | Third-party Live Apps (wallet-connect, NFT marketplaces, etc.) | Discover | Discover tab |
| **MoonPay (Buy)** | MoonPay's own Live App, wrapped via PTX | PTX + MoonPay | Buy flow |
| **Recover / Ledger Sync** | Account recovery and cross-device sync | Recover | Settings |

All of them follow the same pattern: separate repo, manifest entry, Wallet-API permissions, sandboxed webview. The complexity hides in the business logic, the partner integrations, and the legal constraints — which is exactly where QA spends its time.

### 7.1.7 QA Implications of the Live App Model

The Live App model changes how QA thinks about bugs:

- A bug in the Wallet shell (device flow, account sync, wallet-api bridge) affects **all** Live Apps → reproduce on at least two Live Apps before blaming the shell
- A bug in the Live App (quote rendering, provider selection, KYC flow) is **scoped** to that repo → open the ticket in the Live App's Jira project, not in the Wallet's
- An end-to-end test that opens a Live App has to wait for the webview to load; this is slower and flakier than intra-Wallet navigation — budget for it
- Release windows are **per Live App** → a swap regression does not require a Wallet hotfix; the swap team rolls back swap
- Firebase feature flags can live in two projects at once — one for the Wallet, one for the Live App — and a QA session may need to coordinate both (see Chapter 7.3)

<div class="chapter-outro">
<strong>Key takeaway:</strong> A Live App is a remote, sandboxed web app wired into Ledger Wallet through a manifest and the Wallet API. Each Live App has its own repo, deployment, Firebase, and release cadence — which is why Swap QA work rarely happens inside <code>ledger-live</code> alone. The Wallet shell is the runtime; the Live App is the product. Part 3 Ch 3.4 is the user-facing companion to this chapter — when you need to re-anchor in "what does this look like to the person tapping the screen", Part 3 Ch 3.4 is the round-trip back.
</div>

### 7.1.8 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz -- Live Apps in Ledger Wallet</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> What is a Live App, architecturally?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A native screen inside <code>ledger-live-desktop</code></button>
<button class="quiz-choice" data-value="B">B) A plugin DLL loaded into the Electron main process</button>
<button class="quiz-choice" data-value="C">C) A remote web application loaded into a sandboxed webview and talking to the Wallet over the Wallet API</button>
<button class="quiz-choice" data-value="D">D) A background worker running on the device</button>
</div>
<p class="quiz-explanation">A Live App is a remote web app embedded in a sandboxed webview. It never runs native code. Everything privileged (accounts, signing, network) is brokered by the Wallet API bridge.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> Which statement about Live App releases is true?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A new Live App version requires a new Ledger Wallet release</button>
<button class="quiz-choice" data-value="B">B) Each Live App releases independently from the Wallet shell, on its own cadence</button>
<button class="quiz-choice" data-value="C">C) Live Apps are rebuilt on every Wallet CI run</button>
<button class="quiz-choice" data-value="D">D) Live Apps are stored inside the installer and updated via Electron auto-updater</button>
</div>
<p class="quiz-explanation">Decoupled releases are the main reason Live Apps exist. Swap, for example, can ship several times a week without Wallet desktop or mobile releasing at all. The manifest points at a URL that the Live App team updates independently.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> The Wallet API is best described as...</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A JSON-RPC bridge that brokers every privileged call a Live App needs to make, with permissions enforced at the bridge</button>
<button class="quiz-choice" data-value="B">B) A REST API hosted by Ledger's backend</button>
<button class="quiz-choice" data-value="C">C) A WebSocket used only for streaming market prices</button>
<button class="quiz-choice" data-value="D">D) A shared library linked into the Live App</button>
</div>
<p class="quiz-explanation">Wallet API is an in-process JSON-RPC bridge between the webview and the shell. The shell enforces manifest permissions at the bridge — a Live App cannot bypass them even if it tries.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> Why are manifests important from a QA perspective?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) They contain the Live App's source code</button>
<button class="quiz-choice" data-value="B">B) They replace feature flags</button>
<button class="quiz-choice" data-value="C">C) They are loaded only in production</button>
<button class="quiz-choice" data-value="D">D) They control which URL the Wallet loads per environment and which Wallet-API permissions the app has — a misconfigured manifest can break an otherwise correct Live App</button>
</div>
<p class="quiz-explanation">If the manifest points at the wrong URL or denies a required permission, the Live App will fail even if its own code is flawless. When a Live App is broken in one environment only, always compare manifests first.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> A webview-based Live App is slower to load than a native screen. What does this mean for E2E test design?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Use <code>page.waitForTimeout(5000)</code> before every action</button>
<button class="quiz-choice" data-value="B">B) Budget for webview boot time and Wallet-API handshake in fixture/setup, not inside assertions — and prefer deterministic waits on app-specific selectors over sleeps</button>
<button class="quiz-choice" data-value="C">C) Never test Live Apps end-to-end</button>
<button class="quiz-choice" data-value="D">D) Disable webviews in test mode</button>
</div>
<p class="quiz-explanation">Live Apps need deterministic waits on app-specific UI elements (e.g., the quotes list) because the boot adds latency the Wallet shell does not have. Fixed sleeps create flakiness; Playwright's auto-waiting on a real element is what you want.</p>
</div>

<div class="quiz-score"></div>
</div>

---


## Swap Live App -- Architecture & Components

<div class="chapter-intro">
The Swap Live App is the most business-critical Live App in Ledger Wallet. It is also the most architecturally complex — several repos, several backends, several providers, and a regulated perimeter. This chapter is the map you need to debug a swap issue without guessing where the problem lives.
</div>

### 7.2.1 What Swap Does

From the user's point of view, Swap exchanges one crypto for another directly from Ledger Wallet without leaving the app and without giving custody to anyone. Under the hood, "directly from Ledger Wallet" hides a lot of moving parts:

1. The user picks a source account and a destination currency
2. The Swap Live App asks multiple providers for quotes in parallel
3. The user picks a provider (or the best-rate pick is preselected)
4. The Wallet shell builds a transaction on the source chain, the device signs it, the Wallet broadcasts it
5. The provider receives the source asset, converts it, and sends the destination asset to the user's destination account
6. Swap polls the provider for status until the trade reaches a terminal state (completed, failed, refunded)

Ledger does not hold funds at any step. The provider takes custody briefly between inbound and outbound; Ledger's role is limited to quote aggregation, transaction orchestration, and status reporting. That boundary is a legal one — see 33.9 for why.

### 7.2.2 The Swap Repo Landscape

Swap lives in several repos. You will need to know each one by name.

| Repo | What it is | Language / stack | QA interest |
|---|---|---|---|
| **`swap-live-app`** | The Swap Live App itself — the webview UI users see | Next.js 15 (App Router) + React 19 + TypeScript 5.9 + Tailwind 4; containerized and deployed to AWS EKS via ArgoCD | Most frontend bugs live here |
| **`swap`** | The Swap backend: quote aggregator + completer (polls providers and updates status) | Scala (service) + Scala (completer), PostgreSQL, RabbitMQ | Quote discrepancies, status-polling issues, provider outages surface here |
| **`swap-configuration`** | Configuration for supported pairs, limits, fees, provider enablement | Rust + CSV configuration files | "Why is this pair unavailable?" almost always starts here |
| **`exchange-sdk`** | Shared TypeScript SDK used by Live Apps (Swap, PTX) and by the Wallet shell to drive exchange flows | TypeScript | Wallet-API exchange contract lives here |
| **`wallet-api`** | The JSON-RPC bridge itself — `client`, `server`, `core`, `simulator`, `manifest-validator` packages | TypeScript monorepo | Manifest schema, RPC methods, error types are defined here |
| **`ledger-live`** (this repo) | The Wallet shell, including E2E tests for swap | pnpm + Turborepo monorepo | E2E desktop (`send.swap.spec.ts`, `accounts.swap.spec.ts`) and mobile tests |
| **`ledger-live-assets`** | Static assets: provider logos, provider configs, icons | Static CDN-served | Missing logo, wrong provider label → check here |

A **swap bug does not always belong to Swap** — it might live in `ledger-live` (the shell), in `exchange-sdk` (the bridge contract), in `wallet-api` (the protocol itself), or in the provider. Triage by layer.

### 7.2.3 Frontend: the `swap-live-app` Repo

`swap-live-app` is a **Next.js 15 (App Router) + React 19 + TypeScript 5.9 + Tailwind 4** monorepo. It is **not** deployed on Vercel — the application is containerized by GitHub Actions (`build.yml`), pushed to a JFrog OCI registry (`jfrog.ledgerlabs.net/ptx-oci-prod-green/swap-live-app`), and deployed to **AWS EKS via ArgoCD** (GitOps). Chapter 7.3 covers the environments and URLs.

**Repo layout** (from `apps/docs/architecture/index.md`):

```
swap-live-app/
├── apps/
│   ├── live-app/           # Main swap application (Next.js 15, App Router)
│   └── docs/               # Documentation site (VitePress)
├── packages/
│   ├── ui-desktop/         # Design system components (Atomic Design)
│   ├── formatter/          # Number/currency formatting with i18n
│   ├── utils/              # Shared utilities (cn helper)
│   ├── testing-tools/      # Testing utilities and MSW setup
│   ├── window-transport-simulator/   # Wallet API transport simulator
│   ├── eslint-config/      # Shared ESLint configurations
│   ├── typescript-config/  # Shared TypeScript configurations
│   └── tailwind-config/    # Shared Tailwind CSS configuration
├── argocd/                 # ArgoCD deployment configs (stg, ppr, prd, prx)
├── .github/workflows/      # CI/CD (build, e2e, release, rollback, code freeze)
└── .rules/                 # Project-wide coding standards
```

**Inside `apps/live-app/src/`:**

```
src/
├── app/             # Next.js App Router pages & layouts
├── components/      # Feature and page-specific components
├── context/         # React contexts (Wallet API, user state)
├── hooks/           # Custom React hooks
├── locales/         # i18n strings
├── queries/         # React Query API calls to the swap backend
├── remote-config/   # Firebase remote config (feature flags, partner config)
├── schemas/         # Zod schemas for runtime validation
├── state/           # Jotai atoms
├── swap-api/        # Typed client for the `swap` backend
├── wallet-api/      # Thin wrapper around @ledgerhq/wallet-api-client
└── middleware.ts    # Next.js edge middleware
```

**Key stack traits to remember as QA:**

- **Next.js App Router** (not Pages Router) — URL routing is filesystem-based under `src/app/`
- **Styling** — Tailwind 4 + Ledger's Lumen design system (`@ledgerhq/lumen-design-core`, `@ledgerhq/lumen-ui-react`) + `@ledgerhq/react-ui`. Tailwind utilities come from a custom preset — QA should not be tempted to read arbitrary class values like `w-[107px]`; only design-token sizes are valid
- **State** — Jotai atoms (not Redux/Zustand) for local state; React Query for backend calls
- **Feature flags** — Firebase Remote Config (see chapter 34)
- **UI component library** — `@workspace/ui-desktop` is the in-repo design system, organised Atomic-Design style (atoms / molecules / organisms / primitives), documented via Storybook
- **Simulator** — `@workspace/window-transport-simulator` + `@ledgerhq/wallet-api-simulator` mock the Wallet-API bridge so the Live App can run standalone at `http://localhost:3000` during dev
- **Analytics / monitoring** — Datadog RUM (`@datadog/browser-rum`), Segment (`@segment/analytics-next`), Mixpanel (`mixpanel-browser`) are all bundled into `apps/live-app`

The Live App is **stateless across sessions** from the Wallet's perspective — if you reopen Swap, it reboots. This matters in tests: every `openSwap()` in an E2E spec is a full webview boot (plus Next.js hydration).

### 7.2.4 Backend: the `swap` Service and Completer

The `swap` backend is two processes sharing a PostgreSQL database and a RabbitMQ (Amazon MQ) queue:

```
              HTTPS from Live App
                     │
                     ▼
        ┌────────────────────────┐
        │       swap service      │
        │  (Scala, REST, quotes,  │
        │   trade creation)       │
        └────────┬───────────────┘
                 │ enqueue
                 ▼
        ┌────────────────────────┐
        │       RabbitMQ          │
        │      (Amazon MQ)        │
        └────────┬───────────────┘
                 │ consume
                 ▼
        ┌────────────────────────┐
        │       swap completer    │
        │  (Scala, polls provider │
        │   status, persists it)  │
        └────────┬───────────────┘
                 │
                 ▼
        ┌────────────────────────┐
        │      PostgreSQL         │
        │  (trades, statuses,     │
        │   audit)                │
        └────────────────────────┘
```

Responsibilities:

- **`swap` service** — receives quote and trade requests from the Live App, fans them out to providers, ranks quotes, returns the best, and creates the trade record
- **`swap completer`** — background worker that polls providers for long-running trades and updates status in PostgreSQL; the Live App (and Wallet) read status from the service endpoint
- **PostgreSQL** — system of record for every trade, status history, audit
- **RabbitMQ** — decouples trade creation (synchronous, user-facing) from status polling (asynchronous, provider-dependent)

The **completer is why a swap you started can "complete later"** even if you close Swap. Status is owned by the backend, not the Live App.

### 7.2.5 Configuration: the `swap-configuration` Repo

`swap-configuration` holds the declarative rules that the service and the Live App obey:

- Which swap pairs are supported, per provider
- Per-pair min/max amounts
- Per-provider fee tiers
- Which providers are enabled globally and per environment
- KYC requirements per provider
- Geolocation / jurisdiction rules

It is **Rust** tooling over **CSV** sources. QA almost never edits this repo, but you will read it constantly:

- "Why can't I swap X to Y?" → grep the pair in `swap-configuration`
- "Why is Changelly disabled on pre-prod?" → same
- "Why does the min amount differ between stg and prod?" → same

When you see a behavior that looks like a bug but might be a config intent, **check `swap-configuration` before opening a ticket**.

### 7.2.6 Shared Contracts: `exchange-sdk`

`exchange-sdk` is a TypeScript package that both the Wallet shell and the Swap Live App consume. It defines:

- The Wallet-API exchange contract (types, method signatures, error codes)
- The shape of swap quotes, trade requests, and status responses
- Helpers for building the transaction the device will sign

Bugs in `exchange-sdk` are particularly painful because they surface as mysterious errors at the Wallet-API bridge — the Live App thinks it sent a valid call, the shell thinks it received a malformed one, and the stack trace points at neither. When you see `INVALID_PARAMS` or similar bridge errors from Swap, check if `exchange-sdk` was recently bumped on either side.

### 7.2.7 Static Assets: `ledger-live-assets`

This repo serves static assets (images, provider configs) through a CDN. For Swap specifically:

- Provider logos (Changelly, CIC, 1inch, Paraswap, MoonPay, Thorswap, Uniswap, ...)
- Provider metadata (display name, URL, KYC flag)
- Icons used in the Live App UI

If a provider shows up with a broken or missing logo in prod, that is an assets repo issue, not a Swap issue.

### 7.2.8 Providers

Swap aggregates multiple providers. Each provider is either a CEX (custodial, KYC-prone) or a DEX (non-custodial, usually KYC-free). For QA, the CEX/DEX distinction drives the flow:

| Provider | Type | KYC required? | Notes |
|---|---|---|---|
| **Changelly** | CEX | Conditional | Float rates, widely supported pairs |
| **CIC** | CEX | Conditional | Partner-specific fiat off-ramp |
| **1inch** | DEX | No | EVM DEX aggregator |
| **Paraswap** | DEX | No | EVM DEX aggregator |
| **MoonPay** | CEX (fiat-on-ramp, via Swap on some flows) | Yes | Different flow — often has its own Live App |
| **Thorswap** | DEX | No | Cross-chain (BTC, ETH, LTC, ...) |
| **Uniswap** | DEX | No | EVM, canonical DEX |

Two things to remember in tests:

- **KYC path diverges**: `selectExchangeWithoutKyc` picks a provider the test can actually go through without a human. Using `selectExchange` (including KYC-capable providers) may land on Changelly/CIC and block the test behind a KYC screen.
- **DEX swaps need token approvals** before the actual swap — `app.swap.ensureTokenApproval(...)` handles this. If a test flakes on a token swap only, missing or stale approval is the first suspect.

### 7.2.9 The Regulatory Boundary (MiCA / CASP)

Since late 2024, swap in the EU falls under MiCA. Two consequences matter for QA:

- Ledger operates Swap under an **RTO** (Reception and Transmission of Orders) model. Ledger does not custody funds and does not match orders — providers do.
- Provider availability varies **by user jurisdiction**. A pair that works in stg from a French test IP may not work from a US-VPN test session.

This is why QA environments have stg/ppr/prod split, and why the list of enabled providers differs across envs (see Chapter 7.3).

### 7.2.10 Putting It All Together

The end-to-end exchange flow involves four actors — the **Live App** (frontend), the **Exchange SDK** (shared contracts), the **Wallet API** (JSON-RPC bridge exposed by the Wallet shell), and the **Swap backend** (service + completer). The sequence below is the canonical happy path for a swap trade:

![Swap Live App sequence diagram — Live App, Exchange SDK, Wallet API, and Swap Backend](images/Swap-live-app-schema.png)

In words, step by step:

1. The Wallet shell reads the manifest and opens the Live App URL
2. The Live App boots, uses `exchange-sdk` types to request quotes from the `swap` service
3. The service consults `swap-configuration` to know which providers and pairs are enabled
4. The service hits each enabled provider in parallel
5. The Live App shows quotes, the user picks one
6. The Live App asks the Wallet API (via `exchange.start`) to initialize an exchange nonce on the device
7. The Live App then calls `exchange.complete`; the Wallet shell builds the tx, the device signs, and the Wallet broadcasts
8. The `swap` service writes the trade record; the **completer** polls the provider until the trade reaches a terminal state
9. The Live App queries status until completion; logos and provider metadata come from `ledger-live-assets`

**When you triage a swap bug, ask by layer, top to bottom:**

- Live App UX / rendering → `swap-live-app`
- Wallet-API bridge / shell device flow → `ledger-live` + `exchange-sdk`
- Quote rejection, wrong pair availability, stale status → `swap` backend
- Supported pair / min-max / provider enablement → `swap-configuration`
- Missing logo / wrong provider label → `ledger-live-assets`
- Provider-specific failure → the provider itself (escalate)

### 7.2.11 Token Approval -- the Gating Step Before Any DEX Swap

Everything in 6.2.10 assumed a happy-path "user clicks, device signs, swap completes". For a swap **on a DEX** that involves an **ERC-20 token** as the source asset, that path has an extra step that does not exist for native-coin or CEX swaps: the user must first authorize the DEX router contract to move their tokens. This section is the QA-focused walkthrough of that gating step — what it is, when it fires, what the user and the device see, and how the QAA framework simulates it deterministically in regression.

#### What an approval is

In the EVM world, an ERC-20 **approval** is an on-chain authorization that says: "this spender contract is allowed to move up to this amount of *my* tokens out of *my* wallet". The token contract stores it as a mapping `allowance[owner][spender] = amount`. Without that allowance, any attempt by the spender to call `transferFrom(owner, recipient, amount)` reverts.

This is the same primitive Part 6 Chapter 6.2 introduced when discussing manual debug tools (revoke.cash, the Wallet Inspector); cross-link there if you need a refresher on what the on-chain state actually looks like. The mechanism is universal — every ERC-20 supports it, every DEX uses it.

> **Note:** The approval is **per (owner, spender, token)** triple. Approving Uniswap V3 router on USDC does **not** approve 1inch on USDC, and does **not** approve Uniswap on USDT. Each (provider, token) pair is its own allowance row. This matters for tests, because seeding one allowance does not magically cover a sibling test that picks a different provider.

#### When it triggers — the CEX/DEX boundary

The gating step only exists for **DEX providers**. The boundary is sharp:

| Provider category | Examples | Token approval needed? | Why |
|---|---|---|---|
| **DEX** | Thorchain, Uniswap, LiFi, ParaSwap, 1inch, Velora | **Yes** | The provider is a smart contract; it must `transferFrom` your tokens, which requires a prior `approve` |
| **CEX** | Changelly, Exodus | **No** | The provider is a custodial off-chain service; you send tokens to a deposit address with a normal `transfer` (no allowance involved) |

For the tester, the practical consequence is: a swap test that targets a CEX provider runs without an approval phase, while the same test re-routed to a DEX picks up an extra device-signing screen and an extra on-chain transaction before the swap itself can be submitted. This is the single most common reason a swap test "works on Changelly but flakes on Uniswap" — the DEX path has more moving parts, and the approval is one of them.

#### The user-facing experience

When a DEX provider is selected and the source token has insufficient allowance, the Swap Live App inserts an "approve" step before the swap step. The user sees two device prompts in a row instead of one — first to authorize the spender, then to execute the swap.

<photo of: Swap Live App showing "approve USDC for Uniswap" step>

The Live App renders a dedicated screen that names the **token** (USDC), the **spender** (Uniswap router), and the **amount** the approval will cover. Modern Live App builds default to "exact amount" (the swap amount, not unlimited) — this is a security-conservative choice that costs an extra approval per swap but limits the blast radius if the spender is ever compromised. The user can typically toggle to "unlimited" if they swap the same token repeatedly and want to save gas; the QAA framework currently mirrors whatever the Live App offers as the default and does not exercise the unlimited path in regression.

<photo of: Device review screen for the approve transaction>

On the device, the approve transaction looks like any other EVM transaction: a contract call with `data: 0x095ea7b3 + spender + amount` (the function selector for `approve(address,uint256)` followed by the two ABI-encoded arguments). The device displays the contract address, the function name (resolved against Ledger's clear-signing plugin if available, otherwise shown as raw calldata), and the gas. Once the user confirms, the Wallet shell broadcasts the tx; the Live App polls until it sees the allowance increase on-chain, then proceeds.

<photo of: Swap Live App proceeding to the actual swap step after approval>

Once the allowance is confirmed, the Live App transitions out of the approval state and into the regular swap-execution screen. From here on the flow is identical to a non-token swap — same exchange-SDK calls, same `exchange.start` / `exchange.complete`, same device signing dance described in 6.2.10. The approval is purely additive; it does **not** change the rest of the flow, only gates entry to it.

#### The technical path -- it is just an EVM transaction

There is no special Wallet-API method for approvals. The Live App builds an ordinary EVM transaction with these fields:

```
to:       <ERC-20 token contract address>   // e.g., USDC contract
value:    0
data:     0x095ea7b3                          // approve(address,uint256) selector
        + 000…<spender address>             // 32-byte ABI-encoded spender
        + 000…<amount>                       // 32-byte ABI-encoded amount (or 2^256-1 for unlimited)
gas:      ~50_000 (typical)
```

It then hands this transaction to the Wallet shell via the regular `walletApi.transaction.sign` (or the equivalent `exchange.complete`-style call when the approval is being driven by the exchange flow). The shell drives the device, the device returns a signature, the shell broadcasts. From the Wallet's perspective the approve is **an EVM transaction like any other** — the only thing that distinguishes it is the calldata, and the only thing the user notices is that the device shows a different summary screen.

> **Note:** This is also why an approval **persists across Live App reboots**. The state is on-chain, recorded against `(owner, spender, token)` in the token contract's storage. If you approve USDC for Uniswap on Monday, close Ledger Live, reopen on Friday, and start a new swap with the same provider and same token, the Live App will detect the existing allowance and skip the approval step entirely. This is desirable for users (less friction, less gas) but inconvenient for QA, because it means **a previous run can leave state that hides the approval flow on the next run**. The next subsection (6.2.12) is exactly about the cleanup mechanism that solves this.

#### The QAA test path — `ensureTokenApproval`

In the QAA desktop framework, the swap POM exposes `ensureTokenApproval` as the canonical entry point:

```typescript
// e2e/desktop/tests/page/swap.page.ts (sketch)
async ensureTokenApproval(
  fromAccount: Account | TokenAccount,
  provider: Provider,
  minAmount: string,
): Promise<void> {
  // 1. Check the current on-chain allowance via the swap backend / RPC
  const current = await this.readAllowance(fromAccount, provider);

  // 2. If allowance >= minAmount, the approval step is already covered — skip
  if (BigNumber(current).gte(minAmount)) return;

  // 3. Otherwise, pre-seed the allowance using the CLI hook
  await approveTokenCommand({
    account: fromAccount,
    spender: provider.spenderAddress,
    amount: minAmount,
  });

  // 4. Wait for the allowance to be visible on-chain (poll the token contract)
  await this.waitForAllowance(fromAccount, provider, minAmount);
}
```

The CLI hook `approveTokenCommand` lives in `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts` (the same module that hosts `liveDataWithAddressCommand`, `genTransactionCommand`, and friends — see Part 6 Ch 6.4 for the full command catalog). It builds the approve transaction the same way the Live App does, signs it via the headless CLI binding to `ledger-live-cli`, and broadcasts it to the test network — all **before** the Playwright test starts walking through the UI.

The point of `ensureTokenApproval` is **idempotence**: whether the allowance already exists from a prior run, was set by a different test, or has just been wiped by a revoke, the method guarantees that by the time it returns, the on-chain allowance is at least `minAmount` for `(fromAccount, provider, fromAccount.currency)`. From the Playwright test's perspective, the next swap step now has the precondition it needs — independently of how the previous run ended. (See Part 6 Ch 6.7 for the full lifecycle of CLI-seeded preconditions in the desktop swap fixture.)

> **Cross-link:** Part 3 Ch 3.4 introduces token approval at a high level as "the extra device prompt you see for token swaps". This is the deep dive that explains *why* it exists and *how* QA simulates it. If you arrived from there, the executive summary is: it is an ERC-20 `approve(spender, amount)` transaction signed by the device, broadcast like any other EVM tx, and the QAA framework pre-seeds it via `approveTokenCommand` so the Playwright test does not have to drive a second device prompt.

#### Pitfalls and debugging matrix

The approval step is the single most common source of "this swap test only flakes on DEX" reports. The matrix below maps observed symptoms to most-likely root causes.

| Symptom | Most-likely cause | First check |
|---|---|---|
| Test hangs at "select provider" with no error | Provider list rendered but `selectExchangeWithoutKyc` cannot find a non-KYC match | Inspect the rendered provider list; the env may have all DEX providers disabled in `swap-configuration` or in the `swap-live-app` Firebase |
| `INSUFFICIENT_ALLOWANCE` revert post-broadcast | `ensureTokenApproval` called but the broadcast either failed silently or the test moved on before the chain confirmed | Add a `waitForAllowance` poll with a timeout; check the test network's block time |
| Approval step rendered when QA expected it to be skipped | A previous run revoked, or the spender address changed (provider router redeployed) | Re-read the spender address from the provider metadata; do not hardcode |
| Approval step **not** rendered when QA expected it | Allowance from a prior run still on-chain (broadcast was on); test never entered the gating path | Confirm the env's broadcast mode; if on, make sure 6.2.12's revocation runs before the test |
| Device shows raw calldata instead of "Approve USDC for Uniswap" | The token / contract is missing from the device's clear-signing plugin | This is a device-app concern, not a swap concern; file against the relevant coin app |
| `approveTokenCommand` succeeds locally but the Live App still asks for approval | The CLI seeded the approval against a **different** spender (router upgrade, wrong env's address) | Print the spender both sides — CLI and Live App — and confirm they match |

> **Note:** `ensureTokenApproval` is **not** a side-effect-free check. When the allowance is below `minAmount`, it broadcasts (in broadcast-on mode) or simulates (in broadcast-off mode) an actual transaction. This means it consumes test gas, increments the test account's nonce, and produces an entry in the test network's block explorer. Plan accordingly when budgeting test seeds and gas.

#### Verbatim excerpt — the calldata builder

For reference, here is the canonical EVM calldata builder used by both the Live App and the CLI hook. The two paths produce **identical** bytes, which is what makes the QAA framework's pre-seeding faithful to the real user flow:

```typescript
// Conceptual — the actual implementation lives in libs/coin-modules/coin-evm
// and in the swap-live-app's transaction-builder utility.
function buildApproveCalldata(spender: string, amount: BigNumberish): string {
  const selector = "0x095ea7b3"; // keccak256("approve(address,uint256)").slice(0, 10)
  const paddedSpender = ethers.utils.hexZeroPad(spender, 32);
  const paddedAmount = ethers.utils.hexZeroPad(BigNumber.from(amount).toHexString(), 32);
  return selector + paddedSpender.slice(2) + paddedAmount.slice(2);
}
```

The selector `0x095ea7b3` is one of those four bytes you eventually memorize as a swap-domain QA — it is the fingerprint of an ERC-20 approval, the thing you grep for when scanning calldata to confirm "yes, this is an approve transaction". (For symmetry: `0xa9059cbb` is `transfer(address,uint256)` and `0x23b872dd` is `transferFrom(address,address,uint256)`. The DEX router contract is the one calling `transferFrom` against the allowance you just granted.)

### 7.2.12 Token Revocation -- Clearing Allowance State Between Regression Runs

If approval is the gating step, revocation is the cleanup step. It is the **other half** of the same `approve(spender, amount)` mechanism — only with `amount = 0`. Setting an allowance to zero clears the row in the token contract's storage and forces the next swap to go through the approval flow again. For users this is rare (they revoke when they suspect a malicious dApp); for QA it is a fundamental hygiene primitive in the regression environment.

#### The same `approve(spender, 0)` mechanism

There is no special "revoke" function in ERC-20. To revoke, you call `approve(spender, 0)`. The transaction shape is identical to an approval — same selector (`0x095ea7b3`), same two ABI arguments, same gas profile. Only the amount differs.

```
to:       <ERC-20 token contract address>
value:    0
data:     0x095ea7b3
        + 000…<spender address>
        + 000…000000                         // amount = 0
gas:      ~30_000 (storage write that clears a slot is slightly cheaper)
```

The on-chain effect: `allowance[owner][spender] = 0`. The next time the spender tries `transferFrom`, it reverts. The next time the user attempts a swap on that DEX with that token, the Live App will detect the zero allowance and re-render the approval step.

#### When it matters — broadcast-on regression vs nightly-broadcast-off

Revocation is a **lifecycle concern**, not a feature concern. It only matters when the test environment **persists state between runs** — which in practice means the regression test environments that have transaction broadcast enabled.

| Test mode | Broadcast | State persistence | Revocation needed? |
|---|---|---|---|
| Local dev / Speculos default | Off (`DISABLE_TRANSACTION_BROADCAST=1`) | None — no tx ever leaves the framework | **No** |
| Nightly (broadcast off) | Off | None | **No** |
| Manual regression on `ppr` | On (real testnet broadcast) | Allowances accumulate run after run | **Yes** |
| Pre-release validation | On | Allowances accumulate | **Yes** |

The default in `setupEnv(true)` is broadcast-off — most runs do not need revocation. But **the moment you flip broadcast on**, every approval seeded by `ensureTokenApproval` becomes persistent on-chain, and the next run will see "oh, the allowance is already there" and silently skip the approval step. That is exactly the behavior we want in *production* (less friction for end users) and exactly the behavior we *do not want* in regression (because the approval flow would never be exercised). Revocation is the lever that re-enters the gating step.

#### What the user-facing path looks like (and what it does not)

The Ledger Wallet UI **does not expose token revocation directly** as a Swap feature. There is no "revoke this allowance" button in the Swap Live App, because the Live App is purpose-built for swapping, not for allowance management. Users who want to revoke must use:

- **External tools** like [revoke.cash](https://revoke.cash) — connect the wallet, see all active allowances, send a `approve(spender, 0)` tx for the ones you want to clear (Part 6 Ch 6.3 walks through this manually as a debugging aid)
- **The Wallet Inspector** for read-only on-chain inspection (Part 6 Ch 6.3 also covers this — useful when you need to *verify* an allowance state without changing it)
- **The QAA CLI hook** — for tests, which is what 6.2.11 and the rest of this section describe

The asymmetry is intentional: approving is part of the swap UX (you have to see it to consent), while revoking is a power-user / security operation that lives in dedicated tools. From a QA perspective, that means there is no "click revoke" path to automate in the Live App — the revocation happens **outside the user-facing flow** entirely, in the test fixture's setup/teardown.

#### The QAA infrastructure -- `revokeTokenCommand` and `revokeTokenApproval`

The senior QAA's QAA-615 commit (full line-by-line walkthrough in Part 6 Ch 6.8) introduced two paired symbols:

**1. `revokeTokenCommand` in `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts`** — the CLI primitive. It is the mirror image of `approveTokenCommand`:

```typescript
// libs/ledger-live-common/src/e2e/cliCommandsUtils.ts (sketch — see Part 6 Ch 6.8 for the verbatim walkthrough)
export const revokeTokenCommand = ({
  account,
  spender,
}: {
  account: TokenAccount;
  spender: string;
}) => ({
  command: "tx",
  args: [
    "--accountId", account.id,
    "--recipient", account.token.contractAddress,
    "--data", buildApproveCalldata(spender, "0"),
    "--amount", "0",
  ],
});
```

It produces the same kind of CLI invocation `approveTokenCommand` does, with `amount = 0` baked into the calldata. The result is a transaction the headless CLI signs and broadcasts (when broadcast is on) or just signs and discards (when broadcast is off — in which case the call is a no-op for state but still useful as a smoke-test of the calldata builder).

**2. `revokeTokenApproval` in `e2e/desktop/tests/page/swap.page.ts`** — the POM method that wraps it. Sketch:

```typescript
// e2e/desktop/tests/page/swap.page.ts (sketch)
async revokeTokenApproval(
  fromAccount: TokenAccount,
  provider: Provider,
): Promise<void> {
  await revokeTokenCommand({
    account: fromAccount,
    spender: provider.spenderAddress,
  });
  await this.waitForAllowance(fromAccount, provider, "0");
}
```

The POM wrapper waits for the on-chain allowance to drop back to zero before returning, so a subsequent `ensureTokenApproval` in the same test (or in `beforeEach`) sees a clean slate and re-runs the full approval flow.

#### Where `revokeTokenApproval` is called from

Two patterns dominate, both visible in current swap specs:

1. **`afterEach` cleanup** — the test that exercised the approval path calls `revokeTokenApproval` on its way out, leaving the chain in the same state it found it. This is the safest pattern: every test owns its own setup *and* its own teardown.
2. **`beforeEach` reset** — the test calls `revokeTokenApproval` first, *then* `ensureTokenApproval`, so it does not depend on whether the previous run cleaned up. This is the pattern used in regression environments where multiple tests run in sequence and any of them might have crashed mid-flow.

The choice between the two is a function of how aggressive the regression suite is. Nightly suites with broadcast off can skip both. Manual regression on `ppr` typically uses the `beforeEach` reset pattern because it is more defensive.

> **Cross-link:** The full line-by-line walkthrough of QAA-615 — the senior QAA commit that introduced both `revokeTokenCommand` and `revokeTokenApproval`, the rationale, the test plan, and the rollout — is in Part 6 Ch 6.8. If you want to see *how* a senior QAA designs and ships a primitive like this (rather than just *what* the primitive does), Ch 6.8 is the read.

> **Cross-link:** The general lifecycle of CLI-seeded preconditions and teardown in swap fixtures is covered in Part 6 Ch 6.7. Ch 6.4 is the catalog of all `cliCommandsUtils` helpers. Ch 6.3 covers the manual debug tools (revoke.cash, Wallet Inspector) you would use to inspect or fix allowance state by hand if a test environment ends up in a stuck state.

#### When to call which — decision matrix

The two paired POM methods have overlapping use cases. The matrix below clarifies which to call when:

| Scenario | Call `ensureTokenApproval` | Call `revokeTokenApproval` |
|---|---|---|
| Test must drive the approval UI in the Live App | Yes (it pre-seeds — see note below) | Optionally before, to force the path |
| Test asserts the swap path *after* approval already exists | Yes | No |
| Test asserts the **approval flow itself** end-to-end | No (do not pre-seed) | Yes, before the test (force a clean slate) |
| Cleanup after a broadcast-on regression run | No | Yes |
| Local Speculos run, broadcast off | Optional (no-op for state) | Skip (no-op) |
| Pre-prod nightly with broadcast off | Optional | Skip |
| Manual regression on `ppr` with broadcast on | Yes (in `beforeEach`) | Yes (in `beforeEach`, before the ensure) |

> **Note:** When a test is meant to **assert the approval UI itself** (a B2CQA case dedicated to "does the approve screen render correctly"), do **not** call `ensureTokenApproval` in `beforeEach` — that would skip the very UI you are trying to assert. Instead, call `revokeTokenApproval` to guarantee a clean slate, then drive the approval through the UI in the test body, then assert.

#### Verbatim sketch — the QAA-615 commit shape

The senior QAA's commit added these symbols (sketched here; Part 6 Ch 6.8 walks through every line). The shape is intentionally minimal:

```typescript
// libs/ledger-live-common/src/e2e/cliCommandsUtils.ts (added in QAA-615)

export const approveTokenCommand = ({
  account,
  spender,
  amount,
}: {
  account: TokenAccount;
  spender: string;
  amount: string;
}): CliCommand => ({
  command: "tx",
  args: [
    "--accountId", account.id,
    "--recipient", account.token.contractAddress,
    "--data", buildApproveCalldata(spender, amount),
    "--amount", "0",
  ],
});

export const revokeTokenCommand = ({
  account,
  spender,
}: {
  account: TokenAccount;
  spender: string;
}): CliCommand => ({
  command: "tx",
  args: [
    "--accountId", account.id,
    "--recipient", account.token.contractAddress,
    "--data", buildApproveCalldata(spender, "0"),
    "--amount", "0",
  ],
});
```

The two commands are **almost identical** — only the third argument to `buildApproveCalldata` differs. This is by design: it makes the two operations symmetric in the codebase, easy to review, and trivially testable side by side. A maintenance bug fixed in one is automatically a candidate fix for the other.

```typescript
// e2e/desktop/tests/page/swap.page.ts (added in QAA-615)

async ensureTokenApproval(
  fromAccount: TokenAccount,
  provider: Provider,
  minAmount: string,
): Promise<void> {
  const current = await this.readAllowance(fromAccount, provider);
  if (BigNumber(current).gte(minAmount)) return;
  await this.cli.run(approveTokenCommand({
    account: fromAccount,
    spender: provider.spenderAddress,
    amount: minAmount,
  }));
  await this.waitForAllowance(fromAccount, provider, minAmount);
}

async revokeTokenApproval(
  fromAccount: TokenAccount,
  provider: Provider,
): Promise<void> {
  await this.cli.run(revokeTokenCommand({
    account: fromAccount,
    spender: provider.spenderAddress,
  }));
  await this.waitForAllowance(fromAccount, provider, "0");
}
```

Notice the symmetry of the POM wrappers: read state → execute command → wait for the chain to reflect the new state. The "wait for the chain" step is the unsexy but indispensable bit — without it, the next assertion would race the block confirmation and flake on slow test networks.

> **Note:** `waitForAllowance` is itself a small piece of infrastructure worth understanding. It polls the token contract's `allowance(owner, spender)` view function on a fixed interval (commonly 500ms-1s) until it sees the expected value, with a deadline (commonly 30s on testnets, longer on slower chains). When tests "flake on the approval step" without an obvious cause, the deadline is the first knob to inspect.

#### Common pitfalls

1. **Forgetting to revoke between provider switches.** If a test approves USDC for Uniswap, then a later test in the same run targets ParaSwap with USDC, the second test will *also* need its own approval — the first allowance does not transfer. This is correct behavior on-chain, but if your `beforeEach` revokes only against the previous provider, the second test will silently re-use the first's state. **Mitigation:** revoke against the *upcoming* provider in `beforeEach`, not the previous one.

2. **Spender-address drift across environments.** The Uniswap V3 router on mainnet, Goerli, and Sepolia have different addresses. A test that hardcodes the spender will pass on the env it was written against and fail on every other env. **Mitigation:** always read the spender from the provider metadata returned by the swap backend; never hardcode.

3. **Revoking a CEX provider.** A test that calls `revokeTokenApproval` against a CEX (Changelly, Exodus) is a no-op on the user side (CEX swaps do not use allowances) but the CLI will still build and broadcast a `approve(spender, 0)` transaction against whatever address you passed. If the spender does not exist as a contract, the tx is harmless but wastes gas and pollutes the test seed's history. **Mitigation:** guard the call: `if (provider.type === "DEX") await revokeTokenApproval(...)`.

4. **The "first run after a long pause" trap.** When a test environment has been idle for weeks, allowances from old runs may have been auto-cleared by the provider (some routers periodically rotate spender contracts). The test sees no allowance, runs `ensureTokenApproval`, and the approval ends up against a stale spender — the swap then fails. **Mitigation:** treat the spender address as a runtime input, fetch it fresh on every test run.

5. **Using `revokeTokenApproval` in nightly broadcast-off runs.** It is a no-op for state but still makes a CLI round-trip and a `waitForAllowance` poll that *will time out* (because the chain never changes). This adds a 30-second tax to every test for no benefit. **Mitigation:** gate the call on `process.env.DISABLE_TRANSACTION_BROADCAST !== "1"`.

#### Quick reference — full primitive surface

For QA writing a new swap test that involves an ERC-20 source, the full surface area you need is small and worth memorizing:

| Symbol | Layer | Purpose |
|---|---|---|
| `approveTokenCommand` | `cliCommandsUtils.ts` | Build the CLI invocation that signs+broadcasts a `approve(spender, minAmount)` |
| `revokeTokenCommand` | `cliCommandsUtils.ts` | Build the CLI invocation that signs+broadcasts a `approve(spender, 0)` |
| `ensureTokenApproval` | `swap.page.ts` | Idempotent: check current allowance, seed if needed, wait for chain |
| `revokeTokenApproval` | `swap.page.ts` | Force-clear the allowance, wait for chain to reflect zero |
| `readAllowance` (helper) | `swap.page.ts` | Pure read — query `allowance(owner, spender)` view function |
| `waitForAllowance` (helper) | `swap.page.ts` | Poll allowance until it matches expected value or timeout |
| `buildApproveCalldata` (helper) | `coin-evm` | Produce the 68-byte calldata payload for either approve or revoke |

Anything beyond this surface is provider-specific or out of scope — the Live App's own approval UI is rendered by the Live App, not the QAA framework, and the test's job is to drive *around* that UI deterministically, not to recreate it.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><code>swap-live-app</code> repo — the Live App itself</li>
<li><code>swap</code> repo — Scala backend (service + completer)</li>
<li><code>swap-configuration</code> repo — pair/fees/provider configuration</li>
<li><code>exchange-sdk</code> repo — Wallet-API exchange contract</li>
<li><code>ledger-live-assets</code> repo — provider logos and metadata</li>
<li><code>e2e/desktop/tests/specs/send.swap.spec.ts</code> in <code>ledger-live</code> — desktop swap E2E tests</li>
<li><code>e2e/mobile/specs/swap/</code> in <code>ledger-live</code> — mobile swap E2E tests</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Swap is an aggregator plus an orchestrator. The Live App is the UI; the <code>swap</code> service is the quote/trade API; the completer is the status worker; <code>swap-configuration</code> is the rulebook; <code>ledger-live-assets</code> supplies branding. Token approval (6.2.11) is the gating step before any DEX swap; revocation (6.2.12) is the cleanup primitive that keeps regression deterministic. A swap bug is almost always scoped to one of these layers — triaging by layer saves hours of blind debugging. Part 3 Ch 3.4 is the high-level context chapter readers landed on first; this chapter is the architecture deep dive that backs it up.
</div>

### 7.2.13 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz -- Swap Live App Architecture</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> Where does the Swap Live App's frontend source code live, and how is it deployed?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) In <code>ledger-live/apps/swap</code>, bundled with the Wallet shell</button>
<button class="quiz-choice" data-value="B">B) In the separate <code>swap-live-app</code> repo — containerized by GitHub Actions, pushed to JFrog, and deployed to AWS EKS via ArgoCD</button>
<button class="quiz-choice" data-value="C">C) In <code>exchange-sdk</code>, served from npm</button>
<button class="quiz-choice" data-value="D">D) In <code>ledger-live-assets</code>, served from a CDN</button>
</div>
<p class="quiz-explanation">The Swap Live App lives in its own repo. Deployment is <strong>not</strong> Vercel — <code>build.yml</code> builds a Docker image, pushes it to JFrog (<code>jfrog.ledgerlabs.net/ptx-oci-prod-green/swap-live-app</code>), and ArgoCD syncs it to an EKS cluster (GitOps). The Wallet shell loads the Live App by URL, declared in a manifest.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> Which component is responsible for polling providers and updating the status of long-running trades?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The Swap Live App frontend</button>
<button class="quiz-choice" data-value="B">B) The <code>swap</code> service (REST API)</button>
<button class="quiz-choice" data-value="C">C) The <code>swap</code> completer — a background worker consuming from RabbitMQ and writing status to PostgreSQL</button>
<button class="quiz-choice" data-value="D">D) The <code>exchange-sdk</code> package</button>
</div>
<p class="quiz-explanation">The completer is the status worker. It decouples "I just created a trade" (synchronous, user-facing) from "is the trade complete?" (asynchronous, provider-dependent). This is why a trade can complete even after the user closes Swap.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> A user reports that a specific swap pair is unavailable in production only. What repo should you check first?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>swap-configuration</code> — the declarative rules about which pairs are enabled per provider and per environment</button>
<button class="quiz-choice" data-value="B">B) <code>ledger-live</code></button>
<button class="quiz-choice" data-value="C">C) <code>swap-live-app</code></button>
<button class="quiz-choice" data-value="D">D) <code>ledger-live-assets</code></button>
</div>
<p class="quiz-explanation">Pair availability is driven by <code>swap-configuration</code>. Before you assume a code bug, check whether the pair is intentionally disabled or restricted for that environment.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> Why does the desktop E2E test call <code>selectExchangeWithoutKyc</code> instead of <code>selectExchange</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It is faster</button>
<button class="quiz-choice" data-value="B">B) It is the only method that works on Speculos</button>
<button class="quiz-choice" data-value="C">C) It is required by <code>exchange-sdk</code></button>
<button class="quiz-choice" data-value="D">D) To deterministically pick a provider that will not trigger a KYC step mid-test — KYC screens cannot be automated</button>
</div>
<p class="quiz-explanation">KYC-capable providers (Changelly, CIC) can route the test into an identity-verification flow that no E2E test can complete. <code>selectExchangeWithoutKyc</code> restricts the provider pool to those that allow the swap to proceed without human verification.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> A token-pair swap test flakes only on DEX providers. What is the most likely root cause?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Speculos is misconfigured</button>
<button class="quiz-choice" data-value="B">B) The ERC-20 token approval step has not been granted or is stale — DEX swaps require an approval before the swap itself</button>
<button class="quiz-choice" data-value="C">C) The completer is down</button>
<button class="quiz-choice" data-value="D">D) <code>ledger-live-assets</code> is serving stale logos</button>
</div>
<p class="quiz-explanation">DEX swaps on EVM chains require an ERC-20 <code>approve()</code> transaction before the actual swap. The test calls <code>app.swap.ensureTokenApproval(...)</code>; if that step is skipped or the approval is stale, the swap itself will fail on the first DEX provider.</p>
</div>

<div class="quiz-score"></div>
</div>

---


## Swap Live App -- Releases & Firebase Environments

<div class="chapter-intro">
The Swap Live App has its own branching model, its own CI/CD, and its own Firebase — none of which is obvious from inside <code>ledger-live</code>. This chapter maps every environment to the URLs, feature flags, and monitoring you will actually use on your first week of QA work.
</div>

### 7.3.1 Release Model -- Trunk-Based on `develop`

The Swap Live App follows **trunk-based development**. There is **no `release/*` branch** and no Gitflow: `develop` is the single source of truth, feature branches are short-lived, and production is cut from `develop` by running a workflow.

```
develop (trunk — always deployable, auto-deploys to staging)
  │
  ├── feat/<name>    (short-lived, days)
  ├── fix/<name>     (short-lived)
  └── hotfix/<name>  (branched from the latest app@vX.Y.Z tag)
```

Key rules:

- `develop` is the trunk; feature and fix branches merge back quickly
- Every merge to `develop` builds a Docker image tagged `develop` and auto-deploys to **staging** (ArgoCD sync)
- A production release is triggered manually by running the **`release-production.yml`** GitHub Action — it consumes pending **Changesets** (`.changeset/*.md`), bumps versions, commits the bump, and creates an `app@vX.Y.Z` git tag
- The tag push triggers `build.yml` which builds a Docker image tagged `app-vX.Y.Z` (hyphen, not `@`) and ArgoCD picks it up for production
- **Hotfixes** branch from the latest `app@vX.Y.Z` tag, are tagged as a new patch release, and are merged back into `develop` (do **not** cherry-pick — always merge, to avoid divergence)
- **Code freezes** are activated with the `code-freeze.yml` workflow; it flips the `CODE_FREEZE` repo variable and makes the "Code Freeze Check" fail on open PRs, blocking merges

Versioning is **fully automated** by Changesets. Before a PR that introduces a user-facing change, you run `npx changeset`, select the bump type (major/minor/patch), write a summary, and commit the generated `.changeset/*.md` with your PR. The release workflow does the rest.

> **Note:** Git tag format is `app@vX.Y.Z` (with `@`). The corresponding Docker image tag is `app-vX.Y.Z` (with a hyphen). The conversion happens automatically in the pipeline.

### 7.3.2 The Environments and Their URLs

Swap has **four** environments: three long-lived (stg, ppr, prd) and one dynamic per-PR preview (prx). Each long-lived env has an internal AWS hostname (used by Ledger-internal tooling) and a public Cloudflare hostname (the one QA and end users actually hit). Memorize the shape:

| Environment | Purpose | Frontend (Live App, public) | Internal AWS host | ArgoCD config |
|---|---|---|---|---|
| **Staging** (`stg`) | Auto-deploys on every merge to `develop`; internal dev and early QA | `swap-live-app-stg.ledger-test.com` | `swap-live-app.aws.stg.ldg-tech.com` | `argocd/stg/values.yaml` |
| **Pre-prod** (`ppr`) | Final validation — prod-like; QA sign-off | `swap-live-app-ppr.ledger-test.com` | `swap-live-app.aws.ppr.ldg-tech.com` | `argocd/ppr/values.yaml` |
| **Production** (`prd`) | End users | `swap-live-app.ledger.com` | `swap-live-app.aws.prd.ldg-tech.com` | `argocd/prd/values.yaml` |
| **Preview** (`prx`) | Dynamic per-PR environment | — | `swap-live-app-pr<NNN>.aws.stg.ldg-tech.com` | `argocd/prx/values.yaml` |

The backend (the `swap` service QA hits for quotes and trade creation) has its own URLs, versioned in the path:

| Env | Backend URL |
|---|---|
| Staging | `swap-stg.ledger-test.com/v5` |
| Pre-prod | `swap-ppr.ledger-test.com/v5` |
| Production | `swap.ledger.com` |

A few things to note:

- The Live App frontend and the swap backend are **separate services on separate URLs** — the Live App calls the backend via HTTPS. When debugging a swap issue, always check both sides
- `ledger-test.com` is Ledger's public test domain (behind Cloudflare) — do not expose these URLs to end users; they are for QA and dev only
- `ldg-tech.com` is Ledger's internal AWS domain — typically only reachable from Ledger infrastructure
- Provider enablement can differ between environments; a provider may be paused in stg to test fallback, or may be disabled in prod because of a region block
- Each `argocd/{env}/values.yaml` has two fields that control which image runs: `imageTag` (default — `develop` for stg, `main` for prd) and `IMAGE_UPDATE_OVERRIDE_TAG` (empty = automatic updates; set to `app-vX.Y.Z` to pin a version, which is how rollbacks work)

### 7.3.3 Firebase Projects

Swap uses **two separate Firebase projects**. They serve different clients and the distinction catches new QA off guard.

| Firebase project | Who reads it | What it contains |
|---|---|---|
| **`ledger-live-production`** | The Wallet shell (desktop + mobile) | Wallet-wide feature flags, Wallet 4.0 toggles, manifest overrides, the Wallet's own configuration (`shouldShowSwap`, entry-point toggles, KYC gates, etc.) |
| **`swap-live-app`** | The Swap Live App itself | Swap-specific flags: provider enablement per env, A/B experiments inside the Live App, per-pair overrides, UI experiments |

In practice:

- A flag that controls **whether the Swap entry point appears in the Wallet** is in `ledger-live-production`
- A flag that controls **the layout of the quotes list inside the Live App** is in `swap-live-app`
- If QA says "I cannot see Swap in the Wallet", check `ledger-live-production` first
- If QA says "A provider is missing in Swap", check the `swap-live-app` Firebase first, then `swap-configuration`

Each Firebase project has its own staging / prod split (typically separated by project or by remote-config condition). Credentials to these projects are distributed per-role; do not share.

### 7.3.4 Feature Flags -- Where to Look

Swap-related feature flags live in three places at once. When tracing a flag, check all three:

| Source | Examples |
|---|---|
| **Firebase `ledger-live-production`** | `ptxSwap`, `ptxEarn`, manifest overrides, Swap entry-point toggles |
| **Firebase `swap-live-app`** | Provider enablement (e.g., `changelly_enabled`), UI experiments, best-rate algorithm toggles |
| **`swap-configuration` repo** | Pair enablement, min/max amounts, per-provider pair whitelist |

Configuration wins in this order (most specific first):

1. `swap-configuration` — a disabled pair is disabled, full stop
2. Firebase `swap-live-app` — if enabled in config, a provider may still be hidden by a Firebase flag for an A/B test
3. Firebase `ledger-live-production` — if the Wallet hides the Swap entry altogether, none of the above matter

### 7.3.5 The Release Workflow in Practice

A typical swap release (no release branch, Changesets-driven):

```
Ongoing │  PRs merge to develop; each carries a .changeset/*.md if user-facing
Ongoing │  On every merge to develop: build.yml builds app image tagged `develop`
Ongoing │  ArgoCD syncs the new `develop` image to stg (swap-live-app-stg.ledger-test.com)
Day 0   │  prepare-release.yml runs → GitHub issue with next version + changelog preview
Day 0   │  Scope agreed; optionally code-freeze.yml activates CODE_FREEZE to block new merges
Day 1   │  QA regresses on ppr (the same develop-tracking image is promoted to ppr for the gate)
Day 2   │  release-production.yml triggered manually → Changesets consumed, versions bumped,
        │  commit pushed to develop, app@vX.Y.Z git tag created
Day 2   │  Tag push triggers build.yml → Docker image tagged `app-vX.Y.Z` pushed to JFrog
Day 2   │  ArgoCD prd picks up the new image; swap-live-app.ledger.com now serves the new version
Day 2   │  Monitor #ptx-swap-prod and Datadog/Sentry for regressions
```

QA's touchpoints on this timeline:

- **Ongoing (stg)** — smoke + targeted regression as PRs merge; this is where most bugs are caught
- **Day 1 (ppr)** — full regression against production-like data; this is the go/no-go gate
- **Day 2 (prod)** — watchdog on Datadog, Sentry, Mixpanel dashboards for the first 2 hours after deploy

**Rollback:** to roll back production, run `rollback-production.yml` with the environment (`prd`) and the target version (e.g., `app@v1.8.0`). The workflow updates `IMAGE_UPDATE_OVERRIDE_TAG` in `argocd/prd/values.yaml` and commits it to `develop`; ArgoCD syncs within minutes. If the workflow is unavailable, edit `argocd/prd/values.yaml` directly, set `IMAGE_UPDATE_OVERRIDE_TAG: "app-v1.8.0"`, and merge to `develop` — same effect.

### 7.3.6 Monitoring and Observability

When a swap bug is reported, the monitoring stack is where you start:

| Tool | What it shows |
|---|---|
| **Datadog** | Service health, error rates, latency, alerts for the `swap` backend and completer |
| **Sentry** | Client-side errors in the Live App (desktop webview + mobile webview) |
| **Mixpanel** | Funnel metrics — how many users start a quote, how many complete a trade, drop-off per step; there are separate Mixpanel projects for desktop and mobile |
| **Tableau** | Aggregated business metrics; revenue per provider, volume per pair, KYC pass-through |
| **Runscope** | Synthetic uptime checks against the `swap` service endpoints |

And Slack is where the humans are:

| Channel | Purpose |
|---|---|
| **`#ptx-swap-build`** | Swap team engineering chatter, release announcements |
| **`#ptx-swap-prod`** | Production-only: deploys, rollbacks, first-hour watchdog |
| **`#alerts-swap`** | Automated alerts from Datadog and Sentry |
| **`#consumer-service-alerts`** | Broader consumer-backend alerts that sometimes affect Swap |

### 7.3.7 What Changes Across Environments -- Cheat Sheet

| Concern | Staging | Pre-prod | Production |
|---|---|---|---|
| Backend URL | `swap-stg.ledger-test.com/v5` | `swap-ppr.ledger-test.com/v5` | `swap.ledger.com` |
| Frontend URL (public) | `swap-live-app-stg.ledger-test.com` | `swap-live-app-ppr.ledger-test.com` | `swap-live-app.ledger.com` |
| Frontend URL (AWS internal) | `swap-live-app.aws.stg.ldg-tech.com` | `swap-live-app.aws.ppr.ldg-tech.com` | `swap-live-app.aws.prd.ldg-tech.com` |
| Image tag | `develop` (auto) | pinned `app-vX.Y.Z` | pinned `app-vX.Y.Z` |
| Real funds | No (test seeds) | No (test seeds) | Yes — user funds |
| Provider list | May be a subset, may include test providers | Same set as prod | Full production set |
| Firebase project audience | Internal | Internal | External users |
| Monitoring severity | Low | Medium | Critical — paging |
| Broadcast in E2E | Gated by `DISABLE_TRANSACTION_BROADCAST=1` | Same | Never run E2E that broadcasts |

The most common "works on stg, fails on ppr" pattern is a provider-enablement delta — always re-read `swap-configuration` and the `swap-live-app` Firebase flags side by side before assuming a code regression.

<div class="chapter-outro">
<strong>Key takeaway:</strong> Swap ships via <strong>trunk-based development</strong> — <code>develop</code> is the source of truth, Changesets drives versioning, and <code>release-production.yml</code> creates an <code>app@vX.Y.Z</code> tag that ArgoCD then syncs to EKS. Four environments (<code>stg</code> / <code>ppr</code> / <code>prd</code> / <code>prx</code>) each have their own public Cloudflare URL (e.g., <code>swap-live-app.ledger.com</code>) and internal AWS URL. Two Firebase projects split the flag responsibility: <code>ledger-live-production</code> (Wallet-side) and <code>swap-live-app</code> (Live-App-side). When a swap bug depends on environment, the answer is almost always in the URL map, the Firebase flags, or <code>swap-configuration</code> — and often in all three at once.
</div>

### 7.3.8 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz -- Swap Releases & Firebase</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> Which branching model does <code>swap-live-app</code> use?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Classic Gitflow with a <code>release/app@v1.3.xx</code> branch cut from <code>develop</code></button>
<button class="quiz-choice" data-value="B">B) Trunk-based on <code>develop</code> — short-lived <code>feat/*</code> and <code>fix/*</code> branches merge back quickly; production is cut from <code>develop</code> by running <code>release-production.yml</code>, which consumes Changesets and creates an <code>app@vX.Y.Z</code> tag</button>
<button class="quiz-choice" data-value="C">C) Semantic-release on every merge to <code>main</code></button>
<button class="quiz-choice" data-value="D">D) One long-lived release branch per quarter</button>
</div>
<p class="quiz-explanation">The repo is <strong>trunk-based</strong>: <code>develop</code> is the single source of truth. There is no Gitflow, no <code>release/*</code> branch, and no separate <code>main</code>-as-trunk. Versions are managed by <strong>Changesets</strong> and the <code>release-production.yml</code> workflow; a production release is an <code>app@vX.Y.Z</code> tag cut directly from <code>develop</code>.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> You need to toggle a feature flag that controls whether the <strong>Swap tab appears in the Wallet at all</strong>. Which Firebase project owns that flag?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>swap-live-app</code></button>
<button class="quiz-choice" data-value="B">B) <code>ledger-live-production</code> — flags that affect the Wallet shell (including whether a Live App entry point is shown) live in the Wallet's Firebase</button>
<button class="quiz-choice" data-value="C">C) Neither — only <code>swap-configuration</code></button>
<button class="quiz-choice" data-value="D">D) A shared third Firebase project</button>
</div>
<p class="quiz-explanation">Wallet-side flags (including entry-point visibility, Wallet 4.0 toggles, manifest overrides) live in <code>ledger-live-production</code>. Flags inside the Live App UI itself (provider enablement in the Live App, UI experiments) live in the <code>swap-live-app</code> Firebase.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> What is the pre-prod backend URL for swap, and how is it different from the Live App frontend URL?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Backend: <code>swap-ppr.ledger-test.com/v5</code>; Frontend: <code>swap-live-app-ppr.ledger-test.com</code> (two separate services)</button>
<button class="quiz-choice" data-value="B">B) Backend and Frontend are both <code>swap.ledger.com</code></button>
<button class="quiz-choice" data-value="C">C) Backend: <code>swap-stg.ledger-test.com/v5</code>; Frontend: <code>swap-live-app-stg.ledger-test.com</code> (those are staging, not pre-prod)</button>
<button class="quiz-choice" data-value="D">D) Backend: <code>swap-live-app.aws.ppr.ldg-tech.com</code>; Frontend: <code>swap-ppr.ledger-test.com/v5</code> (swapped roles)</button>
</div>
<p class="quiz-explanation">Pre-prod backend is <code>swap-ppr.ledger-test.com/v5</code>; pre-prod frontend Live App is <code>swap-live-app-ppr.ledger-test.com</code>. They are <strong>separate services on separate URLs</strong> — the Live App calls the backend over HTTPS. The <code>/v5</code> in the backend path is the backend API version, independent from the Live App version.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> A pair that works on staging fails silently in pre-prod. Where do you look first?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Restart Speculos</button>
<button class="quiz-choice" data-value="B">B) Redeploy the Live App</button>
<button class="quiz-choice" data-value="C">C) Check the user's seed phrase</button>
<button class="quiz-choice" data-value="D">D) Compare <code>swap-configuration</code> values (pair enablement, min/max) and the <code>swap-live-app</code> Firebase flags between stg and ppr — environment deltas usually explain it</button>
</div>
<p class="quiz-explanation">"Works in stg, fails in ppr" is almost always a configuration delta, not a code regression. Check pair and provider enablement in <code>swap-configuration</code>, then Firebase flags. If both match, escalate to the swap team with the comparison.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> Which Slack channel is the <strong>first</strong> place to watch for production swap regressions immediately after a deploy?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>#ptx-swap-build</code></button>
<button class="quiz-choice" data-value="B">B) <code>#alerts-swap</code></button>
<button class="quiz-choice" data-value="C">C) <code>#ptx-swap-prod</code> — production-focused channel for deploys, rollbacks, and first-hour watchdog</button>
<button class="quiz-choice" data-value="D">D) <code>#consumer-service-alerts</code></button>
</div>
<p class="quiz-explanation"><code>#ptx-swap-prod</code> is the production watchdog channel. <code>#alerts-swap</code> carries automated alerts and is complementary. <code>#ptx-swap-build</code> is engineering chatter; <code>#consumer-service-alerts</code> is broader-than-swap.</p>
</div>

<div class="quiz-score"></div>
</div>

---


## Real Ticket Walkthrough -- QAA-1136 (Swap Coverage Gap)

<div class="chapter-intro">
This is your Part 7 capstone chapter, following the exact same pattern as Part 4 Chapter 4.8 (QAA-1139) and Part 4 Chapter 4.9 (QAA-1141). The ticket is <strong>QAA-1136</strong>, a coverage-gap ticket in the same parent epic as QAA-1141 (<code>QAA-1145 — [LWD-LWM] — coverage gap</code>). It asks us to close missing E2E coverage for several swap pairs on desktop. We will deep-dive on <strong>one representative pair — USDT → ETH (B2CQA-2752)</strong> — and then show how to apply the same technique to the rest.
</div>

### 7.4.1 Understanding the Ticket

**Jira ticket:** QAA-1136 (child of QAA-1145 "[LWD-LWM] — coverage gap")
**Representative Xray test case:** B2CQA-2752 — *Swap USDT (ERC-20) → ETH*
**Scope:** Add desktop E2E coverage for swap pairs currently automated on mobile only

**What the ticket asks:**
1. Compare the desktop `send.swap.spec.ts` against the mobile swap specs and against the Xray coverage dashboard
2. Identify which swap pairs have a `B2CQA-*` test case but no desktop automation
3. Add a desktop test entry per missing pair, following the exact shape of the existing `swaps` array in `send.swap.spec.ts`

**The 9 candidate B2CQA test cases** under QAA-1136 correspond to pairs that are automated on mobile but missing from the desktop spec. The one we will implement end-to-end is:

| B2CQA | Pair | Mobile reference |
|---|---|---|
| **B2CQA-2752** | USDT (ERC-20) → ETH | `e2e/mobile/specs/swap/swapETH_USDT_ETH.spec.ts` |

The other pairs (ETH→SOL, BTC→SOL, USDT(ETH)→SOL, USDC→ETH, ETH→DOT, XRP→USDC, BTC→LTC, ...) follow the exact same pattern — 35.12 documents the reference block for each.

> **Note:** Ticket IDs and the list of 9 B2CQA children are live Jira data. Re-check the ticket via Atlassian MCP before implementing: issue IDs, titles, and the exact pair list can change as the epic evolves.

### 7.4.2 Why This Matters

Coverage-gap tickets look small but carry real risk. Each missing pair is a swap flow that:

- Ships to end users with money at stake
- Is covered on one platform only, leaving the other platform regression-blind
- Counts as an automated test case in the Xray coverage dashboard for one platform but not the other — stakeholders see misleading green

For Swap specifically, the risk compounds because:

- Swap is MiCA-regulated — untested pairs can surface in compliance audits
- Provider routing differs per pair (DEX vs CEX, token approvals vs native), so a gap is rarely "just another currency"
- The desktop and mobile codepaths diverge (Electron + Playwright vs React Native + Detox) — a pair that works on one does not guarantee the other

Closing QAA-1136 means the Xray dashboard truthfully reports LLD coverage for these pairs.

### 7.4.3 Analyzing Current Coverage

Start by reading the desktop swap spec to see exactly what is already covered:

```bash
cd /Users/jerome.portier/src/tries/2026-04-08-LedgerHQ-ledger-live
less e2e/desktop/tests/specs/send.swap.spec.ts
```

Focus on the `swaps` array near the top of the file. It looks like this (abridged):

```typescript
const swaps = [
  {
    fromAccount: Account.ETH_1,
    toAccount: Account.BTC_NATIVE_SEGWIT_1,
    xrayTicket: "B2CQA-2750, B2CQA-3135, B2CQA-620, B2CQA-3450",
    tag: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
          "@ethereum", "@family-evm", "@bitcoin", "@family-bitcoin"],
  },
  {
    fromAccount: Account.BTC_NATIVE_SEGWIT_1,
    toAccount: Account.ETH_1,
    xrayTicket: "B2CQA-2744, B2CQA-2432, B2CQA-3450",
    tag: [/* ... */],
  },
  {
    fromAccount: Account.ETH_1,
    toAccount: TokenAccount.ETH_USDT_1,        // ETH → USDT exists
    xrayTicket: "B2CQA-2749, B2CQA-3450",
    tag: [/* ... */],
  },
  // ... more entries ...
];
```

Now grep for our target ticket:

```bash
grep -rn "B2CQA-2752" e2e/
```

Expected result: **matches in the mobile spec only** — `e2e/mobile/specs/swap/swapETH_USDT_ETH.spec.ts` — and nothing in `e2e/desktop/tests/specs/`. That confirms the gap.

<photo of: Xray coverage dashboard showing B2CQA-2752 marked Automated In LLM only, no LLD entry>

The Xray dashboard view above is the artifact stakeholders look at — and the gap it shows (LLM only, no LLD) is exactly the misleading-green state the ticket exists to close. Bookmark this dashboard URL: you will return to it after the PR merges to confirm the entry now shows both LLD and LLM in **Automated In**.

Also grep for the pair direction:

```bash
grep -n "TokenAccount.ETH_USDT_1" e2e/desktop/tests/specs/send.swap.spec.ts
```

Expected result: matches where `TokenAccount.ETH_USDT_1` is the **source** of a swap (e.g., USDT → BTC with B2CQA-2753) and where it is the **destination** (e.g., ETH → USDT with B2CQA-2749). Neither of these is the USDT → ETH direction we need.

**Conclusion:** USDT → ETH is a real gap. The fix is to add an entry to the `swaps` array.

### 7.4.4 Picking the Representative Case

We deep-dive on **USDT → ETH (B2CQA-2752)** for three reasons:

1. **The mobile reference exists**: `e2e/mobile/specs/swap/swapETH_USDT_ETH.spec.ts` is the canonical implementation. We mirror it on desktop.
2. **The pair is non-trivial**: USDT (ERC-20) → ETH is a token-to-native swap on the EVM, which exercises ERC-20 approval + swap — a codepath the simpler ETH → ETH or BTC → BTC pairs would not.
3. **The shape transfers**: once this one works, adding the 8 others (35.12) is mechanical.

The mobile reference (for orientation):

```typescript
// e2e/mobile/specs/swap/swapETH_USDT_ETH.spec.ts
import { Account } from "@ledgerhq/live-common/e2e/enum/Account";
import { runSwapTest } from "./swap";

const swap = new Swap(TokenAccount.ETH_USDT_1, Account.ETH_1, "40", undefined, Fee.MEDIUM);

runSwapTest(
  swap,
  ["B2CQA-2752", "B2CQA-2048"],
  ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5", "@ethereum", "@family-evm"]
);
```

Key points carried over to the desktop entry:

- `fromAccount = TokenAccount.ETH_USDT_1`
- `toAccount = Account.ETH_1`
- Xray ticket anchor: `B2CQA-2752`
- Tags: all supported devices, `@ethereum`, `@family-evm`
- No need for `@bitcoin` / `@family-bitcoin` tags — BTC is not involved

> **Note:** The desktop spec computes the minimum amount dynamically (`app.swap.getMinimumAmount(fromAccount, toAccount)`), so we do **not** hardcode `"40"` like mobile does. The framework handles that for us.

### 7.4.5 Create a Branch

Following the branch naming convention from Chapter 6 and the `support/` prefix for test-coverage work:

```bash
cd /Users/jerome.portier/src/tries/2026-04-08-LedgerHQ-ledger-live
git checkout develop
git pull
pnpm i
# if install failure:
pnpm clean
pnpm i

git checkout -b support/qaa-1136-swap-usdt-eth
```

We use `support/` because the work is test-coverage improvement (refactor / tests / improvements — see Chapter 6). `test/qaa-1136` is also valid per the Conventional-Commits rules the repo uses and matches chapters 29/30 — pick the convention the Swap team currently uses by checking recent merged PRs in the same area.

### 7.4.6 Run the Existing Swap Suite in Isolation to Confirm the Baseline

Before touching any code, reproduce the current green state.

**Prerequisites** (one-time):

```bash
docker info                                  # Speculos needs Docker

export MOCK=0
export COINAPPS="/path/to/coin-apps"          # your local coin-apps clone
export SEED="your 24-word test seed"
export SPECULOS_IMAGE_TAG="ghcr.io/ledgerhq/speculos:master"
export SPECULOS_DEVICE="nanoSP"

pnpm i
pnpm build:lld:deps
pnpm build:cli
pnpm desktop build:testing
pnpm e2e:desktop test:playwright:setup
```

**Run the existing desktop swap spec in isolation** (ETH → USDT, which is the closest neighbor and uses the same DEX path):

```bash
cd e2e/desktop
pnpm test:playwright send.swap.spec.ts --grep "Swap Ethereum to Tether USD"
```

Or, to isolate the whole swap file and go faster with one worker:

```bash
pnpm test:playwright send.swap.spec.ts -- --workers=1
```

Expected: the existing neighbors pass. If they fail, stop and fix the environment first — your own changes should never be judged against a broken baseline.

> **Tip:** Use Playwright's debug mode to step through the flow visually before adding your own entry:
> ```bash
> PWDEBUG=1 pnpm test:playwright send.swap.spec.ts --grep "Swap Ethereum to Tether USD"
> ```

When you run the baseline in PWDEBUG mode, you walk through the same screens an end user would. Capture each one as you go — these are the visual checkpoints you will compare your USDT → ETH run against.

<photo of: Playwright debugger paused on the Swap entry — accounts list visible with from/to picker>

This is the **pick-from-account** step. The Live App has booted, the Wallet API has handed it the account list, and the user is choosing the source account for the swap. In our spec this is `Account.ETH_1` (or `TokenAccount.ETH_USDT_1` for our new entry); the framework auto-selects via `performSwapUntilQuoteSelectionStep` so the test does not need to click manually. If your run hangs here, the account fixture (CLI-seeded via `liveDataWithAddressCommand`) is probably the suspect.

<photo of: Picking the to-account — destination currency picker open showing ETH highlighted>

This is the **pick-to-account** step. The destination side opens a currency picker; the test selects the target currency by name (Ethereum for our pair). Cross-family pairs (e.g., BTC → SOL) show the importance of having the destination account already created on the test seed — if the account does not exist in `live-common`'s enum, the picker cannot land on it.

<photo of: Amount entry field with the minimum amount auto-filled from getMinimumAmount>

The **set-amount** step. `getMinimumAmount(fromAccount, toAccount)` queries the swap backend for the minimum the provider will accept and the framework fills the field with that value. Hardcoding amounts (like the mobile spec's `"40"`) is the wrong pattern on desktop — the dynamic minimum keeps the test resilient to backend tweaks.

### 7.4.7 Implement the Change

Open `e2e/desktop/tests/specs/send.swap.spec.ts` and **add one entry to the `swaps` array** for USDT → ETH. Do not touch imports, helpers, or the `for` loop — they all already handle this shape.

```typescript
// e2e/desktop/tests/specs/send.swap.spec.ts

const swaps = [
  // ... existing entries unchanged ...

  {
    fromAccount: TokenAccount.ETH_USDT_1,
    toAccount: Account.ETH_1,
    xrayTicket: "B2CQA-2752, B2CQA-3450",
    tag: [
      "@NanoSP",
      "@LNS",
      "@NanoX",
      "@Stax",
      "@Flex",
      "@NanoGen5",
      "@swapSmoke",
      "@ethereum",
      "@family-evm",
    ],
  },

  // ... other existing entries unchanged ...
];
```

**What this does, walk-through:**

1. The outer `for (const { fromAccount, toAccount, xrayTicket, tag } of swaps)` loop iterates over every entry, including ours.
2. `setupEnv(true)` installs Speculos setup with transaction-broadcast disabled.
3. `setExchangeDependencies` is called in `beforeEach` with the source and destination Speculos apps — for our entry, that means Ethereum (for USDT ERC-20) and Ethereum (for ETH).
4. `test.use({ speculosApp: exchangeApp, teamOwner: Team.SWAP, ... })` wires the fixture: the Exchange app is primary, accounts are pre-populated via `cliCommandsOnApp` using `liveDataWithAddressCommand`.
5. Inside the test: `getMinimumAmount(fromAccount, toAccount)` asks the swap backend for the min, a `Swap` instance is built, and `performSwapUntilQuoteSelectionStep` drives the UI up to the quotes list.
6. `selectExchangeWithoutKyc(electronApp, swap)` picks a no-KYC provider (typically a DEX like 1inch or Paraswap for USDT → ETH on EVM).
7. `ensureTokenApproval(fromAccount, provider, minAmount)` handles the ERC-20 approval — **this is why USDT → ETH is the interesting case**, because native-to-native pairs skip this step entirely.
8. Depending on whether the provider is in-app (`provider.app === exchangeApp`) or not, the right device-signing dance is triggered, Speculos accepts, and the test asserts on the final toaster (or the exchange-completed drawer text).

That single entry is the entire code change.

> **Why `B2CQA-3450` alongside `B2CQA-2752`?** `B2CQA-3450` is the umbrella "swap generic flow" Xray test case that most entries in this spec carry. Attach both so the Xray coverage updates correctly for both the generic flow and the specific pair.

### 7.4.8 Rerun Just That Test

Use Playwright's test-name filter to run only our new entry:

```bash
cd e2e/desktop
pnpm test:playwright send.swap.spec.ts --grep "Swap Tether USD \(ERC-20\) to Ethereum"
```

> **Note:** The test name comes from `` `Swap ${fromAccount.currency.name} to ${toAccount.currency.name}` `` in the `for` loop. The exact string depends on how `TokenAccount.ETH_USDT_1.currency.name` is set in `live-common`. Run the test once to see Playwright's exact output, then copy-paste the name into `--grep` for targeted reruns.

**Run three times for stability** — chapter 28 rule:

```bash
pnpm test:playwright send.swap.spec.ts --grep "Swap Tether USD" --repeat-each=3
```

<photo of: Swap quotes screen with multiple providers ranked, best-rate row highlighted>

The **provider selection** step. The Live App has fanned the quote out to every enabled provider for the pair and rendered the ranked list. `selectExchangeWithoutKyc` filters this list to KYC-free providers (DEXes, in our case) and picks one deterministically. If the screenshot shows only CEX providers (Changelly, Exodus), the test will skip the approval flow entirely — that is a sign the DEX side of the configuration is not enabled in the env you are running against.

<photo of: Review/confirm screen in the Live App before sending the swap to the device>

The **review** step. The Live App displays a final summary — from-amount, to-amount, fees, ETA, recipient — before handing off to the Wallet shell. This is the last UI checkpoint before the device prompt. The Playwright assertion at this step is typically a content match against the swap object the test built; if the displayed values drift from the requested ones, the bug is most likely in the quote or the exchange-SDK contract.

<photo of: Speculos device emulator showing the swap accept screen on a Nano S Plus>

The **device signing** step (Speculos). With broadcast off, Speculos is configured to auto-accept; with broadcast on (manual regression on `ppr`) the test driver presses the buttons via Speculos's HTTP API. For our USDT → ETH entry, this is actually the **second** device prompt the test sees — the first one was the approval (covered in 6.2.11), and this one is the swap itself.

<photo of: "Transaction sent" toaster after the swap submits>

The **operation in history** is the final assertion. Either the toaster (`expectTransactionSentToasterToBeVisible`) or the exchange-completed drawer (`verifyExchangeCompletedTextContent`) closes the test. From here, the swap completer takes over server-side — see Ch 7.2.4 — and the trade reaches its terminal state asynchronously. The test does not wait for the terminal state in the broadcast-off mode; the assertion is "the Wallet handed the tx to the broadcast layer", not "the trade settled".

All three runs should pass. If they do not, the most common failures on a DEX token swap are:

- Speculos did not switch apps between Exchange and Ethereum → check `setExchangeDependencies` and `speculos.relaunch`
- Token approval was not granted → `ensureTokenApproval` failed silently; add a wait and re-run
- Provider returned `INSUFFICIENT_LIQUIDITY` → try a different minimum or wait for provider liquidity to recover (backend concern, not a test bug)

### 7.4.9 Verify with Allure

Generate and open the Allure report:

```bash
allure serve allure-results
```

Navigate to **Suites** → **Swap - Accepted (without tx broadcast)** → **Swap Tether USD (ERC-20) to Ethereum** and confirm:

- **Links** section shows `B2CQA-2752` and `B2CQA-3450` as TMS links, each clickable and pointing to Jira
- **Steps** show the full sequence: build swap, perform up to quote selection, select provider, ensure token approval, sign on Speculos, final assertion
- **Screenshots** captured at key steps (start, quote list, device signing, completion)
- **Team ownership** is `Team.SWAP` (consistent with the rest of the file)
- Tags match what you set: `@NanoSP`, `@LNS`, ..., `@ethereum`, `@family-evm`, `@swapSmoke`

<photo of: Allure report Suites view showing "Swap Tether USD (ERC-20) to Ethereum" with green steps and TMS links to B2CQA-2752>

The Allure suites view is the artifact you attach to the PR description so reviewers can confirm the test ran green end-to-end without re-running it themselves. The green check next to each step is the audit trail; the TMS links in the **Links** section are how Xray correlates the run back to the B2CQA test cases.

Press `Ctrl+C` to stop the Allure server.

### 7.4.10 Commit the Change

Follow the Conventional Commits style — test scope, concise description:

```bash
git add e2e/desktop/tests/specs/send.swap.spec.ts
git commit -m "test(desktop): add swap USDT (ERC-20) to ETH coverage (QAA-1136)"
```

Why this shape:

- `test` — commit type for tests (not `feat`, not `fix`)
- `(desktop)` — scope is the desktop E2E suite
- Description is imperative and lowercase
- The Jira ticket (`QAA-1136`) is referenced in the subject to make the history searchable

If you want, add the `B2CQA-2752` ticket in the body:

```bash
git commit -m "test(desktop): add swap USDT (ERC-20) to ETH coverage (QAA-1136)" \
           -m "Closes coverage gap for B2CQA-2752. Mirrors mobile spec swapETH_USDT_ETH.spec.ts."
```

### 7.4.11 Open the PR with `/create-pr`, Mark Ready, and Link Xray

Use the Claude Code command (see Chapter 4.8.9):

```
/create-pr
```

Answer the prompts:

1. **Ticket URL** — `https://ledgerhq.atlassian.net/browse/QAA-1136`
2. **Ticket description** — "Add desktop E2E coverage for swap USDT (ERC-20) → ETH, mirroring mobile coverage"
3. **Change type** — `test`
4. **Change scope** — `e2e/desktop`
5. **Test coverage** — `yes`
6. **QA focus areas** — "Swap USDT → ETH on DEX path, ERC-20 token approval, Speculos Ethereum app switching"
7. **UI changes** — `no`

After the draft PR is created and CI passes:

1. In GitHub, click **"Ready for review"** to publish the PR
2. Once approved, **"Merge pull request"** into `develop`
3. **Update Xray:**
   - Open `B2CQA-2752` in Jira
   - Set **Status** to **Automated**
   - In **Automated In**, **add** LLD next to the existing LLM entry (do not replace — see chapter 30.9)
4. Move `QAA-1136` to **Done** (or the team's equivalent status once **all** 9 B2CQA children are landed) and link the PR
5. The next CI run uploads Allure → Xray updates automatically

### 7.4.12 Reference -- Applying the Same Fix to the Other Missing Pairs

The 8 other missing pairs under QAA-1136 all take the same shape. Each one is a single new object in the `swaps` array. Pair → ticket → entry:

```typescript
// USDT (ERC-20) → ETH — implemented in 35.7
{
  fromAccount: TokenAccount.ETH_USDT_1,
  toAccount: Account.ETH_1,
  xrayTicket: "B2CQA-2752, B2CQA-3450",
  tag: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
        "@swapSmoke", "@ethereum", "@family-evm"],
},

// ETH → SOL — cross-family, EVM → Solana
{
  fromAccount: Account.ETH_1,
  toAccount: Account.SOL_1,
  xrayTicket: "B2CQA-<ETH_SOL>, B2CQA-3450",
  tag: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
        "@ethereum", "@family-evm", "@solana", "@family-solana"],
},

// BTC → SOL — cross-family, BTC → Solana
{
  fromAccount: Account.BTC_NATIVE_SEGWIT_1,
  toAccount: Account.SOL_1,
  xrayTicket: "B2CQA-<BTC_SOL>, B2CQA-3450",
  tag: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
        "@bitcoin", "@family-bitcoin", "@solana", "@family-solana"],
},

// USDT (ERC-20) → SOL — token on EVM → Solana
{
  fromAccount: TokenAccount.ETH_USDT_1,
  toAccount: Account.SOL_1,
  xrayTicket: "B2CQA-<USDT_SOL>, B2CQA-3450",
  tag: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
        "@swapSmoke", "@ethereum", "@family-evm", "@solana", "@family-solana"],
},

// USDC (ERC-20) → ETH
{
  fromAccount: TokenAccount.ETH_USDC_1,
  toAccount: Account.ETH_1,
  xrayTicket: "B2CQA-<USDC_ETH>, B2CQA-3450",
  tag: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
        "@swapSmoke", "@ethereum", "@family-evm"],
},

// ETH → DOT
{
  fromAccount: Account.ETH_1,
  toAccount: Account.DOT_1,
  xrayTicket: "B2CQA-<ETH_DOT>, B2CQA-3450",
  tag: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
        "@ethereum", "@family-evm", "@polkadot", "@family-polkadot"],
},

// XRP → USDC (ERC-20)
{
  fromAccount: Account.XRP_1,
  toAccount: TokenAccount.ETH_USDC_1,
  xrayTicket: "B2CQA-<XRP_USDC>, B2CQA-3450",
  tag: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
        "@xrp", "@family-xrp", "@ethereum", "@family-evm"],
},

// BTC → LTC
{
  fromAccount: Account.BTC_NATIVE_SEGWIT_1,
  toAccount: Account.LTC_1,
  xrayTicket: "B2CQA-<BTC_LTC>, B2CQA-3450",
  tag: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
        "@bitcoin", "@family-bitcoin", "@litecoin", "@family-litecoin"],
},
```

**Process per pair:**

1. Confirm the Xray ID on the Jira ticket (`QAA-1136` has the full list under its children)
2. Confirm the `Account` / `TokenAccount` enum name exists in `libs/live-common/src/e2e/enum/Account.ts` — if a currency enum is missing, adding it is a separate PR on `live-common` first
3. Confirm the device tags match what the pair actually supports (not every pair runs on every device — some are Nano-S-Plus-only, some exclude LNS)
4. Add the entry, rerun `send.swap.spec.ts` with `--grep "Swap <From> to <To>"`, verify Allure
5. Commit per pair or batch by family — the Swap team usually prefers one PR per 2-3 pairs to keep reviews small

> **Guardrail:** Do **not** copy-paste all 9 entries in one PR without running each. Each pair exercises different provider routes, and a single unrelated failure (e.g., a DEX liquidity problem for DOT) will block the entire batch. Land them in small PRs.

### 7.4.13 Reference -- Writing a Swap Test from Scratch

You will rarely write a swap test from scratch — almost every new pair goes through the `swaps` array pattern above. But for the case where you need a custom flow (a swap with a specific provider override, a KYC variant, a swap-to-swap edge case), here is the canonical template:

```typescript
import test from "tests/fixtures/common";
import { Team } from "@ledgerhq/live-common/e2e/enum/Team";
import { Account, TokenAccount } from "@ledgerhq/live-common/e2e/enum/Account";
import { AppInfos } from "@ledgerhq/live-common/e2e/enum/AppInfos";
import { setExchangeDependencies } from "@ledgerhq/live-common/e2e/speculos";
import { Swap } from "@ledgerhq/live-common/e2e/models/Swap";
import { addTmsLink } from "tests/utils/allureUtils";
import { getDescription } from "tests/utils/customJsonReporter";
import { setupEnv, performSwapUntilQuoteSelectionStep } from "tests/utils/swapUtils";
import { liveDataWithAddressCommand } from "@ledgerhq/live-common/e2e/cliCommandsUtils";

const exchangeApp: AppInfos = AppInfos.EXCHANGE;

test.describe("Swap -- USDT (ERC-20) to ETH (custom flow)", () => {
  setupEnv(true);

  const fromAccount = TokenAccount.ETH_USDT_1;
  const toAccount = Account.ETH_1;

  const accPair: string[] = [fromAccount, toAccount].map(acc =>
    acc.currency.speculosApp.name.replace(/ /g, "_"),
  );

  test.beforeEach(async () => {
    setExchangeDependencies(accPair.map(appName => ({ name: appName })));
  });

  test.use({
    teamOwner: Team.SWAP,
    userdata: "skip-onboarding-with-last-seen-device",
    speculosApp: exchangeApp,
    cliCommandsOnApp: [
      [
        { app: fromAccount.currency.speculosApp, cmd: liveDataWithAddressCommand(fromAccount) },
        { app: toAccount.currency.speculosApp, cmd: liveDataWithAddressCommand(toAccount) },
      ],
      { scope: "test" },
    ],
  });

  test(
    "Swap Tether USD (ERC-20) to Ethereum",
    {
      tag: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5",
            "@swapSmoke", "@ethereum", "@family-evm"],
      annotation: { type: "TMS", description: "B2CQA-2752, B2CQA-3450" },
    },
    async ({ app, electronApp, speculos }) => {
      await addTmsLink(getDescription(test.info().annotations, "TMS").split(", "));

      const minAmount = await app.swap.getMinimumAmount(fromAccount, toAccount);
      const swap = new Swap(fromAccount, toAccount, minAmount);

      await performSwapUntilQuoteSelectionStep(app, electronApp, swap, minAmount);
      const provider = await app.swap.selectExchangeWithoutKyc(electronApp, swap);
      swap.setProvider(provider);
      await app.swap.ensureTokenApproval(fromAccount, provider, minAmount);

      if (provider.app) {
        if (provider.app !== exchangeApp) {
          await speculos.relaunch(provider.app.name);
        }
        await app.swap.clickExchangeButton(electronApp);
        await app.swap.clickExecuteSwapButton(electronApp);
        await app.swap.clickContinueButton();
        await app.speculos.verifyAmountsAndAcceptSwap(swap, minAmount);
        await app.swap.expectTransactionSentToasterToBeVisible();
      } else {
        await app.swap.clickExchangeButton(electronApp);
        await app.speculos.verifyAmountsAndAcceptSwap(swap, minAmount);
        await app.swapDrawer.verifyExchangeCompletedTextContent(
          swap.accountToCredit.currency.name,
        );
      }
    },
  );
});
```

Compared to the `swaps`-array approach this is verbose, but it is the right escape hatch when a pair needs a variation the loop cannot express.

<div class="chapter-outro">
<strong>Key takeaway:</strong> QAA-1136 is a coverage-gap ticket in the same epic as QAA-1141. Closing it is not about writing new tests from scratch — it is about <strong>adding one object to an existing <code>swaps</code> array per missing pair</strong>. The workflow is identical to chapters 29/30: understand the ticket → analyze existing coverage → create branch (<code>support/</code>) → baseline run → add entry → rerun 3x → verify Allure → commit (Conventional Commits) → <code>/create-pr</code> → mark ready → update Xray <strong>Automated In</strong> per pair. Always ship per pair or in small batches — never a 9-entry mega-PR. The user-visible flow you are automating here is the same one Part 3 Ch 3.4 walks an end user through; the approval step in particular is the one Ch 7.2.11 explained the architecture of, and Part 6 Ch 6.8 walked through the senior-QAA implementation of.
</div>

### 7.4.14 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz -- QAA-1136 Walkthrough</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> For QAA-1136, what is the minimal correct implementation of the USDT → ETH coverage?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A new spec file <code>swap.usdt.eth.spec.ts</code> with a full custom <code>test.describe</code> block</button>
<button class="quiz-choice" data-value="B">B) A patch to <code>performSwapUntilQuoteSelectionStep</code> to handle ERC-20</button>
<button class="quiz-choice" data-value="C">C) One new object appended to the <code>swaps</code> array in <code>send.swap.spec.ts</code>, reusing every existing helper</button>
<button class="quiz-choice" data-value="D">D) A Firebase feature flag flip in <code>swap-live-app</code></button>
</div>
<p class="quiz-explanation">The <code>swaps</code> array + <code>for</code> loop pattern in <code>send.swap.spec.ts</code> already handles the entire flow (fixtures, Speculos setup, token approval, provider selection, assertion). Adding a pair is a single new object. Writing a new spec or patching helpers would duplicate existing code.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> Why is USDT → ETH a more interesting test case than BTC → BTC would be?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It uses a different provider</button>
<button class="quiz-choice" data-value="B">B) It exercises the ERC-20 <strong>token approval</strong> codepath via <code>ensureTokenApproval</code>, which native-to-native swaps skip entirely</button>
<button class="quiz-choice" data-value="C">C) It does not need a device</button>
<button class="quiz-choice" data-value="D">D) It bypasses Speculos</button>
</div>
<p class="quiz-explanation">Token swaps on DEXs require an ERC-20 <code>approve()</code> before the swap can happen. <code>ensureTokenApproval</code> handles that step. Native-to-native pairs (BTC → BTC, ETH → ETH) do not exercise this codepath, so they are not good representatives for token-swap coverage.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> What is the correct way to update Xray after merging the PR for B2CQA-2752?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Replace LLM with LLD in <strong>Automated In</strong></button>
<button class="quiz-choice" data-value="B">B) Remove LLM since desktop now covers it</button>
<button class="quiz-choice" data-value="C">C) Leave Automated In unchanged</button>
<button class="quiz-choice" data-value="D">D) Add LLD alongside the existing LLM in <strong>Automated In</strong> — both platforms now automate this test case</button>
</div>
<p class="quiz-explanation">Desktop coverage does not invalidate mobile coverage. Both platforms now automate <code>B2CQA-2752</code>, so <strong>Automated In</strong> should reflect both — same rule as chapter 30.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> A teammate proposes to land all 9 missing pairs in a single PR. What is the right pushback?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Ship per pair or in small batches (2-3 entries per PR) — each pair exercises different provider routes and one unrelated DEX failure would block the entire batch</button>
<button class="quiz-choice" data-value="B">B) It is fine — land everything at once</button>
<button class="quiz-choice" data-value="C">C) Only Senior QAs can merge swap changes</button>
<button class="quiz-choice" data-value="D">D) Do it in a single commit on <code>main</code></button>
</div>
<p class="quiz-explanation">Each pair can fail independently (provider liquidity, token approval, Speculos app switching). Small PRs isolate failures and keep reviews fast. This is consistent with the Git workflow rules: small, isolated, meaningful commits.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> You run the new USDT → ETH test and it fails with <code>INSUFFICIENT_LIQUIDITY</code>. What is the correct first action?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Open a bug in <code>ledger-live</code></button>
<button class="quiz-choice" data-value="B">B) Check the provider status (this is a provider-side issue, not a test regression) — investigate on Datadog or the swap backend before changing test code</button>
<button class="quiz-choice" data-value="C">C) Delete the test entry</button>
<button class="quiz-choice" data-value="D">D) Switch to a different currency</button>
</div>
<p class="quiz-explanation"><code>INSUFFICIENT_LIQUIDITY</code> is a provider-side response, not a test bug. Check provider health (Datadog, the <code>swap</code> backend, Slack) before changing code. If it is transient, wait and retry. If it is persistent, escalate to the Swap team with the observed behavior.</p>
</div>

<div class="quiz-score"></div>
</div>

---


## Part 7 Final Assessment

<div class="quiz-container" data-pass-threshold="80">
<h3>Part 7 Final Assessment</h3>
<p class="quiz-subtitle">12 questions · 80% to pass · Covers all Part 7 chapters</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> A Live App is loaded into the Wallet via...</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A native module compiled into the installer</button>
<button class="quiz-choice" data-value="B">B) A sandboxed webview pointing at a remote URL declared in the manifest</button>
<button class="quiz-choice" data-value="C">C) A command-line plugin</button>
<button class="quiz-choice" data-value="D">D) An Electron IPC channel</button>
</div>
<p class="quiz-explanation">A Live App is a remote web app loaded into a sandboxed webview. The URL, permissions, and icon come from the manifest declared in <code>ledger-live</code>.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> Which Wallet-API invariant protects users against a misbehaving Live App?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The Live App cannot access the internet</button>
<button class="quiz-choice" data-value="B">B) The Live App cannot render UI</button>
<button class="quiz-choice" data-value="C">C) Every privileged action (accounts, signing, network) is brokered by the Wallet shell and checked against the manifest's declared permissions</button>
<button class="quiz-choice" data-value="D">D) The Live App is rebuilt locally before launch</button>
</div>
<p class="quiz-explanation">The Wallet-API bridge enforces permissions at the shell. A Live App cannot escalate beyond what its manifest declares.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> The Swap backend is composed of two processes. Which pair?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The <code>swap</code> service (REST, quotes, trade creation) and the <code>swap</code> completer (status poller) — with PostgreSQL and RabbitMQ between them</button>
<button class="quiz-choice" data-value="B">B) <code>swap-live-app</code> and <code>exchange-sdk</code></button>
<button class="quiz-choice" data-value="C">C) Firebase and Vercel</button>
<button class="quiz-choice" data-value="D">D) Speculos and the Ledger device</button>
</div>
<p class="quiz-explanation">The <code>swap</code> service handles synchronous quote/trade requests; the completer polls providers asynchronously for long-running trade statuses. RabbitMQ decouples them; PostgreSQL is the system of record.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> Which repo holds the declarative rules about which swap pairs are enabled per provider and per environment?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>swap-live-app</code></button>
<button class="quiz-choice" data-value="B">B) <code>ledger-live</code></button>
<button class="quiz-choice" data-value="C">C) <code>ledger-live-assets</code></button>
<button class="quiz-choice" data-value="D">D) <code>swap-configuration</code> — Rust tooling over CSV configuration files</button>
</div>
<p class="quiz-explanation"><code>swap-configuration</code> is the declarative ruleset. Pair availability, min/max, fee tiers, provider enablement — all live there. Check it before assuming a code bug.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q5.</strong> Swap follows which branching model?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Trunk-based on <code>develop</code> — short-lived <code>feat/*</code> and <code>fix/*</code> branches, Changesets-driven versioning, <code>release-production.yml</code> creates an <code>app@vX.Y.Z</code> tag from <code>develop</code></button>
<button class="quiz-choice" data-value="B">B) Gitflow with a dedicated <code>release/app@v1.3.xx</code> branch cut from <code>develop</code>, merged to both <code>main</code> and <code>develop</code> when tagged</button>
<button class="quiz-choice" data-value="C">C) Continuous deployment from <code>main</code> with no tags</button>
<button class="quiz-choice" data-value="D">D) None — Swap releases are manual</button>
</div>
<p class="quiz-explanation">Swap is <strong>trunk-based</strong>. <code>develop</code> is the single source of truth; there is no release branch and no Gitflow. Production tags (<code>app@vX.Y.Z</code>) are created by the <code>release-production.yml</code> workflow, which consumes pending Changesets. Hotfixes branch from the latest tag and are merged back into <code>develop</code>.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> Swap has <strong>two</strong> Firebase projects. What is the responsibility split?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) One for Android, one for iOS</button>
<button class="quiz-choice" data-value="B">B) One for pre-prod, one for production</button>
<button class="quiz-choice" data-value="C">C) <code>ledger-live-production</code> holds Wallet-side flags (including whether the Swap entry appears in the Wallet); <code>swap-live-app</code> holds Live-App-side flags (provider enablement inside the app, UI experiments)</button>
<button class="quiz-choice" data-value="D">D) One is public, one is private</button>
</div>
<p class="quiz-explanation">The split is Wallet-side vs. Live-App-side. Remembering this split is essential when tracing why a swap feature is (in)visible for a given user — the flag may be in one project or the other.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q7.</strong> The Swap production backend URL is...</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>swap.ledger.com</code></button>
<button class="quiz-choice" data-value="B">B) <code>swap-prod.ledger-test.com/v5</code></button>
<button class="quiz-choice" data-value="C">C) <code>swap-live-app.ledger.com</code></button>
<button class="quiz-choice" data-value="D">D) <code>swap-live-app.aws.prd.ldg-tech.com</code></button>
</div>
<p class="quiz-explanation">Production backend (the <code>swap</code> service) is <code>swap.ledger.com</code>. Production frontend (the Live App) is <code>swap-live-app.ledger.com</code> (public Cloudflare) or <code>swap-live-app.aws.prd.ldg-tech.com</code> (internal AWS). <code>ledger-test.com</code> is the public test domain used for stg/ppr only.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q8.</strong> On QAA-1136, what is the correct analysis step before writing code?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Rewrite <code>send.swap.spec.ts</code> from scratch</button>
<button class="quiz-choice" data-value="B">B) Open a Jira sub-task per missing pair</button>
<button class="quiz-choice" data-value="C">C) Upgrade <code>exchange-sdk</code></button>
<button class="quiz-choice" data-value="D">D) <code>grep</code> each B2CQA ID under QAA-1136 in <code>e2e/desktop/tests/specs/</code> to confirm the gap — then cross-check against the mobile specs to get the canonical reference</button>
</div>
<p class="quiz-explanation">Never add tests without confirming the gap first. <code>grep</code> the ID in the desktop specs (no match = real gap) and confirm the mobile reference exists — that gives you the canonical flow to mirror.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q9.</strong> Which helper handles the ERC-20 approval step required before a DEX token swap?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>app.swap.ensureTokenApproval(fromAccount, provider, minAmount)</code></button>
<button class="quiz-choice" data-value="B">B) <code>app.swap.getMinimumAmount(fromAccount, toAccount)</code></button>
<button class="quiz-choice" data-value="C">C) <code>performSwapUntilQuoteSelectionStep()</code></button>
<button class="quiz-choice" data-value="D">D) <code>setExchangeDependencies()</code></button>
</div>
<p class="quiz-explanation"><code>ensureTokenApproval</code> performs the ERC-20 approval before the swap. Without it, DEX token swaps will fail. Native-to-native pairs do not need this step.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q10.</strong> The <code>completer</code> lets the Swap backend do something the Live App alone could not. What?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Broadcast transactions</button>
<button class="quiz-choice" data-value="B">B) Track the status of a trade asynchronously until it reaches a terminal state, even if the user closes the Live App</button>
<button class="quiz-choice" data-value="C">C) Sign transactions</button>
<button class="quiz-choice" data-value="D">D) Add new providers</button>
</div>
<p class="quiz-explanation">The completer decouples user-facing trade creation (fast, synchronous) from provider status tracking (slow, asynchronous). It is why a trade can reach its terminal state even after the user closes Swap.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q11.</strong> You see a swap failing in pre-prod but not in staging. What is the most likely cause?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A Speculos version mismatch</button>
<button class="quiz-choice" data-value="B">B) A Node version mismatch in CI</button>
<button class="quiz-choice" data-value="C">C) A configuration delta — <code>swap-configuration</code> rules, or a Firebase flag in <code>swap-live-app</code>, differing between stg and ppr</button>
<button class="quiz-choice" data-value="D">D) A Chrome auto-update</button>
</div>
<p class="quiz-explanation">"Works in stg, fails in ppr" is almost always a configuration delta. Start with <code>swap-configuration</code> (pair/provider enablement) and the <code>swap-live-app</code> Firebase flags side by side.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q12.</strong> After merging the PR for B2CQA-2752 on QAA-1136, what is the correct Jira/Xray hygiene?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Close QAA-1136 immediately</button>
<button class="quiz-choice" data-value="B">B) Delete the original mobile test</button>
<button class="quiz-choice" data-value="C">C) Nothing — CI handles everything</button>
<button class="quiz-choice" data-value="D">D) Set B2CQA-2752 Status=Automated, add LLD to Automated In (keeping LLM); only move QAA-1136 to Done when <strong>all</strong> its B2CQA children are landed</button>
</div>
<p class="quiz-explanation">Per child: mark Automated in Xray and add LLD to Automated In without removing LLM. The parent QAA-1136 stays in progress until every child is landed; otherwise the coverage dashboard will lie.</p>
</div>

<div class="quiz-score"></div>
</div>
