# R7 Audit — Dedupe / Merge Matrix for Option B (Full Linearization)

**TL;DR / Key finding:** the current guide has a **two-pass structure** — Part 2 gives a shallow "in Ledger Live" tour of every tool/framework (Ch 8 PW, Ch 9 Detox, Ch 10 Speculos, Ch 12 Allure), and Parts 5/6 re-teach those same frameworks from zero and then go deep (Ch 22-23 PW, Ch 34-35 Detox, Ch 25 Speculos, Ch 26 Allure). Overlap severity: **~70%** (Ch 8 ↔ Ch 22-23), **~85%** (Ch 9 ↔ Ch 34-35), **~90%** (Ch 10 ↔ Ch 25), **~60%** (Ch 12 ↔ Ch 26). Option B's core win is collapsing these into **one teach-it-once linearization**: Speculos and Allure merge into Part 2 (shared tooling, no framework deep dives); PW/Detox teaching stays in Part 3/Part 4 but shed their shallow Part 2 counterparts. Secondary finding: Ch 17 (GitHub Actions, shallow) and Ch 38 (Mobile CI, deep, ~580 lines) must merge into a single Mastery CI chapter.

Total size: 17,468 lines across 45 chapters + 6 appendix sections. Proposed new total: ~19,750 lines (+13%, absorbs the new Part 0). ~13,700 lines COPY, ~4,200 lines REWRITE, ~1,850 lines NEW.

## 1. Current TOC (one-liner per chapter)

*(Part 1 / part1.md — 955 lines)* Ch 1 What Is Ledger Live (product, platforms, QA role, HW testing, strategy) · Ch 2 Tech Stack (TS/React/RN/Electron/Redux/Metro/PW/Detox/Jest/Allure/pnpm/Turbo) · Ch 3 E2E Test Architecture (cross-platform diagram, components, data-flow) · Ch 4 Monorepo & Team Ownership (tree, `e2e/`, CODEOWNERS) · Ch 5 Dev Env Setup (prereqs, clone, env vars, Speculos Docker, desktop+mobile builds, first run) · Part 1 Final Assessment.

*(Part 2 / part2.md — 1864 lines)* Ch 6 Git Workflow/Hooks/Changesets · Ch 7 pnpm/Turbo/Build Pipeline · **Ch 8 Playwright in Ledger Live** (config, fixtures, locators, @step, tags, userdata, dev workflow, common issues) — overlaps Ch 22-23 · **Ch 9 Detox in Ledger Live** (config, global `app`, WebSocket bridge, helpers, multi-phase init, structure, build/run/debug from wiki) — overlaps Ch 34-35 · **Ch 10 Speculos Deep Dive** (6 models, REST, touch vs non-touch, coin-apps, env vars, Docker, LL-Bot) — overlaps Ch 25 · Ch 11 Firebase & Feature Flags (envs, flags in E2E, Wallet 4.0, anti-patterns, hygiene) · **Ch 12 Allure Reporting & Xray** (local/CI viewing, @step, TMS links, Xray JSON) — overlaps Ch 26 · Part 2 Final.

*(Part 3 / part3.md — 1030 lines)* Ch 13 Desktop E2E Architecture (tree, class hierarchy PageHolder→Component→AppPage, Application hub, fixture lifecycle, CI sharding) · Ch 14 Writing First Desktop E2E Test (anatomy, Speculos test, data-driven, template, 4-step workflow) · Ch 15 Mobile E2E Architecture (tree, 7-phase init, WebSocket bridge in depth, `app.init()` in `beforeAll`) · Ch 16 Writing First Mobile E2E Test (simple, Speculos, desktop→mobile translation, template, helper ref) · Part 3 Final.

*(Part 4 / part4.md — 1044 lines)* **Ch 17 GitHub Actions** (anatomy, E2E workflows, @Gate, composite actions, logs, wiki troubleshooting) — shallow vs Ch 38 · Ch 18 Test Strategy & Best Practices (pyramid, naming, POM rules, data-driven, flaky mgmt) · Ch 19 Debugging Failures (mindset, errors, PW tools, Detox tools, Speculos debug, LLD wiki tips) · **Ch 20 Contributing Back** (workflow, PR checklist, review, changeset rule) — redundant with Ch 6/18/28/39 · Ch 21 AI Agents & Automation (Claude, Rodin, Cursor) · Part 4 Final.

