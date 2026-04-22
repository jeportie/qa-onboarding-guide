# R2 — Ledger Products, Hardware, BOLOS, Firmware, Recover, Enterprise, Dev Platform

Research agent R2 deliverable for the Ledger QA onboarding guide restructure.
All facts below are grounded in Confluence pages actually read; source page IDs
are cited inline. Where a fact is not directly attested, it is flagged as a gap.

---

## 1. Hardware catalog

The canonical source is Ledger OS - Introduction (FW/5737481596, last modified
16 Feb 2026, author Raphael Geslain), which enumerates every shipped Ledger
hardware signer and the chips they use. The Ledger Flex spec sheet
(LEU/4566614017, author Benoit Lucet) is the ground truth for Flex/Stax
physical and display characteristics. For the touch device line, Mobile Device
Onboarding (Engagement/7011795014) confirms that Stax, Flex (Europa) and Apex
share the Sync Onboarding flow.

| Device             | Form factor              | SE chip                | MCU                     | Screen                              | Status (current / legacy)        | Source |
|--------------------|--------------------------|------------------------|-------------------------|-------------------------------------|----------------------------------|--------|
| Ledger Nano S      | USB-A stick, 2 buttons   | ST31H320               | STM32F042               | OLED 128x64 monochrome              | Legacy (not supported in Multisig, see VG-25622) | FW/5737481596 |
| Ledger Blue        | Early large touch device | ST31G480               | STM32L476               | Touch (historical)                  | Legacy                            | FW/5737481596 |
| Ledger Nano X      | USB-C, BLE, 2 buttons    | ST33J2M0 (EAL5+)       | STM32WB55 / STM32WB35   | OLED 128x64 monochrome              | Current (classic line)            | FW/5737481596, LEU/4566614017 |
| Ledger Nano S Plus | USB-C, 2 buttons         | ST33K1M5C (EAL6+)      | STM32F042               | OLED 128x64 monochrome              | Current (classic line)            | FW/5737481596, LEU/4566614017 |
| Ledger Stax        | Touch, curved E-Ink      | ST33K1M5C (EAL6+)      | STM32WB35               | E-Ink 400x672, 16 greys, curved, flexible | Current (touch line)      | FW/5737481596, LEU/4566614017 |
| Ledger Flex        | Touch, flat E-Ink        | ST33K1M5 (EAL6+)       | STM32WB35               | E-Ink 600x480, 2.84 in, 271 ppi, Gorilla Glass | Current (touch line)   | LEU/4566614017 |
| Ledger Nano Gen5   | Classic form factor, next gen | ST33K1M5C         | STM32WB35               | (not documented in pages fetched)   | Current / Upcoming classic replacement | FW/5737481596, HTS/6095274089 |

Additional context:

- Secure Element memory characteristics (FW/5737481596): ST31H320 = 320 KB flash / 10 KB RAM; ST31G480 = 480 KB / 12 KB; ST33J2M0 = 2 MB / 50 KB; ST33K1M5C = 1.5 MB / 64 KB. On Flex about 1.15 MB of the 1.5 MB is available for users apps, language, and picture (LEU/4566614017).
- Stax and Flex both support USB-C, BLE 5.2 and NFC. Stax additionally has Qi wireless charging and curved flexible display; Flex does not stack (stacking is Stax-exclusive) (LEU/4566614017).
- Security certifications: Nano X has CC EAL5+ SE and ANSSI CSPN; Nano S Plus has CC EAL6+ SE and ANSSI CSPN; Stax and Flex have CC EAL6+ SE with ANSSI CSPN in progress / to be done at the time the Flex page was last edited (LEU/4566614017).
- Europa is the internal codename for Ledger Flex (NFC (Stax / Europa), FW/4553539605; OBKR-138 Ship Europa).
- Apex appears in Mobile Device Onboarding (Engagement/7011795014) as a future device also on the Sync Onboarding path alongside Stax and Flex (see Gaps).
- The internal Secure Element personalisation machine is BGIv2, designed to personalise the SE of new products (Stax, Flex, Nano Gen5) (HTS/6233456749), signalling that Nano Gen5 is the next classic-form Ledger device being manufactured.

---

## 2. Software catalog

Canonical list: List of Ledger Product/Services (TH/6898319408, last modified 13 Mar 2026). Ownership and scope reconciled with LLI/5644255440 Ledger Wallet (Ledger Live), WAL/4172578925, CF/6064900341 and VAUL/overview.

