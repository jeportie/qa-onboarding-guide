# R3 — Ledger Live Release Trains, Environments, Firebase, Feature Flags, Hotfixes, Semantic-Release

Research agent: R3
Scope: Ledger Live (Desktop + Mobile) release cadence, environments, Firebase projects,
feature flags, hotfix lanes, semantic-release / versioning.
All facts are sourced from Confluence pages listed at the bottom. Lines marked [Src N]
refer to the numbered sources in section 9.

---

## 1. Release cadence

### 1.1 Cadence observed (from release-page numbering)

LLM (Ledger Live Mobile) uses minor version bumps per release train. Recent trains
observed include LLM 3.82, LLM 3.88, LLM 3.92, LLM 3.94, LLM 3.94.1 (hotfix),
LWM 3.99, LWM 3.101, LWM 3.103. [Src 11, 12, 13, 14, 15, 17, 18, 21]

LLD (Ledger Live Desktop) follows a parallel numbering: LLD 2.94, LLD 2.118.1 (hotfix),
LLD 2.126, LLD 2.128, LLD 2.128.1 (hotfix), LLD 2.130, LLD 2.131, LWD 2.137,
LWD 2.139. [Src 1, 4, 8, 9, 10, 16, 19, 20, 22]

The project uses two app-name prefixes interchangeably: LLM / LLD (older names,
Ledger Live Mobile / Desktop) and LWM / LWD (newer names, Ledger Wallet Mobile /
Desktop). [Src 17, 18, 21, 22, 26]

### 1.2 Alignment — "Dual release"

The hotfix-procedure page describes the canonical pair as "a dual release": one train
produces BOTH an LLM build and an LLD build simultaneously. Example cited:
"release N produced LLM v3.30.0 and LLD v2.26.0". [Src 23]

Sometimes only one app is deployed on a given train (LLM-only or LLD-only). The hotfix
doc enumerates use cases such as "a dual release just went out", "we just released a
LLM build on its own (release N)", and "we need to hotfix the LLD build that was
released previously (release N-1)" — confirming that dual releases are the default but
single-app releases do occur. [Src 23]

### 1.3 Weekly milestones within a train

The LWM 3.103 and LWD 2.137 release checklists both require Smartling translation jobs
to have "an ETA set to Wednesday 5PM at latest", implying a Wednesday translation
cutoff within the release week. [Src 2, 3]

### 1.4 Facts not found

- Exact number of weeks between trains — not stated on any fetched Confluence page.
  The live cadence sheet is an external Google Sheet referenced from the release pages.
- Explicit "release calendar" page in Confluence — not found via Rovo search.

---

## 2. Release train diagram

```
                         LEDGER LIVE RELEASE TRAIN (per release N)
                         ------------------------------------------

  +---------------+  dev work                  [Firebase: development]
  |   develop     | <-- feature branches          [env: .env / dev]
  |   (branch)    |     (feature flagged)           editable by devs
  +-------+-------+
          |
          |  Train Delivery Lead triggers
          |  release-create.yml workflow [Src 5]
          v
  +---------------+  Phase 1: Prerelease Launch   [Firebase: staging]
  |   release     |  - Release branch created       [env: .env.staging]
  |   (branch)    |  - PR "Release ..." opened      read-only for devs
  |               |  - Builds on ledger-live-build   editable by QAE/PM
  |               |    (prerelease)
  +-------+-------+  - Sentry monitoring
          |          - Smartling translation jobs
          |  Phase 2: Release test Campaign (QA)
          |          - Blocker fixes land on
          |            release + cherry-pick to develop
          |          - QA posts GO in #live-releases-org
          |  Phase 3: GO for release — POs ack
          |          - Feature-flag values in
          |            ledger-live-production audited before GO
          v
  +---------------+  Phase 4: Merge & sign         [Firebase: production]
  |     main      |  - release -> main             [env: .env.production]
  |   (branch)    |  - release -> develop           read-only for devs
  |               |  - Exit prerelease mode          editable by PM/Delivery
  |               |  - Create prod builds in ledger-live-build
  |               |  - Publish npm packages
  |               |  - LLD: Quorum signing
  |               |  - LLM: Android Play Console / iOS App Store submission
  +-------+-------+
          |
          v
    +----------+   LLD: Desktop Sign and Publish workflow pushes to CDN
    |   USERS  |   LLM: Android 10% staged rollout -> 100%; iOS 100%
    +----------+   Post-release Sentry monitoring on prod project

  HOTFIX LANE (parallel):                             [Src 23]

  +---------------+  release-create-hotfix.yml
  |     main      | ----------------------------> +---------------+
  | (last prod)   |  Cherry-pick fix + patch      |    hotfix     |
  +---------------+  changeset. NO SQUASH merges. |   (branch)    |
                                                  +-------+-------+
                                                          |
                                  Same phases 2-4 as release, then:
                                  hotfix -> main AND hotfix -> develop
```

