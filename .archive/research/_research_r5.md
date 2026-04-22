# R5 — QAA Team Charter, Test Strategy, Pyramid, Flake Policy, KPIs, Onboarding

Research scope: Quality Assurance (QA) team charter, test strategy, test pyramid, what QAA owns vs feature teams, flake policy, KPIs/coverage goals, intern/newcomer onboarding expectations, Slack/meeting cadence, and tool stack.

Primary space: `QA` (Quality Assurance). Secondary: `CIP` (Coin Integration Playbook), `LEDE` (Ledger Delivery), `CF` (Coin Framework), `WXP`, `VAUL`.

---

## 1. QAA mission and scope

From the QA space home page (pageId 6635357792, "Quality Assurance"):

> "In an ecosystem where security and 'Don't Trust, Verify' are our core tenets, the QA team serves as the final gatekeeper. Our mission is to assess whether every Ledger product and feature, from the firmware on our Secure Elements to the interface of our products, meets the highest standards of reliability and user experience. At Ledger, QA is about risk assessment. We provide the confidence needed to ship products and features to all our users."

The Engineering QA org is split into three sub-teams:

- QA B2C — consumer wallet (Ledger Live Desktop/Mobile, aka LWD/LWM).
- QA B2B — enterprise / Vault.
- QA Automation (QAA / SDET) — the dedicated automation engineering team that owns the E2E frameworks, CI execution, Speculos plumbing, Allure reporting and nightly run stability across LWD, LWM, and Vault.

QAA specifically provides the horizontal automation backbone (Playwright/Detox/Speculos, Allure, flaky-reporter tooling, nightly run orchestration) for all three pillars.

---

## 2. Test pyramid — as practiced

Source: "Testing strategy" (CIP space, pageId 6346375175).

| Level | Purpose | Tooling | Frequency |
|---|---|---|---|
| Unit | Validate logic at function/component level | Jest, React Testing Library | Run on each commit (CI) |
| Integration | Test interactions between modules/services | Jest + mocks, MSW, local backend | Run on each commit (CI) |
| E2E (Speculos) | Validate app behavior in real-like environment with device simulation | Playwright (desktop), Detox (mobile), Speculos | Nightly or pre-release |
| Manual | UX, edge cases, hardware behavior not easily automated | Xray test plans, QA checklists | Before release / when automation not feasible |

Explicit coverage target (from Testing strategy): "Aim for >80% coverage on critical modules" at the unit level. The pyramid is asymmetric by design — "lower-level tests are faster, more stable, and should form the majority of our coverage."

Observed reality (from QA space + "UI E2E Test Stability" pageId 6687817733):
- Desktop/iOS/Android UI E2E runs are never expected to be 100% green; they run against live services, real blockchains, Speculos, and 3rd-party providers (Swap/Buy/Sell/Earn).
- Nightly-only (and pre-release) cadence; tests execute on develop, not frozen code, so they are early-warning of product regressions.
- An archived strategy page (VAUL 4573757597) confirms the historical evolution: plan written by QA, reviewed with devs; E2E suite treated like a demo-gate smoke suite.

---

## 3. What QAA owns vs what feature teams own

Primary sources: "QA - Feature Development & Quality Lifecycle" (6898843819), "Kick off - Quality lifecycle and Regression Scope" (6902677743), "QA - Alignment - Test plans, test cases and regression scope" (6741917736), "DEV/QA Role and Responsibilities - Coin Integration" (CF 4734124042), "WXP & QA Collaboration" (4620812293).

### Feature team (Devs) own
- Unit + integration tests — developer-written, run per-commit. Weight 5 in QA Evaluation Criteria (CO03).
- Mocked E2E tests (Detox mocks, mocked Playwright) — devs write and maintain.
- Automation of regression tests identified by QA — per the "Regression Automation Ownership" table: "Development: Automate the regression tests identified by QA" (Kick off 6902677743).
- Pre-merge E2E green — CO06 "End to end testing has been successfully run on branches before merging to develop" (weight 5).
- CI health monitoring — primary owner per Coin Integration RACI (4734124042).
- Writing E2E test cases when a feature demands new coverage — CO04/CO05 "Developers are empowered to implement test cases in the E2E test suite when needed."

