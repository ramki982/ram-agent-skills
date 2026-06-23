# RFC: `human-interactions` — Declarative Human-in-the-Loop for Agent Skills

**Author:** Ramakrishnan Meenakshi Sundaram ([@ramki982](https://github.com/ramki982))  
**Status:** Draft  
**Created:** 2026-06-01  
**Revised:** 2026-06-23  
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

**Assertive defaults, flexible overrides.** The spec defines sensible defaults for all optional fields so skills work correctly out of the box. Skill authors override only when their workflow requires it. Tool implementors fill in runtime-bound fields (digests, tokens, timestamps) that skill authors cannot know at authoring time.

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

### `references/INTERACTIONS.md` — purpose and format

`references/INTERACTIONS.md` is a normative spec fragment — a YAML document defining when and how a skill collects structured human input. It is loaded by the tool on skill activation, not at startup. The file contains a single top-level array of interaction blocks. Each block declares a trigger, a set of fields, and optional behaviour on skip. Tools that do not recognise this file should degrade gracefully to freeform text collection using the `title`, `description`, and `fields[].label` values as guidance.

**Schema version:** `1.0-draft`

### Two distinct interaction classes

This spec recognises two architecturally distinct classes of human interaction:

**Input collection** — `before-start`, `on-phase`, `on-demand` — gather context, review, or guidance from the human. A human approving an input-collection checkpoint means "I have reviewed this and the workflow may continue." It does **not** authorize any specific subsequent action.

**Authorization** — `on-confirmation` — binds a human decision to one exact, identified action envelope before it executes. A phase approval must never implicitly cascade into action authorization. Even if a human approved a phase gate, the tool must still seek a fresh `confirmation_record` at the point of the irreversible action.

This boundary is normative. Tools must treat it as such.

### `references/INTERACTIONS.md` schema

Each interaction block has the following keys:

| Key | Required | Description |
|-----|----------|-------------|
| `id` | Yes | Unique identifier within the skill. Used in skill body instructions to reference collected values. |
| `trigger` | Yes | When this interaction fires. See [Trigger types](#trigger-types). |
| `title` | Yes | Short heading shown to the human. Max 80 chars. |
| `description` | No | Context or instructions for the human. Shown below the title. |
| `fields` | Yes | Array of input field definitions. At least one required. See [Field types](#field-types). |
| `phase` | Conditional | Required when `trigger` is `on-phase`. Skill-local string matching a phase name used in the skill body (e.g. `after-spec`, `before-implement`). Freeform — skill authors own this namespace. |
| `phase_kind` | No | Optional classifier for `on-phase` blocks. A small controlled vocabulary the tool uses to render appropriate UI, independent of the skill-local `phase` name. See [Phase kind values](#phase-kind-values). |
| `on-skip` | No | `abort` (default) or `continue`. Behaviour if the human dismisses or skips the interaction. |
| `resume` | No | `true` enables the tool to generate a resume contract for this block. See [Resume contract](#resume-contract). Recommended for `before-start` and `on-phase` blocks in long-running workflows. |
| `confirmation_record` | Conditional | Required when `trigger` is `on-confirmation`. Skill author declares intent fields; tool populates runtime-bound fields. See [Confirmation record](#confirmation-record). |

### Trigger types

| Trigger | Class | When it fires |
|---------|-------|---------------|
| `before-start` | Input collection | Once, before skill execution begins. Use for required setup context the agent cannot infer. |
| `on-demand` | Input collection | The agent decides when to invoke, guided by skill body instructions. The most flexible trigger — suited for inputs that depend on runtime conditions. |
| `on-phase` | Input collection | At a named phase boundary defined by the skill's workflow. The `phase` key specifies which boundary. Approval here does not authorize subsequent actions. |
| `on-confirmation` | Authorization | Before a destructive, irreversible, or high-stakes action. Produces a `confirmation_record` binding the human decision to the exact action envelope. |

> **Extensibility note:** This list is intentionally minimal. Future versions of the spec may introduce additional trigger types (e.g. `on-error`, `on-loop-iteration`) in a backward-compatible way. Tools encountering an unrecognised trigger value should treat the interaction as `on-demand`.

### Phase kind values

`phase_kind` is an optional classifier on `on-phase` blocks. It lets tools render appropriate UI without coupling to skill-local phase names. Skill authors set it; tools consume it.

| Value | Meaning |
|-------|---------|
| `review_gate` | Human reviews an artifact (spec, plan, report) before the workflow continues. |
| `pre_side_effect` | Workflow is about to produce an externally visible or durable change. Human confirms readiness. |
| `checkpoint` | General workflow pause for human awareness or lightweight acknowledgement. |
| `approval` | Explicit sign-off required before a significant (but not irreversible) next step. |

Tools encountering an unrecognised `phase_kind` should fall back to a generic review UI.

### Confirmation record

`on-confirmation` blocks must include a `confirmation_record` key. This produces a tamper-evident decision record binding the human's approval to one exact action envelope.

**Skill author declares (intent — known at authoring time):**

```yaml
confirmation_record:
  proposed_action: <string>      # What action will execute (e.g. git-commit, write-and-commit)
  target_resource: <string>      # What resource it targets (e.g. current-phase-files, product.md)
```

**Tool populates at runtime (binding — computed at execution time):**

| Field | Description |
|-------|-------------|
| `interaction_id` | Stable identifier for this confirmation event. |
| `skill_id` | Skill that requested the confirmation. |
| `skill_version` | Version of the skill at time of confirmation. |
| `phase` | Current phase when confirmation was requested, if applicable. |
| `canonical_arguments_digest` | Hash of the exact arguments the action will execute with. If arguments change after confirmation, the old record is invalid. |
| `decision` | `approve`, `deny`, `revise`, or `timeout`. |
| `decided_at` | Timestamp of the human decision. |
| `expires_at` | Default: 30 minutes after confirmation. Stale confirmations must not authorize new actions. |
| `decision_ref` | Opaque reference exposed to the skill runtime to bind the decision to the action. |

**Defaults:** Tools must apply `expires_at: +30m` unless the skill explicitly overrides. A `timeout` decision must be treated identically to `deny`.

### Resume contract

For `before-start` and `on-phase` blocks where `resume: true` is set, the tool generates a resume contract on completion. This allows long-running workflows to resume cleanly after human input without re-prompting or losing structured state.

**Tool populates at runtime:**

| Field | Description |
|-------|-------------|
| `interaction_id` | Identifies which interaction was completed. |
| `collected_values` | Structured, validated values from all fields in the block. |
| `resume_token` | Opaque runtime reference the tool can bind to the next execution step. |
| `expires_at` | Default: 24 hours. Expired resume contracts must not be used to skip re-collection. |

Skill authors set `resume: true` to opt in. The tool handles all runtime fields. No additional skill-side configuration is needed.

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

---

## Acceptance tests

These tests define the minimum conformance bar for tool implementors. A tool that passes all four correctly implements this spec.

**Test 1 — Bait-and-switch prevention** *(contributed by @rpelevin)*  
A `confirmation_record` approves action A with arguments digest A. If the skill subsequently changes the action, target, or digest before execution, the old confirmation must not authorize the new action. The tool must re-request confirmation.

**Test 2 — Structured denial on skip** *(contributed by @rpelevin)*  
A human skips or times out on an `on-confirmation` block with `on-skip: abort`. The skill receives a structured `deny`/`timeout` decision, not missing or freeform text. The skill must not proceed.

**Test 3 — Stale checkpoint invalidation** *(contributed by @piyushbag)*  
A human approves at phase N, but runtime state changes before the tool call executes. The tool must treat the prior confirmation as stale and re-request. The `expires_at` and `canonical_arguments_digest` fields are the mechanism.

**Test 4 — Partial field submission on abort** *(contributed by @piyushbag)*  
Required fields are missing when an `on-phase` block with `on-skip: abort` is submitted. The skill receives a structured `incomplete` result — not freeform text the skill must parse. The tool must not proceed.

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
# Conductor — Human Interaction Checkpoints
# Schema version: 1.0-draft
#
# Two interaction classes:
#   Input collection  — before-start, on-phase, on-demand
#   Authorization     — on-confirmation (produces confirmation_record)
#
# A phase approval does NOT authorize any subsequent action.

- id: project-setup
  trigger: before-start
  title: "Set up your conductor track"
  description: "A few details so conductor can scaffold your project correctly"
  on-skip: abort
  resume: true
  fields:
    - id: project-type
      type: single-select
      label: "What are you starting?"
      required: true
      options: [New project (greenfield), Existing project (brownfield)]
    - id: track-type
      type: single-select
      label: "What kind of work is this track?"
      required: true
      options: [Feature, Bug fix, Refactor, Spike, Chore]
    - id: tech-stack
      type: multi-select
      label: "Detected tech stack — confirm or adjust"
      options:
        source: file
        path: package.json
        extract: "list of frameworks and runtimes from dependencies and devDependencies"
        fallback: [React, Vue, Angular, Node.js, Python, Go, Java, Ruby, Other]
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
  phase_kind: review_gate
  title: "Review the spec before planning"
  description: "Conductor has drafted spec.md — review it and confirm before the implementation plan is built"
  on-skip: abort
  resume: true
  fields:
    - id: spec-approved
      type: confirm
      label: "Spec looks good — proceed to implementation plan?"
      required: true

- id: plan-review
  trigger: on-phase
  phase: after-plan
  phase_kind: review_gate
  title: "Review the implementation plan"
  description: "Conductor has drafted plan.md — review it and confirm before execution starts"
  on-skip: abort
  resume: true
  fields:
    - id: plan-approved
      type: confirm
      label: "Plan looks good — begin implementation?"
      required: true

- id: scope-clarification
  trigger: on-demand
  title: "Clarification needed"
  description: "The agent has reached a decision point that requires your input before proceeding"
  on-skip: continue
  fields:
    - id: affected-modules
      type: multi-select
      label: "Which modules does this change affect?"
      options:
        source: agent
        extract: "top-level directories in src/ relevant to the described goal"
        fallback: []
    - id: clarification-response
      type: textarea
      label: "Additional context or guidance"
      required: true
      placeholder: "Provide guidance so conductor can proceed..."

- id: phase-commit-confirm
  trigger: on-confirmation
  title: "Manual verification — ready to commit this phase?"
  description: "Run the app and verify this phase works as expected before committing"
  on-skip: abort
  confirmation_record:
    proposed_action: git-commit
    target_resource: current-phase-files
  fields:
    - id: verification-passed
      type: confirm
      label: "Manual verification passed. Commit this phase and continue?"
      required: true

- id: docs-sync-confirm
  trigger: on-confirmation
  title: "Approve documentation sync"
  description: "Conductor has drafted edits to product.md, tech-stack.md, and product-guidelines.md"
  on-skip: continue
  confirmation_record:
    proposed_action: write-and-commit
    target_resource: "product.md, tech-stack.md, product-guidelines.md"
  fields:
    - id: docs-approved
      type: confirm
      label: "Documentation changes look correct — write and commit?"
      required: true
```

---

## How tools should implement this

This RFC defines the schema. It does not mandate a specific UI. The following guidance is non-normative.

**Loading:** On skill activation, tools check for `human-interactions: true` in the frontmatter. If present, load `references/INTERACTIONS.md` before executing the skill body.

**`before-start` interactions** should be presented to the human immediately on activation, before the skill body is loaded into agent context, so the agent starts execution with all required values already available.

**Rendering:** Tools should render each interaction block as a native experience appropriate to their environment. Claude.ai might show a structured form panel. Claude Code might prompt sequentially in the terminal. Copilot might render a chat card. The human experience should feel native to the tool — not generic.

**`on-phase` interactions** should be triggered when the agent determines it has completed the named phase. The skill body instructions should make phase boundaries explicit. The `phase_kind` classifier, when present, should inform the tool's rendering choice independently of the `phase` string.

**`on-demand` interactions** are agent-initiated. The skill body should include instructions telling the agent when to invoke the interaction by its `id`.

**`on-confirmation` interactions** must block execution until the human explicitly approves. The tool must compute and bind the `confirmation_record` runtime fields before presenting the confirmation to the human. If arguments change after confirmation is granted, the tool must re-request. An `on-skip: abort` confirmation must halt the skill cleanly with a structured `deny` result — not silently continue.

**Resume contracts:** When `resume: true` is set on a block, the tool generates the resume contract on completion and makes `collected_values` and `resume_token` available to the runtime for the next step. Expired contracts (`expires_at` exceeded) must not be used to bypass re-collection.

**Graceful degradation:** If a tool does not support `human-interactions`, the agent should fall back to asking freeform questions in natural language, using the `title`, `description`, and `fields[].label` values as guidance.

**Collected values** should be made available to the agent by `interaction.id` and `field.id` — e.g. `project-setup.track-type` — so skill body instructions can reference them explicitly.

---

## What this does not cover (intentionally)

- **Rendering implementation details** — left to tools
- **Conditional field logic** (show field B only if field A is "yes") — deferred to a future version
- **Multi-step wizard flows within a single interaction** — deferred
- **Validation rules beyond `required`** — deferred
- **`script` dynamic option sources** — reserved pending sandboxed execution security resolution

These are left out to keep the initial proposal minimal and achievable. The goal is to solve 80% of cases cleanly, not to build a form framework.

---

## Alternatives considered

**Keep it in prose** — The status quo. Skill authors write "ask the user for X before proceeding." This works but produces inconsistent, unvalidated, tool-specific results. Does not scale as the ecosystem grows.

**Full schema in frontmatter** — Putting the entire `human-interactions` array directly in `SKILL.md` frontmatter is the most obvious approach. But with a user having 15 skills installed, all interaction definitions for all skills load at startup on every session — even if none of those skills are invoked. This is the context bloat problem seen in large MCP ecosystems. Rejected in favour of the pointer + references file approach.

**A fully-featured form DSL** — Conditional logic, multi-step wizards, validation expressions. More powerful, but significantly increases implementation burden for tool authors and skill authors alike. Starting minimal gives the ecosystem time to align before adding complexity.

**Single unified interaction class** — Treating `on-confirmation` as just another input-collection trigger. Rejected because authorization binding has fundamentally different runtime requirements: tamper-evidence, expiry, digest binding. Conflating the two classes creates a spec that is underspecified for the cases where it matters most.

---

## Open questions

1. ~~Should `on-phase` phase names be freeform strings or should the spec define a base vocabulary?~~ **Resolved:** Phase names are skill-local freeform strings. The optional `phase_kind` classifier provides the interoperability layer with a small controlled vocabulary. Credit: @rpelevin.

2. Should tools be required to expose collected values to the agent by structured reference (`project-setup.track-type`) or is it sufficient to inject them as natural language context? Structured references enable more precise skill body instructions.

3. Should `human-interactions` support a `version` key at the top level for future schema evolution?

---

## Reference implementation

A working example of this schema applied to the `conductor` skill is available in the [`ram-agent-skills`](https://github.com/ramki982/ram-agent-skills) repository on the [`human-interactions-rfc`](https://github.com/ramki982/ram-agent-skills/tree/human-interactions-rfc) branch. The `conductor` skill there has been updated to use `human-interactions` as a proof of concept.

---

## Acknowledgements

This proposal builds on the work of the [Agent Skills](https://agentskills.io) team at Anthropic, Matt Pocock's agentic skills patterns, and the Google Conductor workflow. Thanks to the growing community of skill authors whose real-world usage made this gap visible.

Community inputs that shaped this revision:
- **@rpelevin** — input-collection vs authorization distinction, `confirmation_record` sub-schema, `phase_kind` classifier, phase/confirmation boundary, acceptance tests 1 and 2
- **@piyushbag** — production workflow validation across hardware validation ops domain, resume contract, acceptance tests 3 and 4, offer to co-author reference INTERACTIONS.md
- **@OutlawAndy** — normative intro for INTERACTIONS.md
