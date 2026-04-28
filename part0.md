# PART 0 — WELCOME TO LEDGER

<div class="chapter-intro">
Before you touch a test file, a Playwright config, or a Speculos emulator, you need a mental model of the thing you are testing. Part 0 gives you that model: who Ledger is, what it ships, what makes those products different from every other company selling "a wallet", and how the two chips inside a Ledger device actually collaborate. Everything that follows in Parts 1 through 8 will make sense once this foundation is in place.
</div>

---

## The Ledger Ecosystem & Product Suite

<div class="chapter-intro">
Ledger is not a single product. It is a hardware line, a desktop app, a mobile app, a marketplace of third-party Live Apps, a developer platform, a B2B custody solution, and a paid seed-recovery service — all orbiting one core promise: your private keys never leave a certified Secure Element. This chapter maps that ecosystem end-to-end so that when a Jira ticket lands on your board, you know which planet it lives on.
</div>

### 0.1.1 What Ledger Does

Ledger was founded in Paris in 2014 and ships hardware wallets — small tamper-resistant devices that store the private keys for cryptocurrency accounts. The company's founding thesis, repeated across every product page, every keynote, and every help-center article, is the phrase:

> **"Not your keys, not your coins."**

Translated for a TypeScript engineer who has never touched crypto: if you hold your funds on Coinbase, Binance, or any other exchange, you do not actually own them. You own a database row that says the exchange owes you X bitcoin. The exchange owns the actual keys. If the exchange is hacked, goes bankrupt, or freezes your account, your database row becomes worthless.

A Ledger hardware wallet reverses this. The private keys are generated on the device, stored on the device, and used for signing on the device. They never touch your laptop, your phone, your router, or Ledger's servers. Every transaction you sign requires you to physically press buttons (or tap the screen on Stax/Flex) to approve it. A remote attacker who fully compromises your computer still cannot spend your funds — they would need the physical device and your PIN.

The technical moat here is not software. It is **silicon**. Every Ledger device is built around a Secure Element (SE) — a certified chip from the same family as the ones in your passport, your SIM card, and your bank's chip-and-PIN card. Ledger's current consumer devices hold **Common Criteria EAL5+ or EAL6+** certifications (per R2 research). Competitors using general-purpose microcontrollers cannot make that claim. Chapter 0.2 goes deep on what this actually buys you.

> **Note:** As a QA Automation engineer you will rarely, if ever, test the silicon itself. The Firmware team, the Donjon (Ledger's internal security research team), and third-party certification labs handle that. Your job is to guard the software-to-hardware interface — the place where Ledger Live, a Live App, or a coin module talks to the device. More on this in section 0.1.6.

#### Why self-custody is a QA concern, not just a marketing concern

Self-custody is not an abstract value proposition you hear once at an all-hands and forget. It shapes every line of code in Ledger Live and, by extension, every test you write. The product's correctness bar is not "does the UI look right?". It is "does the user understand, and visibly approve, exactly what they are signing?". A failed send-to-contact test is a minor annoyance. A flow where the amount displayed on the device does not match the amount in the Ledger Live confirmation screen is a **clear-signing defect** and, potentially, a security incident.

This asymmetry — that some bugs are merely bugs and some bugs are trust-destroying — is why QAA tests so much of the UI prominently inspect the **device screen** (via Speculos screenshot assertions) in addition to the desktop/mobile UI. The device's secure display is the ground truth. If the host says "send 1 BTC" and the device also says "send 1 BTC", the user's intent is faithfully represented. If the host says "send 1 BTC" and the device says "send 100 BTC", something in the host, the coin module, or the embedded app is lying, and the device is the last line of defence.

You will see this principle encoded throughout the codebase:

- Transaction-signing tests assert on both the Ledger Live modal and the Speculos screen.
- Address-verification flows always end with the user reading the address off the device, not off the host.
- Swap tests assert on the device's display of the "you will receive" amount, because the device has to independently recompute and display it.

Internalise this early: **QAA is in the trust-preservation business**. Everything else — speed of test suite, flakiness budget, fixture design — serves that goal.

#### A short history that will come up in coffee-room conversations

A few dates and inflection points that are referenced casually across the company and that, if you do not know them, will make you feel adrift during your first all-hands:

- **2014** — Ledger is founded in Paris. The Nano S (launched 2016) is the product that puts the company on the map.
- **2019** — Nano X launches, adding Bluetooth and a much larger app capacity. The first Ledger device certified CC EAL5+.
- **2020** — A well-known **customer-database leak** exposes email addresses and physical addresses of a subset of customers. Private keys were never at risk (they live on-device, never on servers), but the incident fuels a durable public mistrust around "Ledger and data". It is also why you will see Ledger being unusually careful about collecting personally identifiable information in product flows.
- **2022** — Nano S Plus launches at CC EAL6+. The "classic" line gets a significant hardware refresh.
- **2023** — Ledger Stax (first touch-and-curved-E-Ink device) ships after extended development. **Ledger Recover** launches in October 2023 and triggers a high-visibility public debate about the service's opt-in seed-backup model.
- **2024** — **Ledger Flex** (codename Europa) ships as a flat-screen touch device, slotting below Stax in price. The Sync Onboarding flow matures.
- **2026** (current) — Nano Gen5 is in manufacturing-readiness per `HTS/6233456749`. Touch-line becomes the default recommendation for new users; classic line continues for power users and price-sensitive buyers.

You do not need to memorise this timeline. You do need to recognise "Recover launch", "Stax launch", and "the 2020 data leak" as three events that have shaped product direction, org structure, and the way certain user populations interpret QA findings.

### 0.1.2 Hardware Line-Up

Ledger ships several physical devices. A QAA intern needs to recognise each one on sight, know its form factor, and know whether it is a current or legacy product, because test matrices and Jira tickets will reference them constantly.

The canonical reference is the Confluence page **FW/5737481596 — "Ledger OS - Introduction"** (last modified Feb 2026). The comparison table below is reconciled with **LEU/4566614017 — "Ledger Flex Product Presentation"** and cross-checked against **HTS/6233456749 — "BGIv2 Specification"**.

| Device              | Form factor                   | SE chip (certification) | MCU                    | Screen                              | Status              |
|---------------------|-------------------------------|-------------------------|------------------------|-------------------------------------|---------------------|
| Ledger Nano S       | USB-A stick, 2 buttons        | ST31H320                | STM32F042              | OLED 128x64 monochrome              | Legacy              |
| Ledger Blue         | Early large touch device      | ST31G480                | STM32L476              | Touch (historical)                  | Legacy              |
| Ledger Nano X       | USB-C, BLE, 2 buttons         | ST33J2M0 (EAL5+)        | STM32WB55 / STM32WB35  | OLED 128x64 monochrome              | Current (classic)   |
| Ledger Nano S Plus  | USB-C, 2 buttons              | ST33K1M5C (EAL6+)       | STM32F042              | OLED 128x64 monochrome              | Current (classic)   |
| Ledger Stax         | Touch, curved flexible E-Ink  | ST33K1M5C (EAL6+)       | STM32WB35              | E-Ink 400x672, 16 greys, curved     | Current (touch)     |
| Ledger Flex         | Touch, flat E-Ink             | ST33K1M5 (EAL6+)        | STM32WB35              | E-Ink 600x480, 2.84 in, 271 ppi     | Current (touch)     |
| Ledger Nano Gen5    | Classic form factor, next-gen | ST33K1M5C               | STM32WB35              | (not yet documented publicly)       | Upcoming (per R2)   |

A few things worth internalising:

- **"Classic" line vs "Touch" line.** The classic line (Nano S, Nano X, Nano S Plus, Nano Gen5) uses physical buttons and a small monochrome OLED. The touch line (Stax, Flex, and the historical Blue) uses E-Ink and a capacitive touch panel. Many test flows branch on this distinction — a touch device uses the **Sync Onboarding** flow, a classic device uses a button-driven onboarding.
- **"Europa" is the internal codename for Ledger Flex.** You will see `europa` sprinkled through repo names, Speculos model flags, and CI job matrices. It is not a separate product. (per R2 research, confirmed in `FW/4553539605 — NFC Stax Europa`.)
- **Nano S is legacy.** It is not supported in Ledger Multisig (see VG-25622) and is explicitly excluded from Ledger Recover. If a test spec says "runs on Nano S" without a very good reason, push back.
- **Stax is curved and stackable.** Flex is flat and does not stack. Both support USB-C, BLE 5.2 and NFC. Stax additionally supports Qi wireless charging. (per R2 research, `LEU/4566614017`.)
- **Secure Element memory matters.** On Flex, roughly 1.15 MB of the 1.5 MB SE flash is available for user apps, language packs, and pictures. This is why coin-app installers will sometimes refuse to install a new app — the SE is full and Ledger Live has to negotiate evictions.

> **Gap:** The codename **Apex** appears in `Engagement/7011795014 — Mobile Device Onboarding` as a future device on the Sync Onboarding path alongside Stax and Flex, but no product-presentation page exists in Confluence at the time of writing. Form factor, SE/MCU details, and ship status are all unknown (per R2 research). If a ticket mentions Apex, ask your tech lead before assuming anything.

### 0.1.3 The Software Stack

The hardware is only half the story. The other half is the software that humans actually use, and this is the half QA Automation spends most of its time inside. The canonical taxonomy is **TH/6898319408 — "List of Ledger Product/Services"**.

**Ledger Live Desktop (LLD)**
The companion desktop application. It is an **Electron** app (Chromium + Node.js in a native wrapper) shipped on Windows, macOS, and Linux. It handles device pairing, app installation, account management, portfolio view, send/receive, swap, buy/sell, staking, NFTs, and the in-app "Discover" catalogue of Live Apps. The QAA Desktop E2E suite (Part 4 of this guide) drives LLD via Playwright's Electron support.

**Ledger Live Mobile (LLM)**
The mobile companion. Built in **React Native**, shipped on iOS and Android. Feature parity with LLD is the north star, but the two apps have separate codebases, separate release trains, and separate QA tracks. Mobile E2E uses Detox (covered in Part 5).

**Live Apps**
Third-party and first-party web applications that run **inside a webview** embedded in Ledger Live. They talk to Ledger Live through the **Wallet API** (more on this below). WalletConnect, the Buy/Sell flows (PTX), the Wallet API Tools playground, and many partner apps (DeFi aggregators, exchanges, lending protocols) are all Live Apps. From a QA perspective a Live App is a webview you point at a URL — but with a privileged JSON-RPC bridge to the host.

**Wallet API**
A JS library that Live Apps import to call Ledger Live. Current version is **v3 (react-client)**, a hook-based API explicitly inspired by wagmi.sh. It splits into read hooks (`useBalance`, `useAccounts`, ...) and action hooks (`useSignTransaction`, ...). Under the hood it is JSON-RPC between the webview and the host. Public docs live at `https://wallet.api.live.ledger.com/`.

**Device Management Kit (DMK)**
The JS/TS SDK that any host application (including Ledger Live itself) uses to connect to a device, install/uninstall apps, and perform OS updates. DMK owns the non-trivial logic of computing a **multi-step OS update path** when a user is several firmware versions behind (per R2 research, `WXP/6943703128`). When QA tests "install the Bitcoin app from the manager", the action is routed through DMK.

**Coin Framework + Coin Modules**
The TypeScript libraries inside Ledger Live that know per-blockchain logic: how to compute balances, fetch transaction history, build a send transaction, parse a staking reward, and so on. Being migrated from a monolithic `@ledgerhq/coin-framework` package into `@ledgerhq/coin-module-framework` (the framework) and `@ledgerhq/coin-modules` (the per-coin code). Alpaca is the REST backend that exposes coin modules over HTTP (ADR-025).

**Embedded coin apps**
Native C or Rust binaries that run **on the Secure Element**. Bitcoin App, Ethereum App, Solana App, and so on. These are signed BOLOS applications. The host side (Coin Modules) and the device side (Embedded coin app) are two halves of the same coin integration. Chapter 0.2 covers the embedded side.

**Coin-Tester**
An automated testing framework specifically for coin integrations inside Ledger Live. Source lives at `github.com/LedgerHQ/ledger-live/tree/develop/libs/coin-tester` (per R2 research, `QA/6637191329`). Coin-Tester is where a new blockchain's integration is validated before being exposed to the full Ledger Live test matrix. Parts 5 and 6 of this guide cover when and why you touch it.

**Alpaca**
A REST service that exposes the coin modules over HTTP, owned by the Cloud Wallet team. Introduced by ADR-025 (per R2 research, `CF/6949896196`). Not every coin module is on Alpaca yet — the migration is ongoing. When you see a `@ledgerhq/live-network`-shaped API call in a test, it may be going through Alpaca.

**Ledger Recover**
A paid, opt-in service released October 2023. Splits your seed into three encrypted shards using Pedersen / Shamir Secret Sharing, distributes them across three independent Backup Providers (Ledger, Coincover, Escrowtech), and requires Identity Verification (IDV) to restore. Out of scope for most QAA flows, but you will see Recover-related test IDs in the test matrix for Flex / Stax / Nano Gen5 / Nano S Plus / Nano X. Nano S is explicitly not supported.

**Ledger Enterprise / Vault / Multisig**
Ledger's **B2B product line**. Ledger Vault is a multi-authorization custody solution built around HSMs (Hardware Security Modules) and PSDs (Personal Security Devices — historically Blue, now Stax). Ledger Multisig is a web app for Safe-compatible multisig clear-signing. These products live under the **LES business unit**, run on separate Jira projects (VG for Vault), and have a separate QA team.

> **Note (for the QAA intern):** Vault / Enterprise / Multisig are **out of scope** unless you are explicitly staffed on them. Recognise the names, understand that VG tickets are not your tickets by default, and move on. Your default universe is Ledger Live Desktop + Ledger Live Mobile + their dependencies (Wallet API, Coin Framework, DMK, embedded coin apps via Speculos).

#### Ownership map at a glance

Because the product tree is wide, knowing **which team owns what** is often more useful than knowing the product name. Use this map to route questions:

| Area                                   | Team / owner                        | Jira project(s)   |
|----------------------------------------|-------------------------------------|-------------------|
| Ledger Live Desktop + Mobile           | Ledger Wallet / Live org            | LIVE, LLD, LLM    |
| Wallet API + Live Apps platform        | Wallet API / Live Hub team          | WAL, WALLETCO     |
| DMK / device integration               | WXP team                            | WXP, LIVE         |
| Buy/Sell + partner Live Apps           | PTX team                            | PTX               |
| Coin Framework + per-coin modules      | Coin Integration / Cloud Wallet     | CF, LIVE          |
| Embedded coin apps (on-device)         | Coin Integration + coin-specific    | Varies per coin   |
| BOLOS / Ledger OS                      | Embedded / Firmware                 | FW                |
| Ledger Recover                         | Trust Services / Consumer Services  | CN, TrustServices |
| Ledger Vault + Multisig                | LES (Ledger Enterprise Services)    | VG, VAUL          |
| Security research (offensive)          | Donjon                              | (internal)        |
| HSM / factory infra                    | Infra / HTS                         | INFRASD, HTS      |

When a ticket lands on your board and you cannot tell which team really owns the underlying defect, this table + the Jira project key is usually enough to route it correctly. "This bug is in the buy flow, the Jira project is PTX, I should ping the PTX channel before I start." That single sentence saves hours of mis-triage.

#### Release trains

A subtle but important organisational fact: **each product above has its own release cadence**. Ledger Live Desktop ships roughly every two weeks; Ledger Live Mobile slightly slower because of App Store review; firmware ships on its own multi-week cycle per device model; coin apps ship per-coin, often triggered by an external protocol upgrade (a hard fork on Ethereum, a Cardano era change, a Bitcoin soft-fork activation).

These cadences are **not synchronised**. A bug fix that requires both a desktop change and a firmware change cannot land in a single release — it has to be coordinated across two independent release trains. QAA plans test campaigns around the slowest train on the critical path.

### 0.1.4 Positioning vs Competitors

Knowing how Ledger positions itself against competitors helps you interpret ticket priorities, QA severities, and customer-support escalations.

| Vendor                  | Security chip approach                                 | Certification                    | Closed vs Open firmware          |
|-------------------------|--------------------------------------------------------|----------------------------------|----------------------------------|
| **Ledger**              | Certified Secure Element (ST33/ST31)                   | CC EAL5+ / EAL6+ + ANSSI CSPN    | SE OS closed, apps mostly open   |
| Trezor                  | General-purpose microcontroller (older models) / EAL6+ SE (Safe line) | Mixed; older Trezor One/T have no SE certification | Fully open source                |
| Coldcard                | Secure Element (ATECC608A or similar)                  | No CC certification at SE level  | Fully open source                |
| GridPlus Lattice1       | Secure Element (Microchip)                             | Some CC evaluation (subset)      | Partially open                   |

The core differentiator that appears in every Ledger security pitch: **"we certify at EAL5+ / EAL6+"**. EAL (Evaluation Assurance Level) is a Common Criteria rating; 5+ and 6+ are the tiers used in banking smartcards and government identity documents. Competitors either use non-certified chips or ship a less-certified tier.

The classic counter-argument from the open-source community is that Ledger's SE OS is closed-source and therefore unauditable. Ledger's response is that the SE firmware is reviewed under the CC certification process by independent labs, and that the *apps* that run on the SE (Bitcoin App, Ethereum App, etc.) are open-source and inspectable on GitHub. This is a real tension and you will see it show up in Reddit / Twitter / customer-support threads; know it exists, don't pretend it doesn't.

> **Gotcha:** As a QAA engineer, you will occasionally get bug reports that frame a product decision as a security bug ("why can I not export my seed in plaintext?" — that is the whole point; the seed cannot leave the SE). Before triaging a report as a defect, make sure you are not looking at a *designed* property of the system. The golden rule: if a proposed fix would let a secret leave the SE in cleartext, the fix is wrong and the report is either a misunderstanding or a social-engineering attempt.

### 0.1.5 How Hardware and Software Talk

The physical transport stack between a Ledger device and Ledger Live is deliberately narrow:

- **USB HID** is the primary wire protocol for all current devices. HID (Human Interface Device) is normally used by keyboards and mice; Ledger reuses it because it crosses OS permission boundaries without requiring a driver install on Windows/macOS/Linux.
- **Bluetooth Low Energy (BLE)** is supported on Nano X, Stax, and Flex. Used by Ledger Live Mobile and, optionally, by Ledger Live Desktop.
- **NFC** is a newer addition, used during the Sync Onboarding flow on Stax and Flex.
- **APDUs** (Application Protocol Data Units) are the binary packets that flow over whichever physical layer is in use. Every "sign this transaction" or "give me your public key" call eventually becomes an APDU.

QA does not drive HID directly. Instead, we use **Speculos** — an official Ledger emulator that simulates the SE, the MCU, the screen, and the APDU transport entirely in software. Speculos is how we run device tests in CI (where no physical device exists) and how we script device-side interactions from a test file. Part 3 Chapter 3.3 goes deep on Speculos; for now, park the word and remember it as "the emulator".

A concrete mental model of a single signing flow, end to end:

1. A user clicks "Send 0.1 BTC" in Ledger Live Desktop.
2. The Bitcoin Coin Module (host side, TypeScript) builds an unsigned PSBT (Partially Signed Bitcoin Transaction).
3. DMK opens a communication channel to the connected device (HID, BLE, or Speculos).
4. DMK serialises the PSBT into a sequence of **APDUs** and sends them to the Bitcoin embedded app running on the SE.
5. The embedded Bitcoin app parses the APDUs, reconstructs the transaction, displays every output address and amount on the **secure display** (SE-driven), and waits for user approval.
6. The user presses "approve" (or taps on Stax/Flex). The SE derives the signing key from the master seed, produces a signature, and returns it as an APDU.
7. DMK collects the signed PSBT, passes it back to the coin module, which broadcasts via the Bitcoin node provider.

Every arrow in that sequence is a thing QAA tests. The Speculos emulator stands in for steps 5 and 6 in CI. Your Playwright test drives steps 1, 2, and 7 through the desktop UI; your Speculos harness asserts on step 5 via screen captures and simulates step 6 via scripted button presses.

### 0.1.6 Where QAA Fits

Zoom out. The stack looks roughly like this:

```
  ┌─────────────────────────────────────────────────────────┐
  │  Ledger Live Desktop (Electron)    [Playwright tests]   │  <-- QAA Desktop
  │  Ledger Live Mobile (React Native) [Detox / Maestro]    │  <-- QAA Mobile
  │  Live Apps (webviews)                                   │
  ├─────────────────────────────────────────────────────────┤
  │  Wallet API  │  Coin Framework  │  DMK                  │  <-- QAA integration
  ├─────────────────────────────────────────────────────────┤
  │  APDU transport (HID / BLE / NFC / Speculos)            │  <-- QAA contract layer
  ├─────────────────────────────────────────────────────────┤
  │  MCU firmware                                           │  <-- Firmware team
  │  BOLOS (SE firmware)                                    │  <-- Firmware team / Donjon
  │  Embedded coin apps (signed BOLOS binaries)             │  <-- Coin Integration
  │  Secure Element silicon                                 │  <-- ST / Foundry
  └─────────────────────────────────────────────────────────┘
```

The **top three boxes are the QAA Automation universe**. Your tests drive the Electron app or the React Native app, click through flows, and assert on UI state. When a flow involves a device, your test talks to a Speculos instance over the same APDU channel a real device would use. The **bottom three boxes are someone else's problem** — you are not writing tests against BOLOS internals, you are not fuzzing the SE, you are not reviewing ST33 datasheets. You are verifying that the software-to-hardware interface behaves correctly from a user-observable perspective.

The practical consequence: a bug that reproduces only on a physical device but not on Speculos is still a bug in *our* code (the host, the embedded app, or their contract) 95% of the time. A bug that only reproduces on specific SE silicon is vanishingly rare and gets escalated immediately to Firmware.

> **Gotcha:** Do not spend hours debugging "why does this work on Speculos and fail on a real Nano X?" without first filing the observation with your tech lead. Nine times out of ten the answer is a timing assumption baked into the test — real hardware is slower than Speculos on some operations and faster on others.

#### Resources — Chapter 0.1

- **FW/5737481596** — Ledger OS - Introduction (one-stop SE/MCU overview). `https://ledgerhq.atlassian.net/wiki/spaces/FW/pages/5737481596`
- **TH/6898319408** — List of Ledger Product/Services (canonical product taxonomy). `https://ledgerhq.atlassian.net/wiki/spaces/TH/pages/6898319408`
- **LEU/4566614017** — Ledger Flex Product Presentation (authoritative hardware comparison). `https://ledgerhq.atlassian.net/wiki/spaces/LEU/pages/4566614017`
- **WAL/4172578925** — Wallet API documentation Workshop. `https://ledgerhq.atlassian.net/wiki/spaces/WAL/pages/4172578925`
- **WXP/4251680783** — DXP: Device Management Kit. `https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/4251680783`
- Public developer docs: `https://developers.ledger.com/`
- Wallet API public docs: `https://wallet.api.live.ledger.com/`

<div class="quiz-container" data-pass-threshold="80">
  <h3>Chapter 0.1 Quiz</h3>
  <p class="quiz-subtitle">Ledger ecosystem and product suite</p>
  <div class="quiz-progress"><span class="quiz-progress-bar"></span></div>

  <div class="quiz-question" data-correct="C">
    <p>Which of the following is the primary technical moat that differentiates Ledger from most competitors?</p>
    <div class="quiz-choices">
      <button class="quiz-choice">A) A larger engineering team based in Paris</button>
      <button class="quiz-choice">B) Exclusive partnerships with major exchanges</button>
      <button class="quiz-choice">C) Use of a certified Secure Element (CC EAL5+ / EAL6+) instead of a general-purpose microcontroller</button>
      <button class="quiz-choice">D) A proprietary Bluetooth stack that is faster than BLE 5.2</button>
    </div>
    <p class="quiz-explanation">The Secure Element is the core moat. It is a smartcard-grade certified chip; most competitors either do not use an SE at all or use a lower-tier non-certified chip. EAL5+/EAL6+ is the same tier used in banking cards and passports.</p>
  </div>

  <div class="quiz-question" data-correct="B">
    <p>Which Ledger device is the internal codename "Europa" referring to?</p>
    <div class="quiz-choices">
      <button class="quiz-choice">A) Ledger Nano Gen5</button>
      <button class="quiz-choice">B) Ledger Flex</button>
      <button class="quiz-choice">C) Ledger Stax</button>
      <button class="quiz-choice">D) Ledger Apex</button>
    </div>
    <p class="quiz-explanation">Europa is the internal codename for Ledger Flex. You will see `europa` in repo names, Speculos model flags, and CI matrices — it is not a separate product.</p>
  </div>

  <div class="quiz-question" data-correct="D">
    <p>A Live App that lives inside a webview in Ledger Live talks to the host application through which interface?</p>
    <div class="quiz-choices">
      <button class="quiz-choice">A) Direct HID access to the device</button>
      <button class="quiz-choice">B) The Coin Framework TypeScript API</button>
      <button class="quiz-choice">C) DMK (Device Management Kit)</button>
      <button class="quiz-choice">D) The Wallet API (JSON-RPC under the hood)</button>
    </div>
    <p class="quiz-explanation">Live Apps are webviews that use the Wallet API to request accounts, sign transactions, and share state with Ledger Live. Under the hood it is JSON-RPC between the webview and the host.</p>
  </div>

  <div class="quiz-question" data-correct="A">
    <p>Your Jira ticket is tagged with the project key `VG`. Which of the following is the correct first reaction as a consumer-side QAA intern?</p>
    <div class="quiz-choices">
      <button class="quiz-choice">A) VG is the Vault / Ledger Enterprise Jira project — confirm with your tech lead whether this is actually in your scope before picking it up</button>
      <button class="quiz-choice">B) VG tickets are always Ledger Live Desktop tickets; proceed normally</button>
      <button class="quiz-choice">C) VG means "Verification Group" and is used for cross-team test reviews</button>
      <button class="quiz-choice">D) VG is the Ledger Vault Git repository; clone it and start reading</button>
    </div>
    <p class="quiz-explanation">VG is the Jira project for Ledger Vault / Ledger Enterprise (LES business unit). A consumer-side QAA intern is unlikely to be staffed on VG tickets unless explicitly assigned; always confirm scope first.</p>
  </div>

  <div class="quiz-question" data-correct="C">
    <p>Which of the following statements about the QAA scope is most accurate?</p>
    <div class="quiz-choices">
      <button class="quiz-choice">A) QAA writes security tests against the Secure Element silicon</button>
      <button class="quiz-choice">B) QAA reviews BOLOS syscall parameter checks and submits patches</button>
      <button class="quiz-choice">C) QAA guards the software-to-hardware interface (Ledger Live, Live Apps, coin modules) and uses Speculos to simulate the device side</button>
      <button class="quiz-choice">D) QAA performs Common Criteria certification of firmware releases</button>
    </div>
    <p class="quiz-explanation">QAA lives in the top three layers of the stack: the host app, the integration libraries (Wallet API, Coin Framework, DMK), and the APDU contract layer. Silicon and firmware internals belong to Firmware, the Donjon, and external certification labs.</p>
  </div>

  <div class="quiz-score"></div>
