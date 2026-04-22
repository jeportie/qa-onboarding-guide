# R4 — Jira Ticket Flow, Xray Wiring & Naming Conventions

Scope: Ledger QA Jira project structure, ticket workflow, Xray mapping, status transitions, labelling conventions and prefix naming (QAA, B2CQA, LL, LLM, PTX, LIVE).

Source: https://ledgerhq.atlassian.net (cloudId 5576c6af-2aa6-451e-b92d-9c02363b46fa). Date 2026-04-22.

Constraints during research:
- getJiraProjectIssueTypesMetadata, getTransitionsForJiraIssue, and Confluence searchConfluenceUsingCql / getConfluencePage were denied by the current sandbox permission layer. Issue-type and workflow data were inferred from JQL samples (searchJiraIssuesUsingJql) and from the status, issuetype, and labels fields returned on real tickets.
- PTX, LL, LLM, BACKLOG are NOT actual Jira project keys on the Ledger cloud. They are used as informal label / module prefixes. See section 3 (Naming & prefixes) for the clarified mapping.

---

## 1. Jira Project Matrix

| Project key | Full name                  | Category          | Project type      | Purpose                                                            | Main issue types                                                  | Primary stakeholder team   |
|-------------|----------------------------|-------------------|-------------------|--------------------------------------------------------------------|-------------------------------------------------------------------|----------------------------|
| LIVE        | Ledger Wallet              | Engineering B2C   | software (classic)| Product backlog for Ledger Live Desktop + Mobile (aka LL / LLD / LLM). Holds epics, stories, tasks, bugs and Xray Test cases for the product. | Bug, Task, Story, Epic, Sub-task, Test (Xray 10510)               | LL dev teams + QA + PM     |
| B2CQA       | B2CQA                      | Engineering Paris | software (next-gen, team-managed) | QA repository for Ledger Live. Stores manual test cases, Test Plans, Test Executions and QA-only Bugs. Source of truth for the Xray TMS on B2C side. | Test (Xray 10756), Test Plan (10759), Test Execution (10760), Bug (10450), Task | QA team (Quality Assurance)|
| QAA         | QA Automation              | -                 | software (classic)| Backlog for the QA Automation squad. Tracks tasks to automate B2CQA test cases, spikes, CI/infra work, Allure/Xray tooling. | Task, Bug, Story, Epic, Sub-task                                  | QA Automation squad        |
| WO          | Wallet Operations          | -                 | software (classic)| Downstream support / operations ticketing around Ledger Wallet (release ops, incident follow-up). | Task, Bug, Story                                                  | Wallet Ops / Support       |
| CSERVICES   | Ledger Wallet Discovery    | Delivery          | product_discovery | Discovery board for Wallet services ideation.                       | Idea (JPD)                                                        | Product discovery          |
| QUEST       | Ledger Quest               | Engineering B2C   | software (classic)| Growth/engagement (Quests) delivery backlog adjacent to LIVE.       | Task, Story, Bug                                                  | Growth / Engagement        |
| LB          | Ledger Button              | -                 | product_discovery | Discovery for the Ledger Button (Sign-in with Ledger) feature.      | Idea                                                              | LB discovery               |
| LBD         | Ledger Button Delivery     | -                 | software (classic)| Delivery backlog for Ledger Button (auth/Sync). QA surfaces here with dedicated Test Plans (see B2CQA-4611). | Task, Story, Bug, Test Plan (referenced from B2CQA)               | LB delivery + QA           |
| OS          | Ledger OS Discovery        | Delivery          | product_discovery | Firmware / OS discovery                                            | Idea                                                              | OS team                    |
| SECU        | Ledger Security            | Delivery          | product_discovery | Security discovery                                                 | Idea                                                              | Security team              |
| DAFLE       | Data Analytics for Ledger Enterprise | -       | software          | B2B analytics tooling                                              | Task, Story                                                       | Enterprise analytics       |

