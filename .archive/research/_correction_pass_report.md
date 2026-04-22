# Correction-Pass Report (W0)

Date: 2026-04-22
Scope: Verify and apply 5 factual corrections (C1–C5) from R3/R4/R6 to the QA onboarding guide.

## Per-correction verdicts

### C1 — ENVFILE naming (`.env.mock` / `.env.mock.prerelease`)

**Verdict: FALSE (claim rejected). Guide is correct as-is — no change applied.**

Steelman preserved: the hypothesis that E2E-mobile env files could differ from release env files is **verified true** in the monorepo.

Evidence from `/Users/jerome.portier/src/tries/2026-04-08-LedgerHQ-ledger-live/apps/ledger-live-mobile/`:

- `.env.mock` and `.env.mock.prerelease` both exist on disk alongside release-oriented files (`.env.ios.release`, `.env.android.prerelease`, etc.).
- `detox.config.js` passes `ENVFILE=.env.mock` and `ENVFILE=.env.mock.prerelease` to every Detox build (iOS Debug/Staging/Release and Android assembleDetox / assembleDetoxPreRelease).
- `package.json` includes `"android:mock": "cd android && ENVFILE=.env.mock ./gradlew assembleStagingRelease"`.
- `rspack.config.mjs` branches on `process.env.ENVFILE` containing `"mock"` to switch Detox mode.

Conclusion: R3's release-environment file list (`.env`, `.env.testing`, `.env.staging`, `.env.production`) is a **different namespace** from the E2E-mobile Detox file set. The guide's references to `.env.mock` / `.env.mock.prerelease` in part6.md Ch 33.7 / 37 / 38 are factually correct.

### C2 — Jira project keys

**Verdict: PARTIALLY CORRECT. Applied 1 targeted fix; preserved remaining usages as shorthand / labels (Rodin steelman).**

Evidence from Atlassian MCP `getVisibleJiraProjects`:

- Projects that exist: `LIVE` (Ledger Wallet), `B2CQA`, `QAA`, `DT`, `LBD`.
- Projects that do **not** exist as keys: `LLM`, `LLD`, `PTX`, `BACKLOG`.

Guide scan:

- Every `LLM`/`LLD` reference in part1.md / part2.md / part4.md / part5.md is used as **product shorthand** ("Ledger Live Mobile", "Ledger Live Desktop") or as the **value of Xray's `Automated In` field** — both of which R4 confirms are correct real-world usage (labels on LIVE tickets, NOT project keys). No change required.
- Every `PTX` reference in part7.md describes the **PTX team / Buy-Sell Live App**, not a Jira key. Consistent with R4 §4.34: "PTX -- referenced in onboarding docs ... not a Jira project key ... filed inside LIVE under Recover/Apex epics". No change required.
- **One incorrect claim found and fixed**: part6.md line 5163 listed `` `LIVE` / `BACKLOG-*` `` as the Project column for Bugs. `BACKLOG` is not a Jira project (confirmed via MCP search → 0 results). Corrected to `` `LIVE` (tagged with `LLM` / `LLD` labels) `` to match R4's documented reality.

### C3 — Release mechanism: semantic-release vs changesets

**Verdict: VERIFIED CORRECT. No change needed — the guide already uses changesets consistently.**

Evidence:

- `package.json` root has no `semantic-release` dependency. It declares `"@changesets/cli": "2.27.7"` and scripts `"bump": "changeset version"`, `"release": "changeset publish"`, `"changelog": "changeset add"`.
- `.changeset/` directory contains hundreds of pending changeset files.
- `.github/workflows/release-prerelease.yml` line 88 runs `pnpm changeset version`; `release-final.yml` line 86 runs `pnpm changeset publish`.

Guide scan: `grep -n "semantic-release"` across all guide files returned **zero hits**. part1.md tooling table (line 237), part2.md Chapter 6.5 ("Changesets"), and appendix.md glossary all describe changesets correctly. "Semantic Versioning (semver)" references are accurate — that is the versioning convention, not the tooling.

### C4 — Prod and prerelease builds are same artifact (since 2024)

**Verdict: AMBIGUOUS — flagged for human review, no change applied.**

Evidence from `.github/workflows/`:

- `release-prerelease.yml` dispatches `pre-desktop.yml` / `pre-mobile.yml` workflows in the separate `ledger-live-build` repo (lines 133-162).
- `release-final.yml` dispatches `release-desktop.yml` / `release-mobile.yml` in the same `ledger-live-build` repo (lines 160-190).

