
# APPENDIX

<div class="chapter-intro">
<strong>Your daily reference shelf.</strong> This appendix consolidates every command, variable, device model, troubleshooting recipe, and term you'll encounter in the Ledger Live QA ecosystem. Bookmark it -- you'll come back here constantly during your first weeks.
</div>

---

## A: Complete Command Reference

### Setup

```bash
mise install                              # Install all tools (Node, pnpm, gitleaks, etc.)
pnpm i                                   # Install all dependencies
pnpm i --filter="package..."             # Install for specific package (+ all its deps)
```

### Desktop E2E Setup

```bash
# Install deps for desktop + CLI + E2E
pnpm i --filter="ledger-live-desktop..." \
       --filter="live-cli..." \
       --filter="ledger-live" \
       --filter="@ledgerhq/dummy-*-app..." \
       --filter="ledger-live-desktop-e2e-tests" \
       --unsafe-perm

# Build dependencies
pnpm build:lld:deps

# Build the CLI (used for test data setup)
pnpm build:cli

# Build the desktop app in testing mode
pnpm desktop build:testing

# Install Playwright browsers
pnpm e2e:desktop test:playwright:setup
```

### Mobile E2E Setup

```bash
# Install deps for mobile + CLI + E2E
pnpm i --filter="live-mobile..." \
       --filter="ledger-live" \
       --filter="live-cli..." \
       --filter="ledger-live-mobile-e2e-tests"

# Build dependencies
pnpm build:llm:deps

# Build the CLI
pnpm build:cli

# === Android (release -- debug builds broken locally) ===
pnpm mobile e2e:build -c android.emu.release

# === iOS (debug) ===
pnpm mobile pod                           # Install CocoaPods
pnpm mobile e2e:build -c ios.sim.debug
```

### Development

```bash
pnpm dev:lld                              # Start Desktop dev server
pnpm dev:llm:ios                          # Start Mobile iOS dev server
pnpm dev:llm:android                      # Start Mobile Android dev server
```

> **Pro tip** -- For live reload during library development, run `pnpm --filter @ledgerhq/<lib-name> run watch` in a separate terminal. Changes to the library will propagate to the consuming app in real time (from wiki: Assorted Tips).

### Building

```bash
pnpm build:lld                            # Build Desktop (+ deps via Turbo)
pnpm build:lld:deps                       # Build Desktop dependencies only
pnpm build:llm:deps                       # Build Mobile dependencies only
pnpm build:libs                           # Build ALL libraries
pnpm build:cli                            # Build CLI tool
pnpm desktop build:testing                # Build Desktop in testing mode (E2E)
pnpm turbo build --filter=@ledgerhq/<lib> # Build a specific library
```

### Linting & Type-Checking

```bash
pnpm lint                                 # Lint entire monorepo
pnpm lint:fix                             # Lint + auto-fix
pnpm typecheck                            # Type-check everything
pnpm desktop typecheck                    # Type-check desktop only
pnpm mobile typecheck                     # Type-check mobile only
pnpm --filter ledger-live-desktop-e2e-tests typecheck  # Type-check desktop E2E
pnpm --filter ledger-live-desktop-e2e-tests lint        # Lint desktop E2E
pnpm --filter ledger-live-mobile-e2e-tests typecheck   # Type-check mobile E2E
```

### Unit & Integration Tests

```bash
pnpm desktop test:jest                    # All desktop unit/integration tests
pnpm desktop test:jest "filename"         # Desktop test file matching "filename"
pnpm mobile test:jest                     # All mobile unit/integration tests
pnpm mobile test:jest "filename"          # Mobile test file matching "filename"
```

### Desktop E2E Tests

```bash
pnpm e2e:desktop test:playwright                      # Run all desktop E2E tests
pnpm e2e:desktop test:playwright settings.spec.ts     # Run a single spec file
pnpm e2e:desktop test:playwright --grep "Settings"    # Filter by test name pattern
pnpm e2e:desktop test:playwright --grep "@smoke"      # Run only smoke tests
pnpm e2e:desktop test:playwright --grep "@NanoSP"     # Run only NanoSP-compatible tests
pnpm e2e:desktop allure:generate                      # Generate Allure report from results
pnpm e2e:desktop allure:open                          # Open Allure report in browser
pnpm e2e:desktop allure                               # Generate + open (combined)

# Debug mode (opens Playwright Inspector):
cd e2e/desktop
PWDEBUG=1 npx playwright test --config=playwright.config.ts settings.spec.ts
```

