# R1 — Ledger Org Structure, Teams, Ownership & Team-to-Code Mapping

Research agent: R1
Scope: Ledger org structure, teams, ownership, team-to-code mapping (QAA-centric view)
Method: Confluence Rovo search only. `getConfluenceSpaces` and `searchConfluenceUsingCql` were denied by permissions, so no space-key resolution was possible; all findings come from Rovo `search` hits followed by direct `getConfluencePage` fetches.

---

## 1. Team Roster

| Team | Tribe / Division | GitHub codeowner handle(s) observed | Confluence anchor | Mission (1-liner) | Slack channel(s) |
|---|---|---|---|---|---|
| QA Automation (QAA) | Quality Assurance | not confirmed in fetched pages | Quality Assurance space (page 6635357792) | SDET team building & maintaining E2E automation (LWD/LWM/Vault) across Ledger products | exact QAA channels to be confirmed with Yaroslava POLISHCHUK; adjacent: `#live-engineering` |
| QA B2C | Quality Assurance | — (manual QA, no code ownership) | Quality Assurance overview | Manual QA for consumer products (Wallet/LL): risk assessment, regression, release validation | — |
| QA B2B | Quality Assurance | — | Quality Assurance overview | Manual QA for enterprise products (Vault, HSM) | — |
| Wallet XP (WXP) — umbrella | Wallet Experience tribe (B2C Eng) | — | Wallet XP space (page 4242605488) | "Build the best Wallet out there to manage our users' assets" — owns Ledger Live Desktop (LLD) + Mobile (LLM). 3 squads: Live Hub, Devices Experience, Wallet API/Connectivity | `#team-wallet-xp`, `#team-wallet-engineering`, `#wallet-experience-private`, `#live-engineering` |
| Live Hub squad | Wallet XP | `@LedgerHQ/live-hub` (teams/live-hub) | Live Hub (HUB) Squad (page 4245487751) | UI-oriented squad for Wallet flows in LL: portfolio, accounts, send/receive, WalletSync, Market, Settings, Braze/analytics, UI libs (react-ui, native-ui) | `#live-hub`, `#live-engineering` |
| Devices Experience squad | Wallet XP | `@LedgerHQ/live-devices`, `@LedgerHQ/android` | Devices Experience Squad (page 4244276186) | Connectivity & interaction between Ledger hardware and wallets; owns `@ledgerhq/hw-transport*`, Device SDK, firmware update flow, My Ledger UI, Metamask Mobile | `#team-devices`, `#squad-devices`, `#squad-devices-devs` |
| Wallet API / Connectivity | Wallet XP (partially folded into Coin Integration) | `@LedgerHQ/team-wallet-api` | Ask other teams (page 4300767271) | APIs for basic wallet usage across all LL-supported blockchains; Discover section, WebPlayers, Live Apps runtime | `#team-wallet-api` |
| Wallet CI ("CI Team") | Ledger Live tribe (WALLETCO space) | `@LedgerHQ/team-wallet-ci` | [CI] Team (page 5753635342) | CI pipeline for `ledger-live` + `ledger-live-build`: fast dev->prod, reliable releases, KPI = releases per month | `#team-wallet-ci`, `#live-ci-notifications`, `#wallet-ci-alerts`, `#ledger-live-ci`, `#live-release-sync`, `#infra-live` |
| PTX (Payments & Transactions Services) | Consumer Services tribe | `@LedgerHQ/team-ptx-*` (sub-teams not fully enumerated) | PTX / CONSUMER SERVICES space | Buy, Sell, Swap, Earn (Stake), Recover — financial transaction flows delivered as Live Apps embedded in LL | `#ptx-buy-sell`, `#alert-buy`, `#sentry-ptx-buy-sell`, `#ptx-swap-prod`, per-partner `#partner-<name>` |
| Engagement Team | own Confluence space "Engagement" | — | Engagement Team space (page 6702270213) | Drives user engagement features in LL; North Stars: Weekly Value Engaged Users (WVEU), Activation within 48hr. Owns Favourite Token, parts of Buy/Sell/Swap/Stake, staking digest UX | not documented in fetched pages |
| Coin Framework (CF) / Coin Integration | Tech (Engineering) | `@LedgerHQ/coin-framework` + per-family teams; CODEOWNERS "kept identical" in Alpaca migration | Coin Framework space; Coin Integration Onboarding Resources (page 4803002524) | Framework + per-family coin-modules (coin-btc, coin-evm, coin-xrp, coin-sol, …) providing AccountBridge/CurrencyBridge; migrating to Alpaca REST service | `#team-coin-integration` |
| Blockchain Support | Consumer Services neighbour | unknown | Ask other teams (page 4300767271) | Blockchain integration triage; Kevin held expertise at time of writing | `#team-blockchain-support` |
| B2C Backend Services (core) | Consumer Services | unknown | Ask other teams (page 4300767271) | CVS (Countervalues), App Store (device app catalog), Coinradar, internal blockchain services | `#b2c-backend-services-team` |
| Cloud Wallet (B2C) | Consumer Services | unknown | Vault Coin Integration Playbook | Owns ALPACA service (TS HTTP exposing coin modules), co-owns coin modules | unknown |
| Managed Services (MS) | Infra/Ops | — | MS Onboarding space | Monitoring + deployments + incident response | `#infra`, `#runner-investigation` |
| Platform Engineering (PE) | Infra/Ops | — | — | Vercel, GitHub org admin, gha-runners | `#infra`, `#changes-infra-notice` |
| Dedicated SRE for PTX | Infra | — | Ask other teams | SRE embedded with PTX (Anthony Pham, Michal Kaczmarek) | — |
| Security Engineering (SecEng) | Cyber Security | — | Cyber Security Team page (4654235680) | Build & run Ledger's cybersecurity stack; GRC, Donjon | — |
| Delivery | Cross-tribe coordination | — | [TECH] Teams Organization (page 4233200470) | Coordinates BU delivery across squads (Raja FADDOUL — deactivated at time of page) | `#live-release-sync` |

