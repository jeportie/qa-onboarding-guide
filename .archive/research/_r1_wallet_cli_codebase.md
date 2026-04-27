# R1 — wallet-cli Codebase Map

Date: 2026-04-27
Source: `/Users/jerome.portier/src/tries/2026-04-08-LedgerHQ-ledger-live/apps/wallet-cli/`
Skill doc: `/Users/jerome.portier/src/tries/2026-04-08-LedgerHQ-ledger-live/.agents/skills/ledger-wallet-cli/SKILL.md`
Briefing: `/Users/jerome.portier/qa-onboarding-guide/.archive/research/_cli_part_briefing.md`

This document is the primary research input for the new "Part 5 — CLI Automation" of the QA onboarding guide. It maps every file under `apps/wallet-cli/`, documents the Bunli command framework, the Device Management Kit (DMK) integration, the wallet abstraction, and the test harness ("Speculos analog" for CLI). All paths are absolute under the monorepo `apps/wallet-cli/` root unless otherwise noted.

> **Headline finding:** The codebase is materially richer than the briefing implied. The `commands/` tree contains **3 groups** (`account`, `session`) and **10 leaf commands** — not 5. There is a real **`session` layer** (YAML-persisted account labels under `XDG_STATE_HOME/ledger-wallet-cli/session.yaml`) that wires `account discover` → `balances`/`operations`/`send`. There is a **`bunli generate` codegen step** (`.bunli/commands.gen.ts`) that statically wires every command into the Bun-compiled binary. `send` already accepts `--data` (EVM raw calldata), so the QAA-615 spike's "minimum viable revoke" is `send --data 0x095ea7b3…` — a new `revoke` subcommand would be a thin wrapper, not new infrastructure.

---

## Section 0 — ASCII tree of `src/`

```
apps/wallet-cli/
├── package.json                  72 lines
├── bunli.config.ts               20 lines
├── bunfig.toml                    6 lines
├── tsconfig.json                 12 lines
├── project.json                  19 lines  (Nx target)
├── README.md                     78 lines
├── CHANGELOG.md                  ~13k bytes
├── .oxlintrc.json   .oxfmtrc.json  .gitignore
├── .bunli/
│   └── commands.gen.ts          215 lines  (codegen — DO NOT EDIT)
└── src/
    ├── cli.ts                    20 lines  (entry — Bunli wiring)
    ├── config.ts                 24 lines  (LiveConfig schema for wallet-cli)
    ├── live-common-setup.ts      80 lines  (registers coin modules + DMK transport)
    ├── embed-usb-native.ts       33 lines  (Bun --compile native addon embed)
    ├── output.ts                338 lines  (CommandOutput abstraction — human/json)
    │
    ├── commands/
    │   ├── inputs.ts             53 lines   (shared option helpers — was "shared-options.ts")
    │   ├── inputs.test.ts        19 lines
    │   ├── balances.ts           34 lines
    │   ├── operations.ts         45 lines
    │   ├── receive.ts            48 lines
    │   ├── send.ts              158 lines
    │   ├── account/
    │   │   ├── index.ts           9 lines  (group definition)
    │   │   ├── discover.ts       89 lines
    │   │   └── fresh-address.ts  39 lines
    │   └── session/
    │       ├── index.ts           9 lines  (group definition)
    │       ├── view.ts           21 lines
    │       ├── view.test.ts
    │       ├── reset.ts          28 lines
    │       └── reset.test.ts
    │
    ├── device/
    │   ├── register-dmk-transport.ts    172 lines
    │   ├── wallet-cli-dmk-transport.ts   38 lines
    │   ├── dmk.ts                        20 lines  (DMK builder)
    │   ├── connect-ledger-app.ts        201 lines  (ConnectAppDeviceAction wrapper)
    │   ├── connect-ledger-app.test.ts
    │   ├── mock-dmk.ts                  191 lines  (test seam)
    │   ├── wallet-cli-device-error.ts    85 lines  (error mapping)
    │   ├── wallet-cli-device-error.test.ts
    │   └── node-webusb/
    │       ├── index.ts
    │       ├── node-webusb-constants.ts
    │       ├── NodeWebUsbApduSender.ts
    │       └── NodeWebUsbTransport.ts
    │
    ├── session/
    │   ├── session-store.ts            128 lines  (YAML persistence)
    │   ├── session-store.test.ts       169 lines
    │   └── bridge-device-session.ts     48 lines  (withCurrencyDeviceSession)
    │
    ├── shared/
    │   ├── log.ts                       11 lines  (debug() namespace "wallet-cli")
    │   ├── response.ts                  15 lines  (makeEnvelope — JSON wrapper)
    │   ├── response.test.ts             34 lines
    │   ├── ui.ts                        67 lines  (spinner, colors, isInteractive)
    │   ├── ui.test.ts                   66 lines
    │   └── accountDescriptor/
    │       ├── README.md
    │       ├── index.ts                 24 lines  (re-export barrel)
    │       ├── v0.ts                    10 lines  (legacy WalletSync schema)
    │       ├── v1.ts                   147 lines  (ADR descriptor)
    │       ├── v1.test.ts              106 lines
    │       ├── network.ts              131 lines
    │       ├── network.test.ts         130 lines
    │       ├── adapters.ts             212 lines  (toV0 / toV1)
    │       ├── adapters.test.ts        118 lines
    │       └── test-fixtures.ts          4 lines
    │
    ├── wallet/
    │   ├── README.md
    │   ├── index.ts                     96 lines  (WalletAdapter — routing)
    │   ├── models.ts                   107 lines  (Zod schemas, AccountDescriptor V0)
    │   ├── models.test.ts               56 lines
    │   ├── compatibility/
    │   │   ├── bridge.ts               295 lines  (BridgeAdapter — live-common bridge)
    │   │   └── alpaca.ts                90 lines  (AlpacaAdapter — direct API)
    │   ├── intents/
    │   │   ├── index.ts                 17 lines  (TransactionIntentSchema)
    │   │   ├── index.test.ts           189 lines
    │   │   ├── parse-amount.ts          57 lines  ("0.5 ETH" parser)
    │   │   ├── parse-amount.test.ts     68 lines
    │   │   └── families/
    │   │       ├── bitcoin.ts           21 lines
    │   │       ├── evm.ts               12 lines  ← `data` field already present
    │   │       └── solana.ts            14 lines
    │   └── formatter/
    │       ├── index.ts                  3 lines
    │       ├── human.ts                 96 lines
    │       ├── human.test.ts           268 lines
    │       └── json.ts                  58 lines
    │
    └── test/
        ├── commands/
        │   ├── README.md
        │   ├── balances.test.ts         88 lines
        │   ├── discover.test.ts         79 lines
        │   ├── operations.test.ts      125 lines
        │   ├── receive.test.ts          33 lines
        │   ├── send.test.ts             72 lines
        │   └── session.test.ts          77 lines
        └── helpers/
            ├── cli-runner.ts            52 lines  (Bun.spawn wrapper)
            ├── constants.ts              8 lines  (ETH descriptors / fixtures)
            ├── dmk-intercept.ts         29 lines  (mock DMK install)
            ├── http-intercept.ts       146 lines  (fetch + http patch)
            ├── eth-sync-routes.ts       37 lines  (default ETH HTTP fixtures)
            ├── mock-server.ts           58 lines  (Bun.serve route table)
            ├── session-fixture.ts       27 lines  (XDG_STATE_HOME tmpdir)
            ├── wrapper.ts                8 lines  (full intercept wrapper)
            └── wrapper-local.ts          3 lines  (no HTTP intercept)
```

Total source: **~5,860 lines** (excluding test fixtures, generated, and node-webusb internals).

---

## Section 1 — Top-level configuration

### `package.json` — 72 lines

- **name:** `@ledgerhq/wallet-cli` (private to monorepo)
- **bin:** `wallet-cli` → `./src/cli.ts` (executed under Bun shebang)
- **type:** `module` (ESM)
- **engines:** `bun: ">=1.1.0"` — Bun, not Node
- **scripts:**

