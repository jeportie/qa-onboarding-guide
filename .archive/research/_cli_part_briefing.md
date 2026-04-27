# CLI Part — Research Briefing

Date: 2026-04-24
Trigger: QAA-615 (Spike: Revoke token approval using CLI)

## Tickets in scope

### QAA-615 — primary
- Type: Task (Spike)
- Status: In Progress
- Assignee: Jerome PORTIER
- Reporter: Gabriel BECERRA
- Labels: LLD, LLM, UI
- Parent epic: QAA-919 "SWAP regression coverage automation"
- Linked to: QAA-613 (Discovery — Connected)
- Description (verbatim):
  > Hey team. As we have the objectif of testing the whole flow with token approval + swap, we need to find a way to revoke the approval to be able to run the test again. This will be helpful for automatic regression testing and not nighlty executions.

**Key nuance:** "automatic regression testing AND NOT nightly executions" — revoke is for the broadcast-enabled regression suite. Nightly QAA-613 tests have broadcast disabled, so on-chain allowance never moves and revoke isn't strictly needed for them. But the regression suite (which DOES broadcast) needs revoke between runs to keep the test deterministic.

### QAA-613 — sibling, the nightly tests that consume QAA-615
- Description: "implement the token approval flow with Thorchain, Uniswap, and LiFi. Broadcast will be disabled so just the first part of the flow will happen, no actual swap."
- Labels: LLD, LLM, UI
- Status: In Progress
- Sequence chosen by Jerome: do QAA-615 first (so the CLI and the guide section land), then QAA-613 (UI tests) on top.

### Family for context (parent QAA-919)
- QAA-617: CI environment droplist (LLD/LLM, TO DO)
- QAA-702: History ERC20 export (LLM, in Code Review — already shipped per guide)
- QAA-703: Discreet mode tests (LLD/LLM, Open)
- QAA-704: Feedback form tests (LLD, Open)
- QAA-705: No device connected (LLD, Open)
- QAA-706: Cross-account warning DEX providers — 1inch/Velora/Uniswap (LLD/LLM, Open)
- QAA-721: Empty Provider link in history (LLD/LLM, Open)
- QAA-722: Use MIN amount from request (LLD/LLM, Open)
- QAA-815: Device-connection changes preserve (LLM, Done)
- QAA-1053: DEX swap automation Speculos investigation (Done — relevant for the close+reopen pattern)

## Victor's confirmation (Slack DM, 2026-04-24)

> "yes en gros c'est pcq l'approve du token c'est a faire que 1x. et apres c'est good. donc faut ajouter un hook (before ou after) le test pour clean justement. et je crois que c'est potentiellement possible de faire ca avec le CLI mais faut investiguer."

Translated: "yes basically because the token approve only needs to be done once. After that it's good. So we need to add a before/after hook to the test to clean up. And I think it's potentially possible to do this with the CLI but it needs investigation."

This confirms: the CLI is **test-data infrastructure**, not test code. Pre-test hook revokes any allowance left over from previous runs so the next run starts at allowance=0.

## What "token approval" means (for chapter intro context)

ERC-20 tokens don't sit at user addresses — they sit in the token contract under a ledger entry. To swap USDC via a DEX (Thorchain, Uniswap, LiFi…) the user must first sign an `approve(spender = DEX_router, amount = N)` transaction giving that contract permission to withdraw N tokens from their address. THEN the actual swap transaction happens. Two signatures on the device, in sequence. Each provider has a different router address — Thorchain, Uniswap V3 router, LiFi diamond contract. Without the approve, the swap fails with `ERC20: insufficient allowance`.

To **revoke** an approval, the user signs `approve(spender = same_DEX_router, amount = 0)`. This is what `wallet-cli revoke` would emit. Same APDU shape as `approve(N)`, just with N=0.

## Why a CLI part in the guide is justified now

