# Part 5 v2 — Consolidated Writer Briefing

**Source of truth.** Branch `support/qaa_add_revoke_token` (commit `4fa4868a35`) authored by Abdurrahman SASTIM. The legacy `apps/cli` (`@ledgerhq/live-cli`) is the canonical QAA CLI. wallet-cli is **out of scope** for QAA and must not be taught.

This is the single primary source for the 6 writer agents producing Part 5 v2. Read it before writing.

---

## 1. The senior's commit (3 files, 37 lines)

### `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts` +20

```ts
export const revokeTokenCommand = async (account: TokenAccount, spender: string) => {
  const original = setDisableTransactionBroadcastEnv("0");

  const result = runCliTokenApproval({
    currency: account.currency.speculosApp.name,
    index: account.index,
    spender,
    token: account.currency.id,
    mode: "revokeApproval",
    waitConfirmation: true,
  });

  try {
    await approveToken();
  } finally {
    setDisableTransactionBroadcastEnv(original);
  }
  return await result;
};
```

### `e2e/desktop/tests/page/swap.page.ts` +15

```ts
@step("Ensure token approval")  // copy-paste bug; final PR should say "Revoke token approval"
async revokeTokenApproval(fromAccount: Account | TokenAccount, provider: Provider) {
  if (!provider.contractAddress || !fromAccount.parentAccount) return;

  const previousSpeculosPort = getEnv("SPECULOS_API_PORT");
  const speculos = await launchSpeculos(fromAccount.currency.speculosApp.name);
  try {
    const result = await revokeTokenCommand(fromAccount, provider.contractAddress);
    await allure.description(`Token revoke result for ${provider.uiName}:\n\n ${result}`);
  } finally {
    await cleanSpeculos(speculos, previousSpeculosPort);
  }
}
```

### `e2e/desktop/tests/specs/provider.swap.spec.ts` +1 -1

```diff
- await app.swap.ensureTokenApproval(fromAccount, provider, minAmount);
+ //await app.swap.ensureTokenApproval(fromAccount, provider, minAmount);
+ await app.swap.revokeTokenApproval(fromAccount, provider);
```

**This is dev-loop code, not the final shape.** Final PR will likely keep both: revoke as a `beforeEach` cleanup, ensureTokenApproval as the per-test setup.

---

## 2. The 5-layer architecture

| # | Path | Role |
|---|---|---|
| 1 | `apps/cli/bin/index.js` | Legacy CLI binary (Node, Commander) |
| 2 | `libs/ledger-live-common/src/e2e/runCli.ts` | `child_process.spawn` engine + retries |
| 3 | `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts` | Typed TypeScript wrappers |
| 4 | `libs/ledger-live-common/src/e2e/families/evm.ts` | Speculos device-tap automation |
| 5 | `e2e/desktop/tests/page/swap.page.ts`, `e2e/mobile/page/trade/swap.page.ts` | POM methods orchestrating Speculos + CLI |

---

## 3. apps/cli command catalog (16 blockchain commands)

Path: `apps/cli/src/commands/blockchain/`

| File | Purpose | QAA uses? |
|---|---|---|
| `bot.ts` | Speculos bot (long-running test stress) | No (out of QAA scope) |
| `botPortfolio.ts` | Bot portfolio management | No |
| `botTransfer.ts` | Bot-driven transfers | No |
| `broadcast.ts` | Broadcast a signed tx | Maybe |
| `confirmOp.ts` | Confirm an operation | No |
| `derivation.ts` | Derivation path utility | No |
| `estimateMaxSpendable.ts` | Estimate max sendable | No |
| `generateTestScanAccounts.ts` | Test fixture generator | No |
| `generateTestTransaction.ts` | Test fixture generator | No |
| `getAddress.ts` | **Read device address** | **YES** — `runCliGetAddress` |
| `getTransactionStatus.ts` | Get tx status | No |
| `receive.ts` | Receive flow utility | No |
| `send.ts` | **Sign + broadcast a tx; supports `--mode approve\|revokeApproval`** | **YES** — `runCliTokenApproval` |
| `signMessage.ts` | Sign EIP-191/712 | No |
| `sync.ts` | Account sync | No |
| `testDetectOpCollision.ts`, `testGetTrustedInputFromTxHash.ts` | Test utilities | No |
| `tokenAllowance.ts` | **Read ERC-20 allowance** | **YES** — `runCliGetTokenAllowance` |