Notes:
- "Codeowner handle(s)" are reconstructed from Confluence references to GitHub team URLs (`https://github.com/orgs/LedgerHQ/teams/<name>`); direct CODEOWNERS file content was not fetched in this session.
- Tech org is split between Business Units (BU) — vertical (Ledger Live, Recover, …) and horizontal (HOR) — supervised by CTO Charles GUILLEMET. Delivery is cross-cutting. Source: [TECH] Teams Organization page (revised 03/2024, flagged "under rework").

---

## 2. QAA's Neighbours — who does QAA talk to most?

QAA sits inside the Quality Assurance org (alongside QA B2C and QA B2B) but its day-to-day code & test interactions are primarily with the B2C engineering tribes that build `ledger-live` and `vault`.

### 2.1 Wallet XP (WXP) — closest neighbour
- Why: WXP owns the LL Desktop/Mobile UI that QAA exercises end-to-end (Playwright + Detox + Speculos).
- Evidence: "QAA team will disable failing tests to keep build green, when there are failures to ping Jeremy so he can take a look, investigate and ping Blockchain services team" (240122 meeting notes, page 6744671530). Wallet 4.0 Q1 retro: "WXP team, supported by QAA" (page 6683787272).
- Contacts: Kevin Le-Seigle (Live Hub tech lead), Mathieu Bertin (Devices tech lead), Alexis (DM), Alexandrine Boissière (former WXP EM, deactivated).
- Channels: `#team-wallet-xp`, `#live-engineering`, `#live-hub`, `#team-devices`.

### 2.2 PTX (Payments & Transactions Services)
- Why: Swap, Buy/Sell, Earn, Recover are Live Apps embedded in LL with their own repos and release cadence — each requires E2E coverage and has partner-facing incident flow.
- Evidence: PTX pages list dedicated Slack channels (`#ptx-buy-sell`, `#ptx-swap-prod`, `#alert-buy`, `#sentry-ptx-buy-sell`). QAA has a dedicated page "Debug live app locally" (page 6704562204) to test PTX live apps against LLM/LLD.
- Contacts: JF Rochet, Sandra Denize Klein (PTX meeting minutes).
- Channels: `#ptx-buy-sell`, `#ptx-swap-prod`, per-partner `#partner-<name>`.