These are **four distinct workflow files** in a repo this agent cannot see. The source-side ref can be identical (both can target the `release` branch), but whether the downstream build produces the same binary artifact or merely the same source snapshot is not verifiable without reading `ledger-live-build`.

R3's claim that "prod and prerelease are the same artifact since 2024" is plausible but unverifiable from this repo alone. The guide currently does not make a strong claim in either direction — part6.md Ch 37/38 discuss `.env.mock.prerelease` as a **test-only** variant of the E2E mock app, which is orthogonal to the release artifact question. No misleading text found; no edit made.

**Follow-up: human reviewer should inspect `LedgerHQ/ledger-live-build/.github/workflows/{pre,release}-{desktop,mobile}.yml`** to confirm same-artifact claim and decide whether a clarifying note is warranted.

### C5 — Coin Framework Alpaca migration state

**Verdict: PARTIALLY CORRECT — guide does NOT describe the framework as monolithic; no misleading text found. No change applied.**

Evidence:

- Guide search for "monolith", "monolithic" → zero hits.
- part4.md line 700, 715: describes `coin-families-contract.md` rule that forbids `if (family === "evm")` in generic code — that is explicitly an **anti-monolith** contract.
- part1.md line 483, 548: lists `libs/coin-modules/` as "Coin-specific logic" — this accurately matches the monorepo's `libs/coin-modules/coin-{bitcoin,evm,…}` layout (29 family directories observed).
- part6.md line 2406, 2429: uses `registerAllCoins` and describes "coin-module boot sequence" — modular, not monolithic.

Steelman preserved: the monorepo is genuinely mid-migration (`libs/ledger-live-common/src/bridge/generic-alpaca/` exists; R6 confirms 4 of 29 families are "alpacaized" with only XRP fully extracted). But the guide makes no outdated claim that would mislead a QA reader; it only describes the layout the reader will see today, which is correct. Adding an "as of 2026-04" Alpaca-migration note would be inside-out engineering trivia for a QA onboarding document — per the Hard Guardrail, features are implemented only when required and testable for user value.

**Follow-up: if a future chapter genuinely discusses `libs/ledger-live-common/src/bridge/generic-alpaca/` internals, an Alpaca note should be added there — flagged as a future concern, not a current gap.**

## Files edited

| File | Lines changed | Change |
|------|---------------|--------|
| `/Users/jerome.portier/qa-onboarding-guide/part6.md` | 1 line (5163) | Replaced fabricated `BACKLOG-*` Jira key with correct "`LIVE` (tagged with `LLM` / `LLD` labels)". |

**Total: 1 file, 1 line changed.**

## Claims NOT changed and why (Rodin steelman preserved)

1. **`.env.mock` / `.env.mock.prerelease` mentions across part6.md** — verified real and in active use by Detox. R3's claim pertained to release envs, a different file set.
2. **`LLM` / `LLD` shorthand throughout the guide** — confirmed by R4 as correct real-world usage (labels on LIVE, values of Xray "Automated In" field, common product shorthand). Changing to "LIVE ticket with LLM label" everywhere would obscure the working convention the reader will encounter daily.
3. **`PTX` as team / Live App name in part7.md** — not claimed to be a Jira key; correctly described as a team/product name.
4. **Changesets / semver references** — guide already accurate.
5. **Coin Framework descriptions in part1.md / part4.md / part6.md** — already describes modular architecture; no monolithic claim to refute.

## Open follow-ups / gaps

1. **C4 (prod/prerelease artifact)**: needs a human with access to `LedgerHQ/ledger-live-build` workflows to settle whether R3's "same artifact since 2024" claim is true. If true, a single-sentence callout near part2.md §5 (release cycle) or part6.md §37.12 (`production_firebase` flag) may be warranted.
2. **C5 (Alpaca migration)**: not a current gap but a future consideration if the guide ever goes below the bridge API surface.
3. **Broader review opportunity**: The R4 research also flags that `LIVE` project does not populate Jira "components" at all — the de-facto component dimension is the LLM/LLD label. If the guide's Jira section ever recommends using components, it should be corrected; not investigated in this pass since no such recommendation was found on a grep.

## Aggregate impact

Applied 1/5 corrections (C2 partial). C1 was factually wrong about the guide (E2E files are distinct from release files and DO exist). C3 and C5 require no action (guide already accurate). C4 is ambiguous and flagged for human review.