Non-project prefixes / aliases seen in the wild (see section 3):
- LL  -- informal alias for the LIVE project (Ledger Live).
- LLD -- Ledger Live Desktop -- appears only as a label on LIVE issues.
- LLM -- Ledger Live Mobile -- appears only as a label on LIVE issues.
- LWD / LWM -- Ledger Wallet Desktop / Mobile (v4 rebrand of LLD/LLM), used in summaries and labels on LIVE.
- PTX -- referenced in onboarding docs as Protect / Ledger Recover flows; not a Jira project key on the cloud -- this work is filed inside LIVE under Recover/Apex epics (e.g. LIVE-21062, LIVE-16602) with labels Recover, Apex.
- BACKLOG -- not a Jira project; colloquial name for the unrefined LIVE + B2CQA ingress lane.

---

## 2. Workflow Diagrams (inferred from observed statuses)

### 2.1 LIVE (LL / LLM / LLD) bug workflow

Observed statuses on LIVE Bug tickets: New, TO DO, Open, In Progress, Code Review, QA, Done, Released, Abandoned, On Hold, Cancelled.

```
                +-----+
                | New |   <-- default for bug on creation (id 10003)
                +--+--+
                   |
                   v
     +--------+  +-----+
     | TO DO  |->|Open |   (triage decision, owner assigned)
     +----+---+  +--+--+
          |         |
          v         v
          +----+--------+
               |
               v
        +------+------+                 +---------+
        | In Progress |---------------> | On Hold |
        +------+------+                 +----+----+
               |                             |
               v                             v
        +------+------+                 (back to In Progress)
        | Code Review |
        +------+------+
               |
               v
        +------+------+   (QA verification on nightly / release build)
        |     QA      |
        +------+------+
               |
      +--------+--------+
      v                 v
+-----+----+      +-----+-----+
|   Done   |      | Abandoned |  (rejected / dup / cannot-repro)
+----+-----+      +-----------+
     |
     v
+----+-----+
| Released |  (shipped in a version)
+----+-----+
     |
     v
+----+------+
| Cancelled |  (post-release removal, scoped-out)
+-----------+
```

Transition owners (by convention observed on ticket comments and assignee changes):
- New -> TO DO / Open: Triage lead (PM/QA) re-classifies reporter draft.
- Open -> In Progress: Dev picks up.
- In Progress -> Code Review: Dev pushes PR; reviewer added.
- Code Review -> QA: PR merged, build tagged; QA engineer assigned for verification on nightly (ledger-live-mobile 4.1.0-nightly.*, ledger-live-desktop nightly).
- QA -> Done: QA signs off; bug flagged deploymentPROD label to schedule prod release.
- Done -> Released: Release manager after store publication.
- Any -> Abandoned / Cancelled: Triage or PM.

### 2.2 B2CQA -- Test case workflow

Observed statuses on Test (10756) and Test Plan (10759): To Do, In Test Review, Manual Test, Automated, Obsolete, Done (Execution only).

```
              +-------+
              | To Do |   author drafts a manual test case
              +---+---+
                  |
                  v
         +--------+---------+
         |  In Test Review  |  peer reviewer / test lead
         +----+---------+---+
              |         |
              v         v
      +-------+--+   +--+--------+
      | Manual   |   | Obsolete  |  (retired / superseded)
      |  Test    |   +-----------+
      +----+-----+
           |
           v   (QA Automation squad links to QAA-xxxx)
      +----+-----+
      | Automated|   status once an e2e/unit spec implements the Xray
      +----------+   $TmsLink('B2CQA-xxxx') tag; test becomes part of
                     nightly / release CI.
```

Test Execution (10760) has a lighter 2-state loop:
```
   +------+      +------+
   | TO DO|  ->  | Done |
   +------+      +------+
```
(Used for one-off runs: Nano Gen5 1.1.0-rc5 non-reg = B2CQA-5449.)

### 2.3 QAA -- Automation backlog workflow

Standard Jira software workflow:
```
  Open --> In Progress --> In Review --> Done
              |                            ^
              v                            |
          On Hold --------------------------
```
Abandoned exists for QAA Bug type when the failure is not reproducible or is owned by another squad (e.g. QAA-1168).