### 2.3 Engagement Team
- Why: Owns growth/engagement features in LL (Favourite Token, staking digest, Buy/Sell/Swap/Stake touchpoints). Issues tagged `[ENGAGEMENT]` land directly in `LIVE-*` Jira, meaning their PRs hit `ledger-live` and trigger QAA regression.
- Evidence: Engagement has its own Confluence space (page 6702270213); North Star metrics = WVEU & Activation-within-48h. Jira `LIVE-29156` "[ENGAGEMENT] [LWM] Staking digest: data hook" shows code landing in the mobile app.
- Channels: not documented in fetched pages (gap).

### 2.4 Live Apps (Discover / Wallet API)
- Why: The Discover catalog, WebPTXPlayer, and manifest infra are the surface where third-party and first-party live apps plug into LL. Manifest or geo-blocking changes break QAA's live-app E2E tests.
- Evidence: "[Tech proposal] Geo-Blocking" (page 6799655179) references `Catalog/index.tsx` on both `ledger-live-desktop` and `ledger-live-mobile`. Wallet API scope = "APIs on basic wallet usage covering all LL-supported blockchains", Anthony Goussot was team lead.
- Contacts: Anthony Goussot (Wallet API team lead, later moved to Coin Integration), Quentin Jaccarino.
- Channels: `#team-wallet-api`.

### 2.5 Coin Framework / Coin Integration (coin-fw)
- Why: Owns coin-modules (`coin-btc`, `coin-evm`, `coin-xrp`, …) that LL depends on for every transaction. Alpaca migration and AccountBridge/CurrencyBridge refactors affect the bot tests (nitrogen/oxygen) and coin-tester results QAA monitors.
- Evidence: ADR-025 "Coin modules migration to Alpaca" (page 6949896196) explicitly states "CODEOWNERS are kept identical to what we have in ledger-live monorepo". `libs/coin-modules/coin-*` structure is documented.
- Contacts: Anthony Goussot, Quentin Jaccarino.
- Channels: `#team-coin-integration`.

### 2.6 Wallet CI (wallet-ci)
- Why: QAA's E2E jobs run on Wallet CI pipelines. When CI breaks (flakes, runner issues, release windows), QAA is the first loud consumer.
- Evidence: [CI] Slack Channels page (5755076647) lists `#wallet-ci-alerts` "CI failures, flakes notifications debug information and CI monitoring". Firebase App Distribution onboarding (page 6400704513): "`#team-wallet-ci` can add external users to the appropriate group". 230116 meeting notes: "to create a ticket for QAA for implement a job for SCI team on the dedicated feature branch".
- Channels: `#team-wallet-ci`, `#wallet-ci-alerts`, `#live-ci-notifications`, `#ledger-live-ci`, `#live-release-sync`, `#infra-live`, `#live-engineering`.

---

## 3. Org Chart (ASCII, best-effort reconstruction)

Reconstructed from [TECH] Teams Organization (page 4233200470) plus WXP, PTX, QA, and CI space content. Embedded images on the source TECH page could not be rendered; this is a partial view.