Sources for flow: [Src 2, 3, 5, 7, 23, 24].

Key 2024 decision: prod and prerelease builds are the SAME artifact — "No more way to
distinguish a prod vs a prerelease build (= no more -next dist tags)." The version
published to users is only known when QA issues a GO. [Src 5]

---

## 3. Firebase environment matrix

The Feature Flagging page lists four Firebase projects. [Src 6]

- Development / ledger-live-development / local dev builds, feature-branch PRs / Devs Editor, EM Editor, QAE Product Delivery Viewer / enabled:true / Local dev only
- Testing / ledger-live-testing / QA E2E suites and mock tests / QAE Editor, EM Editor, Devs Product Delivery Viewer / not specified / Internal test runs
- Staging / ledger-live-staging / Beta builds and prerelease QA campaign / all roles Editor / enabled:false / Recover Staging, LedgerLive Beta build [Src 25]
- Production / ledger-live-production / Shipped prod LLD and LLM / Product Editor, EM and Delivery Editor backup, Devs and QAE Viewer / enabled:false / Recover Prod; Slack qa-b2c-releases-bugs-tracking

Feature flags are NOT frozen on production. Prod Firebase Remote Config is mutable;
changes are monitored (not blocked) in Slack qa-b2c-releases-bugs-tracking via the
GitHub repo LedgerHQ/qa-b2c-release-watcher. [Src 7]

### 3.1 ENVFILE and .env mapping

Per the Feature Flagging page section Switching environment [Src 6]:

Desktop (LLD) - loaded by renderer.webpack.config.js:getDotenvPathFromEnv() and
src/firebase-setup.js:

- .env (default) -> ledger-live-development
- .env.testing -> ledger-live-testing
- .env.staging -> ledger-live-staging
- .env.production -> ledger-live-production

Mobile Android - build-variant driven:

- android/app/google-services.json (debug) -> ledger-live-development
- android/app/src/stagingRelease/google-services.json -> ledger-live-staging
- android/app/src/release/google-service.json -> ledger-live-production

Mobile iOS - selected by GOOGLE_SERVICE_INFO_NAME env var set in the matching
.env.staging / .env.production file; switched in AppDelegate.m:

- ios/GoogleService-Info.plist (default/dev) -> ledger-live-development
- ios/GoogleService-Info-Staging.plist -> ledger-live-staging
- ios/GoogleService-Info-Production.plist -> ledger-live-production

### 3.2 Env vars used by QA (non-Firebase) - from Ledger Live environments page [Src 4]

ANALYTICS_LOGS=1, ANALYTICS_CONSOLE=1, DEV_TOOLS=1, EXPERIMENTAL_CURRENCIES=...,
EXPERIMENTAL_EXPLORERS=1, MOCK=1, DEBUG_THEME=1, DEBUG_UPDATE=1,
DEBUG_SWAP_STATUS, DEVICE_PROXY_URL=ws://..., SWAP_API_BASE=https://swap.staging.aws.ledger.fr,
SWAP_DISABLED_PROVIDERS=..., DISABLE_TRANSACTION_BROADCAST=1,
VERBOSE=error/warn/info/http/verbose/debug/silly, and per-currency overrides like
API_COSMOS_BLOCKCHAIN_EXPLORER_API_ENDPOINT=...