</div>

---

## Hardware Deep Dive & BOLOS

<div class="chapter-intro">
This chapter opens the device up. You will learn why every Ledger device contains two chips rather than one, what the BOLOS operating system actually does, how an embedded coin app is installed and isolated, how a firmware release rolls out in four phases, and which device families the QAA test matrix cares about. None of this will make you a firmware engineer — it will make you a QAA engineer who can read a firmware-adjacent Jira ticket without asking what half the words mean.
</div>

### 0.2.1 The Two-Chip Architecture

The single most important fact about a Ledger device is that it contains **two chips**, not one:

1. A **Secure Element (SE)** — a certified smartcard-grade microcontroller that holds the master seed, derives keys, performs signatures, and drives the secure display.
2. A **general-purpose microcontroller (MCU)** — an STM32 family chip that drives peripherals: USB, BLE, NFC, power management, the power button, and (on some devices) the screen.

The canonical source is **FW/5737481596 — "Ledger OS - Introduction"**, which is explicit that Ledger OS is "the combination of the SE-side embedded software, the MCU-side embedded software, and the SEPH protocol over the link between them".

#### Why split the work across two chips?

The SE is the crown jewel. It is **certified** — each time the Firmware team wants to modify the SE-side OS, the change has to go through a chain of reviews and, for public firmware updates, re-certification. Certification is slow and expensive. You do not want your USB driver code, your Bluetooth stack, or your battery-management logic living under that constraint.

The MCU, by contrast, is cheap and replaceable. You can iterate on BLE firmware, fix a USB regression, or change the charge-curve of the battery without touching the SE. The MCU takes untrusted input from the outside world — a USB packet from a host that might be compromised, a BLE advertisement from an unknown device, an NFC tap from a random surface — and forwards it across the SEPH link to the SE. The SE decides what to do with it.

The **CC EAL certification applies only to the SE**, not the MCU (per R2 research, `FW/5737481596`). When marketing says "EAL6+ certified", they mean the SE. The MCU is a regular STM32.

#### The three-pillar security model

Also from FW/5737481596:

1. **Security at rest** — all secrets (master seed, derived keys) live in the SE. The SE has active shielding, encrypted memory, sensors (light, voltage, temperature) that can wipe the chip on tamper, and a scrambled internal layout.
2. **Security at use** — access is PIN-gated. Three wrong PIN attempts wipe the device. The communication protocols on the wire are secured, and Ledger OS curates every piece of incoming data before it reaches app code.
3. **Secure display** — the screen and the input (buttons or touch panel) are **wired directly to the SE** on Nano X, Nano S Plus, Stax, and Flex. Only SE-side code can request a display update. This means a compromised host cannot draw a fake "approve transaction" prompt on the device.

> **Gotcha:** On the original **Nano S**, the MCU drives the display — the SE sends `SEPROXYHAL_TAG_SCREEN_DISPLAY_STATUS` across the link and the MCU paints. On Nano X, Nano S Plus, Stax, and Flex, the SE drives the display directly via `screen_*` APIs. This architectural difference is one of the reasons Nano S is legacy — the secure-display property is weaker there.

#### What the SE silicon buys you concretely

A certified Secure Element is not just "a chip with a CC stamp on it". The EAL rating is evidence of a long list of concrete defensive features, attested in `FW/3295281471 — Secure Elements`:

- **Active shielding.** Metal mesh layers above the circuit; cutting through them triggers defensive reactions.
- **Encrypted memory bus.** Data between the CPU core and flash/RAM is encrypted on the chip; probing pins with an oscilloscope yields ciphertext, not plaintext.
- **Scrambled internal layout.** The physical placement of logic gates on the die is not linear — reverse-engineering via decapping and microscopy is dramatically harder than on a commodity MCU.
- **Environmental sensors.** Light, voltage, temperature, and clock-glitch detectors can mute or wipe the chip. A common attack is to freeze a chip and glitch its clock to skip a PIN check; the SE notices and shuts down.
- **Formal security lifecycle.** Chip development, personalisation, app signing, and field usage each go through documented, audited pipelines.

Ledger **does not design the SE silicon**. It buys blank SEs from STMicroelectronics and loads its own OS on them. The moat is the combination (certified chip + Ledger-specific OS + Ledger-specific app-signing infrastructure). This is why Ledger's app deployment pipeline (`FW/3971186947`) is load-bearing — an attacker who could sign apps with Ledger's key would break the whole model, so the signing is HSM-gated with separation of duties.

### 0.2.2 Chip Catalog

Cross-referenced from FW/5737481596 and LEU/4566614017. Memorise the columns you care about (SE chip, MCU chip, certification); ignore the rest until you need them.

| Device          | SE chip      | SE certification | MCU chip                 | SE flash / RAM     |
|-----------------|--------------|------------------|--------------------------|--------------------|
| Nano S (legacy) | ST31H320     | (legacy tier)    | STM32F042                | 320 KB / 10 KB     |
| Blue (legacy)   | ST31G480     | (legacy tier)    | STM32L476                | 480 KB / 12 KB     |
| Nano X          | ST33J2M0     | CC EAL5+         | STM32WB55 / STM32WB35    | 2 MB / 50 KB       |
| Nano S Plus     | ST33K1M5C    | CC EAL6+         | STM32F042                | 1.5 MB / 64 KB     |
| Stax            | ST33K1M5C    | CC EAL6+         | STM32WB35                | 1.5 MB / 64 KB     |
| Flex (Europa)   | ST33K1M5     | CC EAL6+         | STM32WB35                | 1.5 MB / 64 KB     |
| Nano Gen5       | ST33K1M5C    | (in progress)    | STM32WB35                | 1.5 MB / 64 KB     |

Worth noting:

- The **ST33 family** (K1M5C / K1M5 / J2M0) is the current generation. ST33 parts have an MMU, which matters for BOLOS's memory-mapping strategy (see 0.2.4).
- The **ST31 family** is older (used on Nano S and Blue) and has no MMU — BOLOS has to simulate position-independent code with a compile-time virtual address plus a runtime PIC macro.
- The **STM32WB family** is a dual-core Cortex-M (Cortex-M4 + Cortex-M0+) with a built-in BLE 5.2 radio. This is why Nano X, Stax, and Flex can do Bluetooth — the MCU silicon has a radio on die.
- On **Flex**, roughly 1.15 MB of the SE's 1.5 MB flash is usable for apps, language packs, and pictures (the "Custom Lock Screen" feature on touch devices stores user images on-SE). (per R2 research, `LEU/4566614017`.)

> **Note:** The **BGIv2** personalisation machine described in `HTS/6233456749` is designed to personalise the SE at the factory for Stax, Flex, and **Nano Gen5**. This is external corroboration that Nano Gen5 is the next classic-form device in manufacturing (as of 2026-04). Treat it as a target in your test matrix planning, but do not yet ship tests against it unless your tech lead tells you the device is ready.

### 0.2.3 BOLOS — What the OS Actually Does

**BOLOS** stands for **Blockchain Open Ledger Operating System**. It is the firmware that runs on the SE side of every Ledger device. The canonical definition is in **ENGB2C/3644882985 — "[Arch] Introduction to BOLOS"**:

> BOLOS is the name of the firmware embedded in all Ledger Hardware Wallets (stands for Blockchain Open Ledger Operating System). BOLOS provides a lightweight, open-source framework for developers to build source code portable applications that run in a secure environment.

BOLOS is organised into six modules:

1. **Input/output** — USB, BLE, NFC transport handling.
2. **Cryptography** — hardware-accelerated primitives (ECDSA, EdDSA, SHA families, AES).
3. **Persistent storage** — app-scoped NVM that survives reboots.
4. **Personalisation** — the master-seed interface. Apps never see the seed itself; they request derived keys.
5. **Endorsement & application attestation** — proves to a remote verifier that a signature really came from a genuine Ledger device running a genuine app.
6. **User interface** — drawing primitives, button/touch event dispatch.

#### The Dashboard

When you power on a Ledger device and you are not inside a coin app, you see the **Dashboard**. The Dashboard is a privileged BOLOS application — it is what the host PC or phone talks to when installing/deleting apps, reading the firmware version, getting or setting the device name, and updating firmware (ENGB2C/3644882985). If you have ever run `listApps` in a Ledger Live log, the response came from the Dashboard.

#### Kernel vs Userland

The SE-side BOLOS is split into:

- **HAL (Hardware Abstraction Layer)** — drives SE features and hardware crypto.
- **Board System Package** — drives SE-connected peripherals.
- **Kernel** (privileged, direct peripheral access) and **Userland** (restricted), separated by the **syscall barrier**.

Syscalls enforce parameter checks, distinct RAM, distinct NVM, and a distinct stack between kernel and app. Apps cannot corrupt the kernel; they can only call into it via typed entry points.

#### SEPH — the SE / MCU link protocol

The two chips talk over a link protocol called **SEPH** (Secure Element PROXY HAL). From a QAA perspective you will not write SEPH packets directly, but you will see the tag names in Speculos logs and in error messages:

- `SEPROXYHAL_TAG_CAPDU_EVENT` — an APDU command arrived from the host. This is the event that wakes up a coin app.
- `SEPROXYHAL_TAG_BUTTON_PUSH_EVENT` — a button was pressed on a classic device.
- `SEPROXYHAL_TAG_FINGER_EVENT` — a touch event on Stax / Flex.
- `SEPROXYHAL_TAG_SCREEN_DISPLAY_STATUS` — the SE has finished a display update (or, on Nano S, the SE is asking the MCU to paint).
- `SEPROXYHAL_TAG_TICKER_EVENT` — a periodic tick, used for timeouts and animations.

When a Speculos log dumps a TLV trace, these are the tags that show up. Knowing them at a glance will save you thirty minutes the first time a test hangs on a ticker event nobody dispatched.

#### Endorsement and attestation

BOLOS includes an **endorsement** module, which is the mechanism by which a Ledger device can prove to a remote verifier that "yes, I am a genuine Ledger device, and yes, I am running a genuine (signed) Ledger app". This is what powers the **Genuine Check** flow in Ledger Live (the "your device is genuine" dialog shown just after onboarding) and what gates Ledger Recover eligibility. QA tests the Genuine Check flow end-to-end; QA does not implement or audit the cryptography behind it.

### 0.2.4 The BOLOS Application Model

From **TA/3478093827 — "[ARCH] BOLOS application & SDK"**:

- A BOLOS app is **native bytecode + RAM data + flash permanent storage**. The memory mapping is pseudo-static and fixed at compile time.
- On older SEs (Cortex-M without address virtualisation), the link file uses **virtual addresses** (`0xC0DE0000` for code, `0xDA7A0000` for data on Nano X and Nano S Plus) and a **PIC (Position-Independent Code) macro** converts virtual addresses to real ones at runtime. Nano X, running on the ST33 with an MMU, has an adapted PIC behaviour.
- **One app runs at a time.** When the user selects an app from the Dashboard, the kernel programs the MPU/MMU with the app descriptor (code size, storage size), hands over all the SE's RAM to the app, and jumps to the app entry point.
- Apps communicate with BOLOS via **syscalls** (`SYS_Call`) which are ARM software exceptions. The kernel runs the syscall in supervisor mode then returns to user mode.
- Events are **TLV-formatted messages**. An app reads events via `io_seph_recv(...)`, handles them (`SEPROXYHAL_TAG_CAPDU_EVENT` for an incoming APDU, `SEPROXYHAL_TAG_BUTTON_PUSH_EVENT` for a button press), and posts replies via typed BOLOS APIs or `io_seph_send`.
- Error handling uses a **Try/Catch/Throw macro system** built on `setjmp`/`longjmp`.

The practical QAA consequence is that there is only one "active app" at a time on the device. If a test needs to test a flow that spans two coin apps (say, swap Bitcoin to Ethereum), Ledger Live is the one orchestrating app switches on the device, and the device is briefly back on the Dashboard in between. Your test has to tolerate that transition.

#### What this means for test timing

Every "switch app" step involves the Dashboard tearing down the current app's memory mapping, wiping the SE's RAM, programming the MPU/MMU for the new app, and relaunching. On a real device this takes on the order of **one to three seconds**. On Speculos it is faster, but not instant. Your test must:

- Wait for the app-switch to complete before sending the next APDU (Ledger Live's internal signer does this for you — do not reinvent it).
- Allow for the Dashboard to briefly appear in screen-capture assertions between two app-specific flows.
- Not assume storage state is shared across apps. Each app has its own NVM region; the Ethereum app cannot read a setting written by the Bitcoin app.

#### Shared code and the "Moving to BOLOS" effort

The Embedded SDK (C and Rust) ships a large amount of **shared code** that every coin app uses: the NBGL UI framework for Stax/Flex display flows, the standard APDU parser, the Bagl/BAGL framework on classic devices, common crypto helpers. Historically this code lived in the SDK and was linked into every app binary. An active tech-debt effort (per R2 research, `FW/6273138715` and `FW/6173196300`) is migrating chunks of this shared code **into BOLOS itself** so that apps can call it via syscalls rather than re-link it.

For QAA this mostly matters when a regression ticket says something like "after FW X.Y.Z, the Cardano app's review screen renders differently". The most likely cause is not Cardano team changes — it is an NBGL change that moved from the SDK into BOLOS.

### 0.2.5 Embedded Coin Apps

Every coin we support has an **embedded coin app** — a native binary compiled from either the **C SDK** (`github.com/LedgerHQ/ledger-secure-sdk`) or the **Rust SDK** (`github.com/LedgerHQ/ledger-device-rust-sdk`). Examples: the Bitcoin app, the Ethereum app, the Solana app, the Cosmos app.

Each app:

- Is built into an `app.elf` + manifest. For C apps the installer tool is `ledgerblue.loadApp`; for Rust it is `cargo-ledger` (per R2 research, `FW/6679429130`).
- Is **signed by Ledger** through an HSM-based build pipeline (`FW/3971186947 — App deployment pipelines explained & how-to`). Unsigned or wrongly-signed apps are rejected at install time by the Dashboard.
- Is **installed into a slot** in the SE's flash. Ledger Live (via DMK) orchestrates install, uninstall, and update.
- Is **isolated from other apps** — BOLOS's MPU/MMU programming means the Ethereum app literally cannot read the Bitcoin app's storage.

For QAA, the interaction model is:

- You do not install real apps in a CI run. You run **Speculos**, which loads an app binary from the **coin-apps** repo, and your test drives the Speculos APDU interface exactly as it would drive a real device.
- The coin-apps repo is consumed by Live QA and PTX for automated testing against the latest app binaries — this is the contract surface between host-side coin modules and device-side coin apps.

> **Note:** When a QAA test expects "version X.Y.Z of the Bitcoin app", the version string refers to the embedded coin app binary, not the BOLOS version, not the MCU firmware version, and not the Ledger Live version. All four of these version numbers exist independently.

#### App slots, memory, and why installs fail

The SE's flash is partitioned into **slots**. Each installed app (Bitcoin, Ethereum, Solana, ...) occupies one slot. Language packs and, on touch devices, custom lock-screen images also consume SE flash.

When a user asks Ledger Live to install a new app, DMK performs a capacity check. If there is not enough contiguous space, Ledger Live proposes an eviction: "remove another app to make room." The eviction UX is subtle — the user's **accounts are preserved** (accounts live in Ledger Live's host-side storage, not in the app's on-device storage), but any app-specific on-device settings (for example, blind-signing enabled on the Ethereum app) are lost and must be re-enabled after re-install.

From a QAA perspective, this creates two test surfaces:

1. **The happy-path install.** App fits, install succeeds, app appears in the manager list. Covered by basic manager smoke tests.
2. **The eviction path.** App does not fit, Ledger Live prompts the user to uninstall something, user accepts, new app is installed. This path has a history of regressions — the fragmentation algorithm on older firmwares occasionally reports "enough space" and then fails halfway through the install. Always include at least one eviction-path test when asserting on manager behaviour across a firmware update.

> **Gotcha:** On the very smallest SEs (Nano S, ST31H320 with 320 KB flash), app combinations that fit on a Nano S Plus do not fit on Nano S. A test that "installs Bitcoin, Ethereum, Cardano, Polkadot, and Solana simultaneously" will silently pass on Nano X and fail on Nano S not because of a logic bug but because of physics. Parameterise app-install counts per device family, or explicitly exclude Nano S.

#### The coin-apps repository

The **coin-apps** repository (referenced in `FW/3971186947`) aggregates compiled, signed app binaries keyed by device model and firmware version. Both Live QA and PTX pull from it for automated testing. When your Speculos fixture declares `bitcoin@2.2.4 on nanoX@2.5.0`, it is resolving against the coin-apps matrix.

Three gotchas commonly trip up new QAA engineers here:

1. **The version pinned in CI lags the version released to users.** This is intentional — QA tests a tag before it is promoted to the public Ledger Live manager. Do not file a "Bitcoin app version mismatch" ticket without checking whether it is a lag or a defect.
2. **A new firmware can break an old app binary.** Apps are compiled against a specific BOLOS ABI. When BOLOS changes (for example, when shared code migrates from SDK to BOLOS), every app needs to be recompiled against the new SDK. That recompile is what shows up in the coin-apps repo.
3. **Speculos model selection has to match the app binary's target.** A `stax` app binary will not load on a Speculos instance launched with `--model nanoX`. You will get a cryptic "app not found" or "signature mismatch" error.

### 0.2.6 Firmware Release Lifecycle

Firmware releases are not a `git push`. They are a multi-week, multi-team coordination exercise documented in **NANO/2869264439 — "Process for firmware releases"**.

A firmware update touches **three components in this order** (per `BO/5346164745`):

1. **The Secure Element OS** — updated via an **OSU** (Operating System Updater), a small transitional image that migrates the SE from the old OS to a state where the new OS can be installed, then installs the final OS. DMK contains non-trivial logic to compute a **multi-step OSU path** when a user is several versions behind (`WXP/6943703128`).
2. **The MCU firmware** — flashed conditionally via the `/mcu` endpoint.
3. **The Bootloader** — flashed last.

Each firmware update **requires a matching OSU**; Ledger Live (via DMK) orchestrates the whole sequence.

The release process itself has **four phases** (NANO/2869264439, originally written around Nano X firmware 2.0.0 but still the reference):

**Phase 1 — Scoping & planning.** Firmware and Product define scope: features, security fixes from the Donjon, hardware changes, UX changes. The release timeline is set. The **update paths** are defined (which previous versions can reach the new one — not every old version supports a direct jump). A kick-off meeting brings in Firmware, Hardware, Production, Ledger Live, Coin Integration, Marketing, and Customer Success.

**Phase 2 — Development.** Firmware team implements scope. UI/wording can be parallelised. Ledger Live dependencies that track the firmware change (e.g. a new APDU, a new device model) are developed in parallel.

**Phase 3 — Testing.** RC1 (Release Candidate 1) is pushed to **provider 1** (the internal firmware distribution endpoint) along with recompiled apps. QA is split across three streams:

- **Live QA** — end-to-end scenarios, update paths, Live-compatible blockchains, USB and BLE transports.
- **General firmware / non-regression QA (Kiev team)** — onboarding, latest update path, PIN, passphrase, non-regression, third-party wallets, non-Live blockchains.
- **Product QA** — the changelog and every update path.

The Production team ships devices for QA via a CUS ticket. Speculos and the **Calvados CI** automate a large slice of the regression surface.

**Phase 4 — Release.** Marketing and CS prepare comms and Help Center updates. The FW team publishes the final version on provider 1 the morning of release with **progressive rollout**: typically **10-25% on D, 50-60% on D+1, and 100% by end of week**. Mixpanel / Periscope / Zendesk analytics are tracked daily, and a go/no-go decision is taken each day. If something breaks, the rollout can be paused or rolled back before the general population is exposed.

Two release types coexist: **public updates** (via Ledger Live manager, what users see) and **production-only updates** (flashed at the factory and never exposed via Live; example: LNX 1.2.4-6). A later public update can merge a production-only version into its update path.

> **Gotcha:** If you are writing a test that exercises "updating firmware from X to Y", you are writing a test that depends on the **update path** defined in Phase 1. Not every X → Y jump is supported. Check the update path matrix before assuming your test premise is valid.

#### What QAA actually owns inside the release process

Within Phase 3 (Testing), the QAA team is responsible for the **Live QA** stream: end-to-end flows on Ledger Live Desktop and Mobile, exercised against the RC firmware. The split is:

- **Live QA (you)** — desktop and mobile E2E, account discovery, swap, buy/sell, staking, NFT gallery, USB/BLE transport sanity. Every flow that a real user would touch inside Ledger Live.
- **Kiev team / General firmware QA** — device-centric flows that do not require Ledger Live: onboarding, PIN, passphrase, third-party wallet compatibility, non-Live blockchains. Their tests frequently use Speculos and bespoke harnesses; their work rarely shows up in the same Jira projects as yours.
- **Product QA** — the changelog walk-through and the exhaustive update-path matrix. This stream has a dedicated QA lead and a dedicated template per device model.

This division matters because test ownership is frequently ambiguous at the boundaries. If a bug reproduces only during a firmware-update flow started from within Ledger Live, it might be yours; if it reproduces from a cold-boot onboarding without Ledger Live, it is Kiev's. Clarify before you adopt a ticket.

#### Production-only updates

Some firmware updates are **production-only**: they are flashed at the factory on new devices and are never exposed to end users via the Ledger Live manager. The canonical example in R2 is LNX 1.2.4-6. These exist because certain hardware batches require a version the public should never try to install directly. QAA rarely interacts with production-only versions, but you may see them in update-path diagrams as intermediate nodes. Do not assume a version you have never heard of is a typo — check the production-only list first.

### 0.2.7 Speculos — The QA Emulator

**Speculos** is Ledger's official SE emulator. It is how QA (and embedded-app developers) drive device flows in environments where a physical device is unavailable or impractical — CI runners, cloud test infrastructure, parallel test shards.

You do not need to master Speculos right now. Park these three facts:

1. **Speculos emulates the SE + MCU + display + buttons/touch.** Your test can send APDUs, capture screens, simulate button presses, and query device state.
2. **Speculos loads a real embedded app binary.** The Bitcoin app binary you use in CI is the same binary (or a build of the same source) as the one a user would install.
3. **Speculos is not a perfect substitute for a real device.** Timing is different. Some hardware features (NFC field detection, specific BLE pairing quirks, battery-drain edge cases) are not modelled. Always end a feature campaign with at least one pass on real hardware.

Full Speculos coverage lives in **Part 3 Chapter 3.3**. For now, when you see `speculos` in a config file or a Jira ticket, you know what it is.

#### Why Speculos exists at all

The obvious question is: if real devices are available, why emulate them? Three reasons:

1. **CI scale.** A single CI run of the Ledger Live Desktop suite executes hundreds of parameterised device-involving tests across multiple device models. Plugging in a USB rack of real devices for every commit does not scale.
2. **Reproducibility.** A real device carries state (installed apps, seed, settings). A Speculos instance starts from a known, scripted state every time. This removes a huge class of "works on my machine" flakiness.
3. **Exploration.** An embedded-app developer debugging an APDU parser can run Speculos under `gdb` and step through the device-side code. You cannot attach a debugger to a real SE without breaking the tamper seals and voiding every certification.

#### What Speculos is not

- Speculos is **not a security-testing tool**. It emulates the SE's behaviour, not its tamper-resistance, not its side-channel resistance, not its fault-injection resistance. Do not use it to argue about device security.
- Speculos is **not a public release binary**. It is developer-facing infrastructure. Users do not run Speculos.
- Speculos does **not replace a final pass on real hardware**. Every feature campaign must conclude with at least one execution on each supported physical device family, tracked in the release sign-off.

### 0.2.8 Device Families in the QAA Test Matrix

The consumer-side QAA test matrix, as of 2026-04, covers:

| Family   | Devices                                   | Notes                                                |
|----------|-------------------------------------------|------------------------------------------------------|
| Classic  | Nano S (legacy), Nano X, Nano S Plus, Nano Gen5 (planned) | Button navigation; OLED; smallest screen real estate |
| Touch    | Stax, Flex (Europa)                       | Sync Onboarding; E-Ink; larger flows                 |
| Legacy   | Blue, Nano S                              | Regression only; limited feature support             |

When you read a test file, the device model usually appears either:

- As a Speculos launch flag (`--model nanox`, `--model stax`, `--model europa`).
- As a Jest / Playwright parameter in a `.each([...])` loop.
- As a tag on a Jira ticket (`device:flex`, `device:nanoS+`).

Your job, reading a ticket, is to work out which families it applies to and whether your test needs to be parameterised. A test that claims to cover "all devices" but only actually runs against Nano X is a red flag that your reviewer will catch.

> **Note:** Because Nano S is legacy, **most new feature work explicitly excludes it**. If a ticket does not mention Nano S, assume it is out of scope. If in doubt, confirm with the feature PM or tech lead — do not burn half a day writing a Nano S variant of a test that was never supposed to exist.

#### Mapping device names to Speculos model flags

For reference — this lookup table saves time when you are reading somebody else's test fixture and the naming is inconsistent:

| Human-readable name  | Typical Speculos flag     | Common repo identifier |
|----------------------|---------------------------|------------------------|
| Nano S               | `--model nanos`           | `nanoS`                |
| Nano X               | `--model nanox`           | `nanoX`                |
| Nano S Plus          | `--model nanosp`          | `nanoSP` / `nanoS+`    |
| Stax                 | `--model stax`            | `stax`                 |
| Flex                 | `--model europa`          | `europa` / `flex`      |
| Nano Gen5 (planned)  | (TBD, not yet in CI)      | `nanoGen5`             |

Note that **Flex appears as `europa`** in Speculos. This is the most common naming trip-up for new QAA engineers — a test that passes under `--model europa` but fails under `--model flex` is almost always passing because `flex` silently falls back to a default model, not because Flex is genuinely broken. Always use the canonical flag.

#### Touch vs classic test branches

Because touch and classic devices have fundamentally different UI paradigms, many E2E tests split into two branches. A swap flow on Nano X is "press right button to scroll through outputs, two-button press to confirm"; the same swap flow on Flex is "tap 'Continue', review each output, tap 'Hold to sign'". These branches live side-by-side in the codebase, and the test runner dispatches based on device model. When you inherit a test that only covers the classic branch, your first question is whether the touch branch exists elsewhere or needs to be written.

### 0.2.9 Putting It All Together

At this point you have seen every major moving part of a Ledger device. Let us compress the whole picture into a one-page mental model you can carry into every test you write.

**The static picture.** A Ledger device is two chips (SE + MCU) connected by SEPH. The SE runs BOLOS, which hosts the Dashboard plus exactly one active app at a time. Apps are signed C/Rust binaries installed into SE flash slots. On classic devices the SE drives the display via the MCU (Nano S) or directly (Nano X / S Plus). On touch devices (Stax / Flex) the SE drives a higher-resolution E-Ink panel directly. The MCU handles USB, BLE, NFC, power, and — on some models — the display.

**The dynamic picture.** The host (Ledger Live + DMK) opens a channel to the device and speaks APDUs. An APDU is routed by the Dashboard to the currently-active app, which handles it, displays something on the secure screen, waits for user approval, and responds. For signing flows the device-side display is the **ground truth** that the user reviews before approving.

**The release picture.** Every firmware release touches three components (SE OS, MCU, Bootloader) in that order, requires a matching OSU, goes through four phases (scope / dev / test / release), and rolls out progressively (10-25% → 50-60% → 100%) with daily go/no-go checks. Coin apps release independently. Ledger Live releases independently again.

**Your seat.** You are in the Live QA stream. You write Playwright tests (desktop) and Detox/Maestro tests (mobile) that drive the host UI, talk to Speculos on the device side, and assert on both surfaces. You do not write BOLOS, you do not design silicon, you do not certify firmware — but you are the last line between a regression and a user's trust.

One concrete prescription if you remember nothing else from this chapter: **always assert on the device screen for any flow where the user approves something.** Host-only assertions are insufficient. The attack model assumes the host can lie. The device cannot.

#### A worked example — sending 0.01 ETH, mapped to the stack

Take a minimal real-world flow and walk through which layers are involved. User wants to send 0.01 ETH from Ledger Live Desktop to a friend's address.

1. **UI layer (LLD, React + Electron).** User opens the Ethereum account, clicks "Send", enters the recipient address and amount, clicks "Continue". Your Playwright test drives this with locators like `getByTestId("account-send-button")` and `fill("0.01")`.
2. **Coin Module layer (`@ledgerhq/coin-evm` or similar TS package).** Builds an unsigned transaction object: nonce, gas price, gas limit, to-address, value, chain ID. Fetches the current nonce from the backend. Computes fee estimates.
3. **DMK layer.** Opens a signing session. Confirms the Ethereum embedded app is open on the device. Serialises the transaction into the Ethereum app's signing APDU format.
4. **Ethereum embedded app (on SE).** Receives the APDU stream. Parses the RLP-encoded transaction. Displays "Review transaction" → recipient → amount → gas → chain — each as a separate screen. User scrolls through and approves.
5. **SE cryptographic primitives.** The Ethereum app calls a BOLOS syscall to derive the signing key from the master seed at the BIP-44 path for the chosen account, computes the ECDSA signature over the transaction hash, returns the signature to userland.
6. **DMK receives signed tx.** Reassembles the signed transaction.
7. **Coin Module broadcasts.** Posts the signed transaction to an Ethereum node provider.
8. **UI confirms.** LLD shows "Transaction sent" toast, refreshes the account view.

Your test asserts on: the send modal flow (steps 1-2), the device screens (steps 4-5 via Speculos), the success toast (step 8), and the appearance of the new operation in the account's operations list (step 8). A well-written test never skips the Speculos assertion, because that is the step at which "do the host and the device agree about what the user is signing?" is verified.

#### Vocabulary checkpoint

If these terms feel natural to you now, the chapter has done its job. If any feel foreign, re-read the relevant section before moving on:

- SE, MCU, BOLOS, Dashboard, syscall, SEPH
- APDU, HID, BLE, NFC
- Speculos, OSU, coin-apps repo
- Nano X, Nano S Plus, Stax, Flex (Europa), Nano Gen5
- DMK, Wallet API, Coin Framework, Live App
- EAL5+, EAL6+, Donjon, secure display, clear-signing
- Live QA stream, Kiev team, Product QA, progressive rollout

#### Where to go next

- **Part 1** — Ledger Live foundations: product, tech stack, E2E architecture, monorepo, and your dev environment.
- **Part 3 Chapter 3.3** — Speculos deep dive.
- **Part 4** — Desktop E2E end-to-end: Playwright from zero, Electron, codebase, daily workflow, and two real ticket walkthroughs.
- **Part 5** — Mobile E2E end-to-end: React Native primer, Detox, mobile codebase, and the QAA-702 walkthrough.
- **Part 6** — The Swap Live App.
- **Part 7** — Mastery: test strategy, cross-platform debugging, CI/CD, AI agents.

#### Resources — Chapter 0.2

- **FW/5737481596** — Ledger OS - Introduction. `https://ledgerhq.atlassian.net/wiki/spaces/FW/pages/5737481596`
- **FW/3295281471** — Secure Elements (what an SE is, why it matters). `https://ledgerhq.atlassian.net/wiki/spaces/FW/pages/3295281471`
- **ENGB2C/3644882985** — [Arch] Introduction to BOLOS. `https://ledgerhq.atlassian.net/wiki/spaces/ENGB2C/pages/3644882985`
- **TA/3478093827** — [ARCH] BOLOS application & SDK. `https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/3478093827`
- **NANO/2869264439** — Process for firmware releases. `https://ledgerhq.atlassian.net/wiki/spaces/NANO/pages/2869264439`
- **BO/5346164745** — Firmware update related issues (SE / MCU / Bootloader order). `https://ledgerhq.atlassian.net/wiki/spaces/BO/pages/5346164745`
- **FW/3971186947** — App deployment pipelines explained & how-to. `https://ledgerhq.atlassian.net/wiki/spaces/FW/pages/3971186947`
- **HTS/6233456749** — BGIv2 Specification (SE personalisation; Stax / Flex / Nano Gen5). `https://ledgerhq.atlassian.net/wiki/spaces/HTS/pages/6233456749`
- Embedded C SDK: `https://github.com/LedgerHQ/ledger-secure-sdk`
- Embedded Rust SDK: `https://github.com/LedgerHQ/ledger-device-rust-sdk`
- Public developer docs: `https://developers.ledger.com/docs/device-app/getting-started`

<div class="quiz-container" data-pass-threshold="80">
  <h3>Chapter 0.2 Quiz</h3>
  <p class="quiz-subtitle">Hardware deep dive and BOLOS</p>
  <div class="quiz-progress"><span class="quiz-progress-bar"></span></div>

  <div class="quiz-question" data-correct="B">
    <p>The CC EAL5+ / EAL6+ certification on a Ledger device applies to which component specifically?</p>
    <div class="quiz-choices">
      <button class="quiz-choice">A) The MCU (the STM32 microcontroller)</button>
      <button class="quiz-choice">B) The Secure Element (the ST33 / ST31 smartcard chip)</button>
      <button class="quiz-choice">C) The Bluetooth stack</button>
      <button class="quiz-choice">D) Ledger Live Desktop's Electron build</button>
    </div>
    <p class="quiz-explanation">CC (Common Criteria) EAL certification is a chip-level certification. On a Ledger device it applies only to the Secure Element. The MCU is a regular non-certified STM32.</p>
  </div>

  <div class="quiz-question" data-correct="C">
    <p>How many BOLOS applications can be running on the Secure Element at the same time?</p>
    <div class="quiz-choices">
      <button class="quiz-choice">A) As many as fit in flash, running preemptively</button>
      <button class="quiz-choice">B) Up to four, multiplexed by the kernel</button>
      <button class="quiz-choice">C) Exactly one — the selected app gets all the SE RAM; the MPU/MMU is reprogrammed on each app switch</button>
      <button class="quiz-choice">D) Zero — apps run on the MCU, not the SE</button>
    </div>
    <p class="quiz-explanation">BOLOS runs exactly one app at a time on the SE. When the user selects an app from the Dashboard, the kernel programs the MPU/MMU with the app descriptor and hands over all of the SE's RAM.</p>
  </div>

  <div class="quiz-question" data-correct="A">
    <p>During a firmware release, what is the correct order in which the three components are updated on the device?</p>
    <div class="quiz-choices">
      <button class="quiz-choice">A) Secure Element OS (via OSU), then MCU firmware, then Bootloader</button>
      <button class="quiz-choice">B) Bootloader, then MCU firmware, then Secure Element OS</button>
      <button class="quiz-choice">C) All three are flashed atomically in a single transaction</button>
      <button class="quiz-choice">D) MCU firmware only — the SE OS is never updated post-factory</button>
    </div>
    <p class="quiz-explanation">SE OS (through an OSU transitional image) first, then MCU firmware (conditionally via /mcu), then Bootloader. Each firmware update requires a matching OSU and the whole sequence is orchestrated by Ledger Live / DMK.</p>
  </div>

  <div class="quiz-question" data-correct="D">
    <p>Typical progressive rollout percentages for a public Ledger firmware release are best described as:</p>
    <div class="quiz-choices">
      <button class="quiz-choice">A) 100% on day one, rolled back only if a critical bug is found</button>
      <button class="quiz-choice">B) 1% on day one, 5% on day two, 10% by end of week</button>
      <button class="quiz-choice">C) Internal only for the first week, then 100% to the public on day eight</button>
      <button class="quiz-choice">D) 10-25% on day one, 50-60% on day two, 100% by end of week, with daily go/no-go checks</button>
    </div>
    <p class="quiz-explanation">Per NANO/2869264439, the Firmware team typically rolls out 10-25% on D, 50-60% on D+1, 100% by end of week, tracking Mixpanel / Periscope / Zendesk daily and making a go/no-go call each day.</p>
  </div>

  <div class="quiz-question" data-correct="B">
    <p>A new QAA test file introduces a parameterised loop that runs on `['nanoS', 'nanoX', 'nanoS+', 'stax', 'flex']`. A reviewer pushes back. Which of the following is the most likely reason?</p>
    <div class="quiz-choices">
      <button class="quiz-choice">A) `stax` should be written as `staxV1`</button>
      <button class="quiz-choice">B) Nano S is legacy and most new feature work explicitly excludes it; the ticket almost certainly does not cover it</button>
      <button class="quiz-choice">C) The list is missing Ledger Blue, which is still actively supported</button>
      <button class="quiz-choice">D) Parameterised loops are banned in QAA tests</button>
    </div>
    <p class="quiz-explanation">Nano S is legacy and explicitly out of scope for most new feature work (it is not supported by Multisig, not supported by Recover, and has architectural differences such as MCU-driven display). Including it by default burns CI time and masks real coverage gaps.</p>
  </div>

  <div class="quiz-score"></div>
</div>
## Teams and Ownership

<div class="chapter-intro">
Ledger is not one monolithic engineering team — it is a federation of tribes, squads, and horizontal teams that each own a slice of the product you will be testing. As a QAA engineer, you will ping more teams in a single week than most developers ping in a quarter: WXP for UI regressions, PTX when a swap test goes red, Coin Integration when a family-specific flow breaks, Wallet CI when a pipeline hangs. This chapter gives you the org map and the names. By the end, when someone says "tag live-hub on this" or "ask wallet-ci", you will know exactly who they mean. (R1)
</div>

### 0.3.1 The Org Map — QAA's web of neighbours

Ledger's engineering is split by the CTO (Charles GUILLEMET) into **vertical Business Units** (BUs) aligned with products, and **horizontal teams** (HOR) that support every BU. QAA sits inside the Quality Assurance organisation, alongside QA B2C (manual consumer QA) and QA B2B (manual Vault/HSM QA). But QAA's daily orbit is dominated by the engineering tribes that produce the code you test — not by the other QA teams. (R1)

Here is the reconstructed chart. The edges matter more than the boxes — trace which teams QAA talks to most:

```
                              Ledger SAS
                                  |
                               CTO (Charles GUILLEMET)
                                  |
   +------------------------------+-------------------------------+
   |                              |                               |
VERTICAL BUs                  HORIZONTAL (HOR)              Delivery
   |                              |                          (cross-BU)
   |                              |
   +-- Wallet XP tribe (WXP)      +-- Coin Framework / Coin Integration
   |    |                         |     (coin-modules, AccountBridge)
   |    +-- Live Hub squad        |     @LedgerHQ/coin-integration
   |    |    @LedgerHQ/live-hub   |
   |    +-- Devices Experience    +-- Platform Engineering (Vercel, GH, runners)
   |    |    @LedgerHQ/live-devices
   |    +-- Wallet API            +-- Managed Services (deploy, monitor)
   |    |    @LedgerHQ/team-wallet-api
   |    +-- Wallet CI team        +-- Security Engineering (SecEng, Donjon)
   |         @LedgerHQ/team-wallet-ci
   |
   +-- Consumer Services tribe (PTX)
   |    +-- Buy / Sell  (#ptx-buy-sell)
   |    +-- Swap        (#ptx-swap-prod)
   |    +-- Earn / Stake
   |    +-- Recover
   |    +-- B2C Backend Services (CVS, AppStore, coinradar)
   |    +-- Cloud Wallet (ALPACA service)
   |    +-- Dedicated SRE for PTX
   |
   +-- Engagement team  (WVEU, Activation-48h KPIs)
   |
   +-- Vault / Enterprise (B2B)  (HSM, vault-front, les-multisig, minivault)
   |
                                              +-- QA Automation (QAA)   <-- YOU
                                              |     (SDETs, E2E frameworks,
Quality Assurance org ------------------------+     Speculos, Allure, nightlies)
                                              +-- QA B2C   (manual LL/Wallet)
                                              +-- QA B2B   (manual Vault/HSM)
```

The seven boxes you will interact with every single week are: **Live Hub**, **Devices Experience**, **PTX**, **Engagement**, **Live Apps / Wallet API**, **Coin Integration**, and **Wallet CI**. Each owns a different class of failure you will see in nightly reports. The rest of this chapter walks through them in that order. (R1)

> **Caveat — this map is a snapshot.** The canonical Confluence page `[TECH] Teams Organization` (page 4233200470) has been flagged "under rework" since 03/2024. People move, Wallet API was partially folded into Coin Integration around 2024, and Delivery's cross-BU lead (Raja Faddoul) is now deactivated. Use this chart as scaffolding; confirm names in Slack before escalating. (R1)

### 0.3.2 Wallet XP tribe — the UI owners

**Wallet XP (WXP)** — "Wallet Experience" — is the tribe that builds the Ledger Live Desktop (LLD) and Mobile (LLM) user interfaces. Every time a Playwright test clicks a button or a Detox test scrolls a list, you are exercising WXP code. WXP's stated mission: "Build the best Wallet out there to manage our users' assets." (R1)

WXP is split into three squads, of which two matter most to QAA day-to-day:

**Live Hub squad** — GitHub team `@LedgerHQ/live-hub`.

- **Owns:** Portfolio screen, Accounts list, Send/Receive flows, WalletSync (the multi-device sync feature), Market screen, Settings, Braze-backed analytics surfaces, UI libs (`libs/ui/`, `react-ui`, `native-ui`).
- **Why QAA cares:** most desktop and mobile E2E specs under `e2e/desktop/tests/specs/` and `e2e/mobile/` drive Live Hub's UI. A UI refactor or a renamed `data-testid` on a Live Hub screen breaks dozens of tests at once.
- **Slack:** `#live-hub`, `#live-engineering`.
- **Typical ping:** "Hi Live Hub, the portfolio chart disappears after WalletSync — repro in LWD 2.139 nightly, Allure link below." (R1)

**Devices Experience squad** — GitHub team `@LedgerHQ/live-devices` (and `@LedgerHQ/android` for Android specifics).

- **Owns:** Hardware-device connectivity (`@ledgerhq/hw-transport*`), Device Management Kit (DMK), My Ledger screen (firmware + app management), Metamask Mobile integration, device onboarding flow.
- **Packages you will touch:** `@ledgerhq/hw-transport`, `@ledgerhq/hw-transport-node-speculos`, `@ledgerhq/hw-transport-node-speculos-http`, `@ledgerhq/device-sdk-core`, `@ledgerhq/device-management-kit`. These live under `libs/ledgerjs/packages/` and `libs/device-sdk/` in the monorepo.
- **Why QAA cares:** anything that touches Speculos through the transport layer — APDU errors, connectivity drops in the middle of a signing flow, firmware-update flows — lands here. The `hw-transport-node-speculos-http` package in particular is the bridge between your Playwright test and the Speculos Docker container; a breaking change in that package is a QAA-wide event.
- **Slack:** `#team-devices`, `#squad-devices`, `#squad-devices-devs`.
- **Typical ping:** "LWM Nightly regressed on `nano_usb_firmware_update` — the hw-transport USB handler looks changed in commit X." (R1)

**Wallet API / Connectivity** — GitHub team `@LedgerHQ/team-wallet-api`.

- **Owns:** APIs used by every LL-supported blockchain, the Discover section surfacing Live Apps, the WebPTXPlayer runtime.
- **Status note:** The "Ask other teams" page (2024-05) still lists Wallet API as its own squad, but the Coin Integration onboarding page says "the scope of this team is now under coin-integration". Anthony Goussot — historically Wallet API lead — is now listed under Coin Integration. Treat the boundary as fuzzy and cross-check in Slack. (R1)
- **Slack:** `#team-wallet-api`.

### 0.3.3 PTX tribe — money flows

**PTX** — **P**ayments & **T**ransactions Services — is the tribe that builds the financial-action Live Apps: **Buy**, **Sell**, **Swap**, **Earn/Stake**, and **Recover**. Each is delivered as a Live App embedded in LL with its own repo and its own release cadence — PTX ships independently of the dual train for the core wallet.