*(Part 5 / part5.md — 3653 lines)* Ch 22 Playwright from Zero (locators, actions, assertions, auto-wait, config line-by-line, CLI, PWDEBUG) · Ch 23 Playwright Advanced (fixtures, `base.extend<T>()`, `TestFixtures`, `test.use()`, POM, @step decorator, tags, workers, retries) · Ch 24 Electron Desktop Testing (how PW launches it, env, `ElectronApplication` vs `Page`, startup sequence, webviews) · **Ch 25 Speculos Mastery** (recap, fixture in common.ts, speculosUtils.ts, `SpeculosDevice`, REST, env, debug) — merge with Ch 10 · **Ch 26 Allure & Xray** (pipeline, @step chain, artifacts, TMS/bug links, team ownership, reading, custom Xray reporter) — merge with Ch 12 · Ch 27 Codebase Deep Dive (config, fixture, POMs, components, modals, utilities, userdata, enums, modular dialogs) · Ch 28 Daily Workflow Ticket→PR (10 steps) · Ch 29 QAA-1139 Walkthrough (Add Account BTC) · Ch 30 QAA-1141 Walkthrough (Market Filter Starred) · Ch 31 Exercises & Challenges (7 timed + 1 challenge) · Part 5 Final.

*(Part 6 / part6.md — 6364 lines, largest file)* Ch 32 React Native Primer (core primitives, testID, Platform, Metro, deep links, dev menu) · Ch 33 Mobile Toolchain (macOS-only for iOS, proto/mise, Xcode/CocoaPods, Studio/Gradle/adb, ENVFILE, build cmds, checklist) · Ch 34 Detox from Zero (matcher-action-expectation, matchers, actions, expectations, `element()`, `waitFor`, `device`, first test, EarlGrey vs Espresso) · Ch 35 Detox Advanced (detox.config.js, CLI, idle detection, sync troubles, Jest runner, globalSetup, WebSocket bridge, @Step, $TmsLink) · Ch 36 Mobile Codebase Deep Dive (tree, setup.ts, Jest configs, tsconfig, bridge, helpers, models, page, specs, userdata, 36.12 cross-ref to Ch 27) · Ch 37 Running & Debugging Mobile (build, execution, e2e-ci.mjs, debugging cookbook, artifacts, Allure, TmsLink single-run, Speculos on mobile) · Ch 38 Mobile CI & Sharding (reusable workflow, triggers, env table, sharding, matrix, artifact flow, smoke vs full, prod vs staging FB, ASCII sequence) · Ch 39 Daily Mobile Workflow · Ch 40 QAA-702 Walkthrough (Swap History ERC20 Export) · Ch 41 Mobile Exercises · Part 6 Final.

*(Part 7 / part7.md — 1575 lines)* Ch 42 Live Apps in Ledger Wallet (what/why, load arch, Manifest API, Wallet API, catalog, QA implications) · Ch 43 Swap Live App Architecture (repo landscape, frontend swap-live-app, swap service + Completer, swap-configuration, exchange-sdk, ledger-live-assets, providers, MiCA/CASP) · Ch 44 Swap Releases & Firebase Envs (trunk on develop, URLs, FB projects, feature flags, release workflow, monitoring, env cheat-sheet) · Ch 45 QAA-1136 Swap Walkthrough · Part 7 Final.

*(Appendix / appendix.md — 983 lines)* A Complete Command Reference · B Env Vars · C Ledger Live Stack (per-lib blurbs) · D Supported Devices & Coins · E Troubleshooting FAQ · F Glossary.

## 2. Duplicate / Overlap Matrix