The canonical list lives in ledger-live-common/src/env.ts (envDefinitions). [Src 4]

Gap: the requested tuple .env.mock / .env.mock.prerelease was NOT found verbatim
in any Confluence page. The official files documented are .env, .env.testing,
.env.staging, .env.production. [Src 6]

---

## 4. Feature flags catalog (QAA-facing)

### 4.1 Naming convention [Src 6]

- Firebase key: feature_{awesome_feature_name} - snake_case, feature_ prefix
  (e.g. feature_llm_usb_firmware_update).
- Codebase key: {awesomeFeatureName} - camelCase, no prefix (e.g. llmUsbFirmwareUpdate).
- Conversion in code: feature_ + _.snakeCase(camelCaseId). Care needed around tokens like
  TRC20 because snakeCase/camelCase are not bijective. [Src 6]

### 4.2 Flag value schema [Src 6]

Always JSON (never bare booleans), minimum: enabled true or false.
Version gating supported via: mobile_version ">=v3.37.0" or desktop_version.
Conditions: desktop_device, ios_device, android_device, plus Firebase-native
"User in random percentile" (used for percentage rollouts).

### 4.3 Concrete flag families cited on Confluence

I could NOT find a single authoritative top-flags catalog page in Confluence.
Individual flags named on the pages I fetched:

- feature_market / market - example used in the flag-creation checklist [Src 6]
- feature_learn / learn - example used in the API-usage docs [Src 6]
- feature_llm_usb_firmware_update / llmUsbFirmwareUpdate - naming-convention example [Src 6]
- ptxSwapReceiveTRC20WithoutTrx - conversion-edge-case example [Src 6]
- ptxEarn - cited in the How to activate a Feature Flag tutorial as the search term for the Beta Earn Dashboard on LLD [Src 26]
- ldmkTransport - cited in the How to activate the feature flag in Ledger Live Desktop tutorial [Src 27]
- feature_protect_services_desktop, feature_protect_services_mobile - Recover / Ledger Protect flag families, documented with keys enabled, protectID (values: protect-local, protect-local-dev, protect-staging, protect-prod), staxCompatible, ledgerliveStorageState, bannerSubscriptionNotification, openRecoverFromSidebar, compatibleDevices, onboardingRestore, onboardingCompleted, deep-link URIs (upsellURI, learnMoreURI, alreadySubscribedURI, alreadyDeviceSeededURI, restore24URI, homeURI, loginURI). [Src 25]
- feature_currency_* - per-currency toggles; inconsistent schema (some platform-specific, some global enable/disable only) - Anti-Pattern 1 [Src 7]
- config_nanoapp_* - controls device deprecation, min firmware versions, device warning/error screens, node and explorer URIs, NFT display - God config Anti-Pattern 2; shared between DMK team (device lifecycle) and Coin Integration (min firmware). [Src 7]
- config_nanoapp_zcash, config_currency_sonic_blaze - Anti-Pattern 3 examples of flags outside their expected grouping. [Src 7]
- mainNavigation, assetSection, brazePlacement, lwmVersion4 - LWM / Ledger Wallet v4.0 navigation flags, carrying implicit dependencies (Anti-Pattern 5). [Src 7]

Gap: ptxSwapLiveApp and llmNetworkBasedAddAccountFlow were NOT found on any
Confluence page via the Rovo search. They likely exist in code but are not
individually documented.

### 4.4 How to toggle a flag from the app (what QAA does daily) [Src 26]

LLD: Settings -> Developer tab -> Define feature flags used in Ledger Live -> Show
-> search (case-sensitive) -> toggle. If Developer tab hidden: About tab -> click
version number 10 times.

LLM: Settings -> About -> tap Powered by Ledger multiple times -> Debug section
appears -> Debug -> Configuration -> Feature flags -> toggle.

A missing default in defaultFeatures.ts will hide the flag from the in-app list -
used on purpose occasionally. [Src 6]

### 4.5 Known pitfalls relevant to QA [Src 7, 28]

