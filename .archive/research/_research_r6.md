# R6 — Live Apps Platform & Coin Framework

Research agent: R6
Scope: Live Apps (dApps inside Ledger Live) and Coin Framework (how coins are
integrated into LLD/LLM).
Confluence spaces surfaced:
- WAL — WALLET-API (Live Apps + wallet-api)
- CIP — Coin Integration Playbook
- CF — Coin Framework
- PTX — Consumer Services (Swap / Exchange / PTX)
- LLI — Ledger Live (how-to / LQA)
- QA — Quality Assurance

All claims below are grounded in pages fetched via Rovo search on
ledgerhq.atlassian.net. Direct page URLs are cited inline.

---

## 1. Live Apps platform

A Live App is an external application (dApp or web app, owned by Ledger or a
partner) that is loaded inside Ledger Live Desktop (LLD) and Ledger Live
Mobile (LLM) and allowed to request signatures from the Ledger device through
the Ledger Live "context". In the Ledger Live UI, Live Apps surface under the
Discover tab (and, historically, a dedicated Market tab, now moving to the
top-right compass icon in Ledger Wallet v4.0) and under the PTX entry points
(Swap, Buy, Sell, Earn).

Grounding:
- "Ledger Live (both Mobile and Desktop) allows for Apps to be run within its
  Context, which in turn allows the App to request signatures from Ledger
  devices via that context." — Testing the [L] Market Live App in Ledger Live
  (NFT space, page 4084171039).
- "The Manifest is an config file that allows external applications and
  decentralized applications (dApps) to be integrated inside the Ledger Live
  software as a Live app." — Manifest V2, WAL space, page 3924230200.
- "The Ledger Wallet API is a specialized library designed to seamlessly
  integrate decentralized applications (Live Apps) within Ledger Live." —
  Ledger Live container page, CIP space, page 5389025304.

### 1.1 The manifest (Manifest V2)

Every Live App is declared by a manifest JSON stored in the `manifest-api`
repository (github.com/LedgerHQ/manifest-api). The manifest is what Ledger
Live reads to know a Live App exists, its URL, its permissions, its supported
currencies and its visibility.

Manifest V2 fields (from Manifest V2 page 3924230200):

- id: string, kebab-case identifier (e.g. `swap-live-app-demo-3-stg`).
- author: optional string; when set, suppresses the "external app" disclaimer
  for Ledger-owned apps.
- name, url, homepageUrl, supportUrl, icon: human-facing metadata.
- platforms: `"desktop" | "mobile" | "all"`.
- apiVersion: wallet-api version compatibility.
- manifestVersion: `2` (V2 differs from V1; many V1 fields moved to config).
- categories: array used by Discover categorisation.
- currencies: array — `["*"]` wildcard or explicit list.
- content: `{shortDescription, description}` — English only in V2.
- permissions: wallet-api permission scopes.
- domains: optional allow-listed navigation domains.
- type: `"dapp" | "walletApp" | "webBrowser"` — main dispatch type.
- params: shape depends on `type` (see below).
- visibility: `"complete" | "searchable" | "deep"` — controls whether the app
  shows in the main catalogue, only via search, or only via deep link.

Params by type:

- dapp — `{ dappUrl, nanoApp, dappName, networks: [{chainID, nodeURL,
  currency}] }`. Used for EVM/web3 dapps that ride on the Ethereum app.
- walletApp — free-form params; the Swap Live App and the Buy/Sell Live Apps
  use this mode because they talk to the wallet-api rather than to a single
  EVM chain.
- webBrowser — `{ webUrl, webAppName, currencies }`. Limited web-browser
  context.

### 1.2 The Wallet API protocol

The Wallet API is the JSON-RPC-style bridge between a Live App (running in a
webview/iframe inside LLD or LLM) and the Ledger Live host. It is how a Live
App gets the user's accounts, requests a signature, or triggers an Exchange
flow.

Key facts:
- Client SDK + server lives in github.com/LedgerHQ/wallet-api.
- Inside the ledger-live monorepo the integration code is at
  `libs/ledger-live-common/src/wallet-api` (entry confirmed on the CIP
  "Ledger Live" architecture page, 5389025304).