> **Important — PTX is NOT a Jira project key.** When you see a Jira ticket prefixed `PTX-1234` you are looking at a different project (older Product-Tech cross-team board). Actual PTX engineering tickets live under per-app keys (e.g., `SWAP-*`, `BUY-*`, `EARN-*`) or land in `LIVE-*` when the work touches the host app. Don't fat-finger a `PTX-*` ticket at a swap dev and expect action. (R4)

PTX's typical contact points for QAA:

- **JF Rochet, Sandra Denize Klein** — recurring attendees in PTX-adjacent triage meetings.
- **Slack:** `#ptx-buy-sell`, `#alert-buy`, `#sentry-ptx-buy-sell`, `#ptx-swap-prod`, per-partner `#partner-<name>` channels (Coinbase, MoonPay, etc.).
- **Dedicated SRE:** Anthony Pham, Michal Kaczmarek — an SRE pair embedded with PTX because partner incidents hit user wallets directly.

QAA maintains a dedicated Confluence page — "Debug live app locally" (6704562204) — specifically because PTX live apps require a special local-dev handshake to test against LLD/LLM. (R1)

### 0.3.4 Engagement tribe — growth surfaces

The **Engagement team** owns user-engagement features in LL. Their North Star metrics are **WVEU** (Weekly Value Engaged Users) and **Activation within 48h**. They contribute code to the main `ledger-live` monorepo under both desktop and mobile apps — so their PRs trigger QAA regression just like WXP's do. (R1)

Concrete features they own:

- Favourite Token / favourites list
- The staking digest UX
- Partial ownership of Buy/Sell/Swap/Stake entry points (the "upsell" surfaces)
- Onboarding variations and growth experiments

Tickets with the `[ENGAGEMENT]` tag in Jira (e.g., `LIVE-29156` — "[ENGAGEMENT] [LWM] Staking digest: data hook") land directly in `ledger-live`. Engagement does not have a well-documented Slack channel; when in doubt, ask in `#live-engineering`. (R1)

### 0.3.5 Live Apps team — the Discover catalog

Separate from PTX — though the boundary is blurry — is the **Live Apps** ownership. This covers:

- The **Discover catalog** inside LL (`Catalog/index.tsx` on both `ledger-live-desktop` and `ledger-live-mobile`).
- The **manifest infrastructure** — the `manifest-api` repo, which holds the JSON manifest for every first-party and third-party Live App (URL, scopes, geo-blocking rules).
- The **WebPTXPlayer / WebPlayers** runtime that embeds a Live App inside LL as a sandboxed webview.

Most first-party Live Apps are owned by PTX (Buy/Sell/Swap/Earn) or by Engagement (growth surfaces). Third-party Live Apps are maintained by partners but the manifest and the catalog slot belong to the Live Apps owners. A manifest change or a geo-blocking rule change breaks QAA's live-app E2E tests without any visible PR in the host app. (R1)

### 0.3.6 Coin Framework / Coin Integration — the family owners

**Coin Framework** (CF) — sometimes called **Coin Integration** — is the horizontal team that owns the per-blockchain code. GitHub team: `@LedgerHQ/coin-framework` (plus per-family sub-teams).

The structure:

- `libs/coin-framework` — the shared abstractions: `AccountBridge`, `CurrencyBridge`, transaction status, signing flow.
- `libs/coin-modules/coin-*` — one directory per blockchain family: `coin-btc`, `coin-evm`, `coin-xrp`, `coin-sol`, `coin-cosmos`, `coin-near`, and so on (roughly 29 families as of April 2026).

**Alpaca migration — work in progress.** Ledger is migrating the coin modules from an in-process TypeScript bridge to a hosted REST service called **Alpaca** (owned by the Cloud Wallet team under Consumer Services). ADR-025 ("Coin modules migration to Alpaca", Confluence page 6949896196) states "CODEOWNERS are kept identical to what we have in ledger-live monorepo." As of April 2026 the migration is incremental — 4 of the 29 families have been touched, and only **xrp** has been fully extracted into the Alpaca repo. The rest still live in-process inside `libs/coin-modules/`. (R6)

Why QAA cares: the bot suites (`nitrogen`, `oxygen`) and the coin-tester harness that QAA monitors are driven by coin-modules. A broken `AccountBridge` method in `coin-evm` will cascade into every EVM chain's nightly. When a bot run goes red for a specific family, Coin Integration is your first ping — `#team-coin-integration`. (R1, R6)

**AlpacaApi — the migration target.** The Alpaca service exposes a standard HTTP surface so the app no longer needs per-family bridge code baked in. The method set (per ADR-022 / ADR-023) includes `broadcast`, `combine`, `estimateFees`, plus family-specific extensions. Each migrated family implements a `create{Coin}Api` registered in `getAlpacaApi`, and exposes an `AlpacaSigner` from `getSigner`. For QAA, this means the test path for a migrated family runs through one extra HTTP hop — the Alpaca service — which adds a failure mode (Alpaca service down) that did not exist in the in-process world. When xrp (the one fully-migrated family as of April 2026) regresses in nightly, check Alpaca service health before blaming Coin Framework code. (R6)

### 0.3.7 Wallet CI — the pipelines you depend on

**Wallet CI** — GitHub team `@LedgerHQ/team-wallet-ci` — is the team that owns the continuous-integration pipelines for both `ledger-live` and `ledger-live-build`. Their KPI is literally **releases per month**. (R1)

- **Owns:** every `.github/workflows/*.yml` file in the monorepo, the GitHub Actions runner pool (self-hosted runners plus gha-runners managed by Platform Engineering), the release-orchestration workflows (`release-create.yml`, `release-create-hotfix.yml`, `release-rollback-desktop.yml`), the CI monitoring dashboards.
- **Slack channels** (you will lurk in most of these):
  - `#team-wallet-ci` — primary
  - `#wallet-ci-alerts` — CI failures, flakes, runner issues (QAA watches this one closely)
  - `#live-ci-notifications` — workflow pass/fail notifications
  - `#ledger-live-ci` — broader CI discussion
  - `#live-release-sync` — release-train coordination
  - `#infra-live` — infrastructure-side

**Why QAA cares more than any other team:** when a runner pool is down, or the merge queue is flaky, or a release workflow is stuck, QAA's nightlies are the first loud consumer. You will tag `@qa-automation` and cross-tag `#team-wallet-ci` when infra issues block the nightly signal. (R1, R5)

### 0.3.8 QAA — your team

The **QA Automation (QAA) team** is where you sit. You are part of the Quality Assurance organisation, alongside QA B2C (manual consumer QA) and QA B2B (manual Vault/HSM QA). QAA's charter: provide the **horizontal automation backbone** — Playwright for desktop, Detox for mobile, Speculos plumbing, Allure reporting, nightly orchestration — for all three pillars (LWD, LWM, Vault). (R5)

The key people you will meet in your first two weeks — mapped to what they teach:

| Name | Role on QAA | What they teach you | First meeting |
|---|---|---|---|
| **Yaroslava POLISHCHUK** | Lead / primary onboarding buddy | Team, processes, Slack + Firebase access, meeting invites | Day 1 |
| **Victor ALBER** | LWD automation trainer | Playwright + Speculos desktop framework, page objects | First 2 weeks |
| **Abdurrahman SASTIM** | LWM automation trainer | Detox mobile framework, iOS/Android simulator pool | First 2 weeks |
| **Oleksandra BOYKO** | Vault automation trainer | `vault-e2e-tests`, `vault-public-api-tests` | First month (if B2B scope) |
| **Gabriel BECERRA** | Regression walkthrough | How the release regression set is built and run | First month |
| **Laure DUCHEMIN** | QA locker / devices | Physical Nano device hand-off for regression runs | Day 1 |

(R1, R5)

QAA's GitHub team handle is reconstructed as `@LedgerHQ/qaa` (confirmed via `.github/workflows/test-ui-e2e-only-desktop.yml` CODEOWNERS — `@ledgerhq/wallet-ci @ledgerhq/qaa`). Vault-specific repos (`vault-e2e-tests`, `vault-public-api-tests`) are owned by QAA members embedded in B2B. (R1)

### 0.3.9 CODEOWNERS conventions

Every repo uses a `CODEOWNERS` file (either at repo root or under `.github/CODEOWNERS`) to auto-request reviewers on PRs. The pattern `path owner` means: any PR touching `path` must be reviewed by `owner`. Last matching pattern wins, which makes the file order-sensitive.

Here is a real extract from `LedgerHQ/ledger-live/CODEOWNERS` — the patterns that matter most for QAA work:

```gitignore
# CI team
.github                                                 @ledgerhq/wallet-ci
.github/workflows/test-ui-e2e-only-desktop.yml          @ledgerhq/wallet-ci  @ledgerhq/qaa
.github/workflows/test-mobile-e2e-reusable.yml          @ledgerhq/wallet-ci  @ledgerhq/qaa
.github/workflows/test-design-system-reusable.yml       @ledgerhq/wallet-ci  @ledgerhq/wallet-xp
tools                                                   @ledgerhq/wallet-ci

# Platform & Wallet XP catchall
apps/ledger-live-desktop/                                @ledgerhq/platform
apps/ledger-live-mobile/                                 @ledgerhq/platform
apps/ledger-live-desktop/src/                            @ledgerhq/wallet-xp
apps/ledger-live-mobile/src/                             @ledgerhq/wallet-xp

# Wallet — libs
libs/ui/                                                 @ledgerhq/wallet-xp
libs/live-countervalues/                                 @ledgerhq/wallet-xp
libs/live-wallet/                                        @ledgerhq/wallet-xp
libs/ledger-live-common/src/featureFlags/                @ledgerhq/platform
libs/ledger-live-common/src/featureFlags/walletFeaturesConfig/ @ledgerhq/wallet-xp

# Coin integration
apps/**/families/                                                   @ledgerhq/coin-integration
libs/coin-*/                                                        @ledgerhq/coin-integration
libs/ledger-live-common/src/bot/                                    @ledgerhq/coin-integration
libs/ledger-live-common/src/bridge/                                 @ledgerhq/coin-integration
libs/ledger-live-common/src/families/                               @ledgerhq/coin-integration
.github/workflows/test-coin-modules-integ.yml                       @ledgerhq/coin-integration
.github/workflows/test-bridge-pr.yml                                @ledgerhq/coin-integration @ledgerhq/wallet-ci

# PTX team
apps/ledger-live-desktop/src/renderer/screens/swapWeb/              @ledgerhq/ptx
apps/ledger-live-desktop/src/renderer/screens/exchange/             @ledgerhq/ptx
apps/ledger-live-desktop/src/renderer/screens/earn/                 @ledgerhq/ptx
apps/ledger-live-desktop/src/renderer/components/WebPTXPlayer/      @ledgerhq/ptx
apps/ledger-live-mobile/src/screens/Swap/                           @ledgerhq/ptx
libs/ledger-live-common/src/exchange/                               @ledgerhq/ptx
libs/exchange-module/                                               @ledgerhq/ptx

# Cryptoassets — split ownership (mid-rework)
libs/ledgerjs/packages/cryptoassets                                 @ledgerhq/wallet-xp
libs/ledgerjs/packages/cryptoassets/src/currencies.ts               @ledgerhq/coin-integration

# Partner-owned coin modules (external + internal co-ownership)
libs/coin-modules/coin-aptos/     @ledgerhq/blockchain-support @ledgerhq/ledger-third-party-leads
libs/coin-modules/coin-cardano/   @ledgerhq/blockchain-support @ledgerhq/ledger-third-party-leads
libs/coin-modules/coin-sui/       @ledgerhq/ledger-partner-hoodies @ledgerhq/ledger-third-party-leads
libs/coin-modules/coin-hedera/    @ledgerhq/ledger-partner-blockydevs @ledgerhq/ledger-third-party-leads
```

Read this carefully. Notice five things:

1. **QAA co-owns the E2E workflows** — any change to `test-ui-e2e-only-desktop.yml` or `test-mobile-e2e-reusable.yml` requires a QAA review. That is why you will get pinged on PRs from Wallet CI that touch those files.
2. **Last match wins.** `apps/ledger-live-desktop/` is owned by `@ledgerhq/platform`, but `apps/ledger-live-desktop/src/` (one level deeper) is owned by `@ledgerhq/wallet-xp`. A PR that modifies only `src/` will request WXP, not Platform.
3. **Coin families are carved out.** The `apps/**/families/` glob gives `@ledgerhq/coin-integration` ownership of per-family code inside both desktop and mobile apps, overriding the WXP catch-all. This is important when your PR touches a family-specific testId — CI will request Coin Integration, not WXP.
4. **PTX owns the exchange surfaces.** `screens/swapWeb/`, `screens/exchange/`, `screens/earn/`, the `WebPTXPlayer/` component, and `libs/exchange-module/` all belong to PTX. When a Swap test breaks, the code change is almost always under one of these paths.
5. **Partner-owned families exist.** Some coin families (Aptos, Cardano, Sui, Hedera, etc.) have **dual ownership** — a Ledger internal team (`@ledgerhq/blockchain-support` or a named partner-liaison team) plus an external partner team. When a family-specific test regresses because of a partner-side change, you cannot simply raise a Jira ticket and wait — you must go via the partner-liaison channel. Confirm the right route in `#team-blockchain-support` before pinging.

> **Practical drill.** Before a tricky PR, try this: `git diff --name-only develop...HEAD | xargs -I{} grep "{}" CODEOWNERS`. This surfaces which CODEOWNERS lines match your diff — but remember GitHub applies them top-to-bottom with last-match-wins, so order matters. Use the drill to spot surprises, then walk the actual precedence manually.

> **Practical rule.** Before opening a PR, run `git diff --name-only develop...HEAD` and mentally walk it through the `CODEOWNERS` rules. If you touched a family directory, expect a Coin Integration review — plan for 24-48h to get that approval. If you touched only `e2e/` files, only QAA will be requested and you can move fast.

### 0.3.10 Slack channels & meeting cadence

Your ambient awareness of what is happening across the org comes almost entirely from Slack and a handful of recurring meetings. QAA's cadence: (R5)

**Daily:**

- **Standup (Scrum Daily)** — 15 minutes, devs + QA + PO + DM attend. You listen; occasionally ask.
- **Firefighting rotation** — a 10:15 daily meeting during your rotation week (mandatory when rotating). QAA rotates through this with the rest of the engineering floor.
- **12:00 Triage — the Golden Rule.** By noon every working day, no nightly failure may be "silent." Every failure must have an owner, a direction, and visibility in Slack. This is QAA's single most important daily ritual. You will triage results for your scope (LWD, LWM, or Vault) and post in `#live-repo-health` with owners tagged. If you are out, delegation is mandatory. (R5)
- **Nightly triage thread** — `#live-repo-health` has one thread per nightly run. You read it, classify failures (bug / product-change / flake / infra / explorer 5xx / funds), and reply with actions.

**Weekly:**

- **QA LES Weekly** — QAA-wide weekly. Standing agenda: Announcements → Firefighting recap → Round-table across QAA members (Victor, Abdu, Oleksandra) → Features & projects tracker.
- **Pre-Release sync** — 30 minutes, QA + Delivery Manager, reviews the upcoming release train's QA status.
- **1:1 with Yaroslava** — first month, weekly; tapers to bi-weekly.
- **Roadmap Sync** — 30 minutes, PM + DM + Team Lead. You attend selectively.

**Bi-weekly (sprint cadence, 2 weeks):**

- **Bug Triage** — 30 minutes, QA + PM + Team Lead, Jira-driven. Important if you own open bugs.
- **Sprint Review (Demo) + Retrospective** — 1 hour at end of each sprint.
- **BU Demo** — 1 hour across all software teams.

**Quarterly:**

- **Regression scope refinement** — the quarterly meeting where QA-B2C selects which manual test cases move into the regression set, and QAA commits to automating a subset.

**The per-rotation firefighting role.** At Ledger, a rotating "firefighter" role cycles across engineering squads week by week. When you are on rotation, the 10:15 daily FF meeting is mandatory, and you own triage of urgent production issues that affect your scope for the week. QAA participates in this rotation alongside feature teams — expect your first rotation roughly 3-6 months into the role, once you have enough product surface knowledge to classify incoming incidents. Before that, you shadow whoever is rotating from QAA that week. (R5)

**Slack channels to join on day one:**

- `#live-repo-health` — the nightly triage thread lives here. Non-negotiable.
- `#qa-automation` — the QAA internal channel (exact name confirmed with Yaroslava on day one; this is the canonical ping target).
- `#live-engineering` — the tribe-wide engineering stream. Ambient awareness of what WXP and Devices are doing.
- `#team-wallet-ci`, `#wallet-ci-alerts` — CI health. You will lurk here.
- `#live-releases-org`, `#live-release-sync` — release-train coordination. Read-only unless you are a Train QA.
- `#team-devices`, `#live-hub` — closest tribe neighbours.
- `#team-coin-integration` — for Coin Framework / Alpaca questions.
- `#ptx-buy-sell`, `#ptx-swap-prod` — only when a PTX test goes red.
- `#explorer-users` — to escalate Explorer 5xx / blockchain-node issues.
- `#qa-b2c-releases-bugs-tracking` — prod feature-flag change monitor (watches `LedgerHQ/qa-b2c-release-watcher`).
- `#live-prerelease-sentry-alerts` — Sentry alerts for prerelease builds. Noisy but useful during Phase 2.
- `#cs-quality-assurance` — customer-success cross-over; CSMs raise product issues here.
- `#launchpad` — cross-team coordination for major launches.

> **Quick reference — what channel for what failure.** When a nightly goes red, the routing is almost mechanical:
>
> - UI regression on a Live Hub screen → `#live-hub` + tag `@live-hub-oncall`
> - Transport / device-connectivity regression → `#team-devices`
> - Swap / Buy-Sell / Earn regression → `#ptx-swap-prod` or `#ptx-buy-sell`, cross-tag the partner channel if partner-specific
> - Coin-family regression (BTC, ETH, SOL, XRP, etc.) → `#team-coin-integration` (+ `#team-blockchain-support` for externally-owned families)
> - Runner / workflow / merge-queue issue → `#wallet-ci-alerts`
> - Explorer 5xx → `#explorer-users`
> - Speculos seed out of funds → `#live-repo-health` + tag `@qa-automation`
> - Flaky but not reproducible → Allure "Report Flaky" button (auto-issues a GitHub ticket)
>
> Save this mapping — it is the single most-used piece of QAA tribal knowledge.

### 0.3.11 Train roles — the rotating release trio

For every release train, three rotating roles coordinate the cut: (R3)

- **Train Delivery** — triggers `release-create.yml`, opens the "Release …" PR, owns the calendar.
- **Train Engineer** — technical owner of the release branch, merges blocker fixes, handles the merge-to-main / merge-to-develop dance, runs the Quorum signing step on LLD and the store-submission step on LLM.
- **Train QA** — posts the GO/NO-GO in `#live-releases-org` based on the Phase 2-3 QA campaign, owns the acceptance of the prerelease build, coordinates re-tests after blocker fixes.

As a new QAA hire, you will **not** be a Train QA in your first months — the rotation is owned by the B2C manual QA team with QAA support. Your job in the train is to keep the E2E nightly green against the `release` branch, feed the Train QA real data on automation-covered regressions, and help classify any failure that falls into the "product change vs automation-broke" bucket. (R3, R5)

What the Train QA actually looks at to post GO:

- **Manual regression run-through** on a physical Nano — roughly 200 B2C test cases for LWD+LWM combined, plus ~400 for Vault on the B2B side. Owned by B2C QA, supported by QAA on test-case definition and Xray automation links.
- **Automation nightly signal** from QAA — the Allure report for the last nightly against the `release` branch. Must be honest: classified failures, no silent retries, flakes tagged via the "Report Flaky" Chrome extension.
- **Xray coverage summary** — Jira dashboard 10427 showing automation-test-coverage; any gap versus the regression plan is called out.
- **Feature-flag audit** — values in `ledger-live-production` Remote Config are reviewed in Phase 3. Any flag that is "on" in prod and "off" in staging is flagged for discussion; the reverse is rarer but happens.

Your daily contribution during an active train is concrete: classify every nightly failure by noon, post a triage summary in `#live-repo-health`, and raise Jira tickets (with the `qaa` label, logs, videos, and an Allure link) for any real defects. If you do this cleanly for two weeks, the Train QA will start citing your nightly summary verbatim — which is the point. (R3, R5)

<div class="chapter-outro">
<strong>Key takeaway:</strong> QAA does not own a product surface — it owns a <em>capability</em> (automation) that every other team consumes. Your effectiveness is measured by how well you can route failures to the right owning team: WXP for UI breaks, Devices for transport, Coin Integration for family-specific bugs, PTX for live-app regressions, Wallet CI for infra, Engagement for growth-surface weirdness. Spend your first two weeks lurking in the Slack channels above before you start posting.
</div>

### 0.3.12 Quiz — Teams & Ownership

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Teams & Ownership</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> A nightly LWD test that clicks the Portfolio's "Add Account" button starts failing because the <code>data-testid</code> was renamed. Which team owns the fix?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>@ledgerhq/coin-integration</code> — accounts are coin-specific</button>
<button class="quiz-choice" data-value="B">B) <code>@ledgerhq/team-wallet-ci</code> — CI is red</button>
<button class="quiz-choice" data-value="C">C) <code>@ledgerhq/live-hub</code> (Live Hub squad, Wallet XP) — they own Portfolio, Accounts, Send/Receive</button>
<button class="quiz-choice" data-value="D">D) <code>@ledgerhq/qaa</code> — we should adapt the test</button>
</div>
<p class="quiz-explanation">Live Hub owns Portfolio, Accounts, Send/Receive, WalletSync, Market, Settings. When a <code>data-testid</code> disappears without warning, that is a product-change broken-testability issue — QAA's doctrine is to <em>not</em> adapt tests around unstable selectors but to ask Live Hub to restore the testId. Coin Integration only owns per-family code under <code>libs/coin-*/</code> and <code>apps/**/families/</code>.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> You see a Jira ticket prefixed <code>PTX-1847</code>. What should you assume?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It is a Swap bug — PTX owns Swap</button>
<button class="quiz-choice" data-value="B">B) <code>PTX-*</code> is a separate (older) Jira project key, not the PTX tribe's engineering board — check context before routing</button>
<button class="quiz-choice" data-value="C">C) It is a Buy/Sell partner ticket</button>
<button class="quiz-choice" data-value="D">D) It is automatically routed to the swap squad by Jira</button>
</div>
<p class="quiz-explanation"><code>PTX</code> is NOT the Jira project key for the PTX tribe's engineering work. PTX engineering tickets live under per-app keys (<code>SWAP-*</code>, <code>BUY-*</code>, <code>EARN-*</code>) or in <code>LIVE-*</code> when they touch the host app. The <code>PTX-*</code> prefix belongs to an older cross-functional board. Confirm in Slack before pinging a swap engineer with a wrong ticket number.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> You are triaging a nightly failure at 11:30 a.m. and the owning team has not responded yet. What is the Golden Rule?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) By 12:00 every failure must have an owner, a direction, and visibility in Slack — push the ping, don't wait</button>
<button class="quiz-choice" data-value="B">B) Wait until 14:00 for a response; silence can mean work in progress</button>
<button class="quiz-choice" data-value="C">C) Retry the failing test; if it passes twice, close the triage</button>
<button class="quiz-choice" data-value="D">D) Escalate to the CTO</button>
</div>
<p class="quiz-explanation">The 12:00 Triage is QAA's single most important daily ritual. Every failure must be classified and routed before noon, even if the routing is "I'm blocked on Live Hub's answer — here is the Slack thread." Silent failures erode trust in the nightly signal faster than any single flaky test ever could. Never re-run tests to make a failure disappear — that is explicitly against QAA doctrine.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> A PR modifies <code>apps/ledger-live-desktop/src/renderer/families/ethereum/send.tsx</code>. Which team will CODEOWNERS request as reviewer?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>@ledgerhq/wallet-xp</code> — it is under <code>apps/ledger-live-desktop/src/</code></button>
<button class="quiz-choice" data-value="B">B) <code>@ledgerhq/platform</code> — it is under <code>apps/ledger-live-desktop/</code></button>
<button class="quiz-choice" data-value="C">C) <code>@ledgerhq/qaa</code> — anything E2E-adjacent</button>
<button class="quiz-choice" data-value="D">D) <code>@ledgerhq/coin-integration</code> — the <code>apps/**/families/</code> glob overrides the WXP and Platform catch-alls (last match wins)</button>
</div>
<p class="quiz-explanation">CODEOWNERS uses "last match wins" precedence. The file lists both <code>apps/ledger-live-desktop/src/</code> (WXP) and <code>apps/**/families/</code> (Coin Integration). Because <code>apps/**/families/</code> appears later and matches the path, Coin Integration is requested — not WXP. Always walk your diff through the rules before opening the PR so you can plan for the right approval timeline.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> Which of the following is the correct status of the Alpaca migration as of April 2026?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) All 29 coin families have been migrated to Alpaca; the <code>libs/coin-modules</code> directory is deprecated</button>
<button class="quiz-choice" data-value="B">B) Migration is incremental — roughly 4 of 29 families have been touched, and only xrp has been fully extracted into the Alpaca repo</button>
<button class="quiz-choice" data-value="C">C) Alpaca was cancelled in 2025 and coin modules remain in-process</button>
<button class="quiz-choice" data-value="D">D) Only EVM chains have been migrated</button>
</div>
<p class="quiz-explanation">Per R6 and ADR-025, the migration is deliberately incremental. Four of twenty-nine families have seen Alpaca work, but only <strong>xrp</strong> has been fully extracted as of April 2026. The rest still run in-process inside <code>libs/coin-modules/</code>. CODEOWNERS are "kept identical" between the two repos so that coin-integration reviews both sides.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> Who is your primary onboarding buddy — the person who sets up your Firebase access, Slack invites, and recurring meeting invitations?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Victor ALBER — he trains LWD automation</button>
<button class="quiz-choice" data-value="B">B) Abdurrahman SASTIM — he trains LWM automation</button>
<button class="quiz-choice" data-value="C">C) Yaroslava POLISHCHUK — QAA lead and primary onboarding buddy</button>
<button class="quiz-choice" data-value="D">D) Laure DUCHEMIN — she hands out test devices from the locker</button>
</div>
<p class="quiz-explanation">Yaroslava POLISHCHUK is the QAA lead and the primary onboarding buddy per the QAA Onboarding Checklist. Non-ticket access (Firebase, Slack channels, recurring meeting invites) is explicitly granted by her. Victor, Abdurrahman, and Oleksandra are your training pairs for LWD, LWM, and Vault respectively. Laure owns the physical device locker.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## The Release Cycle & Environments