Plus `apps/cli/src/commands/live/`: liveData (used by `runCliLiveData`), and `apps/cli/src/commands/device/` (out of QAA scope).

**bin / scripts** from `apps/cli/package.json`:
```json
"bin": { "ledger-live": "./bin/index.js" },
"scripts": {
  "prebuild": "zx ./scripts/gen.mjs",
  "build": "zx ./scripts/build.mjs",
  "test": "zx ./scripts/test.mjs"
}
```

The CLI is invoked from tests as `node apps/cli/bin/index.js <args>` via `child_process.spawn`. Pre-build via `zx scripts/gen.mjs` generates the commands index.

### 3.1 send.ts — the approve/revoke command

`apps/cli/src/commands/blockchain/send.ts` is a Commander command. Key flags from the source:

- `--ignore-errors` (Boolean)
- `--disable-broadcast` (Boolean)
- `--wait-confirmation` (Boolean) + `--wait-confirmation-timeout` (Number ms)
- `--format` (default | json | silent)
- Plus `scanCommonOpts` (currency, index, scheme, etc.) + `inferTransactionsOpts` (amount, spender, token, **mode**, **approveAmount**, etc.)

Key code path for the broadcast switch:

```ts
.pipe(
  // …
  bridge.signOperation({ account, transaction: t, deviceId: opts.device || "" })
    .pipe(
      map(toSignOperationEventRaw),
      // ── BROADCAST GATE ──
      ...(opts["disable-broadcast"] || getEnv("DISABLE_TRANSACTION_BROADCAST")
        ? []  // skip broadcast if either is truthy
        : [
            concatMap((e) => {
              if (e.type === "signed") {
                // log, then call bridge.broadcast(...)
                // after broadcast, optionally waitForTransactionConfirmation()
              }
              return of(e);
            }),
          ]),
      // …
    );
```

So `DISABLE_TRANSACTION_BROADCAST=1` → CLI signs locally but skips the `bridge.broadcast` call. `=0` → broadcast happens.

The `--mode revokeApproval` value is parsed by `inferTransactionsOpts` (in `apps/cli/src/transaction.ts`), which dispatches into `libs/coin-modules/coin-evm/src/cli-transaction.ts:inferTransactions` — that's where the EVM `mode` is converted into the transaction shape (recipient = token contract, amount = 0, data = `getErc20ApproveData(spender, 0n)`).

### 3.2 tokenAllowance.ts — the allowance reader

`apps/cli/src/commands/blockchain/tokenAllowance.ts` builds a minimal EVM account from `--ownerAddress` and calls `getEvmTokenAllowance` from `libs/ledger-live-common/src/families/evm/getTokenAllowance.ts`. Outputs JSON when `--format json` is passed (parsed by `parseTokenAllowanceCliOutput` in cliCommandsUtils).

---

## 4. Layer 2 — `libs/ledger-live-common/src/e2e/runCli.ts`

### Spawn engine (verbatim)