### QA (feature-team QAs, B2C/B2B) own
- Test plans and test cases in Xray — created during Phase 2 (Test Design, in parallel with dev).
- Spec challenge — reviewing PO specs for edge cases, error handling, testability (CS01 weight 5).
- Feature testing / exploratory testing against non-mocked builds (CO02).
- Regression scope definition — QA selects which test cases go into the global regression scope; QA writes/owns the tickets that mandate automation.
- Manual non-regression before release (per RACI).
- Nightly monitor role — daily triage of nightly results in scope (see Section 4).

### QAA (SDET) own
- Playwright + Speculos framework for LWD (desktop E2E).
- Detox framework for LWM (mobile E2E — iOS and Android).
- Speculos plumbing, seeds, fund topping for the E2E suites.
- Allure report infrastructure including the "Report Flaky" Chrome extension (Flaky Test Reporter) — auto-creates GitHub issues from Allure runs.
- Nightly execution + infrastructure triage (Mac pool for iOS, emulator pool for Android).
- QAA PR review of any regression test that devs automate — "QAA Review" by one QAA member to "ensure the test is suitable for automation and fits CI criteria" (pageId 6741917736).
- Vault E2E tests (vault-e2e-tests, vault-public-api-tests repos) owned by QAA members embedded in B2B.
- Firebase for mobile distribution / run management.
- Xray automation links — QAA ensures test case IDs map correctly to Playwright/Detox specs.

### Summary line
"QA owns the tickets that define which tests must be automated; Development automates them; QAA reviews the resulting PR for correctness and alignment with original test intent." (verbatim, 6902677743).


---

## 4. Flake policy

Sources: "UI E2E Test Stability" (6687817733), "Monitor Role – Responsibilities & Process" (6754467881), "QA - E2E Tests results analysis" (6673694742), "QA - LES Weekly - 01 2026" (6736150663), "QA - LES Weekly - 03 2026" (6858768403).

### Core doctrine
- A red UI E2E run is a signal, not noise. "Forcing UI E2E tests to be green at all costs hides real product issues and reduces trust" (6687817733).
- No blanket retry policy; retries would mask genuine product defects.
- Failures must be classified and owned, not ignored or re-run until green.

### Monitor role — daily triage ("Golden Rule")
> "By 12:00 every working day, there should be no 'silent' failures in nightly runs. Every failure must have an owner, a direction, and visibility in Slack."

Each B2C QA is responsible for monitoring results for their scope on mobile + desktop, daily. Delegation is mandatory when absent.

### Bug vs flake decision matrix

| Failure type | Classification | Action |
|---|---|---|
| Reproducible — product defect | Bug | Create Jira ticket, label qaa, include repro + logs + videos, assign to feature team. Track ETA in the Slack thread. |
| Reproducible — product update (feature flag, A/B test, UI refactor, removed data-test-id) | Process gap / breaking change | Contact the owning squad; if testability was broken (e.g. lost test-ids), request product fix, don't adapt tests around unstable selectors. |
| Not reproducible manually | Flaky | Use the "Report Flaky" Chrome extension button in Allure reports — auto-creates a GitHub issue with project/environment/branch/Allure link; auto-notifies the QAA team. |
| Runner / infra / build failure | Infra issue | Tag @qa-automation in Slack; QAA investigates and creates a PE ticket if needed. |
| Explorer 5xx | External | Post in #explorer-users, link back in the nightly Slack thread. |
| Lack of funds on Speculos seed | Operational | In #live-repo-health, tag @qa-automation for a fund request. |

### Flaky Test Reporter (deployed by QAA, Q1 2026)
- A Chrome extension adding a "Report Flaky" button to every failed/broken Allure test.
- Generates a GitHub issue automatically with project, environment, branch and Allure report link.
- Explicit guidance: "Please use the extension responsibly: do a quick check of the test before submitting a report, so we don't overwhelm the QAA team with duplicates."

### Quarantine / re-run posture (planned)
- B01 in the E2E analysis brainstorm (6673694742): "Support automatic selective re-execution of failed tests, newly failed tests, or tests matching specific tags" — status In progress, owner QAA. No auto-rerun is enabled by default today.
- B02 / B03: tagging test criticality (Critical / Smoke) and failure category (Network Failure / UI element not found / FF mismatch) to power selective rerun.
- Retention of historical failure data (B06) is in progress via Allure/Datadog.

