# R3 — ERC-20 approve / revoke helpers in Ledger Live monorepo

**Spike**: QAA-615 — `wallet-cli revoke` command.
**Question**: Is the wheel already invented somewhere?
**Short answer**: **YES, multiple times.** A complete approve+revoke pipeline already
exists end-to-end in the legacy stack (`coin-evm` builder + `apps/cli send --mode
revokeApproval` + `apps/cli tokenAllowance`). The only thing not yet wired is the
`wallet-cli` (Bunli) command surface — but every primitive needed is one import
away (`@ledgerhq/coin-evm`, `@ledgerhq/live-common`).

Repo scanned: `/Users/jerome.portier/src/tries/2026-04-08-LedgerHQ-ledger-live/`

---

## 1. Helper inventory

| Package | File | Symbol | Signature | Purpose | Reusable from `wallet-cli`? |
|---|---|---|---|---|---|
| `@ledgerhq/coin-evm` | `libs/coin-modules/coin-evm/src/logic/getErc20Data.ts:10-14` | `getErc20ApproveData` | `(spender: string, amount: bigint) => Buffer` | Encodes `approve(address,uint256)` calldata via `ethers.Interface`. | **Yes, directly.** |
| `@ledgerhq/coin-evm` | `libs/coin-modules/coin-evm/src/logic/getErc20Data.ts:4-8` | `getErc20Data` | `(recipient: string, amount: bigint) => Buffer` | Encodes `transfer(address,uint256)` calldata. | Yes (sibling helper, useful as template). |
| `@ledgerhq/coin-evm` | `libs/coin-modules/coin-evm/src/cli-transaction.ts:159-169` | `makeCliTools()` (default export) | `() => { options, inferAccounts, inferTransactions }` | Full CLI mode handler implementing `mode: "send" \| "revokeApproval" \| "approve"`. Wires `--token`, `--spender`, `--approveAmount` (incl. `unlimited`) into a valid `Transaction`. | **Yes — copy/adapt directly.** This is the canonical reference implementation. |
| `@ledgerhq/coin-evm` | `libs/coin-modules/coin-evm/src/cli-transaction.ts:37` | `UNLIMITED_APPROVAL_AMOUNT` | `2n ** 256n - 1n` | uint256 max = "unlimited" approval sentinel. | Yes. |
| `@ledgerhq/coin-evm` | `libs/coin-modules/coin-evm/src/abis/erc20.abi.json` | (JSON) | — | Full ERC-20 ABI (transfer, approve, allowance, balanceOf, Approval event…). | Yes (already loaded by helpers above). |
| `@ledgerhq/coin-evm` | `libs/coin-modules/coin-evm/src/network/node/rpc.common.ts:237-250` | `getTokenAllowance` (RPC variant) | `(api, currency, owner, contract, spender) => Promise<BigNumber>` | Reads allowance via `ethers.Contract.allowance()` over JSON-RPC. | Yes (via `getNodeApi(currency).getTokenAllowance`). |
| `@ledgerhq/coin-evm` | `libs/coin-modules/coin-evm/src/network/node/ledger.ts:309-331` | `makeGetTokenAllowance` (Ledger explorer variant) | `(config, fetch) => NodeApi["getTokenAllowance"]` | Reads allowance via Ledger's contract-read endpoint (`/blockchain/v4/.../contract/read`). | Yes (this is what production uses). |
| `@ledgerhq/coin-evm` | `libs/coin-modules/coin-evm/src/network/node/types.ts:172` | `NodeApi.getTokenAllowance` | `(currency, owner, contract, spender) => Promise<BigNumber>` | The unified NodeApi method exposed via `getNodeApi(currency)`. | Yes. |
| `@ledgerhq/live-common` | `libs/ledger-live-common/src/families/evm/getTokenAllowance.ts:29-71` | `getEvmTokenAllowance` | `(account: Account, tokenId: string, spender: string) => Promise<{ allowance: BigNumber; unit; symbol; tokenId; owner; spender; contractAddress }>` | High-level allowance reader: resolves token via cryptoassets store, validates chain match, then calls `getNodeApi(currency).getTokenAllowance`. | **Yes — best ergonomics for verification.** |
| `@ledgerhq/coin-evm` | `libs/coin-modules/coin-evm/src/deviceTransactionConfig.ts:71-106` | `inferDeviceTransactionConfigWalletApi` (approve branch) | (decode helper) | Decodes a device-bound approve tx into UI fields ("Type: Approve", "Amount: …", "Address: …"). Detects `ffffffff…` as "Unlimited". | Yes, but only useful if you need to render a confirm screen. |
| `@ledgerhq/evm-tools` | `libs/evm-tools/src/selectors/index.ts:23-26` | `ERC20_CLEAR_SIGNED_SELECTORS.APPROVE` | `"0x095ea7b3"` | Function selector constant. | Yes. |
| `apps/cli` | `apps/cli/src/commands/blockchain/tokenAllowance.ts:122-213` | default export (`tokenAllowance` command) | `commander`-style `{description, args, job}` | Reference allowance-checker CLI command (legacy live-common CLI). Demonstrates the exact pattern wallet-cli should mirror. | Adapt to Bunli `defineCommand`. |
| `libs/ledger-live-common` | `libs/ledger-live-common/src/e2e/runCli.ts:222-232` | `runCliTokenApproval` | `(opts: TokenApprovalOpts) => Promise<string>` | Test wrapper that invokes the legacy CLI with `send --mode revokeApproval/approve`. Confirms the e2e contract. | Reference only. |