- The Exchange/Swap flow uses two dedicated wallet-api methods,
  `start exchange` and `complete exchange`, with a payload signed by the
  Exchange device app. See "Partner integration SWAP - Frameworks Overview"
  (PTX space, 6287622174).
- The Exchange SDK (github.com/LedgerHQ/exchange-sdk) is the higher-level
  library used by the Swap Live App and by CEX partner live apps to call the
  Wallet API. See SWAP tech page (PTX, 3804496035).
- Manifests gained an `author` field so that Ledger-owned web3 apps no longer
  trigger the "external app" modal. See "Live - Wallet API Weekly Update"
  (ENGB2C, 4037181769).

### 1.3 How Ledger Live loads a Live App

Per "Debug live app locally" (QA, 6704562204) and the CIP Ledger Live page:

1. Ledger Live fetches the list of manifests from the `manifest-api` (remote,
   per environment).
2. For Discover / PTX entry points, LLD and LLM render a screen component
   (e.g. `apps/ledger-live-mobile/src/screens/PTX/index.tsx`) that picks the
   manifest for the feature being launched.
3. The Live App is loaded in a sandboxed webview/iframe pointed at the
   manifest's `url` (or `dappUrl` / `webUrl` depending on `type`).
4. The wallet-api client (inside the Live App) connects to the wallet-api
   transport exposed by Ledger Live; permissions declared in the manifest
   gate what the Live App is allowed to request.
5. Helpers such as `isWhitelistedDomain` in
   `libs/ledger-live-common/src/wallet-api/helpers.ts` enforce allowed
   navigation.

### 1.4 The marketplace (Discover)

The Discover tab is Ledger Live's Live Apps marketplace. Visibility of each
Live App is controlled by three levers, in this order:

1. `visibility` in the manifest (`complete | searchable | deep`).
2. `feature_*` Firebase Remote Config flags that gate whole families of Live
   Apps (Swap, Buy/Sell, specific partner live apps).
3. Per-env `manifest-api` contents (dev / staging / prerelease / prod).

Examples of how QA turns feature flags on for Live Apps:
- TON Swap integration (PTX 5134876674): "set the feature flag
  `ptxSwapLiveAppDemoThree` with `{enabled:true,
  params:{manifest_id:'swap-live-app-demo-3-stg'}}`".
- Tutorial Feature flags (PTX 4059496451): shows editing
  `feature_portfolio_exchange_banner` in the Firebase console.

### 1.5 Where the Swap Live App fits

The Swap Live App is the flagship `walletApp`-type Live App. Its entry point
is in the PTX area of Ledger Live and it replaces the legacy in-app swap UI.
See section 7 below for the full breakdown.

---

## 2. Live App lifecycle (dev, staging, prerelease, prod)

The Live Apps platform inherits the Ledger Live multi-environment model (also
R3's topic). There are four Firebase projects, each driving a Remote Config
that tells Ledger Live which Live Apps exist and which feature flags are on:

- development: `ledger-live-development` — local dev, internal pre-alpha.
- staging: `ledger-live-staging` — internal QA, nightly builds, Speculos E2E.
- prerelease (pre-prod): shared staging stack + ppr backends — release
  candidates, regression.
- production: `ledger-live-production` — shipped LLD / LLM.

Grounding:
- "1 - Firebase Feature Flags in Ledger Live and Recover" (TrustServices,
  4413620396) lists the `ledger-live-development` and `ledger-live-staging`
  Firebase projects as the sources of feature flags.
- "Feature Flagging" (WALLETCO, 3549298959): "Firebase remote config (STAGING
  and PRODUCTION), add the feature with a default config of `{enabled:
  false}`. Add the default config `{enabled: false}` in Ledger Live Common
  `src/featureFlags/defaultFeatures.ts`."
- "Feature Flags and Firebase Config Models" (QA, 6896255151): "Production
  Firebase Remote Config changes are monitored in the Slack channel
  `#qa-b2c-releases-bugs-tracking`."
