# Part 5 v2 — ground-up rewrite briefing

**Trigger:** the Part 5 v1 (committed at `40be619`) taught wallet-cli + Bun + Bunli + DMK as the QAA CLI surface. This was wrong. **wallet-cli is out of scope for QAA.** All QAA CLI work uses the legacy `apps/cli` (`@ledgerhq/live-cli`).

**Source of truth:** the senior's branch `support/qaa_add_revoke_token` shows the actual architecture in production today: a 5-layer integration (binary → spawn engine → typed wrappers → device-tap automation → POM methods).

## What v2 must teach

1. The legacy `apps/cli` as the canonical QAA CLI (not wallet-cli)
2. ERC-20 approvals and revokes — the blockchain primer
3. Manual revoke tools — revoke.cash, Etherscan
4. The 5-layer integration in detail (runCli.ts + cliCommandsUtils.ts + families/evm.ts + Speculos macros + POMs)
5. Speculos lifecycle in CLI tests (snapshot/restore, app-per-Speculos)
6. `DISABLE_TRANSACTION_BROADCAST` env semantics
7. Daily CLI workflow against this stack
8. The QAA-615 walkthrough — actual senior's commit, every line explained
9. Exercises + Final Assessment

## Out of scope for v2

- wallet-cli, Bun, Bunli, DMK, V1 descriptors — deleted entirely
- The old Reference Implementation (5.8.18) — gone
- The cliCommandsUtils-as-future-integration framing — replaced with cliCommandsUtils-as-current-reality

## Repo paths

- Guide: `/Users/jerome.portier/qa-onboarding-guide/`
- Monorepo: `/Users/jerome.portier/src/tries/2026-04-08-LedgerHQ-ledger-live/` (currently on `support/qaa_add_revoke_token`)
- The senior's commit: `4fa4868a35` (3 files, 37 lines)

## The senior's commit (verified ground truth)

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
@step("Ensure token approval")  // copy-paste bug; should be "Revoke token approval"
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

This is **dev-loop code, not the final PR shape.** The final PR will likely keep both and order them as a `beforeEach` revoke + the per-test approve.

## The 5 layers (verified)

1. **Binary**: `apps/cli/bin/index.js` (legacy `@ledgerhq/live-cli`, Node, Commander)
2. **Spawn engine**: `libs/ledger-live-common/src/e2e/runCli.ts` — `child_process.spawn("node", [LEDGER_LIVE_CLI_BIN, ...args])`, "+" arg separator, retries
3. **Typed wrappers**: `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts`
4. **Device-tap automation**: `libs/ledger-live-common/src/e2e/families/evm.ts` (`approveToken()` dispatcher → `approveTokenTouchDevices` for Stax/Flex / `approveTokenButtonDevice` for Nano S/X)
5. **POM methods**: `e2e/desktop/tests/page/swap.page.ts` and `e2e/mobile/page/trade/swap.page.ts`

## Speculos lifecycle macros

`e2e/desktop/tests/utils/speculosUtils.ts`:
- `launchSpeculos(appName, testTitle?, previousDevice?)` — stops previous, starts new with the named app, mutates `SPECULOS_API_PORT`
- `cleanSpeculos(speculos, previousPort?)` — stops + unregisters transport + restores port

## Broadcast control

`DISABLE_TRANSACTION_BROADCAST` env — `"1"` blocks broadcast, `"0"` enables it. Default is broadcast disabled. `approveTokenCommand` and `revokeTokenCommand` flip it to `"0"` because allowance state must actually move on-chain. The `try/finally` env-restore is the safety net.

## New chapter outline (10 chapters)

5.1 CLI in QA — apps/cli is canonical, wallet-cli out of scope
5.2 Approvals & revokes blockchain primer
5.3 Manual revoke tools (revoke.cash, Etherscan)
5.4 The 5-layer integration deep dive
5.5 Speculos lifecycle in CLI tests
5.6 DISABLE_TRANSACTION_BROADCAST and broadcast discipline
5.7 Daily CLI workflow
5.8 Walkthrough QAA-615 (the senior's commit, line-by-line)
5.9 Exercises
Part 5 Final Assessment

## Part 4 structural template (mirror byte-for-byte)

`<div class="chapter-intro">` / `<div class="chapter-outro">` wrappers; `<div class="quiz-container" data-pass-threshold="80">` quizzes with `<h3>` / `<p class="quiz-subtitle">` / `<div class="quiz-progress">` / `<button class="quiz-choice">` / `<p class="quiz-explanation">` / `<div class="quiz-score">`. NO emojis. NO Co-Authored-By.