| Product                              | Platform(s)                  | Scope                                                                                           | Team owner (where attested)             | Source |
|--------------------------------------|------------------------------|-------------------------------------------------------------------------------------------------|-----------------------------------------|--------|
| Ledger Live Desktop (LLD)            | Windows, macOS, Linux        | Companion app; device management, accounts, swap, buy/sell, staking, NFTs                       | Ledger Wallet / Live org                | TH/6898319408, LLI/3545759821, LLI/5644255440 |
| Ledger Live Mobile (LLM)             | iOS, Android                 | Mobile companion app; mirrors LLD feature set                                                   | Ledger Wallet / Live org                | TH/6898319408, LLI/5644255440 |
| Swap UI + Swap services              | In Ledger Live + backend     | Crypto-to-crypto swaps via providers                                                            | Live + backend teams                    | TH/6898319408 |
| Buy/Sell (PTX) + services            | In Ledger Live + backend     | Fiat-to-crypto flows; live apps built by partners                                               | PTX team                                | PTX/3826090020 |
| Staking / Rewards UI + services      | In Ledger Live + backend     | Staking flows per coin module                                                                   | Live + Coin Integration                 | TH/6898319408 |
| NFT gallery                          | In Ledger Live               | NFT management                                                                                  | Live                                    | TH/6898319408 |
| Wallet API (v3 react-client)         | JS lib + live apps           | API that lets Live Apps talk to Ledger Live (accounts, sign tx, storage)                        | Wallet API team (WAL space)             | WAL/4172578925, WALLETCO/4243522282 |
| Live Apps (Discover + Buy/Sell + WalletConnect + partners) | Webviews inside LLD/LLM | Third-party and first-party apps embedded in Live; use the Wallet API                           | Live Hub / PTX / partners               | WALLETCO/4243522282, PTX/3826090020, ENGB2C/4037181769 |
| Device Management Kit (DMK)          | JS SDK                       | Host-side SDK to connect to devices, install apps, run OS updates                               | WXP team                                | WXP/4251680783, WXP/4305813511, WXP/6943703128 |
| Embedded C SDK (ledger-secure-sdk)   | Native on-device             | C SDK for BOLOS apps                                                                            | Embedded / Firmware                     | FW/5737481596 |
| Embedded Rust SDK (ledger-device-rust-sdk) | Native on-device       | Rust SDK for BOLOS apps                                                                         | Embedded / Firmware                     | FW/5737481596 |
| Coin apps (Bitcoin App, Ethereum App, etc.) | On-device (Secure Element) | Per-blockchain signing and protocol logic, built on BOLOS                                 | Coin Integration + coin-specific teams  | TH/6898319408, FW/3971186947 |
| Coin modules / Coin Framework (TS)   | Ledger Live TS libraries     | Per-coin libraries for balances, tx, staking; split coin-module-framework + coin-modules        | Coin Integration team / Cloud Wallet    | CF/6064900341, CF/6129025043, CF/6949896196 |
| Alpaca                               | Backend service              | REST service exposing coin modules (HTTP); moved via ADR-025                                    | Cloud Wallet team                       | VAUL/5738037406, CF/6949896196 |
| Ledger Recover                       | In-app flow + web frontend + backend | Paid seed backup / restore service, based on Shamir and IDV                            | Trust Services / Consumer Services      | TrustServices/4282253487, CN/5925896228, BO/6148030569 |
| Ledger Recover V2 / Multi-user       | Roadmap                      | Adds trusted-person flows and AI/deepfake resistant IDV                                         | Consumer Services / Trust               | CN/6747193449, CN/6412271698 |
| Ledger Enterprise (Vault UI + HSM + orchestrator) | Web app + HSM + Vault App on Stax | B2B multi-auth custody; HSMs + PSDs + governance rules                             | LES (Ledger Enterprise Services) BU     | VAUL/overview, VAUL/2969534947, VAUL/4136861922 |
| Ledger Enterprise Mobile App         | Mobile                       | Companion to Ledger Vault; review / track requests                                              | LES                                     | VAUL/5751275617 |
| Ledger Multisig                      | Web app                      | Secure interface for Safe multisig clear-signing                                                | Dedicated product (VG Jira)             | VG-25161, VG-25622 |
| CL Card (powered by Ledger)          | Card + backend               | Card + wallet-to-card payment flows                                                             | Card / Payments                         | TH/6898319408 |
| HSM (B2B Vault HSM, Ledger HSM)      | Physical HSM appliances      | Personalisation of SE at factory, Vault governance, transaction signatures                      | Infra / LES                             | INFRASD/4170809356, HTS/6233456749 |

---

## 3. BOLOS primer (SE / MCU split, app model, firmware release flow)

### 3.1 What BOLOS is

From the canonical Architecture page (ENGB2C/3644882985, [Arch] Introduction to BOLOS):

> BOLOS is the name of the firmware embedded in all Ledger Hardware Wallets (stands for Blockchain Open Ledger Operating System). BOLOS provides a lightweight, open-source framework for developers to build source code portable applications that run in a secure environment.

It is organised into six modules: input/output (USB, BLE), cryptography (with hardware acceleration), persistent storage, personalisation (master seed interface), endorsement & application attestation, and user interface.

A privileged Dashboard app runs on BOLOS and is what the user sees when no other app is launched. The Dashboard is also what the host PC or phone talks to when loading/deleting apps, reading firmware version, getting/setting the device name, and updating the firmware (ENGB2C/3644882985).

### 3.2 Two-chip architecture (SE + MCU)

Ledger OS - Introduction (FW/5737481596) is explicit: a Ledger signer is built around two chips:

- A Secure Element (SE) is a specialised microcontroller evolved from the smartcard industry (banking / SIM / passport chips). The SE has active shielding, encrypted memory, sensors (light, voltage, temperature) that can mute or wipe the chip, and a scrambled internal layout. Certified CC EAL5+ or EAL6+ depending on the model. Ledger buys blank SEs from the foundry (ST) and loads its own OS, which is the differentiator vs competitors that use off-the-shelf chips (FW/3295281471 Secure Elements).
- A general-purpose MCU (STM32 family) drives peripherals: USB, BLE, NFC, power management, and, on Nano X / S Plus / Stax / Flex, the display.