```ts
export const LEDGER_LIVE_CLI_BIN = path.resolve(__dirname, "../../../../apps/cli/bin/index.js");

export function runCliCommand(command: string): Promise<string> {
  console.warn(`[CLI] Executing: ledger-live ${command.replace(/\+/g, " ")}`);

  return new Promise((resolve, reject) => {
    const args = command.split("+");
    const child = spawn("node", [LEDGER_LIVE_CLI_BIN, ...args], {
      stdio: "pipe",
      env: process.env,
    });

    let output = "";
    let errorOutput = "";

    child.stdout.on("data", data => { output += data.toString(); });
    child.stderr.on("data", data => { errorOutput += data.toString(); });

    child.on("exit", code => {
      if (code === 0) {
        resolve(output);
      } else {
        const errorDetails = [
          `❌ Failed to execute CLI command`,
          `🔍 Command: ${command}`,
          `💱 Currency: ${parseCliFlag(command, "currency")}`,
          `🔢 Index: ${parseCliFlag(command, "index") ?? "N/A"}`,
          `🔢 Exit Code: ${code}`,
          errorOutput ? `🧾 CLI Error : ${errorOutput.trim()}` : "",
        ].join("\n");
        reject(new Error(errorDetails));
      }
    });

    child.on("error", error => reject(new Error(`Error executing CLI command: ${sanitizeError(error)}`)));
  });
}

export async function runCliCommandWithRetry(command, retries = 3, delayMs = 3000) {
  let lastError = null;
  for (let attempt = 1; attempt <= retries; attempt++) {
    try { return await runCliCommand(command); }
    catch (err) {
      lastError = err instanceof Error ? err : new Error(String(err));
      const willRetry = attempt < retries && isRetryableError(lastError.message);
      if (!willRetry) throw sanitizeError(lastError);
      console.warn(`⚠️ CLI attempt ${attempt}/${retries} failed with retryable error – retrying in ${delayMs}ms…`);
      await sleep(delayMs);
    }
  }
  throw sanitizeError(lastError!);
}
```

### Public engine helpers

| Helper | CLI command produced | Notes |
|---|---|---|
| `runCliLiveData(opts)` | `liveData --currency X --index N [--scheme M] --add --appjson <path>` | Seeds account |
| `runCliGetAddress(opts)` | `getAddress --currency X --path m/...` | Reads device address |
| `runCliGetTokenAllowance(opts)` | `tokenAllowance --currency X --token Y --spender Z --index N --ownerAddress 0x... [--format json]` | Reads allowance |
| `runCliTokenApproval(opts)` | `send --currency X --mode approve\|revokeApproval --token Y --spender Z --index N [--approveAmount N] [--wait-confirmation]` | Sign + (maybe) broadcast |
| Plus `runCliKeyRingProtocol`, `runCliLedgerSync` (WalletSync — not QAA scope) | | |

### The "+" arg separator quirk

Args are joined with `+` to avoid shell-escape pain. The engine splits them back into argv before spawn. Example: `runCliTokenApproval` builds:
```
"send+--currency+ethereum+--mode+revokeApproval+--token+ethereum/erc20/usd_coin+--spender+0xRouter+--index+0+--wait-confirmation"
```
Split → `["send", "--currency", "ethereum", "--mode", "revokeApproval", ...]`.

---

## 5. Layer 3 — `cliCommandsUtils.ts` (10 typed wrappers)

| Helper | Inputs | Returns | Used by |
|---|---|---|---|
| `getAccountAddress(account)` | `Account \| TokenAccount` | `Promise<string>` (also writes to `account.address`) | Every spec needing device address |
| `liveDataCommand(account, options?)` | account + options | curried `(userdataPath?) => Promise<void>` | Desktop fixture `cliCommands: [...]` |
| `liveDataWithAddressCommand(account, options?)` | same | curried, also caches address | When spec needs both seed + address |
| `liveDataWithParentAddressCommand(parent, child)` | parent + child token | curried | Token account resolution |
| `liveDataWithRecipientAddressCommand(tx, options?)` | a `Transaction` | curried | Send flow specs |
| `parseTokenAllowanceCliOutput(output)` | CLI stdout | `{ allowanceStr, unitMagnitude }` | Internal parser |
| `isTokenAllowanceSufficientCommand(account, spender, minAmount)` | TokenAccount, spender, min | allowance string if ≥ min, else `0` | `ensureTokenApproval` POM (the pre-check) |
| `approveTokenCommand(account, spender, approveAmount)` | TokenAccount, spender, amount | CLI stdout | Sets allowance, broadcast on |
| **`revokeTokenCommand(account, spender)` (NEW)** | TokenAccount, spender | CLI stdout | Clears allowance to 0, broadcast on |
| `setDisableTransactionBroadcastEnv(value)` | string \| undefined | previous env value | Broadcast env management |

The curried-function pattern enables the `cliCommands: [liveDataCommand(account)]` fixture syntax — fixture engine calls each curried function with the userdataPath at the right point in the lifecycle.

---

## 6. Layer 4 — `families/evm.ts` (Speculos device-tap automation)

