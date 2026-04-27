# R4 — Confluence research: wallet-cli, DMK, CLI testing strategy

Research agent: **R4**
Scope: Confluence (Tech Hub, Wallet XP, Quality Assurance, Live Apps, Coin Framework, LES Knowledge Hub, Architecture, Clear Signing, CIP)
Date: 2026-04-27
Method: Rovo Search (`mcp__claude_ai_Atlassian__search`) + targeted page fetches via `getConfluencePage`. CQL search was denied for this session, so probes were issued through Rovo Search (which already fans out across Confluence + Jira).

---

## 1. Top wallet-cli pages

| Title | Space | URL | Summary |
| --- | --- | --- | --- |
| Ledger Wallet CLI (PRD) | VAUL — LES Knowledge Hub | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/6907625515/Ledger+Wallet+CLI | The canonical PRD for the new `ledger-wallet-cli`. Defines phases v0 (connectivity, discover, balance, send), v1 (swap), v2 (encryption + earn). Specifies command syntax `[executable] [resource] [action] [flags]`, `--format json` everywhere, Account Descriptor V1 (`account:1:utxomain:xpub...:m/84h/0h/0h`), `--dry-run` for destructive commands, USB-only via DMK node-hid transport, `~/.ledger-wallet-cli/{accounts,keys,cache,config.yaml}` storage layout, terminal UX state machine for device disconnect/lock/app open/clear-sign/reject. |
| Ledger Wallet CLI internal testing | VAUL — LES Knowledge Hub | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/7039778832/Ledger+Wallet+CLI+internal+testing | Author: Charles Seille. The "how to dogfood" guide. Two tracks: (a) raw terminal in Claude Code, (b) natural language prompts ("Discover my Bitcoin accounts", "Show me the last 10 transactions for [descriptor]"). Lists the canonical errors (`Timeout has occurred`, `UnknownDeviceExchangeError`, `Transaction Cancelled`) with remediation. Mentions the `/ledger-wallet-cli` Claude skill. Reference for the QA onboarding "running the CLI" chapter. |
| Ledger Wallet CLI — User Stories: Accounts, Swap & Earn | VAUL | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/6948782086/Ledger+Wallet+CLI+User+Stories+Accounts+Swap+Earn | Per-phase US for v0/v1/v2. Includes `swap execute --quote-id $QUOTE_ID --format json`, `swap status --swap-id $SWAP_ID`. AC: every command returns structured JSON, accounts stored in `~/.ledger-wallet-cli/accounts/`. |
| Encryption CLI: User Stories & Use Cases | VAUL | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/6946291827/Encryption+CLI+User+Stories+Use+Cases | Phase v0.5. Replaces GPG workflow: `secrets init --domain`, `secrets encrypt --key`. Useful as future testing scope but not required for v0/v1 QA chapters. |
| AI Recommendations: Repurposing Existing CLIs | VAUL | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/6933020722/AI+Recommendations+Repurposing+Existing+CLIs | Compares `apps/cli` (`@ledgerhq/live-cli`, ~2018) and `vault-cli` to seed the new wallet-cli. Calls out historical use of `apps/cli` for: coin integration testing, QA automation (generates `app.json` for LL automated tests), and feature prototyping. |
| Install and use Ledger Wallet CLI | BO | https://ledgerhq.atlassian.net/wiki/spaces/BO/pages/4065853730/Install+and+use+Ledger+Wallet+CLI | Legacy install doc for `@ledgerhq/live-cli` (npm global install). Useful only for the "old CLI" appendix — the new wallet-cli does **not** install via this command. |
| ADR — CLI Storage | TA — Architecture | https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/6978928805/ADR+CLI+Storage | Status: **Proposed**. YAML chosen over INI/TOML for `~/.ledger/cli` config. Two competing schemas for device-id binding (in-descriptor vs. context-file). TBD. |
| Ledger Live CLI for AI Agent (architecture study) | TA | https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/6951665759/Ledger+Live+CLI+for+AI+Agent | Audit of the 59 commands in the legacy `apps/cli`. Tags each as `v0` / `v0.5` / `not in scope` and `reuse as-is` / `reuse with refactor` / `not tested`. Spells out friction points (string output not JSON, exit code always 0/1, no introspection, no timeout, no dry-run for destructive ops, no command composition). Proposes machine-readable error codes (`DEVICE_NOT_FOUND=10`, `INSUFFICIENT_FUNDS=20`, ...) and `ledger-live schema` self-describing endpoint. Compares MCP / CLI+Skills / REST API options — decision leaning to **CLI + Skills**. |
| Ledger Bitcoin Multisig — CLI PRD | VAUL | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/6996754670/Ledger+Bitcoin+Multisig+CLI+PRD | Sister CLI PRD for BTC multisig. Same flag conventions, same DMK transport. Useful precedent. |
| Archive — Ledger Enterprise CLI [PRD] | VAUL | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/6905069669/Archive+Ledger+Enterprise+CLI+PRD | Older Vault-side CLI PRD, archived. Skip. |
| Ledger Live monorepo package visibility inventory | WXP | https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/6975684839/Ledger+Live+monorepo+package+visibility+inventory | Confirms `cli` consumes `live-common`, `hw-app-btc`, `hw-app-eth`, `hw-app-str`. Useful for the architecture chapter. |
| LLM (ledger-live-mobile) with a real ledger device | PTX | https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/4223959176/LLM+ledger-live-mobile+with+a+real+ledger+device | Documents `ledger-live proxy` HTTP-bridge command for connecting LLM to a real device — useful for the QA mobile lab. |