| Script | Command | Purpose |
|---|---|---|
| `start` | `bun run ./src/cli.ts` | Run CLI from source (the one writers will use) |
| `dev` | `bunli dev` | Dev mode with hot reload |
| `generate` | `bunli generate` | Regenerate `.bunli/commands.gen.ts` from `src/commands/*` |
| `generate:check` | `bunli generate && git diff --exit-code -- .bunli/commands.gen.ts` | CI gate — fails PR if codegen drifted |
| `build` | `bunli build` | Standalone binary per target → `dist/` |
| `typecheck` | `tsc --noEmit -p tsconfig.json` | Type-only check |
| `test` | `bun test src/` | Run all `*.test.ts` under `src/` |
| `coverage` | `bun test src/ --coverage` | Coverage (lcov per `bunfig.toml`) |
| `lint` | `oxlint src bunli.config.ts` | Lint |
| `lint:fix` | `oxfmt -c ../../.oxfmtrc.json … && oxlint … --fix` | Fix lint + format |
| `format` / `format:check` | `oxfmt …` | Format check |

- **Notable runtime deps:** `@bunli/core@0.9.1`, `@bunli/utils@0.6.0`, `@ledgerhq/device-management-kit` (catalog), `@ledgerhq/live-dmk-shared` (workspace), `@ledgerhq/live-common`, `@ledgerhq/live-wallet`, `@ledgerhq/coin-bitcoin`, `@ledgerhq/coin-evm`, `@ledgerhq/coin-solana`, `usb@2.17.0`, `yocto-spinner`, `zod`, `purify-ts`, `rxjs`.
- **Dev:** `bunli@0.9.1`, `oxlint`, `oxfmt`, `bun-types`.

### `bunli.config.ts` — 20 lines

```ts
export default defineConfig({
  name: "wallet-cli",
  version: "0.1.0",
  description: "Ledger Wallet CLI",
  commands: { directory: "./src/commands" },
  build: {
    entry: "./src/cli.ts",
    outdir: "./dist",
    minify: true,
    targets: ["darwin-arm64", "linux-arm64", "linux-x64", "windows-x64"],
  },
});
```

Bunli scans `src/commands/` for `defineCommand` / `defineGroup` modules during `bunli generate`. The `usb` native addon is embedded via the explicit `require()` in `src/embed-usb-native.ts`.

### `bunfig.toml` — 6 lines

Bun runtime config — `[test]` section with `coverageReporter = ["lcov"]` and `coverageDir = "coverage"`.

### `tsconfig.json` — 12 lines

Extends root `tsconfig.base.json`. Notable: `moduleResolution: "Bundler"`, `module: "ESNext"`, `noEmit: true`, types `["node", "bun-types", "w3c-web-usb"]`. Includes `src/**/*.ts` and `bunli.config.ts`. Test files are typechecked but never bundled into the binary.

### `project.json` — 19 lines (Nx)

Declares Nx targets `coverage`, `generate`, `generate:check`, `build`. Marks `generate:check` and `build` as `cache: false`. Outputs: `{projectRoot}/coverage/**` for coverage, `{projectRoot}/.bunli/commands.gen.ts` for generate. Nx invocations from repo root use `pnpm --filter @ledgerhq/wallet-cli <target>` or `pnpm wallet-cli <script>`.

### `.bunli/commands.gen.ts` — 215 lines (CODEGEN)

This is the cornerstone of how Bunli works in compiled mode. It is **generated by `bunli generate`** from the `src/commands/*` tree and committed. Header line: "This file was automatically generated by Bunli. You should NOT make any changes in this file as it will be overwritten."

What it does:

1. Statically imports every command module:
   ```ts
   import Account from '../src/commands/account/index.js'
   import Balances from '../src/commands/balances.js'
   import Discover from '../src/commands/account/discover.js'
   import FreshAddress from '../src/commands/account/fresh-address.js'
   import Operations from '../src/commands/operations.js'
   import Receive from '../src/commands/receive.js'
   import Reset from '../src/commands/session/reset.js'
   import Send from '../src/commands/send.js'
   import Session from '../src/commands/session/index.js'
   import View from '../src/commands/session/view.js'
   ```
2. Builds a `modules` map and a `metadata` map (mirrored Zod schema shape per option).
3. Calls `registerGeneratedStore(createGeneratedHelpers(modules, metadata))`.
4. Auto-registers commands on import.

Why it exists: at the top of `cli.ts` there is a comment saying `createCLI()` "normally tries to import .bunli/commands.gen.ts from process.cwd() via a file:// URL. Our @bunli/core patch removes that dynamic import entirely because it can hang in Bun standalone mode; this static import registers commands instead." So the CLI uses a **patched** `@bunli/core` to avoid the dynamic import, and `commands.gen.ts` is statically `import`ed by `cli.ts`.

**Implication for new commands:** add a file under `src/commands/` (or a sub-group), then run `pnpm generate`. The PR will fail if you forget — `generate:check` runs in CI.

---

## Section 2 — Entry point

### `src/cli.ts` — 20 lines (full text)

```ts
#!/usr/bin/env bun
import "./embed-usb-native";
import { createCLI } from "@bunli/core";
import "./live-common-setup";
// createCLI() normally tries to import .bunli/commands.gen.ts from process.cwd() via a file:// URL.
// Our @bunli/core patch removes that dynamic import entirely because it can hang in Bun standalone
// mode, this static import registers commands instead.
// This side-effect import registers commands in the standalone binary.
import "../.bunli/commands.gen";
import bunliConfig from "../bunli.config";
import { disposeWalletCliDmkTransportFully } from "./device/register-dmk-transport";

// Pass config explicitly so the compiled binary does not depend on cwd for bunli.config.* discovery.
const cli = await createCLI(bunliConfig as unknown as Parameters<typeof createCLI>[0]);
await cli.run();

// Release the process-wide DMK + node-usb hotplug listeners (see persistentDmk in register-dmk-transport).
// Error paths already call process.exit(1) inside bunli, so this only runs on success.
await disposeWalletCliDmkTransportFully();
process.exit(0);
```

Order of side effects matters:

1. **Embed the USB native addon** — must run before any `usb` import.
2. **Live-common setup** — registers coin modules, supported currencies, the DMK transport module, USER_ID env, LiveConfig.
3. **commands.gen import** — registers every command in the Bunli store.
4. **createCLI(bunliConfig)** — explicit config, no cwd discovery.
5. **cli.run()** — Bunli parses argv, dispatches the right `defineCommand` handler.
6. **disposeWalletCliDmkTransportFully()** — graceful shutdown of DMK + node-usb hotplug listeners on success path. Error paths exit via Bunli's `process.exit(1)`.

---

## Section 3 — Bunli command framework

### Command anatomy

Every command file exports a default `defineCommand({...})`:

```ts
import { defineCommand, option } from "@bunli/core";
import { z } from "zod";

export default defineCommand({
  name: "<name>",
  description: "<description>",
  options: { /* keyed by flag name; option(zodSchema, meta) */ },
  handler: async ({ flags, positional }) => { /* … */ },
});
```

A group (subcommand parent) uses `defineGroup`:

```ts
import { defineGroup } from "@bunli/core";
import DiscoverCommand from "./discover";
import FreshAddressCommand from "./fresh-address";

export default defineGroup({
  name: "account",
  description: "Account management commands",
  commands: [DiscoverCommand, FreshAddressCommand],
});
```

Bunli auto-converts Zod schemas in `option(...)` into CLI flags + help text.

### Shared option helpers — `src/commands/inputs.ts` (53 lines)

The briefing called this `shared-options.ts`; the actual filename is `inputs.ts`. It exports the canonical option builders and account-resolution helpers used by every leaf command:

```ts
export const accountOption = option(z.string().min(1).optional(), {
  description: "Account descriptor or session label (e.g. ethereum-1). Can also be the first positional arg.",
  short: "a",
});

export const outputOption = option(OutputFormatSchema.default("human"), {
  description: "Output format: human (default) or json",
});

export function resolveAccountArg(account: string | undefined, positional: readonly string[]): string {
  const arg = account ?? positional[0];
  if (!arg) throw new Error("Missing account: use --account <descriptor-or-label> …");
  return arg;
}

// Contains ":" → descriptor passthrough; no ":" → session label lookup.
export async function resolveAccountInput(input: string): Promise<string> {
  if (input.includes(":")) return input;
  const session = await Session.read();
  const entry = session.accounts.find(e => e.label === input);
  if (!entry) throw new Error(`No account labeled "${input}" in session. …`);
  return entry.descriptor;
}

export async function resolveAccountDescriptor(input: string): Promise<AccountDescriptor> {
  return parseAccountDescriptor(await resolveAccountInput(input));
}
export async function resolveAccountDescriptorV1(input: string): Promise<AccountDescriptorV1> {
  return parseV1(await resolveAccountInput(input));
}
```

