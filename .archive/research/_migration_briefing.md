# Mobile E2E Migration — Rewrite Briefing

As of 2026-04-22, the mobile E2E codebase has migrated from `apps/ledger-live-mobile/e2e/` to a top-level Nx/Turbo peer workspace at `e2e/mobile/`.

## Authoritative paths

- **Canonical mobile E2E workspace**: `/Users/jerome.portier/src/tries/2026-04-08-LedgerHQ-ledger-live/e2e/mobile/`
  - Own `package.json`, `project.json` (Nx), `detox.config.js`, `jest.config.js`, `jest.environment.ts`, `jest.globalSetup.ts`, `jest.globalTeardown.ts`, `babel.config.js`, `tsconfig.test.json`
  - 197 `.spec.ts` files under `specs/`
  - `bridge/` (307-line server.ts, differs from legacy — canonical is here)
  - `page/index.ts` — the `Application` class (lazy-init POM hub)
  - `page/` subdirs: `accounts/`, `discover/`, `drawer/`, `liveApps/`, `manager/`, `market/`, `onboarding/`, `settings/`, `stax/`, `trade/`, `wallet/` — plus top-level `common.page.ts`, `passwordEntry.page.ts`, `speculos.page.ts`, `error.page.ts`
  - `models/`, `helpers/`, `userdata/`, `scripts/`, `types/`, `utils/`

- **Legacy (shrinking, to be migrated)**: `/Users/jerome.portier/src/tries/2026-04-08-LedgerHQ-ledger-live/apps/ledger-live-mobile/e2e/`
  - 17 remaining specs: `market.spec.ts`, `wallet-api.spec.ts`, `unknown-currency-resilience.spec.ts`, `manager.spec.ts`, `deeplinks.spec.ts`, `onboarding.spec.ts`, `onboardingReadOnly.spec.ts`, `password.spec.ts`, `languageChange.spec.ts`, `addAccounts/addAccount.spec.ts`, `swap/dexSwap.spec.ts` (skipped), `delegate/cosmos.spec.ts`, `delegate/delegate.spec.ts`, `receive/receiveFlow.spec.ts`, `receive/currencies.spec.ts`, `send/send.spec.ts`, `send/currencies.spec.ts`
  - Own `setup.ts`, `jest.config.js`, `jest.environment.ts`, `jest.globalSetup.ts`, `tsconfig.test.json`, `babel.config.detox.js`
  - Also has `bridge/`, `helpers/`, `mocks/`, `models/`, `page/`, `userdata/` — parallel to the canonical workspace
  - Reason tests linger here: not all have been ported yet; framework infra overlaps during transition.

## ENVFILE names — UNCHANGED

Detox still uses `apps/ledger-live-mobile/.env.mock` and `apps/ledger-live-mobile/.env.mock.prerelease` (names unchanged; these files live under the app directory, not the workspace). Verified in `e2e/mobile/detox.config.js`:

```js
const ENV_FILE_MOCK = path.join("apps", "ledger-live-mobile", ".env.mock");
const ENV_FILE_MOCK_PRERELEASE = path.join("apps", "ledger-live-mobile", ".env.mock.prerelease");
```

## Build / run commands — CHANGED

From `e2e/mobile/package.json` scripts:

| Purpose | Canonical command | Old guide said |
|---|---|---|
| Build iOS debug | `pnpm --filter e2e-mobile run build:ios:debug` | `pnpm mobile e2e:build -c ios.sim.debug` |
| Build Android debug | `pnpm --filter e2e-mobile run build:android:debug` | `pnpm mobile e2e:build -c android.emu.debug` |
| Build iOS release | `nx run live-mobile:e2e:build -- --configuration ios.sim.release` (also `pnpm build:ios` from `e2e/mobile/`) | `pnpm mobile e2e:build -c ios.sim.release` |
| Build Android release | `nx run live-mobile:e2e:build -- --configuration android.emu.release` | `pnpm mobile e2e:build -c android.emu.release` |
| Run iOS (from `e2e/mobile/`) | `pnpm test:ios` (release) or `pnpm test:ios:debug` | `pnpm mobile e2e:test -c ios.sim.debug` |
| Run Android (from `e2e/mobile/`) | `pnpm test:android` or `pnpm test:android:debug` | `pnpm mobile e2e:test -c android.emu.debug` |
| Allure local | `cd e2e/mobile && pnpm allure` (runs `allure generate ./artifacts` then `allure open`) | `pnpm mobile allure` (or equivalent) |
| Typecheck | `pnpm --filter e2e-mobile run typecheck` (invokes `node ./scripts/typecheck.js`) | — |
| Lint | `pnpm --filter e2e-mobile run lint` (uses `oxlint`) | — |