### Bunli adoption (Jira-side, since no Confluence page yet)

| Ticket | URL | Highlight |
| --- | --- | --- |
| LIVE-29495 — Session layer: discover persists to session | https://ledgerhq.atlassian.net/browse/LIVE-29495 | Discovered accounts persisted to `~/.local/state/ledger-wallet-cli/session.yaml` via `@bunli/utils stateDir()`. Confirms wallet-cli uses **Bunli** + **XDG Base Dirs**. |
| LIVE-29404 — `--dry-run` flag silently ignored | https://ledgerhq.atlassian.net/browse/LIVE-29404 | Bug: `@bunli/core`'s `option()` requires explicit value for `z.boolean()` options; bare `--dry-run` is dropped. Critical safety issue (real broadcast). |
| LIVE-29312 — AI rules maintenance & skills distribution (exploratory) | https://ledgerhq.atlassian.net/browse/LIVE-29312 | Open question: is auto-generation of rules from CLI introspection feasible with Bunli? Should there be a shared `@ledgerhq/skills` package? |
| LIVE-29310 — CI pipeline for binaries | https://ledgerhq.atlassian.net/browse/LIVE-29310 | Mentions "see bunli release ?" — the team is considering Bunli's release tooling. |
| LIVE-29308 — Installation/distribution strategy (exploratory) | https://ledgerhq.atlassian.net/browse/LIVE-29308 | "Bunli already produces standalone self-contained binaries (~94MB, Bun runtime embedded)" + alternative `npm install -g @ledgerhq/wallet-cli` path. |
| LIVE-28265 — Plan to adapt current ledger-live CLI | https://ledgerhq.atlassian.net/browse/LIVE-28265 | Strategic ticket: `apps/cli` has ~70 commands, business logic battle-tested but CLI shape needs refactor. |
| LIVE-28269 — Connect transport layer to DMK | https://ledgerhq.atlassian.net/browse/LIVE-28269 | "Need to leverage the DMK by reusing/adapting `hw/ConnectApp` … Using this ConnectApp in a silent way for each commands required". This is the seed for the silent-app-open behaviour the PRD describes. |

---

## 2. Top DMK pages

| Title | Space | URL | Summary |
| --- | --- | --- | --- |
| Device Management Kit (overview) | CS — Clear Signing | https://ledgerhq.atlassian.net/wiki/spaces/CS/pages/7014678571/Device+Management+Kit | Best one-pager. DMK = the library between client (wallet, CLI) and the physical signer. Owns: ERC-7730 metadata fetch, APDU packaging, USB/BLE transport, error handling, fallback to blind signing. Two operating modes: Clear Signing vs. Fallback. Five key responsibilities laid out: metadata resolution, payload construction, transport management, signer abstraction, fallback handling. |
| LedgerJS vs DMK — Benchmark & Comparison | WXP — Wallet XP | https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/6995411067/LedgerJS+vs+DMK+Benchmark+Comparison | The pitch deck doc. Tier model: third-party wallet on **LedgerJS** = blind signing only; **DMK without origin token** = basic gated experience; **DMK + Ledger commercial agreement (origin token)** = full clear signing + Transaction Checks (Blockaid/Cyvers) + ENS. Mentions >10x reduction in connectivity errors. Includes a feature matrix + migration path + supported signer kits (`@ledgerhq/device-signer-kit-{ethereum,solana,bitcoin,hyperliquid,cosmos,aleo,zcash}`). |
| DMK Language Packages Management | WXP | https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/6962675778/DMK+Language+Packages+Management | Language pack lifecycle migration to DMK. Includes a link to `device-management-kit/src/api/command/os/ListLanguagePackCommand.ts`. |
| Device Management Kit Internal testing phase | WXP | https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/5549588557/Device+Management+Kit+Internal+testing+phase | DMK rolled into LLD for the transport feature — main advantage: device reconnection handled, no premature USB close. Has Gherkin-style test scenarios for "Allow Ledger Manager event" and similar. |
| FIRST PROMPT [DXP] DMK 26' Migration Strat | WXP | https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/6594658575/FIRST+PROMPT+DXP+DMK+26+Migration+Strat | 2026 DMK migration strategy: improved connectivity, unified device action implementation across LLD and LLM, better state management. |
| Device Management Kit (DMK) Verification | ECOM | https://ledgerhq.atlassian.net/wiki/spaces/ECOM/pages/5328306234/Device+Management+Kit+DMK+Verification | DMK used for ledger.com eligibility verification (device model name, type). Tangential. |
| How to :: Mock DMK | CIP | https://ledgerhq.atlassian.net/wiki/spaces/CIP/pages/6551306298/How+to+Mock+DMK | Coin integration playbook — for a coin without a Device Signer Kit yet, fall back to direct `@ledgerhq/device-management-kit` methods inside `libs/live-signer-COIN_NAME`. Useful pattern when the wallet-cli adds a chain whose Signer Kit is missing. |
| Discovery and connection logic study | WXP | https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/7045516602/Discovery+and+connection+logic+study | Live Mobile DMK (`libs/live-dmk-mobile`) wraps `@ledgerhq/device-management-kit` for React Native. Less relevant for CLI but documents the wrapping pattern. |
| Button Device actions (TO BE REVIEWED) | BI | https://ledgerhq.atlassian.net/wiki/spaces/BI/pages/6850052160/Button+Device+actions+TO+BE+REVIEW+BY+TEAM | Defines "Device Action" = unit of work sent to the DMK. Extends `XStateDeviceAction` from `@ledgerhq/device-management-kit`. |
| [Coin Framework] Coin Modules Developments Mapping | CF — Coin Framework | https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/6129025043/Coin+Framework+Coin+Modules+Developments+Mapping | Per-coin status of "DMK signer", "Alpaca", "Generic adapter", "Coin tester", "Extracted from ledger-live". Useful to know which coins the wallet-cli can actually exercise via DMK today. |