### Mobile E2E Tests (from e2e/mobile/)

```bash
pnpm test:android                         # All Android tests (release build)
pnpm test:android myTest.spec.ts          # Single file (Android)
pnpm test:ios:debug                       # All iOS tests (requires Metro running)
pnpm test:ios:debug myTest.spec.ts        # Single file (iOS)
pnpm allure                               # Generate + open mobile Allure report
```

> **Note**: iOS debug tests require Metro running in a separate terminal: `pnpm mobile start` (from repo root).

### Git & Versioning

```bash
pnpm commit                               # Interactive conventional commit helper
pnpm changeset                            # Add a changeset for versioning
```

### Docker (Speculos)

```bash
docker pull ghcr.io/ledgerhq/speculos:latest          # Pull latest Speculos image
docker images | grep speculos                          # Verify image exists locally
docker ps --filter name=speculos                       # List running Speculos containers
docker rm -f $(docker ps -aq --filter name=speculos)   # Kill ALL leftover containers
docker logs <container-id>                             # View container logs
docker run --rm ghcr.io/ledgerhq/speculos:latest --help  # Verify Docker image works

# Manual Speculos launch (for debugging):
docker run --rm -it -p 5000:5000 \
  -v ~/coin-apps:/speculos/apps \
  ghcr.io/ledgerhq/speculos:latest \
  --model nanosp \
  --sdk 1.0 \
  apps/nanoSP/1.1.1/Ethereum/app_1.13.0.elf
```

### Speculos REST API (Manual Testing)

```bash
# Check Speculos is alive
curl http://127.0.0.1:5000/

# Take a screenshot
curl http://127.0.0.1:5000/screenshot -o screenshot.png

# Get current screen events (text displayed)
curl http://127.0.0.1:5000/events

# Press buttons (non-touch devices)
curl -X POST http://127.0.0.1:5000/button/left
curl -X POST http://127.0.0.1:5000/button/right
curl -X POST http://127.0.0.1:5000/button/both    # Confirm

# Touch screen (touch devices: Stax, Flex, Gen5)
curl -X POST http://127.0.0.1:5000/finger -d '{"action":"press-and-release","x":200,"y":400}'

# Send raw APDU command
curl -X POST http://127.0.0.1:5000/apdu -d '{"data":"e003000000"}'

# Kill the Speculos session
curl -X DELETE http://127.0.0.1:5000/
```

### Ledger Live Bot Commands

```bash
# Run the bot (from live-common or CLI)
pnpm build:cli && pnpm run:cli bot --seed "your 24 word seed" -f bitcoin

# Available flags:
#   --seed <mnemonic>       BIP39 mnemonic to use
#   -f, --family <family>   Coin family to test (bitcoin, ethereum, solana, etc.)
#   --device <model>        Device model for Speculos (nanoSP, stax, etc.)
#   --mutations <count>     Max number of mutations per account
```

### Performance Profiling

```bash
# Renderer process profiling (from wiki: Performance Tips)
# 1. Open DevTools in Ledger Live Desktop (Ctrl+Shift+I)
# 2. Go to Performance tab > Record
# 3. Reproduce the slow action
# 4. Stop recording > Analyze flame chart

# Main process profiling
# Start desktop with --inspect flag:
pnpm dev:lld -- --inspect
# Then open chrome://inspect in Chrome to connect to the Node.js debugger
```

---

## B: Environment Variables Reference

### Required for E2E Testing

| Variable | Example | Description |
|---|---|---|
| `MOCK` | `0` | `0` = use real Speculos (Docker). `1` = mocked device (no Docker, uses stub responses). Always `0` for real E2E tests. |
| `SPECULOS_DEVICE` | `nanoSP` | Device model to emulate. Values: `nanoS`, `nanoSP`, `nanoX`, `stax`, `flex`, `nanoGen5` |
| `SPECULOS_IMAGE_TAG` | `ghcr.io/ledgerhq/speculos:latest` | Docker image for Speculos. Use `latest` unless debugging a specific version. |
| `COINAPPS` | `~/coin-apps` | Path to the cloned coin-apps repository. Required for local runs. Auto-set in CI. |

