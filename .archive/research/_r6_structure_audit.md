# R6 — Structural Audit for Part 5 CLI Automation

Date: 2026-04-27
Author: research agent R6
Purpose: Hand the CLI-Part writers (a) the exact structural template Part 4 imposes, (b) the verbatim quiz HTML, (c) every cross-reference and renumbering operation needed once Part 5 = CLI is inserted between current Part 4 (Mobile) and current Part 5 (Swap → becomes Part 6) and current Part 6 (Mastery → becomes Part 7).

---

## 1. Part 4 Mobile chapter shape (the template)

Part 4 has **11 H2 chapters + 1 Final Assessment** = 12 H2 sections total. Each chapter has between 6 and 18 H3 subsections, numbered `### 4.X.Y`.

### 1.1 H2 chapter list (verbatim title casing)

| # | Title | H3 count | Last H3 (always Quiz) |
|---|-------|---------:|-----------------------|
| 4.1 | `## Mobile E2E -- Architecture Deep Dive` | 6 | `### 4.1.6 Quiz` |
| 4.2 | `## Writing Your First Mobile E2E Test` | 6 | `### 4.2.6 Quiz` |
| 4.3 | `## React Native Primer for TypeScript Engineers` | 8 | `### 4.3.8 Quiz` |
| 4.4 | `## Mobile Toolchain & Environment Setup` | 10 | `### 4.4.10 Quiz` |
| 4.5 | `## Detox from Zero: Core Concepts` | 11 | `### 4.5.11 Quiz` |
| 4.6 | `## Detox Advanced: Config, Synchronization & Bridge` | 10 | `### 4.6.10 Quiz` |
| 4.7 | `## Mobile Codebase Deep Dive: Every File Explained` | 16 | `### 4.7.16 Cross-Reference with Part 3 Chapter 3.6` (no Quiz heading; quiz embedded — see notes) |
| 4.8 | `## Running & Debugging Mobile E2E Tests` | 11 | `### 4.8.11 Chapter 4.8 Quiz` |
| 4.9 | `## Your Daily Mobile Workflow: From Ticket to PR` | 12 | `### 4.9.12 Quiz` |
| 4.10 | `## Real Ticket Walkthrough: QAA-702 — Swap History ERC20 Export` | 18 | `### 4.10.18 Quiz` |
| 4.11 | `## Mobile Exercises & Challenges` | 8 | (no Quiz — exercises are the assessment) |
| – | `## Part 4 Final Assessment` | – | 10-question quiz |

### 1.2 Numbering style

- H2: bare title, no number prefix in heading text. Title case with double-dash where Part 3 used em-dash: `## Mobile E2E -- Architecture Deep Dive`. Note Part 3 also uses `--` (e.g. `## Desktop E2E -- Architecture Deep Dive`).
- H3: `### 4.X.Y Title` — three-level dotted number, then space, then title. Number always present, even on Quiz: `### 4.1.6 Quiz`.

### 1.3 Final Assessment shape

```
## Part 4 Final Assessment

<intro paragraph 3-5 sentences recapping what the part covered>

<a id="part-4-final-assessment"></a>

<div class="quiz-container" data-pass-threshold="80">
<h3>Part 4 Final Assessment</h3>
<p class="quiz-subtitle">10 questions · 80% to pass · Covers Chapters 4.1-4.11</p>
…10 quiz-question divs…
<div class="quiz-score"></div>
</div>

<div class="chapter-outro">
<strong>Part 4 complete.</strong> …recap…<strong>Next: Part 5</strong> …
</div>
```

Two distinguishing features the writers must preserve:

1. The `<a id="part-X-final-assessment"></a>` anchor sits **above** the `quiz-container` div, between the intro paragraph and the quiz markup. This is what the sidebar links to via `part4#part-4-final-assessment`.
2. The closing `<div class="chapter-outro">` after the final quiz contains a "Next: Part X" sentence — it points forward. For Part 5 CLI it should point to Part 6 Swap (the renumbered old Part 5).

Final assessment quiz length:
- Part 3: 10 questions (verify in part3 file if you mirror exactly)
- Part 4: 10 questions, 80% pass
- Part 5 (current Swap): 12 questions, 80% pass

Recommended for new Part 5 CLI: **10 questions, 80% pass** (mirror Part 4 exactly).

---

## 2. Quiz HTML format — verbatim

### 2.1 Per-chapter quiz (Part 4 Chapter 4.1, lines 154-219, byte-for-byte)

```markdown
### 4.1.6 Quiz

<!-- ── Chapter 4.1 Quiz ── -->

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz</h3>
<p class="quiz-subtitle">5 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> How many spec files does the mobile E2E suite have?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) 26 (same as desktop)</button>
<button class="quiz-choice" data-value="B">B) 73</button>
<button class="quiz-choice" data-value="C">C) 184</button>
<button class="quiz-choice" data-value="D">D) 250</button>
</div>
<p class="quiz-explanation">The mobile suite has 184 spec files covering send (68), swap (23), addAccount (17), delete (13), verify address (12), delegate (10), earn (9), settings (6), and more.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q2.</strong> Why is <code>app.init()</code> called in <code>beforeAll</code> instead of <code>beforeEach</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Because Detox doesn't support <code>beforeEach</code></button>
<button class="quiz-choice" data-value="B">B) Because the tests don't need any setup</button>
<button class="quiz-choice" data-value="C">C) Because Jest requires it for parallel execution</button>
<button class="quiz-choice" data-value="D">D) Because launching a native app, connecting the bridge, and populating data is expensive (10-30s) — doing it per test would be prohibitively slow</button>
</div>
<p class="quiz-explanation">Unlike desktop (where Playwright fixtures quickly create fresh state), mobile app launch and bridge connection take 10-30 seconds. Running this per test would make the suite impractically slow.</p>
</div>

<!-- …Q3 Q4 Q5 follow same shape… -->

<div class="quiz-score"></div>
</div>

---
```

