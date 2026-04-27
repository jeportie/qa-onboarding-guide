# R5 — Wiki, Agent Skills, and In-Tree READMEs

Mining on-disk Ledger documentation that is NOT in Confluence. Three sources:

1. `/Users/jerome.portier/src/tries/2026-04-14-LedgerHQ-ledger-live.wiki/` — GitHub wiki mirror (snapshot 2026-04-14)
2. `/Users/jerome.portier/src/tries/2026-04-08-LedgerHQ-ledger-live/.agents/skills/` — agent skills shipped in the monorepo
3. In-tree READMEs and docs under the same monorepo

Mission: extract content relevant to the new "CLI Automation" Part — Bunli, DMK, USB transports, account descriptors V1, ERC-20 approve, swap, allowance, hooks, common-commands.

---

## 1. Wiki pages relevant to the CLI Part

The wiki is a small (~25 page) historical doc set. None of it covers `wallet-cli`, Bunli, DMK, V1 descriptors, ERC-20 approve, or swap automation. The wiki still references the **legacy `ledger-live` CLI** (`apps/cli`). Verdict: useful only as background and as evidence that the wiki is **out of date** for CLI Automation.

| Title | Path | Summary | Quotable bits |
| --- | --- | --- | --- |
| Developing with `ledger-live` CLI | `LLC/LLC:dev-with-cli.md` | 17-line page describing how to build/run the **legacy** CLI. Refers to Node 14, `pnpm build:cli`, `pnpm run:cli`. Still the only CLI page in the wiki. | "Ledger Live Common exposes a command line tool (CLI) that can be used to test Ledger Live features in a terminal. To install it, you will need to have an instance of Node 14 (LTS)." |
| Developing | `LLC/LLC:developing.md` | Index page. Points to the CLI page above. | "**You can use `ledger-live` CLI to test your changes**, for this you will need to configure it, follow [Developing with `ledger-live` CLI](./LLC:dev-with-cli)" |
| Hardware Wallet logic | `LLC/LLC:hw.md` | Explains live-common's modular transport system: `registerTransportModule`, `withDevice(id)(job)`, race-condition protection. Predates DMK; mentions `TransportWebHID`, `TransportWebUSB` as examples. Useful background for the chapter that explains how `wallet-cli` registers a DMK module via the same `registerTransportModule` API. | "live-common unifies a modular and multi-transport system that allows to register some 'transport modules' using `registerTransportModule` (typically in your `live-common-setup.js`)." / "you can use the generic `open(id)` function OR if you need a race condition protection, you can use `withDevice(id)(job)`" |
| ledger-live bot | `LLC/LLC:bot.md` | Overview of the Speculos bot: stateless, configless, autonomous transaction engine. **Explicitly notes**: "The bot isn't able to perform swaps". This is the closest "automation framework" reference in the wiki. | "**End to End**: I rely on the complete 'Ledger stack': live-common which is the same logic behind Ledger Live (derives same accounts, use same transaction logic,..) and Speculos which is the Ledger devices simulator!" / "The bot isn't able to perform swaps" |
| LLC: intro | `LLC/LLC:intro.md` | Architecture overview of live-common. Lists CurrencyBridge, AccountBridge, Manager, HW logic. | "This library does not have/is agnostic of: The actual HW transport you use." |
| LLC: AccountBridge / CurrencyBridge / account / derivation | `LLC/*` | Background on the bridge stack that `wallet-cli`'s `BridgeAdapter` wraps. Helpful conceptual background for the chapter on `WalletAdapter` routing. | (architecture only — no CLI specifics) |
| LJS: MigrateWebUSB | `LJS/LJS:MigrateWebUSB.md` | Old transport migration story (U2F → WebUSB). Pre-DMK era. | (historical) |
| _Sidebar | `_Sidebar.md` | Index. **Has zero entries for `wallet-cli`, Bunli, DMK, swap, or ERC-20 approve.** | (proves the gap) |

Wiki gap summary: the wiki has **no page on `wallet-cli`, no page on DMK, no page on Bunli, no page on the V1 descriptor format, no swap or approve automation page**. The only CLI mentioned is the deprecated one. Anything CLI-Automation-related has to come from the in-tree READMEs and agent skills, not the wiki.

---

## 2. Agent skills relevant to the CLI Part

