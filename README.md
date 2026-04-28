<div style="text-align: center; margin: 40px 0 20px 0;">
  <img src="images/Ledger-logo-long.svg" alt="Ledger" style="height: 48px;" />
</div>

<div style="text-align: center; margin-bottom: 8px;">
  <h1 style="font-size: 2em; font-weight: 700; color: #1e293b; margin: 0;">QA Automation Engineer</h1>
  <h2 style="font-size: 1.3em; font-weight: 400; color: #64748b; margin: 4px 0 0 0;">Onboarding Guide — Ledger Live</h2>
</div>

<div style="display: flex; justify-content: center; gap: 12px; margin: 24px 0; flex-wrap: wrap;">
  <span style="background: #f1f5f9; color: #475569; padding: 6px 14px; border-radius: 99px; font-size: 0.85em; font-weight: 500;">55 Chapters</span>
  <span style="background: #f1f5f9; color: #475569; padding: 6px 14px; border-radius: 99px; font-size: 0.85em; font-weight: 500;">63 Interactive Quizzes</span>
  <span style="background: #f1f5f9; color: #475569; padding: 6px 14px; border-radius: 99px; font-size: 0.85em; font-weight: 500;">7 Appendices</span>
</div>

---

<div style="background: linear-gradient(135deg, #f8fafc 0%, #f1f5f9 100%); border-radius: 16px; padding: 32px; margin: 32px 0; border: 1px solid #e2e8f0;">

**Welcome to the Ledger Live QA team.**

This guide is your single reference for everything you need to become an effective QA Automation Engineer on Ledger Live. It starts with the Ledger ecosystem itself — products, hardware, teams, release cycle, ticket flow — then walks you through the full stack: monorepo, shared tooling, desktop E2E, mobile E2E, CLI automation, the Swap Live App, and the mastery layer covering test strategy, debugging, CI/CD, and AI-assisted automation.

Every chapter builds on the previous one. Each ends with an interactive quiz (80% to pass). Part 0 grounds you in the company; Parts 1-2 give you the common vocabulary; Parts 3-5 are your desktop, mobile, and CLI deep dives; Part 6 covers the Swap Live App; Part 7 closes with the contributor mindset.

</div>

<div style="display: grid; grid-template-columns: 1fr 1fr; gap: 16px; margin: 32px 0;">

<div style="background: #ffffff; border: 1px solid #e2e8f0; border-radius: 12px; padding: 20px; border-top: 3px solid #6366f1;">
  <strong style="color: #6366f1;">Audience</strong><br>
  <span style="color: #475569; font-size: 0.92em;">New QA Automation Engineer interns joining the Ledger Live team.</span>
</div>

<div style="background: #ffffff; border: 1px solid #e2e8f0; border-radius: 12px; padding: 20px; border-top: 3px solid #8b5cf6;">
  <strong style="color: #8b5cf6;">Structure</strong><br>
  <span style="color: #475569; font-size: 0.92em;">Eight progressive parts (0 through 7) plus six appendices, ordered so you never meet a concept before you need it.</span>
</div>

<div style="background: #ffffff; border: 1px solid #e2e8f0; border-radius: 12px; padding: 20px; border-top: 3px solid #22c55e;">
  <strong style="color: #22c55e;">Quizzes</strong><br>
  <span style="color: #475569; font-size: 0.92em;">Each chapter ends with an interactive quiz. Each part ends with a final assessment. 80% to pass.</span>
</div>

<div style="background: #ffffff; border: 1px solid #e2e8f0; border-radius: 12px; padding: 20px; border-top: 3px solid #eab308;">
  <strong style="color: #eab308;">Navigation</strong><br>
  <span style="color: #475569; font-size: 0.92em;">Use the sidebar to jump between chapters. Use the search bar to find any topic instantly.</span>
</div>

</div>

---

## Reading Order

| | Part | Chapters | What you'll learn |
|---|------|----------|-------------------|
| **0** | [Welcome to Ledger](part0.md) | 0.1-0.6 | Ledger ecosystem, hardware & BOLOS, teams & ownership, release cycle & environments, ticket flow (Jira/Xray), security posture |
| **1** | [Foundations](part1.md) | 1.1-1.5 | Ledger Live product, tech stack, E2E architecture, monorepo layout, dev environment setup |
| **2** | [Shared Tooling](part2.md) | 2.1-2.5 | Git + changesets, pnpm/Turbo, Speculos device emulation, Firebase & feature flags, Allure & Xray |
| **3** | [Desktop E2E](part3.md) | 3.1-3.10 | Desktop architecture + first test, Playwright zero-to-advanced, Electron, codebase catalog, daily workflow, QAA-1139 & QAA-1141 walkthroughs, exercises |
| **4** | [Mobile E2E](part4.md) | 4.1-4.11 | Mobile architecture + first test, React Native primer, mobile toolchain, Detox zero-to-advanced, bridge + POMs, daily workflow, QAA-702 walkthrough, exercises |
| **5** | [CLI Automation](part5.md) | 5.1-5.9 | apps/cli (the canonical QAA CLI), ERC-20 approvals + revokes blockchain primer, manual revoke tools, the five-layer integration, Speculos lifecycle, broadcast discipline, daily CLI workflow, QAA-615 walkthrough |
| **6** | [Swap Live App](part6.md) | 6.1-6.4 | Live Apps in Ledger Wallet, Swap architecture, releases & Firebase environments, QAA-1136 walkthrough |
| **7** | [Mastery & Contributing](part7.md) | 7.1-7.4 | Test strategy & best practices, cross-platform debugging, CI/CD & test sharding, AI agents & automation |
| **A** | [Appendix](appendix.md) | A-G | Commands, env vars, Ledger Live stack, devices, troubleshooting FAQ, git cheat sheet, glossary |

---

<div style="text-align: center; color: #94a3b8; font-size: 0.9em; margin-top: 48px; padding: 24px 0; border-top: 1px solid #e2e8f0;">
  <em>Ship quality. Break nothing. Test everything.</em>
</div>