<div class="chapter-intro">
Ledger Live does not ship once a quarter — it ships on a <strong>dual train</strong> that runs two parallel tracks (LWD and LWM), with a hotfix lane in the wings for blocker fixes. As a QAA engineer, you are not a passive observer of the train: your nightly signal <em>is</em> the Phase 3 QA gate. When QAA says "no-go", the release slips. This chapter covers the cadence, the phases, the Firebase and <code>.env</code> environment matrix, the feature-flag system you will override daily in E2E, and the hotfix lane. By the end, you will read a release-page checklist the way a Train Engineer does. (R3)
</div>

### 0.4.1 The Dual-Train Model

There is one release process but **two trains** running in parallel:

- **LWD** — Ledger Wallet Desktop (formerly Ledger Live Desktop, LLD). Versions numbered `2.x.y` — e.g., LWD 2.137, LWD 2.139.
- **LWM** — Ledger Wallet Mobile (formerly Ledger Live Mobile, LLM). Versions numbered `3.x.y` — e.g., LWM 3.101, LWM 3.103.

> **Name migration.** The project is in the middle of a rename from "Ledger Live" to "Ledger Wallet." You will see both prefixes interchangeably — `LLD` / `LWD`, `LLM` / `LWM`. They refer to the same apps. Newer release pages use `LWD` / `LWM`; older ones use `LLD` / `LLM`. The monorepo directories are still named `apps/ledger-live-desktop` and `apps/ledger-live-mobile`. (R3)

The two trains are **independent** in the sense that each has its own version number, its own release PR, and can ship on its own. But they share **everything** underneath:

- The same monorepo (`LedgerHQ/ledger-live`).
- The same CI pipeline (owned by Wallet CI).
- The same Firebase projects (for Remote Config and feature flags).
- The same release-creation workflows (`release-create.yml`, `release-create-hotfix.yml`).
- The same Train Delivery / Train Engineer / Train QA trio.

The canonical case is a **dual release** — one train cut produces both an LWM build and an LWD build at the same cadence. The hotfix-procedure page documents the reference pair: "release N produced LLM v3.30.0 and LLD v2.26.0." But **single-app releases do occur** — an LWM-only train, or an LWD-only hotfix lane — when only one platform has changes worth shipping. (R3)

Cadence observed from release-page numbering: recent trains include LWM 3.99 → 3.101 → 3.103 on mobile and LWD 2.137 → 2.139 on desktop. The exact number of weeks between trains is not stated in any Confluence page — the live cadence sheet is an external Google Sheet linked from every release page. Treat the cadence as roughly weekly-to-bi-weekly and confirm by looking at the last two release pages in Confluence. (R3)

> **Orientation tip.** When someone says "the 3.103 train" they mean "the LWM release numbered 3.103 and its paired LWD release (likely 2.138 or similar)." You will navigate between release pages by searching Confluence for `LWM 3.NNN` or `LWD 2.NNN` — each page is self-contained and follows the same phase structure, so once you read one you have read them all. (R3)

### 0.4.2 Phases 0 through 5

A single release train flows through six numbered phases. The LWD 2.137 and LWM 3.103 release checklists (R3 sources 2 and 3) both use the same phase schema. As QAA, you care most about Phase 3 — that is your gate. (R3)

| Phase | Name | Who drives | Firebase env | What happens | QAA involvement |
|---|---|---|---|---|---|
| 0 | Prep | Train Delivery | `development` | Calendar locked, scope frozen, release pages created in Confluence. | Read the release page; know what is shipping. |
| 1 | Prerelease Launch | Train Delivery + Train Engineer | `staging` | `release-create.yml` triggered. `release/*` branch forked from `develop`. "Release …" PR opened. Builds land in `ledger-live-build` as prereleases. Smartling translation jobs kick off (Wed 5pm cutoff). Sentry starts watching the prerelease project. | Point nightly at the `release` branch. Start classifying failures. |
| 2 | Release Test Campaign (QA) | Train QA + QAA | `staging` | Manual regression + automation nightlies run against the release branch. Blocker fixes land on `release` **and** are cherry-picked to `develop`. | Classify failures, raise bugs, track ETAs. This is your busiest phase. |
| 3 | GO for Release | Train QA posts GO | `staging` | Train QA posts GO in `#live-releases-org`. POs acknowledge. Feature-flag values in `ledger-live-production` are audited before GO. | **This is QAA's gate.** No-go from nightlies means release slips. (R3, R5) |
| 4 | Merge & Sign | Train Engineer | `production` | `release` → `main` AND `release` → `develop` (no squash, both directions). Exit prerelease mode. Create prod builds in `ledger-live-build`. Publish npm packages. LWD: Quorum signing. LWM: Play Console (Android) + App Store (iOS) submission. | Observe — this is Engineering's phase. |
| 5 | Post-Release Monitoring | Train QA + QAA | `production` | LWD: CDN publish. LWM: Android 10% staged rollout → 100%, iOS 100%. Sentry monitoring on the prod project. Quick rollback if a blocker is found (triggers the hotfix lane instead). | Watch Sentry for regressions in early hours. |

**The key QAA fact:** Phase 3 is a **QA gate**. The Train QA's GO message is backed by (a) the manual regression result and (b) the automation nightly signal. If the nightlies are red with real product defects, there is no GO. Your job in Phases 1-3 is to make sure the signal is honest — classify flakes out, but do not sweep product defects under the rug by re-running until green. (R3, R5)

#### The release-train diagram

Here is the end-to-end flow on a single page. Read it once top-to-bottom before you look at a real release page — it will make the Confluence checklists parse instantly. (R3)

```
                         LEDGER LIVE RELEASE TRAIN (per release N)
                         ------------------------------------------

  +---------------+  dev work                       [Firebase: development]
  |   develop     | <-- feature branches               [env: .env / dev]
  |   (branch)    |     (feature flagged)               editable by devs
  +-------+-------+
          |
          |  Train Delivery triggers
          |  release-create.yml  (Phase 1)
          v
  +---------------+  Phase 1: Prerelease Launch        [Firebase: staging]
  |   release/*   |  - release/* branch forked          [env: .env.staging]
  |   (branch)    |    from develop                     read-only for devs
  |               |  - "Release ..." PR opened          editable by QAE/PM
  |               |  - Prerelease builds on
  |               |    ledger-live-build
  +-------+-------+  - Smartling jobs (Wed 5pm cutoff)
          |          - Sentry prerelease watching
          |
          |  Phase 2: Release Test Campaign (QA)
          |          - Manual regression (B2C QA)
          |          - Automation nightly (QAA) on release branch
          |          - Blocker fixes: PR -> release
          |            AND cherry-pick PR -> develop
          |          - Daily 12:00 triage, failures classified
          |
          |  Phase 3: GO for Release
          |          - Train QA posts GO in #live-releases-org
          |          - POs acknowledge
          |          - Production Remote Config audited
          v
  +---------------+  Phase 4: Merge & Sign           [Firebase: production]
  |     main      |  - release -> main  (NO SQUASH)    [env: .env.production]
  |   (branch)    |  - release -> develop (NO SQUASH)   read-only for devs
  |               |  - Exit prerelease mode             editable by PM/Delivery
  |               |  - Prod builds in ledger-live-build
  |               |  - npm publish (AFTER go, not during prerelease)
  |               |  - LWD: Quorum signing
  |               |  - LWM: Android Play Console / iOS App Store submission
  +-------+-------+
          |
          v
    +----------+   Phase 5: Post-Release Monitoring
    |   USERS  |   - LWD: Desktop Sign & Publish -> CDN
    +----------+   - LWM: Android 10% staged -> 100%; iOS 100%
                   - Sentry monitoring on prod
                   - If blocker found: trigger hotfix lane


  HOTFIX LANE (parallel, fork from main):

  +---------------+   release-create-hotfix.yml (manual)
  |     main      | ------------------------------------> +---------------+
  | (last prod)   |   Cherry-pick fix + PATCH changeset   |    hotfix     |
  +---------------+   NO SQUASH merges                    |   (branch)    |
                                                          +-------+-------+
                       Same Phases 2-4 as release, then:          |
                       hotfix -> main AND hotfix -> develop       |
                                                                  v
                                                          Users get patched build
```

Read this once, print it if you must, and keep it next to the release-page checklist on your first train. It is the mental model that unlocks every other Confluence doc. (R3)

A few details worth internalising from the diagram:

- **`develop` never freezes.** Feature work keeps landing on `develop` throughout a release train. When the release cuts, `release/*` forks from the current tip of `develop`, and any subsequent work on `develop` is NOT in that release — it will ship with the next train.
- **Blocker fixes land twice.** Every fix merged into `release` is also cherry-picked into `develop` so the bug does not regress on the next cut.
- **The double merge at Phase 4 is NOT a squash.** `release → main` and `release → develop` both preserve commit history so that Changesets' version-bump commits and the signed release tag line up deterministically. (R3)

### 0.4.3 Versioning — Changesets, not semantic-release

Ledger Live uses **Changesets** (`@changesets/cli`) — not Angular-style semantic-release — to version and publish packages. (R3)

> **Terminology warning.** If someone in onboarding docs or a wiki search says "semantic-release", they mean "the concept of versioning from commits." Ledger Live does not use the `semantic-release` npm tool. The actual tool is Changesets. Anyone searching the codebase for a `semantic-release` config will find nothing. (R3)

The mechanics:

- Every PR that affects a published package must include a **changeset file** — a Markdown file under `.changeset/*.md` that declares the bump type (`patch`, `minor`, `major`) and a human-readable description.
- Commit messages are lint-gated via `commitlint.yml` against the Conventional Commits spec. The commit type (`feat`, `fix`, `test`, `refactor`, etc.) drives nothing automatically — it is for humans. The changeset file drives the actual version bump.
- Hotfix PRs must include a **patch-only changeset** (never `minor`, never `major`).
- On release-branch merge, the Changesets CLI bumps package versions and publishes to npm — but **npm publishing now happens after GO-to-prod**, not during prerelease. It used to happen earlier; that was stopped after buggy prerelease packages leaked onto npm. (R3)

A minimal changeset file looks like this:

```markdown
---
"@ledgerhq/coin-ethereum": patch
"@ledgerhq/live-common": patch
---

Fix EIP-1559 gas-estimation overflow for extremely high base-fee scenarios.
```

You will create these by running `pnpm changeset` in the monorepo root, which opens an interactive wizard. As QAA, your PRs rarely need a changeset — `e2e/` files are not published — but any PR that touches `libs/` or `@ledgerhq/live-common` will need one.

### 0.4.4 The Firebase Environment Matrix

Ledger Live uses **four Firebase projects**, one per environment. Each one hosts its own Remote Config (feature flags), its own App Distribution channel, and its own analytics stream. (R3)

| Firebase project | Used by | Feature flag editor rights | Canonical consumer |
|---|---|---|---|
| `ledger-live-development` | Local dev builds, feature-branch PRs | Devs Editor, EM Editor, QAE Product Delivery Viewer | Developers on `develop` |
| `ledger-live-testing` | QA E2E suites, mock tests | QAE Editor, EM Editor, Devs Product Delivery Viewer | **QAA automation runs** |
| `ledger-live-staging` | Beta builds, prerelease QA campaign | All roles Editor (permissive) | Phase 1-3 of the train |
| `ledger-live-production` | Shipped prod LLD + LLM | Product Editor, EM + Delivery Editor backup, Devs + QAE Viewer | Phase 4-5 (shipped users) |

**Feature flags are NOT frozen on production.** The prod Remote Config is mutable at any time — a PM can flip a flag live. Changes are monitored (not blocked) in Slack `#qa-b2c-releases-bugs-tracking` via the `LedgerHQ/qa-b2c-release-watcher` GitHub repo. This is important: a test that passed on Monday can fail on Wednesday because someone flipped a flag in production Remote Config. The Remote Config fetch is cached ~12h by default on clients, which adds lag. (R3)

#### Mapping the env to the app

The Firebase project the app talks to is selected by an `.env` file chosen at build time. On desktop, this is resolved by `renderer.webpack.config.js:getDotenvPathFromEnv()` and `src/firebase-setup.js`:

- `.env` (default) → `ledger-live-development`
- `.env.testing` → `ledger-live-testing`
- `.env.staging` → `ledger-live-staging`
- `.env.production` → `ledger-live-production`

On Android it is build-variant driven via `android/app/google-services.json` (debug), `src/stagingRelease/google-services.json`, and `src/release/google-services.json`. On iOS, `GOOGLE_SERVICE_INFO_NAME` is set in the matching `.env.*` file and picked up in `AppDelegate.m` to select between `GoogleService-Info.plist`, `GoogleService-Info-Staging.plist`, and `GoogleService-Info-Production.plist`. (R3)

> **⚠ Callout — two coexisting `.env` namespaces.** You will see two families of `.env` files in the monorepo, and they are BOTH real. Do not confuse them:
>
> - **Release / runtime env files** — `.env`, `.env.testing`, `.env.staging`, `.env.production`. These select the **Firebase project** (and thus the feature-flag source) at app build time. Used for dev builds, prereleases, and prod.
> - **Mobile E2E Detox env files** — `.env.mock`, `.env.mock.prerelease`. These are specific to the Detox mock-test pipeline on mobile — they configure the mocked backends and canned fixtures the Detox suite uses. They are NOT Firebase-project selectors; they are a separate namespace.
>
> Both sets coexist in the repo. The release-env set is documented on the Feature Flagging Confluence page (R3). The Detox mock set is not documented there and must be discovered from the mobile repo structure. When a mobile dev says "use `.env.mock`", they mean the Detox namespace — not `.env.testing`. (R3)

#### Non-Firebase QAA-relevant env vars

From the QA space "Ledger Live environments" page, the knobs QAA flips most often are: (R3)

- `MOCK=1` — switch network layer to mocks (skip real blockchain calls).
- `DISABLE_TRANSACTION_BROADCAST=1` — run through the full sign flow but do not broadcast. **On CI, real broadcasts happen only on Monday nightly** to control costs; other runs keep this on.
- `SPECULOS_DEVICE=nanoSP` (or `nanoX`, `stax`, `flex`) — select the device model.
- `SPECULOS_IMAGE_TAG=ghcr.io/ledgerhq/speculos:master` — pin the emulator image.
- `SWAP_API_BASE=https://swap.staging.aws.ledger.fr` — point swap tests at staging.
- `ANALYTICS_LOGS=1`, `ANALYTICS_CONSOLE=1`, `DEV_TOOLS=1`, `DEBUG_THEME=1`, `DEBUG_UPDATE=1` — dev/debug surfaces.
- `VERBOSE=error|warn|info|http|verbose|debug|silly` — log verbosity.
- Per-currency overrides like `API_COSMOS_BLOCKCHAIN_EXPLORER_API_ENDPOINT=...` — point one chain at a custom node.

The canonical list lives in `libs/ledger-live-common/src/env.ts` (`envDefinitions`). That is your authoritative source; Confluence is always a lagging mirror. (R3)

### 0.4.5 Feature Flags — Firebase Remote Config

Feature flags are how product teams ship code dark and then turn it on per-percentage, per-version, per-platform. The runtime is **Firebase Remote Config**. (R3)

**Naming convention:**

- Firebase key: `feature_{awesome_feature_name}` — snake_case, `feature_` prefix. Example: `feature_llm_usb_firmware_update`.
- Codebase key: `{awesomeFeatureName}` — camelCase, no prefix. Example: `llmUsbFirmwareUpdate`.
- Conversion: `feature_ + _.snakeCase(camelCaseId)`. Watch out — `snakeCase` / `camelCase` are not bijective. Tokens like `TRC20` break the round-trip (`ptxSwapReceiveTRC20WithoutTrx` is the documented edge case). (R3)

**Flag value schema** is always JSON, never a bare boolean:

```json
{ "enabled": true, "mobile_version": ">=v3.37.0" }
```

Conditions supported: `enabled`, `mobile_version`, `desktop_version`, `desktop_device`, `ios_device`, `android_device`, plus Firebase-native "User in random percentile" (used for percentage rollouts).

#### Anti-pattern flag families you will meet

There is no single authoritative catalog of every flag in the product. The QA space "Feature Flags and Firebase Config Models" page (R3 source 7) documents the common anti-pattern families you will inherit: (R3)

- **`feature_currency_*`** — per-currency toggles. Inconsistent schema (some platform-specific, some global enable/disable only). Anti-Pattern 1.
- **`config_nanoapp_*`** — a "god config" controlling device deprecation, min firmware versions, device warning/error screens, node and explorer URIs, NFT display. Shared between DMK (device lifecycle) and Coin Integration (min firmware). Anti-Pattern 2.
- **`config_nanoapp_zcash`, `config_currency_sonic_blaze`** — flags named outside their expected grouping. Anti-Pattern 3.
- **`mainNavigation`, `assetSection`, `lwmVersion4`, `brazePlacement`** — LWM navigation flags, carrying **implicit dependencies** (e.g., `assetSection` depends on `mainNavigation` being on a specific variant). Anti-Pattern 5 — QA may end up exercising invalid combinations.
- **`feature_protect_services_desktop`, `feature_protect_services_mobile`** — Ledger Protect / Recover family. Keys include `enabled`, `protectID` (`protect-local` / `protect-local-dev` / `protect-staging` / `protect-prod`), `staxCompatible`, `compatibleDevices`, deep-link URIs (`upsellURI`, `loginURI`, `restore24URI`, etc.).
- **`ptxEarn`** — gates the Beta Earn Dashboard on LWD.
- **`ldmkTransport`** — gates the new transport layer on LWD.
- **`market`, `learn`** — Market screen and Learn content, used as examples in the flag-creation docs.

#### How QAA overrides flags

In E2E tests, you control flags in two ways:

1. **Via the app's Developer UI** (when exploring manually):
   - **LWD**: Settings → Developer tab → "Define feature flags used in Ledger Live" → Show → search (case-sensitive) → toggle. If the Developer tab is hidden: About tab → click the version number 10 times.
   - **LWM**: Settings → About → tap "Powered by Ledger" multiple times → Debug section appears → Debug → Configuration → Feature flags → toggle.
2. **Programmatically in the test** — via the Playwright/Detox fixture. The test setup injects flag overrides that bypass Remote Config entirely, giving you deterministic behaviour. You will learn the exact fixture API in Part 3 (Chapter 4.7 walks the full pattern).

#### Worked example — tracking a feature-flag regression

Concrete scenario you will hit in your first month. A Swap E2E test starts failing overnight on LWD. Your triage sequence:

1. **Open the Allure report** — the failure is on the "Choose provider" step; the expected provider tile does not render.
2. **Check if a flag gates that provider.** In the codebase: `grep -r "providerName" libs/ledger-live-common/src/exchange/`. You find a gate on `ptxSwapNewProvider`.
3. **Check Firebase Remote Config** for `feature_ptx_swap_new_provider` in `ledger-live-testing`. It reads `{ "enabled": true }` — no change.
4. **Check `defaultFeatures.ts`** in the codebase — fallback is `{ enabled: false }`. If Firebase is rate-limited, the test would see the provider disabled — which matches the symptom.
5. **Check `#qa-b2c-releases-bugs-tracking`** — is there a Firebase incident or a Remote Config edit overnight?
6. **Check `#wallet-ci-alerts`** — is there a runner-side network issue?
7. If nothing else, **force a Remote Config refetch** in the test setup (via the fixture override), re-run, see if it passes.

If the test passes after the forced refetch, it is a Firebase rate-limit flake — tag it via the Flaky Reporter, note in `#live-repo-health`. If it still fails after forced refetch with the flag explicitly set to `enabled:true`, it is a real PTX product defect — raise a Jira ticket against PTX with the Allure link. That is the bug-vs-flake decision matrix in action. (R5)

> **Two traps when working with flags:**
>
> - **Remote Config caches ~12h by default** (`minimumFetchIntervalMillis=0` only in DEV). To force-refetch on an installed app, delete IndexedDB (desktop) or uninstall/reinstall (mobile). (R3)
> - **Firebase rate-limits fall back to defaults.** When Firebase is rate-limited, Remote Config fetches fail silently and the app falls back to hard-coded defaults in `defaultFeatures.ts`. In practice your flag appears "off." This is a known source of E2E flakiness. When a flag-gated test regresses without a flag config change, check rate-limit status before assuming a product bug. (R3)

### 0.4.6 The Hotfix Lane

When a blocker is found in a version already released to users, the **hotfix lane** kicks in. It shares the same Phase 0-5 checklist as a normal release (same Train QA / Train Engineer / Train Delivery trio), but the branching mechanics are different. (R3)

**The sequence:**

1. **Trigger `release-create-hotfix.yml`** — manual workflow, never automatic. Parameters:
   - `Tag Version`: `latest` if hotfixing the most recent release; an explicit version (e.g., `2.25.0`, `3.55.1`) if hotfixing an N-1 release.
   - `Application`: `LLM` or `LLD` — matters only for cross-app scenarios.
