---
name: human-interaction-demo
description: >
  A reference skill demonstrating the human-interactions spec. Shows all
  trigger types (before-start, on-phase, on-demand, on-confirmation), all
  field types (text, textarea, single-select, multi-select, confirm, ranked),
  and both static and dynamic options (source: file, source: agent).
  Use this skill to understand how to add human-interactions to your own skills.
compatibility: Designed for tools that support the human-interactions spec.
metadata:
  author: ramki982
  version: "1.0"
  purpose: reference implementation
human-interactions: true
---

# Human Interaction Demo Skill

This skill is a **living reference** for the `human-interactions` spec. It does not perform real work — it exists to show skill authors exactly how every feature of the spec is used in practice.

Read `references/INTERACTIONS.md` alongside this file to see how the schema maps to the execution points below.

---

## What this demonstrates

| Feature | Where shown |
|---------|-------------|
| `trigger: before-start` | Step 1 — collected before anything runs |
| `trigger: on-phase` | Step 2 — fires at a named phase boundary |
| `trigger: on-demand` | Step 3 — agent decides when to invoke |
| `trigger: on-confirmation` | Step 4 — blocks on explicit human approval |
| Static `options` array | `preferences` field in `gather-context` |
| `options.source: file` | `existing-items` field in `gather-context` |
| `options.source: agent` | `affected-areas` field in `mid-task-check` |
| `on-skip: abort` | `final-confirm` — halts if skipped |
| `on-skip: continue` | `mid-task-check` — proceeds if skipped |
| `fallback` on dynamic options | All dynamic fields include a fallback |

---

## Execution flow

### Step 1 — Before anything runs
**Invoke interaction `gather-context`** (`trigger: before-start`)

The tool renders this form before loading the skill body into agent context. All collected values are available to the agent via `gather-context.<field-id>` throughout execution.

### Step 2 — After the first phase completes
**Invoke interaction `phase-checkpoint`** (`trigger: on-phase`, `phase: after-first-phase`)

Fires when the agent determines it has completed the first phase. The agent should make the phase boundary explicit in its output before invoking this interaction.

Use `phase-checkpoint.proceed` to decide whether to continue or stop.

### Step 3 — When the agent needs clarification
**Invoke interaction `mid-task-check`** (`trigger: on-demand`)

The agent invokes this when it reaches an ambiguous decision point during execution. Reference `gather-context.scope` and `gather-context.existing-items` to frame the clarification question.

Because `on-skip: continue`, the agent proceeds with its best guess if the human skips.

### Step 4 — Before any irreversible action
**Invoke interaction `final-confirm`** (`trigger: on-confirmation`)

Always invoke this before writing, deleting, or committing anything. Because `on-skip: abort`, skipping halts execution cleanly — never silently continues.

---

## How to adapt this for your own skill

1. Copy `references/INTERACTIONS.md` into your skill's `references/` directory
2. Remove interaction blocks you don't need
3. Replace field `label`, `options`, and `extract` values with ones relevant to your skill's domain
4. Add `human-interactions: true` to your `SKILL.md` frontmatter
5. Add invocation notes in your skill body at the points where each interaction should fire

That's it. The tool handles rendering. You handle intent.