| Old chapter | Overlaps with | Severity | Action |
|---|---|---|---|
| Ch 8 PW in Ledger Live (8.1-8.6) | Ch 22 + Ch 23.1-23.6 | **redundant ~70%** | Cut 8.1-8.6. Move Ledger-specific fixture nuance into new Part 3 Ch 3.4 (PW advanced). |
| Ch 8.7-8.8 wiki dev-workflow + common issues | Ch 28, Ch 19 | partial | Merge into Daily Workflow (3.8) and Debugging (6.2). |
| Ch 9 Detox in Ledger Live (9.1-9.10) | Ch 15 + Ch 34 + Ch 35 + Ch 36 | **redundant ~85%** | Cut entirely. 9.1→35.1, 9.2→15.2/34, 9.3→15.3+35.7, 9.4→36.6, 9.5→15.2, 9.6-9.10→37+38. |
| Ch 10 Speculos Deep Dive | Ch 25 Speculos Mastery | **redundant ~90%** | Merge into single Part 2 Ch 2.3 Speculos. LL-Bot (10.8) is unique — keep. |
| Ch 10.8 LL-Bot | Appendix A bot commands | partial | Narrative stays in 2.3, commands stay in Appendix A. |
| Ch 11 Firebase & Feature Flags | Ch 33.7 ENVFILE + Ch 44 (Swap FB) + Ch 38.12 | complementary | Keep as Part 2 canonical. Ch 33.7 thins to pointer. Cross-links only. |
| Ch 12 Allure/Xray (12.1-12.9) | Ch 26 | **redundant ~60%** | Merge into Part 2 Ch 2.5. Custom Xray reporter deep dive (26.7) survives. |
| Ch 13 Desktop Arch | Ch 27 Codebase | complementary (map vs encyclopedia) | Keep both in Part 3 (3.1 + 3.7), enforce no duplication. |
| Ch 14 First Desktop Test | Ch 22-23, Ch 28-29 | partial | Keep as motivational anchor before teach-from-zero. Trim to ~300 lines. |
| Ch 15 Mobile Arch | Ch 36 Codebase | complementary | Keep both in Part 4 (4.1 + 4.7). |
| Ch 15.3 WebSocket Bridge | Ch 9.3 + Ch 35.7 | redundant | Merge into one bridge section inside Part 4. |
| Ch 16 First Mobile Test | Ch 34, Ch 40 | partial | Keep as anchor. Trim to ~300 lines. |
| Ch 17 GitHub Actions | Ch 13.5 + Ch 38 | **shallow vs deep** | Merge into Part 6 Ch 6.3 (CI chapter): anatomy (17.1-17.2) + desktop sharding (13.5) + mobile sharding (38 body). |
| Ch 18 Test Strategy | Ch 20 PR checklist, Ch 39 review etiquette | partial | Keep as canonical. Absorb 20.2. |
| Ch 19 Debugging | Ch 25.8, Ch 37.5 | complementary | Keep as cross-cutting Part 6. Platform-debug stays in each Part. |
| Ch 19.7 LLD wiki debug tips | Ch 37.5 | partial | Merge into Part 3 debugging subsection. |
| Ch 20 Contributing Back | Ch 28, Ch 39, Ch 6 | **redundant ~60%** | Cut. Redistribute: 20.1→Part 6 workflow; 20.2→Ch 18.5; 20.3→Part 6 Mastery; 20.4→Ch 6. |
| Ch 21 AI Agents | — | unique | Keep in Part 6 Mastery (6.4). |
| Ch 27 Codebase Deep Dive | Ch 13.1, 13.2 | partial | Deeper catalog kept; arch map lighter. |
| Ch 28 Daily Workflow | Ch 39 (mobile twin), Ch 29/30/40/45 walkthroughs | complementary | Keep in Part 3. Extract shared "Ticket→PR template" to Part 6. |
| Ch 29, 30 Walkthroughs | — | unique | Keep in Part 3. |
| Ch 31 Exercises | — | unique | Keep. |
| Ch 33 Mobile Toolchain | Ch 5.6 | partial | Canonical mobile setup; Ch 5.6 thins to 20-line pointer. |
| Ch 36 Mobile Codebase | Ch 15, Ch 9.4 | complementary | Keep. 36.12 ("Cross-Reference with Part 5 Ch 27") rewrite as mobile↔desktop file-diff or delete. |
| Ch 37 Running & Debugging Mobile | Ch 19 | complementary | Keep. |
| Ch 38 Mobile CI | Ch 17 | covered above | Becomes body of merged Ch 6.3. |
| Ch 39 Daily Mobile Workflow | Ch 28 | complementary | Keep in Part 4. |
| Ch 40, 41 | — | unique | Keep. |
| Ch 42-45 Swap | Ch 11, Ch 38.12 | complementary | Keep, move to Part 5. Cross-link. |