### DMK-related tickets worth knowing
- **LIVE-28652** — Create a DMK Singleton + Shared Provider in `live-dmk-shared` (one DMK instance per app).
- **LIVE-23712** — Deduplicate DMK dependencies in LW (linked via `pnpm desktop why @ledgerhq/device-management-kit`).
- **LIVE-22746** — Verify all errors are remapped in `connectAppEventMapper`. Cross-reference: `libs/ledger-live-common/src/hw/connectAppEventMapper.ts` vs. `packages/device-management-kit/src/internal/secure-channel/model/Errors.ts`.
- **DSDK-1076** — Use pnpm catalogs for DMK packages (`@ledgerhq/context-module`, `@ledgerhq/device-management-kit`).

---

## 3. CLI testing strategy / approach

There is **no dedicated Confluence page** for QA testing of the new `ledger-wallet-cli` yet. The closest matches are:

| Title | Space | URL | What it covers |
| --- | --- | --- | --- |
| Ledger Wallet CLI internal testing | VAUL | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/7039778832/Ledger+Wallet+CLI+internal+testing | Manual dogfooding script. Pre-conditions (real device + coin app open + only one parallel command), 6-step prompt suite (discover → balance → operations → receive → dry-run → send), expected results table, common errors, reporting template. **This is the only QA-leaning artefact for the new CLI.** |
| Testing strategy | CIP | https://ledgerhq.atlassian.net/wiki/spaces/CIP/pages/6346375175/Testing+strategy | Overall LL testing pyramid (unit → integration → E2E → manual hardware). Sets the framing for "where does CLI testing fit". |
| Testability Strategy | QA | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6637060671/Testability+Strategy | "Automation framework must be available and allow us to effectively validate at every level from unit, integration, component, API and end to end." Useful boilerplate framing. |
| QA — Feature Development & Quality Lifecycle | QA | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6898843819/QA+Feature+Development+Quality+Lifecycle | Process: Test Design in Xray, QA + QAA dual review, automation gates. |
| Kick off — Quality lifecycle and Regression Scope | QA | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6902677743/Kick+off+Quality+lifecycle+and+Regression+Scope | QA owns the test-plan ticket; QAA reviews PRs. |
| QA — Evaluation Criteria | QA | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6636437536/QA+Evaluation+Criteria | R01 "QA strategy has been produced early on". Standard R-codes. |
| QA Decision Log – EVM Scope Testing Strategy | QA | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6886588443/QA+Decision+Log+EVM+Scope+Testing+Strategy | 2026-03-10 decision: scope of EVM validation (touchscreen vs. legacy variability, LNS deprecation impact). Helps frame what "EVM in wallet-cli" should cover. |
| AI Adoption for QA — Candidate topics | QA | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6829965432/AI+Adoption+for+QA+Candidate+topics | "QA report from Jira/PR/tests", "Nightly E2E analysis", "Test plan/strategy automation". The CLI is a natural agent surface for these. |
| Vault E2E testing framework | QA | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6736685052/Vault+E2E+testing+framework | Existing pattern: `vault-cli` is consumed by the QA E2E framework for preconditions, test data, and CI. Direct precedent for "QA reuses CLI to bootstrap E2E state". |
| QA Framework information | QA | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6736117929/QA+Framework+information | Supplements the above; references `vault-cli` presets used during test execution. |
| [QA] - Automation strategy (Vault) | VAUL | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/2811756923/QA+Automation+strategy | Three layers (unit / integration / system) — useful template for the wallet-cli QA strategy chapter. |
| Swap :: Update E2E tests | CIP | https://ledgerhq.atlassian.net/wiki/spaces/CIP/pages/5493194794/Swap+Update+E2E+tests | Concrete pnpm commands: `pnpm desktop build:testing`, `pnpm desktop test:playwright:setup`, `pnpm desktop test:playwright`. Includes Speculos invocation patterns for swap. |
| Swap :: Testing Guide | CIP | https://ledgerhq.atlassian.net/wiki/spaces/CIP/pages/5458788446/Swap+Testing+Guide | Pre-conditions for Swap testing on Device App + LL. |