Twenty-four skills live under `.agents/skills/`. Only one is dedicated to `wallet-cli`. A few others are tangentially relevant.

| Skill | Path | Relevance | Summary / quotable bits |
| --- | --- | --- | --- |
| `ledger-wallet-cli` | `.agents/skills/ledger-wallet-cli/SKILL.md` | **Primary source.** Authoritative usage guide: invocation pattern, supported networks, V1 descriptor grammar, command-by-command flags, output envelope, common errors. | "`wallet-cli` is an experimental USB-based CLI for Ledger wallet flows, built on the Device Management Kit (DMK). Supported networks: **bitcoin**, **ethereum**, **base**, **solana** (mainnet and testnets). On Ledger devices, **Base uses the Ethereum app**." / "**Device contention:** Only one command can use the device at a time. Never run two device-required commands in parallel — they will conflict and fail with `[object Object]` or a garbled APDU error." / "**Sandbox:** Commands that open the USB device (`account discover`, `receive`, `send`) **must** be run with `dangerouslyDisableSandbox: true`" / "Ticker is **mandatory** in `--amount` (e.g. `'0.5 ETH'`, `'0.001 BTC'`). It drives asset resolution — no `--token` flag." / V1 grammar `account:1:<type>:<network>:<env>:<xpub_or_address>:<path>` |
| `e2e-swap` | `.agents/skills/e2e-swap/SKILL.md` | Tangentially relevant — covers Playwright/Detox swap E2E, **not** CLI-driven swap. Useful only as the "where swap testing lives today" reference, contrasted against the absence of swap in `wallet-cli`. | "Updating swap tests in `e2e/desktop` or `e2e/mobile`." / "For fragile swap amounts, fetch minimum dynamically with: `app.swap.getMinimumAmount(...)` (desktop), `app.swapLiveApp.getMinimumAmount(...)` (mobile)" |
| `e2e-desktop-onboard` / `e2e-mobile-onboard` | `.agents/skills/e2e-*-onboard/SKILL.md` | Off-topic for CLI but relevant adjacent: "interactive setup wizard for running E2E (Playwright/Detox) tests locally". Contrast point: Bunli `defineCommand` is the wallet-cli analog of these wizards, but for blockchain ops not UI. | "Interactive setup wizard for running desktop E2E (Playwright + Speculos) tests locally" |
| `e2e-desktop-add-or-update` | `.agents/skills/e2e-desktop-add-or-update/SKILL.md` | Mentions Speculos. Adjacent to "wallet-cli + Speculos" hypothetical pairing (which the skill files do **not** cover today). | "Add E2E tests for new coins or update existing Playwright tests for ledger-live-desktop. … Speculos device simulator." |
| `feature-dev`, `pre-review`, `create-pr`, `create-changeset`, `run-tests`, `cleanup`, `fix-ci` | `.agents/skills/*/SKILL.md` | Generic dev workflow skills. Relevant to "how Ledger contributors run/lint/test/PR `wallet-cli`" but contain nothing CLI-specific. | n/a |
| `redux-slice`, `rtk-query-api`, `dialogs-slice`, `ldls-native`, `ldls-web` | `.agents/skills/*/SKILL.md` | UI/state-management skills. Not relevant to CLI. | n/a |
| `zod-schemas` | `.agents/skills/zod-schemas/SKILL.md` | Tangentially relevant — `wallet-cli` uses Zod heavily (intent schemas, session schema, `option(z.string()…)` for Bunli flags). The skill is about generic Zod patterns. Useful when the CLI-Automation chapter explains "every flag is validated by Zod, every output payload validated by Zod". | "Zod schema validation patterns for API types" |
| `testing`, `test-coverage` | `.agents/skills/*/SKILL.md` | Jest/MSW oriented (Desktop/Mobile). Wallet-cli uses **`bun test`**, not Jest, so these don't apply. | "Write unit and integration tests for Ledger Wallet apps. Use for Jest tests (Desktop/Mobile), MSW handlers" |
| `client-ids` | `.agents/skills/client-ids/SKILL.md` | Tangentially relevant — wallet-cli sets `USER_ID="wallet-cli"` to keep the DMK firmware-distribution salt stable (see `live-common-setup.ts`). The skill covers DeviceId, UserId, DatadogId privacy rules. | "Privacy-protected ID management with @ledgerhq/client-ids — DeviceId, UserId, DatadogId, export-rules.json" |