2. **Workflow creates a `hotfix` branch from `main`** — not from `develop`. This is the critical distinction. You are forking from what users currently run, not from integration.
3. **Engineers cherry-pick the fix commit** into `hotfix` and include a **patch-only changeset** — no minor, no major.
4. **PR targets `hotfix`** — merged with **NO SQUASH**. (Squash would break the cherry-pick lineage.)
5. **A second PR cherry-picks the fix to `develop`** — also NO SQUASH — so `develop` does not regress when the next regular train cuts.
6. **After QA GO**: merge `hotfix` → `main` AND `hotfix` → `develop`.

**Hard constraints:** (R3)

- Only **one hotfix branch can exist at a time**. If you need to hotfix LLM on release N and LLD on release N-1, you run the workflow twice, sequentially. This is a hard bottleneck during incidents.
- **LLD has a rollback lane** (`release-rollback-desktop.yml`) — rollback takes effect a few hours after trigger. LLM has no rollback for shipped stores; it is always forward-fix via hotfix.
- **Emergency comms channels**: `#live-releases-org`, `#live-release-sync`, `#live-ci-notifications`, `#live-prerelease-sentry-alerts`, `#launchpad`.

As a QAA hire, you will not drive a hotfix in your first months, but you will participate in the Phase 2 campaign on the hotfix branch — the same nightly-against-`release` routine, just pointed at the `hotfix` branch instead.

**Worked scenario — when a hotfix would be needed.** A user reports that Send on LWD 2.137 crashes on Solana with amounts above 100 SOL. Reproduced on the shipped prod build. Steps:

1. Product confirms blocker. Train Delivery opens the hotfix process in `#live-releases-org`.
2. Train Engineer runs `release-create-hotfix.yml` with `Tag Version: 2.137.0` and `Application: LLD`. The workflow forks `hotfix` from `main`.
3. A Coin Integration engineer writes the fix, opens a PR into `hotfix` with a **patch-only** changeset (`coin-solana: patch`). NO SQUASH on merge.
4. The same engineer opens a second PR cherry-picking the commit into `develop` (also NO SQUASH) so the fix lands for the next regular train.
5. QAA points nightlies at `hotfix` branch — exercises Send-on-Solana plus the surrounding regression set (Send-on-Bitcoin, Send-on-EVM) to catch collateral damage.
6. Train QA posts GO for the hotfix in `#live-releases-org`.
7. Train Engineer merges `hotfix` → `main` (NO SQUASH) and `hotfix` → `develop` (NO SQUASH). Version bumps to 2.137.1. LWD Sign and Publish workflow pushes the patched build to CDN within hours. LWM — if this were a mobile hotfix — would need a store re-submission, adding days.
8. Post-release: close the incident channel, post-mortem scheduled within 5 working days.

Notice what is missing from this list — no place where QAA unilaterally decides to run a hotfix. The workflow trigger, the fix, the signing, the store submission are all Engineering's. QAA's contribution is the regression signal that confirms the fix does not break something else. (R3)

### 0.4.7 Prod and Prerelease — "the same artifact"

> **~ Contestable claim.** The 2024 CI page "Ledger Live CI — Prod = Prerelease Builds" (R3 source 5) states that prod and prerelease builds produce the **same artifact**: "No more way to distinguish a prod vs a prerelease build (= no more `-next` dist tags)." The version that ships to users is only known at the moment QA issues the GO. This was positioned as a 2024 architectural decision. (R3)
>
> However, the correction-pass (C4) was unable to verify this claim without access to the internal `ledger-live-build` repo (which holds the actual build scripts and signing flow). The LLD sign-and-publish workflow and the LLM store-submission jobs might re-wrap or re-sign the artifact between prerelease and prod in ways the public-facing CI docs do not expose.
>
> Write this in your head as "**commonly understood as single-artifact**" rather than a hard architectural guarantee. If you are ever asked to produce a signed SBOM or compare hashes between a prerelease build and the final prod build, consult Wallet CI directly. Do not repeat the claim as fact in a blameful incident post-mortem. (R3)

Two concrete consequences that ARE safely documented:

- **LLD prerelease auto-update was removed.** Each new prerelease must be downloaded fresh by testers. (R3)
- **npm packages are published AFTER GO-to-prod**, not during prerelease — because prerelease packages were "bug-leaking" into public npm consumers. (R3)

### 0.4.8 QAA gates the train

The headline summary of this whole chapter, and the single sentence you should memorize:

**Phase 3 is QAA's gate. No-go → release slips.**

The Train QA's GO message in `#live-releases-org` rests on two signals: the manual regression result from B2C QA, and the automation nightly signal from QAA. If the nightlies against the `release` branch are red with product defects that the Train QA cannot explain, there is no GO — the release waits for a fix to land on `release` and be re-tested.

This is why QAA's flake doctrine is so strict. Every retry, every "let me just re-run that" hides signal. Every suppressed failure is one more degree of freedom the Train QA loses when deciding GO/NO-GO. Your credibility as a QAA engineer is measured, more than by anything else, by the honesty of the signal you ship into Phase 3. (R3, R5)

<div class="chapter-outro">
<strong>Key takeaway:</strong> The dual train ships on its own cadence; Changesets drives versioning; four Firebase projects host four Remote Config streams; two <code>.env</code> namespaces coexist (release vs Detox-mock); feature flags are Firebase-backed and mutable in prod; hotfixes fork from <code>main</code> with patch-only changesets and cherry-pick back to <code>develop</code>; and Phase 3 is your gate. Read one LWD release page and one LWM release page on Confluence before your first week's 1:1 with Yaroslava — they are the single best way to internalize this rhythm.
</div>

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/3549298959/Feature+Flagging">Feature Flagging (Confluence)</a> — naming convention, schema, env mapping</li>
<li><a href="https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6667501580/LWD+2.137+-+Ledger+Live+Desktop+release">LWD 2.137 release page</a> — a canonical Phase 0-5 checklist</li>
<li><a href="https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6694010898/LWM+3.103+-+Ledger+Live+Mobile+release">LWM 3.103 release page</a> — the mobile-side equivalent</li>
<li><a href="https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6896255151/Feature+Flags+and+Firebase+Config+Models">Feature Flag anti-patterns (QA space)</a> — the families you will encounter</li>
<li><a href="https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/5234032693/How+to+hotfix+Ledger+Live">How to hotfix Ledger Live</a> — the authoritative hotfix runbook</li>
<li><a href="https://github.com/LedgerHQ/ledger-live/wiki/Changesets">Ledger Live wiki — Changesets</a> — the actual versioning tool (not semantic-release)</li>
</ul>
</div>

### 0.4.9 Quiz — Release Cycle & Environments

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Release Cycle & Environments</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> Which versioning tool does Ledger Live actually use?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>semantic-release</code> — driven by Conventional Commit types</button>
<button class="quiz-choice" data-value="B">B) <code>lerna version</code> — monorepo-wide bumps</code></button>
<button class="quiz-choice" data-value="C">C) <strong>Changesets</strong> (<code>@changesets/cli</code>) — each PR adds a <code>.changeset/*.md</code> file declaring the bump type</button>
<button class="quiz-choice" data-value="D">D) <code>npm version</code> manually per-package</button>
</div>
<p class="quiz-explanation">Ledger Live uses <strong>Changesets</strong>, not <code>semantic-release</code>. Commit messages are lint-gated against Conventional Commits for readability, but the actual bump is driven by changeset files. Anyone searching the repo for "semantic-release" will find nothing — this is a common source of confusion for new hires.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> Your LWM Detox mock-test pipeline says it reads <code>.env.mock.prerelease</code>. Which Firebase project does this file select?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>ledger-live-staging</code> — because "prerelease" is in the name</button>
<button class="quiz-choice" data-value="B">B) None — <code>.env.mock</code> and <code>.env.mock.prerelease</code> are the Detox mock-test namespace; they don't select a Firebase project. The release namespace (<code>.env</code>, <code>.env.testing</code>, <code>.env.staging</code>, <code>.env.production</code>) is what selects Firebase.</button>
<button class="quiz-choice" data-value="C">C) <code>ledger-live-production</code></button>
<button class="quiz-choice" data-value="D">D) <code>ledger-live-testing</code></button>
</div>
<p class="quiz-explanation">The two <code>.env</code> namespaces coexist. The <strong>release namespace</strong> (<code>.env</code>, <code>.env.testing</code>, <code>.env.staging</code>, <code>.env.production</code>) is what <code>renderer.webpack.config.js</code> and <code>firebase-setup.js</code> read to pick a Firebase project. The <strong>Detox mock namespace</strong> (<code>.env.mock</code>, <code>.env.mock.prerelease</code>) configures the mocked backends and canned fixtures for mobile mock tests — it is unrelated to Firebase project selection. New hires frequently confuse the two.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> A hotfix PR lands a commit on the <code>hotfix</code> branch. The engineer squash-merges it. What is the problem?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The hotfix procedure explicitly requires <strong>NO SQUASH</strong> merges — squash breaks the cherry-pick lineage when the same commit must be replayed onto <code>develop</code></button>
<button class="quiz-choice" data-value="B">B) Squash is fine; hotfixes are allowed to squash</button>
<button class="quiz-choice" data-value="C">C) The engineer forgot the Changesets file — squash is unrelated</button>
<button class="quiz-choice" data-value="D">D) Squash means the release won't ship until Monday</button>
</div>
<p class="quiz-explanation">The "How to hotfix Ledger Live" page is explicit: hotfix PRs merge with <strong>NO SQUASH</strong>. The reason is cherry-pick lineage — the same fix commit must be replayed onto <code>develop</code> via a second PR, also NO SQUASH, and onto <code>main</code> after QA GO. Squashing would collapse the commit hashes and make the cherry-pick flow error-prone. Hotfix changesets must also be patch-only — never minor, never major.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> You are asked to add a feature-flag override for <code>ptxEarn</code> in a desktop E2E test. Which Firebase project's Remote Config does the test actually read from?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>ledger-live-production</code> — tests must match real users</button>
<button class="quiz-choice" data-value="B">B) <code>ledger-live-staging</code> — staging is where Phase 2 QA runs</button>
<button class="quiz-choice" data-value="C">C) <code>ledger-live-development</code> — developers edit these flags</button>
<button class="quiz-choice" data-value="D">D) <code>ledger-live-testing</code> — QAE-editable Firebase project dedicated to QA E2E suites, selected by <code>.env.testing</code></button>
</div>
<p class="quiz-explanation"><code>ledger-live-testing</code> is the Firebase project where QAE (QA Engineers) have Editor rights and where the E2E suites read feature-flag defaults from. It is selected by the <code>.env.testing</code> file. <code>development</code> is for local dev, <code>staging</code> is for the Phase 1-3 prerelease campaign, and <code>production</code> is live users. In tests you typically override flags programmatically via the fixture — but when the fixture doesn't override a flag, the fallback read is against <code>ledger-live-testing</code>.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> At which phase of the release train does QAA exercise its GO/NO-GO gate?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Phase 1 — Prerelease Launch, when the release branch is created</button>
<button class="quiz-choice" data-value="B">B) Phase 2 — Release Test Campaign, during blocker-fix landing</button>
<button class="quiz-choice" data-value="C">C) Phase 3 — GO for Release, when Train QA posts GO in <code>#live-releases-org</code> backed by the nightly signal</button>
<button class="quiz-choice" data-value="D">D) Phase 5 — Post-Release Monitoring, when Sentry catches regressions</button>
</div>
<p class="quiz-explanation">Phase 3 is the GO decision. The Train QA's message in <code>#live-releases-org</code> rests on the manual regression result from B2C QA <em>and</em> the automation nightly signal from QAA. If nightlies are red with unexplained product defects, there is no GO — the train slips until fixes land on <code>release</code> and are re-tested. Phase 2 is where the signal is generated; Phase 3 is where it is acted on.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q6.</strong> A flag-gated E2E test regressed overnight. You check and no code changed, no flag config was edited in Firebase. What is a plausible cause mentioned in the Feature-Flag Issues doc?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The changeset file was deleted from the PR</button>
<button class="quiz-choice" data-value="B">B) Firebase rate-limited the app — Remote Config fetches failed silently and the app fell back to hard-coded defaults in <code>defaultFeatures.ts</code>, which show the flag as "off"</button>
<button class="quiz-choice" data-value="C">C) The Train QA posted NO-GO and the release was rolled back</button>
<button class="quiz-choice" data-value="D">D) The Alpaca migration moved the flag to a different Firebase project</button>
</div>
<p class="quiz-explanation">Firebase rate-limits are a known source of E2E flakiness. When fetches fail, the app silently falls back to <code>defaultFeatures.ts</code> — which typically shows the flag as disabled. The test then runs against the wrong UI state. Mitigations under discussion per R3 include retiring aged flags, defining baselines in code, and snapshotting Remote Config templates at release time. When a flag-gated test regresses without an obvious change, always rule out rate-limit fallback before assuming a product bug.</p>
</div>

<div class="quiz-score"></div>
</div>

---
## The Ticket Lifecycle - Jira, Xray and QAA's Flow

<div class="chapter-intro">
Chapters 0.1 through 0.4 gave you the what — what Ledger ships, how hardware works, who owns what, how releases move. Chapter 0.5 gives you the where — where the work actually lives day to day. At Ledger, that place is Jira. Every test case, every bug, every automation task is a ticket. This chapter teaches you the three projects that matter for QAA, how a single B2CQA test case links to a Playwright spec via Xray, and what you do on a morning when nightly went red. By the end you will be able to open Jira, find your ticket, find the test case it implements, and understand what "Automated" means in an Xray cell.
</div>

### 0.5.1 The Three Jira Projects That Matter

Ledger's Jira instance (`ledgerhq.atlassian.net`) holds dozens of projects, but as a QAA intern you will live in three of them. Learn their keys now — they are everywhere in Slack, PRs, commit messages, and ticket titles. (R4)

| Project key | Name | What it holds | Issue types | Primary stakeholder | Example ticket |
|-------------|------|---------------|-------------|---------------------|----------------|
| `LIVE` | Ledger Wallet | Product backlog for Ledger Live Desktop + Mobile + Wallet v4 (LWD/LWM). Epics, stories, bugs, and LIVE-side Xray Tests. | Bug, Task, Story, Epic, Sub-task, Test (10510) | LL dev teams + QA + PM | `LIVE-29397` (Swap 1inch bug) |
| `B2CQA` | B2CQA | QA's **test case repository**. Source of truth for manual Test cases, Test Plans, Test Executions, and QA-local bugs. Xray TMS for everything B2C. | Test (10756), Test Plan (10759), Test Execution (10760), Bug (10450) | B2C QA engineers | `B2CQA-2697` (Receive address warning — ETH) |
| `QAA` | QA Automation | Your team's backlog. Automation tasks, CI/infra work, Allure tooling, Speculos plumbing, flake triage, check-funds. | Task, Bug, Story, Epic | QAA squad | `QAA-1129` (Automate Receive warning tests on Mobile) |

Three other projects surface occasionally but you will not own work in them: `WO` (Wallet Operations — release ops / support follow-up), `LBD` (Ledger Button Delivery — Sign-in with Ledger), `QUEST` (growth and engagement features adjacent to LIVE). Ignore them until someone tags you.

> **Mental model:** B2CQA is the *library of what should work*. LIVE is the *ledger of what is broken or being built*. QAA is the *queue of what still needs to be automated*. Every QAA ticket you pick up will reference at least one B2CQA test, and most QAA bug tickets will reference at least one LIVE bug.

### 0.5.2 "LLM" and "LLD" Are Labels, Not Projects

This one trips up every new joiner. When you see `LLM-1234` in Slack, **it is not a Jira project**. There is no `LLM` project key on the Ledger cloud. (R4)

Here is the actual mapping:

| Token seen in Slack / PR / ticket | What it really is | Where it lives |
|----------------------------------|-------------------|----------------|
| `LIVE-29397` | Real Jira key | `LIVE` project |
| `LL-29397` | Colloquial alias for `LIVE-` | Read as `LIVE-29397` |
| `LLM` | **Label** on a LIVE ticket | `labels = "LLM"` — means mobile-only |
| `LLD` | **Label** on a LIVE ticket | `labels = "LLD"` — means desktop-only |
| `LWM` / `LWD` | Summary tag + label | Wallet 4.0 rebrand of LLM / LLD |
| `PTX` | **Not a project key.** Refers to Recover / Apex work. | Filed in `LIVE` with labels `Recover`, `Apex` |
| `BACKLOG` | **Not a project.** Slang for unrefined ingress. | Any project's grooming lane |

Why does this matter? Because the LIVE project deliberately does **not** use Jira Components. The product team decided long ago that a `LLM` / `LLD` label was more flexible than a component field. That decision propagated. Today, if you want to filter all mobile-only bugs, you write `project = LIVE AND labels = "LLM"`, not `project = LLM`. (R4)

> **Practical rule:** If someone says "file this under LLM," they mean *create a LIVE ticket with the `LLM` label*. Do not try to find a project called LLM — you will waste 20 minutes.

### 0.5.3 B2CQA Test Types — The Numbers You Will See

Xray on the B2CQA project uses three custom issue types. You will see their numeric IDs in Jira URLs and bulk-import spreadsheets. Remember them: (R4)

| ID | Name | Purpose | Example |
|----|------|---------|---------|
| **10756** | Test | A single manual test case. Has preconditions, steps, expected results, Automated-in field. | `B2CQA-2697` — Receive · verify address warning (ETH) |
| **10759** | Test Plan | A curated set of Tests that must pass for a release gate. | `B2CQA-5551` — Wallet 4.0 · Asset Aggregation Test Plan |
| **10760** | Test Execution | One concrete run of a Test Plan — passes and fails recorded per test. | `B2CQA-5449` — Nano Gen5 1.1.0-rc5 non-reg |

A Test goes through a short workflow: `To Do` → `In Test Review` (peer review) → `Manual Test` (validated, runnable) → `Automated` (once a spec implements it). `Obsolete` is the tombstone for retired tests. A Test Execution just has `TO DO` → `Done`. (R4)

A subtlety that matters on day one: Tests can **split** from a parent manual test via the `splits from` / `splits to` link type. That is how `B2CQA-2697` (ETH receive warning) relates to `B2CQA-652` (original generic receive warning). When you automate `B2CQA-2697`, the parent does not auto-update — only the child does. (R4)

### 0.5.4 Xray Wiring — How a Spec Links to a Test Case

This is the single most important mechanic in the QAA workflow. A Playwright or Detox spec declares which B2CQA test it implements via a TMS (Test Management System) annotation. Once the annotation ships in `main` and the CI run passes, that test case gets an "Automated" status in Xray. (R4)

**The annotation on desktop (Playwright):**

```typescript
test("Receive address warning · ETH", {
  tag: ["@NanoSP"],
  annotation: { type: "TMS", description: "B2CQA-2697" },
}, async ({ app }) => {
  await addTmsLink(getDescription(test.info().annotations, "TMS").split(", "));
  // ...
});
```

**The annotation on mobile (Detox + Jest):**

```typescript
describe("Receive", () => {
  $TmsLink("B2CQA-2697");
  it("warns on ETH address verification", async () => { /* ... */ });
});
```

Both syntaxes do the same thing: they tell the Xray runner "this spec proves that test case." When the test passes in CI, the runner pushes the result onto the B2CQA Test Execution that was created for that pipeline. The Test's status moves from `Manual Test` to `Automated` — this last transition is **currently manual**: a QAA engineer clicks the button in Jira after the PR merges. Do not expect automation here; check the Xray execution tab on the B2CQA ticket after your PR lands. (R4)

> **Why the manual step?** Because the annotation can be present before the spec is stable. Moving to Automated is a human decision: "this test now reliably proves the feature works." Flipping the status too early is how stale automated tests survive in the report.

### 0.5.5 The Typical QAA Intern Flow — From Ticket to Merge

Here is a week in the life of an automation task. Let us say you pick up `QAA-1129` — "Automate Receive address warning tests on Mobile." (R4)(R5)

```
┌─────────────────────────────────────────────────────────────────────┐
│  QAA-1129 lands in your column                                       │
│  Description table lists: B2CQA-2697 (ETH), B2CQA-2698 (BTC), ...    │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           v
┌─────────────────────────────────────────────────────────────────────┐
│  1. Open B2CQA-2697 in Jira. Read preconditions + steps.             │
│  2. Find the existing coverage: does a desktop spec already do this? │
│  3. Read the linked Test Plan (B2CQA-5551 etc.) for context.         │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           v
┌─────────────────────────────────────────────────────────────────────┐
│  4. Create feature branch from develop:                              │
│        feat/qaa-1129-receive-address-warning-eth                     │
│  5. Write the spec in e2e/mobile/specs/receive/.                     │
│  6. Add $TmsLink("B2CQA-2697") (mobile) or TMS annotation (desktop). │
│  7. Run locally against Speculos 3+ times to catch flakiness.        │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           v
┌─────────────────────────────────────────────────────────────────────┐
│  8. Push branch, open PR. CI runs the new spec.                      │
│  9. QAA reviewer checks the PR (test intent, selectors, fixtures).   │
│ 10. Merge. Nightly picks it up on develop.                           │
│ 11. Manually move B2CQA-2697 from "Manual Test" → "Automated".       │
│ 12. Move QAA-1129 to Done. Close out.                                │
└─────────────────────────────────────────────────────────────────────┘
```

Steps 7 and 11 are where interns most commonly miss the mark. Step 7: running once is not enough — three runs catch race conditions that single runs miss (see Chapter 0.5.8 on flake doctrine). Step 11: forgetting to flip the Xray status means the coverage dashboard still shows a manual-only test, and your work is invisible in the Q-end review.

### 0.5.6 Filing a Bug Caught by E2E

Nightly is red and you have confirmed it is a product defect, not a flake. The defect goes in **LIVE**, not B2CQA. Here is the exact recipe: (R4)

1. **Project:** `LIVE`.
2. **Issue type:** `Bug`.
3. **Summary:** Prefix with bracketed scope tags, e.g. `[SWAP][1inch] POST /api.1inch.dev fails often, making swap fail.`
4. **Labels — always:** `qaa` (marks the bug as caught by automation — this is how your team filters the dashboard).
5. **Labels — platform:** `LLM` for mobile-only, `LLD` for desktop-only, both if cross-platform.
6. **Labels — if release-critical:** `deploymentPROD` + `blocker`. This pair signals the release manager to cherry-pick into the next hotfix.
7. **Description — STR / ER / AR block:**
   - **STR** (Steps To Reproduce) — copy from the Xray test case so anyone can retrace.
   - **ER** (Expected Result) — what should have happened.
   - **AR** (Actual Result) — what did happen, with the failure signature.
8. **Allure link** — paste the full `ledger-live.allure.green.ledgerlabs.net` URL. This is the de-facto QA report artefact.
9. **Link back** — link the LIVE bug to the B2CQA test it breaks (`blocks`) and to the QAA task that owns the automation (`relates to`).

Real example to model your bug on: `LIVE-29397` (`[SWAP][1inch] POST /api.1inch.dev fails often...`) — caught by nightly, labelled `qaa`, fixed by the Swap squad, released. (R4)

> **Common beginner mistake:** filing the bug in B2CQA instead of LIVE. B2CQA bugs are only for defects inside the test itself (selector drift, test-plan error). If the product is broken, the bug is a **LIVE** ticket.

### 0.5.7 The Six-Way Failure Classification

When nightly fails, your first job is not to fix it — it is to **classify** it. Ledger QAA uses six buckets. Classify wrong and the ticket goes to the wrong owner, which costs everyone a day. (R5)

```
                           Nightly run is red. A test failed.
                                          │
                                          v
                          ┌───────────────────────────────┐
                          │ Can you reproduce it locally? │
                          └───────────────┬───────────────┘
                                          │
                       ┌──────────────────┴──────────────────┐
                       │                                     │
                    YES │                                     │ NO
                       v                                     v
       ┌───────────────────────────────┐    ┌──────────────────────────────┐
       │ Is it a product defect?       │    │ Infra error?                 │
       │ (app does wrong thing)        │    │ (runner, Docker, network)    │
       └───────────────┬───────────────┘    └──────────────┬───────────────┘
                       │                                   │
              ┌────────┴──────────┐                 ┌──────┴──────────┐
              │                   │                 │                 │
             YES                 NO                YES               NO
              v                   v                 v                 v
        ┌──────────┐      ┌──────────────┐    ┌──────────┐    ┌─────────────┐
        │  1. BUG  │      │ Did the      │    │ 4. INFRA │    │ Explorer    │
        │  → LIVE  │      │ product      │    │ → @qa-   │    │ 5xx?        │
        │  label   │      │ change       │    │ automation│    │             │
        │  qaa     │      │ testability? │    │ Slack     │    └──────┬──────┘
        └──────────┘      │ (data-testid │    │ + PE     │           │
                          │  removed, FF │    │ ticket   │       ┌───┴────┐
                          │  flipped...) │    └──────────┘       │        │
                          └──────┬───────┘                      YES       NO
                                 │                               v         v
                          ┌──────┴──────┐                  ┌──────────┐  ┌──────────────┐
                          │             │                  │5.EXTERNAL│  │ Lack of      │
                         YES           NO                  │→ #explorer│  │ funds on     │
                          v             v                  │ -users    │  │ Speculos?    │
                    ┌────────────┐  ┌─────────┐            └──────────┘  └──────┬───────┘
                    │2. PROCESS- │  │3. FLAKY │                                 │
                    │   GAP      │  │ → Report│                                 v
                    │ → squad    │  │ Flaky   │                          ┌──────────────┐
                    │ owner,     │  │ btn in  │                          │ 6.OPERATIONAL│
                    │ no test    │  │ Allure  │                          │ → @qa-auto   │
                    │ workaround │  └─────────┘                          │ in #live-    │
                    └────────────┘                                       │ repo-health  │
                                                                         └──────────────┘