```ts
export async function approveToken() {
  if (isTouchDevice()) {
    return approveTokenTouchDevices();   // Stax / Flex
  }
  return approveTokenButtonDevice();     // Nano S / X / S+
}

export async function approveTokenTouchDevices() {
  await waitForReviewTransaction();
  await pressUntilTextFound(DeviceLabels.HOLD_TO_SIGN);
  await longPressAndRelease(DeviceLabels.HOLD_TO_SIGN, 3);  // 3-second hold
}

export const approveTokenButtonDevice = withDeviceController(
  ({ getButtonsController }) => async () => {
    await waitFor(DeviceLabels.REVIEW_TRANSACTION_TO);
    await pressUntilTextFound(DeviceLabels.SIGN_TRANSACTION);
    await getButtonsController().both();  // press both buttons
  },
);
```

The CLI subprocess and `approveToken()` run **in parallel**: while the CLI is blocked waiting for device confirmation, this helper *is* the user, driving Speculos's REST API to walk through screens.

Screen primitives (also in `families/evm.ts` or imported from a sibling):
- `waitFor(label)` — wait for a screen-text label to appear
- `pressUntilTextFound(label)` — press button repeatedly until label appears
- `pressAndRelease(direction, x, y)` — simulate a button or touch press
- `longPressAndRelease(label, seconds)` — hold to sign on touch devices
- `fetchCurrentScreenTexts(port)` — pull current screen text via Speculos REST
- `withDeviceController(fn)` — DI wrapper providing button controller

`DeviceLabels.*` constants come from a shared label registry (touch and button devices use the same logical labels but possibly different on-device strings).

---

## 7. Layer 5 — POM methods (`swap.page.ts`)

### Existing `ensureTokenApproval` (the twin of the new revoke method)

```ts
async ensureTokenApproval(fromAccount, provider, minAmount) {
  if (!provider.contractAddress || !fromAccount.parentAccount) return;

  // Pre-check — skip the device dance if we already have enough
  const currentAllowance = await isTokenAllowanceSufficientCommand(
    fromAccount, provider.contractAddress, minAmount,
  );
  console.log("CLI result: Current Allowance: ", currentAllowance);
  if (currentAllowance) return;

  const previousSpeculosPort = getEnv("SPECULOS_API_PORT");
  const speculos = await launchSpeculos(fromAccount.currency.speculosApp.name);
  try {
    const result = await approveTokenCommand(
      fromAccount,
      provider.contractAddress,
      new BigNumber(minAmount).times(12).div(10).toFixed(),  // 1.2x for slippage buffer
    );
    await allure.description(`Token approval result for ${provider.uiName}:\n\n ${result}`);
  } finally {
    await cleanSpeculos(speculos, previousSpeculosPort);
  }
}
```