---

## 3. Ticket Naming & Prefix Conventions

| Prefix    | Real Jira project? | Used for                                                       | Who files         |
|-----------|--------------------|----------------------------------------------------------------|-------------------|
| LIVE-*    | Yes (LIVE)         | Product work on Ledger Live / Ledger Wallet (LLD + LLM + LWD + LWM). Bugs, stories, epics, product-side Xray Test (10510). | Devs, PMs, QA, Support |
| B2CQA-*   | Yes (B2CQA)        | Manual test cases, Test Plans, Test Executions, QA-owned bugs (automation flakes not in LIVE scope). Source of truth for Xray on B2C. | QA engineers |
| QAA-*     | Yes (QAA)          | Automation engineering tasks: implement an automation, CI pipeline work, check-funds tooling, Allure upload, spikes. | QA Automation |
| LL-*      | No                 | Colloquial alias for LIVE-* in Slack, commits, PR titles. Must be read as LIVE-*. | - |
| LLM       | No (label)         | labels = "LLM" on LIVE tickets to tag Mobile-only defects.    | Reporter |
| LLD       | No (label)         | labels = "LLD" on LIVE tickets for Desktop-only.              | Reporter |
| LWM/LWD   | No (summary tag)   | Bracketed in summaries ([LWM], [LWD]) for Wallet v4 rebrand.  | Reporter |
| PTX-*     | No                 | Protect / Recover work. Filed under LIVE with labels Recover, Apex. | - |
| BACKLOG   | No                 | Informal, meaning unrefined work awaiting triage in any project.| - |

### Linking rule -- Xray / $TmsLink

A Playwright / Detox / WDIO test in the ledger-live monorepo that implements a B2CQA test case annotates it with the Xray TMS tag:

```ts
$TmsLink("B2CQA-2697")
```

The automation step in the QAA backlog (e.g. QAA-1129 Automate Receive address warning tests on Mobile) references the target B2CQA test in a description table:

```
| Xray ticket    | Test name                        | Coin(s) |
| B2CQA-2697     | Receive - verify address warning | ETH     |
```

On execution the Xray runner pushes the result back to the B2CQA Test object and its parent Test Plan / Execution. When the automation lands, the B2CQA test's status is manually moved Manual Test -> Automated by the QA automation owner.

Relationship between B2CQA test variants and a parent manual test is expressed via Work item split Jira link type (splits from / splits to). Example: B2CQA-2697 splits from B2CQA-652.

---

## 4. Status Map

| Status name       | Applied to                               | Workflow category | Owner of the transition                               |
|-------------------|------------------------------------------|-------------------|-------------------------------------------------------|
| New               | LIVE bug, QAA bug                        | To Do             | Jira default on create                                |
| TO DO / To Do     | LIVE, QAA, B2CQA                         | To Do             | Reporter / triage                                     |
| Open              | QAA tasks                                | To Do             | Assignee pick-up                                      |
| On Hold           | LIVE                                     | To Do             | Dev / PM                                              |
| In Progress       | LIVE, QAA                                | In Progress       | Dev starts work                                       |
| Code Review       | LIVE                                     | In Progress       | Dev opens PR -> GitHub integration updates field      |
| QA                | LIVE                                     | In Progress       | QA engineer (manual verify on nightly)                |
| In Test Review    | B2CQA                                    | To Do             | Peer QA reviewer on drafted test                      |
| Manual Test       | B2CQA                                    | Done (green)      | Test lead when test case is validated for exec        |
| Automated         | B2CQA                                    | Done (green)      | QA Automation after $TmsLink ships in a spec          |
| Obsolete          | B2CQA                                    | Done (green)      | Test lead (cleanup)                                   |
| Done              | LIVE, QAA, B2CQA Test Exec               | Done (green)      | Assignee                                              |
| Released          | LIVE                                     | Done (green)      | Release manager after store publication               |
| Abandoned         | LIVE, QAA, B2CQA                         | Done (green)      | Triage / PM -- rejected, duplicate, cannot-repro      |
| Cancelled         | LIVE                                     | Done (green)      | PM when a test/story is dropped post-refinement       |