- Firebase Remote Config is cached ~12h by default; minimumFetchIntervalMillis=0 in DEV.
  To force-refetch on an installed app, QA must delete IndexedDB (Desktop) or
  uninstall/reinstall (Mobile). [Src 6]
- When Firebase rate-limits the app, Remote Config fetches fail and the app falls
  back to defaults - in practice feature flags are seen as off. Known source of
  E2E flakiness. Mitigations proposed: retire aged flags, define baseline in code,
  snapshot Remote Config templates at release time. [Src 28]
- Dependent flags (e.g. assetSection depends on mainNavigation) are not expressed in
  config - QA may end up exercising invalid combinations in production. [Src 7]

---

## 5. Hotfix policy

### 5.1 When and who

Hotfix lane is triggered when a blocker is found in a version already released to users.
The LLM 2.97.1 hotfix and LLD 2.118.1 / 2.128.1 hotfix pages show hotfixes follow the
EXACT SAME checklist as a normal release (Phases 0-5) - run by the same Train QA /
Train Engineer / Train Delivery trio on a hotfix branch. [Src 17, 22, 24]

### 5.2 How (branching mechanics) [Src 23, 2, 3]

- Trigger the release-create-hotfix.yml workflow (manual, not automatic).
- Workflow params:
  - Tag Version: latest if hotfixing the most recent release; explicit version
    (e.g. 2.25.0, 3.55.1) if hotfixing an N-1 release.
  - Application: LLM or LLD - matters only for cross-app hotfix scenarios.
- The workflow creates a hotfix branch from main (not from develop).
- Engineers cherry-pick the fix commit into hotfix, include a patch changeset only
  (no minor, no major).
- PR targets hotfix -> merge (NO SQUASH).
- A second PR cherry-picks the fix to develop (NO SQUASH).
- After QA GO: merge hotfix -> main AND hotfix -> develop.

### 5.3 Constraints [Src 23]

- Only one hotfix branch can exist at a time. Hotfixing both N (LLM) and N-1 (LLD)
  requires two sequential runs.
- LLD rollback is available via release-rollback-desktop.yml and impacts users only
  a few hours after it has been triggered. [Src 2]

### 5.4 Emergency coms chans [Src 2, 3]

Slack: live-releases-org, live-release-sync, live-ci-notifications,
live-prerelease-sentry-alerts, launchpad.

---

## 6. Semantic-release + main / develop branching

### 6.1 Branch model (observed, not labelled semantic-release) [Src 8, 2, 3]

Topology:

- develop -> release -> main (on GO: release merges to BOTH main and develop, no squash)
- main -> hotfix -> main + develop (on hotfix GO: same dual merge)