```

The six buckets: (R5)

| # | Bucket | Signal | Action |
|---|--------|--------|--------|
| 1 | **Bug** | Reproducible locally, product does the wrong thing | File in `LIVE` with `qaa` label + platform label; add Allure link |
| 2 | **Process-gap** | Reproducible but caused by a testability regression (lost `data-testid`, new feature flag, A/B test) | Contact owning squad — **do not** patch tests around unstable selectors |
| 3 | **Flaky** | Cannot reproduce manually, passes on retry | Use the Flaky Test Reporter Chrome extension button in Allure — it auto-files a GitHub issue |
| 4 | **Infra** | Runner error, Docker crash, CI build failure | Tag `@qa-automation` in Slack; create PE ticket if sustained |
| 5 | **External** | Blockchain explorer / third-party 5xx (1inch, Exodus, etc.) | Post in `#explorer-users` with link back to the nightly Slack thread |
| 6 | **Operational** | Speculos seed out of funds, account missing | Tag `@qa-automation` in `#live-repo-health` for a fund request |

> **Why six and not two?** Because "bug vs. not-a-bug" is too coarse. A flake ticket routed to a product squad wastes their triage slot. An infra failure filed against a feature team gets closed as "cannot-repro" and then silently reopens next week. The six buckets map one-to-one onto six different owners — each with a different SLA and a different Slack channel.

### 0.5.8 Flake Doctrine — "Signal, Not Noise"

Every QAA team on the planet fights flaky tests. Ledger's doctrine is explicit and unusual: **we do not auto-retry to hide failures.** (R5)

The canonical phrasing, from the `UI E2E Test Stability` page:

> "Forcing UI E2E tests to be green at all costs hides real product issues and reduces trust."

Four principles flow from this: (R5)

1. **A red run is a signal, not noise.** Every failure gets a classification (see 0.5.7) and an owner in Slack before noon. No "it'll probably pass next time."
2. **No blanket retry policy.** The framework does not silently rerun failing tests. A flaky test that retried-to-green once is still a flaky test — it gets a `Report Flaky` GitHub issue, not a silent pass.
3. **Selective rerun is in progress, not in effect.** The brainstorm item `B01` ("Support automatic selective re-execution of failed tests... matching specific tags") is marked *In progress* — not *done*. Do not assume your test will be retried if it fails. Today, one failure = one investigation. (R5)
4. **Locator failure is not flake.** If `getByTestId("...")` returns nothing, that is either a product change or a real bug. Calling it "flake" and moving on will bury a regression. (R5)

The enforcement mechanism is cultural + tooling:

- **Cultural:** the Monitor Role (Section 0.5.9) mandates that every B2C QA classifies every failure in their scope daily. Silent failures are a policy violation.
- **Tooling:** the Flaky Test Reporter Chrome extension adds a "Report Flaky" button to each failed Allure test. Clicking it opens a GitHub issue with the project, environment, branch, and Allure link pre-filled. QAA gets auto-notified. The extension's own guidance: "do a quick check of the test before submitting a report, so we don't overwhelm the QAA team with duplicates." (R5)

> **Intern guardrail:** if a test passes 2 out of 3 local runs, **it is not ready to merge**. Three-out-of-three is the minimum before opening a PR. Your reviewer will ask; be prepared to show the Allure links for all three runs.

### 0.5.9 Daily 12:00 Triage — The Monitor Role

The Golden Rule, from the `Monitor Role – Responsibilities & Process` Confluence page: (R5)

> "By 12:00 every working day, there should be no 'silent' failures in nightly runs. Every failure must have an owner, a direction, and visibility in Slack."

How it works in practice:

- **Who:** Each B2C QA engineer monitors nightly results for their scope (desktop, mobile, or both). The Monitor Role rotates.
- **Cadence:** Daily. Delegation is mandatory when absent.
- **Time-box:** About 30 minutes. If triage takes longer than 30 minutes, escalate to the Monitor Role lead rather than silently slipping past noon.
- **Output:** A Slack thread under the automated nightly report (in `#live-repo-health`) with one post per failure: classification (bucket 1-6 from Section 0.5.7), owner handle, and link to the filed ticket or GitHub issue.
- **Escalation:** If a test has failed for three consecutive nightlies with no owner, it is auto-escalated to the QAA lead.

As an intern, you will shadow the Monitor Role for at least two weeks before rotating in. Use those two weeks to internalize the six-way classification — that is the skill the role exists to exercise.

### 0.5.10 KPIs — What "Good" Looks Like

QAA's KPIs split into two families: *numeric* (measurable gates) and *qualitative* (dashboard status). (R5)

**Numeric targets:**

| Metric | Target | Source |
|--------|--------|--------|
| Unit test coverage on critical modules | **>80%** | Testing strategy page |
| Public API test coverage (Vault) | 100% (achieved PI 26.1) | LES Weekly 01-2026 |
| Nightly triage closure | All failures owned by 12:00 | Monitor Role doctrine |
| Pre-merge E2E | Green on develop branch before merge | QA Evaluation Criteria CO06 |

**Qualitative dashboard status:**

| Surface | Status label | Definition |
|---------|--------------|------------|
| LWM Nightly Stability | Stable / Monitoring / Maintenance | "Stable" = zero flakiness, failures are bugs only |
| LWD Nightly Stability | Stable / Monitoring / Maintenance | Same definition as LWM |
| Vault B2B regression | ~400 test cases | Maintained by Vault QA |
| Vault B2C regression | ~200 test cases | Maintained by B2C QA |

There is no published flake-rate percentage gate. Stability is tracked qualitatively because E2E runs against live services (blockchains, Speculos, Swap/Buy/Sell providers) — a pure percentage would punish the team for externalities it does not own. The Merge Queue Rollout page explicitly lists "Define an acceptable flake threshold" as a *pre-requisite*, not a solved problem. Do not quote a percentage to your manager. (R5)

> **What interns are evaluated on at day 90:** access + environment + first paired run complete; one automation task delivered end-to-end; at least one nightly triage owned. The QAA Onboarding Checklist is a binary list — it does not set a "three specs by day 90" target. The qualitative signal is whether your PRs merge cleanly on first review.

### 0.5.11 Tools of the Trade

Five tools you will touch every day once you are productive: (R5)

| Tool | Where | What it does |
|------|-------|--------------|
| **Flaky Test Reporter** | Chrome extension | Adds a "Report Flaky" button to every failed/broken Allure test. Auto-creates a GitHub issue with project + env + branch + Allure link. Notifies QAA automatically. |
| **Allure dashboards** | `ledger-live.allure.green.ledgerlabs.net` | Per-run, per-branch E2E reports. Step trace, screenshots, Speculos screenshots, videos, network logs, stdout. The one link every LIVE `qaa` bug contains. |
| **Xray execution tab** | Jira → B2CQA ticket → Xray | Shows every execution of this test. Green/red per run. After your PR merges, check here to confirm the spec ran and the status moved. |
| **Jira dashboards** | `ledgerhq.atlassian.net/jira/dashboards/10427` | Automation test coverage. Dashboard 10216 = QAA Bugs per-squad. Dashboard 16388 = Vault Canton strategy. |
| **Speculos** | `ghcr.io/ledgerhq/speculos:latest` | The Docker-based firmware emulator. Local dev and CI both use the same image. See Part 3 for full deep-dive. |

You will also hear about Datadog RUM (`app.datadoghq.eu/rum/sessions`) — it is the production observability tool. QAA queries it when a bug ticket mentions "happens in production but not in CI." Not a daily tool, but a useful escape hatch.

### 0.5.12 JQL Patterns You Will Actually Use

Jira's query language (JQL) is how you slice the three projects in practice. Memorise these patterns — they will save you 10-20 minutes a day once they are muscle memory. (R4)

**Find all bugs caught by automation that still need attention:**

```jql
project = LIVE AND labels = "qaa" AND statusCategory != "Done"
ORDER BY priority DESC, created DESC
```

**Find mobile-only release blockers waiting for cherry-pick:**

```jql
project = LIVE AND labels in ("LLM")
  AND labels in ("deploymentPROD", "blocker")
  AND status = "Done"
ORDER BY updated DESC
```

**Find B2CQA tests that should be automated but aren't yet:**

```jql
project = B2CQA AND issuetype = "Test"
  AND status = "Manual Test"
  AND labels not in ("Automated")
ORDER BY created ASC
```

**Find your own QAA backlog, most recent first:**

```jql
project = QAA AND assignee = currentUser()
  AND statusCategory != "Done"
ORDER BY updated DESC
```

**Find a specific B2CQA test by keyword (useful when Slack mentions a feature, not a key):**

```jql
project = B2CQA AND issuetype = "Test"
  AND (summary ~ "receive" AND summary ~ "warning")
```

Save the queries you use more than twice as "filters" in Jira — you can then reuse them as Slack-integration sources, dashboard widgets, and scheduled digests.

### 0.5.13 The Five Most Common Intern Mistakes

Patterns observed in the first 30 days of every new QAA joiner. Recognise them in advance:

1. **Filing a bug in `B2CQA` when it belongs in `LIVE`.** A product defect always lives in LIVE. B2CQA bugs are for test-side defects only.
2. **Forgetting to flip `Manual Test` → `Automated`.** Your PR merges, the coverage dashboard stays flat, your work is invisible at quarter-end.
3. **Retrying a flaky spec to green and shipping it.** The next run will embarrass you publicly in Slack.
4. **Using a personal wallet seed or mainnet address in a test file.** Even "just for local testing" with a `TODO: remove before commit" comment. Don't.
5. **Pasting a full Allure URL with sensitive query parameters into a public GitHub issue comment.** Allure URLs themselves are fine — but double-check what you are linking to before hitting enter.

<div class="chapter-outro">
<strong>Key takeaway:</strong> Jira is a three-project universe for QAA — <code>LIVE</code> for product work, <code>B2CQA</code> for test cases, <code>QAA</code> for automation backlog. <code>LLM</code> and <code>LLD</code> are labels, not projects. Every Playwright/Detox spec links to a <code>B2CQA</code> test via a TMS annotation; the Xray status flip from <code>Manual Test</code> to <code>Automated</code> is manual. When nightly goes red, classify into one of six buckets before filing anything, and never retry-to-green. The Monitor Role enforces a noon cut-off: by 12:00 every failure has a name next to it.
</div>

### 0.5.14 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Ticket Lifecycle, Jira & Xray</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> You see a Slack message: "Please investigate LLM-4821." How should you interpret this?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) LLM is a Jira project key for Ledger Live Mobile — open <code>ledgerhq.atlassian.net/browse/LLM-4821</code></button>
<button class="quiz-choice" data-value="B">B) LLM is deprecated — the ticket no longer exists</button>
<button class="quiz-choice" data-value="C">C) LLM is a label, not a project. The sender means a LIVE ticket with the <code>LLM</code> label. Ask them for the real <code>LIVE-*</code> key or search LIVE with <code>labels = "LLM" AND key = "LIVE-4821"</code></button>
<button class="quiz-choice" data-value="D">D) LLM refers to a language model — ignore the message</button>
</div>
<p class="quiz-explanation">There is no <code>LLM</code> project on the Ledger cloud. <code>LLM</code> and <code>LLD</code> are labels on LIVE tickets used in place of Jira Components (which the LIVE project deliberately does not use). When someone writes <code>LLM-*</code> they almost always mean a <code>LIVE-*</code> ticket with the <code>LLM</code> label — ask for the exact LIVE key.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> You just wrote a Playwright spec that implements the test case <code>B2CQA-2697</code>. The PR merged. What is the final step to get the test case to show "Automated" in Xray?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Nothing — CI transitions it automatically when the spec runs green</button>
<button class="quiz-choice" data-value="B">B) Manually transition the B2CQA ticket from <code>Manual Test</code> to <code>Automated</code> in Jira after the PR merges</button>
<button class="quiz-choice" data-value="C">C) File a new QAA ticket to request the status change</button>
<button class="quiz-choice" data-value="D">D) Update the B2CQA summary to include "[AUTOMATED]"</button>
</div>
<p class="quiz-explanation">The <code>Manual Test</code> → <code>Automated</code> transition is currently manual at Ledger. The automation lands in <code>main</code>, CI runs the spec, and then a QAA engineer flips the Xray status by hand on the B2CQA ticket. This is deliberate — it forces a human judgement that the test is stable enough to count as automated coverage.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> Nightly failed on Desktop Swap ETH → USDT. You reproduce the failure locally and confirm the app's backend call to <code>api.1inch.dev</code> returns 500. Where do you file the bug?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) In <code>B2CQA</code> as a Bug (type 10450) with label <code>qaa</code></button>
<button class="quiz-choice" data-value="B">B) In <code>QAA</code> as a Bug — since it was caught by automation</button>
<button class="quiz-choice" data-value="C">C) In <code>QAA</code> as a Task — since the fix might be in the automation</button>
<button class="quiz-choice" data-value="D">D) In <code>LIVE</code> as a Bug with labels <code>qaa</code> + <code>LLD</code>, attach the Allure link, and add <code>deploymentPROD</code> + <code>blocker</code> if it is release-critical</button>
</div>
<p class="quiz-explanation">Product defects (the app behaving incorrectly) always go in <code>LIVE</code>, never in <code>B2CQA</code> or <code>QAA</code>. <code>B2CQA</code> bugs are reserved for defects in the test itself (broken selectors, test plan drift). The <code>qaa</code> label marks that automation caught it; the platform label (<code>LLD</code> here) narrows the scope; <code>deploymentPROD</code> + <code>blocker</code> signal the release manager.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> A test passed locally twice but failed once on the third run. You cannot reproduce the failure again. What is the correct action per Ledger's flake doctrine?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Do not open the PR yet — the test is not ready. 3-of-3 local runs is the minimum before merge. Investigate the intermittent failure before shipping.</button>
<button class="quiz-choice" data-value="B">B) Open the PR. CI will auto-retry on failure, so intermittent tests are fine.</button>
<button class="quiz-choice" data-value="C">C) Open the PR with an <code>@retry</code> annotation to mask the flake.</button>
<button class="quiz-choice" data-value="D">D) Open the PR — 2/3 is close enough, the reviewer will decide.</button>
</div>
<p class="quiz-explanation">Ledger's explicit doctrine is "a red UI E2E run is a signal, not noise" and "forcing UI E2E tests to be green at all costs hides real product issues." There is no blanket auto-retry. Selective rerun is still *in progress*, not in effect. Intermittent tests are not "fine" — they dilute the signal. Three-out-of-three is the minimum bar before a reviewer will approve.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> Under the Monitor Role, by what time must every nightly failure have an owner and a Slack thread entry?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) By end of day (EOD)</button>
<button class="quiz-choice" data-value="B">B) By 12:00 on the working day</button>
<button class="quiz-choice" data-value="C">C) By the next sprint planning</button>
<button class="quiz-choice" data-value="D">D) Before the next nightly run starts</button>
</div>
<p class="quiz-explanation">The Monitor Role's Golden Rule states: "By 12:00 every working day, there should be no 'silent' failures in nightly runs. Every failure must have an owner, a direction, and visibility in Slack." Triage is time-boxed to about 30 minutes; if it overruns, escalate rather than slip past noon. Silent failures are a policy violation.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> A nightly E2E test started failing after a new feature flag shipped. The feature flag disabled a UI element that the test relied on. The product behaviour is intentional — the team wanted the element hidden. The test cannot find the selector. Which failure bucket is this?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Bug — the element is missing from the UI</button>
<button class="quiz-choice" data-value="B">B) Flaky — not reproducible across environments</button>
<button class="quiz-choice" data-value="C">C) Process-gap — the product change broke testability and needs coordination with the owning squad, not a silent test patch</button>
<button class="quiz-choice" data-value="D">D) Infra — CI can't find the element</button>
</div>
<p class="quiz-explanation">This is a textbook process-gap failure. The product change is intentional but it broke the test's assumptions (and possibly other tests too). The doctrine is explicit: contact the owning squad, flag the testability regression, and don't "adapt tests around unstable selectors." Patching the test silently would paper over a coordination failure and set a bad precedent.</p>
</div>

<div class="quiz-score"></div>
</div>

<div class="chapter-outro">
You now know how tickets move through Jira at Ledger. The last chapter of Part 0 covers the invisible but load-bearing layer beneath everything QAA does — security. When the product you test is a hardware wallet, the rules for what you say in public, what you commit, and who you escalate to are stricter than in any other industry. Chapter 0.6 covers the ones that apply to you on day one.
</div>

---

## Security Posture & Responsible Disclosure

<div class="chapter-intro">
Ledger's entire product category rests on one promise: the user controls their keys, and nobody else — not even Ledger — can move their funds. Every test you write, every log you paste, every commit message you push either reinforces that promise or leaks around it. This chapter is short because the rules are simple, but the rules are non-negotiable. Read it twice.
</div>

### 0.6.1 Why Security Is THE Product

At a software company, security is one feature among many. At Ledger, security **is** the product. A hardware wallet without security is a USB-C keychain.

The phrase you will hear in every all-hands, in every onboarding deck, in every incident retro:

> **"Not your keys, not your coins."**

It is a crypto community slogan that Ledger adopted as a product principle. It means: if someone else holds your private key (an exchange, a custodian, a cloud wallet), you do not actually own your cryptocurrency — you own a promise. Ledger devices exist so that *you* hold the key, and it never leaves the Secure Element. Every Ledger product is a variation on that theme.

The consequence: a single CVE in the wrong place is not a "ship a hotfix and move on" event. It is a **trust catastrophe**. Users who lose confidence in the key-custody guarantee do not return. Unlike a social app where users tolerate outages, a wallet user who once thought "maybe my keys are exposed" has lost the product forever.

This shapes every engineering decision at Ledger — including yours:

- **Features ship slower** because a review cycle includes a security review, not just a code review.
- **Dependencies are audited** harder than in most shops. A supply chain attack on `npm` is a direct product risk (see 0.6.7).
- **Test environments are isolated** from production seeds, production backends, and production keys. The Speculos emulator uses test seeds with test funds on dedicated testnets.
- **Firmware and OS binaries are treated as secrets** pre-release. Leaking a pre-release firmware hash can burn months of Donjon work.

> **The orientation shift for a new QAA intern:** at a typical B2C shop, the highest-severity bug is "app crashes on launch." At Ledger, the highest-severity bug is "app signs a transaction the user did not intend." The first costs you a session; the second costs you your life savings. Your test priorities should reflect that ordering.

### 0.6.2 QAA's Security Boundary

You are a QA Automation engineer, not a security researcher. The boundary is clear and the distinction matters:

| Who | What they test | Tools |
|-----|----------------|-------|
| **Donjon** (security research) | Crypto primitives, side-channel attacks, fault injection, firmware vulnerabilities, supply chain threats | Hardware labs, formal verification, reverse engineering, internal red team |
| **QAA** (you) | Software **behaviour** at the app, SDK, and CI interface: does the app do what the spec says, under the conditions the user will face? | Playwright, Detox, Speculos, Allure, Xray |

You do **not** test whether the Secure Element resists a voltage-glitching attack. You do **not** attempt to extract a seed from a running device. Those are explicit Donjon activities and attempting them outside that context can violate internal policy and external law.

What you **do** catch — at the software behaviour layer — matters for security:

- **Regressions that weaken an existing security control** (e.g. an address verification warning that silently stops appearing because a `data-testid` was removed and the component underneath also stopped rendering).
- **Signing flow drift** (e.g. the "verify on device" step gets skipped on a code path the team did not notice).
- **Clipboard / deep-link behaviour** that could be abused (e.g. a malicious deeplink dropping the user into a pre-filled send flow — the test should confirm the confirm-on-device step still fires).
- **Feature-flag surprises** (e.g. a flag that was meant to hide a dev-only endpoint stays off in production — your E2E run on develop can catch it early).

Your leverage is at the *interface*. When the app asks the device to sign, you verify the right amount shows on the device. When the app warns about an unverified address, you verify the warning is visible. You are not breaking crypto — you are proving the crypto ceremony is still reachable.

### 0.6.3 Donjon — Ledger's Security Team

Donjon (French for "keep" or "dungeon" — the strongest tower of a castle) is Ledger's in-house security research organisation. Founded in 2018, it publishes ongoing research on wallet security, audits Ledger's own hardware and software, and runs the public bug bounty program.

**What Donjon does, in rough priority order:**

1. **Internal audit** of firmware, BOLOS, Secure Element integration, and coin apps before release.
2. **Attack research** — side-channel analysis, fault injection, hardware tear-down on Ledger devices and competitors' devices.
3. **Supply chain monitoring** — dependency audits on critical npm packages, PyPI packages, and Docker base images used across Ledger products.
4. **Bug bounty program administration** — triaging and rewarding external researchers.
5. **Public research publication** — the `donjon.ledger.com` site hosts write-ups on wallet security, CTF challenges, and tools like `lascar` and Speculos itself (which originated in Donjon).

**What you should know as an intern:**

- Donjon is not in your Slack day-to-day. You reach them through your tech lead, not directly, unless you are explicitly introduced.
- If you find something that looks like a real vulnerability (not a test flake, not a UX bug — an actual exploitable behaviour), your job is **not** to investigate further. Your job is to stop, flag it privately, and hand it to Donjon via your tech lead.
- The line between "I found a weird bug" and "I found a vulnerability" is fuzzy. When in doubt, err on the side of private escalation. Nobody has ever been criticised at Ledger for being too cautious about a suspected vulnerability.

### 0.6.4 Bug Bounty Program

Ledger runs a public bug bounty program, hosted at `donjon.ledger.com`. The program is the official channel for external researchers — and for any external-facing write-up of a vulnerability.

**Scope (typical — check the site for the current authoritative list):**

- Ledger firmware and BOLOS OS
- Coin apps (Ethereum, Bitcoin, Solana, etc.)
- Ledger Live Desktop and Mobile
- Ledger Live backends (public APIs)
- Ledger Recover infrastructure

**Out of scope (typical):**

- Social engineering against Ledger employees
- Physical attacks against Ledger offices
- Denial-of-service attacks against production services
- Reports on publicly known vulnerabilities with no new angle
- Issues in third-party integrations where Ledger is not the vulnerability owner

**How reports are submitted:** Via the `donjon.ledger.com` bounty page. Reports include steps to reproduce, proof of concept, and PGP-encrypted details for sensitive findings. The Donjon PGP public key is published on the same page.

**Rewards:** The program offers graduated rewards by severity (informational, low, medium, high, critical) with critical findings in the firmware / Secure Element layer at the top of the range. Exact amounts are published on the Donjon bounty page and updated over time.

As a QAA intern, you are not going to file bounty reports yourself (you are inside the company). But you **are** likely to encounter reports that came through the program when triaging tickets — Donjon sometimes opens private LIVE tickets from bounty submissions. If you see a LIVE ticket with restricted visibility and a Donjon assignee, do not probe — assume it is a live bounty investigation and move on.

### 0.6.5 Responsible Disclosure Rules

These are absolute. Violating them is a fireable offence and, depending on jurisdiction, potentially a criminal one.

**NEVER — without exception:**

- Post vulnerability details (real or suspected) in **public GitHub** — issues, PR descriptions, commit messages, gists.
- Paste vulnerability details in **public Slack channels** — `#general`, `#engineering`, `#live-repo-health`, and any channel that is not explicitly marked private and security-cleared.
- Include vulnerability details in **commit messages** — these are permanent, public, and indexed by GitHub search forever.
- Discuss vulnerabilities on **external channels** — Twitter/X, LinkedIn, personal blog, Discord, email to external addresses, even a "friends in crypto" DM.
- Attempt to **reproduce or exploit** a suspected vulnerability beyond what is strictly needed to confirm its existence.

