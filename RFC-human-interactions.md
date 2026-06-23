# RFC: `human-interactions` — Declarative Human-in-the-Loop for Agent Skills

**Author:** Ramakrishnan Meenakshi Sundaram ([@ramki982](https://github.com/ramki982))  
**Status:** Draft  
**Created:** 2026-06-01  
**Repo:** [agentskills/agentskills](https://github.com/agentskills/agentskills)

---

## Summary

Add an optional `human-interactions: true` signal to the `SKILL.md` frontmatter, paired with a `references/INTERACTIONS.md` file that declaratively specifies when and how an agent should gather structured input from a human during skill execution. The frontmatter signal costs 2 tokens at startup. The full schema loads only when the skill is activated — consistent with the spec's existing progressive disclosure model.

---

## Motivation

The Agent Skills spec today is excellent at packaging *agent* knowledge — instructions, scripts, references. But skills that require human input (approval, context, decisions) have no standard way to declare those needs. Every skill author improvises, and every tool that implements the spec re-invents how to gather human input from scratch.

This shows up clearly in workflow-heavy skills like `conductor`, `grill-me`, and `to-prd` — skills that are fundamentally *collaborative*: they need human context to start, human approval at phase boundaries, and human sign-off before destructive actions. Today, a skill can only express these needs as prose instructions like:

> *"Ask the user for the project name and tech stack before proceeding."*

The agent then improvises a question, the tool renders it however it likes, and the human answers in freeform text. The result is:

- **Inconsistent UX** across tools — the same skill feels different in Claude Code vs Cursor vs Copilot
- **No structured validation** — required fields can be skipped, inputs arrive in unpredictable shapes
- **Lost intent** — the skill author can't express *how* to ask (a confirmation dialog vs a multi-select vs a text field), only *that* something should be asked
- **Friction for tool implementors** — every tool has to guess at the interaction model from prose

### The gap in one sentence

> A skill today cannot communicate its UI needs for gathering human input in a generic, tool-agnostic way.

---

## Proposal

Add an optional `human-interactions: true` key to the `SKILL.md` frontmatter as a lightweight signal. The full interaction schema lives in `references/INTERACTIONS.md` and is loaded by the tool only when the skill is activated.

```
my-skill/
├── SKILL.md                  ← human-interactions: true  (2 tokens at startup)
├── references/
│   └── INTERACTIONS.md       ← full schema, loaded on activation
└── ...
```

This design directly addresses the context bloat problem seen in large tool ecosystems. A user with 15 skills installed pays exactly 2 extra tokens per skill at startup — regardless of how many interaction blocks that skill defines. The full schema for any given skill only enters context when that skill is actually being used.

### Design principles

**Declarative, not prescriptive.** The skill declares *what* to collect and *when*. The tool decides *how* to render it. A rich form widget in Claude.ai, numbered prompts in a terminal, a chat card in Copilot — same schema, different presentation. Skill authors don't write UI; they write intent.

**Backward-compatible.** The field is fully optional. All existing skills continue to work unchanged. Tools that don't recognise `human-interactions` ignore it gracefully and fall back to freeform text questions as today.

**Progressive disclosure by design.** The `human-interactions: true` signal in frontmatter loads at startup alongside `name` and `description` — 2 tokens, not 200. The full `references/INTERACTIONS.md` schema loads only on skill activation, consistent with how the spec already handles `scripts/` and `references/`. Tools can pre-render a `before-start` form as the very first thing on activation, before the skill body runs.

**Extensible.** The trigger types and field types defined here cover the most common 80% of cases. Additional trigger types and field types can be added in future versions of the spec in a fully backward-compatible way — tools that encounter an unrecognised `trigger` or field `type` should fall back to freeform text collection for that block.

---

## Specification

### `SKILL.md` frontmatter

A skill that supports human interactions adds exactly one key to its frontmatter:

```yaml
human-interactions: true
```

No other frontmatter changes are required. This signal tells tools to load `references/INTERACTIONS.md` on skill activation.

### `references/INTERACTIONS.md` schema

This file contains a YAML document — an array of **interaction blocks**. Each block has the following keys:

| Key | Required | Description |
|-----|----------|-------------|
| `id` | Yes | Unique identifier within the skill. Used in skill body instructions to reference collected values. |
| `trigger` | Yes | When this interaction fires. See [Trigger types](#trigger-types). |
| `title` | Yes | Short heading shown to the human. Max 80 chars. |
| `description` | No | Context or instructions for the human. Shown below the title. |
| `fields` | Yes | Array of input field definitions. At least one required. See [Field types](#field-types). |
| `phase` | Conditional | Required when `trigger` is `on-phase`. Free-form string matching a phase name used in the skill body (e.g. `after-spec`, `before-implement`). Freeform to accommodate the diversity of phase structures across different skills. |
| `on-skip` | No | `abort` (default) or `continue`. Behaviour if the human dismisses or skips the interaction. |

### Trigger types

| Trigger | When it fires |
|---------|---------------|
| `before-start` | Once, before skill execution begins. Use for required setup context the agent cannot infer. |
| `on-demand` | The agent decides when to invoke, guided by skill body instructions. The most flexible trigger — suited for inputs that depend on runtime conditions. |
| `on-confirmation` | Before a destructive, irreversible, or high-stakes action. Requires explicit human approval to proceed. |
| `on-phase` | At a named phase boundary defined by the skill's workflow. The `phase` key specifies which boundary. |

> **Extensibility note:** This list is intentionally minimal. Future versions of the spec may introduce additional trigger types (e.g. `on-error`, `on-loop-iteration`) in a backward-compatible way. Tools encountering an unrecognised trigger value should treat the interaction as `on-demand`.

### Field types

Each entry in `fields` has the following keys:

| Key | Required | Description |
|-----|----------|-------------|
| `id` | Yes | Unique identifier within the interaction block. |
| `type` | Yes | Input type. See field type table below. |
| `label` | Yes | Human-readable label for the field. |
| `required` | No | Boolean. Default `false`. |
| `options` | Conditional | Required for `single-select`, `multi-select`, and `ranked`. Either a static array of strings or a dynamic source definition. See [Dynamic options](#dynamic-options). |
| `default` | No | Default value. Type must match the field type. |
| `placeholder` | No | Placeholder text for `text` and `textarea` fields. |
| `hint` | No | Additional guidance shown below the field. |

**Supported field types:**

| Type | Description |
|------|-------------|
| `text` | Single-line free text. |
| `textarea` | Multi-line free text. Suited for descriptions, goals, requirements. |
| `single-select` | Choose exactly one from `options`. |
| `multi-select` | Choose one or more from `options`. |
| `confirm` | Boolean yes/no. The natural pairing for `on-confirmation` triggers. |
| `ranked` | Rank a list of `options` by priority. Captures ordered preference. |

> **Extensibility note:** Additional field types may be added in future versions in a backward-compatible way. Tools encountering an unrecognised `type` should render it as a `text` field.

### Dynamic options

For `single-select`, `multi-select`, and `ranked` fields, `options` can be a **dynamic source definition** instead of a static array. This lets the skill build choices from the user's actual project context rather than hardcoded guesses.

```yaml
options:
  source: file | agent
  path: <relative path>        # required for source: file
  extract: <hint string>       # describes what to parse out
  fallback: [...]              # static array used if source resolution fails
```

**Supported source types:**

| Source | Description |
|--------|-------------|
| `file` | Read a file already present in the repository. The tool reads the file; the agent uses the `extract` hint to parse out the relevant list of options. No code execution — consistent with how skills already read reference files. |
| `agent` | The agent infers options from context it already has in scope (open files, prior conversation, codebase structure). No file path needed — just an `extract` hint describing what to look for. |

> **`script` sources** — running a bundled script to generate options dynamically — are reserved for a future version of this spec, pending resolution of sandboxed execution security requirements. Skill authors who need script-generated options should use `source: agent` with an `extract` hint as an interim approach.

**`fallback` is strongly recommended** for all dynamic sources. If source resolution fails (file not found, agent cannot infer), the tool falls back to the static `fallback` array so the interaction can still proceed.

**Examples:**

```yaml
# Detect tech stack from project files
- id: tech-stack
  type: multi-select
  label: "Detected tech stack — confirm or adjust"
  options:
    source: file
    path: package.json
    extract: "list of frameworks and runtimes from dependencies and devDependencies"
    fallback: [React, Vue, Node.js, Python, Go, Java, Other]

# List pending tracks from the conductor registry
- id: resume-track
  type: single-select
  label: "Which track do you want to resume?"
  options:
    source: file
    path: conductor/tracks.md
    extract: "track names with status pending or in-progress"
    fallback: []

# Agent infers affected modules from codebase structure
- id: affected-modules
  type: multi-select
  label: "Which modules does this change affect?"
  options:
    source: agent
    extract: "top-level directories in src/ relevant to the described goal"
    fallback: []
```

---

## Example

The following shows the `conductor` skill updated with this proposal. `SKILL.md` carries only the 2-token signal. All interaction definitions live in `references/INTERACTIONS.md`.

### `conductor/SKILL.md` (frontmatter only shown)

```yaml
---
name: conductor
description: >
  Structured development track management. Runs a robust cycle of
  specifications, implementation plans, manual verify steps, and
  checked-off checklists with associated git commit hashes.
  Use when starting a new feature, bug fix, refactor, or spike.
human-interactions: true
---
```

### `conductor/references/INTERACTIONS.md`

```yaml
# Human interaction checkpoints for the conductor skill.
# Loaded by the tool on skill activation. Not loaded at startup.

- id: project-setup
  trigger: before-start
  title: "Set up your conductor track"
  description: "A few details before conductor scaffolds your track"
  fields:
    - id: track-type
      type: single-select
      label: "What are you building?"
      required: true
      options: [Feature, Bug fix, Refactor, Spike]
    - id: tech-stack
      type: multi-select
      label: "Tech stack"
      options: [React, Vue, Node.js, Python, Go, Java, Other]
    - id: scope-notes
      type: textarea
      label: "Describe the goal"
      required: true
      placeholder: "What problem does this solve? What's in scope?"
    - id: workflow-priorities
      type: ranked
      label: "Rank your priorities for this track"
      options: [Correctness, Speed of delivery, Test coverage, Code readability]

- id: spec-review
  trigger: on-phase
  phase: after-spec
  title: "Review the spec before planning"
  description: "Conductor has generated spec.md — review it and confirm before the plan is built"
  on-skip: abort
  fields:
    - id: spec-approved
      type: confirm
      label: "Spec looks good — proceed to implementation plan?"
      required: true

- id: scope-clarification
  trigger: on-demand
  title: "Scope clarification needed"
  description: "The agent has reached a decision point that requires your input"
  on-skip: continue
  fields:
    - id: clarification-response
      type: textarea
      label: "Your answer"
      required: true

- id: commit-confirm
  trigger: on-confirmation
  title: "Manual verification — ready to commit?"
  description: "Run the app and verify this phase works as expected before committing"
  on-skip: abort
  fields:
    - id: verification-passed
      type: confirm
      label: "Manual verification passed. Commit this phase and continue?"
      required: true
```

---

## How tools should implement this

This RFC defines the schema. It does not mandate a specific UI. The following guidance is non-normative.

**Loading:** On skill activation, tools check for `human-interactions: true` in the frontmatter. If present, load `references/INTERACTIONS.md` before executing the skill body. This file is not loaded at startup — only on activation.

**`before-start` interactions** should be presented to the human immediately on activation, before the skill body is loaded into agent context, so the agent starts execution with all required values already available.

**Rendering:** Tools should render each interaction block as a native experience appropriate to their environment. Claude.ai might show a structured form panel. Claude Code might prompt sequentially in the terminal. Copilot might render a chat card. The human experience should feel native to the tool — not generic.

**`on-phase` interactions** should be triggered when the agent determines it has completed the named phase. The skill body instructions should make phase boundaries explicit.

**`on-demand` interactions** are agent-initiated. The skill body should include instructions telling the agent when to invoke the interaction by its `id`.

**`on-confirmation` interactions** must block execution until the human explicitly approves. An `on-skip: abort` confirmation should halt the skill cleanly, not silently continue.

**Graceful degradation:** If a tool does not support `human-interactions`, the agent should fall back to asking freeform questions in natural language, using the `title`, `description`, and `fields[].label` values as guidance.

**Collected values** should be made available to the agent by `interaction.id` and `field.id` — e.g. `project-setup.track-type` — so skill body instructions can reference them explicitly.

---

## What this does not cover (intentionally)

- **Rendering implementation details** — left to tools
- **Conditional field logic** (show field B only if field A is "yes") — deferred to a future version
- **Multi-step wizard flows within a single interaction** — deferred
- **Validation rules beyond `required`** — deferred

These are left out to keep the initial proposal minimal and achievable. The goal is to solve 80% of cases cleanly, not to build a form framework.

---

## Alternatives considered

**Keep it in prose** — The status quo. Skill authors write "ask the user for X before proceeding." This works but produces inconsistent, unvalidated, tool-specific results. Does not scale as the ecosystem grows.

**Full schema in frontmatter** — Putting the entire `human-interactions` array directly in `SKILL.md` frontmatter is the most obvious approach. But with a user having 15 skills installed, all interaction definitions for all skills load at startup on every session — even if none of those skills are invoked. This is the context bloat problem seen in large MCP ecosystems. Rejected in favour of the pointer + references file approach.

**A fully-featured form DSL** — Conditional logic, multi-step wizards, validation expressions. More powerful, but significantly increases implementation burden for tool authors and skill authors alike. Starting minimal gives the ecosystem time to align before adding complexity.

---

## Open questions

1. Should `on-phase` phase names be freeform strings (current proposal) or should the spec define a base vocabulary (e.g. `after-spec`, `after-plan`, `after-implement`) that skills can extend? Freeform is more flexible; a base vocabulary improves interoperability.

2. Should tools be required to expose collected values to the agent by structured reference (`project-setup.track-type`) or is it sufficient to inject them as natural language context? Structured references enable more precise skill body instructions.

3. Should `human-interactions` support a `version` key at the top level for future schema evolution?

---

## Reference implementation

A working example of this schema applied to the `conductor` skill is available in the [`ram-agent-skills`](https://github.com/ramki982/ram-agent-skills) repository. The `conductor` skill there has been updated to use `human-interactions` as a proof of concept.

---

## Acknowledgements

This proposal builds on the work of the [Agent Skills](https://agentskills.io) team at Anthropic, Matt Pocock's agentic skills patterns, and the Google Conductor workflow. Thanks to the growing community of skill authors whose real-world usage made this gap visible.