## 3. Option B Destination Mapping

- `Ch 8 "Playwright in Ledger Live"` | Part 3 | folded | Cut. Unique bits → 3.4 (PW advanced) + 3.8 (daily workflow) + 3.11 (exercises/debugging).
- `Ch 9 "Detox in Ledger Live"` | Part 4 | folded | Cut. Bits → 4.1 + 4.5 + 4.6 + 4.7.
- `Ch 10 "Speculos Deep Dive"` | Part 2 | 2.3 | Merge with Ch 25 → the one Speculos chapter.
- `Ch 13 "Desktop E2E Architecture"` | Part 3 | 3.1 | Becomes first chapter of Part 3.
- `Ch 22 "Playwright from Zero"` | Part 3 | 3.3 | Move + keep.
- `Ch 25 "Speculos Mastery"` | Part 2 | 2.3 | Merge with Ch 10. Fixture internals → Part 3 Ch 3.6.
- `Ch 38 "Mobile CI & Sharding"` | Part 6 | 6.3 | Merge with Ch 17 into Mastery CI chapter.

## 4. Content to Drop Entirely

1. **Ch 8 body (8.1-8.9)** — all re-taught in Ch 22-23.
2. **Ch 9 body (9.1-9.11)** — all re-taught in Ch 34-35; wiki content in 37/38.
3. **Ch 10 "Deep Dive" chapter wrapper** — content moves to merged 2.3, container dies.
4. **Ch 12 chapter wrapper** — body merges with Ch 26.
5. **Ch 20 Contributing Back entire chapter** — redistributed into Ch 6, 18, 19, Part 6.
6. **Ch 25 chapter wrapper** — body merges with Ch 10 into 2.3.
7. **Ch 26 chapter wrapper** — body merges with Ch 12 into 2.5.
8. **Ch 36.12 "Cross-Reference with Part 5 Ch 27"** — loses meaning after merge. Delete or rewrite as file-diff.
9. **Part 3 Final Assessment (old, Ch 13-16)** — dissolved; split into new Part 3 and Part 4 finals.
10. **Part 2 Final Assessment (old)** — rewritten (5 ch instead of 7).

## 5. Content Gaps (expand during merge)

1. Ch 11.7 Feature Flag Hygiene — add worked stale-flag migration.
2. Ch 13.4 Fixture Lifecycle — 36 lines, needs sequence diagram.
3. Ch 13.5 Sharding in CI — stub, expand during Ch 17+Ch 38 merge.
4. Ch 15.3 WebSocket Bridge — 17 lines, merge with 9.3 + 35.7 into authoritative section.
5. Ch 17.4 @Gate Job — 8 lines, expand with workflow snippet.
6. Ch 17.5 Composite Actions — 25 lines, produce combined desktop+mobile table.
7. Ch 19.6 Common Failure Patterns — link rows to concrete recipes.
8. Ch 20.3 Code Review Expectations — 12 lines, expand in Part 6 Mastery.
9. Ch 21.3 Rodin Rule — 10 lines, add worked QA-ticket example or link out.
10. Ch 24.6 Webviews — 36 lines, cross-link to Part 5 (Swap = webview-hosted Live App).
11. Ch 33.7 ENVFILE — 20 lines, under-explains `.env.mock.prerelease` vs `.env.mock`.
12. Ch 35.4 Sync Troubles — add more concrete patterns.
13. Ch 36.11 How a Test Runs — add diagram mirroring Ch 3.
14. Ch 38.13 Sharding ASCII Sequence — mirror on desktop side.
15. Ch 42.4/42.5 Manifest API + Wallet API — ~60 lines each, add concrete manifest excerpts.
16. Ch 43.9 MiCA/CASP boundary — 9 lines, expand or mark out-of-scope.
17. Ch 44.6 Monitoring — 20 lines, add Datadog/Sentry examples.
18. **Part 0 (new)** — entire section is a gap, filled by R0-R6 output: Welcome to Ledger, Security Model, Who Is Who, First Week, Newcomer Glossary.
19. **Unified debug catalog** — currently split across Ch 19/25.8/37.5.