Key invariants writers must preserve:

- Outer wrapper: `<div class="quiz-container" data-pass-threshold="80">`. Threshold defaults to 80 for chapter quizzes; some chapters drop to 75 (Chapter 4.2 sets `data-pass-threshold` implicitly via subtitle "75% to pass" but kept the data attribute at 80 — writers should keep `80` unless there's a strong reason otherwise).
- Subtitle line: `<p class="quiz-subtitle">N questions · 80% to pass</p>`. Note the middle-dot `·` (U+00B7), not a regular dot.
- Progress bar div is constant: `<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>`.
- Each question: `<div class="quiz-question" data-correct="X">` where X is the letter A/B/C/D. The `data-correct` attribute is what the JS reads; without it the quiz silently passes everything.
- Question prompt: `<p><strong>QN.</strong> question text</p>` — Q-number always bold, period after.
- Choices: 4 buttons, `data-value="A".."D"`, label always starts `A) `, `B) `, etc. Inline `<code>` is fine.
- Explanation: `<p class="quiz-explanation">…</p>` — shown after the user picks an answer, regardless of right/wrong.
- Score footer: `<div class="quiz-score"></div>` — empty; the JS fills it.
- Comment marker before the quiz block: `<!-- ── Chapter X.Y Quiz ── -->` (em-dashes around "Chapter X.Y Quiz"). Convention but not load-bearing.
- Closing horizontal rule `---` after the quiz separates the chapter from the next H2.

### 2.2 Final Assessment quiz (Part 4, lines 6267-6389)

Identical shape, with three differences:

1. H2 heading is `## Part X Final Assessment` (no chapter number; standalone).
2. Anchor element above the quiz: `<a id="part-X-final-assessment"></a>`. Required for sidebar deep-link.
3. Quiz `<h3>` is `Part X Final Assessment` (not just `Quiz`), and the subtitle includes a coverage range: `<p class="quiz-subtitle">10 questions · 80% to pass · Covers Chapters X.1-X.N</p>`.
4. After the quiz close `</div>`, an outro div in the Part 4 style:

```html
<div class="chapter-outro">
<strong>Part X complete.</strong> …recap of what the reader now owns…<strong>Next: Part Y</strong> …handoff to the next part…
</div>
```

---

## 3. chapter-intro / chapter-outro divs

There are **22 occurrences** in Part 4 of `chapter-intro` or `chapter-outro` divs.

### 3.1 chapter-intro

- One per H2 chapter, placed immediately after the H2 heading and a blank line.
- Single paragraph, 2-5 sentences, 60-200 words.
- HTML allowed inside (`<code>`, `<strong>`, `<em>`).
- No nested headings.

Example (Chapter 4.1):

```html
<div class="chapter-intro">
The mobile E2E suite tests Ledger Live Mobile (React Native) using Detox + Jest. While the high-level patterns mirror the desktop suite — Page Object Model, Speculos emulation, feature flag overrides — the implementation differs significantly. This chapter maps the directory structure, the initialization flow, the WebSocket bridge, and the key architectural differences from desktop.
</div>
```

### 3.2 chapter-outro

- One per H2 chapter, **placed before the Quiz H3** (or before the Final Assessment quiz) — never after the quiz at chapter level. (Part-level outro after the Final Assessment quiz is the exception.)
- Single paragraph, often starts `<strong>Key takeaway:</strong>` for chapters or `<strong>Part X complete.</strong>` for the Final Assessment.
- 80-300 words.
- Functions as the bridge to either the Quiz or the next chapter / next part.

Example chapter-outro (4.1, line 150):

```html
<div class="chapter-outro">
<strong>Key takeaway:</strong> Mobile E2E testing has more constraints than desktop: slower builds, limited parallelism, and the WebSocket bridge as an indirection layer. But the patterns are similar — Page Object Model, Speculos for device emulation, feature flag overrides. Understanding the constraints helps you design better tests.
</div>
```

### 3.3 Other inline div patterns Part 4 uses (not strictly needed but stylistic)

- `<div class="resource-box">` with `<h4>Resources</h4>` and a `<ul>` of external links — Chapter 4.1 line 140 has one. Use sparingly, where official docs deserve a callout (Bun docs, DMK docs).

---

## 4. Anchor format (docsify slugger)

Docsify generates anchors from H2 titles by lowercasing, then:

1. Removing apostrophes entirely (`'` → empty).
2. Replacing spaces with hyphens.
3. Replacing `&` with `amp` (or rather: `&` becomes `&amp;` in HTML, which the slugger then renders as `amp` between hyphens).
4. Stripping colons, em-dashes, parens, slashes (becomes nothing or hyphen — varies).
5. Collapsing `--` (two dashes) into `-`.

Empirical evidence from `_sidebar.md` and observed Part 4 titles:

| Title | Anchor |
|-------|--------|
| `## Mobile E2E -- Architecture Deep Dive` | `mobile-e2e-architecture-deep-dive` |
| `## Writing Your First Mobile E2E Test` | `writing-your-first-mobile-e2e-test` |
| `## React Native Primer for TypeScript Engineers` | `react-native-primer-for-typescript-engineers` |
| `## Mobile Toolchain & Environment Setup` | `mobile-toolchain-amp-environment-setup` |
| `## Detox from Zero: Core Concepts` | `detox-from-zero-core-concepts` |
| `## Detox Advanced: Config, Synchronization & Bridge` | `detox-advanced-config-synchronization-amp-bridge` |
| `## Mobile Codebase Deep Dive: Every File Explained` | `mobile-codebase-deep-dive-every-file-explained` |
| `## Running & Debugging Mobile E2E Tests` | `running-amp-debugging-mobile-e2e-tests` |
| `## Your Daily Mobile Workflow: From Ticket to PR` | `your-daily-mobile-workflow-from-ticket-to-pr` |
| `## Real Ticket Walkthrough: QAA-702 — Swap History ERC20 Export` | `real-ticket-walkthrough-qaa-702-swap-history-erc20-export` |
| `## Mobile Exercises & Challenges` | `mobile-exercises-amp-challenges` |
| `## Part 4 Final Assessment` | `part-4-final-assessment` |

Rules confirmed:
- `&` → `amp` between hyphens. Wrapped in dashes: `-amp-`.
- `:` → swallowed; replaced by hyphen if at a word boundary, else dropped.
- `--` → `-`.
- `'` → silently removed (Part 0 historical fix: the writers had to rename a `0.3 Teams' Ownership` to `0.3 Teams and Ownership` because the apostrophe broke the anchor link from the sidebar; the dropped apostrophe collapsed the slug awkwardly).
- Em-dash `—` → hyphen.
- Numbers and hyphens preserved.

### 4.1 Anchor predictions for proposed CLI Part 5 titles

| Proposed title | Predicted anchor | Risk |
|----------------|------------------|------|
| `## CLI Automation -- Architecture Deep Dive` | `cli-automation-architecture-deep-dive` | safe |
| `## Your First wallet-cli Command` | `your-first-wallet-cli-command` | safe |
| `## Bun & Bunli Primer` | `bun-amp-bunli-primer` | safe (the `amp` is consistent with other titles) |
| `## Device Management Kit (DMK) Primer` | `device-management-kit-dmk-primer` | parens dropped — verify; safer to drop them: `## Device Management Kit Primer` |
| `## wallet-cli Codebase Deep Dive` | `wallet-cli-codebase-deep-dive` | safe |
| `## Account Descriptors V1` | `account-descriptors-v1` | safe |
| `## Running & Debugging wallet-cli` | `running-amp-debugging-wallet-cli` | safe |
| `## Testing wallet-cli (mock DMK + http intercept + cli-runner)` | `testing-wallet-cli-mock-dmk-http-intercept-cli-runner` | parens + plus signs are dicey; recommend `## Testing wallet-cli with Mock DMK and HTTP Intercept` |
| `## Daily CLI Workflow` | `daily-cli-workflow` | safe (could mirror Part 4: `## Your Daily CLI Workflow: From Ticket to PR` → `your-daily-cli-workflow-from-ticket-to-pr`) |
| `## Real Ticket Walkthrough: QAA-615 — wallet-cli revoke` | `real-ticket-walkthrough-qaa-615-wallet-cli-revoke` | safe (mirrors Part 4 4.10) |
| `## CLI Exercises & Challenges` | `cli-exercises-amp-challenges` | safe |
| `## Part 5 Final Assessment` | `part-5-final-assessment` | safe |

**Recommended sanitised titles** (drop parens, drop colons mid-title where possible):

```
## CLI Automation -- Architecture Deep Dive
## Your First wallet-cli Command
## Bun & Bunli Primer
## Device Management Kit Primer
## wallet-cli Codebase Deep Dive
## Account Descriptors V1
## Running & Debugging wallet-cli
## Testing wallet-cli with Mock DMK and HTTP Intercept
## Your Daily CLI Workflow: From Ticket to PR
## Real Ticket Walkthrough: QAA-615 — wallet-cli revoke
## CLI Exercises & Challenges
## Part 5 Final Assessment
```

---

## 5. Cross-references in Part 4 that point to Part 5 / Part 6

Every match for `Part 5`, `Part 6`, `part5`, `part6` in `/Users/jerome.portier/qa-onboarding-guide/part4.md`. Each must be reviewed once the renumber lands.

| Line | Current text | Action |
|-----:|--------------|--------|
| 860 | `… come back here.` (mentions "rest of Part 6" which is actually a typo — there is no Part 6 reference here other than this prose) | **Already wrong even pre-renumber.** Was probably meant to say "rest of Part 4". Fix to "rest of Part 4" during the renumber pass. |
| 4091 | `…see Part 6 Chapter 6.3.` | Renumber → `Part 7 Chapter 7.3` (Mastery → Part 7). |
| 5084 | `… cardinal sin of the onboarding guide's Part 5 and still a cardinal sin here.` | Context: this paragraph says "the onboarding guide's Part 5" referring to the Xray-link discipline taught earlier. After renumber, old Part 5 (Swap) becomes Part 6. But this sentence is actually claiming the rule is taught "in Part 5" — verify if it's referring to Part 2 ch 2.5 (Allure & Xray) instead, in which case it's a long-standing typo that should become `Part 2`. **Likely typo — investigate**. |
| 6392 | `<strong>Next: Part 5</strong> steps out of the apps and into the shared runtime — the <strong>Ledger Wallet Components</strong> library and the <strong>Swap Live App</strong>` | This is the part-outro. Two changes: (a) renumber `Part 5` → `Part 6` if we want to keep pointing at Swap (pedagogically — Mobile then Swap then CLI doesn't flow as well), OR (b) rewrite to point at the new CLI Part 5. **Recommended: rewrite (b)** — the new Part 5 is the natural continuation from Mobile E2E (you've tested two UI surfaces, now meet the third E2E surface — the CLI). |

Inside Part 4 there are also Chapter-internal cross-references like `Chapter 6.3` and `Chapter 6.3.5` — those refer to **Part 6 Mastery Ch 6.3 (CI/CD)**, not Part 6 the new Swap. They become **Chapter 7.3** and **Chapter 7.3.5** after renumber.

Lines that mention `Chapter 6.X` and need to become `Chapter 7.X`:
- 4601: "CI achieves parallelism across **shards**, not workers (Chapter 6.3)."  → Chapter 7.3
- 4677: "(the list of spec file paths computed by the sharding step (Chapter 6.3.5))"  → Chapter 7.3.5
- 4957: "The Chapter 6.3 CI workflow has a `speculos_device` input"  → Chapter 7.3
- 5029: "each CI shard is its own runner with its own simulator/emulator(s) (Chapter 6.3)"  → Chapter 7.3
- 6213: "The sharding algorithm (Chapter 6.3) is irrelevant here"  → Chapter 7.3

Also line 1119: `Part 1 Chapter 4 explained the monorepo layout` — refers to old Part 1 Ch 1.4. **Already incorrect prose (says "Chapter 4" when it means "1.4")**. Independent typo; not affected by renumber.

---

## 6. Part 5 (current Swap) and Part 6 (current Mastery)

### 6.1 Part 5 (Swap) H2 list (becomes new Part 6)

| Line | Current heading |
|-----:|-----------------|
| 1 | `## Live Apps in Ledger Wallet` |
| 310 | `## Swap Live App -- Architecture & Components` |
| 623 | `## Swap Live App -- Releases & Firebase Environments` |
| 850 | `## Real Ticket Walkthrough -- QAA-1136 (Swap Coverage Gap)` |
| 1426 | `## Part 5 Final Assessment` |

H3s use `### 5.X.Y Title` shape. After renumber, they all become `### 6.X.Y Title`.

Final Assessment slot: line 1426 → becomes `## Part 6 Final Assessment`. The anchor `<h3>Part 5 Final Assessment</h3>` (line 1429) becomes `<h3>Part 6 Final Assessment</h3>`. Subtitle line 1430 mentions `Part 5 chapters` — becomes `Part 6 chapters`.

### 6.2 Part 5 cross-references (everything to fix)

Lines where Part 5 references its own chapters (need internal renumber `5.X` → `6.X`):
- All H3 numbers (lines 7, 20, 40, 86, 153, 213, 228, 242, 316, 329, 345, 400, 444, 463, 473, 483, 502, 511, 555, 629, 654, 681, 699, 715, 741, 762, 782, 856, 877, 893, 947, 981, 999, 1041, 1087, 1110, 1128, 1151, 1180, 1268, 1358) — but those are headings, not in-text refs.

In-text cross-refs:
- Line 236: "see Chapter 4.5" — context says "feature flags can live in two projects … see Chapter 4.5". Chapter 4.5 in current Part 5 is about Releases/Firebase, but `4.5` is also Part 4 Detox from Zero. **Ambiguous prose**. After renumber, current 5.4.5 lives at 6.4.5? No — Part 5 currently has only 4 H2 chapters (5.1-5.4), so `Chapter 4.5` here is a loose forward reference to current 5.3 (Releases & Firebase). **Prose error**. Recommend: change to `see Chapter 6.3` or `see Section 5.3` (in old numbering) → `see Chapter 6.3` post-renumber.
- Line 347: `Chapter 4.5 covers the environments and URLs.` — same ambiguity. Should refer to 5.3 (now 6.3 post-renumber).
- Line 509: "(see Chapter 4.5)" — same as line 236. → Chapter 6.3 post-renumber.
- Line 853: "This is your Part 5 capstone chapter, following the exact same pattern as Part 3 Chapter 3.8 (QAA-1139) and Part 3 Chapter 3.9 (QAA-1141)." — `Part 5 capstone` becomes `Part 6 capstone`; `Part 3` references unchanged.
- Line 983: "Following the branch naming convention from Chapter 6 …" — `Chapter 6` here is suspicious; sounds like Chapter 6.X (Part 6 Mastery, branching/git). Becomes `Chapter 7.X` post-renumber. Recommend: replace with explicit `Part 7 Ch 7.X` if you can identify which subsection.
- Line 997: "see Chapter 6" — same, becomes Chapter 7.
- Line 1153: "Use the Claude Code command (see Chapter 3.8.9)" — `Part 3 Ch 3.8.9` unchanged.
- Line 1426: `## Part 5 Final Assessment` → `## Part 6 Final Assessment`.
- Line 1429: `<h3>Part 5 Final Assessment</h3>` → `<h3>Part 6 Final Assessment</h3>`.
- Line 1430: subtitle `Covers all Part 5 chapters` → `Covers all Part 6 chapters`.

Also: anchor `<a id="part-5-final-assessment"></a>` if present (need to verify) → becomes `part-6-final-assessment`.

### 6.3 Part 6 (Mastery) H2 list (becomes new Part 7)

| Line | Current heading |
|-----:|-----------------|
| 1 | `## Test Strategy & Best Practices` |
| 201 | `## Debugging Failures Like a Pro` |
| 334 | `## CI/CD & Test Sharding` |
| 1369 | `## AI Agents & Automation Rules` |
| 1613 | `## Part 6 Final Assessment` |

H3s use `### 6.X.Y` — all become `### 7.X.Y`.

### 6.4 Part 6 cross-references

| Line | Current | Action |
|-----:|---------|--------|
| 243 | `See Part 3, Chapter 3.3.8` | unchanged |
| 613 | `(see Part 3 Ch 3.1)` | unchanged |
| 615 | `(see Part 6 Ch 6.2)` | self-ref → `(see Part 7 Ch 7.2)` |
| 617 | `Cross-reference Part 3 Ch 3.1.5 …  Part 2 Ch 2.3` | unchanged |
| 871 | `Cross-reference Part 2 Ch 2.5` | unchanged |
| 1013 | `Cross-reference … Part 3 Ch 3.8 (desktop) and Part 4 Ch 4.9 (mobile)` | unchanged |
| 1267 | `### 6.3.13 Chapter 6.3 Quiz` | becomes `### 7.3.13 Chapter 7.3 Quiz` |
| 1545 | `<!-- ── Chapter 6.4 Quiz ── -->` | becomes `Chapter 7.4 Quiz` |
| 1613 | `## Part 6 Final Assessment` | becomes `## Part 7 Final Assessment` |
| 1615 | "the end of the guide. Part 6 covered…  Chapter 6.1 / 6.2 / 6.3 / 6.4 …" | becomes Part 7 / Chapter 7.1 / 7.2 / 7.3 / 7.4 |
| 1620 | `<h3>Part 6 Final Assessment</h3>` | becomes `<h3>Part 7 Final Assessment</h3>` |
| (anchor) | `<a id="part-6-final-assessment">` if present | becomes `part-7-final-assessment` |

---

## 7. Sidebar + README — current and proposed shapes

### 7.1 Current `_sidebar.md`

(Exact contents above; reproduced here in compact form.)

```
- [**Part 4 -- Mobile E2E**](part4)
  - 4.1 … 4.11
  - [Part 4 Assessment](part4#part-4-final-assessment)

- [**Part 5 -- Swap Live App**](part5)
  - 5.1 Live Apps in Ledger Wallet
  - 5.2 Swap Architecture
  - 5.3 Releases & Firebase
  - 5.4 Walkthrough: QAA-1136
  - [Part 5 Assessment](part5#part-5-final-assessment)

- [**Part 6 -- Mastery & Contributing**](part6)
  - 6.1 Test Strategy & Best Practices
  - 6.2 Debugging Failures
  - 6.3 CI/CD & Test Sharding
  - 6.4 AI Agents & Automation
  - [Part 6 Assessment](part6#part-6-final-assessment)
```

### 7.2 Proposed `_sidebar.md` after CLI insertion

```
- [**Part 4 -- Mobile E2E**](part4)
  - … (unchanged)
  - [Part 4 Assessment](part4#part-4-final-assessment)

- [**Part 5 -- CLI Automation**](part5)
  - [5.1 CLI Architecture](part5#cli-automation-architecture-deep-dive)
  - [5.2 First wallet-cli Command](part5#your-first-wallet-cli-command)
  - [5.3 Bun & Bunli Primer](part5#bun-amp-bunli-primer)
  - [5.4 DMK Primer](part5#device-management-kit-primer)
  - [5.5 wallet-cli Codebase](part5#wallet-cli-codebase-deep-dive)
  - [5.6 Account Descriptors V1](part5#account-descriptors-v1)
  - [5.7 Running & Debugging](part5#running-amp-debugging-wallet-cli)
  - [5.8 Testing wallet-cli](part5#testing-wallet-cli-with-mock-dmk-and-http-intercept)
  - [5.9 Daily CLI Workflow](part5#your-daily-cli-workflow-from-ticket-to-pr)
  - [5.10 Walkthrough: QAA-615](part5#real-ticket-walkthrough-qaa-615-wallet-cli-revoke)
  - [5.11 CLI Exercises](part5#cli-exercises-amp-challenges)
  - [Part 5 Assessment](part5#part-5-final-assessment)

- [**Part 6 -- Swap Live App**](part6)
  - [6.1 Live Apps in Ledger Wallet](part6#live-apps-in-ledger-wallet)
  - [6.2 Swap Architecture](part6#swap-live-app-architecture-amp-components)
  - [6.3 Releases & Firebase](part6#swap-live-app-releases-amp-firebase-environments)
  - [6.4 Walkthrough: QAA-1136](part6#real-ticket-walkthrough-qaa-1136-swap-coverage-gap)
  - [Part 6 Assessment](part6#part-6-final-assessment)

- [**Part 7 -- Mastery & Contributing**](part7)
  - [7.1 Test Strategy & Best Practices](part7#test-strategy-amp-best-practices)
  - [7.2 Debugging Failures](part7#debugging-failures-like-a-pro)
  - [7.3 CI/CD & Test Sharding](part7#cicd-amp-test-sharding)
  - [7.4 AI Agents & Automation](part7#ai-agents-amp-automation-rules)
  - [Part 7 Assessment](part7#part-7-final-assessment)
```

This requires renaming files: `part5.md` (Swap) → `part6.md`, `part6.md` (Mastery) → `part7.md`, then creating new `part5.md` (CLI). Use `git mv` to preserve history.

### 7.3 README current (relevant slice)

```
| **5** | [Swap Live App](part5.md) | 5.1-5.4 | … |
| **6** | [Mastery & Contributing](part6.md) | 6.1-6.4 | … |
```

Plus the prose at line 22: `…desktop E2E, mobile E2E, the Swap Live App, and the mastery layer covering test strategy, debugging, CI/CD, and AI-assisted automation.`

Plus line 24: `Parts 3-4 are your desktop and mobile deep dives; Part 5 covers the Swap Live App; Part 6 closes with the contributor mindset.`

Plus line 11: `<span … >45 Chapters</span>` and line 12: `52 Interactive Quizzes`.

Plus line 37: `Seven progressive parts (0 through 6)`.

Plus line 64: row for Part 6 + row for Part 5 in the reading-order table.

### 7.4 README proposed

```
| **5** | [CLI Automation](part5.md) | 5.1-5.11 | wallet-cli architecture, Bun + Bunli, DMK, account descriptors, mock DMK testing, daily CLI workflow, QAA-615 walkthrough, exercises |
| **6** | [Swap Live App](part6.md) | 6.1-6.4 | Live Apps in Ledger Wallet, Swap architecture, releases & Firebase environments, QAA-1136 walkthrough |
| **7** | [Mastery & Contributing](part7.md) | 7.1-7.4 | Test strategy & best practices, cross-platform debugging, CI/CD & test sharding, AI agents & automation |
```

Prose updates:
- Line 22: insert "CLI automation" between "mobile E2E" and "the Swap Live App".
- Line 24: change to: `Parts 3-5 are your desktop, mobile, and CLI deep dives; Part 6 covers the Swap Live App; Part 7 closes with the contributor mindset.`
- Line 11: `45 Chapters` → recompute. New Part 5 adds 11 H2s. So ~45 + 11 = **~56 Chapters**.
- Line 12: `52 Interactive Quizzes` → +11 chapter quizzes + 1 final = **+12 → 64 Interactive Quizzes**.
- Line 37: `Seven progressive parts (0 through 6)` → `Eight progressive parts (0 through 7)`.

---

## 8. Renumber matrix (file/line | old → new | reason)

This is the master list. Each row = one substitution to apply.

### 8.1 File renames (do these first, with `git mv`)

| Operation | Reason |
|-----------|--------|
| `git mv part6.md part7.md` | Mastery moves to Part 7 |
| `git mv part5.md part6.md` | Swap moves to Part 6 |
| Create new `part5.md` | New CLI part |

### 8.2 Inside (new) `part6.md` (formerly Swap, was `part5.md`)

| Line (orig) | Old text | New text | Reason |
|-----------:|----------|----------|--------|
| 236 | `(see Chapter 4.5)` | `(see Chapter 6.3)` | Stale ref; was misnumbered; correct target is Releases/Firebase chapter, now 6.3 |
| 347 | `Chapter 4.5 covers the environments and URLs.` | `Chapter 6.3 covers the environments and URLs.` | same |
| 509 | `(see Chapter 4.5)` | `(see Chapter 6.3)` | same |
| 853 | `your Part 5 capstone chapter` | `your Part 6 capstone chapter` | renumber |
| 983 | `from Chapter 6` | `from Part 7` | renumber + clarify (was Mastery, now Part 7) |
| 997 | `see Chapter 6` | `see Part 7` | renumber |
| ~1426 | `## Part 5 Final Assessment` | `## Part 6 Final Assessment` | renumber |
| ~1429 | `<h3>Part 5 Final Assessment</h3>` | `<h3>Part 6 Final Assessment</h3>` | renumber |
| ~1430 | `Covers all Part 5 chapters` | `Covers all Part 6 chapters` | renumber |
| (any) | `<a id="part-5-final-assessment">` | `<a id="part-6-final-assessment">` | renumber (verify presence) |
| Headings | `### 5.X.Y` | `### 6.X.Y` | global replace within file (see below) |
| Headings | `## Part 5 Final Assessment` | `## Part 6 Final Assessment` | covered above |

For the H3 renumber inside the new part6.md, run: `sed -i '' 's/^### 5\./### 6./' part6.md`. There are ~40 such lines.

### 8.3 Inside (new) `part7.md` (formerly Mastery, was `part6.md`)

| Line (orig) | Old text | New text | Reason |
|-----------:|----------|----------|--------|
| 615 | `(see Part 6 Ch 6.2)` | `(see Part 7 Ch 7.2)` | self-ref renumber |
| 1267 | `### 6.3.13 Chapter 6.3 Quiz` | `### 7.3.13 Chapter 7.3 Quiz` | renumber |
| 1369 | `## AI Agents & Automation Rules` (H3 numbers `6.4.X` inside) | unchanged H2; renumber inside `### 6.4.X` → `### 7.4.X` | sed |
| 1545 | `<!-- ── Chapter 6.4 Quiz ── -->` | `<!-- ── Chapter 7.4 Quiz ── -->` | renumber |
| 1613 | `## Part 6 Final Assessment` | `## Part 7 Final Assessment` | renumber |
| 1615 | prose mentioning `Part 6`, `Chapter 6.1`, `6.2`, `6.3`, `6.4` | swap each `6` → `7` | renumber |
| 1620 | `<h3>Part 6 Final Assessment</h3>` | `<h3>Part 7 Final Assessment</h3>` | renumber |
| (any) | `<a id="part-6-final-assessment">` | `<a id="part-7-final-assessment">` | renumber |
| Headings | `### 6.X.Y` | `### 7.X.Y` | global sed |

For Part 7 H3 renumber: `sed -i '' 's/^### 6\./### 7./' part7.md`. Plus all `Chapter 6.X` references inside prose need to become `Chapter 7.X`. Use a careful global replace (NOT chapter 6.X.Y — chapter X.Y where X starts at 1).

Caveat: there are also references in this file to `Part 3 Ch 3.X`, `Part 2 Ch 2.X`, `Part 4 Ch 4.X` — those are **not** to be touched. Use `grep -n "Part 6\b" part7.md` and `grep -n "Chapter 6\." part7.md` to enumerate carefully before sed.

### 8.4 Inside `part4.md`

| Line | Old | New | Reason |
|-----:|-----|-----|--------|
| 4091 | `see Part 6 Chapter 6.3` | `see Part 7 Chapter 7.3` | Mastery→Part 7 |
| 4601 | `(Chapter 6.3)` | `(Chapter 7.3)` | same |
| 4677 | `(Chapter 6.3.5)` | `(Chapter 7.3.5)` | same |
| 4957 | `Chapter 6.3` | `Chapter 7.3` | same |
| 5029 | `(Chapter 6.3)` | `(Chapter 7.3)` | same |
| 6213 | `(Chapter 6.3)` | `(Chapter 7.3)` | same |
| 5084 | `Part 5` (in prose about Xray invisibility) | `Part 2` (likely the original intent — Allure & Xray is Part 2 Ch 2.5) | typo fix; verify with author; not strictly a renumber |
| 6392 | `<strong>Next: Part 5</strong> steps out of the apps and into the shared runtime — the <strong>Ledger Wallet Components</strong> library and the <strong>Swap Live App</strong>` | rewrite to: `<strong>Next: Part 5</strong> steps off the GUI altogether and onto the command line — the <strong>wallet-cli</strong> workspace that powers test-data hooks like the QAA-615 token-approval revoke. Same Bun runtime, same DMK transport, same Bunli command shape — but no UI to drive.` | new Part 5 is CLI |
| 860 | `during the rest of Part 6` | `during the rest of Part 4` | typo fix; not affected by renumber |

### 8.5 Inside `_sidebar.md`

Wholesale rewrite of lines 55-67 per §7.2 above. Plus optional new Part 5 block inserted at line 55.

### 8.6 Inside `README.md`

| Line | Old | New | Reason |
|-----:|-----|-----|--------|
| 11 | `45 Chapters` | `56 Chapters` | +11 from new Part 5 |
| 12 | `52 Interactive Quizzes` | `64 Interactive Quizzes` | +11 chapter quizzes + 1 final |
| 22 | prose "the Swap Live App, and the mastery layer" | "CLI automation, the Swap Live App, and the mastery layer" | reflect new structure |
| 24 | "Parts 3-4 are your desktop and mobile deep dives; Part 5 covers the Swap Live App; Part 6 closes" | "Parts 3-5 are your desktop, mobile, and CLI deep dives; Part 6 covers the Swap Live App; Part 7 closes" | renumber + new part |
| 37 | `Seven progressive parts (0 through 6)` | `Eight progressive parts (0 through 7)` | +1 part |
| 63 | row `\| **5** \| [Swap Live App] \| …` | replace with new CLI row + shifted Swap row | +1 part |
| 64 | row `\| **6** \| [Mastery & Contributing] \| …` | shift to `**7**` | +1 part |

Approximate total renumber-pass operations: **~55 individual substitutions** plus 2 file renames plus 1 new file.

---

## 9. Proposed Part 5 CLI chapter outline

Briefing requested 11 chapters mirroring Part 4. Recommendation below addresses the "is 11 too many" question.

### 9.1 Recommended 11-chapter outline (mirrors Part 4)

| # | Title | H3 count target | Notes |
|---|-------|----------------:|-------|
| 5.1 | CLI Automation -- Architecture Deep Dive | 5-6 | wallet-cli vs legacy `apps/cli`; place in monorepo; relation to E2E hooks; runtime model (Bun, single binary, no UI). End with Quiz. |
| 5.2 | Your First wallet-cli Command | 5-6 | `pnpm wallet-cli start receive --currency bitcoin`. Walk a single command end-to-end. |
| 5.3 | Bun & Bunli Primer | 6-8 | Bun runtime (Zig, native FS APIs, `bun test`), Bunli framework (command tree, flag parsing, `define()` API). |
| 5.4 | Device Management Kit Primer | 6-8 | DMK transport, USB session, APDU exchange, signer/runner abstraction, mock DMK. |
| 5.5 | wallet-cli Codebase Deep Dive | 12-15 | Every file in `apps/wallet-cli/src/` — commands/, lib/, transports/, signers/, test/. Mirror Part 4 4.7 shape. |
| 5.6 | Account Descriptors V1 | 5-7 | The `account:1:<type>:<network>:<env>:<xpub>:<path>` string format, why it matters for cross-language interop, parser details. |
| 5.7 | Running & Debugging wallet-cli | 8-10 | Local invocation, `--verbose`, log files, debugger attach (Bun inspector), Speculos integration. |
| 5.8 | Testing wallet-cli with Mock DMK and HTTP Intercept | 8-10 | `bun test` setup, `cli-runner`, `mock-DMK`, `http-intercept`, `mock-server`. The infrastructure that lets a CLI test run hermetically. |
| 5.9 | Your Daily CLI Workflow: From Ticket to PR | 10-12 | Mirror Part 4 4.9. Pick ticket, branch, write command, write test, run mock then real, PR. |
| 5.10 | Real Ticket Walkthrough: QAA-615 — wallet-cli revoke | 14-18 | The capstone. ERC20 approval/revoke; spike comparison of three implementation paths; ship the chosen one; integrate into a hook for QAA-613 broadcast tests. |
| 5.11 | CLI Exercises & Challenges | 7-8 | Mirror Part 4 4.11. Add a new currency, add a new command, write a fixture-style hook, port a Speculos call from desktop fixtures. |
| – | Part 5 Final Assessment | 10 questions | Mirror Part 4 final. |

### 9.2 Question for the user — flagged in report

> **Is 11 chapters too many for the CLI part?** wallet-cli is a smaller surface area than Detox + the entire mobile toolchain. A trimmed 8-chapter outline would be tighter:
>
> 1. Architecture Deep Dive
> 2. Your First wallet-cli Command
> 3. Bun & Bunli Primer (merged with DMK Primer? — possibly two separate)
> 4. Device Management Kit Primer
> 5. Codebase Deep Dive (incl. Account Descriptors as one of the subsections)
> 6. Running, Debugging & Testing (merged 5.7 + 5.8)
> 7. Daily Workflow + QAA-615 Walkthrough (merged 5.9 + 5.10)
> 8. Exercises
> + Final Assessment
>
> **R6's recommendation: 11 chapters, mirror Part 4.** Reasoning:
>
> - Pedagogical parity with Part 3 / Part 4 — readers expect the same shape on each E2E flavor and learning to navigate that shape is itself part of the onboarding.
> - The codebase deep-dive deserves its own chapter (5.5) — `apps/wallet-cli/src/` has ~20 files and the test infrastructure has another ~5; conflating that with the Bunli primer hides too much.
> - Account descriptors (5.6) are a self-contained spec worth a focused chapter — they're the contract between wallet-cli, DMK signers, and the upcoming hook patterns. Burying them in 5.5 dilutes them.
> - The QAA-615 walkthrough (5.10) is the spike deliverable — it must be a top-level chapter so future readers find it via the sidebar without spelunking.
> - "Daily workflow" (5.9) before the walkthrough establishes the lifecycle vocabulary the walkthrough then uses.
> - Worst case, individual chapters are shorter than Part 4 equivalents — that is fine and consistent with smaller surface area. Better short-and-clear than merged-and-muddled.
>
> Counter-argument (`~ Contestable` per the Socratic rule): if the writer estimates 5.7 + 5.8 will each be under ~50 lines, merge them. Decide after the writer drafts both.

---

## 10. Title sanitisation — final pass

Verified anchors, clean of apostrophes/colons/parens/`&` ambiguity:

| Final title | Final anchor | OK? |
|-------------|--------------|-----|
| `## CLI Automation -- Architecture Deep Dive` | `cli-automation-architecture-deep-dive` | yes |
| `## Your First wallet-cli Command` | `your-first-wallet-cli-command` | yes |
| `## Bun & Bunli Primer` | `bun-amp-bunli-primer` | yes |
| `## Device Management Kit Primer` | `device-management-kit-primer` | yes (dropped DMK acronym + parens) |
| `## wallet-cli Codebase Deep Dive` | `wallet-cli-codebase-deep-dive` | yes |
| `## Account Descriptors V1` | `account-descriptors-v1` | yes |
| `## Running & Debugging wallet-cli` | `running-amp-debugging-wallet-cli` | yes |
| `## Testing wallet-cli with Mock DMK and HTTP Intercept` | `testing-wallet-cli-with-mock-dmk-and-http-intercept` | yes (no parens, no plus signs) |
| `## Your Daily CLI Workflow: From Ticket to PR` | `your-daily-cli-workflow-from-ticket-to-pr` | yes (colon strips cleanly mid-title; matches Part 4 4.9 pattern exactly) |
| `## Real Ticket Walkthrough: QAA-615 — wallet-cli revoke` | `real-ticket-walkthrough-qaa-615-wallet-cli-revoke` | yes (colon and em-dash both swallowed) |
| `## CLI Exercises & Challenges` | `cli-exercises-amp-challenges` | yes |
| `## Part 5 Final Assessment` | `part-5-final-assessment` | yes |

Avoid:
- Apostrophes: use "Daily" not "Today's" etc.
- Parentheses with content: `(DMK)` and `(mock DMK + http intercept + cli-runner)` both produce flaky anchors.
- Plus signs: docsify slugger sometimes encodes them; safer to write "and".
- Slashes: avoid `wallet-cli / DMK`; use "and" or em-dash.

---

## 11. Final checklist for the writers

Before the writers start drafting Part 5, hand them this short list:

1. Use the H2 + H3 numbering shown in §9.1.
2. Each chapter starts with a `<div class="chapter-intro">` paragraph (60-200 words, see §3.1).
3. Each chapter ends with a `<div class="chapter-outro">` paragraph (80-300 words) **before** the Quiz H3.
4. Each chapter ends with `### 5.X.Y Quiz` and the `<div class="quiz-container" data-pass-threshold="80">` block (see §2.1 for byte-for-byte template).
5. Each quiz has 4-7 questions; default is 5.
6. Final Assessment: 10 questions, includes `<a id="part-5-final-assessment"></a>` above the quiz container, includes a closing `<div class="chapter-outro">` after the quiz that says "Next: Part 6" and previews Swap.
7. Cross-references to Part 0/1/2/3/4 stay as `Part X Chapter X.Y`. Internal references stay `Chapter 5.X.Y`. References to old Part 5 (Swap) become `Part 6`. References to old Part 6 (Mastery) become `Part 7`.
8. Run a final sed pass after writing:
   ```
   grep -n "Part [567]\b" part5.md     # sanity-check forward refs
   grep -n "Chapter [567]\." part5.md  # sanity-check cross-refs
   ```
9. Update `_sidebar.md` and `README.md` per §7.2 and §7.4.
10. After CLI part lands, do the renumber pass on the existing files per §8.

---

## Appendix: tight numerical summary

- Part 4 = 11 chapters + Final Assessment, ~6,400 lines total.
- Part 5 (current Swap) = 4 chapters + Final Assessment, ~1,570 lines.
- Part 6 (current Mastery) = 4 chapters + Final Assessment, ~1,740 lines.
- Sidebar = 76 lines.
- README = 71 lines.
- Total renumber substitutions: ~55 individual edits + 2 file renames + 1 new file.
- New Part 5 target size: 6,000-8,000 lines (depends on whether 5.10 walkthrough captures the QAA-615 spike with full code or summary).
- New global chapter count: 56 (was 45) — assumes 11 new H2s in Part 5.
- New global quiz count: 64 (was 52) — 11 new chapter quizzes + 1 new final.
