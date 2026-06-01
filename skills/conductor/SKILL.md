---
name: conductor
description: Orchestrate context-driven software development by managing specs, plans, and implementations using the Conductor methodology.
version: 2.0.0
human-interactions: true
---

# Conductor: Context-Driven Development Protocol

Conductor is a systematic context-driven methodology that ensures all development tasks follow a rigorous, high-quality lifecycle: **Context -> Specification & Plan -> Implementation -> Documentation Sync**.

Instead of writing code directly, this skill guides you through formalizing requirements, structuring interactive checkpoints, managing a persistent tracks registry, and executing task lists step-by-step.

> **Human-in-the-loop note:** This skill declares structured human interaction checkpoints in `references/INTERACTIONS.md`. On activation, load that file first. Invoke interactions by their `id` at the points described in each protocol below.

---

## 1. Directory & State Structure

A project managed by Conductor maintains its source of truth under a `conductor/` directory in the repository root:

```text
conductor/
├── index.md                  # Context index referencing core guides and tracks
├── product.md                # Product context, goals, users, and high-level features
├── product-guidelines.md     # Brand, voice, style guidelines, and UX standards
├── tech-stack.md             # Selected languages, frameworks, databases, and APIs
├── workflow.md               # Custom development and testing workflows
├── tracks.md                 # Project Tracks Registry (master list of all tracks)
├── code_styleguides/         # Subdirectory with language/framework style guides
│   ├── python.md
│   └── javascript.md
└── tracks/                   # Subdirectory for individual tracks
    └── track_id_YYYYMMDD/
        ├── index.md          # Track index
        ├── spec.md           # Track specification (requirements & scope)
        ├── plan.md           # Phased task checklist
        └── metadata.json     # Track state and configuration
```

---

## 2. Core Operational Protocols

### Protocol 2.1: Project Setup (`/conductor:setup`)
Use this protocol when initializing a new or existing project with Conductor:

1. **Invoke interaction `project-setup`** (`trigger: before-start`) — collect project type, track type, tech stack, goal, and workflow priorities before any other step.
2. **Audit Existing State**: Check if any Conductor files already exist. Report status and offer to resume or overwrite.
3. **Project Inception**:
   - *Brownfield (Existing)*: Scan project configuration files (`package.json`, `requirements.txt`, `pyproject.toml`, etc.) to confirm the inferred tech stack against `project-setup.tech-stack`.
   - *Greenfield (New)*: Initialize git repository (if missing). Use `project-setup.scope-notes` as the starting point for what to build.
4. **Interactive Documentation Drafting**:
   - **Product (`product.md`)**: Pre-populate from `project-setup.scope-notes`. Gather target users, high-level features, and goals.
   - **Product Guidelines (`product-guidelines.md`)**: Standardize branding, UX, and prose preferences.
   - **Tech Stack (`tech-stack.md`)**: Confirm against `project-setup.tech-stack`. List and agree upon languages, frameworks, databases, and cloud providers.
   - **Workflow (`workflow.md`)**: Customize testing rules and commit frequency. Reference `project-setup.workflow-priorities` to set defaults.
   - **Style Guides (`code_styleguides/`)**: Import appropriate style guides based on the tech stack.
5. **Finalize Scaffolding**: Create `conductor/index.md` as the main navigation map, and write the initial master `tracks.md` file.

---

### Protocol 2.2: New Track Creation (`/conductor:newTrack`)
Use this protocol when launching a new feature, bug fix, chore, or refactoring task:

1. **Prerequisites Check**: Ensure `product.md`, `tech-stack.md`, and `workflow.md` exist and are fully populated.
2. **Track Identification**: Define a short unique identifier for the track (e.g., `feature_login_20260530`) based on `project-setup.track-type` and a description.
3. **Draft the Specification (`spec.md`)**:
   - Interactively ask clarifying questions (up to 4, batched together). If scope is unclear at any point, **invoke interaction `scope-clarification`** (`trigger: on-demand`).
   - Draft sections: Overview, Functional Requirements, Non-Functional Requirements, Acceptance Criteria, and Out of Scope.
   - **Invoke interaction `spec-review`** (`trigger: on-phase`, `phase: after-spec`) — present the draft and wait for approval before proceeding.
4. **Generate the Implementation Plan (`plan.md`)**:
   - Translate the approved `spec.md` into an actionable, phased to-do list.
   - Ensure plan structure matches `workflow.md` and `project-setup.workflow-priorities`.
   - Append a final verification checkpoint at the end of each phase:
     `- [ ] Task: Conductor - User Manual Verification '<Phase Name>'`
   - **Invoke interaction `plan-review`** (`trigger: on-phase`, `phase: after-plan`) — present the plan and wait for approval before proceeding.
5. **Update Registries & Commit**:
   - Create the track folder, write `metadata.json`, `spec.md`, `plan.md`, and `index.md`.
   - Append the track to the master `conductor/tracks.md` registry.
   - Commit changes with message: `chore(conductor): Add new track '<track_description>'`.

---

### Protocol 2.3: Task Implementation (`/conductor:implement`)
Use this protocol when implementing tasks from the current active track:

1. **Track Selection**: Load the tracks registry. Find the first incomplete track or prompt the user to choose.
2. **Update Status**: Set track status to in-progress (`[~]`) in the `tracks.md` registry.
3. **Loop through Tasks in `plan.md`**:
   - Target the first incomplete task.
   - Execute the development workflow specified in `workflow.md`.
   - Update the task status to completed (`[x]`) in `plan.md`.
   - For any "Phase Completion Verification" task: pause execution, present the completed features, and **invoke interaction `phase-commit-confirm`** (`trigger: on-confirmation`) before committing.
4. **Finalize Track**:
   - Update the track status to completed (`[x]`) in the master `tracks.md` registry.
   - Commit the registry update: `chore(conductor): Mark track '<track_description>' as complete`.

---

### Protocol 2.4: Project Documentation Sync
Upon completing a track, automatically synchronize the master product documents with the new state:

1. **Load Contexts**: Read track's `spec.md` and the master product-level documentation (`product.md`, `tech-stack.md`, `product-guidelines.md`).
2. **Propose Modifications**: Draft the exact edits (or unified diffs) to reflect the new features or capabilities in the master documents.
3. **Invoke interaction `docs-sync-confirm`** (`trigger: on-confirmation`) — present proposed changes and wait for approval. Once approved, write the edits and commit: `docs(conductor): Synchronize docs for track '<track_description>'`.

---

### Protocol 2.5: Status Check (`/conductor:status`)
Generate a precise snapshot of the project state:

1. **Registry Scan**: Read `conductor/tracks.md`.
2. **Progress Analysis**: Count completed, active, and pending tracks and tasks.
3. **Format & Present**: Render a structured dashboard displaying:
   - **Current Date/Time** & overall Project Health.
   - **Current Phase and Task** marked as in-progress.
   - **Next Action Needed** and outstanding blockers.
   - **Metrics**: Total Phases, Total Tasks, Completion percentage.

---

### Protocol 2.6: Track Review & Cleanup
Validate completed tracks or archive them:

- **Review**: Validate that implementation meets the guidelines in `product-guidelines.md` and fulfills the spec acceptance criteria.
- **Archive**: Move the track folder to `conductor/archive/<track_id>`, remove its section from `conductor/tracks.md`, and commit the archival.
- **Delete**: **Invoke interaction `phase-commit-confirm`** for final confirmation before permanent deletion. Delete the track directory, remove it from the tracks registry, and commit the changes.