## 6. Renumbering Table (Old → New)

| Old | Old title | New | New title |
|---|---|---|---|
| — | — | 0.1-0.5 | Part 0 Welcome (NEW) |
| Ch 1 | What Is Ledger Live? | 1.1 | What Is Ledger Live? |
| Ch 2 | Tech Stack | 1.2 | Tech Stack |
| Ch 3 | E2E Test Architecture | 1.3 | E2E Test Architecture |
| Ch 4 | Monorepo & Teams | 1.4 | Monorepo & Teams |
| Ch 5 | Dev Env Setup | 1.5 | Dev Env Setup |
| Ch 6 | Git Workflow | 2.1 | Git, Hooks & Changesets |
| Ch 7 | pnpm & Turbo | 2.2 | pnpm, Turbo & Build Pipeline |
| Ch 10 + 25 | Speculos ×2 | 2.3 | Speculos — Device Emulation |
| Ch 11 | Firebase & Feature Flags | 2.4 | Firebase & Feature Flags |
| Ch 12 + 26 | Allure/Xray ×2 | 2.5 | Allure Reporting & Xray |
| Ch 13 | Desktop Arch | 3.1 | Desktop E2E — Architecture |
| Ch 14 | First Desktop Test | 3.2 | Your First Desktop Test |
| Ch 22 | PW from Zero | 3.3 | Playwright from Zero |
| Ch 23 | PW Advanced | 3.4 | Playwright Advanced |
| Ch 24 | Electron | 3.5 | Electron — Desktop App Testing |
| Ch 25 residual | Speculos fixture | 3.6 | Desktop Speculos Integration |
| Ch 27 | Codebase Deep Dive | 3.7 | Desktop Codebase — Every File |
| Ch 28 | Daily Workflow | 3.8 | Daily Desktop Workflow |
| Ch 29 | QAA-1139 | 3.9 | Walkthrough QAA-1139 |
| Ch 30 | QAA-1141 | 3.10 | Walkthrough QAA-1141 |
| Ch 31 | Exercises | 3.11 | Desktop Exercises |
| Ch 15 | Mobile Arch | 4.1 | Mobile E2E — Architecture |
| Ch 16 | First Mobile Test | 4.2 | Your First Mobile Test |
| Ch 32 | RN Primer | 4.3 | React Native Primer |
| Ch 33 | Mobile Toolchain | 4.4 | Mobile Toolchain & Env Setup |
| Ch 34 | Detox from Zero | 4.5 | Detox from Zero |
| Ch 35 | Detox Advanced | 4.6 | Detox Advanced |
| Ch 36 | Mobile Codebase | 4.7 | Mobile Codebase |
| Ch 37 | Running & Debugging Mobile | 4.8 | Running & Debugging Mobile |
| Ch 39 | Daily Mobile Workflow | 4.9 | Daily Mobile Workflow |
| Ch 40 | QAA-702 | 4.10 | Walkthrough QAA-702 |
| Ch 41 | Mobile Exercises | 4.11 | Mobile Exercises |
| Ch 42 | Live Apps | 5.1 | Live Apps in Ledger Wallet |
| Ch 43 | Swap Arch | 5.2 | Swap Live App — Architecture |
| Ch 44 | Swap Releases/FB | 5.3 | Swap — Releases & Firebase |
| Ch 45 | QAA-1136 | 5.4 | Walkthrough QAA-1136 |
| Ch 18 | Test Strategy | 6.1 | Test Strategy & Best Practices |
| Ch 19 | Debugging | 6.2 | Debugging Failures — Cross-Platform |
| Ch 17 + 38 | CI ×2 | 6.3 | CI/CD & Test Sharding |
| Ch 21 | AI Agents | 6.4 | AI Agents & Automation Rules |
| Ch 8 | PW in LL | — | **Cut** |
| Ch 9 | Detox in LL | — | **Cut** |
| Ch 20 | Contributing Back | — | **Cut** |
| Appx A-F | — | Appx A-F | Keep |