The two To Do / TO DO entries (ids 10000 and 10630) come from different workflow schemes (classic LIVE vs next-gen B2CQA). LIVE also has a legacy uppercased TO DO (id 10000).

---

## 5. Label Conventions (top LIVE Bug labels, last ~100 updated)

Frequency out of 100 most-recently updated LIVE Bug tickets:

| Count | Label                   | Meaning                                                                |
|-------|-------------------------|------------------------------------------------------------------------|
| 34    | deploymentPROD          | Fix must be cherry-picked into the next production release             |
| 20    | localization_issues     | i18n / translation defect (handled by copy team)                       |
| 16    | created_by_triage       | Automatic stamp from the triage automation bot                         |
| 15    | Wallet-v4               | Scope: Wallet v4 (the LWM / LWD rebrand)                               |
| 15    | qaa                     | Ticket was raised by or involves the QA Automation CI                  |
| 7     | Growth                  | Engagement / Quest team scope                                          |
| 7     | blocker                 | Blocks release; pair with deploymentPROD                               |
| 3     | Refined                 | Passed grooming                                                        |
| 2     | LLM                     | Mobile-only                                                            |
| 2     | Analytics               | Analytics/tracking event defect                                        |
| 1     | Perps                   | Perpetuals integration                                                 |
| 1     | onboarding              | Onboarding flow                                                        |
| 1     | lw                      | Ledger Wallet catch-all                                                |
| 1     | canton_LL_integration   | Canton chain integration                                               |
| 1     | Braze                   | Braze (push notifications) integration                                 |
| 1     | Aleo_Integration        | Aleo chain integration                                                 |

Also observed on QAA tickets: LLD, LLM, UI, CI, automation, check-funds, next-env.

Guidance:
- deploymentPROD + blocker is the standard tag set the release manager uses to freeze a hotfix branch.
- qaa is what QA Automation searches on to know "a bug was caught by e2e automation, someone please look at it".
- LLM / LLD replace "components" -- LIVE project deliberately does not use Jira components, so label is the dimension to filter on.

---

## 6. Xray Wiring

Xray is configured on both LIVE and B2CQA projects. Issue-type IDs observed:
- LIVE uses Xray Test type 10510 (description: "This is the Xray Test Issue Type...").
- B2CQA uses Xray Test 10756, Test Plan 10759 and Test Execution 10760.

### 6.1 Test authoring (B2CQA)

1. QA engineer files a Test in B2CQA, status To Do -> In Test Review. Summary convention: [Feature][Platform] <user story>, e.g. V4 - Tx History - US2: Access transaction history (B2CQA-5359).
2. Peer reviewer moves to Manual Test -- it is now executable.
3. The Test is attached to one or more Test Plans (B2CQA-4611 [Ledger Button] Non regression Test Plan, B2CQA-5551 Wallet 4.0 - Asset Aggregation, B2CQA-5456 Wallet 4.0 - My Wallet).

### 6.2 Automation (QAA -> B2CQA)

