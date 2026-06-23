# human-interaction-demo — Interaction Checkpoints
#
# This file is a fully annotated reference implementation of the
# human-interactions spec. Every feature is demonstrated with comments
# explaining why each choice was made.
#
# Loaded by the tool on skill activation. NOT loaded at startup.

# ─────────────────────────────────────────────
# TRIGGER: before-start
# Fires once, before the skill body runs.
# Use for context the agent needs from the very first step.
# ─────────────────────────────────────────────
- id: gather-context
  trigger: before-start
  title: "Tell me about your task"
  description: "This context shapes everything the skill does — take 30 seconds to fill it in"
  on-skip: abort           # can't proceed without this context
  resume: true             # tool generates resume contract so long-running workflows don't re-prompt

  fields:
    # FIELD TYPE: text — single-line free input
    - id: task-name
      type: text
      label: "Task name"
      required: true
      placeholder: "e.g. Migrate auth to OAuth2"
      hint: "A short name used to label outputs and commits"

    # FIELD TYPE: textarea — multi-line free input
    - id: scope
      type: textarea
      label: "Describe what you want to achieve"
      required: true
      placeholder: "What's the goal? What's out of scope?"

    # FIELD TYPE: single-select with STATIC options
    - id: urgency
      type: single-select
      label: "How urgent is this?"
      required: true
      options: [Critical — today, High — this week, Normal — no rush]
      default: Normal — no rush

    # FIELD TYPE: multi-select with DYNAMIC options (source: file)
    # The tool reads package.json; the agent extracts relevant frameworks.
    # If package.json doesn't exist, falls back to the static list.
    - id: existing-items
      type: multi-select
      label: "Relevant dependencies — confirm or adjust"
      hint: "Detected from package.json. Edit as needed."
      options:
        source: file
        path: package.json
        extract: "list of frameworks and libraries from dependencies and devDependencies"
        fallback: [React, Vue, Node.js, Express, Prisma, Other]

    # FIELD TYPE: ranked — drag-to-order priority list
    - id: preferences
      type: ranked
      label: "Rank what matters most for this task"
      options: [Code quality, Speed, Test coverage, Minimal changes, Documentation]


# ─────────────────────────────────────────────
# TRIGGER: on-phase
# Fires at a named phase boundary.
# The skill body must make the phase boundary explicit before invoking.
# `phase` is a free-form string — matches whatever the skill body uses.
# ─────────────────────────────────────────────
- id: phase-checkpoint
  trigger: on-phase
  phase: after-first-phase    # skill-local name — matches whatever the skill body uses
  phase_kind: review_gate     # tool-facing classifier — tells the tool how to render this gate
  title: "Phase complete — review before continuing"
  description: "The first phase has finished. Review the output before the next phase begins."
  on-skip: abort
  resume: true                # tool generates resume contract on completion
  # NOTE: approving this gate does NOT authorize any subsequent action.
  # If the next step is irreversible, an on-confirmation block must be invoked separately.

  fields:
    # FIELD TYPE: confirm — yes/no gate
    - id: proceed
      type: confirm
      label: "Output looks correct — proceed to the next phase?"
      required: true


# ─────────────────────────────────────────────
# TRIGGER: on-demand
# The agent invokes this when it decides it needs clarification.
# The skill body should tell the agent when to invoke it by id.
# on-skip: continue means the agent proceeds with its best guess if skipped.
# ─────────────────────────────────────────────
- id: mid-task-check
  trigger: on-demand
  title: "Clarification needed"
  description: "The agent has hit a decision point and needs your input"
  on-skip: continue           # non-blocking — agent can make a reasonable guess

  fields:
    # DYNAMIC options (source: agent)
    # No file path needed — the agent infers from codebase context it already has.
    # fallback ensures the field is always renderable even if inference fails.
    - id: affected-areas
      type: multi-select
      label: "Which areas of the codebase are affected?"
      hint: "Inferred from your codebase structure. Adjust if needed."
      options:
        source: agent
        extract: "top-level directories or modules relevant to the task described in gather-context.scope"
        fallback: []

    - id: clarification-response
      type: textarea
      label: "Additional context"
      required: false
      placeholder: "Anything else the agent should know before proceeding?"


# ─────────────────────────────────────────────
# TRIGGER: on-confirmation
# Blocks execution until the human explicitly approves.
# on-skip: abort means skipping halts the skill — never silently continues.
# Use before any write, delete, or commit operation.
# ─────────────────────────────────────────────
- id: final-confirm
  trigger: on-confirmation
  title: "Ready to apply changes?"
  description: "This will write changes to your codebase. Review the proposed diff before confirming."
  on-skip: abort              # destructive action — must never proceed without approval
  confirmation_record:
    proposed_action: write-changes   # skill author declares intent
    target_resource: codebase        # tool binds digest, expiry, decision at runtime
  # Tool populates at runtime: interaction_id, canonical_arguments_digest,
  # decision, decided_at, expires_at (+30m default), decision_ref.
  # If arguments change after confirmation, the old record is invalid — tool must re-request.

  fields:
    - id: confirmed
      type: confirm
      label: "Changes look correct — apply them now?"
      required: true
