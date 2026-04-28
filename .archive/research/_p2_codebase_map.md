# Part 2 — Ledger Wallet Features — codebase map

**Goal**: dispatch 9 chapter writers efficiently. Each writer gets the inventory below + a focused file list of what to read for their specific chapter.

## Repo

`/Users/jerome.portier/src/tries/2026-04-08-LedgerHQ-ledger-live`

## Top-level layouts

### Desktop screens — `apps/ledger-live-desktop/src/renderer/screens/`
account, accounts, asset, bank, card, customImage, dashboard, earn, exchange, manager, market, platform, recover, settings, stake, swapWeb, USBTroubleshooting

### Mobile screens — `apps/ledger-live-mobile/src/screens/` (legacy)
Account, Accounts, AccountSettings, Analytics, Assets, BleDevicePairingFlow, ClaimRewards, CustomImage, DeviceConnect, **Discover**, **FirmwareUpdate**, **Manager**, MyLedgerChooseDevice, MyLedgerDevice, Onboarding, OperationDetails, Platform, **Portfolio**, PostOnboarding, Protect, **PTX**, **ReceiveFunds**, **SendFunds**, Settings, SignMessage, SignRawTransaction, SignTransaction, etc.

### Mobile MVVM screens — `apps/ledger-live-mobile/src/mvvm/features/` (newer)
Accounts, AssetDetail, Assets, **Borrow**, **Buy**, Card, Crypto, CryptoAddresses, DeeplinkInstallApp, DeviceSelection, FirmwareUpdate, FlowWizard, LandingPages, LaunchScreen, LedgerSyncEntryPoint, Market, MarketBanner, MemoTag, ModularDrawer, **MyWallet**, NftEntryPoint, Noah, NotificationsPrompt

### Coin modules — `libs/coin-modules/`
30 families: aleo, algorand, aptos, bitcoin, canton, cardano, casper, celo, concordium, cosmos, **evm**, filecoin, hedera, icon, internet_computer, kaspa, mina, multiversx, near, polkadot, solana, stacks, stellar, sui, tezos, ton, tron, vechain, zcash-shielded, plus coin-module-boilerplate

### Live-common families — `libs/ledger-live-common/src/families/`
28 families (similar list, includes xrp which isn't in coin-modules — it's still in live-common)

### Desktop POMs — `e2e/desktop/tests/page/`
- `account.page.ts` (9.2k) — account view, send/receive entry points
- `accounts.page.ts` (5.1k) — accounts list
- `assets.page.ts` (2.9k)
- `buyAndSell.page.ts` (14k)
- `earn.base.page.ts`, `earn.dashboard.page.ts`, `earn.v2.dashboard.page.ts`
- `market.page.ts` (4k)
- `portfolio.page.ts` (12k)
- `settings.page.ts` (6k)
- **`swap.page.ts` (22k)** — biggest POM; covers swap + token approval + revoke
- `webViewApp.page.ts` (6k) — Live App webview
- subdirs: `dialog/`, `drawer/`, `modal/`

### Mobile POMs — `e2e/mobile/page/`
- top-level: `common.page.ts`, `passwordEntry.page.ts`, `speculos.page.ts`, `error.page.ts`
- subdirs: `accounts/`, `discover/`, `drawer/`, `liveApps/`, `manager/`, `market/`, `onboarding/`, `settings/`, `stax/`, **`trade/`** (send/receive/swap), **`wallet/`** (portfolio/navigation)

## Per-feature file pointers (for writers)

### 2.1 Send and Receive
**Desktop UI**: `apps/ledger-live-desktop/src/renderer/screens/account/`, plus modals at `apps/ledger-live-desktop/src/renderer/modals/Send/`, `Receive/`
**Mobile UI (legacy)**: `apps/ledger-live-mobile/src/screens/SendFunds/`, `ReceiveFunds/`
**Common**: `libs/ledger-live-common/src/families/<family>/transaction.ts`, `bridge/js.ts`, plus the cross-family `transaction/index.ts`
**Coin specifics**: `libs/coin-modules/coin-evm/src/transaction.ts`, `coin-bitcoin/src/`, `coin-solana/src/`
**POMs**: `e2e/desktop/tests/page/send.page.ts` (or part of account.page.ts), receive equivalents; `e2e/mobile/page/trade/send.page.ts`, `e2e/mobile/page/trade/receive.page.ts`
**Specs**: `e2e/desktop/tests/specs/send.tx.spec.ts`, `newSendFlow.tx.spec.ts`, `receive.address.spec.ts`

### 2.2 Portfolio and Countervalues
**Desktop**: `apps/ledger-live-desktop/src/renderer/screens/dashboard/`, `accounts/`, `asset/`
**Mobile (legacy)**: `apps/ledger-live-mobile/src/screens/Portfolio/`, `Account/`, `Accounts/`, `Assets/`
**Mobile (MVVM)**: `apps/ledger-live-mobile/src/mvvm/features/Assets/`, `AssetDetail/`, `MyWallet/`, `Crypto/`
**Common**: `libs/ledger-live-common/src/portfolio/`, `countervalues/`, `account/`
**POMs**: `e2e/desktop/tests/page/portfolio.page.ts`, `accounts.page.ts`, `assets.page.ts`; `e2e/mobile/page/wallet/portfolio.page.ts`, `mainNavigation.page.ts`, `accounts/`
**Specs**: portfolio-related, account-list, balance display