### Flaky reporter rule (draft, LES Weekly 03-2026)
General rule for flaky classification:
- locator failure / not found -> not applicable (likely a real breaking change or bug, not flake).
- 50x from backend -> flake candidate.
- Full documented rule is TBD.

---

## 5. KPIs / coverage targets — actual numbers

Several numeric and categorical KPIs are tracked; some are explicit, others implicit in dashboards.

### Explicit numeric targets
- Unit test coverage: >80% on critical modules (6346375175 Testing strategy).
- Public API test coverage (Vault): 100% threshold achieved in PI 26.1 (LES Weekly 01-2026, via revault PR #3855). Pricing plan scaled to 500k lines in Codacy / coverage tool.
- Vault B2B regression: close to 400 test cases (pageId 6741917736).
- Vault B2C regression: close to 200 test cases.
- iOS Mac pool: 12 machines total, shared across dev CI + SDET nightly + release validation — identified as a structural bottleneck for iOS E2E stability.

### Categorical / dashboard KPIs
- Nightly stability status (6673694742):
  - LWM Nightly Stability: Stable — "Zero flakiness reported; current failures are strictly related to bugs."
  - LWD Nightly Stability: Stable — "Consistency maintained; failures are identified as bugs, not flakiness."
- QA Evaluation Criteria CM02: "Metrics — Number of open bugs / Percentage of bugs raised by the QA team / Severity" (weight 4).
- Automation test coverage dashboard: Jira dashboard 10427.
- QAA Bugs dashboard (per-squad defects): Jira dashboard 10216.
- Vault Canton test strategy dashboard: Jira 16388.
- Signal Quality scoring in QA Incident Management Template (6637191723):
  - Tests flaky / ignored: 20 penalty points
  - Stable but shallow: 10
  - Stable and reliable: 0
- Coverage Level scoring in same template:
  - No coverage: 30
  - Partial (happy path only): 20
  - Full coverage: 10
  - Full + edge cases: 0

### Ledger Wallet BU KPI set (LEDE 5839552734)
- Bug Fix Rate
- Epic Delivery vs Commitment
- OKR Progress
- Velocity trends
- Retro action-item closure rate

No published flake-rate percentage threshold nor a published minimum pass-rate gate was found — it is treated qualitatively ("stable / monitoring / maintenance").


---

## 6. Intern / newcomer onboarding expectations

Two canonical checklists:

- QAA Onboarding Checklist (6647873614) — for SDET joiners, last updated 19 Mar 2026.
- Newcomer Onboarding checklist (6736674834) — for B2B/Vault QA joiners, last updated 01 Feb 2026.

The QAA checklist is explicitly structured as a 2-week top-to-bottom priority list. Work items map cleanly to first-week / first-month / first-quarter milestones.

### First week — Access, equipment, environment

From 6647873614:

1. Get a Ledger device from the QA team locker (ask Laure Duchemin). Onboard it, familiarize with basic flows.
2. File IT access tickets (template links provided):
   - GitHub account added to LedgerHQ org
   - Added to the QAA GitHub group
   - VPN access
   - 1Password vaults (Vault - Team QA Consumer, Vault - Team Vault Quality)
   - Apple Developer account
   - Google Play Console access (for old app versions)
   - Vault cluster access
3. Non-ticket access requests (ask people):
   - Firebase — ask Yaroslava Polishchuk
   - Slack channels — ask Yaroslava Polishchuk
   - Meeting invites (recurring) — ask Yaroslava Polishchuk
4. Read foundational docs:
   - PGP signature setup
   - HSM certificates (Vault)
   - Basic blockchain training (Ledger Lumapps)
5. Clone repos: ledger-live, coin-apps, vault-e2e-tests, vault-public-api-tests.
6. Install Cursor IDE; the ledger-live repo ships .cursor/ AI commands, rules, and skills.
7. Set up LWD and LWM locally (iOS and/or Android).

### First two weeks — E2E test environment

- Desktop E2E (Playwright + Speculos): shortcut /e2e-desktop-onboard Cursor command (interactive wizard). Manual path: Docker Desktop + pull ghcr.io/ledgerhq/speculos:latest.
- Mobile E2E (Detox): shortcut /e2e-mobile-onboard. Manual: Xcode + Ruby >=3.0 + Bundler + applesimutils + simulator named iOS Simulator; Java matching build.gradle, Android SDK, emulator named Android_Emulator; JAVA_HOME, ANDROID_HOME.
- Vault E2E: run vault-e2e-tests and vault-public-api-tests locally.

### First month — Regression testing and paired training

- Access the regression test cases used for release validation, and run the regression on a physical Nano.
- Walkthrough meeting on LWD + LWM with Gabriel Becerra, covering regression questions.
- Automation training sessions (pair with named engineers):
  - LWD automation — Victor Alber
  - LWM automation — Abdurrahman Sastim
  - Vault automation — Oleksandra Boyko
- Teams, responsibilities and key contacts briefing with Yaroslava Polishchuk.
- Weekly sync with Yaroslava to gauge progress and unblock.

### First quarter — integration

- Complete the onboarding feedback form (Google Form, linked in checklist).
- Own at least one test suite stream end-to-end (nightly monitoring rotation is implied once onboarded).
- For Vault/B2B QA joiners: additional AWS/EKS setup (kubectl, k9s, aws-cli), ConfigCat, Datadog, ArgoCD familiarity, plus crypto basics via Ledger Academy and Google Classroom (Blockchain 101 — beginner, medium, advanced-optional).

### B2B-specific onboarding extras (6736674834)
- LinkedIn Learning: Unix Essential Training, Git Essential Training.
- Google Ledger Account + MFA (2FAS or 1Password).
- Docker Desktop is prohibited at Ledger — use Colima (macOS) or Docker Engine (Linux), or the remote vault-device-api at https://vault-device-api.aws.stg.ldg-tech.com/.
- PGP key upload to https://keyserver.ledgerlabs.net/.
- VPN via infra docs (PKB 3478650893).

### Not found
- No formal "first-quarter milestone" doc (e.g. "write 3 E2E specs by day 90"). The framing is by access + pairing + first-regression-run. An explicit success-metric doc for interns is a gap.

---

## 7. Slack / meeting cadence

Sources: 6754467881, 6736150663, 6858768403, 5839552734, 6902677743.

### Daily
- Firefighting rotation: daily FF meeting at 10:15 — mandatory attendance during one's rotation (from LES Weekly 01-2026).
- Nightly-monitor daily triage: each QA reviews their-scope nightly by noon and posts triage in the dedicated Slack thread for the test execution notification.
- Scrum Daily standup: 15 min, attended by devs + QA + PO + DM (LEDE WoW).

### Weekly
- QA LES Weekly (6858768403, 6736150663): Vault/B2B scope. Standing sections: Announcements, Firefighting and nightly, Round-table with QAA team members (Victor Alber, Oleksandra Boyko, Abdurrahman Sastim), Features/Projects tracker.
- Sync with onboarding mentor (weekly, first month): with Yaroslava Polishchuk.
- Roadmap Sync: 30 min, PM + DM + Team Lead (BU-wide).
- Ledger Wallet Pre-Release: 30 min, QA + DM.
- Product Backlog Refinement: 45 min.

### Bi-weekly
- Bug Triage: 30 min, QA + PM + Team Lead, Jira-driven.
- Key Project Review: 45 min, DM + leadership, via Monday.
- Sprint Review (Demo) + Retrospective: 1h at end of each sprint (2-week cadence).
- BU Demo: 1h across all software teams.

### Quarterly
- Regression scope refinement meeting (6902677743): "Quarterly refinement meeting (e.g. next one before end of June)." Optionally bi-weekly/monthly refinement of existing test cases.

### Slack channels referenced
- #live-repo-health — daily nightly report thread; tag @qa-automation for fund requests.
- #explorer-users — to escalate Explorer 503s / blockchain-node issues.
- #cs-quality-assurance — customer-success cross-over (CSM project mentions).

### Cadence summary
- PI = 6 weeks, Sprint = 2 weeks, Kanban or Scrum per team, release cadence weekly / bi-weekly / CD depending on surface (LEDE 5839552734).


---

## 8. Tool stack as QAA practices it

| Tool | Practice |
|---|---|
| Playwright | Desktop E2E driver for LWD (Ledger Wallet Desktop). Co-drives Speculos. Suite is structured by feature (specs/send, specs/onboarding). Also powers Vault vault-e2e-tests and vault-public-api-tests. |
| Detox | Mobile E2E driver for LWM (iOS + Android). Known-fragile in over-subscribed environments (see Mock Test Optimisation 5915639992). |
| Speculos | Ledger hardware wallet emulator (Nano S/X/Stax). Run via Docker image ghcr.io/ledgerhq/speculos:latest. Speculos interaction improvements on iOS are on the QAA roadmap. |
| Jest / React Testing Library | Unit + integration layer, owned by dev teams. MSW for network mocks. |
| Allure | E2E reporting (per-run, per-branch). Flaky Test Reporter Chrome extension integrates with Allure to raise GitHub issues. Restoring the Allure API (B08 brainstorm) is in progress to power a unified Unit+Mock+E2E dashboard. |
| Xray | Test case repository inside Jira. Test plans and cases authored via standardized spreadsheet template; imported into Xray. Regression scope managed as Xray test sets. Test coverage dashboard: Jira dashboard 10427. Linked to Playwright/Detox specs. |
| Firebase | Mobile distribution / test-run management (access granted by Yaroslava). |
| GitHub / GitHub Actions | Source of truth + CI runner orchestration. QAA group GitHub membership required for PR reviews. |
| Cursor IDE | Primary IDE. Ships onboarding commands /e2e-desktop-onboard and /e2e-mobile-onboard. AI rules + skills live in .cursor/ inside ledger-live. Copilot AI Agent for E2E test writing is in beta/dev (QAA-owned). |
| Datadog | Observability for Vault (PRD/PPR on app.datadoghq.eu, STG/SBX on ledgerstg.datadoghq.eu). Used for CI flake metrics ("Use Datadog CI metrics to identify flaky tests and unstable jobs" — Merge Queue Rollout 6777405509). |
| ArgoCD | Continuous delivery platform (Vault). Used to verify component tag deployments. |
| ConfigCat | Feature flag / config management. |
| 1Password | Secrets + vault-team certificates. |
| Playwright MCP / Atlassian MCP | QAA experimentation with MCP-driven Playwright test generation (6482329677). |
| Jira dashboards | Test coverage (10427), QAA Bugs per-squad (10216), Vault/Canton (16388). |

Python is acknowledged as not the main language at QAA — this shapes framework/repo ownership decisions (Future framework requirements 6745161852).

---

## 9. Top 5 Confluence URLs to cite

1. Quality Assurance (space home) — mission, team structure
   https://ledgerhq.atlassian.net/wiki/spaces/QA/overview (pageId 6635357792)
2. Testing strategy (Ledger Live) — pyramid, coverage target, tooling
   https://ledgerhq.atlassian.net/wiki/spaces/CIP/pages/6346375175/Testing+strategy
3. QA - Feature Development and Quality Lifecycle — end-to-end spec->test->automation->PR flow
   https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6898843819/QA+-+Feature+Development+Quality+Lifecycle
4. UI E2E Test Stability — Current State, Root Causes, and Path Forward — flake doctrine
   https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6687817733/UI+E2E+Test+Stability+Current+State+Root+Causes+and+Path+Forward
5. QAA Onboarding Checklist — the canonical onboarding doc
   https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6647873614/QAA+Onboarding+Checklist

---

## 10. Gaps

Things that should exist for a complete onboarding guide but that I could not find evidenced in Confluence:

1. No explicit flake-rate SLA / numeric threshold. Stability is tracked qualitatively (Stable / Monitoring / Maintenance). Merge Queue Rollout page notes "Define an acceptable flake threshold" is still a pre-requisite, not a set number.
2. No formal auto-rerun / quarantine policy. Selective re-execution is in progress (brainstorm item B01). Today policy is "don't re-run to hide a bug"; operational rerun rules are TBD (LES Weekly 03-2026).
3. No formal E2E coverage target by surface (e.g. "90% of critical user journeys must have an E2E"). The 80% target is unit-level only.
4. No documented intern success metrics at day 30 / 60 / 90 (e.g. number of specs delivered, number of bug triages owned). The QAA checklist is binary (access + environment + first pairing).
5. Full "QA team charter" as a single page does not exist. The charter is effectively distributed across the QA space home, QA Evaluation Criteria, QA Feature Development and Quality Lifecycle, and the Onboarding Checklist.
6. Flaky classification rule set (e.g. "locator failure -> not flake, 5xx -> flake") is explicitly marked "tbd" in LES Weekly 03-2026. A published SOP would close this.
7. Archived historical strategy (VAUL 4573757597 "Engineering - QA and Testing - Strategy and policy (archieved)") indicates there was once a centralized policy, but it is explicitly archived and not replaced by an equivalently centralized doc in the QA space.
8. B2C-only formal newcomer checklist — the 6736674834 doc is Vault/B2B-shaped with HSM/AWS detours. B2C wallet QA joiners may need a lighter-weight variant.

---

## 11. Raw links

### Fetched in full
- https://ledgerhq.atlassian.net/wiki/spaces/QA/overview — 6635357792 Quality Assurance (home)
- https://ledgerhq.atlassian.net/wiki/spaces/CIP/pages/6346375175/Testing+strategy — 6346375175
- https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6898843819/QA+-+Feature+Development+Quality+Lifecycle — 6898843819
- https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6687817733/UI+E2E+Test+Stability+Current+State+Root+Causes+and+Path+Forward — 6687817733
- https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6636437536/QA+-+Evaluation+Criteria — 6636437536
- https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6812467327/QA+-+Evaluation+template — 6812467327
- https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6902677743/Kick+off+-+Quality+lifecycle+and+Regression+Scope — 6902677743
- https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6647873614/QAA+Onboarding+Checklist — 6647873614
- https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6736674834/Newcomer+Onboarding+checklist — 6736674834
- https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6741917736/QA+-+Alignment+-+Test+plans+test+cases+and+regression+scope — 6741917736
- https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6754467881/Monitor+Role+Responsibilities+Process — 6754467881
- https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6736150663/QA+-+LES+Weekly+-+01+2026 — 6736150663
- https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6673694742/QA+-+E2E+Tests+results+analysis+-+Definition+of+problems+and+solutions+findings — 6673694742
- https://ledgerhq.atlassian.net/wiki/spaces/LEDE/pages/5839552734/Ways+of+Working+Ledger+Wallet+BU — 5839552734
- https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/4734124042/DEV+QA+-+Role+and+Responsibilities+-+Coin+Integration — 4734124042

### Surfaced in search, not fully fetched (follow-up candidates)
- https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6858768403/QA+-+LES+Weekly+-+03+2026 — flake classification draft, rerun automation discussion
- https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/6777405509/Merge+Queue+Rollout — pre-req flake threshold + Datadog CI metrics
- https://ledgerhq.atlassian.net/wiki/spaces/WXP/pages/4620812293/WXP+QA+Collaboration — WXP-specific RACI
- https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/4573757597/Engineering+-+QA+Testing+-+Strategy+and+policy+archieved — archived strategy/policy
- https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6637191723/QA+-+Incident+Management+Template — signal-quality and coverage scoring bands
- https://ledgerhq.atlassian.net/wiki/spaces/WALLETCO/pages/5915639992/Mock+Test+Optimisation — Detox oversubscription notes
- https://ledgerhq.atlassian.net/wiki/spaces/VAUL/pages/6482329677/QA+Playwright+and+Atlassian+MCP+for+QAA — MCP-assisted Playwright
- https://ledgerhq.atlassian.net/wiki/spaces/QA/pages/6745161852/Future+framework+requirements — language/repo constraints
- https://ledgerhq.atlassian.net/wiki/spaces/CF/pages/4803002524/Coin+Integration+Onboarding+Resources — cross-team onboarding reference
- https://ledgerhq.atlassian.net/jira/dashboards/10427 — Automation test coverage dashboard
- https://ledgerhq.atlassian.net/jira/dashboards/10216 — QAA Bugs per-squad dashboard

---

End of R5 research.