1. **Real existing surface to teach.** `apps/wallet-cli/` is a non-trivial workspace — Bun + Bunli + DMK + ~20 source files + a full mocked test runner. Students will not absorb it from a sidebar pointer.
2. **Active development.** The legacy `apps/cli` is being replaced by `apps/wallet-cli`. Onboarding needs to land readers on the right one.
3. **It will be permanent infrastructure.** Once `wallet-cli revoke` exists, every E2E hook calling it becomes a documented pattern. Future tickets (QAA-617 environment droplist, QAA-706 cross-account warnings, QAA-722 min-amount) will all need their own hooks; the CLI part becomes the canonical "how to add a fixture command" reference.
4. **Pedagogical parity.** Part 3 = Desktop, Part 4 = Mobile, **Part 5 = CLI** is the third E2E flavor. Treating it as a peer (not a sub-section) matches reality and lets the renumbering settle.

## Proposed numbering

Insert **new Part 5 "CLI Automation"** between current Part 4 (Mobile) and current Part 5 (Swap):

- Part 0 Welcome → unchanged
- Part 1 Foundations → unchanged
- Part 2 Shared Tooling → unchanged
- Part 3 Desktop E2E → unchanged (chapters 3.1-3.10 + Final)
- Part 4 Mobile E2E → unchanged (chapters 4.1-4.11 + Final)
- **Part 5 CLI Automation → NEW** (chapters 5.1-5.N + Final)
- Part 6 Swap Live App → renumbered from old Part 5
- Part 7 Mastery & Contributing → renumbered from old Part 6

Sidebar update + README update + cross-ref renumbering pass needed afterwards.

## Repo paths agents will need

- Guide: `/Users/jerome.portier/qa-onboarding-guide/`
- Monorepo: `/Users/jerome.portier/src/tries/2026-04-08-LedgerHQ-ledger-live/`
- Wiki mirror: `/Users/jerome.portier/src/tries/2026-04-14-LedgerHQ-ledger-live.wiki/`
- Speculos: `/Users/jerome.portier/src/tries/2026-04-14-LedgerHQ-speculos/`
- Swap Live App: `/Users/jerome.portier/src/tries/2026-04-17-LedgerHQ-swap-live-app/`
- Wallet API: `/Users/jerome.portier/src/tries/2026-04-17-LedgerHQ-wallet-api/`
- Coin apps (BOLOS): `/Users/jerome.portier/src/tries/2026-04-13-LedgerHQ-coin-apps/`

## wallet-cli quick reference (from .agents/skills/ledger-wallet-cli/SKILL.md)

- Bin: `wallet-cli` (also `pnpm --silent wallet-cli start <cmd>`)
- Runtime: Bun + Bunli framework
- Transport: Device Management Kit (DMK) over USB
- Currencies: bitcoin, ethereum, base (uses ETH app), solana — mainnet + testnets
- Existing commands: `account discover`, `receive`, `send`, `balances`, `operations`
- Account descriptor V1: `account:1:<type>:<network>:<env>:<xpub_or_address>:<path>`
- Tests: `apps/wallet-cli/src/test/` — bun test + cli-runner + mock-DMK + http-intercept + mock-server
- Sandbox warning: device-touching commands need `dangerouslyDisableSandbox: true`

**Approve / revoke is missing today.** The 5 commands above do not include allowance management. QAA-615 is exactly the spike to add it.

## Open spike question (the deliverable QAA-615 must answer)

Three implementation paths to compare:
1. **New `revoke` subcommand** (`wallet-cli revoke <account> --spender <addr>`) — cleanest UX, requires the most code.
2. **Generalize `send` to accept arbitrary calldata** (`wallet-cli send <account> --to <token-contract> --calldata 0x095ea7b3<spender_padded>0000…0`) — minimum new surface, max user error potential.
3. **Both — `revoke` as a thin wrapper around generalized `send`** — best of both, one implementation.

The spike chooses one and ships a PoC.