### Optional / Advanced

| Variable | Example | Description |
|---|---|---|
| `SEED` | `abandon abandon ... about` | BIP39 mnemonic (24 words) for deterministic account generation. Stored in CI secrets. Ask your team lead for the local value. |
| `E2E_ENABLE_WALLET40` | `1` | Enable Wallet 4.0 UI during tests. Changes navigation and layout significantly. |
| `E2E_FEATURE_FLAGS_JSON` | `{"myFlag":{"enabled":true}}` | Override feature flags as a JSON string. Merged with defaults. |
| `SPECULOS_API_PORT` | `5000` | REST API port for Speculos. Auto-assigned by the fixture system (you rarely set this manually). |
| `SPECULOS_ADDRESS` | `http://127.0.0.1` | Base URL for Speculos REST API. Auto-set by the fixture system. |
| `DISABLE_TRANSACTION_BROADCAST` | `1` | Prevent transactions from being broadcast to the real blockchain. Always `1` in tests. |
| `NODE_OPTIONS` | `--max-old-space-size=14336` | Node.js memory limit. CI uses 14GB. Increase locally if you get OOM errors. |

### CI-Specific

| Variable | Description |
|---|---|
| `GITHUB_TOKEN` | GitHub API token for authenticated Docker pulls from `ghcr.io` and CI operations |
| `ALLURE_SERVER_URL` | URL of the Allure reporting server for uploading results |
| `XRAY_CLIENT_ID` / `XRAY_CLIENT_SECRET` | Credentials for Jira/Xray test result import |

### Development

| Variable | Description |
|---|---|
| `DEBUG` | Enable debug logging. Formats: `speculos:*`, `live-common:*`, `hw-transport:*` |
| `VERBOSE` | Enable verbose output in CLI commands |
| `PLAYWRIGHT_TRACE` | Set to `on` to capture Playwright traces for debugging |

---

## C: Supported Devices & Coins

### Device Models

| Device | Env Value | Touch | Display | Screen Size | Tag | Form Factor | Notes |
|--------|-----------|-------|---------|-------------|-----|-------------|-------|
| **Nano S** | `nanoS` | No | BAGL | 128x32 | `@LNS` | USB stick | Legacy. Limited memory. Requires specific Docker image SHA. |
| **Nano S Plus** | `nanoSP` | No | BAGL | 128x64 | `@NanoSP` | USB stick | Most common test device. Good default for development. |
| **Nano X** | `nanoX` | No | BAGL | 128x64 | `@NanoX` | USB stick + BLE | Bluetooth not relevant in Speculos. |
| **Stax** | `stax` | **Yes** | NBGL | 400x672 | `@Stax` | Card/touchscreen | Touch interactions: tap, swipe, long press. |
| **Flex** | `flex` | **Yes** | NBGL | 480x600 | `@Flex` | Card/touchscreen | Similar to Stax, different form factor. |
| **Nano Gen 5** | `nanoGen5` | **Yes** | NBGL | -- | `@NanoGen5` | USB + touchscreen | Newest device. Internally codenamed "apex". |

### Device Interaction Comparison

| Action | Non-Touch (Nano S/SP/X) | Touch (Stax/Flex/Gen5) |
|--------|-------------------------|------------------------|
| Navigate screens | `pressRight()` / `pressLeft()` | `swipe("left")` / `swipe("right")` |
| Confirm/select | `pressBoth()` | `tap(x, y)` on "Approve" label |
| Cancel | Navigate to "Reject" + `pressBoth()` | `tap(x, y)` on "Reject" label |
| Long confirm | N/A | `longPress(x, y, 3)` (hold for 3 seconds) |

### Coin Families & Modules