This is the **session label resolver**: any flag value that does not contain `":"` is looked up by label against `session.yaml`. So `wallet-cli balances ethereum-1` works after `account discover ethereum`.

### Command-by-command reference

#### `account discover` — `commands/account/discover.ts` (89 lines)

| Field | Value |
|---|---|
| Group | `account` |
| Flags | `--network/-n <string>`, `--output <human\|json>` |
| Positional | First arg used as network if `--network` absent |
| Device | **Required** (USB) |
| Network | Required for sync (live-common bridge over `withCurrencyDeviceSession`) |

Handler flow:
1. Parse network arg (default env = `main`).
2. Open DMK USB session via `withCurrencyDeviceSession(currencyId, …)` → opens currency-specific Manager app.
3. Call `WalletAdapter.discoverAccounts(network, WALLET_CLI_DMK_DEVICE_ID)` (returns Observable<DiscoveredAccount>).
4. Stream each `discoveredAccount` to `out.discoveredAccount(d)` (human: print line; json: buffer).
5. On stream complete: call `Session.read() → session.addDescriptors(...) → session.write()` for **session persistence** (deduped by descriptor string, generated labels like `ethereum-1`, `bitcoin-native-1`).
6. `out.flushDiscovery()` and (in human mode) `out.sessionSaved(added)`.

Output JSON envelope: `{"command":"account discover","network":"ethereum:main","accounts":["account:1:address:ethereum:main:0x…:m/44h/60h/0h/0/0", …]}`.

#### `account fresh-address` — `commands/account/fresh-address.ts` (39 lines)

| Field | Value |
|---|---|
| Group | `account` |
| Flags | `--account/-a <descriptor-or-label>`, `--output` |
| Device | **No** |

Returns the next unused receive address. For `address` type (EVM/Solana) the address is in the descriptor — no I/O. For `utxo` (Bitcoin) it triggers a bridge sync to find the next unused address.

#### `balances` — `commands/balances.ts` (34 lines)

| Field | Value |
|---|---|
| Group | (top-level) |
| Flags | `--account/-a <descriptor-or-label>`, `--output` |
| Device | **No** |
| Output | `[{ asset: "ethereum", amount: "1.5 ETH" }, …]` envelope |

Routes to `WalletAdapter.getAccountBalances(descriptor)` which uses Alpaca for EVM family, full bridge sync otherwise.

#### `operations` — `commands/operations.ts` (45 lines)

| Field | Value |
|---|---|
| Group | (top-level) |
| Flags | `--account/-a`, `--limit/-l <int≥1>`, `--cursor <str>`, `--output` |
| Device | **No** |
| Output | `{ operations: [...], nextCursor: "…" }` |

NB: `wallet/index.ts` currently bypasses Alpaca for ops (correctness issues — flagged in the comment) and always uses bridge sync. `--cursor` is therefore unused today. `--limit` slices the bridge result.

Includes internal operations (ETH contract-call traces — flagged with `parentId`) and ERC-20 token operations from sub-accounts.

#### `receive` — `commands/receive.ts` (48 lines)

| Field | Value |
|---|---|
| Group | (top-level) |
| Flags | `--account/-a`, `--verify/-v <bool, default true>`, `--output` |
| Device | **Required when `--verify=true` (default)** |
| Output | `{ address: "0x…" }` envelope |

If `--verify`, opens DMK session via `withCurrencyDeviceSession(descriptor.currencyId, …)`, calls `wallet.verifyAddress(descriptor, WALLET_CLI_DMK_DEVICE_ID)` which routes to `BridgeAdapter.verifyAddress` → `bridge.receive(account, { deviceId, verify: true })`. The user must confirm on device. Use `--verify=false` to skip device.

#### `send` — `commands/send.ts` (158 lines) — THE CANONICAL "command shape"

| Field | Value |
|---|---|
| Group | (top-level) |
| Flags | see table below |
| Device | **Required unless `--dry-run`** |

Flags (verbatim from source):

| Flag | Schema | Required | Description |
|---|---|---|---|
| `--account/-a <str>` | `z.string().min(1).optional()` | optional | Account descriptor or session label |
| `--to/-t <addr>` | `z.string().min(1, …)` | **yes** | Recipient address |
| `--amount <"V TICKER">` | `z.string().min(1, …)` | **yes** | e.g. `0.001 BTC`, `0.01 ETH`, `0.4 USDT` — ticker drives asset resolution |
| `--fee-per-byte <sat>` | `z.string().min(1).optional()` | bitcoin | Custom fee per byte |
| `--rbf` | `z.boolean().optional()` | bitcoin | Replace-By-Fee |
| `--mode <m>` | `z.string().min(1).optional()` | solana | `send`, `stake.createAccount`, `stake.delegate`, `stake.undelegate`, `stake.withdraw` |
| `--validator <addr>` | `z.string().min(1).optional()` | solana | Validator address |
| `--stake-account <addr>` | `z.string().min(1).optional()` | solana | Stake account address |
| `--memo <text>` | `z.string().min(1).optional()` | solana | Memo/tag |
| `--data <0x…>` | `z.string().regex(/^0x([0-9a-fA-F]{2})*$/).optional()` | **evm** | **EVM raw calldata as 0x-hex** |
| `--dry-run` | `z.boolean().default(false)` | optional | Prepare/validate without signing |
| `--output` | `OutputFormatSchema.default("human")` | optional | human/json |

**Critical for QAA-615:** `--data` already exists. The path forward is a thin `revoke` wrapper that builds `--data 0x095ea7b3${spender_padded_32}${zeros_32}` and calls `send` against the token-contract `--to` and `--amount '0 <TICKER>'`. No new device or APDU code needed.

Handler flow:
1. Resolve account descriptor.
2. Look up `family` from `getCryptoCurrencyById(descriptor.currencyId)`.
3. Pick an `INTENT_BUILDERS[family]` (bitcoin / evm / solana). EVM builder maps `--data` → `intent.data`.
4. Validate via `TransactionIntentSchema.parse(intentData)` (Zod discriminated union).
5. **If `--dry-run`:** call `wallet.prepareSend(descriptor, intent)` (no device), output `sendDryRun`.
6. **Else:** open DMK session via `withCurrencyDeviceSession(descriptor.currencyId, …)`. Inside, subscribe to `wallet.send(...)` Observable. Each event hits `out.sendEvent(event)` (spinner updates / JSON accumulator). When the stream completes, call `out.sendComplete()`.

`SendEvent` discriminated union (`wallet/models.ts`):

```ts
type SendEvent =
  | { type: "prepared"; recipient: string; amount: string; fees: string }
  | { type: "device-streaming"; progress: number; index: number; total: number }
  | { type: "device-signature-requested" }
  | { type: "device-signature-granted" }
  | { type: "dry-run" }
  | { type: "broadcasted"; txHash: string };
```

#### `session view` — `commands/session/view.ts` (21 lines)

Reads `session.yaml`, prints all `(label, descriptor)` pairs (human: aligned table; json: array).

#### `session reset` — `commands/session/reset.ts` (28 lines)

Wipes `session.yaml`. If file is corrupt, treats it as empty and overwrites.

### "Canonical command shape" — what `revoke` should copy

`balances.ts` is the cleanest model for a **device-free** command. `send.ts` is the model for a **device-required** command. Pattern (~25 lines):

```ts
// 1. import { defineCommand, option } from "@bunli/core";
//    import shared option helpers from "./inputs";
//    import { createCommandOutput } from "../output";
// 2. defineCommand({ name, description, options: { account: accountOption, …, output: outputOption }, handler })
// 3. handler:
//    const ctx = { command: "<name>", network: "", account: "" };
//    const out = createCommandOutput(flags.output, ctx);
//    await out.run(async () => {
//      const descriptor = await resolveAccountDescriptor(resolveAccountArg(flags.account, positional));
//      ctx.network = networkStringFromCurrencyId(descriptor.currencyId);
//      ctx.account = descriptor.id;
//      // device-required path:
//      await withCurrencyDeviceSession(descriptor.currencyId, async () => {
//        // call WalletAdapter / WalletAdapter.send / etc.
//      });
//    });
```

