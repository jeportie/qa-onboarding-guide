# PART 6 — CLI AUTOMATION

Part 6 covers the CLI surface QAA owns: `apps/cli`, the typed wrappers around it, the Speculos device-tap layer that drives signing, and the broadcast discipline that keeps regression tests deterministic. Ten chapters; this fragment contains the first two — the framing chapter (6.1) and the blockchain primer (6.2). Read them in order; everything after assumes the vocabulary established here.

The reference branch for everything in Part 6 is `support/qaa-615-add-revoke-token` at commit `4fa4868a35`, authored by Abdurrahman SASTIM. Three small files, thirty-seven lines added — the canonical example of "do one thing well" in QAA tooling. By the end of the part you should be able to read those three files at a glance and know exactly which layer each line touches.

## CLI in QA — apps/cli is the canonical CLI

<div class="chapter-intro">
You have learned the desktop E2E surface (Part 4) and the mobile E2E surface (Part 5). This chapter introduces the third surface QAA owns: the command-line interface. The CLI is not a product the customer touches — it is internal test-data infrastructure. It seeds accounts, reads device addresses, queries on-chain allowances, and signs approve / revoke transactions. The QAA-canonical binary is <code>apps/cli</code>; <code>apps/wallet-cli</code> is experimental and out of scope for this part.
</div>

### 6.1.1 What this part teaches and why a third E2E surface

Parts 4 and 5 covered the two product-facing E2E suites: Playwright drives Ledger Live Desktop, Detox + Jest drives Ledger Live Mobile. Both can tap UI elements and read screens. Neither can cheaply seed an account, query an ERC-20 allowance, or burn a deterministic on-chain transaction before a test runs. That is why a third surface exists: the CLI.

### 6.1.2 Why a CLI in QA

The CLI is test-data infrastructure. Three jobs are easier in a CLI subprocess than in a UI driver.

**Test-data seeding.** A swap test needs an Ethereum account with a known balance, a known token balance, a known address, and a known derivation path. Building that state through the UI takes minutes per test and breaks every time a screen changes. The CLI builds it in one subprocess call, decoupled from the UI. Concretely: `liveDataCommand(account)` writes a JSON userdata file that the desktop app reads at launch, and the app boots straight into the prepared state. The UI driver never has to click through "Add account → Choose currency → Choose derivation → Confirm" four times before each test. That preparation cost — measured in tens of seconds per account, multiplied by every spec — collapses into a single subprocess call.

**Allowance state determinism.** ERC-20 allowances live on-chain. They survive simulator restarts, repo re-clones, and CI runners. A nightly run that broadcasts an `approve(spender, 100)` leaves `allowance[seed][spender] = 100` permanently. The next run's "fresh approval" device screen reads "you already approved 100 — approve more?" and any test that asserts "Approve 100 USDC" content will fail. Manual recovery via revoke.cash is the human fallback (Chapter 6.3); the CLI hook is the automated one. This is the headline reason QAA-615 exists, and it's why Chapter 6.2 spends time on the storage model — without that mental picture, the determinism problem looks like a flaky test rather than a structural reality. There is no shortcut around this; you cannot "reset the chain" between test runs. You can only reset the slots you touch.

**One-shot operations Playwright and Detox shouldn't do.** Reading a device address, signing a single revoke transaction, broadcasting it, and exiting is five lines of CLI. It is fifty lines of Page Object Model with a Speculos device dance. Use the right tool: UI drivers for UI behaviour, CLI for state operations. The CLI is also the only place where a non-product code path can reasonably live — the harness can call `tokenAllowance` to make a routing decision (skip the device dance if allowance is already sufficient) without polluting the desktop or mobile app with test-only code. This separation of concerns — product code stays product-only, test-only utilities live in `apps/cli` and `cliCommandsUtils.ts` — is one of the reasons the codebase stays maintainable.

In short: the UI drivers test what the user does; the CLI prepares the world the user does it in. Confusing the two — trying to revoke through the UI, or trying to verify swap UI through CLI subprocesses — wastes test time and produces fragile suites.

### 6.1.3 The two CLIs — apps/cli vs apps/wallet-cli

The monorepo contains two CLI applications. **Only `apps/cli` is the QAA canonical CLI.** Use it. Teach it. Read its source.

| | `apps/cli` | `apps/wallet-cli` |
|---|---|---|
| Package name | `@ledgerhq/live-cli` | `@ledgerhq/wallet-cli` |
| Stack | Node + Commander | Bun + Bunli + DMK |
| Status | Mature, used by E2E | Experimental |
| QAA usage | **Canonical** — every E2E CLI helper invokes it | Not used by QAA |
| Binary | `ledger-live` (`bin/index.js`) | Separate, not wired into E2E helpers |
| Coverage in this part | Full | Out of scope |

`apps/wallet-cli` is a future-facing rewrite using Bun and the new Device Management Kit. It is not part of the QAA workflow today. Earlier drafts of this onboarding guide taught wallet-cli; that material was wrong. If you read it elsewhere, ignore it. Everything in Part 6 from here on refers to `apps/cli`.

> **Verify:** if a future PR adds wallet-cli wrappers in `cliCommandsUtils.ts`, the canonical-CLI claim must be revisited. As of `support/qaa-615-add-revoke-token` @ `4fa4868a35`, every wrapper invokes `apps/cli/bin/index.js`.

The reason this section exists at all: earlier drafts of Part 6 of the onboarding guide taught wallet-cli as the canonical CLI, complete with Bun setup instructions, Bunli command authoring patterns, and DMK descriptor walkthroughs. That material was wrong — not "out of date", but actively misleading because no QAA test path uses any of it. If you onboard a teammate and they cite "I read in the guide that we use wallet-cli", that's a copy of the obsolete draft. Send them this chapter instead.

A historical note that is occasionally useful: `apps/wallet-cli` was started as the long-term replacement for `apps/cli` once Bun and DMK matured. That replacement has not landed, the wallet-cli command surface is incomplete, and no E2E test currently invokes it. If you see a Slack thread or an old PR description that says "wallet-cli is the future", treat it as aspirational. For the work in front of you in 2025, `apps/cli` is the one that runs in production and the one Part 6 teaches.

### 6.1.4 Where apps/cli sits in the monorepo

```
apps/
|-- cli/                              # @ledgerhq/live-cli
|   |-- bin/
|   |   `-- index.js                  # entry: `ledger-live` binary
|   |-- src/
|   |   |-- commands/
|   |   |   |-- blockchain/           # 16 blockchain commands (catalog below)
|   |   |   |-- live/                 # liveData, etc. (account fixtures)
|   |   |   `-- device/               # firmware / app management (out of QAA scope)
|   |   |-- transaction.ts            # inferTransactionsOpts (parses --mode, --token, --spender)
|   |   `-- scan.ts                   # scanCommonOpts (--currency, --index, --scheme)
|   |-- scripts/
|   |   |-- gen.mjs                   # zx-driven prebuild: generates command index
|   |   |-- build.mjs                 # zx-driven build
|   |   `-- test.mjs                  # zx-driven local tests
|   `-- package.json
|
`-- wallet-cli/                       # OUT OF SCOPE — do not study for QAA work
```

The `package.json` highlights:

```json
{
  "name": "@ledgerhq/live-cli",
  "bin": { "ledger-live": "./bin/index.js" },
  "scripts": {
    "prebuild": "zx ./scripts/gen.mjs",
    "build": "zx ./scripts/build.mjs",
    "test": "zx ./scripts/test.mjs"
  }
}
```

`zx` is Google's Bash-in-JavaScript helper. The `prebuild` step regenerates the command index — a single file that imports every command in `src/commands/**` so Commander can register them. After cloning and installing, you must `pnpm --filter @ledgerhq/live-cli build` once before E2E tests can find the binary.

The runtime stack is Node + Commander. No Bun, no Bunli, no DMK. Reading the source is straightforward TypeScript.

A handful of Commander idioms recur across the blockchain commands and are worth recognising on first read:

- `scanCommonOpts` — a shared options bundle (`--currency`, `--index`, `--scheme`, `--xpub`, etc.) attached to every command that needs to identify an account.
- `inferTransactionsOpts` — a shared options bundle for transaction-shaped commands (`--amount`, `--recipient`, `--token`, `--mode`, `--approveAmount`, etc.). Lives in `apps/cli/src/transaction.ts` and dispatches into family-specific handlers (e.g. `libs/coin-modules/coin-evm/src/cli-transaction.ts`).
- `--format` — most commands accept `default | json | silent` and swap their output writer accordingly. The wrappers in `cliCommandsUtils.ts` always pass `--format json` when they need to parse output (`tokenAllowance`, mainly).
- `--ignore-errors` — sometimes used to keep the rxjs pipeline from short-circuiting on partial failures.

You will not need to touch the CLI source on day one of a QAA fix campaign. But when something goes wrong — the wrapper passes args that the CLI doesn't recognise, or the output format changed and the parser breaks — being able to read `send.ts` end-to-end is the difference between a 30-minute fix and a half-day of guessing.

A note on `bin/index.js` itself: it is a thin shim. It imports the auto-generated commands index produced by `prebuild`, registers each command with Commander, and dispatches based on argv. There is no bespoke argument parsing in the entry point — Commander handles it. This is why a stale `prebuild` (no commands index, or a commands index that omits a recently-added command) produces "unknown command" errors instead of silent failures. If `apps/cli` doesn't seem to know about a command you can see in the source, run `pnpm --filter @ledgerhq/live-cli build` and try again.

### 6.1.5 The 16 blockchain commands at a glance

Path: `apps/cli/src/commands/blockchain/`.

| File | Purpose | QAA uses? |
|---|---|---|
| `bot.ts` | Speculos bot (long-running test stress) | No — out of QAA scope |
| `botPortfolio.ts` | Bot portfolio management | No |
| `botTransfer.ts` | Bot-driven transfers | No |
| `broadcast.ts` | Broadcast a pre-signed tx | Maybe (debug only) |
| `confirmOp.ts` | Confirm an operation | No |
| `derivation.ts` | Derivation path utility | No |
| `estimateMaxSpendable.ts` | Estimate max sendable amount | No |
| `generateTestScanAccounts.ts` | Test fixture generator | No |
| `generateTestTransaction.ts` | Test fixture generator | No |
| **`getAddress.ts`** | **Read device address** | **YES** — `runCliGetAddress` |
| `getTransactionStatus.ts` | Get tx status | No |
| `receive.ts` | Receive flow utility | No |
| **`send.ts`** | **Sign + (maybe) broadcast a tx; supports `--mode approve\|revokeApproval`** | **YES** — `runCliTokenApproval` |
| `signMessage.ts` | Sign EIP-191 / EIP-712 messages | No |
| `sync.ts` | Account sync | No |
| `testDetectOpCollision.ts`, `testGetTrustedInputFromTxHash.ts` | Test utilities | No |
| **`tokenAllowance.ts`** | **Read ERC-20 allowance** | **YES** — `runCliGetTokenAllowance` |

Plus `apps/cli/src/commands/live/`:

- `liveData.ts` — generates the account fixture file Ledger Live consumes at startup. Used by `runCliLiveData` and is QAA's most heavily used CLI helper. Every spec that needs a pre-loaded account fans out from here.

And `apps/cli/src/commands/device/` — firmware / app management, out of QAA scope.

Of the 16 blockchain commands, **four** are part of the QAA workflow. Everything else is either bot infrastructure (long-running stress tests, separate concern) or low-level test utilities the wrappers in `cliCommandsUtils.ts` do not expose. Stay focused on the four.

A few of the "out of scope" entries are worth a one-line description so you don't waste time on them:

- `bot.ts` — runs a Speculos-driven stress loop that exercises send / sync / portfolio over many accounts and many currencies. It is its own product surface, owned by a different team. If you find yourself reading it, you've taken a wrong turn.
- `botTransfer.ts`, `botPortfolio.ts` — building blocks for the bot above.
- `broadcast.ts` — takes a hex-encoded signed transaction and broadcasts it. Useful only for very specific debug scenarios where you have a signed tx blob from somewhere else and want to push it to chain.
- `confirmOp.ts`, `getTransactionStatus.ts` — utilities for waiting on or inspecting an operation by hash. The QAA wrappers fold the equivalent functionality into the `--wait-confirmation` flag of `send`.
- `derivation.ts` — derive an address for a given path / xpub combo. The QAA equivalent is `getAddress` on a real device, which gives you both the path and the on-device confirmation that the device computes the same thing.
- `signMessage.ts` — EIP-191 / EIP-712 signing. The QAA E2E layer doesn't currently exercise this through the CLI; message-signing flows are tested via the UI driver.

If the day-job lands you in any of these, ask first whether you're solving the right problem at the right layer. Most of the time the answer is "use one of the four canonical commands instead".

**Why `liveData` lives under `commands/live/` and not `commands/blockchain/`.** Because it's not a blockchain command. It builds an off-chain JSON userdata file from public derivation data — no transaction, no signature, no RPC call. Putting it under `commands/live/` makes the boundary clear: blockchain commands talk to chains and devices; live commands prepare client-side state. The directory layout is a hint about what each command does.

This split also explains why `liveData` has no Speculos requirement: it doesn't need a device because it doesn't sign anything. It can run on a laptop with no Docker, no emulator, no network, and produce a fully populated userdata file in milliseconds. Tests that only need account fixtures (most of them) thus avoid the full Speculos overhead — they just consume the userdata file the fixture engine generated.

### 6.1.6 The four commands QAA actually uses

**`liveData`** (under `commands/live/`) — the workhorse. Generates a JSON userdata file with the accounts your test needs. Called via `runCliLiveData` and wrapped in the curried fixture helpers `liveDataCommand`, `liveDataWithAddressCommand`, `liveDataWithParentAddressCommand`, `liveDataWithRecipientAddressCommand`. Every desktop test fixture's `cliCommands: [...]` array starts with one of these. Forward-ref Chapter 6.4 for the curried-function pattern. The reason this command is so central: a single `liveData` invocation can produce a userdata snapshot covering Bitcoin, Ethereum, Polygon, Solana, an ERC-20 sub-account, and a derivation-path variant — enough world to run almost any spec.

**`getAddress`** — reads a device address for a given currency and derivation path. Called via `runCliGetAddress`. Used implicitly through `getAccountAddress(account)`, which both fetches the address and writes it back onto the in-memory account object so subsequent assertions can reference it. This command is the bridge between "the test thinks the account is at index 0 on path m/44'/60'/0'/0/0" and "the chain agrees that's address `0xabc...def`": without it, send-flow assertions ("verify the recipient on screen matches the address you expect") cannot be written.

**`tokenAllowance`** — reads the on-chain ERC-20 allowance for `(owner, spender, token)`. Called via `runCliGetTokenAllowance`. Wrapped by `isTokenAllowanceSufficientCommand` — the pre-check that lets `ensureTokenApproval` skip the device dance when the allowance is already large enough. Output is parsed by `parseTokenAllowanceCliOutput`. This is a read-only RPC call — no signing, no broadcast, no Speculos required, fast (~1 second). The pre-check pattern (read first, only act if necessary) is what keeps `ensureTokenApproval` cheap on warm seeds.

**`send --mode approve|revokeApproval`** — the same `send` command does both. With `--mode approve` and `--approveAmount N` it sets `allowances[you][spender] = N`. With `--mode revokeApproval` it sets it back to `0`. Wrapped by `runCliTokenApproval`, then by `approveTokenCommand` / `revokeTokenCommand` in `cliCommandsUtils.ts`. Forward-ref Chapter 6.4 for the layer-by-layer walkthrough and Chapter 6.6 for the broadcast gate that sits inside this command.

That's the surface. Four commands. Eight or nine wrappers. Everything in Part 6 is built on these.

A useful mental model: `liveData` is the "before" hook (set up state cheaply, no device required); `getAddress` and `tokenAllowance` are the "during" reads (cheap queries the test driver makes to decide what to do); `send --mode ...` is the "before / after" hook that actually moves on-chain state (expensive, requires a Speculos device, optionally broadcasts). The four commands cover the full lifecycle: prepare, observe, mutate.

| Command | Cost | Speculos required? | Network call? | Mutates state? |
|---|---|---|---|---|
| `liveData` | low | no | no | local file only |
| `getAddress` | low | yes (cheap, no signing) | no | no |
| `tokenAllowance` | low | no | yes (RPC read) | no |
| `send --mode approve\|revokeApproval` | high | yes (full signing) | yes (RPC + broadcast) | yes (when not gated) |

The cost asymmetry here is exactly why the test fixtures lean so heavily on `liveData` and the pre-check pattern: the cheap operations run unconditionally; the expensive ones run only when necessary.

**A note on `runCliTokenApproval`'s broadcast-on assumption.** The wrapper that produces the `send --mode approve|revokeApproval` command line does not itself manage the broadcast env. That's why both `approveTokenCommand` and `revokeTokenCommand` (the higher-level wrappers around `runCliTokenApproval`) explicitly flip `DISABLE_TRANSACTION_BROADCAST` to `"0"` before calling and restore it after. If you call `runCliTokenApproval` directly without flipping the env first — and the calling test had broadcast disabled by default — your approve / revoke will sign and silently not land. Always go through `approveTokenCommand` / `revokeTokenCommand` for approve and revoke flows; that's where the env discipline lives.

### 6.1.7 The five-layer integration at a glance

| # | Path | Role |
|---|---|---|
| 1 | `apps/cli/bin/index.js` | Legacy CLI binary (Node, Commander) |
| 2 | `libs/ledger-live-common/src/e2e/runCli.ts` | `child_process.spawn` engine + retries |
| 3 | `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts` | Typed TypeScript wrappers |
| 4 | `libs/ledger-live-common/src/e2e/families/evm.ts` | Speculos device-tap automation |
| 5 | `e2e/desktop/tests/page/swap.page.ts`, `e2e/mobile/page/trade/swap.page.ts` | POM methods orchestrating Speculos + CLI |

Read this table top-down: it's the call stack of a single revoke. A POM method (5) calls a typed wrapper (3); the wrapper builds an arg string and hands it to the spawn engine (2); the engine forks `node apps/cli/bin/index.js` (1); meanwhile, in parallel, the device-tap helper (4) drives Speculos's REST API to walk the on-screen review and press hold-to-sign. Both halves must finish for the revoke to land. Chapter 6.4 walks the full path line by line.

Why five layers? Because each one solves a different concern.

- Layer 1 is the binary itself, the only thing that knows how to actually build, sign, and broadcast a transaction.
- Layer 2 isolates "spawn a Node child process" from "construct CLI args" — the spawn engine is generic, and the same `runCliCommand` services every helper.
- Layer 3 is where typing lives: `revokeTokenCommand(account: TokenAccount, spender: string)` is checkable at compile time; the equivalent string-building inside a spec would not be.
- Layer 4 is the bridge to Speculos. The CLI subprocess and the device-tap helper run *concurrently*: the CLI blocks waiting for an APDU response; the helper drives the screens in parallel until a hold-to-sign completes; only then does the CLI receive its signature and broadcast.
- Layer 5 is per-feature orchestration: the swap POM knows it needs to flip `DISABLE_TRANSACTION_BROADCAST`, snapshot the Speculos port, swap to an Ethereum-app device, run the revoke, restore the port, and unflip the env. None of that belongs in a CLI wrapper; it's swap-specific glue.

A diagram for orientation:

```
   Spec (Playwright / Detox test file)
       |
       v
   POM method (e.g. swap.page.ts:revokeTokenApproval)
       |  flips DISABLE_TRANSACTION_BROADCAST="0"
       |  snapshots SPECULOS_API_PORT
       |  starts new Speculos for the right app
       v
   Typed wrapper (cliCommandsUtils.ts:revokeTokenCommand)
       |  builds args: send --currency X --mode revokeApproval ...
       v
   Spawn engine (runCli.ts:runCliCommand)              ─╮
       |  child_process.spawn("node", [bin, ...args])  │
       v                                               │ in parallel
   apps/cli (Node + Commander)                         │
       |  signOperation → broadcast (gated by env)     │
       |  blocks on device APDU response               │
       ^                                               │
       │ APDU                                          │
       │                                               │
   Speculos Docker container (REST API)                │
       ^                                               │
       │ /button, /touch, /screenshot                  │
       │                                               │
   families/evm.ts:approveToken()  ─────────────────── ╯
       walks screens, presses Hold to sign for 3s
```

Read this as: the typed wrapper calls the spawn engine, which forks a CLI subprocess; the CLI dials Speculos for signing; the device-tap helper, started in parallel by the POM method, drives Speculos's REST API to walk through the on-screen review and execute the hold-to-sign. When the device-tap completes, Speculos returns the APDU response to the CLI, which broadcasts (or doesn't) and exits.

Every chapter from here unpacks one slice of this diagram.

### 6.1.8 Footguns to know upfront

Five things will catch you out if you don't know them.

**The CLI is invoked as a subprocess, not a programmatic API.** `runCli.ts` calls `child_process.spawn("node", [LEDGER_LIVE_CLI_BIN, ...args])` and reads stdout. There is no `import { send } from "@ledgerhq/live-cli"`. Every call pays Node startup cost (~300-500 ms). This is by design — the CLI was a standalone binary first, the E2E harness wrapped it later. Don't try to short-circuit with a programmatic call; the wrappers expect stdout strings.

A small but important consequence: stdout parsing is the wrappers' contract. When `tokenAllowance` outputs `--format json`, the wrapper expects a parseable JSON line; when `send` outputs default text, the wrapper expects specific success / error markers. If a CLI command's output format changes — even for a "harmless" log line addition — the wrappers can break in non-obvious ways. The `parseTokenAllowanceCliOutput` helper is fragile in exactly this way: it greps for a known JSON shape inside potentially noisy stdout. This is a known sharp edge, and Chapter 6.4 covers the parsing layer in detail.

**The `+` separator in arg lists.** The engine joins args with `+` and splits them back inside the spawn call:

```ts
const args = command.split("+");
const child = spawn("node", [LEDGER_LIVE_CLI_BIN, ...args], { ... });
```

So `runCliTokenApproval` builds:

```
"send+--currency+ethereum+--mode+revokeApproval+--token+ethereum/erc20/usd_coin+--spender+0xRouter+--index+0+--wait-confirmation"
```

…which splits into the ten-element argv `apps/cli` expects. The reason: shell-escape pain is awful when args are forwarded across multiple processes and `+` never appears in real CLI arg values. If you build a custom command string with literal `+` characters in a value, you will break the splitter. Don't.

**`DISABLE_TRANSACTION_BROADCAST` env can silently flip behaviour.** The CLI's broadcast step is gated on `process.env.DISABLE_TRANSACTION_BROADCAST` (and the equivalent CLI flag `--disable-broadcast`). When set to `"1"` the CLI signs locally and exits without calling `bridge.broadcast(...)`. When unset or `"0"` it broadcasts. Many test files set the flag at module load; the approve / revoke wrappers temporarily flip it back to `"0"` inside a try/finally. If the finally is missing, every subsequent CLI call in the run silently broadcasts. Chapter 6.6 deep-dives this.

The "silent" part is what gets people. There is no error message when broadcast is suppressed, no warning, no log line in the default format. The CLI looks like it succeeded (it did sign), the test driver continues, and only later — when the next test reads on-chain state and finds the previous "approval" never actually landed — does the failure surface, often far from the cause. Discipline around the env flag is non-negotiable; treat it like an `O_NONBLOCK` flag on a file descriptor.

**Speculos port mutation is global state.** The Speculos lifecycle helpers set `process.env.SPECULOS_API_PORT` (and the live-env mirror `setEnv("SPECULOS_API_PORT", port)`) so the spawned CLI knows which Docker container to connect to. This is a global mutation. Two parallel tests cannot both start Speculos on different ports without serialising the env update. Chapter 6.5 walks the snapshot/restore pattern.

This is also why Playwright's per-worker test parallelism interacts oddly with CLI-driven specs: each worker is its own Node process with its own `process.env`, so workers don't fight each other directly — but tests *within* the same worker that both want their own Speculos on different ports definitely do. The discipline is "one Speculos at a time per worker, snapshot/restore around any swap-in".

**Two Speculos instances cannot share a port.** Each Speculos Docker container binds to a host port to expose its REST API. The CLI subprocess uses the env-var port to dial that container. If two containers needed to share one port, the OS would reject the second `bind()`. The harness prevents this by running one Speculos at a time per worker and serialising port swaps. A test that already has one Speculos running for the Exchange app, and now wants to start a second Ethereum-app Speculos to issue a revoke, must save the previous port, launch the new device, run the revoke, and restore the previous port in `finally`. The senior's `revokeTokenApproval` POM method on `swap.page.ts` does exactly this:

```ts
const previousSpeculosPort = getEnv("SPECULOS_API_PORT");
const speculos = await launchSpeculos(fromAccount.currency.speculosApp.name);
try {
  // do the revoke
} finally {
  await cleanSpeculos(speculos, previousSpeculosPort);
}
```

Without the snapshot you leak the port; without the restore you break the swap test that runs next.

A sixth thing worth flagging early: **there is no transactional rollback**. If a CLI revoke broadcasts and then the test fails for an unrelated reason, the on-chain state is already mutated. The revoke is permanent. Tests that broadcast must be designed assuming every successful broadcast lands and every failed broadcast leaves state in an undefined intermediate place — typically "broadcast succeeded but the harness crashed before it noticed". This is why broadcast-enabled tests require careful state management and why `DISABLE_TRANSACTION_BROADCAST=1` is the default in most CI paths.

**Output parsing assumes a stable format.** Already mentioned in passing, but it's worth its own bullet because it is the source of the most surprising regressions. The wrappers in `cliCommandsUtils.ts` parse stdout — sometimes greping for a substring, sometimes parsing JSON. If a CLI command starts emitting an extra log line (debug print, deprecation warning, even a leading newline) the parser can break. The fix is always in the parser, not the CLI: the canonical output format is whatever the CLI emits, and the parsers accommodate it. But "test fails because the CLI now logs an extra warning" is a real failure mode. When triaging a sudden mass test failure, run the failing CLI command by hand and eyeball the stdout against what the parser expects.

**Retry semantics are coarse.** The spawn engine's `runCliCommandWithRetry` retries on a list of "retryable" error patterns (network blips, RPC timeouts, transient Speculos issues). It does not retry on signing rejections, calldata errors, or "no such command". The retry policy is shared by every CLI helper, so a flake in one helper benefits all helpers — but the policy is also a cudgel: a non-retryable error fails fast and surfaces as the test failure. If you're seeing a "command failed" without retries, check the error message against `isRetryableError`'s pattern list to confirm whether the error type was eligible.

### 6.1.9 What this part covers and what it does not

**Covers.** `apps/cli` and the four commands above; the spawn engine in `runCli.ts`; the typed wrappers in `cliCommandsUtils.ts`; the Speculos device-tap automation in `families/evm.ts`; the POM methods in `swap.page.ts`; Speculos lifecycle (`speculosUtils.ts`); the `DISABLE_TRANSACTION_BROADCAST` flow; the daily QAA workflow; and a line-by-line walkthrough of the QAA-615 commit.

**Does not cover.**
- `apps/wallet-cli` — out of QAA scope, do not study.
- The Speculos bot at `apps/cli/src/commands/blockchain/bot.ts` — separate stress-test surface, not part of QAA's E2E loop.
- WalletSync via CLI (`runCliKeyRingProtocol`, `runCliLedgerSync`) — used by Ledger Sync features, not part of the approve/revoke story.
- Deep coin-evm internals — Chapter 6.2 covers what you need about ERC-20 calldata; the bridge layer at `libs/coin-modules/coin-evm/src/cli-transaction.ts:inferTransactions` is referenced but not traced line-by-line.

If a question lands in your inbox about wallet-cli, redirect to the team that owns it. QAA owns `apps/cli`.

**A minimal mental model to carry into the next chapters.** The CLI exists for state. State is either off-chain (account fixtures via `liveData`) or on-chain (allowances via `tokenAllowance` to read, `send --mode approve|revokeApproval` to write). State writes need a Speculos device because the CLI needs to obtain a hardware-style signature. State writes are gated by `DISABLE_TRANSACTION_BROADCAST` — the difference between "sign and stop" (no chain mutation) and "sign and broadcast" (chain mutation). Tests choose between those modes deliberately; the wrappers help by encapsulating the env management inside try/finally blocks.

Hold that picture in mind for Chapters 6.3 onward and the rest of Part 6 will compose cleanly.

### 6.1.10 A first orientation in practice

To make this concrete, here is what your first ten minutes with `apps/cli` look like in a typical fix-campaign flow.

**Step 1 — locate the binary.** From the monorepo root:

```
$ ls apps/cli/bin/index.js
apps/cli/bin/index.js
```

If that path doesn't exist, the prebuild hasn't run. Run `pnpm --filter @ledgerhq/live-cli build` and try again.

**Step 2 — find the wrappers.** Open `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts` and read end-to-end. It is short — under 250 lines — and every helper you'll touch is named in there. Pay attention to `setDisableTransactionBroadcastEnv` and the try/finally pattern around `approveTokenCommand` / `revokeTokenCommand`.

**Step 3 — find the spawn engine.** Open `libs/ledger-live-common/src/e2e/runCli.ts`. Eighty lines. Read `runCliCommand` once; you now understand how every CLI invocation is wired.

**Step 4 — find the device-tap layer.** Open `libs/ledger-live-common/src/e2e/families/evm.ts`. The two functions `approveTokenTouchDevices` and `approveTokenButtonDevice` are the entire device-side of the approve / revoke flow. Note that they share `approveToken()` as the entry point, which dispatches based on `isTouchDevice()`.

**Step 5 — find an existing call site.** `e2e/desktop/tests/page/swap.page.ts` imports `approveTokenCommand`, `isTokenAllowanceSufficientCommand`, and (on the senior's branch) `revokeTokenCommand`. Read `ensureTokenApproval` once. This is the canonical pattern.

**Step 6 — find a fixture-style call site.** Open `e2e/desktop/tests/specs/earn.v2.spec.ts` and look at any of the `cliCommands: [liveDataCommand(...)]` entries. This is the *other* canonical pattern: instead of calling a CLI helper imperatively from a POM method, you declare the CLI calls the test needs as part of the test's metadata, and the desktop fixture engine in `e2e/desktop/tests/fixtures/common.ts` runs them at the right moment. Most spec authors interact with the CLI exclusively through this pattern — they never call `runCli*` themselves; they list the curried fixture helpers in their test's `cliCommands` array and trust the engine.

You now have the full call stack in your head: spec → POM method → wrapper → spawn engine → CLI binary, with the device-tap helper running in parallel. You also have the alternative entry point: spec metadata → fixture engine → curried wrapper → spawn engine → CLI binary. Everything else in Part 6 is depth on top of this skeleton.

### 6.1.11 Resources

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><code>apps/cli/README.md</code> — the binary's own quickstart</li>
<li>Branch <code>support/qaa-615-add-revoke-token</code> @ commit <code>4fa4868a35</code> — the senior's reference implementation of <code>revokeTokenCommand</code> + <code>revokeTokenApproval</code></li>
<li><code>libs/ledger-live-common/src/e2e/runCli.ts</code> — the spawn engine (~80 lines, read it once end-to-end)</li>
<li><code>libs/ledger-live-common/src/e2e/cliCommandsUtils.ts</code> — the typed wrappers (~250 lines)</li>
<li><code>libs/ledger-live-common/src/e2e/families/evm.ts</code> — Speculos device-tap helpers</li>
<li><a href="https://github.com/google/zx">Google zx</a> — the prebuild scripting layer</li>
<li><a href="https://github.com/tj/commander.js">Commander.js</a> — the CLI argument parser</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> <code>apps/cli</code> is the canonical QAA CLI. Sixteen blockchain commands exist; four matter (<code>liveData</code>, <code>getAddress</code>, <code>tokenAllowance</code>, <code>send --mode approve|revokeApproval</code>). The CLI is not a library — it is a subprocess invoked via <code>child_process.spawn</code>, and its behaviour is shaped by two pieces of global state: <code>SPECULOS_API_PORT</code> and <code>DISABLE_TRANSACTION_BROADCAST</code>. Get those two right and the rest of Part 6 lands cleanly.
</div>

### 6.1.12 Quiz

<!-- ── Chapter 6.1 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Chapter 6.1 Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="A">
<p><strong>Q1.</strong> Which CLI is the QAA canonical CLI?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>apps/cli</code> (<code>@ledgerhq/live-cli</code>) — Node + Commander</button>
<button class="quiz-choice" data-value="B">B) <code>apps/wallet-cli</code> (<code>@ledgerhq/wallet-cli</code>) — Bun + Bunli + DMK</button>
<button class="quiz-choice" data-value="C">C) Both, depending on the test</button>
<button class="quiz-choice" data-value="D">D) Neither — QAA uses a programmatic API instead</button>
</div>
<p class="quiz-explanation"><code>apps/cli</code> is canonical and is what every E2E helper in <code>cliCommandsUtils.ts</code> spawns. <code>apps/wallet-cli</code> is experimental and out of scope for QAA.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> Why does QAA need a CLI surface in addition to Playwright (desktop) and Detox (mobile)?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The UI drivers cannot launch Ledger Live</button>
<button class="quiz-choice" data-value="B">B) Playwright and Detox are too slow for any test</button>
<button class="quiz-choice" data-value="C">C) The CLI handles test-data infrastructure (seeding accounts, reading allowances, signing one-shot approve / revoke tx) more cheaply and deterministically than UI drivers</button>
<button class="quiz-choice" data-value="D">D) The CLI is required to compile the desktop app</button>
</div>
<p class="quiz-explanation">UI drivers test what users do; the CLI prepares the world they do it in. Allowance state determinism (Chapter 6.2) is the headline reason a CLI revoke hook exists.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q3.</strong> The CLI is invoked from tests via what mechanism?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Direct TypeScript import of <code>send</code> / <code>tokenAllowance</code> from <code>@ledgerhq/live-cli</code></button>
<button class="quiz-choice" data-value="B">B) <code>child_process.spawn("node", [LEDGER_LIVE_CLI_BIN, ...args])</code> in <code>runCli.ts</code></button>
<button class="quiz-choice" data-value="C">C) HTTP request to a running CLI daemon</button>
<button class="quiz-choice" data-value="D">D) WebSocket bridge from Detox</button>
</div>
<p class="quiz-explanation">The CLI is a Node subprocess, not a library. <code>runCli.ts</code> spawns <code>node apps/cli/bin/index.js</code>, captures stdout, and reports exit codes. Every call pays Node startup cost.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> Why does <code>runCliCommand</code> join args with <code>+</code> and split them inside the spawn call?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Commander requires <code>+</code>-separated args</button>
<button class="quiz-choice" data-value="B">B) It enables Bash globbing of arg lists</button>
<button class="quiz-choice" data-value="C">C) It is required by Speculos's REST API</button>
<button class="quiz-choice" data-value="D">D) To avoid shell-escape pain when forwarding args across processes; <code>+</code> never appears in real CLI arg values</button>
</div>
<p class="quiz-explanation">The <code>+</code> separator is a convention. The engine splits on <code>+</code> before passing argv to <code>spawn</code>, which itself does not invoke a shell — the trick is purely about safely composing the arg string in a single TypeScript variable.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q5.</strong> What does <code>LEDGER_LIVE_CLI_BIN</code> resolve to?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) An absolute path to <code>apps/cli/bin/index.js</code> computed via <code>path.resolve(__dirname, "../../../../apps/cli/bin/index.js")</code></button>
<button class="quiz-choice" data-value="B">B) The string <code>"ledger-live"</code> resolved via <code>$PATH</code></button>
<button class="quiz-choice" data-value="C">C) An npm package URL</button>
<button class="quiz-choice" data-value="D">D) The path to <code>apps/wallet-cli/bin/index.js</code></button>
</div>
<p class="quiz-explanation">The constant in <code>runCli.ts</code> is a relative-path resolution from the compiled location of <code>runCli.ts</code> back up to the monorepo root and into <code>apps/cli/bin/index.js</code>. It does not depend on <code>$PATH</code> and it does not use the <code>ledger-live</code> bin alias.</p>
</div>

<div class="quiz-score"></div>
</div>

---


## Approvals and revokes — the blockchain primer

<div class="chapter-intro">
Before we walk the five layers of CLI integration, you need a mental model of what an ERC-20 approval actually <em>is</em> on chain — and why the state it creates is the QA problem the revoke hook exists to solve. This chapter is a focused primer: ERC-20 storage, the <code>approve</code> selector, calldata layout, the unlimited-approval footgun, and how the Ledger device clear-signs the operation.
</div>

### 6.2.1 What you'll learn

You will learn how ERC-20 token contracts store balances and allowances; what the `approve(address,uint256)` function does at the calldata level; why allowances persist across test runs and break determinism; and how `revoke` is the same function with `amount = 0`. Forty minutes of theory, then we wire it into the CLI in Chapter 6.3 onward. If you already know ERC-20 inside out, skim 6.2.6 (the QA framing of why the state matters) and 6.2.9 (clear-sign behaviour on the device) — those are the QA-specific pieces that connect the blockchain primer to the rest of Part 6.

### 6.2.2 ERC-20 storage model

An ERC-20 token contract holds two pieces of state, both indexed by Ethereum address:

```
USDC contract @ 0xA0b8...eB48:
  balances: {
    0xYou:   100_000_000   // 100 USDC (6 decimals)
    0xAlice:  50_000_000   // 50 USDC
    ...
  }
  allowances: {
    0xYou: {
      0xUniswap: 0
      0xLifi:    0
      0xRouter:  100_000_000   // you've allowed Router to spend 100 USDC of yours
    }
    0xAlice: { ... }
  }
```

`balances[addr]` answers "how much do I own?". `allowances[owner][spender]` answers "how much of `owner`'s balance is `spender` allowed to move on `owner`'s behalf?". Both are persistent on-chain state. Both survive every off-chain restart you can think of: simulator restarts, repo re-clones, CI runner recycles, your laptop reboot.

The only way to change `allowances[you][spender]` is to send a transaction that calls a function on the token contract — almost always `approve(spender, newAmount)`.

Two storage details matter for QA work. First, `balances[addr]` and `allowances[owner][spender]` are *separate* mappings; spending allowance does not move balance. A spender contract calls `transferFrom(owner, recipient, amount)`, which atomically decreases `allowances[owner][spender]` by `amount` and decreases `balances[owner]` by `amount` (transferring it to `recipient`). A revoke does not refund balance — it only zeroes the delegation. Second, every token contract is its own state silo: USDC's allowances map is independent of USDT's, of DAI's, of every other ERC-20. Revoking USDC does nothing for USDT. Tests that touch multiple tokens need a revoke per token they touched.

A worked illustration of the two-mapping separation:

```
Before:
  USDC.balances[Alice]            = 1000
  USDC.allowances[Alice][Router]  = 100

Alice signs and sends:
  Router.swapExactTokensForETH(100, ...) on Router contract

Inside Router.swap(...):
  Router calls USDC.transferFrom(Alice, Router, 100)
  USDC contract atomically:
    balances[Alice]            -= 100   → 900
    balances[Router]           += 100   → +100
    allowances[Alice][Router]  -= 100   → 0
  Router then routes the 100 USDC through pools, eventually
  sending ETH to Alice.

After:
  USDC.balances[Alice]            = 900
  USDC.allowances[Alice][Router]  = 0   (consumed exactly)
  Alice received some ETH (off the USDC contract)
```

In this example the swap consumed the allowance exactly, so the post-state allowance is already zero — no explicit revoke needed. In practice, most swaps approve a slippage-padded amount (the `× 1.2` in `ensureTokenApproval`), the swap consumes only the actual amount, and a small leftover allowance remains. That leftover is benign but persistent — the next test sees a non-zero allowance and may take a different on-device path. Hence the revoke hook.

### 6.2.3 Why allowances exist

ERC-20 was finalised as EIP-20 in late 2015. The original design separated two concerns:

1. **Ownership.** Only the address that owns tokens can move them.
2. **Delegation.** A protocol (a DEX, a lending market, a staking contract) needs to move user tokens during a swap or deposit, but the user does not want to send tokens to the protocol first and then trust it to send them back.

Allowances are the delegation primitive. The user calls `approve(spender, N)` once, and afterwards the spender contract can call `transferFrom(user, anywhere, ≤N)` without the user signing each individual movement. The user remains the owner; the spender is a bounded delegate.

This is why every ERC-20 swap, every Aave deposit, every Compound supply, every staking deposit starts with an approval transaction. It's also why "infinite approvals" became the industry default — and why they're a security footgun (6.2.8).

The flow for a typical swap, from the chain's point of view:

```
Tx 1:  user --approve(Router, 100)--> USDC contract
       (sets allowances[user][Router] = 100; gas: ~46k)

Tx 2:  user --swap(usdc=100, ...)--> Router
       Inside Router's logic:
         Router --transferFrom(user, Router, 100)--> USDC contract
         (atomically: allowances[user][Router] -= 100, balances[user] -= 100,
          balances[Router] += 100)
         …Router then routes those 100 USDC through one or more pools,
         eventually delivering output tokens back to the user…
```

Two transactions, two signatures. The approval is the user saying "yes, this Router can take up to 100 USDC of mine"; the swap is what the Router does with that permission. After the swap completes, `allowances[user][Router]` is back to 0 (the swap consumed exactly the approved amount). If the user had infinite-approved, the allowance would still be near-infinite afterwards — which is convenient, and dangerous.

### 6.2.4 The approve operation

`approve(address spender, uint256 amount)` is a standard ERC-20 function. Its 4-byte selector is the first four bytes of the keccak-256 hash of the function signature:

```
keccak256("approve(address,uint256)")[0:4] = 0x095ea7b3
```

Calldata for `approve(0xRouter, 100)` is sixty-eight bytes:

```
0x095ea7b3                                                          // 4-byte selector
  000000000000000000000000<20-byte-spender-address>                 // 32 bytes: spender, left-padded
  000000000000000000000000000000000000000000000000000000000000_0064 // 32 bytes: amount = 100, left-padded
```

Concretely, an approve-100-USDC tx to Router `0x1111...2222` looks like:

```
0x095ea7b3
00000000000000000000000011111111111111111111111111111111_22222222
0000000000000000000000000000000000000000000000000000000000000064
```

(Whitespace and `_` added for readability; the on-chain bytes are contiguous.)

The Ledger device, when displaying this transaction, recognises the selector `0x095ea7b3` and decodes the two arguments. Instead of the generic "Contract Data — Are you sure?" warning that unrecognised contract calls trigger, the device renders a clear-sign screen: "Approve <token>", amount, spender. We come back to this in 6.2.9.

For QAA work, the calldata is built by `getErc20ApproveData(spender, amount)` in the EVM family of `libs/coin-modules/coin-evm`. You do not construct the calldata yourself — the CLI does it for you when you pass `--mode approve --token X --spender Y --approveAmount N`. Knowing the layout matters only when you're debugging: if a test broadcasts a tx and Etherscan decodes the input data unexpectedly, walk the bytes against this table to confirm the selector is `0x095ea7b3` and the two 32-byte words match the expected spender and amount. A mismatch usually means the test passed the wrong `--token` or `--spender` flag and the CLI built calldata against the wrong contract or recipient.

A worked decode of a real revoke calldata blob:

```
0x095ea7b3                                                         ← approve selector
000000000000000000000000a73c628eaf6e283e26a7b1f8001cf186aa4c0e8e   ← spender (the Router)
0000000000000000000000000000000000000000000000000000000000000000   ← amount = 0 (revoke)
```

That's all 68 bytes. The transaction's `to` field is the token contract (e.g. USDC's `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` on mainnet), the `value` is `0` (you're not sending ETH), and the `data` field is the 68 bytes above. The signed transaction wraps these in the standard Ethereum tx envelope — nonce, gas params, signature — but the *intent* is fully captured in the calldata.

If you ever need to hand-verify what `apps/cli` produced, an Etherscan tx-input decoder or a quick Python `eth_abi.decode(['address','uint256'], data[4:])` will give you back the spender and amount. This is how Chapter 6.8 verifies that the senior's revoke commit produces the right calldata for the right spender.

A short note on EVM endianness: all `uint256` values, including `amount`, are encoded as **big-endian** 32-byte words. The decimal value `100` becomes `0x...0064` at the rightmost end of the 32-byte slot, padded with zeros on the left. The decimal value `2^256 − 1` becomes `0xFFFF...FFFF` (all 32 bytes set). Do not get confused if you eyeball calldata and see "lots of leading zeros, then a small number at the end" — that is the canonical encoding, not a bug. The selector itself (`0x095ea7b3`) is the only piece that's not 32-byte-aligned; selectors are 4 bytes by design, and Solidity dispatch logic reads exactly the first 4 bytes of calldata.

Token decimals also matter when reading amounts. USDC has 6 decimals: `100_000_000` in the calldata means 100 USDC. ETH and most tokens have 18 decimals: `100000000000000000000` means 100 tokens. If a test logs an amount as raw smallest-units, multiplying by `10^decimals` in your head saves a lot of "wait, did we approve 100 USDC or 100 micro-USDC?" confusion.

### 6.2.5 Three operations, one function

| Op | Calldata | On-chain effect |
|---|---|---|
| Approve N | `0x095ea7b3 + spender(32) + N(32)` | `allowances[you][spender] = N` |
| Revoke | `0x095ea7b3 + spender(32) + 0(32)` | `allowances[you][spender] = 0` |
| Unlimited | `0x095ea7b3 + spender(32) + (2^256 − 1)(32)` | `allowances[you][spender] = 2^256 − 1` |

All three are the same function. The only difference is the amount. There is no separate `revoke()` function on standard ERC-20 — revoke is a convention that means "approve zero".

Some newer tokens implement `increaseAllowance` and `decreaseAllowance` to mitigate a 2017-era race condition (the "approve front-running" bug), but those are extensions; the core ERC-20 spec only has `approve`.

The race condition: if `allowances[user][spender] = 100` and the user wants to change it to `50`, the spender can race the new approval — `transferFrom` 100 first, then receive the new approval of 50, then `transferFrom` another 50, for a total of 150 against the user's stated intent. The mitigation that wallets and dApps adopted: revoke first (set to 0), then re-approve to the new value. Two transactions, two signatures, no race window. This is why most "change allowance" UIs do the two-step dance even today.

For QAA, the practical implication is that a clean test pre-state is *zero allowance*, not "the right allowance". The revoke hook gets you to a known floor; the test then sets up the exact allowance it wants. That's structurally simpler than trying to detect "is the current allowance the one I want?" and conditionally adjusting it.

The race condition also clarifies why "approve, do the thing, revoke" is the safe full-cycle test pattern: every step has a deterministic on-chain effect, and the post-state matches the pre-state (zero), so the test is fully reversible. Compare with "increase by N, do the thing, decrease by N": same in-flight semantics, but it relies on `increaseAllowance` / `decreaseAllowance` being implemented (not all tokens do) and on no other test having mutated the slot in between.

**Why no separate revoke ABI?** Because adding a function to a deployed token contract is impossible without redeploying. ERC-20 was finalised in late 2015; tokens deployed since then ship with whatever ABI they had at deploy time. A new function in the standard would only apply to newly-deployed tokens. Convention won out: every wallet, every dApp, every block explorer agreed that "revoke" means "approve zero", and that convention is now load-bearing. The same logic applies to anything else you might wish ERC-20 had — there's no way to add it retroactively to existing tokens, so the ecosystem either lives without it or layers external infrastructure (off-chain indexers, helper contracts) on top.

This convention does have a gotcha. Some token contracts implement non-standard `approve` semantics — refusing to allow a non-zero approve when the current allowance is non-zero (USDT historically did this). For those tokens, the only path to change a non-zero allowance is "approve zero, then approve the new amount" — two transactions. QAA tests that touch such tokens need the revoke step woven into normal flow, not just as an end-of-test cleanup.

### 6.2.6 Why allowance state is the QA problem

Allowance state lives on the token contract on-chain. It is not in your repo. It is not in `~/.ledger-live`. It is not in the Speculos Docker volume. It is on the chain.

That means:

- A run that broadcasts `approve(Router, 100)` permanently leaves `allowances[seed][Router] = 100`.
- The next run starts with `allowances[seed][Router] = 100` already in place, regardless of how clean your local environment is.
- A "fresh approval" device screen now reads "you already approved 100 — approve 50 more?" — different copy, different screen count, different hold-to-sign timing.
- Any test that asserts the device shows "Approve 50 USDC" against the previous state's "Approve 50 more USDC" fails on a string mismatch.

**Worked example.** Nightly run #1 issues `approve(Router, 100)` against the QAA test seed at 02:00. The test passes. The on-chain allowance is now 100. Nightly run #2 starts at 02:00 the next day. The test fixture imports the same seed, opens the same swap flow, taps "Allow", and the device — which queries the token contract via the network plugin — reads the existing 100 and renders an "increase allowance" flow instead of an "initial approval" flow. The test driver waits for the "Initial approval — Hold to sign" label, never sees it, times out, fails.

Nothing in the test driver, the simulator, or the harness changed between run #1 and run #2. Only the chain state did. Without a revoke hook between iterations, every nightly run after the first is a different test from the first one.

That is the QA problem. Manual revoke tools (Chapter 6.3) are the human fallback. The CLI revoke hook (the senior's commit, walked in Chapter 6.8) is the automated solution.

Three more failure modes worth naming, because each one looks like a different bug but has the same root cause:

- **"Test passes locally, fails in CI."** The CI runner uses a shared seed that nightlies have already touched; your laptop uses a freshly-revoked seed because you ran revoke.cash an hour ago. Same code, different chain state, different test outcome.
- **"Test passes the first time, fails on the second."** First run leaves an allowance behind; second run sees the increase-allowance flow instead of the fresh-approval flow.
- **"Test passes for a week, then starts failing for no reason."** Some other test in the same suite started broadcasting (perhaps a teammate flipped `DISABLE_TRANSACTION_BROADCAST` to `0` for a debugging session and forgot the `finally`), accumulated allowance state on the shared seed, and now your assertion against fresh-approval copy breaks.

All three are the same problem in different costumes. The cure is the same: revoke between iterations, on a known-clean seed, with discipline around the broadcast env (Chapter 6.6).

**The seed matters.** QAA's test seed is shared infrastructure. Multiple developers, multiple CI runs, multiple debugging sessions touch the same address space on the same testnets. The chain doesn't care that "this allowance came from Alice's local debugging at 3pm and should be irrelevant to Bob's nightly at 2am" — they're the same `(owner, spender, token)` triple as far as the storage slot is concerned. Hygiene around the test seed is a team-level discipline, not just a per-test concern. Chapter 6.3 covers the manual cleanup tools because human-driven hygiene is a real part of the workflow, not just an emergency fallback.

A useful comparison: think of the test seed like a shared development database. You wouldn't expect tests to run reliably on a database that other people are mutating concurrently and that doesn't get cleaned between runs. The seed-on-chain situation is structurally identical, except (a) you can't `TRUNCATE TABLE allowances` to reset, you have to issue revoke transactions one by one, and (b) the cleanup costs gas. Hence the importance of making the cleanup automatic via the CLI hook, and of managing broadcast-enabled tests against Ethereum mainnet with real funds.

**The opposite of "broadcast" is not "do nothing" — it's "sign and discard".** The CLI still runs through the full sign flow when broadcast is disabled: it prompts the device, walks the screens, builds an APDU, gets a signature back, and then does *not* push the signed bytes to chain. This is the right behaviour for tests that want to assert "the device shows the right approval screen" without touching state — the signing exercise is real, but the on-chain consequence is absent. The wrappers around `approveTokenCommand` and `revokeTokenCommand` flip this exact behaviour: they need broadcast on, because the *point* of the call is the on-chain mutation, not the screen flow assertion.

A subtler point about the `Approval` event: every `approve(spender, N)` emits `Approval(owner, spender, N)`. Indexers and block explorers consume this event to display "current allowances per token" views. revoke.cash, for example, builds its UI by walking `Approval` events for a given owner and computing the current state per `(token, spender)` pair. That is why a revoke that doesn't actually change state still emits the event, and why you can verify a revoke landed by looking for the `Approval(_, _, 0)` log on the broadcast tx receipt. If the receipt has the event but the on-chain `allowance(owner, spender)` query returns non-zero, you've found a divergence — most likely between the broadcast you observed and a later approve that crossed it on the wire. This rarely happens in QA testing because tests run sequentially per worker, but it's a useful debugging concept.

The cross-check is also useful when verifying that a CLI revoke actually landed in CI. The CLI returns "broadcast successful" when the RPC accepts the tx — that doesn't mean it's mined. A test that needs the post-revoke state to be observable should `--wait-confirmation` (which polls until the tx is included in a block), and afterwards a `tokenAllowance` read can confirm the slot is zero. Three sources of truth: the wait-confirmation succeeded, the tx receipt has the `Approval(_, _, 0)` event, and the `allowance(owner, spender)` query returns zero. All three agreeing is the definition of "the revoke landed".

### 6.2.7 Revoke = approve(spender, 0)

Mechanically, a revoke is identical to an approve with `amount = 0`. Same selector, same calldata layout, same gas profile, same APDU shape on the Ledger device, same on-chain receipt format. There is no distinct "revoke" event in the ERC-20 standard; the `Approval(owner, spender, 0)` event a revoke emits is the same shape as the event an "approve zero" emits.

The Ledger device's clear-sign decoder is what makes the user-facing experience differ. When the firmware's ERC-20 plugin recognises `amount == 0`, it may render the screen as "Revoke <token>" instead of "Approve <token>, Amount: 0". The exact wording depends on firmware version and device family.

> **Verify:** the exact device-screen text on Stax / Flex / Nano S+ for `amount == 0` (e.g., "Revoke USDC" vs "Approve USDC — Amount: 0"). Confirm by running `apps/cli` in `--mode revokeApproval` against a Speculos Ethereum device and reading <code>fetchCurrentScreenTexts</code> output. The CLI integration tests use whichever literal label `DeviceLabels.HOLD_TO_SIGN` resolves to; the QAA E2E layer never needs to assert the visual revoke vs approve-zero wording explicitly.

The on-chain receipt is indistinguishable from any other allowance change. A block explorer like Etherscan typically labels both `approve(spender, 0)` and `approve(spender, N)` as "Approve" in its decoded function-call view, with the amount column reading 0 for revokes.

Practically, this means three things for QAA work:

1. The CLI's `--mode revokeApproval` produces the same calldata shape as `--mode approve` with `amount = 0`. There is no separate "revoke" code path inside `apps/cli`; both modes route through the same EVM transaction builder, just with different amounts.
2. The Speculos device-tap helper `approveToken()` works for both modes without modification. Same screen sequence, same hold-to-sign, same final APDU.
3. The on-chain receipt for a revoke is fungible with any zero-amount approve: a tx that sets allowance to zero by any means produces the same `Approval(owner, spender, 0)` event. This is why Etherscan's "Token Approvals" view treats both interchangeably and why revoke.cash's "is this revoked?" check is a simple "is the current allowance zero?" query.

A subtle behaviour: the EVM allows zero-amount approvals to skip the actual SSTORE if the slot is already zero. Some token contracts implement this optimisation (writing to a slot that's already `0` costs gas; emitting a no-op event might be skipped). Most don't. From the test's point of view, calling `approve(spender, 0)` against an already-zero slot is a no-op — it broadcasts, costs gas, emits an `Approval(owner, spender, 0)` event, and the post-state is still zero. Idempotent in the on-chain sense.

This idempotence is why `revokeTokenApproval` has no pre-check the way `ensureTokenApproval` does. `ensureTokenApproval` checks "is the allowance already at least N?" and skips if yes (saves a device dance and a broadcast). `revokeTokenApproval` just always revokes — the cost is small, the result is deterministic, and the conditional logic to skip would itself be a possible source of bugs. "Always revoke" is simpler and correct.

### 6.2.8 Unlimited approval — the security footgun

`amount = 2^256 − 1` (the max `uint256` value, often displayed as `0xFFFF...FFFF`) is the "infinite approval" pattern. dApps adopted it for two reasons:

- **Gas savings.** One `approve(spender, ∞)` is cheaper than approving each swap individually.
- **UX.** The user signs one approval, then every subsequent swap is a single signature instead of two (approve + swap).

The cost: if the spender contract is ever compromised — by an upgrade, an admin-key takeover, a logic bug, a governance attack — the attacker can `transferFrom(victim, attacker, victim_balance)` for every victim that infinite-approved that contract. This was the vector in the BadgerDAO front-end compromise of December 2021 (~$120M drained), in multiple 2022 phishing campaigns where users were tricked into infinite-approving drainer contracts, and in the rolling pattern of "approval phishing" that revoke.cash exists to clean up.

The industry response: wallet UI warnings on infinite approvals, the rise of EIP-2612 `permit` (signature-based approvals with no on-chain allowance write), and revoke.cash itself.

EIP-2612 `permit` is worth a sentence because it occasionally appears in test scenarios. It lets a token holder sign an off-chain message authorising a spender for a specific amount with a specific deadline; the spender contract then submits both the user's `permit` signature and the swap call in a single transaction. Net effect: the user signs once instead of twice, and there's no persistent on-chain allowance to clean up afterwards. From a QAA determinism perspective, `permit`-based flows are easier — they leave no allowance state behind. But not all tokens implement `permit`, and Ledger's clear-sign for `permit` is a separate device flow with its own labels. If a test exercises a permit flow, the device-tap helper needs a different label set and `revokeTokenApproval` is irrelevant (no allowance to revoke). For now, the QAA mainstream is still classic `approve` + revoke.

The pragmatic posture for tests: assume classic `approve` until proven otherwise; if a token is `permit`-capable and the dApp under test uses `permit`, design the test around the no-allowance semantics; if a test mixes both (some flows use `permit`, some don't), keep the revoke hook for the classic-approve flows and skip it for `permit` flows. A single test should not change which flow it uses run-to-run; deterministic test design picks one and sticks with it.

QAA tests almost never use unlimited approval. The `ensureTokenApproval` helper passes `minAmount × 1.2` (a 20% slippage buffer over the swap amount), not `2^256 − 1`. We use exact, finite amounts so the device renders the same screen flow every run, and so the revoke hook can cleanly zero the slot when the test ends.

There is one place QAA might legitimately want to test infinite approval: a regression test for the device's "infinite-approval warning" screen. If product behaviour requires a different on-device warning when the amount is `2^256 − 1`, that path needs explicit coverage. But that's a single-purpose test, not the default. Every other ERC-20 swap / earn / delegate test uses finite amounts.

> **Verify:** the exact threshold above which Ledger firmware considers an amount "effectively infinite" and shows a warning screen. Older firmware checks for the literal `2^256 − 1`; newer firmware may use a heuristic threshold (e.g. amount > 10^28). If a test needs to assert "the warning appears", confirm with current firmware behaviour first.

**A subtle gas point.** A revoke transaction costs gas. On Ethereum mainnet that's anywhere from a few cents (when the network is quiet) to several dollars (when it's congested). On testnets it's effectively free. QAA tests run on Ethereum mainnet with real funds. The cumulative cost of revoke hooks across many test runs is a real budget item — the team weighs "broadcast disabled" coverage against "fully on-chain" coverage explicitly. For day-to-day work, "always revoke after every test that approved" is the right default.

### 6.2.9 Clear-sign behaviour on the Ledger device

When the Ledger device receives an Ethereum transaction whose `to` field is an ERC-20 contract and whose calldata starts with `0x095ea7b3`, the firmware's ERC-20 plugin decodes the calldata into structured fields:

- The token symbol (resolved from the `to` address via the device's bundled token registry, or via a "trusted name" plugin response signed by Ledger's services).
- The spender address.
- The amount, formatted with the token's decimals.

Instead of the fallback "Contract Data — Are you sure?" warning that displays for unrecognised contract calls, the device shows a sequence like:

```
Review transaction
Approve <token>
Amount: <human-readable>
Spender: <0xabc...def>
[Hold to sign]
```

This is the screen sequence `families/evm.ts:approveTokenTouchDevices` walks via `pressUntilTextFound(DeviceLabels.HOLD_TO_SIGN)` and `longPressAndRelease(DeviceLabels.HOLD_TO_SIGN, 3)`. The button-device variant uses `pressUntilTextFound(DeviceLabels.SIGN_TRANSACTION)` and a both-buttons press to confirm.

Two things matter for QAA. First, the device-tap automation is keyed off the *labels* (`DeviceLabels.*` constants), not absolute screen positions, so it survives minor copy changes. Second, both `--mode approve` and `--mode revokeApproval` go through the *same* clear-sign path on the device — only the amount field differs. The Speculos device-tap helper does not need to branch on mode; it just walks "Review → press until Hold to sign → hold". Forward-ref Chapters 6.4 and 6.5.

If clear-sign is *not* available — for an unrecognised token contract, an unrecognised chain, or a firmware that lacks the ERC-20 plugin — the device falls back to the generic "Contract Data — Are you sure?" warning. That warning *can* be signed (with a "I accept the risk" toggle in the firmware's settings on some device families), but the test would need to drive a different label sequence. QAA tests by default run on chains and tokens where clear-sign is reliable; if you add a new chain to the matrix and the device falls back to generic-contract-data, expect the device-tap helper to need a new label set.

Two more subtleties of clear-sign worth noting:

- **Token recognition is data-driven.** The firmware ships a registry of known token contracts; for tokens outside that registry, Ledger's "trusted name" service can sign a JSON descriptor at the wallet level that the firmware verifies before rendering. If neither path resolves, clear-sign degrades to "Approve <unknown token at 0x...>" — which is still better than the fully generic warning but worse than a clean named display.
- **Spender display.** The device shows the spender as a hex address by default. Some firmware versions can resolve known spender addresses (e.g. major DEX routers) to human-readable names via the same trusted-name infrastructure. If your test asserts on the spender display, know whether the spender for the chain / version under test resolves to a name or to a raw hex.

For most QAA work on mainstream EVM chains and well-known tokens, clear-sign just works. The edge cases above are worth knowing exist; they're rarely the thing that breaks a test on a normal day.

> **Verify:** older firmware versions or non-ERC-20 token standards (ERC-721, ERC-1155, BEP-20 on BSC) may render different labels. The QAA-canonical path is ERC-20 on EVM L1s with current production firmware. If a test introduces a new chain or asset class, re-verify the device labels with <code>fetchCurrentScreenTexts</code> on a real Speculos run before relying on <code>approveToken()</code>.

An aside on the touch-device flow specifically: on Stax and Flex, the "Hold to sign" gesture is a three-second hold on a specific UI region, not a button press. The Speculos device-tap helper simulates this with `longPressAndRelease(DeviceLabels.HOLD_TO_SIGN, 3)`. If the user releases early — or the helper sleeps less than three seconds — the firmware aborts the signature and the CLI subprocess waits forever (or until its `--wait-confirmation-timeout` fires). When a test hangs at the device-tap stage on touch devices, "did the long-press actually hit three seconds?" is the first thing to check.

### 6.2.10 The QA bridge — why the QAA-615 hook exists

Putting the pieces together:

1. Allowances are persistent on-chain state (6.2.6).
2. Regression tests that broadcast leave allowances behind every run (6.2.6).
3. Revoke is `approve(spender, 0)` — same selector, same shape, no special primitive (6.2.7).
4. The Ledger device clear-signs both approve and revoke through the same screen flow (6.2.9).
5. Therefore: a CLI hook that issues `send --mode revokeApproval` between test iterations cleanly resets state, and the existing Speculos device-tap helper (`approveToken()` in `families/evm.ts`) drives the device side without modification.

That is exactly what the senior's QAA-615 commit on `support/qaa-615-add-revoke-token` adds: `revokeTokenCommand` in `cliCommandsUtils.ts` (the typed wrapper), `revokeTokenApproval` on `swap.page.ts` (the POM method), and the swap spec switching from `ensureTokenApproval` to `revokeTokenApproval` for the dev loop. Chapters 6.3 through 6.8 unpack each layer.

The manual alternative — revoke.cash, Etherscan's Write tab — is what humans use when the harness isn't available. Same on-chain effect, different UX, same primitive: `approve(spender, 0)`.

**Why `revokeTokenApproval` is a POM method and not a CLI helper.** The senior's commit puts `revokeTokenApproval` on `swap.page.ts`, not in `cliCommandsUtils.ts`. The CLI wrapper layer (`revokeTokenCommand`) does the typed CLI call; the POM method (`revokeTokenApproval`) does the swap-test-specific orchestration: snapshot the Speculos port, swap to an Ethereum-app device, run the revoke, restore the port, attach the result to the Allure report. That separation matters: the CLI wrapper is reusable across any spec that needs to revoke an allowance; the POM method is specific to "I am in the middle of a swap test and need to inject a revoke at this point". Chapter 6.4 walks both. If a future feature needs revokes outside the swap context — say, a test for the new "Manage approvals" UI in Ledger Live — it would import `revokeTokenCommand` directly and write its own POM method, without touching the swap-page version.

**Connection to the rest of Part 6.** Everything from here builds on the picture in 6.2.5: same selector, three amounts, one function. Chapter 6.3 covers the manual tools that read and write that slot for human cleanup. Chapter 6.4 walks how the CLI wrapper builds the calldata for you. Chapter 6.5 covers the Speculos lifecycle that makes the device-side signing possible. Chapter 6.6 is about choosing whether to actually mutate chain state. Chapter 6.7 is the daily workflow. Chapter 6.8 is the senior's QAA-615 commit annotated line by line. By the end of Part 6 you should be able to read the senior's `revokeTokenApproval` method, point at every line, name the layer it touches, and explain why each choice was made.

**One more framing.** The blockchain primer in this chapter is the *what*. The five-layer integration in Chapter 6.4 is the *how*. The footguns in Chapter 6.6 are the *what could go wrong*. Hold these three together as you go: every line of code in Part 6 is doing one of three things — describing what to do on chain, wiring how to do it through the test harness, or guarding against a known footgun. If you can map a piece of code to one of those three categories you understand it well enough to maintain it.

A final pre-Chapter-6.3 checklist. Before moving on, you should be able to:

- Name the four QAA-canonical CLI commands (6.1.6).
- Draw the five-layer integration diagram from memory (6.1.7).
- Explain why a CLI exists in addition to UI drivers (6.1.2).
- State the `approve` selector and the calldata layout from memory (6.2.4 / 6.2.5).
- Give one sentence on why allowance state breaks test determinism (6.2.6).
- Name the two pieces of global state the CLI workflow depends on: `SPECULOS_API_PORT` and `DISABLE_TRANSACTION_BROADCAST` (6.1.8).

If any of those don't come easily, re-read the relevant subsection. The rest of Part 6 will be friendlier when these are second-nature.

**A small reading order recommendation.** If you came to this guide as a backend engineer who's never written E2E tests, read Chapters 6.3 (manual tools, hands-on UX), then 6.7 (daily workflow, more hands-on UX), then 6.4 (the layered architecture). The hands-on chapters give you a concrete picture before the layered architecture asks you to hold five abstractions at once. If you came as a UI-test engineer who's never touched ERC-20, read Chapter 6.2 carefully (this one), then 6.4 to see how it's wired into the harness, then 6.6 to understand the safety controls. Both paths converge at Chapter 6.8.

**Glossary anchors for what's coming.** Quick definitions you'll see across the next chapters:

- **Speculos** — the Ledger device emulator, runs as a Docker container, exposes a REST API the test harness drives.
- **APDU** — Application Protocol Data Unit, the binary message format the host uses to talk to the (real or emulated) Ledger device.
- **Bridge** (in `bridge.signOperation`, `bridge.broadcast`) — the per-currency adapter inside live-common that knows how to build, sign, and broadcast transactions for that currency. Not to be confused with the WebSocket bridge in mobile E2E (Part 5).
- **POM** — Page Object Model. The convention where each app screen is an object with methods that perform the actions a test needs.
- **Allure** — the test reporting framework. The `allure.description(...)` calls you'll see attach human-readable context to test results.
- **Userdata** — the JSON file format Ledger Live consumes at startup to pre-populate accounts and settings. `liveData` produces it; the desktop fixture engine points the app at it via `--userdata <path>`.

These names recur. Knowing what they mean now means the next chapters read at full speed.

**Where Chapter 6.3 picks up.** The next chapter walks the manual revoke tools — revoke.cash, Etherscan's Read and Write tabs — that humans use when the harness is unavailable, when something has gone wrong, or when you want to audit chain state directly. The mechanics are the same as everything in this chapter: same `approve(spender, 0)` selector, same on-chain effect, same `Approval(_, _, 0)` event. The difference is the UX: a human in a browser instead of a CLI subprocess. Knowing both is essential, because the manual tools are the ground truth you fall back to when the automation lies.

<div class="chapter-outro">
<strong>Key takeaway:</strong> An ERC-20 allowance is one slot in the token contract's nested mapping <code>allowances[owner][spender]</code>. Approve sets it; revoke sets it to zero; both are calls to the same <code>approve(address,uint256)</code> function with selector <code>0x095ea7b3</code>. Allowance state is on-chain and persistent — and that persistence is the determinism problem the CLI revoke hook (Chapter 6.8) exists to solve. Up next, Chapter 6.3 covers the manual revoke tools you reach for when the harness isn't available.
</div>

### 6.2.11 Quiz

<!-- ── Chapter 6.2 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Chapter 6.2 Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> What is the function selector <code>0x095ea7b3</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The selector for <code>transfer(address,uint256)</code></button>
<button class="quiz-choice" data-value="B">B) The selector for <code>approve(address,uint256)</code> — <code>keccak256("approve(address,uint256)")[0:4]</code></button>
<button class="quiz-choice" data-value="C">C) The selector for <code>revoke(address)</code> — a separate ERC-20 function</button>
<button class="quiz-choice" data-value="D">D) The selector for <code>permit(...)</code> from EIP-2612</button>
</div>
<p class="quiz-explanation">Standard ERC-20 has no separate revoke function. Revoke is <code>approve(spender, 0)</code> — same selector <code>0x095ea7b3</code>, same calldata shape, just <code>amount = 0</code>.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> What on-chain state does a successful revoke transaction move?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>balances[you] = 0</code> on the token contract</button>
<button class="quiz-choice" data-value="B">B) An entry in a global revoke registry contract</button>
<button class="quiz-choice" data-value="C">C) <code>allowances[you][spender] = 0</code> on the token contract</button>
<button class="quiz-choice" data-value="D">D) Nothing — revoke is purely off-chain wallet state</button>
</div>
<p class="quiz-explanation">Allowances live on the token contract, indexed by <code>(owner, spender)</code>. A revoke writes <code>0</code> into the slot for that pair. It does not touch the owner's balance and there is no global revoke registry.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> A nightly E2E run yesterday broadcast <code>approve(Router, 100)</code>. Today's run starts the same swap test. What screen will the Ledger device most likely render when the user reaches the approval step?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) An approval flow that reflects the existing 100 allowance — different copy / step count from a "fresh approval" flow, and a likely test failure if the test asserts fresh-approval text</button>
<button class="quiz-choice" data-value="B">B) The same fresh-approval flow as yesterday, because allowance state resets when Speculos restarts</button>
<button class="quiz-choice" data-value="C">C) The same fresh-approval flow as yesterday, because allowance state resets when the repo is re-cloned</button>
<button class="quiz-choice" data-value="D">D) An error screen, because the previous allowance blocks any new approve</button>
</div>
<p class="quiz-explanation">Allowance state is persistent on-chain and survives Speculos restarts and repo re-clones. The device queries the token contract for the current allowance and renders a different flow when one already exists. This is exactly the determinism problem the revoke hook solves.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> Why does QAA almost never use unlimited approval (<code>amount = 2^256 − 1</code>) in tests?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The Ledger device refuses to sign infinite approvals</button>
<button class="quiz-choice" data-value="B">B) Speculos cannot represent <code>uint256</code> max</button>
<button class="quiz-choice" data-value="C">C) Infinite approvals cost more gas than finite ones</button>
<button class="quiz-choice" data-value="D">D) Finite, exact amounts (e.g. <code>minAmount × 1.2</code>) keep device screens reproducible run-to-run and make revoke cleanup verifiable; infinite approvals are also the historical attack vector behind major DeFi drains</button>
</div>
<p class="quiz-explanation">The <code>ensureTokenApproval</code> helper passes <code>minAmount × 1.2</code> as a slippage buffer, not infinity. Determinism (same screens every run) is the testing reason; infinite-approval drainer attacks (BadgerDAO 2021 etc.) are the broader security context.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> The Ledger device's clear-sign behaviour for <code>approve(spender, N)</code> depends on…</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Nothing — every contract call shows the generic "Contract Data — Are you sure?" warning</button>
<button class="quiz-choice" data-value="B">B) The firmware recognising the selector <code>0x095ea7b3</code> and decoding the calldata into token / amount / spender fields, displaying a structured Approve / Revoke screen instead of the generic warning</button>
<button class="quiz-choice" data-value="C">C) The CLI passing a <code>--clear-sign</code> flag</button>
<button class="quiz-choice" data-value="D">D) Speculos special-casing the screen on the host side</button>
</div>
<p class="quiz-explanation">Clear-sign is a firmware feature: the ERC-20 plugin recognises selectors like <code>0x095ea7b3</code> and decodes the args into structured fields. The CLI doesn't pass a clear-sign flag and Speculos doesn't special-case it — Speculos just renders whatever the firmware outputs.</p>
</div>

<div class="quiz-score"></div>
</div>

---
## Manual Revoke Tools — revoke.cash and Etherscan

<div class="chapter-intro">
The CLI is the QAA's first tool, but it is not the only one. When local Speculos breaks, when CI revoke fails and you need on-chain truth, when you want to audit a real test seed without spinning up the harness — you reach for browser-based tools. revoke.cash and Etherscan together cover every operational need a QAA has when the automation is unavailable or untrustworthy. This chapter is short, practical, and ends with a hygiene checklist you should internalise before touching the shared QAA test seed.
</div>

### 6.3.1 Why Manual Tools Exist Alongside the CLI

The CLI hook revokes allowances during nightly runs. So why bother with manual tools at all?

Four scenarios force the question.

**1. Local Speculos broke mid-test.** You ran a swap spec by hand on a feature branch. Speculos crashed halfway through the device dance. The CLI never reached the broadcast step, but the previous test already left a non-zero allowance on-chain. Now your local seed is dirty, the Speculos container is in a bad state, and re-running the harness keeps failing on setup. You need a way to clean the on-chain state without restarting the entire e2e suite.

**2. You need to audit a real seed.** Someone hands you an Ethereum mainnet address and asks "what's this seed approving?" You don't want to write a one-off script. You want a UI that lists every active allowance in two seconds.

**3. CI revoke failed, and you want to see on-chain truth.** A nightly run failed. Allure shows the revoke step threw an error. You don't trust the test logs — they could be wrong about which spender was approved or how much. You go to the block explorer and read the allowance directly from the contract.

**4. You're verifying after the hook ran.** The CI logs say "revoke succeeded." You want to confirm the on-chain state matches the log claim before you tell the team the test seed is clean.

In each case, the manual tool is the ground-truth oracle. The CLI is part of the system under test; revoke.cash and Etherscan are not. They read the same blockchain the CLI writes to, but through a completely independent code path. That independence is what makes them useful for debugging.

### 6.3.2 revoke.cash

`revoke.cash` is the canonical manual revoke tool. Open-source, run by Rosco Kalis, hosted at `https://revoke.cash`. Source on GitHub at `RevokeCash/revoke.cash`. Multi-chain: Ethereum, Polygon, Arbitrum, Optimism, BSC, Avalanche, plus most networks including Holesky.

#### Connect modes

Three ways to point revoke.cash at an address.

**WalletConnect.** You scan a QR code with a wallet on your phone (Ledger Live, MetaMask Mobile, Rabby, etc.). The wallet exposes its current address; revoke.cash reads allowances for that address.

**MetaMask (or other browser wallet).** Click "Connect Wallet." MetaMask pops up. You pick the account. revoke.cash now sees that address.

**Paste-an-address (read-only).** The most useful mode for QAA work. Drop any address into the search bar and revoke.cash will list its allowances without any wallet connection at all. You can audit a seed without proving you own it. Obviously, with a paste-only connection, you cannot click "Revoke" — that requires a signing wallet — but the audit view is what you need 80 percent of the time.

#### What it lists

For a connected (or pasted) address, revoke.cash queries on-chain allowance state across the chains it supports and renders:

- **ERC-20 allowances** — every spender with a non-zero `allowance(you, spender)` on every token the address has ever interacted with. Each row shows token name, spender (with a label if the spender is a known router or aggregator — Uniswap, 1inch, LiFi, ParaSwap, etc.), and the current allowance amount (decoded into the token's display units, with a "unlimited" badge when the value is `2^256 - 1`).
- **ERC-721 approvals** — both per-token approvals (`approve(spender, tokenId)`) and operator-level approvals (`setApprovalForAll(operator, true)`).
- **ERC-1155 approvals** — operator approvals only (ERC-1155 doesn't have per-id approval).

Each row has a "Revoke" button. If you connected with a signing wallet, clicking it builds an `approve(spender, 0)` (or `setApprovalForAll(operator, false)` for NFTs) and asks the wallet to sign. After signing, the wallet broadcasts the tx; revoke.cash polls until the allowance reads zero, then strikes the row out.

#### Cost

Gas only. revoke.cash takes no fee, no commission, no premium. Each revoke is one ERC-20 `approve` call (~46 000 gas typical, less if the storage slot was already touched in this block) per spender per token. On mainnet at 30 gwei that's around $3 per revoke; on testnets it's free if you have faucet ETH.

#### What the UI looks like

Picture a tabular list. Each row carries a token logo, the token symbol, the spender address (often pretty-printed: "Uniswap V3 Router"), the current allowance, the chain badge, the date of the last interaction, and a red "Revoke" button on the right edge. A search bar at the top filters by token name or spender address. A chain selector at the top right switches between Ethereum mainnet, Polygon, etc. — the address stays connected; only the chain changes.

If you arrive with a paste-only connection, the "Revoke" buttons are greyed out and a banner at the top says "connect a wallet to revoke." All other inspection capabilities work.

### 6.3.3 Etherscan Read Contract Tab

`https://etherscan.io/token/<contract>#readContract`

This is the read-only inspection path. No wallet, no JavaScript wallet handshake, no risk. You arrive at the contract page on Etherscan, click the "Contract" tab, then "Read Contract." A list of every public read function the contract exposes appears.

For ERC-20 you care about `allowance(owner, spender)`. Type the owner address (your seed) and the spender address (the router you're auditing). Click "Query." Etherscan does an `eth_call` to a public node and returns the raw `uint256`.

The number is in the token's smallest unit — you must divide by `10^decimals` to get the human-readable amount. For USDC (6 decimals) a return of `1000000000` means 1000 USDC. For DAI (18 decimals) the same hex value would be a tiny fraction. Always check the `decimals()` view (right above `allowance` in most ERC-20 contracts) before you interpret the number.

This is the cheapest possible audit. No wallet connect, no transaction, no gas, no JavaScript trust assumption. You're just reading public state via Etherscan's hosted RPC.

The same UI works on every Etherscan-family explorer:

- `etherscan.io` — Ethereum mainnet
- `holesky.etherscan.io` — Holesky (reference only)
- `polygonscan.com` — Polygon
- `arbiscan.io` — Arbitrum
- `optimistic.etherscan.io` — Optimism
- `bscscan.com` — BSC
- `snowtrace.io` — Avalanche

The path is always `/token/<contract>#readContract`. Layout differs cosmetically; the function list and "Query" button are the same.

### 6.3.4 Etherscan Write Contract Tab

`https://etherscan.io/token/<contract>#writeContract`

This is the manual write path. Same contract page, different tab — "Write Contract" instead of "Read Contract." The list of public write functions appears, including `approve(spender, amount)`.

Click "Connect to Web3." MetaMask (or another supported wallet) pops up. Pick the account that holds the allowance you want to change. Etherscan shows your address as connected.

Click `approve`. Two fields: `_spender` and `_value`. Enter the spender address and `0` for the value. Click "Write." MetaMask asks for signature. Confirm. Etherscan returns a tx hash and links to the explorer view of the pending tx.

Once mined, the allowance is zero. Verify by going back to the Read tab and querying `allowance(owner, spender)`.

#### When you reach for this instead of revoke.cash

revoke.cash covers 99 percent of cases. The 1 percent where Etherscan Write wins:

- **Weird testnet tokens** that revoke.cash doesn't index (its scanner pulls from a few public APIs; obscure testnet deployments slip through).
- **Custom or vanity ERC-20s** deployed for QA-only purposes and never registered with token lists.
- **Forked or modified `approve` semantics** — some non-standard tokens have weird `approve` (e.g., revert if `from != 0 && to != 0`, the old USDT footgun). Etherscan Write lets you call any function exposed by the contract ABI, so you can work around the ones revoke.cash assumes. For example, on legacy USDT-style tokens you may need two calls: first `approve(spender, 0)`, then verify, then call `approve(spender, newValue)` if you wanted to lower without zeroing first.
- **Non-standard token contracts that expose extra revoke-like methods** (`decreaseAllowance`, `permit`, custom admin functions). Pick the right method from the function list directly.

The Write tab is also the right tool when you need to read the raw calldata Etherscan generates — you can copy it before signing and paste it into a calldata decoder to confirm the bytes match what you expect.

### 6.3.5 Other Tools

Two adjacent tools come up in QAA conversation; neither replaces revoke.cash for this workflow, but both deserve a one-paragraph mention.

**DeBank** (`debank.com`) — a portfolio aggregator that, among many other features, has an "Approvals" page per address. The list overlaps with revoke.cash. DeBank also shows DeFi positions and token balances, so QAA sometimes uses it to sanity-check what's actually held on a seed before testing a swap. It can revoke too, with a connected wallet, the same way revoke.cash does. revoke.cash is the more focused tool for this exact job.

**Zapper Smart Wallet** (`zapper.xyz`) — similar story: portfolio dashboard with an approvals view. Not actively recommended for QAA work because the allowance list is sometimes stale (cached) compared to revoke.cash's near-real-time read. Useful for one-glance overviews of a seed's overall DeFi exposure but not for clean-up sessions.

For block explorers across other chains, the pattern is identical to Etherscan:

- Ethereum mainnet → `etherscan.io`
- Polygon → `polygonscan.com`
- Arbitrum → `arbiscan.io`
- Optimism → `optimistic.etherscan.io`
- BSC → `bscscan.com`
- Avalanche → `snowtrace.io`

Same Read Contract / Write Contract tabs, same `allowance(owner, spender)` and `approve(spender, value)` functions. Once you've used Etherscan, you've used all of them.

### 6.3.6 A Worked Debug Walk-Through

Concrete picture. Nightly run on the QAA seed `0xBEEF...0001` (Ethereum mainnet). Allure shows the swap test failed at the post-condition assertion `expect(allowance).toBe(0)`. The CLI revoke step in the logs reads "exit code 0" — i.e. the subprocess succeeded.

Three possible explanations:

1. **The on-chain state is correctly zero**, and the test's post-condition reader is broken (parsing wrong, reading the wrong contract, reading the wrong owner).
2. **The on-chain state is non-zero**, the CLI subprocess succeeded but the broadcast was disabled (env var not flipped, or `--disable-broadcast` was set somewhere upstream).
3. **The on-chain state is non-zero**, the CLI broadcast happened but the transaction reverted (out of gas, contract guard, weird non-standard token).

You diagnose with two manual tabs.

**Tab 1: revoke.cash, paste-only.** Drop `0xBEEF...0001`, switch to Ethereum mainnet. If the row for the failing token/spender pair shows up with a non-zero allowance, the on-chain state is genuinely dirty — eliminate explanation 1. If the row does not show up (or shows zero), the on-chain state is clean and your test's assertion logic is the bug — that's explanation 1, look in the spec, not in the CLI.

**Tab 2: Etherscan Read.** Same address pair, query `allowance(0xBEEF...0001, <spenderAddress>)` on the token contract. Cross-check what revoke.cash showed. If they agree on non-zero, you're in explanation 2 or 3.

**Distinguishing 2 from 3.** Open the CLI log for the failing run. Look for the broadcast line — the live-common send command logs `broadcasted txHash: 0x...` when broadcast happened. If you see the line, broadcast happened; the tx is on-chain. Look it up in the explorer's transaction view: did it succeed, or did it revert? A reverted approve tx looks confusingly successful at the receipt level (status 0 vs 1 is the only difference). If status is 0, that's explanation 3 — token contract rejected the call. Time to look at the contract's `approve` source for non-standard guards.

If the `broadcasted txHash` line is missing from the log, broadcast was skipped — that's explanation 2. Search the test code for `setDisableTransactionBroadcastEnv` or `DISABLE_TRANSACTION_BROADCAST` and find where it was set to `"1"` and not restored. The next chapter (6.6) covers the discipline that prevents this.

The whole investigation takes about three minutes once you know the playbook. Without manual tools, you would be re-running the test with extra logging — a 30-minute round trip per data point.

### 6.3.7 When QAA Reaches for Manual Tools

A short list of the operational moments when you stop typing CLI commands and open a browser tab:

- **Cleanup after manual testing.** You ran a swap by hand, didn't run the cleanup hook, the test seed is dirty. Open revoke.cash, paste the seed address, click Revoke on each non-zero row. Done in a minute.
- **One-off audits.** "What's currently approved on the QAA test seed?" Paste the address into revoke.cash. Read the list. No need to write or run a script.
- **CI revoke debug.** Nightly failed at the revoke step. You go to revoke.cash, see the row still showing a non-zero allowance, confirm the CI hook genuinely didn't land. Or you see the row already at zero — meaning the CI revoke succeeded but the test's post-condition assertion is wrong.
- **Post-hook verification.** CI just finished, claims success. Five-second sanity check on revoke.cash before you announce the seed is clean.
- **Auditing a senior's branch.** Reviewing a PR that adds a new revoke flow? Run the e2e against a sandbox seed, then check on revoke.cash that the post-test state matches what the test asserted.
- **Onboarding a new test seed.** Generated a fresh QA seed for a new project? Check revoke.cash on that address right after first use to confirm the baseline is empty before any tests run.

### 6.3.8 Hygiene Rules

The QAA test seed is shared infrastructure. Multiple engineers across desktop and mobile use the same mainnet seeds for swap, send, earn, and delegate specs. Treat it like a shared dev database: clean up after yourself, never assume someone else did.

A short discipline:

- **Never connect someone else's wallet.** If a colleague hands you a seed-phrase, do not import it into your MetaMask just to revoke. Use the paste-only audit mode of revoke.cash, identify what needs to be cleaned, and ask them to clean it themselves with their wallet. The principle: only your wallet talks to your wallet.
- **Use Ethereum mainnet.** QA swap regression runs on real Ethereum mainnet with the team's shared seed. Mainnet allowances cost real gas to revoke — the team budgets for this. If you are unsure whether a test should broadcast, ask your senior.
- **Clean up after manual sessions.** If you ran a flow by hand, end the session by checking revoke.cash and reverting any non-zero allowances you created. Don't leave the seed dirty for the next engineer.
- **Verify before announcing.** Don't tell the team "the test seed is clean" until you've checked it on a block explorer or revoke.cash. Test logs lie when assertions are weak; on-chain state is the ground truth.
- **Document non-obvious approvals.** If a test deliberately creates and leaves an approval (rare, but it happens — for example a regression test for "what does the UI do when an approval already exists"), add a comment in the spec and a note in the team doc so the next person doesn't try to clean it.
- **Never paste your personal address into a test slot.** The QAA test seed is the QAA test seed. Don't substitute your own seed because the QAA seed ran out of test ETH. Top up the QAA seed from the faucet instead.

### 6.3.9 Past Incidents — Why This Tooling Matters

Infinite ERC-20 approvals were the attack vector in several major DeFi drains. The pattern: a user clicks "approve unlimited" once on a dapp, the dapp (or its compromised contract) later calls `transferFrom(user, attacker, all_tokens)`, the user's tokens leave their wallet without any second user signature. The allowance is the standing authorisation; the attacker just needs to find a way to trigger the transfer.

**BadgerDAO, December 2021.** Frontend compromise injected malicious approval prompts. Users who had previously approved the legitimate Badger contracts had nothing to fear from old approvals — but the injected prompts asked for new, broader approvals on tokens like WBTC. Users approved, the attackers' contract drained around $120m worth of tokens. The takeaway: even users who knew about approvals were not protected if they were socially engineered into signing new ones. The mitigation was post-hoc — revoke any approval you don't actively need.

**Multiple 2022 phishing campaigns.** Throughout 2022, dozens of phishing sites mimicking real dapps asked users to "claim an airdrop" or "verify their wallet," but the actual signature presented was an unlimited `approve(attackerContract, 2^256-1)` on the user's most valuable token. Once signed, the attacker drained at their leisure. Some campaigns ran for weeks before being fully shut down.

The industry response had three parts:

- **revoke.cash adoption.** The tool existed before these incidents but became standard wallet-hygiene practice afterwards. Many wallet UIs now link directly to revoke.cash from their settings page.
- **Wallet UI warnings on infinite approvals.** Modern Ledger Live, MetaMask, and Rabby flag unlimited approvals at signing time, often with red banners and explicit "this gives the contract permission to spend any amount" wording. The clear-sign decoder on Ledger devices does the same — when you see `approve(spender, 0xff..ff)` on a Nano S Plus screen, the device labels it unambiguously.
- **EIP-2612 permit.** Newer ERC-20 tokens implement `permit(owner, spender, value, deadline, v, r, s)` — a signature-based approval that doesn't write to allowance storage. Each `permit` is single-use and time-limited. No standing allowance to drain. USDC, DAI, and most major tokens deployed since 2021 implement this. Older tokens (USDT, WBTC) don't, which is why allowance hygiene still matters for those specifically.

For QAA, the lesson is concrete: when you see an unlimited approval in test logs, that's almost always a test bug or a UX regression worth flagging. Tests should approve only the amount needed — `ensureTokenApproval` in `swap.page.ts` uses `1.2 × minAmount` for slippage, not infinity, deliberately. Anything that approves `2^256-1` in a test is suspicious and should be challenged in code review.

<div class="chapter-outro">
<strong>Key takeaway:</strong> revoke.cash is the daily driver. Etherscan Read is the audit oracle. Etherscan Write is the escape hatch when revoke.cash doesn't list a token. The shared QAA test seed is infrastructure — clean up after yourself, verify on-chain before you announce anything, and remember why allowance hygiene matters in the first place.
</div>

### 6.3.10 Quiz

<!-- ── Chapter 6.3 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Chapter 6.3 Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> Which manual revoke tool is open-source and run by Rosco Kalis?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Etherscan Write Contract</button>
<button class="quiz-choice" data-value="B">B) revoke.cash</button>
<button class="quiz-choice" data-value="C">C) DeBank</button>
<button class="quiz-choice" data-value="D">D) Zapper Smart Wallet</button>
</div>
<p class="quiz-explanation">revoke.cash is the open-source, multi-chain canonical revoke tool. Source on GitHub at RevokeCash/revoke.cash. Etherscan is closed-source; DeBank and Zapper are commercial portfolio aggregators.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> When would you reach for Etherscan's Write Contract tab instead of revoke.cash?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) When you don't have a wallet connected at all</button>
<button class="quiz-choice" data-value="B">B) When you only need to read the current allowance</button>
<button class="quiz-choice" data-value="C">C) When revoke.cash doesn't list the token (weird testnet token, custom ABI, non-standard approve semantics)</button>
<button class="quiz-choice" data-value="D">D) When you want to revoke for someone else's wallet</button>
</div>
<p class="quiz-explanation">revoke.cash covers ~99% of cases. Etherscan Write is the escape hatch when revoke.cash's scanner doesn't index a token, or when you need to call a non-standard function (decreaseAllowance, permit, custom admin methods).</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> Calling <code>allowance(owner, spender)</code> on Etherscan's Read Contract tab returns:</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A raw uint256 in the token's smallest unit — you must divide by 10^decimals to get the display amount</button>
<button class="quiz-choice" data-value="B">B) A pre-formatted human-readable amount</button>
<button class="quiz-choice" data-value="C">C) A boolean indicating whether an allowance exists</button>
<button class="quiz-choice" data-value="D">D) The list of all spenders with non-zero allowances</button>
</div>
<p class="quiz-explanation">Solidity returns the raw uint256. USDC has 6 decimals, so a returned value of 1000000000 means 1000 USDC. DAI has 18 decimals, so the same hex would be a tiny fraction. Always check decimals() before interpreting.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> Which is NOT a hygiene rule for the shared QAA test seed?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Clean up non-zero allowances after manual testing sessions</button>
<button class="quiz-choice" data-value="B">B) Verify on-chain (revoke.cash or Etherscan Read) before announcing the seed is clean</button>
<button class="quiz-choice" data-value="C">C) Use `DISABLE_TRANSACTION_BROADCAST=1` during development to avoid unintended on-chain state changes</button>
<button class="quiz-choice" data-value="D">D) Import a colleague's seed phrase into your wallet so you can revoke on their behalf</button>
</div>
<p class="quiz-explanation">Never import someone else's seed phrase. The principle is: only your wallet talks to your wallet. If a colleague's seed needs cleaning, ask them to do it. Options A, B, and C are all correct hygiene practices: clean up after manual sessions, verify on-chain state, and use broadcast-disabled mode during development.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> Why was EIP-2612 permit introduced as a response to allowance-vector incidents?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It cancels all existing on-chain allowances at once</button>
<button class="quiz-choice" data-value="B">B) It replaces standing allowance storage with single-use, time-limited signed approvals — no standing approval for an attacker to drain</button>
<button class="quiz-choice" data-value="C">C) It encrypts approval transactions so attackers can't see them</button>
<button class="quiz-choice" data-value="D">D) It requires a Ledger device for every transfer</button>
</div>
<p class="quiz-explanation">EIP-2612 permit lets a user sign an approval off-chain that the spender then redeems in the same transaction as the transfer. Each permit is one-shot and time-bounded. There is no standing allowance in storage, so no allowance-drain attack vector. USDC and DAI implement permit; older tokens (USDT, WBTC) do not.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## The Five-Layer Integration

<div class="chapter-intro">
The CLI does not stand alone. When a Playwright spec calls <code>app.swap.revokeTokenApproval(account, provider)</code>, the line of code you wrote in a page object reaches across <strong>five distinct layers</strong> before a Speculos-emulated Ledger device finally signs an ERC-20 <code>approve(spender, 0)</code> transaction and broadcasts it to Ethereum mainnet. Each layer absorbs a different concern: subprocess plumbing, typed wrappers, device-tap automation, Speculos lifecycle, test orchestration. Most QA work — by far — lives in the topmost layer (the page object methods you write per ticket). The four layers below it are infrastructure that already exists and rarely changes. This chapter catalogues those layers, shows the real code from the senior engineer's <code>support/qaa-615-add-revoke-token</code> branch, and traces the complete end-to-end call path. Keep this open when you land on a CLI integration ticket: it will tell you which layer to touch and which to leave alone.
</div>

### 6.4.1 Why Five Layers

The CLI integration is not a tower of abstraction for its own sake. Each layer absorbs a concern that the layers above it should not see. If you collapsed the architecture into one big function — say, a page object method that called `child_process.spawn` directly and also drove Speculos — the result would be unreadable, untestable, and impossible to reuse across the desktop and mobile test workspaces. The five-layer split is the price the codebase pays for two properties:

1. **Reuse.** Both `e2e/desktop/` (Playwright) and `e2e/mobile/` (Detox/Jest) import the exact same `cliCommandsUtils` from `libs/ledger-live-common/src/e2e/`. There is no desktop CLI helper and a separate mobile CLI helper. The CLI integration is genuinely shared infrastructure, and that is only possible because the lower four layers know nothing about Playwright or Detox.
2. **Locality of change.** When you add a new CLI helper for a ticket — say, "give me a typed wrapper for the new `tokenAllowance --json` output" — you touch one file (Layer 3). When you add a new POM method that uses an existing helper, you touch one file (Layer 5). The layer boundaries are designed so that the most common QA work touches the smallest surface area.

Here is the layer table, with the path you would `cd` into to read the source:

| # | Path | Role | Touch frequency for QA |
|---|---|---|---|
| 1 | `apps/cli/bin/index.js` (and `apps/cli/src/commands/blockchain/*`) | The legacy CLI binary — Node + Commander, package `@ledgerhq/live-cli`, bin name `ledger-live`. | **Rare.** Only when adding a brand-new CLI subcommand. |
| 2 | `libs/ledger-live-common/src/e2e/runCli.ts` | The spawn engine. `child_process.spawn`, the `+` arg separator, retry policy, error envelope. | **Rare.** Engine extensions only. |
| 3 | `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts` | Typed TypeScript wrappers — the curried `liveDataCommand`, plus `approveTokenCommand`, `revokeTokenCommand`, `isTokenAllowanceSufficientCommand`, etc. | **Occasional.** New helper per new ticket family. |
| 4 | `libs/ledger-live-common/src/e2e/families/evm.ts` | Speculos device-tap automation — `approveToken`, `approveTokenTouchDevices`, `approveTokenButtonDevice`, the screen primitives. | **Rare.** New device-tap pattern. |
| 5 | `e2e/desktop/tests/page/swap.page.ts`, `e2e/mobile/page/trade/swap.page.ts`, and per-spec `*.spec.ts` files | The POM methods (`ensureTokenApproval`, `revokeTokenApproval`) and the specs that call them. | **Frequent.** This is where most of your tickets land. |

The unwritten rule is: a layer-5 author should never have to read or edit anything in layers 1, 2, or 4. If a ticket forces you down that far, raise it as a question — the senior engineers will either point you at an existing helper or extend layer 3 for you. Layer 3 is the negotiated boundary between QA work and platform work.

### 6.4.2 Layer 1 — The Binary `apps/cli`

The bottom of the stack is a regular Node CLI. It lives at `apps/cli/` and is published as `@ledgerhq/live-cli`. The bin entry from `apps/cli/package.json`:

```json
{
  "name": "@ledgerhq/live-cli",
  "version": "24.39.0",
  "description": "ledger-live CLI version",
  "bin": {
    "ledger-live": "./bin/index.js"
  },
  "scripts": {
    "prebuild": "zx ./scripts/gen.mjs",
    "build": "zx ./scripts/build.mjs",
    "test": "zx ./scripts/test.mjs"
  }
}
```

Three things to notice. First, **the bin name is `ledger-live`, not `live-cli` or `lcli`**. Every error envelope, every log line, every shell example in this guide will refer to `ledger-live`. Second, **the entry point is `./bin/index.js`** — a built JavaScript file produced by `zx scripts/build.mjs`. The TypeScript source lives in `apps/cli/src/`, but the spawn engine targets the built JS. If you edit the source and forget to run `pnpm --filter @ledgerhq/live-cli build`, your tests will run the stale binary and you will spend an hour debugging code that is not the code you think you are running. Third, **the `prebuild` step (`zx ./scripts/gen.mjs`) generates the commands index** — the file that wires every `apps/cli/src/commands/blockchain/*.ts` into the Commander dispatcher. If you add a brand-new CLI command, the generator picks it up automatically; you do not edit the index by hand.

Inside `apps/cli/src/commands/blockchain/` there are sixteen `.ts` files, each one a Commander subcommand. The two that QA cares about are:

- **`send.ts`** — signs and (optionally) broadcasts a transaction. Supports `--mode approve | revokeApproval`, plus `--token`, `--spender`, `--approveAmount`, `--wait-confirmation`. This is the file driven by `runCliTokenApproval` (Layer 2) and indirectly by `approveTokenCommand` and `revokeTokenCommand` (Layer 3).
- **`tokenAllowance.ts`** — reads the current ERC-20 allowance from the token contract. Supports `--format json` for machine-readable output. Driven by `runCliGetTokenAllowance` and indirectly by `isTokenAllowanceSufficientCommand`.

The other fourteen commands (`bot.ts`, `botPortfolio.ts`, `derivation.ts`, `generateTestScanAccounts.ts`, `signMessage.ts`, etc.) are out of QAA scope. They exist for live-common bots, fixture generation, or one-off engineering work — not for QA regression suites. You will see them in the directory listing; ignore them.

The key code path inside `send.ts` — the broadcast gate — is where the `DISABLE_TRANSACTION_BROADCAST` environment variable (covered in detail in Chapter 6.6) flips the binary between "sign-only" and "sign-and-broadcast" mode:

```ts
// apps/cli/src/commands/blockchain/send.ts (excerpt)
return bridge
  .signOperation({
    account,
    transaction: t,
    deviceId: opts.device || "",
  })
  .pipe(
    map(toSignOperationEventRaw),
    // @ts-expect-error more voodoo stuff
    ...(opts["disable-broadcast"] || getEnv("DISABLE_TRANSACTION_BROADCAST")
      ? []
      : [
          concatMap((e: any) => {
            if (e.type === "signed") {
              l(`✔️ has been signed! ${JSON.stringify(e.signedOperation)}`);
              return from(
                bridge
                  .broadcast({
                    account,
                    signedOperation: e.signedOperation,
                  })
                  .then(async op => { /* ... */ }),
              );
            }
            return of(e);
          }),
        ]),
  );
```

This is the only place in the entire stack where `bridge.broadcast(...)` is called for the approve/revoke path. Either flag — the `--disable-broadcast` CLI argument or the `DISABLE_TRANSACTION_BROADCAST` env var — being truthy short-circuits the broadcast operator to an empty array, and the rxjs pipe just passes the signed-operation event through without sending it on-chain. Layer 3 toggles this gate via the env var; you (Layer 5) almost never see it directly. Chapter 6.6 covers the discipline around this flag in detail.

### 6.4.3 Layer 2 — `runCli.ts`, the Spawn Engine

Layer 2 is one file: `libs/ledger-live-common/src/e2e/runCli.ts`. It contains two engine functions (`runCliCommand` and `runCliCommandWithRetry`), one constant (`LEDGER_LIVE_CLI_BIN`), and a small set of public helpers that produce specific CLI command strings (`runCliLiveData`, `runCliGetAddress`, `runCliTokenApproval`, `runCliGetTokenAllowance`).

#### The spawn engine, verbatim

```ts
// libs/ledger-live-common/src/e2e/runCli.ts
import path from "path";
import { spawn } from "child_process";
import { sanitizeError, sleep } from "./index";

export const LEDGER_LIVE_CLI_BIN = path.resolve(__dirname, "../../../../apps/cli/bin/index.js");
```

`LEDGER_LIVE_CLI_BIN` is computed at module load. `__dirname` is wherever the built copy of `runCli.js` lives inside `node_modules/@ledgerhq/live-common/lib/e2e/` (or the source directory, depending on how the workspace is consumed); the four `..` segments walk back up to the monorepo root, then forward into `apps/cli/bin/index.js`. The result is an absolute path that the spawn call below feeds to `node`. If your tests start failing with `Cannot find module '/.../apps/cli/bin/index.js'`, this is the constant to investigate — it usually means the CLI was not built.

The flag parser used in error messages:

```ts
function parseCliFlag(command: string, flag: string): string | undefined {
  const parts = command.split("+");
  const idx = parts.findIndex(p => p === `--${flag}`);
  return idx !== -1 && idx + 1 < parts.length ? parts[idx + 1] : undefined;
}
```

This is purely cosmetic: when a CLI command fails, the error envelope wants to surface "which currency" and "which account index" without the caller having to plumb that through. `parseCliFlag` re-parses the same `+`-joined command string the engine just spawned.

The retryable-error classifier:

```ts
function isRetryableError(message: string): boolean {
  const retryablePatterns = [
    /503/i,
    /502/i,
    /504/i,
    /GeneralDmkError/i,
    /ECONNREFUSED/i,
    /ETIMEDOUT/i,
    /ECONNRESET/i,
    /socket hang up/i,
    /timeout/i,
  ];
  return retryablePatterns.some(pattern => pattern.test(message));
}
```

This is the policy that decides "transient network issue, try again" vs. "deterministic logic failure, surface immediately." The classifier is intentionally conservative: only HTTP gateway codes (502/503/504), generic DMK errors, low-level socket failures, and the literal string "timeout" trigger a retry. A `BigNumber error` or a `Speculos device not found` does not retry — those are bugs in your test setup, and silently retrying would just delay the failure.

Now the main engine function:

```ts
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

    child.stdout.on("data", data => {
      output += data.toString();
    });

    child.stderr.on("data", data => {
      errorOutput += data.toString();
    });

    child.on("exit", code => {
      if (code === 0) {
        resolve(output);
      } else {
        const currency = parseCliFlag(command, "currency");
        const index = parseCliFlag(command, "index");
        const indexText = index && index !== "undefined" ? index : "N/A";

        const errorDetails = [
          `❌ Failed to execute CLI command`,
          `🔍 Command: ${command}`,
          `💱 Currency: ${currency}`,
          `🔢 Index: ${indexText}`,
          `🔢 Exit Code: ${code}`,
          errorOutput ? `🧾 CLI Error : ${errorOutput.trim()}` : "",
        ].join("\n");

        reject(new Error(errorDetails));
      }
    });

    child.on("error", error => {
      reject(new Error(`Error executing CLI command: ${sanitizeError(error)}`));
    });
  });
}
```

A few things to notice:

- **The log line is the most useful debugging artefact in the entire stack.** `[CLI] Executing: ledger-live send --currency ethereum --mode revokeApproval --token ethereum/erc20/usd_coin --spender 0xRouter --index 0 --wait-confirmation` is a string you can paste into your shell (after a manual Speculos boot) and reproduce the exact command the test was running. When a test fails inscrutably in CI, this line is the first place to look in the Allure log.
- **Args travel as one string with `+` separators, then split before spawn.** `command.split("+")` reverses the join. The reason for the `+` convention is shell-escape avoidance: the helpers in `cliCommandsUtils.ts` build the command with `cliOpts.push("--currency+ethereum")` and `cliOpts.join("+")`, never having to worry about quoting spaces, ampersands, or shell-special characters. The trade-off is that you cannot pass an argument that legitimately contains a `+` character. None of the CLI commands need to.
- **`stdio: "pipe"`** captures stdout and stderr into the parent process's buffers. You do not see the CLI's output in your terminal in real time during a test run; you see it after the fact in `output` (returned on success) or in the error envelope (on failure).
- **`env: process.env`** is critical. The spawned child inherits the parent's environment — including `SPECULOS_API_PORT` (set by `launchSpeculos`) and `DISABLE_TRANSACTION_BROADCAST` (toggled by `setDisableTransactionBroadcastEnv`). This is how Layer 5 communicates with Layer 1: not through arguments, but through environment variables that travel down through the spawn.
- **The error envelope is a multi-line, emoji-flagged Markdown-ish block.** It is designed to be copy-pasted into a Jira ticket or Slack message verbatim. The currency, the index, the exit code, and the trimmed stderr are all there in one rectangle. When you triage a flake, you do not need to reconstruct context — Layer 2 has done it for you.

The retry wrapper:

```ts
export async function runCliCommandWithRetry(
  command: string,
  retries: number = 3,
  delayMs: number = 3000,
): Promise<string> {
  let lastError: Error | null = null;
  const currency = parseCliFlag(command, "currency");

  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      return await runCliCommand(command);
    } catch (err: unknown) {
      lastError = err instanceof Error ? err : new Error(String(err));
      const willRetry = attempt < retries && isRetryableError(lastError.message);

      if (!willRetry) {
        throw sanitizeError(lastError);
      }

      console.warn(
        `⚠️ CLI attempt ${attempt}/${retries}${currency ? ` for ${currency}` : ""} failed with retryable error – retrying in ${delayMs}ms…`,
        lastError.message,
      );

      await sleep(delayMs);
    }
  }

  throw sanitizeError(lastError!);
}
```

Three retries, three-second delay between attempts, only retryable errors bounce back into the loop. Every typed wrapper in Layer 3 calls `runCliCommandWithRetry`, never `runCliCommand` directly — so by default every CLI invocation in the test suite gets up to three tries against transient network or Speculos issues. Non-retryable errors (a malformed argument, a missing account, a `BigNumber` parse error) fail fast on the first attempt.

#### The public engine helpers

| Helper | CLI command produced | Caller |
|---|---|---|
| `runCliLiveData(opts)` | `liveData --currency X --index N [--scheme M] --add --appjson <path>` | `liveDataCommand` (Layer 3) |
| `runCliGetAddress(opts)` | `getAddress --currency X --path m/... --derivationMode M [--verify]` | `getAccountAddress` (Layer 3) |
| `runCliTokenApproval(opts)` | `send --currency X --mode approve\|revokeApproval --token Y --spender Z --index N [--approveAmount N] [--wait-confirmation]` | `approveTokenCommand`, `revokeTokenCommand` (Layer 3) |
| `runCliGetTokenAllowance(opts)` | `tokenAllowance --currency X --token Y --spender Z --index N [--format json] --ownerAddress 0x...` | `isTokenAllowanceSufficientCommand` (Layer 3) |

One verbatim, so you can see the pattern:

```ts
export function runCliTokenApproval(opts: TokenApprovalOpts): Promise<string> {
  const cliOpts = ["send"];
  cliOpts.push(`--currency+${opts.currency}`);
  cliOpts.push(`--mode+${opts.mode}`);
  cliOpts.push(`--token+${opts.token}`);
  cliOpts.push(`--spender+${opts.spender}`);
  cliOpts.push(`--index+${opts.index}`);
  if (opts.approveAmount) cliOpts.push(`--approveAmount+${opts.approveAmount}`);
  if (opts.waitConfirmation) cliOpts.push("--wait-confirmation");
  return runCliCommandWithRetry(cliOpts.join("+"));
}
```

The `TokenApprovalOpts` type is defined at the top of the file:

```ts
export type TokenApprovalOpts = {
  currency: string;
  index: number;
  spender: string;
  approveAmount?: string;
  token: string;
  waitConfirmation?: boolean;
  mode: "revokeApproval" | "approve";
};
```

`mode: "revokeApproval" | "approve"` is the discriminated union that selects between the two flows. This is the type-level guarantee that you cannot pass `mode: "transferTo"` or `mode: ""` and have it silently produce a malformed command — TypeScript stops you at the call site.

### 6.4.4 Layer 3 — `cliCommandsUtils.ts`, Typed Wrappers

Layer 3 lives in `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts`. This is the file QA engineers extend most often. It exports:

| Helper | Inputs | Returns | Used by |
|---|---|---|---|
| `getAccountAddress(account)` | `Account \| TokenAccount` | `Promise<string>` (also writes to `account.address`) | Every spec needing the device-derived address |
| `liveDataCommand(account, options?)` | account + options | curried `(userdataPath?) => Promise<void>` | Desktop fixture `cliCommands: [...]` |
| `liveDataWithAddressCommand(account, options?)` | same | curried, also caches address | When a spec needs both seed + address |
| `liveDataWithParentAddressCommand(parent, child)` | parent + child token | curried | Token account address resolution |
| `liveDataWithRecipientAddressCommand(tx, options?)` | a `Transaction` | curried | Send-flow specs |
| `parseTokenAllowanceCliOutput(output)` | CLI stdout | `{ allowanceStr, unitMagnitude }` | Internal parser |
| `isTokenAllowanceSufficientCommand(account, spender, minAmount)` | TokenAccount, spender, min | allowance string if ≥ min, else `0` | `ensureTokenApproval` (Layer 5) |
| `approveTokenCommand(account, spender, approveAmount)` | TokenAccount, spender, amount | CLI stdout | Sets allowance, broadcast on |
| **`revokeTokenCommand(account, spender)` (NEW)** | TokenAccount, spender | CLI stdout | Clears allowance to 0, broadcast on |
| `setDisableTransactionBroadcastEnv(value)` | `string \| undefined` | previous env value | Broadcast env management |

Let us look at four of these verbatim to see the patterns.

#### `getAccountAddress` — read once, cache on the account

```ts
export const getAccountAddress = async (account: Account | TokenAccount): Promise<string> => {
  if (account.currency.id === Currency.HBAR.id) {
    invariant(account.address, "hedera: account address must be pre-set");
    return account.address;
  }

  const { address } = await runCliGetAddress({
    currency: account.currency.speculosApp.name,
    path: account.accountPath,
    derivationMode: account.derivationMode,
  });

  account.address = address;
  return address;
};
```

Two patterns to notice. First, **the Hedera special case is hard-coded**. HBAR accounts derive their address differently — the test suite pre-sets `account.address` from a fixture, and `getAccountAddress` honours that. If you ever need to add a similar special case, this is the canonical place. Second, **the call mutates the input.** `account.address = address` writes the resolved address back onto the account object so subsequent code can read `account.address` without re-running the CLI. It is the classic "memoise on the parameter" trick.

#### `liveDataCommand` — the curried-function pattern

```ts
export const liveDataCommand =
  (account: Account | TokenAccount, options?: LiveDataCommandOptions) =>
  async (userdataPath?: string) => {
    await runCliLiveData({
      currency: options?.currency ?? account.currency.speculosApp.name,
      index: account.index,
      ...(options?.useScheme && account.derivationMode ? { scheme: account.derivationMode } : {}),
      add: true,
      appjson: userdataPath,
    });
  };
```

This is the curried-function pattern that powers the desktop fixture engine. Notice the shape: `liveDataCommand` takes an `account` and returns *another function* that takes a `userdataPath`. When a spec writes:

```ts
test.use({
  cliCommands: [liveDataCommand(account)],
});
```

…it is passing the partially-applied function — bound to `account` but waiting for the userdata path — into the fixture. The Playwright fixture engine, at the right point in the lifecycle, calls each entry in `cliCommands` with the test's userdata path. The currying lets the spec author specify the *what* (which account) at test definition time, while the fixture engine supplies the *where* (which userdata file) at runtime. Without currying, the spec author would have to thread the userdata path through every fixture by hand.

The mobile workspace consumes the same helper but bypasses the curry: `e2e/mobile/jest.environment.ts` line 138 does `Object.assign(this.global, cliCommandsUtils)` so every helper becomes a global, and mobile specs call `await liveDataCommand(account)(userdataPath)` directly — manually applying both arguments.

#### `isTokenAllowanceSufficientCommand` — the pre-check

```ts
export const isTokenAllowanceSufficientCommand = async (
  account: TokenAccount,
  spenderAddress: string,
  minAmount: string,
) => {
  const ownerAddress = account.parentAccount?.address ?? account.address;
  if (!ownerAddress) throw new Error("Token allowance check requires the main account address");

  const output = await runCliGetTokenAllowance({
    currency: account.currency.speculosApp.name,
    token: account.currency.id,
    spenderAddress,
    index: account.index,
    format: "json",
    ownerAddress,
  });

  const { allowanceStr, unitMagnitude } = parseTokenAllowanceCliOutput(output);

  const smallestUnit = { name: "smallest", code: "", magnitude: unitMagnitude } as const;
  const minInSmallestUnit = parseCurrencyUnit(smallestUnit, minAmount);
  const minStr = minInSmallestUnit.toFixed(0);

  const allowanceBi = BigInt(allowanceStr);
  const minBi = BigInt(minStr);
  if (allowanceBi >= minBi) return allowanceStr;
  return 0;
};
```

This is the read-side wrapper for `tokenAllowance`. It does three things `runCliGetTokenAllowance` cannot do alone:

1. **Resolves the owner address.** A `TokenAccount` does not carry its own owner address — the underlying ETH account does. The line `account.parentAccount?.address ?? account.address` reaches up to the parent for ERC-20 token accounts, falling back to the account's own address if no parent is present.
2. **Parses the JSON output.** `parseTokenAllowanceCliOutput` extracts `{ allowanceStr, unitMagnitude }` from the CLI's JSON stdout. The CLI returns the raw allowance in the token's smallest units; the unit magnitude lets the caller know how many decimals to apply.
3. **Compares as BigInts.** The min amount comes in as a human-readable string (e.g. `"100"` USDC). It gets multiplied through `parseCurrencyUnit` into the same smallest-unit representation as the allowance, then both go through `BigInt(...)` for an exact comparison. JavaScript `Number` arithmetic would lose precision on values larger than 2^53.

The return contract is a tiny piece of API design: the function returns `allowanceStr` (a non-empty string) when the allowance is sufficient, and the literal `0` when it is not. Truthiness is the test: `if (currentAllowance) return;` in the POM (Layer 5) skips the device dance when allowance is already enough. Returning the string instead of `true` lets the caller log the actual allowance value if it wants to.

#### `approveTokenCommand` and the new `revokeTokenCommand`

These are the two wrappers that drive the device. First the existing one:

```ts
export const approveTokenCommand = async (
  account: TokenAccount,
  spender: string,
  approveAmount: string,
) => {
  const original = setDisableTransactionBroadcastEnv("0");

  const result = runCliTokenApproval({
    currency: account.currency.speculosApp.name,
    index: account.index,
    spender,
    token: account.currency.id,
    mode: "approve",
    approveAmount,
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

Now the new one from the senior's commit, verbatim:

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

Side-by-side these two helpers tell the whole story. They differ in exactly two places: the `mode` (`"approve"` vs `"revokeApproval"`) and the absence of `approveAmount` in the revoke (because revoke is `approve(spender, 0)` — there is no amount to set). Everything else — the env flip, the parallel `approveToken()` device tap, the try/finally restore — is identical.

The control flow is worth stepping through carefully because it is the one piece of multi-track logic you must understand to debug these helpers:

1. **`setDisableTransactionBroadcastEnv("0")` flips the env to broadcast-on.** It returns the previous value, captured in `original`. If broadcast was already on, `original` is `"0"`; if it was off (`"1"`), `original` is `"1"`; if it was unset, `original` is `undefined`. The helper restores that exact original at the end. Chapter 6.6 covers why broadcast must be on for this command and off for most others.
2. **`runCliTokenApproval(...)` is called *without* `await`.** It returns a `Promise<string>` that is held in the `result` variable. The CLI subprocess is now spawned and running in parallel — sitting in `bridge.signOperation(...)`, waiting for the device to confirm the transaction.
3. **`await approveToken()` is the device tap.** While the CLI subprocess is blocked waiting for the device, this helper *is* the device. It drives Speculos's REST API to walk through the review screens and press the buttons. Layer 4 covers the screen primitives in detail. The crucial point here is the parallelism: the CLI and the device tap are two cooperating coroutines, neither one finishing without the other.
4. **The `try/finally` guarantees env restoration.** If `approveToken()` throws (a screen never appears, a button press fails, Speculos times out), the `finally` block still runs and `setDisableTransactionBroadcastEnv(original)` puts the env back. Without this discipline, a single failed test would mutate `process.env` for the rest of the run, silently flipping broadcast for every subsequent CLI call. Chapter 6.6 has the full discipline section.
5. **`return await result;`** awaits the CLI subprocess's resolution. By this point the device tap has sent the buttons that confirm the signature; the CLI has built the signed transaction, called `bridge.broadcast(...)`, waited for the on-chain confirmation (because `waitConfirmation: true`), and resolved with its stdout. The stdout is a multi-line block ending with the broadcast operation hash, which the POM passes into Allure for traceability.

The `setDisableTransactionBroadcastEnv` helper itself is a few lines at the bottom of the file:

```ts
const ENV_KEY = "DISABLE_TRANSACTION_BROADCAST";

export function setDisableTransactionBroadcastEnv(value: string | undefined): string | undefined {
  const previous = process.env[ENV_KEY];
  if (value === undefined) {
    delete process.env[ENV_KEY];
  } else {
    process.env[ENV_KEY] = value;
  }
  return previous;
}
```

Three rules: read the previous value, set the new value (or `delete` it if `undefined` was passed), return the previous. The "return the previous" contract is the foundation of the try/finally restore pattern — without it, you would have to read the env *before* calling the helper and pass it in, which is a footgun.

> See Chapter 6.6 for the broadcast discipline in depth: when broadcast is on, when it is off, why the default is off, and the full set of read sites and set sites.

### 6.4.5 Layer 4 — `families/evm.ts`, Device-Tap Automation

Layer 4 lives in `libs/ledger-live-common/src/e2e/families/evm.ts`. This is where Speculos is *driven*: the layer above (Layer 3) has spawned a CLI subprocess that is now blocked waiting for the device to confirm a transaction; this layer is the user, walking through the review screens and pressing the buttons.

#### The screen primitives

Imported at the top of `evm.ts`:

```ts
import {
  containsSubstringInEvent,
  fetchCurrentScreenTexts,
  waitForReviewTransaction,
  pressUntilTextFound,
  waitFor,
} from "../speculos";
import { getSpeculosModel, isTouchDevice } from "../speculosAppVersion";
import {
  longPressAndRelease,
  pressAndRelease,
  swipeRight,
} from "../deviceInteraction/TouchDeviceSimulator";
import { DeviceLabels } from "../enum/DeviceLabels";
import { withDeviceController } from "../deviceInteraction/DeviceController";
```

The primitives:

- **`waitFor(label)`** — block until the given screen-text label appears on the Speculos display. Polls the Speculos REST API. Times out after the suite-level timeout if the label never appears.
- **`waitForReviewTransaction()`** — sugar for `waitFor("Review transaction")` (or the touch-device equivalent). Used at the start of the device tap to confirm the CLI has reached the signing prompt.
- **`pressUntilTextFound(label)`** — repeatedly press the right button (button device) or scroll right (touch device) until the given label appears. Returns the array of screen-text events seen along the way; this array is what `validateTransactionData` later checks for amount, recipient, and ENS-name correctness.
- **`pressAndRelease(direction, x, y)`** — simulate a single button or touch press at the given coordinates.
- **`longPressAndRelease(label, seconds)`** — hold to sign on touch devices. Used when the final screen demands a multi-second hold.
- **`fetchCurrentScreenTexts(port)`** — pull the current screen text from Speculos's REST API. Used for read-only screen inspection.
- **`withDeviceController(fn)`** — DI wrapper that provides a `getButtonsController()` to the inner function. The buttons controller exposes `.both()`, `.left()`, `.right()` for explicit button presses on Nano S/X/S+.
- **`DeviceLabels.*`** — a shared registry of screen-text constants (`REVIEW_TRANSACTION_TO`, `SIGN_TRANSACTION`, `HOLD_TO_SIGN`, `ACCEPT`). Touch and button devices use the same logical label keys but the actual on-device strings may differ; the enum centralises that mapping.

#### `approveToken` — the public entry point

```ts
export async function approveToken() {
  if (isTouchDevice()) {
    return approveTokenTouchDevices();
  }
  return approveTokenButtonDevice();
}
```

Three lines. The branch is on `isTouchDevice()`, which reads the current Speculos device model (from the env-configured firmware) and returns `true` for Stax and Flex, `false` for Nano S, X, and S+. The two implementations share the same logical flow but use different physical interactions.

#### `approveTokenTouchDevices` — Stax and Flex

```ts
export async function approveTokenTouchDevices() {
  await waitForReviewTransaction();
  await pressUntilTextFound(DeviceLabels.HOLD_TO_SIGN);
  await longPressAndRelease(DeviceLabels.HOLD_TO_SIGN, 3);
}
```

Three steps: wait for the review screen, scroll until the "Hold to sign" prompt appears, then perform a 3-second hold to confirm. The hold duration is hard-coded to 3 seconds — if Stax firmware ever changes that requirement, this is the constant to update.

#### `approveTokenButtonDevice` — Nano S/X/S+

```ts
export const approveTokenButtonDevice = withDeviceController(
  ({ getButtonsController }) =>
    async () => {
      await waitFor(DeviceLabels.REVIEW_TRANSACTION_TO);
      await pressUntilTextFound(DeviceLabels.SIGN_TRANSACTION);
      await getButtonsController().both();
    },
);
```

The structure mirrors the touch-device variant but uses button presses. `withDeviceController` is the DI wrapper that provides `getButtonsController()`. The flow is:

1. **`waitFor(DeviceLabels.REVIEW_TRANSACTION_TO)`** — block until the device renders "Review transaction To" (the recipient screen).
2. **`pressUntilTextFound(DeviceLabels.SIGN_TRANSACTION)`** — press the right button repeatedly to advance through the review screens (recipient, amount, fees, contract data, etc.) until "Sign transaction" appears.
3. **`getButtonsController().both()`** — press both buttons simultaneously to confirm.

Note that `approveTokenTouchDevices` is exported as a regular `async function` while `approveTokenButtonDevice` is a `withDeviceController(...)`-wrapped value. The shape is different because the touch variant does not need the buttons controller; touch interactions go through `longPressAndRelease` which uses Speculos's touch endpoint, not the button endpoint. The DI wrapper is the cleanest way to opt in to the buttons controller only on the button-device branch.

#### Parallelism — the moving parts

The diagram everyone needs to internalise is this:

```
TIME ──────────────────────────────────────────────────────────────────►

CLI subprocess (Layer 1):    spawn → signOperation → [block on device] → broadcast → confirm
                                                          ▲                ▲
                                                          │                │
                                                          │  (APDU)        │  (broadcast result)
                                                          │                │
Device tap (Layer 4):                            waitFor → press → press → both()
                                                 (drives Speculos REST API in parallel)
```

The CLI subprocess and the device tap are two coroutines running in the same Node process. The CLI's subprocess (Layer 1) is the one calling `bridge.signOperation(...)`, which under the hood opens an APDU transport to Speculos and waits for the device to respond with a signed payload. The device tap (Layer 4) is the one calling Speculos's REST API to render screens and press buttons. They communicate through Speculos itself — the device — not through any direct JavaScript wiring.

Critically, both must complete for the whole thing to succeed. If the device tap finishes early (because the screen labels are wrong and `pressUntilTextFound` returns prematurely), the CLI still hangs waiting for an APDU response. If the CLI fails early (an invalid argument, a missing token), the device tap will hang waiting for a screen that never renders. The `waitConfirmation: true` flag on `runCliTokenApproval` plus the suite-level timeouts on `waitFor` are the fail-safes that prevent the test from hanging forever.

### 6.4.6 Layer 5 — POM Methods

Layer 5 is where you live. The two POM methods that orchestrate everything below are `ensureTokenApproval` and the new `revokeTokenApproval`, both in `e2e/desktop/tests/page/swap.page.ts`. Verbatim from the file as it stands on the senior's branch:

```ts
@step("Ensure token approval")
async ensureTokenApproval(
  fromAccount: Account | TokenAccount,
  provider: Provider,
  minAmount: string,
) {
  if (!provider.contractAddress || !fromAccount.parentAccount) return;

  const currentAllowance = await isTokenAllowanceSufficientCommand(
    fromAccount,
    provider.contractAddress,
    minAmount,
  );
  console.log("CLI result: Current Allowance: ", currentAllowance);
  if (currentAllowance) return;

  const previousSpeculosPort = getEnv("SPECULOS_API_PORT");
  const speculos = await launchSpeculos(fromAccount.currency.speculosApp.name);
  try {
    const result = await approveTokenCommand(
      fromAccount,
      provider.contractAddress,
      new BigNumber(minAmount).times(12).div(10).toFixed(),
    );
    await allure.description(`Token approval result for ${provider.uiName}:\n\n ${result}`);
  } finally {
    await cleanSpeculos(speculos, previousSpeculosPort);
  }
}

@step("Ensure token approval")
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

A few observations.

The `@step("Ensure token approval")` decorator on `revokeTokenApproval` is a copy-paste bug that the senior left behind for code review — the final PR should change it to `@step("Revoke token approval")`. The `step` decorator publishes the method name into the Allure timeline so reviewers can see "this test entered the revoke step at 14:23:01" in the report. Two methods reporting the same step name will collide visually in the timeline.

The two methods share a common structure that is worth naming explicitly — the **"snapshot the port, launch a fresh Speculos, run a CLI command, restore"** pattern:

1. **Guard.** `if (!provider.contractAddress || !fromAccount.parentAccount) return;` — bail early for native-asset providers (no contract address) and non-token accounts. This is the cheap escape hatch that lets the same POM method be called for every provider in a parameterised test, without the caller having to filter.
2. **Snapshot.** `const previousSpeculosPort = getEnv("SPECULOS_API_PORT");` — capture the current Speculos port. The test may already have a Speculos instance running (the Exchange app for the swap UI flow). We are about to launch a *different* Speculos instance (the Ethereum app for the approve/revoke transaction). Restoring the port at the end lets the swap UI code resume against its original Exchange-app Speculos.
3. **Launch.** `const speculos = await launchSpeculos(fromAccount.currency.speculosApp.name);` — start a new Speculos for the token's chain (e.g. Ethereum for USDC). This call sets `SPECULOS_API_PORT` to the new instance's port and registers the HTTP transport so the CLI subprocess can connect. Chapter 6.5 covers the full lifecycle.
4. **Act.** Inside the try, the CLI helper runs (`approveTokenCommand` or `revokeTokenCommand`), which spawns the CLI subprocess and concurrently drives the device tap. The result string goes into Allure for traceability.
5. **Restore.** `await cleanSpeculos(speculos, previousSpeculosPort);` in the `finally` — stop the new Speculos, unregister its transport, and reset `SPECULOS_API_PORT` to the original. Even on throw, this runs.

The differences between the two methods are exactly the differences in semantics:

- **`ensureTokenApproval` does a pre-check.** Calling `isTokenAllowanceSufficientCommand` before launching Speculos avoids the entire device dance when the allowance is already enough — a major speedup in re-runs. `revokeTokenApproval` does not pre-check because revoking an already-zero allowance is a no-op gas-wise, and the test author wants the deterministic "after this method, allowance is zero" guarantee.
- **`ensureTokenApproval` applies a 1.2x slippage buffer.** `new BigNumber(minAmount).times(12).div(10).toFixed()` multiplies the minimum amount by 1.2 before passing it to `approveTokenCommand`, so the approved allowance leaves headroom for swap-rate fluctuations between the time the test approves and the time it executes. `revokeTokenApproval` has no such math because revoke always sets to zero.

Layer 5 is the integration point. From the perspective of a layer-5 author, the entire stack below collapses into "two helpers, one Speculos lifecycle, one Allure description." That is the abstraction the four lower layers were designed to provide.

### 6.4.7 The End-to-End Trace

When a desktop swap spec calls `await app.swap.revokeTokenApproval(account, provider)`, here is the complete sequence of events. Read it once with the layer numbers to anchor each step in the file you would open to debug it:

```
1. POM (Layer 5):  getEnv("SPECULOS_API_PORT")  → save previous port
                   (the Exchange-app Speculos already running for the swap UI)

2. POM (Layer 5):  launchSpeculos(currency.speculosApp.name)  → starts Ethereum-app Speculos
                   ↳ utils/speculosUtils.ts (Chapter 6.5)
                       ↳ startSpeculos(testTitle, specs[appName], previousPort)
                           ↳ Docker container OR remote pool (REMOTE_SPECULOS=true)
                       ↳ setEnv("SPECULOS_API_PORT", device.port)
                       ↳ process.env.SPECULOS_API_PORT = String(device.port)
                       ↳ CLI.registerSpeculosTransport(port, getSpeculosAddress())
                       ↳ allure.description("SPECULOS\n" + appInfo)

3. POM (Layer 5):  await revokeTokenCommand(account, spender)
                   ↳ Layer 3: setDisableTransactionBroadcastEnv("0")  → returns original
                   ↳ Layer 3: runCliTokenApproval({
                                 currency: "ethereum",
                                 mode: "revokeApproval",
                                 token: "ethereum/erc20/usd_coin",
                                 spender: "0xRouter",
                                 index: 0,
                                 waitConfirmation: true,
                               })  → returns Promise<string> (NOT awaited yet)
                       ↳ Layer 2: runCliCommandWithRetry(joined-command-string)
                           ↳ Layer 2: spawn("node", [LEDGER_LIVE_CLI_BIN, ...args])
                               ↳ Layer 1: apps/cli/bin/index.js parses argv
                               ↳ Layer 1: dispatches to send.ts
                               ↳ Layer 1: scan(account) + inferTransactions(mode=revokeApproval)
                               ↳ Layer 1: bridge.signOperation(...) — waits for device APDU
                                          (subprocess is now blocked here)
                   ↳ Layer 3: await approveToken()  ← runs in parallel with the subprocess
                       ↳ Layer 4: isTouchDevice() → false (assume Nano X)
                       ↳ Layer 4: approveTokenButtonDevice()
                           ↳ Layer 4: waitFor(REVIEW_TRANSACTION_TO)
                           ↳ Layer 4: pressUntilTextFound(SIGN_TRANSACTION)
                           ↳ Layer 4: getButtonsController().both()
                       ↳ device confirms → APDU response sent to CLI subprocess
                   ↳ Layer 1: subprocess unblocks, builds + signs tx
                   ↳ Layer 1: broadcast gate: DISABLE_TRANSACTION_BROADCAST="0" → broadcast runs
                   ↳ Layer 1: bridge.broadcast({ account, signedOperation })
                   ↳ Layer 1: waitForTransactionConfirmation (because --wait-confirmation)
                   ↳ Layer 1: subprocess writes result to stdout, exits 0
                   ↳ Layer 2: runCliCommand resolves with output string
                   ↳ Layer 3: finally → setDisableTransactionBroadcastEnv(original)
                   ↳ Layer 3: return await result;  → the CLI stdout

4. POM (Layer 5):  allure.description(`Token revoke result for ${provider.uiName}:\n\n ${result}`)

5. POM (Layer 5):  finally → cleanSpeculos(speculos, previousSpeculosPort)
                   ↳ stopSpeculos(device.id)
                   ↳ unregisterTransportModule("speculos-http-" + port)
                   ↳ setEnv("SPECULOS_API_PORT", previousPort)  → restore Exchange-app port
                   ↳ process.env.SPECULOS_API_PORT = String(previousPort)
```

Sixteen distinct things happen between the spec's one-liner and the test moving on. The reason to read this trace once with the layers attached is that, when something goes wrong, the failure mode usually points at a specific layer — and the first move is to open that layer's file.

A tour of the common failure shapes:

- **"Cannot find module .../apps/cli/bin/index.js"** → Layer 1 not built. Run `pnpm --filter @ledgerhq/live-cli build`.
- **"❌ Failed to execute CLI command ... Exit Code: 1"** → Layer 1 returned non-zero. Read the `🧾 CLI Error` line in the envelope to see what the CLI itself logged.
- **"⚠️ CLI attempt 1/3 ... retrying in 3000ms"** → Layer 2 retry policy fired. Transient. If it happens 3 times, the underlying error is no longer transient.
- **"REVIEW_TRANSACTION_TO not found within timeout"** → Layer 4 device tap could not find a screen label. Either the device firmware changed (label drift), the CLI subprocess never reached the signing screen (a Layer 1 problem), or `SPECULOS_API_PORT` is pointing at the wrong device (a Layer 5 / Chapter 6.5 problem).
- **"Speculos not started"** → Layer 5 / Chapter 6.5. `startSpeculos` could not boot a container. Local Docker issue, or remote pool exhaustion.
- **The test passes but the next test runs against the wrong env** → `setDisableTransactionBroadcastEnv` did not restore. Almost always means a try/finally was forgotten in a new helper. Chapter 6.6.

### 6.4.8 Where the Layers Are Imported From

The layer-3 helpers are the single point of contact between specs and the CLI. Here is the catalogue of how they are consumed across the desktop test workspace today, which should anchor any new helper you add in real usage patterns:

| Spec | Helpers imported |
|---|---|
| `tests/page/swap.page.ts` | `approveTokenCommand`, `isTokenAllowanceSufficientCommand`, `revokeTokenCommand` |
| `tests/specs/earn.v2.spec.ts` | `liveDataCommand`, `liveDataWithAddressCommand` |
| `tests/specs/newSendFlow.tx.spec.ts` | `liveDataWithRecipientAddressCommand` |
| `tests/specs/receive.address.spec.ts` | `liveDataCommand` |
| `tests/specs/validation.swap.spec.ts` | `liveDataWithAddressCommand` |
| `tests/specs/earn.spec.ts` | `liveDataCommand`, `liveDataWithAddressCommand` |
| `tests/specs/rename.account.spec.ts` | `liveDataCommand` |
| `tests/specs/delete.account.spec.ts` | `liveDataCommand` |
| `tests/specs/settings.spec.ts` | `liveDataCommand` |
| `tests/specs/delegate.spec.ts` | `liveDataCommand` |
| `tests/specs/entrypoint.swap.spec.ts` | `liveDataWithAddressCommand` |
| `tests/specs/accounts.swap.spec.ts` | `liveDataWithAddressCommand` |
| `tests/specs/subAccount.spec.ts` | `liveDataCommand`, `liveDataWithAddressCommand`, `liveDataWithParentAddressCommand` |
| `tests/specs/buySell.spec.ts` | `liveDataCommand` |
| `tests/specs/activate.private.balance.spec.ts` | `liveDataCommand` |
| `tests/specs/send.swap.spec.ts` | `liveDataWithAddressCommand` |
| `tests/specs/ui.swap.spec.ts` | `liveDataWithAddressCommand` |
| `tests/specs/provider.swap.spec.ts` | `liveDataWithAddressCommand` |
| `tests/specs/send.tx.spec.ts` | (multiple) |

The pattern is overwhelmingly **`liveDataCommand` plus, optionally, an address-resolving variant.** The vast majority of specs need exactly one thing from Layer 3: "seed the account, optionally also tell me its device-derived address." The approve/revoke pair is, today, used in only one file — `swap.page.ts` — because token-allowance is a swap-specific concern. As QAA expands its broadcast-enabled coverage to staking, NFT, and lending flows, expect `approveTokenCommand` and `revokeTokenCommand` (and possibly siblings yet to be written) to spread.

The mobile workspace consumes the same helpers but through a different mechanism. `e2e/mobile/jest.environment.ts` line 29 imports `* as cliCommandsUtils`, and line 138 does `Object.assign(this.global, cliCommandsUtils)`. Every helper becomes a Jest global. Type definitions live at `e2e/mobile/types/global.d.ts:104-111`. A mobile spec calls e.g. `await liveDataCommand(account)(userdataPath)` *without an import statement*, because the helper is already on the global. This is a deliberate divergence between the two workspaces: desktop uses ESM imports (Playwright/Vitest convention); mobile uses globals (Jest convention, easier interop with Detox's existing globals).

The desktop fixture engine is the other consumption path. `e2e/desktop/tests/fixtures/common.ts` consumes the `cliCommands: [...]` field from each test's fixture data and runs each curried function in the test setup phase. A typical use:

```ts
// from a hypothetical earn spec
test.use({
  cliCommands: [liveDataCommand(account)],
  userdataFile: "skip-onboarding",
});
```

The fixture engine, reading the array, calls each entry with the resolved userdata path. The currying on `liveDataCommand` is what makes this work — the spec author binds the account at write time, the engine binds the userdata path at run time.

### 6.4.9 Error Propagation

Each layer has a different mode of failure and a different recovery story:

**Layer 1 (CLI exits non-zero).** The CLI process writes its error to stderr and exits with a non-zero code. Layer 2's `child.on("exit", code => …)` handler builds the multi-line error envelope (`❌ Failed to execute CLI command`, currency, index, exit code, trimmed stderr) and rejects the promise. The error envelope is the most actionable single artefact for a CLI failure — paste it into a Jira ticket as-is.

**Layer 2 (retryable error).** `runCliCommandWithRetry` catches the rejection from `runCliCommand`, consults `isRetryableError` against the error message, and either retries (up to 3 attempts, 3-second delay) or rethrows. The retry loop is silent on success and emits a `⚠️ CLI attempt N/3 …` warning on each retry. If you see two warnings followed by success, the underlying issue is transient and probably acceptable; if you see three followed by failure, the issue has a non-retryable root cause hiding behind a retryable signature, and the final exception is the one that matters.

**Layer 3 (env restore failure).** If `setDisableTransactionBroadcastEnv` is called but the `finally` block is omitted, an exception in the helper body (the device tap, typically) leaves `process.env.DISABLE_TRANSACTION_BROADCAST` mutated for the rest of the run. Subsequent CLI calls — even ones that have nothing to do with approvals — will see the wrong broadcast state and silently behave differently. The `try/finally` in `approveTokenCommand` and `revokeTokenCommand` is the defensive pattern that prevents this. Any new helper that flips an env var must do the same. Chapter 6.6 covers the discipline in full.

**Layer 4 (device tap timeout).** The screen primitives (`waitFor`, `pressUntilTextFound`) reject when their internal timeout elapses. A common cause is firmware label drift: the device renders "Sign Transaction" with a capital T but `DeviceLabels.SIGN_TRANSACTION` was registered as "Sign transaction". Another common cause is the CLI subprocess not actually reaching the signing screen — usually because of a Layer 1 argument error that takes the CLI down a different code path. If the device tap rejects, the CLI subprocess (Layer 1) is still blocked waiting for an APDU; it will hang until its own `wait-confirmation-timeout` (default 120 seconds) expires. The two timeouts together cap the total failure time at around two minutes.

**Layer 5 (Speculos cleanup miss).** If a POM method's `finally` is omitted (or the `cleanSpeculos` call inside it fails silently), the Docker container leaks and the transport-module registry retains a stale entry. Subsequent tests in the same process may pick up the stale Speculos port via `getEnv("SPECULOS_API_PORT")` and route their CLI calls to a dead device. Chapter 6.5 covers the lifecycle hygiene in full and shows exactly why the snapshot-and-restore pattern matters.

The general rule: errors propagate up through the layers, but each layer adds context. By the time an exception reaches the test runner, it should carry enough information (the error envelope from Layer 2, the helper name from Layer 3, the POM step name from Layer 5's `@step` decorator) for a triage engineer to know which file to open without re-running anything.

### 6.4.10 The Complete File List

Here is every file a Layer 5 author might touch, ranked by frequency:

**Frequent — one or both of these per ticket:**
- `e2e/desktop/tests/page/swap.page.ts`, `e2e/desktop/tests/page/<feature>.page.ts` — POM methods.
- `e2e/mobile/page/trade/swap.page.ts`, `e2e/mobile/page/<feature>/<feature>.page.ts` — mobile POM methods.
- `e2e/desktop/tests/specs/*.spec.ts`, `e2e/mobile/specs/**/*.spec.ts` — the test files themselves.
- `e2e/desktop/tests/fixtures/common.ts` (occasionally) — when adding a new `cliCommands`-style fixture field.
- `e2e/desktop/tests/userdata/*.json`, `e2e/mobile/userdata/*.json` — when a new account or feature flag is needed.

**Occasional — a few times a quarter:**
- `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts` — Layer 3 helper extensions for new tickets.
- `libs/ledger-live-common/src/e2e/enum/Account.ts`, `Currency.ts`, `Provider.ts` — when a ticket introduces a new currency, account fixture, or swap provider.

**Rare — once or twice a year for a typical QA engineer:**
- `apps/cli/src/commands/blockchain/*.ts` — only when adding a brand-new CLI subcommand. Coordinate with the platform team; this is shared with non-QA consumers.
- `libs/ledger-live-common/src/e2e/runCli.ts` — engine extensions (a new public engine helper, a new retry pattern). Almost always platform-team work, not QA.
- `libs/ledger-live-common/src/e2e/families/evm.ts` — new device-tap pattern (a new transaction type with a new screen flow). Coordinate with the firmware team for the screen labels.
- `libs/ledger-live-common/src/e2e/speculos.ts`, `speculosCI.ts`, `speculosAppVersion.ts` — Speculos lifecycle and version detection. Almost never QA work; covered in Chapter 6.5.

Touching a file from the "rare" tier should always come with a code review request from the senior engineers. Touching a file from the "frequent" tier should be a routine PR.

### 6.4.11 Cross-Reference With Part 4 and Part 5

This chapter sits between the desktop and mobile codebase deep dives. Part 4 Chapter 4.6 catalogues `e2e/desktop/`; Part 5 Chapter 6.7 catalogues `e2e/mobile/`. Both workspaces describe their own page objects, fixtures, runners, and configurations in detail. **The CLI integration layer this chapter describes lives below both of them.**

The crucial fact for a new QA engineer is this: layers 1 through 4 are *one* codebase, used identically by both workspaces. There is no desktop CLI helper and a separate mobile CLI helper. There is no Playwright-specific spawn engine and a separate Detox-specific spawn engine. `cliCommandsUtils.ts` is imported by Playwright specs through ESM and by Jest specs through Jest globals, but it is the same TypeScript file. Bug-fix one helper, both workspaces benefit. Add one helper, both can use it.

Layer 5 is where the workspaces diverge:

- **Desktop POM methods** live in `e2e/desktop/tests/page/*.page.ts`. They use Playwright's `Page` API, the `@step(...)` decorator from `tests/misc/reporters/step`, and Allure via `allure-js-commons`.
- **Mobile POM methods** live in `e2e/mobile/page/<feature>/*.page.ts`. They use Detox's matcher API (`element(by.id(...))`, `device.launchApp(...)`), often a different `@step` implementation, and the same Allure library.

But both call into the same Layer 3 helpers. When you read `e2e/mobile/page/trade/swap.page.ts` and see `await revokeTokenCommand(account, spender)`, that is the *same function* the desktop swap page object calls. The shared boundary is healthy infrastructure: it forces the lower layers to stay framework-agnostic, and it lets QA engineers move between desktop and mobile tickets without re-learning the CLI integration.

The one place to be careful: when a Layer 3 helper takes a curried form (`liveDataCommand(account)(userdataPath)`), the desktop fixture engine applies the second argument for you, but the mobile global usage requires you to apply both arguments by hand. Read the file you are working in. If you see `cliCommands: [...]` in a fixture, you are on desktop; if you see `await liveDataCommand(account)(userdataPath)` directly in a spec, you are on mobile.

<div class="quiz-container" data-pass-threshold="80">
  <h3>Chapter 6.4 Quiz</h3>
  <p class="quiz-subtitle">6 questions · 80% to pass</p>
  <div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

  <div class="quiz-question" data-correct="C">
    <p><strong>Q1.</strong> Which of the five layers is the typed wrapper layer where most new per-ticket helpers are added?</p>
    <div class="quiz-choices">
      <button class="quiz-choice" data-value="A">A) Layer 1 — apps/cli/bin/index.js</button>
      <button class="quiz-choice" data-value="B">B) Layer 2 — runCli.ts</button>
      <button class="quiz-choice" data-value="C">C) Layer 3 — cliCommandsUtils.ts</button>
      <button class="quiz-choice" data-value="D">D) Layer 4 — families/evm.ts</button>
    </div>
    <p class="quiz-explanation">Layer 3 (<code>cliCommandsUtils.ts</code>) is the typed-wrapper layer. It exports <code>liveDataCommand</code>, <code>approveTokenCommand</code>, <code>revokeTokenCommand</code>, and friends. Layers 1, 2, and 4 are infrastructure that QA engineers extend rarely.</p>
  </div>

  <div class="quiz-question" data-correct="B">
    <p><strong>Q2.</strong> Why is <code>liveDataCommand</code> a curried function (<code>(account, options?) =&gt; (userdataPath?) =&gt; Promise&lt;void&gt;</code>) instead of a single-call function?</p>
    <div class="quiz-choices">
      <button class="quiz-choice" data-value="A">A) Because TypeScript performance is faster with curried generics</button>
      <button class="quiz-choice" data-value="B">B) So the spec author binds the account at write time and the fixture engine binds the userdata path at run time</button>
      <button class="quiz-choice" data-value="C">C) Because Playwright fixtures only accept thunks</button>
      <button class="quiz-choice" data-value="D">D) To prevent the helper from being awaited twice</button>
    </div>
    <p class="quiz-explanation">The currying lets the fixture engine schedule the call at the right point in the test lifecycle. The spec writes <code>cliCommands: [liveDataCommand(account)]</code>; the engine, knowing the userdata path, calls each entry with that path. This is the single most important pattern in <code>cliCommandsUtils.ts</code>.</p>
  </div>

  <div class="quiz-question" data-correct="A">
    <p><strong>Q3.</strong> What is the retry policy in <code>runCliCommandWithRetry</code>?</p>
    <div class="quiz-choices">
      <button class="quiz-choice" data-value="A">A) Up to 3 attempts, 3-second delay between attempts, only retryable error patterns trigger a retry</button>
      <button class="quiz-choice" data-value="B">B) Unlimited attempts with exponential backoff</button>
      <button class="quiz-choice" data-value="C">C) Up to 5 attempts, 1-second delay, retries on every error</button>
      <button class="quiz-choice" data-value="D">D) Single attempt; the caller is responsible for retrying</button>
    </div>
    <p class="quiz-explanation">The defaults are <code>retries = 3</code> and <code>delayMs = 3000</code>. Only error messages matching the patterns in <code>isRetryableError</code> (HTTP 502/503/504, <code>GeneralDmkError</code>, <code>ECONNREFUSED</code>, <code>ETIMEDOUT</code>, <code>ECONNRESET</code>, "socket hang up", "timeout") trigger a retry; deterministic logic errors fail fast.</p>
  </div>

  <div class="quiz-question" data-correct="D">
    <p><strong>Q4.</strong> What does the constant <code>LEDGER_LIVE_CLI_BIN</code> resolve to at runtime?</p>
    <div class="quiz-choices">
      <button class="quiz-choice" data-value="A">A) The path to <code>apps/cli/src/index.ts</code></button>
      <button class="quiz-choice" data-value="B">B) The npm-published binary at <code>node_modules/.bin/ledger-live</code></button>
      <button class="quiz-choice" data-value="C">C) Whatever <code>which ledger-live</code> returns on the host</button>
      <button class="quiz-choice" data-value="D">D) An absolute path to <code>apps/cli/bin/index.js</code> in the monorepo, computed from <code>__dirname</code> with four <code>..</code> segments</button>
    </div>
    <p class="quiz-explanation">The line is <code>path.resolve(__dirname, "../../../../apps/cli/bin/index.js")</code>. It walks back from the location of the built <code>runCli.js</code> to the monorepo root, then forward into the CLI's built <code>bin/index.js</code>. If the CLI is not built, the constant points at a non-existent file and spawn fails.</p>
  </div>

  <div class="quiz-question" data-correct="B">
    <p><strong>Q5.</strong> In the end-to-end trace of <code>revokeTokenApproval</code>, what runs <em>in parallel</em> with the spawned CLI subprocess while the device is being signed?</p>
    <div class="quiz-choices">
      <button class="quiz-choice" data-value="A">A) Nothing — the CLI subprocess runs alone and the device tap runs after it exits</button>
      <button class="quiz-choice" data-value="B">B) The Layer 4 device tap (<code>approveToken</code>), which drives Speculos's REST API to walk through the review screens and press the buttons</button>
      <button class="quiz-choice" data-value="C">C) A second CLI subprocess that pushes button events</button>
      <button class="quiz-choice" data-value="D">D) The Allure reporter, which records APDU traffic in real time</button>
    </div>
    <p class="quiz-explanation">The CLI subprocess sits inside <code>bridge.signOperation(...)</code> waiting for the device's APDU response. While it waits, the layer-3 helper <code>await</code>s <code>approveToken()</code>, which is the Layer 4 device tap driving Speculos via REST to render screens and press buttons. They are two cooperating coroutines; both must complete for the whole flow to succeed.</p>
  </div>

  <div class="quiz-question" data-correct="C">
    <p><strong>Q6.</strong> Inside <code>revokeTokenCommand</code>, why is the call to <code>setDisableTransactionBroadcastEnv(original)</code> wrapped in a <code>finally</code> block?</p>
    <div class="quiz-choices">
      <button class="quiz-choice" data-value="A">A) Because <code>finally</code> blocks run faster than non-finally blocks in V8</button>
      <button class="quiz-choice" data-value="B">B) Because <code>process.env</code> is read-only outside <code>finally</code></button>
      <button class="quiz-choice" data-value="C">C) Because if the device tap throws, the env var would otherwise stay flipped to <code>"0"</code> for the rest of the run, silently changing broadcast behaviour for every later CLI call</button>
      <button class="quiz-choice" data-value="D">D) To satisfy a TypeScript exhaustiveness check on the union type of the env value</button>
    </div>
    <p class="quiz-explanation">The <code>try/finally</code> is the safety net for env-var mutations. Without it, an exception between the flip and the restore would leak global state across tests. Chapter 6.6 covers the broadcast-discipline pattern in detail.</p>
  </div>

  <div class="quiz-score"></div>
</div>

<div class="chapter-outro">
You now have a layer-by-layer map of how a single line in a desktop spec — <code>await app.swap.revokeTokenApproval(account, provider)</code> — descends through five distinct files, spawns a Node subprocess, drives a Speculos REST API in parallel with that subprocess, and reaches an on-chain broadcast on Ethereum mainnet. The next chapter zooms in on one of the supporting cast members of this trace: the Speculos lifecycle. <code>launchSpeculos</code> and <code>cleanSpeculos</code> appeared at the top and bottom of every Layer 5 method in this chapter; Chapter 6.5 takes them apart, shows what <code>startSpeculos</code> does inside (Docker container or remote pool), explains the <code>SPECULOS_API_PORT</code> snapshot-and-restore pattern in full, and gives you the playbook for debugging Speculos boot failures.
</div>

## Speculos Lifecycle in CLI Tests

<div class="chapter-intro">
The CLI subprocess and the Speculos device-tap automation run in parallel during every approve or revoke test. The CLI is blocked waiting for device confirmation; the Speculos helper drives the simulator's REST API as if a human were pressing buttons. Both processes need to agree on which Speculos instance is the device — and that agreement is brokered by a small, careful pair of utility functions in <code>e2e/desktop/tests/utils/speculosUtils.ts</code>. This chapter walks through that pair, the snapshot-and-restore pattern that lets a single test orchestrate two Speculos instances, the local-versus-remote toggle, and the failure modes you will hit if you skip cleanup.
</div>

### 6.6.1 Why Speculos Lifecycle Matters in CLI Tests

A CLI test that signs a transaction has two parallel actors:

- **The CLI subprocess** — `node apps/cli/bin/index.js send --currency ethereum --mode revokeApproval ...` — spawned by `runCliCommand`, blocked at `bridge.signOperation` waiting for `hw-transport-node-speculos-http` to return signed bytes.
- **The Speculos device-tap helper** — `approveToken()` from `families/evm.ts` — driving the Speculos REST API, polling screen text, pressing buttons or doing the touch-screen hold.

Both actors need to talk to the same Speculos container. They identify it by port. The lifecycle utilities are responsible for:

1. Starting Speculos with the right firmware and app for the test.
2. Telling the CLI subprocess which port to use (via env var inheritance and explicit transport registration).
3. Telling the device-tap helper which port to use (same env var, read directly).
4. Stopping Speculos and unregistering its transport when the test is done.

If any one of those steps is wrong or skipped, you get one of: a transport leak (next test picks up a stale entry), a port collision (`startSpeculos` errors out at setup), a stopped-Speculos timeout (test hangs waiting for a container that no longer exists), or a Docker container leak (the previous Speculos keeps running and burning memory until the worker is recycled).

Most QAA spec writers don't think about this layer day-to-day — they call `launchSpeculos` and `cleanSpeculos` and trust the rest. But when something goes wrong, you need to know what these functions do. This chapter is that knowledge.

### 6.6.2 The Two Functions

Both live in `e2e/desktop/tests/utils/speculosUtils.ts`. Here they are verbatim.

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

About thirty lines of code. Every line earns its place — there is no spare ceremony here. The next subsections walk through each function step by step.

### 6.6.3 What `launchSpeculos` Does, Step by Step

Signature:

```ts
launchSpeculos(appName, testTitle?, previousDevice?): Promise<SpeculosDevice>
```

The three parameters:

- `appName` — string key into the `specs` map. `"Ethereum"`, `"Bitcoin"`, `"Solana"`, etc. Spaces are replaced with underscores before lookup, so `"Exchange App"` becomes `"Exchange_App"` for the spec key.
- `testTitle` — optional human-readable label that gets passed to `startSpeculos` as the container name suffix. Spaces are replaced with underscores. Used so that `docker ps` shows readable names while a test suite is running. Defaults to `"cli_speculos"` if omitted.
- `previousDevice` — optional `SpeculosDevice` from a prior `launchSpeculos` call. If passed, it is cleaned up first (see step 1 below) and its port is reused for the new device.

Step by step:

**1. Sanitise testTitle.** `testTitle.replace(/ /g, "_")` — Docker container names cannot contain spaces.

**2. Clean up previousDevice if given.** `if (previousDevice) await cleanSpeculos(previousDevice);` — this is the one-line "swap" pattern. If a test had Speculos A running and now needs Speculos B, the helper stops A and unregisters its transport before starting B. Importantly, this is the only branch that performs cleanup automatically; if you call `launchSpeculos` twice without passing `previousDevice` to the second call, the first one keeps running and you leak a container.

**3. Start the new Speculos.** `startSpeculos(testTitle ?? "cli_speculos", specs[appName.replace(/ /g, "_")], previousDevice?.port)` — this is the heavy lift. It pulls the spec for the requested app (firmware path, app path, model id), then either spawns a Docker container locally or asks the remote Stargate pool for a Speculos instance. `previousDevice?.port` is passed as a port hint — if a previous device existed, the helper tries to reuse its port for the new instance. This matters in the swap-then-revoke case: starting on the same port avoids a port reservation race. `startSpeculos` lives in `@ledgerhq/live-common/e2e/speculos` and returns a `SpeculosDevice` object with `id`, `port`, `appName`, `appVersion`, `dependencies`, and a few internal handles.

**4. Invariant check.** `invariant(device, "[E2E Setup] Speculos not started");` — fail loudly if `startSpeculos` returned undefined. This shouldn't happen in practice (it would throw before returning) but is a defensive belt-and-braces.

**5. Wait for readiness if remote.** `if (process.env.REMOTE_SPECULOS === "true") { await waitForSpeculosReady(device.id); }` — when running against the production CI Speculos pool, the device may not be immediately responsive after `startSpeculos` returns. `waitForSpeculosReady` polls the remote API until the container reports ready. Local Docker is reliably synchronous and skips this step.

**6. Set the port env var, twice.** `setEnv("SPECULOS_API_PORT", device.port);` and `process.env.SPECULOS_API_PORT = device.port.toString();` — both forms are written, defensively, because the codebase has two ways to read it: `getEnv("SPECULOS_API_PORT")` from `@ledgerhq/live-env` (used by the device-tap helper) and `process.env.SPECULOS_API_PORT` (used by anything that reads raw env). Setting both means whichever consumer reads first sees the right value. The CLI subprocess inherits `process.env` at spawn time, so it gets the right port automatically.

**7. Register the transport.** `CLI.registerSpeculosTransport(device.port.toString(), getSpeculosAddress());` — this tells the live-common transport registry that there is a `speculos-http-<port>` transport now available. The CLI subprocess, when it instantiates its `hw-transport-node-speculos-http`, looks up its config from the transport registry. Without registration, the CLI either uses a default port (wrong) or errors out. `getSpeculosAddress()` returns either `"localhost"` (Docker-local default) or the `SPECULOS_ADDRESS` env var when set (remote pool case).

**8. Write the Allure description.** Composes a string like `"SPECULOS\nApp: Ethereum (1.10.4)"` (plus a list of dependencies if the app has them — Ethereum loaded with the Plugin app for clear-signing has both shown). `allure.description` writes this to the test's Allure report so anyone investigating a failed test can immediately see which Speculos was active.

**9. Return the device handle.** The caller stores it for the matching `cleanSpeculos` in `finally`.

### 6.6.4 What `cleanSpeculos` Does

Signature:

```ts
cleanSpeculos(speculos, previousPort?)
```

Two parameters:

- `speculos` — the `SpeculosDevice` returned by `launchSpeculos`.
- `previousPort` — optional. The port that was in use before this Speculos was launched. When passed, the cleanup restores it.

Step by step:

**1. Stop Speculos.** `stopSpeculos(speculos.id)` — kills the Docker container locally, or asks Stargate to release the remote slot. Idempotent; calling stop twice on the same id is a no-op.

**2. Unregister the transport.** `unregisterTransportModule("speculos-http-" + String(speculos.port));` — removes the entry from the live-common transport registry. Without this step, the registry would keep accumulating dead `speculos-http-<port>` entries for every test in the suite. Eventually a later test, calling `registerSpeculosTransport` for a *different* port, would find a stale entry on the same port number first and route to the dead container. Test hangs, eventually times out.

**3. Restore previousPort if given.** `if (previousPort) { setEnv("SPECULOS_API_PORT", previousPort); process.env.SPECULOS_API_PORT = String(previousPort); }` — if the caller saved the port that was active before this Speculos was launched, restore it. This is the second half of the snapshot-and-restore pattern (see 6.5.6). Without restoration, after a sub-test inside a larger swap test cleans up its revoke Speculos, the env points to nothing and the outer swap test's next CLI call fails.

### 6.6.5 The `specs` Map

`specs` is imported from `@ledgerhq/live-common/e2e/speculos`. It is a flat object keyed by app name:

```ts
const specs = {
  Ethereum:    { firmware: "...", app: "...", model: "nanoSP", ... },
  Bitcoin:     { firmware: "...", app: "...", model: "nanoSP", ... },
  Solana:      { firmware: "...", app: "...", model: "nanoSP", ... },
  Exchange_App:{ firmware: "...", app: "...", model: "nanoSP", ... },
  ...
};
```

Each entry contains:

- `firmware` — path to the firmware ELF inside the Speculos Docker image. Different device models have different firmware paths.
- `app` — path to the app ELF. For Ethereum, this is the Ethereum app build for the matching firmware.
- `model` — the device model id (`"nanoSP"`, `"nanoX"`, `"stax"`, `"flex"`). Determines screen size, button layout, and the device-tap helper branch (`isTouchDevice()`).
- Optional `dependencies` — sibling apps that need to be loaded alongside (e.g. Plugin Ethereum for clear-sign decoders, or Exchange app for swap flows).

The map is the single source of truth for "what does the Ethereum app on Speculos look like for QAA tests." Updating to a new app version is a one-line change in the map, propagated to every test automatically.

The `appName.replace(/ /g, "_")` in `launchSpeculos` accommodates app names with spaces — `"Exchange App"` is keyed as `Exchange_App` in the map.

### 6.6.6 The Snapshot-and-Restore Pattern

This is the canonical wrapper around `launchSpeculos`/`cleanSpeculos` in any test that needs to interact with a Speculos device.

Here is `ensureTokenApproval` from `e2e/desktop/tests/page/swap.page.ts` — the existing twin of the new revoke method. Verbatim:

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

And here is the new `revokeTokenApproval`, also verbatim from the senior's commit:

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

The shape is identical:

```ts
const previousSpeculosPort = getEnv("SPECULOS_API_PORT");
const speculos = await launchSpeculos(fromAccount.currency.speculosApp.name);
try {
  // ... CLI command + device tap ...
} finally {
  await cleanSpeculos(speculos, previousSpeculosPort);
}
```

**Why this shape?**

The outer test (a swap spec, for example) is already running with one Speculos active — the Exchange app Speculos that the swap flow uses. When the test wants to do a token revoke, it needs the *Ethereum* app Speculos, not the Exchange app Speculos. So:

1. **Snapshot.** `getEnv("SPECULOS_API_PORT")` reads the currently active Speculos port (the swap's Exchange Speculos) into a local variable.
2. **Swap.** `launchSpeculos("Ethereum")` starts a new Speculos with the Ethereum app. The env var now points to the Ethereum Speculos. The CLI subprocess will pick this up.
3. **Use.** Inside the `try`, the `revokeTokenCommand` runs against the Ethereum Speculos. The CLI signs a revoke transaction; the device-tap helper drives the Ethereum app's screen flow.
4. **Restore.** `cleanSpeculos(speculos, previousSpeculosPort)` stops the Ethereum Speculos, unregisters its transport, and restores the env var to the original Exchange Speculos port. The outer swap test continues as if nothing happened.

**Why two Speculos instances at all?** Each Speculos container runs *one* app at a time. The Exchange app and the Ethereum app are different apps. The swap flow needs Exchange (because the swap is mediated through Ledger's Exchange app for the clear-sign decoder of swap-specific calldata); the revoke is a plain ERC-20 `approve(spender, 0)` on Ethereum which needs the Ethereum app. You can't time-share — the swap is mid-flow, holding state on the Exchange Speculos. So you start a second Speculos in parallel, do the revoke, tear it down, and the swap's Speculos is untouched.

**Why the `finally`?** Because if `revokeTokenCommand` throws — the device-tap fails, the broadcast errors out, the transaction reverts on-chain — the cleanup must still run. Otherwise the second Speculos leaks, the env var stays pointed at the dead container, and every subsequent CLI call in the outer test fails with a confusing transport error.

The pattern is dense but local: one snapshot variable, one launch, one `finally` block. Once you internalise it, every multi-Speculos test reads the same.

### 6.6.7 A Concrete Timeline — Swap with Mid-Test Revoke

Walk through the full timeline of a `provider.swap.spec.ts` run that calls `revokeTokenApproval` before exercising the swap. Every step matters; the order is what makes the orchestration safe.

**T=0.** Test starts. Test fixture sets up the desktop app, runs `liveDataWithAddressCommand` to seed the account. No Speculos yet. `SPECULOS_API_PORT` is unset (or set by a previous test that cleaned up correctly).

**T=1.** Test enters its body. Calls `await app.swap.swapAndCheck(...)`. The swap flow internally calls `launchSpeculos("Exchange App")`. At this point:
- `previousDevice` is undefined, so no cleanup branch runs.
- `startSpeculos("swap_test", specs.Exchange_App, undefined)` spawns a Docker container exposing the Speculos REST API on a fresh port — say port 41001.
- `SPECULOS_API_PORT` is set to `"41001"` (both via `setEnv` and `process.env`).
- `CLI.registerSpeculosTransport("41001", "localhost")` adds the entry `speculos-http-41001 → http://localhost:41001` to the live-common transport registry.
- Allure description gets `App: Exchange App (3.0.0)`.

The swap flow now drives the Exchange Speculos. The test holds the device handle in a variable like `swapSpeculos`.

**T=2.** Mid-test, before broadcasting the swap, the test wants to clear any leftover token allowance. It calls `await app.swap.revokeTokenApproval(fromAccount, provider)`. Inside that method:
- `previousSpeculosPort = getEnv("SPECULOS_API_PORT")` reads `"41001"` and stores it.
- `launchSpeculos(fromAccount.currency.speculosApp.name)` is called with `"Ethereum"` (assuming an EVM swap).
  - `previousDevice` not passed — the previous device is the *swap's Exchange Speculos*, but we don't want to stop it. We want it preserved. So we do *not* pass it as the third arg. Instead, the swap Speculos keeps running while we start a sibling.
  - `startSpeculos("cli_speculos", specs.Ethereum, undefined)` spawns a *second* container, on a *different* port — say 41002.
  - `SPECULOS_API_PORT` gets overwritten to `"41002"`.
  - `CLI.registerSpeculosTransport("41002", "localhost")` adds the `speculos-http-41002` entry. The registry now has both 41001 and 41002.
  - Allure picks up another description: `App: Ethereum (1.10.4)`. The test report shows both blocks.

**T=3.** The test calls `revokeTokenCommand(fromAccount, provider.contractAddress)`. This:
- Sets `DISABLE_TRANSACTION_BROADCAST="0"` (saving the previous value).
- Spawns `node apps/cli/bin/index.js send --currency ethereum --mode revokeApproval ...` via `runCliCommand`. The CLI subprocess inherits `process.env`, so it reads `SPECULOS_API_PORT=41002`. It instantiates `hw-transport-node-speculos-http`, looks up `speculos-http-41002` in the registry, finds the constructor we registered at T=2, and connects to `localhost:41002`. It begins signing.
- In parallel, the test code awaits `approveToken()` from `families/evm.ts`. That helper polls Speculos REST at port 41002 (read from `SPECULOS_API_PORT`), pages through the review screens, and presses both buttons (or holds-to-sign on touch devices). The CLI subprocess unblocks with signed bytes. CLI broadcasts the tx. CLI exits with code 0.

**T=4.** `revokeTokenCommand` restores `DISABLE_TRANSACTION_BROADCAST` to its prior value. Returns the CLI stdout.

**T=5.** Back in `revokeTokenApproval`, the `try` block ends successfully. `finally` runs: `cleanSpeculos(speculos, previousSpeculosPort)`:
- `stopSpeculos(speculos.id)` kills the Ethereum Speculos container at port 41002.
- `unregisterTransportModule("speculos-http-41002")` removes that entry.
- `previousSpeculosPort = "41001"` was passed, so `setEnv("SPECULOS_API_PORT", "41001")` and `process.env.SPECULOS_API_PORT = "41001"` restore the swap Speculos's port.

**T=6.** Control returns to the swap flow. The Exchange Speculos at port 41001 is still running. `SPECULOS_API_PORT` is back to `"41001"`. The transport registry still has `speculos-http-41001`. The next CLI call (or the next device-tap) inside the swap flow finds everything as it was at T=2, before the revoke nest. The swap proceeds, broadcasts, completes.

**T=7.** Swap test ends. The outer fixture or the swap flow cleans up the swap Speculos (`cleanSpeculos(swapSpeculos)` — no port restoration needed because at T=0 nothing was active). Container at 41001 stops. Registry is clean. `SPECULOS_API_PORT` reset.

The whole orchestration is fifteen lines of test code. The two utility functions handle every subtle bit. Once you trace this once, you'll trust the pattern.

### 6.6.8 Local Speculos vs Remote

`process.env.REMOTE_SPECULOS === "true"` is the toggle.

**Local mode (default).** `REMOTE_SPECULOS` unset or `"false"`. `startSpeculos` spawns a Docker container on the local machine. The container exposes the Speculos REST API on a host-bound port. `getSpeculosAddress()` returns `"localhost"`. This is what you get when you run `pnpm e2e:test` on your laptop, or when a CI runner is configured for self-hosted Speculos.

**Remote mode.** `REMOTE_SPECULOS=true`. `startSpeculos` does not spawn anything locally. Instead, it asks the Stargate-managed Speculos pool to allocate a slot. Stargate is the production CI Speculos cluster. The pool returns a host:port that points to a remote Speculos instance. `getSpeculosAddress()` returns whatever `SPECULOS_ADDRESS` is set to — typically the Stargate gateway hostname. `waitForSpeculosReady(device.id)` polls the remote API until the container reports ready; this extra wait exists because remote allocations have higher startup latency than local Docker.

The same code path produces the right device handle in both modes. The lifecycle utilities don't branch based on local vs remote; they only branch on the readiness wait. Everything else — port management, transport registration, env var setting, allure description — is identical.

The relevant env vars:

- `REMOTE_SPECULOS` — `"true"` to enable remote mode. Anything else is local.
- `SPECULOS_ADDRESS` — the host (or hostname) where Speculos is reachable. `"localhost"` for Docker-local; a Stargate-issued hostname for remote.
- `SPECULOS_API_PORT` — set by `launchSpeculos`; read by `cleanSpeculos`, by the device-tap helpers, and inherited by the spawned CLI subprocess.

For day-to-day spec writing, you don't think about local vs remote. You write the spec, it works in your local Docker setup, and it also works in CI with `REMOTE_SPECULOS=true`. The lifecycle layer absorbs the difference.

### 6.6.9 The Transport Registration Step

The single most subtle line in `launchSpeculos`:

```ts
CLI.registerSpeculosTransport(device.port.toString(), getSpeculosAddress());
```

What does it actually do?

`hw-transport-node-speculos-http` is a transport implementation in `@ledgerhq/hw-transport-node-speculos-http`. It is the bridge between live-common's transport API (`open`, `exchange`, `close`) and Speculos's HTTP API. To use it, code calls something like:

```ts
const transport = await SpeculosHttpTransport.open("speculos-http-<port>");
```

But `transport` resolution is dynamic — the `open` call goes through a registry of named transports, and `speculos-http-<port>` is dynamic per-instance. The CLI subprocess and the device-tap helper both need to find the right transport entry by name, but the name encodes the port, which is only known at runtime after Speculos starts.

`CLI.registerSpeculosTransport(port, address)` adds an entry to the live-common transport registry mapping the name `speculos-http-<port>` to a constructor that opens a connection to `<address>:<port>`. After this call, any code that does `transport.open("speculos-http-<port>")` succeeds.

The CLI subprocess inherits `process.env` (including `SPECULOS_API_PORT`) at spawn time. When the CLI internally instantiates its transport, it reads `SPECULOS_API_PORT`, builds the name `speculos-http-<port>`, and calls `open` on it. The registry entry registered just above is what makes this resolve to the right Speculos.

The cleanup counterpart, `unregisterTransportModule("speculos-http-" + port)`, removes that entry. If you skip the unregister step, the registry accumulates dead entries; if a later test happens to reuse a port number for a new Speculos, the registry might find the old (dead) entry first.

This subsystem is invisible to you when it works. When it breaks — when you see "transport not found" or "ECONNREFUSED on port XXXX" or a CLI hang waiting for a response that never comes — the transport registry is usually where the bug lives. Knowing the registry exists and how to inspect it is the difference between a one-hour debug session and a one-day one.

### 6.6.10 Error Modes

What happens when something goes wrong?

**Skipped `cleanSpeculos`.** The Speculos container keeps running locally (or the remote slot stays allocated). The transport registry retains the `speculos-http-<port>` entry. Two failure modes downstream:
- A subsequent test, on a fresh `launchSpeculos`, may receive the same port from the remote pool (or from local Docker port reuse) and find the stale entry resolved to a now-dead container. Mismatch, hang, timeout.
- Memory pressure: each leaked Docker Speculos eats hundreds of megabytes. Run a hundred tests with leaks and your CI worker runs out of RAM.

The correct fix is always `try`/`finally`. There is no scenario where you want to leak.

**Port collision on `startSpeculos`.** `startSpeculos` errors out with something like "port already in use." This usually means a previous test left its Speculos running and the new test got the same port assignment. Symptom: test fails on setup, before any spec code runs. Diagnosis: check `docker ps` for stale Speculos containers. Mitigation: kill them and restart; root cause is almost always a missing `cleanSpeculos`.

**Forgot to restore `SPECULOS_API_PORT`.** You called `cleanSpeculos(speculos)` without the second argument. The Speculos itself is correctly stopped; the env var is left pointing at the now-dead port. The next CLI call in the outer test reads `SPECULOS_API_PORT`, points the spawned subprocess at the dead port, and the CLI hangs trying to connect. Eventually times out. The subtle part: the failure is several lines after the missing argument, so reading the stack trace doesn't immediately point you at the bug. Mitigation: always pass `previousSpeculosPort` to `cleanSpeculos` if you snapshotted it.

**Two `launchSpeculos` calls without intermediate cleanup.** You called `launchSpeculos("Ethereum")`, then later `launchSpeculos("Bitcoin")` without cleaning up the first. If you passed the first device handle as the third argument (`previousDevice`) to the second call, the helper cleans up for you — that's why the parameter exists. If you didn't pass it, the first Speculos leaks. The Bitcoin one starts up fine but the Ethereum one keeps running until the worker is recycled. In CI, the leak is noticed when the next batch of tests spawn and the host runs out of free ports or memory. Mitigation: either pass `previousDevice`, or call `cleanSpeculos` explicitly between the two `launchSpeculos` calls.

**Transport registered but never unregistered.** Symptom: tests pass individually but the suite slows down over time as the transport registry grows. Eventually, port reuse plus a stale entry causes a confusing routing error. Mitigation: pair every `register` with an `unregister`. The cleanup helper does this for you when called.

**REMOTE_SPECULOS set incorrectly.** If `REMOTE_SPECULOS=true` is set but no Stargate pool is reachable, `waitForSpeculosReady` polls forever (or until the test timeout). If `REMOTE_SPECULOS=false` (or unset) but the local Docker daemon isn't running, `startSpeculos` errors immediately. Both produce loud, easy-to-diagnose failures. The dangerous case is mismatched `SPECULOS_ADDRESS` — pointing at a host that exists but isn't a Speculos pool. Then the calls succeed-then-fail mid-test in confusing ways. Mitigation: set both env vars together, never one without the other.

### 6.6.11 Allure Integration

The last few lines of `launchSpeculos`:

```ts
let info = `App: ${device.appName || ""} (${device.appVersion || ""})`;
if (device.dependencies?.length) {
  info += `\nDependencies: ${device.dependencies?.map(dep => dep.name + " (" + dep.appVersion + ")").join(", ") || ""}`;
}
await allure.description("SPECULOS\n" + info);
```

Every `launchSpeculos` writes a Speculos description block to the current test's Allure report. The block includes:

- The app name and version (e.g. `Ethereum (1.10.4)`).
- Any dependencies and their versions (e.g. for Ethereum with Plugin: `Ethereum (1.10.4)\nDependencies: Plugin Ethereum (1.4.0)`).

When a test fails, the Allure UI shows this block in the description tab. You see immediately which Speculos was active, which app version was loaded, and whether the test was using a particular plugin variant. For tests that launch two Speculos instances (the snapshot-and-restore case), each call contributes its own description block; the report shows both.

This is the single highest-leverage debugging artifact in the Speculos lifecycle. It has saved many hours that would otherwise have been spent re-running tests with verbose logs to figure out which app a failed signature came from.

A small operational note: `allure.description` is *additive* — calling it multiple times in the same test doesn't overwrite, it appends. So a snapshot-and-restore test gets both Speculos blocks in the description. Easy to read; nothing to configure.

<div class="chapter-outro">
<strong>Key takeaway:</strong> <code>launchSpeculos</code> and <code>cleanSpeculos</code> are thirty lines of glue holding the CLI subprocess and the device-tap helper to the same Speculos instance, across local and remote modes. The snapshot-and-restore pattern is what lets a swap test do a revoke mid-flow without disturbing the outer Speculos. Every step earns its place — skip one and you get container leaks, transport registry corruption, or hangs that look like they came from elsewhere. When in doubt, mirror the <code>ensureTokenApproval</code>/<code>revokeTokenApproval</code> shape exactly.
</div>

### 6.6.12 Quiz

<!-- ── Chapter 6.5 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Chapter 6.5 Quiz</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> Why does a swap-then-revoke test need TWO Speculos instances?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) For redundancy in case one Docker container crashes</button>
<button class="quiz-choice" data-value="B">B) To run the test twice in parallel</button>
<button class="quiz-choice" data-value="C">C) Each Speculos runs one app at a time — swap needs the Exchange app, revoke needs the Ethereum app</button>
<button class="quiz-choice" data-value="D">D) One handles signing, the other handles broadcasting</button>
</div>
<p class="quiz-explanation">A single Speculos container loads one app at a time. The swap flow holds state on the Exchange Speculos; the revoke needs an Ethereum Speculos. The snapshot-and-restore pattern lets the test launch a second Speculos for the revoke, then tear it down so the outer swap continues against its original Exchange Speculos.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> What does <code>unregisterTransportModule("speculos-http-" + port)</code> do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Stops the Speculos Docker container</button>
<button class="quiz-choice" data-value="B">B) Removes the named transport entry from the live-common transport registry, preventing later tests from picking up a stale entry pointing at a dead container</button>
<button class="quiz-choice" data-value="C">C) Closes the WebSocket bridge to the React Native app</button>
<button class="quiz-choice" data-value="D">D) Resets the SPECULOS_API_PORT env var</button>
</div>
<p class="quiz-explanation">The live-common transport registry maps names like <code>speculos-http-&lt;port&gt;</code> to constructors. <code>unregisterTransportModule</code> removes that mapping so the registry stays clean across tests. Stopping the container is <code>stopSpeculos</code>; the env reset is the third <code>cleanSpeculos</code> step.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> What happens if you forget to pass <code>previousSpeculosPort</code> to <code>cleanSpeculos</code> after snapshotting it at launch?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The Speculos is correctly stopped, but SPECULOS_API_PORT is left pointing at the dead port; subsequent CLI calls in the outer test connect to a stopped container and time out</button>
<button class="quiz-choice" data-value="B">B) Nothing — the cleanup is idempotent</button>
<button class="quiz-choice" data-value="C">C) The Docker container leaks but the test still passes</button>
<button class="quiz-choice" data-value="D">D) The transport registry is corrupted and the entire suite crashes</button>
</div>
<p class="quiz-explanation">The Speculos itself is fine — <code>stopSpeculos</code> always runs. But the env var restore is gated on <code>previousPort</code> being passed. Without restoration, the next CLI call uses the dead port. The failure happens several lines after the bug, making it tricky to trace.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> What is the role of the <code>specs</code> map in <code>launchSpeculos</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It defines test specifications and assertions</button>
<button class="quiz-choice" data-value="B">B) It is a list of Playwright fixtures used per test</button>
<button class="quiz-choice" data-value="C">C) It maps test titles to Allure descriptions</button>
<button class="quiz-choice" data-value="D">D) It maps app names (Ethereum, Bitcoin, etc.) to firmware path, app path, model id, and optional dependencies — the blueprint <code>startSpeculos</code> uses to launch the right device</button>
</div>
<p class="quiz-explanation">The <code>specs</code> map (imported from <code>@ledgerhq/live-common/e2e/speculos</code>) is the single source of truth for which firmware and app to launch per app name. Updating to a new app version means updating one entry in this map.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> What does <code>process.env.REMOTE_SPECULOS === "true"</code> change in the lifecycle flow?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It disables Speculos entirely and runs against a real device</button>
<button class="quiz-choice" data-value="B">B) It routes <code>startSpeculos</code> to the Stargate-managed remote Speculos pool, and adds a <code>waitForSpeculosReady</code> readiness poll because remote allocation has startup latency</button>
<button class="quiz-choice" data-value="C">C) It enables a verbose debug log mode</button>
<button class="quiz-choice" data-value="D">D) It runs every test twice for reliability</button>
</div>
<p class="quiz-explanation">Local mode (default) spawns Docker; remote mode asks Stargate for a slot. The only branch in the helper code is the <code>waitForSpeculosReady</code> call — port management, transport registration, env vars, and Allure are identical in both modes.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> Why must the launch/clean pair be wrapped in <code>try</code>/<code>finally</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because TypeScript syntax requires it for async functions</button>
<button class="quiz-choice" data-value="B">B) Because Allure can't write the description otherwise</button>
<button class="quiz-choice" data-value="C">C) Because if the inner CLI call throws, cleanup must still run — otherwise the Speculos container leaks, the env var stays pointed at a dead port, and the transport registry retains a stale entry</button>
<button class="quiz-choice" data-value="D">D) Because Docker requires explicit cleanup signals</button>
</div>
<p class="quiz-explanation">The <code>finally</code> block guarantees cleanup in both the success and exception paths. Skip the <code>finally</code> and any thrown error from the device-tap or CLI subprocess leaks the container, leaves SPECULOS_API_PORT corrupted, and pollutes the transport registry — symptoms that show up in *later* tests, not this one.</p>
</div>

<div class="quiz-score"></div>
</div>

---

With the Speculos lifecycle internalised, the next stop is the final piece of the broadcast picture: `DISABLE_TRANSACTION_BROADCAST`, the env-var gate that determines whether a CLI-signed transaction actually hits the network or just produces a signed-bytes envelope. Chapter 6.6 walks through that gate, the discipline of try/finally around it, and why mobile and desktop have different defaults.
## DISABLE_TRANSACTION_BROADCAST and Broadcast Discipline

<div class="chapter-intro">
The previous chapter showed how Speculos runs alongside the CLI subprocess and how the two share an HTTP port via the transport registry. This chapter zooms in on a single environment variable — <code>DISABLE_TRANSACTION_BROADCAST</code> — and on the discipline that surrounds it. The variable is one of the smallest pieces of state in the whole CLI integration, but it is also one of the most consequential: it is the single switch that decides whether a signed transaction lands on a real testnet or evaporates as a string in a CLI log line. Get it right and your tests are deterministic, cheap, and reproducible. Get it wrong and you either pay gas you did not budget for or, worse, leave on-chain state behind that breaks the next test in the suite.
</div>

### 6.6.1 Why Broadcast Control Matters in QA

There is a fundamental tension at the heart of every signing test in Ledger Live's QAA suite. On one side, automated tests want to be **deterministic** — every run should start from the same state and produce the same observable outcome. On the other side, the artefacts that signing tests verify (signed payload bytes, device-screen renderings, intermediate Bridge events) generally do not require the transaction to actually be sent. So the simplest, fastest, cheapest behaviour is: **sign the transaction, capture the signed bytes, throw them on the floor**. No gas, no pending mempool entries, no chain state to worry about.

Most QAA tests work exactly this way. A send-flow regression checks that the device shows the correct recipient, the correct amount, the correct fee — once the user (Speculos) approves, the test asserts on the signed event and exits. There is no need for the transaction to ever touch a node.

But a small and important class of tests is fundamentally different. **Allowance state lives on-chain.** When you call `approve(spender, N)` on an ERC-20 contract, the new allowance is written to the token's storage and stays there until another `approve` overwrites it. If your test broadcasts an approve in run #1 and then run #2 starts up expecting `allowance == 0`, you have a stale-state bug that no amount of seed-resetting in your test runner can fix. The seed is local; the allowance is global. The chain remembers.

That is the tension. Most tests want broadcast off so they are cheap and stateless. Allowance-bearing tests need broadcast on so the test seed can actually be reset to a known allowance value (typically zero) between runs. The wrong default for either category breaks the suite.

The way the codebase resolves this tension is **broadcast is off by default per-test (via the env var) and explicitly turned on for the brief window where it is needed (via the helper)**. The helper restores the previous value on its way out so the surrounding test environment is unchanged. This chapter walks through the mechanics: what values mean what, where the variable is read, where it is set, and the try/finally discipline that keeps the whole system honest.

### 6.6.2 The Env Var Values

`DISABLE_TRANSACTION_BROADCAST` is a string. It can hold three meaningful states:

| Value | Meaning | Effect on `send` |
|---|---|---|
| `"1"` (or any truthy string except `"0"`) | Broadcast disabled | CLI signs, prints the signed event, **does not** call `bridge.broadcast` |
| `"0"` | Broadcast enabled | CLI signs, then submits the signed payload via `bridge.broadcast` |
| `undefined` (unset) | Falsy → broadcast enabled by default | Same as `"0"` |

> **Verify:** the actual coercion is `getEnv("DISABLE_TRANSACTION_BROADCAST")` returning a string that is then evaluated for truthiness in a JavaScript `||` chain. Empty string and `"0"` are both falsy in this context (`!!"0"` is `true` in plain JS, but `live-env`'s typed `getEnv` returns the value coerced through its declared schema, which for boolean-like envs treats `"0"` as `false`). When in doubt, set `"1"` to disable and `"0"` to enable, and never rely on "unset implies disabled".

The variable lives in two places at once. `live-env`'s internal config registry holds the in-process value that any live-common module reads via `getEnv("DISABLE_TRANSACTION_BROADCAST")`. The OS-level `process.env` holds the value that subprocesses inherit when the test spawns the CLI. The next subsections explain why both matter and which code reads from which.

The default state — **unset** — means `getEnv` falls back to whatever schema default `live-env` declares. In the QAA suite the practical baseline you see when running locally is "broadcast happens", because tests that need broadcast off set it explicitly to `"1"` and tests that need it on set it explicitly to `"0"`. The unset case is the no-test, no-helper path — a stray `runCli` call from a debug scratchpad would broadcast for real.

### 6.6.3 The End-to-End Flow

The single most important snippet in this whole chapter is the rxjs gate inside `apps/cli/src/commands/blockchain/send.ts`. It is short enough to quote in full and worth reading line by line:

```ts
.pipe(
  // ...
  bridge.signOperation({ account, transaction: t, deviceId: opts.device || "" })
    .pipe(
      map(toSignOperationEventRaw),
      // ── BROADCAST GATE ──
      ...(opts["disable-broadcast"] || getEnv("DISABLE_TRANSACTION_BROADCAST")
        ? []
        : [
            concatMap((e) => {
              if (e.type === "signed") {
                return from(
                  bridge.broadcast({ account, signedOperation: e.signedOperation })
                    // ...
                );
              }
              return of(e);
            }),
          ]),
      // ...
    );
```

**Reading it line by line:**

1. `bridge.signOperation({...})` is called first. This produces a stream of events: typically `device-permission-requested`, `device-permission-granted`, `device-streaming` (chunked transmission), and finally `signed`. The signed event carries the signed payload bytes and the optimistic operation. **Signing happens regardless of the broadcast gate.** The gate decides only what happens to the signed bytes after they exist.

2. `.pipe(map(toSignOperationEventRaw), ...)` converts the events into a serialisable shape. This is what the CLI prints to stdout.

3. The `...(condition ? [] : [...])` is JavaScript's spread-into-an-array trick used to **conditionally include an rxjs operator**. If the condition is truthy, the spread expands `[]` (zero operators added). If the condition is falsy, the spread expands a single-element array containing one `concatMap` operator.

4. The condition is `opts["disable-broadcast"] || getEnv("DISABLE_TRANSACTION_BROADCAST")`. **Two gates** — the CLI flag `--disable-broadcast` AND the environment variable. Either truthy is enough to skip broadcast. This redundancy is intentional: the flag lets a CLI invocation disable broadcast inline (useful for one-off manual testing), while the env var lets a test harness disable broadcast for every CLI call in a test file without rebuilding the command line for each one.

5. When the gate evaluates as **broadcast skipped** (truthy), the pipeline ends after the `map(toSignOperationEventRaw)`. The signed event is printed and the rxjs observable completes. The signed bytes are never submitted to a node. The CLI exits cleanly.

6. When the gate evaluates as **broadcast enabled** (falsy), the `concatMap` operator is inserted. For each emitted event:
   - If it is the `signed` event, call `bridge.broadcast({ account, signedOperation: e.signedOperation })`. Convert the resulting promise to an observable with `from(...)`. The downstream pipeline waits for the broadcast to complete before continuing.
   - Otherwise (any non-signed event — device permissions, streaming, etc.), pass it through unchanged via `of(e)`.

The fact that **the gate is at the operator-injection level, not inside the operator itself**, is why the disabled path produces zero overhead — there is literally no broadcast operator in the pipeline, so there is nothing to short-circuit. This is rxjs idiomatic; it is also the reason a debugger break inside the broadcast block reveals nothing when the gate is closed: that code never ran.

### 6.6.4 Read Sites in live-common

The variable is read in nine places across the monorepo. The CLI reads it once, in the file you just walked through. Live-common reads it in eight more places — every site where signing produces something the consumer might either submit on-chain or hold for later.

**CLI commands:**
- `apps/cli/src/commands/blockchain/send.ts` — the rxjs gate quoted verbatim above.
- `apps/cli/src/commands/blockchain/botTransfer.ts` — the bot's transfer command uses the same gate idiom, because the bot also signs first and broadcasts second; QAA does not invoke the bot directly but the symmetry is worth knowing about.

**live-common modules (`libs/ledger-live-common/src/`):**
- `exchange/swap/setBroadcastTransaction.ts` — the swap-specific path. Inside the swap acceptance flow, after the user signs the on-chain swap transaction, this module is responsible for broadcasting it. It checks `DISABLE_TRANSACTION_BROADCAST` to decide whether to actually call the node or return a simulated success. This is the path mobile swap tests exercise.
- `wallet-api/react.ts`, `wallet-api/useDappLogic.ts`, `wallet-api/ACRE/server.ts` — three files in the Live Apps signing path. When a Live App requests a transaction signature via the wallet API, these sites consult the env var to decide whether to broadcast or to return only the signed bytes back to the dApp. Broadcast-disabled Live App tests rely on this gate to assert what the dApp would see post-sign without actually sending the transaction.
- `hooks/useBroadcast.ts` — the React hook used by both desktop and mobile UI for the in-app "broadcast this signed tx" step. The same env-var check exists here so that UI-layer e2e tests can sign through the UI and assert on intermediate state without paying gas.
- `bot/engine.ts` — the long-running bot engine; same gate, same reasoning as `botTransfer.ts`.

The pattern is consistent: **wherever a signed payload is about to leave the process**, there is an env-var check. There is no central enforcer. Each broadcast-capable site owns the check.

### 6.6.5 Set Sites

Reading is one half; writing is the other. Three groups of code write the variable.

**Canonical helper — `setDisableTransactionBroadcastEnv` in `cliCommandsUtils.ts`.** This is the only setter QAA test code should call. Its signature is:

```ts
setDisableTransactionBroadcastEnv(value: string | undefined): string | undefined
```

It writes to **both** `live-env` (via `setEnv`) and `process.env` (so spawned subprocesses inherit the new value). It returns the **previous** value so the caller can restore it. That return value is the cornerstone of the try/finally discipline (next subsection).

**Mobile swap module-load setter — `e2e/mobile/specs/swap/otherTestCases/swap.other.ts`.** Mobile swap tests default-off broadcast at module load:

```ts
setEnv("DISABLE_TRANSACTION_BROADCAST", true);
```

This is fired once when the spec file is loaded. Every CLI/Bridge call originating from the spec inherits the disabled state. The mobile swap suite is the highest-volume consumer of CLI calls outside the desktop suite, and it relies on this default to keep the suite reproducible without per-test boilerplate.

**The new `revokeTokenCommand` and the existing `approveTokenCommand`.** Both are token-allowance helpers; both **need** broadcast on for their work and **must not** leave broadcast on when they exit. They follow the canonical pattern verbatim:

```ts
const original = setDisableTransactionBroadcastEnv("0");

const result = runCliTokenApproval({
  // ...
  waitConfirmation: true,
});

try {
  await approveToken();
} finally {
  setDisableTransactionBroadcastEnv(original);
}
return await result;
```

The pattern in plain English:
1. Capture the previous value while flipping to `"0"` (broadcast enabled). The return value of `setDisableTransactionBroadcastEnv` gives you the previous value in one call.
2. Kick off the CLI subprocess (which will sign and, because broadcast is enabled, submit the transaction).
3. In parallel, drive Speculos through the device-tap dance via `approveToken()`.
4. Once the device tap is done — or even if it threw — restore the previous value in `finally`.
5. Return the awaited CLI result.

The order matters. The CLI process is started *before* the try block so that `approveToken()` and the CLI subprocess race in parallel. If you started the CLI inside the try, you would serialise them and the test would deadlock — the CLI would block waiting for a device confirmation that no one is performing.

### 6.6.6 The Try/Finally Discipline

Every flip of `DISABLE_TRANSACTION_BROADCAST` must be paired with a restore in a `finally` block. Not after the try. Not in a side-effect callback. **In `finally`.**

The reason becomes obvious once you walk through the failure scenarios.

**Scenario 1 — `approveToken()` throws because the device tap fails.** Speculos misses a screen, `pressUntilTextFound` times out, an exception bubbles up. Without `finally`:

```ts
const original = setDisableTransactionBroadcastEnv("0");
const result = runCliTokenApproval({...});
await approveToken();              // <-- throws here
setDisableTransactionBroadcastEnv(original);  // <-- never runs
```

The throw escapes; the restore never runs. `DISABLE_TRANSACTION_BROADCAST` stays at `"0"` for the rest of the process lifetime. Every subsequent CLI call in the same Jest/Playwright worker silently broadcasts when it was supposed to be off. Allowance state leaks; the next test that asserts "expected zero allowance" sees the residue and fails for unrelated-looking reasons.

**Scenario 2 — `runCliTokenApproval` rejects because the CLI subprocess crashes.** Same problem in reverse: the await on the result inside the function rejects, the finally still gets the restore, but only because there is a `finally`.

**Scenario 3 — a test runner abort signal mid-flow.** If Jest or Playwright sends a SIGTERM, the JS runtime executes `finally` blocks before exiting. If you used a plain post-await line, the runtime exits without running it.

The pattern that handles all three correctly is the one in `revokeTokenCommand`:

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

Note the **second** `await result` outside the try/finally. The CLI subprocess promise is awaited *after* the env has been restored. If `runCliTokenApproval` rejects, the rejection propagates after the env is already restored, so the restore is guaranteed regardless of which side fails. If you put the `await result` inside the try, the same restore semantics still hold, but you risk re-entering the env-restore logic if you wrap the whole thing in another try elsewhere. The current shape is the conservative one and it is what the codebase consistently uses.

> **Tip:** when you write a new helper that flips broadcast, copy the existing pattern character-for-character. The cost of getting this wrong is a flaky suite that nobody knows is flaky because the symptoms surface in unrelated tests two hours later.

### 6.6.7 Speculos Interaction

Broadcast off does **not** mean signing off. Speculos still receives the unsigned transaction, walks through its normal "review transaction → sign" UI flow, and emits the signed bytes back over the speculos-http transport. The signed bytes travel up the device → transport → bridge stack and surface in the rxjs pipeline as a `signed` event. From there:

- **Broadcast disabled:** the `signed` event is the terminal state of the pipeline. The CLI prints it (in `default` or `json` format) and exits. The signed bytes are visible in CLI stdout but go nowhere. They are dropped on the floor. No `bridge.broadcast`, no node submission, no on-chain effect.
- **Broadcast enabled:** the `signed` event triggers the `concatMap` operator. `bridge.broadcast({ account, signedOperation: e.signedOperation })` ships the bytes to a node (the configured RPC endpoint for that currency). The node returns a transaction hash (or an error). The optimistic operation is updated with the hash; the CLI prints `✔️ broadcasted! optimistic operation: ...` and exits.

This is why a `--mode revokeApproval` call with broadcast disabled is **useful for nothing in production tests** — the revoke needs to land on chain or it has not happened. It is, however, useful for debugging. Run the same revoke command with `DISABLE_TRANSACTION_BROADCAST=1` and you can verify that signing works, the device-tap automation walks the right screens, the data field is the right calldata, and so on, all without spending a single cent of real ETH. Once that local validation passes, set the variable to `"0"` and let the broadcast happen on Ethereum mainnet.

The decoupling also means Speculos and the broadcast gate are orthogonal concerns. Speculos always emulates a hardware device; the broadcast gate decides what happens to the device's output. Tests that need to assert on device-screen content (for clear-signing audits, e.g.) run with broadcast off so they are cheap; tests that need to verify on-chain state changes run with broadcast on. The choice is per-test and per-helper, not global.

### 6.6.8 `--wait-confirmation` Interaction (a Gotcha)

`runCliTokenApproval` builds its CLI command with `--wait-confirmation` whenever its `waitConfirmation` option is true. The flag exists on `send.ts` and adds a final step to the rxjs pipeline: after the broadcast resolves, wait for the transaction to be confirmed on-chain (via `waitForTransactionConfirmation`) before exiting.

Critically, `waitForTransactionConfirmation` lives **inside the broadcast block** in the rxjs pipe. Looking back at the gate from §6.6.3:

```ts
...(opts["disable-broadcast"] || getEnv("DISABLE_TRANSACTION_BROADCAST")
  ? []
  : [
      concatMap((e) => {
        if (e.type === "signed") {
          return from(
            bridge.broadcast({ account, signedOperation: e.signedOperation })
              // ── waitForTransactionConfirmation lives in HERE ──
          );
        }
        return of(e);
      }),
    ]),
```

The `--wait-confirmation` step only runs when the broadcast block is part of the pipeline. If broadcast is disabled, the broadcast block is replaced with an empty operator list and `waitForTransactionConfirmation` never runs — there is no transaction hash to poll for, so there is nothing meaningful to wait on.

**This is a silent no-op**, not an error. The CLI does not warn that `--wait-confirmation` was supplied with broadcast disabled. It simply exits as soon as the signed event is printed. This is fine in normal flows — `revokeTokenCommand` and `approveTokenCommand` always set broadcast on before invoking, so the `--wait-confirmation` is honoured. The gotcha shows up only when someone copy-pastes a CLI command for manual debugging and forgets to clear `DISABLE_TRANSACTION_BROADCAST`.

> **Tip:** if you are debugging a CLI invocation by hand and `--wait-confirmation` seems to do nothing, check `echo $DISABLE_TRANSACTION_BROADCAST`. A leftover `1` from a previous shell session is the most common culprit.

### 6.6.9 `setEnv` (live-env) vs `process.env`

There are two parallel environment registries in play, and the `setDisableTransactionBroadcastEnv` helper writes to both. Understanding why is essential for reasoning about test isolation.

**`live-env`'s typed registry.** The package `@ledgerhq/live-env` exports `setEnv(key, value)` and `getEnv(key)` functions backed by an internal `EnvName → typed value` map. Every live-common module that reads an env var goes through `getEnv`, not `process.env`. The benefits are: type safety (each env name has a declared type — boolean, string, number — and `getEnv` returns the typed value), declared defaults (the schema declares the default for unset keys), and centralised validation (invalid values throw at registration time).

**`process.env`.** Plain Node.js OS-level environment. Subprocesses inherit it. Unrelated to live-env's registry.

The two are normally in sync because live-common's bootstrap reads `process.env` once and pushes the values into live-env. After bootstrap, however, modifying one does not modify the other. This is where the asymmetry matters:

- **Internal callers** (anything inside live-common, including `useBroadcast`, `setBroadcastTransaction`, the wallet-api gates) read via **`getEnv`**, so they see the live-env registry's value. To change what they see, you must call `setEnv`.
- **CLI subprocesses** are spawned by `runCliCommand` via `child_process.spawn` with `env: process.env`. The spawned `node apps/cli/bin/index.js ...` process inherits **`process.env`**, not the parent's live-env registry. Inside the CLI, `getEnv` will re-read `process.env` and rebuild its own live-env registry from those values. So to change what the CLI sees, you must mutate **`process.env`** before spawning.

Hence `setDisableTransactionBroadcastEnv` writes both:

```ts
export function setDisableTransactionBroadcastEnv(value: string | undefined) {
  const previous = process.env.DISABLE_TRANSACTION_BROADCAST;
  setEnv("DISABLE_TRANSACTION_BROADCAST", value === "1");  // typed boolean for live-env
  if (value === undefined) {
    delete process.env.DISABLE_TRANSACTION_BROADCAST;
  } else {
    process.env.DISABLE_TRANSACTION_BROADCAST = value;
  }
  return previous;
}
```

> **Verify:** the precise body of `setDisableTransactionBroadcastEnv` may evolve. The principle — write both sides defensively — is stable.

If you only set live-env, the parent process's internal callers see the new value but the spawned CLI subprocess inherits the unchanged `process.env` and ignores it. If you only set `process.env`, the CLI sees the new value but the parent process's live-common modules (the ones that decide whether to broadcast in `useBroadcast`, for example) read from live-env and see the stale value. Both must move together for the abstraction to hold.

### 6.6.10 Verifying Broadcast Happened

Once a test has flipped broadcast on, run the helper, and flipped it back, how do you actually confirm the transaction landed? The answer is layered.

**Layer 1 — CLI stdout.** When broadcast succeeds, the CLI emits a line like:

```
✔️ broadcasted! optimistic operation: { hash: "0xabc...", value: "0", ... }
```

This is your immediate signal. The optimistic operation is the local representation of the transaction; the `hash` field is the transaction hash that was returned by the node. If you see this line, the broadcast call returned without error.

**Layer 2 — `--wait-confirmation`.** When the helper passes `waitConfirmation: true` (the default for `revokeTokenCommand` and `approveTokenCommand`), the CLI also waits for the transaction to be confirmed before exiting. A successful confirmation typically appears in stdout as a confirmed-operation log; a timeout will throw. So if your helper returns at all, the transaction was both broadcast and confirmed within the timeout window.

**Layer 3 — the next test's allowance read.** The most important layer in practice. After a revoke, the next test that calls `getEvmTokenAllowance` (via `tokenAllowance` CLI or via `isTokenAllowanceSufficientCommand`) should see the allowance you just set. If revoke set it to zero, the next call returns zero; if approve set it to N, the next call returns N. This is the **after-the-fact verification** pattern and it is the only one that proves chain state. Layers 1 and 2 prove the broadcast call returned; only layer 3 proves the chain saw it.

**Layer 4 — out-of-band tools.** For one-off audits (Chapter 6.3), open `revoke.cash` or Etherscan's read-contract tab for Ethereum mainnet, plug in the seed's address, and read the allowance directly. This bypasses the test harness entirely and is useful when something feels wrong and you want to know what the chain actually says.

The four layers form a verification ladder. In automated tests, layers 1 and 3 are the everyday signals — layer 1 fails fast if the broadcast did not return, layer 3 confirms the state actually changed. Layer 2 catches confirmation timeouts. Layer 4 is the manual escape hatch.

### 6.6.11 Chapter 6.6 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Chapter 6.6 Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> What does <code>DISABLE_TRANSACTION_BROADCAST="1"</code> mean for a <code>send</code> CLI invocation?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The CLI skips signing entirely and exits with code 0</button>
<button class="quiz-choice" data-value="B">B) The CLI signs and broadcasts but does not wait for confirmation</button>
<button class="quiz-choice" data-value="C">C) The CLI signs (Speculos still walks the device flow), but the rxjs broadcast operator is not added to the pipeline, so the signed bytes are printed to stdout and never submitted to a node</button>
<button class="quiz-choice" data-value="D">D) Equivalent to passing <code>--ignore-errors</code> — broadcast is attempted, errors are swallowed</button>
</div>
<p class="quiz-explanation">The gate is at the rxjs operator-injection level. When the env var (or the CLI flag) is truthy, the spread expands to <code>[]</code> — no broadcast operator is part of the pipeline at all. Signing still happens; the signed bytes are emitted and the pipeline ends.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> You pass <code>--wait-confirmation</code> to <code>send</code> but also have <code>DISABLE_TRANSACTION_BROADCAST=1</code> in the environment. What happens?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The CLI errors out with "cannot wait for confirmation while broadcast is disabled"</button>
<button class="quiz-choice" data-value="B">B) The flag is a silent no-op — <code>waitForTransactionConfirmation</code> lives inside the broadcast block, which is replaced by <code>[]</code> when broadcast is disabled, so there is no transaction to wait on; the CLI exits as soon as the signed event is printed</button>
<button class="quiz-choice" data-value="C">C) The CLI waits for any pending transaction on the account, regardless of whether the current run broadcast one</button>
<button class="quiz-choice" data-value="D">D) The flag forces broadcast back on, overriding the env var</button>
</div>
<p class="quiz-explanation">The confirmation wait is part of the broadcast block in the rxjs pipeline. Disable broadcast and the entire block (including the confirmation wait) is omitted. No warning is printed — this is the gotcha.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> Why is the env-restore call placed in <code>finally</code> rather than after the <code>try</code> block in <code>revokeTokenCommand</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) If <code>approveToken()</code> throws (e.g., a Speculos screen-tap timeout), a post-try restore would never run, leaving <code>DISABLE_TRANSACTION_BROADCAST</code> stuck at <code>"0"</code> for the rest of the process and silently corrupting subsequent tests</button>
<button class="quiz-choice" data-value="B">B) <code>finally</code> is just a stylistic preference — the two forms are equivalent</button>
<button class="quiz-choice" data-value="C">C) The TypeScript compiler emits a warning if you place state-restoration outside <code>finally</code></button>
<button class="quiz-choice" data-value="D">D) <code>finally</code> blocks run before the awaited promise resolves, which is what the test needs</button>
</div>
<p class="quiz-explanation"><code>finally</code> guarantees execution even on throw. Without it, an exception escapes the try block and the restore line never runs — broadcast stays enabled for the rest of the worker, and subsequent CLI calls broadcast unintentionally. This produces flake that surfaces in tests unrelated to the original failure.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> Why does <code>setDisableTransactionBroadcastEnv</code> write to both <code>live-env</code>'s registry (via <code>setEnv</code>) and <code>process.env</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Defensive duplication for backwards compatibility with an old env-loading library</button>
<button class="quiz-choice" data-value="B">B) <code>process.env</code> is read-only inside ESM modules, so live-env is the canonical store</button>
<button class="quiz-choice" data-value="C">C) Only one of the two writes actually matters; the other is dead code from a refactor</button>
<button class="quiz-choice" data-value="D">D) Internal live-common consumers read via <code>getEnv</code> (live-env's typed registry), but spawned CLI subprocesses inherit <code>process.env</code> only — both must change so that both audiences see the new value</button>
</div>
<p class="quiz-explanation">The two registries are independent after bootstrap. Modules inside the parent process read live-env; the CLI subprocess re-bootstraps live-env from <code>process.env</code> on startup. Setting both keeps internal and subprocess consumers aligned.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> A QAA test runs a revoke helper, asserts no errors are thrown, and moves on. Two test runs later, an unrelated allowance test fails because the seed has a non-zero allowance. What is the most likely root cause?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The CLI subprocess crashed silently during signing</button>
<button class="quiz-choice" data-value="B">B) Speculos lost its Docker container between runs and replayed a stale signature</button>
<button class="quiz-choice" data-value="C">C) The revoke broadcast either failed (no <code>✔️ broadcasted!</code> line in stdout) or was a silent no-op because <code>DISABLE_TRANSACTION_BROADCAST</code> was not flipped to <code>"0"</code> for that run — the after-the-fact <code>getEvmTokenAllowance</code> read is the only true verification of state change</button>
<button class="quiz-choice" data-value="D">D) <code>--wait-confirmation</code> timed out and the test should retry</button>
</div>
<p class="quiz-explanation">The verification ladder from §6.6.10 — CLI stdout, wait-confirmation, follow-up allowance read, manual revoke.cash — exists exactly because "no thrown error" is not proof of on-chain state change. If the revoke ran with broadcast disabled (env var still <code>"1"</code>, or the helper's flip-to-<code>"0"</code> was bypassed), the signed bytes were dropped and the chain was unchanged.</p>
</div>

<div class="quiz-score"></div>
</div>

<div class="chapter-outro">
You now know what <code>DISABLE_TRANSACTION_BROADCAST</code> does, where it is read, where it is written, and why the try/finally restore is non-negotiable. The next chapter zooms back out from the single env var to the full daily workflow — pick up a ticket, orient yourself in the codebase, choose the right change shape, iterate locally, and ship a PR. The broadcast discipline you just learned is one of the moving parts; the workflow is the assembly line that keeps it moving.
</div>

---

## Daily CLI Workflow

<div class="chapter-intro">
Chapters 6.1 through 6.6 covered the architecture, the primitives, the env discipline, and the moving parts. This chapter is the assembly line: the exact, repeatable sequence you will follow every time a CLI-related QAA ticket lands in your queue. It mirrors the daily-workflow chapters in Parts 4 and 5 (desktop and mobile) but tracks the realities of CLI work — the legacy <code>apps/cli</code>, the five-layer architecture, the Speculos lifecycle, and the four change shapes that cover almost every ticket. Follow it every time. Even small tickets benefit from the recon pass; especially small tickets, because that is when you are most tempted to skip it and most likely to misjudge scope.
</div>

### 6.7.1 Workflow at a Glance

Every CLI ticket follows the same backbone:

```
pick a ticket
   │
   │ read AC, find linked B2CQA, identify scope
   ▼
orient yourself (5-min recon)
   │
   │ where does the change land in the 5-layer model?
   ▼
decide change shape
   │  (one of four — see 6.7.3)
   ▼
branch from develop
   │
   │ support/qaa-XXX-... or feat/qaa-XXX-...
   ▼
iterate locally
   │
   │ tight inner loop on Speculos, sub-minute per turn
   ▼
smoke against a real device (only if needed)
   │
   │ REMOTE_SPECULOS=true or USB-connected Ledger
   ▼
typecheck / lint / clean up commits
   │
   ▼
PR + reviewer routing
```

The only step that varies in size is "decide change shape". The rest is the same shape whether you are adding a one-line spec change or a brand-new typed helper. Treat the backbone as muscle memory and reserve mental bandwidth for the change-shape decision and the actual implementation.

### 6.7.2 Orientation — The 5-Minute Recon

Before writing a single line of code, run the recon. Five minutes spent here saves hours of going down the wrong path.

**Step 1 — Branch off `develop`.** Never `main`. Never the previous feature branch.

```bash
git checkout develop
git pull
git checkout -b support/qaa-XXX-add-revoke-helper
```

**Step 2 — Refresh dependencies.** `pnpm i` is typically a no-op when no `package.json` has changed, but it is cheap and it surfaces lockfile drifts before they bite you mid-iteration.

```bash
pnpm i
```

**Step 3 — Read the ticket end-to-end.** Twice. The first pass to get the gist; the second pass to spot the precise verb in the AC. "Add a helper" and "wire an existing helper into a fixture" sound similar but land in different layers. The verb tells you which layer.

**Step 4 — Identify the layer.** Use the table from Chapter 6.4 as a mental sieve:

| Verb in AC | Likely layer |
|---|---|
| "Add a CLI command for X" | Layer 1 (apps/cli) + Layer 2 + Layer 3 |
| "Add a typed helper for X" | Layer 3 (`cliCommandsUtils.ts`) |
| "Wire helper into the swap POM" | Layer 5 (`swap.page.ts`) |
| "Add a regression test for X" | Spec file only (`tests/specs/...`) |
| "Fix the device-tap dance for X" | Layer 4 (`families/evm.ts`) |

Most tickets land squarely in one layer. Tickets that span multiple layers (a brand-new CLI command, e.g.) are the ones to flag with your lead before starting.

**Step 5 — Open the relevant file.** Whatever layer you identified, open the file in your editor and skim it for context. For most QAA tickets, that means:

```bash
code libs/ledger-live-common/src/e2e/cliCommandsUtils.ts
```

Read the existing helpers. Note the naming conventions. Note the curried-function pattern (`liveDataCommand` returns a function that takes `userdataPath`). Note that `approveTokenCommand` and `revokeTokenCommand` set broadcast on, while `liveDataCommand` does not (because liveData has no on-chain side-effect).

**Step 6 — Find existing usage.** Search for the helpers you will likely interact with:

```bash
grep -rn "approveTokenCommand\|isTokenAllowanceSufficientCommand\|revokeTokenCommand" e2e/
```

Read three or four usage sites. The patterns repeat — most consumers are POM methods in `swap.page.ts` or fixture entries in `tests/specs/*.spec.ts`. After fifteen minutes of grep + read, you have a mental map of where your change fits.

> **Tip:** if a CLI helper does not yet exist for the operation you are automating, do not invent one in the spec or the POM. Add the helper to `cliCommandsUtils.ts` first, then call it from the higher layers. Specs that build their own CLI commands inline are a maintenance bomb.

### 6.7.3 Picking the Change Shape

Almost every CLI-related QAA ticket maps to one of four change shapes. Identifying yours up front is the single most important framing decision of the day.

#### Shape A — New CLI command needed

The ticket needs an operation that no `apps/cli` command supports today. Example: a hypothetical "register a domain on ENS" ticket would need a brand-new `apps/cli/src/commands/blockchain/registerEnsDomain.ts` because none of the 16 existing commands cover it.

**Layers touched:** 1 (CLI), 2 (engine helper in `runCli.ts`), 3 (typed wrapper in `cliCommandsUtils.ts`), and possibly 4 (if the new operation needs a new device-tap helper).

**Who to talk to first:** the wallet-ci team. They own `apps/cli`. New CLI commands need their review and there is often architectural prior art ("this overlaps with `signMessage` — extend that instead of adding a new command"). Do not start implementing without that conversation; you will rewrite half of it.

**Frequency:** rare. Maybe one or two tickets per quarter.

#### Shape B — New typed helper needed

The CLI command exists but no `cliCommandsUtils.ts` wrapper does. The senior's QAA-615 work — adding `revokeTokenCommand` — is exactly this shape. The underlying `send --mode revokeApproval` already worked; the helper layer needed a new typed entry point.

**Layers touched:** 3 only.

**Who to talk to first:** wallet-xp owns `cliCommandsUtils.ts` (it lives in `libs/ledger-live-common/src/e2e/`). A passing review here is straightforward as long as the new helper follows the existing naming and try/finally conventions.

**Frequency:** common. Most "automate this on-chain operation" tickets land here.

#### Shape C — New POM method needed

The CLI command and helper both exist but the POM method that orchestrates Speculos + helper for a particular UI flow does not. Example: a swap-cleanup hook that needs to revoke an allowance before each test. The senior's `revokeTokenApproval` POM method on `swap.page.ts` is exactly this shape.

**Layers touched:** 5 only (the relevant page object). Possibly a fixture file in `e2e/desktop/tests/fixtures/` if the new method should be invoked declaratively from `cliCommands: [...]`.

**Who to talk to first:** the team that owns the page (wallet-xp for `swap.page.ts`; PTX for swap-specific fixtures).

**Frequency:** common. Pairs naturally with shape B — sometimes the same PR adds the helper *and* the POM method.

#### Shape D — Just a new spec using existing POMs

The CLI command, helper, and POM method all exist. You are writing a regression spec under `tests/specs/` that consumes them. This is the lowest-cost change shape and the most common.

**Layers touched:** spec file only.

**Who to talk to first:** the test owners for the relevant area (wallet-xp for desktop e2e; mobile QA for mobile e2e).

**Frequency:** very common. The bread and butter of QAA work.

#### The decision tree

```
Does the operation exist as a CLI command?
   no  → Shape A (new CLI command, talk to wallet-ci)
   yes ↓
Does a cliCommandsUtils.ts helper wrap it?
   no  → Shape B (new helper)
   yes ↓
Does the relevant POM expose a method that calls the helper?
   no  → Shape C (new POM method)
   yes ↓
Are you adding new test coverage that consumes existing POM methods?
   yes → Shape D (spec only)
```

Walk the tree top-to-bottom. When you reach a "no", stop — that is your shape. Do not pre-emptively add layers below your stopping point unless the ticket explicitly needs them.

### 6.7.4 Local Iteration Loop

Once you have branched, identified the shape, and started writing, your inner loop is what determines how productive the day is. The target is **sub-minute per iteration** for spec changes and **under two minutes** for changes that touch live-common (which require a TypeScript rebuild of the library).

The canonical command to run a single test that uses the CLI:

```bash
cd e2e/desktop
pnpm run test:e2e --grep "B2CQA-XXXX"
```

`--grep` is Playwright's filter against the test title. Every test title carries the B2CQA tag, so this is the single-test filter you use 90% of the time. Variants:

- `--debug` — opens Playwright Inspector, lets you step through actions, see DOM at each step. Good for the first run on a new test.
- `--ui` — interactive UI mode. Tree view of all tests, click to run, watch artifacts in real-time. Good for exploratory debugging.
- `--reuse-context` — reuses the Electron context across tests if you have multiple in the grep. Saves ~30s per spec on Speculos boot but can mask state-leak bugs, so use only when the tests are independent.
- `--workers=1` — force serial. Useful when you suspect a Speculos port collision.

When the test runs, three things happen behind the scenes:

1. Playwright starts Electron (the desktop binary).
2. The fixture engine sees `cliCommands: [...]` in the test data and runs each curried function. If your helper is in there, this is when it fires.
3. The test body runs; POM methods that call CLI helpers fire as the test progresses.

For the first run on a fresh repo, expect ~30 seconds of Speculos Docker container boot. Subsequent runs reuse the container and start in 2–5 seconds.

> **Verify:** the exact root pnpm script for desktop e2e tests is `pnpm desktop e2e:test --grep ...` from the monorepo root, or the path-prefixed `cd e2e/desktop && pnpm run test:e2e --grep ...` from the workspace. Both invoke the same Playwright config. If neither resolves, check `e2e/desktop/package.json` for the current script name on your branch.

#### Watching artifacts in real-time

Each run writes to `e2e/desktop/artifacts/<run-id>/`. Tail it in a second terminal:

```bash
cd e2e/desktop
ls -lt artifacts/ | head -5
# pick the latest run-id
tail -f artifacts/<run-id>/test-name/output.log
```

For CLI invocations specifically, the spawn engine (`runCli.ts`) prints `[CLI] Executing: ledger-live <command>` to stderr; that line shows up in the run log and is your single-best signal that the CLI is being invoked with the args you expect.

#### When the loop is broken

If a single test takes more than two minutes per iteration, something is wrong. Common culprits:

- **Speculos won't start.** Docker daemon is down, or a previous Speculos container is wedged on the port. `docker ps` and `docker rm -f $(docker ps -aq --filter ancestor=ghcr.io/ledgerhq/speculos)` clears it.
- **Live-common changes not picked up.** If you edited `cliCommandsUtils.ts`, the desktop test imports the compiled output. `pnpm build:libs` recompiles.
- **You are accidentally running the full suite.** `--grep` requires a string; if you forget the value Playwright runs everything. Always quote the B2CQA tag.

### 6.7.5 Running a Single Test That Uses the CLI

The most common iteration command, in full:

```bash
cd e2e/desktop
pnpm run test:e2e --grep "B2CQA-1234"
```

What you should see, in order:

1. `[CLI] Executing: ledger-live liveData --currency ethereum ...` — the fixture seeding the account.
2. The Electron window opening (or running headless, depending on your config).
3. Optionally `[CLI] Executing: ledger-live tokenAllowance ...` if the test pre-checks an allowance.
4. The Speculos device window — either as a Docker container's HTTP UI on a local port, or virtualised entirely.
5. `[CLI] Executing: ledger-live send --mode revokeApproval ...` if the test invokes a revoke.
6. `✔️ broadcasted! optimistic operation: { hash: ... }` if broadcast was enabled.
7. The spec asserting on UI state and either passing or failing.

If you do not see step 1, the fixture wiring is broken. If you see step 1 but not step 2, Electron failed to start. If you see step 5 but not step 6, the broadcast did not return — likely a Speculos device-tap timeout or a node connectivity issue. Each missing step points to a different layer; learning to read the sequence is half the debugging skill.

#### Docker prerequisite

Speculos runs in Docker by default. If Docker is not running, the test fails almost immediately with a transport-registration error. Start Docker Desktop (or `dockerd` on Linux) before iterating. The first Speculos boot pulls the image; subsequent boots reuse it.

### 6.7.6 Smoke Against a Real Device

Most QAA work stays on Speculos. But before merging changes that affect signing flows for high-stakes coins (ETH, BTC, etc.), run a smoke against a real device once. The pyramid:

1. **Speculos local Docker** — the everyday default. Fast, deterministic, isolated.
2. **Speculos remote pool** — `REMOTE_SPECULOS=true` plus `SPECULOS_ADDRESS=<pool-host>:<port>`. Same protocol, different host. Useful when your laptop's Docker is misbehaving or when you want to mirror what CI does. Most of the time, identical-looking results to local Speculos.
3. **Real Ledger device wired via USB.** The final smoke. Open Ledger Live's developer-mode "use real device" path (or set the appropriate env to bypass Speculos), connect a Nano X / S+ / Stax / Flex with the relevant app installed, and run the same test. The device will physically buzz and ask for confirmation; you tap; the test should pass identically to Speculos.

Real-device smokes are slow (you are physically pressing buttons) and not scriptable — they are a final manual sanity check, not part of the inner loop. Do them once before PR, not once per commit.

> **Tip:** the most common surprise on real-device smoke is screen-text mismatch. Speculos uses the same `DeviceLabels` registry as a real device, but firmware-version drift can introduce wording differences. If your test passes on Speculos and fails on a real device with a `pressUntilTextFound` timeout, the device is on a different app version. Update the app on the device (Ledger Live's Manager) and retry.

### 6.7.7 Reading Existing Usage

The fastest way to learn the right shape for your change is to read three or four existing examples. Open these specs and scan the `cliCommands: [...]` lines:

**`e2e/desktop/tests/specs/earn.v2.spec.ts`** (around line 86):

```ts
test.describe.configure({ mode: "serial" });
test.use({
  cliCommands: [liveDataCommand(account)],
});
```

The fixture engine sees `liveDataCommand(account)` (the curried function). When the test starts, the engine calls it with the userdataPath argument the fixture supplies. The result is a seeded account ready in the test environment before the test body runs.

**`e2e/desktop/tests/specs/delegate.spec.ts`** (around line 110):

Same pattern, different account. The test data is different but the fixture mechanic is identical. Reading multiple specs that use the same helper makes the contract obvious — `liveDataCommand` always returns a curried function, the fixture engine always hands it `userdataPath`.

**`e2e/desktop/tests/specs/provider.swap.spec.ts`** (line 11):

```ts
test.use({
  cliCommands: [liveDataWithAddressCommand(account)],
});
```

Same shape, but the helper is `liveDataWithAddressCommand` — same as `liveDataCommand` plus it caches the device address on the `account` object. Useful when the test body needs the address (`account.address`) without making a separate `getAddress` call.

**`e2e/desktop/tests/page/swap.page.ts`**:

```ts
async revokeTokenApproval(fromAccount, provider) {
  // ...
  const result = await revokeTokenCommand(fromAccount, provider.contractAddress);
  // ...
}
```

This is the **other** way to consume CLI helpers — directly from a POM method, not through the fixture engine. Use this shape when the CLI call must happen in the middle of a test (e.g., between assertions on UI state), not just as setup. The senior's QAA-615 commit added exactly this pattern.

### 6.7.8 Writing Tests as You Go (TDD-Lite)

Once you know the shape, you can iterate effectively in a TDD-lite loop:

1. **Stub the POM method first.** Add the method signature and body (with a `console.log` inside) to the POM file.
2. **Write the spec calling it.** Or modify the existing spec to call it.
3. **Run with `.only` to isolate.** Playwright's `test.only` (or `test.describe.only`) limits the run to one test — combine with `--grep` for a stable single-target run.
4. **Watch it fail.** The console.log from step 1 should appear in stdout. If it does not, the wiring (import, instantiation, fixture) is broken. Fix that before doing any real work.
5. **Fill in the POM method.** Replace `console.log` with the actual `revokeTokenCommand(...)` call (or whatever your helper is).
6. **Watch it go green.** If the test now passes, you are done — refactor as needed and remove `.only`.

The advantage over write-everything-then-run is that you isolate the wiring problem (import, fixture, instantiation) from the logic problem (what the CLI actually does). Two small failures are much faster to debug than one big one.

> **Warning:** Always remove `test.only` before pushing. CI runs it as you wrote it, and a stray `.only` will reduce the entire run to one test, which can mask real failures elsewhere. Most repos lint against `.only` in CI; the lint catches it before it merges, but the failed CI run still wastes time.

### 6.7.9 Codegen, Lint, Typecheck

Unlike wallet-cli (which has a `bunli generate` step), the legacy `apps/cli` has **no codegen step** in the QAA workflow. There is a build step (`pnpm --filter live-cli build`, ultimately `zx ./scripts/build.mjs`) that you run only if you actually edited an `apps/cli` source file. For changes confined to `cliCommandsUtils.ts`, page objects, fixtures, or specs, no rebuild is needed — TypeScript compiles on demand via the test runner's transformer.

Before pushing, run the standard quality checks:

```bash
# Workspace-local typecheck
pnpm --filter ledger-live-common typecheck
pnpm --filter ledger-live-desktop-e2e typecheck

# Lint your changed files
pnpm lint
```

The exact filter names vary by branch; check `package.json` at the workspace root for the canonical scripts. The typecheck is non-negotiable — it catches the kind of error (a renamed type, a missing export) that would otherwise blow up CI five minutes after you push.

> **Verify:** if `pnpm typecheck` does not exist as a root-level script on your branch, run the workspace-specific equivalents instead. The point is to typecheck both `live-common` (where helpers live) and `desktop-e2e` (where they are consumed) before opening a PR.

### 6.7.10 Commit and PR

Commits follow the global Conventional Commits convention (see `git-workflow.md` in the global rules). For CLI work, the typical scopes are `desktop`, `mobile`, `coin`, or `common`.

The senior's commit on QAA-615 was in this style:

```
test(desktop): add token approval command in e2e
```

The verb is imperative, lowercase. The scope identifies the area (`desktop` since the change ultimately serves the desktop e2e suite). The description names the new capability concretely.

**Branch naming:**

| Prefix | When to use |
|---|---|
| `support/qaa-XXX-...` | Test helpers, refactors, fixture additions, infra tweaks |
| `feat/qaa-XXX-...` | New test capability that did not exist before (e.g., the first-ever revoke helper) |
| `bugfix/qaa-XXX-...` | Fixing a flake, fixing a broken existing helper |
| `chore/qaa-XXX-...` | Tooling, configs, dependency bumps |

Names use kebab-case. Keep them short and action-oriented. For QAA-615, `support/qaa-615-add-revoke-token` (the senior used underscores; the global convention prefers kebab — pick what your team's recent branches use).

**Pushing and opening a PR:**

```bash
git push -u origin support/qaa-XXX-add-revoke-helper
gh pr create --base develop --title "test(desktop): add revoke token command in e2e" \
  --body "$(cat <<'EOF'
## Summary
- Adds `revokeTokenCommand` to `cliCommandsUtils.ts`.
- Adds `revokeTokenApproval` POM method to `swap.page.ts`.
- Wires QAA-615 acceptance test.

## Test plan
- [x] `pnpm e2e:test` passes locally on Speculos (run specific test by file/tag via Detox)
- [x] Typecheck passes for `ledger-live-common` and `desktop-e2e`
- [x] Lint passes
EOF
)"
```

Cross-reference Part 0 Chapter 0.4 for the broader release flow — how `develop` rolls into `main` via the weekly release branches, where Xray sees your test results, and how the changelog is auto-generated.

### 6.7.11 Reviewer Routing

Open `.github/CODEOWNERS` near each file you touched and note the owners. The recurring routes for CLI work:

| Path | Owner |
|---|---|
| `apps/cli/**` | `@ledgerhq/wallet-ci` (and the relevant coin-module team for coin-specific changes) |
| `libs/ledger-live-common/src/e2e/**` | `@ledgerhq/wallet-xp` |
| `libs/ledger-live-common/src/families/<coin>/**` | the coin-module team for that coin (e.g., `@ledgerhq/coin-evm` for EVM) |
| `e2e/desktop/**` | `@ledgerhq/wallet-xp` |
| `e2e/desktop/tests/specs/*.swap.spec.ts`, `swap.page.ts` | `@ledgerhq/wallet-xp` + `@ledgerhq/ptx` (PTX owns swap product) |
| `e2e/mobile/**` | `@ledgerhq/wallet-xp` (mobile QA shares this owner) |

For a change that adds a swap-related helper plus a swap POM method plus a swap regression spec, you will end up with two reviewers (`wallet-xp` for the e2e workspace, `ptx` for the swap-specific paths). Tag both in the PR description if they are not auto-added by CODEOWNERS.

> **Tip:** if your change touches `apps/cli` even in passing (e.g., you added a flag to an existing command), wallet-ci review is non-optional. Ping them in the PR thread early — they review CLI changes carefully because the CLI is shared infra used by the bot, by QAA, and by manual operators.

### 6.7.12 Common Debugging Recipes

Even with the best discipline, things break. Here are the recurring failure modes and what they look like.

#### Speculos won't start

```
Error: Failed to start Speculos: port 5000 already in use
```

A previous Speculos container did not clean up. Either:

```bash
docker ps                                # find the orphan
docker rm -f <container-id>              # kill it
# or, blanket clean:
docker rm -f $(docker ps -aq --filter ancestor=ghcr.io/ledgerhq/speculos)
```

If Docker itself is not running:

```bash
# macOS
open -a "Docker"
# Linux
systemctl start docker
```

Re-run after Docker is healthy.

#### CLI subprocess hangs

The CLI command is printed but never returns:

```
[CLI] Executing: ledger-live send --currency ethereum --mode approve ...
(no output for 2 minutes, then test times out)
```

Usually a missed Speculos screen tap. The CLI is blocked waiting for `bridge.signOperation` to emit `signed`, which depends on the device confirming the transaction, which depends on `approveToken()` walking the right screens. If `approveToken()` is racing the CLI and failing silently, increase the test's `waitConfirmation` timeout temporarily and retry — if it now passes, the device-tap dance was just slow, not broken.

If a longer timeout does not help, dump the Speculos screen during the hang. Open the Speculos web UI (the port is logged on container start) and see what screen is currently displayed. If it is stuck on a screen `pressUntilTextFound` doesn't recognise (e.g., a new "Scam token warning" page), the device-tap helper in `families/evm.ts` needs a new label.

#### Allowance check returns the wrong value

`isTokenAllowanceSufficientCommand` returns 0 even though you just approved. Three possibilities:

1. **Wrong spender.** The approve was for `0xRouterA`; the read is against `0xRouterB`. Double-check the `provider.contractAddress` you passed both ways.
2. **Wrong owner.** The approve was on account index 0; the read is on index 1. The two indices have different addresses. Check the `account` object passed to both helpers — make sure it is the same one.
3. **The approve didn't actually broadcast.** Re-check `DISABLE_TRANSACTION_BROADCAST` — if it was `"1"` during the approve, the transaction was signed but never sent. Verification ladder from §6.6.10 applies — look for `✔️ broadcasted!` in CLI stdout.

#### Broadcast didn't happen

You ran a revoke; no error was thrown; the next test still sees a non-zero allowance.

Most likely the env var was not flipped. Either the helper's `setDisableTransactionBroadcastEnv("0")` was bypassed (a copy-paste regression that omits the line) or a parent test left `DISABLE_TRANSACTION_BROADCAST="1"` in `process.env` and the helper's flip-and-restore is restoring it back to `"1"` after the call.

Confirm by running with verbose CLI output and looking for the broadcast line. If it is missing, the gate from §6.6.3 evaluated truthy and the broadcast operator was never injected.

If the helper looks correct, check whether some module-load hook in the spec set the var globally. Mobile swap's `swap.other.ts` does this (§6.6.5); a desktop spec that imports a mobile helper transitively might inherit the same hook. Search: `grep -rn 'DISABLE_TRANSACTION_BROADCAST' e2e/ libs/`.

#### Test passes locally but fails in CI

The classic e2e symptom. Order of suspects, in increasing rarity:

1. **Allowance state from a previous CI run is stale.** Check on-chain via Etherscan; if there is residual allowance, the suite needs a revoke `beforeEach`.
2. **Different Speculos image.** CI pins a specific Speculos commit; locally you have whatever Docker pulled most recently. If your test depends on a behaviour that changed between versions, this manifests.
3. **Network flake to the Ethereum mainnet RPC.** Less common — most QAA tests use stable RPC endpoints — but a transient RPC outage can drop a broadcast. Re-run the CI job once before assuming the test is broken.
4. **Race between fixture seeding and the test body.** If the fixture's CLI call is async and the test body assumes seeding is done, the assumption can hold locally (where fixtures finish faster than the test starts) and break on a slower CI runner. Check the fixture engine's `await` discipline.

### 6.7.13 Chapter 6.7 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Chapter 6.7 Quiz</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> A QAA ticket asks you to "wire <code>revokeTokenCommand</code> into the swap before-each cleanup hook". Which change shape is this?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Shape A — new CLI command needed (talk to wallet-ci first)</button>
<button class="quiz-choice" data-value="B">B) Shape B — new typed helper needed in <code>cliCommandsUtils.ts</code></button>
<button class="quiz-choice" data-value="C">C) Shape C — new POM method on <code>swap.page.ts</code> (or fixture-engine wiring) — the helper already exists, you are just orchestrating it</button>
<button class="quiz-choice" data-value="D">D) Shape D — spec only</button>
</div>
<p class="quiz-explanation"><code>revokeTokenCommand</code> is the helper. Wiring it into a before-each is a POM/fixture concern, not a helper or CLI concern. Shape C — touch <code>swap.page.ts</code> (or the relevant fixture file) and invoke the existing helper. The CLI command is unchanged; the helper is unchanged; only the orchestration moves.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> You need a brand-new typed helper to wrap an existing <code>tokenAllowance</code> CLI invocation with a slightly different shape (returns a structured object rather than a raw string). Where do you start?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>apps/cli/src/commands/blockchain/tokenAllowance.ts</code> — change the CLI output format</button>
<button class="quiz-choice" data-value="B">B) <code>libs/ledger-live-common/src/e2e/cliCommandsUtils.ts</code> — add a new helper that calls <code>runCliGetTokenAllowance</code> and post-processes the output</button>
<button class="quiz-choice" data-value="C">C) <code>e2e/desktop/tests/page/swap.page.ts</code> — bake the parsing into the POM method</button>
<button class="quiz-choice" data-value="D">D) Inline a parser in the spec file that needs the structured object</button>
</div>
<p class="quiz-explanation">Layer 3 is the right home for a new typed helper that wraps an existing CLI command. The CLI is unchanged (so wallet-ci review is not needed); the POM should not contain parsing logic (the POM is for UI orchestration); inlining in a spec is a maintenance bomb. <code>cliCommandsUtils.ts</code> with a thin parsing function is the canonical pattern.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> What does <code>cliCommands: [liveDataWithAddressCommand(account)]</code> in a Playwright test config achieve, mechanically?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The fixture engine receives a curried function; before the test body runs, the engine calls it with the lifecycle's <code>userdataPath</code>; the call seeds the account via the CLI and caches the device address on <code>account.address</code></button>
<button class="quiz-choice" data-value="B">B) It registers a Playwright project named <code>liveDataWithAddressCommand</code> and runs only that project</button>
<button class="quiz-choice" data-value="C">C) It synchronously calls the CLI at module load time, before Playwright spins up</button>
<button class="quiz-choice" data-value="D">D) It marks the test as requiring a real device, skipping it on Speculos</button>
</div>
<p class="quiz-explanation">The curried-function pattern is the contract between fixture-declarable CLI helpers and the fixture engine. The engine sees a function, calls it with the userdataPath, awaits the result. The test body then runs against a world where the CLI seeding has already happened.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> Your local iteration loop is running a single test in 30+ seconds even though only one TypeScript file changed. What is the most likely cause?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Playwright re-installs node_modules on every run</button>
<button class="quiz-choice" data-value="B">B) Speculos reboots between test runs by design — there is no way to make it faster</button>
<button class="quiz-choice" data-value="C">C) The CLI subprocess re-bundles itself on every invocation</button>
<button class="quiz-choice" data-value="D">D) Speculos is booting fresh because no <code>--reuse-context</code> is set, or the previous Speculos container was killed; subsequent runs with reused state are 2–5 seconds, fresh boots are ~30 seconds</button>
</div>
<p class="quiz-explanation">The first Speculos container boot dominates the loop time on a cold cache. Reuse the container or the test context across runs to stay in the sub-minute zone. If you are restarting Docker every iteration, you are paying the full boot cost every time.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> When does a CLI-related QAA ticket need wallet-ci as a reviewer?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Always — wallet-ci reviews every PR that touches e2e tests</button>
<button class="quiz-choice" data-value="B">B) Never — wallet-ci is a different team that does not touch QAA work</button>
<button class="quiz-choice" data-value="C">C) When the PR touches anything under <code>apps/cli/</code> (a new command, a new flag, a behavioural tweak to an existing command). Pure changes to <code>cliCommandsUtils.ts</code>, page objects, or specs are routed to wallet-xp instead</button>
<button class="quiz-choice" data-value="D">D) Only when the change affects the bot</button>
</div>
<p class="quiz-explanation">CODEOWNERS routes <code>apps/cli/**</code> to wallet-ci; <code>libs/ledger-live-common/src/e2e/**</code> and <code>e2e/desktop/**</code> route to wallet-xp. A PR that changes the CLI itself needs wallet-ci review; a PR that only adds a typed helper and a spec does not.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q6.</strong> You signed off locally on a test that broadcasts a revoke on Ethereum mainnet and want to confirm the transaction actually landed before opening the PR. Which is the most reliable verification?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The CLI exited with code 0 — that is sufficient</button>
<button class="quiz-choice" data-value="B">B) Run a follow-up <code>tokenAllowance</code> call (or open <code>revoke.cash</code> on the seed's address) and read the actual on-chain allowance — if it is now zero, the revoke landed</button>
<button class="quiz-choice" data-value="C">C) Re-run the same revoke; if it is idempotent it must have worked the first time</button>
<button class="quiz-choice" data-value="D">D) Check the local Speculos logs for a "transaction signed" line</button>
</div>
<p class="quiz-explanation">Layer 3 of the verification ladder (Chapter 6.6.10) — the after-the-fact allowance read — is the only one that proves chain state changed. Exit code 0 means the broadcast call returned, but a failed broadcast can also exit 0 in some configurations. Speculos logs prove signing, not broadcast. Re-running tells you nothing about the first run.</p>
</div>

<div class="quiz-score"></div>
</div>

<div class="chapter-outro">
You now have the full daily workflow — recon, change shape, iteration loop, smoke, commit, PR — and the debugging vocabulary to read the failure modes that show up in practice. The next chapter takes a real ticket end-to-end: <strong>QAA-615</strong>, the senior's commit that introduced <code>revokeTokenCommand</code>, walked line-by-line through every file changed, every choice made, every alternative considered. After you have followed that walkthrough, the architecture, the env discipline, and the workflow will all click into a single coherent picture you can reproduce on your own tickets.
</div>

## Walkthrough QAA-615 — Line-by-Line

<div class="chapter-intro">
This is your Part 6 capstone. Unlike the QAA-702 (Part 5) and QAA-1136 (Part 7) walkthroughs — where you start from a green baseline and add a missing scenario — QAA-615 is a <strong>spike that already has a starting commit</strong>. A senior engineer (Abdurrahman SASTIM) pushed branch <code>support/qaa-615-add-revoke-token</code> with commit <code>4fa4868a35</code>: 3 files, 37 lines. The wiring works; the shape is incomplete; one of the lines contains a copy-paste bug. Your job is to read every one of those lines, understand <em>why</em> the senior wrote them, then identify what is still missing before this can be merged. By the end of this chapter you will be able to take the senior's branch, finish the work, and ship the spike report.
</div>

### 6.8.1 Understanding the Ticket

**Jira ticket:** QAA-615 — *[CLI] Add revoke-approval helper for the swap regression suite*
**Parent epic:** QAA-919 — *Swap regression coverage automation*
**Type:** Spike
**Status:** In Progress (the senior's branch is the spike's draft deliverable)
**Branch:** `support/qaa-615-add-revoke-token`
**Commit:** `4fa4868a35` by Abdurrahman SASTIM, dated 2026-04-27

The ticket asks one specific question:

> Find the smallest, lowest-risk way to revoke an existing ERC-20 allowance from inside an E2E test, so that the broadcast-enabled regression suite can clean up provider allowances between iterations.

**What "spike" means in this team's vocabulary.** A spike is a time-boxed investigation whose deliverable is a written answer plus a code proof-of-concept. The PR that closes it does not need to be production-perfect — it needs to demonstrate that the chosen path works and document why the alternatives were rejected. The QAA-615 senior's commit is exactly that: a working PoC, not a final shape.

**Pavlo's "2 flows" comment.** Pavlo OKHONKO commented on QAA-615 on 2026-03-12:

> "Two flows to keep separate: nightly (broadcast off, no revoke needed) vs regression (broadcast on, revoke required between tests). Don't try to make one helper serve both."

That comment is the architectural seed for the work. It tells the senior: the new helper is not a general-purpose utility; it exists to serve the broadcast-enabled regression flow specifically. The nightly suite already runs deterministically with `DISABLE_TRANSACTION_BROADCAST=1` and does not need cleanup, because the device signs but never sends — on-chain state never changes.

**Why the parent epic matters.** QAA-919 *Swap regression coverage automation* is the umbrella for moving the swap suite from synthetic-fixture nightly runs to broadcast-enabled regression runs. Each child ticket addresses one infrastructure prerequisite: QAA-613 the test hooks, QAA-614 the seed-funding workflow, QAA-615 the revoke helper, and so on. The pieces are designed to land separately.

### 6.8.2 Why This Matters

The broadcast distinction (Chapter 6.6) is the core context. To recap:

- **Nightly suite** — `DISABLE_TRANSACTION_BROADCAST=1`. The CLI signs locally but never sends. On-chain allowance never changes. No revoke needed; no real money at risk; ~10 minutes of run time.
- **Regression suite** — `DISABLE_TRANSACTION_BROADCAST=0`. The CLI signs and broadcasts. Allowances are real; the QAA test seed actually changes state on Ethereum mainnet. Without a revoke hook, run #2 sees whatever allowance run #1 left behind.

**Without QAA-615**, the broadcast-enabled regression suite has a state-leak problem:

1. Test A approves spender X for 100 USDT to perform a swap.
2. Test A passes; the test runner moves on without cleanup.
3. Test B starts. Its `ensureTokenApproval` pre-check sees `currentAllowance >= minAmount` and skips the approve flow entirely.
4. Test B passes for the wrong reason — it never exercised the device confirmation it was supposed to.

In a worst case, the leak hides a real regression: the approve screen rendering changes, but Test B never asks for it because the leftover allowance is enough. Coverage looks green; the device-tap path silently rotted.

**Revoke restores determinism.** A `revoke` hook in `beforeEach` zeroes the allowance before every test. Test B's pre-check now correctly sees `currentAllowance == 0`, runs the approve flow, and exercises the screens it is supposed to.

Cross-references:
- Chapter 6.6 — broadcast discipline and the env-flip pattern
- Chapter 6.5 — Speculos lifecycle and the snapshot/restore pattern
- Chapter 6.4 — the 5-layer architecture that makes the wrapper trivial

### 6.8.3 The Senior's Branch

```bash
cd /path/to/ledger-live
git fetch origin support/qaa-615-add-revoke-token
git checkout -b support/qaa-615-add-revoke-token origin/support/qaa-615-add-revoke-token
git show 4fa4868a35
```

Three files, 37 lines added, 1 line edited (commented out).

| File | Lines | Purpose |
|---|---|---|
| `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts` | +20 | New `revokeTokenCommand` typed wrapper (Layer 3) |
| `e2e/desktop/tests/page/swap.page.ts` | +15 | New `revokeTokenApproval` POM method (Layer 5) |
| `e2e/desktop/tests/specs/provider.swap.spec.ts` | +1 / -1 | Dev-loop swap: comment out `ensureTokenApproval`, call `revokeTokenApproval` instead |

The commit message is one line:

```
test: add token approval command in e2E
```

That message is *deliberately understated* — the senior is signalling "this is the spike PoC; the polished commit message will land with the final PR". Don't echo the same message when you ship; rewrite it for the final PR.

**One thing the commit does not contain:** mobile parity, the spike report, the corrected `@step` description, the proper hook ordering. Those four items are your work.

### 6.8.4 File 1 — `cliCommandsUtils.ts` Line by Line

The new function lives next to its existing twin `approveTokenCommand`. Read it as a unit:

```ts
// libs/ledger-live-common/src/e2e/cliCommandsUtils.ts
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

Walk it line by line.

#### Line 1 — the signature

```ts
export const revokeTokenCommand = async (account: TokenAccount, spender: string) => {
```

- `TokenAccount` (not `Account`) — by type, only token sub-accounts can be revoked. A native Account (ETH, BTC) has no allowance to clear; the type system prevents calling this with a wrong shape.
- `spender: string` — the contract address that holds the allowance. In swap tests that is `provider.contractAddress` (e.g., 1inch's router on Ethereum, OKX's router, etc.).
- No `approveAmount` argument. Compare to its twin:

  ```ts
  export const approveTokenCommand = async (
    account: TokenAccount, spender: string, approveAmount: string,
  ) => { ... };
  ```

  The revoke is **always to zero**. The senior didn't add an `approveAmount: "0"` parameter because that would be a footgun — callers could pass `"100"` and turn revoke into a partial approve.

#### Line 2 — flip the broadcast env

```ts
const original = setDisableTransactionBroadcastEnv("0");
```

- `"0"` means `DISABLE_TRANSACTION_BROADCAST` is now falsy → the CLI **will** broadcast.
- `original` captures whatever the env was before the flip. Could be `"1"`, `"0"`, or `undefined`. We must restore it later, regardless.
- Why "0" and not unset? Because the function explicitly opts into broadcast. The whole reason to revoke is to change on-chain state; signing without sending would defeat the purpose.

This is the same flip `approveTokenCommand` does. The pattern is documented in Chapter 6.6.7.

#### Lines 4-11 — build and start the CLI subprocess

```ts
const result = runCliTokenApproval({
  currency: account.currency.speculosApp.name,
  index: account.index,
  spender,
  token: account.currency.id,
  mode: "revokeApproval",
  waitConfirmation: true,
});
```

- `runCliTokenApproval` is the Layer-2 spawn wrapper (Chapter 6.4). It returns a `Promise<string>` that resolves with the CLI's stdout when the subprocess exits cleanly.
- `currency: account.currency.speculosApp.name` — the *speculos app* name, not the parent chain ID. For an ERC-20 USDT account, `account.currency.speculosApp.name` is `"Ethereum"` (the app that signs ERC-20s). This is what gets passed to `--currency` and is also what `launchSpeculos` uses (you'll see it again in File 2).
- `index: account.index` — the account index in the seed's derivation tree. The senior is reading this off the `TokenAccount` directly.
- `spender` — passed straight through to `--spender`. The CLI bakes it into the calldata.
- `token: account.currency.id` — the live-common currency ID, e.g., `"ethereum/erc20/usd_coin"`. The CLI resolves this to a contract address internally.
- `mode: "revokeApproval"` — the magic flag. This is parsed by `inferTransactionsOpts` in `apps/cli/src/transaction.ts` and dispatched to `libs/coin-modules/coin-evm/src/cli-transaction.ts:inferTransactions`, where the EVM mode is converted into the transaction shape (`recipient = token contract`, `amount = 0`, `data = getErc20ApproveData(spender, 0n)`). See Chapter 6.4 for the trace.
- `waitConfirmation: true` — adds `--wait-confirmation` to argv. The CLI will keep its subprocess alive until the device signs *and* broadcasts *and* the tx is confirmed in a block. Without this flag, the CLI exits immediately after broadcast and your test moves on before the chain has settled — leading to flaky pre-checks on the next iteration.

**Note the assignment**, not `await`. The function fires the CLI subprocess and stores the still-pending promise in `result`. Awaiting it now would block the device-tap helper from running in parallel — and the device tap is what *unblocks* the CLI.

#### Lines 13-17 — the device-tap dance

```ts
try {
  await approveToken();
} finally {
  setDisableTransactionBroadcastEnv(original);
}
```

- `approveToken()` is the Layer-4 helper from `libs/ledger-live-common/src/e2e/families/evm.ts` (Chapter 6.4 §6). It drives the Speculos REST API to walk through the approval screens — `waitFor(REVIEW_TRANSACTION_TO)`, `pressUntilTextFound(SIGN_TRANSACTION)`, then both buttons (or hold-to-sign on touch devices).
- The function name is a misnomer for this context — `approveToken` *means "tap through the approve-or-revoke confirmation screens"*. Both transactions render the same `ERC20.APPROVE` selector to the device; the only difference is the on-screen amount (zero for revoke, N for approve). The device flow is identical, so the helper is reused.
- `try/finally` around it is the safety net. If `approveToken()` throws (label timeout, button-press error), we still hit the `finally` block and restore the env. Without that, the env would be left at `"0"` for the rest of the test run, silently flipping broadcast on for every subsequent CLI call. Chapter 6.6.7 covers this in detail.

The CLI subprocess and `approveToken()` run in parallel. While the CLI is blocked at `bridge.signOperation` waiting for the device, `approveToken()` *is* the user — driving the simulator to sign.

#### Line 18 — return the CLI stdout

```ts
return await result;
```

- This is the await we deferred earlier. By the time we hit it, the device has signed (because `approveToken()` resolved), the CLI has broadcast (because the env was `"0"`), and the chain has confirmed (because `waitConfirmation: true`).
- The returned string is the CLI's stdout. With default formatting, it's a human-readable log; with `--format json`, it's a structured envelope. The POM method (File 2) attaches this string to the Allure report so reviewers can see what landed.

**What this function does NOT do:**
- It does not check if there's anything to revoke. Calling revoke on a zero allowance is a no-op gas-wise but still costs a tx fee and a confirmation. If you wanted to skip the device dance when allowance is already zero, you'd add an `isTokenAllowanceSufficientCommand(account, spender, 0)`-style pre-check — the senior chose not to, probably because in regression context the revoke runs in `beforeEach` regardless and the no-op check would add complexity for marginal savings.

### 6.8.5 File 2 — `swap.page.ts` Line by Line

```ts
// e2e/desktop/tests/page/swap.page.ts
@step("Ensure token approval")
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

#### Line 1 — `@step("Ensure token approval")` — **THE COPY-PASTE BUG**

The senior copied this from `ensureTokenApproval` and forgot to update the description. In Allure reports, every test step rendered from this method will be labelled *"Ensure token approval"* — wrong on its face, and especially confusing when both methods run in the same test.

**Fix before merge:**

```ts
@step("Revoke token approval")
async revokeTokenApproval(...) { ... }
```

This is the smallest, most obvious item on your do-list. Don't ship without it.

#### Line 2 — the signature

```ts
async revokeTokenApproval(fromAccount: Account | TokenAccount, provider: Provider) {
```

- Note `Account | TokenAccount`. The POM accepts a wider type than the Layer-3 helper (`TokenAccount` only). Why? Because callers in the swap spec already type their `fromAccount` as `Account | TokenAccount` (a swap can be from a native or a token account), and the POM wants to be a drop-in replacement at those call sites. The early-return guard on the next line catches the native-account case.
- `provider: Provider` — the swap provider enum (1inch, OKX, etc.). Carries `contractAddress`, `uiName`, `app`, etc.

#### Line 3 — the early-return guard

```ts
if (!provider.contractAddress || !fromAccount.parentAccount) return;
```

- `!provider.contractAddress` — some providers don't need an allowance (CEX flows, native-to-native). For those, revoke is meaningless; skip.
- `!fromAccount.parentAccount` — only a `TokenAccount` carries `parentAccount` (its EVM parent). A native `Account` returns `undefined` here. The guard catches the `Account | TokenAccount` widening from line 2.

**A subtle gap.** If a caller passed a native `Account` *with* a `parentAccount` (impossible by current types, but the type system doesn't enforce that the discriminant matches the flag), the guard would let it through and the helper would crash inside `revokeTokenCommand` because `account.currency.speculosApp.name` would not match the `TokenAccount` shape the CLI wrapper expects. Tightening this is on your do-list (6.8.7 #4).

#### Line 5 — snapshot the current Speculos port

```ts
const previousSpeculosPort = getEnv("SPECULOS_API_PORT");
```

- The fixture has already started a Speculos for the *swap* itself, on the Exchange app. That instance's port lives in `SPECULOS_API_PORT`.
- We're about to replace it with a fresh Speculos running the *Ethereum* app (because revoke is an EVM tx, not a swap-sig request). When we're done, we must restore the original port, so the rest of the test continues to talk to the Exchange Speculos.
- This is the snapshot/restore pattern from Chapter 6.5.

#### Line 6 — launch a fresh Speculos for the source coin's app

```ts
const speculos = await launchSpeculos(fromAccount.currency.speculosApp.name);
```

- For a USDT→ETH swap with `fromAccount = TokenAccount.ETH_USDT_1`, `fromAccount.currency.speculosApp.name` is `"Ethereum"`. We launch a Speculos running the Ethereum app — the one that knows how to render ERC20.APPROVE clear-sign screens.
- `launchSpeculos` registers the Speculos transport, sets `SPECULOS_API_PORT` to the new port, and returns a `SpeculosDevice`. See Chapter 6.5.

#### Lines 7-9 — call the Layer-3 helper, attach to Allure

```ts
try {
  const result = await revokeTokenCommand(fromAccount, provider.contractAddress);
  await allure.description(`Token revoke result for ${provider.uiName}:\n\n ${result}`);
```

- `revokeTokenCommand` does the broadcast-flip + spawn + device-tap + restore (you just walked it in 6.8.4).
- `allure.description(...)` attaches the CLI stdout to the test's Allure entry. Reviewers see the actual transaction hash, gas used, and confirmation status without having to dig through CI logs.
- The string includes `${provider.uiName}` so reviewers can tell *which* provider's revoke this was — useful when the spec iterates over multiple providers in one run.

#### Line 11 — restore the previous Speculos

```ts
} finally {
  await cleanSpeculos(speculos, previousSpeculosPort);
}
```

- `cleanSpeculos` stops the Speculos process, unregisters its transport, and restores `SPECULOS_API_PORT` to `previousSpeculosPort`.
- The `finally` ensures it runs even if the revoke threw — without it, the next test would inherit a stale port and a leaked Speculos container.
- After this line, the world is back to the swap's Exchange Speculos, exactly as it was before `revokeTokenApproval` was called.

### 6.8.6 File 3 — `provider.swap.spec.ts` Line by Line

The diff is two lines:

```diff
  const minAmount = await app.swap.getMinimumAmount(fromAccount, toAccount);
- await app.swap.ensureTokenApproval(fromAccount, provider, minAmount);
+ //await app.swap.ensureTokenApproval(fromAccount, provider, minAmount);
+ await app.swap.revokeTokenApproval(fromAccount, provider);
  const swap = new Swap(fromAccount, toAccount, minAmount, provider);
```

#### What was removed

`await app.swap.ensureTokenApproval(fromAccount, provider, minAmount)` ran the approve flow with a 1.2x slippage buffer (see Chapter 6.6's twin walkthrough). It set the allowance high enough that the swap that followed could pull tokens.

#### What was added

`//await app.swap.ensureTokenApproval(...)` — same line, commented out. The senior left the original code in as a marker; this is a common signal of "I'll un-comment this once the new line is proven".

`await app.swap.revokeTokenApproval(fromAccount, provider)` — the new method from File 2. It clears whatever allowance is on-chain and returns.

#### What this means

**This is dev-loop code, not the final shape.** The senior wanted to test the revoke path *in isolation*. With both calls in place, a failure would be ambiguous (revoke or approve?). With only the revoke, a failure is unambiguously a revoke problem.

The senior was iterating. The final PR must restore the approve and add the revoke as a `beforeEach` cleanup that runs *before* the per-test approve. The proposed shape is in 6.8.8.

If you ship the senior's commit as-is, every swap test will revoke (good for state hygiene) but never approve (the swap then fails because allowance is zero — unrelated to the spike's intent). That's why this can't be the final shape.

### 6.8.7 What's Still TODO Before Merge

The senior left you six items. Work them in this order:

#### 1. Fix the `@step` copy-paste bug

```ts
// e2e/desktop/tests/page/swap.page.ts:589 (approximately)
- @step("Ensure token approval")
+ @step("Revoke token approval")
async revokeTokenApproval(fromAccount: Account | TokenAccount, provider: Provider) {
```

5-second fix. Smallest commit you'll write all sprint. Do it first; it changes Allure rendering for every subsequent test run you do.

#### 3. Restore proper hook ordering in `provider.swap.spec.ts`

The dev-loop swap (6.8.6) replaces approve with revoke. The final shape keeps both: revoke as a `beforeEach` cleanup, approve as the per-test setup. Proposed diff:

```diff
  for (const { fromAccount, toAccount, provider, xrayTicket } of providerFlowTests) {
    test.describe(`Swap - ${provider.uiName} flow`, () => {
      setupEnv(true);
      test.use({ /* ... */ });

+     test.beforeEach(async ({ app }) => {
+       await app.swap.revokeTokenApproval(fromAccount, provider);
+     });

      test(
        `Swap - ${provider.uiName} flow`,
        { /* tag/annotation */ },
        async ({ app }) => {
          await addTmsLink(getDescription(test.info().annotations, "TMS").split(", "));

          const minAmount = await app.swap.getMinimumAmount(fromAccount, toAccount);
-         //await app.swap.ensureTokenApproval(fromAccount, provider, minAmount);
-         await app.swap.revokeTokenApproval(fromAccount, provider);
+         await app.swap.ensureTokenApproval(fromAccount, provider, minAmount);
          const swap = new Swap(fromAccount, toAccount, minAmount, provider);

          await performSwapUntilQuoteSelectionStep(app, swap, minAmount);
          // ... rest unchanged ...
        },
      );
    });
  }
```

Why `beforeEach` and not the test body: the cleanup is a fixture concern, not a logic step. Putting it in `beforeEach` makes it run on retries too (Playwright retries reuse the same `beforeEach`) and surfaces revoke failures separately from the swap failure in Allure.

#### 4. Mobile parity

`e2e/mobile/page/trade/swap.page.ts` already imports `approveTokenCommand` (consolidated briefing §10). Add a parallel `revokeTokenApproval` method, mirroring the desktop shape. The mobile global injection (`Object.assign(this.global, cliCommandsUtils)` at `e2e/mobile/jest.environment.ts:138`) makes `revokeTokenCommand` available on `global` automatically; only the POM method needs adding. Code in 6.8.10.

#### 5. Tighten the type guard

`revokeTokenCommand`'s signature is `(account: TokenAccount, spender: string)`. The POM accepts `Account | TokenAccount` (correct, for ergonomics) and guards on `!fromAccount.parentAccount`. But the guard only checks one of the two discriminants of `TokenAccount`. Tighten the call site so TypeScript can narrow:

```ts
async revokeTokenApproval(fromAccount: Account | TokenAccount, provider: Provider) {
  if (!provider.contractAddress) return;
  if (!("parentAccount" in fromAccount) || !fromAccount.parentAccount) return;
  // fromAccount is now TokenAccount within the rest of the body
  ...
  const result = await revokeTokenCommand(fromAccount, provider.contractAddress);
  ...
}
```

Optional but cheap. Do it if review pushes back; ship without it otherwise.

#### 6. Spike report

QAA-615 is a spike. The deliverable is **not** just code — it's a written answer. Outline in 6.8.16.

#### 7. Test against a real device

Speculos is canonical for CI but a real Nano S Plus or Stax run on Ethereum mainnet confirms the clear-sign content matches expectations. Quick validation:

- Plug a Nano S Plus seeded with the QAA test seed
- Switch to Ethereum mainnet (the network your team uses for swap regression)
- Run the spec with `MOCK=0` (to use a real device instead of Speculos)
- Tap through the revoke screens manually
- Verify the device displays "Type: Approve / Amount: 0 USDT / Spender: 0xRouter"

Pass that test once before declaring done.

### 6.8.8 The Proposed Final Spec Shape

Putting items #2 and #1 together, the final `provider.swap.spec.ts` body looks like:

```ts
for (const { fromAccount, toAccount, provider, xrayTicket } of providerFlowTests) {
  test.describe(`Swap - ${provider.uiName} flow`, () => {
    setupEnv(true);

    test.use({
      teamOwner: Team.SWAP,
      userdata: "skip-onboarding-with-last-seen-device",
      speculosApp: provider.app,
      cliCommandsOnApp: [
        [
          { app: fromAccount.currency.speculosApp, cmd: liveDataWithAddressCommand(fromAccount) },
          { app: toAccount.currency.speculosApp,   cmd: liveDataWithAddressCommand(toAccount)   },
        ],
        { scope: "test" },
      ],
    });

    test.beforeEach(async ({ app }) => {
      // Clean state before each iteration. No-op if already zero.
      await app.swap.revokeTokenApproval(fromAccount, provider);
    });

    test(
      `Swap - ${provider.uiName} flow`,
      {
        tag: ["@NanoSP", "@LNS", "@NanoX", "@Stax", "@Flex", "@NanoGen5", "@ethereum", "@family-evm"],
        annotation: [{ type: "TMS", description: xrayTicket }],
      },
      async ({ app }) => {
        await addTmsLink(getDescription(test.info().annotations, "TMS").split(", "));

        const minAmount = await app.swap.getMinimumAmount(fromAccount, toAccount);
        await app.swap.ensureTokenApproval(fromAccount, provider, minAmount);
        const swap = new Swap(fromAccount, toAccount, minAmount, provider);

        await performSwapUntilQuoteSelectionStep(app, swap, minAmount);
        await app.swap.selectSpecificProvider(provider);

        await app.swap.clickExchangeButton();
        await app.swap.checkElementsPresenceOnSwapApprovalStep();
        await app.swap.clickExecuteSwapButton();
        await app.swap.clickContinueButton();
        await app.speculos.verifyAmountsAndAcceptSwap(swap, minAmount);
        await app.swap.expectTransactionSentToasterToBeVisible();
      },
    );
  });
}
```

Read the order:
1. `beforeEach` revokes — state goes to zero.
2. `ensureTokenApproval` runs the pre-check (`isTokenAllowanceSufficientCommand` returns 0 because we just revoked), so it always runs the approve flow.
3. The swap runs against a freshly-approved allowance every time.

### 6.8.9 Speculos Coordination

The final spec spawns Speculos **three times** per test:

1. `revokeTokenApproval` — Ethereum app (revoke calldata)
2. `ensureTokenApproval` — Ethereum app (approve calldata)
3. The fixture's swap Speculos — Exchange app (swap signing)

Each `launchSpeculos` / `cleanSpeculos` pair takes ~10–30 seconds depending on local Docker vs remote Speculos pool. Three pairs is 30–90 seconds of overhead per test. The swap suite has ~10 tests today; that's 5–15 minutes of pure Speculos boot time per run.

**Possible optimisation: share the Ethereum Speculos between revoke and approve.**

Both run on the same app. You could refactor:

```ts
// pseudo-code, not the final API
test.beforeEach(async ({ app }) => {
  await app.speculos.withApp("Ethereum", async () => {
    await revokeTokenCommand(fromAccount, provider.contractAddress);
    if (currentAllowanceIsBelowMin) {
      await approveTokenCommand(fromAccount, provider.contractAddress, minAmount * 1.2);
    }
  });
});
```

That would halve the boot overhead. **It is out of scope for QAA-615.** File it as a follow-up; mention it in the spike report's "recommended next steps" section. Do not slip it into the spike PR — review-scope creep is the canonical way for spike PRs to die.

For now, accept the three-launch shape. Sub-30-second-per-test Speculos overhead is the team's current baseline.

### 6.8.10 Mobile Implementation

Mobile parity is item #3 on your do-list. The proposed addition:

```ts
// e2e/mobile/page/trade/swap.page.ts
// (somewhere near the existing approve/swap methods)

async revokeTokenApproval(fromAccount: Account | TokenAccount, provider: Provider) {
  if (!provider.contractAddress || !("parentAccount" in fromAccount) || !fromAccount.parentAccount) return;

  const previousSpeculosPort = getEnv("SPECULOS_API_PORT");
  const speculos = await launchSpeculos(fromAccount.currency.speculosApp.name);
  try {
    // revokeTokenCommand is already global on mobile via jest.environment.ts:138
    const result = await revokeTokenCommand(fromAccount, provider.contractAddress);
    await allure.description(`Token revoke result for ${provider.uiName}:\n\n ${result}`);
  } finally {
    await deleteSpeculos(speculos.id);
    if (previousSpeculosPort > 0) {
      await registerSpeculos(previousSpeculosPort);
    }
  }
}
```

Differences from desktop:
- Type-guarded narrowing (item #4 applied here too).
- Allure API differs slightly between Playwright and jest-allure-circus. Match whatever the existing mobile POM methods do for attachments.
- No `@step` decorator on mobile (the mobile POM uses `@Step` capitalised — confirm the file's existing convention before adding; copy from `approveTokenApproval` if it exists, or from a neighbouring `@Step` method).

The `revokeTokenCommand` import is automatic via the global injection — no `import` statement needed in the mobile POM file. That said, do add it explicitly if your team's lint rules require it; the global is for spec files, not POM files.

Wire it into the mobile spec the same way as desktop: `beforeEach` cleanup, then per-test approve. The mobile equivalent of `provider.swap.spec.ts` lives at `e2e/mobile/specs/swap/`.

### 6.8.11 Verifying Revoke Worked

Three ways to confirm the revoke landed, ranked by effort:

#### Option A — CLI read-back (canonical)

```ts
// after the revoke, in a debug spec or one-off script
import { runCliGetTokenAllowance, isTokenAllowanceSufficientCommand } from "@ledgerhq/live-common/e2e/cliCommandsUtils";

const allowance = await isTokenAllowanceSufficientCommand(fromAccount, provider.contractAddress, "1");
console.log("Allowance after revoke:", allowance);  // expect "0" or false
```

This uses the existing reader (Chapter 6.4 §5). Zero return means the revoke landed.

#### Option B — revoke.cash (Chapter 6.3)

Open `https://revoke.cash`, paste the QAA test address (read-only, no wallet connection needed), select Ethereum mainnet, find the token + spender pair. After a successful revoke, the entry should disappear from the list.

#### Option C — Etherscan read tab

`https://etherscan.io/token/<contract>#readContract`, scroll to `allowance(owner, spender)`, paste the test wallet address as `owner` and the provider router as `spender`. Returns 0 in raw smallest units after a successful revoke.

Use Option A in the spec itself if you ever want a hard assertion. Use Options B/C for one-off audits and during the spike's manual verification step.

### 6.8.12 The PR Shape

Slice the spike PR into four small, reviewable commits plus a non-code deliverable:

| # | Type | Files | Rationale |
|---|---|---|---|
| 1 | `test(common)` | `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts` | The senior's existing commit, possibly with the `@step` description fix folded in (it's small enough that a separate commit feels heavy). |
| 2 | `test(desktop)` | `e2e/desktop/tests/page/swap.page.ts`, `e2e/desktop/tests/specs/provider.swap.spec.ts` | The POM method (with corrected `@step`) and the `beforeEach` wiring. |
| 3 | `test(mobile)` | `e2e/mobile/page/trade/swap.page.ts`, mobile spec | Mobile parity. |
| 4 | `refactor(common)` | `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts` | (Optional) Tightened type guard, if review asked for it. |
| — | (no commit) | Confluence or Jira | Spike report. Lives outside the repo. |

Why four commits and not one squash:

- Reviewers can land #1 independently (it's a pure live-common change with no spec wiring, and other teams can already use `revokeTokenCommand` once it's merged).
- Mobile parity (#3) might be reviewed by a different team owner than desktop (#2). Splitting them lets each owner review their own surface.
- The optional refactor (#4) is a "nice to have" — keeping it separate means you can drop it if review pushes back without unwinding the spike.

### 6.8.13 Reviewer Routing

Code-owners on each surface:

| File | Likely owner |
|---|---|
| `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts` | live-common owner + wallet-xp |
| `e2e/desktop/tests/page/swap.page.ts`, `provider.swap.spec.ts` | wallet-xp + Swap (PTX) team |
| `e2e/mobile/page/trade/swap.page.ts`, mobile spec | wallet-xp mobile + Swap (PTX) |

Tag the reviewers in your PR description, not just via CODEOWNERS auto-routing. The Swap team specifically wants visibility on anything that changes their regression flow — they own the contract that `revokeTokenApproval` is implementing.

### 6.8.14 Common First-Run Failures

Things that will go wrong on your first end-to-end run, and how to debug:

#### "CLI subprocess hung at confirmation"

Symptom: the test sits at `await app.swap.revokeTokenApproval(...)` for 60+ seconds, then Playwright times out.

Cause: `approveToken()` couldn't find the expected screen labels. Either the device class doesn't match the labels in `families/evm.ts`, or the Speculos firmware version renders the screen differently.

Debug:
- `console.log(await fetchCurrentScreenTexts(device.port))` — see what's actually on screen
- Check `families/evm.ts` for the `DeviceLabels` constants vs. what Speculos shows
- Confirm `SPECULOS_DEVICE` matches your `--currency` (Nano S Plus vs Stax render different labels)

#### "DISABLE_TRANSACTION_BROADCAST not honoured"

Symptom: revoke completes but allowance on-chain doesn't change.

Cause: env not propagated to the subprocess, or the flip-and-restore logic ran out of order.

Debug:
- Add `console.log("ENV:", process.env.DISABLE_TRANSACTION_BROADCAST)` inside `revokeTokenCommand` after the flip
- Confirm `setDisableTransactionBroadcastEnv` actually mutates `process.env` (it should — the helper at the bottom of `cliCommandsUtils.ts` does both `setEnv` and `process.env`)
- Check there's no upstream test setting `DISABLE_TRANSACTION_BROADCAST=1` after `revokeTokenCommand` flipped it but before the CLI started

#### "Speculos port collision"

Symptom: `launchSpeculos` errors with "port in use" or the test inherits the wrong device.

Cause: previous test's `cleanSpeculos` was skipped — usually because a `try/finally` was missing somewhere.

Debug:
- `docker ps` — see if old Speculos containers are still up
- `lsof -i :<port>` — see what's holding the port
- Confirm every `launchSpeculos` in your codebase has a paired `cleanSpeculos` inside a `finally`
- Kill the strays: `docker kill $(docker ps -q --filter ancestor=ghcr.io/ledgerhq/speculos:master)`

#### "Allure shows wrong description"

Symptom: every step in the Allure report says "Ensure token approval" twice.

Cause: the `@step` copy-paste bug from 6.8.7 #1.

Fix: change the decorator to `@step("Revoke token approval")`. You knew this was coming.

#### "Test passes locally, fails in CI"

Symptom: green on your machine, red on CI.

Causes (in decreasing order of likelihood):
- `REMOTE_SPECULOS=true` on CI — the remote pool has different latency. Add `await waitForSpeculosReady(device.id)` (already in `launchSpeculos` for remote; check the path is hit)
- CI's QAA seed has different account state than yours — the on-chain allowance differs at run-start
- CI is pointed at a different RPC endpoint than your local — confirm `DEFAULT_NETWORK` env

#### "Allowance is zero before revoke even runs"

Not a failure, but worth noting: if a previous CI run's `afterEach` (or a previous test's natural cleanup) zeroed the allowance, your `beforeEach` revoke is a no-op. The test is still correct. Don't add a "did anything happen?" assertion — revoke from zero is valid.

### 6.8.15 What's NOT in Scope for QAA-615

The spike has a finite boundary. Things you *do not* need to ship:

- **Symmetric `runCliApprove` standalone command.** `approveTokenCommand` already exists; nothing new needed there.
- **Token registry expansion.** Adding new ERC-20s to the test fixture is QAA-919's overall scope but a separate ticket. Don't pull it in.
- **Multi-spender batch revoke.** A single-call helper that revokes from N spenders at once would be useful but is a future optimisation. File as follow-up.
- **Migration of every test to use revoke as `beforeEach`.** Only `provider.swap.spec.ts` is in scope for QAA-615. Other specs (e.g., `validation.swap.spec.ts`) might benefit, but each is a separate PR.
- **EIP-2612 permit support.** Some newer tokens (USDC v2, DAI) support gasless permit signatures, which would be a much faster cleanup mechanism than on-chain revoke. Out of scope; would need a different CLI command and a different device flow.
- **Speculos sharing optimisation.** 6.8.9 — file as follow-up, do not bundle into spike PR.
- **Spike report → Confluence migration.** If your team writes spike reports in Jira comments, keep them in Jira. Don't promote a comment to a Confluence page as part of the same PR.

The senior already drew this boundary by writing 37 lines instead of 200. Honour it.

### 6.8.16 The Spike Report

The spike report is a non-code deliverable. Post it as a Jira comment on QAA-615 (or a Confluence page, depending on your team's convention). Outline:

#### Section 1 — Question

Restate verbatim what the ticket asked. One paragraph.

> QAA-615 asked: find the smallest, lowest-risk way to revoke an existing ERC-20 allowance from inside an E2E test, so that the broadcast-enabled regression suite can clean up provider allowances between iterations.

#### Section 2 — Answer

Yes, here's what shipped. Link the PR. One paragraph.

> Shipped: a thin `revokeTokenCommand` typed wrapper in `cliCommandsUtils.ts`, plus a `revokeTokenApproval` POM method on both desktop and mobile swap pages, plus a `beforeEach` cleanup hook in `provider.swap.spec.ts`. Total ~50 LOC of new code (mobile parity adds 20 to the senior's 37). PR: <link>.

#### Section 3 — Path chosen

State the path and why.

> **Path C — thin wrapper following the existing `approveTokenCommand` pattern.** Reuses the legacy CLI's existing `--mode revokeApproval` flag (already in `apps/cli/src/commands/blockchain/send.ts`); reuses the existing `runCliTokenApproval` Layer-2 spawn helper; reuses the existing `approveToken()` Layer-4 device-tap helper. No changes to the CLI binary, no new Speculos plumbing, no new device labels. The whole spike is symmetry with the existing approve flow.

#### Section 4 — Alternatives rejected

Two paragraphs each, max.

- **Path A — wallet-cli new `revoke` subcommand.** Rejected. wallet-cli is out of QAA scope (per Chapter 6.1). Building parallel infra in a CLI we don't own would create maintenance debt for two teams.
- **Path B — manual `revoke.cash` script.** Rejected for automation. Useful for one-off cleanup (Chapter 6.3) but not deterministic enough for CI.
- **Path D — bypass the device entirely with EIP-2612 permits.** Rejected for scope. Would require token-by-token `permit()` support audit and a different signing path. File as a future optimisation.

#### Section 5 — LOC actually shipped

Show the numbers. Reviewers love numbers.

> - `cliCommandsUtils.ts`: +20
> - `swap.page.ts` (desktop): +15
> - `swap.page.ts` (mobile): +20
> - `provider.swap.spec.ts`: +3 / -1
> - **Total: ~57 LOC of new code, 0 LOC of CLI changes.**

#### Section 6 — Recommended next steps

Bulleted list. Each one a candidate follow-up ticket.

- Speculos-app sharing between revoke and approve (6.8.9)
- Multi-spender batch revoke helper (6.8.15)
- Migrate other broadcast-enabled specs to use revoke as `beforeEach`
- EIP-2612 permit support investigation (separate spike)

#### Section 7 — Evidence

Links and screenshots.

- PR link
- Allure run link with the new step visible (after fixing `@step`)
- Manual real-device run screenshot (6.8.7 #6)
- Etherscan link showing allowance went from non-zero to zero on the test seed

A 7-section, ~1-page spike report is the right size. Do not write 4 pages. The deliverable is "the question is answered"; not "every neuron I had during the investigation".

### 6.8.17 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — QAA-615 Walkthrough</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> The senior's <code>revokeTokenApproval</code> POM method is decorated with <code>@step("Ensure token approval")</code>. Why is this wrong, and what is the fix?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The decorator should be <code>@Step</code> with a capital S; rename the import</button>
<button class="quiz-choice" data-value="B">B) The decorator should be removed; revoke does not need an Allure step</button>
<button class="quiz-choice" data-value="C">C) Copy-paste from <code>ensureTokenApproval</code>: every Allure entry rendered from this method will be mislabelled. Change to <code>@step("Revoke token approval")</code></button>
<button class="quiz-choice" data-value="D">D) The decorator's argument must include the test ticket: <code>@step("QAA-615 — Revoke token approval")</code></button>
</div>
<p class="quiz-explanation">The senior left a copy-paste residue from the twin <code>ensureTokenApproval</code> method. Allure step descriptions are user-visible — when both methods run in the same test, the Allure report would show "Ensure token approval" twice, confusing reviewers. Fix is a one-word change in the decorator argument. This is item #1 on the do-list.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> Inside <code>revokeTokenCommand</code>, the senior writes <code>const original = setDisableTransactionBroadcastEnv("0")</code>. What does this do and why?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Disables broadcast for the next CLI call so the revoke is signed but not sent</button>
<button class="quiz-choice" data-value="B">B) Sets <code>DISABLE_TRANSACTION_BROADCAST</code> to a falsy <code>"0"</code> (i.e., enables broadcast), and captures the previous value so the <code>finally</code> block can restore it. Required because revoke must actually change on-chain state to be useful</button>
<button class="quiz-choice" data-value="C">C) Sets a timeout of 0 ms on the broadcast pipeline</button>
<button class="quiz-choice" data-value="D">D) Caches the original env so it can be passed to <code>runCliTokenApproval</code></button>
</div>
<p class="quiz-explanation">The <code>"0"</code> string makes the env flag falsy in the CLI's check (<code>opts["disable-broadcast"] || getEnv("DISABLE_TRANSACTION_BROADCAST")</code>). Capturing <code>original</code> is the safety net: the <code>finally</code> block restores whatever the env was before, ensuring the test doesn't leak a flipped broadcast state into subsequent CLI calls. See Chapter 6.6.7.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> The POM method does <code>const previousSpeculosPort = getEnv("SPECULOS_API_PORT")</code> before <code>launchSpeculos</code>, and <code>cleanSpeculos(speculos, previousSpeculosPort)</code> in the <code>finally</code>. Why?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The fixture has already started a Speculos for the swap itself (Exchange app). The revoke needs a different app (Ethereum app), so we save the port, swap to the Ethereum-app Speculos for the revoke, then restore the swap's Speculos port so the rest of the test continues to talk to the Exchange Speculos</button>
<button class="quiz-choice" data-value="B">B) Speculos crashes if launched twice in the same process; the snapshot is a workaround</button>
<button class="quiz-choice" data-value="C">C) <code>previousSpeculosPort</code> is logged to Allure for debugging</button>
<button class="quiz-choice" data-value="D">D) The port number is used as a randomness seed for transaction nonces</button>
</div>
<p class="quiz-explanation">The snapshot/restore pattern (Chapter 6.5) is how a single test coordinates multiple Speculos instances on different apps. The swap fixture starts the Exchange-app Speculos; the revoke runs against the Ethereum-app Speculos; cleanup restores the Exchange-app port so the rest of the swap test continues normally. Without the snapshot, the rest of the test would inherit the (now-stopped) Ethereum-app Speculos's port and break.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q4.</strong> The senior's <code>provider.swap.spec.ts</code> diff comments out <code>ensureTokenApproval</code> and replaces it with <code>revokeTokenApproval</code>. Is this the final shape that should ship in the PR?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Yes — revoke now serves the same purpose as ensureTokenApproval did</button>
<button class="quiz-choice" data-value="B">B) Yes — the swap suite no longer needs allowance setup once revoke is wired in</button>
<button class="quiz-choice" data-value="C">C) No — this is dev-loop code. The senior commented out the approve to test revoke in isolation. The final shape uses revoke as a <code>beforeEach</code> cleanup AND keeps <code>ensureTokenApproval</code> as the per-test setup, so each iteration starts at zero and ends with a fresh allowance</button>
<button class="quiz-choice" data-value="D">D) No — revoke should run in <code>afterEach</code>, not as a replacement for approve</button>
</div>
<p class="quiz-explanation">Without <code>ensureTokenApproval</code>, the swap step has no allowance to spend and will fail. Revoke and approve are complementary: revoke clears stale state, approve sets the allowance the swap needs. Final shape: revoke in <code>beforeEach</code>, approve in the test body (6.8.8). <em>Either</em> <code>beforeEach</code> <em>or</em> <code>afterEach</code> would work for the cleanup; <code>beforeEach</code> is preferred because it survives test retries cleanly.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> What does <code>fromAccount.parentAccount.address</code> represent for a <code>TokenAccount</code>, and why does the POM check <code>!fromAccount.parentAccount</code> in its early-return guard?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The parent's address is the spender to revoke; the guard skips empty parents</button>
<button class="quiz-choice" data-value="B">B) <code>parentAccount</code> is the EVM (or other native) account that holds the token sub-account. Its <code>address</code> is the EOA on chain — the <em>owner</em> of any allowance. The guard skips native <code>Account</code>s (which have no <code>parentAccount</code>) because they have no allowance to revoke</button>
<button class="quiz-choice" data-value="C">C) <code>parentAccount.address</code> is the device's master public key</button>
<button class="quiz-choice" data-value="D">D) It's the previous account in iteration order; used for log breadcrumbs</button>
</div>
<p class="quiz-explanation">In live-common's account model, a TokenAccount (e.g., USDT on Ethereum) has a <code>parentAccount</code> pointer to its native parent (the Ethereum account). The parent's <code>address</code> is the EOA that owns the allowance on chain — what would be passed as <code>owner</code> to <code>allowance(owner, spender)</code>. Native accounts have no parent and no allowance to clear; the guard skips them.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q6.</strong> The CLI is invoked with <code>waitConfirmation: true</code> and the broadcast env flipped to <code>"0"</code>. What is the actual behaviour, end-to-end?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The CLI signs the revoke tx, broadcasts it (because broadcast is enabled), then blocks until the tx is confirmed in a block before exiting. The wrapping promise resolves only after on-chain confirmation, so the next test step sees the new allowance state</button>
<button class="quiz-choice" data-value="B">B) The CLI signs only — the env disables broadcast — and waits for a fake "confirmation" emitted by Speculos</button>
<button class="quiz-choice" data-value="C">C) The CLI broadcasts but exits immediately; <code>waitConfirmation</code> is a no-op when broadcast is enabled</button>
<button class="quiz-choice" data-value="D">D) <code>waitConfirmation</code> sets a timeout on the device-tap helper, not the broadcast pipeline</button>
</div>
<p class="quiz-explanation">The CLI's rxjs pipe (Chapter 6.4 §3.1) only takes the broadcast branch when <code>DISABLE_TRANSACTION_BROADCAST</code> is falsy <em>and</em> <code>--disable-broadcast</code> is not passed. With env <code>"0"</code>, broadcast happens. The <code>--wait-confirmation</code> flag (set by <code>waitConfirmation: true</code>) keeps the subprocess alive until <code>waitForTransactionConfirmation</code> resolves — i.e., the tx is in a block. This is what makes the revoke deterministic for the next test: when the wrapper returns, the chain has settled.</p>
</div>

<div class="quiz-score"></div>
</div>

<div class="chapter-outro">
You now own the whole shape of QAA-615. The senior's commit was a runway, not a destination — 37 lines that prove the path works, with four obvious do-list items waiting on top: the <code>@step</code> rename, the proper hook ordering, mobile parity, and a tightened type guard. Plus the spike report. None of those is hard; all of them are necessary. Ship them. Chapter 6.9 turns the page from this guided walkthrough to your own hands: a set of CLI exercises and challenges that exercise every layer of what you just read, with grading rubrics so you can self-assess before bringing your work to a reviewer.
</div>

## CLI Exercises and Challenges

<a id="cli-exercises-and-challenges"></a>

<div class="chapter-intro">
Reading the previous eight chapters gives you the map. These exercises put you on the trail. They progress from a single CLI invocation typed by hand to a paper-sketch of mobile parity for the senior's revoke method. Every exercise has an explicit verification step; if you can't verify it yourself, grab a reviewer before moving on. Do them in order — each builds on the muscle memory of the one before. The first three are warm-ups (hand-run, paper read, trace memo); 4 and 5 force you to think about local-only debugging hygiene and Speculos lifecycle leaks; 6 is an audit pattern you'll repeat throughout your career; 7 maps directly onto the QAA-615 follow-up work that may land on your desk.
</div>

### 6.9.1 Exercise 1: Read your first allowance (15 min)

**Objective.** Run the legacy `apps/cli` directly, with no test wrapping, to read an ERC-20 allowance for an arbitrary owner/spender pair on Ethereum mainnet. This grounds every later abstraction (`runCliGetTokenAllowance`, `isTokenAllowanceSufficientCommand`, `ensureTokenApproval`) in a single thing you've already typed.

**Instructions.**
1. From the monorepo root, run:
   ```bash
   node apps/cli/bin/index.js tokenAllowance \
     --currency ethereum \
     --token ethereum/erc20/usd_coin \
     --spender 0x1111111254EEB25477B68fb85Ed929f73A960582 \
     --ownerAddress 0xYOUR_TEST_ADDRESS \
     --format json
   ```
2. Replace `0xYOUR_TEST_ADDRESS` with any Ethereum mainnet address you want to inspect (read-only, no signing involved).
3. Read the JSON output line by line. Identify the `allowanceStr` value and the `unitMagnitude`.

**Verification.** Stdout is valid JSON. The `allowanceStr` is a string of digits in the token's smallest unit (for USDC: 6 decimals, so `1000000` = 1 USDC). If you used a fresh address, the value will be `"0"`.

**Hints.**
- The CLI prebuild step (`zx ./scripts/gen.mjs`) must have run at least once. If you get "command not found" inside the CLI, run `pnpm --filter @ledgerhq/live-cli prebuild` first.
- `--format json` is what makes the output machine-parseable. Without it you'll get a human-readable line that the typed wrapper can't parse.
- The spender `0x1111...0582` is 1inch v5 router; any contract that ever called `approve()` against your address will surface here.

**Stretch goal.** Pipe the output through Node and call `parseTokenAllowanceCliOutput` (from `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts`) on the raw stdout. Confirm it returns `{ allowanceStr, unitMagnitude }` and matches what your eye saw in the JSON.

**Why this is the first exercise.** Every later abstraction — the typed wrapper, the curried fixture helper, the POM method that orchestrates Speculos — collapses to *this exact subprocess invocation* under the hood. If you can't run the CLI by hand, you can't reason about what the harness is doing on your behalf when something fails. Spending fifteen minutes here saves you hours later when a CI failure surfaces a stack trace pointing at `runCliCommand`.

**Common mistake.** Skipping `--format json` because the human-readable output looks "clearer". The typed wrappers parse JSON; the human-readable shape is for engineers reading their terminal. If you only ever call the CLI in human-readable mode, you'll be surprised when `parseTokenAllowanceCliOutput` throws on stdout you produced by hand. Always pass `--format json` when you intend to feed the output back into TypeScript — even ad hoc.

### 6.9.2 Exercise 2: Wrap an allowance read in a typed helper (20 min)

**Objective.** Read the existing `isTokenAllowanceSufficientCommand` and sketch a hypothetical sibling helper. This is a paper exercise — you are not shipping code, you are training your eye on the wrapper pattern.

**Instructions.**
1. Open `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts`.
2. Read `isTokenAllowanceSufficientCommand(account, spender, minAmount)` end to end. Note: it builds CLI args, calls `runCliGetTokenAllowance`, parses with `parseTokenAllowanceCliOutput`, compares against `minAmount`, and returns a string (the allowance) when sufficient or the numeric `0` otherwise.
3. On paper (or in a scratch file you don't commit), sketch a `getCurrentAllowanceCommand(account, spender)` that returns the raw `allowanceStr` always — no comparison, no boolean coercion. Signature: `(account: TokenAccount, spender: string) => Promise<string>`.
4. Decide: would you curry it? Would the curry buy you anything?

**Verification.** Your sketch lists three things explicitly: (a) the CLI args you'd build, (b) which engine helper you'd call (`runCliGetTokenAllowance`), (c) which parser you'd use (`parseTokenAllowanceCliOutput`). You can articulate why this isn't a fixture-shaped curried helper.

**Hints.**
- Re-read briefing §5 (the Layer 3 wrapper table) before sketching.
- The curried-function pattern (`(account) => (userdataPath) => Promise<void>`) exists because the desktop fixture engine threads `userdataPath` in at the right lifecycle moment. A one-shot read needs no such threading — a plain async function is the right shape.
- `getEvmTokenAllowance` (in `libs/ledger-live-common/src/families/evm/getTokenAllowance.ts`) is what `tokenAllowance.ts` calls under the hood. Your wrapper would still go through the CLI, not directly through this function — that's the point of the five-layer separation.

**Stretch goal.** Write 50 words explaining why a typed wrapper around an allowance read is more valuable than calling `getEvmTokenAllowance` directly from a test. (Answer involves: subprocess isolation, the same code path as the broadcast-enabled write, and the Speculos transport registration that the CLI already handles.)

**Why this exercise is paper-only.** Adding real helpers to `cliCommandsUtils.ts` requires CODEOWNERS review and a JIRA ticket. The point here is to internalize the *shape* of a Layer-3 wrapper so that, when QAA-XXX one day asks for "a way to read the allowance without the boolean coercion", you've already drawn the picture and you don't waste the first half-hour staring at a blank file.

### 6.9.3 Exercise 3: Trace a `cliCommands` fixture in a real test (30 min)

**Objective.** Walk a single line of test metadata down through five layers and back up. By the end you should be able to draw the lifecycle on a whiteboard.

**Instructions.**
1. Open `e2e/desktop/tests/specs/earn.v2.spec.ts` and jump to line 86 (or any of the lines listed in briefing §10: 132, 195, 220, 254, 298, 327 — they all use the same pattern).
2. Find the `cliCommands: [liveDataCommand(account)]` field.
3. Walk the chain in order:
   - **Layer 5 (the spec):** what helper is in the array? Note: `liveDataCommand(account)` is a *call* — it returns a curried function. The array holds the curried result.
   - **Fixture engine:** open `e2e/desktop/tests/fixtures/common.ts` and find where the `cliCommands` field is consumed. Each curried function is invoked with the `userdataPath` for the test.
   - **Layer 3 (`cliCommandsUtils.ts`):** read `liveDataCommand` itself. It builds a CLI command string and calls `runCliLiveData`.
   - **Layer 2 (`runCli.ts`):** `runCliLiveData` formats the args (with the `+` separator quirk), then calls `runCliCommand` which spawns `node apps/cli/bin/index.js …` as a subprocess.
   - **Layer 1 (`apps/cli`):** the `liveData` command (in `apps/cli/src/commands/live/`) reads the args, runs an account scan, and writes the resulting account JSON into the userdataPath's `app.json`.
   - **Back up to the spec:** Live boots with that `app.json` already populated. The test asserts on UI state that depends on the seeded account.
4. Write a 200-word memo titled "Lifecycle of a single cliCommands entry" describing what you just traced.

**Verification.** Show the memo to a peer. They should be able to identify, from your text alone, (a) where the curry is unwrapped, (b) where the subprocess is spawned, (c) what file gets mutated on disk, and (d) what the test ultimately asserts on.

**Hints.**
- The `+` separator (briefing §4) is easy to miss. The engine joins args with `+`, then splits them back to argv before `spawn` — this avoids shell-escaping pain.
- `liveData` is an `apps/cli/src/commands/live/` command, not a `blockchain/` command. The blockchain subdir is for tx-shaped commands (send, broadcast, getAddress, tokenAllowance).
- If your memo skips the "what gets mutated on disk" step, re-read it. That's the load-bearing part.

**Stretch goal.** Repeat the trace for `liveDataWithAddressCommand` (used by `validation.swap.spec.ts`, `accounts.swap.spec.ts`, etc.). Identify the one extra thing it does compared to `liveDataCommand` (hint: it caches the device address on the account object after seeding).

**Calibration check.** A clean trace memo should make the reader say "ah, of course" at every transition. If a peer reads it and asks "wait, when does the curry get called?" your memo missed the fixture-engine link. If they ask "where does the JSON go?" your memo missed the userdataPath mutation. Iterate until the memo answers both questions on first read.

**Common mistake.** Treating `liveDataCommand(account)` as a function call that does the work. It is *not*. It returns a curried function. The work happens later, when the fixture engine invokes that curried function with `userdataPath`. A new joiner who reads `cliCommands: [liveDataCommand(account)]` and assumes the seeding has already happened by the time the array is built will be confused when they try to assert on `account.address` immediately after — the address isn't populated until the engine actually runs the curried call.

### 6.9.4 Exercise 4: Sketch a revoke.cash verification helper (45 min)

**Objective.** Design — on paper — a debugging-only helper that bridges manual revoke.cash workflow with programmatic allowance polling. This helper will *never* go into a CI test. It exists to make local debugging less awful.

**Instructions.**
1. Sketch the API of `manualRevokeAndVerify(account: TokenAccount, spender: string): Promise<void>`.
2. The helper does two things in order:
   - **(a) Print the revoke.cash URL** for the account's address: `https://revoke.cash/address/<account.address>?chainId=1` (Ethereum mainnet). Print it big, with newlines around it, so the engineer running the script can't miss it. Optionally `console.log` instructions: "Open this URL, connect MetaMask, click Revoke for the spender at address X, sign in MetaMask. This script will detect the on-chain change and exit."
   - **(b) Poll `getEvmTokenAllowance`** every 10 seconds. Time out at 5 minutes. Exit cleanly when the allowance drops to `0`. Print elapsed time on each tick so the engineer can see progress.
3. Document the use case explicitly: when Speculos is acting up, when a CI revoke failed and you want to clean state by hand without restarting the harness, when auditing what a real seed has approved.
4. Document why this never goes into a CI test: it requires human input. CI is non-interactive. There is no MetaMask in CI. The whole point of the typed wrappers (Ch 6.4) is to drive Speculos so this kind of human-in-the-loop helper is unnecessary in CI.

**Verification.** Your sketch is one TypeScript file's worth of pseudocode (not real code). It has: function signature, a clear `console.log` block for the URL, a polling loop with a timeout, a clear comment block stating "DEBUGGING ONLY — DO NOT USE IN CI". A peer reading it should immediately understand the flow.

**Hints.**
- `getEvmTokenAllowance` is the read-only function from `libs/ledger-live-common/src/families/evm/getTokenAllowance.ts`. You can call it directly from a debugging helper — no subprocess needed because no signing is involved.
- The 5-minute timeout is generous; revoking on Ethereum mainnet usually confirms in 15-60 seconds. The timeout exists to avoid an infinite hang if the engineer wandered off.
- For mainnet debugging (very rare for QAA), swap the `chainId=11155111` to `chainId=1` and adjust the network in the polling call.

**Stretch goal.** Add a third option to your sketch: an "auto-paste" mode that, on macOS, runs `pbcopy` with the revoke.cash URL so the engineer can `Cmd+V` into a browser. Document that this is shell-out and is therefore platform-specific (Linux has `xclip`, Windows has `clip`).

**The line you must not cross.** A debugging helper that lives in a developer's local scratch folder is fine. The same helper, copied into the test harness "in case it's useful", becomes a footgun: a non-deterministic test that requires human input cannot run in CI, will be skipped, and skipped tests rot. Codify the rule for yourself now: human-in-the-loop helpers stay outside `e2e/` directories. If you find one that snuck in, that's a CF-XXX ticket.

### 6.9.5 Exercise 5: Speculos snapshot/restore manual test (45 min)

**Objective.** Demonstrate, on your own machine, the failure mode that the Speculos snapshot/restore pattern exists to prevent. Forewarned is forearmed.

**Instructions.**
1. Create a scratch script `scripts/leak-speculos-port.ts` (do not commit). It should do, in order:
   - (a) Launch Speculos with the Ethereum app on a port — call it `portA`. Save `portA` to a variable. Set `SPECULOS_API_PORT=portA`.
   - (b) Run `node apps/cli/bin/index.js getAddress --currency ethereum --path "44'/60'/0'/0/0"` and confirm it succeeds.
   - (c) Launch a *second* Speculos with the Bitcoin app on `portB`. Set `SPECULOS_API_PORT=portB`. Do **not** save the original `portA`.
   - (d) Run `node apps/cli/bin/index.js getAddress --currency bitcoin --path "44'/0'/0'/0/0"` and confirm it succeeds.
   - (e) Stop both Speculos processes (`stopSpeculos` for each). Crucially, **do not restore `SPECULOS_API_PORT` to anything** — leak it.
2. Now, in the same Node process or shell session, run a third `getAddress` against currency `ethereum`. The env var still points at `portB` (or — depending on order — at a stopped Speculos). Observe what happens.

**Verification.** The third command fails. The exact failure mode depends on whether `portB` is reused, in TIME_WAIT, or vanished entirely. Most common: a connection-refused error from `hw-transport-node-speculos-http`. Document the failure in three sentences: what you expected, what happened, why.

**Hints.**
- The pattern in `e2e/desktop/tests/utils/speculosUtils.ts` (briefing §8) is what `cleanSpeculos(speculos, previousPort?)` exists to handle. When `previousPort` is passed, it restores the env var. When omitted, the env var is left at its current (now-stopped) value.
- This is exactly why two-app tests in `swap.page.ts` save the previous port: `const previousSpeculosPort = getEnv("SPECULOS_API_PORT")` *before* launching, then pass it to `cleanSpeculos(speculos, previousSpeculosPort)` in `finally`.
- If the third command happens to succeed, you may have hit a port-reuse coincidence. Run a fourth, fifth, sixth — eventually one will fail. The failure is non-deterministic, which is exactly why the pattern is mandatory rather than "add it if you remember".

**Stretch goal.** Now fix your scratch script: save `previousPort` before the second launch, restore it in a `finally` block. Run the third `getAddress` again. Confirm it succeeds. You have now reproduced — by hand — the discipline that the desktop POM enforces.

**Why a manual repro matters.** You could read the snapshot/restore pattern in `swap.page.ts` a dozen times and still forget to apply it the first time you write a two-app test. Reproducing the failure mode by hand burns the muscle memory in. The first time you reach for a POM method that needs to swap Speculos apps, you'll *feel* the urge to save the previous port — which is exactly the goal of this exercise.

### 6.9.6 Exercise 6: Find a CLI helper that's missing a `try/finally` env restore (60 min)

**Objective.** Audit the codebase for the `setDisableTransactionBroadcastEnv` discipline. Either you find a bug, or you confirm the pattern is held everywhere — both outcomes are valuable.

**Instructions.**
1. Grep the monorepo:
   ```bash
   rg "setDisableTransactionBroadcastEnv" --type ts -n
   ```
2. For every call site, check:
   - (a) Is the return value (the original env value) captured into a variable?
   - (b) Is there a corresponding `setDisableTransactionBroadcastEnv(original)` call that restores it?
   - (c) Is the restore inside a `finally` block, so it runs even on throw between flip and restore?
3. Flag any call site that fails (a), (b), or (c). Write a short audit memo: which file, which line, what's missing, what the fix would be.
4. If you find no bugs, write the memo anyway: "Audit clean. Pattern held in N call sites. Sites: [list]." This is a real engineering deliverable; "we checked and nothing's wrong" is not the same as "we didn't check".

**Verification.** Your memo identifies every call site by file:line. Each site has a one-line verdict: `clean`, `missing-finally`, `missing-restore`, or `unrestored-by-design`. (The mobile module-load `setEnv("DISABLE_TRANSACTION_BROADCAST", true)` in `e2e/mobile/specs/swap/otherTestCases/swap.other.ts` is intentionally unrestored — the whole spec file runs broadcast-off; that's fine and you should annotate it as such.)

**Hints.**
- Briefing §9 lists every read site and set site. Use it as your ground truth.
- The canonical clean pattern is `approveTokenCommand` and `revokeTokenCommand` in `cliCommandsUtils.ts`. They flip to `"0"`, run the CLI command, then in `finally` restore `original`. Compare every other site to this template.
- Don't conflate "no `finally`" with "no restore". A function that restores at the end (after the CLI promise resolves) but not in `finally` is buggy: any throw between flip and restore leaks the env mutation to the rest of the test run.

**Stretch goal.** If you find a bug, open a JIRA ticket (QAA-XXX) describing it with a reproducer. If you don't, propose a lint rule (`eslint-plugin-local`) that flags `setDisableTransactionBroadcastEnv("0")` calls not followed by a `try { ... } finally { setDisableTransactionBroadcastEnv(...) }` block within the same function. Discuss tradeoffs.

**The audit pattern, generalized.** Replace `setDisableTransactionBroadcastEnv` with any function whose name starts with `set`, mutates global state (env, registry, in-memory cache), and returns a "previous value" handle. Every such function deserves the same audit. The QAA codebase has a handful: `setEnv`, `setDisableTransactionBroadcastEnv`, the Speculos port mutations in `launchSpeculos`/`cleanSpeculos`. Carry this audit reflex into Part 7 and beyond.

### 6.9.7 Exercise 7: Mobile parity for the senior's revoke (90 min)

**Objective.** Sketch the mobile-side equivalent of the desktop `revokeTokenApproval` POM method. This is the exercise that maps directly to the QAA-615 follow-up work outlined in Ch 6.8.10.

**Instructions.**
1. Open `e2e/mobile/page/trade/swap.page.ts`. Find the existing `approveTokenCommand` usage (briefing §10 names it; it's near the top of the file in a method shaped like `ensureTokenApproval`).
2. Compare with the desktop `swap.page.ts` `ensureTokenApproval` (briefing §7). They're structural twins, modulo:
   - **Allure attachment shape.** Mobile uses a slightly different Allure helper — find the existing `await allure...` calls in the mobile POM and copy that idiom.
   - **Mobile globals.** Mobile injects `cliCommandsUtils` as globals via `e2e/mobile/jest.environment.ts:138`. So `revokeTokenCommand` will already be a global in mobile specs. Check `e2e/mobile/types/global.d.ts:104-111` to confirm it's typed, or note that you'd need to add it.
   - **Mobile-specific guards.** The mobile POM may have additional early-return guards (platform branches via `isAndroid()`/`isIos()`, network reachability checks). Audit the existing `ensureTokenApproval` for these and mirror them in your sketch.
3. Sketch — on paper or in a scratch file you don't commit — a `revokeTokenApproval(fromAccount, provider)` mobile method. Same shape as desktop's: save previous Speculos port, launch Speculos for the account's currency, call `revokeTokenCommand` (the global), Allure-describe the result, clean Speculos in `finally`.
4. List the exact diffs between your mobile sketch and the desktop method. There should be 3-5 of them.
5. Sanity-check your sketch against the senior's commit (briefing §1, the desktop `revokeTokenApproval`). For every line of the desktop method, your sketch has either (a) an identical line, or (b) a justified mobile-specific deviation. There should be no orphan lines on either side.

**Verification.** Your sketch reads, structurally, like a one-to-one translation of the desktop method. Your diff list is honest: every difference is justified, none are arbitrary. A peer should be able to take your sketch, point at any line, and tell you *why* it differs from the desktop equivalent.

**Hints.**
- Mobile specs do not import `cliCommandsUtils`; the helpers are globals. So your sketch may have *no* import statement for `revokeTokenCommand` — instead, the global type definition in `global.d.ts` makes TypeScript happy.
- The mobile equivalent of `launchSpeculos` and `cleanSpeculos` may live in a different util file (search `e2e/mobile/` for `speculos`). The shape is the same, the path differs.
- If `revokeTokenCommand` is not yet typed in `global.d.ts`, that's part of the work — note it in your diff list.
- Do not merge this. The exercise terminates at "sketch and diff". Real merge work goes through QAA review and a QAA-615 follow-up ticket.

**Stretch goal.** Estimate the work breakdown for shipping this for real: types update (small), POM sketch (medium), spec wiring (medium), CI green-light requirements (depends on whether the CI runner has Speculos on the mobile path). Compare with how the senior shipped the desktop side in 37 lines across 3 files (briefing §1). Are the mobile changes likely to be larger or smaller? Why?

**Capstone purpose.** The previous six exercises trained individual reflexes (read the CLI, trace a fixture, audit env discipline, reproduce a port leak). This one binds them together: the mobile parity work touches every layer. You'll add a global type (Layer 3 mobile-injection surface), call `revokeTokenCommand` (Layer 3 wrapper, already shared), launch and clean Speculos with port restore (Layer 5 lifecycle), trust the broadcast-env discipline inside `revokeTokenCommand` itself (Layer 3 again), and produce an Allure attachment for observability. If you can sketch this method without re-reading the briefing, you've internalized Part 6.

**Common mistake.** Copy-pasting the desktop `@step("Revoke token approval")` decorator into the mobile sketch. Mobile uses the `@Step` decorator from a different module path (check the existing mobile POM imports). The decorator name is the same in spirit but the import path differs. While you're at it, check that you don't replicate the senior's copy-paste bug from QAA-615 — your mobile sketch should say "Revoke token approval", not "Ensure token approval".

### 6.9.8 Self-check checklist

Before you call Part 6 done, walk this checklist. Each item maps to one of the seven exercises plus a couple of integrative checks. If any answer is "no" or "not sure", the chapter reference next to it is your next read.

- [ ] I can run `node apps/cli/bin/index.js tokenAllowance` from memory with the right flags. (Ex 1, Ch 6.1)
- [ ] I can explain why typed wrappers go through the CLI subprocess instead of calling `getEvmTokenAllowance` directly. (Ex 2, Ch 6.4)
- [ ] I can trace a `cliCommands: [liveDataCommand(account)]` line down five layers and back. (Ex 3, Ch 6.4)
- [ ] I know which helpers stay out of CI and why (human-in-the-loop = local-only). (Ex 4, Ch 6.3)
- [ ] I have personally observed the failure mode of a leaked `SPECULOS_API_PORT`. (Ex 5, Ch 6.5)
- [ ] I have audited every `setDisableTransactionBroadcastEnv` call site for `try/finally` discipline. (Ex 6, Ch 6.6)
- [ ] I can sketch a mobile parity method for `revokeTokenApproval` without re-reading the desktop POM. (Ex 7, Ch 6.8)
- [ ] I know which CLI is canonical for QAA work and when (rarely) `wallet-cli` is justified. (Ch 6.1, Ch 6.7)
- [ ] I can recognize the `@step` copy-paste bug in QAA-615 and explain why it's an observability bug, not a correctness bug. (Ch 6.8)
- [ ] I can explain in one sentence why `0x095ea7b3` is the same selector for both an approve and a revoke. (Ch 6.2)
- [ ] I know the difference between the Etherscan Read tab (free, no wallet) and the Etherscan Write tab (gas, MetaMask) and when each is the right manual tool. (Ch 6.3)
- [ ] I can explain why broadcast-enabled regression tests need a revoke hook between iterations even when the test itself doesn't care about allowance state. (Ch 6.2, Ch 6.6)

### 6.9.9 Diagnostic playbook for common CLI failures

When an exercise (or a real test) fails, work through this short playbook before reaching for help. Most failures map to one of five symptoms.

| Symptom | First check | Likely cause | Reference |
|---|---|---|---|
| `MODULE_NOT_FOUND` from `apps/cli/bin/index.js` | Did the prebuild run? `pnpm --filter @ledgerhq/live-cli prebuild` | Generated commands index missing | Ch 6.1 |
| `ECONNREFUSED` to a Speculos port | `echo $SPECULOS_API_PORT`; is the process still running? | Leaked port from prior cleanup; missing snapshot/restore | Ch 6.5, Ex 5 |
| Test broadcasts a real transaction unexpectedly | `echo $DISABLE_TRANSACTION_BROADCAST` | Earlier `try/finally` was missing; env mutation leaked | Ch 6.6, Ex 6 |
| `parseTokenAllowanceCliOutput` throws on stdout | Did you pass `--format json` to the CLI? | Human-readable output, not JSON | Ch 6.4, Ex 1 |
| `getAddress` returns the wrong address | Which Speculos port is the env var pointing at? | Speculos for the wrong app (ETH vs BTC) | Ch 6.5, Ex 5 |

If the symptom isn't on this list, the next read is the `runCliCommand` error block in `runCli.ts` — the `❌ Failed to execute CLI command` template includes currency, index, exit code, and CLI stderr. Nine times out of ten the answer is in those four lines.

**Worked example.** Imagine a flaky test that broadcasts a transaction every third run. First check: does any `setDisableTransactionBroadcastEnv("0")` call lack a `finally`-wrapped restore? Run the audit from Exercise 6. If clean, second check: does any helper in the test setup call `setDisableTransactionBroadcastEnv` and exit on a path that bypasses the restore (e.g., an early `return`)? An early-return after the flip is almost the same bug as a missing `finally`, just spelled differently. The remedy is the same: pair every flip with a guaranteed restore, and prefer `try/finally` to manual restoration even on the happy path.

### 6.9.10 Where to go from here

If you've checked every box and finished all seven exercises, the natural next steps are:

- **Pair with the senior who wrote QAA-615** (briefing §1) on the final-shape PR. The dev-loop commit is on `support/qaa-615-add-revoke-token`; the final PR will likely keep both `revokeTokenApproval` (as `beforeEach`) and `ensureTokenApproval` (as per-test setup). Volunteering to review that PR is the cleanest way to confirm your model matches reality.
- **Pick up a follow-up ticket** that touches `cliCommandsUtils.ts`. Anything in the QAA backlog tagged "CLI" or "test-data hooks" is fair game. You're now able to read those tickets without translation.
- **Move to Part 7** (Swap Live App). The same allowance state, Speculos transport, and broadcast discipline you just learned reappear there, wrapped in a Live App iframe. Most of what you learned in Part 6 transfers; the new content is the postMessage protocol and the Live App sandboxing model.
- **Volunteer for the QAA-615 mobile parity work** sketched in Exercise 7. The desktop ships first; the mobile follow-up is concrete, scoped, and a perfect first cross-platform contribution.
- **Read one CLI command source file end to end** (e.g., `apps/cli/src/commands/blockchain/send.ts`). You don't need to memorize it; you need the muscle memory of having traced the rxjs pipe from sign to broadcast at least once.
- **Sit in on a QAA review** of a `swap.page.ts` change. Watching how seniors challenge POM additions (does it duplicate an existing helper? does it leak env state? does the `@step` decorator string match the action?) is faster than reading any style guide.
- **Run the full QAA-615 spec locally** end to end on Ethereum mainnet, with broadcast on, against the QAA test seed. Watch a real revoke land on-chain via Etherscan. Confirm the on-chain allowance dropped to zero. This grounds every abstraction you've learned in a verifiable on-chain effect.
- **Write a small internal post-mortem** on the QAA-615 dev-loop commit, framed as: "what makes this not the final shape, and what would final-shape review look for?" Share it with one peer for feedback. The exercise of articulating the distinction in writing solidifies it more than any number of re-reads.

<div class="chapter-outro">
<strong>Key takeaway.</strong> The exercises ramp from a single CLI invocation to a paper-sketched mobile parity for a method you watched a senior write. Exercises 1-3 are warm-ups: hand-run, paper trace, fixture lifecycle memo. 4 and 5 force you to think about what stays out of CI (debugging helpers) and why disciplined env-restoration matters (port-leak failure modes). 6 is an audit pattern you'll repeat throughout your career — every codebase has env-mutating helpers, every one of them needs `try/finally` discipline, every team has at least one missing it. 7 is the capstone: take the senior's commit and translate it to a sister platform, the way real QA tickets evolve. The self-check checklist is the bar: if you can tick every box, you are past the Part 6 onboarding line.

The diagnostic playbook in 6.9.9 is the artifact you'll reach for most often in your first months — keep it open in a tab. The "Where to go from here" list in 6.9.10 turns the onboarding momentum you've built into concrete next contributions instead of a stalled "now what?" gap. Both belong in your bookmarks.
</div>

---


---

## CLI Commands Cookbook

<a id="cli-commands-cookbook"></a>

<div class="chapter-intro">
This chapter is the copy-paste reference. The earlier chapters explained <em>why</em> the four QAA-canonical commands exist and how the five-layer wrappers consume them. This one collects the raw <code>pnpm run:cli ...</code> invocations you will reach for during a debug session, sorted by the question you're trying to answer. Treat it as a kitchen book, not a tutorial — every entry assumes you've read 6.1-6.6 and know the trade-offs. The recipes were factored out of real on-call sessions: rebuild native deps when the device disappears, escape mock mode when accounts come back as <code>mock:1:...</code>, find the CAL token id you can never remember, drive an approve/revoke against a live Stax over USB, generate a userdata fixture without a device, and so on.
</div>

### 6.10.1 Setup and meta

```bash
# install
pnpm i

# build the CLI once (produces apps/cli/lib/cli.js consumed by the bin)
pnpm build:cli

# rebuild on every save during development
pnpm dev:cli

# run an arbitrary CLI command
pnpm run:cli <command> [...flags]

# rebuild native device deps (node-hid, usb) — required after a Node bump
# or when `deviceInfo` reports the device but every transport call hangs
pnpm build:device-deps
```

The `pnpm run:cli` script just invokes `apps/cli/bin/index.js`, which loads `apps/cli/lib/cli.js`. If you forgot to build, you'll see a stale dispatch — `pnpm build:cli` then re-run.

### 6.10.2 Environment hygiene (read this before anything else fails)

```bash
# fail-loud audit before any device-touching command
env | grep -E '^(MOCK|DEVICE_PROXY_URL|SPECULOS_|CI)='

# the canonical "I want a real device" reset
unset MOCK DEVICE_PROXY_URL SPECULOS_API_PORT SPECULOS_APDU_PORT
```

Why this comes first: `MOCK` is a *string* env. `getEnv("MOCK")` returns `"0"` for `MOCK=0`, and `if ("0")` is truthy in JavaScript, so the bridge silently swaps in the mock implementation. The tell is account ids of the form `mock:1:<family>:MOCK_<family>_0` and a "successful" send with no device prompt. The env var declaration in `libs/env/src/env.ts:630-633` even warns: *"Avoid falsy values."*

| Env var | Effect |
|---|---|
| `MOCK` | any non-empty value enables mock bridges; **leave unset** for real-device runs |
| `DEVICE_PROXY_URL` | route HID through a remote proxy (`createTransportHttp`) — overrides USB |
| `SPECULOS_API_PORT` | use Speculos HTTP transport; **unregisters HID** so a real device is invisible |
| `SPECULOS_APDU_PORT` (+ `SPECULOS_BUTTON_PORT`, `SPECULOS_HOST`) | same for the APDU/TCP variant |
| `DISABLE_TRANSACTION_BROADCAST` | sign-only mode (no RPC push); see 6.6 for the discipline |
| `CI` | suppresses HID auto-registration in the setup file |

### 6.10.3 Device introspection

```bash
# device + firmware identity (must be on dashboard, not in an app)
pnpm run:cli deviceInfo

# list installed apps on the device
pnpm run:cli listApps

# list apps available for Speculos (requires COINAPPS env var)
pnpm run:cli speculosList

# read & verify the address on-device for a given currency
pnpm run:cli getAddress --currency ethereum --path "44'/60'/0'/0/0" --verify

# print the derivation schemes for *every* supported currency (no flags)
pnpm run:cli derivation
```

`DeviceOnDashboardExpected` from `deviceInfo` is not a failure — it just means an app is already open. That's exactly the state `send` and `signMessage` want; switch to dashboard only for firmware/app management commands.

### 6.10.4 Account discovery and the `--id` format

```bash
# scan with the device (slow) — returns one real account
pnpm run:cli sync --currency ethereum --index 0 --length 1

# from a saved Live userdata file (no device required)
pnpm run:cli sync --appjsonFile ~/Library/Application\ Support/Ledger\ Live/app.json \
  --currency ethereum --index 0 --length 1

# paginate operations to keep the output manageable
pnpm run:cli sync --currency ethereum --index 0 --paginateOperations 20
```

The id printed at the end of every account block is what subsequent commands consume:

```
js:2:ethereum:0xE32ad14b89F334dF1CD1036c2a0E39A19248b75a:
└─ <type>:<version>:<currencyId>:<xpubOrAddress>:<derivationMode>
```

The trailing `:` is the empty default derivation mode and is significant — keep it. Single-quote the id when passing it on the shell because of the colons.

### 6.10.5 Send native and ERC-20

```bash
# native ETH send
pnpm run:cli send \
  --id 'js:2:ethereum:0xE32ad14b89F334dF1CD1036c2a0E39A19248b75a:' \
  --recipient 0xRECIPIENT \
  --amount 0.001 \
  --wait-confirmation

# ERC-20 transfer (uses the token sub-account; --token resolves it via CAL)
pnpm run:cli send \
  --id 'js:2:ethereum:0xE32ad14b89F334dF1CD1036c2a0E39A19248b75a:' \
  --token ethereum/erc20/usd__coin \
  --recipient 0xRECIPIENT \
  --amount 1.5

# send-to-self for sanity checks (no recipient guessing)
pnpm run:cli send --id '...' --self-transaction --amount 0.0001

# send max balance
pnpm run:cli send --id '...' --recipient 0xRECIPIENT --use-all-amount
```

### 6.10.6 Approvals — revoke, approve, allowance

```bash
# revoke (set allowance to 0)
pnpm run:cli send \
  --id 'js:2:ethereum:0xE32...8b75a:' \
  --mode revokeApproval \
  --token ethereum/erc20/usd_tether__erc20_ \
  --spender 0x40aA958dd87FC8305b97f2BA922CDdCa374bcD7f \
  --wait-confirmation

# approve a finite amount in token units (USDC has 6 decimals → 10 = 10 * 10^6)
pnpm run:cli send \
  --id '...' \
  --mode approve \
  --token ethereum/erc20/usd__coin \
  --spender 0xSPENDER \
  --approveAmount 10 \
  --wait-confirmation

# approve the unlimited (2^256 - 1) allowance
pnpm run:cli send --id '...' --mode approve \
  --token ethereum/erc20/usd__coin \
  --spender 0xSPENDER \
  --approveAmount unlimited

# read the current allowance — no device, no signing, no broadcast
pnpm run:cli tokenAllowance \
  --currency ethereum \
  --ownerAddress 0xE32...8b75a \
  --token ethereum/erc20/usd__coin \
  --spender 0xSPENDER \
  --format json
```

**Argument order trap.** `--spender` is always an `0x…` address, `--approveAmount` is always a number (or `unlimited`). Swapping them produces `invalid address (argument="_spender", value="10", ...)` because ethers tries to encode the number as an address. Pin them in your muscle memory: spender → who, amount → how much.

**USDT quirk.** USDT's contract reverts if you go from a non-zero allowance directly to a different non-zero allowance. Always revoke first if there's an existing allowance, then approve. Other ERC-20s don't share this behaviour.

### 6.10.7 CAL token id lookup

The slug under `<chain>/<standard>/<slug>` is generated, not free-form. Don't guess — query the CAL service:

```bash
# by ticker on a specific chain
curl -s 'https://crypto-assets-service.api.ledger.com/v1/tokens?network=ethereum&ticker=USDC&output=id,name,ticker,contract_address&limit=5' | jq

# by contract address (most precise)
curl -s 'https://crypto-assets-service.api.ledger.com/v1/tokens?network=ethereum&contract_address=0xdAC17F958D2ee523a2206206994597C13D831ec7&output=id,ticker&limit=1' | jq

# cross-chain by ticker (find every USDC across L2s)
curl -s 'https://crypto-assets-service.api.ledger.com/v1/tokens?network_family=evm&ticker=USDC&output=id,ticker,contract_address' | jq

# verify a candidate id before running send
TOKEN_ID="ethereum/erc20/usd__coin"
curl -s "https://crypto-assets-service.api.ledger.com/v1/tokens?id=${TOKEN_ID}&output=id" | jq
# [] → wrong id → CLI will error with `Token <…> not found`
```

Two converter quirks to remember (both handled client-side in `libs/ledgerjs/packages/cryptoassets/src/api-token-converter.ts`):

- MultiversX: pass `multiversx/esdt/*`; the API gets `elrond/esdt/*`.
- Stellar: `stellar/asset/<ADDR>` is lowercased before the API call — case is forgiving here.

### 6.10.8 Broadcast control and confirmations

```bash
# sign only — print the signed payload, do not push to the chain
pnpm run:cli send --id '...' --recipient ... --amount ... --disable-broadcast

# same but driven by env (test-fixture pattern)
DISABLE_TRANSACTION_BROADCAST=1 pnpm run:cli send --id '...' --recipient ... --amount ...

# wait for on-chain confirmation (EVM only; default timeout 120s)
pnpm run:cli send --id '...' --recipient ... --amount ... \
  --wait-confirmation --wait-confirmation-timeout 60000

# push a pre-signed operation JSON (the same shape `send --disable-broadcast` prints)
pnpm run:cli broadcast --id '<accountId>' --signed-operation /tmp/signed.json
# the file must contain an object with `operation` and `signature` keys; pass `-` for stdin
```

The "ignore-errors" flag keeps a multi-tx pipeline going past a single failure:

```bash
pnpm run:cli send --id '...' --recipient 0xA --amount 0.01 \
  --recipient 0xB --amount 0.02 --ignore-errors --format json
```

### 6.10.9 Output shaping

```bash
# machine-parseable, one JSON object per emitted event
pnpm run:cli send ... --format json

# silence the human log entirely (useful when the wrapper parses stderr)
pnpm run:cli send ... --format silent
```

The only `--format` consumers in `apps/cli/src/commands/blockchain/` are `send` and `tokenAllowance`. Other commands print their own output shape — don't pass `--format` to them.

### 6.10.10 Speculos and proxy transports

```bash
# Speculos HTTP API (DMK transport) — the QAA default in CI
SPECULOS_API_PORT=5001 pnpm run:cli send --id '...' --recipient ... --amount ...

# Speculos APDU/TCP variant (lower-level, used by some legacy setups)
SPECULOS_APDU_PORT=9999 SPECULOS_BUTTON_PORT=9998 SPECULOS_HOST=127.0.0.1 \
  pnpm run:cli getAddress --currency ethereum

# Remote real-device proxy (rare, useful for cross-machine debugging)
DEVICE_PROXY_URL=ws://host:port pnpm run:cli deviceInfo

# Force-remove every Speculos Docker container at the end of a manual session
pnpm run:cli cleanSpeculos
```

`SPECULOS_API_PORT` *unregisters* the HID transport (see `apps/cli/src/live-common-setup.ts:131`). If a Speculos var is set, your physical Stax becomes invisible until you `unset` it.

### 6.10.11 Live data fixtures (no device needed)

```bash
# generate a userdata snapshot for one currency and dump JSON to stdout
pnpm run:cli liveData --currency ethereum --add > /tmp/userdata.json

# update an existing app.json in-place (matches a running Live install)
pnpm run:cli liveData --appjson ~/Library/Application\ Support/Ledger\ Live/app.json \
  --currency ethereum --add

# repeat the command per currency to build a multi-chain fixture
pnpm run:cli liveData --appjson /tmp/app.json --currency ethereum --add
pnpm run:cli liveData --appjson /tmp/app.json --currency bitcoin --add
pnpm run:cli liveData --appjson /tmp/app.json --currency polygon --add
```

`--currency` is single-valued in `scanCommonOpts`; running the command once per chain against a shared `--appjson` file is the canonical fan-out pattern.

`liveData` is the workhorse referenced in 6.1.6 — no Speculos, no RPC, no signing. Run it freely.

### 6.10.12 Quick debug recipes

```bash
# is my account scanning correctly? show me a balance and a few ops
pnpm run:cli sync --currency ethereum --index 0 --paginateOperations 10

# what's the on-chain allowance right now?
pnpm run:cli tokenAllowance --currency ethereum \
  --ownerAddress 0xE32...8b75a \
  --token ethereum/erc20/usd__coin \
  --spender 0xSPENDER --format json

# estimate the max spendable on the main account and every token sub-account
pnpm run:cli estimateMaxSpendable --id '...'

# device addresses for a non-default path
pnpm run:cli getAddress --currency ethereum --path "44'/60'/1'/0/0" --verify

# sign an arbitrary message (EIP-191)
pnpm run:cli signMessage --currency ethereum --path "44'/60'/0'/0/0" --message "hello"
```

### 6.10.13 The "nothing happens on Stax" checklist

When `send` prints `✔️ broadcasted!` but the device never woke up, run through this in order:

1. `env | grep -E '^(MOCK|DEVICE_PROXY_URL|SPECULOS_)='` — must be empty.
2. Account id starts with `js:` (or `bitcoin:` / `xrp:` / etc.), **not** `mock:`.
3. `lsof | grep -i ledger` — only one process should hold the HID handle. Close Live Desktop, the Manager web UI, ledgerctl.
4. The Stax is unlocked and the **correct currency app** is open (Ethereum app for an Ethereum send, not Bitcoin).
5. `pnpm build:device-deps` if `node-hid` was rebuilt for a different Node version recently.
6. As a last resort: replug the cable (Stax over USB occasionally needs a fresh enumeration after macOS sleep).

### 6.10.14 What is *not* in this cookbook

`bot.ts`, `botPortfolio.ts`, `botTransfer.ts` — out of QAA scope (different team owns the stress harness). `confirmOp`, `getTransactionStatus`, `testDetectOpCollision`, `testGetTrustedInputFromTxHash` — useful only inside the bot or in coin-specific debugging; reach for `--wait-confirmation` instead. `generateTestScanAccounts`, `generateTestTransaction` — fixture generators superseded by `liveData` for QAA flows. If you find yourself typing one of these by hand, double-check that the answer isn't "use `liveData` + the four canonical commands".

---

## Part 6 Final Assessment

You have walked the full QAA CLI stack: the `apps/cli` binary that is the canonical CLI for QA work, the blockchain primer that makes ERC-20 allowances less mysterious, the manual revoke tools (revoke.cash, Etherscan) that fill the gaps when the harness isn't available, the five-layer integration that turns one Bash command into a typed POM method that drives Speculos in parallel, the Speculos lifecycle and snapshot/restore pattern that keeps two-app tests honest, the broadcast-env discipline with its non-negotiable `try/finally`, the daily workflow with its `wallet-cli` escalation rules, and one real ticket (QAA-615) walked line by line. This assessment samples the load-bearing ideas across Chapters 6.1-6.8. Eighty percent to pass. If you miss a question, jump back to the referenced chapter — the answer is always grounded in a concrete file, command, or commit, not general knowledge.

The questions are weighted toward the patterns you'll repeat in daily work: which CLI to pick (Q1), how on-chain state behaves (Q2), when to reach for manual tools (Q3), the architecture that ties layers together (Q4-Q5), the discipline that keeps tests deterministic (Q6-Q7), the workflow rules that prevent hour-long detours (Q8), and the close-reading skills that catch dev-loop artifacts before they ship (Q9-Q10). If your weak spots cluster around one chapter, that's your re-read; if they're scattered, do a slow pass of the whole part with the briefing open in a second pane.

**Quick reference card (consult after the quiz, not during).**

| Concept | Anchor |
|---|---|
| Canonical QAA CLI | `apps/cli/bin/index.js` (legacy `@ledgerhq/live-cli`) |
| Approve/revoke selector | `0x095ea7b3` = `keccak256("approve(address,uint256)")[0:4]` |
| Layer 2 spawn engine | `libs/ledger-live-common/src/e2e/runCli.ts` |
| Layer 3 typed wrappers | `libs/ledger-live-common/src/e2e/cliCommandsUtils.ts` |
| Layer 4 device-tap | `libs/ledger-live-common/src/e2e/families/evm.ts` |
| Layer 5 desktop POM | `e2e/desktop/tests/page/swap.page.ts` |
| Speculos lifecycle utils | `e2e/desktop/tests/utils/speculosUtils.ts` |
| Broadcast env helper | `setDisableTransactionBroadcastEnv` in `cliCommandsUtils.ts` |

<a id="part-5-final-assessment"></a>

<div class="quiz-container" data-pass-threshold="80">
<h3>Part 6 Final Assessment</h3>
<p class="quiz-subtitle">10 questions · 80% to pass · Covers Chapters 6.1-6.8</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> Which CLI is the canonical tool for QAA test data hooks?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>libs/wallet-cli</code> (the Bun + Bunli + DMK CLI)</button>
<button class="quiz-choice" data-value="B">B) <code>apps/cli</code> (the legacy <code>@ledgerhq/live-cli</code>, invoked as <code>node apps/cli/bin/index.js</code>)</button>
<button class="quiz-choice" data-value="C">C) The Speculos REST API directly, with no CLI in front</button>
<button class="quiz-choice" data-value="D">D) <code>ledger-cli</code> from the published npm package</button>
</div>
<p class="quiz-explanation"><code>apps/cli</code> is what every spec under <code>e2e/desktop/tests/specs/</code> and every <code>cliCommands: [...]</code> fixture call resolves to via <code>runCliCommand</code>. The path resolution is hard-coded in <code>libs/ledger-live-common/src/e2e/runCli.ts</code>: <code>LEDGER_LIVE_CLI_BIN = path.resolve(__dirname, "../../../../apps/cli/bin/index.js")</code>. <code>wallet-cli</code> is a separate workspace, used for non-QAA work; treating it as the QAA tool is the most common new-joiner mistake and produces a confusing 30-minute detour every time.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> What does the calldata <code>0x095ea7b3</code> represent on an ERC-20 token contract?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The contract's bytecode prelude</button>
<button class="quiz-choice" data-value="B">B) The function selector for <code>transfer(address,uint256)</code></button>
<button class="quiz-choice" data-value="C">C) The function selector for <code>approve(address,uint256)</code> — i.e. <code>keccak256("approve(address,uint256)")[0:4]</code> — followed in calldata by the spender address (32 bytes) and the allowance amount (32 bytes)</button>
<button class="quiz-choice" data-value="D">D) The selector for <code>balanceOf(address)</code></button>
</div>
<p class="quiz-explanation"><code>0x095ea7b3</code> is the four-byte function selector for <code>approve</code>. Both an "approve N" and a "revoke" share the same selector — the difference is whether the trailing 32-byte amount is non-zero (approve) or zero (revoke). Ledger's clear-sign decoder recognizes this selector and renders human-readable approve/revoke screens. The infinite-approval footgun (calldata <code>...</code> with amount <code>2^256 - 1</code>) was the vector in major DeFi drains (BadgerDAO 2021, multiple 2022 phishing campaigns) — which is why revoke.cash and EIP-2612 permit exist.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> When is revoke.cash the right manual tool, and when is Etherscan's Write tab the right tool?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) revoke.cash when you need a one-click UI to enumerate every active allowance on an address and revoke any of them; Etherscan's Write tab when revoke.cash doesn't list a specific token (rare edge cases) and you need to call <code>approve(spender, 0)</code> directly via Connect+MetaMask</button>
<button class="quiz-choice" data-value="B">B) revoke.cash for mainnet, Etherscan only for testnets</button>
<button class="quiz-choice" data-value="C">C) They are interchangeable; pick whichever loads faster</button>
<button class="quiz-choice" data-value="D">D) Etherscan first, revoke.cash only as a fallback when Etherscan is down</button>
</div>
<p class="quiz-explanation">revoke.cash's value is its enumeration UI — it scans the address and surfaces every active allowance per token+spender. Etherscan's Write tab is a generic contract-write surface for the rare case revoke.cash doesn't index a token (obscure tokens, fresh deployments). Both connect via the same WalletConnect/MetaMask flow; both broadcast a real transaction and cost gas.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> In the five-layer integration, which layer does device-tap automation (driving Speculos's REST API to walk through the on-device approval screens)?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Layer 1 — <code>apps/cli/bin/index.js</code></button>
<button class="quiz-choice" data-value="B">B) Layer 2 — <code>libs/ledger-live-common/src/e2e/runCli.ts</code></button>
<button class="quiz-choice" data-value="C">C) Layer 3 — <code>libs/ledger-live-common/src/e2e/cliCommandsUtils.ts</code></button>
<button class="quiz-choice" data-value="D">D) Layer 4 — <code>libs/ledger-live-common/src/e2e/families/evm.ts</code> (with <code>approveToken()</code>, <code>approveTokenButtonDevice</code>, <code>approveTokenTouchDevices</code>)</button>
</div>
<p class="quiz-explanation">Layer 4 is the device-tap layer. While the CLI subprocess (Layer 1, spawned by Layer 2) is blocked waiting for a hardware confirmation, <code>approveToken()</code> in <code>families/evm.ts</code> drives Speculos's REST API to navigate menus, press buttons, hold-to-sign on touch devices. The CLI and the device-tap helper run in parallel — that's the trick. <code>approveToken()</code> branches on <code>isTouchDevice()</code>: button devices (Nano S/X/S+) get <code>pressUntilTextFound</code> + <code>buttonsController.both()</code>; touch devices (Stax/Flex) get <code>longPressAndRelease(HOLD_TO_SIGN, 3)</code>. Same logical labels via <code>DeviceLabels.*</code>, different physical interactions.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> Why are several Layer-3 helpers (<code>liveDataCommand</code>, <code>liveDataWithAddressCommand</code>, <code>liveDataWithRecipientAddressCommand</code>) curried — i.e. <code>(account) =&gt; (userdataPath) =&gt; Promise&lt;void&gt;</code> — instead of plain async functions?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) For functional-programming style points</button>
<button class="quiz-choice" data-value="B">B) To support partial application across spec files</button>
<button class="quiz-choice" data-value="C">C) Because the desktop fixture engine (in <code>e2e/desktop/tests/fixtures/common.ts</code>) consumes a <code>cliCommands: [...]</code> array and threads in the userdataPath at the right point in the test lifecycle. Currying lets specs declare <code>cliCommands: [liveDataCommand(account)]</code> at metadata-time and have the engine call <code>(curried)(userdataPath)</code> at setup-time</button>
<button class="quiz-choice" data-value="D">D) To avoid TypeScript inference issues</button>
</div>
<p class="quiz-explanation">The curry is fixture-shaped: the spec author binds <code>account</code> when writing the test; the engine binds <code>userdataPath</code> when setting up. One-shot reads (like <code>isTokenAllowanceSufficientCommand</code>) don't need this pattern — they're plain async functions because there's no second-stage binding to thread.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q6.</strong> Why does a swap test that needs to revoke a token approval before the test runs save the previous Speculos port and restore it in a <code>finally</code> block?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) For Allure attachment hygiene</button>
<button class="quiz-choice" data-value="B">B) Because two-app tests run a second Speculos (e.g., the Ethereum app for the revoke) on a different port, which mutates <code>SPECULOS_API_PORT</code>. Without saving and restoring, the rest of the test run — including the main Exchange-app Speculos that drove the swap — would point at a now-stopped port, causing connection-refused failures on every subsequent CLI call</button>
<button class="quiz-choice" data-value="C">C) Because Speculos randomizes its port and the previous port is unreachable after restart</button>
<button class="quiz-choice" data-value="D">D) Because Allure requires the original port to render device screenshots</button>
</div>
<p class="quiz-explanation">The <code>SPECULOS_API_PORT</code> env var is process-global. Mutating it for a sub-step (revoke on Ethereum app) and forgetting to restore leaks the change to every later CLI invocation. The pattern in <code>cleanSpeculos(speculos, previousPort?)</code> exists exactly to make this restoration idiomatic.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q7.</strong> What does the <code>try/finally</code> block in <code>approveTokenCommand</code> and <code>revokeTokenCommand</code> guard against?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) An unhandled throw between the env flip (<code>setDisableTransactionBroadcastEnv("0")</code>) and the env restore would leak the broadcast-on state to every subsequent CLI call in the run, silently flipping broadcast for tests that expected it to stay off. The <code>finally</code> ensures restoration runs even on throw</button>
<button class="quiz-choice" data-value="B">B) Speculos crashes — the <code>finally</code> kills the simulator</button>
<button class="quiz-choice" data-value="C">C) The CLI subprocess hanging — the <code>finally</code> sends SIGKILL</button>
<button class="quiz-choice" data-value="D">D) Allure attachment failures — the <code>finally</code> writes a fallback record</button>
</div>
<p class="quiz-explanation">Env mutations are process-global and persist across awaits. A throw between flip and restore leaves <code>DISABLE_TRANSACTION_BROADCAST</code> set to <code>"0"</code> for the rest of the run; later tests assuming broadcast-off would silently broadcast real transactions. <code>finally</code> turns "we restore on the happy path" into "we always restore" — non-negotiable. The same pattern applies to any other env-mutating helper: capture the original value, pair the mutation with a guaranteed restore, never trust an explicit happy-path restore alone.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q8.</strong> When does the daily QAA workflow legitimately escalate to <code>wallet-cli</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Whenever <code>apps/cli</code> is slow</button>
<button class="quiz-choice" data-value="B">B) For every test that involves a swap</button>
<button class="quiz-choice" data-value="C">C) Whenever you need to read an allowance — <code>wallet-cli</code> is faster</button>
<button class="quiz-choice" data-value="D">D) Almost never. <code>wallet-cli</code> is out of QAA scope; the canonical CLI for QAA test hooks is <code>apps/cli</code>. Escalation to <code>wallet-cli</code> is reserved for rare cross-team work where a wallet-cli-specific feature (e.g., DMK transport pinning, certain BLE flows) is genuinely required and a senior has signed off</p></button>
</div>
<p class="quiz-explanation">The new-joiner trap is treating <code>wallet-cli</code> as a general-purpose CLI just because it's newer. For QAA work, <code>apps/cli</code> is the answer in 99% of cases. Reaching for <code>wallet-cli</code> usually signals you've taken a wrong turn — pause, ask, then proceed only with sign-off.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q9.</strong> The senior's QAA-615 commit has <code>@step("Ensure token approval")</code> on the <code>revokeTokenApproval</code> method. What's the issue?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>@step</code> should be <code>@Step</code> with a capital S</button>
<button class="quiz-choice" data-value="B">B) The decorator string is a copy-paste from <code>ensureTokenApproval</code> — it lies about what the method does. The Allure report will label every revoke step as "Ensure token approval", which is actively misleading. Final PR will fix this to "Revoke token approval"</button>
<button class="quiz-choice" data-value="C">C) <code>@step</code> is deprecated; use <code>@TestStep</code> instead</button>
<button class="quiz-choice" data-value="D">D) Decorators don't work on async methods in this version of TypeScript</button>
</div>
<p class="quiz-explanation">The bug is observability, not correctness. The method runs fine; the Allure step name is wrong. This is a classic dev-loop artifact: the senior copied the existing method's shape including its decorator, and forgot to update the string. Final PR review catches it.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q10.</strong> The QAA-615 commit comments out <code>ensureTokenApproval</code> and replaces it with <code>revokeTokenApproval</code>. The Part 6 briefing notes this is "dev-loop code, not the final shape." What is the final shape likely to be?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The final shape removes <code>ensureTokenApproval</code> entirely and only revokes</button>
<button class="quiz-choice" data-value="B">B) The final shape collapses both methods into a single <code>handleTokenApproval</code> that takes a mode flag</button>
<button class="quiz-choice" data-value="C">C) The final shape keeps both: <code>revokeTokenApproval</code> as a <code>beforeEach</code> cleanup that resets allowance to zero between iterations, and <code>ensureTokenApproval</code> as the per-test setup that approves the slippage-padded amount the test actually needs. The dev-loop commit only flipped them temporarily because the senior was iterating on the revoke path</button>
<button class="quiz-choice" data-value="D">D) The final shape calls neither — the swap provider auto-approves at swap time</button>
</div>
<p class="quiz-explanation">Both methods are needed for a deterministic broadcast-enabled regression test: revoke first to clear stale allowance from the previous run, then approve the right amount for this run. The commented-out <code>ensureTokenApproval</code> in the dev-loop commit is just the senior temporarily disabling the second half while iterating on the first half. Final PR will keep both. Recognizing this distinction — between code that helps a human iterate and code that is ready to ship — is one of the highest-leverage skills in code review; the <code>// await app.swap.ensureTokenApproval(...)</code> line is the smoking gun.</p>
</div>

<div class="quiz-score"></div>
</div>

<div class="chapter-outro">
<strong>Part 6 complete.</strong> You own the QAA CLI stack end to end:

- <code>apps/cli</code> as the canonical binary, invoked from tests as <code>node apps/cli/bin/index.js</code> via <code>child_process.spawn</code>. The Bun-based <code>wallet-cli</code> stays out of QAA scope.
- ERC-20 approvals as on-chain state that survives reboots, simulator restarts, and repo re-clones. The <code>0x095ea7b3</code> selector renders human-readable on Ledger devices because the clear-sign decoder recognizes it.
- revoke.cash and Etherscan as the manual escape hatches when the harness can't help — debugging local seeds, auditing what's actually approved, recovering from a broken Speculos session.
- The five-layer integration: <code>apps/cli/bin/index.js</code> → <code>runCli.ts</code> → <code>cliCommandsUtils.ts</code> → <code>families/evm.ts</code> (device-tap parallel-driver) → <code>swap.page.ts</code> POM. Each layer has one job; the curry pattern at Layer 3 is what makes the <code>cliCommands: [...]</code> fixture syntax possible.
- The Speculos snapshot/restore pattern. Two-app tests save the previous port and restore it in <code>finally</code>; failure to do so leaks a now-stopped port to every subsequent CLI call.
- The <code>DISABLE_TRANSACTION_BROADCAST</code> env discipline. Flip to <code>"0"</code> with <code>setDisableTransactionBroadcastEnv</code>, restore <code>original</code> in <code>finally</code>. No exceptions; an unhandled throw between flip and restore silently flips broadcast for the rest of the run.
- The daily workflow that prefers small typed wrappers over hand-run CLI sessions, with rare <code>wallet-cli</code> escalation only on senior sign-off.
- One real ticket (QAA-615) walked line by line — three files, 37 lines, the <code>@step</code> copy-paste bug, and the dev-loop-vs-final-shape distinction that separates "code that helped a senior iterate" from "code that ships".

<strong>Next: Part 7</strong> climbs back up the stack to the Swap Live App — the embedded UI that exercises everything you just learned. Same allowance state, same Speculos transport, same broadcast discipline — but now wrapped in a Live App iframe with a postMessage protocol of its own. The work you put into understanding why <code>revokeTokenCommand</code> exists is exactly what makes the Swap Live App's flows make sense: when the iframe asks the host to sign an approve transaction, you'll know which CLI command would emit the same calldata, which Speculos screen would appear, and which env flag would have prevented broadcast in a test. Carry that mental model forward.
</div>