Ledger OS is the combination of the SE-side embedded software, the MCU-side embedded software, and the SEPH protocol over the link between them. Inputs (USB, BLE, NFC, power button) are received on the MCU and forwarded to the SE; the SE curates and processes them (FW/5737481596, FW/5732106306).

The SE side is split into:

- HAL (Hardware Abstraction Layer) driving SE features / cryptography.
- Board System Package driving SE-connected peripherals.
- Kernel (privileged, direct peripheral access) and Userland (restricted), separated by the syscall barrier. Syscalls enforce parameter checks, distinct RAM, distinct NVM and a distinct stack (FW/5737481596).

### 3.3 Security model

Three-pillar model (FW/5737481596):

1. Security at rest: secrets live in the SE.
2. Security at use: PIN-protected; three wrong PIN attempts wipe memory; communication protocols secured; Ledger OS curates incoming data.
3. Secure display: the display and buttons/touch panel are directly wired to the SE; only SE-side code can request a display update.

### 3.4 BOLOS application model

From [ARCH] BOLOS application & SDK (TA/3478093827):

- A BOLOS app is native bytecode + RAM data + flash permanent storage. The memory mapping is pseudo-static and set at compilation time.
- Because the Cortex-M MPU does not support address virtualisation, the link file uses virtual addresses (0xC0DE0000 for code, 0xDA7A0000 for data on Nano X and S Plus) and a PIC macro converts them at runtime. Nano X has a ST33 with an MMU so PIC behaviour is adapted.
- One app runs at a time. The selected app gets all the RAM. Before launch, BOLOS programs the MPU/MMU with the app descriptor (code size, storage size).
- Apps communicate with BOLOS through syscalls (SYS_Call) which are ARM software exceptions. The kernel runs the syscall in supervisor mode, then returns to user mode.
- Events are TLV-formatted messages. Apps read events via io_seph_recv(...), handle them (e.g. SEPROXYHAL_TAG_CAPDU_EVENT, SEPROXYHAL_TAG_BUTTON_PUSH_EVENT), and post replies via typed BOLOS APIs or io_seph_send. A Try/Catch/Throw macro system (built on setjmp/longjmp) is used for error handling.
- On Nano S the MCU drives the display (app sends SEPROXYHAL_TAG_SCREEN_DISPLAY_STATUS). On Nano X / S Plus the SE drives the display directly via screen_* APIs.

The Embedded SDK also contains shared modules (NBGL UI, standard app APDU parser, etc.). There is active work to move common code from the SDK into BOLOS itself (FW/6273138715, FW/6173196300 tech debt reduction).

### 3.5 Firmware release flow

Process for firmware releases (NANO/2869264439) documents a four-phase process (originally written around LNX 2.0.0 but still the reference):

1. Scoping & planning: firmware + product define scope (features, security fixes from the Donjon, hardware changes, UX), define the release timeline and the update paths (which previous versions can reach the new one). Kick-off with all stakeholders: firmware, hardware, production, Ledger Live, coin integration, marketing, customer success.
2. Development: firmware team implements scope; UI/wording can be parallelised; Live dependencies are developed in parallel where required.
3. Testing: RC1 is pushed to provider 1 plus recompiled apps. QA is split across Live QA (end-to-end, update paths, Live-compatible blockchains, USB/BLE), General firmware / non-regression QA (Kiev team: onboarding, latest update path, PIN, passphrase, non-regression, third-party wallets, non-Live blockchains) and Product QA (changelog and all update paths). The production team ships devices for QA via a CUS ticket. Tests are also automated via Speculos and on Calvados CI.
4. Release: marketing and CS prepare comms and Help Center updates; the FW team publishes the final version on provider 1 the morning of release with progressive rollout (typically 10-25% on D, 50-60% on D+1, 100% by end of week). Mixpanel / Periscope / Zendesk analytics are tracked daily; a go/no-go decision is taken each day.

Two release types: public updates (via Ledger Live manager) and production-only updates (flashed at the factory; example given: LNX 1.2.4-6). A later public update may merge a production-only version into its update path.

Firmware updates touch three components in this order (BO/5346164745):
1. The Secure Element OS: updated via an OSU (Operating System Updater) that transitions the SE to the next version, then the final OS is installed. DMK contains a multi-step OS update path computation (WXP/6943703128).
2. The MCU firmware (flashed conditionally via /mcu).
3. The Bootloader.

Each firmware update requires a matching OSU; the orchestration is done by Ledger Live / DMK.

---

## 4. Recover primer (what it is, threat model, what QA tests)

### 4.1 What it is

RECOVER (TrustServices/4282253487, released October 2023, previously called Protect) and RECOVER (CN/5925896228, owner Luc Hebrard, Senior PM, last updated April 2026):

> Ledger Recover is a paid service that allows Ledger device owners to backup and restore their seed securely by proving their identity.