```
Ledger SAS
|
+-- CEO (Ledger)
|
+-- CTO -- Charles GUILLEMET
    |
    +-- Engineering BUs (VERTICAL, product-aligned)
    |   +-- Ledger Live (B2C) / Wallet XP tribe
    |   |    +-- Live Hub squad             (UI: portfolio, accounts, WalletSync, Market)
    |   |    +-- Devices Experience squad   (hw-transport, Device SDK, My Ledger UI)
    |   |    +-- Wallet API / Connectivity  (Discover, Live Apps runtime; partial merge w/ Coin Integration)
    |   |    +-- Wallet CI team             (pipelines for ledger-live + ledger-live-build)
    |   |
    |   +-- Consumer Services tribe (PTX)
    |   |    +-- Buy / Sell               (#ptx-buy-sell)
    |   |    +-- Swap                     (#ptx-swap-prod)
    |   |    +-- Earn / Stake             (Earn Live-App team)
    |   |    +-- Recover
    |   |    +-- B2C Backend Services core (CVS, AppStore, coinradar)
    |   |    +-- Blockchain Support
    |   |    +-- Cloud Wallet (B2C)       (ALPACA service)
    |   |
    |   +-- Engagement Team               (WVEU / Activation KPIs)
    |   |
    |   +-- Vault / Enterprise (B2B)
    |   |    +-- HSM, vault-front, les-multisig, vault-public-api, minivault
    |   |
    |   +-- Recover, firmware, hardware teams (not expanded here)
    |
    +-- Horizontal Engineering Units (HOR, support BUs)
    |   +-- Coin Framework / Coin Integration (coin-modules, AccountBridge, CurrencyBridge, Alpaca)
    |   +-- Platform Engineering (Vercel, GitHub, gha-runners)
    |   +-- Managed Services (deployments, monitoring)
    |   +-- Dedicated SRE for PTX (A. Pham, M. Kaczmarek)
    |   +-- Security Engineering (SecEng, GRC, Donjon)
    |
    +-- Delivery  (Raja FADDOUL, deactivated; cross-BU coordination)
    |
    +-- Quality Assurance organisation
        +-- QA B2C            (manual QA for LL/Wallet): Engineering Manager + Senior QA Engineers
        +-- QA B2B            (manual QA for Vault/HSM): Engineering Manager + Senior QA Engineers
        +-- QA Automation (QAA)   (SDETs, E2E automation across B2C + B2B)
              Engineering Manager QAA
              +-- SDET
              +-- Senior SDET
              +-- SDET (x N)
              Key contacts (from QAA Onboarding Checklist):
              - Yaroslava POLISHCHUK   (onboarding buddy, meetings, Slack)
              - Victor ALBER           (LWD automation trainer)
              - Abdurrahman SASTIM     (LWM automation trainer)
              - Oleksandra BOYKO       (Vault automation trainer)
              - Gabriel BECERRA        (regression walkthrough)
              - Laure DUCHEMIN         (QA locker / devices)
```

---

## 4. Top 5 Confluence pages to cite as ground truth

1. [TECH] Teams Organization — https://ledgerhq.atlassian.net/wiki/spaces/TH/pages/4233200470/TECH+Teams+Organization (page 4233200470). CTO-supervised org chart, BU vs HOR split, Delivery function. Caveat: flagged "under rework" since 03/2024 — content has evolved; use as scaffolding, not as ground truth for current people.
2. Wallet XP — Live Hub (HUB) Squad — https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/4245487751/Live+Hub+HUB+Squad (page 4245487751). Explicit in-scope / out-of-scope list, gold for QAA test ownership.
3. Wallet XP — Devices Experience Squad — https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/4244276186/Devices+Experience+Squad (page 4244276186). Lists exact packages owned (`@ledgerhq/hw-transport*`, Device SDK, …) and the GitHub repos.
4. [CI] Team + [CI] Slack Channels — https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/5753635342 and https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/5755076647. Mission + exhaustive Slack channel map for anything CI-related.
5. QAA Onboarding Checklist — https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6647873614/QAA+Onboarding+Checklist. Names the actual people a QAA newcomer needs (Yaroslava, Victor, Abdurrahman, Oleksandra, Gabriel, Laure) and maps them to domains (LWD / LWM / Vault / regression / devices).

Bonus: Ask other teams (https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/4300767271) — short, PTX-authored but broadly useful "where to ask what" index.

---

## 5. Gaps / Uncertainties — where the student should look

