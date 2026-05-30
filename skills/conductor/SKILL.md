---
name: conductor
description: Orchestrate context-driven software development by managing specs, plans, and implementations using the Conductor methodology.
version: 1.0.0
---

# Conductor: Context-Driven Development Protocol

Conductor is a systematic context-driven methodology that ensures all development tasks follow a rigorous, high-quality lifecycle: **Context -> Specification & Plan -> Implementation -> Documentation Sync**.

Instead of writing code directly, this skill guides you through formalizing requirements, structuring interactive checkpoints, managing a persistent tracks registry, and executing task lists step-by-step.

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
        ├── plan.md           #Phased task checklist
        └── metadata.json     # Track state and configuration
```

---

## 2. Core Operational Protocols

### Protocol 2.1: Project Setup (`/conductor:setup`)
Use this protocol when initializing a new or existing project with Conductor:

1. **Audit Existing State**: Check if any Conductor files already exist. Report status and offer to resume or overwrite.
2. **Project Inception (Greenfield vs. Brownfield)**:
   - *Brownfield (Existing)*: Scan project configuration files (`package.json`, `requirements.txt`, `pyproject.toml`, etc.) to infer the technology stack and current project goal. Ask the user to confirm.
   - *Greenfield (New)*: Initialize git repository (if missing), and ask the user "What do you want to build?" using a text input.
3. **Interactive Documentation Drafting**:
   - **Product (`product.md`)**: Gather target users, high-level features, and goals interactively or autogenerate from concept.
   - **Product Guidelines (`product-guidelines.md`)**: Standardize branding, UX, and prose preferences.
   - **Tech Stack (`tech-stack.md`)**: List and agree upon programming languages, frameworks, database systems, and cloud providers.
   - **Workflow (`workflow.md`)**: Customize testing rules (e.g., target test coverage, commit frequency, and manual verification steps).
   - **Style Guides (`code_styleguides/`)**: Import appropriate style guides based on the tech stack.
4. **Finalize Scaffolding**: Create `conductor/index.md` as the main navigation map, and write the initial master `tracks.md` file.

---

### Protocol 2.2: New Track Creation (`/conductor:newTrack`)
Use this protocol when launching a new feature, bug fix, chore, or refactoring task:

1. **Prerequisites Check**: Ensure `product.md`, `tech-stack.md`, and `workflow.md` exist and are fully populated.
2. **Track Identification**: Define a short unique identifier for the track (e.g., `feature_login_20260530` or `bugfix_db_timeout_20260530`) based on a description.
3. **Draft the Specification (`spec.md`)**:
   - Interactively ask clarifying questions (up to 4, batched together).
   - Draft sections: Overview, Functional Requirements, Non-Functional Requirements, Acceptance Criteria, and Out of Scope.
   - Present the draft to the user for approval.
4. **Generate the Implementation Plan (`plan.md`)**:
   - Translate the approved `spec.md` into an actionable, phased to-do list.
   - Ensure plan structure matches `workflow.md` (e.g., if TDD is required, each item must list "Write Tests" followed by "Implement").
   - Append a final verification checkpoint at the end of each Phase:
     `- [ ] Task: Conductor - User Manual Verification '<Phase Name>'`
   - Prompt the user to review and approve the generated plan.
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
   - Execute the development workflow specified in `workflow.md` (e.g., Code -> Test -> Verify -> Commit).
   - Update the task status to completed (`[x]`) in `plan.md`.
   - For any "Phase Completion Verification" task, pause execution, present the completed features to the user, and ask for manual verification approval before proceeding.
4. **Finalize Track**:
   - Update the track status to completed (`[x]`) in the master `tracks.md` registry.
   - Commit the registry update: `chore(conductor): Mark track '<track_description>' as complete`.

---

### Protocol 2.4: Project Documentation Sync
Upon completing a track, automatically synchronize the master product documents with the new state:

1. **Load Contexts**: Read track's `spec.md` and the master product-level documentation (`product.md`, `tech-stack.md`, `product-guidelines.md`).
2. **Propose Modifications**: Draft the exact edits (or unified diffs) to reflect the new features or capabilities in the master documents.
3. **Confirm and Write**: Present proposed changes to the user for approval. Once approved, write the edits and commit them: `docs(conductor): Synchronize docs for track '<track_description>'`.

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
- **Delete**: Prompt for final confirmation, delete the track directory permanently, remove it from the tracks registry, and commit the changes.