Add the file under `src/commands/`, run `pnpm generate`, and Bunli wires it.

---

## Section 4 — Device layer (`src/device/`)

### `dmk.ts` — 20 lines

```ts
export function createDeviceManagementKit(): DeviceManagementKit {
  const userId = getEnv("USER_ID") || "wallet-cli";
  const firmwareDistributionSalt = UserHashService.compute(userId).firmwareSalt;
  return new DeviceManagementKitBuilder()
    .addTransport(nodeWebUsbTransportFactory)
    .addLogger(new LedgerLiveLogger(LogLevel.Warning))
    .addConfig({ firmwareDistributionSalt })
    .build();
}
```

Single factory. Picks the **node-webusb transport** (vendored under `src/device/node-webusb/`). USER_ID defaults to `wallet-cli` so the firmware distribution salt is stable for this CLI's identity.

### `register-dmk-transport.ts` — 172 lines

The process-wide DMK lifecycle manager. Key concepts:

- **Singleton DMK** (`persistentDmk`): one DMK per process. Each `createDeviceManagementKit()` call adds node-usb hotplug listeners; closing+recreating stacks listeners and breaks the 3rd+ in-process connect.
- **Test seam:** `_setTestDmkTransport(t)` swaps in a `MockDeviceManagementKit` for tests. Used by `test/helpers/dmk-intercept.ts`.
- **`WALLET_CLI_DMK_DEVICE_ID = "wallet-cli-dmk"`** — the device id passed to live-common `withDevice` / bridge methods.
- **`registerWalletCliDmkTransport()`** registers a live-common transport module (via `registerTransportModule({ id, open, disconnect })`) so anywhere in live-common code that does `withDevice("wallet-cli-dmk")(…)` ends up using DMK.
- **Lifecycle helpers:**
  - `ensureWalletCliDmkTransport()` — creates the transport on demand, with a 60s connect timeout. Detects `BUSY` initial state and asks user to retry.
  - `resetWalletCliDmkSession()` — drops the active USB session (called between commands; see `withCurrencyDeviceSession`).
  - `disposeWalletCliDmkTransportFully()` — disconnect + close persistent DMK on process exit.
- **Process exit hooks:** SIGINT (130) and SIGTERM (143) trigger a graceful teardown.

### `wallet-cli-dmk-transport.ts` — 38 lines

`hw-transport`-compatible wrapper that exposes `dmk` and `sessionId` so live-common's `isDmkTransport(transport)` returns true. Implements `exchange(apdu)` by calling `dmk.sendApdu(...)` and concatenating `data + statusCode`. Default APDU abort = 120s.

```ts
async exchange(apdu: Buffer, { abortTimeoutMs }: { abortTimeoutMs?: number } = {}): Promise<Buffer> {
  const { data, statusCode } = await this.dmk.sendApdu({
    sessionId: this.sessionId,
    apdu: new Uint8Array(apdu.buffer, apdu.byteOffset, apdu.byteLength),
    abortTimeout: abortTimeoutMs ?? WALLET_CLI_DEFAULT_APDU_ABORT_MS,
  });
  return Buffer.from([...data, ...statusCode]);
}
```

### `connect-ledger-app.ts` — 201 lines

Wraps DMK's `ConnectAppDeviceAction` (the same one used by Live LDK) to open a Manager app by name on an existing DMK session. Differences vs Live:

- **Positive `unlockTimeout`** (60s) — gives the user time to enter PIN after starting a command.
- **Stall timer** = `unlockTimeout + 60s` — wall-clock cancel via `executeDeviceAction().cancel()` when DMK/USB blocks without scheduling timers.
- **Retry on transport framing errors** — `ReceiverApduError` / `UnknownDeviceExchangeError` from polling are retried up to 5 times with 3s delay (a known DMK polling bug).

It also writes user-visible stderr messages on `UnlockDevice` and `ConfirmOpenApp` interactions:

```
[~] Waiting: Ledger detected but locked. Enter your PIN on the device.
[i] Opening Ethereum app on your Ledger... ACTION REQUIRED: Confirm on device screen.
```

### `mock-dmk.ts` — 191 lines (THE TEST SEAM)

`MockDeviceManagementKit` — a minimal stand-in for the real DMK used by integration tests.

What it mocks (and what it does NOT):
- **Mocks at `executeDeviceAction` level** — the XState machine and InternalApi are bypassed entirely. E2E tests with real hardware cover the layers omitted here.
- `listenToAvailableDevices` returns one `FAKE_DEVICE` (Nano S Plus).
- `connect` returns `"mock-session-id"`.
- `getDeviceSessionState` returns CONNECTED or LOCKED based on `initialState`.
- `executeDeviceAction`:
  - **`ConnectAppDeviceAction`** (input has `application`) → completes immediately.
  - **`CallTaskInAppDeviceAction`** (input has `task` function and `appName`) → returns the configured result from `appResults[appName]`.
- `sendApdu` fallback: returns `0x6D00` (INS not supported) when no app result matches; returns `0x5515` (locked) when state is locked.

Test inputs are passed via env:
- `WALLET_CLI_MOCK_DMK=1` — install the mock transport
- `WALLET_CLI_MOCK_DMK_STATE=connected|locked`
- `WALLET_CLI_MOCK_APP_RESULTS=<JSON>` — per-app results, e.g. `{"Ethereum":{"publicKey":"0x…","address":"0x…"}}`

**Implication for revoke testing:** the EVM signer's `signTransaction` task in CallTaskInAppDeviceAction will receive the configured Ethereum app result. To test a revoke flow end-to-end the mock needs to return a signed tx blob. Look at `test/commands/send.test.ts` for the pattern — although it's currently a `--dry-run` test, so the signed-tx path is not yet covered by an integration test.

### `wallet-cli-device-error.ts` — 85 lines

`toWalletCliDeviceError(error)` maps DMK / hw-transport errors to user-friendly messages:

| Input error | Mapped message |
|---|---|
| `ManagerDeviceLockedError` / `LockedDeviceError` / `LOCKED_DEVICE` | "Device is locked. Unlock your Ledger with your PIN and try again." |
| `SECURITY_STATUS_NOT_SATISFIED` | "Security error (often locked or operation not allowed)…" |
| `CONDITIONS_OF_USE_NOT_SATISFIED` | `[x] Transaction Cancelled: Rejected on device. No funds moved.` |
| `CLA_NOT_SUPPORTED` / `INS_NOT_SUPPORTED` | "Could not execute on the app. Open the correct app for this currency…" |
| `SendApduTimeoutError` | "Timed out talking to the Ledger over USB. Retry; check cable." |
| `ReceiverApduError` / `UnknownDeviceExchangeError` | "Could not communicate (garbled APDU). Unlock and retry." |
| anything else | passthrough |

This is what produces the strings the SKILL.md "Common errors" table documents. New commands should call this from `bridge-device-session.ts` (already wired) or directly when raising tx-specific errors.

### `embed-usb-native.ts` — 33 lines

Force Bun's `--compile` bundler to embed the `usb` N-API native addon. node-gyp-build uses dynamic `fs.readdirSync` resolution that Bun can't statically detect. A direct `require()` per platform branch + storing on `globalThis.__usbNativeAddon` lets the patched `usb/dist/usb/bindings.js` skip node-gyp-build at runtime. With `minify: true`, Bun evaluates `process.platform`/`process.arch` as compile-time constants, so each `--target` binary contains only its prebuilt.

Branches: darwin (universal), linux x64, linux arm64, win32 x64.

### `node-webusb/` — DMK transport implementation

Vendored node-WebUSB transport for DMK. Not fully read; consists of `index.ts` (factory), `node-webusb-constants.ts`, `NodeWebUsbApduSender.ts`, `NodeWebUsbTransport.ts`. Used as `addTransport(nodeWebUsbTransportFactory)` in `dmk.ts`. **Honest gap:** the test harness never enters this code (mock-dmk replaces it wholesale), so writers should mention it as "the production transport, replaced by mock-dmk under test."

---

