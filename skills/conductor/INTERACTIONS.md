# Conductor — Human Interaction Checkpoints
#
# Loaded by the tool on skill activation. NOT loaded at startup.
# Defines all structured human-in-the-loop checkpoints for the conductor skill.
# See references in conductor/SKILL.md for where each interaction is invoked.

# 1. Gather project context before setup begins
- id: project-setup
  trigger: before-start
  title: "Set up your conductor track"
  description: "A few details so conductor can scaffold your project correctly"
  on-skip: abort
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

# 2. Human reviews generated spec before planning begins
- id: spec-review
  trigger: on-phase
  phase: after-spec
  title: "Review the spec before planning"
  description: "Conductor has drafted spec.md — review it and confirm before the implementation plan is built"
  on-skip: abort
  fields:
    - id: spec-approved
      type: confirm
      label: "Spec looks good — proceed to implementation plan?"
      required: true

# 3. Human reviews generated plan before implementation begins
- id: plan-review
  trigger: on-phase
  phase: after-plan
  title: "Review the implementation plan"
  description: "Conductor has drafted plan.md — review it and confirm before execution starts"
  on-skip: abort
  fields:
    - id: plan-approved
      type: confirm
      label: "Plan looks good — begin implementation?"
      required: true

# 4. Agent-initiated: ask for clarification when scope is ambiguous mid-execution
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

# 5. Human manually verifies and signs off before each phase commit
- id: phase-commit-confirm
  trigger: on-confirmation
  title: "Manual verification — ready to commit this phase?"
  description: "Run the app and verify this phase works as expected before committing"
  on-skip: abort
  fields:
    - id: verification-passed
      type: confirm
      label: "Manual verification passed. Commit this phase and continue?"
      required: true

# 6. Human approves documentation sync changes before they are written
- id: docs-sync-confirm
  trigger: on-confirmation
  title: "Approve documentation sync"
  description: "Conductor has drafted edits to product.md, tech-stack.md, and product-guidelines.md"
  on-skip: continue
  fields:
    - id: docs-approved
      type: confirm
      label: "Documentation changes look correct — write and commit?"
      required: true
