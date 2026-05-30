# Ram's Agentic Skills (`ram-agent-skills`)

A premium, curated collection of agentic coding and productivity skills designed to enable software engineering excellence through structured workflows, high feedback, and rigorous testing.

This repository blends the **Google Conductor Workflow** with **Matt Pocock's Agentic Skills**, providing coding assistants with the ultimate toolset to write clean, correct, and highly traceable code.

---

## 🌟 Heartfelt Attribution & Thanks

This repository is built on top of incredible, foundational work from the open-source engineering community. We owe deep gratitude and full attribution to:

1. **Matt Pocock (`@mattpocock`)**: Sincere thanks for his pioneering work on [mattpocock/skills](https://github.com/mattpocock/skills). His concepts of **Vertical Slice TDD**, disciplined **Diagnose loops**, and relentless **Grilling sessions** are the gold standard for agentic software engineering.
2. **Google Conductor Team**: Sincere thanks for the structured **Conductor Track & Workflow framework**. Conductor turns chaotic "vibe-coding" into a highly-managed, track-based progression of specifications, implementation plans, manual verification steps, and micro-commits.

---

## 🛠️ Included Skills

This library equips your AI agents with 15 specialized skills:

### 🚀 Google Conductor
- **`conductor`** — Structured development track management. Runs a robust cycle of specifications, implementation plans, manual verify steps, and checked-off checklists with associated git commit hashes.

### 🧪 Software Engineering
- **`tdd`** — **Test-Driven Development** with a strict Red-Green-Refactor loop. Encourages vertical slices (one behavioral test → one minimal green implementation → refactor) and warns against horizontal slice batching.
- **`diagnose`** — Disciplined bug diagnosis loop: Reproduce → Minimize → Hypothesize → Instrument → Fix → Regression-test.
- **`grill-with-docs`** — Relentless pre-coding alignment interview that updates `CONTEXT.md` and ADRs to establish a shared domain language.
- **`improve-codebase-architecture`** — Analyzes the workspace to find code-deepening opportunities, keeping public interfaces small and internal implementations deep.
- **`zoom-out`** — Commands the agent to step back and explain codebase structures or file groups within the context of the whole system.
- **`prototype`** — Quickly builds throwaway terminal prototypes or multi-variant UI designs to evaluate choices before committing to production.
- **`to-issues`** — Breaks high-level specs or plans into discrete, independently-grabbable GitHub/Linear issues.
- **`to-prd`** — Synthesizes the active conversation context into a comprehensive Product Requirements Document (PRD).
- **`triage`** — Triages backlog issues through a state machine of defined triage roles.

### ⚡ Productivity
- **`grill-me`** — Relentless interview skill for non-coding plans, grilling you until every decision branch is resolved.
- **`caveman`** — Ultra-compressed communication mode to reduce token usage by 75% without sacrificing technical accuracy.
- **`handoff`** — Compiles the current session into a high-density, concise handoff card for another agent to pick up.
- **`write-a-skill`** — Creates new, compliant agent skills with progressive disclosure and bundled resources.

---

## ⚙️ Installation

To install these skills into your workspace using the standard `skills.sh` installer, run the following command in your terminal:

```bash
npx skills@latest add ramki982/ram-agent-skills
```

Select the skills you want to enable, and your agent will immediately load them into its environment.

---

## 📄 License

This repository is licensed under the MIT License. See [LICENSE](LICENSE) for details.
All skills imported from `mattpocock/skills` retain their original MIT license and copyright by Matt Pocock.
All skills imported from Google's Conductor framework retain their original licenses and copyright by Google.