## Section 5 — Wallet layer

### `wallet/index.ts` — `WalletAdapter` (96 lines)

The clean facade. Routes to `BridgeAdapter` (legacy live-common bridge) or `AlpacaAdapter` (direct API) per family:

```ts
private static readonly alpacaFamilies = new Set(["evm"]);

discoverAccounts(network, deviceId): Observable<DiscoveredAccount> { /* always bridge */ }
async getAccountBalances(descriptor): Promise<Balance[]> { /* alpaca for evm, else bridge */ }
async getAccountOperations(descriptor, options) { /* always bridge today (alpaca bypassed) */ }
async getFreshAddress(descriptor): Promise<string> { /* always bridge */ }
async verifyAddress(descriptor, deviceId): Promise<string> { /* always bridge */ }
async prepareSend(descriptor, intent) { /* always bridge */ }
send(descriptor, intent, deviceId, dryRun): Observable<SendEvent> { /* always bridge */ }
```

### `wallet/models.ts` — Zod schemas + V0 descriptor (107 lines)

Defines the V0 `AccountDescriptor` (the legacy WalletSync shape used by live-common bridge), `Balance`, `Operation`, `SendEvent`, `OutputFormatSchema`. Provides:

- `parseShortAccountDescriptor("js:2:bitcoin:xpub…:native_segwit:0")` — V0 short form.
- `parseAccountDescriptor(input)` — accepts both V0 (`js:2:…`) and V1 (`account:1:…`); V1 routes through `toV0(parseV1(input))`.

### `wallet/models.test.ts` — 56 lines

Covers `parseShortAccountDescriptor` (valid, non-zero index, missing colon, NaN index) and `parseAccountDescriptor` delegation paths.

### `wallet/README.md`

Documents the routing rules above and notes that `compatibility/` is internal — **do not import `bridge.ts` or `alpaca.ts` from outside `wallet/`**.

### `shared/accountDescriptor/` — Account descriptor V1

This is the **ADR descriptor module**. Briefing said it was implemented in `wallet/index.ts` and `wallet/models.ts`; in fact it lives under `src/shared/accountDescriptor/` as a self-contained module.

- `v0.ts` (10 lines) — V0 schema (legacy WalletSync).
- `v1.ts` (147 lines) — V1 schema, types, `serializeV1`, `parseV1`. Two variants: `utxo` (xpub + hardened-only path) and `address` (address + full BIP44 path). Both `'` and `h` accepted (h preferred — shell-safe).
- `network.ts` (131 lines) — `Network = { name, env }`, `parseNetworkArg`, `serializeNetwork`, `networkFromCurrencyId`, `currencyIdFromNetwork`, alias `mainnet → main`.
- `adapters.ts` (212 lines) — `toV0` / `toV1`, scheme-driven (no hardcoded coin tables) using `@ledgerhq/ledger-wallet-framework/derivation`.
- `README.md` — full spec (used as primary reference for the guide chapter).

V1 string format:
```
account:1:<utxo|address>:<network_name>:<network_env>:<xpub_or_address>:<path>
```

Examples:
```
account:1:utxo:bitcoin:main:xpub6BosfCn…:m/84h/0h/0h
account:1:address:ethereum:main:0x71C7656EC7ab88b098defB751B7401B5f6d8976F:m/44h/60h/0h/0/0
account:1:address:solana:main:7xCU4XQfL…:m/44h/501h/0h/0h
account:1:utxo:bitcoin:test:tpubD8L6U…:m/84h/1h/0h
```