---

## 2. The 3 most reusable helpers (verbatim)

### 2.1 `getErc20ApproveData` — the calldata builder

`libs/coin-modules/coin-evm/src/logic/getErc20Data.ts` (lines 1-14):

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

This is **the** ERC-20 approve calldata builder. Pure function, zero side effects,
no device, no network. Returns the raw 4-byte selector (`0x095ea7b3`) + ABI-encoded
`(spender, amount)` as a Buffer ready to drop into `transaction.data`. For revoke,
call with `amount = 0n`.

### 2.2 `cli-transaction.ts` — the full approve/revoke recipe

`libs/coin-modules/coin-evm/src/cli-transaction.ts` (lines 1-132, key excerpt):

```ts
import { parseCurrencyUnit } from "@ledgerhq/coin-module-framework/currencies";
import { getCryptoAssetsStore } from "@ledgerhq/cryptoassets/state";
import { BigNumber } from "bignumber.js";
import { getErc20ApproveData } from "./logic/getErc20Data";
import { Transaction } from "./types";

const modes = ["send", "revokeApproval", "approve"] as const;
const UNLIMITED_APPROVAL_AMOUNT = 2n ** 256n - 1n;

// inside inferTransactions (mode === "revokeApproval" | "approve"):
const tokenCurrency = await getCryptoAssetsStore().findTokenById(tokenId);
// validate: tokenCurrency.parentCurrency.id === mainAccount.currency.id
const tokenContractAddress = tokenCurrency.contractAddress;

let amountApproved: bigint;
if (mode === "revokeApproval") {
  amountApproved = 0n;
} else {
  const trimmed = approveAmountStr.trim().toLowerCase();
  amountApproved = trimmed === "unlimited"
    ? UNLIMITED_APPROVAL_AMOUNT
    : BigInt(parseCurrencyUnit(tokenCurrency.units[0], trimmed).toFixed(0));
}

const data = getErc20ApproveData(spender, amountApproved);

return transactions.map(({ transaction }): Transaction => {
  const { nft: _nft, ...rest } = transaction;
  return {
    ...rest,
    mode: "send" as const,            // EVM "send" mode + custom data == arbitrary contract call
    recipient: tokenContractAddress,  // tx.to = the token contract, NOT the spender
    amount: new BigNumber(0),         // no native value
    useAllAmount: false,
    data,                             // ERC-20 approve(spender, amount) calldata
  };
});
```

**Three load-bearing facts** any new revoke implementation must respect:

1. `tx.to` = **token contract address** (not the spender, not the owner).
2. `tx.value` = **0** (revoke moves no native ETH).
3. `tx.data` = **`getErc20ApproveData(spender, 0n)`**.
4. The high-level `mode` stays `"send"` because the EVM family treats this as a
   plain "send with calldata" — the `revokeApproval`/`approve` mode names are
   purely a CLI affordance and are flattened to `"send"` before signing.

### 2.3 `getEvmTokenAllowance` — the verification helper

`libs/ledger-live-common/src/families/evm/getTokenAllowance.ts` (lines 29-71):

```ts
export async function getEvmTokenAllowance(
  account: Account,
  tokenId: string,
  spender: string,
): Promise<GetEvmTokenAllowanceResult> {
  if (account.currency.family !== "evm") {
    throw new Error(`The account currency must be EVM …`);
  }

  const tokenCurrency = await getCryptoAssetsStore().findTokenById(tokenId);
  if (!tokenCurrency) throw new Error(`No token found for id "${tokenId}".`);
  if (tokenCurrency.parentCurrency.id !== account.currency.id) {
    throw new Error(`Token "${tokenId}" is on chain "…", but the account is on "…"`);
  }

  // Ensure EVM coin config is set (e.g. when CLI runs tokenAllowance without bridge)
  createApi(getCurrencyConfiguration<EvmConfigInfo>(account.currency.id), account.currency.id);

  const nodeApi = getNodeApi(account.currency);
  const allowance = await nodeApi.getTokenAllowance(
    account.currency,
    account.freshAddress,
    tokenCurrency.contractAddress,
    spender,
  );

  return {
    allowance,
    unit: tokenCurrency.units[0],
    symbol: tokenCurrency.ticker,
    tokenId,
    owner: account.freshAddress,
    spender,
    contractAddress: tokenCurrency.contractAddress,
  };
}
```

