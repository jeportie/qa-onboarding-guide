# PART 5 — CLI AUTOMATION

You finished Part 4 standing on a Detox simulator with a WebSocket bridge between your test process and a React Native app. Part 5 walks off that stack altogether. There is no UI to drive, no Speculos container to spawn, no bridge to inject state. There is a real Ledger over USB, a Bun process, and a command line. The QA value is the same: deterministic test data on demand.

---

## CLI Automation -- Architecture Deep Dive

<div class="chapter-intro">
The mobile suite gave you a third surface — a React Native app, native builds, and a bridge to inject state. The CLI is your fourth surface and the cleanest. There is no app, no UI, no bridge. The Ledger device <em>is</em> the UI: the screen and two buttons (or the touchscreen) are the only confirmation surface, and a Bun process talks to that device over USB through the Device Management Kit. This chapter places <code>wallet-cli</code> in the monorepo, contrasts it with the legacy <code>apps/cli</code>, explains why a CLI part exists in this guide at all, and warns you about the footguns before you run a single command.
</div>

### 5.1.1 What you'll learn

You'll learn that wallet-cli is the third E2E flavour after Desktop and Mobile, but inverted: the device replaces the application, the application is missing entirely, and the same DMK + USB transport that ships inside Ledger Live ships here too. You'll learn where it sits in the monorepo, why it exists, and which of its sharp edges to avoid.

### 5.1.2 Why a CLI in QA at all

Your first instinct after Part 4 is probably "I already have Detox and Playwright; why would I want a CLI?" The answer is that the CLI is **test-data infrastructure**, not a third E2E framework competing with the first two. It does the things Playwright and Detox do badly.

The triggering ticket — **QAA-615** "Spike: Revoke token approval using CLI" — is the canonical example. The QAA team is automating the swap regression suite. To swap an ERC-20 token through a DEX (Thorchain, Uniswap, LiFi), the user has to sign two transactions on the device:

1. `approve(spender = DEX_router, amount = N)` — gives the DEX contract permission to withdraw `N` tokens from the user's address.
2. The actual swap transaction.

Without step 1, step 2 fails with `ERC20: insufficient allowance`. Step 1 is **persistent on-chain state**: once the user has approved the DEX router, the allowance is recorded inside the ERC-20 token contract under the user's address. It does not reset when the test ends. It does not reset when you reboot the simulator. It sits on-chain until somebody explicitly sets it back to zero.

That makes the test non-idempotent. The first run approves and swaps. The second run finds the allowance already set and skips the approval step — which means the test no longer exercises the approval flow. Worse, if the test asserts on the approval flow's UI (button labels, modals, gas estimates), it now fails for the wrong reason.

The fix is what Victor proposed in the Slack thread that opened the spike:

> *"yes en gros c'est pcq l'approve du token c'est a faire que 1x. et apres c'est good. donc faut ajouter un hook (before ou after) le test pour clean justement. et je crois que c'est potentiellement possible de faire ca avec le CLI mais faut investiguer."*

Translated: add a before/after hook that revokes the approval. Revoking is just `approve(spender, amount = 0)` — same APDU shape, zero amount. After that the test starts at allowance = 0 every run.

Why a CLI for that hook and not Playwright?

- **No UI overhead.** A revoke is one transaction on one account. Driving it through Live's UI means: launch the Electron binary, wait for the bridge, navigate to the swap page, click "manage allowances", confirm on Speculos, wait for broadcast, parse the receipt. That's 60+ seconds and ten flaky waits. A CLI command does it in three seconds.
- **No app build.** You can revoke an allowance without building Live Desktop or Live Mobile. That matters for a hook that runs before every regression test on every CI shard.
- **One-shot data ops are not what test runners are for.** Playwright's job is to drive a user-facing UI and assert visible state. Hammering it into a "broadcast a transaction" tool produces tests that are slow, brittle, and impossible to read. The CLI is the right shape for the right job.
- **Local debugging.** When the regression suite fails on a Tuesday, you want to reproduce the broken state in 30 seconds, not stand up the full Live stack. `wallet-cli send --data 0x095ea7b3…` reproduces the exact APDU traffic Live would produce, against a real device.

There's a precedent inside Ledger. The QA team already uses **vault-cli** as test-data infrastructure for the Vault E2E suite (Confluence: VAUL/2811756923 — Vault QA Automation strategy). The pattern is the same: a domain-specific CLI runs as a precondition step, populates state, and lets the user-facing E2E framework focus on user-facing assertions. Part 5 generalises that pattern to wallet flows.

The other tickets sitting in QAA-919's family follow the same pattern. **QAA-613** (the sibling — "Implement the token approval flow with Thorchain, Uniswap, LiFi") is the consumer of QAA-615. **QAA-617** (CI environment droplist) will need a CLI hook to reset CI fixtures. **QAA-722** (use MIN amount from request) will need a CLI helper to fetch the dynamic minimum. Once wallet-cli has revoke, every future swap-regression hook becomes a documented one-liner.

### 5.1.3 Two CLIs in the monorepo: a brief and honest history

Open the monorepo and you will find **two** workspaces under `apps/` whose names contain "cli". Onboarding guides and old wiki pages mix them up freely. They are different tools with different audiences. Knowing which is which is the first non-trivial fact in this part.

**`apps/cli/`** — the legacy CLI. Published on npm as `@ledgerhq/live-cli` (a public package). Built on Node.js (the README still pins Node 14 — `lts/fermium`). Uses the older `@ledgerhq/hw-transport-node-hid` stack and the V0 account-id format (`js:2:bitcoin:xpub…:native_segwit:0`). Wraps `@ledgerhq/live-common` directly. Its README documents **62 commands** including the heavyweights:

- `swap` — perform an arbitrary swap between two currencies on the same seed.
- `bot` / `botPortfolio` / `botTransfer` — Speculos-driven regression engine ("the bot").
- `app` / `appUninstallAll` / `listApps` / `managerListApps` — manager / app installation.
- `firmwareUpdate` / `firmwareRepair` — firmware operations.
- `tokenAllowance` — read on-chain allowances.
- `genuineCheck`, `deviceInfo`, `getBatteryStatus`, `repl` — device introspection.
- `generateTestScanAccounts`, `generateTestTransaction` — fixture generators.
- `signMessage`, `broadcast`, `getTransactionStatus`, `balanceHistory`, `countervalues`, `portfolio` — read/write helpers.

Output is free-form text. A few commands accept `--format json` but the flag's behaviour varies. Errors come out as `console.error(String(e.message))`. Exit code is 0 (success) or 1 (anything else). Confluence has an architecture audit of this surface (TA/6951665759 — "Ledger Live CLI for AI Agent") that catalogues the friction points; it's the cleanest description of why a successor was needed.

**`apps/wallet-cli/`** — the experimental replacement. Private to the monorepo (`"private": true` in `package.json`, name `@ledgerhq/wallet-cli`). Built on Bun and Bunli. Uses the **Device Management Kit (DMK)** over USB. Speaks the V1 account descriptor format (`account:1:address:ethereum:main:0x…:m/44h/60h/0h/0/0`). Has a smaller surface — today, around ten commands across two groups (exact catalog in Ch 5.4) — but every command speaks a unified `--output human|json` envelope. Version sits at **0.1.0** in `bunli.config.ts`. The README opens with:

> "This software is experimental. It is provided 'as is,' without obligation to develop, support, or repair features. Ledger shall not be liable for damages arising from its use."

So the practical state of play in 2026 is that wallet-cli is the right place for new work — but legacy `apps/cli` still ships features wallet-cli has not replicated. Notably:

- **Swap** — wallet-cli has no `swap` command. The legacy CLI has one. Neither has on-chain `tokenAllowance` reads as a first-class command, though wallet-cli can broadcast an arbitrary `--data` calldata and the legacy CLI exposes `tokenAllowance` as a query.
- **Speculos bot** — `bot`/`botPortfolio` are legacy-only. wallet-cli does not currently target Speculos at all (it's USB-only via DMK).
- **App management** — `app install/uninstall`, `listApps`, `genuineCheck`, `firmwareUpdate` — legacy-only.
- **Test fixture generation** — `generateTestScanAccounts`, `generateTestTransaction` — legacy-only.
- **Sign arbitrary messages** — `signMessage` — legacy-only.

Conversely, wallet-cli ships things the legacy CLI does not:

- DMK transport (with the connectivity gains documented in WXP/6995411067 — LedgerJS vs DMK Benchmark, "internal data shows a >10x reduction in connectivity errors after migrating to DMK").
- Bunli command framework with Zod-validated flags.
- Single-binary build via `bunli build` (`darwin-arm64`, `linux-arm64`, `linux-x64`, `windows-x64`).
- V1 versioned account descriptors with shell-safe hardened markers (`h` instead of `'`).
- A unified JSON envelope on every command.
- A persistent session layer (`session view` / `session reset`) — labels at `XDG_STATE_HOME/ledger-wallet-cli/session.yaml`.
- `--dry-run` send mode that skips device + broadcast.
- A real test harness: mock DMK + http-intercept + mock-server.

> **Verify:** the count of legacy commands ("62" / "59" / "70" / "around 70") varies between the architecture study, the briefing, and the legacy README. They're all the same surface, the differences come from how you count subcommands and grouped commands. The exact number doesn't matter for QA work — what matters is that the surface is large, mostly stable, but built on stale runtime tooling.

**Practical decision rule** (the only one that matters in the next twelve months):

- New CLI work targets `apps/wallet-cli/`. Adding `revoke` for QAA-615? wallet-cli. Adding a fixture command for QAA-617? wallet-cli.
- If you need a feature wallet-cli does not yet replicate — Speculos `bot`, `swap`, app management, firmware ops — fall back to legacy `apps/cli` and flag the dependency in your PR description.
- Do **not** mix the two in a single hook. They have different transport stacks (DMK vs hw-transport-node-hid), different descriptor formats (V1 vs V0), and different Node/Bun runtimes. Sequencing them in one shell script works in development and breaks in CI.

A useful mental model: think of the two CLIs as two generations of the same family tree. The legacy `apps/cli` is the parent, with broad surface and the scars of a 2018-era stack. The new `apps/wallet-cli` is the child, with a narrower surface but a modern runtime. They share a grandparent (`@ledgerhq/live-common`) and that's it. When you read either codebase, ask which generation you're in: V0 descriptors and `hw-transport-*` mean legacy; V1 descriptors and DMK mean new. The two never mix in source code, and they shouldn't mix in your hooks either.

The R5 wiki research is also worth saying out loud: the GitHub wiki has **zero pages** about wallet-cli, Bunli, DMK, or V1 descriptors. The only CLI page in the wiki (`LLC/LLC:dev-with-cli.md`) still references the deprecated tool and Node 14. You will not find this material on the wiki. This guide and the in-tree READMEs (`apps/wallet-cli/README.md`, `apps/wallet-cli/src/wallet/README.md`, `apps/wallet-cli/src/test/commands/README.md`, `apps/wallet-cli/src/shared/accountDescriptor/README.md`) are the canonical sources.

### 5.1.4 Where wallet-cli sits in the monorepo

Part 1 introduced you to the monorepo layout: `apps/` for shippable applications, `libs/` for shared libraries, `e2e/` for the test workspaces. Part 3 placed Desktop E2E at `e2e/desktop/`. Part 4 placed Mobile E2E at `e2e/mobile/`. wallet-cli sits in a third location — and this distinction matters.

```
apps/
|-- cli/                             # legacy @ledgerhq/live-cli (Node, hw-transport)
|-- wallet-cli/                      # @ledgerhq/wallet-cli (Bun, DMK, USB)
|   |-- package.json                 # "@ledgerhq/wallet-cli", private, bin: wallet-cli
|   |-- bunli.config.ts              # name + commands directory + build targets
|   |-- bunfig.toml                  # Bun runtime config (test coverage reporter)
|   |-- tsconfig.json                # extends root, types: node + bun-types + w3c-web-usb
|   |-- project.json                 # Nx targets (coverage / generate / build)
|   |-- README.md                    # status + commands + prerequisites
|   |-- CHANGELOG.md                 # changesets
|   |-- .bunli/
|   |   `-- commands.gen.ts          # CODEGEN — auto-registered command tree
|   `-- src/
|       |-- cli.ts                   # entry — Bunli wiring, side-effect imports
|       |-- config.ts                # LiveConfig schema for wallet-cli
|       |-- live-common-setup.ts     # registers coin modules + DMK transport
|       |-- embed-usb-native.ts      # forces Bun --compile to embed usb addon
|       |-- output.ts                # CommandOutput abstraction (human/json)
|       |-- commands/                # Bunli defineCommand modules
|       |-- device/                  # DMK builder + register-transport + mock-DMK
|       |-- session/                 # YAML-persisted account labels
|       |-- shared/                  # accountDescriptor + log + ui + response
|       |-- wallet/                  # WalletAdapter facade (bridge + alpaca)
|       `-- test/                    # bun test suite
|           |-- commands/            # per-command integration tests
|           `-- helpers/             # cli-runner / mock-server / http-intercept
|-- ledger-live-desktop/             # the Electron desktop app
|-- ledger-live-mobile/              # the React Native mobile app
e2e/
|-- desktop/                         # the Playwright workspace (Part 3)
`-- mobile/                          # the Detox workspace (Part 4)
```

Compared with Part 3 and Part 4, the structural difference is:

- **`e2e/desktop/`** is a peer workspace dedicated to testing `apps/ledger-live-desktop/`. The application under test lives in `apps/`; the tests live in `e2e/`.
- **`e2e/mobile/`** is the same arrangement for `apps/ledger-live-mobile/`.
- **`apps/wallet-cli/`** is its own application. It has no peer `e2e/cli/`. Its tests live **inside the application workspace**, at `apps/wallet-cli/src/test/`. There is no separate test workspace because there is no separate test runner — `bun test` executes the same TypeScript that the production binary executes, with mock seams installed via env vars.

This is a design choice, not an oversight. Three reasons:

1. **The CLI is small.** Around 5,860 lines of source. Splitting it into a peer test workspace would add Nx and pnpm boilerplate that buys nothing. Detox and Playwright workspaces exist because the test code is ten times larger than the app code; here it's the inverse.
2. **The mock seams are inside the source tree.** `src/device/mock-dmk.ts` and `src/test/helpers/dmk-intercept.ts` install the mock device by calling `_setTestDmkTransport()` — an internal export from `src/device/register-dmk-transport.ts`. Putting the tests in a peer workspace would force that internal export to leak across a workspace boundary.
3. **Bun's test runner runs from any directory.** `bun test src/` walks the tree. There is no Jest config to maintain at a peer level, no Detox config, no Playwright project file.

So when you read sentences like "wallet-cli's E2E tests" in this part, mentally translate that to "wallet-cli's integration tests under `apps/wallet-cli/src/test/commands/`". They are the equivalent of Part 3's `e2e/desktop/tests/specs/` and Part 4's `e2e/mobile/specs/`, but they live in-place.

### 5.1.5 The transport layer at a glance

The Device Management Kit (DMK) replaces the older `hw-transport-*` stack. You'll get a full primer in Chapter 5.3, but you need three facts now:

1. **DMK is the library between client code and the physical device.** Confluence (CS/7014678571 — Device Management Kit) puts it like this:

   > "The Device Management Kit is the library that sits between clients (for instance the Ledger Wallet) and the physical Ledger signer. In the context of Clear Signing, it is the client-side orchestration layer responsible for fetching ERC-7730 metadata, packaging it alongside the raw transaction, and delivering it to the device over USB or Bluetooth."

2. **wallet-cli wires DMK over USB only.** No Bluetooth (the node-side BLE story is unsettled). No HTTP-Speculos transport. The CLI's `device/dmk.ts` builds DMK with a single transport factory:

   ```ts
   return new DeviceManagementKitBuilder()
     .addTransport(nodeWebUsbTransportFactory)
     .addLogger(new LedgerLiveLogger(LogLevel.Warning))
     .addConfig({ firmwareDistributionSalt })
     .build();
   ```

   That's the entire production transport surface. Mock tests replace it wholesale via `MockDeviceManagementKit`.

3. **DMK contributions you do not pay for in legacy.** The benchmark doc (WXP/6995411067 — LedgerJS vs DMK) reports a **>10x reduction in connectivity errors** after migrating to DMK. The reconnection state machine, the device state introspection, the per-app signer kits (`@ledgerhq/device-signer-kit-ethereum`, `…-solana`, `…-bitcoin`, …) — all live in DMK.

The trade-off is that DMK is bigger surface area to learn than `hw-transport-node-hid`. Chapter 5.3 walks the whole stack.

### 5.1.6 The runtime

Bun, not Node. The package.json declares `"engines": { "bun": ">=1.1.0" }`. Why it matters:

- **Faster startup.** A `pnpm wallet-cli start -- balances --help` finishes in under half a second. Cold Node is over a second on the same hardware.
- **Native TypeScript.** No `tsc` step. No `ts-node`. `bun run ./src/cli.ts` executes TypeScript directly. The codebase still has a `pnpm typecheck` script (`tsc --noEmit`) for type errors that Bun's bundler doesn't catch, but you don't need a build step to run.
- **Native test runner.** `bun test` is built in. Coverage via `bun test --coverage` (lcov, configured in `bunfig.toml`).
- **`bun --compile`** produces a single ~94 MB binary that embeds the Bun runtime and the `usb` native addon. That's the QA-relevant artifact: the binary you can drop into a CI shard without installing Node, pnpm, or any monorepo packages.

Bun is not a casual choice and it isn't a clean replacement for Node either. The team had to patch `@bunli/core` to avoid a dynamic `import()` that hangs in standalone mode. They had to embed the `usb` N-API addon manually because Bun's bundler can't follow `node-gyp-build`'s dynamic `fs.readdirSync`. They call `LiveConfig.setConfig` twice (once via ESM, once via CJS) because Bun resolves the same module to two different singletons depending on the import shape. None of these are Bun bugs in the usual sense; they are the cost of building inside a wider monorepo whose libraries assume Node's resolver semantics. Chapter 5.2 unpacks the implications.

### 5.1.7 Comparison to Part 3 / Part 4 stacks

The mental model you built in Part 3 (Playwright + page objects + Speculos REST) and Part 4 (Detox + thin specs + Speculos via WebSocket bridge) does not transfer to Part 5 cleanly. Here's the side-by-side:

| Aspect | Desktop (Part 3) | Mobile (Part 4) | CLI (Part 5) |
|---|---|---|---|
| **Runner** | Playwright (test framework) | Detox + Jest | Bunli (handler framework, not a test framework) |
| **Test framework** | `playwright/test` | `jest` | `bun test` |
| **Transport** | `hw-transport-node-speculos-http` | DMK over WebSocket bridge | DMK over USB (node-webusb) |
| **Execution location** | `e2e/desktop/` (peer workspace) | `e2e/mobile/` (peer workspace) | `apps/wallet-cli/src/test/` (in-place) |
| **Mock layer** | Speculos REST + HTTP-Transport mocks | bridge server + Detox sync | mock-DMK + http-intercept + mock-server |
| **First-class concept** | Page Object | thin spec + driver | command + intent + bridge adapter |
| **App under test** | Electron (Live Desktop) | React Native (Live Mobile) | None — the device is the UI |
| **Build step before run** | Rspack bundle (~2 min) | Xcode/Gradle native (~10-30 min) | None (`bun run ./src/cli.ts`) |
| **Parallelism** | Full (random ports) | 1-3 workers (device limit) | One device per process (singleton DMK) |
| **State injection** | Playwright fixture | WebSocket bridge | `account discover` writes session.yaml |
| **Fixture format** | userdata JSON | userdata JSON | session.yaml + V1 descriptors |
| **Output** | HTML report + Allure | Allure | JSON envelope on stdout |
| **Pass/fail convention** | Test runner exit code | Test runner exit code | Process exit code (0 ok, 1 error) |

Three things to internalise from this table:

- **Bunli is not a test runner.** Playwright and Detox + Jest are *frameworks for asserting on a running application*. Bunli is a *framework for declaring CLI commands*. You declare a command with `defineCommand({ name, description, options, handler })`, Bunli parses argv, validates flags via Zod, and dispatches your handler. Test assertions in this part are written with `bun test` plus a custom `cli-runner` helper that spawns the actual `cli.ts` and inspects stdout/stderr/exit code. Two different layers, two different concerns.
- **The transport story changes shape, not direction.** All three parts ultimately drive a Ledger device (real or simulated). Desktop reaches it through Speculos's HTTP transport. Mobile reaches it through the bridge that proxies APDUs between the React Native app and the Speculos container. CLI reaches it directly through DMK over USB to a real device — or through a `MockDeviceManagementKit` for tests. Same APDU exchanges underneath.
- **The "no UI" property is the design freedom.** Speculos exists because Detox and Playwright cannot tap on a physical Nano S. The CLI doesn't have that problem because there's nothing to tap on — the device's own screen is the only confirmation surface, the user reads it, the user presses the buttons. That's why the CLI can be USB-only: it doesn't need to virtualise a button press. Speculos is still useful for testing command-side correctness without a physical device, but wallet-cli does not currently wire it. Chapter 5.3 covers why and what the workaround is.

### 5.1.8 What this Part covers and what it doesn't

**Covers:**

- wallet-cli architecture, runtime, codebase walk (this chapter, 5.4, 5.5).
- Bun and Bunli primer, your first command (Chapter 5.2).
- DMK primer — transports, sessions, device actions, the singleton (Chapter 5.3).
- Account descriptors V1 — V0 vs V1 history, parser internals, why hardened path uses `h` not `'` (Chapter 5.6 / will be referenced from here forward).
- Daily workflow: from ticket to PR for a CLI feature (Chapter 5.7).
- Testing patterns: mock DMK + http-intercept + cli-runner (Chapter 5.6).
- The QAA-615 spike walkthrough end-to-end — designing `revoke`, choosing a path, shipping the PoC, integrating into a test hook (Chapter 5.8).
- CLI exercises and a Part 5 final assessment.

**Does not cover:**

- Legacy `apps/cli` deeply. We name what it has that wallet-cli lacks (so you know when to reach for it) but we don't teach its 62 commands. The README at `apps/cli/README.md` is the reference for that surface; the architecture study at TA/6951665759 is the analysis.
- Wallet API and Live Apps. Those are Part 6 (which used to be Part 5 — the renumbering happened to make room for this part). If you're looking for `swap-live-app`, it's there.
- DMK internals — XState machines, secure-channel cryptography, firmware-update protocol. The CLI consumes DMK through its public API (`createDeviceManagementKit`, `dmk.executeDeviceAction`, `dmk.sendApdu`). The internals are documented at the DMK package level; this guide stays at the integration boundary.
- ERC-7730 metadata authoring. The CLI can clear-sign transactions whose contracts have ERC-7730 descriptors registered with Ledger's metadata service, but writing those descriptors is the Clear Signing team's surface (CS/6038847989).

### 5.1.9 Footguns to know upfront

Before you run `bun install` or `pnpm wallet-cli start`, internalise the following list. These bite people in week one and are not flagged anywhere except the SKILL.md and a handful of source comments.

**1. Base uses the Ethereum app on the device.**

The CLI's SKILL.md says it explicitly:

> "Supported networks: bitcoin, ethereum, base, solana (mainnet and testnets). On Ledger devices, **Base uses the Ethereum app**."

The README and source say nothing. If you `discover` a Base account and the CLI prompts you to open "Ethereum" on the device, that's expected. The device firmware handles Base, Polygon, Arbitrum, Optimism, and other EVM L2s through the Ethereum app — they share a chainId-driven derivation. Internally the CLI's `withCurrencyDeviceSession` looks up `getCryptoCurrencyById(currencyId).managerAppName` and opens whatever the live-common metadata says — for any EVM family chain, that's "Ethereum".

> **Verify:** confirm that `live-common-setup.ts` actually wires Base into `setSupportedCurrencies()`. R1 flagged that the README lists only bitcoin / ethereum / solana; the SKILL.md mentions base; the source has not been confirmed to call `setSupportedCurrencies(["base"])` explicitly. Base support may be implicit through EVM family detection or may not be reachable at all in the CLI surface today.

**2. Persistent DMK to avoid stacking node-usb hotplug listeners.**

A comment in `src/device/register-dmk-transport.ts` reads:

> "One DMK per CLI process: each `createDeviceManagementKit()` adds node-usb hotplug listeners; closing + recreating stacks listeners and breaks the 3rd in-process connect (same pattern as the former node-hid kit)."

Translation: do not call `createDeviceManagementKit()` more than once per process. If you write a test that loops `for (const account of accounts) { await someCliFunction() }` and that function calls `createDeviceManagementKit()` internally, the third iteration will fail with a garbled APDU error. The CLI handles this by exposing one process-wide singleton (`persistentDmk`) and a `_setTestDmkTransport(t)` seam for tests. Respect the singleton.

**3. Sandbox: device-touching commands need `dangerouslyDisableSandbox: true` in the Bash tool.**

This applies to AI agents driving the CLI from inside Claude Code or Cursor — and that includes you when you write QA hooks with Claude Code's help. From SKILL.md:

> "Sandbox: Commands that open the USB device (`account discover`, `receive`, `send`) **must** be run with `dangerouslyDisableSandbox: true` in the Bash tool — the sandbox blocks USB access and causes a `Timeout has occurred` error. Commands that don't need the device (`balances`, `operations`, `send --dry-run`) work fine in the sandbox."

Plain shell users (CI, local dev terminal) are not affected. Only the agent harness sandbox.

**4. Storage layout in flux.**

There are two competing storage layouts in flight, and they disagree:

- **ADR (TA/6978928805 — "ADR — CLI Storage", status: Proposed)** says `~/.ledger/cli/*.yaml`.
- **LIVE-29495 (the actually shipping code)** uses `@bunli/utils stateDir(APP_NAME)` which on Linux/macOS resolves to `~/.local/state/ledger-wallet-cli/session.yaml` (XDG state dir), not `~/.ledger/cli/`.

The shipping code wins until the ADR moves from Proposed to Accepted. Don't be surprised if the location changes in a future release. Tests use `XDG_STATE_HOME` to redirect to a temp dir per test (`test/helpers/session-fixture.ts`).

**5. wallet-cli is v0.x experimental; expect breakage.**

The README states it twice — once at the top:

> "Experimental command-line tool for Ledger Wallet flows over USB, built on the Device Management Kit (DMK) and Bunli. Version 0.1.0 (see `bunli.config.ts`)."

— and once under "Status":

> "wallet-cli is **not production-ready**. Behavior and flags may change without notice."

For QA hooks consumed by the regression suite, this is a real risk: a flag rename in wallet-cli 0.2.x will break your hook silently if your hook does not pin the wallet-cli workspace version. Treat wallet-cli the way you'd treat any pre-1.0 dependency: pin via the monorepo lockfile, add an integration smoke test, plan for refactor effort on minor bumps.

**6. Device contention.**

From SKILL.md again:

> "Only one command can use the device at a time. Never run two device-required commands in parallel — they will conflict and fail with `[object Object]` or a garbled APDU error. Run device commands sequentially."

This is enforced at the DMK singleton level (you cannot open two USB sessions to the same device anyway). Practical implication for hooks: in `Promise.all([revoke(), swap()])`, the second one will fail unpredictably. Always serialise device-touching steps.

**7. `--data` is unsanitised raw calldata.**

The EVM intent schema (`src/wallet/intents/families/evm.ts`) accepts `--data` as any 0x-prefixed hex string with even length. The CLI does not parse it, does not match it against a whitelist of selectors, does not warn. If you pass calldata that authorises an unlimited transfer, the device will display the raw transaction (clear-signed if there's an ERC-7730 descriptor for the contract, blind-signed if not), and any human pressing the button has the final word. For automated hooks, treat `--data` the way you'd treat raw SQL: only build it from validated components, never accept it as an external string from a config file or env var.

**8. Bunli's `--dry-run` boolean flag.**

From the open ticket **LIVE-29404** ("`--dry-run` flag silently ignored"):

> "@bunli/core's `option()` requires explicit value for `z.boolean()` options; bare `--dry-run` is dropped."

This is a live Bunli behaviour, not a wallet-cli bug. If you write `wallet-cli send … --dry-run`, Bunli may not bind it. The current workaround at the time of writing is `--dry-run true` or `--dry-run=true`. **This is the most dangerous footgun on the list:** a CI hook intending a dry run that gets interpreted as a real broadcast moves real funds. Verify the behaviour against the version you're using and add an explicit `=true` in production hooks. Chapter 5.7 covers safe broadcast guards.

### 5.1.10 Resources

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/6907625515/Ledger+Wallet+CLI">Ledger Wallet CLI (PRD) — VAUL/6907625515</a> — the canonical product spec for v0/v1/v2 phases, command grammar, JSON envelope, terminal UX, security principles. Quote source for everything "what this CLI is supposed to do".</li>
<li><a href="https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/7039778832/Ledger+Wallet+CLI+internal+testing">Ledger Wallet CLI internal testing — VAUL/7039778832</a> — the dogfood runbook: prerequisites, the six-prompt suite, expected results, common errors and remediation.</li>
<li><a href="https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/6951665759/Ledger+Live+CLI+for+AI+Agent">Ledger Live CLI for AI Agent (architecture study) — TA/6951665759</a> — audit of the legacy <code>apps/cli</code>'s 59-62 commands, friction points, and the proposed error-code taxonomy.</li>
<li><a href="https://ledgerhq.atlassian.net/wiki/spaces/CS/pages/7014678571/Device+Management+Kit">Device Management Kit overview — CS/7014678571</a> — DMK's five responsibilities, the Client → DMK → Signer diagram, the Clear Signing fallback story.</li>
<li><a href="https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/6995411067/LedgerJS+vs+DMK+Benchmark+Comparison">LedgerJS vs DMK Benchmark — WXP/6995411067</a> — the tier model: LedgerJS = blind sign, DMK no token = gated, DMK + token = full clear signing. The "&gt;10x connectivity error reduction" quote.</li>
<li><a href="https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/6978928805/ADR+CLI+Storage">ADR — CLI Storage — TA/6978928805</a> — the (Proposed) storage layout ADR: YAML, <code>~/.ledger/cli</code>. Conflicts with the actually shipping XDG-based location.</li>
<li><a href="https://ledgerhq.atlassian.net/browse/QAA-615">QAA-615 — Spike: Revoke token approval using CLI</a> — the Jira ticket that motivated this guide part.</li>
<li><a href="https://ledgerhq.atlassian.net/browse/LIVE-29404">LIVE-29404 — <code>--dry-run</code> flag silently ignored</a> — the open Bunli boolean-flag bug.</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> wallet-cli is the third E2E surface — the one without a user-facing application. It exists because some test-data operations (allowance revokes, fixture resets, account discovery) are simply not what Playwright and Detox are for, and a domain-specific CLI does them faster and more reliably. It lives at <code>apps/wallet-cli/</code> next to (but not replacing) the legacy <code>apps/cli</code>. It runs on Bun, talks to the device through DMK over USB, and tests itself via in-place mock seams rather than a peer test workspace. Six footguns matter on day one: Base uses the Ethereum app, the DMK is a process-wide singleton, the agent sandbox blocks USB, the storage layout ADR conflicts with what ships, the v0.x version contract is "expect breakage", and Bunli's bare boolean flags currently get dropped. Internalise those before Chapter 5.2 puts you in front of a real prompt.
</div>

### 5.1.11 Quiz

<!-- ── Chapter 5.1 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> Why does QAA-615 propose a CLI command (rather than a Playwright test) to revoke a token allowance between regression runs?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Playwright cannot sign transactions on a Ledger device</button>
<button class="quiz-choice" data-value="B">B) Playwright tests are not permitted to broadcast to mainnet</button>
<button class="quiz-choice" data-value="C">C) Token allowances are persistent on-chain state that does not reset between runs; a one-shot data op is faster, less flaky, and doesn't need a built application</button>
<button class="quiz-choice" data-value="D">D) The Ledger device only exposes its USB transport to CLI processes</button>
</div>
<p class="quiz-explanation">An ERC-20 allowance lives inside the token contract. Once approved, it stays approved until explicitly set to zero. To re-test the approval flow, you must revoke between runs. Doing that through the Live UI takes 60+ seconds and a built app; a CLI does it in three. The CLI is test-data infrastructure, not a Playwright competitor.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> Where do wallet-cli's integration tests live?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) In a peer workspace at <code>e2e/cli/</code>, mirroring <code>e2e/desktop/</code> and <code>e2e/mobile/</code></button>
<button class="quiz-choice" data-value="B">B) Inside the application workspace at <code>apps/wallet-cli/src/test/</code></button>
<button class="quiz-choice" data-value="C">C) Under <code>libs/test-helpers/wallet-cli/</code></button>
<button class="quiz-choice" data-value="D">D) Inside <code>apps/cli/</code> alongside the legacy CLI tests</button>
</div>
<p class="quiz-explanation">wallet-cli is small (~5,860 lines) and uses internal mock seams (<code>_setTestDmkTransport</code>) that would leak across a workspace boundary. Tests live in-place at <code>apps/wallet-cli/src/test/</code>, run with <code>bun test src/</code>, and spawn the actual <code>cli.ts</code> via a custom <code>cli-runner</code> helper.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> Which CLI does the team direct new work to, given that <code>apps/cli</code> still exists alongside <code>apps/wallet-cli</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>apps/wallet-cli/</code> for new work; fall back to <code>apps/cli/</code> only when wallet-cli does not yet replicate the feature (swap, bot, app management, firmware ops)</button>
<button class="quiz-choice" data-value="B">B) <code>apps/cli/</code> for everything because it has 62 commands and is on npm</button>
<button class="quiz-choice" data-value="C">C) Either one, freely interchanged within a single hook</button>
<button class="quiz-choice" data-value="D">D) <code>apps/wallet-cli/</code> only when the operation is read-only; everything else goes through legacy</button>
</div>
<p class="quiz-explanation">wallet-cli has the modern stack (Bun, DMK, V1 descriptors, JSON envelope) and is where new work belongs. Legacy <code>apps/cli</code> is still consulted for features wallet-cli has not replicated, but the two should never be mixed in one shell script — different transports, different descriptor formats, different runtimes.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> What does the SKILL.md mean by "Base uses the Ethereum app"?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) wallet-cli does not support Base at all</button>
<button class="quiz-choice" data-value="B">B) Base transactions are blind-signed, never clear-signed</button>
<button class="quiz-choice" data-value="C">C) The Base app on the device is named "Ethereum" for branding reasons</button>
<button class="quiz-choice" data-value="D">D) On the Ledger device firmware, EVM L2s like Base, Polygon, Arbitrum and Optimism are signed via the Ethereum app — there is no separate Base app to open. wallet-cli's <code>withCurrencyDeviceSession</code> opens whatever the live-common metadata's <code>managerAppName</code> declares, which is "Ethereum" for any EVM family.</button>
</div>
<p class="quiz-explanation">This is a silent constraint in source and README — only the SKILL.md flags it explicitly. The device firmware uses the Ethereum app for any chainId in the EVM family. If you discover a Base account and see "Open the Ethereum app", that's expected.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> Why does <code>register-dmk-transport.ts</code> hold a process-wide DMK singleton instead of creating one per command?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) DMK is too expensive to instantiate</button>
<button class="quiz-choice" data-value="B">B) Bun does not garbage-collect DMK objects</button>
<button class="quiz-choice" data-value="C">C) Each <code>createDeviceManagementKit()</code> call adds node-usb hotplug listeners; closing and recreating stacks listeners and breaks the 3rd in-process connect</button>
<button class="quiz-choice" data-value="D">D) The Ledger device only accepts one DMK session per firmware lifetime</button>
</div>
<p class="quiz-explanation">A source comment in <code>register-dmk-transport.ts</code> spells it out. Tests respect the singleton via the <code>_setTestDmkTransport()</code> seam.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Bun & Bunli Primer

<div class="chapter-intro">
The CLI runs on a stack you have not seen earlier in the guide. Part 3's desktop suite is Node-on-Electron. Part 4's mobile suite is Node-on-Detox. Part 5's CLI is Bun. The command framework on top of Bun is Bunli. This chapter covers just enough Bun to stop you tripping over the differences from Node, then takes you through Bunli's command shape carefully — because every command you write or read will look like the same five-piece skeleton — and ends with you running your first wallet-cli command.
</div>

### 5.2.1 Why Bun

Bun is a JavaScript runtime, package manager, bundler, and test runner. It started in 2022 as "a faster Node" and reached 1.0 in 2023. By 2026 it is stable enough to ship production CLIs on. The wallet-cli team picked it; here's the reasoning condensed.

**Speed.** Bun's startup is on the order of 50-150 ms cold for a non-trivial TypeScript file. Node's is closer to 200-500 ms with `tsx` or `ts-node`. For a CLI you might invoke 50 times in a regression hook, a 150 ms-per-invocation saving compounds. The numbers are not magic — Bun is written in Zig, uses JavaScriptCore (Safari's engine, faster startup than V8), and has a faster module resolver — but the practical effect is "runs and exits before you notice".

**Native TypeScript.** No `tsc` step before run. No `ts-node`. No `tsx`. `bun run ./src/cli.ts` executes the file. The wallet-cli's `start` script is literally `"start": "bun run ./src/cli.ts"`. There is still a `pnpm typecheck` script (`tsc --noEmit -p tsconfig.json`) that catches type errors Bun's bundler doesn't enforce at runtime — Bun strips types, it doesn't check them — but you don't pay the type-check cost on every run.

**Native test runner.** Bun ships `bun test`. No Jest, no Vitest, no setup. The test framework is API-compatible with Jest in the parts that matter (`describe`, `it`, `expect`, `beforeAll`/`afterEach`, mocks). It also has a `--coverage` flag that emits lcov by default. The wallet-cli's `bunfig.toml` configures it:

```toml
[test]
coverageReporter = ["lcov"]
coverageDir = "coverage"
```

That's the whole test runner config. Compare with `e2e/mobile/jest.config.js` and the per-environment Jest setup files from Part 4.

**`bun --compile`** produces a single ~94 MB binary that embeds the Bun runtime and all dependencies. The wallet-cli targets four platforms (`darwin-arm64`, `linux-arm64`, `linux-x64`, `windows-x64`) via `bunli build`. For QA, the implication is that you can drop a wallet-cli binary into a CI runner without a Node install or a pnpm install. Open ticket **LIVE-29308** ("Installation/distribution strategy") tracks the alternative `npm install -g @ledgerhq/wallet-cli` path; today the standalone binary is the primary distribution model.

**What Bun is not.** Bun is not yet 100% Node-compatible. There are corners where it diverges:

- Bun's resolver treats CJS and ESM imports of the same module as **separate singletons**. wallet-cli's `live-common-setup.ts` calls `LiveConfig.setConfig(walletCliConfig)` twice for that reason — once via ESM, once via CJS — because Alpaca-routed families read ESM and bridge-routed families read CJS.
- Bun's bundler can't follow `node-gyp-build`'s dynamic `fs.readdirSync` for native addons. wallet-cli works around this with `embed-usb-native.ts`, a manual `require()` of the platform-specific `usb/prebuilds/*/node.napi.node`.
- `@bunli/core`'s default `createCLI()` does a dynamic `import('file://...')` of the generated commands manifest. In Bun standalone mode that hangs. wallet-cli applies a local patch to `@bunli/core` and replaces the dynamic import with a static one in `cli.ts` (you'll see the comment in the next section).

These are not deal-breakers — the team works around them — but they are the kind of thing a "just use Bun" pitch elides. You will see Bun-specific workarounds in the wallet-cli source and they all trace back to one of the three issues above.

The reader question to keep in mind: is Bun the right choice for *your* CLI? For wallet-cli, yes — speed and the standalone binary outweigh the workarounds. For a one-off ten-line script, probably not. For Part 5 you don't need to relitigate the choice; you just need to know the runtime well enough to read the source.

### 5.2.2 Bunli at a glance

Bunli is a CLI framework for Bun. It is to Bun what Yargs or Commander is to Node — it parses argv, validates flags, dispatches to handlers, generates `--help`. Three things distinguish it:

1. **Command-and-handler model**, similar to oclif or Cobra (Go) but lighter. You declare a command with `defineCommand({ name, description, options, handler })`. Bunli builds the parser tree, validates with Zod, calls the handler.
2. **Codegen step.** `bunli generate` walks the commands directory and produces a static command manifest. The manifest is committed to `.bunli/commands.gen.ts`. CI verifies it via a `generate:check` script.
3. **`bunli build` produces standalone binaries.** Each target (`darwin-arm64`, `linux-arm64`, `linux-x64`, `windows-x64`) gets its own Bun-compiled binary with the runtime embedded.

Where Bunli lives in the project:

- **`bunli.config.ts`** — top-level config: name, version, commands directory, build targets.
- **`bunfig.toml`** — Bun runtime config (the `[test]` section affects `bun test`).
- **`.bunli/commands.gen.ts`** — codegen output. Auto-generated, committed.
- **`src/commands/`** — per-command modules. Each file (or sub-directory `index.ts` for groups) exports a `defineCommand` or `defineGroup` default.

The `bunli.config.ts` file in the wallet-cli is small. Verbatim:

```ts
import { defineConfig } from "bunli";

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

Five fields you'll touch and zero you won't:

- `name` — the binary name. After `bunli build`, the binary is `wallet-cli`.
- `version` — currently `0.1.0`. Pre-1.0 means breaking changes are allowed.
- `description` — appears in `--help`.
- `commands.directory` — Bunli scans this tree for `defineCommand`/`defineGroup` modules.
- `build` — entry, outdir, targets, minify flag.

The minify flag is the reason `embed-usb-native.ts` works. With minify on, Bun evaluates `process.platform` and `process.arch` at compile time, dead-code-eliminating the branches for other platforms. Each binary contains only its own platform's prebuilt USB addon.

Now look at the entry file (`src/cli.ts`, full text — 20 lines):

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

Six side effects, in order:

1. **`embed-usb-native`** — fires the platform-specific `require()` so Bun bundles the native addon.
2. **`createCLI` import** — pulls in the patched `@bunli/core`.
3. **`live-common-setup`** — registers coin modules with live-common, registers the DMK transport module so anywhere live-common does `withDevice("wallet-cli-dmk")(…)` it routes through DMK, configures `LiveConfig` (twice, ESM + CJS).
4. **`commands.gen`** — the generated manifest. Importing it triggers `registerGeneratedStore(...)`, statically wiring every command into the Bunli store.
5. **`createCLI(bunliConfig)`** — builds the CLI from the explicit config.
6. **`cli.run()`** — Bunli parses argv, validates flags, dispatches.

After `cli.run()` returns, the success path explicitly disposes the DMK to release node-usb hotplug listeners. Error paths exit via Bunli's own `process.exit(1)` and skip the dispose; that's intentional because errors already triggered cleanup.

The two non-obvious decisions:

- **Why static import of `commands.gen.ts`?** Because Bun standalone mode hangs on `@bunli/core`'s default dynamic `import('file://...')`. The local patch replaces dynamic with static.
- **Why pass `bunliConfig` explicitly?** Because the standalone binary can't find `bunli.config.ts` at runtime (its `cwd` is wherever the user runs it from, not the project root). The config is bundled in.

### 5.2.3 The Bunli command shape

Every Bunli command file follows the same skeleton. The smallest non-trivial example in the wallet-cli is `src/commands/balances.ts` — 34 lines. Here it is annotated.

```ts
// 1. Imports — Bunli's defineCommand and Zod's option helper, plus the
//    shared option helpers (account-resolution, output format) and the
//    output abstraction.
import { defineCommand, option } from "@bunli/core";
import { z } from "zod";
import { accountOption, outputOption, resolveAccountArg, resolveAccountDescriptor } from "./inputs";
import { createCommandOutput } from "../output";
import { networkStringFromCurrencyId } from "../shared/accountDescriptor";
import { WalletAdapter } from "../wallet";

// 2. The default export. defineCommand is Bunli's command constructor.
export default defineCommand({

  // 3. Name — what the user types after the binary. For a top-level
  //    command, this is the verb directly: `wallet-cli balances`.
  name: "balances",

  // 4. Description — used in `--help` output, both for the parent and
  //    in the command's own help.
  description: "Print balances for an account (native + token sub-accounts).",

  // 5. Options — keyed by flag name. Each value is `option(zodSchema, meta)`.
  //    Bunli auto-generates --help text from the Zod descriptions and
  //    validates the parsed value against the schema before calling the
  //    handler.
  options: {
    account: accountOption,    // `--account/-a <descriptor-or-label>`
    output: outputOption,      // `--output human|json` (default human)
  },

  // 6. The handler. Receives `flags` (the validated option values) and
  //    `positional` (the array of positional args). Returns Promise<void>.
  handler: async ({ flags, positional }) => {

    // 6a. Build the JSON-envelope context. `command`, `network`, and
    //     `account` are filled in as we resolve them.
    const ctx = { command: "balances", network: "", account: "" };

    // 6b. createCommandOutput returns either a HumanCommandOutput or a
    //     JsonCommandOutput depending on flags.output. Both expose the
    //     same semantic methods (`balances`, `address`, `sendEvent`).
    const out = createCommandOutput(flags.output, ctx);

    // 6c. out.run(fn) wraps the body. Errors thrown inside fn are
    //     caught by the JSON output (which writes an error envelope and
    //     exit(1)) or rethrown by the human output (so Bunli's default
    //     error display kicks in).
    await out.run(async () => {

      // 6d. Resolve the account argument. resolveAccountArg picks the
      //     value from --account, falling back to positional[0]. It
      //     throws if neither is set.
      const accountArg = resolveAccountArg(flags.account, positional);

      // 6e. resolveAccountDescriptor handles two shapes:
      //     - contains ":" → it's a descriptor, parse it directly.
      //     - no ":" → it's a session label, look it up in session.yaml.
      const descriptor = await resolveAccountDescriptor(accountArg);

      // 6f. Fill in the envelope context now that we know the network.
      ctx.network = networkStringFromCurrencyId(descriptor.currencyId);
      ctx.account = descriptor.id;

      // 6g. Fetch the data via WalletAdapter (which routes to Alpaca
      //     for EVM, bridge sync otherwise).
      const balances = await WalletAdapter.getAccountBalances(descriptor);

      // 6h. Hand the data to the output abstraction. It formats
      //     human-readable lines on stdout or buffers for the JSON
      //     envelope.
      out.balances(balances);
    });
  },
});
```

Five things to internalise:

- **Zod everywhere.** Flags are Zod schemas. Intent shapes are Zod schemas. Outputs are Zod schemas. Validation happens before your handler runs and after your handler returns. There is no manual `if (typeof x === "string")`.
- **`out.run(fn)` is the canonical wrapper.** Every command's handler calls `out.run(async () => { … })`. That's where error-to-envelope translation happens, where the JSON shape is computed, where the human spinner is started and stopped. Don't write commands that bypass it.
- **Three resolution helpers**, all in `src/commands/inputs.ts` (the file the briefing called `shared-options.ts`). `resolveAccountArg` picks between `--account` and positional. `resolveAccountInput` handles label-vs-descriptor. `resolveAccountDescriptor` parses to a V0 descriptor (the legacy live-common shape). For V1-only commands there's `resolveAccountDescriptorV1`.
- **Context object is mutable.** `ctx.network` and `ctx.account` are filled in *after* the descriptor is resolved. The output abstraction reads `ctx` each time you call a method — so when your handler eventually calls `out.balances(...)`, the envelope already has the right `network` and `account` set.
- **The handler returns nothing.** Bunli does not collect a return value. Output goes through `out.*` methods (which know whether to write text or buffer JSON). Exit code is 0 unless `out.run` catches an error.

For comparison, look at the legacy CLI's `balances` equivalent in `apps/cli/`. It's free-form `console.log` with no envelope, no Zod, no shared option helpers. That's the Yargs/Commander idiom — and it's why TA/6951665759's friction-point list opens with "Output is human-readable strings. There is no structured JSON envelope."

The architectural value of the Bunli command shape is that **every command file looks the same**. Once you've read `balances.ts`, you've read the structure of `operations.ts`, `receive.ts`, `send.ts`, and the future `revoke.ts`. The differences are in:

- **Which options are declared** — `send.ts` has `--to`, `--amount`, `--data`, `--dry-run`, etc.
- **Whether the handler opens a device session** — `balances` and `operations` skip `withCurrencyDeviceSession`; `receive` (with `--verify`), `send`, and `account discover` use it.
- **Which `WalletAdapter` method is called** — `getAccountBalances`, `getAccountOperations`, `verifyAddress`, `send`, etc.
- **What the output type is** — a list, a single value, a stream of events.

Those are the four axes of variation. The plumbing — option parsing, error envelope, JSON serialisation, spinner, exit code — is identical across all commands. Once you can write one command, you can write all of them.

The Bunli command-and-handler model gives you the structure for free. You write five lines of declaration, you get `--help`, type validation, parsing, dispatch, and a typed `flags` object inside your handler.

A group command (subcommand parent) uses `defineGroup` instead of `defineCommand`. Example from `src/commands/account/index.ts`:

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

This makes `wallet-cli account discover` and `wallet-cli account fresh-address` valid invocations. Bunli composes `name` segments to build the dispatch tree.

### 5.2.4 The codegen step

Bunli's static command manifest lives at `.bunli/commands.gen.ts`. Its header is unambiguous:

> "This file was automatically generated by Bunli. You should NOT make any changes in this file as it will be overwritten."

What it does (215 lines, condensed):

```ts
// 1. Static imports of every command module.
import Account from '../src/commands/account/index.js';
import Balances from '../src/commands/balances.js';
import Discover from '../src/commands/account/discover.js';
import FreshAddress from '../src/commands/account/fresh-address.js';
import Operations from '../src/commands/operations.js';
import Receive from '../src/commands/receive.js';
import Reset from '../src/commands/session/reset.js';
import Send from '../src/commands/send.js';
import Session from '../src/commands/session/index.js';
import View from '../src/commands/session/view.js';

// 2. Builds a `modules` map (path → command) and a `metadata` map
//    (path → mirrored option schema).

// 3. Calls registerGeneratedStore(createGeneratedHelpers(modules, metadata)).

// 4. Auto-registers commands on import.
```

Why the codegen exists:

1. **Static dispatch.** Without the manifest, Bunli would walk `src/commands/` at runtime via `fs.readdir`. That works in dev but breaks in standalone-binary mode where the directory doesn't exist on the user's filesystem.
2. **Type-safe command registry.** The generated `metadata` map mirrors every command's option schema, which downstream tooling can introspect (autocompletion, AI skills).
3. **Single source of truth.** The team can add a command, run `bunli generate`, and the manifest is updated. CI verifies via `pnpm generate:check`.

The `generate:check` script is a CI gate:

```json
"generate:check": "bunli generate && git diff --exit-code -- .bunli/commands.gen.ts"
```

It runs the generator, then `git diff --exit-code` fails if anything changed. If the developer added a command but forgot to commit the regenerated manifest, CI catches it. This is identical in spirit to the lockfile drift check in many monorepos: the generated artefact is committed and verified.

For the QA reader, three implications:

- **Adding a command is two steps**: write the file under `src/commands/`, run `pnpm generate`. Forgetting step 2 fails CI.
- **Don't hand-edit `commands.gen.ts`**. Your change will be obliterated on the next `pnpm generate`.
- **The manifest is also why `cli.ts` does the static import**. The manifest registers commands as a side effect of being imported. Static import pulls in the side effect; dynamic import would be slower and (in Bun standalone) hangs.

### 5.2.5 Shared options across commands

`src/commands/inputs.ts` (53 lines) defines the option helpers every leaf command imports. Two of them are flags; the rest are resolution functions.

```ts
import { option } from "@bunli/core";
import { z } from "zod";
import { OutputFormatSchema, parseAccountDescriptor } from "../wallet/models";
import { parseV1, type AccountDescriptorV1 } from "../shared/accountDescriptor";
import { Session } from "../session/session-store";

// Shared flag definitions.

export const accountOption = option(z.string().min(1).optional(), {
  description: "Account descriptor or session label (e.g. ethereum-1). Can also be the first positional arg.",
  short: "a",
});

export const outputOption = option(OutputFormatSchema.default("human"), {
  description: "Output format: human (default) or json",
});

// Resolution helpers.

export function resolveAccountArg(account: string | undefined, positional: readonly string[]): string {
  const arg = account ?? positional[0];
  if (!arg) throw new Error("Missing account: use --account <descriptor-or-label> or pass as first positional argument.");
  return arg;
}

// Contains ":" → V0/V1 descriptor passthrough; no ":" → session label lookup.
export async function resolveAccountInput(input: string): Promise<string> {
  if (input.includes(":")) return input;
  const session = await Session.read();
  const entry = session.accounts.find(e => e.label === input);
  if (!entry) {
    throw new Error(
      `No account labeled "${input}" in session. Available labels: ${session.accounts.map(e => e.label).join(", ") || "(none)"}. Run "wallet-cli account discover <network>" to populate.`
    );
  }
  return entry.descriptor;
}

export async function resolveAccountDescriptor(input: string) {
  return parseAccountDescriptor(await resolveAccountInput(input));
}

export async function resolveAccountDescriptorV1(input: string): Promise<AccountDescriptorV1> {
  return parseV1(await resolveAccountInput(input));
}
```

Three things this file does that you should not duplicate per-command:

1. **`accountOption`** standardises the `--account/-a` flag with a uniform description and shorthand. Every command that takes an account adopts this exact flag definition.
2. **`outputOption`** standardises `--output human|json` with a `human` default. Every command supports both modes by including this option.
3. **`resolveAccountInput`** implements the **session label resolver**. If your input contains `":"`, it's a descriptor; pass it through. If not, it's a label like `ethereum-1`; look it up in `session.yaml`. After running `account discover ethereum`, you get auto-generated labels (`ethereum-1`, `ethereum-2`, …) and every subsequent command accepts the short label.

The "labels resolve when no colon" rule is concise enough to memorise. It also explains why the V1 descriptor uses `h` for hardened path segments instead of `'`: a single quote breaks shell quoting, and `h` is a documented alias in the V1 spec. Hardened path `m/44'/60'/0'/0/0` becomes `m/44h/60h/0h/0/0`.

When you write a new command (say, `revoke`), you should import the same helpers:

```ts
import { accountOption, outputOption, resolveAccountArg, resolveAccountDescriptor } from "./inputs";

export default defineCommand({
  name: "revoke",
  description: "...",
  options: {
    account: accountOption,
    spender: option(z.string().regex(/^0x[0-9a-fA-F]{40}$/), { description: "DEX router address" }),
    output: outputOption,
  },
  handler: async ({ flags, positional }) => {
    const descriptor = await resolveAccountDescriptor(resolveAccountArg(flags.account, positional));
    // …
  },
});
```

You get the right flag descriptions, the right shorthand, the right session-label behaviour, the right error messages — all from one import.

### 5.2.6 Output: human vs JSON envelope

Every wallet-cli command supports `--output human` (default, colourful, spinner-driven) and `--output json` (one envelope per process to stdout).

The JSON envelope is the contract for any test hook consuming the CLI. It looks like this for success (from `src/shared/response.ts`):

```ts
// makeEnvelope(command, network, data, account?)
return {
  status: "success",
  command,
  network,
  ...(account == null ? {} : { account }),
  ...data,
  timestamp: new Date().toISOString(),
};
```

So a successful `balances` command returns:

```json
{
  "status": "success",
  "command": "balances",
  "network": "ethereum:main",
  "account": "account:1:address:ethereum:main:0x71C7656EC7ab88b098defB751B7401B5f6d8976F:m/44h/60h/0h/0/0",
  "balances": [
    { "asset": "ethereum", "amount": "1.5 ETH" },
    { "asset": "ethereum/erc20/usd_tether__erc20_", "amount": "100 USDT" }
  ],
  "timestamp": "2026-04-27T14:32:00.000Z"
}
```

For a successful `send`:

```json
{
  "status": "success",
  "command": "send",
  "network": "ethereum:main",
  "account": "account:1:address:ethereum:main:0x…:m/44h/60h/0h/0/0",
  "tx_hash": "0xabc...123",
  "amount": "0.5 ETH",
  "fee": "0.0003 ETH",
  "timestamp": "2026-04-27T14:32:00.000Z"
}
```

For a `--dry-run` send, the payload is `{ dry_run: true, recipient, amount, fee }` and there is no `tx_hash`.

The error envelope is **a different shape** — and this is a real wart, flagged in R1:

```json
{ "ok": false, "error": { "command": "balances", "message": "No account labeled \"foo\" in session…" } }
```

There's no `status: "success"` on errors. There's `ok: false` instead. Tests check `data.ok === false` for error and `data.command` for success. R1 calls this "a real wart" and recommends future versions unify on a single shape; for now, hooks must handle both.

Why human and JSON share the same handler:

- **`createCommandOutput(flags.output, ctx)`** picks the implementation:
  - **`HumanCommandOutput`** uses `yocto-spinner` on stderr, ANSI colours on stdout, and re-throws errors so Bunli's default error display fires.
  - **`JsonCommandOutput`** buffers data, emits a single envelope at the end via `writeSync(1, JSON.stringify(envelope))`. On error it writes the error envelope and `process.exit(1)`. (It uses `writeSync(1, …)` because Bun.spawn on Linux doesn't reliably capture fd 2 (stderr) — observable in CI, hence the workaround.)

For a hook, the contract is simple:

1. Spawn `wallet-cli <cmd> --output json`.
2. Capture stdout. Wait for process exit.
3. `JSON.parse(stdout.trim())`. If `parsed.ok === false`, the command failed — read `parsed.error.message`. Otherwise read the success fields.
4. Don't capture stderr for parsing — that's where spinners and human messages go in mixed-mode CI runs.

A worked example. Consider the QAA-613 nightly hook that has to ensure allowance is zero before each test:

```ts
// e2e/desktop/hooks/revoke-before-swap.ts (illustrative — Chapter 5.8 ships the real version)
import { spawn } from "node:child_process";

export async function ensureAllowanceZero(label: string, spender: string): Promise<void> {
  const args = ["wallet-cli", "start", "--", "revoke", label, "--spender", spender, "--output", "json"];
  const proc = spawn("pnpm", args, { stdio: ["ignore", "pipe", "inherit"] });
  let stdout = "";
  proc.stdout.on("data", (chunk) => { stdout += chunk; });
  const code = await new Promise<number>((res) => proc.on("close", res));
  const parsed = JSON.parse(stdout.trim());
  if (parsed.ok === false || code !== 0) {
    throw new Error(`revoke failed: ${parsed.error?.message ?? "unknown"}`);
  }
  // parsed.tx_hash is now available for logging.
}
```

That's the entire integration shape. The hook is twenty lines, the test it serves is unchanged, and the wallet-cli does the heavy lifting (build calldata, sign on device, broadcast, parse receipt). When the regression suite runs, this hook executes before each swap test, the chain state resets to allowance=0, and the test starts deterministic.

This pattern is also what makes Chapter 5.8's QAA-615 walkthrough land cleanly — the spike's deliverable is the `revoke` command, the consumer is exactly this kind of pre-test hook.

### 5.2.7 Bun's test runner — first peek

Bun's test runner is the default for `bun test src/`. It's API-compatible with Jest in the parts you'll use (`describe`, `it`, `expect`, `beforeAll`, `afterEach`, `vi.mock` equivalents).

For wallet-cli, `pnpm test` runs `bun test src/`. `pnpm coverage` adds `--coverage`. Coverage output is lcov per `bunfig.toml`.

A single test from `src/commands/inputs.test.ts` (19 lines) gives you the flavour:

```ts
import { describe, it, expect } from "bun:test";
import { resolveAccountArg } from "./inputs";

describe("resolveAccountArg", () => {
  it("prefers --account over positional", () => {
    expect(resolveAccountArg("ethereum-1", ["ethereum-2"])).toBe("ethereum-1");
  });
  it("falls back to positional[0] when --account is undefined", () => {
    expect(resolveAccountArg(undefined, ["ethereum-2"])).toBe("ethereum-2");
  });
  it("throws when neither is set", () => {
    expect(() => resolveAccountArg(undefined, [])).toThrow(/Missing account/);
  });
});
```

Three things to notice:

- `import { describe, it, expect } from "bun:test"` — note the `bun:` namespace. In Jest you import from `@jest/globals` (or rely on globals); in Bun you import from `bun:test`.
- No config file. The test runner picks up `*.test.ts` files under the path you give it (`bun test src/`).
- Mocks are colocated. Tests that need a mock device install it via env vars (`WALLET_CLI_MOCK_DMK=1`). Tests that need a mock HTTP server spin up a `MockServer` in `beforeEach`.

Chapter 5.6 walks the entire test harness: `cli-runner`, `mock-DMK`, `http-intercept`, `mock-server`, `session-fixture`. For now, the takeaway is:

- Unit tests live next to the file they cover (`inputs.test.ts` next to `inputs.ts`).
- Integration tests live under `src/test/commands/` and spawn the actual CLI subprocess to assert on stdout.
- Both run under `bun test src/`.

### 5.2.8 Your first wallet-cli command — `balances --help`

Time to actually run something. We'll do this in three steps and avoid touching the device.

**Step 1: install dependencies.**

From the repo root:

```bash
pnpm install
```

This installs everything in the monorepo. wallet-cli is one workspace among many — pnpm picks it up automatically.

**Step 2: get help.**

The cleanest first invocation is `--help`. It needs no device, no network, and works in the agent sandbox:

```bash
pnpm wallet-cli start -- balances --help
```

The `--` separates pnpm's own argv from the wallet-cli's. `pnpm wallet-cli start` resolves to `pnpm --filter @ledgerhq/wallet-cli run start --` (per the root-level passthrough in the monorepo's `package.json`), which resolves to `bun run ./src/cli.ts`, which then receives `balances --help` as its own argv.

You should see something like:

```
Usage: wallet-cli balances [options]

Print balances for an account (native + token sub-accounts).

Options:
  -a, --account <string>     Account descriptor or session label (e.g. ethereum-1).
                             Can also be the first positional arg.
      --output <human|json>  Output format: human (default) or json (default: "human")
  -h, --help                 Show help
```

The flag list and descriptions come from your Zod schemas in `inputs.ts`. The `-a` shorthand comes from `accountOption`'s `short: "a"`. The default value of `--output` comes from `outputOption`'s `OutputFormatSchema.default("human")`.

**Step 3: try a no-device JSON call.**

`balances` does not need the device — it routes to Alpaca (for EVM) or to a bridge sync (other families). Both are network calls, not USB. So we can run it in the sandbox.

Use a known address (Vitalik's, padded with the deterministic Hardhat address for path):

```bash
pnpm wallet-cli start -- balances \
  account:1:address:ethereum:main:0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045:m/44h/60h/0h/0/0 \
  --output json
```

You'll get something like:

```json
{
  "status": "success",
  "command": "balances",
  "network": "ethereum:main",
  "account": "account:1:address:ethereum:main:0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045:m/44h/60h/0h/0/0",
  "balances": [
    { "asset": "ethereum", "amount": "<some non-zero ETH amount> ETH" },
    { "asset": "ethereum/erc20/usd_coin", "amount": "<USDC balance> USDC" }
  ],
  "timestamp": "2026-04-27T14:32:00.000Z"
}
```

> **Verify:** the actual balances depend on Vitalik's wallet at the time you run this. The `assets` keys come from live-common's CAL (Crypto Assets List) — the format is `<chain>/<token-standard>/<asset-id>`.

**Step 4 (optional, if you have a device):** discover your own accounts.

```bash
pnpm wallet-cli start -- account discover ethereum --output json
```

The CLI will:
1. Detect the device over USB.
2. Print `[~] Waiting: Ledger detected but locked. Enter your PIN on the device.` to stderr if locked.
3. Print `[i] Opening Ethereum app on your Ledger... ACTION REQUIRED: Confirm on device screen.` and prompt you on the device.
4. Stream discovered accounts as JSON envelope fields.
5. Persist labels to `~/.local/state/ledger-wallet-cli/session.yaml`.

After this, `pnpm wallet-cli start -- balances ethereum-1 --output json` works without re-typing the descriptor.

Remember the sandbox rule from 5.1.9: device-touching commands need `dangerouslyDisableSandbox: true` in the agent harness. Plain shell users don't hit this.

### 5.2.9 The 5+ commands at a glance

There's a discrepancy between the SKILL.md (which says "5 commands") and the actual codebase (R1 found 10 leaf commands across 2 groups). The likely explanation: the SKILL.md was written when the project shipped 5 commands; the codebase has grown since. The Confluence PRD (VAUL/6907625515) tracks the *target* surface, not the *current* one. Always consult the source tree of the day.

Here's the catalog as of R1's audit:

| Command | Group | Device? | Purpose |
|---|---|---|---|
| `account discover` | account | yes | Scan accounts for a network, persist labels to session.yaml |
| `account fresh-address` | account | no | Next unused receive address (no device unless UTXO + sync needed) |
| `balances` | (top) | no | Native + token balances for an account |
| `operations` | (top) | no | Transaction history with `--limit`, `--cursor` |
| `receive` | (top) | yes (default) | Returns address; `--verify=true` confirms on device |
| `send` | (top) | yes (unless `--dry-run`) | Build, sign, broadcast a transaction |
| `session view` | session | no | Print all `(label, descriptor)` pairs |
| `session reset` | session | no | Wipe `session.yaml` |

Plus there are unit-only helpers (`inputs.ts` resolution functions) that are not commands.

Notable missing commands relative to the legacy CLI: no `swap`, no `bot`, no `app install`, no `firmwareUpdate`, no `signMessage`, no `tokenAllowance`, no `genuineCheck`. Chapter 5.4 walks each of these eight commands deeply with example output and edge cases. For now, the takeaway is that the surface is small, every command speaks the same envelope, and the device contract (which commands need it, which don't) is uniform.

The PRD adds future commands not yet implemented:

- `swap quote / execute / status` (v1).
- `secrets init / encrypt` (v0.5).
- `token approve / revoke` (the QAA-615 spike target — not in PRD as a named verb today, see the chapter outro of 5.1).

### 5.2.10 Bunli vs Yargs vs Commander

A quick positioning paragraph because you'll have used at least one of these in past Node CLIs.

- **Yargs** — the workhorse of Node CLIs. Fluent builder API (`yargs.command(...).option(...).argv`). Excellent help generator. Validates types loosely (it's mostly stringly-typed). The thing you reach for when you need a CLI in five minutes.
- **Commander** — the other workhorse. Class-based API. Slightly cleaner for nested subcommands. Same loose typing. Shipped in many old codebases.
- **oclif** — Salesforce's CLI framework. Class-per-command. Heavy, opinionated, stuck in CommonJS for a long time. Powerful for very large CLIs (Heroku, Salesforce, Adobe).
- **Bunli** — Bun-native, codegen-based, Zod-validated. The thing you pick when you're already on Bun and want typed flags.

Why the team picked Bunli is not documented in any ADR R4 found. Inferred reasons:

- Bun-native, no Node-bridging.
- Static codegen plays well with Bun's standalone-binary mode (no runtime fs walks).
- Zod-typed flags eliminate a class of runtime errors (mistyped string-as-boolean, missing required, wrong enum).
- Single-binary build through `bunli build`.

You won't choose between these for QA hooks — you'll consume what wallet-cli provides — but knowing where Bunli sits in the framework landscape helps when you read its source and wonder why a `defineCommand` instead of a `class extends Command`.

### 5.2.11 Resources

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://bun.sh/docs">Bun documentation</a> — the runtime, package manager, bundler, test runner. Start with the "CLI" section for <code>bun run</code> / <code>bun test</code> / <code>bun --compile</code>.</li>
<li><a href="https://www.npmjs.com/package/bunli">Bunli on npm</a> — official package readme. Defines <code>defineCommand</code>, <code>defineGroup</code>, <code>option</code>.</li>
<li><a href="https://zod.dev">Zod documentation</a> — the schema validator behind every <code>option(...)</code> in wallet-cli.</li>
<li><a href="https://ledgerhq.atlassian.net/browse/LIVE-29495">LIVE-29495</a> — Session layer: discover persists to session.yaml. Confirms the XDG state-dir layout.</li>
<li><a href="https://ledgerhq.atlassian.net/browse/LIVE-29404">LIVE-29404</a> — bare <code>--dry-run</code> silently dropped. Live Bunli boolean-flag bug.</li>
<li><a href="https://ledgerhq.atlassian.net/browse/LIVE-29308">LIVE-29308</a> — installation/distribution strategy. Standalone binary vs npm install path.</li>
<li>In-tree: <code>apps/wallet-cli/README.md</code>, <code>apps/wallet-cli/src/wallet/README.md</code>, <code>apps/wallet-cli/src/test/commands/README.md</code>, <code>apps/wallet-cli/src/shared/accountDescriptor/README.md</code>.</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Bun is the runtime — fast cold start, native TypeScript, native test runner, single-binary build. Bunli is the framework — declare commands as <code>defineCommand({ name, description, options, handler })</code>, validate flags with Zod, dispatch through a static codegen manifest. Every wallet-cli command follows the same five-piece skeleton: imports, default export, name, options keyed by flag, async handler that wraps its body in <code>out.run(...)</code>. Shared option helpers in <code>src/commands/inputs.ts</code> standardise <code>--account/-a</code> and <code>--output</code> across every command, and the session-label resolver makes <code>--account ethereum-1</code> equivalent to the full V1 descriptor after a <code>discover</code>. JSON output follows a stable success envelope (<code>status: "success"</code>, <code>command</code>, <code>network</code>, <code>account</code>, command-specific payload, <code>timestamp</code>) and a separate error envelope (<code>ok: false</code>, <code>error.message</code>) — hooks must handle both shapes. You ran your first command (<code>balances --help</code>, then a no-device JSON call), saw the envelope, and have the catalog of eight commands in your head. Next chapter walks DMK end-to-end so you understand what happens when <code>send</code> actually opens a device session and exchanges APDUs.
</div>

### 5.2.12 Quiz

<!-- ── Chapter 5.2 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> Why does <code>cli.ts</code> statically import <code>../.bunli/commands.gen</code> instead of letting <code>@bunli/core</code> discover commands at runtime?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The team prefers static imports for code style reasons</button>
<button class="quiz-choice" data-value="B">B) <code>@bunli/core</code>'s default behaviour is a dynamic <code>import('file://...')</code> of the manifest, which can hang in Bun standalone-binary mode; a local patch removes the dynamic import and the static one registers commands as a side effect</button>
<button class="quiz-choice" data-value="C">C) Static imports are required for ESM compatibility</button>
<button class="quiz-choice" data-value="D">D) Dynamic imports break Zod schema validation</button>
</div>
<p class="quiz-explanation">A comment in <code>cli.ts</code> explains: "Our @bunli/core patch removes that dynamic import entirely because it can hang in Bun standalone mode, this static import registers commands instead." Each command module's import has a side effect of registering itself into Bunli's command store.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> What is the contract for a wallet-cli JSON success envelope?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Always <code>{ ok: true, data: ... }</code></button>
<button class="quiz-choice" data-value="B">B) Always <code>{ result: ..., error: null }</code></button>
<button class="quiz-choice" data-value="C">C) <code>{ status: "success", command, network, account?, ...payload, timestamp }</code></button>
<button class="quiz-choice" data-value="D">D) The shape varies per command and must be consulted in <code>src/output.ts</code></button>
</div>
<p class="quiz-explanation"><code>src/shared/response.ts</code>'s <code>makeEnvelope</code> spells out the success shape. The error shape is different (<code>{ ok: false, error: { command, message } }</code>) — a real inconsistency hooks have to handle.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> What does <code>resolveAccountInput("ethereum-1")</code> do when the input contains no colon?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Treats the input as a session label, reads <code>session.yaml</code>, and returns the matching descriptor — or throws with a list of available labels</button>
<button class="quiz-choice" data-value="B">B) Throws because all account inputs must be V1 descriptors</button>
<button class="quiz-choice" data-value="C">C) Tries to fetch the descriptor from the network</button>
<button class="quiz-choice" data-value="D">D) Generates a new descriptor with the given label</button>
</div>
<p class="quiz-explanation">The session-label rule is concise: "contains <code>:</code> → descriptor passthrough; no <code>:</code> → session label lookup." After running <code>account discover ethereum</code>, the auto-generated label <code>ethereum-1</code> is registered, and every subsequent command accepts the short form.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> Why does the team commit <code>.bunli/commands.gen.ts</code> to the repository and verify it in CI via <code>generate:check</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because it's faster to import a committed file than to generate one</button>
<button class="quiz-choice" data-value="B">B) Because Bun cannot regenerate it at install time</button>
<button class="quiz-choice" data-value="C">C) Because the codegen depends on network access that CI does not have</button>
<button class="quiz-choice" data-value="D">D) Because the manifest is the single source of truth for the static command tree (Bun standalone can't walk <code>src/commands/</code> at runtime), and committing it plus a <code>git diff --exit-code</code> CI gate ensures developers regenerate after adding/removing commands</button>
</div>
<p class="quiz-explanation">Without the gate, a developer could add a new command file, forget to run <code>pnpm generate</code>, and the CI green-lights a build that doesn't ship the new command. The gate prevents that drift.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> Which command can you run in the agent sandbox without setting <code>dangerouslyDisableSandbox: true</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>account discover ethereum</code></button>
<button class="quiz-choice" data-value="B">B) <code>receive --account ethereum-1</code> (default <code>--verify=true</code>)</button>
<button class="quiz-choice" data-value="C">C) <code>balances --account ethereum-1 --output json</code></button>
<button class="quiz-choice" data-value="D">D) <code>send --to 0x… --amount '0.001 ETH'</code></button>
</div>
<p class="quiz-explanation">The SKILL.md sandbox rule: device-touching commands (<code>account discover</code>, <code>receive</code> with verify, <code>send</code>) need the sandbox bypass. <code>balances</code>, <code>operations</code>, and <code>send --dry-run true</code> work in the sandbox because they hit network only.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q6.</strong> What is the difference between <code>defineCommand</code> and <code>defineGroup</code> in Bunli?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>defineCommand</code> is for sync commands and <code>defineGroup</code> is for async</button>
<button class="quiz-choice" data-value="B">B) <code>defineCommand</code> declares a leaf command with a handler; <code>defineGroup</code> declares a parent that composes child commands under a shared name (e.g. <code>account discover</code>, <code>account fresh-address</code>)</button>
<button class="quiz-choice" data-value="C">C) <code>defineGroup</code> is the legacy API and <code>defineCommand</code> replaces it</button>
<button class="quiz-choice" data-value="D">D) They're aliases — Bunli accepts either form interchangeably</button>
</div>
<p class="quiz-explanation"><code>account/index.ts</code> exports <code>defineGroup({ name: "account", commands: [DiscoverCommand, FreshAddressCommand] })</code>, which is what makes <code>wallet-cli account discover</code> and <code>wallet-cli account fresh-address</code> dispatch correctly. Leaf commands are <code>defineCommand</code>; groups compose them.</p>
</div>

<div class="quiz-score"></div>
</div>

---

<div class="chapter-outro">
<strong>Bun and Bunli installed.</strong> You've placed wallet-cli in the monorepo, distinguished it from the legacy <code>apps/cli</code>, learned why the CLI exists as test-data infrastructure (the QAA-615 origin story), and absorbed the eight footguns that bite people in week one. You've also learned the runtime (Bun) and the command framework (Bunli), read the canonical command shape in <code>balances.ts</code>, and run your first invocation. <strong>Next: Chapter 5.3 — Device Management Kit Primer</strong>. The DMK is the layer between wallet-cli and the physical Ledger over USB. It replaces the older <code>hw-transport-*</code> stack with a typed, signer-aware, reconnection-resilient transport. We'll walk the builder, the singleton lifecycle (and why it has to be a singleton), the <code>ConnectAppDeviceAction</code> wrapper, the error mapper that produces the messages you saw on stderr, and finally the <code>MockDeviceManagementKit</code> seam that makes every CLI integration test hermetic.
</div>
## Device Management Kit Primer

<div class="chapter-intro">
The Device Management Kit (DMK) is the library that sits between any client (Ledger Live Desktop, Ledger Live Mobile, the wallet-cli, third-party wallets) and the physical Ledger signer. It owns transport selection, session lifecycle, APDU framing, ERC-7730 metadata fetch, signer abstraction, and the fallback to blind signing. For QA, the DMK is the layer where "the device behaves" or "the device errors" — and where the wallet-cli plugs into the same stack the desktop and mobile apps already use. This chapter maps what DMK replaces, how its session model differs from the older hw-transport family, how the wallet-cli wires DMK over USB, and where the moving parts live in source.
</div>

### 5.3.1 What DMK replaces

Before DMK there was **LedgerJS** — the `@ledgerhq/hw-transport-*` family. The wallet (any wallet, ours or a third party's) imported a transport package per environment:

- `@ledgerhq/hw-transport-node-hid` — Node desktop, USB HID
- `@ledgerhq/hw-transport-webusb` — browser, WebUSB
- `@ledgerhq/hw-transport-webhid` — browser, WebHID
- `@ledgerhq/hw-transport-web-ble` — browser, Bluetooth
- `@ledgerhq/hw-transport-http` — HTTP-Speculos for tests

…and on top of that a per-app helper: `@ledgerhq/hw-app-eth`, `@ledgerhq/hw-app-btc`, `@ledgerhq/hw-app-solana`, etc. Each helper exposed methods like `signTransaction(path, rawTx)` that wrote raw APDUs to the transport. The wallet itself was responsible for: opening the transport, opening the right embedded app on the device, recovering when the user pressed a button, dealing with disconnect, and packaging clear-sign metadata if any.

The DMK consolidates all of that. From the **LedgerJS vs DMK Benchmark & Comparison** page (Confluence WXP/6995411067), three operational tiers:

| Tier | Surface | Clear Signing | Transaction Checks |
|---|---|---|---|
| LedgerJS | `hw-transport-*` + `hw-app-*` | None — blind signing only | None |
| DMK without origin token | `@ledgerhq/device-management-kit` + `device-signer-kit-*` | Gated, basic experience | None |
| DMK + commercial agreement | DMK + origin token | Full ERC-7730 clear signing | Blockaid / Cyvers / ENS |

Internal data on the same page reports a **>10× reduction in connectivity errors** after migrating to DMK. That number is for the desktop/mobile wallet, not the CLI specifically — but the wallet-cli rides the same code path (see 5.3.4) so it inherits the same connectivity gains.

The signer kits replace the per-app helpers:

- `@ledgerhq/device-signer-kit-ethereum`
- `@ledgerhq/device-signer-kit-solana`
- `@ledgerhq/device-signer-kit-bitcoin`
- `@ledgerhq/device-signer-kit-hyperliquid`
- `@ledgerhq/device-signer-kit-cosmos`
- `@ledgerhq/device-signer-kit-aleo`
- `@ledgerhq/device-signer-kit-zcash`

> **Verify:** the wallet-cli pulls the Ethereum signer kit transitively through `@ledgerhq/live-common`'s `DmkSignerEth` rather than depending on it directly. Bitcoin and Solana paths follow the same pattern. The exact dependency tree is whatever `pnpm why @ledgerhq/device-signer-kit-ethereum` reports inside `apps/wallet-cli` at the time of writing.

### 5.3.2 The session model

The single biggest mental shift moving from `hw-transport` to DMK is the **session**.

`hw-transport` was almost stateless. Each transport was a thin wrapper around a USB / HID / BLE handle. The wallet code wrote APDUs to it and read APDUs back. If the device was locked, the wallet found out by getting back a `0x6982 SECURITY_STATUS_NOT_SATISFIED` from a probing APDU. If the device disconnected, the wallet got a USB error on the next exchange. There was no concept of "device state" in the transport itself — the wallet had to maintain a state machine on top of every command.

DMK flips that. A DMK instance opens a **session** to a discovered device, and the session **tracks state** for you. The session emits an `Observable<DeviceSessionState>` that can be in states like `CONNECTED`, `LOCKED`, `BUSY`. The session has a refresher that periodically pings the device so reads stay fresh. APDU exchanges happen via `dmk.sendApdu({ sessionId, apdu })` — note the `sessionId`, not a transport handle. Higher-level **device actions** (XState machines under the hood) wrap multi-APDU sequences like "open this app, unlock if needed, return when ready" into a single observable.

The wallet-cli leans on this. From `register-dmk-transport.ts`:

```ts
const sessionId = await dmk.connect({ device });

const sessionState = await firstValueFrom(dmk.getDeviceSessionState({ sessionId })).catch(
  () => null,
);
const status =
  sessionState && typeof sessionState === "object" && "deviceStatus" in sessionState
    ? sessionState.deviceStatus
    : null;

if (status === DeviceStatus.BUSY) {
  await dmk.disconnect({ sessionId }).catch(() => {});
  throw new Error(
    "[wallet-cli] The Ledger device did not respond to the initial ping. " +
      "Please run the command again — the retry usually succeeds.",
  );
}
```

Two things worth noticing for QA:

1. **The CLI checks session state immediately after `connect`.** If the device responds `BUSY`, the CLI disconnects and asks the user to retry. This is a wallet-cli policy, not a DMK default — the DMK would happily keep the session in `BUSY` and let you wait.
2. **The session refresher is left enabled** (the comment in source: *"matching the DMK sample app behaviour"*). That means session state stays accurate without the CLI having to poll.

### 5.3.3 Transports under DMK

The DMK exposes **one API at the top** and lets you plug in different transports underneath. The four shipping transports are:

| Transport | Where it runs | Used by |
|---|---|---|
| WebUSB / node-WebUSB | Browser + Node | Ledger Live Desktop (Node), wallet-cli (Node), web wallets |
| WebHID | Browser only | Some web wallets |
| Bluetooth (BLE) | Mobile (RN) + browser | Ledger Live Mobile, browser wallets that support BLE |
| HTTP-Speculos | Node + browser | Tests, CI, anywhere a real device isn't available |

A DMK instance is constructed with one or more transport factories, then the same `dmk.connect`, `dmk.sendApdu`, `dmk.executeDeviceAction` API is used regardless of which transport accepts the device. The wallet-cli registers exactly one factory: `nodeWebUsbTransportFactory` (see 5.3.4). The mobile app registers BLE. Tests register a mock (see 5.3.6) or HTTP-Speculos.

> **Verify:** the wallet-cli today does not register an HTTP-Speculos transport. We confirm that explicitly in 5.3.10 and treat it as a current limitation.

### 5.3.4 wallet-cli's DMK wiring

Four files own the wallet-cli's DMK wiring, all under `apps/wallet-cli/src/device/`:

- `dmk.ts` — the DMK builder (~20 lines)
- `register-dmk-transport.ts` — the process-wide DMK lifecycle manager (~172 lines)
- `wallet-cli-dmk-transport.ts` — the `hw-transport` shim that exposes the DMK to legacy live-common code (~38 lines)
- `connect-ledger-app.ts` — the `ConnectAppDeviceAction` wrapper (~201 lines)

#### `dmk.ts` — the builder

```ts
import {
  DeviceManagementKit,
  DeviceManagementKitBuilder,
  LogLevel,
} from "@ledgerhq/device-management-kit";
import { nodeWebUsbTransportFactory } from "./node-webusb";
import { LedgerLiveLogger } from "@ledgerhq/live-dmk-shared/services/LedgerLiveLogger";
import { UserHashService } from "@ledgerhq/live-dmk-shared/services/UserHashService";
import { getEnv } from "@ledgerhq/live-env";

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

What this gives us:

- A single Node-WebUSB transport. The wallet-cli is **USB-only** today.
- The shared `LedgerLiveLogger` at `Warning` level — same logger Ledger Live uses, so DMK warnings are routed through the same channel.
- A `firmwareDistributionSalt` derived from the `USER_ID` env (or `"wallet-cli"` default). This salt is part of how Ledger phases firmware rollouts; for QA work it does not matter, but it must be present.

The builder pattern (`DeviceManagementKitBuilder`) is the public DMK API. You can chain `.addTransport()`, `.addLogger()`, `.addConfig()` — each returns the builder. You finalize with `.build()`.

#### `register-dmk-transport.ts` — the lifecycle manager

The most consequential file in this directory. The headline pattern is the **persistent DMK instance**. Here is the comment from source, verbatim:

```ts
let singleton: Singleton | null = null;
/** One DMK per CLI process: each `createDeviceManagementKit()` adds node-usb hotplug listeners; closing + recreating stacks listeners and breaks the 3rd+ in-process connect (same pattern as the former node-hid kit). */
let persistentDmk: DeviceManagementKit | null = null;
let exitHooksRegistered = false;
```

The trap, in plain language: **`createDeviceManagementKit()` is not free**. Each call attaches new USB hotplug listeners (via `node-usb`, the underlying library). If the CLI naively closed and recreated the DMK between commands, listeners would stack. The third in-process connect would deadlock or silently fail.

So the wallet-cli keeps **one DMK alive for the life of the process**. Sessions come and go inside that one DMK. There are three lifecycle helpers:

```ts
function getOrCreatePersistentDmk(): DeviceManagementKit {
  if (!persistentDmk) {
    persistentDmk = createDeviceManagementKit();
  }
  return persistentDmk;
}

export async function ensureWalletCliDmkTransport(): Promise<WalletCliDmkTransport> {
  if (_testTransport) {
    singleton ??= { dmk: _testTransport.dmk, transport: _testTransport };
    return singleton.transport;
  }

  if (singleton) {
    return singleton.transport;
  }

  const dmk = getOrCreatePersistentDmk();
  const sessionId = await connectFirstUsbDevice(dmk);
  const transport = new WalletCliDmkTransport(dmk, sessionId);
  singleton = { dmk, transport };
  return transport;
}
```

Read the singleton check carefully. `singleton` holds *the active session*. `persistentDmk` holds *the DMK itself*. Resetting the session does **not** tear down the DMK:

```ts
export async function resetWalletCliDmkSession(): Promise<void> {
  const held = singleton;
  if (!held) {
    return;
  }
  singleton = null;
  await held.dmk.disconnect({ sessionId: held.transport.sessionId }).catch(() => {});
}
```

Only on process exit do we close the kit fully:

```ts
export async function disposeWalletCliDmkTransportFully(): Promise<void> {
  await resetWalletCliDmkSession();
  if (persistentDmk) {
    closeDmkQuietly(persistentDmk);
    persistentDmk = null;
  }
}
```

…and that runs in two places: the SIGINT/SIGTERM handlers (registered once) and `cli.ts` after `cli.run()` returns. Process exit is the only legitimate trigger for `dmk.close()`.

##### Why this matters for QA

If you write an E2E hook that spawns multiple wallet-cli commands in quick succession — `revoke`, then `send`, then `operations` — each command is a **separate `wallet-cli` process**. Each process gets its own DMK. The hotplug-listener-stacking problem does not apply across processes. Inside a single process (e.g. when the CLI offers an interactive REPL, which it does not today, or when bun-tests reuse the binary), the singleton matters.

#### Discovering the device

`connectFirstUsbDevice` is where the `Observable<DiscoveredDevice[]>` model becomes visible:

```ts
const discovered = await firstValueFrom(
  dmk.listenToAvailableDevices({}).pipe(
    filter((list: DiscoveredDevice[]) => list.length > 0),
    timeout(CONNECT_TIMEOUT_MS),
  ),
);
const device = discovered[0];
if (!device) {
  throw new Error("No Ledger device found. Unlock the device and try again.");
}
const sessionId = await dmk.connect({ device });
```

`listenToAvailableDevices` returns a hot observable of *currently visible* devices. The CLI takes the first emission with at least one device, then `dmk.connect({ device })` opens a session against that one. `CONNECT_TIMEOUT_MS = 60_000`: the user has 60 seconds to plug in and unlock.

#### `wallet-cli-dmk-transport.ts` — the shim

Live-common still consumes a classic `hw-transport`-shaped object in many places (the bridges, especially). To keep the wallet-cli from rewriting all of live-common, the wallet-cli wraps the DMK in an `hw-transport`-compatible class:

```ts
export class WalletCliDmkTransport extends TransportClass {
  readonly dmk: DeviceManagementKit;
  sessionId: string;

  constructor(dmk: DeviceManagementKit, sessionId: string) {
    super();
    this.dmk = dmk;
    this.sessionId = sessionId;
  }

  async exchange(
    apdu: Buffer,
    { abortTimeoutMs }: { abortTimeoutMs?: number } = {},
  ): Promise<Buffer> {
    const { data, statusCode } = await this.dmk.sendApdu({
      sessionId: this.sessionId,
      apdu: new Uint8Array(apdu.buffer, apdu.byteOffset, apdu.byteLength),
      abortTimeout: abortTimeoutMs ?? WALLET_CLI_DEFAULT_APDU_ABORT_MS,
    });
    return Buffer.from([...data, ...statusCode]);
  }

  close(): Promise<void> {
    return Promise.resolve();
  }
}
```

This is the gluing class. It exposes `.dmk` and `.sessionId` so live-common's `isDmkTransport(transport)` returns true and code that *can* use DMK directly (e.g. `DmkSignerEth`) bypasses the legacy APDU path. For code that *can't* yet, the inherited `exchange()` route still works — it's just APDU in, APDU out.

The `WALLET_CLI_DEFAULT_APDU_ABORT_MS = 120_000` is a 2-minute hard ceiling so a stuck USB exchange cannot block the CLI indefinitely. This is the wallet-cli's policy on top of the DMK's `abortTimeout` parameter.

#### Registering the transport with live-common

At the end of `register-dmk-transport.ts`:

```ts
export function registerWalletCliDmkTransport(): void {
  if (registered) {
    return;
  }
  registered = true;

  registerWalletCliDmkProcessExitHooks();

  registerTransportModule({
    id: MODULE_ID,
    open: (id, _timeoutMs, _context, _matchDeviceByName) => {
      if (id !== WALLET_CLI_DMK_DEVICE_ID) {
        return undefined;
      }
      return ensureWalletCliDmkTransport();
    },
    disconnect: id => {
      if (id !== WALLET_CLI_DMK_DEVICE_ID) {
        return null;
      }
      return resetWalletCliDmkSession();
    },
  });
}
```

This is the bridge to live-common's `withDevice(...)` API. Anywhere in live-common code where a bridge does:

```ts
withDevice("wallet-cli-dmk")(transport => /* … */ )
```

…live-common asks every registered transport module whether it owns that id. The wallet-cli's module says yes for `"wallet-cli-dmk"` and returns the singleton transport. Live-common is none the wiser that USB is going through DMK.

#### `connect-ledger-app.ts` — opening the embedded app

The last piece. Before signing an Ethereum transaction we need the Ethereum app to be open on the device. DMK ships a device action for that: `ConnectAppDeviceAction`. The wallet-cli's wrapper adds a few QA-grade hardenings:

- A **positive unlock timeout** (`CONNECT_APP_UNLOCK_TIMEOUT_MS = 60_000`). Live's wallet uses 0 (instant fail if locked); the CLI gives the user a minute to enter their PIN.
- A **silence watchdog** (`CONNECT_APP_SILENCE_TIMEOUT_MS = unlockTimeout + 60_000`). `RxJS timeout()` alone does not fire when DMK/USB blocks without scheduling a timer; the wrapper adds a wall-clock `setTimeout` and calls `cancel()` from the executeDeviceAction return value.
- **Retry on transport framing errors** (up to 5 attempts, 3-second delay). Comment from source:

```ts
/**
 * When the device is locked inside an app (e.g. Ethereum), the DMK's `waitForDeviceUnlock` polling
 * may receive an unparseable APDU response (`ReceiverApduError`) instead of error code 5515.
 * The polling misidentifies this as "unlocked" and the action fails immediately.
 * We retry the whole device action so the user has time to unlock.
 */
const MAX_TRANSPORT_ERROR_RETRIES = 5;
const TRANSPORT_ERROR_RETRY_DELAY_MS = 3_000;
```

> **Verify:** this retry behaviour is a workaround for what looks like a race in DMK's `waitForDeviceUnlock` polling. The condition (`ReceiverApduError` vs the expected `0x5515` LOCKED_DEVICE) is documented in source but not in any Confluence page I found. Treat the comment as the canonical explanation.

The progress messages — *"[~] Waiting: Ledger detected but locked. Enter your PIN on the device."* and *"[i] Opening Ethereum app… ACTION REQUIRED: Confirm on device screen."* — match the canonical terminal prompts in the wallet-cli PRD (VAUL/6907625515 §4) almost word-for-word.

### 5.3.5 Device errors

`apps/wallet-cli/src/device/wallet-cli-device-error.ts` is the error-mapping policy in 86 lines. It takes anything thrown by DMK or live-common and maps it to a wallet-cli-flavoured `Error` with a user-facing message. The shape is a single function:

```ts
export function toWalletCliDeviceError(error: unknown): Error {
  if (error instanceof ManagerDeviceLockedError || error instanceof LockedDeviceError) {
    return new Error(
      "[wallet-cli] Device is locked. Unlock your Ledger with your PIN and try again.",
      { cause: error },
    );
  }

  if (error instanceof TransportStatusError) {
    const { statusCode } = error;
    if (statusCode === StatusCodes.LOCKED_DEVICE) { /* … */ }
    if (statusCode === StatusCodes.SECURITY_STATUS_NOT_SATISFIED) { /* … */ }
    if (statusCode === StatusCodes.CONDITIONS_OF_USE_NOT_SATISFIED) {
      return new Error("[x] Transaction Cancelled: Rejected on device. No funds moved.", {
        cause: error,
      });
    }
    if (
      statusCode === StatusCodes.CLA_NOT_SUPPORTED ||
      statusCode === StatusCodes.INS_NOT_SUPPORTED
    ) { /* … */ }
  }

  if (error instanceof SendApduTimeoutError || isSendApduTimeoutTagged(error)) { /* … */ }
  if (isTransportFramingError(error)) { /* … */ }

  if (error instanceof Error) {
    return error;
  }

  return new Error(String(error));
}
```

For QA, this function is the **error catalogue**. Five distinct error categories the CLI surfaces:

| Trigger | What QA sees |
|---|---|
| `LockedDeviceError` / `ManagerDeviceLockedError` / status `0x5515` | `[wallet-cli] Device is locked. Unlock your Ledger with your PIN and try again.` |
| status `0x6982` (`SECURITY_STATUS_NOT_SATISFIED`) | `[wallet-cli] The device reported a security error… Unlock the device and try again.` |
| status `0x6985` (`CONDITIONS_OF_USE_NOT_SATISFIED`) — user rejected | `[x] Transaction Cancelled: Rejected on device. No funds moved.` |
| status `0x6E00` / `0x6D00` (CLA / INS not supported) | `[wallet-cli] Could not execute the command on the app. Unlock the Ledger, open the correct app for this currency, and try again.` |
| `SendApduTimeoutError` (DMK class or `_tag`) | `[wallet-cli] Timed out talking to the Ledger over USB. The device may be waiting for your PIN, locked, or busy. Retry the command, check the cable, and make sure no other app is using the device.` |
| `ReceiverApduError` / `UnknownDeviceExchangeError` (transport framing) | `[wallet-cli] Could not communicate with the device (garbled APDU). The device may be locked or busy. Unlock it and try again.` |

Two subtleties worth filing away:

1. **The function passes through unrelated errors.** If DMK throws something that isn't in the catalog, `toWalletCliDeviceError` returns it unchanged (or wraps `String(error)` if it isn't even an Error). That is a deliberate "don't swallow what we don't understand" stance.
2. **`cause` is set on every wrapped error.** Stack traces survive the rewrite. When you debug a CLI failure, `err.cause` is the original DMK / hw-transport error; the wrapper message is for the user.

The `[x]` and `[~]` and `[i]` prefixes are the wallet-cli's plain-text equivalents of the PRD's `[✖]` / `[⧖]` / `[ℹ]` glyphs (PRD §4). Same semantics, terminal-safe.

### 5.3.6 Mock DMK

`apps/wallet-cli/src/device/mock-dmk.ts` is the wallet-cli's **Speculos analog for unit tests**. It replaces the real `DeviceManagementKit` with a class that returns canned results from `executeDeviceAction` and `sendApdu`, without ever talking to USB. From the file's own docstring:

```ts
/**
 * Minimal mock of DeviceManagementKit for integration tests.
 *
 * Mocks at the executeDeviceAction level — the XState machine and InternalApi
 * are bypassed entirely. E2E tests with real hardware cover the layers omitted here.
 *
 *  - ConnectAppDeviceAction (input.application) → Completed immediately
 *  - CallTaskInAppDeviceAction (input.task, input.appName) → Completed with appResults[appName]
 *
 * Device state is controlled via `initialState`.
 * Coin-specific task results are provided via `appResults`.
 */
export class MockDeviceManagementKit {
  private readonly _state: MockDeviceState;
  private readonly _appResults: MockAppResults;
  // …
}
```

The `_state` is `"connected"` or `"locked"`. The `_appResults` is a per-app map: tests inject what `CallTaskInAppDeviceAction` should return for each `appName`:

```ts
export type MockAppResults = Record<string, Record<string, unknown>>;
// Example for Ethereum: { "Ethereum": { publicKey: "...", address: "0x..." } }
// Example for Solana:   { "Solana": { address: Buffer.from("...") } }
// Example for Bitcoin:  { "Bitcoin": { publicKey: "...", bitcoinAddress: "..." } }
```

The discovery side returns a single fake device:

```ts
const FAKE_DEVICE: DiscoveredDevice = {
  id: "mock-device-id",
  name: "Ledger Nano S Plus (mock)",
  deviceModel: new DeviceModel({
    id: "mock-device-id",
    model: DeviceModelId.NANO_SP,
    name: "Ledger Nano S Plus",
  }),
  transport: "mock-transport",
};

listenToAvailableDevices(_args: unknown): Observable<DiscoveredDevice[]> {
  return new BehaviorSubject([FAKE_DEVICE]).asObservable();
}
```

…and the locked path emits a `DeviceActionStatus.Error` with `_tag: "LockedDevice"`:

```ts
private _lockedAction<Output, Error, IntermediateValue>(): ExecuteDeviceActionReturnType<
  Output, Error, IntermediateValue
> {
  const observable = new Observable<DeviceActionState<Output, Error, IntermediateValue>>(
    subscriber => {
      subscriber.next({
        status: DeviceActionStatus.Error,
        // Note: DmkSignerEth._mapError will produce new Error(undefined) = empty message.
        // A future improvement would emit a proper DmkError with errorCode "5515".
        error: { _tag: "LockedDevice" } as unknown as Error,
      });
      subscriber.complete();
    },
  );
  return { observable, cancel: () => {} };
}
```

The mock is installed via the test seam in `register-dmk-transport.ts`:

```ts
let _testTransport: WalletCliDmkTransport | null = null;

/** @internal Test seam — install before the CLI starts (e.g. from dmk-intercept.ts) to bypass USB discovery. */
export function _setTestDmkTransport(t: WalletCliDmkTransport | null): void {
  _testTransport = t;
  singleton = null;
}
```

The test helper `apps/wallet-cli/src/test/helpers/dmk-intercept.ts` constructs a `WalletCliDmkTransport` over a `MockDeviceManagementKit` and calls `_setTestDmkTransport`. Subsequent `ensureWalletCliDmkTransport()` calls return the mock instead of trying to discover real hardware.

We will walk this end-to-end in **Chapter 5.6 — Testing the wallet-cli**, where the mock-DMK + http-intercept + mock-server triad is the wallet-cli's full Speculos analog.

### 5.3.7 USB native binding

`apps/wallet-cli/src/embed-usb-native.ts` exists for one reason: **standalone-binary distribution**. The full file:

```ts
/**
 * Force Bun to embed the `usb` N-API native addon into the standalone binary.
 *
 * `usb` loads its native binding via node-gyp-build which uses dynamic path
 * resolution (fs.readdirSync) that Bun's bundler cannot detect at compile time.
 * A direct require() makes Bun embed the .node file into the executable, and
 * storing it on globalThis lets the patched usb/dist/usb/bindings.js skip
 * node-gyp-build entirely (see patches/usb@2.9.0.patch).
 *
 * With `minify: true`, Bun evaluates process.platform / process.arch as
 * compile-time constants for each --target, so only the matching prebuilt
 * ends up in each platform binary.
 *
 * @see https://bun.sh/docs/bundler/executables#embed-n-api-addons
 */

if (process.platform === "darwin") {
  (globalThis as Record<string, unknown>).__usbNativeAddon =
    require("../node_modules/usb/prebuilds/darwin-x64+arm64/node.napi.node");
} else if (process.platform === "linux" && process.arch === "x64") {
  (globalThis as Record<string, unknown>).__usbNativeAddon =
    require("../node_modules/usb/prebuilds/linux-x64/node.napi.glibc.node");
} else if (process.platform === "linux" && process.arch === "arm64") {
  (globalThis as Record<string, unknown>).__usbNativeAddon =
    require("../node_modules/usb/prebuilds/linux-arm64/node.napi.armv8.node");
} else if (process.platform === "win32") {
  (globalThis as Record<string, unknown>).__usbNativeAddon =
    require("../node_modules/usb/prebuilds/win32-x64/node.napi.node");
}
```

In plain language:

- `usb` is the npm package `node-usb` — the native code that talks to the OS USB stack on macOS / Linux / Windows.
- It normally loads its binding via `node-gyp-build`, which scans `prebuilds/` at runtime to pick the right `.node` file.
- Bun's bundler can't see that scan, so a vanilla `bun build --compile` would produce a binary missing the binding.
- The fix: explicitly `require()` the prebuilt for each `(platform, arch)` and stash the addon on `globalThis.__usbNativeAddon`. A patched `usb/dist/usb/bindings.js` (`patches/usb@2.9.0.patch` in the workspace) then reads from `globalThis` instead of `node-gyp-build`.
- With `minify: true` in `bunli.config.ts`, Bun sees `process.platform === "darwin"` as a compile-time constant per `--target`. Only the matching branch survives in each platform binary; the other prebuilts get tree-shaken.

For QA work this file is **invisible** — you'll never edit it — but it's load-bearing for the standalone binary release path (LIVE-29308 and LIVE-29310, both in flight). If a release build ever ships and silently can't find a Ledger over USB on one platform, this is the first file to look at.

### 5.3.8 Signing flows

A signed transaction in the wallet-cli is the composition of:

1. A **descriptor** (account: family + xpub or address + derivation path).
2. A **transaction intent** (family-specific shape: bitcoin / evm / solana).
3. A **device session** opened via `withCurrencyDeviceSession`.
4. A **bridge `signOperation`** call that streams events as the device works.
5. A **bridge `broadcast`** call that returns a signed `Operation` with an on-chain hash.

The canonical entry point is `commands/send.ts`. Inside the handler, after building and validating the intent:

```ts
const spin = out.spin(`Connect device and open ${colors.bold(descriptor.currencyId)} app…`);
await withCurrencyDeviceSession(descriptor.currencyId, async () => {
  spin?.success("Device session established");
  out.spin(`Preparing ${colors.bold(descriptor.currencyId)} transaction…`);

  await lastValueFrom(
    wallet
      .send(descriptor, intent, WALLET_CLI_DMK_DEVICE_ID, dryRun)
      .pipe(tap(event => out.sendEvent(event))),
  );

  out.sendComplete();
});
```

Three things to read carefully:

1. **`withCurrencyDeviceSession(currencyId, …)`** is what opens the DMK session **and** opens the right embedded app on the device. It's implemented in `apps/wallet-cli/src/session/bridge-device-session.ts`, which calls `ensureWalletCliDmkTransport()` then `connectLedgerApp(dmk, sessionId, managerAppName)`. Inside the `try { return await fn() } finally { resetWalletCliDmkSession() }`, the session is dropped after each command so the device is free for the next one.
2. **`wallet.send(descriptor, intent, WALLET_CLI_DMK_DEVICE_ID, dryRun)`** is `WalletAdapter.send(...)`. It returns an `Observable<SendEvent>` — events of type `device-streaming`, `device-signature-requested`, `device-signature-granted`, `signed`, `broadcast`. Each event hits `out.sendEvent(event)` and updates the spinner / accumulates JSON.
3. **The signing path stays on legacy live-common bridges.** `WalletAdapter.send` delegates to `BridgeAdapter.send`, which calls `bridge.signOperation(...)`. The bridge internally drives `DmkSignerEth` (for EVM) or the equivalent signer kit, which uses DMK's `executeDeviceAction` to run `signTransaction` on the embedded app. The wallet-cli does **not** call `signEthereumTransaction` directly anywhere — it goes through the bridge so it gets free gas/nonce/EIP-1559 estimation, free clear-sign metadata, and free broadcast.

ETH-app APDU specifics (the exact INS values, the chunked payload framing for big transactions, the ERC-7730 metadata streaming) are out of scope for this primer — the signer kit owns all of that. The wallet-cli's job ends at "I sent an `Observable<SendEvent>` to the user".

### 5.3.9 Clear-sign and the ETH plugin

When the wallet-cli sends an EVM transaction with non-empty calldata, the device tries to **clear-sign** it. Clear signing means: the embedded Ethereum app decodes the calldata, recognises a known function selector, and renders human-readable fields on the device screen ("Type: Approve / Amount: 0 USDT / Address: 0x58df…1cf") instead of asking the user to verify a raw hex blob.

The decoder is selector-driven. The wallet-cli does not need to do anything special to trigger it — it just sends a normal EVM transaction whose `data` field starts with a recognised 4-byte selector. The Ethereum app does the rest, with help from ERC-7730 metadata that DMK packages alongside the transaction.

The most relevant selector for QA-615 is **`0x095ea7b3`**, the ERC-20 `approve(address spender, uint256 amount)` function. The Ethereum app's plugin system recognises it and produces:

- `Type: Approve` (or `Revoke` when `amount == 0`)
- `Amount: <human-readable>` (e.g. `0 USDT`, or `Unlimited` for `2**256 - 1`)
- `Address: <spender>` (with ENS / trusted-name resolution if available)

That happens whether the wallet-cli emits the transaction via a future `wallet-cli revoke` command or via the existing `wallet-cli send … --data 0x095ea7b3…`. The path through DMK is identical; only the user-facing CLI surface differs.

What the wallet-cli **does not** do today: ship its own clear-sign tier upgrade. From the LedgerJS-vs-DMK tier model (5.3.1), full clear signing requires a **commercial origin token**. Ledger's first-party CLI inherits Ledger's own token transitively through the DMK builder (the `firmwareDistributionSalt` config in `dmk.ts`).

> **Verify:** the exact mechanism by which the wallet-cli's first-party status flows into DMK's tier decision is not documented in `apps/wallet-cli/src/`. It is plausibly handled inside `@ledgerhq/live-dmk-shared` or downstream. The PRD says *"Embed Clear Signing token in CLI packaging so users can Clear Sign operations in swap that require Clear Signing and get Transaction Check simulation results"* (VAUL/6907625515 §3 enablers) — note "Embed", suggesting the token is bundled into the binary at build time.

We walk a full approve-then-revoke clear-sign session, screen by screen, in **Chapter 5.8 — Clear-sign walkthrough**.

### 5.3.10 DMK on Speculos

A current limitation, important to flag:

**The wallet-cli is USB-only today.** `dmk.ts` registers `nodeWebUsbTransportFactory` and nothing else. There is no HTTP-Speculos transport in the DMK builder, no `--speculos-port` flag in any command, no `withSpeculos(...)` helper.

This means:

- All `wallet-cli` commands that touch the device (`account discover`, `receive`, `send`, the future `revoke`) require **a real Ledger plugged into USB**.
- The wallet-cli's unit tests use the **mock DMK** (5.3.6), not Speculos.
- Speculos-based emulator testing — the bedrock of desktop and mobile E2E — has **no equivalent in wallet-cli today**.

The legacy `apps/cli` did support Speculos (commands like `bot`, `botPortfolio`, `cleanSpeculos`, `speculosList`, per the AI Agent architecture study TA/6951665759). The wallet-cli rewrite has not surfaced an equivalent.

What that means for QA hooks: a CI step that wants to exercise the wallet-cli (e.g. `wallet-cli revoke` between regression runs of the swap suite) **must run on an agent with a real Ledger attached**, or the suite must mock at a higher level (the CLI's stdout). This is currently an open question for the QAA-919 family of regression-automation tickets.

> **Verify:** there is no Confluence page that confirms Speculos support is on the wallet-cli roadmap. The wallet-cli PRD (VAUL/6907625515) does not mention Speculos. LIVE-28269 ("Connect transport layer to DMK") references the silent-app-open behaviour but not an emulator transport. Treat "USB-only" as a true current limitation, not a temporary one.

A possible future shape: the DMK builder gains an `.addTransport(httpSpeculosTransportFactory)` line guarded by an env flag, and `connectFirstUsbDevice` learns to prefer the Speculos transport when `SPECULOS_API_PORT` is set. That is plausible architecturally — the DMK API is identical at the top — but it has to be built. **Today it does not exist.**

### 5.3.11 Resources

Confluence:

- **Device Management Kit (overview)** — https://ledgerhq.atlassian.net/wiki/spaces/CS/pages/7014678571/Device+Management+Kit — the canonical one-pager. DMK's five responsibilities (metadata resolution, payload construction, transport management, signer abstraction, fallback) and the Client → DMK → Signer diagram.
- **LedgerJS vs DMK — Benchmark & Comparison** — https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/6995411067/LedgerJS+vs+DMK+Benchmark+Comparison — the tier model (LedgerJS / DMK no token / DMK + token), the supported signer kits list, and the >10× connectivity-error reduction.
- **DMK Internal testing phase** — https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/5549588557/Device+Management+Kit+Internal+testing+phase — Gherkin-style scenarios for DMK transport behaviour ("Allow Ledger Manager event" and similar). Useful when writing QA test cases for DMK-backed flows.
- **Discovery and connection logic study** — https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/7045516602/Discovery+and+connection+logic+study — `libs/live-dmk-mobile` wraps DMK for React Native; the wrapping pattern is the precedent for `apps/wallet-cli/src/device/`.
- **How to :: Mock DMK** — https://ledgerhq.atlassian.net/wiki/spaces/CIP/pages/6551306298/How+to+Mock+DMK — coin-integration playbook for using `@ledgerhq/device-management-kit` directly when a Signer Kit doesn't exist yet.
- **FIRST PROMPT [DXP] DMK 26' Migration Strat** — https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/6594658575/FIRST+PROMPT+DXP+DMK+26+Migration+Strat — 2026 migration plan; useful for "where is DMK going" context.

Tickets:

- **LIVE-28652** — *Create a DMK Singleton + Shared Provider in `live-dmk-shared`*. The wallet-cli's `persistentDmk` pattern is the CLI-shaped sibling of this one-DMK-per-app rule.
- **LIVE-28269** — *Connect transport layer to DMK*. Source ticket for the silent ConnectApp pattern that `connect-ledger-app.ts` implements.
- **LIVE-22746** — *Verify all errors are remapped in `connectAppEventMapper`*. Cross-reference for `wallet-cli-device-error.ts` parity with live-common.

Source:

- `apps/wallet-cli/src/device/dmk.ts`
- `apps/wallet-cli/src/device/register-dmk-transport.ts`
- `apps/wallet-cli/src/device/wallet-cli-dmk-transport.ts`
- `apps/wallet-cli/src/device/connect-ledger-app.ts`
- `apps/wallet-cli/src/device/wallet-cli-device-error.ts`
- `apps/wallet-cli/src/device/mock-dmk.ts`
- `apps/wallet-cli/src/embed-usb-native.ts`
- `apps/wallet-cli/src/session/bridge-device-session.ts`

<div class="chapter-outro">
<strong>Key takeaway:</strong> DMK replaces the LedgerJS <code>hw-transport</code> + <code>hw-app</code> stack with a session-aware, transport-agnostic, signer-kit-backed library. The wallet-cli wires DMK as a single persistent instance per process (to avoid stacking node-usb hotplug listeners), wraps it in an <code>hw-transport</code>-shaped shim so legacy live-common bridges keep working, and drives signing through those bridges so it inherits free clear-sign, free gas estimation, and free broadcast. The CLI is USB-only today; Speculos support would require a separate transport factory that does not yet exist.
</div>

### 5.3.12 Chapter 5.3 Quiz

<!-- ── Chapter 5.3 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> What is the main reason the wallet-cli keeps a single persistent DMK instance for the life of the process, instead of creating a new one per command?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) DMK construction is slow (several seconds), so caching is a performance win</button>
<button class="quiz-choice" data-value="B">B) DMK requires a Ledger-issued license token that takes time to acquire</button>
<button class="quiz-choice" data-value="C">C) Each <code>createDeviceManagementKit()</code> call attaches new node-usb hotplug listeners; closing and recreating the DMK stacks listeners and breaks the third in-process connect</button>
<button class="quiz-choice" data-value="D">D) Bunli's command framework requires a singleton DMK to dispatch commands</button>
</div>
<p class="quiz-explanation">From <code>register-dmk-transport.ts</code>: "One DMK per CLI process: each <code>createDeviceManagementKit()</code> adds node-usb hotplug listeners; closing + recreating stacks listeners and breaks the 3rd+ in-process connect." Sessions come and go inside the persistent DMK; only process exit triggers <code>dmk.close()</code>.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> The biggest mental shift moving from <code>hw-transport</code> to DMK is:</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) DMK uses synchronous APIs while <code>hw-transport</code> was async</button>
<button class="quiz-choice" data-value="B">B) DMK opens a session and tracks device state (CONNECTED / LOCKED / BUSY); <code>hw-transport</code> was almost stateless and the wallet had to maintain a state machine on top</button>
<button class="quiz-choice" data-value="C">C) DMK requires Bluetooth; <code>hw-transport</code> only supported USB</button>
<button class="quiz-choice" data-value="D">D) DMK mandates TypeScript while <code>hw-transport</code> was JavaScript-only</button>
</div>
<p class="quiz-explanation"><code>hw-transport</code> was a thin handle around USB/HID/BLE — the wallet probed for lock state via APDU error codes and detected disconnect from USB errors on the next exchange. DMK's session emits an <code>Observable&lt;DeviceSessionState&gt;</code> with a refresher that periodically pings the device, plus higher-level "device actions" (XState machines) that wrap multi-APDU sequences.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> Why does <code>WalletCliDmkTransport</code> extend <code>HwTransport</code> instead of just exposing the DMK directly?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) live-common's bridges still consume an <code>hw-transport</code>-shaped object in many places; the shim exposes <code>.dmk</code> + <code>.sessionId</code> so <code>isDmkTransport(transport)</code> returns true while keeping the legacy <code>exchange()</code> path working for code that hasn't been ported yet</button>
<button class="quiz-choice" data-value="B">B) DMK's API is unstable, so the team wraps it for forward compatibility</button>
<button class="quiz-choice" data-value="C">C) <code>hw-transport</code> is required by Bun's bundler</button>
<button class="quiz-choice" data-value="D">D) Performance reasons — <code>hw-transport</code> is faster than DMK</button>
</div>
<p class="quiz-explanation">The shim is a compatibility layer. Code that can use DMK directly (e.g. <code>DmkSignerEth</code>) calls <code>transport.dmk.executeDeviceAction(...)</code>. Code that only knows the old API calls <code>transport.exchange(apdu)</code>, which the shim implements via <code>dmk.sendApdu</code>.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> When <code>connect-ledger-app.ts</code> retries up to 5 times on a <code>ReceiverApduError</code> or <code>UnknownDeviceExchangeError</code>, what is the underlying problem it is working around?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Bad USB cables that drop bytes</button>
<button class="quiz-choice" data-value="B">B) Slow firmware on older Nano S devices</button>
<button class="quiz-choice" data-value="C">C) Network errors when fetching ERC-7730 metadata</button>
<button class="quiz-choice" data-value="D">D) When the device is locked inside an app, DMK's <code>waitForDeviceUnlock</code> polling can receive an unparseable APDU response instead of the expected <code>0x5515</code> LOCKED_DEVICE code, misidentify it as "unlocked", and fail the action immediately — so the wallet-cli retries to give the user time to actually unlock</button>
</div>
<p class="quiz-explanation">Verbatim from the source comment: "When the device is locked inside an app (e.g. Ethereum), the DMK's <code>waitForDeviceUnlock</code> polling may receive an unparseable APDU response (<code>ReceiverApduError</code>) instead of error code 5515. The polling misidentifies this as 'unlocked' and the action fails immediately. We retry the whole device action so the user has time to unlock."</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> What is the wallet-cli's strategy for emulator-based testing without a real Ledger?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It registers an HTTP-Speculos transport in <code>dmk.ts</code> alongside the USB factory</button>
<button class="quiz-choice" data-value="B">B) Unit tests use the <code>MockDeviceManagementKit</code> (a mock at the <code>executeDeviceAction</code> level installed via the <code>_setTestDmkTransport</code> seam); there is no Speculos transport wired into the DMK today, so any hardware-touching command requires a real device</button>
<button class="quiz-choice" data-value="C">C) The wallet-cli reuses the legacy <code>apps/cli</code> Speculos transport via shell invocation</button>
<button class="quiz-choice" data-value="D">D) Tests run against the desktop app's Speculos integration over a Detox-style WebSocket bridge</button>
</div>
<p class="quiz-explanation"><code>mock-dmk.ts</code> bypasses the XState machine and InternalApi, returning canned <code>executeDeviceAction</code> results. It is installed via <code>_setTestDmkTransport</code> from <code>test/helpers/dmk-intercept.ts</code>. The <code>dmk.ts</code> builder registers <code>nodeWebUsbTransportFactory</code> only — there is no HTTP-Speculos transport. This is a current limitation, not a temporary one (5.3.10).</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> When the wallet-cli sends an EVM transaction whose calldata starts with <code>0x095ea7b3</code> (ERC-20 <code>approve</code>), what produces the human-readable "Type: Approve / Amount: 0 USDT / Address: 0x..." screens on the device?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The wallet-cli formats the strings and pushes them to the device as text APDUs</button>
<button class="quiz-choice" data-value="B">B) <code>connect-ledger-app.ts</code> intercepts the calldata and translates it before sending</button>
<button class="quiz-choice" data-value="C">C) The Ethereum embedded app's clear-sign plugin recognises the selector and renders the fields on-device, with help from ERC-7730 metadata that DMK packages alongside the transaction; the wallet-cli does nothing special</button>
<button class="quiz-choice" data-value="D">D) The DMK does the rendering on the host and streams a screenshot to the device</button>
</div>
<p class="quiz-explanation">Clear signing is selector-driven and lives on the device. DMK's job is to fetch the ERC-7730 metadata from the Metadata Service and package it into the APDU payload alongside the raw transaction. The Ethereum app decodes the selector, applies the metadata, and renders the screens. The wallet-cli stays on the legacy live-common bridge path and benefits from this transparently.</p>
</div>

</div>

Up next: **Chapter 5.4 — Codebase Deep Dive** maps the rest of `apps/wallet-cli/` — the Bunli command framework, the wallet adapter, the session layer, the output abstraction, and the test harness — building on the device foundations laid in this chapter.
## wallet-cli Codebase Deep Dive

<div class="chapter-intro">
This is your reference chapter for the CLI side. It catalogues every file in the <code>apps/wallet-cli/</code> workspace, shows the load-bearing ones verbatim, and explains how each piece connects to the rest of the binary. Keep this chapter open when you land in an unfamiliar wallet-cli source file — it will tell you what the file does, why it exists, and where to look next. The first half (5.4.1–5.4.6) covers the workspace layout, the top-level configs, the entry point, the live-common bootstrap, and every leaf command under <code>src/commands/</code>. The second half (5.4.7–5.4.15) covers the device layer, the wallet adapter, the session store, the test harness, and the recurring patterns you should copy when you add a new command.
</div>

### 5.4.1 The Workspace at the Repo Root

`wallet-cli` lives at `apps/wallet-cli/` inside the Ledger Live monorepo, peer to `apps/ledger-live-desktop/` and `apps/ledger-live-mobile/`. Unlike the two GUI apps, it is a Bun-only workspace: a single ESM entry point compiled by Bunli into per-target standalone binaries. There is no React, no Electron, no Detox — just `@bunli/core`, `@ledgerhq/device-management-kit`, `@ledgerhq/live-common`, the `usb` N-API addon, and a thin set of helpers. Total source weight is roughly **5,860 lines** across `src/` (excluding generated and vendored code), so the entire codebase fits in your head.

Here is the full layout to two or three levels:

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
│   └── commands.gen.ts          214 lines  (codegen — DO NOT EDIT)
├── node_modules/                 (workspace-local, populated by pnpm)
└── src/
    ├── cli.ts                    20 lines  (entry — Bunli wiring)
    ├── config.ts                 24 lines  (LiveConfig schema for wallet-cli)
    ├── live-common-setup.ts      80 lines  (registers coin modules + DMK transport)
    ├── embed-usb-native.ts       33 lines  (Bun --compile native addon embed)
    ├── output.ts                338 lines  (CommandOutput abstraction — human/json)
    │
    ├── commands/
    │   ├── inputs.ts             52 lines   (shared option helpers)
    │   ├── inputs.test.ts        19 lines
    │   ├── balances.ts           34 lines
    │   ├── operations.ts         45 lines
    │   ├── receive.ts            48 lines
    │   ├── send.ts              158 lines
    │   ├── account/
    │   │   ├── index.ts           9 lines  (group definition)
    │   │   ├── discover.ts       88 lines
    │   │   └── fresh-address.ts  38 lines
    │   └── session/
    │       ├── index.ts           9 lines  (group definition)
    │       ├── view.ts           20 lines
    │       ├── view.test.ts      89 lines
    │       ├── reset.ts          28 lines
    │       └── reset.test.ts     98 lines
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
    │   ├── log.ts                       11 lines
    │   ├── response.ts                  15 lines
    │   ├── response.test.ts             34 lines
    │   ├── ui.ts                        67 lines  (spinner, colors, isInteractive)
    │   ├── ui.test.ts                   66 lines
    │   └── accountDescriptor/
    │       ├── README.md
    │       ├── index.ts                 24 lines
    │       ├── v0.ts                    10 lines
    │       ├── v1.ts                   147 lines
    │       ├── v1.test.ts              106 lines
    │       ├── network.ts              131 lines
    │       ├── network.test.ts         130 lines
    │       ├── adapters.ts             212 lines
    │       ├── adapters.test.ts        118 lines
    │       └── test-fixtures.ts          4 lines
    │
    ├── wallet/
    │   ├── README.md
    │   ├── index.ts                     96 lines  (WalletAdapter — routing facade)
    │   ├── models.ts                   107 lines  (Zod schemas, AccountDescriptor V0)
    │   ├── models.test.ts               56 lines
    │   ├── compatibility/
    │   │   ├── bridge.ts               295 lines  (BridgeAdapter — live-common bridge)
    │   │   └── alpaca.ts                90 lines  (AlpacaAdapter — direct API)
    │   ├── intents/
    │   │   ├── index.ts                 17 lines
    │   │   ├── index.test.ts           189 lines
    │   │   ├── parse-amount.ts          57 lines
    │   │   ├── parse-amount.test.ts     68 lines
    │   │   └── families/
    │   │       ├── bitcoin.ts           21 lines
    │   │       ├── evm.ts               12 lines
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
            ├── cli-runner.ts            52 lines
            ├── constants.ts              8 lines
            ├── dmk-intercept.ts         29 lines
            ├── http-intercept.ts       146 lines
            ├── eth-sync-routes.ts       37 lines
            ├── mock-server.ts           58 lines
            ├── session-fixture.ts       27 lines
            ├── wrapper.ts                8 lines
            └── wrapper-local.ts          3 lines
```

A few things worth pointing out before we start poking at individual files:

- **Tests live next to the code they exercise.** Pure unit tests (Zod schemas, parsers, formatters, `inputs.ts`, the session store) sit beside their implementation under `src/` as `*.test.ts`. The integration tests — the ones that spawn the compiled CLI as a subprocess and assert on stdout JSON — live under `src/test/commands/`. Both kinds are picked up by the same `bun test src/` run; the split is by **scope**, not by location.
- **`node_modules/` is workspace-local.** pnpm installs wallet-cli's own copy of `@bunli/core`, `usb`, `yocto-spinner`, etc. directly under `apps/wallet-cli/node_modules/`. The `usb` prebuilt `.node` files referenced by `embed-usb-native.ts` resolve there at compile time, not against the monorepo's hoisted root.
- **There is no `src/index.ts`.** The package's `bin` entry (`./src/cli.ts`) is the only public entry point. Everything else under `src/` is internal.
- **No `dist/` is committed.** `bunli build` produces `dist/wallet-cli-darwin-arm64`, `dist/wallet-cli-linux-x64`, etc. on demand. The compiled binaries are the deliverables; the source tree is the workspace.

### 5.4.2 Top-level Configs

Six files at the repo root of `apps/wallet-cli/` configure the build, the test runner, the type-checker, the Nx graph, and Bunli itself. We walk them in the order Bunli reads them.

#### `package.json` — workspace identity, scripts, dependencies

```json
{
  "name": "@ledgerhq/wallet-cli",
  "version": "0.1.1",
  "private": true,
  "description": "Ledger Wallet CLI using Device Management Kit (USB)",
  "bin": {
    "wallet-cli": "./src/cli.ts"
  },
  "type": "module",
  "scripts": {
    "start": "bun run ./src/cli.ts",
    "dev": "bunli dev",
    "generate": "bunli generate",
    "generate:check": "bunli generate && git diff --exit-code -- .bunli/commands.gen.ts",
    "build": "bunli build",
    "typecheck": "tsc --noEmit -p tsconfig.json",
    "test": "bun test src/",
    "coverage": "bun test src/ --coverage",
    "lint": "oxlint src bunli.config.ts",
    "lint:ci": "oxlint src bunli.config.ts --quiet",
    "lint:fix": "oxfmt -c ../../.oxfmtrc.json src bunli.config.ts && oxlint src bunli.config.ts --fix --quiet",
    "format": "oxfmt -c ../../.oxfmtrc.json src bunli.config.ts",
    "format:check": "oxfmt -c ../../.oxfmtrc.json src bunli.config.ts --check"
  }
}
```

The scripts table is the single most important table in the workspace. Memorise the columns or pin this page:

| Script | Command | When you run it |
|---|---|---|
| `start` | `bun run ./src/cli.ts` | Day-to-day: runs the CLI from source against your USB Ledger. This is what `pnpm wallet-cli <args>` ends up calling. |
| `dev` | `bunli dev` | Hot-reload dev mode. Same handler, faster edit loop. |
| `generate` | `bunli generate` | Regenerate `.bunli/commands.gen.ts` from `src/commands/*`. **Run this every time you add or rename a command file.** |
| `generate:check` | `bunli generate && git diff --exit-code -- .bunli/commands.gen.ts` | The CI gate. Fails the PR if you forgot to commit a regeneration. |
| `build` | `bunli build` | Produce the four standalone binaries listed in `bunli.config.ts → build.targets`. |
| `typecheck` | `tsc --noEmit -p tsconfig.json` | Type-only pass. No emit, no bundle. Tests are typechecked here too. |
| `test` | `bun test src/` | Run every `*.test.ts` under `src/` — unit and integration. Bun's runner, not Jest. |
| `coverage` | `bun test src/ --coverage` | Same with lcov reporter (configured in `bunfig.toml`). |
| `lint` / `lint:ci` / `lint:fix` | `oxlint` ± `oxfmt` | Rust-based lint/format toolchain. Same toolchain the desktop and mobile workspaces use. |
| `format` / `format:check` | `oxfmt` against the shared `../../.oxfmtrc.json` | Format only. |

A few dependency highlights, by category:

- **Bunli runtime:** `@bunli/core@0.9.1`, `@bunli/utils@0.6.0`, `bunli@0.9.1` (dev). Bunli is a Bun-native command framework: `defineCommand` + `defineGroup`, Zod-validated flags, codegen for compiled-mode dispatch.
- **Device stack:** `@ledgerhq/device-management-kit` (catalog version), `@ledgerhq/live-dmk-shared`, `@ledgerhq/hw-transport`. The DMK is the same one Live LDK uses; we wire it to the vendored node-WebUSB transport under `src/device/node-webusb/`.
- **Coin families:** `@ledgerhq/coin-bitcoin`, `@ledgerhq/coin-evm`, `@ledgerhq/coin-solana`. Only these three are wired in `live-common-setup.ts` (see 5.4.5). Adding a new family is a config change there, not a workspace change.
- **Live-common surface:** `@ledgerhq/live-common`, `@ledgerhq/live-config`, `@ledgerhq/live-env`, `@ledgerhq/live-wallet`, `@ledgerhq/cryptoassets`, `@ledgerhq/types-live`, `@ledgerhq/types-cryptoassets`, `@ledgerhq/errors`, `@ledgerhq/logs`, `@ledgerhq/coin-module-framework`, `@ledgerhq/ledger-wallet-framework`. The CLI is a real consumer of live-common; it is **not** a parallel implementation.
- **Native USB:** `usb@2.17.0`. The N-API addon embedded by `embed-usb-native.ts`.
- **Misc:** `bignumber.js`, `purify-ts`, `rxjs`, `yocto-spinner`, `zod`, `debug`. `react` is a transitive peer of live-common but not used by the CLI itself.
- **Engines:** `"bun": ">=1.1.0"`. There is no Node fallback; `bun` is required.

#### `bunli.config.ts` — verbatim

```ts
import { defineConfig } from "@bunli/core";

export default defineConfig({
  name: "wallet-cli",
  version: "0.1.0",
  description: "Ledger Wallet CLI",
  commands: {
    directory: "./src/commands",
  },
  // Entry is cli only — *.test.ts under src/ are typechecked but not compiled into the binary.
  build: {
    entry: "./src/cli.ts",
    outdir: "./dist",
    minify: true,
    targets: ["darwin-arm64", "linux-arm64", "linux-x64", "windows-x64"],
    // The `usb` native addon is embedded via a direct require() in src/embed-usb-native.ts.
    // node-gyp-build uses dynamic resolution that Bun can't detect; the explicit require() fixes that.
    // With minify, process.platform branches are dead-code-eliminated per target.
  },
});
```

Two halves to read here:

1. **`commands.directory: "./src/commands"`** is the input to `bunli generate`. Bunli walks that tree, finds every `defineCommand` / `defineGroup` default export, and emits `.bunli/commands.gen.ts` (5.4.2 below). If you put a command file anywhere else, Bunli will not see it.
2. **`build.targets`** is the matrix of standalone binaries `bunli build` produces. Four targets: `darwin-arm64`, `linux-arm64`, `linux-x64`, `windows-x64`. Each one gets its own `dist/<target>/wallet-cli` executable, with the right `usb` prebuilt embedded. Note there is no `darwin-x64` target — the assumption is that intel Macs run the Linux binary inside a container or use Rosetta on the arm64 binary.

The `minify: true` flag is what makes `embed-usb-native.ts` work correctly: with minification on, Bun evaluates `process.platform` and `process.arch` as compile-time constants for each `--target`, so each binary contains only its matching `.node` prebuilt, not all four. The `// With minify, process.platform branches are dead-code-eliminated per target.` comment in the file is making that promise explicit.

#### `bunfig.toml` — verbatim

```toml
# Bun runtime configuration for wallet-cli.

[test]
coverageReporter = ["lcov"]
coverageDir = "coverage"
```

That is the entire file. Two settings, both for `bun test`:

- `coverageReporter = ["lcov"]` makes `bun test --coverage` write `coverage/lcov.info` instead of the default text reporter. CI then uploads that file to Codecov.
- `coverageDir = "coverage"` is where the report lives. The Nx target `coverage` (in `project.json`) declares `{projectRoot}/coverage/**` as its output, so Nx caches it.

There is **no `[install]` or `[run]` section here** — nothing changes how Bun resolves modules at runtime, nothing rewrites jsxImportSource. The CLI is intentionally vanilla Bun.

#### `tsconfig.json` — verbatim

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "noEmit": true,
    "rootDir": ".",
    "types": ["node", "bun-types", "w3c-web-usb"],
    "customConditions": ["import", "types", "default"],
    "moduleResolution": "Bundler",
    "module": "ESNext"
  },
  "include": ["src/**/*.ts", "bunli.config.ts"]
}
```

Twelve lines, but they encode three important decisions:

- **`extends: "../../tsconfig.base.json"`** picks up the monorepo's strict settings (strict mode on, `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`, all the usual safety nets). The CLI does not weaken them.
- **`types: ["node", "bun-types", "w3c-web-usb"]`** says "this binary runs on Bun, talks to USB, and uses Node-shaped APIs". `bun-types` is the Bun runtime types package. `w3c-web-usb` is the WebUSB interface DMK consumes via the vendored node-WebUSB transport.
- **`moduleResolution: "Bundler"` + `customConditions: ["import", "types", "default"]`** is the modern resolution mode. It lets TS read `package.json` `exports` maps the same way Bun does at runtime, so `import "@ledgerhq/live-common/families/evm/setup"` resolves to the same file in both worlds. The `customConditions` order is deliberate — `import` first means we always prefer the ESM build of any dual-published package; `default` is the fallback.

`noEmit: true` is the giveaway that this `tsc` invocation is a type-checker only. Bun handles bundling at build time; at dev time, Bun strips types on the fly.

#### `project.json` — Nx targets

```json
{
  "name": "@ledgerhq/wallet-cli",
  "$schema": "../../node_modules/nx/schemas/project-schema.json",
  "projectType": "application",
  "targets": {
    "coverage": {
      "outputs": ["{projectRoot}/coverage/**"]
    },
    "generate": {
      "outputs": ["{projectRoot}/.bunli/commands.gen.ts"]
    },
    "generate:check": {
      "cache": false
    },
    "build": {
      "cache": false
    }
  }
}
```

Nineteen lines, only declaring **outputs** and **cacheability** — Nx infers the rest from `package.json` scripts. Three takeaways:

- **`coverage` and `generate` are cached**, with explicit outputs so Nx knows what to restore on a hit. A re-run of `nx run @ledgerhq/wallet-cli:generate` after a no-op edit returns instantly.
- **`generate:check` is `cache: false`** because it is a verification step, not a build step. We always want it to actually run `git diff --exit-code` against the working tree; cached "yes it passed last time" is meaningless.
- **`build` is `cache: false`** too. The default Bun build produces four standalone binaries with embedded native addons; Nx's normal output-hashing does not cleanly cover the per-platform binary contents, so we let `bunli build` re-run every time and trust its own incremental layer.

You will most often invoke these via `pnpm --filter @ledgerhq/wallet-cli <script>` or `pnpm wallet-cli <script>` from the repo root.

#### `.bunli/commands.gen.ts` — codegen output

The file at `apps/wallet-cli/.bunli/commands.gen.ts` is **214 lines, generated, and committed**. Its first two lines explain themselves:

```ts
// This file was automatically generated by Bunli.
// You should NOT make any changes in this file as it will be overwritten.
```

What `bunli generate` does is walk `src/commands/`, find every default export that is a `defineCommand(...)` or `defineGroup(...)` value, and emit:

1. **A static import for every command module:**

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

2. **A `names` tuple and a `modules` map** — the "narrow list of command names to avoid typeof-cycles in types":

   ```ts
   const names = [
     'account', 'balances', 'discover', 'fresh-address',
     'operations', 'receive', 'reset', 'send', 'session', 'view'
   ] as const
   ```

3. **A `metadata` map** that mirrors every `option(...)` declaration as plain data (Zod method, default, description, short flag, regex pattern). A single send-command entry looks like:

   ```ts
   'data': {
     type: 'z.string.regex.optional', required: false, hasDefault: false,
     description: 'EVM calldata as 0x-prefixed hex (e.g. 0xd0e30db0)',
     pattern: '^0x([0-9a-fA-F]{2})*$',
     schema: { type: 'zod', method: 'optional', args: [] },
     validator: '(val) => true'
   }
   ```

   This metadata is what Bunli renders into `--help` text and uses to build the JSON schema for tooling that introspects the binary.

4. **A single side-effect call**, registering everything into the global Bunli store:

   ```ts
   export const generated = registerGeneratedStore(createGeneratedHelpers(modules, metadata))
   ```

The reason this file exists at all is the comment near the top of `cli.ts`: `createCLI()` normally tries to dynamically `import('file:///…/.bunli/commands.gen.ts')` from `process.cwd()`, but that dynamic import hangs when Bun's `--compile` flag has produced a single-file binary. Our patched `@bunli/core` removes the dynamic import; `cli.ts` does the static `import "../.bunli/commands.gen"` instead. So the file is part of the **shipped binary**, not just a tooling artefact.

The two related scripts:

- **`pnpm generate`** — regenerate the file. Run this every time you add a new command file, rename one, change the option set, or alter a `defineCommand` description.
- **`pnpm generate:check`** — `bunli generate && git diff --exit-code -- .bunli/commands.gen.ts`. The CI gate. If you forgot to regenerate, this script fails with a diff and the PR is blocked.

> **Note:** R1 reports 215 lines, the on-disk file is 214 lines; the discrepancy is a trailing newline. Treat both as accurate.

The 10 names emitted here account for **8 leaf commands** (`balances`, `operations`, `receive`, `send`, `discover`, `fresh-address`, `view`, `reset`) plus **2 groups** (`account`, `session`). That is the full surface of the CLI.

### 5.4.3 Entry Point — `src/cli.ts`

The entry point is 20 lines. It is short on purpose, because every line that does real work is a side-effect import:

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

Read it as six steps in order, because **the order matters**:

1. **`#!/usr/bin/env bun`** — the shebang. The compiled binary embeds Bun, so this only matters when running the source file directly via `bun run ./src/cli.ts` or `pnpm wallet-cli <args>`. In compiled mode the binary's startup is Bun's own.

2. **`import "./embed-usb-native";`** — must run **before any `usb` import**. This is the file (5.4.2 cross-reference: `src/embed-usb-native.ts`, 33 lines) that does a direct `require("../node_modules/usb/prebuilds/<target>/node.napi.node")` per platform. It stores the loaded N-API addon on `globalThis.__usbNativeAddon`. The patched `usb/dist/usb/bindings.js` (see `patches/usb@2.9.0.patch` in the monorepo root) reads `globalThis.__usbNativeAddon` and skips `node-gyp-build` entirely. If you re-order `cli.ts` so anything else imports `usb` first, the binary breaks.

3. **`import { createCLI } from "@bunli/core";`** — pulls in the (patched) Bunli runtime.

4. **`import "./live-common-setup";`** — registers the three coin modules (bitcoin/evm/solana), pins the supported currencies, registers the DMK transport module under the device id `"wallet-cli-dmk"`, and writes the `LiveConfig` schema. Side effects only; no exports are consumed. We dissect this file in 5.4.5.

5. **`import "../.bunli/commands.gen";`** — the static-import workaround for compiled mode. Registers every command and group into the Bunli store. Without this line the CLI starts up fine but `wallet-cli --help` shows zero commands.

6. **`import bunliConfig from "../bunli.config";`** — the config object. Passed explicitly to `createCLI(...)` so the compiled binary does not have to discover `bunli.config.*` from `process.cwd()`. (This is the second of the two compiled-mode workarounds; the first is the static commands-gen import.)

After the imports:

- `await createCLI(bunliConfig as unknown as Parameters<typeof createCLI>[0])` — instantiates the CLI. The `as unknown as` cast smooths over a type drift between the patched `@bunli/core` and the un-patched Bunli config types.
- `await cli.run()` — Bunli parses `process.argv`, finds the matching command, validates flags via Zod, calls the handler. Errors thrown inside the handler are caught here; in human mode Bunli prints a stack and `process.exit(1)`s, in JSON mode our `JsonCommandOutput` (see 5.4.7 in the second half) writes a `{ ok: false, error: { … } }` envelope first.
- `await disposeWalletCliDmkTransportFully()` — graceful teardown of the singleton DMK + node-usb hotplug listeners. **Only runs on the success path.** Error paths exit before reaching this line because Bunli's default error handler `process.exit(1)`s. SIGINT / SIGTERM hooks (registered inside `register-dmk-transport.ts`, see 5.4.8 in the second half) cover the cancellation path.
- `process.exit(0)` — explicit success exit. Without this, the process would wait for the DMK's `setInterval` timers to drain.

That is the entire bootstrap. There is no router, no middleware, no plugin pipeline. Every interesting decision lives one import away.

### 5.4.4 Configuration — `src/config.ts`

`config.ts` defines the LiveConfig schema for the CLI. It is 24 lines:

```ts
import type { ConfigSchema } from "@ledgerhq/live-config/LiveConfig";
import { appConfig } from "@ledgerhq/live-common/apps/config";
import { bitcoinConfig } from "@ledgerhq/live-common/families/bitcoin/config";
import { evmConfig } from "@ledgerhq/live-common/families/evm/config";
import { solanaConfig } from "@ledgerhq/live-common/families/solana/config";

const countervaluesConfig: ConfigSchema = {
  config_countervalues_refreshRate: {
    type: "number",
    default: 60 * 1000,
  },
  config_countervalues_marketCapBatchingAfterRank: {
    type: "number",
    default: 20,
  },
};

export const walletCliConfig: ConfigSchema = {
  ...countervaluesConfig,
  ...appConfig,
  ...bitcoinConfig,
  ...evmConfig,
  ...solanaConfig,
};
```

What `LiveConfig` does inside live-common is hold a process-wide settings dictionary that bridges, sync code, and explorer endpoints all read from. Each family has its own slice of keys: `bitcoinConfig` brings in BIP44 lookahead defaults and explorer endpoints; `evmConfig` brings in chain-id-keyed RPC and indexer URLs (Alpaca, Etherscan-shaped explorers, gas oracles); `solanaConfig` brings in cluster RPC URLs and validator settings. `appConfig` brings in the Manager-app catalogue tunables.

`countervaluesConfig` is two CLI-specific knobs the apps-config schema does not own:

- `config_countervalues_refreshRate` — how often live-common refreshes fiat conversions. 60 seconds. We do not actually print countervalues from the CLI, but the bridge sync pulls them anyway, so we set a sane default.
- `config_countervalues_marketCapBatchingAfterRank` — at what market-cap rank live-common switches to batched-by-id queries (cheaper, slightly less fresh). 20.

The merged `walletCliConfig` is consumed twice in `live-common-setup.ts`, once on each of the **two** `LiveConfig.instance` singletons (5.4.5).

**What this file does NOT contain:**

- **No env-var reads.** wallet-cli does not have a "config from environment" pattern the way the desktop app does (`LEDGER_DEBUG=1`, `EXPERIMENTAL_PROVIDER_OVERRIDE`, etc.). The only env vars wallet-cli reads are:
  - `USER_ID` — read in `live-common-setup.ts` to seed the DMK firmware-distribution salt; defaults to `"wallet-cli"` if unset.
  - `WALLET_CLI_MOCK_DMK`, `WALLET_CLI_MOCK_DMK_STATE`, `WALLET_CLI_MOCK_APP_RESULTS` — read by `register-dmk-transport.ts` and `mock-dmk.ts` to install the test transport (see the second half of this chapter).
  - `XDG_STATE_HOME` — read by `session-store.ts` to locate `session.yaml`.
  - `DEBUG=wallet-cli` — read by the `debug` package, namespaced via `shared/log.ts`.
- **No endpoint overrides.** The default endpoints come from each family's own config slice. To point wallet-cli at a custom explorer or RPC, you set the live-common env via `setEnv(...)` from `@ledgerhq/live-env` — the same way the desktop and mobile apps do — but wallet-cli does not currently expose a CLI flag for that.

> **Verify:** if you find yourself wanting "wallet-cli --rpc-url=…", the right place to add it is a top-level Bunli option in a future `cli-globals.ts`, not here in `config.ts`. R1 does not document a planned design here.

### 5.4.5 live-common Setup — `src/live-common-setup.ts`

This is the file that turns a vanilla Bun process into a process that speaks live-common. It is 80 lines and worth reading in full because the comments document non-obvious bundler interactions:

```ts
import { setupCalClientStore } from "@ledgerhq/cryptoassets/cal-client";
import { setSupportedCurrencies } from "@ledgerhq/live-common/currencies/index";
import { walletCliConfig } from "./config";
import { registerCoinModules } from "@ledgerhq/live-common/coin-modules/registry";
import type { CoinModuleLoader } from "@ledgerhq/live-common/coin-modules/types";
import { setWalletAPIVersion } from "@ledgerhq/live-common/wallet-api/version";
import { WALLET_API_VERSION } from "@ledgerhq/live-common/wallet-api/constants";
import { LiveConfig } from "@ledgerhq/live-config/LiveConfig";
import { setEnv } from "@ledgerhq/live-env";
import { registerWalletCliDmkTransport } from "./device/register-dmk-transport";

/**
 * Ensure USER_ID is set so DMK firmware distribution salt is stable for this CLI.
 */
if (!process.env.USER_ID) {
  process.env.USER_ID = "wallet-cli";
}

/**
 * Wallet-cli-specific coin-module loaders (bitcoin, evm, solana only).
 *
 * We define these inline instead of importing the shared coinModuleLoaders from live-common
 * because Bun's --compile bundler statically resolves all require() calls — even lazy ones
 * inside arrow functions — which would pull in every coin family's dependency tree (including
 * packages like @walletconnect/sign-client that break CJS/ESM interop under Bun).
 */
/* eslint-disable @typescript-eslint/no-require-imports */
const walletCliLoaders: CoinModuleLoader[] = [
  {
    family: "bitcoin",
    loadSetup: () => require("@ledgerhq/live-common/families/bitcoin/setup"),
    loadTransaction: () => require("@ledgerhq/coin-bitcoin/transaction").default,
    loadDeviceTxConfig: () => require("@ledgerhq/coin-bitcoin/deviceTransactionConfig").default,
    loadWalletApiAdapter: () =>
      require("@ledgerhq/live-common/families/bitcoin/walletApiAdapter").default,
    loadPlatformAdapter: () =>
      require("@ledgerhq/live-common/families/bitcoin/platformAdapter").default,
    loadAccount: () => require("@ledgerhq/coin-bitcoin/account").default,
  },
  {
    family: "evm",
    loadSetup: () => require("@ledgerhq/live-common/families/evm/setup"),
    loadTransaction: () => require("@ledgerhq/coin-evm/transaction").default,
    loadDeviceTxConfig: () => require("@ledgerhq/coin-evm/deviceTransactionConfig").default,
    loadWalletApiAdapter: () =>
      require("@ledgerhq/live-common/families/evm/walletApiAdapter").default,
    loadPlatformAdapter: () =>
      require("@ledgerhq/live-common/families/evm/platformAdapter").default,
    loadValidateAddress: () => require("@ledgerhq/coin-evm/logic/validateAddress").validateAddress,
  },
  {
    family: "solana",
    loadSetup: () => require("@ledgerhq/live-common/families/solana/setup"),
    loadTransaction: () => require("@ledgerhq/coin-solana/transaction").default,
    loadDeviceTxConfig: () => require("@ledgerhq/coin-solana/deviceTransactionConfig").default,
    loadWalletApiAdapter: () =>
      require("@ledgerhq/live-common/families/solana/walletApiAdapter").default,
  },
];

setWalletAPIVersion(WALLET_API_VERSION);
registerCoinModules(walletCliLoaders);
setSupportedCurrencies(["bitcoin", "ethereum", "solana"]);
// Set config on the ESM singleton (used by alpacaized families like EVM whose
// bridge code is reached through ESM imports).
LiveConfig.setConfig(walletCliConfig);
// Also set on the CJS singleton — Bun's bundler resolves ESM imports to lib-es/
// and require() to lib/, creating separate LiveConfig.instance singletons.
// Non-alpacaized families (solana, bitcoin) load their bridge via require() in
// the lazy loaders above, so they read from the CJS instance.
require("@ledgerhq/live-config/LiveConfig").LiveConfig.setConfig(walletCliConfig);
// TODO: wallet-cli should own its Redux store setup (createRtkCryptoAssetsStore + RTK middleware)
// instead of relying on setupCalClientStore from @ledgerhq/cryptoassets/cal-client (test-helpers).
setupCalClientStore();
// Also require() — the ESM import would set the flag on a different module instance
// than the CJS setup.ts loaded by the lazy loaders above.
require("@ledgerhq/live-common/families/solana/setup").setSolanaLdmkEnabled(true);
registerWalletCliDmkTransport();

setEnv("LEDGER_CLIENT_VERSION", "wallet-cli/0.1.0");
```

Six things to call out, because at least three of them are surprising on first read:

1. **`USER_ID` defaulting.** DMK derives a "firmware distribution salt" from `UserHashService.compute(userId)` (see `src/device/dmk.ts`). If two users run wallet-cli with the same `USER_ID`, they get the same salt, and Ledger's firmware-rollout server treats them as the same install for canary purposes. Defaulting to `"wallet-cli"` makes the CLI's identity stable across machines, which is what we want for QA: deterministic firmware-rollout buckets, not per-developer randomness.

2. **Inline `walletCliLoaders` instead of importing the shared catalog.** The comment is the rationale: live-common's shared `coinModuleLoaders` includes every family the monorepo supports, and Bun's `--compile` bundler statically resolves every `require()` it sees — even ones inside arrow functions — pulling in things like `@walletconnect/sign-client` that have CJS/ESM interop bugs under Bun. By defining the loaders inline with only the three families wallet-cli needs, the binary stays small and avoids the broken transitive deps. **If you ever need to support a new family** (e.g. Aptos, Cardano), you add a fourth entry here, not in live-common.

3. **The dual `LiveConfig.setConfig` call.** This is the most counter-intuitive line in the file. Live-common is published with both ESM (`lib-es/`) and CJS (`lib/`) builds. Bun's bundler resolves `import { LiveConfig } from "@ledgerhq/live-config/LiveConfig"` to the ESM build, but our coin-module loaders use `require(...)` which resolves to the CJS build. Both builds have a `class LiveConfig { static instance: …; static setConfig(...) }`, and **the ESM and CJS modules are different module instances with separate static fields**. So `LiveConfig.setConfig(walletCliConfig)` only writes to one of them. We do it twice — once on the ESM import and once via `require("@ledgerhq/live-config/LiveConfig").LiveConfig.setConfig(...)` — to guarantee both instances see the config. Without this, EVM (alpacaized, ESM path) sees the config but Bitcoin and Solana (CJS path) do not.

4. **`setSolanaLdmkEnabled(true)` via `require`.** Same root cause: the Solana setup module loaded by the lazy CJS loader is a different instance from the ESM-imported one. We force-enable the LDMK signing path on the CJS instance because that is the one the bridge actually consumes when signing.

5. **`setupCalClientStore()`** is acknowledged as a wart by the inline TODO. CAL ("Crypto Asset Library") is the live-common store that holds token metadata. The `cal-client` helper sets up a minimal Redux store for it; it lives in `@ledgerhq/cryptoassets/cal-client` because that is where the test helpers live. The TODO says wallet-cli should own its CAL store setup directly via `createRtkCryptoAssetsStore` instead of borrowing the test-helper. For now, we borrow.

6. **`setEnv("LEDGER_CLIENT_VERSION", "wallet-cli/0.1.0")`** stamps every outgoing live-common HTTP request with a `User-Agent`-style identifier. This is how Ledger's explorers and indexers recognise wallet-cli traffic in their logs, and is the one place in the bootstrap where the CLI announces itself.

After all of that, **`registerWalletCliDmkTransport()`** registers a live-common transport module under the device id `"wallet-cli-dmk"` (the constant `WALLET_CLI_DMK_DEVICE_ID` exported from `register-dmk-transport.ts`). Anywhere in live-common code that does `withDevice("wallet-cli-dmk")(observable)` ends up using DMK over node-WebUSB. We dissect that wiring in 5.4.8 in the second half.

### 5.4.6 The Commands Directory

`src/commands/` is the surface area of the CLI. Eight leaf commands and two groups, all picked up by `bunli generate` and exposed at the top of `wallet-cli --help`. Before walking each one, two cross-cutting helpers matter for every command file in the tree.

#### `commands/inputs.ts` — the shared option helpers (52 lines)

Every leaf command imports from this file. It owns the canonical `--account` flag, the canonical `--output` flag, and the function that turns a session label into a real account descriptor:

```ts
import { option } from "@bunli/core";
import { z } from "zod";
import { OutputFormatSchema, parseAccountDescriptor } from "../wallet/models";
import type { AccountDescriptor } from "../wallet/models";
import { parseV1 } from "../shared/accountDescriptor";
import type { AccountDescriptorV1 } from "../shared/accountDescriptor";
import { Session } from "../session/session-store";

export const accountOption = option(z.string().min(1).optional(), {
  description: "Account descriptor or session label (e.g. ethereum-1). Can also be the first positional arg.",
  short: "a",
});

export const outputOption = option(OutputFormatSchema.default("human"), {
  description: "Output format: human (default) or json",
});

export function resolveAccountArg(
  account: string | undefined,
  positional: readonly string[],
): string {
  const arg = account ?? positional[0];
  if (!arg) {
    throw new Error(
      "Missing account: use --account <descriptor-or-label> or pass it as the first positional argument.",
    );
  }
  return arg;
}

// Contains ":" → descriptor passthrough; no ":" → session label lookup.
export async function resolveAccountInput(input: string): Promise<string> {
  if (input.includes(":")) return input;
  const session = await Session.read();
  const entry = session.accounts.find(e => e.label === input);
  if (!entry) {
    throw new Error(
      `No account labeled "${input}" in session. Run \`account discover\` first or pass a full descriptor.`,
    );
  }
  return entry.descriptor;
}

/** Resolve to a V0 AccountDescriptor. Accepts V1 string, V0 string, or session label. */
export async function resolveAccountDescriptor(input: string): Promise<AccountDescriptor> {
  return parseAccountDescriptor(await resolveAccountInput(input));
}

/** Resolve to a V1 AccountDescriptorV1. Accepts V1 string or session label. */
export async function resolveAccountDescriptorV1(input: string): Promise<AccountDescriptorV1> {
  return parseV1(await resolveAccountInput(input));
}
```

Three helpers, one rule each:

- **`accountOption` / `outputOption`** are the standard flags every command uses. Defining them once means every command has the same `--account/-a` short alias and the same `--output human|json` semantics. Drift is impossible.
- **`resolveAccountArg(flags.account, positional)`** lets users pass the account either as `--account ethereum-1` or as the first positional: `wallet-cli balances ethereum-1`. Both work everywhere.
- **`resolveAccountInput(input)`** is the **session label resolver**. The rule is mechanical: if the string contains a colon (`:`) it is a full descriptor (V0 `js:2:…` or V1 `account:1:…`) and we pass it through unchanged. Otherwise it is a label like `ethereum-1` or `bitcoin-native-1`, and we look it up in `session.yaml`. If we cannot find it, the error message points the user at `account discover` — the only command that populates the session.

> **R1 reports this file as 53 lines; the on-disk count is 52 (no trailing-newline difference between the snapshot taken at research time and now). Treat both as accurate to within one line.**

The leaf commands all follow the same shape: a `defineCommand({ name, description, options, handler })` default export, options keyed by flag name, and a handler that builds a `ctx` object, creates a `CommandOutput`, runs `out.run(async () => …)`, and dispatches to `WalletAdapter`. We walk each one in turn.

#### Command 1 — `balances` (top-level, no device)

| Field | Value |
|---|---|
| File | `src/commands/balances.ts` |
| Lines | 34 |
| Group | (top-level) |
| Flags | `--account/-a <descriptor-or-label>`, `--output human\|json` |
| Device | **No** |
| Handler signature | `async ({ flags, positional }) => void` |
| Dispatches to | `WalletAdapter.getAccountBalances(descriptor)` |
| Output shape | `{ balances: [{ asset, amount, accountId }, …] }` envelope |

This is the simplest device-free command — and the canonical model for any new "fetch some data" command. Verbatim:

```ts
import { defineCommand } from "@bunli/core";
import { WalletAdapter } from "../wallet";
import { networkStringFromCurrencyId } from "../shared/accountDescriptor";
import { walletCliDebug } from "../shared/log";
import { createCommandOutput } from "../output";
import { accountOption, outputOption, resolveAccountArg, resolveAccountDescriptor } from "./inputs";

export default defineCommand({
  name: "balances",
  description: "Fetch native and token balances for an account descriptor (no device required)",
  options: {
    account: accountOption,
    output: outputOption,
  },
  handler: async ({ flags, positional }) => {
    const ctx = { command: "balances", network: "", account: "" };
    const out = createCommandOutput(flags.output, ctx);

    await out.run(async () => {
      const descriptor = await resolveAccountDescriptor(resolveAccountArg(flags.account, positional));
      ctx.network = networkStringFromCurrencyId(descriptor.currencyId);
      ctx.account = descriptor.id;
      walletCliDebug(`balances: account=${descriptor.id}, output=${flags.output}`);
      const wallet = new WalletAdapter();

      const balances = await out.withActivity(
        `Fetching balances for ${ctx.network}…`,
        "Balances fetched",
        () => wallet.getAccountBalances(descriptor),
      );
      await out.balances(balances);
    });
  },
});
```

The shape to internalise:

- The `ctx` object is mutable on purpose: `network` and `account` are filled in **after** descriptor resolution, so that any error envelope thrown later carries them.
- `out.run(fn)` is the unified error-catching wrapper. In human mode it re-throws (Bunli prints the stack); in JSON mode it catches, writes the error envelope, and `process.exit(1)`s.
- `out.withActivity(start, success, fn)` is the spinner wrapper. In human mode it shows `⠋ Fetching balances for ethereum:main…` on stderr while the bridge sync runs, then flips to `✓ Balances fetched` on success. In JSON mode it is a no-op around `await fn()`.
- `WalletAdapter.getAccountBalances` routes to Alpaca for EVM and to the live-common bridge for everything else (5.4.10 in the second half).

Sample real invocation:

```bash
$ wallet-cli balances ethereum-1 --output json
{"status":"success","command":"balances","network":"ethereum:main","account":"js:2:ethereum:0x…:","balances":[{"asset":"ethereum","amount":"0.124 ETH","accountId":"account:1:address:ethereum:main:0x…:m/44h/60h/0h/0/0"}],"timestamp":"2026-04-27T09:03:11.044Z"}
```

#### Command 2 — `operations` (top-level, no device)

| Field | Value |
|---|---|
| File | `src/commands/operations.ts` |
| Lines | 45 |
| Group | (top-level) |
| Flags | `--account/-a`, `--limit/-l <int≥1>`, `--cursor <str>`, `--output` |
| Device | **No** |
| Handler signature | `async ({ flags, positional }) => void` |
| Dispatches to | `WalletAdapter.getAccountOperations(descriptor, { limit, cursor })` |
| Output shape | `{ operations: […], nextCursor: string\|null }` envelope |

Verbatim handler:

```ts
handler: async ({ flags, positional }) => {
  const ctx = { command: "operations", network: "", account: "" };
  const out = createCommandOutput(flags.output, ctx);
  const wallet = new WalletAdapter();

  await out.run(async () => {
    const descriptor = await resolveAccountDescriptor(resolveAccountArg(flags.account, positional));
    ctx.network = networkStringFromCurrencyId(descriptor.currencyId);
    // accountId remapped to V1 descriptor — internal live-common id is intentionally dropped
    ctx.account = serializeV1(toV1(descriptor));
    walletCliDebug(
      `operations: account=${descriptor.id}, limit=${flags.limit ?? "default"}, output=${flags.output}`,
    );
    const page = await out.withActivity(
      `Fetching operations for ${ctx.network}…`,
      "Operations fetched",
      () => wallet.getAccountOperations(descriptor, { limit: flags.limit, cursor: flags.cursor }),
    );
    await out.operations(page.operations, descriptor.currencyId, page.nextCursor);
  });
},
```

Two things worth knowing:

- **The `ctx.account` for operations is the V1 descriptor**, not live-common's internal id. Every other command uses `descriptor.id` (the V0 internal id); operations specifically remaps to V1 because the operation list it returns also exposes V1-shaped account ids, and we want the envelope's `account` field consistent with the rows.
- **`--limit` and `--cursor` are documented "Alpaca families only".** R1 reports that today the `WalletAdapter.getAccountOperations` path always falls back to a full bridge sync (the Alpaca operations path is bypassed because of correctness issues with internal ops and pagination). So `--cursor` is currently inert and `--limit` is a slice of the full bridge result. The flags are kept on the surface so callers can rely on them once the Alpaca path is re-enabled.

The output includes **internal operations** (EVM contract-call traces, marked with `parentId`) and **token operations** from sub-accounts (ERC-20 movements on EVM). That is a deliberate design choice — the CLI is a research and triage tool, not a Live wallet UI; surfacing internal ops is useful when you are debugging a contract interaction.

#### Command 3 — `receive` (top-level, device unless `--verify=false`)

| Field | Value |
|---|---|
| File | `src/commands/receive.ts` |
| Lines | 48 |
| Group | (top-level) |
| Flags | `--account/-a`, `--verify/-v <bool, default true>`, `--output` |
| Device | **Required when `--verify=true` (default)**, no device when `--verify=false` |
| Handler signature | `async ({ flags, positional }) => void` |
| Dispatches to | `WalletAdapter.verifyAddress(descriptor, deviceId)` (verify path) or `WalletAdapter.getFreshAddress(descriptor)` (no-verify path) |
| Output shape | `{ address: "0x…" }` envelope |

Verbatim handler, with the device-vs-no-device split visible in the `if (flags.verify)` branch:

```ts
handler: async ({ flags, positional }) => {
  const ctx = { command: "receive", network: "", account: "" };
  const out = createCommandOutput(flags.output, ctx);
  const wallet = new WalletAdapter();


  await out.run(async () => {
    const descriptor = await resolveAccountDescriptor(resolveAccountArg(flags.account, positional));
    ctx.network = networkStringFromCurrencyId(descriptor.currencyId);
    ctx.account = descriptor.id;
    if (flags.verify) {
      const spin = out.spin(`Connect device and open ${colors.bold(descriptor.currencyId)} app…`);
      await withCurrencyDeviceSession(descriptor.currencyId, async () => {
        spin?.success("Device session established");
        const verifySpin = out.spin("Confirm address on your Ledger device…");
        const address = await wallet.verifyAddress(descriptor, WALLET_CLI_DMK_DEVICE_ID);
        verifySpin?.success("Address confirmed on device");
        out.address(address);
      });
    } else {
      out.address(await wallet.getFreshAddress(descriptor));
    }
  });
},
```

This is the first command in the tour that opens a device session, so the pattern is worth fixing in your head:

1. **`out.spin(text)`** is the device-prompt spinner. In an interactive TTY it shows `⠋ Connect device and open ethereum app…` on stderr until you plug in your Ledger and unlock it. In a non-interactive context (CI, test harness, AI agent shell) `out.spin()` returns `undefined` and the conditional `spin?.success(…)` calls are no-ops.
2. **`withCurrencyDeviceSession(currencyId, fn)`** is the device-session boundary, defined in `src/session/bridge-device-session.ts`. It opens a DMK USB session, asks the device to open the right Manager app for the currency (Ethereum app for `ethereum`, Bitcoin app for `bitcoin`, Solana app for `solana`), runs your callback, and tears down the session in a `finally`. Errors out of the callback are mapped to user-friendly messages by `toWalletCliDeviceError(...)`.
3. **`WALLET_CLI_DMK_DEVICE_ID = "wallet-cli-dmk"`** is the constant the live-common bridge consumes when it does `withDevice(deviceId)`. The transport module registered in 5.4.5 is the one that responds to that id.

`--verify=false` skips all of that and just returns the next unused fresh address from the bridge. For EVM and Solana this is a no-op (the address is in the descriptor); for Bitcoin it triggers a bridge sync to find the next unused receive address. This is exactly the same logic as `account fresh-address`, just exposed under a friendlier flag.

#### Command 4 — `send` (top-level, device unless `--dry-run`)

| Field | Value |
|---|---|
| File | `src/commands/send.ts` |
| Lines | 158 |
| Group | (top-level) |
| Flags | see the table below |
| Device | **Required unless `--dry-run`** |
| Handler signature | `async ({ flags, positional }) => void` |
| Dispatches to | `WalletAdapter.prepareSend(descriptor, intent)` (dry-run) or `WalletAdapter.send(descriptor, intent, deviceId, dryRun)` (Observable<SendEvent>) |
| Output shape | `{ events: [SendEvent, …], txHash: string }` envelope (collected from the Observable) |

`send` is the largest leaf command and the one you will edit most often, so here is the full flag set verbatim from the source (names, schemas, descriptions, required flags):

| Flag | Schema | Required | Description |
|---|---|---|---|
| `--account/-a <str>` | `z.string().min(1).optional()` | optional | Account descriptor or session label |
| `--to/-t <addr>` | `z.string().min(1, "Recipient address is required (--to <address>)")` | **yes** | Recipient address |
| `--amount <"V TICKER">` | `z.string().min(1, "Amount is required (--amount '<value> <TICKER>', e.g. '0.01 ETH')")` | **yes** | Amount including ticker (e.g. `0.001 BTC`, `0.01 ETH`, `0.4 USDT`). The ticker drives asset resolution. |
| `--fee-per-byte <sat>` | `z.string().min(1).optional()` | bitcoin | Fee per byte in satoshis |
| `--rbf` | `z.boolean().optional()` | bitcoin | Enable Replace-By-Fee |
| `--mode <m>` | `z.string().min(1).optional()` | solana | `send`, `stake.createAccount`, `stake.delegate`, `stake.undelegate`, `stake.withdraw` |
| `--validator <addr>` | `z.string().min(1).optional()` | solana | Validator address (staking only) |
| `--stake-account <addr>` | `z.string().min(1).optional()` | solana | Stake account address (staking only) |
| `--memo <text>` | `z.string().min(1).optional()` | solana | Memo / tag |
| `--data <0x…>` | `z.string().regex(/^0x([0-9a-fA-F]{2})*$/).optional()` | **evm** | EVM calldata as 0x-prefixed hex (e.g. `0xd0e30db0`) |
| `--dry-run` | `z.boolean().default(false)` | optional | Prepare and validate transaction but do not sign or broadcast |
| `--output` | `OutputFormatSchema.default("human")` | optional | Output format |

The handler builds a per-family `TransactionIntent`, validates it with Zod, and then either calls `wallet.prepareSend` (dry-run) or subscribes to the `wallet.send` Observable inside a `withCurrencyDeviceSession`:

```ts
const INTENT_BUILDERS: Record<string, IntentBuilder> = {
  bitcoin: flags => ({
    family: "bitcoin",
    recipient: flags.to,
    amount: flags.amount,
    feePerByte: flags["fee-per-byte"],
    rbf: flags.rbf,
  }),
  evm: flags => ({
    family: "evm",
    recipient: flags.to,
    amount: flags.amount,
    data: flags.data,
  }),
  solana: flags => ({
    family: "solana",
    recipient: flags.to,
    amount: flags.amount,
    mode: flags.mode,
    validator: flags.validator,
    stakeAccount: flags["stake-account"],
    memo: flags.memo,
  }),
};
```

Three things to internalise:

- **The intent is a discriminated union on `family`.** `TransactionIntentSchema` (in `src/wallet/intents/index.ts`) is `z.discriminatedUnion("family", [BitcoinIntent, EvmIntent, SolanaIntent])`. The Zod parse on line `TransactionIntentSchema.parse(intentData)` is a real validation gate — if you pass `--data 0xZZZ` to a Bitcoin send it fails right there with a typed Zod error.
- **The `--data` flag already exists for EVM.** This is the load-bearing detail for the QAA-615 token-revoke spike (covered in 5.4.13/5.4.14 in the second half): a `revoke` command can be a thin wrapper that builds `--data 0x095ea7b3${spender_pad32}${zeros_pad32}` and calls `send` against the token contract with `--amount '0 <TICKER>'`. No new device or APDU code, no new bridge plumbing — just intent-building.
- **The `wallet.send(...)` return type is `Observable<SendEvent>`.** The handler subscribes via `lastValueFrom(observable.pipe(tap(event => out.sendEvent(event))))`. Every event hits `out.sendEvent` synchronously: in human mode each one updates the spinner; in JSON mode they are buffered and emitted as a single `events` array on completion.

The `SendEvent` discriminated union (`wallet/models.ts`):

```ts
type SendEvent =
  | { type: "prepared"; recipient: string; amount: string; fees: string }
  | { type: "device-streaming"; progress: number; index: number; total: number }
  | { type: "device-signature-requested" }
  | { type: "device-signature-granted" }
  | { type: "dry-run" }
  | { type: "broadcasted"; txHash: string };
```

Sample dry-run invocation:

```bash
$ wallet-cli send ethereum-1 --to 0xRecipient… --amount '0.001 ETH' --dry-run --output json
{"status":"success","command":"send","network":"ethereum:main","account":"js:2:ethereum:0x…:","prepared":{"recipient":"0xRecipient…","amount":"0.001 ETH","fees":"0.000042 ETH"},"timestamp":"2026-04-27T…"}
```

#### Command 5 — `account discover` (group `account`, device required)

| Field | Value |
|---|---|
| File | `src/commands/account/discover.ts` |
| Lines | 88 |
| Group | `account` |
| Flags | `--network/-n <string>`, `--output` |
| Positional | First arg used as network if `--network` absent |
| Device | **Required** (USB) |
| Handler signature | `async ({ flags, positional }) => void` |
| Dispatches to | `WalletAdapter.discoverAccounts(network, deviceId): Observable<DiscoveredAccount>` |
| Output shape | `{ accounts: [V1-descriptor-string, …] }` envelope; also persists labels to `session.yaml` |

Discover is the "front door" — every workflow starts here, because it is the only command that populates the session label store. The handler:

```ts
handler: async ({ flags, positional }) => {
  const networkArg = flags.network ?? positional[0];
  if (!networkArg) {
    throw new Error(
      'Missing network: use --network <name> or -n <name>, e.g. "bitcoin", "ethereum:goerli".',
    );
  }

  const network = parseNetworkArg(networkArg);
  const currencyId = currencyIdFromNetwork(network);
  const networkStr = `${network.name}:${network.env}`;
  walletCliDebug(`account discover: network=${networkStr}, output=${flags.output}`);

  const out = createCommandOutput(flags.output, {
    command: "account discover",
    network: networkStr,
  });

  await out.run(async () => {
    const spin = out.spin(`Connect device and open ${colors.bold(network.name)} app…`);
    const discoveredDescriptors: AccountDescriptorV1[] = [];

    await withCurrencyDeviceSession(currencyId, async () => {
      spin?.success("Device session established");

      const wallet = new WalletAdapter();
      const scanSpin = out.spin(`Scanning for ${colors.bold(network.name)} accounts…`);
      let count = 0;

      await new Promise<void>((resolve, reject) => {
        wallet.discoverAccounts(network, WALLET_CLI_DMK_DEVICE_ID).subscribe({
          next: d => {
            out.discoveredAccount(d);
            discoveredDescriptors.push(d.descriptor);
            count++;
            if (scanSpin) scanSpin.text = `Scanning… (${count} found so far)`;
          },
          error: err => {
            scanSpin?.error("Scan failed");
            reject(err);
          },
          complete: () => {
            scanSpin?.success(`Found ${count} account${count === 1 ? "" : "s"}`);
            resolve();
          },
        });
      });

      out.flushDiscovery();

      let added = 0;
      try {
        const session = await Session.read();
        added = session.addDescriptors(discoveredDescriptors);
        if (added > 0) await session.write();
      } catch {
        // Session persistence failure is non-fatal; discovery output is already flushed
      }

      if (added > 0) out.sessionSaved(added);
    });
  });
},
```

What to internalise:

1. **Network parsing.** `parseNetworkArg("ethereum")` → `{ name: "ethereum", env: "main" }`; `parseNetworkArg("ethereum:goerli")` → `{ name: "ethereum", env: "goerli" }`. The alias `mainnet → main` is applied here. `currencyIdFromNetwork({name, env})` is the inverse of `networkFromCurrencyId(currencyId)` from the descriptor module (5.4.7 in the second half).
2. **Two spinners.** The first one prompts the user to connect-and-unlock the device. The second one updates live with the running discovered-count. In JSON mode both are no-ops.
3. **The Observable subscribe pattern is wrapped in a Promise** because `withCurrencyDeviceSession` expects a Promise-returning callback. The `next` handler streams each discovered account to `out.discoveredAccount(d)` for the human/JSON formatter, and accumulates `d.descriptor` (V1) into `discoveredDescriptors` for the session-save step.
4. **Session persistence is wrapped in `try { … } catch { /* non-fatal */ }`.** Discovery output has already been printed by `out.flushDiscovery()`; if YAML write fails, the user still got their accounts. They just have to re-run discovery before using labels.
5. **`session.addDescriptors(...)` dedupes by serialized V1**, so re-running discovery on the same device after adding a new account on it surfaces `Added N accounts` only for the genuinely new ones.

JSON output envelope:

```json
{
  "status": "success",
  "command": "account discover",
  "network": "ethereum:main",
  "accounts": [
    "account:1:address:ethereum:main:0x71C7656EC7ab88b098defB751B7401B5f6d8976F:m/44h/60h/0h/0/0",
    "account:1:address:ethereum:main:0x…:m/44h/60h/0h/0/1"
  ],
  "timestamp": "2026-04-27T…"
}
```

Sample real invocation:

```bash
$ wallet-cli account discover ethereum
[~] Waiting: Ledger detected but locked. Enter your PIN on the device.
[i] Opening Ethereum app on your Ledger... ACTION REQUIRED: Confirm on device screen.
✓ Device session established
✓ Found 3 accounts
[+] Saved 3 new accounts to session as: ethereum-1, ethereum-2, ethereum-3
```

#### Command 6 — `account fresh-address` (group `account`, no device)

| Field | Value |
|---|---|
| File | `src/commands/account/fresh-address.ts` |
| Lines | 38 |
| Group | `account` |
| Flags | `--account/-a <descriptor-or-label>`, `--output` |
| Device | **No** |
| Handler signature | `async ({ flags, positional }) => void` |
| Dispatches to | `WalletAdapter.getFreshAddress(toV0(v1))` for utxo descriptors; reads `v1.address` directly for address descriptors |
| Output shape | `{ address: "…" }` envelope |

The handler is the cleanest example of the V1-descriptor branching:

```ts
handler: async ({ flags, positional }) => {
  const ctx = { command: "account fresh-address", network: "", account: "" };
  const out = createCommandOutput(flags.output, ctx);
  const wallet = new WalletAdapter();


  await out.run(async () => {
    const v1 = await resolveAccountDescriptorV1(resolveAccountArg(flags.account, positional));
    ctx.network = serializeNetwork(v1.network);
    ctx.account = serializeV1(v1);

    // address-based (EVM, Solana…): no sync needed, address is in the descriptor
    // utxo (Bitcoin…): next unused address requires a blockchain scan
    const address =
      v1.type === "address"
        ? v1.address
        : await out.withActivity(
            `Scanning ${v1.network.name} blockchain for fresh address…`,
            "Fresh address resolved",
            () => wallet.getFreshAddress(toV0(v1)),
          );
    out.address(address);
  });
},
```

Two flavours:

- **`v1.type === "address"`** (EVM, Solana, anything where the descriptor carries the address): no I/O at all. The address is `v1.address`. The command returns instantly.
- **`v1.type === "utxo"`** (Bitcoin and other UTXO chains): the descriptor only carries the xpub, not the next-unused address. We have to call `wallet.getFreshAddress(toV0(v1))`, which kicks off a bridge sync to scan the chain for unused output indices.

This is the only leaf command that resolves to a V1 descriptor (`resolveAccountDescriptorV1`) instead of the V0 descriptor (`resolveAccountDescriptor`), because the V1/V0 type discriminator is exactly what lets us decide whether a sync is needed.

#### Command 7 — `session view` (group `session`, no device)

| Field | Value |
|---|---|
| File | `src/commands/session/view.ts` |
| Lines | 20 |
| Group | `session` |
| Flags | `--output` |
| Device | **No** |
| Handler signature | `async ({ flags }) => void` |
| Dispatches to | `Session.read()` |
| Output shape | `{ accounts: [{ label, descriptor }, …] }` envelope |

The simplest leaf command, verbatim:

```ts
export default defineCommand({
  name: "view",
  description: "Display all accounts stored in the current session",
  options: {
    output: outputOption,
  },
  handler: async ({ flags }) => {
    const out = createCommandOutput(flags.output, { command: "session view", network: "all" });

    await out.run(async () => {
      const { accounts } = await Session.read();
      out.sessionView(accounts);
    });
  },
});
```

Note `network: "all"` in the context — the session is currency-agnostic, so no single network applies. `out.sessionView(accounts)` renders an aligned `label  descriptor` table in human mode, or a `[{ label, descriptor }, …]` array in JSON mode.

Sample invocation:

```bash
$ wallet-cli session view
ethereum-1        account:1:address:ethereum:main:0x71C…:m/44h/60h/0h/0/0
ethereum-2        account:1:address:ethereum:main:0x9aF…:m/44h/60h/0h/0/1
bitcoin-native-1  account:1:utxo:bitcoin:main:xpub6BosfCn…:m/84h/0h/0h
```

#### Command 8 — `session reset` (group `session`, no device)

| Field | Value |
|---|---|
| File | `src/commands/session/reset.ts` |
| Lines | 28 |
| Group | `session` |
| Flags | `--output` |
| Device | **No** |
| Handler signature | `async ({ flags }) => void` |
| Dispatches to | `Session.read()` (with corrupt-file fallback to `Session.from([])`), `session.clear()`, `session.write()` |
| Output shape | `{ cleared: <count> }` envelope |

Verbatim:

```ts
handler: async ({ flags }) => {
  const out = createCommandOutput(flags.output, { command: "session reset", network: "all" });

  await out.run(async () => {
    let session: Session;
    try {
      session = await Session.read();
    } catch {
      // Corrupt session file — treat as empty and overwrite below
      session = Session.from([]);
    }
    const count = session.clear();
    await session.write(); // always write: fixes corrupt files too
    out.sessionReset(count);
  });
},
```

The interesting bit is the `try/catch` around `Session.read()`. The session store's `read()` throws on YAML parse errors with a message that points the user at `session reset`. So `session reset` itself has to handle the parse-failure case: it falls back to an empty session via `Session.from([])`, then writes — overwriting the corrupt file with a valid empty document. After `session reset`, `session view` always succeeds.

Sample invocation:

```bash
$ wallet-cli session reset --output json
{"status":"success","command":"session reset","network":"all","cleared":5,"timestamp":"2026-04-27T…"}
```

#### The two groups — `account` and `session`

The remaining two entries in the `commands.gen.ts` `names` tuple are **groups**, not leaves. They exist to give the help text a tree shape:

```ts
// src/commands/account/index.ts  (9 lines)
import { defineGroup } from "@bunli/core";
import DiscoverCommand from "./discover";
import FreshAddressCommand from "./fresh-address";

export default defineGroup({
  name: "account",
  description: "Account management commands",
  commands: [DiscoverCommand, FreshAddressCommand],
});
```

```ts
// src/commands/session/index.ts  (9 lines)
import { defineGroup } from "@bunli/core";
import ViewCommand from "./view";
import ResetCommand from "./reset";

export default defineGroup({
  name: "session",
  description: "Session management commands",
  commands: [ViewCommand, ResetCommand],
});
```

`defineGroup` does not have a `handler` — it just collects subcommands. Running `wallet-cli account` with no subcommand prints the group help; running `wallet-cli account discover` resolves through this index file to `discover.ts`. Bunli's codegen records both the group entry (with `commands: [...]`) and the leaf entries (each with their own metadata) in `commands.gen.ts`, which is why the `names` tuple has 10 entries for 8 leaves.

> **R1 vs SKILL.md:** R1 says "10 commands"; an older SKILL.md draft said "5 commands". R1 is fresher (April 2026 snapshot) and is correct for the current tree. The mismatch is the original 5-command shape (`balances`, `operations`, `receive`, `send`, plus one device check) before the `account` and `session` groups were added.

That covers every entry under `src/commands/`. The second half of this chapter picks up the device layer, the wallet adapter, the session store, the test harness, and the recurring patterns you should copy when you add a new command of your own.

<!-- continued in _part5_ch5_4_part2.md -->
### 5.4.7 The wallet directory

`src/wallet/` is the only place in the codebase that knows what an "account" is. Everything below it (DMK, transports, USB) deals in bytes; everything above it (commands, output formatting) deals in string envelopes. The wallet layer is the bridge between the two, and it is deliberately small.

Three files carry the load:

- `src/wallet/index.ts` — `WalletAdapter`, the routing class.
- `src/wallet/models.ts` — Zod schemas for the wire types (`AccountDescriptor`, `Balance`, `Operation`, `SendEvent`).
- `src/wallet/intents/` — the family-specific schemas the command layer hands down.

Two compatibility adapters live below it: `src/wallet/compatibility/bridge.ts` (the legacy live-common `AccountBridge` driver) and `src/wallet/compatibility/alpaca.ts` (a direct API client used today only for `getAccountBalances` on EVM). The `WalletAdapter` picks one of the two per call:

```ts
// apps/wallet-cli/src/wallet/index.ts
export class WalletAdapter {
  private static readonly alpacaFamilies = new Set(["evm"]);
  private readonly bridge = new BridgeAdapter();
  private readonly alpaca = new AlpacaAdapter();

  async getAccountBalances(descriptor: AccountDescriptor): Promise<Balance[]> {
    const { family } = getCryptoCurrencyById(descriptor.currencyId);
    if (WalletAdapter.alpacaFamilies.has(family)) return this.alpaca.getBalances(descriptor);
    return this.bridge.getBalances(descriptor);
  }
  // …
}
```

The routing rule is one-line and the rest of the class is dispatch:

| Method | Adapter used | Notes |
|---|---|---|
| `discoverAccounts` | bridge | Always — Alpaca has no discover API |
| `getAccountBalances` | alpaca for `evm`, bridge otherwise | Fast path, no full sync |
| `getAccountOperations` | bridge | Alpaca temporarily disabled, see comment in `index.ts` |
| `getFreshAddress` / `verifyAddress` | bridge | Always |
| `prepareSend` / `send` | bridge | Always — Alpaca cannot sign |

Read `src/wallet/index.ts` end-to-end before writing a new command. It is 96 lines. The `prepareSend` / `send` split is what lets `--dry-run` skip the device entirely — `prepareSend` returns `{ amount, fees, recipient }` synchronously after a sync, while `send` returns an `Observable<SendEvent>` that drives the device through the bridge's `signOperation` → `broadcast` pipeline.

#### models.ts — the wire shape

`models.ts` defines what the rest of the binary may pass around. Three rules:

1. No `BigNumber`, no `Date`, no circular refs. Everything is JSON-serialisable.
2. `AccountDescriptor` is the legacy V0 (live-common) shape: `{ id, currencyId, freshAddress, seedIdentifier, derivationMode, index }`. The CLI exposes V1 (ADR) descriptors in its UI but converts to V0 at the boundary — see `5.4.8`.
3. `SendEvent` is a discriminated union with six variants: `prepared`, `device-streaming`, `device-signature-requested`, `device-signature-granted`, `dry-run`, `broadcasted`. Both formatters (human and JSON) switch on `event.type`.

The `Operation` schema carries a `parentId` field "set on internal operations (e.g. ETH contract calls): the id of the parent operation". The `operations` command's human formatter indents children below their parent (`writeStdout(op.parentId ? `  ${line}` : line)` in `output.ts:134`). That is the only place the `parentId` is used today.

#### Intents — the only extensibility hook for the send pipeline

The intents directory (`src/wallet/intents/`) is the API the `send` command speaks to the wallet layer. It is also — per R5 — the **only** place to extend the send pipeline without touching live-common. Memorise this; it returns in 5.8 (QAA-615).

```ts
// apps/wallet-cli/src/wallet/intents/index.ts
export const TransactionIntentSchema = z.discriminatedUnion("family", [
  BitcoinTransactionIntentSchema,
  EvmTransactionIntentSchema,
  SolanaTransactionIntentSchema,
]);
export type TransactionIntent = z.infer<typeof TransactionIntentSchema>;
```

A discriminated union on `family`. Three families today: bitcoin, evm, solana. Adding a fourth means a new file in `families/` and a new entry in this array — nothing else changes at this level.

The EVM family schema is the smallest of the three and the most consequential for QAA-615:

```ts
// apps/wallet-cli/src/wallet/intents/families/evm.ts
import { z } from "zod";
import { AmountWithTickerSchema } from "./bitcoin";

export const EvmTransactionIntentSchema = z.object({
  family: z.literal("evm"),
  recipient: z.string(),
  amount: AmountWithTickerSchema,
  data: z
    .string()
    .regex(/^0x([0-9a-fA-F]{2})*$/, "data must be 0x-prefixed hex with an even number of digits")
    .optional(),
});
```

Four fields. The `data` field is the load-bearing one: any 0x-prefixed even-length hex string is accepted, which means `send` already accepts arbitrary EVM calldata. Bitcoin's schema adds `feePerByte` and `rbf`; Solana's adds a `mode` enum (`send`, `stake.createAccount`, `stake.delegate`, `stake.undelegate`, `stake.withdraw`), `validator`, `stakeAccount`, and `memo`. They do not have a `data` field — the device-side primitives are different.

#### How `--data 0x…` flows from `send` to the device

Trace a single command-line invocation:

```bash
wallet-cli send ethereum-1 \
  --to 0xdAC17F958D2ee523a2206206994597C13D831ec7 \
  --amount '0 ETH' \
  --data 0x095ea7b3000000000000000000000000<spender>0000000000000000000000000000000000000000000000000000000000000000
```

1. **`send.ts:32-55`** — `INTENT_BUILDERS["evm"]` maps the flags object to:
   ```ts
   { family: "evm", recipient: flags.to, amount: flags.amount, data: flags.data }
   ```
2. **`send.ts:132`** — `TransactionIntentSchema.parse(intentData)` validates the shape. The `data` regex rejects malformed hex before any device work happens.
3. **`send.ts:148-152`** — `wallet.send(descriptor, intent, WALLET_CLI_DMK_DEVICE_ID, dryRun)` returns an Observable; each `SendEvent` is piped through `out.sendEvent(event)` for formatting.
4. **`WalletAdapter.send`** — delegates straight to `BridgeAdapter.send`.
5. **`BridgeAdapter.send`** — calls `buildTxExtras(family, intent)`, which for `"evm"` strips the `0x` and converts the hex into a `Buffer` that is patched onto the live-common `Transaction` (per R5 / R2: `bridge.ts:194-198`). From there the bridge owns the rest: `signOperation` (DMK signer) → `broadcast` (explorer POST).

The pedagogical point: by the time you reach the bridge, you are inside live-common; the wallet-cli itself does not call `signEthereumTransaction` directly anywhere. The intents schema is the *only* surface area you have to think about when extending `send`. It is also why `5.8` will turn out to be a small ticket and not a refactor.

> **Verify:** the `buildTxExtras` snippet (`bridge.ts:194-198`) is quoted from R2's appendix; the file lives at `apps/wallet-cli/src/wallet/compatibility/bridge.ts` and is 295 lines. If you are reading this on a checkout where the line numbers have drifted, search for the literal `case "evm":` block.

The `wallet/formatter/` subdirectory holds the two output renderers that `output.ts` orchestrates: `human.ts` (~96 lines) prints colored, indented text; `json.ts` (~58 lines) returns plain objects that `JsonCommandOutput` wraps in `makeEnvelope`. They are stateless — they take a `Balance` / `Operation` / `DiscoveredAccount` and return a string or object. The state (active spinner, accumulated send result) lives in `output.ts`.

---

### 5.4.8 Account Descriptors V1

Every command that targets an account ultimately resolves to a string of the form `account:1:…`. The format is specified in an ADR ("ADR — Account descriptor", linked from the v1.ts header) and lives in `src/shared/accountDescriptor/v1.ts`. Spend ten minutes on this section — descriptors are how you talk to the CLI in scripts, in tests, and in the QAA-613 hooks the rest of Part 5 leans on.

#### Grammar

A V1 descriptor is a colon-separated string with two shapes, distinguished by the third segment:

```
UTXO:    account:1:utxo:<network_name>:<network_env>:<xpub>:<hardened_path>
ADDRESS: account:1:address:<network_name>:<network_env>:<address>:<bip44_path>
```

Field by field:

| Position | Field | Meaning |
|---|---|---|
| 0 | `account` | Purpose discriminator. Always literal `account`. |
| 1 | `1` | Schema version. Today only `1` exists. V0 (`js:2:…`) is parsed by a separate path — see below. |
| 2 | `utxo` \| `address` | Account type. `utxo` for UTXO-based families (Bitcoin), `address` for account-based families (EVM, Solana). |
| 3 | network name | Lowercase blockchain name. e.g. `bitcoin`, `ethereum`, `base`, `solana`. |
| 4 | network env | `main` for mainnets. Otherwise the testnet/devnet suffix from the live-common currencyId — `testnet`, `goerli`, `sepolia`, `devnet`. The user-facing alias `mainnet` is normalised to `main` by `parseNetworkArg` in `network.ts:33`. |
| 5 | xpub or address | UTXO: the extended public key (`xpub`/`ypub`/`zpub`). Address: the derived address (0x… for EVM, base58 for Solana). |
| 6 | path | UTXO: hardened-only BIP32 path (regex enforced). Address: full BIP44 derivation path including non-hardened change/index. |

The two type-specific Zod schemas are in `v1.ts`:

```ts
// apps/wallet-cli/src/shared/accountDescriptor/v1.ts
export const UtxoAccountDescriptorV1Schema = z.object({
  purpose: z.literal("account"),
  version: z.literal("1"),
  type: z.literal("utxo"),
  network: NetworkSchema,
  xpub: z.string().min(1),
  path: z
    .string()
    .regex(/^m(\/\d+[h'])+$/, "UTXO path must only contain hardened segments, e.g. m/84h/0h/0h"),
});

export const AccountBasedDescriptorV1Schema = z.object({
  purpose: z.literal("account"),
  version: z.literal("1"),
  type: z.literal("address"),
  network: NetworkSchema,
  address: z.string().min(1),
  path: z.string().min(1),
});
```

The UTXO `path` regex is the only structural rule beyond "right number of colons". Address paths are accepted without a regex — `m/44h/60h/0h/0/0` is the convention but the schema only requires non-empty.

#### Hardened path notation: `h` vs `'`

A real subtlety. BIP32 traditionally writes hardened segments with an apostrophe — `m/84'/0'/0'`. That apostrophe is shell-hostile (it breaks every `bash`/`zsh` quoting rule the moment you try to paste a descriptor into a command line). The ADR-specified canonical form uses **`h`** instead — `m/84h/0h/0h` — and the regex `/^m(\/\d+[h'])+$/` accepts either character. `serializeV1` always emits `h`; `parseV1` will accept either on input.

> **Always emit `h` when generating descriptors yourself** (in test fixtures, in CI scripts, in pasted snippets). Apostrophe-form descriptors parse correctly today but make piping the output of `account discover` into a shell command much more painful than it needs to be.

#### Worked examples — all four supported families

The README and per-family schemas confirm wallet-cli ships with four families wired today: bitcoin, ethereum, base (also EVM), and solana. Per R1 the current binary's command surface supports them all. Below is one canonical descriptor for each.

```
# Bitcoin mainnet, native segwit (BIP84). UTXO descriptor.
account:1:utxo:bitcoin:main:xpub6BosfCnifzxcFwrSzQiqu2DBVTshkCXacvNsWGYJVVhhawA7d4R5WSE1S2G4UrqdKFNvJx3bR7MNfYTc4FXnAFzBVNMcJYHx5ENKnG9WNzh:m/84h/0h/0h

# Bitcoin testnet, native segwit. Note env suffix maps to currencyId bitcoin_testnet.
account:1:utxo:bitcoin:testnet:tpubD8L6U...:m/84h/1h/0h

# Ethereum mainnet, BIP44. Address descriptor.
account:1:address:ethereum:main:0x71C7656EC7ab88b098defB751B7401B5f6d8976F:m/44h/60h/0h/0/0

# Base mainnet (EVM L2). Identical shape to Ethereum.
account:1:address:base:main:0x71C7656EC7ab88b098defB751B7401B5f6d8976F:m/44h/60h/0h/0/0

# Solana mainnet, full hardened path. Address descriptor (Solana is account-based).
account:1:address:solana:main:7xCU4XQfL8589X6vVt8q5F7J3Z9T1z6W6X6X6X6X6X:m/44h/501h/0h/0h
```

A few observations:

- The `network env` suffix is **not** the live-common currencyId. `bitcoin_testnet` (the live-common id) becomes `bitcoin:testnet` (the descriptor's `name:env`). The mapping is bidirectional and lives in `network.ts:95-130` (`networkFromCurrencyId` / `currencyIdFromNetwork`).
- Base reuses the same `address` shape as Ethereum because it shares the EVM family (and the BIP44 path uses Ethereum's coin type 60). Adding a new EVM L2 to wallet-cli is a cryptoassets registration; it requires no descriptor changes here.
- Solana's path is fully hardened (`m/44h/501h/0h/0h`) — the `address` schema accepts that because the regex is only enforced on `utxo`-type paths.

#### Parsing flow

The parser (`parseV1` in `v1.ts:91`) is a deliberate split-on-colon, not a regex. It tolerates xpubs / addresses that contain colons in some hypothetical future format by joining everything between segment 4 (env) and the last segment (path) back together:

```ts
const path = rest.at(-1)!;
const xpubOrAddress = rest.slice(0, -1).join(SEP);
```

In practice no current address or xpub contains a colon, but the parser is robust to that future change. If you write a descriptor by hand, count the colons: 6 for a V1 descriptor (7 segments).

#### V0 backward compatibility

The legacy "WalletSync short form" still works as input. `wallet/models.ts:54-59` routes the input string through one of two parsers:

```ts
export function parseAccountDescriptor(input: string): AccountDescriptor {
  if (input.startsWith("account:1:")) {
    return toV0(parseV1(input));
  }
  return parseShortAccountDescriptor(input);
}
```

The V0 short form is `<accountId>:<index>` where `accountId` is the live-common-encoded id — `js:2:bitcoin:xpub…:native_segwit:0`. So the full V0 string is something like `js:2:bitcoin:xpub6BosfC…:native_segwit:0:0` (note the trailing `:0` is the index, the rest is `decodeAccountId`-friendly). You will see V0 strings in fixtures (`test/helpers/constants.ts`) and in older `session.yaml` files. New work should always emit V1.

#### Adapters: V0 ↔ V1

The `compatibility` adapters in `src/shared/accountDescriptor/adapters.ts` (212 lines per R1) provide `toV0(v1)` and `toV1(v0)`. The wallet layer (`WalletAdapter`) speaks V0 because the live-common bridge expects V0; commands resolve V1 input from the user, then convert on the way down. `account discover` does the reverse on the way up: it gets V0 descriptors back from the bridge, runs `toV1`, and prints the V1 string in its output envelope. The `session.yaml` store (5.4.x in part 1) persists V1 strings.

> **Verify:** `adapters.ts` is 212 lines per the R1 inventory; if your checkout drifts, the function names you want are `toV0` and `toV1`, exported from `src/shared/accountDescriptor/index.ts`.

---

### 5.4.9 Shared utilities

`src/shared/` is the small-utilities drawer. Three files matter outside of `accountDescriptor/`.

**`src/shared/log.ts`** (11 lines). A thin wrapper over the `debug` package. One namespace, `wallet-cli`, exposed through `walletCliDebug(message, ...args)`. Set `DEBUG=wallet-cli` (or `DEBUG=wallet-cli:*`) to enable it; without it, calls are no-ops. Used most visibly in `bridge-device-session.ts` to trace device-session lifecycle ("Ensuring DMK transport…", "Resetting device session…"). When a command hangs in CI, this is the first variable you flip on.

**`src/shared/response.ts`** (15 lines). One function, `makeEnvelope(command, network, data, account?)`. Builds the canonical JSON envelope used by every JSON-mode response:

```ts
export function makeEnvelope(
  command: string,
  network: string,
  data: Record<string, unknown>,
  account?: string,
): Record<string, unknown> {
  return {
    status: "success",
    command,
    network,
    ...(account == null ? {} : { account }),
    ...data,
    timestamp: new Date().toISOString(),
  };
}
```

That `status: "success"` and the trailing `timestamp` are the two fields every consumer can rely on — `JsonCommandOutput` uses this for all success outputs and writes a separate hand-built `{ ok: false, error: { … } }` shape for failures. So if you grep an automation script for "did this CLI invocation succeed", `.status === "success"` (or `.ok === false` for the error envelope) is the test.

**`src/shared/ui.ts`** (67 lines). Three exports: `colors` (re-exported from `@bunli/utils`), `writeStdout` (re-exported from `@bunli/utils`), and the `spinner(text)` / `withSpinner(...)` helpers around `yocto-spinner`. The interesting bit is `isInteractive()`:

```ts
export function isInteractive(): boolean {
  if (
    process.env.CLAUDECODE ||
    process.env.CLAUDE_CODE ||
    process.env.CURSOR_AGENT ||
    process.env.CODEX_ENABLED ||
    process.env.GEMINI_CLI ||
    process.env.OPENCODE ||
    process.env.AMP_CURRENT_THREAD_ID ||
    process.env.AGENT === "amp"
  )
    return false;
  return process.stderr.isTTY === true;
}
```

The spinner is suppressed when stderr is not a TTY **or** when one of seven AI-agent environment variables is set. The CLI test runner (`cli-runner.ts`) sets `CLAUDECODE=1` explicitly before spawning the binary so output is deterministic. If you are scripting wallet-cli yourself and your output looks "stripped of progress", that is intended — the spinner writes to stderr only in interactive mode, never to stdout.

---

### 5.4.10 Session layer

The file is `src/session/bridge-device-session.ts` (48 lines). The name is doing a lot of work and it is *not* what you might guess.

> **Disambiguation.** This is **not** the WebSocket "bridge" from Part 4 (the mobile Detox bridge that ferries Redux messages between the Jest worker and the RN app). That was a network protocol. The "bridge" in `bridge-device-session.ts` is the live-common `AccountBridge` — the legacy account-driver abstraction the wallet layer talks to. Two unrelated namespaces, same English word.

What the file actually does is wrap any block of code that needs the device in a **device session**. A "session" here means: ensure the DMK transport exists, drive the `ConnectAppDeviceAction` to open the right Manager-app on the device, run the user's function, then reset the DMK session on the way out. The whole thing lives in 30 lines:

```ts
// apps/wallet-cli/src/session/bridge-device-session.ts
export async function withCurrencyDeviceSession<T>(
  currencyId: string,
  fn: () => Promise<T>,
): Promise<T> {
  const { managerAppName } = getCryptoCurrencyById(currencyId);
  walletCliDebug("Ensuring DMK transport…");
  try {
    const transport = await ensureWalletCliDmkTransport();
    walletCliDebug(`Connecting Ledger app (${managerAppName})…`);
    await connectLedgerApp(transport.dmk, transport.sessionId, managerAppName);
    walletCliDebug("Device session ready.");
    try {
      return await fn();
    } finally {
      walletCliDebug("Resetting device session…");
      await resetWalletCliDmkSession();
    }
  } catch (e) {
    throw toWalletCliDeviceError(e);
  }
}
```

The flow:

1. Look up the canonical `managerAppName` for the currency (e.g. `"Ethereum"` for `ethereum`, `"Bitcoin"` for `bitcoin`).
2. `ensureWalletCliDmkTransport()` — singleton DMK transport (USB) is created lazily and cached.
3. `connectLedgerApp(...)` — runs DMK's `ConnectAppDeviceAction`, which navigates the device's Manager UI to the requested app and opens it. Returns once the device is in the app or throws.
4. The user's `fn` runs *inside* the open app. This is the critical line: every send / verify / discover happens with the device already in the right app, so the bridge does not have to redo the dance.
5. The `finally` always resets the session — the DMK abstraction wants a clean slate between currency changes.
6. Any thrown error is funnelled through `toWalletCliDeviceError` (a mapper that turns DMK errors into user-friendly `WalletCliDeviceError` instances — see `device/wallet-cli-device-error.ts`).

A second helper, `withBridgeDeviceSession(family, fn)`, is a thin family→currencyId mapper for older callers that still pass `"evm"` instead of `"ethereum"`. New code should use `withCurrencyDeviceSession` directly.

`send`, `discover`, and `receive --verify=true` all wrap their critical section in this helper. `balances`, `operations`, and `fresh-address` (for non-UTXO families) do not — they are device-free.

---

### 5.4.11 Output formatting

`src/output.ts` (338 lines) is the one place that decides what shape `human` vs `json` mode emits. Every command handler builds *one* `CommandOutput` instance and calls semantic methods on it; the implementation handles the rest.

```ts
// apps/wallet-cli/src/output.ts
export function createCommandOutput(format: "human" | "json", ctx: OutputContext): CommandOutput {
  const humanFmt = new HumanFormatter(getCryptoAssetsStore());
  if (format === "json") return new JsonCommandOutput(ctx, humanFmt);
  return new HumanCommandOutput(humanFmt);
}
```

The `CommandOutput` interface has two halves:

- **Lifecycle:** `withActivity`, `spin`, `run`, `fail`. These wrap async work with a spinner (human) or no-op (json) and centralise error handling. `JsonCommandOutput.run` catches errors, writes a `{ ok: false, error: { command, message } }` envelope to stdout, and `process.exit(1)`s. `HumanCommandOutput.run` re-throws so Bunli's default error handler can render the failure.
- **Data:** `balances`, `operations`, `address`, `discoveredAccount`, `flushDiscovery`, `sessionSaved`, `sessionReset`, `sessionView`, `sendDryRun`, `sendEvent`, `sendComplete`. One method per output shape.

The send pipeline shows the split most clearly:

```ts
// HumanCommandOutput.sendEvent — drives the active spinner
sendEvent(event: SendEvent): void {
  const s = this._activeSpin;
  switch (event.type) {
    case "prepared":
      s?.clear();
      this._printTransactionLines(event);
      if (s) s.text = "Confirm transaction on device…";
      break;
    case "device-streaming":
      if (s) s.text = `Streaming to device… ${Math.round(event.progress * 100)}%`;
      break;
    case "device-signature-requested":
      if (s) s.text = "Please confirm on device…";
      break;
    case "device-signature-granted":
      if (s) s.text = "Signed, broadcasting…";
      break;
    case "dry-run":
      s?.success("Dry run complete (transaction not broadcasted)");
      break;
    case "broadcasted":
      s?.success(`Broadcasted  ${colors.dim(event.txHash)}`);
      break;
  }
}

// JsonCommandOutput.sendEvent — accumulates into _sendResult; nothing is written until sendComplete
sendEvent(event: SendEvent): void {
  if (event.type === "prepared") {
    this._sendResult.recipient = event.recipient;
    this._sendResult.amount = event.amount;
    this._sendResult.fee = event.fees;
  } else if (event.type === "broadcasted") {
    this._sendResult.tx_hash = event.txHash;
  } else if (event.type === "dry-run") {
    this._sendResult.dry_run = true;
  }
}
```

Same Observable, two completely different rendering strategies: the human formatter mutates a single live spinner so the user sees a smooth narrative; the JSON formatter accumulates into a single envelope that is emitted once at `sendComplete()`. That guarantees scripts piping `--output json` see exactly one JSON document on stdout per command.

One subtlety in `JsonCommandOutput.run`:

```ts
async run(fn: () => Promise<void>): Promise<void> {
  try { await fn(); }
  catch (e) {
    // Bun.spawn on Linux does not reliably capture fd 2 (stderr) from subprocesses.
    // writeSync(1, ...) is a synchronous POSIX syscall — immune to process.exit() buffer drain.
    writeSync(1, this._errorEnvelope(e) + "\n");
    process.exit(1);
  }
}
```

The error envelope is written *to stdout* (fd 1) using a synchronous `writeSync`, not through `console.error`. The comment explains why: `Bun.spawn` on Linux truncates fd 2 capture on `process.exit`. This matters for the test harness — `runCli` reads `proc.stdout`/`proc.stderr` and asserts on both. The lesson: in JSON mode, success and failure both come down stdout, distinguished by `.status === "success"` vs `.ok === false`.

---

### 5.4.12 The test directory tour

`src/test/` is where the Speculos-analog lives. Full deep dive in 5.6; here is the inventory so you know what each file is for when 5.6 cross-references it.

**`src/test/commands/`** — black-box command tests. Each spec spawns the CLI as a subprocess (under Bun), with HTTP and DMK fully mocked, asserts on stdout/stderr/exit code.

| File | Lines (per R1) | Covers |
|---|---|---|
| `balances.test.ts` | 88 | `balances` command |
| `discover.test.ts` | 79 | `account discover` |
| `operations.test.ts` | 125 | `operations` |
| `receive.test.ts` | 33 | `receive` |
| `send.test.ts` | 72 | `send` (dry-run + signed-with-mock-DMK paths) |
| `session.test.ts` | 77 | `session view` and `session reset` |
| `README.md` | 14 | The mock layer table — quoted in 5.4.9 |

**`src/test/helpers/`** — the harness. Spend time on these in 5.6.

| File | Lines | Role |
|---|---|---|
| `cli-runner.ts` | 52 | `runCli(args, env)` — `Bun.spawn` wrapper with `NO_COLOR=1`, `CLAUDECODE=1`, stdio piping. Used by every command test. |
| `wrapper.ts` | 8 | The default entrypoint for spawned tests — installs HTTP intercept then imports `cli.ts`. |
| `wrapper-local.ts` | 3 | Same idea, no HTTP intercept — for commands that make no network calls. |
| `http-intercept.ts` | 146 | Patches `globalThis.fetch` and `node:http`/`node:https` so all outbound HTTP is routed to the mock server. Activated by `WALLET_CLI_MOCK_PORT`. |
| `mock-server.ts` | 58 | `Bun.serve`-based route table. Routes are passed in by the test. |
| `eth-sync-routes.ts` | 37 | The default ETH HTTP fixtures (balance, nonce, gas estimate, gastracker, sync routes) — re-used across ETH-flavoured tests. |
| `dmk-intercept.ts` | 29 | Installs `MockDeviceManagementKit` when `WALLET_CLI_MOCK_DMK=1`. Coin-app results come from `WALLET_CLI_MOCK_APP_RESULTS` (JSON env var). |
| `constants.ts` | 8 | ETH descriptors and other test fixtures. |
| `session-fixture.ts` | 27 | Sets `XDG_STATE_HOME` to a fresh tmpdir so `session.yaml` is isolated per test. |

The two environment variables you will see scattered across tests:

- `WALLET_CLI_MOCK_PORT=<port>` — turns on `http-intercept` so all fetch/http traffic redirects to `localhost:<port>` where `mock-server.ts` is listening.
- `WALLET_CLI_MOCK_DMK=1` (with `WALLET_CLI_MOCK_APP_RESULTS=<json>`) — turns on `dmk-intercept` so the DMK transport is replaced by a mock that returns the supplied app results without touching USB.

These are the only two flags to know when reading a `*.test.ts` file. The rest is per-spec setup.

> Forward reference: 5.6 is the deep dive — how `cli-runner` spawns the binary, how `http-intercept` patches fetch, how `mock-dmk` returns synthetic app results. For now you have the map.

---

### 5.4.13 How a command actually executes

To consolidate everything in 5.4 so far, here is the sequence diagram for one `pnpm wallet-cli start -- balances ethereum-1 --output json` invocation. ASCII; one swimlane per layer.

```
Shell           pnpm/Nx           Bun runtime         cli.ts                Bunli
-----           -------           -----------         ------                -----
$ pnpm wallet-cli start -- balances ethereum-1 --output json
   |
   |--filter--> @ledgerhq/wallet-cli
                      |
                      |--script "start"--> bun run ./src/cli.ts -- balances ethereum-1 --output json
                                                  |
                                                  |--shebang "#!/usr/bin/env bun"
                                                  |--ESM resolver kicks in
                                                  |
                                                  |  side effect imports (in order):
                                                  |   1. embed-usb-native     (require usb addon early)
                                                  |   2. live-common-setup    (register coin modules + DMK transport)
                                                  |   3. .bunli/commands.gen  (auto-register every command)
                                                  |   4. bunli.config         (build-time metadata)
                                                  |
                                                  |--createCLI(bunliConfig)
                                                  |          |
                                                  |          |--read argv ["balances","ethereum-1","--output","json"]
                                                  |          |--look up "balances" in registered commands
                                                  |          |--Zod-parse flags + positional
                                                  |          |
                                                  |          v
                                                  |   handler({ flags, positional })
                                                  |          |
                                                  |          |--resolveAccountArg → "ethereum-1"
                                                  |          |--resolveAccountDescriptor("ethereum-1")
                                                  |          |     |--Session.read() (XDG_STATE_HOME/ledger-wallet-cli/session.yaml)
                                                  |          |     |--lookup label → V1 descriptor string
                                                  |          |     `--parseV1 → { type:"address", network, address, path }
                                                  |          |
                                                  |          |--ctx = { command:"balances", network:"ethereum:main", account:<id> }
                                                  |          |--out  = createCommandOutput("json", ctx)   → JsonCommandOutput
                                                  |          |--wallet = new WalletAdapter()
                                                  |          |
                                                  |          |--out.run(async () => {
                                                  |          |     wallet.getAccountBalances(descriptor)
                                                  |          |        |
                                                  |          |        |--family === "evm"  → AlpacaAdapter.getBalances
                                                  |          |        |       |
                                                  |          |        |       `--HTTP GET (alpaca API; balances command does NOT
                                                  |          |        |          open a DMK session — no withCurrencyDeviceSession here)
                                                  |          |        v
                                                  |          |     [{ assetId, balance }, …]
                                                  |          |     out.balances(items)
                                                  |          |        |
                                                  |          |        `--JsonCommandOutput.balances → writeStdout(envelope)
                                                  |          |              |
                                                  |          |              `--makeEnvelope("balances","ethereum:main",
                                                  |          |                                { balances:[…] }, accountId)
                                                  |          |                  → JSON.stringify → stdout
                                                  |          })
                                                  |
                                                  |--disposeWalletCliDmkTransportFully()  (no-op here; no DMK was opened)
                                                  `--process.exit(0)
```

Three things to notice:

1. **DMK is not always engaged.** `balances` in EVM goes through Alpaca — no `withCurrencyDeviceSession`, no device prompt, no app-open. Compare that with `send`, where the swimlane gets a fourth lane (DMK) between Bunli and stdout, and the handler wraps its critical section in `withCurrencyDeviceSession` so DMK's `ConnectAppDeviceAction` runs first.
2. **Command registration happens at import time.** By the time `createCLI` runs, every command is already in Bunli's store via the side-effect import of `.bunli/commands.gen`. The CLI is one giant statically-linked module — there is no dynamic discovery in compiled mode.
3. **The output is one JSON document.** `JsonCommandOutput` accumulates and emits once. If you pipe to `jq` you do not need `--slurp` — it is already a single envelope.

---

### 5.4.14 Cross-reference: comparing wallet-cli to mobile and desktop layouts

The three workspaces solve different problems and are organised accordingly. A table is the fastest way to see the contrast — keep it in mind when you switch contexts.

| Concern | `apps/wallet-cli/src/` | `e2e/mobile/` | `e2e/desktop/tests/` |
|---|---|---|---|
| **Workspace location** | `apps/wallet-cli/` (a real product, not a test workspace) | Peer to desktop, repo-root level | Peer to mobile, repo-root level |
| **Runtime** | Bun (single binary, ESM) | Node + Detox (Jest worker) + RN runtime on simulator | Node + Playwright + Electron renderer |
| **Test surface** | `src/test/commands/*.test.ts` (black-box subprocess) | `e2e/mobile/specs/*.spec.ts` (driver matrix) | `e2e/desktop/tests/specs/*.spec.ts` (Playwright specs) |
| **Page Object Model** | None — there is no UI | Yes — `e2e/mobile/page/` (32-getter `Application` hub) | Yes — `e2e/desktop/tests/page-objects/` |
| **Bridge concept** | "bridge" = live-common `AccountBridge`. No network. | "bridge" = WebSocket between Jest and RN. ACK protocol. | No equivalent (Playwright drives Electron directly) |
| **Speculos analog** | `MockDeviceManagementKit` (no Docker, in-process) + mock HTTP | Real Speculos in Docker, talked to over USB-mock | Real Speculos in Docker, same approach as mobile |
| **Output contract** | Stable: `human` (text) and `json` (envelope). The CLI **is** the contract. | Allure report + CSV artifacts + screenshots | Allure report + screenshots |
| **Account model** | V1 descriptor strings (`account:1:…`) parsed by `accountDescriptor/v1.ts` | Live-common `Account`/`TokenAccount` enums (e.g. `Account.ETH_1`) | Same live-common enums as mobile |
| **Configuration** | `bunli.config.ts` + `tsconfig.json`. No `detox.config.js`. | `detox.config.js` (multi-device matrix) + `jest.config.js` | `playwright.config.ts` |
| **CI sharding** | One job (commands fast — ~seconds) | Sharded matrix (asymmetric iOS/Android caps) | Sharded matrix (homogeneous Linux runners) |
| **Test runner** | `bun test src/` | `pnpm mobile e2e:test -c <device>` | `pnpm desktop e2e:test` |
| **What you write to test** | Subprocess assertions on stdout/stderr/exit code | UI-level interactions and bridge messages | UI-level interactions |
| **What you write to extend behaviour** | A new file under `src/commands/` + `bunli generate` | Almost always a new spec + page-object additions | Same as mobile |

Two takeaways. **First**, the wallet-cli is a *product*, not a test artifact. Mobile and desktop test the apps; wallet-cli **is** an app — its users (the QAA-613 hooks, ad-hoc scripts) consume the JSON envelope as a contract. **Second**, the "bridge" terminology collision is permanent: in Part 4 it meant a WebSocket; here it means live-common's account-driver layer. Read the imports.

---

### 5.4.15 Chapter 5.4 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — wallet-cli Codebase Deep Dive</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> The <code>src/</code> tree groups files by responsibility. Which one of these statements about the layout is correct?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>src/wallet/</code> contains DMK transport code; <code>src/device/</code> contains intent schemas</button>
<button class="quiz-choice" data-value="B">B) <code>src/session/</code> persists YAML to <code>~/.config</code>; <code>src/shared/</code> holds the live-common bridge adapter</button>
<button class="quiz-choice" data-value="C">C) <code>src/commands/</code> is leaf handlers; <code>src/wallet/</code> is the wallet+intents abstraction; <code>src/device/</code> is DMK + USB; <code>src/session/</code> holds the session store and the device-session helper; <code>src/shared/</code> holds account descriptors and tiny utilities</button>
<button class="quiz-choice" data-value="D">D) <code>src/output.ts</code> sits inside <code>src/commands/</code> because every command uses it</button>
</div>
<p class="quiz-explanation">The directory layout is the architecture. Commands are leaves; wallet, device, session, and shared are the lower layers. <code>output.ts</code> is at the top level because both human and JSON formatters orchestrate <em>across</em> commands. Mixing these up is the most common reason new contributors put files in the wrong place.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> A teammate proposes adding a <code>--memo &lt;text&gt;</code> flag to <code>send</code> for EVM transactions, mirroring Solana. Where do they need to make the change so the schema validates the new field?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Only in <code>src/commands/send.ts</code> — the schema accepts unknown fields</button>
<button class="quiz-choice" data-value="B">B) <code>src/wallet/intents/families/evm.ts</code> needs the new optional field added to <code>EvmTransactionIntentSchema</code>; the <code>INTENT_BUILDERS["evm"]</code> entry in <code>send.ts</code> needs to forward <code>flags.memo</code>; downstream <code>BridgeAdapter.send</code> must consume it</button>
<button class="quiz-choice" data-value="C">C) Only in <code>src/wallet/intents/index.ts</code> — that file owns all family rules</button>
<button class="quiz-choice" data-value="D">D) In <code>output.ts</code>, because the formatter renders memos</button>
</div>
<p class="quiz-explanation">Zod's <code>z.object</code> rejects unknown keys by default. The intent schema in <code>families/evm.ts</code> is the source of truth — adding a field anywhere else without updating the schema means <code>TransactionIntentSchema.parse</code> throws on the unknown key. The intents directory is the only extensibility hook for the send pipeline; the schema, the builder, and the bridge adapter all have to agree.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> Today's <code>send</code> command on EVM accepts a <code>--data 0x…</code> flag. What does it actually do, and what is its consequence for QAA-615 (the upcoming <code>revoke</code> spike)?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It is a no-op flag reserved for a future release</button>
<button class="quiz-choice" data-value="B">B) It overrides the recipient address</button>
<button class="quiz-choice" data-value="C">C) It signs the data as a personal message, not a transaction</button>
<button class="quiz-choice" data-value="D">D) It is hex-validated by the EVM intent schema, then passed through <code>WalletAdapter.send</code> → <code>BridgeAdapter.send</code> → <code>buildTxExtras</code>, which strips <code>0x</code> and patches a <code>Buffer</code> onto the live-common <code>Transaction</code>. The wallet-cli already accepts arbitrary EVM calldata — a <code>revoke</code> command becomes a thin wrapper that builds the right calldata and reuses the existing pipeline</button>
</div>
<p class="quiz-explanation">The <code>data</code> field is end-to-end wired today: EVM intent schema validates the hex, the bridge adapter converts to <code>Buffer</code>, the live-common <code>Transaction</code> picks it up, and the device clear-signs against the calldata selector. That is why R2 estimates QAA-615 at ~150 LOC of production code — the plumbing is already there.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> Which of the following descriptors is well-formed and parsable by <code>parseV1</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>account:1:address:ethereum:main:0x71C7656EC7ab88b098defB751B7401B5f6d8976F:m/44h/60h/0h/0/0</code></button>
<button class="quiz-choice" data-value="B">B) <code>account:2:address:ethereum:main:0x71C7656EC7ab88b098defB751B7401B5f6d8976F:m/44'/60'/0'/0/0</code></button>
<button class="quiz-choice" data-value="C">C) <code>account:1:utxo:bitcoin:main:xpub6BosfC...:m/84/0/0</code></button>
<button class="quiz-choice" data-value="D">D) <code>account:1:address:ethereum:mainnet:0x71...</code> (no path)</button>
</div>
<p class="quiz-explanation">A passes every check: purpose <code>account</code>, version <code>1</code>, type <code>address</code>, valid network <code>ethereum:main</code>, address, and a full BIP44 path with hardened-<code>h</code> notation. B fails on <code>version: "2"</code> (only "1" is accepted; the apostrophe form would otherwise be fine — <code>parseV1</code> accepts both <code>h</code> and <code>'</code>). C fails the UTXO-path regex (<code>m/84/0/0</code> has no hardened markers). D has only six segments — <code>parseV1</code> requires at least seven. The <code>mainnet</code> spelling in D would also be normalised to <code>main</code> by <code>parseNetworkArg</code>, but the missing path is the disqualifier here.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> Walking through the sequence diagram in 5.4.13, why does <code>balances ethereum-1 --output json</code> never invoke <code>withCurrencyDeviceSession</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because <code>--output json</code> disables device interactions</button>
<button class="quiz-choice" data-value="B">B) Because the descriptor was resolved from the session store rather than the device</button>
<button class="quiz-choice" data-value="C">C) Because <code>WalletAdapter.getAccountBalances</code> routes EVM balances through the Alpaca direct API, which is an HTTP call to the explorer — no signing, no app-open, no device session needed. <code>withCurrencyDeviceSession</code> is only wrapped around code that drives the device (send, discover, verify-address)</button>
<button class="quiz-choice" data-value="D">D) Because <code>balances</code> is mocked in test mode</button>
</div>
<p class="quiz-explanation">The Alpaca path is read-only and stateless — it is a remote query, not a wallet operation. The CLI engages DMK only when it must (signing, discovery via on-device pubkey derivation, or address verification). Balances and operations are purely informational on EVM and do not need the device.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q6.</strong> Compared with <code>e2e/mobile/</code> and <code>e2e/desktop/tests/</code>, which statement most accurately captures what is structurally different about <code>apps/wallet-cli/src/</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) wallet-cli has its own POM hub; mobile and desktop do not</button>
<button class="quiz-choice" data-value="B">B) wallet-cli is a real product (a Bun-compiled binary) whose <em>contract</em> is the JSON envelope on stdout. Mobile and desktop are test workspaces that drive a UI through Detox/Playwright. The "bridge" word means live-common's <code>AccountBridge</code> here, not the WebSocket bridge from Part 4</button>
<button class="quiz-choice" data-value="C">C) wallet-cli is the only one that uses Speculos</button>
<button class="quiz-choice" data-value="D">D) wallet-cli has no test directory</button>
</div>
<p class="quiz-explanation">wallet-cli sits under <code>apps/</code>, not <code>e2e/</code>, because it is shipped as a product. Its tests (<code>src/test/commands/</code>) are subprocess-level black-box tests, not UI specs. The "bridge" namespace collision — live-common <em>vs</em> mobile-WebSocket — is the most common source of confusion when crossing from Part 4 to Part 5.</p>
</div>

<div class="quiz-score"></div>
</div>

<div class="chapter-outro">
<strong>Key takeaway.</strong> You now own the full <code>src/</code> map: <code>WalletAdapter</code> as the routing class with two compatibility adapters underneath, the four-family intent schemas (the only extensibility hook for <code>send</code>, with EVM's <code>data</code> field already wired end-to-end), Account Descriptors V1 with their colon grammar and shell-safe <code>h</code> notation, the small shared utilities, the <code>withCurrencyDeviceSession</code> helper that wraps every device-using command, the JSON envelope contract enforced by <code>output.ts</code>, and the test directory inventory you will revisit in 5.6. With the codebase mapped, <strong>Chapter 5.5</strong> moves to <em>using</em> the binary — running it with <code>pnpm wallet-cli start</code>, debugging it under Bun's inspector, and reading what <code>--debug</code> output actually means when a command hangs.
</div>
## Running and Debugging wallet-cli

<div class="chapter-intro">
Chapter 5.4 walked you through every file under <code>apps/wallet-cli/</code>: the Bunli command framework, the DMK transport layer, the wallet adapter, the V1 account descriptor module. You can read the codebase. Now you have to run it. This chapter turns that map into hands-on muscle memory: how to install once, how to invoke from two different cwds, when to plug in a device, what JSON envelope to expect on stdout, where the spinners go, and which errors mean what. By the end you should be able to discover an Ethereum account from a real Nano X on your laptop, capture the JSON envelope of <code>balances</code> into a shell pipeline, and recognise the four or five errors the device layer is going to throw at you on day one.
</div>

<a id="running-and-debugging-wallet-cli"></a>

### 5.5.1 Prerequisites

Before running a single command, three layers of toolchain must be in place. Unlike Detox (Chapter 4.4) or Playwright (Chapter 3.4) — where the runner manages its own browser or simulator — `wallet-cli` shells straight through to a USB device. Three things have to line up: the Bun runtime, the platform USB stack, and the device itself.

**Bun runtime:**
- Bun >= 1.1.0. The version is pinned in `apps/wallet-cli/package.json` under `engines.bun`. Local development currently tracks `bun-types@1.3.12` per the dev dependencies, but anything `>= 1.1.0` works.
- Install with `curl -fsSL https://bun.sh/install | bash` if not present, or `brew install oven-sh/bun/bun` on macOS.
- Verify with `bun --version`. The CLI will refuse to start under Node — its shebang is `#!/usr/bin/env bun` and the entry script `src/cli.ts` imports Bun-only globals like `Bun.spawn` indirectly through the test layer.

**Linux USB stack:**

```bash
sudo apt-get update
sudo apt-get install libudev-dev libusb-1.0-0-dev
```

These two packages back the `usb` N-API native addon. Without them, the addon fails to load on first DMK call and you get an opaque dynamic-linker error on the very first `account discover`. The wallet-cli `README.md` lists this as the only Linux-specific prerequisite. On Debian/Ubuntu it is `libudev-dev` + `libusb-1.0-0-dev`; on Fedora/RHEL the equivalents are `libusbx-devel` + `systemd-devel`.

**macOS USB stack:**
- Install Xcode Command Line Tools (`xcode-select --install`). The `usb` addon ships prebuilt darwin binaries, but the dynamic linker still needs the system `libusb` provided by the CLT bundle.
- No `brew install libusb` step is required for the prebuilt path. If you ever re-build the addon from source (`npm rebuild usb`), then `brew install libusb` becomes necessary.

**Windows:**
- The `bunli build` target list includes `windows-x64`, but development on Windows is not common in QA — most engineers SSH into a Linux box for device work, or use macOS directly.

**Device-side prerequisites:**
- A Ledger device on USB — Nano S Plus, Nano X, or Stax. Nano X works in USB mode (not Bluetooth — the CLI only knows the node-WebUSB transport).
- The relevant coin app is open on the device for any device-required command. `account discover ethereum` needs the **Ethereum** app open. `account discover bitcoin` needs the **Bitcoin** app open. `account discover base` needs the Ethereum app open (Base uses the Ethereum app on device — see the `live-common-setup.ts` family mapping).
- The device unlocked (PIN entered).

> **Tip:** `connect-ledger-app.ts` in the device layer tries to open the right Manager app for you (it wraps DMK's `ConnectAppDeviceAction`). If the wrong app is open, the CLI will print `[i] Opening Ethereum app on your Ledger... ACTION REQUIRED: Confirm on device screen.` and wait. If the device is locked, you will see `[~] Waiting: Ledger detected but locked. Enter your PIN on the device.`. These two stderr lines are the canonical "device handshake" stage and are documented in the `connect-ledger-app.ts` source verbatim.

### 5.5.2 Install once

`wallet-cli` is a workspace inside the `ledger-live` monorepo. Its dependencies live alongside every other Live package, so a single `pnpm i` at the repo root sets it up:

```bash
cd /path/to/ledger-live
pnpm i
```

You normally do not need a separate `bun install` inside `apps/wallet-cli/`. The `package.json` declares `engines.bun >= 1.1.0` but its dependencies are pulled by `pnpm` against the workspace lockfile, then **executed** by `bun` at run time. The two tools play different roles: `pnpm` resolves and links, `bun` runs the source.

There is one exception. If you are iterating on `wallet-cli` standalone — outside the monorepo, in a copy of `apps/wallet-cli/` — you would need:

```bash
cd apps/wallet-cli
bun install
```

But that is a rare flow. The common path is monorepo-rooted `pnpm i`, period.

After install, sanity-check by running the `--help`:

```bash
pnpm --silent wallet-cli start --help
```

This prints the Bunli-generated top-level help — every group (`account`, `session`) and every leaf command (`balances`, `operations`, `receive`, `send`) with one-line descriptions. If you see this, your install is healthy.

### 5.5.3 Two ways to invoke

You can invoke the CLI from two cwds, and the syntax differs in exactly one respect: whether you cross a `pnpm` filter boundary.

**Form A — from the repo root:**

```bash
pnpm --silent wallet-cli start -- <command> [flags]
```

This is the form quoted in `SKILL.md` and in the `wallet-cli/README.md`. The expansion is:

1. `pnpm wallet-cli` runs the root-level passthrough script defined in the monorepo `package.json`. That script is just `pnpm --filter @ledgerhq/wallet-cli`.
2. `start` is the script name in `apps/wallet-cli/package.json` (`bun run ./src/cli.ts`).
3. The `--` separator is **required** — without it, `pnpm` consumes flags meant for the CLI as flags meant for itself. With it, everything to the right is forwarded verbatim.
4. `--silent` is the pnpm flag, not a wallet-cli flag. It suppresses pnpm's own preamble (`> @ledgerhq/wallet-cli@0.1.0 start ...`). On a `--output json` run, that preamble would corrupt the JSON envelope on stdout. Always pass `--silent` when you intend to parse the output.

**Form B — from inside `apps/wallet-cli/`:**

```bash
cd apps/wallet-cli
pnpm start -- <command> [flags]
```

This is shorter. Now `pnpm start` resolves directly against this package's `package.json`, no filter needed. The `--` separator is still required for the same reason.

Equivalent forms also work:

```bash
# from any cwd, use the workspace filter explicitly
pnpm --filter @ledgerhq/wallet-cli start -- <command> [flags]

# from apps/wallet-cli/, drive bun directly (skip pnpm entirely)
bun run ./src/cli.ts <command> [flags]
```

The last form is what `pnpm start` ultimately runs. It's the lowest-level invocation and is useful when you want to skip pnpm completely (for instance, inside a hook script that already has its cwd set).

> **Tip:** Pick Form A or Form B and stick with it within a session. Mixing them in shell history breeds confusion when you copy-paste a half-remembered command and discover the `--` separator went missing.

### 5.5.4 Device-required vs no-device commands

Every CLI command falls into one of two buckets: it talks to a USB device, or it doesn't. This single property drives almost every operational decision — whether a sandbox run will succeed, whether an in-CI replay can be hermetic, whether you need a coin app open. The matrix below is the canonical reference (lifted from `SKILL.md`):

| Command            | Device required                   | Disable sandbox              |
| ------------------ | --------------------------------- | ---------------------------- |
| `account discover` | Yes                               | **Yes**                      |
| `receive`          | Yes (optional with `--no-verify`) | **Yes**                      |
| `send`             | Yes (unless `--dry-run`)          | **Yes** (unless `--dry-run`) |
| `balances`         | No                                | No                           |
| `operations`       | No                                | No                           |
| `account fresh-address` (EVM/Solana) | No                | No                           |
| `session view` / `session reset` | No                    | No                           |

"Disable sandbox" refers to the **Bash tool** in agent contexts (Claude Code, Cursor, etc.). When you drive the CLI from a sandboxed agent, USB ioctls are blocked by default. Any command with "Yes" in the second column must be invoked with `dangerouslyDisableSandbox: true` — the agent's tool-call parameter that lifts the restriction for a single command. Plain shell users (you running `pnpm wallet-cli start ...` in your terminal) never hit this, because there's no sandbox.

The pattern that drives the matrix: **device-required = USB transport opens = sandbox blocks**. `account discover` opens DMK to read the device's xpub; `receive` opens DMK to verify the address on-screen (unless `--no-verify`); `send` opens DMK to sign and broadcast (unless `--dry-run`, which validates the transaction shape with no device access). `balances` and `operations` only call the Ledger Explorer API over HTTPS and never touch the device — sandbox is fine.

> **Why `send --dry-run` is safe:** `send` parses the intent, runs `prepareTransaction` against the bridge (gas estimate, nonce, fee data via HTTPS), and emits a `dry_run` envelope without ever opening DMK. No `connectLedgerApp`, no `signOperation`, no broadcast. You can run it in any sandboxed environment, in CI, in a unit test — and the test suite does exactly that (see `send.test.ts` in 5.6.10).

### 5.5.5 First device session

Plug in a Nano X over USB. Open the Ethereum app on the device. Then, from the repo root:

```bash
pnpm --silent wallet-cli start -- account discover ethereum
```

What you should see on stderr (it goes to stderr because the spinner uses `yocto-spinner` on fd 2):

```
[~] Discovering ethereum:main accounts...
```

If the device is locked or the Ethereum app isn't open, the human-mode handshake prints these lines on stderr (verbatim from `connect-ledger-app.ts`):

```
[~] Waiting: Ledger detected but locked. Enter your PIN on the device.
[i] Opening Ethereum app on your Ledger... ACTION REQUIRED: Confirm on device screen.
```

After you confirm, sync runs and on stdout you get one address per line plus a V1 descriptor:

```
0x71C7656EC7ab88b098defB751B7401B5f6d8976F  account:1:address:ethereum:main:0x71C7656EC7ab88b098defB751B7401B5f6d8976F:m/44h/60h/0h/0/0
0x... account:1:address:ethereum:main:0x...:m/44h/60h/1h/0/0
...
[i] Saved 3 account(s) to session.yaml
```

The third column is the V1 descriptor introduced in Chapter 5.4. It is **shell-safe** — hardened path segments use `h` instead of `'`, so you can paste the descriptor into any subsequent command without quoting.

To get JSON instead:

```bash
pnpm --silent wallet-cli start -- account discover ethereum --output json
```

stdout shape:

```json
{
  "status": "success",
  "command": "account discover",
  "network": "ethereum:main",
  "accounts": [
    "account:1:address:ethereum:main:0x71C7...:m/44h/60h/0h/0/0",
    "account:1:address:ethereum:main:0x83A2...:m/44h/60h/1h/0/0"
  ],
  "timestamp": "2026-04-27T11:42:00.000Z"
}
```

The envelope is always `{ status, command, network, [account?], <payload>, timestamp }`. The `[account?]` field is present only on commands that target a single account (`balances`, `operations`, `receive`, `send`). `account discover` is plural by definition.

A successful `discover` also writes `session.yaml` under `XDG_STATE_HOME/ledger-wallet-cli/` (or `~/.local/state/ledger-wallet-cli/` on Linux/macOS by default). From now on you can refer to those accounts by label:

```bash
pnpm --silent wallet-cli start -- balances ethereum-1 --output json
pnpm --silent wallet-cli start -- balances ethereum-2 --output json
```

The label resolver lives in `commands/inputs.ts` (`resolveAccountInput`): any value that does not contain `:` is looked up against `session.yaml`. Anything with `:` is treated as a descriptor passthrough (V0 `js:2:...` or V1 `account:1:...`).

### 5.5.6 Debug logging

The CLI uses two layered logging conventions: a **debug-namespace** convention (`debug` package) for wallet-cli internals, and a **DMK log level** convention (`LedgerLiveLogger`) for the device layer.

**wallet-cli internal logs** live in `shared/log.ts`:

```ts
// shared/log.ts (11 lines)
import createDebug from "debug";
export const debug = createDebug("wallet-cli");
```

Activate with the `DEBUG` environment variable:

```bash
# all wallet-cli debug lines
DEBUG=wallet-cli pnpm --silent wallet-cli start -- balances ethereum-1

# all wallet-cli plus any sub-namespace
DEBUG=wallet-cli:* pnpm --silent wallet-cli start -- account discover ethereum

# everything (very loud — includes live-common, axios, rxjs etc.)
DEBUG=* pnpm --silent wallet-cli start -- balances ethereum-1
```

These logs go to **stderr**. That is intentional: stderr is reserved for diagnostics, stdout is reserved for the JSON envelope. If you redirect `stdout` into `jq`, debug lines do not contaminate the parse.

**DMK device logs** are configured in `device/dmk.ts`:

```ts
// device/dmk.ts (excerpt)
.addLogger(new LedgerLiveLogger(LogLevel.Warning))
```

The default `LogLevel.Warning` means only warnings and errors from DMK reach stderr. To get full DMK trace, you'd need to either rebuild the binary with a different level or patch `dmk.ts` locally. There isn't an environment variable today that flips this — it is hard-coded in the factory. (This is a real limitation; if you need verbose APDU tracing for a hard bug, the workaround is to edit `dmk.ts` to `LogLevel.Debug` and rerun.)

**stdout vs stderr discipline:**
- **stdout** — JSON envelope (success or error in JSON mode), or human-readable result lines (in human mode).
- **stderr** — spinner, device handshake messages, `DEBUG=` namespace logs, error messages in human mode, errors mapped via `toWalletCliDeviceError` before the JSON envelope is committed.

The reason this matters is the **JSON output discipline**. When a hook script does:

```bash
result=$(pnpm --silent wallet-cli start -- balances ethereum-1 --output json)
echo "$result" | jq '.balances[0].amount'
```

…it captures only stdout. Any debug-namespace noise on stderr is irrelevant to the parse. If you accidentally drop `--silent`, pnpm's preamble line lands on stdout and breaks the parse — that is the single most common JSON-pipeline footgun.

> **Tip:** A test from `output.ts` reveals one important quirk: the JSON error path uses `writeSync(1, …)` plus `process.exit(1)` instead of writing to stderr. The comment says "Bun.spawn on Linux does not reliably capture fd 2 (stderr) — observable in CI." So even errors land on stdout in JSON mode, with `{ ok: false, error: { command, message } }` shape. Tests assert this exact shape — see 5.6.9.

### 5.5.7 Common errors

Every error you'll hit on day one comes from one of two sources: validation (Zod schemas in `commands/inputs.ts` and `wallet/intents/`) or device mapping (`wallet-cli-device-error.ts`). The table below maps the user-facing string to its diagnostic step. The five most common are also documented in `SKILL.md`:

| Error                                                            | Cause                                     | First diagnostic step |
| ---------------------------------------------------------------- | ----------------------------------------- | --------------------- |
| `Amount must include a ticker, e.g. '0.5 ETH' or '0.001 BTC'`    | `--amount` given a bare number            | Re-quote the value: `--amount '0.5 ETH'` not `--amount 0.5`. The Zod regex requires `<number> <TICKER>` with a space separator. |
| `Ticker UNKN not found in account. Available: ETH, USDT, ...`    | Ticker not in account balances            | Run `balances <descriptor> --output json` and copy a ticker from `data.balances[].asset`. The "Available:" suffix in the error already lists them. |
| `No currencyId mapping for network "x:y"`                        | Unsupported network/env combination       | Check `shared/accountDescriptor/network.ts` for the supported aliases. `bitcoin`, `ethereum`, `solana`, `base` are valid. `ethereum:mainnet` aliases to `:main`. `polygon` etc. are not yet supported. |
| `UnknownDeviceExchangeError`                                     | Device disconnected or wrong app          | Re-plug the cable, unlock the device, open the right coin app, then retry. The `connect-ledger-app.ts` retry loop will retry up to 5 times automatically — this error means it gave up. |
| `[x] Transaction Cancelled: Rejected on device. No funds moved.` | User pressed reject on device             | Expected outcome. Either the user pressed reject deliberately (cancel test), or the transaction shape on the device screen does not match what the user expected (look at calldata in the `--data` flag and re-derive). |
| `Timeout has occurred`                                           | Sandbox blocked USB                       | If you're running from an agent (Claude Code, Cursor), set `dangerouslyDisableSandbox: true` on the Bash tool call. If you're in a plain shell, this means the device disconnected mid-call — re-plug. |
| `Device is locked. Unlock your Ledger with your PIN and try again.` | PIN entry timed out                     | Enter the PIN on the device within the 60s `unlockTimeout` (set in `connect-ledger-app.ts`). |
| `Could not execute on the app. Open the correct app for this currency...` | Wrong coin app open                | Open the matching app on device. Base needs Ethereum, Solana needs Solana, Bitcoin needs Bitcoin. |
| `Timed out talking to the Ledger over USB. Retry; check cable.`  | `SendApduTimeoutError`                    | Cable issue or the device backgrounded. Re-plug; on macOS, try a different port or remove a USB hub from the chain. |
| `Could not communicate (garbled APDU). Unlock and retry.`        | `ReceiverApduError` after retries         | Same as above — DMK polling occasionally hits a transient transport-framing error. The internal retry already covers most cases. If you see this, re-plug. |

Most of these strings are produced by the mapper in `device/wallet-cli-device-error.ts`. New commands that introduce new error paths should call `toWalletCliDeviceError(e)` in their catch block to keep the surface uniform — you'll see `withCurrencyDeviceSession` already does this.

> **Tip:** The "Available:" list in the `Ticker UNKN not found` error is the fastest way to discover what tickers a discovered account actually has. Pass an obviously-bogus ticker (`--amount '1 ZZZ'`) on a `--dry-run` send and the error tells you everything the account holds: `Available: ETH, USDT, USDC, DAI, ...`. Faster than scrolling `balances` output.

### 5.5.8 Debugging with `bun --inspect`

Bun ships a Chrome DevTools-compatible debugger. The CLI is a Bun program, so attaching the debugger is one flag away.

**Launch with debugger waiting:**

```bash
cd apps/wallet-cli
bun --inspect-brk ./src/cli.ts balances account:1:address:ethereum:main:0x71C7...:m/44h/60h/0h/0/0
```

`--inspect-brk` blocks the process at startup and prints a `devtools://` URL to stderr. Open it in a Chromium-based browser (Chrome, Edge, Brave). You get the full DevTools panel: Sources, breakpoints, call stack, watch expressions, REPL.

**Launch without breaking at startup:**

```bash
bun --inspect ./src/cli.ts balances <descriptor>
```

Useful when you want to trigger a breakpoint by code change rather than at boot. Pair this with a `debugger;` statement in the source you're debugging.

**Setting a conditional breakpoint:**

In DevTools, click the line gutter of `commands/balances.ts` and right-click the breakpoint marker to add a condition. For example, in the `balances` handler, break only when an EVM account is being queried:

```javascript
descriptor.currencyId === "ethereum"
```

**The `dbg` skill alternative:** the global `dbg` skill (`~/.claude/skills/dbg/SKILL.md`) wraps the same protocol with a CLI you can drive from inside an agent context. Useful when you want repeatable scripted breakpoints.

```bash
dbg launch --brk bun ./src/cli.ts balances <descriptor>
dbg break src/wallet/index.ts:42
dbg state
dbg continue
```

`dbg` exposes the same V8/CDP machinery Bun's debugger uses. Pick whichever interface (visual DevTools or CLI) you prefer — they connect to the same underlying inspector.

> **Caveat:** `--inspect` only works on the `bun run` path, not on the standalone binary produced by `bunli build`. If you need to debug a `dist/wallet-cli` binary, you have to rebuild without `--minify` and even then symbol fidelity is reduced. Always debug from source.

### 5.5.9 Iterating on a single command

When you're authoring a new command (the QAA-615 `revoke` spike, for instance), edit-save-rerun is your inner loop. Three tools help:

**`pnpm dev` — Bunli dev mode with hot reload:**

```bash
cd apps/wallet-cli
pnpm dev
```

This expands to `bunli dev`. It boots the CLI under a watcher; saving any file under `src/commands/` regenerates `.bunli/commands.gen.ts` in-memory and reloads. Useful when you're shaping option schemas and want fast feedback on the `--help` rendering.

**`bun --watch` — re-run on save:**

```bash
cd apps/wallet-cli
bun --watch run ./src/cli.ts balances <descriptor> --output json | jq '.balances[0]'
```

`--watch` re-spawns the CLI on every save under the watched directory. Pair with `jq` for the inner-loop equivalent of "edit code -> see result". Note that `--watch` re-runs the whole command from scratch each time, so device-required runs would re-enter the PIN+app handshake — use `--watch` for `--dry-run` or no-device commands.

**`pnpm run start` — explicit no-watcher run:**

```bash
cd apps/wallet-cli
pnpm start -- balances <descriptor>
```

This is the canonical "I just want to run it once" form. It shells out to `bun run ./src/cli.ts`, no watcher.

A realistic tight loop when adding a new flag to `send`:

```bash
# Terminal 1: edit src/commands/send.ts
$EDITOR src/commands/send.ts

# Terminal 2: re-run on save against a dry-run case
bun --watch run ./src/cli.ts send <descriptor> \
  --to 0xDEF... --amount '0.1 ETH' --dry-run --output json \
  | jq '.dry_run, .recipient, .amount'
```

Save the file in Terminal 1; Terminal 2 re-runs in <1s and the JSON drops out. No device involvement.

> **Tip:** Whenever you add or remove a command file under `src/commands/`, run `pnpm generate` to regenerate `.bunli/commands.gen.ts`. The CI gate `pnpm generate:check` (which runs in PR validation) will fail your PR if you forgot, with a `git diff --exit-code` failure on the generated file. Always commit the regenerated `.bunli/commands.gen.ts` alongside your command.

### 5.5.10 Reading help

Every Bunli `defineCommand` carries a `description` plus per-flag `description` strings. These are auto-rendered by Bunli's `--help`:

**Top-level help:**

```bash
pnpm --silent wallet-cli start -- --help
```

Lists every group (`account`, `session`) and every leaf command (`balances`, `operations`, `receive`, `send`).

**Group help:**

```bash
pnpm --silent wallet-cli start -- account --help
pnpm --silent wallet-cli start -- session --help
```

Lists subcommands within the group.

**Leaf command help:**

```bash
pnpm --silent wallet-cli start -- send --help
pnpm --silent wallet-cli start -- balances --help
pnpm --silent wallet-cli start -- account discover --help
```

Each prints flags, descriptions, defaults, and required-vs-optional. The `description` strings are pulled directly from `option(z.…(), { description: "…" })` calls in the command source and from the shared option helpers in `commands/inputs.ts`.

> **Tip:** When `SKILL.md` and the source disagree on a flag, **trust the source** and rerun `--help`. Bunli help is generated from the live Zod schemas at runtime, so it cannot drift the way a hand-written doc can. `SKILL.md` lags reality by definition; `--help` is reality.

A worked example:

```bash
$ pnpm --silent wallet-cli start -- send --help

Usage: wallet-cli send [options]

Options:
  --account, -a <string>      Account descriptor or session label (e.g. ethereum-1)...
  --to, -t <address>          Recipient address (required)
  --amount <"V TICKER">       e.g. '0.001 BTC', '0.01 ETH', '0.4 USDT' (required)
  --fee-per-byte <satoshis>   Custom fee per byte (bitcoin only)
  --rbf                       Enable Replace-By-Fee (bitcoin only)
  --mode <mode>               Solana mode: send | stake.createAccount | stake.delegate | ...
  --validator <address>       Validator address (Solana staking)
  --stake-account <address>   Stake account address (Solana staking)
  --memo <text>               Memo/tag (Solana)
  --data <0x...>              EVM raw calldata as 0x-hex
  --dry-run                   Prepare and validate without signing or opening the device
  --output <human|json>       Output format (default: human)
  -h, --help                  display help for command
```

The `--data` flag is the fact that unlocked the QAA-615 spike: EVM raw calldata is already plumbed end-to-end. A `revoke` subcommand would just be a wrapper that builds `0x095ea7b3<spender_padded>0000…0` and hands off to `send`.

### 5.5.11 The sandbox trap

If you're driving the CLI from inside an agent (Claude Code, Cursor, an automation pipeline using a sandboxed Bash tool), there is exactly one thing to remember:

> **Device-required commands need the sandbox disabled. Non-device commands do not.**

Concretely, when an agent invokes a Bash tool to run a wallet-cli command, the tool call has a `dangerouslyDisableSandbox` parameter. The decision rule is the device-required column from 5.5.4:

| Command                                  | `dangerouslyDisableSandbox` |
| ---------------------------------------- | --------------------------- |
| `account discover <network>`             | **true** |
| `receive <descriptor>`                   | **true** |
| `receive <descriptor> --no-verify`       | false (no device) |
| `send <descriptor> ... `                 | **true** |
| `send <descriptor> ... --dry-run`        | false (no device) |
| `balances <descriptor>`                  | false |
| `operations <descriptor>`                | false |
| `account fresh-address` (EVM/Solana)     | false |
| `session view` / `session reset`         | false |

Why this matters: the sandbox blocks USB ioctls. A device-required command spends ~30s opening the USB transport, then **silently times out** with `Timeout has occurred`. The error is genuinely confusing — it doesn't say "sandbox" anywhere. It looks identical to a flaky USB cable. The first time you hit it, you'll spend 10 minutes re-plugging cables before remembering the rule.

Plain shell users (you, in Terminal.app, running `pnpm wallet-cli start ...`) never hit this. The sandbox is only a thing when an agent harness is mediating the call.

> **Citation:** This rule is documented in `SKILL.md` (line 20) and is the single most common pitfall the skill warns against. Read it once, internalize it, never forget.

### 5.5.12 Chapter 5.5 Quiz

<!-- ── Chapter 5.5 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> Why do you need to pass <code>--silent</code> to <code>pnpm</code> when invoking <code>wallet-cli</code> with <code>--output json</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because <code>pnpm</code> defaults to noisy debug output that slows down command execution</button>
<button class="quiz-choice" data-value="B">B) Because <code>--silent</code> tells the CLI to skip the device handshake</button>
<button class="quiz-choice" data-value="C">C) Because <code>pnpm</code> prints a script preamble (<code>&gt; @ledgerhq/wallet-cli@... start ...</code>) on stdout that would corrupt the JSON envelope when parsed</button>
<button class="quiz-choice" data-value="D">D) Because <code>--silent</code> is required by Bunli to enable JSON mode</button>
</div>
<p class="quiz-explanation">Without <code>--silent</code>, pnpm prints a script-resolution preamble line on stdout before the CLI's own output. That preamble breaks any downstream JSON parser. The flag is a pnpm flag, not a wallet-cli flag.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> Which of these commands does NOT require the sandbox to be disabled when run from a Claude Code Bash tool?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>account discover ethereum</code></button>
<button class="quiz-choice" data-value="B">B) <code>send &lt;descriptor&gt; --to 0xABC --amount '0.5 ETH' --dry-run</code></button>
<button class="quiz-choice" data-value="C">C) <code>receive &lt;descriptor&gt;</code> (default <code>--verify=true</code>)</button>
<button class="quiz-choice" data-value="D">D) <code>send &lt;descriptor&gt; --to 0xABC --amount '0.5 ETH'</code></button>
</div>
<p class="quiz-explanation"><code>--dry-run</code> validates and prepares the transaction (gas estimate, nonce, fee) over HTTPS but never opens the USB transport. No device, no sandbox concern. The other three all open DMK and would time out under sandbox.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> The error <code>Timeout has occurred</code> when running <code>account discover</code> from an agent harness most likely means:</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The device's PIN is incorrect</button>
<button class="quiz-choice" data-value="B">B) The Ethereum app on the device has crashed</button>
<button class="quiz-choice" data-value="C">C) The Ledger Explorer API is down</button>
<button class="quiz-choice" data-value="D">D) The sandbox blocked the USB ioctls — re-run with <code>dangerouslyDisableSandbox: true</code></button>
</div>
<p class="quiz-explanation">The CLI cannot tell the difference between "USB cable disconnected" and "sandbox blocked the call" — both manifest as a 60s timeout. In an agent context, the most common cause is the sandbox. SKILL.md documents this explicitly.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> What does <code>--output json</code> do when an error occurs?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Writes a <code>{ ok: false, error: { command, message } }</code> envelope to stdout and exits with code 1</button>
<button class="quiz-choice" data-value="B">B) Writes the error to stderr in JSON format and exits with code 0</button>
<button class="quiz-choice" data-value="C">C) Suppresses the error entirely so the script doesn't fail</button>
<button class="quiz-choice" data-value="D">D) Logs the error in <code>session.yaml</code> for later inspection</button>
</div>
<p class="quiz-explanation">In JSON mode, <code>output.ts</code>'s <code>JsonCommandOutput</code> writes the error envelope on stdout (using <code>writeSync(1, ...)</code> to bypass a Bun.spawn stderr-capture quirk on Linux) and calls <code>process.exit(1)</code>. The shape <code>{ ok: false, error: { command, message } }</code> is asserted by tests in <code>balances.test.ts</code> and is intentionally different from the success envelope's <code>{ status: "success", ... }</code>.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> After running <code>account discover ethereum</code>, you can refer to the discovered account as <code>ethereum-1</code> in subsequent commands. Where is that label-to-descriptor mapping stored?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) In an environment variable that lives only for the shell session</button>
<button class="quiz-choice" data-value="B">B) In a SQLite database under <code>~/.config/ledger-wallet-cli/</code></button>
<button class="quiz-choice" data-value="C">C) In a YAML file at <code>$XDG_STATE_HOME/ledger-wallet-cli/session.yaml</code></button>
<button class="quiz-choice" data-value="D">D) Nowhere — labels are derived dynamically from the device on every command</button>
</div>
<p class="quiz-explanation">The session store in <code>session/session-store.ts</code> persists labels to YAML at <code>stateDir(APP_NAME)/session.yaml</code> where <code>stateDir</code> resolves XDG env vars. Tests redirect this with <code>XDG_STATE_HOME=/tmp/...</code> via <code>session-fixture.ts</code>.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q6.</strong> Why is <code>--help</code> more reliable than <code>SKILL.md</code> for the current flag list of a command?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because <code>--help</code> is fetched live from a remote server and SKILL.md is local</button>
<button class="quiz-choice" data-value="B">B) Because Bunli generates <code>--help</code> from the live Zod schemas at runtime, so it cannot drift from the source — SKILL.md is hand-maintained and may lag</button>
<button class="quiz-choice" data-value="C">C) Because SKILL.md is encrypted and only Ledger employees can read it</button>
<button class="quiz-choice" data-value="D">D) Because <code>--help</code> is generated from the <code>CHANGELOG.md</code> by a CI step</button>
</div>
<p class="quiz-explanation">Bunli's help renderer reads the <code>option(z.&hellip;(), { description })</code> calls in each <code>defineCommand</code> at runtime. Add a flag, recompile, run <code>--help</code>: the new flag is there. SKILL.md is a Markdown doc maintained by hand and is by definition slower to update.</p>
</div>

</div>

---

## Testing wallet-cli — Mock DMK and HTTP Intercept

<div class="chapter-intro">
A typical Detox spec opens a simulator, boots the app, taps buttons, and asserts on screen text. A typical Playwright spec opens Electron and asserts on the DOM. <code>wallet-cli</code> has no UI — there is nothing to tap, nothing to render. The same conceptual job (drive the system end to end against deterministic mocks of its hard dependencies) is achieved by a different stack: Bun's native test runner, a CLI runner that spawns the binary as a subprocess, a Mock DMK that pretends to be a USB Ledger, and an HTTP intercept that pretends to be the Ledger Explorer. This chapter is the Speculos-analog walk-through for CLI: every helper, every fixture, every integration test, line by line.
</div>

<a id="testing-wallet-cli-mock-dmk-and-http-intercept"></a>

### 5.6.1 Why a separate test pattern

In Part 3 (Desktop) and Part 4 (Mobile), the test infrastructure is dictated by the runner:
- Playwright drives a Chromium-or-Electron page and asserts on locators.
- Detox drives a native simulator and asserts on accessibility IDs.

`wallet-cli` is not a UI. There is no DOM, no view hierarchy, no React component tree to query. It's a Unix command: argv in, JSON envelope on stdout, exit code. The test pattern that fits a Unix command is:
1. Spawn the command as a subprocess with controlled argv and env.
2. Capture stdout/stderr/exitCode.
3. Assert on the captured stdout shape and the exit code.

So the wallet-cli test stack reaches for a different stack:
- **`bun:test`** — Bun's native test runner. Same `describe` / `it` / `expect` shape as Jest, but it runs under Bun (where the CLI itself runs).
- **`cli-runner.ts`** — a thin `Bun.spawn` wrapper that runs the actual CLI binary as a subprocess.
- **`MockDeviceManagementKit`** — a hand-rolled stand-in for DMK that returns canned task results. Lives in production code (`device/mock-dmk.ts`), installed via a test-only seam (`_setTestDmkTransport`).
- **`http-intercept.ts`** — a runtime patch of `globalThis.fetch`, `node:http`, `node:https`, and axios, redirecting all non-localhost URLs to a `Bun.serve` mock server.
- **`mock-server.ts`** — a tiny `Bun.serve`-based route table that answers Ledger Explorer / Alpaca / CAL requests with canned responses.

Together, these replicate the device-and-network environment that the CLI normally needs. A test runs in <2 seconds, deterministically, with no real hardware and no network.

The closest desktop analog is Speculos + a Playwright fixture that intercepts API calls. The closest mobile analog is the Detox bridge mock device + MSW handlers for HTTP. This stack is the same conceptual machine — different parts.

### 5.6.2 The test directory map

```
apps/wallet-cli/src/test/
├── helpers/
│   ├── cli-runner.ts          52 lines  Bun.spawn wrapper (runCli, runLocalCli)
│   ├── constants.ts            8 lines  Fixed addresses and descriptors
│   ├── dmk-intercept.ts       29 lines  Installs MockDeviceManagementKit
│   ├── eth-sync-routes.ts     37 lines  Default ETH HTTP fixture routes
│   ├── http-intercept.ts     146 lines  Patches fetch / node:http / axios
│   ├── mock-server.ts         58 lines  Bun.serve-based route table
│   ├── session-fixture.ts     27 lines  XDG_STATE_HOME tmp dir + session.yaml
│   ├── wrapper.ts              8 lines  Full intercept entrypoint
│   └── wrapper-local.ts        3 lines  No-HTTP-intercept entrypoint
└── commands/
    ├── README.md
    ├── balances.test.ts       88 lines
    ├── discover.test.ts       79 lines
    ├── operations.test.ts    125 lines
    ├── receive.test.ts        33 lines
    ├── send.test.ts           72 lines
    └── session.test.ts        77 lines
```

`helpers/` is the test infrastructure. `commands/` is the per-command integration tests — one file per command. The two `wrapper*.ts` files are the two entrypoints to the CLI under test: `wrapper.ts` enables HTTP intercept (and conditionally DMK intercept); `wrapper-local.ts` does neither (used for `session view` / `session reset` tests that touch only the local session file).

The matching `tests/commands/README.md` documents the contract:

> Each test treats a CLI command as a black box: given known flags and mocked infrastructure, the command must produce the expected stdout shape (JSON envelope) and exit code.

This is the wallet-cli analog of "given a Detox spec and a mock device, the screen must show the expected text" — but for stdout instead of a screen.

### 5.6.3 `cli-runner.ts`

This is the entry seam. Every integration test calls one of two exports: `runCli` (with HTTP intercept) or `runLocalCli` (without). Full text of the file:

```ts
// src/test/helpers/cli-runner.ts (52 lines)
import path from "node:path";

const ROOT = path.resolve(import.meta.dir, "../../..");
const WRAPPER = path.resolve(import.meta.dir, "./wrapper.ts");
const WRAPPER_LOCAL = path.resolve(import.meta.dir, "./wrapper-local.ts");

export type RunResult = {
  stdout: string;
  stderr: string;
  exitCode: number;
};

async function spawnCli(
  wrapper: string,
  args: string[],
  env: Record<string, string>,
): Promise<RunResult> {
  // --cwd (NOT the cwd: subprocess option) makes Bun resolve tsconfig and workspace packages from the wallet-cli root.
  const proc = Bun.spawn(["bun", "--cwd", ROOT, wrapper, ...args], {
    env: {
      ...process.env,
      NO_COLOR: "1",
      CLAUDECODE: "1", // triggers isInteractive() === false → disables spinner
      ...env,
    },
    stdin: "ignore",
    stdout: "pipe",
    stderr: "pipe",
  });

  const [stdout, stderr] = await Promise.all([
    new Response(proc.stdout).text(),
    new Response(proc.stderr).text(),
  ]);
  const exitCode = await proc.exited;
  return { stdout: stdout.trim(), stderr: stderr.trim(), exitCode };
}

export async function runCli(
  args: string[],
  env: Record<string, string> = {},
): Promise<RunResult> {
  return spawnCli(WRAPPER, args, env);
}

/** Like runCli but without HTTP interception — use for commands that make no network calls. */
export async function runLocalCli(
  args: string[],
  env: Record<string, string> = {},
): Promise<RunResult> {
  return spawnCli(WRAPPER_LOCAL, args, env);
}
```

Walkthrough, line by line:

- `const ROOT = path.resolve(import.meta.dir, "../../..");` — `apps/wallet-cli/`. Used as the **bun --cwd** so that tsconfig and workspace package resolution rooted there work correctly regardless of where the test is invoked from.
- `const WRAPPER = ...` and `const WRAPPER_LOCAL = ...` — the two wrapper entrypoints. We dive into them in 5.6.7.
- `Bun.spawn(["bun", "--cwd", ROOT, wrapper, ...args], { ... })` — this is the critical line. **`--cwd` here is the `bun` CLI flag**, not the Bun.spawn `cwd:` option. It tells Bun to resolve tsconfig and workspace packages from `ROOT`, while the subprocess's actual working directory inherits from the test's cwd. This split matters: tests run from the monorepo root, but bun's module resolution must be wallet-cli-rooted.
- `env: { ...process.env, NO_COLOR: "1", CLAUDECODE: "1", ...env }` — merges three layers in order. `NO_COLOR` disables ANSI codes so stdout assertions are clean. `CLAUDECODE: "1"` triggers `isInteractive() === false` (see `shared/ui.ts`), which makes the spinner a no-op proxy. The caller's `env` overrides anything before it.
- `stdin: "ignore"` — no stdin attached. The CLI does not prompt for input.
- `stdout: "pipe"`, `stderr: "pipe"` — both captured. The test asserts on stdout primarily; stderr is captured for diagnostics ("`stderr: ${stderr}`" in failure messages).
- `Promise.all([new Response(proc.stdout).text(), new Response(proc.stderr).text()])` — drains both pipes concurrently. Bun's `Response`-from-stream conversion is the idiomatic way to read a subprocess pipe to completion.
- `proc.exited` — awaited last; returns the exit code.
- `stdout.trim()` and `stderr.trim()` — drops trailing newlines, which simplifies assertions.

**Why in-subprocess, not in-process?** Because the CLI does heavy module-level setup (`embed-usb-native`, `live-common-setup`, `commands.gen` registration, transport hooks, `process.exit` calls) that can't be cleanly torn down after a single invocation. Spawning a subprocess gives each test a clean process state. The `Bun.spawn` cost is ~50-200ms per test — fast enough that running ~100 integration tests stays under a minute.

### 5.6.4 `mock-dmk.ts` (the test seam) and `dmk-intercept.ts`

`MockDeviceManagementKit` lives at `src/device/mock-dmk.ts` (191 lines). It's a minimal stand-in for the real DMK that returns canned task results. The test-side installer is `src/test/helpers/dmk-intercept.ts` — full text:

```ts
// src/test/helpers/dmk-intercept.ts (29 lines)
import type { DeviceManagementKit } from "@ledgerhq/device-management-kit";
import {
  MockDeviceManagementKit,
  type MockDeviceState,
  type MockAppResults,
} from "../../device/mock-dmk";
import { WalletCliDmkTransport } from "../../device/wallet-cli-dmk-transport";
import { _setTestDmkTransport } from "../../device/register-dmk-transport";

const stateEnv = process.env.WALLET_CLI_MOCK_DMK_STATE ?? "connected";

// Coin-specific results injected by tests via WALLET_CLI_MOCK_APP_RESULTS (JSON string).
// Shape: { "<AppName>": { <result fields> }, ... }
// Example: { "Ethereum": { "publicKey": "...", "address": "0x..." } }
const appResultsEnv = process.env.WALLET_CLI_MOCK_APP_RESULTS;
const appResults: MockAppResults = appResultsEnv
  ? (JSON.parse(appResultsEnv) as MockAppResults)
  : {};

const mock = new MockDeviceManagementKit({
  initialState: stateEnv as MockDeviceState,
  appResults,
});

const transport = new WalletCliDmkTransport(
  mock as unknown as DeviceManagementKit,
  "mock-session-id",
);
_setTestDmkTransport(transport);
```

What this file does, in order:

1. Reads two environment variables:
   - `WALLET_CLI_MOCK_DMK_STATE` — `"connected"` (default) or `"locked"`. Drives whether `getDeviceSessionState` returns CONNECTED or LOCKED.
   - `WALLET_CLI_MOCK_APP_RESULTS` — a JSON-encoded `{ "<AppName>": { <result fields> } }` map. The result fields are passed through to whatever `CallTaskInAppDeviceAction` returns. For Ethereum, the typical shape is `{ "Ethereum": { "publicKey": "...", "address": "0x..." } }` — that's what `receive.test.ts` and `discover.test.ts` use.
2. Builds a `MockDeviceManagementKit` with that initial state.
3. Wraps it in a `WalletCliDmkTransport` (the same hw-transport-compatible wrapper the production code uses).
4. Calls `_setTestDmkTransport(transport)` — the test seam exposed by `register-dmk-transport.ts`. From this point on, anywhere in live-common code that does `withDevice("wallet-cli-dmk")(...)` ends up using the mock.

What `MockDeviceManagementKit` actually mocks:

- **`listenToAvailableDevices`** — emits one fake Nano S Plus device.
- **`connect`** — returns `"mock-session-id"`.
- **`getDeviceSessionState`** — returns `CONNECTED` or `LOCKED` per `initialState`.
- **`executeDeviceAction`** — recognizes two action shapes:
  - `ConnectAppDeviceAction` (input has `application`) — completes immediately. Pretends the app opened.
  - `CallTaskInAppDeviceAction` (input has `task` function and `appName`) — returns the configured `appResults[appName]` payload. This is where `receive --verify` gets its address: the test sets `appResults.Ethereum.address`, the CLI calls `task(...)` against the EVM signer, the mock returns the configured result.
- **`sendApdu`** fallback — returns `0x6D00` (INS not supported) when no app result matches; returns `0x5515` (locked) when the state is `locked`.

What it does NOT mock:
- The XState machine inside DMK (bypassed entirely — `executeDeviceAction` is the seam).
- The `InternalApi` layer DMK uses internally.
- The actual APDU exchange. The mock returns canned results at the **task** level, so any logic that relies on intermediate APDU-level state would not be exercised.

The implication: this is good for asserting that "given Ethereum app result X, the receive command returns address X". It's not good for asserting that the CLI handles a malformed APDU response — that path is covered by E2E tests on real hardware (out of scope for this guide).

A concrete example using mock DMK end-to-end (lifted from `receive.test.ts`, which we walk in full in 5.6.9):

```ts
const { stdout, exitCode } = await runCli(
  ["receive", "--account", MOCK_ETH_DESCRIPTOR, "--verify", "--output", "json"],
  {
    WALLET_CLI_MOCK_PORT: String(server.port),
    WALLET_CLI_MOCK_DMK: "1",
    WALLET_CLI_MOCK_APP_RESULTS: JSON.stringify({
      Ethereum: { publicKey: MOCK_ETH_PUBKEY, address: MOCK_ETH_ADDRESS },
    }),
  },
);
expect(JSON.parse(stdout).address.toLowerCase()).toBe(MOCK_ETH_ADDRESS.toLowerCase());
```

The chain: test sets `appResults.Ethereum.address`, wrapper imports `dmk-intercept.ts`, intercept installs the mock, CLI's `receive --verify` opens the (mocked) device, `withCurrencyDeviceSession` runs `connectLedgerApp` (mock returns immediately), bridge calls `bridge.receive({ deviceId, verify: true })`, mock's `CallTaskInAppDeviceAction` returns the configured Ethereum result, address bubbles up to the JSON envelope. End to end, ~1 second.

### 5.6.5 `http-intercept.ts`

This is the most sophisticated helper. It patches three layers — `globalThis.fetch`, `node:http.request`, `node:https.request` — plus axios's adapter, redirecting any non-localhost URL to a local mock server. Full text walkthrough:

```ts
// src/test/helpers/http-intercept.ts (146 lines, excerpts)
import path from "node:path";

const mockPort = process.env.WALLET_CLI_MOCK_PORT;
if (!mockPort) {
  throw new Error("[http-intercept] WALLET_CLI_MOCK_PORT is not set");
}
const mockBase = `http://localhost:${mockPort}`;
```

Step 1: read the mock-server port from `WALLET_CLI_MOCK_PORT`. If unset, throw immediately — the helper refuses to silently no-op.

```ts
const liveCommonDir = path.dirname(require.resolve("@ledgerhq/live-common/package.json"));
const axiosPkgDir = path.dirname(require.resolve("axios/package.json", { paths: [liveCommonDir] }));
// Patch CJS axios instance
(require(path.join(axiosPkgDir, "dist/node/axios.cjs")) as any).defaults.adapter = "fetch";
// Patch ESM axios instance
((await import(path.join(axiosPkgDir, "index.js"))) as any).default.defaults.adapter = "fetch";
```

Step 2: force axios to use its **fetch adapter** instead of its default node:http adapter. The reason is in the comment at the top of the file: under Bun, axios's auto-detection picks the http adapter (because `process` is defined), but we want axios calls to go through the `globalThis.fetch` patch we install below — so we forcibly set `defaults.adapter = "fetch"` on **both** the CJS and ESM instances. (Bun resolves ESM imports to one path and `require()` to another, creating two separate axios singletons; both must be patched.)

```ts
function isLocal(urlStr: string): boolean {
  return (
    urlStr.startsWith("http://localhost") ||
    urlStr.startsWith("https://localhost") ||
    urlStr.startsWith("http://127.0.0.1") ||
    urlStr.startsWith("https://127.0.0.1")
  );
}
```

Step 3: helper that decides whether a URL is local (`http://localhost`, `127.0.0.1`, etc.). Local URLs are passed through unchanged — important because `Bun.serve` itself listens on localhost, and we don't want infinite redirection.

```ts
const origFetch = globalThis.fetch;
(globalThis as Record<string, unknown>).fetch = (input: RequestInfo | URL, init?: RequestInit) => {
  const urlStr =
    typeof input === "string" ? input : input instanceof URL ? input.href : (input as Request).url;
  if (urlStr && !isLocal(urlStr)) {
    const u = new URL(urlStr);
    const redirected = `${mockBase}${u.pathname}${u.search}`;
    if (input instanceof Request) {
      // Preserve the original request's method/headers/body; apply caller's init on top.
      return origFetch(new Request(redirected, input), init);
    }
    return origFetch(redirected, init);
  }
  return origFetch(input, init);
};
```

Step 4: Layer 1 — patch `globalThis.fetch`. Any non-local URL gets redirected to `http://localhost:${WALLET_CLI_MOCK_PORT}${pathname}${search}`. The path-and-query are preserved (so the mock server can route on path), but the host is replaced. When `input` is a `Request` instance, we wrap it with `new Request(redirected, input)` to preserve method, headers, and body.

```ts
const http = require("node:http");
const https = require("node:https");
const origHttpRequest = http.request.bind(http);
// (... helpers buildMockOptions, isExternalOptions, resolveHttpArgs ...)
(http as any).request = function (options: any, ...rest: any[]) {
  if (isExternalOptions(options)) {
    const [mockOpts, cb] = resolveHttpArgs(buildMockOptions(options), options, rest);
    return origHttpRequest(mockOpts as unknown as Parameters<typeof http.request>[0], cb);
  }
  return origHttpRequest(options, ...rest);
};
(https as any).request = function (options: any, ...rest: any[]) {
  const [mockOpts, cb] = resolveHttpArgs(buildMockOptions(options), options, rest);
  return origHttpRequest(mockOpts as unknown as Parameters<typeof http.request>[0], cb);
};
```

Step 5: Layer 2 — patch `node:http.request` and `node:https.request`. Same redirection logic, but at the Node http layer. **HTTPS requests are routed to plain HTTP on the mock port** — the mock server doesn't speak TLS, and we don't need it to since everything is localhost-loopback under the test runner. This layer catches anything that goes around `globalThis.fetch` (rare, but possible — direct `http.request()` callers exist in some live-common modules).

The two `resolveHttpArgs` and `buildMockOptions` helpers reconcile Node's two `http.request` overload shapes (`(options[, callback])` and `(url[, options][, callback])`) — they're plumbing details, not test concepts.

The net effect: any HTTP traffic the CLI emits (Ledger Explorer, Alpaca, CAL, gas tracker, anything) lands on the mock server, regardless of which HTTP client (fetch, axios, node:http) the live-common code happens to use under the hood.

### 5.6.6 `mock-server.ts`

The mock server is a `Bun.serve`-based route table. Full text:

```ts
// src/test/helpers/mock-server.ts (58 lines)
export type Route = {
  method?: string;
  /** URL path pattern to match (string = substring, RegExp = test against pathname+search) */
  match: RegExp | string;
  response: unknown;
  status?: number;
  headers?: Record<string, string>;
};

export class MockServer {
  private _server: ReturnType<typeof Bun.serve> | null = null;
  private _port = 0;

  constructor(private readonly routes: Route[]) {}

  start(): void {
    const { routes } = this;
    const server = Bun.serve({
      port: 0,
      fetch(req) {
        const url = new URL(req.url);
        const pathAndQuery = url.pathname + url.search;

        for (const route of routes) {
          const matches =
            typeof route.match === "string"
              ? url.pathname.includes(route.match)
              : route.match.test(pathAndQuery);

          if (matches && (!route.method || route.method === req.method)) {
            return Response.json(route.response, {
              status: route.status ?? 200,
              headers: { "Content-Type": "application/json", ...(route.headers ?? {}) },
            });
          }
        }

        console.warn(`[mock-server] UNMATCHED: ${req.method} ${url.pathname}`);
        return new Response(`[mock-server] 404 Not Found: ${req.method} ${url.pathname}`, {
          status: 404,
        });
      },
    });
    this._server = server;
    this._port = server.port ?? 0;
  }

  stop(): void {
    this._server?.stop();
    this._server = null;
    this._port = 0;
  }

  get port(): number {
    if (!this._server) throw new Error("MockServer not started");
    return this._port;
  }
}
```

Walkthrough:

- A `Route` is `{ method?, match, response, status?, headers? }`. `match` is either a string (treated as substring of pathname) or a `RegExp` (tested against `pathname + search`).
- `start()` creates a `Bun.serve` on port 0 (kernel picks an available port). The server's `fetch` handler iterates routes in order; first match wins.
- On a match, it returns `Response.json(route.response, { status, headers })`. Default status is 200, default content-type is `application/json`.
- On no match, it logs `[mock-server] UNMATCHED: ...` to stderr (so you see what was missed during a failing test) and returns 404.
- `port` is exposed once the server is listening — tests pass it via `WALLET_CLI_MOCK_PORT` to the CLI subprocess.

**`mock-server` vs `http-intercept` — when to use which:**

- `http-intercept.ts` is **always loaded** when a test uses `runCli` (not `runLocalCli`). It's the patch layer that **redirects** all non-local HTTP to the mock server.
- `mock-server.ts` is the **server** that answers the redirected requests.

You always need both for any HTTP-touching command. The split exists so the patching is one-time global and the routing is per-test programmable.

When you might prefer `mock-server` alone (without `http-intercept`): never, in practice. The patches are what make redirection automatic. If you wrote a test that wanted to `fetch('http://localhost:<port>/...')` directly, you wouldn't need the patches — but no test does that, because the production code under test doesn't.

When you might prefer `http-intercept` with **stateful** mock-server behavior: when you want to inspect what the CLI actually requested, or you need the response to depend on the request body. The simple `Route.response: unknown` shape is static, but you can pass a function-as-response in production code, or stand up your own `Bun.serve` with custom logic per the same pattern. No test does this today; all of them use the static route table.

### 5.6.7 `wrapper.ts` and `wrapper-local.ts`

These two files are the "bootstrap" that the test subprocess actually executes. They are intentionally minimal — they exist so module-level side effects of the CLI (`embed-usb-native`, `live-common-setup`, `commands.gen` registration) run **after** the intercepts are installed, not before.

`wrapper.ts` (full text):

```ts
// src/test/helpers/wrapper.ts (8 lines)
import "./http-intercept";

// When WALLET_CLI_MOCK_DMK=1, install the mock DMK transport before the CLI starts.
if (process.env.WALLET_CLI_MOCK_DMK) {
  await import("./dmk-intercept");
}

await import("../../cli");
```

Order matters:

1. `import "./http-intercept";` — top-level, synchronous. Patches `globalThis.fetch`, `node:http`, `node:https`, axios. **This must run before the CLI imports anything**, because the CLI's module-level code path may already have axios instances created (live-common eager loads), and the patch only takes effect if it runs first.
2. `if (process.env.WALLET_CLI_MOCK_DMK) await import("./dmk-intercept");` — conditional. Only installs the mock DMK when the env var is set. Tests that need the device set `WALLET_CLI_MOCK_DMK: "1"` (see `send.test.ts`, `receive.test.ts`, `discover.test.ts`); tests that only need HTTP (like `balances.test.ts`, `operations.test.ts`) leave it unset.
3. `await import("../../cli");` — finally, import the CLI itself. Its module-level setup runs now. By the time it reaches `createCLI(bunliConfig).run()`, both intercepts are in place.

`wrapper-local.ts` (full text):

```ts
// src/test/helpers/wrapper-local.ts (3 lines)
// No HTTP interception — only for commands that do not make network calls.
export {};
await import("../../cli");
```

This is for `session view` and `session reset` — commands that only read/write `session.yaml` and never hit the network. Loading `http-intercept` would force a `WALLET_CLI_MOCK_PORT` requirement that doesn't make sense for these tests. `runLocalCli` in `cli-runner.ts` points at this wrapper.

The `export {}` is just a TypeScript convention to mark this file as a module. The real work is the dynamic `await import("../../cli")`.

### 5.6.8 `constants.ts`

The fixture seeds. Tiny but load-bearing:

```ts
// src/test/helpers/constants.ts (8 lines)
export const ETH_ADDRESS = "0x71C7656EC7ab88b098defB751B7401B5f6d8976F";
export const ETH_DESCRIPTOR = `account:1:address:ethereum:main:${ETH_ADDRESS}:m/44h/60h/0h/0/0`;

// Ethereum m/44'/60'/0'/0/0 derived from the standard Hardhat/Foundry test mnemonic
// ("test test … junk") — a well-known constant in the Ethereum developer ecosystem.
export const MOCK_ETH_ADDRESS = "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266";
export const MOCK_ETH_PUBKEY = "038318535b54105d4a7aae60c08fc45f9687181b4fdfc625bd1a753fa7397fed75";
export const MOCK_ETH_DESCRIPTOR = `account:1:address:ethereum:main:${MOCK_ETH_ADDRESS}:m/44h/60h/0h/0/0`;
```

Two address pairs. Why two?

- **`ETH_ADDRESS` / `ETH_DESCRIPTOR`** — an arbitrary hard-coded Ethereum address, used by tests that **don't** go through the mock device. `balances.test.ts` and `operations.test.ts` both use this, because their test path is "fetch from Ledger Explorer (mocked HTTP) given the descriptor" — no device is opened.
- **`MOCK_ETH_ADDRESS` / `MOCK_ETH_PUBKEY` / `MOCK_ETH_DESCRIPTOR`** — the well-known **Hardhat/Foundry** test mnemonic's first Ethereum derivation. The mnemonic is `"test test test test test test test test test test test junk"`. The first ETH address derived from it (`m/44'/60'/0'/0/0`) is a famous constant in the Ethereum developer ecosystem. Tests that go through the mock device (`receive.test.ts`, `send.test.ts`, `discover.test.ts`) use this because the mock returns this address as the configured result — and we want the test's expected address to match.

The pubkey field is needed when the test sets `WALLET_CLI_MOCK_APP_RESULTS={"Ethereum":{"publicKey":"...","address":"..."}}`. Both fields go into the JSON-encoded env var.

There's also `eth-sync-routes.ts`, which provides the default Ethereum sync route fixtures (block, balance, txs, erc20 balances, CAL `/v1/currencies`). It's not in `constants.ts` but plays the same "default test data" role for HTTP. Quick reference:

```ts
// src/test/helpers/eth-sync-routes.ts — the routes any ETH bridge sync needs
export const ETH_SYNC_ROUTES: Route[] = [
  { method: "GET", match: /\/block\/current$/, response: { hash: "0x...", height: 20_000_000, time: "...", txs: [] } },
  { method: "GET", match: /\/address\/[^/]+\/balance$/, response: { balance: "0" } },
  { method: "GET", match: /\/address\/[^/]+\/txs/, response: { data: [], token: null } },
  { method: "POST", match: /erc20\/balances/, response: [] },
  { method: "GET", match: /\/v1\/currencies/, response: [{ id: "ethereum" }],
    headers: { "X-Ledger-Commit": "mock-sync-hash-0000000000000000" } },
];
```

The `/v1/currencies` route is subtle — without it (or with an empty body), the CAL store throws `LedgerAPI4xx` because an empty array is interpreted as a 404 by the CAL client. The non-empty body plus the `X-Ledger-Commit` header is what makes it valid.

### 5.6.9 A sample test end-to-end — `balances.test.ts`

This is the simplest non-trivial integration test. It exercises the HTTP intercept layer, the mock server, and the JSON envelope contract — without involving the device. Full file:

```ts
// src/test/commands/balances.test.ts (88 lines)
import { describe, it, expect, beforeAll, afterAll, afterEach } from "bun:test";
import { MockServer } from "../helpers/mock-server";
import { runCli } from "../helpers/cli-runner";
import { makeSessionDir } from "../helpers/session-fixture";
import { ETH_DESCRIPTOR, ETH_ADDRESS } from "../helpers/constants";

// 1.5 ETH in wei
const ETH_BALANCE_WEI = "1500000000000000000";

describe("balances command", () => {
  const server = new MockServer([
    {
      method: "GET",
      match: /\/address\/[^/]+\/balance$/,
      response: { address: ETH_ADDRESS, balance: ETH_BALANCE_WEI },
    },
    {
      method: "GET",
      match: /\/address\/[^/]+\/txs/,
      response: { data: [], token: null },
    },
    {
      method: "POST",
      match: /erc20\/balances/,
      response: [],
    },
  ]);

  let sessionCleanup: (() => void) | undefined;
  beforeAll(() => server.start());
  afterAll(() => server.stop());
  afterEach(() => { sessionCleanup?.(); sessionCleanup = undefined; });

  it("human output: prints ETH balance line", async () => {
    const { stdout, exitCode, stderr } = await runCli(["balances", "--account", ETH_DESCRIPTOR], {
      WALLET_CLI_MOCK_PORT: String(server.port),
    });
    expect(exitCode, `stderr: ${stderr}`).toBe(0);
    expect(stdout).toMatch(/ETH/i);
    expect(stdout).toMatch(/1\.5/);
  });

  it("json output: returns a valid balances envelope", async () => {
    const { stdout, exitCode, stderr } = await runCli(
      ["balances", "--account", ETH_DESCRIPTOR, "--output", "json"],
      { WALLET_CLI_MOCK_PORT: String(server.port) },
    );
    expect(exitCode, `stderr: ${stderr}`).toBe(0);

    const data = JSON.parse(stdout);
    expect(data.command).toBe("balances");
    expect(data.network).toBe("ethereum:main");
    expect(Array.isArray(data.balances)).toBe(true);
    expect(data.balances.length).toBeGreaterThanOrEqual(1);

    const native = data.balances.find((b: { asset: string }) => b.asset === "ethereum");
    expect(native).toBeDefined();
    expect(native.amount).toMatch(/1\.5/);
  });

  it("can resolve a session label to the matching account", async () => {
    const fixture = makeSessionDir([{ label: "ethereum-1", descriptor: ETH_DESCRIPTOR }]);
    sessionCleanup = fixture.cleanup;
    const { stdout, exitCode, stderr } = await runCli(
      ["balances", "--account", "ethereum-1", "--output", "json"],
      { WALLET_CLI_MOCK_PORT: String(server.port), ...fixture.env },
    );
    expect(exitCode, `stderr: ${stderr}`).toBe(0);
    const data = JSON.parse(stdout);
    expect(data.command).toBe("balances");
    expect(data.network).toBe("ethereum:main");
    const native = data.balances.find((b: { asset: string }) => b.asset === "ethereum");
    expect(native?.amount).toMatch(/1\.5/);
  });

  it("json output: invalid descriptor exits with code 1", async () => {
    const { stdout, exitCode } = await runCli(
      ["balances", "--account", "not-a-valid-descriptor", "--output", "json"],
      { WALLET_CLI_MOCK_PORT: String(server.port) },
    );
    expect(exitCode).toBe(1);
    // JSON mode routes all output to stdout; errors have { ok: false, ... }.
    const err = JSON.parse(stdout);
    expect(err.ok).toBe(false);
    expect(err.error.command).toBe("balances");
    expect(err.error.message).toStartWith("No account labeled");
  });
});
```

Walkthrough, top to bottom:

**Imports (lines 1-5).** `bun:test` is Bun's native test API. `MockServer`, `runCli`, `makeSessionDir`, and the constants come from the helpers. Note: no `dmk-intercept` import — `balances` is a no-device command.

**Fixture constants (line 7-8).** `ETH_BALANCE_WEI = "1500000000000000000"` is 1.5 ETH in wei. The bridge sync layer divides by `10^18` to render the human amount.

**`describe("balances command", ...)` (line 10).** A single describe block with shared mock-server fixture across four `it`s.

**`new MockServer([...])` (lines 11-27).** Three routes:
- `GET /address/.../balance` — returns 1.5 ETH for any address. The regex `[^/]+` matches any address segment (the actual address is parameterized by descriptor).
- `GET /address/.../txs` — empty transaction history. Required by the bridge sync to determine token sub-accounts (token detection scans recent txs).
- `POST /erc20/balances` — empty array. The Alpaca direct-API path for token balances.

**Lifecycle hooks (lines 29-32).** `beforeAll` starts the server (so all `it`s share one port). `afterAll` stops it. `afterEach` cleans up the per-test session-dir fixture if one was set.

**`it("human output: prints ETH balance line", ...)` (lines 34-41).** The simplest assertion. Spawns the CLI with `["balances", "--account", ETH_DESCRIPTOR]` and `WALLET_CLI_MOCK_PORT=<server.port>` in env. Checks:
- `exitCode === 0`. The second arg to `expect(... , msg)` is the failure message — including stderr in the message means a failing test gets you the device-handshake / error messages for free.
- `stdout matches /ETH/i` — the human formatter prints the asset symbol.
- `stdout matches /1\.5/` — and the human-readable amount.

**`it("json output: returns a valid balances envelope", ...)` (lines 43-59).** Same but with `--output json`. Asserts the envelope shape:
- `data.command === "balances"`.
- `data.network === "ethereum:main"`.
- `data.balances` is a non-empty array.
- The "ethereum" asset entry has `amount` matching `/1\.5/` — the formatter's human-readable string `"1.5 ETH"`.

**`it("can resolve a session label to the matching account", ...)` (lines 61-74).** Tests the session-label resolver:
1. `makeSessionDir([{ label: "ethereum-1", descriptor: ETH_DESCRIPTOR }])` creates a temp `XDG_STATE_HOME` containing a `session.yaml` mapping `ethereum-1` → the descriptor.
2. The CLI is run with `--account ethereum-1` (no `:`, so the resolver looks up the label).
3. `XDG_STATE_HOME` is forwarded via `env` so the CLI's session store hits the temp dir.
4. Same assertions as the previous test — proves that label resolution produces the same descriptor-driven path.

**`it("json output: invalid descriptor exits with code 1", ...)` (lines 76-87).** Negative path. Passes a label that doesn't exist in any session. Asserts:
- `exitCode === 1`.
- stdout (NOT stderr — JSON mode writes errors on fd 1 too) is parseable JSON with `{ ok: false, error: { command, message } }` shape.
- The error message starts with `"No account labeled"` — the exact prefix from `commands/inputs.ts::resolveAccountInput`.

This last test is the canonical "JSON error envelope contract" assertion. It documents in code that:
1. Errors exit with non-zero code.
2. Errors land on stdout in JSON mode (not stderr).
3. The shape is `{ ok: false, error: { command, message } }` — note the absence of `status: "success"`.

Anyone consuming wallet-cli JSON in a hook script can parse `data.ok === false` to detect errors uniformly across commands.

### 5.6.10 A complex test — `send.test.ts`

This one exercises the full stack: HTTP intercept, mock DMK, route fixtures for gas estimation, all the way through `prepareTransaction`. It's the densest example in the repo. Full file:

```ts
// src/test/commands/send.test.ts (72 lines)
import { describe, it, expect, beforeAll, afterAll } from "bun:test";
import { MockServer } from "../helpers/mock-server";
import { runCli } from "../helpers/cli-runner";
import { ETH_SYNC_ROUTES } from "../helpers/eth-sync-routes";
import { MOCK_ETH_DESCRIPTOR, MOCK_ETH_ADDRESS, MOCK_ETH_PUBKEY } from "../helpers/constants";

// Routes needed by prepareTransaction (gas estimation, nonce, fee data) in addition to sync.
// The balance route is placed first (before ETH_SYNC_ROUTES) so it overrides the default
// zero-balance route — the send command requires sufficient balance to avoid NotEnoughBalance.
const SEND_ROUTES = [
  {
    method: "GET",
    match: /\/address\/[^/]+\/balance$/,
    response: { balance: "1000000000000000000" }, // 1 ETH
  },
  ...ETH_SYNC_ROUTES,
  {
    method: "GET",
    match: /\/address\/[^/]+\/nonce$/,
    response: { address: MOCK_ETH_ADDRESS, nonce: 0 },
  },
  {
    method: "POST",
    match: /\/tx\/estimate-gas-limit/,
    response: { estimated_gas_limit: "21000" },
  },
  {
    method: "GET",
    match: /\/gastracker\/barometer/,
    response: { low: "1", medium: "2", high: "3", next_base: "1" },
  },
];

describe("send --dry-run command", () => {
  const server = new MockServer(SEND_ROUTES);

  beforeAll(() => server.start());
  afterAll(() => server.stop());

  it("json output: exits 0 and returns a dry-run send envelope for ETH", async () => {
    const { stdout, exitCode, stderr } = await runCli(
      [
        "send",
        "--account",
        MOCK_ETH_DESCRIPTOR,
        "--to",
        "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
        "--amount",
        "0.001 ETH",
        "--dry-run",
        "--output",
        "json",
      ],
      {
        WALLET_CLI_MOCK_PORT: String(server.port),
        WALLET_CLI_MOCK_DMK: "1",
        WALLET_CLI_MOCK_APP_RESULTS: JSON.stringify({
          Ethereum: { publicKey: MOCK_ETH_PUBKEY, address: MOCK_ETH_ADDRESS },
        }),
      },
    );
    expect(exitCode, `stderr: ${stderr}`).toBe(0);

    const data = JSON.parse(stdout);
    expect(data.command).toBe("send");
    expect(data.network).toBe("ethereum:main");
    expect(data.dry_run).toBe(true);
    expect(typeof data.recipient).toBe("string");
    expect(typeof data.amount).toBe("string");
    expect(typeof data.fee).toBe("string");
  });
});
```

The dance, line by line:

**`SEND_ROUTES` (lines 10-32).** This is the route table for the entire EVM `prepareTransaction` flow. Five routes:

1. `GET /address/.../balance` — returns 1 ETH (1e18 wei). **Placed before `ETH_SYNC_ROUTES`** so it shadows the default zero-balance route. Why ordering matters: `MockServer` iterates routes in declaration order, first match wins. Without this override, the bridge sync would see balance=0 and the send would fail with `NotEnoughBalance` before even reaching the gas estimate step.
2. `...ETH_SYNC_ROUTES` — block, txs, erc20, CAL. The standard ETH sync background.
3. `GET /address/.../nonce` — returns nonce 0. Required by the EVM bridge to set the transaction's nonce field.
4. `POST /tx/estimate-gas-limit` — returns `21000` (the standard ETH transfer gas limit). Bridge calls this during `prepareTransaction`.
5. `GET /gastracker/barometer` — returns fee tiers (low/medium/high/next_base). The bridge picks one based on the user's selected fee strategy (default: medium).

**`describe("send --dry-run command", ...)` (line 34).** Note the test is `--dry-run` only. The actual signing path is **not** covered by an integration test in the repo today — the comment in R1 flags this as a gap.

**The `runCli` call (lines 41-62).** The CLI args:
- `send --account <descriptor> --to 0x70997970... --amount '0.001 ETH' --dry-run --output json`.
- The `--to` address is the second derivation of the same Hardhat mnemonic — another well-known testnet address.
- `--amount '0.001 ETH'` is the canonical "send a small amount" pattern.
- `--dry-run` skips the device-signing step but still builds the validated transaction.

The env:
- `WALLET_CLI_MOCK_PORT` — points the HTTP intercept at the mock server.
- `WALLET_CLI_MOCK_DMK: "1"` — installs the mock DMK transport.
- `WALLET_CLI_MOCK_APP_RESULTS` — JSON-encoded `{ Ethereum: { publicKey, address } }`. Even in dry-run, the CLI's `withCurrencyDeviceSession` stage runs (it opens the Ethereum app on device for signing), so the mock needs to know the Ethereum app result. Note: the test predicate `--dry-run` short-circuits **after** prepareTransaction but **before** signOperation — so the mock's CallTaskInApp is configured but the task that would actually use it (sign) is never reached.

Wait — looking more carefully at the source, the `send.ts` handler in the dry-run branch **does** call `wallet.prepareSend(descriptor, intent)` without entering `withCurrencyDeviceSession`. So in this specific test, the mock DMK env is set defensively (in case the dry-run path ever changes to require device prep), but it isn't strictly exercised. The HTTP routes are the load-bearing fixtures here.

**Assertions (lines 63-69).** The dry-run envelope:
- `command === "send"`.
- `network === "ethereum:main"`.
- `dry_run === true`.
- `recipient`, `amount`, `fee` are all strings (the human-readable amount formatting — `"0.001 ETH"`, `"0.0003 ETH"`).

What this test does NOT cover (and what an extended `revoke` test would need to cover, per Section 12 of R1):
- The actual signing path (CallTaskInApp returning a signed-tx blob).
- The broadcast step (POST to `/tx/send`).
- The `tx_hash` in the success envelope.

Adding integration coverage for the post-sign path is its own piece of work; the current test stops at the prepareTransaction boundary. For a future `revoke` command (the QAA-615 spike), a richer test would mock the EVM signer's `signTransaction` task to return a canned signed-tx blob and intercept the `tx/send` POST to assert the calldata starts with `0x095ea7b3` (the `approve` selector) and the second 32-byte word is zero.

### 5.6.11 Coverage and `bun test --coverage`

The `coverage` script in `apps/wallet-cli/package.json` is:

```bash
bun test src/ --coverage
```

`bunfig.toml` sets the reporter:

```toml
[test]
coverageReporter = ["lcov"]
coverageDir = "coverage"
```

Run from `apps/wallet-cli/`:

```bash
pnpm coverage
# or directly:
bun test src/ --coverage
```

What it does:
- Discovers every `*.test.ts` under `src/`.
- Runs each test under coverage instrumentation.
- Writes `lcov.info` to `apps/wallet-cli/coverage/`.

The Nx target wraps this:

```bash
pnpm --filter @ledgerhq/wallet-cli coverage
```

CI consumes the lcov file and produces a coverage badge / report. There is no explicit threshold gate in `bunfig.toml` today — coverage is informational, not enforced. (If you want to add a gate, you'd extend the script with `--coverage-threshold=NN`; Bun supports this flag but the project hasn't opted in.)

A useful local pattern: combine coverage with a single test file:

```bash
bun test src/test/commands/balances.test.ts --coverage
```

This shows you the coverage delta from one test only — handy when authoring a new test and you want to see what lines you've started covering.

### 5.6.12 Common assertion shapes

Across the six integration tests, three assertion shapes recur. Recognize them and you can author new tests by template.

**Shape 1 — JSON success envelope:**

```ts
expect(exitCode, `stderr: ${stderr}`).toBe(0);
const data = JSON.parse(stdout);
expect(data.command).toBe("<command-name>");
expect(data.network).toBe("ethereum:main");
expect(data.<payload-key>).toBeDefined();
```

This is the canonical happy-path assertion. The `stderr: ${stderr}` failure message is **important** — when the test fails on a CI machine, you want stderr in the failure log immediately. Without it, you'd have to add console logs and re-run.

**Shape 2 — JSON error envelope:**

```ts
expect(exitCode).toBe(1);
const err = JSON.parse(stdout); // not stderr — see 5.5.6
expect(err.ok).toBe(false);
expect(err.error.command).toBe("<command-name>");
expect(err.error.message).toStartWith("<expected-prefix>");
// or:
expect(err.error.message).toMatch(/<regex>/);
```

The `ok: false` shape is the agreed contract. The error message comes from the source — usually a Zod validation error or a hand-raised `Error()` in the command handler.

**Shape 3 — Side effect on the file system (session-yaml tests):**

```ts
const fixture = makeSessionDir([]); // or with seed entries
cleanup = fixture.cleanup;
await runCli([...], { ...env, ...fixture.env });
const sessionPath = join(fixture.env.XDG_STATE_HOME, "ledger-wallet-cli", "session.yaml");
const session = YAML.parse(await Bun.file(sessionPath).text()) as { accounts: ... };
expect(session.accounts).toEqual(...);
```

This pattern reads the YAML written by the CLI and asserts on its content. Used by `discover.test.ts` for the "session persistence" tests. Bun's `YAML.parse` is built-in (`import { YAML } from "bun"`) — no third-party dep needed.

**Shape 4 — human-mode pattern matching:**

```ts
expect(stdout).toMatch(/ETH/i);
expect(stdout).toMatch(/1\.5/);
expect(stdout).toContain("ethereum-1");
```

For human-mode assertions you don't pin to exact strings (formatting may change). You assert that the **expected substrings** are present. The `i` flag is common — case-insensitive matching for asset tickers.

### 5.6.13 The "no e2e/cli/" rationale

Detox tests live under `e2e/mobile/`. Playwright tests live under `e2e/desktop/`. There's a strong precedent in this monorepo for "E2E tests are a peer workspace alongside the app they test". So why aren't the wallet-cli integration tests under `e2e/cli/`?

The answer is what they actually test:

- **`e2e/desktop/`** — drives the real Electron binary against a real Speculos. Real native build, real Speculos container, end-to-end UI flows.
- **`e2e/mobile/`** — drives the real native simulator via Detox. Real build, real bridge, end-to-end UI flows.
- **`apps/wallet-cli/src/test/`** — spawns the CLI in-process under Bun.spawn, with a **mock** DMK and a **mock** HTTP server. No real device, no real network.

The wallet-cli tests are **integration tests**, not E2E. They cover the command-handler-to-output contract with all hard dependencies replaced. That's a different category from desktop/mobile E2E:

| Layer | Real Speculos? | Real device-or-USB? | Real HTTP? | Workspace location |
|---|---|---|---|---|
| `e2e/desktop/` | Yes | Speculos | Yes (test endpoints) | peer to `apps/ledger-live-desktop/` |
| `e2e/mobile/` | Yes (when `--e2e`) | Speculos / mock-bridge | Yes | peer to `apps/ledger-live-mobile/` |
| `apps/wallet-cli/src/test/` | No | Mock DMK | Mock server | colocated with source |

So the wallet-cli tests live **alongside the source** under `apps/wallet-cli/src/test/`, not under a peer `e2e/cli/` workspace. The ownership model matches: the wallet-cli package owns these tests, and they ship in the same package's `bun test src/` invocation.

If a real-USB-device E2E suite ever materializes for wallet-cli — for instance, a CI pipeline with a USB-attached Speculos box driving wallet-cli through real DMK — that suite **would** warrant an `e2e/cli/` peer workspace. It would have:
- A separate `package.json` with its own dependencies.
- An entry in the root `pnpm-workspace.yaml`.
- A CI job that requires a USB-bus runner.
- Tests that **don't** mock DMK (the whole point of having a real-device E2E layer).

That's a significant infrastructure investment, and as of QAA-615 it doesn't exist. The current strategy is "fast deterministic mocks under `bun test`", and the source-tree colocation reflects that.

> **Tip:** If you ever find yourself wanting to write a "real-device" wallet-cli test (driving an actual Nano X), do not add it to `apps/wallet-cli/src/test/`. That directory is the mock-DMK contract layer; mixing real-device tests there confuses the contract. Either propose a new `e2e/cli/` workspace, or test through the bash hook itself (see Chapter 5.7 for the hook pattern).

### 5.6.14 Chapter 5.6 Quiz

<!-- ── Chapter 5.6 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> Why does <code>cli-runner.ts</code> spawn the CLI as a subprocess instead of importing it in-process?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because <code>bun:test</code> doesn't support in-process imports</button>
<button class="quiz-choice" data-value="B">B) Because the CLI is written in JavaScript and the tests are written in TypeScript</button>
<button class="quiz-choice" data-value="C">C) Because the CLI does heavy module-level setup (USB native embed, live-common-setup, command registration, <code>process.exit</code>) that can't be cleanly torn down between tests — a fresh subprocess gives each test a clean process state</button>
<button class="quiz-choice" data-value="D">D) Because Bun cannot import its own <code>.ts</code> files at runtime</button>
</div>
<p class="quiz-explanation">The CLI calls <code>process.exit</code>, registers transport modules globally in live-common, embeds a native USB addon, and runs Bunli's command-store registration as a side effect. None of that survives an in-process re-import cleanly. <code>Bun.spawn</code> per test costs ~50-200ms but guarantees isolation.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q2.</strong> What does <code>WALLET_CLI_MOCK_DMK=1</code> do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Tells <code>wrapper.ts</code> to import <code>dmk-intercept.ts</code>, which installs a <code>MockDeviceManagementKit</code> via the <code>_setTestDmkTransport</code> seam</button>
<button class="quiz-choice" data-value="B">B) Disables all device output entirely</button>
<button class="quiz-choice" data-value="C">C) Switches the CLI to JSON-only output mode</button>
<button class="quiz-choice" data-value="D">D) Forces the real DMK to use a deterministic random seed</button>
</div>
<p class="quiz-explanation"><code>wrapper.ts</code> conditionally <code>await import("./dmk-intercept")</code> when <code>WALLET_CLI_MOCK_DMK</code> is set. The intercept builds a <code>MockDeviceManagementKit</code> from <code>WALLET_CLI_MOCK_APP_RESULTS</code> and calls <code>_setTestDmkTransport</code> on the production module's test seam.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q3.</strong> Why does <code>http-intercept.ts</code> patch axios's adapter to <code>"fetch"</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because <code>fetch</code> is faster than <code>node:http</code> for benchmark-driven tests</button>
<button class="quiz-choice" data-value="B">B) Because under Bun, axios auto-detects <code>node:http</code> as its default adapter (<code>process</code> is defined), but the helper's <code>globalThis.fetch</code> patch only catches fetch traffic — forcing axios to <code>"fetch"</code> routes its calls through the patch</button>
<button class="quiz-choice" data-value="C">C) Because Bun does not implement <code>node:http.request</code> at all</button>
<button class="quiz-choice" data-value="D">D) Because the mock server only speaks fetch protocol</button>
</div>
<p class="quiz-explanation">Axios picks an adapter at config time. Under Node, it's the http adapter. Under Bun, it's still the http adapter (because Bun emulates <code>process</code>). The fetch patch wouldn't catch axios traffic without forcing <code>defaults.adapter = "fetch"</code> on both the CJS and ESM axios instances. The node:http patch is the second, redundant layer for anything that still bypasses fetch.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> In <code>send.test.ts</code>, why is the balance route placed BEFORE <code>ETH_SYNC_ROUTES</code> in <code>SEND_ROUTES</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because routes are matched in alphabetical order by path</button>
<button class="quiz-choice" data-value="B">B) Because <code>ETH_SYNC_ROUTES</code> is read-only and cannot be modified</button>
<button class="quiz-choice" data-value="C">C) Because Bun.serve only matches the first route that responds successfully</button>
<button class="quiz-choice" data-value="D">D) Because <code>MockServer</code> matches routes in declaration order, first match wins — and <code>ETH_SYNC_ROUTES</code> includes a default zero-balance route. Placing the 1-ETH override first shadows the default so the send doesn't fail with <code>NotEnoughBalance</code></button>
</div>
<p class="quiz-explanation"><code>MockServer.start</code> iterates the <code>routes</code> array in order. The 1-ETH balance route must come first to override the default zero-balance route in <code>ETH_SYNC_ROUTES</code>; otherwise the bridge sync sees balance=0 and fails the dry-run with <code>NotEnoughBalance</code>.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q5.</strong> What's the difference between <code>runCli</code> and <code>runLocalCli</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>runCli</code> uses <code>wrapper.ts</code> (which loads <code>http-intercept</code>); <code>runLocalCli</code> uses <code>wrapper-local.ts</code> (no HTTP intercept) — used for commands that make no network calls (<code>session view</code>, <code>session reset</code>)</button>
<button class="quiz-choice" data-value="B">B) <code>runCli</code> runs against the production CLI; <code>runLocalCli</code> runs against a stripped-down local fork</button>
<button class="quiz-choice" data-value="C">C) <code>runCli</code> requires a real USB device; <code>runLocalCli</code> uses the mock DMK</button>
<button class="quiz-choice" data-value="D">D) They're identical and only differ in name</button>
</div>
<p class="quiz-explanation"><code>runCli</code> spawns the CLI through <code>wrapper.ts</code>, which always loads <code>http-intercept</code> and conditionally loads <code>dmk-intercept</code>. <code>runLocalCli</code> uses <code>wrapper-local.ts</code>, which loads neither. The latter is for tests that only touch the local <code>session.yaml</code> — loading <code>http-intercept</code> would require <code>WALLET_CLI_MOCK_PORT</code> to be set, which doesn't make sense for those tests.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> Why don't the wallet-cli integration tests live under <code>e2e/cli/</code>, the way Detox tests live under <code>e2e/mobile/</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because <code>e2e/cli/</code> is reserved for the legacy <code>apps/cli</code> tests</button>
<button class="quiz-choice" data-value="B">B) Because Bun cannot run tests outside the package they live in</button>
<button class="quiz-choice" data-value="C">C) Because they are integration tests with all hard dependencies mocked (no real DMK, no real network) — that's a different category from <code>e2e/desktop</code>/<code>e2e/mobile</code>, which exercise real Speculos/devices. They live colocated with the source under <code>apps/wallet-cli/src/test/</code> and the wallet-cli package owns them. A real-USB E2E suite would warrant a peer <code>e2e/cli/</code> workspace, but that doesn't exist today</button>
<button class="quiz-choice" data-value="D">D) Because the wallet-cli has no CLI to test — it's only a library</button>
</div>
<p class="quiz-explanation">The <code>apps/wallet-cli/src/test/commands/README.md</code> and the test stack itself make this explicit: every external dependency (DMK, HTTP) is replaced. That's an integration-test category, not E2E. Colocation with the source matches the ownership model. If a real-USB E2E suite ever materializes (Speculos-on-USB driving the real DMK transport), it would justify a peer <code>e2e/cli/</code> workspace at that point — until then, the colocated <code>src/test/</code> tree is the right location.</p>
</div>

</div>

---
## Daily CLI Workflow

<a id="daily-cli-workflow"></a>

<div class="chapter-intro">
You have read the architecture (Ch 5.1), inspected every command (Ch 5.4), absorbed the descriptor V1 schema (Ch 5.4), traced the device session lifecycle through DMK (Ch 5.5), and walked the test harness end to end (Ch 5.6). This chapter ties all of that into the <strong>workflow</strong> &mdash; the exact, repeatable sequence you will run every time you touch <code>apps/wallet-cli/</code>. It mirrors Chapter 4.9 (Daily Mobile Workflow) but tracks the CLI realities: a Bunli command tree under <code>src/commands/</code>, a codegen step that CI enforces, a <code>cli-runner</code> Bun.spawn harness with mocked DMK, an <code>oxlint</code> + <code>oxfmt</code> lint pair, and the device-touching subset of work that needs <code>dangerouslyDisableSandbox: true</code> to run through the agent shell. Follow this every time. The steps do not change just because the change feels small.
</div>

### 5.7.1 The Ticket Lifecycle for CLI Work

Every piece of work on `apps/wallet-cli/` starts in Jira and ends in either an Xray traceability update or, more commonly today, a CI gate on `pnpm generate:check` plus `pnpm test`. The lifecycle has the same **three ticket shapes** as Part 4, with one CLI-specific nuance.

| Ticket type | Project | Example | Purpose |
|---|---|---|---|
| **QAA ticket** | `QAA` | `QAA-615` | A QA work item &mdash; "spike: add revoke command", "add hook to clean allowance state". |
| **LIVE ticket** | `LIVE` | `LIVE-29495` | A platform-side ticket &mdash; the wallet-cli is co-developed with the Wallet XP team, who track work in `LIVE`. CLI bugs (e.g. `LIVE-29404` &mdash; `--dry-run` silently dropped) live there. |
| **B2CQA test case** | `B2CQA` | `B2CQA-2844` | An Xray scenario &mdash; rare for CLI work today (no Xray plan exists yet for wallet-cli, see R4 §7.5), but used when a CLI change unblocks a previously un-automatable B2CQA scenario. |

The CLI-specific nuance: **your QAA ticket is often _infrastructure_, not test code**. QAA-615 ("spike: revoke token approval using CLI") is not a test &mdash; it's a tool that future tests will call from a `before`/`after` hook. That distinction shapes the AC: you are not writing assertions, you are exposing a primitive. Reviewers expect a working command, a JSON envelope shape, mocked tests, and (when applicable) a smoke against a real device.

```
QAA-615 (QA infrastructure ticket)
   │
   │ enables
   ▼
QAA-613 (nightly token-approval flow)            ◄── consumes the new revoke
   │
   │ runs against
   ▼
apps/wallet-cli/src/commands/revoke.ts
   │
   │ tested in
   ▼
apps/wallet-cli/src/test/commands/revoke.test.ts (mocked DMK)
   │
   │ wired through
   ▼
.bunli/commands.gen.ts (codegen committed)
```

Skip the codegen commit and CI fails on `pnpm generate:check`. Skip the mocked test and CI fails on `pnpm test`. Skip the Xray link &mdash; if your QAA has one &mdash; and your work is invisible coverage, same cardinal sin called out in Ch 4.9.

### 5.7.2 Orientation &mdash; The Five-Minute Recon

You have been assigned the ticket. Before opening any source file, do the **five-minute recon**. It catches more "I wrote it twice" mistakes than any other habit in this part.

```bash
# 0. Land in the workspace
cd apps/wallet-cli

# 1. Read the package.json scripts top to bottom
cat package.json | jq '.scripts'

# 2. Get the live command tree from the binary, not from your memory
pnpm --silent wallet-cli start -- --help
pnpm --silent wallet-cli start -- account --help
pnpm --silent wallet-cli start -- send --help

# 3. Scan src/commands/ to confirm the file layout matches the help tree
ls src/commands/
ls src/commands/account/
ls src/commands/session/

# 4. Has a recent PR touched the area you are about to touch?
git log --oneline --since="3 weeks" -- src/commands/

# 5. Are the codegen and tests currently green on dev?
pnpm test --silent | tail -20
pnpm generate:check
```

The answers shape the day:

- **Help shows your command already exists** &rarr; you are extending it, not creating it. Open the existing file; do not start a new one.
- **Help shows a sibling close to what you need** (e.g. `send` exists, you want `revoke`) &rarr; copy the canonical shape from that file, do not invent one.
- **`pnpm generate:check` is dirty before you've changed anything** &rarr; stop. `dev` is broken. Fix that first, or rebase off the green commit.
- **`pnpm test` is red** &rarr; same. Do not pile your work on top of a red baseline.

Write the outcome of the recon as 5 bullets in the Jira ticket. Your reviewer reads those bullets and immediately knows why your diff looks the way it does.

### 5.7.3 Picking the Change Shape &mdash; Decision Tree

Most CLI tickets fit into one of three shapes. Pick the smallest that satisfies the AC. Anything bigger is over-engineered.

```
                         ┌─────────────────────────────────────────┐
                         │  Does the command you need already exist? │
                         └─────────────┬───────────────────────────┘
                                       │
                ┌──────────────────────┴──────────────────────┐
              YES                                              NO
                │                                              │
                ▼                                              ▼
      ┌──────────────────────┐                  ┌──────────────────────────┐
      │ Is a flag missing?   │                  │ Is the change shape a    │
      └──────┬───────────────┘                  │ thin wrapper around an   │
             │                                  │ existing command?        │
       ┌─────┴──────┐                           └────────┬─────────────────┘
      YES           NO                                   │
       │             │                            ┌──────┴──────┐
       ▼             ▼                          YES            NO
  ┌─────────┐  ┌──────────┐                      │              │
  │ Add to  │  │ Add to   │                      ▼              ▼
  │ command │  │ inputs.ts│            ┌─────────────────┐  ┌────────────┐
  │ options │  │ if       │            │ New file under  │  │ New leaf   │
  │ block.  │  │ shared   │            │ src/commands/   │  │ command +  │
  │         │  │ across   │            │ that calls into │  │ family     │
  │         │  │ commands.│            │ existing send/  │  │ intent     │
  │         │  │          │            │ balances logic. │  │ (heaviest) │
  └─────────┘  └──────────┘            └─────────────────┘  └────────────┘
```

Concrete examples:

| Ticket shape | Example | Surface | Effort |
|---|---|---|---|
| Add a flag to one command | `--abort-timeout` on `send` | Modify `send.ts` options schema, plumb to `wallet-cli-dmk-transport.ts`. | ~20 LOC + 1 unit test |
| Add a flag shared across commands | `--rpc-url <url>` on every device-free command | Add to `src/commands/inputs.ts`, import from each leaf. | ~30 LOC + 1 unit test |
| Wrapper command | `revoke` &rarr; calls `send --data 0x095ea7b3...` | New file `src/commands/revoke.ts`, calls into the same EVM intent builder. No new APDU code. | ~80 LOC + 1 mocked + 1 smoke |
| Net-new family-aware leaf | `stake delegate` on a new chain | New file, new family entry in `wallet/intents/families/`, new bridge mapping. | ~250 LOC + tests |

QAA-615's sweet spot is the third row (wrapper). The EVM intent already accepts `--data` (see Ch 5.4); a `revoke` wrapper builds the calldata and forwards. That's why R1's headline finding stands: revoke is a thin wrapper, not new infrastructure.

### 5.7.4 Local Iteration Loop

The CLI's edit-run-edit cycle is the fastest of any surface in this guide. There is no Metro bundler, no app to relaunch, no Detox build &mdash; just `bun run`. Aim for **sub-second feedback**.

```bash
# Tight loop, no device, no network
pnpm --silent start -- balances ethereum-1 --output json | jq .

# With debug logging
DEBUG=wallet-cli pnpm --silent start -- balances ethereum-1

# Hot-reload mode (Bunli)
pnpm dev
```

The `--output json | jq .` pipe is the single most useful habit you can build. Three reasons:

1. **JSON is parseable.** Visual diffs of two runs are trivial with `jq`.
2. **Human output is for humans.** The spinner, colors, and aligned columns are TTY-mode noise &mdash; they make grepping hard.
3. **CI talks JSON.** Every hook future-you writes will pipe wallet-cli output to `jq`. Train the muscle now.

Inside an AI-agent shell (Claude Code, Cursor, Codex), `isInteractive()` from `src/shared/ui.ts` returns `false` because of the `CLAUDECODE` / `CURSOR_AGENT` / `CODEX_ENABLED` env detection. Spinners no-op, output is clean. That is intentional; the test runner relies on it (Ch 5.6).

A productive cycle looks like:

```bash
# Tab 1: keep the help/output handy
watch -n 1 'pnpm --silent start -- revoke --help 2>&1 | tail -30'

# Tab 2: run the test you're writing
bun test src/test/commands/revoke.test.ts --watch

# Tab 3: edit the source file
$EDITOR src/commands/revoke.ts
```

If a single change requires more than 30 s to validate, you are doing too much per iteration. Shrink the change.

### 5.7.5 Writing Tests as You Go &mdash; TDD-Lite

The wallet-cli test harness is mature enough that **mocked tests must come first**. The pattern is canonical TDD-lite (red &rarr; green &rarr; refactor) but tightened for CLI:

1. **Write the failing test against the mock harness.** `Bun.spawn` spawns the CLI with `WALLET_CLI_MOCK_DMK=1` and your mock app result. Assert the JSON envelope shape you want.
2. **Watch it fail meaningfully.** The error should be "command not found" or "missing flag X" &mdash; not a JSON parse error or a sandbox crash. If it crashes, fix the harness first.
3. **Make it pass with the smallest change.** One option, one handler stub, the JSON envelope correct.
4. **Refactor.** Move shared helpers to `src/commands/inputs.ts`. Extract the calldata builder into a pure function. Move the test fixtures to `test/helpers/constants.ts`.

```bash
# The TDD-lite loop
bun test src/test/commands/revoke.test.ts --watch
```

**No real device for steps 1&ndash;3.** The mock DMK seam (`src/device/mock-dmk.ts` &mdash; see Ch 5.6) returns whatever you put in `WALLET_CLI_MOCK_APP_RESULTS`. For a `revoke` test, that's the same Ethereum app shape `send` uses today (`{ "Ethereum": { signature: "0x..." } }`). Lift the fixture from `test/commands/send.test.ts` &mdash; it already wires the routes, the http-intercept, and the session.

The `test/commands/README.md` line is the contract:

> Each test treats a CLI command as a black box: given known flags and mocked infrastructure, the command must produce the expected stdout shape (JSON envelope) and exit code. All external I/O is replaced &mdash; no real device or network needed.

Assert exactly two things per test, no more:

1. The exit code (0 for success, 1 for the failure case under test).
2. The JSON envelope: `command`, `network`, `account`, plus the command-specific fields (`tx_hash`, `dry_run`, `removed`, etc.).

Three or more assertions per test is a smell &mdash; split the test.

### 5.7.6 Smoke Against a Real Device

Only after the mocked tests are green do you plug a device in. Real-device smoke is **not a substitute** for mocked tests; it is the final acceptance check that the APDU shape your mock asserted matches what the embedded app actually expects.

Pre-conditions, every time:

1. Real Ledger plugged in via USB (Nano S Plus, Nano X, Stax, or Flex).
2. Device unlocked with PIN.
3. The relevant coin app **manually opened** on device. For Base, that's the Ethereum app &mdash; the `withCurrencyDeviceSession()` helper maps `evm` family &rarr; `ethereum` managerAppName regardless of the EVM chain.
4. **Only one wallet-cli command at a time.** Two parallel device commands fight over the USB session and one will hang. The internal-testing page (R4 §A.2) is explicit: "Always run device commands one by one &mdash; never in parallel."

Running through Bash with the agent harness is sandboxed by default. USB is outside the sandbox, so a device-touching command will fail with a misleading "Timeout has occurred" error. Set `dangerouslyDisableSandbox: true` for that single Bash call, no more:

```text
Bash command:           pnpm --silent wallet-cli start -- send ethereum-1 --to 0x... --amount '0.001 ETH'
dangerouslyDisableSandbox: true
```

The pattern is: every device-touching invocation gets the flag; every mocked test does not. Do not blanket-enable the flag for the whole session &mdash; the sandbox is your safety net for the 95 % of work that doesn't need it.

After the run, **verify the clear-sign content on device**. The Ethereum app should display the calldata fields decoded by the ERC-7730 metadata service (DMK fetches them; see Ch 5.5 and the DMK overview in R4 §A.3). If the device shows a raw hex blob, that's blind-sign mode &mdash; either the metadata is missing for the contract or the origin token is not embedded. Note it in the ticket; do not silently approve.

### 5.7.7 Codegen Check

If you added or removed a command file under `src/commands/`, you **must** run codegen and commit the result:

```bash
pnpm generate
git add .bunli/commands.gen.ts
```

The generated file (`.bunli/commands.gen.ts`, ~215 lines today) statically imports every command module and registers it before `cli.run()` executes. The CI gate is `pnpm generate:check`, which runs `bunli generate` and then `git diff --exit-code -- .bunli/commands.gen.ts`. If your local generated file disagrees with the committed one, the build fails.

Common gotchas:

- **Adding a new file but forgetting `pnpm generate`** &rarr; CI red.
- **Renaming an existing file** &rarr; codegen import path changes; you must commit the regenerated file.
- **Editing `.bunli/commands.gen.ts` by hand** &rarr; the file's first line says "DO NOT EDIT". Your edit will be overwritten on the next `pnpm generate`. Do not.
- **Adding a flag to an existing command but not changing the file structure** &rarr; you do **not** need to rerun `pnpm generate`. The codegen captures the command tree, not flag-level details (those live in the source).

Treat `pnpm generate` like an autoformatter: run it whenever you change command file structure, commit the result, never edit the output by hand.

### 5.7.8 Lint and Typecheck

The wallet-cli uses `oxlint` (linter) and `oxfmt` (formatter) &mdash; the Rust-toolchain replacements for ESLint and Prettier. They are 10&ndash;100x faster, which is why the workspace adopted them, and they share a config with the rest of the monorepo (`.oxlintrc.json` and `.oxfmtrc.json` at the package root, plus a shared `../../.oxfmtrc.json`).

```bash
# In apps/wallet-cli/
pnpm lint           # oxlint src bunli.config.ts
pnpm lint:fix       # oxfmt + oxlint --fix
pnpm format         # oxfmt apply
pnpm format:check   # oxfmt check (CI gate)
pnpm typecheck      # tsc --noEmit -p tsconfig.json
```

Run the full triplet before you push:

```bash
pnpm lint && pnpm typecheck && pnpm test
```

If `lint` complains and you genuinely cannot fix it (rare &mdash; the rules are deliberate), prefer a single-line `// oxlint-disable-next-line <rule>` over disabling at file scope. Reviewers will ask why you disabled the rule.

### 5.7.9 Session Storage and Labels &mdash; Currently In Flight

This subsection is **informational, not load-bearing**. The wallet-cli's persistent state layout is in active flux between two designs. Do not depend on either path in tests &mdash; the tests redirect via `XDG_STATE_HOME` and a tmpdir (see `test/helpers/session-fixture.ts`, Ch 5.6).

**What ships today (LIVE-29495).** The `session.yaml` file is written via `@bunli/utils stateDir(APP_NAME)` where `APP_NAME = "ledger-wallet-cli"`. On Linux/macOS that resolves to:

```
$XDG_STATE_HOME/ledger-wallet-cli/session.yaml
   default:  ~/.local/state/ledger-wallet-cli/session.yaml
```

Schema (Zod-validated):

```yaml
accounts:
  - label: ethereum-1
    descriptor: account:1:address:ethereum:main:0x71C7...:m/44h/60h/0h/0/0
  - label: bitcoin-native-1
    descriptor: account:1:utxo:bitcoin:main:xpub6BosfCn...:m/84h/0h/0h
```

This is what `account discover` writes after a successful run, what `session view` reads, and what `resolveAccountInput()` in `src/commands/inputs.ts` consults whenever a flag value lacks a `:` (label lookup mode).

**What the ADR proposes (TA/6978928805, "ADR &mdash; CLI Storage", _Proposed_).** A different path:

```
~/.ledger/cli/<file>.yaml
```

with an open question about whether `deviceId` lives inside the descriptor (Solution 1) or in a context file (Solution 2). The ADR is **proposed**, not accepted. R4 §A.6 captures the divergence:

> "Two competing storage layouts in flight."

**What this means for you, today.**

- For tests: never read `~/.local/state/...` directly. Use `makeSessionDir(...)` from `test/helpers/session-fixture.ts` and pass the returned `XDG_STATE_HOME` env to the subprocess. That isolates each test.
- For commands: read/write through `Session.read()` / `session.write()` in `src/session/session-store.ts`. Never hard-code the path.
- For docs: cite "session storage layout" without committing to a directory. When the ADR lands, one find-and-replace updates the guide.
- For PR review: if a reviewer asks "why is the path not `~/.ledger/cli/`?", the answer is "LIVE-29495 shipped first; the ADR is proposed; we await the consolidation ticket."

This is an **honest gap**, not a bug. The codebase has the right indirection (`stateDir(APP_NAME)`) so that flipping to the ADR path is a one-line change.

### 5.7.10 Commit and PR

Conventional Commits, scoped to `cli`:

| Type | Use for |
|---|---|
| `feat(cli):` | A new command or a new user-visible flag. |
| `fix(cli):` | A bug fix on an existing command. |
| `test(cli):` | Adding or improving tests, no production change. |
| `refactor(cli):` | Restructure without behavior change (e.g. extract a calldata builder). |
| `chore(cli):` | Tooling, config, codegen-only commits, dependency bumps. |
| `docs(cli):` | Documentation-only (README, in-source comments, agents skill). |

Branches follow the global rules from `git-workflow.md` (kebab-case, prefix indicates type):

```bash
# New command
git checkout -b feat/cli-qaa-615-revoke-command

# Bug fix
git checkout -b fix/cli-live-29404-dry-run-flag

# Test additions only
git checkout -b test/cli-revoke-mocked-coverage

# Refactor
git checkout -b refactor/cli-extract-calldata-builder
```

The `cli-` segment in the branch name (paralleling Part 4's `llm-`, Part 3's `lld-`) tells reviewers the surface without opening the diff.

PR description checklist (paraphrased from the global PR template):

1. **Title** &mdash; `feat(cli): add revoke command (QAA-615)`. One line, references the ticket.
2. **Jira link** at the top of the description.
3. **Acceptance Criteria** restated, with each item checkbox-ticked.
4. **Approach summary** &mdash; one paragraph: "I added `revoke.ts` as a thin wrapper around `send`'s EVM intent. The calldata is built by `buildErc20RevokeCalldata(spender)`. New mocked test passes; smoke against a Nano S Plus + Ethereum app v1.16 succeeded; clear-sign content showed `Approve` with amount 0."
5. **Test evidence** &mdash; output of `pnpm test`, screenshot of the device clear-sign screen if relevant, CI run link.
6. **CODEOWNERS** auto-requests &mdash; sanity-check the list (see 5.7.11).

For ticket flow detail (assignee transitions, "In Code Review" status moves, the `/create-pr` skill), see Part 0 Ch 0.5.

### 5.7.11 Reviewer Routing

Always look up the live owners in `.github/CODEOWNERS` &mdash; do not paste a stale list from this guide.

```bash
# From the monorepo root
grep -nE "^/?apps/wallet-cli/" .github/CODEOWNERS
grep -nE "^/?apps/cli/" .github/CODEOWNERS
grep -nE "wallet-cli|live-dmk-shared" .github/CODEOWNERS
```

The wallet-cli owners (verify before opening the PR &mdash; this list is a snapshot, not the contract):

| Path | Likely owners |
|---|---|
| `apps/wallet-cli/` | wallet-ci team + Wallet XP (`@ledgerhq/wallet-xp`) |
| `apps/wallet-cli/src/device/` | DMK platform owners (often `@ledgerhq/wallet-xp` or a dedicated DMK team) |
| `apps/cli/` (legacy) | live-common platform owners |
| `libs/live-dmk-shared/` | DMK platform owners |

GitHub's branch protection auto-requests owners. Sanity-check that the list contains, at minimum, the wallet-ci team. If your diff bleeds into `libs/ledger-live-common/src/families/evm/` (e.g. you needed a new helper in `getTokenAllowance.ts`), expect the EVM coin module owners to be added &mdash; that team's review takes longer; budget for it.

If the auto-request misses an owner you know should be there (because their CODEOWNERS rule is stale or your diff is novel), add them manually. Reviewers prefer being added explicitly to discovering a stealth diff three days later.

### 5.7.12 Chapter 5.7 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz &mdash; Daily CLI Workflow</h3>
<p class="quiz-subtitle">6 questions &middot; 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> You added a new file <code>src/commands/revoke.ts</code> and your mocked tests pass locally. CI still fails. What is the most likely cause?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The Ethereum app version on the CI device differs from yours</button>
<button class="quiz-choice" data-value="B">B) <code>oxfmt</code> auto-formatted your file differently in CI</button>
<button class="quiz-choice" data-value="C">C) You forgot to run <code>pnpm generate</code> and commit <code>.bunli/commands.gen.ts</code> &mdash; the <code>generate:check</code> gate fails</button>
<button class="quiz-choice" data-value="D">D) <code>bun test</code> behaves differently under Linux</button>
</div>
<p class="quiz-explanation">Adding or removing a file under <code>src/commands/</code> changes the codegen output. <code>generate:check</code> runs <code>bunli generate</code> in CI and fails if the committed file disagrees. This is the single most common red CI cause for wallet-cli PRs.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> You want to test a new <code>revoke</code> command. The first test you write is:</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Plug a device, open Ethereum app, run the command live, and snapshot the output</button>
<button class="quiz-choice" data-value="B">B) A <code>bun test</code> file under <code>src/test/commands/revoke.test.ts</code> using <code>cli-runner</code> + mocked DMK + http-intercept; assert the JSON envelope shape and exit code</button>
<button class="quiz-choice" data-value="C">C) A unit test of <code>buildErc20RevokeCalldata()</code> only, no end-to-end</button>
<button class="quiz-choice" data-value="D">D) Run it through the agent shell with <code>dangerouslyDisableSandbox: true</code> and pipe to <code>jq</code></button>
</div>
<p class="quiz-explanation">Mocked-DMK integration tests are the canonical first artifact (see Ch 5.6 and the contract in <code>test/commands/README.md</code>). Real-device smoke is the final acceptance check, not the first iteration. Pure unit tests (C) are useful but do not verify the JSON envelope or exit code &mdash; they cover only one layer.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> You run a device-touching command through the agent's Bash tool and get <code>Timeout has occurred</code>. Most likely cause?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The Bash call ran inside the sandbox, which blocks USB. Re-run with <code>dangerouslyDisableSandbox: true</code></button>
<button class="quiz-choice" data-value="B">B) The wallet-cli is hanging on a metadata fetch from the CAL service</button>
<button class="quiz-choice" data-value="C">C) The DMK transport is not registered</button>
<button class="quiz-choice" data-value="D">D) The Ethereum app is too old for ERC-7730</button>
</div>
<p class="quiz-explanation">The sandbox blocks USB device access. Without <code>dangerouslyDisableSandbox: true</code>, DMK can't open the device session and times out after the connect timeout (60s). This is exercise 5.9.6 below &mdash; reproducing the error consciously is the best way to internalize the rule.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> Where does <code>session.yaml</code> live today?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>~/.ledger/cli/session.yaml</code></button>
<button class="quiz-choice" data-value="B">B) <code>~/.config/ledger-wallet-cli/session.yaml</code></button>
<button class="quiz-choice" data-value="C">C) <code>./session.yaml</code> (relative to the working directory)</button>
<button class="quiz-choice" data-value="D">D) <code>$XDG_STATE_HOME/ledger-wallet-cli/session.yaml</code> (default <code>~/.local/state/ledger-wallet-cli/session.yaml</code>) &mdash; LIVE-29495 via <code>@bunli/utils stateDir()</code></button>
</div>
<p class="quiz-explanation">LIVE-29495 ships the XDG state path. The ADR (TA/6978928805) proposes <code>~/.ledger/cli/</code> but is still <em>Proposed</em>. The codebase reads/writes through <code>stateDir(APP_NAME)</code> so a future flip to the ADR layout is a one-line change. Do not depend on the literal path in tests &mdash; use <code>makeSessionDir(...)</code> from the test helpers.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> You need to add a flag <code>--abort-timeout</code> shared across <code>send</code>, <code>receive</code>, and <code>account discover</code>. Where do you put the flag definition?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Inline the same option block into all three command files</button>
<button class="quiz-choice" data-value="B">B) Define it once in <code>src/commands/inputs.ts</code> and import it into the three commands</button>
<button class="quiz-choice" data-value="C">C) Add it to <code>.bunli/commands.gen.ts</code></button>
<button class="quiz-choice" data-value="D">D) Add it as an environment variable handled at process start</button>
</div>
<p class="quiz-explanation"><code>src/commands/inputs.ts</code> is the canonical home for shared option helpers (the briefing called this <code>shared-options.ts</code>; the file was renamed). Three duplicated blocks (A) drift; <code>commands.gen.ts</code> (C) is generated; an env-var (D) bypasses the help system. The shared helper keeps <code>--help</code> consistent across commands.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> You open a PR titled <code>add revoke command</code>. The reviewer asks for a Conventional Commits-style title. Which is correct?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>add(cli): revoke command</code></button>
<button class="quiz-choice" data-value="B">B) <code>WALLET-CLI: add revoke command</code></button>
<button class="quiz-choice" data-value="C">C) <code>feat(cli): add revoke command (QAA-615)</code></button>
<button class="quiz-choice" data-value="D">D) <code>[QAA-615] add revoke</code></button>
</div>
<p class="quiz-explanation">Type (<code>feat</code>) + scope (<code>cli</code>) + imperative description + Jira ID in parens. <code>add</code> is not a Conventional Commits type. <code>WALLET-CLI:</code> and bracketed prefixes are not the convention. The format is enforced by Lerna semantic-release rules.</p>
</div>

<div class="quiz-score"></div>
</div>

<div class="chapter-outro">
<strong>Key takeaway.</strong> The CLI workflow is: pick a QAA ticket &rarr; recon the workspace in 5 minutes &rarr; pick the smallest change shape that satisfies the AC &rarr; tight loop with <code>--output json | jq</code> &rarr; mocked tests first, real device only after green &rarr; <code>pnpm generate</code> if you touched the command tree &rarr; lint, typecheck, test &rarr; <code>feat(cli):</code> commit &rarr; PR with the wallet-ci team auto-requested. The same dance, every ticket. Sub-second feedback is the design goal &mdash; if you're waiting more than 30 s per iteration, shrink the iteration.
</div>

---


## Walkthrough QAA-615 — Implementing wallet-cli revoke

<div class="chapter-intro">
Your CLI capstone, but with a twist. Earlier walkthroughs (Part 4 Ch 4.10, Part 5 Ch 5.4) shipped a complete solution and asked you to follow along. This chapter is different: <strong>you will write the code yourself</strong>. The chapter walks you to the door of every phase — ticket context, architecture, helpers in the monorepo, path tradeoffs, test plan, PR shape — but the final lines of TypeScript are yours to compose. Each phase ends with a question to resolve or a skeleton to fill, not a finished file to copy. Treat it as a Software Carpentry tutorial: scaffolded, not pre-baked. The reward is that when you finish, you will <em>understand</em> the wallet-cli stack the way you understand the desktop and mobile stacks — by having built something inside it.
</div>

> **A note on tone.** This chapter assumes you have read Chapter 5.1 (wallet-cli landscape), Chapter 5.2 (Bunli + DMK + bridge adapter), and at least skimmed the QAA-702 walkthrough in Part 4 Ch 4.10. If you skipped those, the helper names and Bunli idioms below will look unfamiliar. Bounce back, then return.

### 5.8.1 Understanding the Ticket

**Jira ticket:** QAA-615 — *Spike: Revoke token approval using CLI*
**Type:** Task (Spike) · **Status:** In Progress · **Assignee:** Jerome PORTIER · **Reporter:** Gabriel BECERRA
**Parent epic:** QAA-919 *"SWAP regression coverage automation"*
**Linked to:** QAA-613 *(Discovery — Connected)*
**Labels:** `LLD`, `LLM`, `UI`

The ticket text, verbatim:

> Hey team. As we have the objectif of testing the whole flow with token approval + swap, we need to find a way to revoke the approval to be able to run the test again. This will be helpful for automatic regression testing and not nighlty executions.

Read it twice. Three things are worth pausing on before you write a line of code.

**This is a spike, not a feature ticket.** The QAA team uses the word "spike" the way most engineering teams do — a time-boxed investigation whose deliverable is *understanding plus a proof of concept*, not necessarily a polished, fully reviewed shippable feature. A spike answers the question *"is this possible, how would we do it, and what are the tradeoffs?"* before the team commits to a fully-scoped ticket. The PR you will eventually open for QAA-615 will include working code, but its primary deliverable is the **written report** (Section 5.8.12 below). The code is evidence; the report is the deliverable.

**The labels are mismatched with the work.** This is a CLI-only investigation, but the ticket carries `LLD` (Ledger Live Desktop), `LLM` (Ledger Live Mobile), and `UI` labels. Welcome to real Jira hygiene — labels were inherited from the parent epic at creation time and nobody re-labeled. Don't let it confuse you: the work lives entirely inside `apps/wallet-cli/`. If a reviewer asks why a CLI spike has `UI` on it, point them at the parent epic's labels.

**The "regression" word matters.** The reporter explicitly contrasts *automatic regression testing* with *nightly executions*. That single distinction is the load-bearing constraint of the whole spike — see Section 5.8.2 for why.

**Pavlo's "2 flows + edge cases" comment.** On the parent ticket QAA-919, Pavlo OKHONKO commented that the regression suite needs to exercise *two distinct swap flows* (provider routes — Thorchain, Uniswap, LiFi) plus their edge cases. Each flow opens with an `approve(spender = router, amount = N)` and ends with the swap itself. To run any single flow more than once on the same address, you must reset the allowance — which is exactly what `wallet-cli revoke` is for. The hook is per-flow, not per-suite: each flow's `beforeEach` revokes its own router's allowance before retrying.

### 5.8.2 Why This Matters — the regression-suite vs nightly distinction

Victor's Slack message on 2026-04-24, in French, sets the architectural picture:

> "yes en gros c'est pcq l'approve du token c'est a faire que 1x. et apres c'est good. donc faut ajouter un hook (before ou after) le test pour clean justement. et je crois que c'est potentiellement possible de faire ca avec le CLI mais faut investiguer."

Translated: *"yes basically because the token approve only needs to be done once. After that it's good. So we need to add a before/after hook to the test to clean up. And I think it's potentially possible to do this with the CLI but it needs investigation."*

That message is your charter. Two suites, two behaviours:

1. **The nightly suite (QAA-613)** — broadcast disabled. The Detox/Speculos test boots, fakes its way through the approve+swap signature flow on a synthetic chain that never receives the broadcast. On-chain allowance is never actually mutated. The next nightly run starts from the same on-chain state because nothing was published. **Revoke is not strictly necessary here** — the test starts each night from a clean slate by virtue of broadcast=off.
2. **The broadcast-enabled regression suite** — broadcast on. The same test code, but pointing at a real testnet (or, occasionally, mainnet on a test seed) where the broadcast actually publishes. The first run leaves an `approve(router, N)` on chain. The second run's `approve` no-ops on the device's UI ("you already approved this") and the test misreads the state. The suite is no longer deterministic. **Revoke is the cleanup hook** that pulls the on-chain allowance back to zero so the next iteration is identical to the first.

The CLI is **test-data infrastructure**, not test code. It is not invoked by the spec; it is invoked by the harness *around* the spec. Pseudo-code:

```bash
beforeEach() {
  wallet-cli revoke ethereum-1 --token USDC --spender $UNISWAP_V3_ROUTER --output json
  wait_for_confirmation
}
runSpec()
```

That hook is the entire reason this command exists. Every other concern — pretty output, JSON envelope, dry-run mode — bends to that contract.

#### ERC-20 approval primer (compressed)

If you have already read Chapter 0.5 or Chapter 4.10's intro, skim this. Otherwise, the 80-line version:

ERC-20 tokens do not "live" at user addresses the way native ETH does. The ledger is the token contract: a mapping from address → balance, indexed by the user's address. To move USDC, the user signs a transaction whose `to` is the USDC contract and whose calldata is `transfer(recipient, amount)`. The contract checks the sender's balance and updates two ledger entries.

A **decentralised exchange** (Uniswap, Thorchain, LiFi) cannot move the user's tokens directly — the user holds them, the DEX does not. Instead, the DEX needs *permission* to pull tokens out of the user's address into its own router. The ERC-20 standard exposes this as `approve(spender, amount)`: the user signs an `approve(router_address, N)` transaction telling the token contract "the address `router_address` may withdraw up to `N` of my tokens, on my behalf." The router records the allowance in another mapping inside the contract. Then the swap transaction calls the router's `swap()` function, which internally calls the token's `transferFrom(user, router, amount)` — succeeds because the allowance is in place.

So a real swap is **two device signatures, in sequence**: first the approve, then the swap. Each provider has its own router contract, so the same user must approve once per (token, router) pair before any swap through that router will succeed.

To **revoke** an approval, the user signs `approve(spender = same_router, amount = 0)`. Same calldata shape, same selector (`0x095ea7b3`), only the amount field changes from N to 0. The contract overwrites the previous allowance entry with zero. Subsequent `transferFrom` calls from that router fail with `ERC20: insufficient allowance`.

The clear-sign UX on Ledger devices recognises the `approve` selector specifically and renders a friendly "Type: Approve / Amount: 0 USDC / Address: 0x...router" screen instead of raw hex. That is true for any token in the cryptoassets registry — no per-token plugin work needed.

`wallet-cli revoke` is therefore a single function call: build the calldata `approve(spender, 0)`, hand it to the existing send pipeline as a transaction whose `to` is the token contract, sign and broadcast. The hard parts are not the cryptography — they are the **plumbing** (where to slot the command, how to resolve the token contract address from a ticker, what the JSON envelope should look like for the hook).

### 5.8.3 What's Already in the Monorepo

Before you write any new code, inventory what already exists. Every helper below is a file you should open in your editor as you read this section.

**The ERC-20 calldata builder — already shipped.**
`libs/coin-modules/coin-evm/src/logic/getErc20Data.ts:10-14`

```ts
export function getErc20ApproveData(spender: string, amount: bigint): Buffer {
  const contract = new ethers.Interface(ERC20ABI);
  const data = contract.encodeFunctionData("approve", [spender, amount]);
  return Buffer.from(data.slice(2), "hex");
}
```

Pure function. Returns the 4-byte selector `0x095ea7b3` followed by 64 bytes of ABI-encoded `(spender, amount)`. Total 68 bytes. For revoke, call it with `amount = 0n`.

**The allowance reader — already shipped.**
`libs/ledger-live-common/src/families/evm/getTokenAllowance.ts:29-71`

```ts
export async function getEvmTokenAllowance(
  account: Account,
  tokenId: string,
  spender: string,
): Promise<GetEvmTokenAllowanceResult>
```

Resolves the token via the cryptoassets store, validates the chain match, calls the unified node API's `getTokenAllowance` (which under the hood routes to Ledger's `/blockchain/v4/.../contract/read` endpoint or — for local dev — to a JSON-RPC `eth_call` against `allowance(owner, spender)`). Returns `{ allowance, unit, symbol, tokenId, owner, spender, contractAddress }`.

You will need this in Section 5.8.9 when you build the optional `wallet-cli allowance` companion command.

**The legacy CLI's revoke — your reference implementation.**

- `apps/cli/src/commands/blockchain/tokenAllowance.ts` — read command (`commander`-style). The legacy equivalent of what you'll build for verification.
- `libs/coin-modules/coin-evm/src/cli-transaction.ts:10` — declares `const modes = ["send", "revokeApproval", "approve"] as const;`
- `libs/coin-modules/coin-evm/src/cli-transaction.ts:95-130` — `inferTransactions` for the revoke/approve modes. The shape it produces is your blueprint:

```ts
return {
  ...rest,
  mode: "send" as const,            // EVM "send" + custom data == arbitrary contract call
  recipient: tokenContractAddress,  // tx.to = the token contract, NOT the spender
  amount: new BigNumber(0),         // no native value
  useAllAmount: false,
  data,                             // ERC-20 approve(spender, amount) calldata
};
```

Three load-bearing facts in that block:

1. `tx.to` is the **token contract address**, not the spender, not the owner.
2. `tx.value` is **zero** — revoke moves no native ETH.
3. `tx.data` is the bytes from `getErc20ApproveData(spender, amountApproved)`.
4. The high-level `mode` flattens to `"send"`. coin-evm's transaction shape only knows `send | erc721 | erc1155`. The CLI mode names (`approve`, `revokeApproval`) are pure UX affordances; they collapse to `send` before signing because the device decides what to render based on the calldata selector, not the wallet-side mode.

> **Verify:** open `libs/coin-modules/coin-evm/src/cli-transaction.ts` in your editor and confirm the lines above match what you see. Research notes were captured at one moment in time; the file may have moved or been refactored. If the structure shifted, the principles above still hold — only the line numbers change.

**The wallet-cli send pipeline — already accepts custom calldata.**

- `apps/wallet-cli/src/commands/send.ts` — the command shape you will mirror.
- `apps/wallet-cli/src/wallet/intents/families/evm.ts` — the EVM intent schema. Already has an optional `data?: "0x..."` field. R5's audit calls it *"the only extensibility hook on the EVM intent today"* — which means the plumbing for arbitrary contract calls is already wired through, and revoke just rides on top.
- `apps/wallet-cli/src/wallet/compatibility/bridge.ts:194-198` — converts hex `data` from the intent into a `Buffer` and patches it onto the live-common transaction:

```ts
case "evm":
  if (intent.data) {
    const hexData = intent.data.slice(2);              // strip 0x
    if (hexData.length > 0) patch.data = Buffer.from(hexData, "hex");
  }
  break;
```

That is the seam. Anything you put in `intent.data` rides through the bridge unmodified, gets RLP-encoded inside `signOperation`, and lands as the calldata field of the on-chain transaction.

**The signing path — also already in place.** `apps/wallet-cli/src/wallet/compatibility/bridge.ts:151-184` walks `signOperation` → `broadcast`. wallet-cli does *not* call DMK's `signEthereumTransaction` directly; it goes through the live-common AccountBridge abstraction, which internally drives the DMK signer. You inherit clear-sign, gas estimation, nonce management, EIP-1559 fees, and broadcast for free.

**The interactive-debugging escape hatch — already exists.**

`send.ts`'s `--data 0x...` flag, today, can already produce a valid revoke transaction. Try this in a terminal (do not run it yet — read first):

```bash
wallet-cli send ethereum-1 \
  --to 0xdAC17F958D2ee523a2206206994597C13D831ec7 \
  --amount '0 ETH' \
  --data 0x095ea7b300000000000000000000000058df81bababd293aaca0a3a76d70d3b69b25c1cf0000000000000000000000000000000000000000000000000000000000000000
```

That command builds and broadcasts an approve(0x58df…, 0) on USDT mainnet. If you wonder *why are we even writing a revoke command then?* — keep reading. Section 5.8.4 unpacks why this is fine for a debugging session and unfit for a regression-suite hook.

### 5.8.4 Picking the Implementation Path

R2's spike report compares three paths. Read it in full at `_r2_revoke_spike.md` before committing — these are summaries, not substitutes. The three options:

**Path A — dedicated `revoke` subcommand.** New file `commands/revoke.ts`, new entry in `cli.ts`, full prepare→sign→broadcast plumbing duplicated locally. Cleanest UX, most code. Estimated ~150 LOC + ~80 tests.

**Path B — generalize `send` with raw `--calldata`.** Already shipped; the example at the end of 5.8.3 works today. Zero new code in wallet-cli. Tradeoff: every E2E hook has to construct 68 bytes of hex by hand. Easy to mistype `0x095ea7b3` as `0x095ea7b30` (one stray digit shifts the entire ABI offset). Output logs are coupled to hex, making `grep` patterns in the test harness brittle. Estimated 0 LOC of code, ~30 LOC of doc.

**Path C — `revoke` thin wrapper over `send`.** New file `commands/revoke.ts`, but its handler is a near-clone of `send.ts`'s handler that internally constructs the EVM `TransactionIntent` (with `recipient = tokenContract`, `amount = "0 <ticker>"`, `data = approveCalldata`) and delegates to the same `WalletAdapter.send` path. Zero changes to the bridge, the intent schema, or live-common. Estimated ~150 LOC code + ~80 LOC tests, **~225 LOC total**.

R2's recommendation: **Path C**. Rationale:

- ✓ Justified — Path B is implemented but its UX is unfit for unattended hooks (long hex strings, no semantic logging, easy footgun). Victor's hook framing is the load-bearing constraint and rules out B.
- ✓ Justified — Path C reuses 100% of the signing/broadcast plumbing. The new code is a *command shell*, not a new wallet path.
- ~ Contestable — Paths A and C are technically equivalent; the distinction is whether you also factor out a shared `runEvmContractCall(intent)` helper. R2 defers that until `approve` lands so the abstraction is informed by 3 callers (send, approve, revoke), not 2.
- ✗ Unjustified — adding a new EVM transaction `mode` discriminator (`"approve"` | `"revoke"`) to either coin-evm or the wallet-cli intent schema. The clear-sign decoder already keys off the calldata selector and the bridge already routes `data` correctly. A new `mode` would be premature uniformity with no UX gain.

**Now think for yourself.** Before adopting Path C wholesale, steelman the alternatives:

- *What's the strongest argument for Path B?* (Hint: zero new code, zero review surface, shipped today. Forces the hook author to understand the calldata, which has pedagogical value.)
- *What's the strongest argument for Path A?* (Hint: a dedicated command is testable in isolation; you can mock `WalletAdapter.send` and verify revoke calls it with the right intent. With Path C, the test surface is a near-clone of `send.test.ts`.)
- *Is there a fourth path you haven't seen?* (E.g. a single command that takes `--mode revoke|approve` and dispatches internally — the legacy CLI's shape. Why didn't R2 list it?)

> **Decision exercise.** Before you read on, write a one-paragraph note in your scratchpad explaining which path you would ship and why. If your reasoning matches Path C, fine. If you land somewhere else, *write down* what would have to be true for your choice to be the right one. That note is the seed of the spike report you will produce in Section 5.8.12.

The rest of this chapter assumes Path C. If you decide to ship Path A or Path B instead, the phases below still apply with minor renaming.

### 5.8.5 Phase 1 — Read the Legacy Precedent

Before you sketch the new file, open the legacy CLI's revoke implementation in your editor. This is homework, not reading material — answer the questions below as you go.

**Files to open:**

- `apps/cli/src/commands/blockchain/tokenAllowance.ts` — the read-side allowance command.
- `apps/cli/src/commands/blockchain/send.ts` — the legacy send command, which dispatches `--mode revokeApproval`.
- `libs/coin-modules/coin-evm/src/cli-transaction.ts` — the EVM family's CLI handler that produces the actual `Transaction`.

**Questions to answer (write the answers in your scratchpad):**

1. **What's the input shape for a revoke vs an approve in the legacy CLI?** Find where `mode === "revokeApproval"` is checked. Note the difference between revoke (no `--approveAmount` needed) and approve (parses `--approveAmount`, supports the literal string `"unlimited"` mapping to `2^256 - 1`).
2. **What does the legacy CLI do about gas estimation?** Does it pass an explicit `gasLimit`, or does it leave it to `bridge.prepareTransaction`? What does this tell you about how much you need to manage in the wallet-cli command?
3. **How does the legacy CLI resolve the token from a CLI argument?** Is the input a ticker (`USDT`), a token id (`ethereum/erc20/usd_tether__erc20_`), a contract address (`0xdAC1...`), or all three? What helper does the resolution? (Look for `findTokenById` or similar around the legacy `inferTransactions`.)
4. **What does the spender argument look like — `--spender` flag, positional, prompted?** Does the legacy CLI validate the address format? What does it do when validation fails?
5. **What happens if the user passes a token whose `parentCurrency.id` doesn't match the account's currency?** (Trace the validation chain. The error message in the legacy CLI is verbatim what you should reproduce — the test harness above wallet-cli will likely depend on it.)

You don't need to memorise these answers — you need to know *where to look* the next time someone asks. The legacy CLI is going away, but as long as it ships, it is your reference implementation. The wallet-cli rewrite is supposed to behave the same way externally; if your `revoke` command surprises a user who has muscle memory from the legacy CLI, you have a UX bug.

> **Verify:** the line numbers in this section are pulled from research at one snapshot; if your monorepo branch is newer or older, the symbols may have moved. Search by name (`grep -r "revokeApproval" libs/coin-modules/coin-evm/src/`) before quoting line numbers in your PR.

### 5.8.6 Phase 2 — Sketch the Bunli Command Skeleton

Open a new file at `apps/wallet-cli/src/commands/revoke.ts`. Do not paste a complete implementation — draft the skeleton below into your editor and start filling the TODOs one at a time.

```ts
// apps/wallet-cli/src/commands/revoke.ts
import { defineCommand, option } from "@bunli/core";
import { z } from "zod";
// TODO: import lastValueFrom + tap from rxjs (look at send.ts for the pattern)
// TODO: import the WalletAdapter and the EVM intent schema
// TODO: import the device-session helper used by send.ts
// TODO: import the shared command-output helpers (createCommandOutput, etc.)
// TODO: import the calldata builder you decide on (Q1 below)
// TODO: import the account-resolving helpers (resolveAccountArg, resolveAccountDescriptor)

export default defineCommand({
  name: "revoke",
  description: "TODO: write a one-line description; see send.ts for tone and verbosity",
  options: {
    // TODO: account option — same shape as send.ts (positional + --account flag)
    // TODO: --token option — accept ticker OR token id (see Q2 below)
    // TODO: --spender option, regex-validated 0x-prefixed 40-hex-char address
    // TODO: --dry-run boolean flag
    // TODO: --output option (text | json) — reuse the existing outputOption
  },
  handler: async (ctx) => {
    // TODO: parse account descriptor (delegate to resolveAccountDescriptor)
    // TODO: validate that the account family is "evm" — throw a clean error otherwise
    // TODO: resolve the token's contract address (see Q3 below)
    // TODO: build approve(spender, 0) calldata using getErc20ApproveData
    // TODO: construct the EVM TransactionIntent ({ family: "evm", recipient: <token contract>, amount: "0 <ticker>" or "0 ETH", data: "0x..." })
    // TODO: if --dry-run, call wallet.prepareSend(descriptor, intent) and print the envelope; else open the device session and call wallet.send(...)
    // TODO: print a human-readable line OR the JSON envelope, depending on --output
  },
});
```

Now resolve, in your scratchpad, these five questions before you write the bodies. Each has a hint pointing at a real file.

**Q1. Where do you get `getErc20ApproveData` from?**

Two options:
- Direct deep import from `@ledgerhq/coin-evm`:

  ```ts
  import { getErc20ApproveData } from "@ledgerhq/coin-evm/logic/getErc20Data";
  ```

  Pro: zero duplication, identical bytes to what production uses. Con: deep imports past package barriers are typically discouraged in this monorepo. Check `tsconfig.json` `paths` and `package.json` `exports` to confirm whether the deep path is actually exposed.

- Local re-implementation in `apps/wallet-cli/src/wallet/erc20-calldata.ts` using `ethers.Interface` directly (5 lines — copy from `getErc20Data.ts`).

  Pro: clean dep boundary. Con: tiny duplication. R2 actually recommended viem; R3 (which read the actual monorepo) reports the entire EVM family standardises on `ethers` v6 and there is no viem in the workspace. Stick with ethers unless your team has decided otherwise.

  > **Verify:** check `apps/wallet-cli/package.json` to see whether `ethers` is a direct dep or only transitive via `@ledgerhq/coin-evm`. If transitive, decide: add it explicitly, or rely on the workspace alias (`workspace:^`) to surface it.

**Q2. Where do you register the command, and how does Bunli pick it up?**

Bunli's CLI surface is wired in `apps/wallet-cli/src/cli.ts`. Look at the line that registers `send`. Add an analogous line for revoke. Then run `pnpm generate` (or whatever the wallet-cli's codegen command is — verify with the README) to regenerate any auto-generated types or help text. Forgetting this step means your command compiles but `wallet-cli --help` doesn't list it, and the harness can't find it either.

> **Verify:** the precise codegen command in this monorepo. R1 mentions `pnpm generate` but workspaces vary. `cd apps/wallet-cli && pnpm run` will list available scripts.

**Q3. How do you map `--token USDC` to a contract address?**

Three options, ordered from cleanest to most-pragmatic:

- *(a) Require a full token id* (e.g., `ethereum/erc20/usd__coin`). No sync needed — direct lookup via `getCryptoAssetsStore().findTokenById(tokenId)`. UX cost: users have to know the token id format. The format is documented in `libs/cryptoassets/src/data/tokens.ts` (or similar).
- *(b) Accept a ticker, sync the account, look up `subAccounts[].token.contractAddress`*. Mirrors how `parseAmountWithTicker` already works in `apps/wallet-cli/src/wallet/intents/parse-amount.ts`. Costs one sync (~1-2s) per revoke. UX win.
- *(c) Accept the contract address directly* (`--contract 0xdAC1...`). Bypass the token registry entirely. Useful when the token isn't in the registry; loses the clear-sign rendering on the device because the device cannot resolve the token's ticker.

R2 recommends (b) with a fallback to (a). For your spike, (a) alone is acceptable — ship it, mention in the report that ticker resolution is a follow-up.

**Q4. What should the spender argument look like?**

A flag (`--spender 0x...`) with regex validation. The regex `/^0x[0-9a-fA-F]{40}$/` catches the obvious typos. Do *not* accept ENS names — ENS resolution introduces a network call and a domain trust boundary you don't need in a test hook. If a hook author wants ENS, they should resolve it before invoking the CLI.

**Q5. What does the user see when they run revoke and the allowance is already 0?**

This is a real product decision. Three behaviours:

- *(a) Always send the transaction.* Revoke costs gas regardless. The on-chain state is the same after as before, but the test harness pays gas every iteration.
- *(b) Pre-check via `getEvmTokenAllowance`.* If the result is 0, exit 0 with `{ skipped: true, reason: "allowance-already-zero" }` and don't sign anything. Saves gas, makes the command idempotent.
- *(c) Add a `--skip-if-zero` flag.* Default off; opt-in idempotency.

R2 recommends (a) for the spike, deferring (b) to a `--skip-if-zero` flag. Why: the cleanest semantics for a hook are "revoke always emits the tx; you decide upstream when to call it." You can layer (c) on later without breaking callers.

Now think — *what should happen if the user passes `--amount 100` instead of `--amount 0`?* For the QAA-615 spike, the answer is "the command refuses with a clear error, because revoke is amount=0 by definition; users who want partial allowance should use a future `approve --amount N` command." But you should write that decision into the help text so a confused future maintainer doesn't second-guess it.

### 5.8.7 Phase 3 — Wire the Calldata

The ERC-20 approve selector is `0x095ea7b3`. The full calldata for `approve(spender, 0)` is:

- 4 bytes: `0x095ea7b3` (the function selector — the keccak256 hash of `"approve(address,uint256)"`, truncated to its first 4 bytes).
- 32 bytes: the spender address, left-padded with zeroes to fit a 256-bit slot.
- 32 bytes: the amount (0), left-padded to fit a 256-bit slot.

Total 68 bytes. The same shape would be true for `approve(spender, N)` — only the last 32 bytes differ.

Concretely, `approve(0x58df81bababd293aaca0a3a76d70d3b69b25c1cf, 0)` encodes to:

```
0x095ea7b3
00000000000000000000000058df81bababd293aaca0a3a76d70d3b69b25c1cf
0000000000000000000000000000000000000000000000000000000000000000
```

You will not write this by hand. The signature of the helper:

```ts
// libs/coin-modules/coin-evm/src/logic/getErc20Data.ts:10
export function getErc20ApproveData(spender: string, amount: bigint): Buffer
```

Your call site is one line. The interesting question is **how do you turn the `Buffer` into the hex string the wallet-cli intent schema expects?** The schema accepts `data: "0x..."` (a string), not a `Buffer`. So your code is something like:

```ts
const calldataBuffer = getErc20ApproveData(spender, 0n);
const calldataHex = "0x" + calldataBuffer.toString("hex");
// pass calldataHex as intent.data
```

That's it. Don't gold-plate.

> **Verify:** look at `bridge.ts:194-198` again to confirm it strips the `0x` prefix back off before reconstructing the Buffer for the live-common transaction. Yes, you're encoding then decoding — that's the cost of the schema being stringly-typed at the intent boundary. Don't try to short-circuit by passing a Buffer through; you'd break the schema validation step.

> **A quick sanity check.** Once you've wired this up, dry-run your command with a known spender and assert the output `data` field is exactly 138 characters (`0x` + 4-byte selector × 2 + 32 bytes × 2 + 32 bytes × 2 = 2 + 8 + 64 + 64 = 138). If your output is 137 or 139 characters, you have a leading-zero or padding bug. Fix that *before* you bother signing anything.

### 5.8.8 Phase 4 — Send the Signed Transaction

Once your intent is constructed, your handler delegates to the existing send pipeline. The shape, copied from `send.ts`'s handler:

```ts
// inside the handler, after building `descriptor` and `intent`:
if (dryRun) {
  const prepared = await wallet.prepareSend(descriptor, intent);
  // emit a dry-run envelope with prepared.amount, prepared.fees, prepared.recipient
  return;
}

await withCurrencyDeviceSession(descriptor.currencyId, async () => {
  await lastValueFrom(
    wallet
      .send(descriptor, intent, WALLET_CLI_DMK_DEVICE_ID, false)
      .pipe(tap(event => out.sendEvent(event))),
  );
  out.sendComplete();
});
```

Read `send.ts` end to end before you copy this shape. The points to internalise:

- **`WalletAdapter.send` returns an `Observable<SendEvent>`**, not a Promise. You convert it with `lastValueFrom` (rxjs). If you've never used rxjs, the mental model is: an Observable is a stream of events; subscribing pulls them; `lastValueFrom` turns "subscribe and wait until completion, return the last value" into a Promise.
- **The `SendEvent` types** (`device-streaming`, `device-signature-requested`, `device-signature-granted`, `signed`, `broadcasted`) come from the live-common `signOperation` Observable. You don't need to act on each — `out.sendEvent(event)` formats them for human or JSON output. But you should know what they are; print them out during local development.
- **`withCurrencyDeviceSession`** opens the DMK session, prompts the user to connect the device and open the right app (Ethereum), then runs your callback. The session is closed automatically on exit. You do **not** open or close the device yourself.
- **`WALLET_CLI_DMK_DEVICE_ID`** is a constant exported from the device transport registration module. Same value `send.ts` uses.

R2's appendix points to the file:line for `BridgeAdapter.send` if you want to trace what happens after `wallet.send` returns. For your purposes you don't need to — the only thing your code is responsible for is constructing a correct `intent`. Everything below the `wallet.send` call is shared infrastructure.

### 5.8.9 Phase 5 — Verify with `getEvmTokenAllowance`

This is optional for QAA-615 itself — the spike does not require a companion read command — but it's the natural next step and Section 5.9.4 (in Chapter 5.9 Exercises) makes it concrete. Do it now if you have time; defer if you don't.

The companion command, `wallet-cli allowance`, would expose:

```bash
wallet-cli allowance ethereum-1 --token USDC --spender 0xUNISWAP_ROUTER
```

and print the current on-chain allowance for that (account, token, spender) triple.

Skeleton:

```ts
// apps/wallet-cli/src/commands/allowance.ts
import { defineCommand, option } from "@bunli/core";
import { z } from "zod";
// TODO: imports — see revoke.ts; the only new symbol is getEvmTokenAllowance from
//       @ledgerhq/live-common/families/evm/getTokenAllowance

export default defineCommand({
  name: "allowance",
  description: "Read the current ERC-20 allowance for a (token, spender) pair",
  options: {
    // TODO: account, token, spender, output (no --dry-run — this is read-only)
  },
  handler: async (ctx) => {
    // TODO: resolve descriptor and currency
    // TODO: sync the account (so freshAddress and parentCurrency are populated correctly)
    // TODO: const result = await getEvmTokenAllowance(account, tokenId, spender);
    // TODO: format result.allowance using result.unit (BigNumber → human string)
    // TODO: print "<value> <symbol>" (human) or the full result envelope (JSON)
  },
});
```

The questions to resolve as you fill it in:

1. **How do you go from the V1 descriptor to a synced `Account`?** Look at how `send.ts` handles this — there's a `WalletAdapter.sync` (or similar) that returns a live-common Account. The bridge already exposes one.
2. **Token id resolution:** ticker only? token id only? Both? Mirror what you decided in 5.8.6 Q3 — consistency with `revoke` is the win.
3. **Output formatting:** the allowance comes back as a `BigNumber` in raw integer units (e.g., `1000000` for 1 USDC since USDC has 6 decimals). Format it via `formatCurrencyUnit(unit, allowance)` for human output. Pass the raw integer string + the unit metadata for JSON output, so consumers can do their own formatting.

This command is what makes the regression hook **verifiable**: revoke, wait, allowance, assert it's 0. Without it, the hook trusts that revoke worked but cannot prove it. Ship them as a pair if you can.

### 5.8.10 Phase 6 — Tests

The wallet-cli's test harness (`apps/wallet-cli/src/test/`) uses `bun test` plus three custom modules: a `cli-runner` that spawns the binary in-process, a `mock-DMK` that fakes device responses at the `Completed` state level, and an `http-intercept` that catches outbound RPC and explorer calls. Look at `apps/wallet-cli/src/test/commands/send.test.ts` for the canonical shape.

A representative test from the existing `send.test.ts` (paraphrased — open the actual file to see current line numbers):

```ts
describe("send --dry-run", () => {
  it("returns a dry-run envelope with the parsed amount and recipient", async () => {
    const { stdout, exitCode } = await runCli(
      ["send", "--account", MOCK_ETH_DESCRIPTOR, "--to", MOCK_ETH_RECIPIENT, "--amount", "0.001 ETH", "--dry-run", "--output", "json"],
      {
        env: {
          WALLET_CLI_MOCK_APP_RESULTS: JSON.stringify({
            Ethereum: { publicKey: MOCK_ETH_PUBKEY, address: MOCK_ETH_ADDRESS },
          }),
          WALLET_CLI_HTTP_INTERCEPT: ETH_SYNC_ROUTES,
        },
      },
    );
    expect(exitCode).toBe(0);
    const data = JSON.parse(stdout);
    expect(data.command).toBe("send");
    expect(data.amount).toBe("0.001 ETH");
    expect(data.recipient).toBe(MOCK_ETH_RECIPIENT);
    expect(data.dry_run).toBe(true);
  });
});
```

Now write `apps/wallet-cli/src/test/commands/revoke.test.ts`. The cases you should cover, at minimum:

1. **`revoke --dry-run` with a known token returns a JSON envelope** with `command: "revoke"`, `recipient` = the USDT contract (`0xdAC1...`), `amount` matching `/^0(\.0+)? USDT$/`, and `dry_run: true`.
2. **`revoke --dry-run` rejects an unknown token** — exit code non-zero, stderr matches `/not found|unknown token/i`.
3. **`revoke --dry-run` rejects a malformed spender** (`--spender not-an-address`) — exit code non-zero, stderr matches `/spender|address/i`.
4. **`revoke` (mocked DMK) emits device-signature events and a broadcasted txHash** — full flow, with `WALLET_CLI_MOCK_APP_RESULTS` set and the broadcast endpoint mocked to return a fake hash.

Hints for the wiring:

- **Mock-DMK** doesn't care what the calldata is. The signing call dispatches the same way regardless of `transfer` vs `approve`. So you don't need a new mock fixture — `MockAppResults["Ethereum"]` is identical to what `send.test.ts` uses.
- **HTTP routes** the revoke flow needs (same as send): `GET /address/<addr>/balance`, `GET /address/<addr>/nonce`, `POST /tx/estimate-gas-limit`, `GET /gastracker/barometer`, plus the ETH sync routes for `BridgeAdapter.sync`. If you went with ticker resolution (Q3 option b in Phase 2), you may also need a token sub-account fixture; `eth-sync-routes.ts` likely already has one.
- **Broadcast mock** — when you test the signed flow (case 4), intercept the broadcast endpoint and have it return a deterministic 32-byte hash like `0x` + `"a".repeat(64)`. The exact endpoint is in `eth-sync-routes.ts`.

> **Verify:** R2 references `apps/wallet-cli/src/test/helpers/eth-sync-routes.ts` and `apps/wallet-cli/src/device/mock-dmk.ts` as the assets you'll touch. If the names have moved, search by content (`grep -r "ETH_SYNC_ROUTES" apps/wallet-cli/`).

A test you do **not** write at this level: an assertion on the device's clear-sign screen content. The mock-DMK doesn't drive the on-device UI — Speculos does — and Speculos isn't part of the unit test loop. Clear-sign verification belongs in the QAA-613 UI tests downstream, not here.

### 5.8.11 Phase 7 — Manual Sanity Run

Mocked tests prove the wiring; a real device run proves the device renders the approve screen correctly. The manual loop:

1. **Plug in a Ledger device** loaded with a test seed (the QAA team's standard test seed; check the `qa-team-resources` Confluence page or ask in `#qa-automation` if you don't have it).
2. **Open the Ethereum app** on the device. Make sure clear-sign is enabled (Settings → Allow contract data is fine, but for clear-sign you also need the modern descriptor support that's on by default on recent firmware).
3. **Pick a network.** Sepolia is preferred for the spike — gas is free, and you can produce a non-zero allowance ahead of time by running `wallet-cli send` with the long `--data 0x095ea7b3...` form (Section 5.8.3) targeting a fake spender. Mainnet is fine too if you have a real allowance to revoke (e.g., a stale Uniswap approval from a real swap), but you'll spend real gas.
4. **Run** `wallet-cli revoke ethereum-1 --token USDT --spender 0x...`.
5. **On the device, walk through the screens**. You should see:
   - "Type: Approve" (or "Review transaction")
   - "Amount: 0 USDT"
   - "Address: 0x..." (the spender, shortened on Stax/Flex)
   - Fee, network, "Sign?"
6. **Approve and let the broadcast complete.** Capture the txHash from stdout.
7. **Open Etherscan** (or Sepolia's etherscan), paste the txHash, and confirm the tx state: `to` = the USDT contract, `value` = 0, calldata starts with `0x095ea7b3`, ends in 64 trailing zeroes.
8. **Capture a screenshot** of one of the device screens (Speculos has a "save screenshot" command; the device itself can be photographed). The screenshot is evidence for your spike report.

> **Verify:** the QAA test seed and test spender addresses. R2 didn't pin down the exact router addresses on Sepolia. If you don't have a known testnet allowance to revoke, you'll need to set one up first — pick any address as the fake spender (e.g., `0x000000000000000000000000000000000000dEaD`), approve a small allowance, then revoke. The on-chain state doesn't matter; what matters is that the device shows the right screens.

### 5.8.12 Phase 8 — The Spike Report

QAA-615 is a spike. The artefact that gets reviewed is not just the PR — it's a written report explaining what you found, what you shipped, and what you'd recommend next. Without this report, the spike is incomplete even if the code works.

The report can live as a Confluence page under the QAA team space, as a long Jira comment on QAA-615 itself, or as a markdown file in the PR description. Confluence is the canonical home for spike reports in this team — confirm with your manager which they prefer.

A 1-page template you can fill:

```
# QAA-615 — Spike: wallet-cli revoke

## Question
Can we add a CLI command that revokes an ERC-20 token approval, suitable for use as a
before/after hook in the broadcast-enabled regression suite (QAA-613 and follow-ups)?

## Answer
Yes. Implementation shipped in PR #<number>: `wallet-cli revoke <account> --token <ticker> --spender <0x...>`.
Transaction signs `approve(spender, 0)` against the token contract, broadcasts, returns txHash.

## Path chosen
Path C (thin wrapper over the existing `send` pipeline). Rationale: <one paragraph
explaining the load-bearing constraints — hook UX, code reuse, no live-common changes>.

## LOC actually shipped
- commands/revoke.ts: <N> lines
- wallet/erc20-calldata.ts (or deep import from coin-evm — note which): <N> lines
- cli.ts: +2 (registration)
- output.ts: +<N> (revoke envelope variants, if added)
- test/commands/revoke.test.ts: <N> lines
Total: ~<N>00 LOC.

## Where it slots in
Can be invoked as `wallet-cli revoke <descriptor> --token <ticker> --spender <0x...>`
in a shell, or as a child process from the QAA-613 test harness. JSON output mode
(--output json) emits a stable envelope `{ command: "revoke", account, token, spender,
recipient, amount, txHash }` suitable for `jq` parsing.

## What is NOT covered (out of scope for QAA-615)
- Dynamic spender lookup from a known-DEX registry (e.g., resolve `--router uniswap-v3` to the
  current Uniswap V3 router address). Currently the user must pass the address.
- Auto token registry — if the token isn't in `cryptoassets`, the command errors. Adding
  arbitrary `--contract 0x...` support is a follow-up.
- A companion `wallet-cli allowance` read command (sketched in 5.8.9, not shipped here).
- A symmetric `wallet-cli approve --amount N` command (acknowledged as a natural follow-up;
  ~70 LOC given the same helper).

## Recommended next steps
1. Land `wallet-cli allowance` so QAA-613's hook can verify revoke worked, not just trust it.
2. Wire `wallet-cli revoke` into QAA-613's `beforeEach` once that ticket starts.
3. Expand the token registry / accept raw `--contract` for tokens not yet in the registry.
4. Track the same pattern for ERC-721 and ERC-1155 (NFT) approvals, since
   `setApprovalForAll` will eventually have the same regression-suite need.

## Evidence
- Tests: <link to the test file>, <N> cases passing under `bun test`.
- Manual run on Sepolia: txHash <0x...>, screenshots attached.
- PR: <link>.
```

The report is short on purpose. A spike that produces a 5-page report has scope-crept into a feature.

### 5.8.13 PR Shape

The PR for QAA-615 should be small, isolated, and easy to review. Suggested commit decomposition (each small, each green on its own):

1. `feat(cli): add revoke command skeleton` — the new `revoke.ts` file + registration in `cli.ts`. No tests yet, no output formatting; just the command structure compiling.
2. `feat(cli): wire ERC-20 approve calldata into revoke handler` — the `getErc20ApproveData` integration, the intent construction, the dry-run path. After this commit, `revoke --dry-run` works.
3. `feat(cli): wire revoke through the device-session signing path` — the `withCurrencyDeviceSession` + `wallet.send` plumbing. After this commit, the full flow runs against a real device.
4. `feat(cli): add JSON envelope for revoke output` — output formatter changes.
5. `test(cli): add revoke command tests` — the four test cases from Section 5.8.10.
6. (Optional) `docs(cli): document revoke in apps/wallet-cli/README.md` — one paragraph + an example invocation.

**Reviewers:**
- One owner of `apps/wallet-cli` (look at the most recent merged PRs to identify; usually `@ledgerhq/wallet-xp` or whatever team owns the wallet-cli workspace today).
- The QAA SDET who will consume the command for QAA-613 — Victor, since he raised the requirement.
- Optionally, an EVM family owner if your PR touches `coin-evm` (it shouldn't, but if you decide to expose a new helper from the EVM logic module, this becomes mandatory).

**PR title suggestion:** `feat(cli): add revoke command for ERC-20 token approvals (QAA-615 spike)`

**PR body should include:**
- One-paragraph summary lifted from the spike report's "Answer" section.
- A copy-pasteable example invocation.
- A link to the spike report (Confluence or Jira comment).
- A screenshot from the manual run (Section 5.8.11).
- The "What is NOT covered" list, so reviewers don't ask about scope creep.

### 5.8.14 Common First-Run Failures

When you actually run your code for the first time, you will hit issues. Here are five-to-seven likely ones with hints — not full solutions. Practise debugging from the symptom backward.

**1. `Token UNKN not found`** when you run `wallet-cli revoke … --token UNKN`. The token registry doesn't have a ticker `UNKN`. Decision time: do you add the token to the registry (PR to `libs/cryptoassets/src/data/...`), or do you accept a raw `--contract 0x...` flag in your command? For the QAA-615 spike, the answer is "neither — error and document the limitation in the spike report under What is NOT covered." For follow-up tickets, this is the seam where `--contract` lands.

**2. `Cannot read property 'parentAccount' of undefined`** during descriptor parsing. Your V1 descriptor parsing is missing the parent linkage. The descriptor format `account:1:<type>:<network>:<env>:<xpub>:<path>` for a token sub-account requires the parent account to be syncable; if the bridge can't find the parent, the resolver throws. Trace through `resolveAccountDescriptor` to find where the lookup is happening; usually this is a fixture issue (the descriptor points to a token but the sync doesn't return it as a sub-account).

**3. `Insufficient balance` during dry-run, even though you specified amount=0.** Your gas estimation is hitting the live RPC and the test address has 0 ETH. Two fixes: either fund the address with a fraction of an ETH (Sepolia faucet is free), or verify your HTTP intercept routes are catching the gas-estimate call and returning the mocked fee in the test environment.

**4. `Invalid hex data: expected 0x prefix`** at the bridge boundary. You forgot the `"0x" +` prefix when converting the Buffer from `getErc20ApproveData` to a string. Re-read Section 5.8.7's sanity check.

**5. The device shows blind-sign warning ("Data is present, sign?") instead of the friendly "Type: Approve" screen.** Two possible causes:
- The token isn't in the cryptoassets registry on the device build you're running. The clear-sign decoder uses `findTokenByAddressInCurrency` to resolve the recipient (which is the token contract); if that resolution fails, it falls through to blind-sign. Test with a mainstream token (USDT, USDC, DAI) before assuming the code is wrong.
- The Ethereum app on the device is too old. Recent firmware ships clear-sign descriptors for ERC-20; very old firmware does not. Update the device.

**6. `pnpm generate` succeeds but `wallet-cli --help` doesn't list `revoke`.** You registered the command in the wrong place in `cli.ts`, or the codegen target points at a different file. Compare with how `send` is registered. Also check whether there's a runtime registry (e.g., a `defineCli({ commands: [...] })` call somewhere) — Bunli sometimes wires commands at runtime, not at build time.

**7. The unit test passes locally but fails in CI** with a network-error message. Your test is hitting a real RPC because `WALLET_CLI_HTTP_INTERCEPT` isn't set in the CI environment, or your sync route fixture is missing a route the bridge calls. Run with `DEBUG=*` (or whatever the wallet-cli's debug flag is — check the README) to see exactly which URL the test tried to fetch. Add it to `eth-sync-routes.ts`.

If you hit something not on this list, write it down. Add it as the eighth entry in the chapter's reference list when you submit your PR — future readers benefit from your stumbles.

### 5.8.15 Linking Back to QAA-613

QAA-613 is the next ticket in the queue: *"implement the token approval flow with Thorchain, Uniswap, and LiFi. Broadcast will be disabled so just the first part of the flow will happen, no actual swap."* It's the consumer of QAA-615.

The shape of QAA-613's hook, once your `wallet-cli revoke` ships, is roughly five lines of shell:

```bash
# pseudo-code for the QAA-613 beforeEach hook
echo "Revoking $TOKEN allowance for $SPENDER on $ACCOUNT..."
TX_HASH=$(wallet-cli revoke "$ACCOUNT" \
  --token "$TOKEN" \
  --spender "$SPENDER" \
  --output json | jq -r '.txHash')
echo "Revoke broadcast: $TX_HASH"
# wait for confirmation — exact mechanism depends on the test harness
```

Two important nuances:

**(a) QAA-613 itself runs broadcast-disabled.** The on-chain allowance never moves during a QAA-613 run. So strictly, the revoke hook is *not* needed for QAA-613's own runs — the test starts each iteration from the same on-chain state regardless. The reason you ship revoke now anyway is that **the same test code will run in the broadcast-enabled regression suite** (the parent QAA-919 epic's deliverable). When that switch flips, the hook becomes load-bearing without anyone having to add it.

**(b) The hook is per-flow, not per-suite.** Each provider (Thorchain, Uniswap V3, LiFi) has its own router address. Revoking the Uniswap allowance does not reset the Thorchain allowance. The hook runs in the `beforeEach` of each provider's flow and revokes that provider's specific spender. R6's audit confirms this — the test fixture stores per-provider router addresses, so the hook reads `$SPENDER` from the per-flow config.

> **Verify:** the exact Thorchain / Uniswap V3 / LiFi router addresses for the chains the regression suite targets. R2 didn't pin these down, and they change occasionally (especially Uniswap, which has v2/v3/universal-router variants on different networks). The Swap Live App team owns the canonical list — check `swap-configuration` repo or ask `#swap-engineering` before hardcoding.

### 5.8.16 Going Further

Once `revoke` is shipping and being consumed by QAA-613's hook, the natural follow-ups split into four categories. None of these are in scope for QAA-615; all are reasonable items to file as new tickets.

**1. Companion `approve` command.** Symmetric to revoke but takes `--amount N` (or the literal `unlimited`, mapping to `2^256 - 1`). Reuses the same `getErc20ApproveData` helper with a non-zero amount. Estimated ~70 LOC. Useful for the regression suite's *setup* phase: pre-load an allowance before a swap test starts, so the test exercises the swap path rather than the approve path.

**2. Dynamic spender lookup.** Instead of `--spender 0x...` (raw address), accept `--router uniswap-v3` (provider name), and resolve internally to the current router address per (chain, version). Source of truth: `swap-configuration` repo. Pro: hooks become readable (`--router uniswap-v3` instead of a 40-hex-char blob). Con: introduces a network/config dependency on a moving target. File this as a separate ticket so the tradeoff is explicit.

**3. Batch revoke.** Take a comma-separated `--spender 0xA,0xB,0xC` (or read a file) and emit one transaction per spender. Useful when a single test wipes multiple stale allowances at once. Each transaction is signed individually (no EIP-3074 / batched-tx assumptions); the device prompts once per signature. Pseudo-cost: linear in number of spenders, both in gas and in user-tap effort if running on a real device. Acceptable for CI on Speculos, painful for a human.

**4. `--unlimited` toggle on `approve`.** Sets the allowance to `2^256 - 1`, which the device's clear-sign decoder renders as "Unlimited <ticker>". Useful for tests that need a "rich" account that can swap any amount without hitting allowance bounds. Already documented in `cli-transaction.ts:37` (`UNLIMITED_APPROVAL_AMOUNT = 2n ** 256n - 1n`). One-flag addition once `approve` lands.

A practice question to test your understanding: of these four, which one is closest to "free" (smallest review surface, biggest test-suite leverage)? Stop reading and answer before continuing.

> **Answer.** `approve` (item 1). The helper is already in your hand from QAA-615; the command is a 70-LOC clone of `revoke` with one extra option. Once `approve` ships, items 3 and 4 fall out almost for free (batch is a loop, `--unlimited` is one constant). Item 2 is the most useful but the most expensive — it ties wallet-cli to the swap config repo, which is its own moving target.

### 5.8.17 Chapter 5.8 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — QAA-615 Walkthrough</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> The QAA-615 ticket text says the revoke command is for "automatic regression testing AND NOT nightly executions." What does that distinction mean architecturally?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The CLI must run only at night, not during the day</button>
<button class="quiz-choice" data-value="B">B) Nightly executions don't need a CLI; daytime ones do</button>
<button class="quiz-choice" data-value="C">C) The nightly QAA-613 suite runs broadcast-disabled, so on-chain allowance never moves and revoke isn't strictly needed for it; the broadcast-enabled regression suite (parent epic QAA-919) does mutate on-chain state, so it needs revoke as a cleanup hook to keep iterations deterministic</button>
<button class="quiz-choice" data-value="D">D) Speculos cannot run during nightly windows</button>
</div>
<p class="quiz-explanation">The load-bearing constraint is the broadcast switch. Nightly = broadcast off = no state to clean. Regression = broadcast on = state must be reset between runs. Revoke is for the second case. Victor's Slack message frames the command as a `before/after` hook for exactly this reason.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> R2's spike report recommends Path C (thin <code>revoke</code> wrapper over the existing <code>send</code> pipeline). What is the strongest argument <em>against</em> Path B (just rely on <code>send --data 0x095ea7b3...</code>, which already works today)?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Path B is broken on Speculos</button>
<button class="quiz-choice" data-value="B">B) Path B works for interactive debugging but is unfit for unattended test hooks: the hook author has to assemble 68 bytes of hex by hand, a single typo silently produces a different transaction, and test logs become coupled to hex strings rather than to a clean semantic envelope</button>
<button class="quiz-choice" data-value="C">C) Path B requires changes to the EVM intent schema</button>
<button class="quiz-choice" data-value="D">D) Path B is slower at runtime</button>
</div>
<p class="quiz-explanation">Path B is functionally correct — the example invocation in 5.8.3 produces a real revoke transaction. But the regression-suite hook context (Victor's "ajouter un hook before/after le test") is the load-bearing constraint. Hooks run unattended, must be greppable, and must not silently no-op or produce the wrong transaction when an author mistypes a digit. Path B fails all three; Path C addresses all three.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> In the legacy <code>cli-transaction.ts</code>'s <code>revokeApproval</code> path, the produced <code>Transaction</code> has <code>mode: "send"</code>, <code>recipient: tokenContractAddress</code>, <code>amount: BigNumber(0)</code>, and <code>data: getErc20ApproveData(spender, 0n)</code>. Why is <code>recipient</code> the <em>token contract address</em> instead of the spender's address?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because the EVM transaction's <code>to</code> field must point at the contract that contains the function being called. <code>approve(spender, 0)</code> is a function on the token contract, so <code>tx.to = tokenContractAddress</code>; the spender is a parameter encoded inside the calldata, not the recipient of the transaction</button>
<button class="quiz-choice" data-value="B">B) Spender addresses cannot receive transactions</button>
<button class="quiz-choice" data-value="C">C) Live-common refuses to sign transactions whose recipient is a non-contract address</button>
<button class="quiz-choice" data-value="D">D) The clear-sign decoder requires it</button>
</div>
<p class="quiz-explanation">EVM transactions are calls to whichever contract sits at <code>tx.to</code>. To run <code>approve</code>, the transaction must be addressed to the token contract; the spender is just one of the two ABI-encoded arguments inside <code>tx.data</code>. Putting the spender in <code>tx.to</code> would be a no-op transaction sending zero ETH to the spender — wholly different from a revoke.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q4.</strong> <code>getErc20ApproveData(spender: string, amount: bigint): Buffer</code> returns a Buffer; the wallet-cli EVM intent schema's <code>data</code> field expects a string of the shape <code>"0x..."</code>. Why does the bridge's <code>buildTxExtras</code> then strip the <code>0x</code> prefix and reconstruct a Buffer at <code>bridge.ts:194-198</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Bun does not support Buffers natively</button>
<button class="quiz-choice" data-value="B">B) The intent schema is broken — this is a bug</button>
<button class="quiz-choice" data-value="C">C) The intent boundary is stringly-typed (Zod-validated, JSON-serializable, easy to log) while the live-common <code>Transaction.data</code> is a Buffer because that's what the RLP encoder expects. The two-step encode/decode is the cost of having a clean, serialisable intent at the CLI surface and a binary representation at the wallet-internal surface</button>
<button class="quiz-choice" data-value="D">D) The Buffer is replaced by a hex string for security</button>
</div>
<p class="quiz-explanation">Two layers, two representations. The intent is a stable, schema-validated user-facing shape; serialising it as JSON for tests, dry-runs, and logs is much easier when every field is a string or a primitive. The bridge then converts to the internal binary types the bridge and signer expect. Trying to pass a Buffer through the intent boundary would break <code>JSON.stringify</code>, defeat schema validation, and couple the CLI surface to internal types.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> You're writing the unit tests for <code>revoke</code> using mock-DMK. Which of the following does mock-DMK <em>not</em> let you assert?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) That the command exits with code 0</button>
<button class="quiz-choice" data-value="B">B) That the device's screen displays "Type: Approve / Amount: 0 USDT / Address: 0x..." in clear-sign mode</button>
<button class="quiz-choice" data-value="C">C) That the JSON envelope includes a txHash matching <code>/^0x[0-9a-f]{64}$/</code></button>
<button class="quiz-choice" data-value="D">D) That stderr matches <code>/spender|address/i</code> when the spender flag is malformed</button>
</div>
<p class="quiz-explanation">Mock-DMK abstracts at the <code>Completed</code> state level — it returns canned <code>publicKey</code>/<code>address</code> results without running the on-device UI. Exit codes, stdout JSON, stderr text, and event types are all observable. The actual on-screen labels are not — that requires Speculos and lives in QAA-613's UI tests downstream. The QAA-615 spike's tests should not assert screen text.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q6.</strong> The QAA-615 ticket is labelled <code>LLD/LLM/UI</code>, but the work lives entirely inside <code>apps/wallet-cli/</code>. Should you re-label the ticket before merging?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) No — labels are immutable once a ticket is in progress</button>
<button class="quiz-choice" data-value="B">B) Yes — you must remove all incorrect labels before opening the PR</button>
<button class="quiz-choice" data-value="C">C) The labels don't matter</button>
<button class="quiz-choice" data-value="D">D) Mention it in the spike report and to your manager — labels were inherited from the parent epic and don't reflect the spike's actual scope. Whether to re-label depends on the team's Jira hygiene conventions; do not unilaterally re-label without asking, but do flag the mismatch so dashboards and filters reflect reality</button>
</div>
<p class="quiz-explanation">Labels in a real Jira project carry weight (filters, dashboards, escalation routing). Mismatched labels are an organisational hygiene issue, not a code issue, and the right move is to surface them rather than silently fix them. The spike's value is in observation as much as in code — and "the labels are wrong, here's why" is a legitimate finding for the report.</p>
</div>

<div class="quiz-score"></div>
</div>

<div class="chapter-outro">
<strong>Key takeaway.</strong> QAA-615 is small in code (~225 LOC including tests) but large in <em>understanding</em>. The work that earns its keep is the diligence: reading the legacy <code>cli-transaction.ts</code> to internalise the <code>tx.to = tokenContract / tx.value = 0 / tx.data = approveCalldata</code> shape, mapping out the three implementation paths instead of jumping straight at one, choosing Path C with explicit reasoning, writing the spike report, and shipping the PR with the right reviewers. The TypeScript itself follows from those decisions almost mechanically — which is why this chapter handed you scaffolding instead of a finished file. If you wrote the code yourself by working through the Phase TODOs, you now know the wallet-cli stack the way you know the desktop and mobile stacks: from the inside. Chapter 5.9 — Exercises — picks up from here with three companion drills (an <code>allowance</code> read command, a <code>--skip-if-zero</code> idempotency flag, and a <code>--router</code> dynamic-spender lookup) to push the same code further.
</div>

---
## CLI Exercises and Challenges

<a id="cli-exercises-and-challenges"></a>

<div class="chapter-intro">
Reading is not enough. The exercises below move from a 15-minute connectivity smoke to a 90-minute pre-test hook script. Do them in order &mdash; each one builds a skill you will need in the next. Every exercise has a verification step; if you cannot verify it yourself, grab a reviewer before moving on. The capstone (Exercise 7) previews QAA-613's hook pattern, so the work transfers directly to the next ticket on the QA backlog.
</div>

### 5.9.1 Exercise 1: Read Your Own xpub (15 min)

**Objective.** Confirm your local setup works end to end &mdash; Bun, DMK, USB, the Ethereum app, the descriptor V1 schema, the JSON envelope.

**Instructions.**

1. Plug your Ledger in via USB. Unlock with PIN.
2. Open the **Ethereum** app on the device manually.
3. From `apps/wallet-cli/`, run (with the agent sandbox disabled because USB is involved):

   ```bash
   pnpm --silent wallet-cli start -- account discover ethereum --output json | jq '.accounts[0]'
   ```

4. Confirm the output is a JSON descriptor string starting with `account:1:address:ethereum:main:0x...`.
5. Compare the address in the descriptor with the address shown in your real Ledger Live account list. They must match.

**Verification.** The descriptor parses through `parseAccountDescriptor()` (Ch 5.4) and the embedded address is exactly what Ledger Live shows for the same path. Exit code is 0.

**Hints.**

- Use `--no-verify` on `receive` if you want to peek at an address without the device prompting you to confirm. `discover` does not need `--no-verify`; the device only asks for confirmation on `receive --verify=true` (the default).
- The V1 descriptor format and field meanings are in Ch 5.4.
- If you see `Timeout has occurred`, you are running inside the sandbox. Re-run the same Bash with `dangerouslyDisableSandbox: true`.

**Stretch goal.** Pipe the descriptor straight into a `balances` invocation:

```bash
DESC=$(pnpm --silent wallet-cli start -- account discover ethereum --output json | jq -r '.accounts[0]')
pnpm --silent wallet-cli start -- balances "$DESC" --output json | jq '.balances'
```

You have just chained two CLI calls without ever touching `session.yaml` &mdash; useful for ad-hoc work that should not pollute persistent state.

### 5.9.2 Exercise 2: Inspect a Public Address Balance (15 min)

**Objective.** Understand the descriptor V1 format well enough to hand-build one from public data &mdash; no device required.

**Instructions.**

1. Pick any well-known ETH address (e.g. the Vitalik Buterin address `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045`).
2. Hand-build a V1 address-type descriptor:

   ```text
   account:1:address:ethereum:main:0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045:m/44h/60h/0h/0/0
   ```

   (The path is fictional &mdash; that's fine, `balances` does not verify path-to-address derivation.)
3. Run:

   ```bash
   pnpm --silent wallet-cli start -- balances 'account:1:address:ethereum:main:0xd8dA...:m/44h/60h/0h/0/0' --output json | jq .
   ```

4. Read the JSON. Identify the native ETH entry vs. the ERC-20 token entries (USDC, USDT, etc.). Note the `asset` and `amount` fields.

**Verification.** The output is a `balances` JSON envelope with at least one entry where `asset === "ethereum"`. ERC-20 entries (if any) appear as additional objects with `asset` set to a token identifier.

**Hints.**

- No device needed &mdash; `balances` for the EVM family routes through `AlpacaAdapter` (see Ch 5.4 / R1 §5).
- If the request fails with a network error, the http-intercept is **off** here (you're running for real, not under test). Make sure your host has internet.

**Stretch goal.** Compare the JSON output with the ETH balance shown by a public block explorer like Etherscan. They should agree to the rounding shown on the explorer's page.

### 5.9.3 Exercise 3: Add a `--dry-run` Test for `send` (30 min)

**Objective.** Write a `bun test` case that exercises the `send --dry-run` path through `cli-runner`, asserts no signing is attempted, and asserts the JSON envelope contains a `dry_run: true` marker.

**Instructions.**

1. Open `apps/wallet-cli/src/test/commands/send.test.ts` &mdash; this is the existing 72-line dry-run test (R1 §8). Read it end to end.
2. Notice it uses `runCli([...], { WALLET_CLI_MOCK_DMK: "1", WALLET_CLI_MOCK_APP_RESULTS: ... })` and asserts the dry-run envelope.
3. **Add a new `it()` to that file** that:
   - Spawns `send` with `--to 0xf39F... --amount '0.01 ETH' --dry-run`.
   - Uses the same `MOCK_ETH_DESCRIPTOR` from `test/helpers/constants.ts` as the account.
   - Asserts the JSON envelope's `command` field is `"send"`.
   - Asserts no `tx_hash` field is present (a real send returns one; dry-run must not).
   - Asserts the envelope has `recipient`, `amount`, `fee` fields populated &mdash; that's the `prepared` event from `WalletAdapter.send()` (see Ch 5.4 / R1 §3 for the `SendEvent` discriminated union).

**Verification.**

```bash
bun test src/test/commands/send.test.ts
```

Both your new test and the existing one are green. Exit code 0.

**Hints.**

- The fixture file `test/helpers/eth-sync-routes.ts` already provides the routes `send` needs (balance, nonce, gas estimate, gas tracker barometer). Reuse them.
- If the test crashes with `LedgerAPI4xx`, your routes are missing the `/v1/currencies` CAL endpoint. Look at how `send.test.ts` imports `SEND_ROUTES` &mdash; copy that pattern.
- LIVE-29404 documents that `@bunli/core`'s boolean options need an explicit value. Pass `--dry-run=true` or `--dry-run true` &mdash; **not** bare `--dry-run` (the latter is silently dropped).

**Stretch goal.** Add a third test that asserts the **failure case**: drop `--to` from the command, run, and assert exit code 1 plus a JSON error envelope (`{ ok: false, error: { command: "send", message: "..." } }`).

### 5.9.4 Exercise 4: Cross-Reference Allowance from Legacy CLI (45 min)

**Objective.** Read a working pattern in `apps/cli` (legacy), then write a 30-line wallet-cli **read-only** stub that calls into live-common's existing allowance helper. This exercise warms you up for Ch 5.8's QAA-615 walkthrough &mdash; it is not shipped code, just a compile-clean reading exercise.

**Instructions.**

1. Open `apps/cli/src/commands/blockchain/tokenAllowance.ts` (legacy CLI). Read it in full &mdash; it's short.
2. Find the underlying live-common helper it calls. The signature lives at:

   ```
   libs/ledger-live-common/src/families/evm/getTokenAllowance.ts
   ```

   exporting `getEvmTokenAllowance({ owner, spender, contract, currency })` (or similar &mdash; verify the exact shape).
3. Create a new file **on a scratch branch only**: `apps/wallet-cli/src/commands/allowance.ts`. Follow the canonical command shape (Ch 5.4 / R1 §3):
   - `defineCommand({ name: "allowance", description: "Read ERC-20 allowance for a spender", options: { account, token, spender, output }, handler })`.
   - Resolve the account via `resolveAccountDescriptor(...)`.
   - Call `getEvmTokenAllowance({ ... })` with the right inputs.
   - Wrap the result in the JSON envelope via `out.run(...)` and `makeEnvelope(...)`.
4. Run:

   ```bash
   pnpm typecheck && pnpm lint
   ```

   Both must pass. Do **not** run `pnpm generate` (you are not shipping this); do **not** open a PR (this is a learning artefact); do **not** add a test (we are practicing reading and wiring, not coverage).

**Verification.** `pnpm typecheck` and `pnpm lint` both pass with your file in place. The file compiles. You have a private branch with a 30-line read-only stub.

**Hints.**

- The EVM family's bridge (R1 §5, `wallet/compatibility/bridge.ts`) already plumbs `data` through to the bridge transaction &mdash; that's the write path for QAA-615. For the read path, you skip the bridge entirely and call `getEvmTokenAllowance()` directly because there is no transaction to build.
- The legacy `apps/cli` returns plain strings; your wallet-cli stub must return the JSON envelope. That difference (R4 §A.5, friction-point #1) is exactly why the new CLI exists.
- Do not import from `wallet/compatibility/bridge.ts` directly. The README in `wallet/` says: "do not import `bridge.ts` or `alpaca.ts` from outside `wallet/`." For an allowance read, use the live-common helper directly &mdash; that is the correct boundary.

**Stretch goal.** Discard the file (`git checkout -- apps/wallet-cli/src/commands/allowance.ts && rm apps/wallet-cli/src/commands/allowance.ts`). Now write a 200-word note in your QAA-615 ticket comment explaining how Ch 5.8's walkthrough should differ from your stub. Submit the comment for your reviewer's amusement.

### 5.9.5 Exercise 5: Add a JSON Envelope Assertion (45 min)

**Objective.** Internalize the JSON envelope contract by writing a strict-shape assertion. Make it fail first by tweaking the assertion; then make it pass.

**Instructions.**

1. Pick any existing test file under `src/test/commands/`. `balances.test.ts` is a good choice &mdash; it already asserts a few envelope fields loosely.
2. Add a new `it("emits the strict envelope shape", ...)` to that file that runs the same command and asserts **all four** of:
   - `envelope.status === "success"`
   - `envelope.command === "balances"` (or whichever command you picked &mdash; **the exact name**, including any group prefix like `"account discover"`)
   - `envelope.network === "ethereum:main"`
   - `envelope.account === ETH_DESCRIPTOR` (or the `MOCK_ETH_DESCRIPTOR` for device-using commands)
3. Run the test &mdash; it should pass.
4. **Now make it fail** &mdash; tweak one assertion to use a wrong expectation (e.g. `expect(envelope.network).toBe("ethereum:test")`). Re-run; it must fail.
5. Read the failure output. Note how Bun's diff formatter shows the mismatch.
6. Restore the correct assertion. Re-run; green again.

**Verification.** You can describe, without looking at the source, exactly what each of the four envelope fields contains and where they are populated (`makeEnvelope()` in `src/shared/response.ts`).

**Hints.**

- `account` is **only present when the command resolved an account descriptor** &mdash; `session view` and `session reset` do not set it. Choose a command that does (anything taking `--account`).
- The error envelope is **different** (`{ ok: false, error: { command, message } }` &mdash; no `status` key, no `network` key). R1 §5 calls this inconsistency "a real wart" &mdash; do not assert error envelopes with the same helper as success envelopes.
- The success envelope's `timestamp` field changes every run &mdash; do not assert its exact value. Use `expect(envelope.timestamp).toMatch(/^20\d\d-/)` if you want a smoke-level assertion.

**Stretch goal.** Extract your four assertions into a helper `expectStrictEnvelope(envelope, { command, network, account })` in a new file `src/test/helpers/envelope-assertions.ts`. Use it from at least two existing tests. Run all tests &mdash; all green.

### 5.9.6 Exercise 6: Reproduce the "Timeout has occurred" Sandbox Error (60 min)

**Objective.** Internalize the sandbox boundary by triggering the canonical CLI failure mode on purpose, then resolving it on purpose. This exercise exists because every CLI engineer hits this once and wastes an hour debugging the wrong layer.

**Instructions.**

1. Plug your Ledger in. Unlock with PIN. Open the Ethereum app.
2. From an agent shell (Claude Code, Cursor, etc.), run a device-touching command **without** disabling the sandbox:

   ```bash
   pnpm --silent wallet-cli start -- receive ethereum-1 --output json
   ```

   Submit the Bash call with the sandbox **enabled** (the default).
3. Wait until it errors. Note the exact error string verbatim. It will be one of:
   - `Timeout has occurred`
   - `UnknownDeviceExchangeError`
   - A connect timeout from `register-dmk-transport.ts` (60s).
4. Now run the **same command** with `dangerouslyDisableSandbox: true`. Confirm on the device prompt that appears. The command completes; you get a JSON envelope with `data.address`.
5. Open `apps/wallet-cli/src/device/wallet-cli-device-error.ts` &mdash; the error mapping table (R1 §4). Find which row the sandbox error you saw maps to. Read the suggested remediation; note that it is misleading in the sandbox case (the error map assumes a real transport problem, not a sandbox block).

**Verification.** In your private notes, write a 5-line summary distinguishing **sandbox-caused** failures (USB blocked, no transport, timeouts that look like device issues) from **host-caused** failures (device locked, wrong app open, cable problem). Keep this file. Future-you will reference it.

**Hints.**

- The sandbox restricts USB; that's why the connect call hangs until the timeout fires.
- The error mapping in `wallet-cli-device-error.ts` is designed for host-caused failures &mdash; it has no awareness of the sandbox layer above it. That's why the mapped message ("Timed out talking to the Ledger over USB. Retry; check cable.") is unhelpful here.
- This is the same lesson R4 §A.2 codifies: the internal-testing page lists `Timeout has occurred` as the very first canonical error.

**Stretch goal.** Modify your local copy of `wallet-cli-device-error.ts` (do not commit) to add a hint suggesting "If running through an AI agent shell, set `dangerouslyDisableSandbox: true` for this Bash call." Run again, observe the improved error. Discuss with your lead whether this is worth a real PR &mdash; the answer is "probably no" because the CLI shouldn't know about agent harnesses, but the discussion is the point.

### 5.9.7 Exercise 7 &mdash; Stretch: A Pre-Test Hook Script (90 min)

**Objective.** Build a 3-line shell hook that asserts a precondition on the test seed's USDC balance, exits 1 if the precondition fails. This is the exact pattern QAA-613's nightly tests will use to ensure deterministic state &mdash; you are previewing the Ch 5.8 deliverable.

**Instructions.**

1. Identify the ETH descriptor for your test seed (use the one from Exercise 1 or `MOCK_ETH_DESCRIPTOR` from `test/helpers/constants.ts`).
2. Create `scripts/precheck-usdc-clean.sh` (or a `.ts` equivalent) at the repo root:

   ```bash
   #!/usr/bin/env bash
   set -euo pipefail
   DESC="${1:-$ETH_TEST_DESCRIPTOR}"
   pnpm --silent wallet-cli start -- balances "$DESC" --output json \
     | jq -e '.balances[] | select(.asset=="USDC") | .amount=="0 USDC"' \
     >/dev/null || { echo "[precheck] USDC balance is non-zero; clean state required" >&2; exit 1; }
   echo "[precheck] USDC balance clean"
   ```

3. `chmod +x scripts/precheck-usdc-clean.sh`.
4. Run it. If your seed has no USDC entry at all (likely for a fresh dev address), `jq -e` will exit non-zero on the empty selector &mdash; treat that as a failed assertion. Adjust the script if you want "no USDC entry" to be acceptable (use `select(.asset=="USDC") | .amount` and default to zero).
5. Wire the script as a `pretest` script for an experimental playwright job:

   ```json
   "scripts": {
     "test:swap-regression": "playwright test specs/swap/regression",
     "pretest:swap-regression": "scripts/precheck-usdc-clean.sh '$ETH_TEST_DESCRIPTOR'"
   }
   ```

6. Run the playwright suite. The pretest must run first; if the precondition fails, the suite must not start.

**Verification.** Three runs:

- **Run A.** Seed clean (USDC balance == 0). `pretest` exits 0. Playwright runs.
- **Run B.** Seed dirty (USDC balance > 0 &mdash; you can simulate by editing the mock route or by sending dust on testnet). `pretest` exits 1. Playwright does **not** run.
- **Run C.** Wallet-cli is offline (e.g. you're on a plane). `pretest` exits non-zero with a clear network error in stderr. Playwright does not run.

**Hints.**

- `jq -e` propagates the test result as exit code &mdash; that is the whole reason it exists. Without `-e`, `jq` always exits 0 even when the filter produces no output.
- The amount format is `"<value> <ticker>"` &mdash; e.g. `"0 USDC"`, `"123.45 USDC"` (R1 §5; the PRD enforces human-readable amounts with explicit tickers, R4 §A.1).
- For a real CI hook, prefer a `.ts` script that uses `child_process.spawn` and parses JSON in TypeScript &mdash; it is more robust than `jq` chains and gives you better error messages. Bash is fine for the 3-line MVP.
- This is exactly the pattern Ch 5.8 walks through end to end for QAA-615's `revoke` &mdash; the difference is that 5.8's hook **mutates state** (revokes an allowance) while yours **asserts state** (USDC balance is 0). Both are pre-test hooks. Both call `wallet-cli`. Both exit non-zero on failure.

**Stretch goal.** Replace the bash script with a TypeScript version that:

1. Reads the descriptor from an env var or a `.env.local` file.
2. Runs `pnpm wallet-cli start -- balances ... --output json` via `Bun.spawn`.
3. Parses the envelope, picks the USDC entry, compares amount to "0 USDC".
4. On failure, **invokes a hypothetical `wallet-cli revoke`** to clean state, then re-asserts.
5. Exits 0 on green, 1 on un-recoverable failure.

You now have a draft for QAA-613's "before" hook. Show it to your lead before Ch 5.8 &mdash; you'll learn whether your design intuition matches what shipped.

---
## Part 5 Final Assessment

You have walked the full CLI automation stack: `apps/wallet-cli/` versus the
legacy `apps/cli`, the Bun runtime under Bunli's `define()` command tree, the
Device Management Kit replacing the old `@ledgerhq/hw-transport-*` zoo, the
Account Descriptor V1 grammar that glues currencies to xpubs, the mock-DMK plus
http-intercept test rig, and one real spike (QAA-615 — `wallet-cli revoke`)
shipped end to end as a fixture for QAA-613. This assessment samples the
load-bearing ideas across Chapters 5.1-5.11. Eighty percent to pass. If you miss
a question, jump back to the referenced chapter — the answer is always grounded
in a concrete file, command, or APDU.

<a id="part-5-final-assessment"></a>

<div class="quiz-container" data-pass-threshold="80">
<h3>Part 5 Final Assessment</h3>
<p class="quiz-subtitle">10 questions · 80% to pass · Covers Chapters 5.1-5.11</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> The monorepo currently contains two CLIs at <code>apps/cli</code> and <code>apps/wallet-cli</code>. Which statement correctly describes their relationship?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>apps/cli</code> is the active CLI; <code>apps/wallet-cli</code> is an experimental fork that will be deleted</button>
<button class="quiz-choice" data-value="B">B) Both are equal peers — <code>apps/cli</code> handles signing and <code>apps/wallet-cli</code> handles account discovery</button>
<button class="quiz-choice" data-value="C">C) <code>apps/cli</code> is the legacy Node + <code>@ledgerhq/hw-transport-*</code> CLI being phased out; <code>apps/wallet-cli</code> is the active Bun + Bunli + DMK replacement, and new test fixtures (e.g. QAA-615 revoke) must land in <code>apps/wallet-cli</code></button>
<button class="quiz-choice" data-value="D">D) <code>apps/wallet-cli</code> wraps <code>apps/cli</code> with a nicer flag parser but shares its transport code</button>
</div>
<p class="quiz-explanation">Chapter 5.1 lays out the migration story:
<code>apps/cli</code> predates DMK and the live-common refactor and is in
maintenance mode; <code>apps/wallet-cli</code> is the strategic successor — Bun
runtime, Bunli command framework, DMK transport. Onboarding guidance: do not
write new commands or fixtures in the legacy CLI. Run
<code>pnpm wallet-cli start &lt;cmd&gt;</code>, never <code>pnpm cli ...</code>.
The legacy tree still exists because some scripts and a handful of nightly jobs
still call into it, but every QAA fixture from this sprint forward (QAA-615
revoke, QAA-617 environment droplist, QAA-722 min-amount) lands exclusively in
<code>apps/wallet-cli</code>.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> A teammate's first wallet-cli PR uses
<code>console.log</code> for status output, reads <code>process.env.HOME</code>
directly, and shells out via <code>child_process.exec</code>. Which
architectural footgun does this most obviously trip?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) None — wallet-cli is plain Node and these are all idiomatic</button>
<button class="quiz-choice" data-value="B">B) wallet-cli runs under Bun, not Node — and Bunli commands write user-facing output through the injected logger so it can be captured by <code>cli-runner</code> in tests; using <code>console.log</code> and raw <code>child_process</code> bypasses both the runtime's native APIs (<code>Bun.spawn</code>, <code>Bun.file</code>) and the test harness's stdout capture</button>
<button class="quiz-choice" data-value="C">C) Bun forbids <code>console.log</code> at runtime</button>
<button class="quiz-choice" data-value="D">D) <code>process.env</code> is the only correct way to read secrets in wallet-cli</button>
</div>
<p class="quiz-explanation">Chapter 5.1 enumerates the four most common
footguns: assuming Node when the runtime is Bun, bypassing the Bunli logger,
importing from the legacy CLI's <code>lib/</code>, and creating a DMK transport
per command instead of reusing the session.</p>
<p class="quiz-explanation">The injected logger and <code>Bun.*</code> native
APIs are not optional — <code>cli-runner</code> mocks them in tests and real
runs depend on them. A PR that prints with <code>console.log</code> will pass
locally but produce empty assertions in the unit suite, because the test
harness reads from the logger contract, not from process stdout.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> In Bunli's <code>define({ ... })</code> command shape
used by <code>apps/wallet-cli/src/commands/*.ts</code>, what does the
<code>handler</code> argument receive, and how are flags declared?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>handler</code> receives <code>argv: string[]</code> and flags are parsed manually inside the function body</button>
<button class="quiz-choice" data-value="B">B) <code>handler</code> receives a Commander.js <code>program</code> instance; flags are added with <code>.option()</code></button>
<button class="quiz-choice" data-value="C">C) <code>handler</code> receives only the device transport; flags are declared as decorators on the surrounding class</button>
<button class="quiz-choice" data-value="D">D) <code>handler</code> receives a context with typed <code>flags</code>, <code>positionals</code>, <code>logger</code>, and <code>prompt</code> already resolved; flags are declared in <code>define({ flags: { ... }})</code> with Zod schemas, and Bunli's codegen produces a typed manifest from those declarations</button>
</div>
<p class="quiz-explanation">Chapter 5.3 walks through the Bunli
<code>define</code> API: every command exports
<code>define({ name, description, flags, handler })</code> where
<code>flags</code> is a Zod-validated record. Bunli generates a typed manifest
at build time so <code>flags.spender</code> in the handler is statically
<code>0x${string}</code> rather than <code>string | undefined</code>.</p>
<p class="quiz-explanation">Skipping codegen breaks type-checks in CI — see
Chapter 5.9 on the codegen pre-commit gate. The same chapter shows the QAA-615
revoke command's <code>define</code> block as a reference implementation: two
required flags (<code>--account</code>, <code>--spender</code>), one optional
(<code>--gas-limit</code>), all Zod-validated.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> The Device Management Kit (DMK) is what wallet-cli uses
to talk to a Ledger device. What does it replace, and what transports does it
expose?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It replaces the family of <code>@ledgerhq/hw-transport-*</code> packages (hw-transport-node-hid, hw-transport-http, hw-transport-webusb…) with one session-oriented SDK that exposes USB (node-hid under the hood), HTTP (Speculos), and WebHID transports behind a single typed device session</button>
<button class="quiz-choice" data-value="B">B) It replaces Ledger Live's Redux store with a CLI-friendly state machine</button>
<button class="quiz-choice" data-value="C">C) It replaces BOLOS firmware on the device itself</button>
<button class="quiz-choice" data-value="D">D) It is a thin shim over <code>child_process</code> that boots Speculos</button>
</div>
<p class="quiz-explanation">Chapter 5.4 covers DMK end-to-end: one
<code>DeviceManagementKit</code> instance, one <code>DeviceSession</code>,
multiple pluggable transports. wallet-cli wires the USB transport for real
devices and the HTTP transport for Speculos; you select via
<code>--speculos</code> at the CLI surface, but underneath the only thing that
changes is the transport plugged into the same DMK session.</p>
<p class="quiz-explanation">The legacy <code>hw-transport-*</code> packages are
still present in the repo for back-compat but new code must consume DMK. The
APDU exchange API is the same shape; what changed is session lifecycle,
typed-event observability, and a unified error taxonomy.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> An Account Descriptor V1 string in wallet-cli has the
shape
<code>account:1:&lt;type&gt;:&lt;network&gt;:&lt;env&gt;:&lt;xpub_or_address&gt;:&lt;path&gt;</code>.
What is the role of the <code>1</code> after <code>account</code>, and why is
the descriptor string-based rather than a JSON object?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The <code>1</code> is a checksum byte; the string form is a Bun runtime constraint</button>
<button class="quiz-choice" data-value="B">B) The <code>1</code> is the device app version; JSON would not survive shell quoting</button>
<button class="quiz-choice" data-value="C">C) The <code>1</code> is the descriptor schema version, so future fields can be added without breaking parsers; the colon-delimited string form is shell-paste-friendly, copy-paste-survivable across tools, and language-agnostic so non-TS consumers (Speculos hooks, shell scripts, future Rust tooling) can split on <code>:</code></button>
<button class="quiz-choice" data-value="D">D) The <code>1</code> indicates a single-signer account; the string form is required by DMK</button>
</div>
<p class="quiz-explanation">Chapter 5.6 introduces the V1 grammar in detail.
The leading <code>1</code> is a deliberate version slot — V2 will add fields
without changing the prefix, and parsers reject unknown versions explicitly.</p>
<p class="quiz-explanation">The string form trades a little parsing ceremony
for a huge usability win: descriptors travel through CI logs, shell history,
Slack messages, and ticket comments unchanged. The reference parser lives in
<code>src/lib/account/descriptor.ts</code> and is the canonical implementation
— never re-implement the split locally.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q6.</strong> Inside
<code>apps/wallet-cli/src/lib/intents/families/evm.ts</code>, what does the
file actually do, and why is it the entry point for QAA-615's revoke
command?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It exports a React component that renders the EVM transaction form</button>
<button class="quiz-choice" data-value="B">B) It builds the family-specific transaction intent for EVM chains — encoding ERC-20 calldata (<code>transfer</code>, <code>approve</code>, <code>revoke = approve(spender, 0)</code>), populating gas/nonce, and handing the typed intent to the DMK signer; QAA-615 plugs in there because revoke is <code>approve</code> with <code>amount = 0</code> and the calldata builder already handles <code>approve</code></button>
<button class="quiz-choice" data-value="C">C) It is a test fixture file with no production role</button>
<button class="quiz-choice" data-value="D">D) It bridges wallet-cli to the Swap Live App via WebSocket</button>
</div>
<p class="quiz-explanation">Chapter 5.5 maps every file in
<code>src/lib/intents/families/</code>. The <code>evm.ts</code> intent builder
is where currency-agnostic command handlers (send, approve, revoke) are
translated into chain-specific calldata + gas params.</p>
<p class="quiz-explanation">QAA-615's spike (Chapter 5.10) confirmed the
cleanest path is a thin <code>revoke</code> subcommand that calls the same
<code>buildApproveIntent</code> with <code>amount: 0n</code>, reusing the EVM
family code instead of duplicating it. The same file already houses
<code>buildSendIntent</code> and <code>buildApproveIntent</code>, so revoke
needed only a one-line wrapper — not a new abstraction layer.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q7.</strong> You run
<code>wallet-cli revoke --account ... --spender ...</code> against a real
device and it fails with <code>Operation not permitted</code>. The same
command works with <code>--speculos</code>. What is the most likely fix?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Re-run with <code>--verbose</code> — the verbose flag toggles USB permissions</button>
<button class="quiz-choice" data-value="B">B) Reinstall node-hid via <code>pnpm rebuild</code></button>
<button class="quiz-choice" data-value="C">C) Switch to the legacy <code>apps/cli</code> for real-device runs</button>
<button class="quiz-choice" data-value="D">D) Re-invoke through Claude Code or your harness with <code>dangerouslyDisableSandbox: true</code>, because device-touching commands need raw USB access that the default sandbox denies — Speculos works fine in-sandbox because it is HTTP-only</button>
</div>
<p class="quiz-explanation">Chapter 5.7 catalogues the common-error remediation
table. Sandbox-level USB denial is the single most frequent first-day blocker:
Speculos runs over HTTP (and stays inside the sandbox's network allowlist for
<code>localhost</code>), but real device traffic goes through node-hid which
needs raw USB.</p>
<p class="quiz-explanation">The fix is harness-level
(<code>dangerouslyDisableSandbox: true</code>), not a wallet-cli flag. The same
chapter also covers the second-most-common error — a stale DMK session left
over from a previous run — and its fix: <code>pkill -f wallet-cli</code> then
re-launch.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q8.</strong> The wallet-cli test harness lives at
<code>apps/wallet-cli/src/test/</code> and combines <code>cli-runner</code>,
<code>mock-DMK</code>, <code>http-intercept</code>, and <code>mock-server</code>.
What is the specific role of <code>mock-DMK</code> versus
<code>http-intercept</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>mock-DMK</code> replaces the DMK device session with an in-memory script of expected APDU exchanges so signing tests run without any real device or Speculos; <code>http-intercept</code> stubs the explorer / coin-API HTTP calls (balances, fees, broadcast) so tests are deterministic and offline</button>
<button class="quiz-choice" data-value="B">B) <code>mock-DMK</code> and <code>http-intercept</code> are aliases for the same module</button>
<button class="quiz-choice" data-value="C">C) <code>mock-DMK</code> mocks the Bun runtime; <code>http-intercept</code> mocks the file system</button>
<button class="quiz-choice" data-value="D">D) Both are dev-only browser tools and not used in CI</button>
</div>
<p class="quiz-explanation">Chapter 5.8 splits the harness into two
responsibilities: device I/O (DMK) and network I/O (HTTP). A QAA-615 unit test
scripts the expected <code>approve(spender, 0)</code> APDU through
<code>mock-DMK</code> and stubs <code>eth_estimateGas</code> +
<code>eth_sendRawTransaction</code> through <code>http-intercept</code>.</p>
<p class="quiz-explanation">Both must be in place — a real-device test would
skip the mock-DMK layer; a real-network test would skip http-intercept; a pure
unit test uses both. <code>cli-runner</code> wires these together and exposes
the captured logger output for assertions; <code>mock-server</code> covers the
few cases that need a stateful HTTP fixture (e.g. simulating a nonce that
increments).</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q9.</strong> Which pre-commit step is non-negotiable in the
wallet-cli daily workflow, and what commit-message convention does the part
teach?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>pnpm lint --fix</code> only — codegen runs on push, not commit; commit messages are free-form</button>
<button class="quiz-choice" data-value="B">B) <code>pnpm format</code> only — Bunli has no codegen; commits use <code>[wallet-cli]</code> prefix</button>
<button class="quiz-choice" data-value="C">C) <code>pnpm wallet-cli codegen</code> (or <code>pnpm --filter wallet-cli codegen</code>) must run before commit so the generated typed flag manifest stays in sync with <code>define()</code> declarations; commits follow Conventional Commits — e.g. <code>feat(wallet-cli): add revoke command for QAA-615</code></button>
<button class="quiz-choice" data-value="D">D) Only <code>pnpm test</code> — commit messages must include the Jira ticket as the first word</button>
</div>
<p class="quiz-explanation">Chapter 5.9 walks the daily lifecycle ticket-to-PR.
Forgetting codegen is the #1 CI failure pattern: the generated manifest is
checked in, so a stale manifest fails the type-check and the codegen-drift CI
job.</p>
<p class="quiz-explanation">Commit conventions mirror the rest of the monorepo
— Conventional Commits, scope is <code>wallet-cli</code>, ticket id in the
body or trailer. Branch naming follows the same convention as desktop and
mobile work: <code>feat/qaa-615-wallet-cli-revoke</code>,
<code>support/wallet-cli-codegen-pin</code>, etc.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q10.</strong> The QAA-615 spike compared three implementation paths
for revoke. Which path was chosen, what reusable helper did it produce, and
how does QAA-613 consume it?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Path 2 (generalize <code>send</code> with raw calldata); helper is <code>buildCalldata()</code>; QAA-613 calls it inline in each spec</button>
<button class="quiz-choice" data-value="B">B) Path 3 (a thin <code>revoke</code> subcommand wrapping a generalized intent helper); the reusable piece is <code>buildApproveIntent({ spender, amount })</code> in <code>lib/intents/families/evm.ts</code>, called with <code>amount: 0n</code> for revoke and any positive value for approve; QAA-613's nightly token-approval specs gain a <code>beforeEach</code> (or <code>beforeAll</code>) hook that shells out to <code>wallet-cli revoke</code> to reset on-chain allowance to zero, so each broadcast-enabled regression run starts deterministic</button>
<button class="quiz-choice" data-value="C">C) Path 1 (a brand-new <code>revoke</code> command with its own calldata builder, no shared helper); QAA-613 imports the command's TypeScript source directly</button>
<button class="quiz-choice" data-value="D">D) None — the spike concluded revoke is impossible from the CLI and recommended a manual reset</button>
</div>
<p class="quiz-explanation">Chapter 5.10 documents the full QAA-615 spike:
Path 3 wins because it gives users a clean <code>wallet-cli revoke</code> UX
while keeping a single calldata builder shared with <code>approve</code>.</p>
<p class="quiz-explanation">The hook pattern matches Victor's confirmation in
the briefing — the approve only needs to happen once, and a before/after hook
calling the CLI keeps the regression suite re-runnable. Nightly QAA-613 runs
with broadcast disabled do not strictly need revoke (allowance never moves
on-chain), but the broadcast-enabled regression suite does, and the same
fixture serves both.</p>
<p class="quiz-explanation">The hook is a thin shell-out via
<code>Bun.spawn</code> from the test setup, capturing exit code and stderr; on
non-zero exit the test fails fast rather than running against a polluted
allowance state.</p>
</div>

<div class="quiz-score"></div>
</div>

<div class="chapter-outro">
<strong>Part 5 complete.</strong> You own the CLI automation stack end to end:
<code>apps/wallet-cli</code> versus the legacy <code>apps/cli</code>, the Bun
+ Bunli runtime model with its <code>define()</code> command shape and
codegen-checked typed flag manifest, the Device Management Kit replacing the
old <code>hw-transport-*</code> family with one session-oriented SDK over USB
and HTTP (Speculos), the Account Descriptor V1 grammar that travels safely
through CI logs and Slack, the codebase walk through <code>commands/</code>,
<code>lib/intents/families/evm.ts</code>, <code>signers/</code>,
<code>transports/</code>, the sandbox plus common-error remediation table, the
mock-DMK + http-intercept + cli-runner test rig, the daily workflow with its
codegen pre-commit and Conventional Commits, and one spike shipped — QAA-615
— that produced a <code>revoke</code> subcommand and a
<code>buildApproveIntent</code> helper that QAA-613's broadcast-enabled
regression hook now consumes.

<strong>Next: Part 6</strong> takes you up a layer — out of the wallet-cli's
terminal and into the <strong>Swap Live App</strong> that wallet-cli's revoke
fixture supports. The Swap Live App is the React/Vite app embedded in Ledger
Live that orchestrates the swap flow you just unblocked: it talks to
Thorchain, Uniswap, and LiFi, calls the Wallet API to request the two
on-device signatures (approve, then swap), and renders Firebase-driven release
variants per environment. Same monorepo, same Xray traceability, same PR
etiquette, same DMK transport underneath — just a UI surface where wallet-cli
is the silent test-data infrastructure underneath. The surface changes from a
Bun command tree to a Live App embedded in Ledger Live; the professionalism
does not.
</div>