| Family | Package | Representative Chains | E2E Coverage |
|--------|---------|----------------------|--------------|
| **EVM** | `@ledgerhq/coin-evm` | Ethereum, Polygon, BSC, Arbitrum, Optimism, Avalanche, Base, Fantom, Cronos, Moonbeam, Blast, Linea, Scroll, zkSync Era | Extensive |
| **Bitcoin** | `@ledgerhq/coin-bitcoin` | Bitcoin, Litecoin, Dogecoin, Bitcoin Cash, Zcash, Dash, Decred, Qtum, Komodo, Peercoin | Extensive |
| **Solana** | `@ledgerhq/coin-solana` | Solana | Yes |
| **Cosmos** | `@ledgerhq/coin-cosmos` | Cosmos (ATOM), Osmosis, dYdX, Injective, Sei, Celestia, Stargaze, Persistence, Quicksilver, Onomy | Yes |
| **Cardano** | `@ledgerhq/coin-cardano` | Cardano (ADA) | Yes |
| **Polkadot** | `@ledgerhq/coin-polkadot` | Polkadot (DOT) | Yes |
| **Tezos** | `@ledgerhq/coin-tezos` | Tezos (XTZ) | Yes |
| **Algorand** | `@ledgerhq/coin-algorand` | Algorand (ALGO) | Yes |
| **Stellar** | `@ledgerhq/coin-stellar` | Stellar (XLM) | Yes |
| **XRP** | `@ledgerhq/coin-xrp` | XRP | Yes |
| **Tron** | `@ledgerhq/coin-tron` | Tron (TRX) | Yes |
| **NEAR** | `@ledgerhq/coin-near` | NEAR Protocol | Yes |
| **MultiversX** | `@ledgerhq/coin-elrond` | MultiversX (EGLD) | Yes |
| **Aptos** | `@ledgerhq/coin-aptos` | Aptos (APT) | Partial |
| **Sui** | `@ledgerhq/coin-sui` | Sui | Partial |
| **Hedera** | `@ledgerhq/coin-hedera` | Hedera (HBAR) | Partial |
| **Vechain** | `@ledgerhq/coin-vechain` | Vechain (VET) | Partial |
| **Internet Computer** | `@ledgerhq/coin-internet-computer` | ICP | Partial |
| **Kaspa** | `@ledgerhq/coin-kaspa` | Kaspa | Partial |
| **Celo** | `@ledgerhq/coin-celo` | Celo | Partial |
| **Filecoin** | `@ledgerhq/coin-filecoin` | Filecoin (FIL) | Partial |
| **Stacks** | `@ledgerhq/coin-stacks` | Stacks (STX) | Partial |
| **Mina** | `@ledgerhq/coin-mina` | Mina Protocol | Partial |
| **Aleo** | `@ledgerhq/coin-aleo` | Aleo | Partial |

> **80+ total supported cryptocurrencies** across these families, including ERC-20 tokens (USDT, USDC, DAI, LINK, UNI, etc.) and other chain-specific tokens.

---

## D: Troubleshooting FAQ

### Docker & Speculos

**Q: `docker pull ghcr.io/ledgerhq/speculos:latest` fails with "unauthorized"**
A: The image is on GitHub Container Registry. Authenticate first:
```bash
echo $GITHUB_TOKEN | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

**Q: Speculos container won't start / immediately exits**
A: Kill all leftover containers and pull a fresh image:
```bash
docker rm -f $(docker ps -aq --filter name=speculos)
docker pull ghcr.io/ledgerhq/speculos:latest
```

**Q: "Port 5000 already in use" error**
A: A previous Speculos container didn't shut down cleanly:
```bash
# Kill all Speculos containers
docker rm -f $(docker ps -aq --filter name=speculos)
# Or kill any process on port 5000
lsof -ti:5000 | xargs kill -9
```

**Q: Docker Desktop is not running**
A: Start Docker Desktop from Applications. Verify with `docker ps`. On macOS, you can also run `open -a Docker`.

**Q: "no app found for Solana on nanoSP"**
A: The coin-apps repo is missing or outdated:
```bash
# Clone if not already done
git clone https://github.com/LedgerHQ/coin-apps.git ~/coin-apps
export COINAPPS=~/coin-apps

