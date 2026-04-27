# R2 — `wallet-cli revoke` spike (QAA-615)

Date: 2026-04-27
Author: research agent R2
Mission: answer "what code is needed to add a `revoke` subcommand to wallet-cli that emits an `approve(spender, 0)` ERC-20 transaction?"

---

## 1. Executive summary

**Recommendation: Path C — ship `revoke` as a thin command that delegates to the existing send pipeline, plus add an `approve` companion for symmetry.**

The work is *much smaller* than expected. Three reasons:

1. **The ERC-20 approve helper already exists in coin-evm.** `getErc20ApproveData(spender, amount)` lives at
   `libs/coin-modules/coin-evm/src/logic/getErc20Data.ts` and is already used by the legacy CLI's `revokeApproval` and `approve` modes (see `cli-transaction.ts`).
2. **The wallet-cli send pipeline already accepts arbitrary EVM calldata.** `send.ts` exposes a `--data` flag, the EVM intent schema (`wallet/intents/families/evm.ts`) accepts `data`, and the bridge adapter (`wallet/compatibility/bridge.ts:194-198`) already converts hex `data` into a `Buffer` and patches it onto the transaction. Path B is *already 90 % wired*.
3. **Clear-sign is already supported on the device.** `libs/coin-modules/coin-evm/src/deviceTransactionConfig.ts:71-105` decodes the ERC-20 `APPROVE` selector and produces "Type: Approve / Amount: 0 USDT (or Unlimited) / Address: <spender>" screens. No plugin work, no blind-sign fallback for known tokens.

What's actually missing from `wallet-cli`:
- A way for the user to express "approve X tokens to spender Y" *without* hand-encoding 68 bytes of calldata.
- A revoke-specific output line so the test hook can `grep` for it cleanly.

Estimated total work: **~150 LOC of new code + ~80 LOC of tests**, one new command file, one shared helper, no live-common changes.

Headline LOC budget:
- `commands/revoke.ts` (new): ~70
- `commands/approve.ts` (new, optional, free given the helper): ~70
- `wallet/intents/families/evm.ts` (add `mode: 'approve' | 'revoke'` discriminator OR reuse `data` passthrough): 0–10
- `wallet/erc20-calldata.ts` (new tiny helper, viem or ethers Interface): ~30
- `test/commands/revoke.test.ts`: ~80

---

## 2. How `send` signs ETH today (trace)

`apps/wallet-cli/src/commands/send.ts` (handler, lines 110-156):

1. Resolve descriptor from session/CLI arg.
2. Build a `TransactionIntent` via `INTENT_BUILDERS[family]` — for EVM the shape is:
   ```ts
   { family: "evm", recipient, amount, data?: "0x..." }
   ```
3. Validate via Zod (`TransactionIntentSchema.parse`).
4. If `--dry-run`: `wallet.prepareSend(descriptor, intent)` → returns `{ amount, fees, recipient }` (no device).
5. Otherwise: `withCurrencyDeviceSession(currencyId, …)` opens the device session, then
   `wallet.send(descriptor, intent, deviceId, dryRun)` returns an `Observable<SendEvent>`.

`WalletAdapter.send` delegates to `BridgeAdapter.send` (`wallet/compatibility/bridge.ts:124-184`). The signing path:

```ts
// bridge.ts:151
bridge.signOperation({ account, transaction: tx, deviceId }).subscribe({
  next: event => {
    if (event.type === "device-streaming")        emit("device-streaming");
    if (event.type === "device-signature-requested") emit("device-signature-requested");
    if (event.type === "device-signature-granted")   emit("device-signature-granted");
    if (event.type === "signed") signedOperation = event.signedOperation;
  },
  …
});
…
const op = await bridge.broadcast({ account, signedOperation });  // emits txHash
```

**Which DMK API?** None directly. The wallet-cli stays on the **legacy live-common AccountBridge** abstraction (`getAccountBridge(account)`). The bridge's `signOperation` internally drives `DmkSignerEth`, which calls `signTransaction` (an EIP-1559 RLP-encoded tx) through DMK's `executeDeviceAction`. wallet-cli does **not** call `signEthereumTransaction` directly anywhere — it goes through the bridge, which is exactly what we want for `revoke` (free clear-sign, free gas/nonce/EIP-1559 estimation, free broadcast).

**Input shape:** structured live-common `Transaction` object (mode, recipient, amount as BigNumber, optional `data: Buffer`, gasLimit, fee parameters). RLP encoding happens deep inside `signOperation`.