- All feature work merges into develop.
- Release branches (release/*) are forked from develop by release-create.yml.
  The Earn Release Process page requires: "The branch name MUST start with release/". [Src 8]
- Hotfix branches fork from main.

### 6.2 Versioning - Changesets (not classic semver commits) [Src 1]

Ledger Live uses Atlassian changesets (not Angular-style semantic-release) for
versioning and publishing packages. Per the CI State of the Art page: "Bonus
documentation: changeset (tool used for versioning and publishing packages)". [Src 1]

Each fix to a release or hotfix branch must include a changeset file. Hotfixes are
patch changesets only. [Src 23]

Commit messages are linted with commitlint.yml against the Conventional Commits spec. [Src 1]

### 6.3 Publishing artifacts [Src 1, 2, 3]

- Repo LedgerHQ/ledger-live (open source) - code, libs, preliminary release steps, npm publish.
- Repo LedgerHQ/ledger-live-build (internal) - production secrets, produces and signs
  LLD and LLM builds, runs prereleases and nightlies.
- Repo LedgerHQ/certificates (internal) - iOS certificates and profiles.

Four build kinds: nightly, hotfix, prerelease, production. [Src 29]

npm packages are published AFTER GO-to-prod now; they used to be published during
prerelease but that was stopped ("we stop publishing npm packages during the
prerelease phase as they may contain a bug"). [Src 5]

LLD prerelease auto-update was also removed - each prerelease has to be downloaded
fresh. [Src 5]

### 6.4 Auxiliary tooling [Src 1]

- github-bot / Orchestrator - decides which CI workflows run on a PR based on affected
  packages. (Gate+watcher model from Dec 2023 has been partially replaced per
  LedgerHQ/ledger-live pull 7356.)
- Release rollback workflow: release-rollback-desktop.yml. [Src 2]
- Release reference presentation: Google Slides deck
  1Wiztn_xInN6Lckw8U4qOenV_SoPhq20SVF-1IPeXfHU (linked from every release page). [Src 2, 3]

---

## 7. Top 5 Confluence URLs to cite

1. Feature Flagging (canonical) - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/3549298959/Feature+Flagging
2. LWD 2.137 - Ledger Live Desktop release (canonical release checklist) - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6667501580/LWD+2.137+-+Ledger+Live+Desktop+release
3. LWM 3.103 - Ledger Live Mobile release (canonical release checklist) - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6694010898/LWM+3.103+-+Ledger+Live+Mobile+release
4. Feature Flags and Firebase Config Models (QA Space - anti-patterns) - https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6896255151/Feature+Flags+and+Firebase+Config+Models
5. How to hotfix Ledger Live - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/5234032693/How+to+hotfix+Ledger+Live

Honourable mentions: Ledger Live CI State of the Art (4436656382); Ledger Live
environments (QA, 6637191253); LWD/LWM How to activate a Feature Flag (LLI, 4206690378).

---

## 8. Gaps

The following items requested by the brief are NOT resolvable from Confluence pages
available to this research run:

1. Exact release cadence in weeks. No Confluence page states "every N weeks" or gives
   a recurring calendar rule. The authoritative cadence sheet is an external Google
   Sheet linked from every release page but not fetchable through this toolset.
2. .env.mock and .env.mock.prerelease mapping. Documented dotenv files are only .env,
   .env.testing, .env.staging, .env.production. If .env.mock.* files exist, they are
   in the repo but not in Confluence.
3. A canonical top feature flags QAA deals with list. No such catalog page exists in
   Confluence. The two flags the brief names (ptxSwapLiveApp, llmNetworkBasedAddAccountFlow)
   produced zero Rovo hits. Recommendation: generate this list from the code
   (libs/ledger-live-common/src/featureFlags/*) rather than from Confluence.
4. Authoritative semantic-release page. The term semantic-release is not used in the
   Ledger Live docs - the tool is changesets. Anyone searching the onboarding guide
   for semantic-release will need the alias.
5. Alignment windows between LLD and LLM trains. Dual vs single-app releases are
   confirmed to happen, but no published rule says when each kind is used.
6. CQL search tool and Confluence space listing were denied during this run - only the
   Rovo search tool was available. Less-discoverable pages may not have been surfaced.
7. Admin-access RACI for production feature flags - listed as of 2023 in Src 6 with
   names. Current owners (2026) may differ; page was last modified 2025-06-27.

---

## 9. Raw Confluence links (sources)

- [Src 1]  Ledger Live CI State of the Art - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/4436656382/Ledger+Live+CI+State+of+the+Art (last mod 2025-01-15)
- [Src 2]  LWD 2.137 - Ledger Live Desktop release - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6667501580/LWD+2.137+-+Ledger+Live+Desktop+release (2026-01-20)
- [Src 3]  LWM 3.103 - Ledger Live Mobile release - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6694010898/LWM+3.103+-+Ledger+Live+Mobile+release (2026-01-20)
- [Src 4]  Ledger Live environments (QA space) - https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6637191253/Ledger+Live+environments (2026-01-23)
- [Src 5]  Ledger Live CI Prod-equals-Prerelease Builds - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/4505927737/Ledger+Live+CI+-+Prod+Prerelease+Builds (2025-07-29)
- [Src 6]  Feature Flagging - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/3549298959/Feature+Flagging (2025-06-27)
- [Src 7]  Feature Flags and Firebase Config Models (QA) - https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6896255151/Feature+Flags+and+Firebase+Config+Models (2026-03-20)
- [Src 8]  Earn Release Process - https://ledgerhq.atlassian.net/wiki/spaces/LEDE/pages/6623395916/Earn+Release+Process
- [Src 9]  LLD 2.130 - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6237028421/LLD+2.130+-+Ledger+Live+Desktop+release
- [Src 10] LLD 2.131 - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6281691137/LLD+2.131+-+Ledger+Live+Desktop+release
- [Src 11] LLM 3.82 - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/5891260417/LLM+3.82+-+Ledger+Live+Mobile+release
- [Src 12] LLM 3.88 - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6053724193/LLM+3.88+-+Ledger+Live+Mobile+release
- [Src 13] LLM 3.92 - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6139248641/LLM+3.92+-+Ledger+Live+Mobile+release
- [Src 14] LLM 3.94 - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6211960850/LLM+3.94+-+Ledger+Live+Mobile+release
- [Src 15] LLM 3.94.1 hotfix - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6262489089/LLM+3.94.1+-+Ledger+Live+Mobile+release
- [Src 16] LLD 2.128 - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6168182808/LLD+2.128+-+Ledger+Live+Desktop+release
- [Src 17] LLM 2.97.1 hotfix - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6462144829/LLM+2.97.1+hotfix+-+Ledger+Live+Mobile+release (2025-11-17)
- [Src 18] LWM-3.99 - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6519914687/LWM-3.99+-+Ledger+Live+Mobile+release
- [Src 19] LLD 2.94 - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/5284757505/LLD+2.94+-+Ledger+Live+Desktop+release
- [Src 20] LLD 2.126 - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6130794498/LLD+2.126+-+Ledger+Live+Desktop+release
- [Src 21] LWM-3.101 - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6573195272/LWM-3.101+-+Ledger+Live+Mobile+release
- [Src 22] LWD 2.139 - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6718849163/LWD+2.139+-+Ledger+Live+Desktop+release
- [Src 23] How to hotfix Ledger Live - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/5234032693/How+to+hotfix+Ledger+Live (2024-11-18)
- [Src 24] LLD 2.128.1 hotfix - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6211960962/LLD+2.128.1+hotfix+-+Ledger+Live+Desktop+release
- [Src 25] 1 - Firebase Feature Flags in Ledger Live and Recover (Trust Services) - https://ledgerhq.atlassian.net/wiki/spaces/TrustServices/pages/4413620396/1+-+Firebase+Feature+Flags+in+Ledger+Live+and+Recover (2023-12-08)
- [Src 26] LWD/LWM How to activate a Feature Flag - https://ledgerhq.atlassian.net/wiki/spaces/LLI/pages/4206690378/LWD+LWM+-+How+to+activate+a+Feature+Flag (2026-02-18)
- [Src 27] How to activate the feature flag in Ledger Live Desktop (Wallet Experience) - https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/5551849478/How+to+activate+the+feature+flag+in+Ledger+Live+Destop
- [Src 28] Feature Flag Issues - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6883016802/Feature+Flag+Issues (2026-03-13)
- [Src 29] Ledger Live Release Troubleshooting Cookbook - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/4057891034/Ledger+Live+Release+Troubleshooting+Cookbook (2023-12-26)
- [Src 30] LLD 2.118.1 hotfix - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/5949849601/LLD+2.118.1+hotfix+-+Ledger+Live+Desktop+release
- [Src 31] Live Environments (2021, outdated) - https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/1452147411/Live+Environments

Additional GitHub references cited in the above pages:

- github.com/LedgerHQ/ledger-live/wiki/Release-Process
- github.com/LedgerHQ/ledger-live/wiki/Changesets
- github.com/LedgerHQ/ledger-live-build/wiki/Workflows
- github.com/LedgerHQ/ledger-live-build/wiki/Quorum-Signing
- github.com/LedgerHQ/ledger-live-build/wiki/Rollback-Process-for-LLD
- github.com/LedgerHQ/qa-b2c-release-watcher (prod FF change monitor)

---

_End of R3 report._