`toV1` is lossless; `toV0` recovers `derivationMode` and `index` from the path by matching against all derivation modes for that currency, but always returns `freshAddress: ""` (V1 doesn't store it; sync via bridge if needed).

### `wallet/intents/` — Transaction intent schemas

A discriminated union (`family`) of family-specific intents. Per file:

- `families/bitcoin.ts` — `{ family: "bitcoin", recipient, amount: AmountWithTickerSchema, feePerByte?, rbf? }`
- `families/evm.ts` — `{ family: "evm", recipient, amount, data?: 0x-hex }` ← **revoke target**
- `families/solana.ts` — `{ family: "solana", recipient, amount, mode (default "send"), validator?, stakeAccount?, memo? }`
- `index.ts` — `TransactionIntentSchema = z.discriminatedUnion("family", […])`
- `parse-amount.ts` — `parseAmountWithTicker("0.5 ETH", account)` returns `{ assetId, amount: BigNumber }`. Resolves ticker against the native currency or token sub-accounts; throws on missing/unknown ticker with available list.

### `wallet/compatibility/bridge.ts` — `BridgeAdapter` (295 lines)

The bridge to live-common. Key methods:

- `discoverAccounts(currencyId, deviceId): Observable<AccountDescriptor>` — uses `getCurrencyBridge(currency).scanAccounts(...)`, prepares the bridge cache, filters `discovered` events.
- `getBalances(descriptor)` — full sync, returns native balance + token sub-account balances.
- `getOperations(descriptor)` — full sync, flattens internal ops (`parentId` set), token ops from sub-accounts, sorts by ISO date desc.
- `getFreshAddress(descriptor)` / `verifyAddress(descriptor, deviceId)` — sync + bridge.receive for verify.
- `prepareSend / send` — call `buildValidatedTx` (createTransaction → updateTransaction → prepareTransaction → getTransactionStatus, throws on errors), then `signOperation` + `broadcast`. Forwards device events as `SendEvent`s.

`buildTxExtras` per family:
- bitcoin: `feePerByte`, `rbf`
- **evm: `data` is converted from `0x…` hex string to Buffer via `Buffer.from(hex, "hex")`** ← already plumbed through to the EVM bridge transaction
- solana: `mode`, `validator`, `stakeAccountId`, `memo`

### `wallet/compatibility/alpaca.ts` — `AlpacaAdapter` (90 lines)

Direct API client (no bridge sync). Used today only for EVM `getBalances`. `getOperations` is documented but bypassed in `WalletAdapter` due to known correctness issues (missing internal ops, unreliable pagination).

### `wallet/formatter/{human,json}.ts`

- `HumanFormatter` (96 lines) — `formatBalance`, `formatOperation`, `formatDiscoveredAccount`, `formatAmount` (uses live-common `formatCurrencyUnit` for ticker-aware human strings), `formatError`. Color-codes operation types (IN green, OUT red, FEES yellow, REWARD cyan).
- `JsonFormatter` (58 lines) — converts `Balance[]` / `Operation[]` to the JSON envelope shape (`asset` + human-readable `amount` ticker string + `accountId` as V1 descriptor).
- `index.ts` — barrel.

### `output.ts` — `CommandOutput` abstraction (338 lines)

The unified output layer that lets every command call semantic methods (`out.balances(items)`, `out.address(addr)`, `out.sendEvent(event)`, `out.run(fn)`) without sprinkling `if (isHuman)` everywhere. Two implementations:

- **`HumanCommandOutput`** — uses `yocto-spinner` (stderr), colors, formatted lines on stdout. Re-throws errors so Bunli's default error display kicks in.
- **`JsonCommandOutput`** — buffers data, emits a single envelope (`makeEnvelope(command, network, data, account)`). On error: writes `{ ok: false, error: { command, message } }` envelope and `process.exit(1)`. Uses `writeSync(1, …)` because Bun.spawn on Linux does not reliably capture fd 2 (stderr) — observable in CI.

JSON success envelope:
```json
{
  "status": "success",
  "command": "balances",
  "network": "ethereum:main",
  "account": "account:1:address:ethereum:main:0x…",
  "balances": [...],
  "timestamp": "2026-04-27T…"
}
```

JSON error envelope (different shape — note `ok: false`, no `status` key):
```json
{ "ok": false, "error": { "command": "balances", "message": "No account labeled \"foo\" in session…" } }
```

This inconsistency (`status: "success"` vs `ok: false`) is a real wart. Tests check `data.ok === false` for errors and `data.command` for success.

---

## Section 6 — Shared utilities

### `shared/log.ts` — 11 lines

Wraps the `debug` package with namespace `"wallet-cli"`. Activate via `DEBUG=wallet-cli` or `DEBUG=wallet-cli:*`.

### `shared/response.ts` — 15 lines

Single function `makeEnvelope(command, network, data, account?)` — produces the JSON success envelope.

```ts
return { status: "success", command, network, ...(account == null ? {} : { account }), ...data, timestamp: new Date().toISOString() };
```

### `shared/ui.ts` — 67 lines

- `colors` and `writeStdout` re-exported from `@bunli/utils`.
- `isInteractive()` — returns `false` in AI agent contexts (`CLAUDECODE`, `CLAUDE_CODE`, `CURSOR_AGENT`, `CODEX_ENABLED`, `GEMINI_CLI`, `OPENCODE`, `AMP_CURRENT_THREAD_ID`, `AGENT=amp`) **OR** when stderr is not a TTY. Otherwise true.
- `spinner(text)` — returns a `yoctoSpinner` started on stderr if interactive, else a no-op Proxy.
- `withSpinner(text, success, fn, humanMode)` — convenience wrapper.

This is what makes the test harness deterministic: tests set `CLAUDECODE=1` (see `cli-runner.ts`) so spinners no-op, output is clean.

### `shared/ui.test.ts` — 66 lines (not read in detail)

Covers the `isInteractive` matrix and the no-op spinner.

### `shared/response.test.ts` — 34 lines (not read in detail)

Covers envelope shape with/without account.

---

## Section 7 — Session layer

### `session/session-store.ts` — 128 lines

YAML-persisted account-label store at `${stateDir(APP_NAME)}/session.yaml` where `APP_NAME = "ledger-wallet-cli"` and `stateDir` resolves to `XDG_STATE_HOME/ledger-wallet-cli/` on Linux/macOS.

Schema (Zod):
```ts
{ accounts: [ { label: string, descriptor: string }, … ] }
```

Class API:
- `Session.read()` — async; treats ENOENT as empty; throws on parse error pointing user to `session reset`.
- `Session.from(entries)` — synchronous from a list (used in `reset` to recover from corrupt file).
- `accounts: ReadonlyArray<SessionEntry>` — readonly view.
- `clear()` — wipe in memory; returns count.
- `addDescriptors(descriptors: AccountDescriptorV1[])` — dedupe by serialized V1, generate labels via `generateLabel`.
- `write()` — async, mkdir -p the state dir, write YAML.

`generateLabel(descriptor, existing)` template:
- Bitcoin: `bitcoin-{native|legacy|segwit|taproot}-{n}` (parses the BIP44 purpose from path).
- Other: `{name}[-{env-if-not-main}]-{n}`.

Examples: `ethereum-1`, `bitcoin-native-1`, `bitcoin-testnet-1` (env appended only when not `main`).

### `session/bridge-device-session.ts` — 48 lines

This is what the briefing was asking about. **"bridge device session"** here means a **session that combines the live-common bridge stack with a DMK device session** — not DMK bridging.

```ts
export async function withCurrencyDeviceSession<T>(
  currencyId: string,
  fn: () => Promise<T>,
): Promise<T> {
  const { managerAppName } = getCryptoCurrencyById(currencyId);
  try {
    const transport = await ensureWalletCliDmkTransport();
    await connectLedgerApp(transport.dmk, transport.sessionId, managerAppName);
    try {
      return await fn();
    } finally {
      await resetWalletCliDmkSession();
    }
  } catch (e) {
    throw toWalletCliDeviceError(e);
  }
}
```

Lifecycle:
1. `ensureWalletCliDmkTransport()` — DMK USB session.
2. `connectLedgerApp(...)` — open the right Manager app on device.
3. Run the user callback.
4. Always `resetWalletCliDmkSession()` (drop the USB session, keep the persistent DMK).
5. Map any error through `toWalletCliDeviceError`.

There's also a `withBridgeDeviceSession(family, fn)` legacy helper that maps `family → currencyId` (`bitcoin → bitcoin`, `evm → ethereum`, `solana → solana`) and delegates. **Note for revoke:** EVM family always maps to "ethereum" managerAppName, so a revoke against a Base or Polygon contract would still open the Ethereum app on device — same as the production Ledger Live behavior.

---

## Section 8 — Test harness ("Speculos analog" for CLI)

The test harness replaces the two external dependencies of the CLI:

| Layer | Real dependency | Test replacement | Activation |
|---|---|---|---|
| HTTP | Ledger Explorer / Alpaca / CAL | `MockServer` (Bun.serve) + `http-intercept.ts` (patched fetch + node:http) | `WALLET_CLI_MOCK_PORT=<n>` |
| USB / DMK | Real Ledger over USB | `MockDeviceManagementKit` via `dmk-intercept.ts` | `WALLET_CLI_MOCK_DMK=1` + `WALLET_CLI_MOCK_APP_RESULTS=<JSON>` |

Both are **subprocess-based** — tests spawn the actual `cli.ts` via Bun.spawn and assert on stdout/stderr/exit code. This is the closest CLI analog to a desktop Speculos test (which spawns the actual Live binary and asserts on its UI).

### `test/helpers/cli-runner.ts` — 52 lines

```ts
async function spawnCli(wrapper, args, env) {
  const proc = Bun.spawn(["bun", "--cwd", ROOT, wrapper, ...args], {
    env: {
      ...process.env,
      NO_COLOR: "1",
      CLAUDECODE: "1", // disables spinner via isInteractive()
      ...env,
    },
    stdin: "ignore", stdout: "pipe", stderr: "pipe",
  });
  // … return { stdout, stderr, exitCode }
}
export async function runCli(args, env={}) { return spawnCli(WRAPPER, args, env); }
export async function runLocalCli(args, env={}) { return spawnCli(WRAPPER_LOCAL, args, env); }
```

`runCli` uses `wrapper.ts` (full intercept). `runLocalCli` uses `wrapper-local.ts` (no HTTP intercept) for commands that make no network calls (`session view`, `session reset`).

Note `--cwd ROOT` (the **CLI argument** to `bun`, NOT the subprocess option) — this makes Bun resolve tsconfig and workspace packages from the wallet-cli root regardless of the test's cwd.

### `test/helpers/wrapper.ts` — 8 lines

```ts
import "./http-intercept";
if (process.env.WALLET_CLI_MOCK_DMK) {
  await import("./dmk-intercept");
}
await import("../../cli");
```

### `test/helpers/wrapper-local.ts` — 3 lines

```ts
export {};
await import("../../cli");
```

### `test/helpers/dmk-intercept.ts` — 29 lines

Reads `WALLET_CLI_MOCK_DMK_STATE` (default `connected`) and `WALLET_CLI_MOCK_APP_RESULTS` (JSON). Constructs a `MockDeviceManagementKit`, wraps it in `WalletCliDmkTransport`, calls `_setTestDmkTransport(transport)` from `register-dmk-transport.ts` to install it before the CLI starts.

### `test/helpers/http-intercept.ts` — 146 lines

The most sophisticated piece. Two interception layers:

1. **`globalThis.fetch` patch** — redirects any non-localhost URL to `http://localhost:${WALLET_CLI_MOCK_PORT}${pathname}${search}`. Preserves Request method/headers/body when called with a `Request` instance.
2. **`node:http.request` and `node:https.request` patches** — same redirection at the Node http layer. HTTPS requests are routed to plain HTTP on the mock port.

Crucial step: **forces axios to use the fetch adapter**. axios is resolved via `@ledgerhq/live-common`'s node_modules; both CJS (`dist/node/axios.cjs`) and ESM (`index.js`) instances are patched (`defaults.adapter = "fetch"`) so axios calls go through the `globalThis.fetch` patch. Without this, axios would use node:http directly and (per the comments) Bun's `process.defined` check makes axios pick the http adapter by default.

Comment from source:
> `node:http.request` arg overloads are normalized: `(options[, callback])` and `(url[, options][, callback])` shapes both supported.

### `test/helpers/mock-server.ts` — 58 lines

A minimal `Bun.serve` route table. Routes are `{ method?, match: string|RegExp, response, status?, headers? }`. String match = `pathname.includes(...)`; RegExp = `test(pathname + search)`. On no match, prints `[mock-server] UNMATCHED: …` to stderr and returns 404.

```ts
const server = new MockServer(routes);
server.start();
// use server.port
server.stop();
```

### `test/helpers/eth-sync-routes.ts` — 37 lines

Default ETH sync route fixtures (block/current, balance, txs, erc20/balances, /v1/currencies CAL). Reused across discover, receive, send tests.

The `/v1/currencies` route includes a non-empty body and `X-Ledger-Commit` header — without these, the CAL store throws `LedgerAPI4xx` (empty array → 404 → `remapRtkQueryError`).

### `test/helpers/session-fixture.ts` — 27 lines

```ts
export function makeSessionDir(entries: SessionEntry[]) {
  const tmpDir = mkdtempSync(join(tmpdir(), "wallet-cli-test-"));
  const appDir = join(tmpDir, APP_NAME);
  mkdirSync(appDir);
  writeFileSync(join(appDir, SESSION_FILE), YAML.stringify({ accounts: entries }));
  return { env: { XDG_STATE_HOME: tmpDir }, cleanup: () => rmSync(tmpDir, { recursive: true, force: true }) };
}
```

Pass `env` to the CLI subprocess — `XDG_STATE_HOME` redirects `stateDir(APP_NAME)` to a temp dir per test.

### `test/helpers/constants.ts` — 8 lines

```ts
export const ETH_ADDRESS = "0x71C7656EC7ab88b098defB751B7401B5f6d8976F";
export const ETH_DESCRIPTOR = `account:1:address:ethereum:main:${ETH_ADDRESS}:m/44h/60h/0h/0/0`;

// Hardhat/Foundry "test test … junk" mnemonic m/44'/60'/0'/0/0
export const MOCK_ETH_ADDRESS = "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266";
export const MOCK_ETH_PUBKEY = "038318535b54105d4a7aae60c08fc45f9687181b4fdfc625bd1a753fa7397fed75";
export const MOCK_ETH_DESCRIPTOR = `account:1:address:ethereum:main:${MOCK_ETH_ADDRESS}:m/44h/60h/0h/0/0`;
```

The mock device returns `MOCK_ETH_*` (the well-known dev mnemonic address). For tests that don't use the device, `ETH_*` is used.

### `test/commands/README.md`

> Each test treats a CLI command as a black box: given known flags and mocked infrastructure, the command must produce the expected stdout shape (JSON envelope) and exit code. All external I/O is replaced — no real device or network needed.

### Per-test summaries

| Test file | Lines | What it asserts |
|---|---|---|
| `balances.test.ts` | 88 | (1) human output prints `ETH` + `1.5`; (2) JSON envelope with `command:"balances"`, `network:"ethereum:main"`, `balances` array containing `ethereum` asset at `1.5`; (3) **session label resolution** — `--account ethereum-1` resolves via `session.yaml`; (4) invalid descriptor exits code 1 with `{ ok:false, error: { message: "No account labeled…" } }`. |
| `discover.test.ts` | 79 | (1) JSON envelope `command:"account discover"`, `accounts: []` (empty discovery here because mock device routes are minimal); (2) **session persistence** — `account discover` writes `session.yaml` with labels matching `^ethereum-\d+$`; (3) **dedupe** — running discover twice does not duplicate accounts. |
| `operations.test.ts` | 125 | (1) Empty-history → `operations: []`; (2) One OUT tx → operation with `type:"OUT"`, hash matching, `senders` containing the address. Uses a full ETH transaction fixture with confirmations, gas, `transfer_events: []`. |
| `receive.test.ts` | 33 | One test: `receive --verify` with mock DMK returns `data.address` matching the configured `MOCK_ETH_ADDRESS` (case-insensitive). Confirms the entire ConnectApp → CallTaskInApp → bridge.receive(verify=true) chain works through the mock seam. |
| `send.test.ts` | 72 | One test: `send --dry-run` with mock DMK and SEND_ROUTES (balance=1 ETH, nonce, gas estimate, gas tracker barometer) returns `{ dry_run:true, recipient, amount, fee }`. **Does NOT exercise actual signing** — that path is not yet covered by an integration test. |
| `session.test.ts` | 77 | (1) `session view` empty → `No accounts` message; (2) labels and descriptors printed; (3) JSON output with accounts array; (4) `session reset` reports already-empty / removed count; (5) JSON output with `removed` count. Uses `runLocalCli` (no HTTP intercept). |

There are also unit tests not under `test/commands/` but co-located:
- `commands/inputs.test.ts` (19) — `resolveAccountArg` priority/fallback/throw.
- `device/connect-ledger-app.test.ts` — covers retry on transport framing errors.
- `device/wallet-cli-device-error.test.ts` — error mapping.
- `session/session-store.test.ts` (169) — Session class semantics.
- `shared/accountDescriptor/{adapters,network,v1}.test.ts` — V0↔V1 round-trip, network parsing.
- `shared/{response,ui}.test.ts` — envelope and isInteractive.
- `wallet/intents/{index,parse-amount}.test.ts` — Zod schema and `parseAmountWithTicker`.
- `wallet/formatter/human.test.ts` (268) — formatter output.
- `wallet/models.test.ts` — descriptor parsing.
- `commands/session/{view,reset}.test.ts` — colocated session command tests (in addition to `test/commands/session.test.ts`).

This is heavy unit + integration coverage. New commands are expected to add **both** a colocated unit test and a `test/commands/<name>.test.ts` integration test.

---

## Section 9 — Build, run, lint flow

### From repo root (after `pnpm i`)

```bash
pnpm wallet-cli start -- <command> [args]   # via root passthrough script
pnpm --filter @ledgerhq/wallet-cli start -- <command> [args]  # explicit
```

### From `apps/wallet-cli/`

```bash
pnpm start -- <command> [args]    # bun run ./src/cli.ts
pnpm dev                          # bunli dev (hot reload)
pnpm generate                     # regenerate .bunli/commands.gen.ts after adding/removing commands
pnpm generate:check               # CI gate
pnpm build                        # standalone binary -> dist/{darwin-arm64,linux-arm64,linux-x64,windows-x64}
pnpm typecheck                    # tsc --noEmit
pnpm test                         # bun test src/
pnpm coverage                     # bun test src/ --coverage
pnpm lint                         # oxlint
pnpm lint:fix                     # oxfmt + oxlint --fix
pnpm format / pnpm format:check   # oxfmt
```

### Bun version

- `engines.bun: ">=1.1.0"` (per `package.json`).
- Local dev currently uses bun-types `1.3.12` (per devDependencies).
- README says: "Bun ≥ 1.1.0 (`engines` in `package.json`)".

### Linux dev requirement

```bash
sudo apt-get update && sudo apt-get install libudev-dev libusb-1.0-0-dev
```

### Standalone build

`bunli build` produces native binaries per target with the `usb` addon embedded. With `minify: true`, each binary contains only its platform's prebuilt (`process.platform`/`process.arch` are dead-code-eliminated). The README notes that standalone builds under `dist/` "rely on `init-cwd.ts`" — but I did not find an `init-cwd.ts` in the source tree (it may have been renamed; the comment block in `bunli.config.ts` references `embed-usb-native.ts` instead). **Honest gap:** the README is slightly out of date here.

---

## Section 10 — The skill doc (`SKILL.md`)

Already largely excerpted in the briefing. Material additions worth surfacing for the guide:

### Sandbox warning (SKILL.md line 20)

> **Sandbox:** Commands that open the USB device (`account discover`, `receive`, `send`) **must** be run with `dangerouslyDisableSandbox: true` in the Bash tool — the sandbox blocks USB access and causes a `Timeout has occurred` error. Commands that don't need the device (`balances`, `operations`, `send --dry-run`) work fine in the sandbox.

This applies to AI agents driving the CLI from inside Claude Code (which is the most common path for this guide's audience writing test hooks). Plain CI / dev shell users do not hit this restriction.

### Device contention (SKILL.md line 10)

> **Device contention:** Only one command can use the device at a time. Never run two device-required commands in parallel — they will conflict and fail with `[object Object]` or a garbled APDU error. Run device commands sequentially.

This is enforced at the DMK level, not in `wallet-cli` code (the singleton `persistentDmk` simply can't hold two sessions). Implication: any test hook that needs to revoke before swapping must run **sequentially** with the swap test, never in `Promise.all`.

### Network aliases

`bitcoin` (= `:main`), `ethereum:mainnet` (alias → `:main`), `bitcoin:testnet`, `solana:devnet`, `ethereum:goerli`, `ethereum:sepolia`. Base uses the Ethereum app on device.

### "Base uses the Ethereum app"

A subtle production constraint that the test harness reflects: `WALLET_CLI_MOCK_APP_RESULTS` keys are app names ("Ethereum", "Solana", "Bitcoin"), not network names — so a base mock would still use `Ethereum`.

### Common errors

Already mapped in `wallet-cli-device-error.ts` (Section 4 above). The SKILL.md table is essentially the user-facing surface of that mapping.

---

## Section 11 — Cross-reference table

What calls what (top-down):

| Caller | Callee | Purpose |
|---|---|---|
| `cli.ts` | `embed-usb-native.ts` (side effect) | Pin native addon at startup |
| `cli.ts` | `live-common-setup.ts` (side effect) | Coin modules + DMK transport register |
| `cli.ts` | `.bunli/commands.gen.ts` (side effect) | Static command registration |
| `cli.ts` | `@bunli/core::createCLI(bunliConfig)` | Build CLI from config |
| `cli.ts` | `register-dmk-transport::disposeWalletCliDmkTransportFully` | Graceful shutdown |
| `commands/balances.ts` | `commands/inputs.ts` (resolveAccountArg/Descriptor) | Flag → V0 descriptor |
| `commands/balances.ts` | `output::createCommandOutput` | Human/JSON abstraction |
| `commands/balances.ts` | `wallet/index::WalletAdapter.getAccountBalances` | Data fetch |
| `commands/operations.ts` | (same as balances + `getAccountOperations`) | |
| `commands/receive.ts` | `session/bridge-device-session::withCurrencyDeviceSession` | Open device + manager app |
| `commands/receive.ts` | `wallet/index::WalletAdapter.verifyAddress` | Addr verify |
| `commands/send.ts` | `wallet/intents/index::TransactionIntentSchema.parse` | Validate intent |
| `commands/send.ts` | `session/bridge-device-session::withCurrencyDeviceSession` | Open device + ETH/BTC/SOL app |
| `commands/send.ts` | `wallet/index::WalletAdapter.send / prepareSend` | Sign+broadcast |
| `commands/account/discover.ts` | `withCurrencyDeviceSession` | Open device |
| `commands/account/discover.ts` | `WalletAdapter.discoverAccounts` (Observable) | Bridge scan |
| `commands/account/discover.ts` | `session/session-store::Session.{read,addDescriptors,write}` | Persist labels |
| `commands/session/{view,reset}.ts` | `session/session-store::Session` | Session I/O |
| `wallet/index::WalletAdapter` | `wallet/compatibility/bridge.ts::BridgeAdapter` | Always for discover/receive/send/getFreshAddress/operations |
| `wallet/index::WalletAdapter` | `wallet/compatibility/alpaca.ts::AlpacaAdapter` | Only for EVM `getBalances` |
| `wallet/compatibility/bridge.ts` | `@ledgerhq/live-common/bridge/index::{getCurrencyBridge,getAccountBridge}` | Live-common bridge |
| `wallet/compatibility/bridge.ts` | `bridge.signOperation({ deviceId: WALLET_CLI_DMK_DEVICE_ID, … })` | Routes through `registerTransportModule` to DMK |
| `withCurrencyDeviceSession` | `device/register-dmk-transport::ensureWalletCliDmkTransport` | Open USB session |
| `withCurrencyDeviceSession` | `device/connect-ledger-app::connectLedgerApp` | Open Manager app |
| `withCurrencyDeviceSession` | `device/wallet-cli-device-error::toWalletCliDeviceError` | Map errors |
| `device/register-dmk-transport` | `device/dmk::createDeviceManagementKit` | Build DMK (node-webusb) |
| `device/register-dmk-transport` | `live-common::registerTransportModule({ id: WALLET_CLI_DMK_DEVICE_ID })` | Wire DMK into live-common |
| `device/wallet-cli-dmk-transport` | `dmk.sendApdu(...)` | APDU exchange |
| `output.ts` (Json mode) | `shared/response::makeEnvelope` | Envelope shape |
| `output.ts` (Json mode error) | `writeSync(1, …)` + `process.exit(1)` | Bypass stderr buffer drain bug |
| `output.ts` | `wallet/formatter/{human,json}` | Format data |
| Tests `runCli` | `test/helpers/wrapper.ts` → `http-intercept` (+ `dmk-intercept`?) → `cli.ts` | Spawn CLI under intercepts |
| Tests `runLocalCli` | `test/helpers/wrapper-local.ts` → `cli.ts` | Spawn CLI without intercepts |
| `dmk-intercept.ts` | `device/register-dmk-transport::_setTestDmkTransport` | Install MockDmk |
| `http-intercept.ts` | patches `globalThis.fetch`, `node:http.request`, `node:https.request`, axios | Redirect to mock port |

---

## Section 12 — Implications for the QAA-615 spike

Concrete points the writers can lift:

1. **EVM `--data` already exists.** Path 2 in the briefing ("generalize send to accept arbitrary calldata") is **already done**. The spike doesn't need to add CLI surface to ship a working PoC — only documentation and ergonomics.
2. **EVM `data` field is plumbed through `bridge.ts::buildTxExtras`** as `Buffer.from(hex, "hex")` — calldata reaches the EVM bridge transaction unchanged.
3. **A `revoke` subcommand** is a thin wrapper around `send`. Its handler builds `data = "0x095ea7b3" + zero_pad32(spender) + zero_pad32(0)`, calls into the same `WalletAdapter.send` path, and routes through the same `withCurrencyDeviceSession`. Add file at `src/commands/revoke.ts` (or under a new `allowance/` group), run `pnpm generate`, add a test under `test/commands/revoke.test.ts`.
4. **Test harness ready.** `WALLET_CLI_MOCK_APP_RESULTS={"Ethereum":{publicKey,address}}` already supports the verify path. To assert the revoke calldata reaches the bridge, the test needs to inspect the `prepareTransaction` patch — the existing send.test.ts pattern (mock estimate-gas, gas-tracker, balance) is the template.
5. **Device contention is real.** Test hooks that revoke + swap must serialize. The `persistentDmk` singleton enforces this at runtime; in tests, `Bun.spawn` per command serializes naturally.
6. **JSON envelopes** are stable contract: hooks consuming `wallet-cli` should parse `{ status: "success", command, network, account, …, timestamp }` for success and `{ ok: false, error: { command, message } }` for errors.
7. **Session labels** mean test setup can be `account discover ethereum --output json` once, then every subsequent hook can use `--account ethereum-1` rather than carrying the full descriptor.

---

## Section 13 — Honest gaps in this research

Where this report makes claims I could not verify on the file system:

- **`init-cwd.ts`** — referenced in `README.md` as the standalone-build cwd helper, but no file by that name exists in `src/`. The same explanation in `bunli.config.ts` points to `embed-usb-native.ts`. Treating README as out-of-date.
- **`base` currency support** — the README says supported currencies are bitcoin / ethereum / solana, while the SKILL.md and `live-common-setup.ts` mention base via the EVM family using the Ethereum app. There is no explicit `setSupportedCurrencies(["base"])` call. Base support is **implicit via EVM** (any currency with `family: "evm"` should work), but `live-common-setup.ts` only `setSupportedCurrencies(["bitcoin", "ethereum", "solana"])`. So technically: base may not currently be reachable in the CLI surface even though SKILL.md mentions it.
- **`node-webusb/` internals** — I did not read the four files in detail. They are the production transport, replaced wholesale by mock-dmk under test, and unlikely to be material for guide writers.
- **`@bunli/core` patch** — `cli.ts` mentions a patch that removes the dynamic `commands.gen.ts` import, but the patch source is not in this repo (it's in the `bunli` npm package or a local patch under `patches/`). Not investigated. Implication: regenerating Bunli itself could undo this — note for the guide.
- **Coverage of `*.test.ts` file headers** — I summarized 6 integration tests verbatim. The colocated unit tests (e.g. `human.test.ts` 268 lines, `intents/index.test.ts` 189 lines, `session-store.test.ts` 169 lines) were sampled by line counts only, not read in full. They are likely the canonical reference for adding `revoke` unit tests, but are out of scope for this map.
- **CHANGELOG.md** (~13k bytes) — not read. Useful for the guide's "what changed when" section if writers want it.

---

End of report.