### What the legacy `apps/cli` is used for in QA today
From AI Recommendations: Repurposing Existing CLIs (VAUL/6933020722):
- **Coin integration testing** — external teams use the CLI to test new coin integrations.
- **QA automation** — QA generates `app.json` files for LL automated tests via the CLI.
- **Feature prototyping** — several features started life as CLI commands.
- The current `apps/cli` contains ~70 commands wrapping `@ledgerhq/live-common`: scan, sync, bridge, LKRP, swap.

---

## 4. Approve / revoke / allowance management

| Title | Space | URL | What it covers |
| --- | --- | --- | --- |
| Clear Signing — ERC 7730 — Reviews and Evolutions | CS | https://ledgerhq.atlassian.net/wiki/spaces/CS/pages/6038847989/Clear+Signing+ERC+7730+Reviews+and+Evolutions | Long living checklist. Includes "non reg test : ERC20 approves to add - to check", and "Swap USDC>ETH (Base) - Permit allowance: EIP-712 to add". Confirms `approve` / EIP-2612 `permit` are part of CS test scope. |
| Clear Signing — ERC 7730 v2 migration testing for Ledger Wallet | CS | https://ledgerhq.atlassian.net/wiki/spaces/CS/pages/6963232905/Clear+Signing+ERC+7730+v2+migration+testing+for+Ledger+Wallet | Per-network coverage (TC4 ERC-20 ETH ✅ Auto, TC5 ERC-20 Base ❌, TC6 Polygon ❌, TC7 Arbitrum ❌, TC8 Optimism ❌). Reveals current automation gaps in approval testing. |
| Clear Signing coverage | BI | https://ledgerhq.atlassian.net/wiki/spaces/BI/pages/6033178771/Clear+Signing+coverage | Test inventory: "Earn Deposit WETH", "Approval ERC 20, clear signed", "Sign transaction", with example tx hashes. The cleanest list of what an `approve` clear-sign happy-path looks like. |
| ERC-20 New Token Request | VAUL | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/3472261358/ERC-20+New+Token+Request | Process for adding new ERC-20s to the Vault catalog. Tangential to wallet-cli but useful for "what are tokens" intro. |
| ERC-20 FAQ | VAUL | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/920256551/ERC-20+FAQ | Conceptual ERC-20 background. |
| [Earn][Mobile] Non Regression Test Plan Template | PTX | https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/6399295490/Earn+Mobile+Non+Regression+Test+Plan+Template+v0.0.0+Mobile | Includes a "Revoke access anytime on revoke.cash" step with redirect verification. Demonstrates current QA approach: rely on revoke.cash dApp rather than first-party allowance management. |

### Critical Jira tickets for the approve/revoke story

| Ticket | URL | Why it matters |
| --- | --- | --- |
| **QAA-615** — [SWAP] Spike: Revoke token approval using CLI | https://ledgerhq.atlassian.net/browse/QAA-615 | The smoking gun. *"As we have the objective of testing the whole flow with token approval + swap, we need to find a way to revoke the approval to be able to run [the test repeatedly]."* QAA explicitly chose CLI as the revoke tool of choice. **This is the main quotable origin story for "why the new wallet-cli helps QA".** |
| **B2CQA-2844** — [SWAP] Revoke token access | https://ledgerhq.atlassian.net/browse/B2CQA-2844 | The user-facing test case: "check that the user can revoke token access inside Ledger Live", precondition = "user has approved access to token". |
| **VG-11881** — Revoke.cash / Portfolio Management | https://ledgerhq.atlassian.net/browse/VG-11881 | Current solution = redirect to revoke.cash. Confirms there is no first-party allowance manager today. |

**Conclusion for the QA chapter:** the revoke-token-approval flow is currently un-automatable end-to-end because there's no programmatic way to reset approval state between runs. The new `ledger-wallet-cli` (with v1 swap + future allowance commands) is positioned to close this gap.

---

## 5. Bunli framework usage

There is **no dedicated Confluence page** for Bunli adoption — the entire knowledge lives in Jira tickets and (presumably) in the `ledger-wallet-cli` source under `apps/wallet-cli/`. From the tickets:

- The new wallet-cli is built on **`@bunli/core`** (CLI framework) and **`@bunli/utils`** (XDG state dirs, file I/O).
- Session state lives at `~/.local/state/ledger-wallet-cli/session.yaml` via `@bunli/utils stateDir()` (LIVE-29495).
- `@bunli/core`'s `option()` requires explicit value for boolean options — bare `--dry-run` is silently dropped (LIVE-29404). **Active QA risk.**
- Distribution is being explored two ways: standalone Bunli binaries (~94MB with Bun runtime) and `npm install -g @ledgerhq/wallet-cli` (LIVE-29308).
- CI strategy: looking at "bunli release" for multi-package publishing (LIVE-29310).
- Open question: can Bunli introspection generate AI rules / Claude skills automatically (LIVE-29312)?

**No ADR has been published yet** that picks Bunli over alternatives (oclif, commander, citty, yargs). The only ADRs touching the new CLI are:
- ADR — CLI Storage (TA/6978928805) — YAML, `~/.ledger/cli/`.
- ADR — Account Descriptor (linked from the PRD: `https://ledgerhq.atlassian.net/wiki/x/EoDMnwE`).
- ADR — Encryption CLI (linked from the PRD: `https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/6976143576`).

---

## 6. Top 5 quotable URLs for the new CLI part chapters