### 2.3 Stake and Earn
**Desktop**: `apps/ledger-live-desktop/src/renderer/screens/earn/`, `stake/`
**Mobile (legacy)**: `apps/ledger-live-mobile/src/screens/ClaimRewards/`, plus `EarnInfoModal/`
**Mobile (MVVM)**: anything Earn-related
**Common**: `libs/ledger-live-common/src/families/{solana,cardano,cosmos,polkadot,tezos,near,multiversx,celo,tron}/staking/` and similar paths per family
**POMs**: `e2e/desktop/tests/page/earn.base.page.ts`, `earn.dashboard.page.ts`, `earn.v2.dashboard.page.ts`, `modal/delegate.modal.ts`; `e2e/mobile/page/trade/celoManageAssets.page.ts`, `trade/stake.page.ts`, `trade/earnDashboard.page.ts`, `trade/earnV2Dashboard.page.ts`
**Specs**: `e2e/desktop/tests/specs/earn.spec.ts`, `earn.v2.spec.ts`, `delegate.spec.ts`; mobile earn/delegate specs

### 2.4 Swap (overview chapter; deep dive stays in current Part 6 → 7)
**Desktop**: `apps/ledger-live-desktop/src/renderer/screens/swapWeb/` (web-based), `exchange/`
**Mobile**: `apps/ledger-live-mobile/src/screens/PTX/Swap/` (PTX = Post-Transaction eXperience tribe), and the swap-live-app webview
**Common**: `libs/ledger-live-common/src/exchange/swap/`
**POMs**: `e2e/desktop/tests/page/swap.page.ts` (22k — the largest POM), `e2e/mobile/page/trade/swap.page.ts`, `e2e/mobile/page/liveApps/swapLiveApp.page.ts`
**Specs**: `e2e/desktop/tests/specs/provider.swap.spec.ts`, `accounts.swap.spec.ts`, `entrypoint.swap.spec.ts`, `send.swap.spec.ts`, `ui.swap.spec.ts`, `validation.swap.spec.ts`; mobile swap specs

### 2.5 Buy and Sell
**Desktop**: `apps/ledger-live-desktop/src/renderer/screens/exchange/` (covers Buy, Sell)
**Mobile (legacy)**: `apps/ledger-live-mobile/src/screens/Exchange/`, `PTX/`
**Mobile (MVVM)**: `apps/ledger-live-mobile/src/mvvm/features/Buy/`
**Common**: `libs/ledger-live-common/src/exchange/buy/`, `sell/`, `platform/` (Live App integrations)
**POMs**: `e2e/desktop/tests/page/buyAndSell.page.ts` (14k); mobile `e2e/mobile/page/trade/buySell.page.ts`
**Specs**: `e2e/desktop/tests/specs/buySell.spec.ts`
**Providers**: MoonPay, Coinbase Pay, Wyre, Banxa, Transak — partner-app integrations via PTX team

### 2.6 Device Management
**Desktop**: `apps/ledger-live-desktop/src/renderer/screens/manager/`
**Mobile (legacy)**: `apps/ledger-live-mobile/src/screens/Manager/`, `MyLedgerChooseDevice/`, `MyLedgerDevice/`, `BleDevicePairingFlow/`, `FirmwareUpdate/`, `DeviceConnect/`
**Mobile (MVVM)**: `apps/ledger-live-mobile/src/mvvm/features/FirmwareUpdate/`, `DeviceSelection/`
**Common**: `libs/ledger-live-common/src/manager/`, `apps/`, `device-core` lib, Device Management Kit (`@ledgerhq/device-management-kit`)
**POMs**: `e2e/mobile/page/manager/`, `e2e/desktop/tests/page/...` (look for manager-related)
**Specs**: `e2e/desktop/tests/specs/...manager...`, mobile equivalents

### 2.7 Discover / Web3 / WalletConnect
**Desktop**: `apps/ledger-live-desktop/src/renderer/screens/platform/`
**Mobile (legacy)**: `apps/ledger-live-mobile/src/screens/Discover/`, `Platform/`
**Common**: `libs/ledger-live-common/src/wallet-api/`, `live-apps/`, `platform/`
**Live Apps**: Manifest API repo (`github.com/LedgerHQ/manifest-api`), wallet-api-client packages
**WalletConnect**: search for `walletconnect`, `WalletConnect` across apps and libs
**POMs**: `e2e/desktop/tests/page/liveApp.page.ts`, `webViewApp.page.ts`; `e2e/mobile/page/discover/`, `liveApps/`
**Specs**: wallet-api specs in mobile, webview/discover specs

## Repo-wide search patterns for writers

```bash
# All references to a feature term
grep -rln "<term>" apps/ libs/ e2e/ | grep -v node_modules

# Find where a screen is rendered
find apps/ledger-live-desktop/src -name "*.tsx" | xargs grep -l "ScreenName"

# Spec coverage
grep -rln "swap\|Swap" e2e/desktop/tests/specs/ e2e/mobile/specs/
```

## Notes for writers

- This is a **product/architecture** part, not a testing part. Aim chapters at someone learning what Ledger Wallet does and where each feature lives in the code, with QA-relevant tooling (POMs, specs) called out as they appear.
- Use `<photo of …>` placeholders explicitly so the user can replace with real screenshots later.
- Tables are encouraged for: feature variants, file paths per feature, POM method catalogs, supported chains.
- Mermaid diagrams for state machines / user flows where they help (e.g., the send flow has clear states: pick recipient → enter amount → review → device confirmation → broadcast → success).
- Cross-link forward to:
  - Part 3/4 for how this feature is exercised in E2E
  - Part 6 (CLI) for any CLI hook that touches this feature
  - Current Part 6 (Swap deep dive, will become Part 7) for swap specifics
- NO emojis. NO Co-Authored-By.