- "SWAP" tech page (PTX, 3804496035) lists per-env Swap Live App backends:
  `swap-stg.ledger-test.com/v5`, `swap-ppr.ledger-test.com/v5`, and the
  distinct Firebase project `swap-live-app` for Swap-local flags.

Typical Live App promotion, mapped to QA checkpoints:

1. Dev: developer runs the Live App on `localhost:3000`, hard-codes the dev
   manifest into `apps/ledger-live-mobile/src/screens/PTX/index.tsx` or uses
   an `adb reverse tcp:3000 tcp:3000` bridge for Android (see "Debug live
   app locally", QA 6704562204). `isWhitelistedDomain` may be forced to
   `true` to avoid CORS/domain rejection. Feature flag can be overridden
   locally.
2. Staging Firebase: manifest lands in `manifest-api` staging branch,
   Firebase staging flips `feature_*` on with `{enabled: true}`. QAA runs
   Speculos E2E + manual regression on nightly LLD / LLM.
3. Prerelease Firebase: manifest promoted, Firebase prerelease flag still
   gated (rollout_percentage, platform conditions). Release candidates are
   validated here; bugs open QAA tickets (QA ticket priority levels, QA
   6651019308).
4. Production Firebase: flag flipped to `{enabled: true}` in
   `ledger-live-production`. Changes are monitored in
   `#qa-b2c-releases-bugs-tracking`. The Swap Live App itself is deployed on
   Vercel and follows a separate `swap-live-app` Firebase for its internal
   flags.

Tie to R3: R3 covers the Ledger Live desktop/mobile pipeline. R6 reuses R3's
environment list; the Live App layer only adds a second configuration surface
(the `manifest-api` repo + per-env manifest selection) on top of the Firebase
Remote Config flags R3 already describes.

---

## 3. Coin Framework model

The Coin Framework is the set of abstractions and libraries that lets Ledger
Live (both LLD and LLM) talk to any blockchain. The canonical page is
"Bridges" (CIP, 5703696635):

> "Bridges are the standard interface between blockchain logic and front end
> clients (LLD, LLM, CLI). They are included in coin-modules. Implementing
> this interface correctly represents 90% of a blockchain integration (the
> rest will be icons / metadata)."

### 3.1 Core abstractions

- Family: the namespace for one blockchain (e.g. bitcoin, ethereum, polkadot,
  solana, cardano). Historically each family was a folder under
  `libs/ledger-live-common/src/families/<family>`. Since the CoinModule
  migration (CF page 4052942890 — "Convert coin families directory into
  CoinModule"), each family is a separate package under
  `libs/coin-modules/coin-<family>` (source of truth) with a thin passthrough
  in `live-common/src/families/<family>` for compatibility.
- Account: the per-user, per-address state for a given family. Account ids
  follow a convention documented in "Coin modules - ADR-009 - LL Account
  ids" (CF, 6266290256). The shape is produced by `getAccountShape` during
  sync.
- Operation: a historical transaction on an account. Produced during sync;
  can be send / receive / delegation / reward / fee; mapped from the
  blockchain's native tx format by the coin module.
- Bridge: the standardized interface that every family exposes to
  LLD/LLM/CLI. There are two halves (per CIP 5703696635):
  - CurrencyBridge: currency-wide logic — `preload`, `hydrate`,
    `scanAccounts`, `getPreloadStrategy`. Called once per currency, not per
    account.
  - AccountBridge: per-account logic — `sync`, `receive`,
    `createTransaction`, `updateTransaction` / `prepareTransaction`,
    `getTransactionStatus`, `estimateMaxSpendable`, `signOperation`,
    `broadcast`.
- Signer: the side-effectful "talk to the device" layer. Injected into the
  bridge at instantiation so the coin-module itself stays signer-agnostic.
  This is why the same coin-module can be driven by a real Ledger via
  hw-app-xxx, by the Device Management Kit (DMK), or by a mock for testing.
  Pattern documented in CF page 4052942890.
- Synchronization: concrete process driven by `AccountBridge.sync`.
  Guidelines (CIP 5703696635):
  - `sync` is called regularly, so must be optimised for large histories.
  - Incremental sync is mandatory (fetch from latest tx, not from 0).
  - `preloadMaxAge` must be set so currency-wide data is cached.
  - Use tx simulation (not constants) to estimate gas where possible.

### 3.2 The Alpaca generic adapter

"Alpaca" is the newer, generic bridge that factors out the common 90% of a
bridge so coin modules only implement blockchain-specific logic through a
standard AlpacaApi (CF 6133776399 "Alpaca Generic Adapter"):

AlpacaApi methods (excerpt): broadcast, combine, estimateFees,
craftTransaction, getBalance, lastBlock, getBlockInfo, getBlock,
listOperations, optional getStakes, optional getRewards.

BridgeApi methods: validateIntent, getSequence, optional
getChainSpecificRules, optional getTokenFromAsset, optional getAssetFromToken.

Integration points for adding a coin to the generic adapter:
- `libs/ledger-live-common/src/bridge/generic-alpaca/alpaca/index.ts` —
  register `create{Coin}Api` in `getAlpacaApi`.
- `libs/ledger-live-common/src/bridge/generic-alpaca/signer/index.ts` —
  expose an `AlpacaSigner` in `getSigner`.
- `libs/ledger-live-common/src/bridge/generic-alpaca/signer/signTransaction.ts`
  — add `{Coin}signTransaction`.
- `libs/ledger-live-common/src/bridge/generic-alpaca/createTransaction.ts` —
  add the family to the switch.
- `libs/ledger-live-common/src/bridge/impl.ts` — set the family to `true` in
  the `alpacaized` map.

Generic functions (`genericPrepareTransaction`, `genericGetTransactionStatus`,
`genericGetAccountShape`, `genericEstimateMaxSpendable`,
`genericSignOperation`, `genericBroadcast`) live in
`libs/ledger-live-common/src/bridge/generic-alpaca/` and handle fee
estimation, validation, sync, max-spendable, signing and broadcast in a
coin-agnostic way.

### 3.3 Where each family stands

"[Coin Framework] Coin Modules Developments Mapping" (CF, 6129025043) tracks,
per family, five core migrations:
- Coin tester support: only bitcoin, evm, polkadot, solana done.
- DMK signer support: bitcoin, evm, solana done.
- Alpaca API exposure: 12 of 29 done, 4 in progress.
- Generic adapter integration: 4 of 29 done (evm, stellar, tezos, xrp).
- Extracted from `ledger-live` monorepo into `alpaca-coin-module` repo: only
  xrp done; the rest are still in-tree.

This is QA-relevant: the test path for an Alpaca-integrated family is quite
different from the test path for a legacy in-tree family (see section 5).

---

## 4. New-coin integration flow — from request to shipped

Canonical workflow (Device Apps space NA, page 6852771877 — "Coin integration
: integration process"):

1. Commercial setup & SOW. Ledger BD signs with the foundation/partner. The
   SOW lists deliverables (Send/Receive, tokens, Staking, CEX swap, Buy/Sell,
   DApp via WalletConnect, etc.), prerequisites, and target timing (typically
   3 to 6 months).
2. Jira Product Discovery (JPD). Each SOW item becomes a Polaris idea in the
   BI project (`jira/polaris/projects/BI/ideas/view/5893188`), with delivery
   epics linked as children.
3. Epic split by product area. For each JPD idea, at least two delivery epics
   are created, typically:
   - `[BlockchainX][LW] XCoin - Send/Receive` on the LIVE board (Ledger
     Wallet).
   - `[BlockchainX][Device App] XCoin - Send/Receive` on the NAPPS board
     (Device App team).
4. Kickoff + PRD + (sometimes) HLD in Confluence, in the BI space.
5. User stories & engineering tasks under each epic. If BST support is
   required, the epic is prioritised in BST PI planning.
6. Implementation:
   - Device app (firmware-side app for the family): lives in its own
     `app-<family>` repo; signs payloads over APDU.
   - Coin module (TypeScript): the family's bridge + signer + logic, as in
     section 3. See CF 4052942890 for the conversion-to-CoinModule checklist
     and CF 6133776399 for the Alpaca variant.
   - CAL (Crypto Asset List) updates: native coin coefficient, token
     signatures. See PTX Coin integration checklist (4891541509).
   - Firebase feature flags `feature_currency_<currency>` and
     `config_currency_<currency>` (CF 6898778113) gate the new currency
     per-platform and per-network.
7. QA Process (CF 3644621458 — "QA Process for new integrations"):
   - Step 1 — Kickoff (30 min): product demo of scope (sync / send / receive
     / tokens / staking / NFT), main reference wallet, funds in the native
     currency. QAA added to all channels.
   - Step 2 — Test plan & dataset (2-3 days): QAA discovers the coin's
     specifics using the reference wallet and funds, writes the test plan +
     dataset, opens a PR on the partner repo and communicates via
     Slack/Discord. End of step = Definition of Ready met; dev can start.
   - (Later steps not covered in the fetched page; likely execution, bug
     bash, sign-off, monitoring. Gap, see section 9.)
8. Shipping via feature flag. `feature_currency_<name>` is flipped per
   environment, with conditional rules (iOS / Android / Desktop / rollout
   percentage). Prod changes are monitored in
   `#qa-b2c-releases-bugs-tracking` (QA 6896255151).
9. Post-launch monitoring. Entries in CF 6129025043 (A4 instant sync, dynamic
   tokens, CAL lazy loading).

QAA is involved from Step 1 onwards in the QA Process, and from the Epic
split stage onwards in the macro flow. The critical handover to QAA is at
kickoff (step 7.1).

---

## 5. Testing surface — what QAA tests at the Coin Framework layer

Per "QA current state: Coin Integration" (QA, 6637191414) the test surface is
explicitly layered:

- Unit tests on coin modules (owned by Dev): pure logic, transaction
  crafting, parsing. QAA gates coverage.
- Legacy coin bot (shared): send/receive + mutations with real seed. "Not
  maintained, not monitored" today; QAA consumes results.
- Coin tester (Dev + QAA): mock-integration tests at the Bridge level (HTTP
  calls), deterministic. QAA ramps up coverage; currently only EVMs + DOT
  (plus Bitcoin, Solana per CF 6129025043).
- Backend tests (BE team): endpoint snapshots. QAA flags that only ~5% of
  endpoints are Ledger-owned; public endpoints are monitored externally.
- UI automation (Speculos) (QAA): account add + send/receive; staking in
  progress. Nightly, core QAA activity.
- Manual / XRAY (QAA): documented test cases. Still too reliant on this;
  doesn't scale.

Bridge unit vs E2E vs device mock:
- Bridge unit / integration tests run under Jest with a mocked HTTP layer.
  CF 4052942890 describes the `bridge.integration.test.ts` pattern, with a
  `testBridge(dataset)` helper and per-family datasets. For Alpaca families,
  tests live at
  `libs/ledger-live-common/src/bridge/generic-alpaca/tests/*.test.ts` and
  are registered per family (CF 6133776399).
- E2E / Speculos: simulated device signing against a real LLD / LLM build.
  Runs nightly against the staging Firebase.
- Device mock (DMK): newer path via the Device Management Kit
  (github.com/LedgerHQ/device-sdk-ts). "How to :: Mock DMK" (CIP, 6551306298)
  shows the `CreateSigner<COIN_NAMESigner>` pattern used to inject a mock
  `dmk` into the `Transport` for tests. Only bitcoin, evm and solana have
  DMK signer today.

Coin tester architecture (ZCash example in CIP 6973653158) makes the sync
flow unit-testable end-to-end without a device, by driving the coin module
through the same entry points Ledger Live uses.

Swap-specific QA surface (PTX 4891541509 — "PTX Coin integration checklist")
adds:
- Major native to new native swaps, major native to new token, major token
  to new token, max-toggle swaps.
- Operation history + swap history: fiat amount, native amount, sub-accounts
  shown.
- Network fee correctness on swap form + device drawer.
- Error paths (NotEnoughGas, minimum balance errors).
- Chain quirks (TRX daily quota, DOT 1-DOT minimum, etc.).

---

## 6. Mapping to the ledger-live monorepo

Where things live in github.com/LedgerHQ/ledger-live:

- `libs/coin-framework` — cross-family types, helpers, signer abstractions.
  Grounding: CF 4052942890 ("Adapt imports to point to
  `@ledgerhq/coin-framework`").
- `libs/coin-modules/coin-<family>` — per-family bridge + signer + crafting
  logic. Source of truth for the family. Grounding: CF 4052942890.
- `libs/ledger-live-common/src/bridge/generic-alpaca` — generic Alpaca bridge
  (5 files per new family, section 3.2). Grounding: CF 6133776399.
- `libs/ledger-live-common/src/bridge/impl.ts` — registers `alpacaized`
  families. Grounding: CF 6133776399.
- `libs/ledger-live-common/src/families/<family>` — legacy home + UI-only
  files (`react.ts`, `banner.ts`) + passthrough after extraction. Grounding:
  CF 4052942890.
- `libs/ledger-live-common/src/wallet-api` — Wallet API server-side logic,
  domain allowlist helpers. Grounding: CIP 5389025304; QA 6704562204
  (`isWhitelistedDomain` in `helpers.ts`).
- `libs/ledger-live-common/src/exchange` — Exchange / swap / sell enablement.
  Grounding: CIP 5389025304.
- `libs/ledger-live-common/src/featureFlags/defaultFeatures.ts` — default
  `{enabled: false}` for every flag. Grounding: WALLETCO 3549298959.
- `libs/ledger-live-common/src/e2e` — Speculos E2E harness. Grounding: CIP
  5389025304.
- `libs/ledger-live-common/src/bot` — legacy coin bot. Grounding: CIP
  5389025304.
- `libs/ledger-live-common/src/deviceSDK`, `ble`, `hw`, `socket` — device
  transports. Grounding: CIP 5389025304.
- `apps/ledger-live-desktop/src/renderer/components/FirebaseRemoteConfig.tsx`
  — Firebase wiring on LLD. Grounding: WALLETCO 3549298959.
- `apps/ledger-live-mobile/src/screens/PTX/index.tsx` — PTX (Swap/Buy/Sell
  Live App) entry point. Grounding: QA 6704562204.

External repos referenced by the platform:
- github.com/LedgerHQ/manifest-api — manifest JSONs per environment.
- github.com/LedgerHQ/swap-live-app — the Swap Live App itself.
- github.com/LedgerHQ/exchange-sdk — Live App to Wallet API helper.
- github.com/LedgerHQ/wallet-api — Wallet API protocol + client SDK.
- github.com/LedgerHQ/ledger-live-assets — CDN assets (provider icons,
  provider config).
- github.com/LedgerHQ/swap — Scala swap service + completer.
- github.com/LedgerHQ/swap-configuration — Rust/CSV whitelists per provider
  and LL version.
- github.com/LedgerHQ/alpaca-coin-module — the extracted CoinModule repo
  (CF 6129025043; today only xrp has been fully migrated out).
- github.com/LedgerHQ/device-sdk-ts — DMK (Device Management Kit).

---

## 7. Swap Live App — the flagship Live App (why it's its own Part 7)

The Swap Live App is the canonical example of a Live App, which is why it
deserves its own part of the onboarding guide rather than being folded into
the generic Live Apps section. Reasons, all grounded:

1. It is Ledger's own replatformized Swap UI, not a partner's. The "SWAP"
   tech page (PTX 3804496035) lists it as an internal Ledger repo
   (github.com/LedgerHQ/swap-live-app) deployed on Vercel, owned by the PTX
   team, with its own Sentry, Datadog and Mixpanel surfaces and a dedicated
   Firebase project `swap-live-app` for its feature flags.

2. It exercises every piece of the Live Apps platform end-to-end. Per
   "Partner integration SWAP - Frameworks Overview" (PTX 6287622174), it
   uses:
   - `start exchange` / `complete exchange` wallet-api methods.
   - The Exchange SDK (github.com/LedgerHQ/exchange-sdk).
   - The Exchange device app to sign payloads.
   - Ledger Live drawers (the right-panel UI).
   - Manifest API for its per-env manifests (dev, staging, ppr, prod).

3. It is where Ledger Live and the Coin Framework meet. "Swap ::
   Implementation Strategy" (CIP 5392760907) makes this explicit: enabling
   swap for a new coin requires changes on both the Device App and Ledger
   Live / coin-module sides. The Live App itself is the user-visible
   surface; the plumbing goes through the Exchange device app, the
   wallet-api, the Exchange SDK, and the coin module's signer.

4. It has the most mature partner-integration framework catalogue. PTX
   6287622174 lists five frameworks (Centralized Native, DEX-in-CEX, Native
   DEX, and two deprecated Dapp frameworks), each with different wallet-api
   calls, different device-app surfaces, and different QA checklists.

5. Multi-environment behaviour is the most visible here. "TON Swap
   integration" (PTX 5134876674) shows the concrete feature-flag incantation
   to target `swap-live-app-demo-3-stg`. Swap has its own staging and ppr
   backends (`swap-stg.ledger-test.com/v5`,
   `swap-ppr.ledger-test.com/v5`).

6. QA surface is distinct. PTX 4891541509 "PTX Coin integration checklist"
   is 100% Swap-specific: firmware-to-PTX address implementation, CAL token
   signatures, swap-specific network fee paths, chain quirks. This does not
   fit in the generic new-coin QA process (CF 3644621458).

7. Incident surface is non-trivial. Recent Jira entries (TSDINT-1041 for the
   React2Shell RCE on swap-live-app; TSD-8864 where deeplink security
   hardening broke partner flows; PROD-10776 for an ETH-to-DOT signature
   verification failure) show that Swap is a primary attack and regression
   surface.

Practical consequence for QAA onboarding: to understand Live Apps, read the
Swap Live App tech page first; it's the most complete real-world example.

---

## 8. Top 8 Confluence URLs to cite in the onboarding guide

1. Manifest V2 (WAL) —
   https://ledgerhq.atlassian.net/wiki/spaces/WAL/pages/3924230200/Manifest+V2
2. Ledger Live (CIP architecture) —
   https://ledgerhq.atlassian.net/wiki/spaces/CIP/pages/5389025304/Ledger+Live
3. Bridges —
   https://ledgerhq.atlassian.net/wiki/spaces/CIP/pages/5703696635/Bridges
4. Alpaca Generic Adapter —
   https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/6133776399/Alpaca+Generic+Adapter
5. Convert coin families directory into CoinModule —
   https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/4052942890/Convert+coin+families+directory+into+CoinModule
6. Coin integration : integration process —
   https://ledgerhq.atlassian.net/wiki/spaces/NA/pages/6852771877/Coin+integration+integration+process
7. QA Process for new integrations —
   https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/3644621458/QA+Process+for+new+integrations
8. Swap :: Implementation Strategy —
   https://ledgerhq.atlassian.net/wiki/spaces/CIP/pages/5392760907/Swap+Implementation+Strategy

---

## 9. Gaps

What I could not confirm from Confluence:

- Full child tree of the WAL (Wallet API) and CF (Coin Framework) spaces.
  `getConfluencePageDescendants` and `getConfluenceSpaces` were denied by
  permission in this session; I used Rovo search as a proxy. Deep crawling
  (children of a space root) was not possible. A follow-up with those
  permissions would surface e.g. the WAL space homepage and its full page
  tree (the developers.ledger.com/docs/live-app/start-here/ external doc is
  referenced from Manifest V2 but not fetched).
- QA Process full flow. CF 3644621458 only documents Steps 1 and 2 (kickoff
  + test plan). Steps 3+ (execution, bug bash, sign-off, definition of done
  for integration QA) are not in the fetched page. Likely exists on a child
  page or a newer Confluence page.
- Wallet API method inventory. The Wallet API has a formal method list
  (account.list, account.request, transaction.sign, message.sign, bitcoin.*,
  exchange.start, exchange.complete, etc.) but I did not find a Confluence
  page enumerating the current protocol; it's likely in the external
  developers.ledger.com doc site or in the wallet-api README. Worth a
  follow-up via GitHub / dev portal.
- The "prerelease" Firebase environment. The fetched pages show dev,
  staging and production explicitly; "prerelease" is referenced in the
  prompt but in Confluence it's usually called "pre-prod" or "ppr"
  (swap-ppr.ledger-test.com). Exact mapping of prerelease to Firebase
  project should be cross-checked with R3's findings.
- Dynamic token / CAL lazy loading. Referenced in CF 6129025043 but the full
  ADR (TA 5603262535) was not fetched in this session.
- Coin tester full architecture. CIP 6973653158 (ZCash coin tester) was
  surfaced but not fetched; CF 4779802648 (generic coin tester) was also
  not fetched.

---

## 10. Raw links

Confluence pages fetched in this research:
- WAL / Manifest V2 — 3924230200
- CIP / Ledger Live (architecture) — 5389025304
- CIP / Bridges — 5703696635
- CIP / Swap :: Implementation Strategy — 5392760907
- CIP / Swap playbook — 5389287452 (body was empty on fetch)
- CF / Convert coin families directory into CoinModule — 4052942890
- CF / Alpaca Generic Adapter — 6133776399
- CF / [Coin Framework] Coin Modules Developments Mapping — 6129025043
- CF / QA Process for new integrations — 3644621458
- CF / Coin Integration Feature Flags in Prod — 6898778113
- PTX / Partner integration SWAP - Frameworks Overview — 6287622174
- PTX / PTX Coin integration checklist — 4891541509
- PTX / [SWAP] tech page — 3804496035
- NA / Coin integration : integration process — 6852771877
- QA / Debug live app locally — 6704562204
- QA / QA current state: Coin Integration — 6637191414

Confluence pages surfaced via search but not fetched (candidates for later):
- CF / Coin modules - ADR-002 - Account deployment — 6060507358
- CF / Coin modules - ADR-009 - LL Account ids — 6266290256
- CF / Coin modules - ADR-023 - Reorganize generic-alpaca by Coin Family
  — 6925549644
- CF / Coin modules - ADR-025 - Coin modules migration to Alpaca — 6949896196
- CF / [Coin Framework] Weekly Refinement — 5855281381
- CF / Coin Modules Upkeeping — 6826492033
- CF / Coin modules - ADR-001 - Staking API design — 6029213697
- CIP / How to :: Mock DMK — 6551306298
- CIP / ZCash coin tester architecture — 6973653158
- CIP / Zcash QA :: Synchronization process — 6946422817
- PTX / TON Swap integration — 5134876674
- PTX / How to debug swap-live-app and exchange-sdk — 4620845099
- PTX / SWAP Errors Improvement Task Force — 6092652762
- PTX / [Swap] - Functional documentation - CEX live app integration
  — 4160520193
- PTX / [Tech] SWAP — 4025778729
- PTX / Tutorial Feature flags — 4059496451
- WALLETCO / Feature Flagging — 3549298959
- WALLETCO / Feature Flag Issues — 6883016802
- TrustServices / 1 - Firebase Feature Flags in Ledger Live and Recover
  — 4413620396
- QA / Feature Flags and Firebase Config Models — 6896255151
- QA / Ledger Live environments — 6637191253
- QA / QA - Tickets Priority levels — 6651019308
- LLI / Ledger Wallet (Ledger Live) — 5644255440
- LLI / How to LQA Ledger Wallet (Ledger Live) — 3545759821
- NFT / Testing the [L] Market Live App in Ledger Live — 4084171039
- ENGB2C / Live - Wallet API Weekly Update — 4037181769
- BI / ADR 027: Firebase Remote Config integration — 7041220899
- BI / ADR 27.2: dApp config overrides via Firebase Remote Config
  — 7042039963
- VAUL / Engineering - Vault Coin Integration Playbook — 5738037406
- VAUL / New Network Integration Guide — 3710878039

End of R6 research note.