The new `revokeTokenApproval` (senior's commit, full content above in §1) is the same shape minus the pre-check and the slippage math. Always revokes; revoking-an-already-zero allowance is a no-op gas-wise.

---

## 8. Speculos lifecycle (`e2e/desktop/tests/utils/speculosUtils.ts`)

Verbatim:

```ts
import { setEnv } from "@ledgerhq/live-env";
import {
  startSpeculos, specs, stopSpeculos, type SpeculosDevice, getSpeculosAddress,
} from "@ledgerhq/live-common/e2e/speculos";
import invariant from "invariant";
import * as allure from "allure-js-commons";
import { waitForSpeculosReady } from "@ledgerhq/live-common/e2e/speculosCI";
import { CLI } from "./cliUtils";
import { unregisterTransportModule } from "@ledgerhq/live-common/hw/index";

export async function launchSpeculos(appName, testTitle?, previousDevice?): Promise<SpeculosDevice> {
  if (testTitle) testTitle = testTitle.replace(/ /g, "_");

  if (previousDevice) await cleanSpeculos(previousDevice);

  const device = await startSpeculos(
    testTitle ?? "cli_speculos",
    specs[appName.replace(/ /g, "_")],
    previousDevice?.port,
  );
  invariant(device, "[E2E Setup] Speculos not started");

  if (process.env.REMOTE_SPECULOS === "true") {
    await waitForSpeculosReady(device.id);
  }

  setEnv("SPECULOS_API_PORT", device.port);
  process.env.SPECULOS_API_PORT = device.port.toString();
  CLI.registerSpeculosTransport(device.port.toString(), getSpeculosAddress());

  let info = `App: ${device.appName || ""} (${device.appVersion || ""})`;
  if (device.dependencies?.length) {
    info += `\nDependencies: ${device.dependencies?.map(dep => dep.name + " (" + dep.appVersion + ")").join(", ") || ""}`;
  }
  await allure.description("SPECULOS\n" + info);

  return device;
}

export async function cleanSpeculos(speculos, previousPort?) {
  await stopSpeculos(speculos.id);
  unregisterTransportModule("speculos-http-" + String(speculos.port));
  if (previousPort) {
    setEnv("SPECULOS_API_PORT", previousPort);
    process.env.SPECULOS_API_PORT = String(previousPort);
  }
}
```

Key facts:
- `specs` map: `{ Ethereum: <spec>, Bitcoin: <spec>, ... }` — pulls firmware + app paths from `@ledgerhq/live-common/e2e/speculos`
- Both `setEnv("SPECULOS_API_PORT", port)` and `process.env.SPECULOS_API_PORT = ...` are set defensively
- `CLI.registerSpeculosTransport(port, address)` registers `hw-transport-node-speculos-http` so the spawned CLI process can connect
- `unregisterTransportModule("speculos-http-" + port)` on cleanup avoids transport-registry leaks across tests
- Snapshot/restore pattern: tests that need TWO Speculos instances (e.g., Exchange app for swap + Ethereum app for revoke) save the previous port, swap, and restore in `finally`

The `startSpeculos` / `stopSpeculos` primitives live in `@ledgerhq/live-common/e2e/speculos`. `REMOTE_SPECULOS=true` toggles between local Docker and a remote Stargate-managed Speculos pool (production CI).

---

## 9. `DISABLE_TRANSACTION_BROADCAST` flow

### Read sites
- `apps/cli/src/commands/blockchain/send.ts` — `getEnv("DISABLE_TRANSACTION_BROADCAST")` in the rxjs pipe; if truthy, the `.broadcast(...)` operator is skipped (see §3.1)
- `apps/cli/src/commands/blockchain/botTransfer.ts` — bot uses same gate
- `libs/ledger-live-common/src/exchange/swap/setBroadcastTransaction.ts`, `wallet-api/react.ts`, `wallet-api/useDappLogic.ts`, `wallet-api/ACRE/server.ts`, `hooks/useBroadcast.ts`, `bot/engine.ts` — live-common consumers

### Set sites
- `cliCommandsUtils.ts:setDisableTransactionBroadcastEnv` (the canonical helper)
- `e2e/mobile/specs/swap/otherTestCases/swap.other.ts` — `setEnv("DISABLE_TRANSACTION_BROADCAST", true)` at module load (mobile swap tests default-off broadcast)
- `approveTokenCommand` and `revokeTokenCommand` flip to `"0"` in their try/finally

### Default
Unset → falsy → broadcast happens. Most test paths explicitly set `"1"` to keep nightly runs deterministic.

### Try/finally discipline
Every set site that flips the flag must restore the original value, even on throw. The pattern in `cliCommandsUtils.ts`:

```ts
const original = setDisableTransactionBroadcastEnv("0");
const result = runCliTokenApproval({...});
try {
  await approveToken();
} finally {
  setDisableTransactionBroadcastEnv(original);
}
return await result;
```

Without the `finally`, an unhandled throw between flip and restore would leave the env mutated for the rest of the test run, silently flipping broadcast for every subsequent CLI call.

---

## 10. Existing CLI usage in tests (catalog)

### Imports of `cliCommandsUtils` in `e2e/desktop/`

| Spec | Helpers imported |
|---|---|
| `tests/page/swap.page.ts` | `approveTokenCommand, isTokenAllowanceSufficientCommand, revokeTokenCommand` |
| `tests/specs/earn.v2.spec.ts` | `liveDataCommand, liveDataWithAddressCommand` |
| `tests/specs/newSendFlow.tx.spec.ts` | `liveDataWithRecipientAddressCommand` |
| `tests/specs/receive.address.spec.ts` | `liveDataCommand` |
| `tests/specs/validation.swap.spec.ts` | `liveDataWithAddressCommand` |
| `tests/specs/earn.spec.ts` | `liveDataCommand, liveDataWithAddressCommand` |
| `tests/specs/rename.account.spec.ts` | `liveDataCommand` |
| `tests/specs/delete.account.spec.ts` | `liveDataCommand` |
| `tests/specs/settings.spec.ts` | `liveDataCommand` |
| `tests/specs/delegate.spec.ts` | `liveDataCommand` |
| `tests/specs/entrypoint.swap.spec.ts` | `liveDataWithAddressCommand` |
| `tests/specs/accounts.swap.spec.ts` | `liveDataWithAddressCommand` |
| `tests/specs/subAccount.spec.ts` | `liveDataCommand, liveDataWithAddressCommand, liveDataWithParentAddressCommand` |
| `tests/specs/buySell.spec.ts` | `liveDataCommand` |
| `tests/specs/activate.private.balance.spec.ts` | `liveDataCommand` |
| `tests/specs/send.swap.spec.ts` | `liveDataWithAddressCommand` |
| `tests/specs/ui.swap.spec.ts` | `liveDataWithAddressCommand` |
| `tests/specs/provider.swap.spec.ts` | `liveDataWithAddressCommand` |
| `tests/specs/send.tx.spec.ts` | (multiple) |

### Mobile injection
`e2e/mobile/jest.environment.ts:29` imports `* as cliCommandsUtils`, line 138 does `Object.assign(this.global, cliCommandsUtils)` — every helper becomes a global. Type defs at `e2e/mobile/types/global.d.ts:104-111`. Mobile specs call e.g. `await liveDataCommand(account)(userdataPath)` without an import.

### Desktop fixture engine
`e2e/desktop/tests/fixtures/common.ts` consumes the `cliCommands: [...]` field from test fixture data and runs each curried function in the test setup phase.

### `cliCommands: [...]` usages (sample)
- `earn.v2.spec.ts` lines 86, 132, 195, 220, 254, 298, 327 — heavy user
- `delegate.spec.ts` lines 110, 188, 272, 322, 374, 433, 486, 540 — heavy user
- `receive.address.spec.ts` lines 29, 92
- `newSendFlow.tx.spec.ts:113` (uses `liveDataWithRecipientAddressCommand`)
- `earn.spec.ts:55, 213`

This is the **canonical fixture pattern**: declare the CLI calls needed before the test in the test's metadata; the fixture engine runs them; the test then drives the UI knowing the world is in the right state.

---

## 11. Approvals & revokes — blockchain primer

ERC-20 storage:
```
USDC contract:
  balances: { 0xYou: 100, 0xAlice: 50, ... }
  allowances: { 0xYou: { 0xUniswap: 0, 0xLifi: 0, ... }, ... }
```

| Op | Calldata | Effect |
|---|---|---|
| Approve N | `0x095ea7b3 + spender(32) + N(32)` | `allowances[you][spender] = N` |
| Revoke | `0x095ea7b3 + spender(32) + 0(32)` | `allowances[you][spender] = 0` |
| Unlimited | `0x095ea7b3 + spender(32) + 2^256-1(32)` | "Infinite approval" footgun |

Selector `0x095ea7b3` = `keccak256("approve(address,uint256)")[0:4]`. Ledger's clear-sign decoder recognises it and renders human-readable screens.

Allowance state lives **on the token contract on-chain**. It survives reboots, simulator restarts, repo re-clones. Only another `approve(spender, X)` transaction changes it. That's why broadcast-enabled regression tests need a revoke hook between iterations — otherwise run #2 sees state run #1 left behind.

---

## 12. Manual revoke tools

### revoke.cash (`https://revoke.cash`)
- Open-source, runs by Rosco Kalis. Source on GitHub.
- Connect via WalletConnect, MetaMask, or paste an address (read-only)
- Lists all active ERC-20 allowances + ERC-721/1155 approvals per token + spender
- "Revoke" button signs `approve(spender, 0)` via the connected wallet, broadcasts
- Cost: gas only
- When QAA reaches for it: cleanup after manual testing, audit messy local state, recover from broken Speculos sessions

### Etherscan Read tab
`etherscan.io/token/<contract>#readContract`
- Call `allowance(owner, spender)` — read-only, no wallet needed
- Returns current allowance in raw smallest units (wei equivalent for that token's decimals)

### Etherscan Write tab
`etherscan.io/token/<contract>#writeContract`
- Connect MetaMask
- Call `approve(spender, 0)` directly
- Used when revoke.cash doesn't list a token (rare edge cases)

### When QAA uses these
- Local Speculos went weird → clean a real testnet seed without running the harness
- One-off audit ("what's actually approved on this seed?")
- CI revoke fails → see on-chain state directly to debug
- Verifying after the CLI hook ran ("did it actually land?")

### Hygiene
- Never connect a wallet you don't own
- Prefer testnets (Sepolia, Holesky) for QA work
- The QAA test seed is shared infrastructure — clean up after manual sessions

### Past-incident context
Infinite approvals were the vector in major DeFi drains (BadgerDAO 2021, multiple 2022 phishing campaigns). Industry response: revoke.cash adoption, wallet UI warnings on infinite approvals, the rise of EIP-2612 permit (signature-based, no on-chain allowance) for newer tokens.

---

## 13. Old Part 5 — to be deleted entirely

Currently `part5.md` (7,144 lines) teaches wallet-cli + Bun + Bunli + DMK + V1 descriptors. **Replace wholesale.** Don't migrate any chapters; the old structure was wrong.

---

## 14. New Part 5 chapter outline (10 chapters)

| Ch | Title | Approx lines | Writer |
|---|---|---|---|
| 5.1 | CLI in QA — apps/cli is the canonical CLI | 500 | W1 |
| 5.2 | Approvals and revokes — the blockchain primer | 400 | W1 |
| 5.3 | Manual revoke tools — revoke.cash, Etherscan | 250 | W2 |
| 5.4 | The five-layer integration | 900 | W3 |
| 5.5 | Speculos lifecycle in CLI tests | 500 | W2 |
| 5.6 | DISABLE_TRANSACTION_BROADCAST and broadcast discipline | 400 | W4 |
| 5.7 | Daily CLI workflow | 500 | W4 |
| 5.8 | Walkthrough QAA-615 — line-by-line | 700 | W5 |
| 5.9 | CLI Exercises and Challenges | 250 | W6 |
| Final | Part 5 Final Assessment | 200 | W6 |

Total ~4,600 lines. Replaces 7,144 lines of obsolete content.

---

## 15. Style template (mirror Part 4 byte-for-byte)

- `## Chapter Title` per chapter (no H1 except in the very first writer's fragment — "# PART 5 — CLI AUTOMATION")
- `### N.X.Y` numbered subsections
- `<div class="chapter-intro">…</div>` chapter intros
- `<div class="chapter-outro">…</div>` chapter outros
- Quizzes:
```
<div class="quiz-container" data-pass-threshold="80">
  <h3>Chapter N.X Quiz</h3>
  <p class="quiz-subtitle">M questions · 80% to pass</p>
  <div class="quiz-progress"><div class="quiz-progress-bar"></div></div>
  <div class="quiz-question" data-correct="A">
    <p><strong>Q1.</strong> ...</p>
    <div class="quiz-choices">
      <button class="quiz-choice" data-value="A">A) ...</button>
      ...
    </div>
    <p class="quiz-explanation">...</p>
  </div>
  ...
  <div class="quiz-score"></div>
</div>
```

NO emojis. NO Co-Authored-By.

---

## 16. Anchor sanitisation

docsify auto-generates anchors from H2 headings. Apostrophes, colons, parens break navigation. Safe formats:
- `cli-in-qa-apps-cli-is-the-canonical-cli`
- `approvals-and-revokes-the-blockchain-primer`
- `manual-revoke-tools-revoke-cash-and-etherscan`
- `the-five-layer-integration`
- `speculos-lifecycle-in-cli-tests`
- `disable-transaction-broadcast-and-broadcast-discipline`
- `daily-cli-workflow`
- `walkthrough-qaa-615-line-by-line`
- `cli-exercises-and-challenges`
- `part-5-final-assessment`