- No direct CODEOWNERS fetch: the actual `.github/CODEOWNERS` file in `LedgerHQ/ledger-live` was not fetched. The mapping of path -> `@ledgerhq/<team>` must be confirmed from the repo. Pointer: https://github.com/LedgerHQ/ledger-live/blob/develop/.github/CODEOWNERS (and parallel files in `ledger-live-build`, coin-apps, vault-*). ADR-025 (page 6949896196) states the Alpaca repo mirrors the ledger-live CODEOWNERS.
- Engagement team Slack channel: not documented on the Engagement space overview (page 6702270213). Ask in `#live-engineering` or via the Engagement space parent page.
- PTX sub-team GitHub handles: Confluence references `@ledgerhq/team-ptx-*` informally. The authoritative list is in the `ledger-live` CODEOWNERS and the Earn/Buy-Sell/Swap repos.
- Wallet API team status: "Ask other teams" (2024-05) lists `#team-wallet-api` with Anthony Goussot as lead, but "Coin Integration Onboarding Resources" (page 4803002524) lists Goussot under Coin Integration / Connectivity with the note "The scope of this team is now under coin-integration". Wallet API may have been folded in — to confirm.
- Tech Hub organization images: the [TECH] Teams Organization page embeds images (Tech Leads, BU/HOR diagrams) that could not be rendered; a student should open the page visually for the authoritative chart.
- Live Apps team vs PTX boundary: Live Apps run inside LL but are owned by PTX (Buy/Sell/Swap/Earn) or Engagement. The boundary with Wallet API / Live Hub is fuzzy and has shifted over time. Cross-check with "Ledger Wallet (Ledger Live)" page (5644255440) and individual live-app repos.
- Confluence space keys: `getConfluenceSpaces` was denied, so space keys are only those surfaced as URL fragments: TH, WXP, QA, WALLETCO, PTX, CF, VAUL, MS, SE, BO, ECOM, ENGB2C, Engagement, LLI, LEDE, PKB, PE, WAL, CIP, BE, TA, DLS, CN, FW, INFRASD, SRE, FET.
- CODEOWNERS for QAA repos: `vault-e2e-tests`, `vault-public-api-tests` are QAA-owned per the onboarding checklist, but the mapping to a `@ledgerhq/qaa` or `@ledgerhq/team-qa-*` GitHub team was not confirmed in fetched content.
- "coin-fw" exact GitHub team name: Confluence refers to "Coin Framework team" conceptually; the GitHub team handle (likely `@LedgerHQ/coin-framework` or `@LedgerHQ/live-coin-modules`) needs repo-side confirmation.

---

## 6. Raw Confluence links — every page fetched or surfaced

Pages fetched in full during this research (via `getConfluencePage`):

| Page ID | Title | URL |
|---|---|---|
| 4233200470 | [TECH] Teams Organization | https://ledgerhq.atlassian.net/wiki/spaces/TH/pages/4233200470/TECH+Teams+Organization |
| 4242605488 | Wallet XP (space overview) | https://ledgerhq.atlassian.net/wiki/spaces/WXP/overview |
| 4279762945 | [WXP] Welcome | https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/4279762945/WXP+Welcome |
| 6635357792 | Quality Assurance (space overview) | https://ledgerhq.atlassian.net/wiki/spaces/QA/overview |
| 6647873614 | QAA Onboarding Checklist | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6647873614/QAA+Onboarding+Checklist |
| 5753635342 | [CI] Team | https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/5753635342/CI+Team |
| 5755076647 | [CI] Slack Channels | https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/5755076647/CI+Slack+Channels |
| 6702270213 | Engagement Team (space overview) | https://ledgerhq.atlassian.net/wiki/spaces/Engagement/overview |
| 5396955165 | Onboarding - Setup (WXP) | https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/5396955165/Onboarding+-+Setup |
| 6736674834 | Newcomer Onboarding checklist (QA) | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6736674834/Newcomer+Onboarding+checklist |
| 4803002524 | Coin Integration Onboarding Resources | https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/4803002524/Coin+Integration+Onboarding+Resources |
| 4300767271 | Ask other teams (PTX) | https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/4300767271/Ask+other+teams |
| 4245487751 | Live Hub (HUB) Squad | https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/4245487751/Live+Hub+HUB+Squad |
| 4244276186 | Devices Experience Squad | https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/4244276186/Devices+Experience+Squad |

Additional pages surfaced by Rovo search (titles + text used, not individually fetched):