# Or pull latest if already cloned
cd ~/coin-apps && git pull
```

### Build Issues

**Q: `pnpm desktop build:testing` fails**
A: Make sure all dependencies are built first (order matters):
```bash
pnpm build:lld:deps    # Build library dependencies
pnpm build:cli          # Build CLI tool
pnpm desktop build:testing  # Then build the desktop app
```

**Q: TypeScript error: Module '@ledgerhq/xxx' has no exported member 'yyy'**
A: The library needs to be rebuilt after recent changes:
```bash
pnpm turbo build --filter=@ledgerhq/xxx
```

**Q: `pnpm i` fails or hangs**
A: Try cleaning the cache and retrying:
```bash
rm -rf node_modules
pnpm store prune
pnpm i
```

**Q: Out of memory during build (`JavaScript heap out of memory`)**
A: Increase the Node.js memory limit:
```bash
export NODE_OPTIONS="--max-old-space-size=8192"  # 8 GB
```

**Q: iOS pod install fails**
A: Make sure you have the right Xcode version, then clean and retry:
```bash
cd apps/ledger-live-mobile/ios
rm -rf Pods Podfile.lock
cd ../../..
pnpm mobile pod
```

**Q: `pnpm-lock.yaml` merge conflict during rebase**
A: This is the most common merge conflict in the monorepo (from wiki: Issues & Workarounds):
```bash
# Do NOT manually resolve pnpm-lock.yaml conflicts
# Instead, accept either version and regenerate:
git checkout --theirs pnpm-lock.yaml
pnpm i
git add pnpm-lock.yaml
git rebase --continue
```

**Q: Node Action Cache Error in CI**
A: Known CI issue (from wiki: Common CI Troubleshooting). The node_modules cache may be corrupted. Re-run the CI job -- it usually resolves on retry. If persistent, ask a maintainer to purge the GitHub Actions cache.

### Test Issues

**Q: All tests timeout immediately**
A: Check these in order:
1. Docker is running: `docker ps`
2. Env vars are set: `echo $MOCK $SPECULOS_DEVICE`
3. App is built: `ls apps/ledger-live-desktop/.webpack/main.bundle.js`
4. Playwright browsers installed: `pnpm e2e:desktop test:playwright:setup`

**Q: Tests pass locally but fail in CI**
A: Common causes:
- Different device model (CI may test `nanoS`, locally you test `nanoSP`)
- Feature flag differences (CI uses production defaults, you may have overrides)
- Timing issues (CI runners may be slower -- increase timeouts)
- Missing coin-app binary for the CI device model in coin-apps repo

**Q: "Error: SEED not set"**
A: The `SEED` environment variable contains the BIP39 mnemonic used to generate deterministic test accounts. In CI it comes from GitHub Secrets. Locally, ask your team lead for the test SEED value and add it to your `~/.zshrc`.

**Q: Flaky test -- intermittent "Element not found" errors**
A: Add explicit waits before interacting with the element:
```typescript
// BAD
await page.getByTestId("send-button").click();