Chapter-count delta: Old 45 → New ~40 (plus ~5 new Part-0).

## 7. Assessment Strategy

| Old Assessment | Status | New Location |
|---|---|---|
| Part 1 Final | Lightly revised | Part 1 Final — drop Speculos/flag Qs (now in Part 2). |
| Part 2 Final | Heavily rewritten | Part 2 Final — new scope (Git/pnpm/Speculos/FB/Allure). Drop PW/Detox Qs. |
| Part 3 Final (old, Ch 13-16) | Dissolved | Replaced by NEW Part 3 + Part 4 finals. |
| Part 4 Final (old, CI+mastery 17-21) | Dissolved | Replaced by NEW Part 6 final. |
| Part 5 Final (old, desktop) | Merged | Body of NEW Part 3 Final + 3-5 Qs from Ch 13-14. |
| Part 6 Final (old, mobile) | Merged | Body of NEW Part 4 Final + 3-5 Qs from Ch 15-16. |
| Part 7 Final (old, Swap) | Survives | NEW Part 5 Final. |

New matrix: Part 0 Final (NEW orientation quiz), Part 1 Final (revised), Part 2 Final (rewritten), Part 3 Final (merged), Part 4 Final (merged), Part 5 Final (intact), Part 6 Final (merged from old Part 4).

## 8. Estimated Effort

| New Part | COPY | REWRITE | NEW | Total | Notes |
|---|---|---|---|---|---|
| Part 0 Welcome | 0 | 0 | ~1,200 | ~1,200 | 4-5 ch × ~250 lines. Blocked on R0-R6. |
| Part 1 Foundations | ~750 | ~200 | ~50 | ~1,000 | Trim Ch 2 stack, Ch 5.6 mobile. |
| Part 2 Shared Tooling | ~900 | **~1,400** | ~50 | ~2,350 | Speculos merge + Allure merge = biggest rewrite item. |
| Part 3 Desktop | ~3,100 | ~700 | ~150 | ~3,950 | Most of the desktop gold-content is COPY. |
| Part 4 Mobile | ~5,900 | ~800 | ~150 | ~6,850 | Largest Part; most survives in place. |
| Part 5 Swap | ~1,450 | ~100 | ~50 | ~1,600 | Lightest-touch. |
| Part 6 Mastery | ~650 | **~900** | ~150 | ~1,700 | CI merge is the second-biggest rewrite. |
| Appendix | ~950 | ~100 | ~50 | ~1,100 | Light update. |
| **Totals** | ~13,700 | ~4,200 | ~1,850 | ~19,750 | +13% vs today. |

**Bottlenecks (writer-days):** Part 2 Speculos merge (2d); Part 6 CI merge (2d); Part 2 Allure merge (1.5d); Part 3+4 Finals rewrite (1d); Part 0 blocked on R0-R6.

**Parallel streams:** A Part 1+Appendix · B Part 2 merges · C Part 3 + Final · D Part 4 + Final · E Part 5 cross-links · F Part 6 + CI merge · G Part 0 (blocked). A-B-C-D-F parallelizable *once Section 6 renumbering is frozen* — that freeze should be the first action.

## Risk flags

1. `_sidebar.md` + `index.html` need coordinated anchor updates; external bookmarks (Slack/Jira) will break — consider redirect layer.
2. Grep pass for cross-refs before merge: `grep -nE "Part [0-9]+|Chapter [0-9]+|Ch [0-9]+" part*.md` — Ch 19.3 and Ch 36.12 are known forward-pointers that will dangle.
3. Quiz `data-*` attributes are coupled to filenames.
4. `images/` folder embeds may be chapter-numbered.
5. Editorial risk of merges (10+25, 12+26): keep deep sections as explicit `### N.N Deep Dive: ...` subsections rather than prose-blending.