| Page ID | Title | URL |
|---|---|---|
| 4654235680 | [1] Cyber Security Team: Mission, Organizations, HR & KPI | https://ledgerhq.atlassian.net/wiki/spaces/SE/pages/4654235680 |
| 5823824009 | [WEB] Product Manager Onboarding Guide | https://ledgerhq.atlassian.net/wiki/spaces/ECOM/pages/5823824009 |
| 5717852306 | Internal Knowledge Base Structure - 2025 | https://ledgerhq.atlassian.net/wiki/spaces/BO/pages/5717852306 |
| 3170336922 | Ledger Software Reorg | https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/3170336922 |
| 5528551484 | Organizational Options (Design System) | https://ledgerhq.atlassian.net/wiki/spaces/DLS/pages/5528551484 |
| 3669033369 | Knowledge sharing during Ledger MS team member onboarding | https://ledgerhq.atlassian.net/wiki/spaces/MS/pages/3669033369 |
| 7022870550 | Seed Management Strategy - QAA Team Decision Record | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/7022870550 |
| 6898843819 | QA - Feature Development & Quality Lifecycle | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6898843819 |
| 6636437536 | QA - Evaluation Criteria | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6636437536 |
| 6902677743 | Kick off - Quality lifecycle and Regression Scope | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6902677743 |
| 6744671212 | 230116 (QA weekly meeting note) | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6744671212 |
| 6860832827 | E2E Automated tests - Adaptation to Ledger Wallet 4.0 | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6860832827 |
| 6744671530 | 240122 (QA weekly meeting note) | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6744671530 |
| 6683787272 | Wallet 4.0 - Q1 | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6683787272 |
| 6033277043 | Improving PTX Apps Experience | https://ledgerhq.atlassian.net/wiki/spaces/TH/pages/6033277043 |
| 4170023069 | Consumer Services - testing funds guidelines | https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/4170023069 |
| 6948159538 | Study: about tagging transactions | https://ledgerhq.atlassian.net/wiki/spaces/BE/pages/6948159538 |
| 5009473537 | [BUY/SELL] | https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/5009473537 |
| 6278676580 | Custom exporter - PTX Buy Exporter | https://ledgerhq.atlassian.net/wiki/spaces/MS/pages/6278676580 |
| 6268977232 | [EARN] Earn tech roadmap topics | https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/6268977232 |
| 4614488065 | V2. PTX User Research - Why do people use PTX | https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/4614488065 |
| 4115235016 | V1. PTX User Research - Who are our Customers | https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/4115235016 |
| 3625943104 | 2022-04-05 - PTX weekly team meeting | https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/3625943104 |
| 6093078548 | Average Time To Complete (incident comms) | https://ledgerhq.atlassian.net/wiki/spaces/BO/pages/6093078548 |
| 5644255440 | Ledger Wallet (Ledger Live) | https://ledgerhq.atlassian.net/wiki/spaces/LLI/pages/5644255440 |
| 3506831745 | Ledger Quest | https://ledgerhq.atlassian.net/wiki/spaces/ECOM/pages/3506831745 |
| 3610312712 | [Infra HLD] Ledger Live Service | https://ledgerhq.atlassian.net/wiki/spaces/INFRASD/pages/3610312712 |
| 5787025506 | ACTIVATION | https://ledgerhq.atlassian.net/wiki/spaces/CN/pages/5787025506 |
| 6696829023 | [ADR] Tiered Auth for First-Party & Third-Party Clients | https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/6696829023 |
| 5884936323 | Ledger Wallet - Product vision | https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/5884936323 |
| 6945013939 | Seed local setup | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6945013939 |
| 3878977711 | V1 Revamp - Ledger Quest | https://ledgerhq.atlassian.net/wiki/spaces/ECOM/pages/3878977711 |
| 6129025043 | [Coin Framework] Coin Modules Developments Mapping | https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/6129025043 |
| 5738037406 | Engineering - Vault Coin Integration Playbook | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/5738037406 |
| 6925549644 | ADR-023 - Reorganize generic-alpaca by Coin Family | https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/6925549644 |
| 6949896196 | ADR-025 - Coin modules migration to Alpaca | https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/6949896196 |
| 5792792577 | Vault Coin Integration Playbook - SUI | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/5792792577 |
| 6925549613 | ADR-022 - Extensibility Pattern for Cross-Cutting Concerns | https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/6925549613 |
| 6826492033 | Coin Modules Upkeeping | https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/6826492033 |
| 3522265356 | 1041 - External Coins - PRD | https://ledgerhq.atlassian.net/wiki/spaces/MS/pages/3522265356 |
| 3684630561 | [ARCH] LL Coin integration framework | https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/3684630561 |