Skill gap summary: aside from `ledger-wallet-cli`, **no skill teaches Bunli, DMK transports, V1 descriptors, ERC-20 approve, or swap automation**. The `e2e-swap` skill is for UI E2E only.

### Full content highlights — `ledger-wallet-cli/SKILL.md`

This is the most load-bearing file we have. Pull-quotes for the chapter:

- **Invocation root**: `pnpm --silent wallet-cli start <command> [flags]`
- **Sandbox table**: `account discover`, `receive`, `send` need device → require `dangerouslyDisableSandbox: true`. `balances`, `operations`, `send --dry-run` do not.
- **V1 descriptor grammar**: `account:1:<type>:<network>:<env>:<xpub_or_address>:<path>` — hardened segments use `h` (shell-safe) but `'` is also accepted.
- **Network forms**: `bitcoin` = mainnet, `ethereum:mainnet` aliased to `main`, `base` (uses Ethereum app), `ethereum:sepolia`, `bitcoin:testnet`, `solana:devnet`.
- **JSON envelope shape**: `{ status, command, network, account, timestamp, ...payload }`. Per-command payload: `{ accounts: [...] }`, `{ balances: [...] }`, `{ operations: [...], nextCursor }`, `{ tx_hash, amount, fee }`.
- **Send flag matrix**:
  - All families: `--amount '<value> <TICKER>'` (mandatory), `--to`, `--dry-run`, `--output json`.
  - Bitcoin only: `--fee-per-byte`, `--rbf`.
  - Solana only: `--mode` (`send`, `stake.createAccount`, `stake.delegate`, `stake.undelegate`, `stake.withdraw`), `--validator`, `--stake-account`, `--memo`.
- **Common error catalogue**:
  - `Amount must include a ticker` → bare number passed to `--amount`.
  - `Ticker UNKN not found in account. Available: ETH, USDT, ...` → unknown asset.
  - `No currencyId mapping for network "x:y"` → unsupported network/env.
  - `UnknownDeviceExchangeError` → device disconnected or wrong app.
  - `[x] Transaction Cancelled: Rejected on device. No funds moved.` → user rejected.

---

## 3. Internal docs (`docs/`, package READMEs)