**ALWAYS — without exception:**

- **Flag to your tech lead first.** They are your Donjon liaison. They know the escalation protocol and whether something you found is actually in scope.
- Use the **private security Slack channel** (your onboarding buddy will give you the name — it is not listed in the public channel directory).
- If you cannot reach your tech lead, escalate to the QAA lead. If you cannot reach them either, email the Donjon team directly using the address on `donjon.ledger.com`.
- **Assume PGP-grade sensitivity by default.** If you are not sure whether something is sensitive, treat it as sensitive until a human with authority tells you otherwise.

> **The mental test:** before you paste *anything* security-adjacent into *any* medium, ask: "If this were on a hacker forum tonight, would Ledger care?" If yes, do not paste it. Escalate it.

### 0.6.6 What NOT to Commit

The QAA repositories are public (GitHub `LedgerHQ/ledger-live`). Every commit is indexed by search crawlers, scraped by supply-chain research tools, and — critically — permanent. Force-pushing does not delete what has already been scraped.

**Absolute no-commit list:**

| Category | Examples | Why it matters |
|----------|----------|----------------|
| **Seeds / mnemonics** | 12- or 24-word BIP39 phrases, derived private keys, xprv/xpriv | Even test seeds on testnets are fingerprinted and can reveal infrastructure patterns |
| **API keys / tokens** | Blockchain provider keys (Alchemy, Infura, QuickNode), GitHub PATs, Slack tokens, Datadog tokens, Firebase tokens | Auto-scraped within minutes of push; rotation is costly and disruptive |
| **Firmware binaries** | `.elf`, `.hex`, `.bin` files from coin apps or BOLOS builds | Pre-release binaries reveal upcoming features and potential attack surface |
| **Pre-release artefacts** | Unreleased app APKs/dmg builds, internal TestFlight builds, beta installers | Can leak roadmap and give attackers a head start on unreleased surface |
| **Internal URLs** | Staging API hostnames, internal Datadog queries, internal Jira URLs with sensitive data in query strings, internal dashboards | Map of the internal attack surface — low individual value, high aggregate value |
| **Customer PII** | Email addresses, names, transaction histories from support tickets or Datadog RUM sessions | Regulatory (GDPR) and trust risk |
| **Crash dumps / heap dumps** | Raw `core` files, Datadog heap exports | May contain keys or seeds in memory |

**What to do instead:**

- Test seeds: use the shared Speculos test seed from the fixtures layer (`Account.BTC_NATIVE_SEGWIT_1` etc.) — these are publicly known test values that are considered "burnt" for bounty purposes.
- API keys: use environment variables, loaded from 1Password for local dev, from GitHub Actions secrets for CI. Never hard-code.
- Firmware binaries: download at test time from the authenticated artefact store; never check them into the repo.
- Staging URLs: either hard-code the production URL (which is public) or read the staging URL from an env var at runtime.

A pre-commit hook (via `husky` + `git-secrets` or equivalent) should catch most of these. Do **not** rely on it. It is a safety net, not a gate.

### 0.6.7 Past Incidents — Factual Context

Two public incidents shape how Ledger thinks about security today. They are referenced in onboarding, retros, and cross-team planning. Know the one-line version so you are not surprised when they come up in a meeting.

- **2020 — Kroll / customer data leak.** A third-party e-commerce data breach exposed a subset of Ledger customers' email and shipping addresses. No cryptocurrency and no wallet keys were affected. The incident triggered a wave of phishing attempts against affected users and accelerated Ledger's investment in customer-facing anti-phishing communication.
- **2023 — Connect Kit / supply chain compromise.** A compromised developer account pushed a malicious version of the `@ledgerhq/connect-kit` npm package, used by third-party dApps to connect Ledger devices. The malicious code attempted to drain connected wallets on dApps that pulled the compromised version. Ledger published a mitigated version within hours. Ledger devices themselves were unaffected (transactions still required on-device confirmation), but the incident accelerated internal investment in dependency signing, package publication gating, and Donjon's supply chain monitoring.

> **No-FUD rule:** when these come up in conversation, state them factually and stop. Do not speculate about what *could* have happened, do not compare against competitors, do not embellish severity. The facts are stronger than the commentary.

### 0.6.8 QAA Security Checklist — Before You Commit

Five questions, every commit. Two minutes. Internalise them until they are automatic.

```
┌───────────────────────────────────────────────────────────────────┐
│  BEFORE YOU PUSH — 5-QUESTION SECURITY CHECKLIST                  │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. SEEDS / KEYS                                                  │
│     Does my diff contain any seed phrase, private key,            │
│     mnemonic, or xprv/xpriv string? → If yes, remove it.          │
│                                                                   │
│  2. TOKENS / SECRETS                                              │
│     Does my diff contain any API key, token, password,            │
│     or credential? → Move it to env vars / 1Password.             │
│                                                                   │
│  3. INTERNAL URLS                                                 │
│     Does my diff contain a staging/internal hostname,             │
│     a Datadog query, or an internal dashboard URL? →              │
│     Parameterise via env var.                                     │
│                                                                   │
│  4. BINARIES / PRE-RELEASE ARTEFACTS                              │
│     Does my diff check in a firmware binary, APK, dmg,            │
│     or unreleased build? → Remove; download at test time.         │
│                                                                   │
│  5. SECURITY-ADJACENT FINDINGS                                    │
│     Does my commit message or PR description describe a           │
│     vulnerability, an exploit path, or a suspected bypass? →      │
│     STOP. Do not push. Escalate to tech lead privately first.     │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

If the answer to question 5 is ever "yes," your commit workflow pauses right there. There is no rush — no ticket deadline at Ledger is worth a public disclosure. Take the time.

<div class="chapter-outro">
<strong>Key takeaway:</strong> Security at Ledger is not a feature — it is the product itself. QAA's job is to test software behaviour at the interface, not crypto primitives (that is Donjon's job). The rules for what you commit and what you say in public are strict and absolute: never post vulnerability details to public channels, never commit seeds or tokens, always escalate security-adjacent findings privately to your tech lead first. When in doubt, escalate; when sure, still double-check. The two-minute 5-question checklist before every push is the cheapest insurance you will ever buy.
</div>

### 0.6.9 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Security Posture & Responsible Disclosure</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> You are writing a Playwright spec and you need a known BTC address with funds to test a send flow. What do you do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Generate a fresh 24-word seed, write it into the spec file, and push</button>
<button class="quiz-choice" data-value="B">B) Use the shared Speculos test seed via the <code>Account</code> enum in <code>@ledgerhq/live-common/e2e/enum/</code>; never write a seed phrase into code</button>
<button class="quiz-choice" data-value="C">C) Paste your personal wallet seed just for local testing — you will remove it before committing</button>
<button class="quiz-choice" data-value="D">D) Generate a seed, encrypt it with AES, and commit the ciphertext</button>
</div>
<p class="quiz-explanation">Shared Speculos test seeds live in the <code>Account</code> enum in <code>@ledgerhq/live-common/e2e/enum/</code>. They are deliberately "burnt" — publicly known test values on testnets. Never paste any seed phrase (personal or otherwise) into code, even as a placeholder. "I will remove it before commit" is how secrets end up in Git history permanently.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> While debugging a nightly failure, you notice that an app flow allows signing a transaction without prompting the device to confirm. This would let a malicious dApp drain connected wallets silently. What is the first thing you do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) File a LIVE bug with full reproduction steps and an Allure link so the team sees it fast</button>
<button class="quiz-choice" data-value="B">B) Post a quick note in <code>#live-repo-health</code> so the on-call dev sees it</button>
<button class="quiz-choice" data-value="C">C) Stop. Do not file a public ticket or post in public Slack. Flag to your tech lead privately (they are your Donjon liaison) and let them route the disclosure</button>
<button class="quiz-choice" data-value="D">D) Reproduce the issue on mainnet with real funds to confirm severity before reporting</button>
</div>
<p class="quiz-explanation">A vulnerability that could drain wallets is a Donjon matter, not a normal bug. Filing a public LIVE ticket, posting in public Slack, or writing commit messages about it leaks the exploit path before a fix ships. The responsible disclosure rule is absolute: escalate privately to your tech lead first. Also, option D would potentially move real funds — never test security-adjacent hypotheses on mainnet.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> Which of the following is OUTSIDE QAA's test scope at Ledger?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Side-channel attack resistance of the Secure Element (voltage glitching, power analysis)</button>
<button class="quiz-choice" data-value="B">B) Verifying that a "confirm on device" step still triggers in a refactored send flow</button>
<button class="quiz-choice" data-value="C">C) Verifying that an address-verification warning is shown on an unverified receive address</button>
<button class="quiz-choice" data-value="D">D) Verifying that a feature flag correctly hides a dev-only endpoint in production</button>
</div>
<p class="quiz-explanation">Side-channel and fault-injection attacks on the Secure Element are Donjon's domain, not QAA's. B, C, and D are all software behaviour at the interface layer — exactly what QAA E2E tests are designed to catch. Attempting side-channel research outside Donjon's context can violate internal policy.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> Which of the following is acceptable to commit to the public <code>LedgerHQ/ledger-live</code> repository?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A pre-release firmware <code>.elf</code> binary, so CI can run it against Speculos</button>
<button class="quiz-choice" data-value="B">B) A GitHub personal access token hard-coded into a test script, for convenience</button>
<button class="quiz-choice" data-value="C">C) A 24-word mnemonic you generated specifically for this test</button>
<button class="quiz-choice" data-value="D">D) A reference to the shared <code>Account.BTC_NATIVE_SEGWIT_1</code> enum value, which resolves to the known Speculos test seed at runtime</button>
</div>
<p class="quiz-explanation">A, B, and C are all in the absolute no-commit list. Pre-release firmware binaries leak roadmap and attack surface; hard-coded tokens are scraped within minutes; any mnemonic written to a Git-tracked file is compromised the moment it is pushed (and remains compromised forever, even after force-push). Option D uses the shared test-seed enum — publicly known, intentionally burnt, and the correct pattern.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> What is the one-line factual summary of the December 2023 Connect Kit incident?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A Ledger device firmware vulnerability let attackers extract seeds via USB</button>
<button class="quiz-choice" data-value="B">B) A compromised developer account pushed a malicious version of the <code>@ledgerhq/connect-kit</code> npm package that attempted to drain wallets on dApps pulling the compromised version; Ledger devices themselves required on-device confirmation and were unaffected at the device layer</button>
<button class="quiz-choice" data-value="C">C) Ledger accidentally published pre-release firmware to the production update channel</button>
<button class="quiz-choice" data-value="D">D) A customer data breach exposed email addresses and shipping information</button>
</div>
<p class="quiz-explanation">The 2023 incident was a supply-chain compromise of an npm package (<code>@ledgerhq/connect-kit</code>) used by third-party dApps. Ledger hardware itself was not compromised — the on-device confirmation step still fired — but dApps pulling the bad version at browse-time exposed end users' connected wallets to a drainer script. The incident accelerated dependency signing and Donjon's supply chain monitoring investment. Option D describes the unrelated 2020 Kroll customer-data incident.</p>
</div>

<div class="quiz-score"></div>
</div>

<div class="chapter-outro">
You have now walked the full ground of Part 0: what Ledger ships, how the hardware enforces security, who builds it, how releases move, how tickets flow through Jira and Xray, and how the security posture shapes everyday engineering habits. One assessment stands between you and Part 1.
</div>

---

## Part 0 Final Assessment

<div class="chapter-intro">
Part 0 covered six chapters: product ecosystem, hardware and BOLOS, teams and ownership, release cycle and environments, the Jira/Xray ticket lifecycle, and security posture. This final assessment draws from all six. Pass it at 80% and you have the ecosystem context you need to start Part 1 — where we stop talking about the company and start talking about the tools. Give yourself fifteen minutes, no notes, and try to answer each question before reading the explanation.
</div>

<div class="quiz-container" data-pass-threshold="80">
<h3>Part 0 Final Assessment</h3>
<p class="quiz-subtitle">10 questions · 80% to pass · Covers all Part 0 chapters</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> Which statement best describes the relationship between Ledger Live and a Ledger hardware device?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Ledger Live stores the user's private keys and the device signs with a copy</button>
<button class="quiz-choice" data-value="B">B) Ledger Live is a thin GUI — the hardware device performs all blockchain logic</button>
<button class="quiz-choice" data-value="C">C) Ledger Live is the companion app that manages accounts, builds transactions, and displays balances; the device signs transactions offline — private keys never leave the Secure Element</button>
<button class="quiz-choice" data-value="D">D) Ledger Live and the device share private keys over Bluetooth</button>
</div>
<p class="quiz-explanation">Ledger Live builds unsigned transactions, queries balances from public explorers, and manages the user's portfolio view. The device holds the seed in the Secure Element, signs transactions on-device after user confirmation, and returns the signature. Private keys never leave the Secure Element — that is the core security guarantee. Ledger Live itself never sees a private key.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> Why does Ledger ship coin apps as separate binaries installed onto the device rather than bundling all coin logic into firmware?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) To make the device feel faster by avoiding firmware bloat</button>
<button class="quiz-choice" data-value="B">B) So coin-specific logic can be audited, updated, and installed independently under BOLOS's process isolation — a compromised coin app cannot reach another app's keys</button>
<button class="quiz-choice" data-value="C">C) Because the Secure Element cannot run more than one binary at a time</button>
<button class="quiz-choice" data-value="D">D) To let third-party developers install arbitrary code on Ledger devices</button>
</div>
<p class="quiz-explanation">BOLOS (Blockchain Open Ledger Operating System) enforces process isolation between coin apps. Each app has its own derivation path and cannot access data from other apps. This model allows per-coin audit cadence, supports the long tail of blockchains without bloating firmware, and limits blast radius — a bug in a niche coin app cannot compromise BTC keys in the adjacent app.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> Which statement about BOLOS is correct?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) BOLOS runs on the user's computer and manages the device over USB</button>
<button class="quiz-choice" data-value="B">B) BOLOS is a Linux distribution adapted for embedded devices</button>
<button class="quiz-choice" data-value="C">C) BOLOS is a cloud service that validates app signatures before installation</button>
<button class="quiz-choice" data-value="D">D) BOLOS is Ledger's custom OS running inside the Secure Element; it provides cryptographic primitives and a sandbox for coin apps</button>
</div>
<p class="quiz-explanation">BOLOS is a purpose-built embedded OS written specifically to run inside a Secure Element microcontroller. It exposes cryptographic primitives (secp256k1/ed25519 signing, BIP32/BIP39 derivation, hashing) to coin apps via a syscall layer and enforces process isolation between apps. It is neither Linux nor cloud-hosted — it runs inside the chip on the device.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> Which team owns the end-to-end test framework for Ledger Live Desktop (Playwright + Speculos plumbing, Allure infrastructure, nightly orchestration)?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) QA Automation (QAA / SDET) — the dedicated automation team</button>
<button class="quiz-choice" data-value="B">B) The B2C QA team embedded with product squads</button>
<button class="quiz-choice" data-value="C">C) The Ledger Live Desktop dev team</button>
<button class="quiz-choice" data-value="D">D) Donjon</button>
</div>
<p class="quiz-explanation">QAA owns the horizontal automation backbone — Playwright and Detox frameworks, Speculos plumbing, Allure reporting, nightly orchestration, and flake tooling — across LWD, LWM, and Vault. B2C QA owns the manual test cases and the Monitor Role rotation. Dev teams own unit + integration tests and mocked-E2E tests, and they automate regression tests identified by QA, but the automation infrastructure itself is QAA's. Donjon does security research, not E2E automation.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> A feature team's developer has just finished a new feature. Who is responsible for writing the unit tests, and who is responsible for writing the regression test case in Xray?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) QAA writes both</button>
<button class="quiz-choice" data-value="B">B) The developer writes unit tests; a B2C QA engineer authors the Xray Test case during Phase 2 (Test Design, in parallel with dev)</button>
<button class="quiz-choice" data-value="C">C) The developer writes both</button>
<button class="quiz-choice" data-value="D">D) The PO writes both</button>
</div>
<p class="quiz-explanation">The division is explicit: developers own unit and integration tests (weight 5 in QA Evaluation CO03), and they can write E2E specs when needed (CO04/CO05). B2C QA engineers own the Xray Test cases and Test Plans, authored during Phase 2 in parallel with dev, so the test case exists before the feature ships. QAA then reviews any PR that automates an existing regression test — "QA owns the tickets that define which tests must be automated; Development automates them; QAA reviews the resulting PR."</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> Ledger Live runs on multiple environments. In which environment do nightly E2E tests typically run, and why?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Production — to validate user experience against real traffic</button>
<button class="quiz-choice" data-value="B">B) A frozen release candidate — to avoid moving targets</button>
<button class="quiz-choice" data-value="C">C) Against the <code>develop</code> branch — so tests act as early warning for product regressions before a release is cut</button>
<button class="quiz-choice" data-value="D">D) Against <code>main</code> only, post-release</button>
</div>
<p class="quiz-explanation">Nightly E2E runs target the <code>develop</code> branch, not a frozen build. The rationale is explicit in the UI E2E Test Stability doctrine: tests run against in-flight code so that regressions are caught as they land, not after a release gate. This is why the runs touch live services, real blockchains, Speculos, and third-party providers — and why 100% green is explicitly not the target.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q7.</strong> Which of the following release surfaces has the fastest iteration cadence at Ledger?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Web / backend services — continuous delivery behind feature flags; multiple deploys per day on some services</button>
<button class="quiz-choice" data-value="B">B) Ledger Live Desktop — weekly auto-update</button>
<button class="quiz-choice" data-value="C">C) Ledger Live Mobile — bi-weekly store release</button>
<button class="quiz-choice" data-value="D">D) Device firmware / coin apps — gated by security review, multi-week cycles</button>
</div>
<p class="quiz-explanation">Release cadence varies by surface: web/backend services ship continuously behind feature flags, Ledger Live Desktop ships on an auto-update cadence (typically weekly), Mobile ships through the App Store / Play Store on a slower cadence because of store review, and firmware + coin apps go through the slowest cycle because any change is gated by Donjon security review and on-device signature. The further you are from the keys, the faster you can ship.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q8.</strong> Which Jira project holds the manual test cases, Test Plans, and Test Executions that Xray maps to automated specs?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>LIVE</code></button>
<button class="quiz-choice" data-value="B">B) <code>B2CQA</code></button>
<button class="quiz-choice" data-value="C">C) <code>QAA</code></button>
<button class="quiz-choice" data-value="D">D) <code>LLM</code></button>
</div>
<p class="quiz-explanation"><code>B2CQA</code> is the source of truth for Xray on the B2C side. It holds Test (type 10756), Test Plan (10759), and Test Execution (10760) issue types. <code>LIVE</code> holds product bugs and stories. <code>QAA</code> holds automation backlog tasks. <code>LLM</code> is not a project — it is a label on <code>LIVE</code> tickets.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q9.</strong> You find a potential vulnerability in a signing flow while debugging a test. Which action is correct?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Open a public GitHub issue with reproduction steps so more eyes can review</button>
<button class="quiz-choice" data-value="B">B) Post in <code>#engineering</code> Slack so the team is aware quickly</button>
<button class="quiz-choice" data-value="C">C) Write a detailed PR with a failing test that demonstrates the exploit</button>
<button class="quiz-choice" data-value="D">D) Stop. Escalate privately to your tech lead (Donjon liaison). Do not discuss in public channels, GitHub, or commit messages until cleared.</button>
</div>
<p class="quiz-explanation">Responsible disclosure is absolute. Public GitHub, public Slack, and commit messages are permanent and indexed. A vulnerability in a signing flow is a Donjon matter — your tech lead routes it privately. Options A, B, and C would all leak an exploit path before a fix ships and are fireable-level violations of Ledger's disclosure rules.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q10.</strong> Which sequence correctly describes a QAA intern automating the B2CQA test case <code>B2CQA-2697</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Open a LIVE ticket → write the spec → post results in Slack → close the LIVE ticket</button>
<button class="quiz-choice" data-value="B">B) Write the spec with no annotation → merge → CI figures out the mapping automatically</button>
<button class="quiz-choice" data-value="C">C) Pick up a QAA task that references <code>B2CQA-2697</code> → write the spec with a TMS annotation (<code>$TmsLink("B2CQA-2697")</code> or the Playwright annotation equivalent) → 3-of-3 local runs → open PR → QAA review → merge → manually transition the B2CQA ticket from <code>Manual Test</code> to <code>Automated</code></button>
<button class="quiz-choice" data-value="D">D) File the B2CQA ticket yourself, then write the spec</button>
</div>
<p class="quiz-explanation">The full flow is QAA-ticket → read B2CQA ticket → write spec with TMS annotation → 3-of-3 local runs → PR with QAA review → merge → manual status transition on the B2CQA ticket. Option A mixes up LIVE (for product bugs) with QAA (for automation tasks). Option B forgets the annotation — without <code>$TmsLink</code> / TMS annotation, Xray has no way to map the spec to the test case. Option D has the QA engineer author the B2CQA ticket — that is B2C QA's job, not QAA's.</p>
</div>

<div class="quiz-score"></div>
</div>

<div class="chapter-outro">
<strong>Congratulations — you've completed Part 0.</strong> You now know what Ledger ships, how the hardware works, who owns what, how releases move, how tickets flow through Jira and Xray, and how the security posture shapes everyday engineering. The ecosystem is mapped. Now that you know the ecosystem, let's meet the tools. Part 1 starts your first real touch of the codebase — the local environment, the monorepo layout, Playwright, Speculos, and the first commands you'll run before you even think about writing a test. Turn the page.
</div>