The seed is split into three shards using Pedersen Verifiable Secret Sharing (PVSS) / Shamir Secret Sharing (SSS), each encrypted and sent to three independent Backup Providers: Ledger, Coincover and Escrowtech (INFRASD/4254564626 Trust - Recover Network Detailed Design). Restore requires an Identity Verification (IDV) step; Tessi is being replaced by IDnow (TrustServices/6531383435, [Recover] IDnow for Restore IDV - One-Pager Hub).

### 4.2 Architecture (abridged)

From TrustServices/4282253487 and TrustServices/6850248705:

- Recover on-device code lives in Ledger OS (SE-side). Repo: github.com/LedgerHQ/ledger-secure-os. White paper: github.com/LedgerHQ/recover-whitepaper.
- Ledger Live / Recover UI lives in Ledger Live (Wallet XP BU).
- Orchestrator / Stargate are the Trust-owned backend services that coordinate the shard flow and the IDV gate.
- Three Backup Providers (Ledger, Coincover, Escrowtech) each hold one encrypted shard. The aim is that no single provider (not even Ledger) can reconstruct the seed alone.
- POST /backups creates a session (Redis), then backup/prepare is published to the three Backup Providers (TrustServices/6850248705).

### 4.3 Threat model and security evolutions

From CN/6412271698 Ledger Recover Security Evolution: AI-Proofing Strategy:

> For Ledger Recover, which relies on Identity Verification (IDV) for backup retrieval, this threat is paramount. [...] We are evolving Ledger Recover security from a single-point-in-time identity check to a more holistic, multi-signal trust model.

Main threats called out: deepfakes / AI spoofing on the IDV step; single-point-in-time identity check weakness; collusion and leaks between providers; recovery by a non-owner.

Multi-user / V2 (CN/6747193449, CN/6863356000): adds a trusted-person assist flow (a Master creates the backup with the existing Recover flow; a trusted person can help recover while the master is alive or after).

### 4.4 Ledger Academy / help-center ground-truth links

The help-center / academy references are aggregated in BO/6148030569 Ledger Recover - Master, which lists: FAQs, payments, one-time security code, email verification troubleshooting, unsubscription flow, manual verification, cancellation, and the six-part Genesis of Ledger Recover blog series.

### 4.5 What QA tests

Not a single canonical Recover QA checklist page was surfaced in the research window. What is attested:

- Recover backup flow, restore flow and unsubscribe flow are test scopes (OS-74 Flex design release: Recover flows (backup / restore / unsubscribe); final wording, QA'd).
- Jailed device recovery edge case (BO/6147702976, linked from BO/6148030569).
- IDV failure flow testing: Verification Failed (BO/6147702958) and manual IDV workflow (BO/6147702967).
- Stop-Restore security flow (BO/6497763333).
- Payment/subscription flows via Chargebee (BO/6149865485, BO/6384386161).
- End-to-end Recover tested on Stax, Flex, Nano Gen5 and Classic Nano (TA/6696829023: Flex/Stax/Gen5 supported, Nano Classic (S+/X) will be supported in the future, Nano S is not supported).

See Gaps section: a dedicated Recover QA test plan page may exist but was not surfaced by the Rovo queries available.

---

## 5. Ledger Enterprise / Vault

### 5.1 What it is

LES Knowledge Hub (VAUL/overview):

> Ledger Enterprise Solution (LES) is the Business Unit focusing on the B2B products Ledger Enterprise & Ledger Multisig. [...] LES (Ledger Enterprise Services) is a horizontal BU, composed of many different teams and about 90 people.

[Arch] Ledger Vault Service Description (VAUL/2969534947):

> The Ledger Vault is a multi-authorization digital asset wallet management solution that enables clients to manage digital asset accounts, define custom governance rules on the reception or emission of digital asset transactions and user management, all the while keeping full control over their private keys.

Security is built on two pillars: Personal Security Devices (PSDs), the hardware devices used by Administrators / Operators to review and cryptographically sign all sensitive operations on a trusted display; and Hardware Security Modules (HSMs) that store the Master Seed, enforce governance rules, and encrypt client data with a Wrapping Key. HSM and PSDs communicate over a mutually authenticated encrypted channel.

Critical assets (VAUL/2969534947):
- Master Seed: from which all accounts and their keys are derived.
- Wrapping Key: encrypts the Master Seed and governance rules at rest.

### 5.2 Device support

Vault historically ran on Ledger Blue. Stax has become the PSD going forward (VAUL/4136861922 Stax for Ledger Enterprise: Overview; VG-20854 Test Plan - Stax: Allow clients to use Stax as a secure replacement for weBlue). The Ledger Enterprise Mobile App (VAUL/5751275617) lets reviewers track requests originating from a Ledger Enterprise workspace and relies on the Ledger Vault app installed on their Stax.

### 5.3 Surrounding infrastructure

HSM Services HLDs (INFRASD/4170809356) splits the HSM estate into:
- On-premise in Vierzon (Ledger factory): stateless, used for manufacturing Ledger devices and servicing Ledger Live.
- Vault HSM (standalone and cluster) and Ledger HSM: B2B Vault customers, on-premise, each customer data kept in a dedicated compartment.

HSM On-Prem operational responsibility is split between Ledger and the vendor (Thales) (VAUL/6645874730).

### 5.4 Does the QAA intern need to know about it?

Short answer: surface-level awareness, not deep. Rationale, grounded in what was read:

- The QAA Onboarding Checklist (QA/6647873614) lists repo clones for vault-e2e-tests and vault-public-api-tests, suggesting Vault test repositories are part of the QAA universe, but the QA space around QAA focuses on Ledger Live, Coin modules, Buy/Sell, Recover and the Coin Tester (QA/6637191329).
- Vault QA is a distinct practice with its own manager and its own 1Password vaults (Vault - Team Vault Quality), and incident classification uses a dedicated Vault Incident Classification page (TSQM/6915817566).
- Vault test plans (VG-20854) and Vault apps on Stax (VG-20055) run on a separate Jira project (VG = Vault / LES).

So: a QAA intern on the Consumer side should understand what Ledger Vault is conceptually (PSD + HSM + governance), recognise that VG tickets are a different product, and know that Vault flows on Stax run the Vault app (which is a separate embedded app), but is unlikely to execute Vault test plans unless explicitly staffed on them.

---

## 6. Ledger developer platform

The developer platform has four layers, all attested in the pages read:

### 6.1 Embedded SDKs (on-device side)

- C SDK: github.com/LedgerHQ/ledger-secure-sdk (the ex-ledger-secure-sdk / sdk-legacy). Quoted directly in [SDK] Introduction (WXP/4305813511) and in FW/5737481596.
- Rust SDK: github.com/LedgerHQ/ledger-device-rust-sdk (FW/5737481596).
- Both produce an app.elf + manifest consumed by the app installer (ledgerblue.loadApp for C apps, cargo-ledger for Rust) (FW/6679429130).
- The SDK is bundled with BOLOS primitives and NBGL (display) use-case modules. There is ongoing work to split pure-app code from shared-library code so that more lives in BOLOS itself (FW/6273138715, FW/6173196300).
- App deployment pipelines are documented in FW/3971186947 App deployment pipelines explained & how-to, including the HSM-based build, encryption and artifact production steps, and the interaction with the coin-apps repo used by Live QA / PTX for automated testing against latest app binaries.

### 6.2 Host-side SDK / DMK

- Device Management Kit (DMK): JS/TS SDK. From WXP/4251680783 [DXP] Device Management Kit and WXP/4305813511 [SDK] Introduction: DMK is the vision-level Ledger Device SDK whose vision is to enable developers to connect & interact with BOLOS applications directly, configure & update any Ledger devices on any platform.
- DMK is the replacement for the older host stack; OS Update migration to DMK (WXP/6943703128) shows how it computes multi-step OS update paths.

### 6.3 Wallet API (in-Live integration)

From WAL/4172578925 (Wallet API documentation Workshop), WALLETCO/4243522282 and ENGB2C/4037181769:

- Wallet API is how Live Apps talk to Ledger Live. Public docs: https://wallet.api.live.ledger.com/.
- v3 (react-client) is the hook-based version, inspired by wagmi.sh. It groups into read hooks (useBalance, ...) and action hooks (useSignTransaction, ...).
- The API surface is exposed in the Wallet API Tools Live App under the Discover tab for developers to probe.
- Under the hood it uses JSON-RPC between the Live App webview and Ledger Live; storage.get / storage.set can share state (with caveats per WALLETCO/4243522282).
- Live Apps live inside Ledger Live webviews (both LLD and LLM). Examples: WalletConnect as a Live App (WALLETCO/4243522282), Buy/Sell flows (PTX/3826090020; same app is typically both the Discover and the Buy entry point), WebPTXPlayer, Wallet Connect v2 Live App.

### 6.4 Coin Framework (Ledger Live side)

From CF/6064900341 (Coin Framework space), CF/6129025043, CF/6949896196 (ADR-025), CF/6029213697 (ADR-001 Staking API) and VAUL/5738037406:

- @ledgerhq/coin-framework is being split into @ledgerhq/coin-module-framework (per-coin integration framework) and @ledgerhq/coin-modules (actual per-coin code) (LIVE-24097).
- Each coin module exposes balances, tx history, staking (ADR-001), and is exposed over HTTP by Alpaca (ADR-025), a REST service owned by the Cloud Wallet team.
- Coin-Tester (QA/6637191329) is an automated testing framework developed by Ledger to validate blockchain coin integrations within Ledger Live. Source: github.com/LedgerHQ/ledger-live/tree/develop/libs/coin-tester.
- DMK is used by coin modules as the device signer (live-signer-* libraries mentioned in LIVE-24074).

### 6.5 Developer-facing portal

External developer docs land at developers.ledger.com, specifically /docs/device-app/getting-started is the reference for building BOLOS device apps (FW/5737481596 links to it explicitly).

---

## 7. Top 8 Confluence pages to cite as ground truth

These are the canonical pages a QAA onboarding guide should link to for this scope:

1. FW/5737481596 Ledger OS - Introduction: one-stop overview of the Ledger hardware + OS + security model + SE/MCU split; last modified Feb 2026.
   https://ledgerhq.atlassian.net/wiki/spaces/FW/pages/5737481596/Ledger+OS+-+Introduction
2. ENGB2C/3644882985 [Arch] Introduction to BOLOS: canonical BOLOS definition and module list.
   https://ledgerhq.atlassian.net/wiki/spaces/ENGB2C/pages/3644882985/Arch+Introduction+to+BOLOS
3. FW/3295281471 Secure Elements: what a SE is, why it matters, how Ledger uses them.
   https://ledgerhq.atlassian.net/wiki/spaces/FW/pages/3295281471/Secure+Elements
4. TH/6898319408 List of Ledger Product/Services: Tech Hub canonical product taxonomy.
   https://ledgerhq.atlassian.net/wiki/spaces/TH/pages/6898319408/List+of+Ledger+Product+Services
5. LEU/4566614017 Ledger Flex Product Presentation: the authoritative hardware comparison table for Nano S Plus / Nano X / Flex / Stax.
   https://ledgerhq.atlassian.net/wiki/spaces/LEU/pages/4566614017/Ledger+Flex+Product+Presentation
6. NANO/2869264439 Process for firmware releases: four-phase release process with stakeholders, RCs, progressive rollout.
   https://ledgerhq.atlassian.net/wiki/spaces/NANO/pages/2869264439/Process+for+firmware+releases
7. CN/5925896228 RECOVER (current PM master doc, April 2026): current scope of Ledger Recover.
   https://ledgerhq.atlassian.net/wiki/spaces/CN/pages/5925896228/RECOVER
8. VAUL/2969534947 [Arch] Ledger Vault Service Description: PSD+HSM architecture, critical assets (Master Seed, Wrapping Key), admin model.
   https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/2969534947/Arch+Ledger+Vault+Service+Description

---

## 8. Gaps / uncertainties

- Apex device: mentioned in Mobile Device Onboarding (Engagement/7011795014) as being on the Sync Onboarding path alongside Stax/Flex, but no product presentation page surfaced. Form factor, SE/MCU, status unknown within this research window.
- Ledger Nano Gen5: confirmed as a current manufacturing target with STM32WB35 + ST33K1M5C (FW/5737481596, HTS/6233456749), but no full product-presentation page surfaced. Release status (consumer / internal) unclear.
- Dedicated Recover QA test plan: not surfaced. The help-center master (BO/6148030569) and the Recover arch pages cover flows and the Archive - Entry for Ledger Recover (CN/6670647373) covers Braze tests, but a single canonical here is the Recover QA matrix page was not found via Rovo. Might live under the QA space or the TrustServices space but was not returned by the queries available.
- CQL was not accessible to this agent: only Rovo search and getConfluencePage were authorised. Results skew toward Rovo ranking. A second pass with CQL would be able to do targeted space-scoped queries (Tech Hub TH, Ledger Live LLI/WXP/PTX/etc., Coin Framework CF).
- Space keys verified via page URLs (not via getConfluenceSpaces, which was denied): FW (Team: Embedded Software), TA (Team: Architecture), TH (Tech Hub), LLI (Ledger Live), CF (Coin Framework), WAL (Wallet API), WXP, WALLETCO, ENGB2C (B2C - Engineering), CN, LEU (Ledger Europa), VAUL (LES Knowledge Hub), TrustServices, NANO (Product: Ledger Nano), QA, HTS, PTX, BO, INFRASD, TA, Engagement. The exact mapping of Ledger Live to LLI vs WXP vs WALLETCO is not clean; these appear to be partially overlapping product-org spaces.
- Developer platform public site: developers.ledger.com is referenced but was not fetched as a Confluence page (out of scope).
- Ledger Recover technical white paper: public at github.com/LedgerHQ/recover-whitepaper (BO/6148030569). Not fetched here but a QAA intern should read it to ground their Recover mental model.

---

## 9. Raw Confluence links (all pages referenced, by ID)

Hardware and BOLOS / OS (FW):
- 5737481596 Ledger OS - Introduction: https://ledgerhq.atlassian.net/wiki/spaces/FW/pages/5737481596/Ledger+OS+-+Introduction
- 3295281471 Secure Elements: https://ledgerhq.atlassian.net/wiki/spaces/FW/pages/3295281471/Secure+Elements
- 5732106306 Ledger OS - Connectivity: https://ledgerhq.atlassian.net/wiki/spaces/FW/pages/5732106306/Ledger+OS+-+Connectivity
- 4553539605 NFC Stax Europa: https://ledgerhq.atlassian.net/wiki/spaces/FW/pages/4553539605/NFC+Stax+Europa
- 6273138715 Moving shared library code to Bolos: https://ledgerhq.atlassian.net/wiki/spaces/FW/pages/6273138715/Moving+shared+library+code+to+Bolos
- 6173196300 Bolos SDK Technical Debt reduction: https://ledgerhq.atlassian.net/wiki/spaces/FW/pages/6173196300/Bolos+SDK+-+Technical+Debt+reduction
- 6679429130 Application parameters: https://ledgerhq.atlassian.net/wiki/spaces/FW/pages/6679429130/WIP+Application+parameters
- 4809490461 Pi Programmers: https://ledgerhq.atlassian.net/wiki/spaces/FW/pages/4809490461/Pi+Programmers
- 3971186947 App deployment pipelines explained and how-to: https://ledgerhq.atlassian.net/wiki/spaces/FW/pages/3971186947/App+deployment+pipelines+explained+how-to
- 2232090878 Firmware Update Testing: https://ledgerhq.atlassian.net/wiki/spaces/FW/pages/2232090878/Firmware+Update+Testing

BOLOS architecture (TA, ENGB2C):
- 3644882985 Intro to BOLOS: https://ledgerhq.atlassian.net/wiki/spaces/ENGB2C/pages/3644882985/Arch+Introduction+to+BOLOS
- 3478093827 BOLOS application and SDK: https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/3478093827/ARCH+BOLOS+application+SDK
- 3494740112 BOLOS Tasks: https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/3494740112/ARCH+BOLOS+Tasks
- 3684630561 LL Coin integration framework: https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/3684630561/ARCH+LL+Coin+integration+framework
- 5999099908 Integrating Ledger-Sync in BOLOS: https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/5999099908/ARCH+Lessons+Learned+-+Integrating+Ledger-Sync+Application+features+in+BOLOS
- 6597115914 Embedded applications localization: https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/6597115914/ARCH+DRAFT+Embedded+applications+localization
- 6696829023 Tiered Authentication and Authorization: https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/6696829023/ADR+Tiered+Authentication+and+Authorization+for+First-Party+and+Third-Party+Clients+Interacting+with+Ledger+Backend+Services

Hardware test / production (HTS):
- 6095274089 Diagnostic application: https://ledgerhq.atlassian.net/wiki/spaces/HTS/pages/6095274089/Diagnostic+application
- 6233456749 BGIv2 Specification: https://ledgerhq.atlassian.net/wiki/spaces/HTS/pages/6233456749/BGIv2+Specification
- 6157598754 How to order Ledger devices for internal needs: https://ledgerhq.atlassian.net/wiki/spaces/HTS/pages/6157598754/How+to+order+Ledger+devices+for+internal+needs
- 6072041485 SE firmware update procedure: https://ledgerhq.atlassian.net/wiki/spaces/HTS/pages/6072041485/SE+firmware+update+procedure

Device product pages:
- 4566614017 Ledger Flex Product Presentation: https://ledgerhq.atlassian.net/wiki/spaces/LEU/pages/4566614017/Ledger+Flex+Product+Presentation
- 2869264439 Process for firmware releases: https://ledgerhq.atlassian.net/wiki/spaces/NANO/pages/2869264439/Process+for+firmware+releases

Ledger Live / Wallet API / Live Apps:
- 5644255440 Ledger Wallet Ledger Live: https://ledgerhq.atlassian.net/wiki/spaces/LLI/pages/5644255440/Ledger+Wallet+Ledger+Live
- 3545759821 How to LQA Ledger Wallet: https://ledgerhq.atlassian.net/wiki/spaces/LLI/pages/3545759821/How+to+LQA+Ledger+Wallet+Ledger+Live
- 4178051212 LWD LWM translated string in Smartling: https://ledgerhq.atlassian.net/wiki/spaces/LLI/pages/4178051212/LWD+LWM+-+How+to+update+a+translated+string+in+Smartling
- 4251680783 DXP Device Management Kit: https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/4251680783/DXP+Device+Management+Kit
- 4305813511 SDK Introduction: https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/4305813511/SDK+Introduction
- 6943703128 OS Update migration to DMK: https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/6943703128/OS+Update+migration+to+DMK+compute+multi-step+OS+update+path
- 4172578925 Wallet API documentation Workshop: https://ledgerhq.atlassian.net/wiki/spaces/WAL/pages/4172578925/Wallet+API+documentation+Workshop
- 4243522282 Wallet Connect as a Live App: https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/4243522282/Wallet+Connect+as+a+Live+App
- 4037181769 Live Wallet API Weekly Update: https://ledgerhq.atlassian.net/wiki/spaces/ENGB2C/pages/4037181769/Live+-+Wallet+API+Weekly+Update
- 3826090020 Buy Sell Documentation: https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/3826090020/Buy+Sell+-+Documentation
- 5052825616 Braze in Live Apps: https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/5052825616/Braze+in+Live+Apps
- 4177920547 WALLET API migration issue: https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/4177920547/WALLET+API+migration+issue
- 7011795014 Mobile Device Onboarding and Genuine Check Architecture: https://ledgerhq.atlassian.net/wiki/spaces/Engagement/pages/7011795014/Mobile+Device+Onboarding+and+Genuine+Check+Architecture

Coin Framework:
- 6064900341 Coin Framework space overview: https://ledgerhq.atlassian.net/wiki/spaces/CF/overview
- 6129025043 Coin Modules Developments Mapping: https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/6129025043/Coin+Framework+Coin+Modules+Developments+Mapping
- 6949896196 Coin modules ADR-025 migration to Alpaca: https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/6949896196/Coin+modules+-+ADR-025+-+Coin+modules+migration+to+Alpaca
- 6925549613 Coin modules ADR-022 Extensibility: https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/6925549613/Coin+modules+-+ADR-022+-+Extensibility+Pattern+for+Cross-Cutting+Concerns+in+AlpacaApi
- 6029213697 Coin modules ADR-001 Staking API: https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/6029213697/Coin+modules+-+ADR-001+-+Staking+API+design
- 5855281381 Coin Framework Weekly Refinement: https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/5855281381/Coin+Framework+Weekly+Refinement
- 5738037406 Engineering Vault Coin Integration Playbook: https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/5738037406/Engineering+-+Vault+Coin+Integration+Playbook
- 5792792577 Vault Coin Integration Playbook SUI: https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/5792792577/Vault+Coin+Integration+Playbook+-+SUI
- 6637191329 Coin-Tester: https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6637191329/Coin-Tester

Ledger Recover:
- 4282253487 RECOVER Trust Services 2023: https://ledgerhq.atlassian.net/wiki/spaces/TrustServices/pages/4282253487/RECOVER
- 5925896228 RECOVER current PM master April 2026: https://ledgerhq.atlassian.net/wiki/spaces/CN/pages/5925896228/RECOVER
- 6148030569 Ledger Recover Master BO: https://ledgerhq.atlassian.net/wiki/spaces/BO/pages/6148030569/Ledger+Recover+-+Master
- 6412271698 Ledger Recover Security Evolution AI-Proofing Strategy: https://ledgerhq.atlassian.net/wiki/spaces/CN/pages/6412271698/Ledger+Recover+Security+Evolution+AI-Proofing+Strategy
- 6747193449 Recover Multi-user: https://ledgerhq.atlassian.net/wiki/spaces/CN/pages/6747193449/Recover+-+Multi-user
- 6863356000 MAIN Multi-user MVP: https://ledgerhq.atlassian.net/wiki/spaces/CN/pages/6863356000/MAIN+-+Multi-user+MVP
- 6531383435 Recover IDnow for Restore IDV One-Pager Hub: https://ledgerhq.atlassian.net/wiki/spaces/TrustServices/pages/6531383435/Recover+IDnow+for+Restore+IDV+-+One-Pager+Hub
- 6850248705 Recover Arch Event Model DDD Flows and Domain Events: https://ledgerhq.atlassian.net/wiki/spaces/TrustServices/pages/6850248705/Recover+Arch+Event+Model+DDD+Flows+Domain+Events
- 4254564626 Trust Recover Network Detailed Design: https://ledgerhq.atlassian.net/wiki/spaces/INFRASD/pages/4254564626/Trust+-+Recover+Network+Detailed+Design
- 6670647373 Archive Entry for Ledger Recover: https://ledgerhq.atlassian.net/wiki/spaces/CN/pages/6670647373/Archive+-+Entry+for+Ledger+Recover

Ledger Enterprise / Vault / LES:
- VAUL overview LES Knowledge Hub: https://ledgerhq.atlassian.net/wiki/spaces/VAUL/overview
- 2969534947 Arch Ledger Vault Service Description: https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/2969534947/Arch+Ledger+Vault+Service+Description
- 4136861922 Stax for Ledger Enterprise Overview: https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/4136861922/Stax+for+Ledger+Enterprise+Overview
- 5751275617 Help Center Ledger Enterprise mobile app: https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/5751275617/Help+Center+-+Ledger+Enterprise+mobile+application
- 6169460737 Vault Staking: https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/6169460737/Vault+Staking
- 3559653377 B2C Lead from Vault B2B: https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/3559653377/B2C+Lead+from+Vault+B2B
- 4170809356 HSM Services HLDs: https://ledgerhq.atlassian.net/wiki/spaces/INFRASD/pages/4170809356/HSM+Services+HLDs
- 6645874730 HSM On-Prem Operational Responsibility: https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/6645874730/HSM+On-Prem+Operational+Responsibility
- 6915817566 Vault Incident Classification: https://ledgerhq.atlassian.net/wiki/spaces/TSQM/pages/6915817566/Vault+Incident+Classification+Origin+Affected+Service
- 3844800519 LES Releases Process: https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/3844800519/LES+Releases+-+Process
- 5470159058 Template SELF Release Process: https://ledgerhq.atlassian.net/wiki/spaces/FET/pages/5470159058/Template+SELF+Release+Process

Firmware lifecycle / update:
- 5346164745 Firmware update related issues: https://ledgerhq.atlassian.net/wiki/spaces/BO/pages/5346164745/Firmware+update+related+issues+Bootloader+Update+Booting+Bricked+device
- 6783172610 Canton Firmware checks for public release: https://ledgerhq.atlassian.net/wiki/spaces/BI/pages/6783172610/Canton+-+Firmware+checks+for+public+release

QA onboarding and adjacent:
- 6647873614 QAA Onboarding Checklist: https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6647873614/QAA+Onboarding+Checklist
- 6896255151 Feature Flags and Firebase Config Models: https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6896255151/Feature+Flags+and+Firebase+Config+Models
- 6898843819 QA Feature Development and Quality Lifecycle: https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6898843819/QA+-+Feature+Development+Quality+Lifecycle

Product catalog / Tech Hub:
- 6898319408 List of Ledger Product Services Tech Hub: https://ledgerhq.atlassian.net/wiki/spaces/TH/pages/6898319408/List+of+Ledger+Product+Services
- 5046665289 Research Why do people not upgrade from Nano to touch devices: https://ledgerhq.atlassian.net/wiki/spaces/LHP/pages/5046665289/Research+-+Why+do+people+not+upgrade+from+Nano+to+touch+screen+devices+Stax+Flex
- 5736103938 Swollen Leaking Battery Flex Stax: https://ledgerhq.atlassian.net/wiki/spaces/BO/pages/5736103938/Swollen+Leaking+Battery+Flex+Stax