| Title | Path | Summary | Quotable bits |
| --- | --- | --- | --- |
| Common commands | `docs/common-commands.md` | Top-level cheat sheet for every contributor. **No mention of `wallet-cli`** in `pnpm dev:*`/`build:*`/`test:*` — wallet-cli is invoked via `pnpm wallet-cli start`, which is **not in this file**. Confirms the gap. | "All commands must be run from the **repo root** unless stated otherwise." / `pnpm i --ignore-scripts` is the standard install / `pnpm test:family evm` / `pnpm test:family bitcoin` / `pnpm test:family solana` |
| nx-tags | `docs/nx-tags.md` | Build-system tags (out of scope for CLI). | n/a |
| validate-before-finishing | `docs/dev/validate-before-finishing.md` | Pre-PR checklist. | n/a |
| `apps/wallet-cli/README.md` | in-tree | **Authoritative public-facing README** for wallet-cli. Status, commands, prerequisites, build, env. See section 4 below for full content. | "Experimental command-line tool for Ledger Wallet flows over **USB**, built on the **Device Management Kit (DMK)** and [Bunli](https://www.npmjs.com/package/bunli). Version **0.1.0**" |
| `apps/wallet-cli/src/wallet/README.md` | in-tree | **Internal architecture doc** for the `wallet/` layer. Explains routing, internal structure, integration. | "Clean, serializable wallet interface for the CLI. Sits between the legacy live-common bridge stack and CLI commands. All returned types are plain JSON-safe values (no `BigNumber`, no `Date`, no circular refs)." / **Routing rules:** "**balances** — Alpaca for supported families (fast direct API); bridge sync otherwise. **operations** — always bridge sync. Alpaca is currently bypassed due to known correctness issues (missing internal ops, unreliable pagination); the routing code is preserved as a comment for when it is re-enabled. **everything else** (`discoverAccounts`, `getFreshAddress`, `verifyAddress`, `prepareSend`, `send`) — always bridge sync." / `compatibility/` is internal — do not import from outside `wallet/` |
| `apps/wallet-cli/src/test/commands/README.md` | in-tree | Documents the test contract: each command is a black box, all I/O mocked. | "Each test treats a CLI command as a black box: given known flags and mocked infrastructure, the command must produce the expected stdout shape (JSON envelope) and exit code." / `MockServer` (`WALLET_CLI_MOCK_PORT=<n>`) replaces Ledger Explorer HTTP. `MockDeviceManagementKit` (`WALLET_CLI_MOCK_DMK=1`) replaces USB DMK. App results from `WALLET_CLI_MOCK_APP_RESULTS` (JSON). Run with `bun test src/test/commands/`. |
| `apps/wallet-cli/src/shared/accountDescriptor/README.md` | in-tree | **Authoritative spec** for V0/V1 descriptors and adapters. References the ADR. | "**ADR**: ADR - Account descriptor (Confluence TA/6975946770)" / V0 = `js:2:{currencyId}:{seedIdentifier}:{derivationMode}` (live-common bridge primary key) / V1 = `account:1:<type>:<network_name>:<network_env>:<xpub_or_address>:<path>` / Path conventions: UTXO hardened-only `m/{purpose}'/{coin_type}'/{account_index}'` (suffix `/0/0` intentionally omitted because xpub already encodes it) / EVM `m/44'/60'/{index}'/0/0` / Solana `m/44'/501'/{index}'/0'` / `toV1`, `toV0`, `serializeV1`, `parseV1` adapters. **Limitation**: V1 does not store `freshAddress`; `toV0()` always returns `freshAddress: ""`. |
| `apps/wallet-cli/bunli.config.ts` | in-tree | Build config. | `name: "wallet-cli", version: "0.1.0"`, commands directory `./src/commands`, build entry `./src/cli.ts`, **targets**: `darwin-arm64`, `linux-arm64`, `linux-x64`, `windows-x64`, **minify enabled**. Comment: "The `usb` native addon is embedded via a direct require() in src/embed-usb-native.ts. node-gyp-build uses dynamic resolution that Bun can't detect; the explicit require() fixes that." |
| `apps/wallet-cli/src/embed-usb-native.ts` | in-tree | The "embed USB native into Bun standalone binary" trick. Per-platform `require()` of `usb/prebuilds/*/node.napi.node`. | "Force Bun to embed the `usb` N-API native addon into the standalone binary. … `usb` loads its native binding via node-gyp-build which uses dynamic path resolution (fs.readdirSync) that Bun's bundler cannot detect at compile time. A direct require() makes Bun embed the .node file into the executable, and storing it on globalThis lets the patched usb/dist/usb/bindings.js skip node-gyp-build entirely (see patches/usb@2.9.0.patch)." |
| `apps/cli/README.md` | in-tree | Legacy CLI. See section 6. | n/a |

---

## 4. wallet-cli README — full structure with pull-quotes

The README at `apps/wallet-cli/README.md` is the **single best-formatted reference** for a QA engineer onboarding to CLI Automation. Full structure:

### Top of file
> "Experimental command-line tool for Ledger Wallet flows over **USB**, built on the **Device Management Kit (DMK)** and [Bunli](https://www.npmjs.com/package/bunli). Version **0.1.0** (see `bunli.config.ts`)."
>
> "This software is experimental. It is provided 'as is,' without obligation to develop, support, or repair features. Ledger shall not be liable for damages arising from its use."

### Status (v0)
> "wallet-cli is **not production-ready**. Behavior and flags may change without notice."
>
> "**Supported currencies** today: **bitcoin**, **ethereum**, and **solana** (aligned with `live-common-setup.ts`). This is not full Ledger Live coverage and not multi-chain parity with the desktop or mobile apps."

### Commands table
Five commands with one-line roles. **`account discover`** + **`balances`** + **`operations`** + **`send`** + **`receive`**. Note the README lists `account discover` (sub-command), the SKILL also lists session view/reset and `account fresh-address` (which exist in source — see section 7).

> "Typical flow: run `account discover` with a currency id (e.g. `bitcoin`, `ethereum`), then pass the printed descriptor to `balances`, `operations`, or `send`."

### Discoverability
> Each command supports `--help`. From `apps/wallet-cli`, use `pnpm start` in place of `pnpm wallet-cli start` (same args after `--`).

### Output
> "Most commands support `--output human` (default) or `--output json`."