This is the **post-revoke verification helper**: call it before and after the
revoke transaction confirms; if the second call returns `0`, the revoke worked.

---

## 3. Approve UI in LWD / LWM

**Short answer**: **There is no first-party approve flow in the apps.** Approves
happen exclusively through Live Apps (Swap, dApp browser, Earn) via the wallet
API; the apps themselves only render the *device confirmation* of an incoming
approve transaction.

Findings:

- **LWD**: only one approve-aware UI surface exists —
  `apps/ledger-live-desktop/src/renderer/components/TransactionConfirm/ConfirmTitle.tsx:18-22`:
  ```ts
  if (typeTransaction === "Approve") return t("approve.description");
  ```
  This is just the modal *title* shown when the underlying tx data decodes as
  approve. The decoding itself happens in
  `libs/coin-modules/coin-evm/src/deviceTransactionConfig.ts:71-106` (selector
  match on `ERC20_CLEAR_SIGNED_SELECTORS.APPROVE`).
- **LWM**: no approve UI files. Approve strings only exist in
  `apps/ledger-live-mobile/src/locales/*/common.json` as legacy lending-feature
  i18n keys (e.g. `lending.approve`, `compound.approve`) — not wired to a live
  flow.
- **Swap2 / swapWeb screens**: scanned all of
  `apps/ledger-live-desktop/src/renderer/screens/exchange/Swap2/` and
  `apps/.../swapWeb/` — **zero matches for `approve` / `allowance`**. The
  modern swap is a Live App webview; the approve step is constructed by the
  Live App itself and signed via `wallet.transaction.signAndBroadcast`.
- **Wallet API path**: a Live App that needs to approve sends an EVM
  `Transaction` with `data` = the approve calldata to
  `wallet.transaction.signAndBroadcast`. Routing lives in
  `libs/ledger-live-common/src/wallet-api/Exchange/server.ts` and
  `libs/ledger-live-common/src/wallet-api/react.ts`. Live App is responsible
  for building the calldata; LWD/LWM just render the confirm screen using
  `inferDeviceTransactionConfigWalletApi`.

**Implication for QAA-615**: there is no UI helper to copy. The only
end-user-facing "build an approve from scratch" affordance in the entire
monorepo is **the legacy `apps/cli send --mode approve|revokeApproval`
command** (cli-transaction.ts above).

---

## 4. Token-allowance reading helpers (for verifying revoke)

Three layers, choose the highest one available:

| Layer | Symbol | When to use |
|---|---|---|
| **Highest** | `getEvmTokenAllowance(account, tokenId, spender)` from `@ledgerhq/live-common/families/evm/getTokenAllowance` | If you have an `Account` object (sync'd or built from a freshAddress). Returns formatted result with unit & symbol. |
| Mid | `getNodeApi(currency).getTokenAllowance(currency, owner, contract, spender)` | If you have a `CryptoCurrency` and raw addresses. Returns `BigNumber`. |
| Low | direct ABI call: `new ethers.Contract(contract, ERC20Abi, provider).allowance(owner, spender)` | Only if you bypass the NodeApi (not recommended — loses Ledger explorer routing). |

Reference CLI usage: `apps/cli/src/commands/blockchain/tokenAllowance.ts:122-213`
(default export). Note the helper `buildMinimalEvmAccountFromOwnerAddress`
(lines 31-82) which lets you read allowance for *any* address without
requiring a synced account — useful for read-only checks.

---

## 5. Web3 lib in use

**`ethers` v6.** Single, consistent across the repo.

- `libs/coin-modules/coin-evm/package.json` → `"ethers": "catalog:"`
- `libs/ledger-live-common/package.json` → `"ethers": "catalog:"`
- `apps/wallet-cli/package.json` → no direct ethers dep, but transitively via `@ledgerhq/coin-evm` (workspace).

The "catalog:" alias means version is centralized in `pnpm-workspace.yaml`
catalog. Across `coin-evm` source, `ethers.Interface`, `ethers.Contract`,
`ethers.AbiCoder.defaultAbiCoder()`, `ethers.getAddress()` are used — pure v6
APIs.

**No viem, no web3.js anywhere in the EVM family.** `wallet-cli` already
inherits ethers transitively from `@ledgerhq/coin-evm` workspace dep — no new
package needed.

---

## 6. Clear-sign descriptor for `approve()`

**It already exists and is well-supported.** There's nothing to add.

- `libs/evm-tools/src/selectors/index.ts:23-26` declares
  `ERC20_CLEAR_SIGNED_SELECTORS.APPROVE = "0x095ea7b3"`.
- `libs/coin-modules/coin-evm/src/deviceTransactionConfig.ts:40-106` consumes
  the selector: when a tx's data starts with `0x095ea7b3` and the recipient
  resolves to a known token, it decodes `(spender, value)` and emits device
  fields:
  - `Type: Approve`
  - `Amount: <ticker> <value>` — or `Unlimited <ticker>` when value ==
    `0xffff…ffff` (uint256 max).
  - `Address: <spender>` (or `Domain: …` if a forward-resolved ENS is
    available).
- For Ethereum mainnet specifically, the device's Ethereum app already has
  built-in clear-signing for the standard ERC-20 selectors (no per-token
  descriptor needed for transfer/approve). No new descriptor file is required
  for QAA-615.
- The "Approval" Approval event ABI is in `erc20.abi.json` lines 178-199 if
  needed for log parsing.

For **NFT** approves (ERC-721 `approve` and `setApprovalForAll`,
ERC-1155 `setApprovalForAll`) — also fully covered in
`deviceTransactionConfig.ts` lines 109-…, with selectors declared in
`evm-tools/src/selectors/index.ts:28-40`. Out of scope for the QAA-615 ERC-20
spike but worth noting for follow-up.

---

## 7. Verdict for the spike

### Recommendation: **build a thin wallet-cli command on top of `@ledgerhq/coin-evm`** (do *not* copy-paste).

`wallet-cli` already depends on `@ledgerhq/coin-evm` (workspace:^) — the imports
land for free. The exact approach:

1. **New file `apps/wallet-cli/src/commands/revoke.ts`** (Bunli
   `defineCommand`). Mirror the structure of `send.ts:57-158`.
2. **Extend `EvmTransactionIntentSchema`** at
   `apps/wallet-cli/src/wallet/intents/families/evm.ts:4-12` with optional
   `mode: "send" | "revoke" | "approve"`, `spender: string`, `approveAmount: string`.
3. **In the new command's handler**, resolve token via
   `getCryptoAssetsStore().findTokenById(tokenId)`, then call
   `getErc20ApproveData(spender, 0n)` (revoke) or
   `getErc20ApproveData(spender, amount)` (approve) and pass the resulting
   Buffer through the existing `BridgeAdapter.send()` path with
   `recipient = tokenCurrency.contractAddress`, `amount = "0 ETH"`, `data =
   "0x" + buf.toString("hex")`. The existing `bridge.ts:186-208`
   `buildTxExtras` already converts `intent.data` (hex) to `Buffer` — no
   bridge changes needed.
4. **Verification path**: add `apps/wallet-cli/src/commands/allowance.ts` that
   wraps `getEvmTokenAllowance` from `@ledgerhq/live-common/families/evm/getTokenAllowance`.
   Same pattern as legacy `apps/cli/src/commands/blockchain/tokenAllowance.ts`.
5. **Tests**: piggyback on the existing
   `libs/coin-modules/coin-evm/src/logic/getErc20Data.test.ts` style — pure
   unit test on calldata bytes (assert leading `0x095ea7b3` selector and ABI
   layout). The wallet-cli's e2e against testnet can mirror
   `runCliTokenApproval` from `libs/ledger-live-common/src/e2e/runCli.ts:222`.

### Justification

- **Reusing `coin-evm` directly** beats copying because (a) ABI, encoder, and
  edge-cases (unlimited sentinel, address checksum, integer parsing) are
  already battle-tested and shipped to production; (b) the existing
  `cli-transaction.ts:53-132` already encodes the *exact* `Transaction` shape
  the bridge expects (mode flattened to "send", value=0, recipient=contract,
  data=approve-bytes); (c) any future fix in coin-evm (e.g. EIP-2612
  permit support, gas-estimation tweaks) propagates automatically.
- **Building fresh would mean duplicating ABI handling and the
  unlimited-amount sentinel logic** — two places that have historically been
  bug magnets.
- **Copying the legacy `apps/cli` command** is tempting but it's
  `commander`-shaped and lives outside Bunli; you'd lose schema validation
  (zod), the JSON output mode, and the dry-run pattern that wallet-cli's
  `send.ts` already provides. The right move is to mirror `send.ts`'s
  handler shape, with EVM-specific intent fields.
- **`getEvmTokenAllowance` is essential for the verify step** — the spike
  asks "is the wheel invented?" and yes, this is exactly the wheel for the
  *post*-revoke verification. Use it as-is.

### Best reuse target (if forced to pick one)

`libs/coin-modules/coin-evm/src/logic/getErc20Data.ts` →
**`getErc20ApproveData(spender, amount: bigint)`**. Single import, two-line
call site, returns the calldata Buffer. The simplest possible primitive; the
rest of the pipeline (signing, broadcasting, fee estimation) is already
handled by the existing wallet-cli `BridgeAdapter.send()`.