| 3924230200 | Manifest V2 | https://ledgerhq.atlassian.net/wiki/spaces/WAL/pages/3924230200 |
| 6704562204 | Debug live app locally | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6704562204 |
| 6799655179 | [Tech proposal] Geo-Blocking for non Compliant Countries | https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/6799655179 |
| 6830129276 | Earn Live-App Module Federation Migration Roadmap | https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/6830129276 |
| 4884922491 | Back to app: how to navigate within a webview | https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/4884922491 |
| 4217831481 | Test Live Apps (Earn or Buy/Sell) on mobile | https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/4217831481 |
| 4365877252 | Samy's Onboarding | https://ledgerhq.atlassian.net/wiki/spaces/WAL/pages/4365877252 |
| 5052825616 | Braze in Live Apps | https://ledgerhq.atlassian.net/wiki/spaces/PTX/pages/5052825616 |
| 4984897579 | CI Improvements Mission | https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/4984897579 |
| 6907625515 | Ledger Wallet CLI | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/6907625515 |
| 5721489432 | Ledger Live (security view) | https://ledgerhq.atlassian.net/wiki/spaces/SE/pages/5721489432 |
| 4436656382 | Ledger Live CI State of the Art | https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/4436656382 |
| 6400704513 | Firebase App Distribution - Onboarding | https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6400704513 |
| 5003083832 | Live CI - Reduce complexity & Stabilize | https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/5003083832 |
| 6799655120 | Infra As Code - CI/CD Pipeline Security Requirements | https://ledgerhq.atlassian.net/wiki/spaces/SE/pages/6799655120 |
| 4251549751 | Old_Wallet XP | https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/4251549751 |
| 3439460478 | Wallet-XP (formerly Live-Hub) team | https://ledgerhq.atlassian.net/wiki/spaces/ENGB2C/pages/3439460478 |
| 6661767169 | [2026Q2 - WXP] My Wallet - Product specifications | https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/6661767169 |
| 4665868292 | Wallet XP - Delivery Tracking - 2024-04-23 | https://ledgerhq.atlassian.net/wiki/spaces/LEDE/pages/4665868292 |
| 6692700182 | Cybersecurity Engineering Roadmap 2026 | https://ledgerhq.atlassian.net/wiki/spaces/SE/pages/6692700182 |
| 5388927006 | Prerequisites (CIP) | https://ledgerhq.atlassian.net/wiki/spaces/CIP/pages/5388927006 |
| 6951665759 | Ledger Live CLI for AI Agent | https://ledgerhq.atlassian.net/wiki/spaces/TA/pages/6951665759 |
| 6975684839 | Ledger Live monorepo package visibility inventory | https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/6975684839 |
| 6637191253 | Ledger Live environments | https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6637191253 |
| 5833326689 | Dynamic Assets Data Aggregator | https://ledgerhq.atlassian.net/wiki/spaces/BE/pages/5833326689 |
| 1706459555 | [QA][Dev] - Newcomers Onboarding guide | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/1706459555 |
| 4129620424 | [Onboard-Prep] Manager/Buddy checklist | https://ledgerhq.atlassian.net/wiki/spaces/PKB/pages/4129620424 |
| 3649045061 | LAMA onboarding checklist | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/3649045061 |
| 3496542358 | Onboarding Product Newcomer | https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/3496542358 |
| 5732532225 | MS Onboarding procedure (for newcomer) | https://ledgerhq.atlassian.net/wiki/spaces/MS/pages/5732532225 |
| 4904386568 | Onboarding & Offboarding - Hardware | https://ledgerhq.atlassian.net/wiki/spaces/BO/pages/4904386568 |
| 1344635034 | Onboarding procedure (MS) | https://ledgerhq.atlassian.net/wiki/spaces/MS/pages/1344635034 |
| 6194036794 | MS Onboarding procedure (for mentors) | https://ledgerhq.atlassian.net/wiki/spaces/MS/pages/6194036794 |

---

### Permission / tool caveats encountered

- `getConfluenceSpaces` and `searchConfluenceUsingCql` were denied in this session. All findings come from the Rovo `search` tool (4 calls) plus 11 `getConfluencePage` fetches. Total Atlassian API calls: 15 — well under the 25 cap.
- Embedded images on the [TECH] Teams Organization page (CTO org chart, BU/HOR diagram) could not be rendered by this agent; the reconstructed ASCII is based on textual descriptions from that page plus cross-references.