Note: detox still runs from the `e2e/mobile/` directory (the detox config is there), so `pnpm test:detox` resolves to `detox test` with the workspace's config.

## Application POM hub — `e2e/mobile/page/index.ts`

The canonical `Application` class exposes **32 getters** (lazy-initialized POMs). Partial list:
- `assetAccountsPage`, `account`, `accounts`, `addAccount`
- `common`, `customLockscreen`, `deviceValidation`, `discover`
- `ledgerSync`, `manager`, `market`, `onboarding`
- `operationDetails`, `passwordEntry`, `portfolio`, `portfolioEmptyState`
- `receive`, `send`, `settings`, `settingsGeneral`, `settingsHelp`
- `speculos`, `stake`, `swap`, `swapLiveApp` (NEW vs legacy)
- `walletTabNavigator`, `mainNavigation`, `celoManageAssets`
- `transferMenuDrawer`, `buySell`, `earnDashboard`, `earnV2Dashboard`, `modularDrawer`

Key structural features:
- `init(options: ApplicationOptions)` decorated with `@Step("Account initialization")`
- Before init it copies the userdata JSON to a temp file (`temp-userdata-${randomUUID()}`), delegates to `InitializationManager.initialize(options, userdataPath, userdataSpeculos)`, then unlinks
- `getUserdataPath(name)` resolves `userdata/${name}.json` relative to cwd
- `ApplicationOptions` = `InitOptions` (imported from `../utils/initUtil`)

## Bridge — differs from legacy

The canonical `e2e/mobile/bridge/server.ts` is 307 lines (vs 303 in legacy). Contents differ — treat the canonical version as source of truth. Look for the exact message catalog here; **do not** quote the legacy bridge.

## QAA-702 — spec ALREADY EXISTS

`e2e/mobile/specs/swap/otherTestCases/swapExportHistoryOperations.spec.ts` is ~37 lines:

```ts
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
  tags: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5", "@solana", "@family-solana", "@ethereum", "@family-evm"],
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

**Architectural implication**: mobile specs under the canonical workspace follow a **thin-config + shared-driver** pattern. The spec is just a data bundle; the test logic lives in `swap.other` (the driver module alongside the specs). Drivers are `export function runXxx(...)` that invoke `describe()/it()/beforeAll()`.

The pattern is a major departure from the imperative POM-method-call style the old Part 4 Ch 4.10 walkthrough proposes. The existing Ch 4.10 walkthrough is FICTIONAL and must be rewritten around the real architecture.

Shared enums used by swap specs:
- `Account.SOL_1`, `Account.ETH_1`, etc. — from `@ledgerhq/live-common/e2e/enum/Account`
- `Provider.EXODUS`, etc. — from `@ledgerhq/live-common/e2e/enum/Provider`
- `Addresses.SWAP_HISTORY_SOL_FROM`, etc. — from `@ledgerhq/live-common/e2e/enum/Addresses`

The tag pattern (`@NanoSP`, `@family-evm`, etc.) drives CI filters.

## CI workflow — `test-mobile-e2e-reusable.yml`

Inputs include (some added since old Part 4 Ch 4.8/6.3 was written):
- `ref` — branch to test
- `test_filter` — filter by test name / path / tag (comma or pipe separated)
- `tests_type` — `Android Only` | `iOS Only` | `iOS & Android`
- `speculos_device` — `nanoS | nanoSP | nanoX | stax | flex | nanoGen5` (note: `nanoGen5` is NEW)
- `production_firebase` — boolean
- `enable_broadcast` — boolean
- `export_to_xray` — boolean (NEW)
- `test_execution_android` — optional Xray execution key (NEW)
- `test_execution_ios` — optional Xray execution key (NEW)
- `smoke_tests` — boolean
- `enable_wallet40` — boolean (default true)
- `generate_ai_artifacts` — boolean (NEW)

The workflow is a peer to `test-desktop-e2e-reusable.yml` and mirrors its sharding / composite-action pattern.

## Specs directory — 20+ top-level buckets

`e2e/mobile/specs/` subdirectories: `account/`, `addAccount/`, `buySell/`, `delegate/`, `deleteAccount/`, `deposit/`, `earn/`, `ledgerSync/`, `portfolio/`, `send/`, `settings/`, `subAccount/`, `swap/` (plus `swap/otherTestCases/` and `swap/otherTestCases/tooLowAmountForQuoteSwaps/`), `verifyAddress/`, `wallet40/`.

## What changes in the guide (by chapter)

- **Part 4 Ch 4.1 Mobile E2E Architecture** — update all path refs; explain the peer-workspace model; mention the legacy partial migration as context, not primary.
- **Part 4 Ch 4.2 Your First Mobile Test** — rebase on the canonical structure and the thin-spec+driver pattern.
- **Part 4 Ch 4.4 Mobile Toolchain & Env Setup** — commands section: replace `pnpm mobile e2e:...` with `pnpm --filter e2e-mobile run ...` / `nx run live-mobile:e2e:build` / in-workspace `pnpm build:ios` etc. ENVFILE names unchanged.
- **Part 4 Ch 4.6 Detox Advanced** — `detox.config.js` walkthrough must reference `e2e/mobile/detox.config.js`; note ENV_FILE_MOCK constants + cross-workspace paths (`path.join(rootDir, "apps/ledger-live-mobile/ios")`); preserve the configurations list (added `ios.sim.prerelease`, kept `ios.sim.staging`).
- **Part 4 Ch 4.7 Mobile Codebase Deep Dive** — FULL REWRITE. Replace every `apps/ledger-live-mobile/e2e/...` path with `e2e/mobile/...`. Rewrite the Application class contents (32 POMs instead of ~20). Document the driver pattern (`specs/**/swap.other.ts` etc.).
- **Part 4 Ch 4.8 Running & Debugging Mobile** — rewrite run commands; update debug recipes (Metro port; `adb reverse`; simulator selection); bridge port + where to hook.
- **Part 4 Ch 4.9 Daily Mobile Workflow** — update ticket→PR steps to use new commands and paths.
- **Part 4 Ch 4.10 Walkthrough QAA-702 — COMPLETE REWRITE** against the REAL existing spec at `e2e/mobile/specs/swap/otherTestCases/swapExportHistoryOperations.spec.ts`. Read the `swap.other` driver module in the same directory and explain how the 37-line spec declares data and delegates to the shared driver. Teach the pattern.
- **Part 2 Ch 2.5 Allure** — check for mobile-path references.
- **Part 6 Ch 6.3 CI** — mention the NEW workflow inputs (`export_to_xray`, `test_execution_*`, `generate_ai_artifacts`, `nanoGen5` in speculos devices list).
- **Appendix A Commands** — update all mobile commands.
- **Appendix B Env Vars** — noop likely.
- **Appendix F Glossary** — mention the `e2e/mobile/` peer-workspace structure in the E2E entry.

## Rewrite priority

1. **Ch 4.10 QAA-702** — HIGH, fully fictional vs reality; must read the real spec + driver and teach that pattern.
2. **Ch 4.7 Codebase Deep Dive** — HIGH, 1500+ lines of wrong paths and an outdated Application class description.
3. **Ch 4.4 + 4.6 + 4.8** — MEDIUM, factual edits to commands + detox.config walkthrough + run commands.
4. **Ch 4.1 + 4.2 + 4.9** — MEDIUM, architecture overview + first test + daily workflow need path updates.
5. **Part 2/6/Appendix** — LOW, surgical edits.