1. **PRD (the ground truth)** — https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/6907625515/Ledger+Wallet+CLI
   Quote source for command shape, phases, terminal UX, security principles, JSON envelope, account descriptor V1.
2. **Internal testing guide (the QA dogfood path)** — https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/7039778832/Ledger+Wallet+CLI+internal+testing
   Quote source for prerequisites (real device + coin app + one-command-at-a-time), expected-results table, common errors and remediation.
3. **LedgerJS vs DMK benchmark** — https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/6995411067/LedgerJS+vs+DMK+Benchmark+Comparison
   Quote source for why the wallet-cli uses DMK and what "clear signing tier model" means (LedgerJS = blind / DMK no token = gated / DMK + token = full).
4. **DMK overview** — https://ledgerhq.atlassian.net/wiki/spaces/CS/pages/7014678571/Device+Management+Kit
   Quote source for the DMK's five responsibilities (metadata resolution, payload construction, transport management, signer abstraction, fallback) and the diagram Client → DMK → Signer.
5. **Architecture study: Ledger Live CLI for AI Agent** — https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/6951665759/Ledger+Live+CLI+for+AI+Agent
   Quote source for the legacy `apps/cli` audit (59 commands, 5 categories), the friction-point matrix (no JSON envelope, exit codes 0/1, no introspection, no timeout, no dry-run), the proposed error-code taxonomy (`DEVICE_NOT_FOUND=10`, `INSUFFICIENT_FUNDS=20`, ...), and the option study (MCP vs. CLI+Skills vs. REST).