**Output:** a `SignedOperation` from which `bridge.broadcast` produces the canonical `Operation` (with `hash`).

### How the EVM transaction is built (the magic line)

`bridge.ts:194-198` (the `buildTxExtras` switch's `evm` branch):

```ts
case "evm":
  if (intent.data) {
    const hexData = intent.data.slice(2);              // strip 0x
    if (hexData.length > 0) patch.data = Buffer.from(hexData, "hex");
  }
  break;
```

That `patch.data` Buffer is fed to `bridge.updateTransaction(tx, { recipient, amount, …patch })`, the live-common bridge picks up the `data` field, and `craftTransaction` in coin-evm consumes it as the calldata body. **The plumbing is already there.** A revoke transaction is simply:

- `recipient = <token contract address>`
- `amount = 0` (native ETH amount; the calldata carries the 0-token approval)
- `data = "0x095ea7b3" + pad32(spender) + pad32(0)`

---

## 3. ERC-20 calldata in live-common — what helpers exist

### 3.1. The approve helper already exists

`libs/coin-modules/coin-evm/src/logic/getErc20Data.ts`:

```ts
import { ethers } from "ethers";
import ERC20ABI from "../abis/erc20.abi.json";

export function getErc20Data(recipient: string, amount: bigint): Buffer {
  const contract = new ethers.Interface(ERC20ABI);
  const data = contract.encodeFunctionData("transfer", [recipient, amount]);
  return Buffer.from(data.slice(2), "hex");
}

export function getErc20ApproveData(spender: string, amount: bigint): Buffer {
  const contract = new ethers.Interface(ERC20ABI);
  const data = contract.encodeFunctionData("approve", [spender, amount]);
  return Buffer.from(data.slice(2), "hex");
}
```

Both helpers return a raw `Buffer` (not 0x-prefixed). This is exactly what coin-evm's `Transaction.data` field expects.

### 3.2. The legacy CLI already wires "revokeApproval" and "approve"

`libs/coin-modules/coin-evm/src/cli-transaction.ts` (line 10):

```ts
const modes = ["send", "revokeApproval", "approve"] as const;
```

Used by the legacy `apps/cli/src/commands/blockchain/send.ts`. The wallet-cli rewrite simply hasn't surfaced these modes yet. The same pattern (resolve token by id → produce calldata via `getErc20ApproveData(spender, 0n)` → set `recipient = tokenContractAddress`, `amount = 0`, `data = …`) is the blueprint.

Snippet of the legacy `inferTransactions` for revokeApproval (lines 95-130):

```ts
let amountApproved: bigint;
if (mode === "revokeApproval") amountApproved = 0n;
else { /* parse approveAmount, support 'unlimited' = 2**256-1 */ }

const data = getErc20ApproveData(spender, amountApproved);

return transactions.map(({ transaction }): Transaction => ({
  ...rest,
  mode: "send" as const,                  // <- intentionally "send" mode in coin-evm
  recipient: tokenContractAddress,
  amount: new BigNumber(0),
  useAllAmount: false,
  data,
}));
```

Two things to note:
- `mode: "send"` (not a special "approve" mode). coin-evm has only `send | erc721 | erc1155` modes; approve/revoke are *send transactions to a token contract with calldata*. The clear-sign decoder is selector-driven, not mode-driven.
- `tokenContractAddress` comes from `getCryptoAssetsStore().findTokenById(tokenId)`.

### 3.3. Cryptoassets / token registry

- Resolver: `getCryptoAssetsStore().findTokenById(tokenId)` returns `{ contractAddress, ticker, units, parentCurrency, … }`.
- Token id format: `ethereum/erc20/usd_tether__erc20_` for USDT mainnet, `ethereum/erc20/usd__coin` for USDC, etc.
- The synced account's `subAccounts` array also exposes tokens by ticker — `parseAmountWithTicker` already does ticker → tokenId resolution against the synced account (`apps/wallet-cli/src/wallet/intents/parse-amount.ts`).

For revoke, we have two clean input shapes:
- **`--token <ticker>`** (e.g. `USDT`) — resolves against the account's existing token sub-accounts. Most user-friendly.
- **`--token <token-id>`** (e.g. `ethereum/erc20/usd_tether__erc20_`) — explicit, unambiguous. Useful when the token has zero balance and isn't in `subAccounts` yet.

Recommendation: **accept both**. If the input contains `/`, treat as token id; otherwise treat as ticker and look up via the synced account.

---

## 4. Three implementation paths compared

| | Path A: dedicated `revoke` | Path B: generalize `send` with raw `--calldata` | Path C: `revoke` thin wrapper over `send` |
|--|--|--|--|
| **User UX** | `wallet-cli revoke <acct> --token USDT --spender 0x…` | `wallet-cli send <acct> --to 0x<token> --amount '0 ETH' --data 0x095ea7b3…` | Same as A |
| **Discoverability in `--help`** | First-class command, mentioned in Part 5 of the guide | Buried under `send`, requires a long footnote | First-class |
| **User error potential** | Low | High (hand-encoding 68 bytes of hex) | Low |
| **Composability for E2E hooks** | `before(() => exec("wallet-cli revoke …"))` reads naturally | Hooks would assemble hex strings — fragile | Same as A |
| **Work in wallet-cli** | New command + token resolver + calldata helper | **Already done** (the `--data` flag exists) | New thin command + shared calldata helper |
| **Work in live-common** | None (helpers exist) | None | None |
| **Test surface** | New `revoke.test.ts`, mock-DMK already supports it | A `send.test.ts` case with `--data` | New `revoke.test.ts`, mock-DMK already supports it |
| **Estimated LOC** | ~150 new + ~80 tests | 0 new (just add an example to docs); ~30 tests | ~120 new + ~80 tests |
| **Files touched** | `commands/revoke.ts`, `cli.ts`, intent schema (optional), shared erc20 helper, test | 0 (wallet-cli already shipped) | `commands/revoke.ts`, `cli.ts`, shared erc20 helper, test |
| **Future-proofing for `approve`** | Add a sibling `commands/approve.ts` (~70 LOC) | Same hex-encoding burden | Add a sibling `commands/approve.ts` reusing the same helper |
| **Risk of regressions in `send`** | Zero | Zero (already supported) | Zero |
| **Pedagogical value (guide)** | High — lets the CLI part show "how to add a fixture command" | Low — it's a one-line example | High — shows the right factoring (composition over a fat command) |

### 4.1. Path A in detail

A complete new command. Pros: cleanest UX. Cons: duplicates the prepare/sign/broadcast plumbing already in `send.ts` if implemented naively.

### 4.2. Path B in detail

Already shipped. Today this works:

```bash
wallet-cli send ethereum-1 \
  --to 0xdAC17F958D2ee523a2206206994597C13D831ec7 \
  --amount '0 ETH' \
  --data 0x095ea7b300000000000000000000000058df81bababd293aaca0a3a76d70d3b69b25c1cf0000000000000000000000000000000000000000000000000000000000000000
```

This will produce a valid revoke transaction. But:
- The user has to know the USDT contract address.
- The user has to know to pad `amount=0` in the calldata while passing `'0 ETH'` for the native value.
- The clear-sign screen will still show "Type: Approve, Amount: 0 USDT, Address: <spender>" because the device decodes by selector — but the *test hook* has no idea what just happened. `grep` patterns for test logs become coupled to hex.
- Forgetting `0x` or the trailing 32 bytes silently produces a different transaction — easy footgun.

This is fine for **interactive debugging**, terrible for **E2E test hooks**. Victor's Slack message specifically frames this as a hook step ("ajouter un hook before/after le test"), so the surface needs to be safe for unattended use.

### 4.3. Path C in detail (recommended)

`commands/revoke.ts`:
- Surface: `wallet-cli revoke [<account>] --token <ticker|tokenId> --spender 0x…`
- Internally: resolve token → tokenContractAddress; build calldata `getErc20ApproveData(spender, 0n)`; reuse the **same** `WalletAdapter.send` path as `send.ts` by constructing an EVM `TransactionIntent` with `recipient = tokenContractAddress`, `amount = "0 <ticker>"` (or `"0 ETH"` — see open question Q3), `data = "0x" + buf.toString("hex")`.

The command's handler is a near-clone of `send.ts`'s handler, minus the family switch (we know it's EVM) and minus the amount parsing (we hardcode 0).

This means **zero changes** to `WalletAdapter`, `BridgeAdapter`, or the EVM intent schema. The only new module is the command file plus a tiny `wallet/erc20-calldata.ts` that re-exports `getErc20ApproveData` from coin-evm (or rebuilds it locally with viem to avoid the deep import — see open question Q1).

Adding `commands/approve.ts` later is then a 30-line follow-up that takes `--amount` (or `unlimited`) and calls the same helper with a non-zero amount. Symmetry is free.

---

## 5. Recommendation, rationale, LOC, files to touch

### 5.1. Recommendation

**Path C.** Ship `revoke` as a dedicated command that internally produces an EVM `TransactionIntent` with `data` = ERC-20 approve calldata at amount 0, then runs the same prepare→sign→broadcast pipeline as `send`.

### 5.2. Rationale (Socratic / Rodin pass)

- ✓ Justified — Path B is already implemented but its UX is unfit for unattended hooks (long hex strings, no semantic logging, easy footgun). The hook context is the load-bearing constraint.
- ✓ Justified — Path C reuses 100 % of the signing/broadcast plumbing. The new code surface is a *command shell*, not a new wallet path.
- ~ Contestable — Path A is technically equivalent to Path C; the distinction is whether we factor out a shared `runEvmContractCall(intent)` helper. Worth doing only when `approve` lands. Defer until then.
- ✗ Unjustified for QAA-615 — adding a new EVM transaction `mode` ("approve" | "revoke") to either coin-evm or the wallet-cli intent schema. coin-evm's clear-sign already keys off the calldata selector, and the bridge already routes `data` correctly. Adding a discriminator is *premature uniformity* with no clear-sign or UX gain. Skip.

### 5.3. LOC estimate

| File | Status | LOC |
|--|--|--|
| `apps/wallet-cli/src/commands/revoke.ts` | new | ~70 |
| `apps/wallet-cli/src/wallet/erc20-calldata.ts` | new | ~30 |
| `apps/wallet-cli/src/cli.ts` | edit (register command) | +2 |
| `apps/wallet-cli/src/output.ts` | edit (revokeEvent / revokeComplete) | +20 |
| `apps/wallet-cli/src/wallet/formatter/human.ts` | edit (revoke variant of send formatter) | +15 |
| `apps/wallet-cli/src/wallet/formatter/json.ts` | edit (revoke envelope) | +10 |
| `apps/wallet-cli/src/test/commands/revoke.test.ts` | new | ~80 |
| `apps/wallet-cli/src/test/helpers/eth-sync-routes.ts` | possibly edit (token sync routes if not already covered) | ±0 |

Total: **~225 LOC** including tests. **~150 LOC** of production code.

Optional follow-up (out of QAA-615 scope but free given the helper):
- `commands/approve.ts`: ~70 LOC.
- Refactor: extract a tiny `runEvmContractCall(descriptor, intent)` shared between `send`, `approve`, `revoke`. Recommended *after* `approve` lands so the abstraction is informed by 3 callers.

### 5.4. Files to touch (bullet list for the PR description)

- `apps/wallet-cli/src/commands/revoke.ts` (new)
- `apps/wallet-cli/src/wallet/erc20-calldata.ts` (new — thin wrapper around `getErc20ApproveData` or re-implemented with viem; see Q1)
- `apps/wallet-cli/src/cli.ts` (register `revoke`)
- `apps/wallet-cli/src/output.ts` (add `revokeEvent`, `revokeComplete`, `revokeDryRun` outputs — mirror `sendEvent`/`sendComplete`)
- `apps/wallet-cli/src/wallet/formatter/{human,json}.ts` (revoke envelope: `{ command: "revoke", network, account, token, spender, amount: "0 USDT", recipient: <token-contract>, fee, txHash }`)
- `apps/wallet-cli/src/test/commands/revoke.test.ts` (new)
- (Optional) `apps/wallet-cli/README.md` (document the command)

### 5.5. Sketched command file

```ts
// apps/wallet-cli/src/commands/revoke.ts
import { defineCommand, option } from "@bunli/core";
import { z } from "zod";
import { lastValueFrom } from "rxjs";
import { tap } from "rxjs/operators";
import { getCryptoCurrencyById } from "@ledgerhq/live-common/currencies/index";
import { getCryptoAssetsStore } from "@ledgerhq/cryptoassets/state";
import { WalletAdapter } from "../wallet";
import { TransactionIntentSchema } from "../wallet/intents";
import { WALLET_CLI_DMK_DEVICE_ID } from "../device/register-dmk-transport";
import { withCurrencyDeviceSession } from "../session/bridge-device-session";
import { networkStringFromCurrencyId } from "../shared/accountDescriptor";
import { colors } from "../shared/ui";
import { createCommandOutput } from "../output";
import { accountOption, outputOption, resolveAccountArg, resolveAccountDescriptor } from "./inputs";
import { buildErc20ApproveCalldata } from "../wallet/erc20-calldata";

export default defineCommand({
  name: "revoke",
  description: "Revoke an ERC-20 token approval (signs approve(spender, 0))",
  options: {
    account: accountOption,
    token: option(z.string().min(1, "Token ticker or token id is required (--token USDT or --token ethereum/erc20/usd_tether__erc20_)"), {
      description: "Token ticker (e.g. USDT) or full token id (e.g. ethereum/erc20/usd_tether__erc20_)",
    }),
    spender: option(z.string().regex(/^0x[0-9a-fA-F]{40}$/, "Spender must be a 0x-prefixed 20-byte hex address"), {
      description: "Address of the spender whose allowance should be revoked",
      short: "s",
    }),
    "dry-run": option(z.boolean().default(false), {
      description: "Prepare and validate the revoke transaction but do not sign or broadcast",
      argumentKind: "flag",
    }),
    output: outputOption,
  },
  handler: async ({ flags, positional }) => {
    const ctx = { command: "revoke", network: "", account: "" };
    const out = createCommandOutput(flags.output, ctx);
    const wallet = new WalletAdapter();
    const dryRun = flags["dry-run"];

    await out.run(async () => {
      const descriptor = await resolveAccountDescriptor(resolveAccountArg(flags.account, positional));
      ctx.network = networkStringFromCurrencyId(descriptor.currencyId);
      ctx.account = descriptor.id;

      const { family, id: parentId } = getCryptoCurrencyById(descriptor.currencyId);
      if (family !== "evm") throw new Error(`revoke is only supported for EVM accounts; got family "${family}"`);

      // Resolve token: accept either ticker (lookup against subAccounts in the bridge) or token id.
      const tokenInput = flags.token.includes("/") ? flags.token : null;
      const tokenCurrency = tokenInput
        ? await getCryptoAssetsStore().findTokenById(tokenInput)
        : null;
      // If only a ticker was provided, defer token resolution to bridge.prepareSend, which already
      // walks subAccounts via parseAmountWithTicker. We pass amount as "0 <TICKER>" in that case.
      const amountStr = tokenCurrency
        ? `0 ${tokenCurrency.ticker}`
        : `0 ${flags.token}`;
      const recipient = tokenCurrency?.contractAddress ?? "RESOLVE_FROM_BRIDGE"; // see Q3

      const calldata = buildErc20ApproveCalldata(flags.spender, 0n); // returns "0x..." string

      const intent = TransactionIntentSchema.parse({
        family: "evm",
        recipient,
        amount: amountStr,
        data: calldata,
      });

      if (dryRun) {
        const spin = out.spin("Preparing revoke (dry run)…");
        const prepared = await wallet.prepareSend(descriptor, intent);
        spin?.clear();
        out.sendDryRun({ ...prepared, command: "revoke", token: flags.token, spender: flags.spender });
        spin?.success("Dry run complete (transaction not broadcasted)");
        return;
      }

      const spin = out.spin(`Connect device and open ${colors.bold(descriptor.currencyId)} app…`);
      await withCurrencyDeviceSession(descriptor.currencyId, async () => {
        spin?.success("Device session established");
        out.spin(`Preparing ${colors.bold(descriptor.currencyId)} revoke transaction…`);
        await lastValueFrom(
          wallet
            .send(descriptor, intent, WALLET_CLI_DMK_DEVICE_ID, false)
            .pipe(tap(event => out.sendEvent(event))),
        );
        out.sendComplete();
      });
    });
  },
});
```

Note the `RESOLVE_FROM_BRIDGE` placeholder — see open question Q3 about cleanly resolving the token contract address when only a ticker is provided. Two clean options:
- (a) require token id, drop ticker support → lose convenience.
- (b) sync the account first (call `BridgeAdapter.sync`) and look up the token sub-account's `token.contractAddress`. Costs one extra sync but mirrors what `parseAmountWithTicker` already does.

(b) is the right call. We'd add a small `WalletAdapter.resolveErc20Token(descriptor, tickerOrId): { contractAddress, ticker, tokenId }` method.

---

## 6. Clear-sign concerns

The good news: **for any token in the cryptoassets registry, an ERC-20 `approve` transaction is clear-signed today.**

`libs/coin-modules/coin-evm/src/deviceTransactionConfig.ts:71-105` (the `inferDeviceTransactionConfigWalletApi` function) checks the calldata selector against `ERC20_CLEAR_SIGNED_SELECTORS.APPROVE = 0x095ea7b3`. If the recipient is a known ERC-20 token (resolved via `findTokenByAddressInCurrency`), the device shows:

- "Type: Approve"
- "Amount: <TICKER> 0" (or "Unlimited <TICKER>" for `2^256-1`)
- "Address: <spender>"

The 0-amount approve renders as "USDT 0.0" (formatted via `formatCurrencyUnit`). The user reads "I'm authorising spender 0x… to spend 0 USDT", clicks accept, on-chain allowance becomes 0. Perfect clear-sign.

**Failure modes to be aware of:**
- If the token is NOT in the cryptoassets registry, `inferDeviceTransactionConfigWalletApi` throws "Fallback on Blind Signing" (line 368) and the generic blind-sign screen shows ("Data: Present, Amount: 0, Address: <token contract>"). For mainstream tokens (USDT, USDC, DAI, WETH, USDC.e, …) this is a non-issue.
- The device needs the ERC-20 plugin (or modern clear-sign descriptors) loaded in firmware. On Speculos this is provided by the Ethereum app build used in CI; verify with the swap-live-app folks if uncertain.
- If the test wants to assert the screen content, the cli-runner's mock-DMK doesn't drive the on-device UI — Speculos does. wallet-cli unit tests will see "device-signature-requested" / "device-signature-granted" events but not the exact label text. End-to-end tests in QAA-613 (UI tests) will see the screen.

**Speculos / CI:**
- The mock-DMK in `apps/wallet-cli/src/device/mock-dmk.ts` has no view of the device UI; it just emits Completed states. CI tests for the CLI itself can run fully mocked (no Speculos).
- The QAA-613 UI test that *uses* `wallet-cli revoke` as a hook will still spin up Speculos and verify the approve screen there. So the spike doesn't need to touch Speculos at all — the existing CI machinery already handles it.

Bottom line: **the spike does not need a clear-sign workaround**. The infra already produces a clear-signed approve for known tokens.

---

## 7. Test plan

### 7.1. Mock-DMK already supports the revoke flow

`apps/wallet-cli/src/device/mock-dmk.ts` mocks `executeDeviceAction` at the `Completed` state level. The signing call (whether for `transfer`, `approve`, or any other ERC-20 method) is dispatched the same way: `CallTaskInAppDeviceAction` with `appName: "Ethereum"` and a `task` function. The mock returns whatever `MockAppResults["Ethereum"]` you configure.

For a revoke unit test, the `appResults` shape is the same as for `send`:

```ts
WALLET_CLI_MOCK_APP_RESULTS: JSON.stringify({
  Ethereum: { publicKey: MOCK_ETH_PUBKEY, address: MOCK_ETH_ADDRESS },
})
```

There is **nothing approve-specific** to add to the mock. The mock simply doesn't care what the calldata is — that distinction is enforced by the live-common bridge against the HTTP mock server (gas estimation, nonce, balance) and by the device's clear-sign decoder (which the mock bypasses).

### 7.2. HTTP mock-server: which routes the revoke flow needs

Same routes as `send.test.ts` (see `test/commands/send.test.ts` and `test/helpers/eth-sync-routes.ts`):

- `GET /address/<addr>/balance` — needed; revoke's gas costs require a non-zero ETH balance.
- `GET /address/<addr>/nonce` — needed; produces the tx nonce.
- `POST /tx/estimate-gas-limit` — needed; a typical approve costs ~46k gas, but the mock just returns `{ "estimated_gas_limit": "60000" }` to be safe.
- `GET /gastracker/barometer` — needed; provides the EIP-1559 fee parameters.
- ETH sync routes (`ETH_SYNC_ROUTES`) — needed for `BridgeAdapter.sync`.

**One additional route may be needed** if the spike picks the ticker-resolution path (Q3 option b): the sync route that returns the token sub-account / token balance for the synced account. In `eth-sync-routes.ts` there should already be a token list/balance fixture; if not, a single route returning a USDT sub-account with arbitrary balance is enough — the test only needs the token to be *known* to the account, not to have any actual allowance on chain.

### 7.3. Test cases

```ts
// apps/wallet-cli/src/test/commands/revoke.test.ts
describe("revoke --dry-run", () => {
  it("returns a dry-run revoke envelope with the token contract as recipient and 0 amount", async () => {
    const { stdout, exitCode } = await runCli(
      ["revoke", "--account", MOCK_ETH_DESCRIPTOR, "--token", "USDT", "--spender", SPENDER, "--dry-run", "--output", "json"],
      { /* same env vars as send.test.ts */ },
    );
    expect(exitCode).toBe(0);
    const data = JSON.parse(stdout);
    expect(data.command).toBe("revoke");
    expect(data.network).toBe("ethereum:main");
    expect(data.token).toBe("USDT");
    expect(data.spender).toBe(SPENDER);
    expect(data.recipient).toBe(USDT_MAINNET_CONTRACT); // 0xdAC1…
    expect(data.amount).toMatch(/^0(\.0+)? USDT$/);
    expect(data.dry_run).toBe(true);
  });

  it("rejects when the token isn't in the account's chain", async () => {
    const { exitCode, stderr } = await runCli(
      ["revoke", "--account", MOCK_ETH_DESCRIPTOR, "--token", "ethereum/erc20/some_unknown_token", "--spender", SPENDER, "--dry-run"],
      { /* … */ },
    );
    expect(exitCode).not.toBe(0);
    expect(stderr).toMatch(/not found|unknown token/i);
  });

  it("rejects malformed spender addresses", async () => {
    const { exitCode, stderr } = await runCli(
      ["revoke", "--account", MOCK_ETH_DESCRIPTOR, "--token", "USDT", "--spender", "not-an-address", "--dry-run"],
      { /* … */ },
    );
    expect(exitCode).not.toBe(0);
    expect(stderr).toMatch(/spender|address/i);
  });
});

describe("revoke (signed, mocked DMK)", () => {
  it("emits device-signature-* events and a broadcasted txHash", async () => {
    const { stdout, exitCode } = await runCli(
      ["revoke", "--account", MOCK_ETH_DESCRIPTOR, "--token", "USDT", "--spender", SPENDER, "--output", "json"],
      { /* mock app results + mock broadcast route returning a fake hash */ },
    );
    expect(exitCode).toBe(0);
    const data = JSON.parse(stdout);
    expect(data.txHash).toMatch(/^0x[0-9a-f]{64}$/);
  });
});
```

### 7.4. Integration test in QAA-613 UI flows

This is **not** part of QAA-615. Just for context: the QAA-613 nightly tests will, in their `beforeEach`/`afterEach`, run something like:

```bash
wallet-cli revoke ethereum-1 --token USDC --spender $THORCHAIN_ROUTER --output json | jq -r '.txHash'
```

Wait for confirmation, then run the actual swap flow. Since QAA-613 is broadcast-disabled, revoke is a no-op there; but for the broadcast-enabled regression suite (which is what QAA-615 explicitly targets), this is the cleanup hook.

---

## 8. Open questions for the team

**Q1. ERC-20 ABI helper in wallet-cli — re-export from coin-evm or duplicate with viem?**

Two options:
- (a) `import { getErc20ApproveData } from "@ledgerhq/coin-evm/logic/getErc20Data"` — direct deep import. Pro: zero duplication, identical bytes. Con: deep imports past package barriers are typically discouraged in this monorepo.
- (b) Re-implement in `wallet-cli/src/wallet/erc20-calldata.ts` using `viem` (`encodeFunctionData({ abi: erc20Abi, functionName: "approve", args: [spender, 0n] })`). 5 lines. Pro: clean dep boundary, no deep import. Con: tiny duplication.

Recommendation: **(b) with viem**. wallet-cli already runs on Bun and viem is a small dep; the legacy CLI uses ethers but the live-common helper internally uses ethers too — wallet-cli is a fresh codebase and viem is the modern choice. If the team prefers, ethers `Interface` works equally well.

**Q2. Should `revoke` accept an `--amount` flag (defaulting to 0) and become an alias for `approve --amount 0`?**

Pro: one command, two behaviors. Con: violates the "one command does one thing" principle and complicates the help text. Recommendation: keep `revoke` as a zero-arg-amount command. Add a separate `approve --token … --spender … --amount …` command later (out of QAA-615 scope but mentioned in the spike for completeness).

**Q3. Ticker → contract address resolution: sync-first or token-id-only?**

If the user passes `--token USDT`, we need the contract address. Options:
- (a) Require a full token id (`ethereum/erc20/usd_tether__erc20_`). Clean, no sync. UX cost: users must memorize / look up token ids.
- (b) Accept ticker, sync the account, look up `subAccounts[].token.contractAddress`. Mirrors `parseAmountWithTicker`. Costs one sync per revoke (~1–2 s). UX win.
- (c) Accept ticker, scan the cryptoassets store directly by `(parentCurrency.id, ticker)`. Fastest, but ticker collisions across chains are a real risk (USDC on Ethereum vs USDC.e on Avalanche).

Recommendation: **(b)**. The bridge already has to sync to compute gas correctly; one sync covers both. (a) is acceptable as a fallback when the ticker isn't found in the synced account (e.g. revoking a token that has zero balance and is therefore not in `subAccounts`).

**Q4. Output envelope: separate `revoke` / share `send`?**

The current `output.ts` exposes `sendEvent`, `sendComplete`, `sendDryRun`. Mirror them with `revokeEvent`, `revokeComplete`, `revokeDryRun`? Or keep `sendEvent` and just stamp `command: "revoke"` in the envelope?

Recommendation: **stamp `command: "revoke"` in the same `send*` envelopes**, no new functions. The events themselves (`device-signature-requested`, `broadcasted`, `txHash`, …) are identical. Add a `command` field at the top level (already present in `ctx.command`) and keep the envelope shape.

**Q5. Do we need to gate `revoke` by network?**

Today, EVM family in wallet-cli supports `ethereum` and `base` (per the README). Revoke should work on both — and on any future EVM network — with no per-network code. The clear-sign decoder is identical across EVM chains. No gating needed beyond `family === "evm"`.

**Q6. Should the spike PR include `commands/approve.ts` as well?**

QAA-615 is a spike; the deliverable is "find a way to revoke approval". `approve` is not strictly in scope (test hooks need only revoke). However, given that the helper is identical and the cost is ~70 LOC, it could ride along.

Recommendation: **ship only `revoke` in the QAA-615 PR**. Add `approve` as a follow-up (separate PR, separate ticket) once the patterns are reviewed. Avoids review-scope creep.

**Q7. Idempotency — what does revoke return when allowance is already 0?**

Two behaviors possible:
- (a) Always send the transaction. Cost: gas spent for a no-op.
- (b) Pre-check via `getEvmTokenAllowance` (already exists in `libs/ledger-live-common/src/families/evm/getTokenAllowance.ts`) and skip + exit 0 with `{ skipped: true, reason: "allowance-already-zero" }` if the current allowance is 0.

Recommendation: **(a) for the QAA-615 PoC**, defer (b) to a `--skip-if-zero` flag. Test hooks running before each test will benefit from (b) eventually; for now, the cleanest semantics are "revoke always emits the tx, you decide upstream when to call it".

---

## 9. Appendix — load-bearing file references

- `apps/wallet-cli/src/commands/send.ts:30-55` — `INTENT_BUILDERS.evm` already accepts `data`.
- `apps/wallet-cli/src/wallet/intents/families/evm.ts` — EVM intent schema with `data` field.
- `apps/wallet-cli/src/wallet/compatibility/bridge.ts:194-198` — converts hex `data` to Buffer, patches the live-common Transaction.
- `apps/wallet-cli/src/wallet/compatibility/bridge.ts:151-184` — `signOperation` → `broadcast` plumbing (free reuse for revoke).
- `libs/coin-modules/coin-evm/src/logic/getErc20Data.ts` — `getErc20ApproveData(spender, amount)` lives here.
- `libs/coin-modules/coin-evm/src/cli-transaction.ts:95-130` — legacy CLI's `revokeApproval` reference implementation (one tier above what wallet-cli needs).
- `libs/coin-modules/coin-evm/src/deviceTransactionConfig.ts:71-105` — clear-sign decoder for ERC-20 `APPROVE`.
- `libs/evm-tools/src/selectors/index.ts:24` — `ERC20_CLEAR_SIGNED_SELECTORS.APPROVE = "0x095ea7b3"`.
- `libs/ledger-live-common/src/families/evm/getTokenAllowance.ts` — pre-flight allowance check (for Q7 idempotency follow-up).
- `apps/wallet-cli/src/test/helpers/dmk-intercept.ts` and `apps/wallet-cli/src/device/mock-dmk.ts` — mock-DMK already covers any EVM signing call; no approve-specific changes needed.
- `apps/wallet-cli/src/test/commands/send.test.ts` — template for `revoke.test.ts`.

---

## 10. TL;DR for the QAA-615 PR description

> **Adds a `wallet-cli revoke` command that signs and broadcasts an ERC-20 `approve(spender, 0)` transaction. Used by the QAA-613 broadcast-enabled regression suite as a before/after hook to keep token allowances at zero between runs.**
>
> Implementation: thin wrapper that delegates to the existing `WalletAdapter.send` pipeline. The ERC-20 calldata is built locally (viem `encodeFunctionData`); the token contract address is resolved from the synced account's sub-accounts (ticker) or the cryptoassets store (token id). The device clear-signs the transaction natively via the existing ERC-20 selector decoder — no plugin work required.
>
> Files: 1 new command, 1 new tiny helper, 1 new test file, ~225 LOC total.