1. QA Automation squad picks a QAA task referencing one or more B2CQA-* tests (example: QAA-1129 lists B2CQA-2697/2698/2699 with the label table in its description).
2. Spec files in apps/ledger-live-mobile/e2e/specs/** and apps/ledger-live-desktop/tests/specs/** gain a $TmsLink("B2CQA-xxxx") annotation.
3. When the spec is merged and green, the engineer transitions the parent B2CQA test from Manual Test to Automated. This is done manually today.

### 6.3 Execution (B2CQA Test Execution)

A release pipeline creates a Test Execution ticket (e.g. B2CQA-5449 Nano Gen5 1.1.0-rc5 non-reg, B2CQA-5445 Stax 1.10.0-rc5 non-reg). The execution links to a Test Plan and contains pass/fail records per test. Automated executions are pushed by the Xray runner via CI; manual executions are filled in by QA engineers during the release gate.

### 6.4 Roll-up to LIVE

- A failing Test under a B2CQA Test Execution triggers the QA engineer to open a Bug in LIVE (not B2CQA) with label qaa and a link to the failed Allure report (ledger-live.allure.green.ledgerlabs.net).
- The LIVE bug carries the label(s) LLM and/or LLD plus deploymentPROD if it must ship in the release being validated.
- B2CQA is ONLY the destination for bug-in-test (flaky automation, selector issues, test plan drift): e.g. B2CQA-4419 [SWAP][e2e] Search field of asset is not found, B2CQA-4467 [SWAP][LWM] Unexpected 'deeplink' test failure. These have issuetype 10450 (B2CQA-local Bug type) and never leak into LIVE.
- The QAA project takes infra bugs (Allure upload 500s, check-funds missing accounts) -- e.g. QAA-1163, QAA-1162.

---

## 7. Ticket Lifecycle Walkthrough -- "A QA bug found in E2E"

Grounded on live ticket LIVE-29397 [SWAP][1inch] POST /api.1inch.dev fails often, making swap to fail.

1. Discovery -- The nightly e2e job runs the B2CQA test "Swap ETH -> USDT (1inch)". It fails on the POST /api.1inch.dev step. The Allure report is uploaded to ledger-live.allure.green.ledgerlabs.net.
2. Triage of the failure -- QA Automation owner reviews the report, decides it is a product bug (not a test flake).
3. Bug creation in LIVE -- They file LIVE-29397 in the LIVE project with:
   - issuetype = Bug
   - labels = ["qaa"] (found by automation)
   - Description contains STR / ER / AR block and the Allure link.
   - Summary tag [SWAP][1inch] identifies the scope/integration.
4. Auto-assignment -- Slack integration and the triage bot (created_by_triage) pick up the ticket. PM assigns to the SWAP squad. State moves New -> Open -> In Progress.
5. Fix -- Dev (Nizar Sehli on this one) moves it to Code Review when the PR opens, then to QA when merged.
6. Verification -- QA engineer re-runs the failing Xray test on the nightly build that includes the fix. If the test now passes, they move the LIVE bug QA -> Done and add deploymentPROD to schedule inclusion in the next release.
7. Roll-up -- The Xray execution on B2CQA (e.g. the weekly SWAP non-reg Test Execution) shows the test as passed again, keeping the Test Plan green. No B2CQA ticket was ever created -- the defect lived entirely in LIVE.
8. Release -- After the build reaches the stores, release manager transitions Done -> Released.

Counter-example (test-side defect, no LIVE ticket):
B2CQA-4419 [SWAP][e2e] Search field of asset is not found -- test selector issue, fixed in the automation repo only, closed Abandoned once the spec was stabilised. No LIVE ticket opened.

---

## 8. Top Confluence URLs (Gaps)

Confluence search was denied by the sandbox in this session (searchConfluenceUsingCql, getConfluencePage). The following URLs are referenced from Jira tickets or well-known Ledger QA docs and should be opened manually to deepen the research:

1. https://ledgerhq.atlassian.net/wiki/spaces/QA/ -- Quality Assurance space root (contains the ticket workflow and Xray runbook).
2. https://ledgerhq.atlassian.net/wiki/spaces/TLL/ -- Ledger Live tech space -- deploymentPROD / hotfix policy.
3. https://ledgerhq.atlassian.net/wiki/spaces/CN/pages/6209306941/Quick+win+Ledger+Sync+onboarding -- referenced from LIVE-22041, product flow example.
4. https://ledgerhq.atlassian.net/wiki/spaces/CN/pages/6274187272/Quick+win+Ledger+Sync+post-onboarding+-+Functional+Spec -- functional spec + QA acceptance pattern.
5. https://ledger-live.allure.green.ledgerlabs.net/ -- Allure dashboard. Referenced from every automation-reported bug (LIVE-29465, LIVE-29397, B2CQA-4419, B2CQA-4467...). This is the de-facto "QA report" artefact linked from Jira.

---

## 9. Gaps

- Workflow schemes not directly readable. The getTransitionsForJiraIssue MCP tool was denied; workflow shape in section 2 is inferred from the set of statuses observed on live tickets, not from the definitive workflow XML. A follow-up research run with that permission enabled should confirm:
  - exact transition graph on LIVE (esp. On Hold <-> In Progress, and QA -> Released path);
  - presence/absence of a dedicated "Ready for Automation" status on B2CQA (some test plans suggest it but no sample was observed).
- Confluence denied. All five CQL probes failed. The QA onboarding guide cannot cite the canonical "Ticket Lifecycle" / "Xray runbook" page ids. Must be re-run with Confluence read scope.
- Issue-type metadata denied. Custom fields such as platform (iOS/Android/Desktop), fix version, Xray-specific fields, and the TmsLink custom field id are unknown -- only their symptoms are visible in descriptions.
- PTX naming. Onboarding briefs reference PTX but no PTX-* key exists on ledgerhq.atlassian.net. Need to confirm with the Recover/Apex team whether a private project exists or if PTX is purely a synonym for the Recover label set on LIVE.
- BACKLOG project. Same situation -- no Jira project matches. The onboarding guide should either drop the term or clarify it as the ingress lane on a given squad's board.
- Components. LIVE does not populate Jira components on any sample. The de-facto component field is the LLD / LLM / Wallet-v4 label, which is fragile (free text). Section 5 documents the convention but the onboarding guide should recommend formalising it.
- QA vs In QA status name. LIVE uses the bare name QA (id 10122, with a French description indicating it is managed internally by Jira Software), not In QA or Ready for QA. This should be pinned in the glossary so new joiners searching for Ready for QA do not miss tickets.

---

## 10. Raw Links

Jira -- projects:
- LIVE   -> https://ledgerhq.atlassian.net/browse/LIVE
- B2CQA  -> https://ledgerhq.atlassian.net/browse/B2CQA
- QAA    -> https://ledgerhq.atlassian.net/browse/QAA
- LBD    -> https://ledgerhq.atlassian.net/browse/LBD
- WO     -> https://ledgerhq.atlassian.net/browse/WO
- QUEST  -> https://ledgerhq.atlassian.net/browse/QUEST

Sample tickets used in this report:
- LIVE-28178 (Bug / In Progress / Mobile crash)  -- https://ledgerhq.atlassian.net/browse/LIVE-28178
- LIVE-29202 (Bug / Code Review / Wallet-v4,qaa) -- https://ledgerhq.atlassian.net/browse/LIVE-29202
- LIVE-29554 (Bug / New / qaa)                   -- https://ledgerhq.atlassian.net/browse/LIVE-29554
- LIVE-29465 (Bug / In Progress / Perps)         -- https://ledgerhq.atlassian.net/browse/LIVE-29465
- LIVE-29503 (Bug / In Progress / SWAP LLM)      -- https://ledgerhq.atlassian.net/browse/LIVE-29503
- LIVE-29545 (Bug / In Progress / Aleo)          -- https://ledgerhq.atlassian.net/browse/LIVE-29545
- LIVE-29535 (Bug / New / qaa)                   -- https://ledgerhq.atlassian.net/browse/LIVE-29535
- LIVE-29508 (Bug / New / qaa)                   -- https://ledgerhq.atlassian.net/browse/LIVE-29508
- LIVE-29397 (Bug / QA / qaa -- lifecycle example)           -- https://ledgerhq.atlassian.net/browse/LIVE-29397
- LIVE-28690 (Bug / Done / LLM,blocker,deploymentPROD,qaa)  -- https://ledgerhq.atlassian.net/browse/LIVE-28690
- LIVE-28693 (Bug / Done / LLM,blocker,deploymentPROD,qaa)  -- https://ledgerhq.atlassian.net/browse/LIVE-28693
- LIVE-21711 (Bug / Abandoned / LLM,Refined)     -- https://ledgerhq.atlassian.net/browse/LIVE-21711
- LIVE-21062 (Task / Abandoned / Apex,LLD,LLM,Recover) -- https://ledgerhq.atlassian.net/browse/LIVE-21062
- LIVE-22041 (Epic / On Hold / LLD)              -- https://ledgerhq.atlassian.net/browse/LIVE-22041
- LIVE-16602 (Bug / Released / LLD, Recover)     -- https://ledgerhq.atlassian.net/browse/LIVE-16602
- LIVE-25203 (Test / Cancelled)                  -- https://ledgerhq.atlassian.net/browse/LIVE-25203
- LIVE-27838 (Test / Cancelled, RFQ swap)        -- https://ledgerhq.atlassian.net/browse/LIVE-27838
- LIVE-21497 (Test / TO DO, Receive EVM tokens)  -- https://ledgerhq.atlassian.net/browse/LIVE-21497
- B2CQA-652  (parent manual test)                -- https://ledgerhq.atlassian.net/browse/B2CQA-652
- B2CQA-2697 (split Test / Automated -- Ethereum warning) -- https://ledgerhq.atlassian.net/browse/B2CQA-2697
- B2CQA-4611 (Test Plan -- Ledger Button non-reg) -- https://ledgerhq.atlassian.net/browse/B2CQA-4611
- B2CQA-4859 (Test / In Test Review)             -- https://ledgerhq.atlassian.net/browse/B2CQA-4859
- B2CQA-5561 (Test / In Test Review)             -- https://ledgerhq.atlassian.net/browse/B2CQA-5561
- B2CQA-5551 (Test Plan -- Wallet 4.0 Asset Aggregation) -- https://ledgerhq.atlassian.net/browse/B2CQA-5551
- B2CQA-5456 (Test Plan -- Wallet 4.0 My Wallet) -- https://ledgerhq.atlassian.net/browse/B2CQA-5456
- B2CQA-5464 (Test Plan -- Wallet 4.0 Tx History) -- https://ledgerhq.atlassian.net/browse/B2CQA-5464
- B2CQA-5359 (Test Plan -- V4 Tx History US2)    -- https://ledgerhq.atlassian.net/browse/B2CQA-5359
- B2CQA-5449 (Test Execution -- Nano Gen5 1.1.0-rc5 non-reg) -- https://ledgerhq.atlassian.net/browse/B2CQA-5449
- B2CQA-5445 (Test Execution -- Stax 1.10.0-rc5 non-reg)     -- https://ledgerhq.atlassian.net/browse/B2CQA-5445
- B2CQA-1938 (Test Execution -- LLD_0901 / legacy)           -- https://ledgerhq.atlassian.net/browse/B2CQA-1938
- B2CQA-4419 (Bug / Abandoned -- test flake, e2e)            -- https://ledgerhq.atlassian.net/browse/B2CQA-4419
- B2CQA-4467 (Bug / Done -- test selector)                   -- https://ledgerhq.atlassian.net/browse/B2CQA-4467
- B2CQA-4468 (Bug / Done -- test flake LWD)                  -- https://ledgerhq.atlassian.net/browse/B2CQA-4468
- B2CQA-3850 / 3851 / 3852 (Tests / Manual Test / LB)        -- https://ledgerhq.atlassian.net/browse/B2CQA-3850
- QAA-1129 (Task -- automates B2CQA-2697/2698/2699)          -- https://ledgerhq.atlassian.net/browse/QAA-1129
- QAA-613 / 615 / 617 / 703 (automation tasks for SWAP)      -- https://ledgerhq.atlassian.net/browse/QAA-613
- QAA-1168 (Bug / Abandoned -- mobile e2e nightly)           -- https://ledgerhq.atlassian.net/browse/QAA-1168
- QAA-1163 (Bug / TO DO -- Allure upload fails)              -- https://ledgerhq.atlassian.net/browse/QAA-1163
- QAA-1162 (Bug / New -- scheduled check-funds)              -- https://ledgerhq.atlassian.net/browse/QAA-1162

Reports & tooling:
- Allure -- https://ledger-live.allure.green.ledgerlabs.net/
- Datadog RUM -- https://app.datadoghq.eu/rum/sessions