### Honourable mentions
- **QAA-615** (https://ledgerhq.atlassian.net/browse/QAA-615) — the QA-side spike that motivated the revoke-via-CLI capability. Best one-line origin story for the "why" chapter.
- **ADR — CLI Storage** (https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/6978928805/ADR+CLI+Storage) — for the YAML config + `~/.ledger/cli` decision.
- **AI Recommendations: Repurposing Existing CLIs** (https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/6933020722/AI+Recommendations+Repurposing+Existing+CLIs) — for the historical context (apps/cli circa 2018, used by coin integration teams + QA + feature prototypers).

---

## 7. Gaps — what's missing in Confluence and must be inferred from code

### 7.1 Bunli framework
- **No Confluence page** describes Bunli adoption, why it was picked over alternatives, or the project layout under `apps/wallet-cli/`. All knowledge is in Jira tickets (LIVE-29308, LIVE-29310, LIVE-29312, LIVE-29404, LIVE-29495).
- **Action:** read the `apps/wallet-cli/package.json`, `apps/wallet-cli/src/cli.ts`, and the `bunli.config.*` file (if present) to extract the actual command tree, option-binding patterns, and bin entry. Cross-reference with the public Bunli docs via Context7 if available.

### 7.2 The new wallet-cli command tree (live state)
- **The PRD (VAUL/6907625515) describes the *target*, not the *current state*.** The "internal testing" page (VAUL/7039778832) hints at the current state via natural-language prompts but does not list flags or sub-commands.
- **Action:** run `ledger-wallet-cli --help` and `ledger-wallet-cli account --help`, then transcribe. Also check for a `--list-commands --format json` introspection endpoint (mentioned as "machine-readable command manifest" in the PRD §7).

### 7.3 DMK transport selection inside the CLI
- The PRD says "DMK node-hid transport" but no Confluence page documents the exact transport wiring inside the CLI.
- **Action:** look at `apps/wallet-cli/src/transport/*` and at `libs/live-dmk-shared/src/config/dmkInstance.ts` (per LIVE-28652). The DMK Singleton + Shared Provider pattern is referenced but not described.

### 7.4 Bunli + DMK integration shape
- No Confluence page bridges Bunli (`@bunli/core`) and DMK (`@ledgerhq/device-management-kit`). LIVE-28269 refers to "Connect transport layer to DMK" but the implementation pattern (silent ConnectApp wrapping per command) is in code only.
- **Action:** look at how each command builds its DMK action, especially the silent `ConnectApp` reuse mentioned in LIVE-28269.

### 7.5 QA test plan for `ledger-wallet-cli`
- **No Xray test plan, no test-cases page, no QA strategy doc** for the new CLI exists yet. Closest precedent is `vault-cli` (VAUL/2811756923) and the QA Decision Log on EVM (QA/6886588443).
- **Action:** the onboarding guide should propose a draft test plan structure (smoke / regression / device-state matrix / error-recovery / JSON output stability) since there is no canonical one to point to.

### 7.6 Allowance management commands
- The PRD does **not** include `approve` or `revoke` commands as named CLI verbs. QAA-615 is the only artefact treating revocation as a CLI primitive. The current strategy is "redirect to revoke.cash".
- **Action:** flag this as an open product question for the chapter on QA pain points: should `ledger-wallet-cli token approve --spender <addr> --amount 0` (the revoke pattern) be a v1.x deliverable?

### 7.7 Speculos integration for the new CLI
- The legacy `apps/cli` supports `bot`, `botPortfolio`, `cleanSpeculos`, `speculosList` (per the AI Agent architecture study). The new wallet-cli PRD does not mention Speculos at all.
- **Action:** confirm whether the new CLI ships a Speculos transport for CI testing or whether QA must keep using the legacy `apps/cli` for emulator-based work.

### 7.8 Bunli skill / SKILL.md distribution
- The PRD §7 says "ship alongside a published skill (SKILL.md)" — and the internal testing page references "/ledger-wallet-cli skill" — but no Confluence page documents what the skill contains or how it's versioned.
- **Action:** locate `.claude/skills/ledger-wallet-cli/SKILL.md` (or equivalent path) in the monorepo and excerpt the prompt patterns it teaches.

### 7.9 Error code taxonomy
- The architecture study (TA/6951665759) **proposes** an error-code taxonomy (DEVICE_NOT_FOUND=10, INSUFFICIENT_FUNDS=20, ...) but this is a proposal, not a decided ADR. The internal testing page (VAUL/7039778832) lists symbolic errors (`Timeout has occurred`, `UnknownDeviceExchangeError`, `Transaction Cancelled`) without numeric codes.
- **Action:** check whether the codes were actually adopted in the wallet-cli source (`grep -R "CliErrorCode" apps/wallet-cli`).

### 7.10 Distribution & signing
- LIVE-29310 is open ("set up CI pipeline to build and publish release binaries"), and LIVE-29311 (referenced) tracks binary signing separately. There is no Confluence page describing the chosen distribution channel (npm vs. Homebrew vs. GitHub Releases vs. Bun-bundled binary).
- **Action:** flag as in-flight; reference the open tickets only.

---

---

## Appendix A — Key quotable excerpts

### A.1 From the PRD (VAUL/6907625515)

On strategic positioning:
> "Position Ledger as the security layer for agents — giving AI agents hardware-grade signing and secret management through a developer-native interface."

On command grammar (§3):
> "All CLI commands will follow the `[executable] [resource] [action] [flags]` syntax. Every command must support a global `--format json` flag to output pure JSON for scripting and CI/CD piping."

On account-descriptor V1 (§3, v0):
> "EVM / Solana (`type: address`): address is extracted directly from the descriptor — no network call, no sync required.
> Bitcoin (`type: utxo`): delegates to the existing bridge sync to find the next unused external address via blockchain scan.
> Note: Only V1 descriptors are accepted. For on-device address verification with Trusted Display, use `receive`."

On enablers (§3):
> "Update `ledger-live-cli` repository to include an external `ledger-wallet-cli` that can be isolated but uses the same releases and code base.
> Integrate the new DMK node-hid transport for seamless connectivity and device actions.
> Create DMK Signer and Embedded App for encryption, based on LKRP v2 internal test app but adapted and extended for our use case.
> Embed Clear Signing token in CLI packaging so users can Clear Sign operations in swap that require Clear Signing and get Transaction Check simulation results."

On the device state machine (§4) — the canonical terminal prompts:
> "[✖] Error: Ledger device not detected. Action: Please plug in your Ledger device via USB, enter your PIN to unlock it, and run the command again."
> "[⧖] Waiting: Ledger device detected but locked. Action: Enter your PIN on the device to unlock it."
> "[ℹ] Opening Ethereum app on your Ledger... ACTION REQUIRED: Confirm opening the app on your Ledger device screen."
> "[⧖] Action Required: Transaction payload sent to device. Please review the transaction details on your Ledger device screen and confirm or reject directly on the device."
> "[✖] Transaction Cancelled: The operation was rejected on the Ledger device. No funds have been moved."

On security guarantees (§5):
> "The CLI must never access, prompt for, or store the user's 24-word recovery phrase. All money is moved with the user's validation."
> "Encryption Key Isolation: Derived encryption keys are stored only in the local keyring (`~/.ledger-wallet-cli/keys/`). The Ledger backend stores key tree metadata (domain, path) via LKRP — never the key material itself."
> "Payload Verification: For swaps, the CLI acts strictly as a passthrough. The Ledger device's Secure Element is the final source of truth for verifying the JWS signature against the swap provider's public key."

On amount formatting (§5):
> "All amounts are displayed and accepted in human-readable units throughout — never raw units (wei, satoshi, lamport). Numbers without input tickers are not accepted either to distinguish main currency from tokens (e.g. ETH vs USDC)."

On the JSON envelope shape (§5):
> ```json
> {
>   "status": "success",
>   "command": "send",
>   "network": "eth",
>   "account": "<descriptor>",
>   "tx_hash": "0xabc...123",
>   "amount": "0.5 ETH",
>   "fee": "0.0003 ETH",
>   "timestamp": "2025-06-01T14:32:00Z"
> }
> ```

On agent integration (§7):
> "Self-discovery via `--help`: Every subcommand exposes a `--help` that includes concrete examples. The examples are the primary interface for agents — they pattern-match faster off `ledger-wallet-cli send --network eth --to 0x... --amount 0.5 ETH` than off a description."
> "Skill distribution: The CLI should ship alongside a published skill (SKILL.md) so agents using Claude Code or Cursor can invoke CLI operations natively — without shell invocation or manual flag construction."

### A.2 From the internal testing page (VAUL/7039778832)

On the strict serialization rule:
> "Important: Only one command can use the device at a time. Always run device commands one by one — never in parallel."

On the prompt pattern for AI-assisted testing:
> Step 1 — *"Discover my Bitcoin accounts"* / *"Discover my Ethereum accounts"*
> Step 2 — *"What's the balance of [paste your account descriptor here]?"*
> Step 3 — *"Show me the last 10 transactions for [paste your account descriptor]"*
> Step 4 — *"Give me a receive address for [paste your account descriptor]"*
> Step 5 — *"Do a dry-run send of 0.01 ETH from [paste your account] to [any valid address]"*
> Step 6 — *"Send 0.001 ETH from [paste your account] to [recipient address]"*

Verification table (verbatim):

| What to test | Expected result |
| --- | --- |
| Account discovery | Lists your real accounts and addresses |
| Balances | Matches what you see in the Ledger Live app |
| Operations | Shows your real transaction history |
| Receive | Address appears on Ledger screen for confirmation |
| Dry-run send | Shows fee estimate, confirms no funds moved |
| Real send | Ledger prompts on device, tx hash returned on success |

Common errors table (verbatim):

| Error message | What to do |
| --- | --- |
| `Timeout has occurred` | Make sure Ledger is connected via USB and the coin app is open on device |
| `UnknownDeviceExchangeError` | Unplug and replug the Ledger, reopen the coin app, and retry |
| `Transaction Cancelled` | You rejected it on device — this is expected behaviour |
| Command stuck with no output | Wait 10 seconds, cancel, and retry |

### A.3 From the DMK overview (CS/7014678571)

> "The Device Management Kit is the library that sits between clients (for instance the Ledger Wallet) and the physical Ledger signer. In the context of Clear Signing, it is the client-side orchestration layer responsible for fetching ERC-7730 metadata, packaging it alongside the raw transaction, and delivering it to the device over USB or Bluetooth."

> "Without the DMK, Clear Signing metadata would never reach the device — it is the critical bridge that transforms a registry JSON descriptor into a verified, on-device human-readable display."

The role diagram:
> ```
>    Client
>      │
>      ▼
>     DMK
>      │  ① fetches ERC-7730 metadata from Metadata Service
>      │  ② packages metadata + raw transaction into APDU payload
>      │  ③ sends to device over USB / BLE
>      ▼
>   Ledger Signer (device)
>      └─ Ethereum embedded app renders human-readable fields
> ```

On fallback behaviour:
> "The DMK is designed to never block a transaction due to a missing descriptor. If the Metadata Service returns no result, or if signature validation fails, the DMK transparently signals a fallback state to Ledger Wallet. The user is informed that Clear Signing is unavailable and that they will need to blind sign — preserving usability while surfacing the security downgrade clearly."

### A.4 From the LedgerJS vs DMK benchmark (WXP/6995411067)

The clear-signing tier model (the most quotable line):
> "Third-party wallets using LedgerJS have no access to Clear Signing or Transaction Check features at all. All transactions are blind signed.
> Third-party wallets using the DMK without a commercial agreement get access to the DMK's connectivity and developer experience improvements, but Clear Signing and Transaction Checks are gated — only a basic experience is available.
> Third-party wallets using the DMK with a Ledger commercial partnership (origin token) unlock the full 'advanced' experience: complete Clear Signing with ERC-7730 descriptors, Transaction Checks (Blockaid/Cyvers), trusted name resolution, and more."

On end-user experience:
> "With LedgerJS + 3rd-party wallets, all transactions are blind signed: the user sees a raw hex blob on the device and must trust the wallet software entirely. There is no way for the hardware to independently verify or display what the user is actually signing."

On connectivity gains:
> "Internal data shows a >10x reduction in connectivity errors after migrating to DMK."

Supported Signer Kits today:
- `@ledgerhq/device-signer-kit-ethereum` — get address, sign tx, sign message, sign typed data (EIP-712), sign delegation authorization (EIP-7702), verify Safe address
- `@ledgerhq/device-signer-kit-solana`
- `@ledgerhq/device-signer-kit-bitcoin`
- `@ledgerhq/device-signer-kit-hyperliquid`
- `@ledgerhq/device-signer-kit-cosmos`
- `@ledgerhq/device-signer-kit-aleo`
- `@ledgerhq/device-signer-kit-zcash`

### A.5 From the architecture study (TA/6951665759)

On the audit verdict (which legacy commands are reusable):
- **`v0` reuse as-is**: `send` (works modulo not-enough-funds), `sync`, `app` (tested with `--install`).
- **`v0` reuse with refactor**: `receive` (need swipe), `discoverDevices` (graceful exit issue), `listApps` (works only with `--DMK=0`), `liveData`, `balanceHistory`.
- **`v0` not tested**: `broadcast`, `getTransactionStatus`.
- **`v0.5` reuse with refactor**: `ledgerKeyRingProtocol --initMemberCredentials`.
- Everything else (45+ commands) → **not in scope**.

Friction points identified verbatim:
1. "Output is human-readable strings. There is no structured JSON envelope."
2. "Progress messages and results go to the same stdout stream."
3. "Only some commands support `--format json`; the flag name/behavior varies."
4. "Errors are plain `console.error(String(e.message))` — no error codes, no structured body."
5. "Exit code is always `0` (success) or `1` (any error)."
6. "No introspection endpoint."
7. "Several commands have no `description` field."
8. "No timeout control — commands can hang forever waiting for a device."
9. "Some sensitive commands have no dry-run."
10. "No command composition — each command re-scans accounts internally."

Proposed error-code taxonomy:
- Device: `DEVICE_NOT_FOUND=10`, `DEVICE_LOCKED=11`, `DEVICE_APP_NOT_OPEN=12`, `DEVICE_BUSY=13`, `DEVICE_DISCONNECTED=14`
- Tx: `INSUFFICIENT_FUNDS=20`, `INVALID_RECIPIENT=21`, `TRANSACTION_REJECTED=22`, `BROADCAST_FAILED=23`
- Account: `ACCOUNT_NOT_FOUND=30`, `CURRENCY_NOT_SUPPORTED=31`
- Firmware/App: `APP_NOT_FOUND=40`, `FIRMWARE_UP_TO_DATE=41`, `FIRMWARE_UPDATE_FAILED=42`
- Generic: `INVALID_ARGUMENTS=50`, `TIMEOUT=51`, `NETWORK_ERROR=52`, `UNKNOWN_ERROR=99`

Decision on the agentic surface:
> "A narrowed set of commands has been identified as being relevant for agentic usage. They would be made available along with skills for power users: sync, balance / history, swap."

Three options weighed:
1. **MCP server** — pro: any MCP-compliant agent gets capabilities for free; con: end user must install/run the MCP server.
2. **CLI + Skills** — pro: leverage existing CLI; con: per-LLM skill manifest maintenance.
3. **REST API** — pro: remote agents can trigger via web; con: deploy yet another service.
The doc leans toward CLI + Skills (the manifest YAML is fully drafted in the page).

### A.6 From the ADR — CLI Storage (TA/6978928805)

Status: **Proposed**.

Decision (provisional):
> "A simple storage based on filesystem is enough for this scope. … this file (or later files) can be stored in an hidden user directory: `~/.ledger/cli`."
> "For future proof consideration, YAML seems the best choice."

Schema sketch:
```yaml
accounts:
  - label: <LABEL_ACCOUNT#1>
    descriptor: <ACCOUNT_DESCRIPTOR>#1
```

Open question: should `deviceId` go inside the account descriptor (Solution 1) or be tracked at the context-file level (Solution 2)? **TBD.** This collides with the Bunli `stateDir()` choice already implemented in LIVE-29495 (which writes to `~/.local/state/ledger-wallet-cli/session.yaml`, not `~/.ledger/cli/`). Worth flagging in the QA chapter as "two competing storage layouts in flight."

---

## Appendix B — Cross-reference: where each part of the QA story lives

| Story beat | Source of truth | URL |
| --- | --- | --- |
| What is the wallet-cli supposed to do? | PRD | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/6907625515 |
| What does it look like when run? | Internal testing | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/7039778832 |
| Why does it use DMK? | DMK overview + LedgerJS vs DMK | CS/7014678571 + WXP/6995411067 |
| What did the legacy `apps/cli` look like? | AI Agent architecture study | TA/6951665759 |
| What QA pain prompted the revoke story? | QAA-615 (Jira) + B2CQA-2844 | https://ledgerhq.atlassian.net/browse/QAA-615 |
| Where does the CLI store state? | ADR CLI Storage + LIVE-29495 | TA/6978928805 + browse/LIVE-29495 |
| How are clear-signing descriptors fetched? | DMK overview + Clear Signing ERC 7730 | CS/7014678571 + CS/6038847989 |
| Which coins are wired through DMK today? | Coin Modules Developments Mapping | CF/6129025043 |
| What's the Bunli framework choice? | (no Confluence) — Jira only | LIVE-29308, LIVE-29404, LIVE-29495 |
| How does QA reuse a CLI for E2E preconditions today? | Vault E2E testing framework | QA/6736685052 |
| What's the existing legacy install path? | Install and use Ledger Wallet CLI | BO/4065853730 |
| What's the swap test envelope? | Swap :: Update E2E tests + Swap :: Testing Guide | CIP/5493194794 + CIP/5458788446 |
| What's the QA strategy template? | Testability Strategy + QA Evaluation Criteria | QA/6637060671 + QA/6636437536 |

---

## Search trace (what was queried)

Probes attempted (Rovo Search, since CQL was denied):
- `wallet-cli Ledger`
- `Device Management Kit DMK`
- `Bunli`
- `Bunli framework CLI`
- `live-cli ledger-live monorepo apps/cli`
- `CLI testing strategy QA automation`
- `ERC-20 approve revoke allowance Ledger`
- `swap regression testing Ledger Live`

Pages fetched in full (`getConfluencePage`):
- 7039778832 — Ledger Wallet CLI internal testing
- 6907625515 — Ledger Wallet CLI (PRD)
- 7014678571 — Device Management Kit (overview, CS space)
- 6951665759 — Ledger Live CLI for AI Agent (architecture study)
- 6995411067 — LedgerJS vs DMK Benchmark
- 6978928805 — ADR — CLI Storage

Spaces touched (per result metadata): VAUL (LES Knowledge Hub), WXP (Wallet XP), QA, CIP (Coin Integration / Platform), CS (Clear Signing), TA (Architecture), CF (Coin Framework), PTX, BI, BO, ECOM.

Spaces with **no relevant hits** for the wallet-cli topic: Tech Hub (no obvious key surfaced), Live Apps (no hits beyond Swap docs already listed under PTX).