// GOOD
await page.getByTestId("send-button").waitFor({ state: "visible", timeout: 30000 });
await page.getByTestId("send-button").click();
```

**Q: Allure report is empty or missing tests**
A: Check that `allure-results/` directory exists and contains JSON files:
```bash
ls e2e/desktop/allure-results/
# Should contain .json files from the test run
```
If empty, the tests may not have run or the reporter config is wrong.

**Q: "Element not found" after a UI update**
A: The test ID in the source code likely changed. Find the new test ID:
```bash
# Search for the old test ID in the app source
grep -r "old-test-id" apps/ledger-live-desktop/src/
# Then update the page object with the new ID
```

### Mobile-Specific Issues

**Q: Metro bundler won't start**
A: Port 8081 may be occupied. Kill the existing process:
```bash
lsof -ti:8081 | xargs kill -9
pnpm mobile start
```

**Q: iOS simulator not detected by Detox**
A: Verify the simulator is available:
```bash
xcrun simctl list devices available | grep -i iphone
# If no devices, create one in Xcode > Window > Devices and Simulators
```

**Q: Android emulator not detected**
A: Check AVD and adb:
```bash
# List available emulators
emulator -list-avds
# Start one
emulator -avd Pixel_7_Pro_API_35
# Check adb sees it
adb devices
```

**Q: WebSocket bridge connection timeout (mobile E2E)**
A: The bridge port may not be forwarded. For Android:
```bash
# Check that adb reverse is set
adb reverse tcp:8099 tcp:8099
```
For iOS, the bridge connects directly -- check that the simulator is running and the app launched.

**Q: Android E2E tests crash immediately in debug mode**
A: Known issue -- Android debug builds are broken locally due to a Detox/Espresso bug. Always use release builds:
```bash
pnpm mobile e2e:build -c android.emu.release
```

---

## E: Glossary

| Term | Definition |
|------|------------|
| **APDU** | Application Protocol Data Unit. The low-level binary protocol used for all communication between a host application and a smartcard/secure element (Ledger device). Commands are sent as byte arrays. |
| **Allure** | An open-source test reporting framework that generates interactive HTML reports with step traces, screenshots, videos, and environment information. |
| **BAGL** | Ledger's display framework for non-touch devices (Nano S, Nano S Plus, Nano X). Renders UI on small monochrome screens (128x32 or 128x64 pixels). |
| **BIP39** | Bitcoin Improvement Proposal 39. Defines the standard for mnemonic seed phrases (12 or 24 words) used to derive cryptographic keys deterministically. |
| **Bot (Ledger Live Bot)** | A stateless, autonomous test engine that crawls all accounts in a seed, discovers possible operations, and executes randomized mutations. Uses 7 named seeds (Mere Denis through Nitrogen). |
| **Changeset** | A small markdown file (in `.changeset/`) describing changes to a published package. Used by the changesets tool to generate changelogs and version bumps. |
| **CI/CD** | Continuous Integration / Continuous Deployment. Automated pipeline that builds, tests, and deploys code on every change. Ledger Live uses GitHub Actions. |
| **CLI** | Command-Line Interface. Ledger Live's CLI tool (`apps/cli/`) is used in E2E tests to populate test data (accounts, transactions). |
| **CODEOWNERS** | A GitHub file (`.github/CODEOWNERS`) that defines which team must review PRs touching specific file paths. Enforces team ownership. |
| **Coin App** | A compiled firmware application (`.elf` binary) that runs on a Ledger device to support a specific cryptocurrency (e.g., the Ethereum app, Bitcoin app). |
| **Coin Family** | A group of related blockchains sharing the same transaction model and coin module. E.g., the EVM family includes Ethereum, Polygon, BSC, Arbitrum, etc. |
| **Content-Addressable Store** | pnpm's storage mechanism where every package version is stored exactly once, identified by its content hash. Saves disk space in monorepos. |
| **Conventional Commits** | A commit message standard: `type(scope): description`. Enables automated changelog generation and semantic versioning. |
| **Detox** | A grey-box E2E testing framework for React Native by Wix. "Grey-box" means it has visibility into app internals for synchronization. |
| **E2E** | End-to-End testing. Tests that simulate real user interactions across the full application stack, from UI to backend to device. |
| **ELF** | Executable and Linkable Format. The binary format used for compiled Ledger coin applications that run inside Speculos. |
| **Electron** | A framework for building cross-platform desktop applications using web technologies (Chromium + Node.js). Ledger Live Desktop is an Electron app. |
| **Feature Flag** | A configuration toggle stored in Firebase Remote Config that enables or disables features without deploying new code. Supports gradual rollouts and kill switches. |
| **Firebase Remote Config** | Google's cloud service used by Ledger Live to store and serve feature flags. Four environments: Ledger Wallet, Swap, Earn, Buy Sell. |
| **Fixture** | In Playwright, a mechanism for declaring reusable test setup and teardown logic. Fixtures are injected into tests via parameter destructuring. The Ledger Live fixture system handles Speculos, Electron, page objects, and feature flags. |
| **Flaky Test** | A test that intermittently passes or fails without any code changes. Usually caused by timing issues, race conditions, or shared state. |
| **Grey-Box Testing** | Testing approach where the test framework has some visibility into the application's internals (e.g., synchronization with React Native bridge). Detox uses this approach. |
| **hk** | A modern pre-commit hook manager (alternative to Husky) used in the Ledger Live repo. Configured via `hk.pkl`. Runs `gitleaks` to scan for secrets. |
| **IPC** | Inter-Process Communication. In Electron, the mechanism for communication between the main process (Node.js) and renderer process (Chromium). Uses `ipcMain` and `ipcRenderer`. |
| **Jest** | A JavaScript testing framework by Meta (Facebook). Used for unit and integration tests in Ledger Live. Also serves as the test runner for Detox mobile E2E tests. |
| **Ledger Live** | Ledger's official companion application for managing crypto assets. Available as Desktop (Electron), Mobile (React Native), and CLI. |
| **Mise** | A development tool version manager (alternative to nvm, asdf). Used in Ledger Live to manage Node.js, pnpm, and other tool versions consistently. |
| **Mnemonic Seed** | A set of words (usually 24) that encodes a master cryptographic key per BIP39. The same seed always generates the same accounts, enabling deterministic test data. |
| **Monorepo** | A single Git repository containing multiple related projects and packages. Ledger Live is a monorepo with 2 apps, 40+ libraries, 2 E2E suites, and CI tools. |
| **MSW** | Mock Service Worker. A library for intercepting and mocking HTTP requests in tests. Used in Jest integration tests to mock API calls without a real server. |
| **Mutation (Bot)** | A single operation the Ledger Live Bot attempts on an account (e.g., send max, send sub-max, delegate, swap). Each mutation has preconditions, a transaction builder, and assertion checks. |
| **MVVM** | Model-View-ViewModel. An architecture pattern used in newer Ledger Live code. Components are split into Container (wiring), ViewModel (logic), and View (pure UI). |
| **NBGL** | New BAGL. Ledger's modern display framework for touch-screen devices (Stax, Flex, Nano Gen 5). Supports full-color touchscreen interactions. |
| **Page Object Model (POM)** | A design pattern where each UI page or component is represented as a class with locators (element selectors) and methods (user actions). Reduces test code duplication. |
| **Playwright** | A browser automation framework by Microsoft. Used for Ledger Live Desktop E2E testing. Supports Chromium, Firefox, WebKit, and Electron applications. |
| **pnpm** | A fast, disk-efficient package manager optimized for monorepos. Uses a content-addressable store and symlinks instead of copying files. |
| **RTK Query** | A data fetching and caching tool built on Redux Toolkit. Provides auto-generated React hooks for API calls with built-in caching and invalidation. |
| **Rspack** | A Rust-based web bundler that's a drop-in replacement for Webpack, offering 5-10x faster build times. Used for the Ledger Live Desktop build. |
| **SEED** | In the context of E2E tests, the BIP39 mnemonic phrase used to generate deterministic test accounts and addresses in Speculos. Stored as a CI secret. |
| **Sharding** | Splitting a test suite into multiple parallel groups (shards) to reduce total execution time. Ledger Live CI shards desktop E2E tests across multiple runners by device model. |
| **Speculos** | Ledger's open-source device emulator. Runs real device firmware inside a Docker container, exposing a REST API for automated testing without physical hardware. |
| **Team-Split Convention** | A repo convention where multi-team folders are split into `team-<slug>/` subfolders, each owned by a specific team in CODEOWNERS. |
| **TMS** | Test Management System. In Ledger Live, refers to Jira/Xray test case IDs (e.g., `B2CQA-817`). Tests link to TMS IDs via annotations for traceability. |
| **Turbo / Turborepo** | A high-performance monorepo build orchestration tool by Vercel. Manages task dependencies, caching, and parallel execution across packages. |
| **Userdata** | Pre-configured JSON files (in `e2e/desktop/tests/userdata/`) containing serialized app state. Used to start tests from a known state without going through onboarding. |
| **WebSocket Bridge** | A communication channel between the mobile E2E test harness (Jest/Detox) and the React Native app. Enables commands like `importAccounts`, `overrideFeatureFlag`, and `navigate`. |
| **Xray** | A Jira plugin for test management. Tracks test cases, test execution, and links automated test results back to Jira tickets. Results exported as JSON. |

---

<div class="chapter-outro">
<strong>You've reached the end of the Appendix -- and the end of this guide.</strong><br>
Use these references whenever you need to look up a command, variable, device, or term. Keep this guide bookmarked -- you'll refer to it daily during your first weeks.<br><br>
Welcome to the team. Ship quality. Break nothing. Test everything.
</div>