### Prerequisites
> "**[Bun](https://bun.sh)** ≥ 1.1.0 (`engines` in `package.json`)" / "A **Ledger** on USB when using `account discover`, `send`, or `receive --verify`" / "**Linux:** `sudo apt-get install libudev-dev libusb-1.0-0-dev`"

### Setup and run
> "From the **repository root** (after `pnpm i`): `pnpm wallet-cli start -- <command> [args]`"
>
> "`pnpm start` runs `bun run ./src/cli.ts`. Standalone builds under `dist/` rely on `init-cwd.ts` so Bunli config and native bindings resolve correctly; prefer the package scripts when developing from source."

### Build
> "In `apps/wallet-cli`: `pnpm build` (Bunli native bundle → `dist/`)" / "From repo root: `pnpm build:wallet-cli`"

### Environment
> "If `USER_ID` is unset, it defaults to `wallet-cli` so DMK firmware distribution salt stays stable for this CLI (`env-setup.ts`)."

### Relation to the legacy CLI
> "This package is **private** to the monorepo and DMK-focused. It is **not** the published npm package `@ledgerhq/live-cli` ([`apps/cli`](../cli)); that tool has a different scope and distribution model."

---

## 5. CHANGELOG.md highlights

The wallet-cli CHANGELOG (`apps/wallet-cli/CHANGELOG.md`) is dense (~13 KB) but **the entire content is dependency-update churn**. Functional commits are surfaced only via dependency bumps to live-common, coin-bitcoin, coin-evm, coin-solana, cryptoassets, types-live, live-env, live-dmk-shared, ledger-wallet-framework, hw-transport, errors, live-wallet.

Versions shipped:
- `0.1.0` — initial functional baseline (V1 descriptors, send/receive/balances/operations, dry-run; not in CHANGELOG body but stamped in `bunli.config.ts`).
- `0.1.1` — patch via dependency bumps to live-common@34.69.0, coin-bitcoin@0.38.0, coin-evm@3.4.0, coin-solana@0.51.0, cryptoassets@13.46.0, types-live@6.105.0, live-env@2.33.0, live-dmk-shared@0.22.2.
- `0.1.1-next.0..3` — staging channel pre-releases of the same.

Key signal: **no functional `feat`/`fix` lines in the CHANGELOG** — wallet-cli releases are gated by upstream coin-* and live-common bumps (this is what changesets renders by default for an aggregate consumer). Mid-migration evidence comes from comments in source, not the CHANGELOG (see `live-common-setup.ts` TODO: "wallet-cli should own its Redux store setup (createRtkCryptoAssetsStore + RTK middleware) instead of relying on setupCalClientStore from @ledgerhq/cryptoassets/cal-client (test-helpers).").

What's recently shipped / mid-migration (inferred from source comments):
- **Alpaca routing for `operations`** is **temporarily disabled** ("currently bypassed due to known correctness issues (missing internal ops, unreliable pagination); the routing code is preserved as a comment for when it is re-enabled" — `wallet/README.md`).
- **DMK persistence**: persistent DMK kit per-process due to a bug — re-creating DMK stacks node-usb hotplug listeners and breaks the 3rd in-process connect. Comment: "One DMK per CLI process: each `createDeviceManagementKit()` adds node-usb hotplug listeners; closing + recreating stacks listeners and breaks the 3rd+ in-process connect (same pattern as the former node-hid kit)."
- **Solana LDMK enabled**: `setSolanaLdmkEnabled(true)` is called in `live-common-setup.ts`. CJS/ESM dual-instance issue is documented inline.
- **CJS/ESM duality** for LiveConfig: `live-common-setup.ts` calls `LiveConfig.setConfig(walletCliConfig)` **twice** (ESM and CJS) because Bun's bundler resolves ESM imports to lib-es/ and require() to lib/, creating separate singletons. Alpacaized families (EVM) read ESM; non-alpacaized (Solana, Bitcoin) read CJS.

---

## 6. Legacy `apps/cli` — short summary

`apps/cli` is the **published** `@ledgerhq/live-cli` npm package (versus wallet-cli which is `private: true`). The README is 38 KB; the documentation block lists **62 `Usage: ledger-live <command>` lines**. Why it's legacy:

- **Node 14 / `lts/fermium`** prerequisite (vs Bun ≥ 1.1.0 in wallet-cli).
- Built on `@ledgerhq/live-common` directly, **without DMK** — uses the older `@ledgerhq/hw-transport-node-hid`-style stack via `registerTransportModule`.
- Outputs free-form text (a few `--format json` flags); has no unified `--output json` envelope.
- Uses **V0** account ids only (`xpub`-based, `--id`, `--xpub`, `--file`, `--appjsonFile`).
- **Distribution model**: published to npm globally. wallet-cli is intentionally `private` to the monorepo.

What the legacy CLI has that **wallet-cli does not yet replicate**:
- `swap` — `Perform an arbitrary swap between two currencies on the same seed` (legacy CLI swap).
- `bot`, `botPortfolio`, `botTransfer` — Speculos bot harness.
- `app` / `appUninstallAll` / `listApps` / `managerListApps` — device app management.
- `firmwareUpdate` / `firmwareRepair` / `deviceSDKFirmwareUpdate` — firmware ops.
- `sync` — explicit account sync trigger (wallet-cli runs this implicitly via the bridge).
- `signMessage` — sign arbitrary messages.
- `estimateMaxSpendable` — spendable estimate without preparing a transaction.
- `getAddress` — low-level address derivation (wallet-cli's `receive` is higher-level).
- `broadcast` — broadcast pre-signed operations (wallet-cli does send-and-broadcast atomically).
- `genuineCheck`, `deviceInfo`, `deviceVersion`, `discoverDevices`, `getBatteryStatus`, `getDeviceRunningMode` — device introspection.
- `customLockScreenLoad`, `staxFetchAndRestoreDemo` — Stax-specific.
- `repl` — APDU REPL.
- `i18n`, `synchronousOnboarding`, `liveData` — utility commands.
- `celoValidatorGroups`, `cosmosValidators`, `tezosListBakers`, `tronSuperRepresentative`, `polkadotValidators` — staking validator listings for non-supported families.
- `confirmOp`, `getTransactionStatus`, `balanceHistory`, `countervalues`, `portfolio` — read-only queries (wallet-cli has `balances` + `operations` only).
- **`generateTestScanAccounts` / `generateTestTransaction`** — generate test fixtures for live-common datasets. This is QA-relevant.
- `testDetectOpCollision`, `testGetTrustedInputFromTxHash` — internal test helpers.
- `ledgerKeyRingProtocol`, `ledgerSync` — Ledger Sync / key-ring protocol commands.

What **wallet-cli has that legacy doesn't**:
- DMK-based USB transport (live-dmk-shared, ConnectAppDeviceAction).
- Bunli-based command framework (`defineCommand` + Zod-validated `option`).
- Single-binary build (`bunli build` → `darwin-arm64`, `linux-arm64`, `linux-x64`, `windows-x64`).
- Versioned account descriptor V1 (`account:1:…`) with shell-safe hardened segments (`h` instead of `'`).
- Unified `--output human|json` JSON envelope across all commands.
- Persistent session storage (`session view`, `session reset`) — labels and descriptors stored in YAML at `stateDir(APP_NAME)/session.yaml`.
- `--dry-run` send mode that skips device + broadcast.
- Mock infrastructure (`MockServer`, `MockDeviceManagementKit`) baked into the test layer for hermetic command tests.

---

## 7. Gaps — what the wiki/skills/READMEs do NOT cover

These are subjects the CLI-Automation Part will need to source from elsewhere (Confluence, Jira, source code):

1. **ERC-20 approve / allowance flow** — completely absent from wiki, skills, and READMEs. The closest source-code mention is `--data` on `send` (raw EVM calldata). There is **no documented `approve` subcommand**. Approving an ERC-20 spender today requires hand-crafting `0x095ea7b3…` calldata and using `wallet-cli send --data 0x...`. This is an undocumented capability; a chapter on it must be original work.
2. **Swap automation via wallet-cli** — also absent. The `e2e-swap` skill covers UI E2E only. The legacy `apps/cli` has a `swap` command; **wallet-cli has none**. Anyone wanting "headless swap" today must either use the legacy CLI (Node 14, V0 ids) or drive Live Desktop/Mobile through Playwright/Detox.
3. **Hooks** — not in any docs. The wallet-cli `wallet/` layer exposes `WalletAdapter` directly; no React hooks (`useWalletAdapter`, etc.) exist. The wiki's `LLC:apps.md` mentions live-common's React hooks but those are for Ledger Live UI, not CLI.
4. **`account fresh-address` and `session view`/`session reset`** subcommands — exist in source (`commands/account/fresh-address.ts`, `commands/session/{view,reset}.ts`) but **are not in the README's commands table** (only the SKILL hints at session via the `account discover` flow that auto-saves descriptors).
5. **Mock harness usage examples** — `WALLET_CLI_MOCK_PORT`, `WALLET_CLI_MOCK_DMK`, `WALLET_CLI_MOCK_APP_RESULTS` env vars are documented one-line in `test/commands/README.md` but no end-to-end "how to write a CLI mock test" walkthrough exists.
6. **CJS/ESM gotcha** — only commented inline in `live-common-setup.ts`. The chapter on "why does wallet-cli call `LiveConfig.setConfig` twice" needs to quote that comment because it's not in any markdown.
7. **DMK transport registration via `registerTransportModule`** — the wiki's `LLC:hw.md` documents the pattern in general but uses `TransportWebHID`/`TransportWebUSB` examples; the actual DMK adapter (`device/register-dmk-transport.ts`) is the modern equivalent and is not documented anywhere outside source.
8. **`USER_ID` salt** — README mentions it; no rationale beyond "stable salt". The `client-ids` skill mentions UserId privacy rules in general but doesn't connect to wallet-cli.
9. **Ticker resolution / token discovery** — SKILL says ticker is mandatory and drives asset resolution; no documented flow for "how does the CLI resolve `'100 USDT'` to a specific ERC-20 contract per network". `wallet/intents/parse-amount.ts` is the source of truth (`ethereum/erc20/usd~!underscore!~tether` asset id format).
10. **Multi-step approval flows** (e.g. ERC-20 approve before swap) — the `e2e-swap` skill mentions `tapExecuteSwapOnStepApproval()` for mobile UI but there is **no CLI documentation** of the underlying APDU/UI sequence.
11. **`--data` security warning** — `send.ts` accepts raw 0x-hex calldata via Zod regex, but no docs warn against constructing dangerous calldata. QA chapter should include a "be careful what you sign" sidebar.
12. **Base = Ethereum app** — only the SKILL says "On Ledger devices, **Base uses the Ethereum app**". README, source, and wiki are silent. Easy footgun.
13. **Device contention** — only the SKILL documents that two device commands cannot run in parallel (`[object Object]` symptom). README is silent.
14. **`pnpm wallet-cli start` shortcut** — defined in the root `package.json` (not read here, but referenced from README). The `docs/common-commands.md` does **not** list it.

---

## Appendix — architectural facts useful for the chapter

From `apps/wallet-cli/src/cli.ts`:

```ts
#!/usr/bin/env bun
import "./embed-usb-native";
import { createCLI } from "@bunli/core";
import "./live-common-setup";
// createCLI() normally tries to import .bunli/commands.gen.ts from process.cwd() via a file:// URL.
// Our @bunli/core patch removes that dynamic import entirely because it can hang in Bun standalone
// mode, this static import registers commands instead.
import "../.bunli/commands.gen";
import bunliConfig from "../bunli.config";
import { disposeWalletCliDmkTransportFully } from "./device/register-dmk-transport";

const cli = await createCLI(bunliConfig as unknown as Parameters<typeof createCLI>[0]);
await cli.run();

// Release the process-wide DMK + node-usb hotplug listeners
await disposeWalletCliDmkTransportFully();
process.exit(0);
```

Tells the chapter:
- Bun shebang.
- USB native binding embedded **before** anything else.
- live-common setup loaded before commands (registers coin modules, DMK transport, `setSolanaLdmkEnabled(true)`).
- Static import of generated `commands.gen.ts` is intentional (Bun standalone hangs on the dynamic version).
- Process exit explicitly disposes DMK to release node-usb hotplug listeners.

From `apps/wallet-cli/src/device/dmk.ts`:

```ts
return new DeviceManagementKitBuilder()
  .addTransport(nodeWebUsbTransportFactory)
  .addLogger(new LedgerLiveLogger(LogLevel.Warning))
  .addConfig({ firmwareDistributionSalt })
  .build();
```

The DMK is built with a single transport (`nodeWebUsbTransportFactory` in `device/node-webusb/`), warning-level logger, and a deterministic firmware-distribution salt derived from `USER_ID` via `UserHashService.compute`.

From `apps/wallet-cli/src/device/connect-ledger-app.ts`:

- Uses `ConnectAppDeviceAction` from `@ledgerhq/live-dmk-shared` (the same device action LLD uses).
- `unlockTimeout: 60_000` (vs Live's `0`) so users can unlock the device after starting the command.
- Stall detection: a wall-clock `setTimeout` cancels the device action if it makes no progress (RxJS `timeout()` alone is insufficient when DMK/USB blocks without scheduling timers).
- Retries up to **5 times** on `ReceiverApduError` / `UnknownDeviceExchangeError` because "the DMK's `waitForDeviceUnlock` polling may receive an unparseable APDU response (`ReceiverApduError`) instead of error code 5515. The polling misidentifies this as 'unlocked' and the action fails immediately."
- Stderr messaging: `[~] Waiting: Ledger detected but locked. Enter your PIN on the device.` and `[i] Opening <App> app on your Ledger... ACTION REQUIRED: Confirm on device screen.`

From `apps/wallet-cli/src/session/bridge-device-session.ts`:

```ts
export async function withCurrencyDeviceSession<T>(currencyId, fn) {
  const { managerAppName } = getCryptoCurrencyById(currencyId);
  const transport = await ensureWalletCliDmkTransport();
  await connectLedgerApp(transport.dmk, transport.sessionId, managerAppName);
  try { return await fn(); } finally { await resetWalletCliDmkSession(); }
}
```

Every device-using command wraps its work in `withCurrencyDeviceSession(currencyId, …)`. This is the canonical pattern QA should learn. The legacy `withBridgeDeviceSession(family, …)` exists for backward compat and maps `bitcoin → bitcoin`, `evm → ethereum`, `solana → solana`.

From `apps/wallet-cli/src/session/session-store.ts`:

- Persistent session lives in YAML at `stateDir(APP_NAME)/session.yaml` where `APP_NAME = "ledger-wallet-cli"` and `stateDir` comes from `@bunli/utils`.
- Schema (Zod): `{ accounts: [{ label, descriptor }] }`.
- Auto-generated labels: `bitcoin-native-1`, `bitcoin-segwit-1`, `ethereum-1`, `ethereum-sepolia-1`, `solana-devnet-1`. Derivation label decoded from BIP44 purpose: 44=`legacy`, 49=`segwit`, 84=`native`, 86=`taproot`.
- `account discover` automatically appends discovered descriptors to the session via `Session.read() → addDescriptors(...) → session.write()`. Failures are non-fatal (CLI output already flushed).
- `session reset` always rewrites the file even if it was corrupt (recovery path).

From `apps/wallet-cli/src/wallet/intents/families/evm.ts`:

```ts
export const EvmTransactionIntentSchema = z.object({
  family: z.literal("evm"),
  recipient: z.string(),
  amount: AmountWithTickerSchema,
  data: z.string().regex(/^0x([0-9a-fA-F]{2})*$/, "data must be 0x-prefixed hex with an even number of digits").optional(),
});
```

This is the **only EVM extensibility point**. ERC-20 approve, swap, allowance — all would today be done by hand-crafting `data` and broadcasting through `wallet-cli send --data 0x...`.

---

## Summary

- **Wiki**: zero CLI-Automation coverage. Only references the deprecated CLI. Useful for live-common conceptual background only.
- **Agent skills**: `ledger-wallet-cli/SKILL.md` is the most authoritative concise source. `e2e-swap` is for UI E2E, not CLI swap. No skill covers Bunli, DMK, V1 descriptors, ERC-20 approve, or swap automation directly.
- **In-tree READMEs**: `apps/wallet-cli/README.md`, `src/wallet/README.md`, `src/test/commands/README.md`, `src/shared/accountDescriptor/README.md` are the four most load-bearing internal docs and are well-written. They are the de-facto spec for the CLI-Automation Part.
- **Legacy CLI**: published, broad surface (62 commands), Node 14 stack, V0 ids, has swap and bot commands wallet-cli lacks.
- **Major gaps** that the new chapter must fill from scratch: ERC-20 approve flow, swap automation, multi-step approvals, `--data` security guidance, mock harness walkthrough, ticker resolution mechanics, hooks (none), and the Base-uses-Ethereum-app footgun.
