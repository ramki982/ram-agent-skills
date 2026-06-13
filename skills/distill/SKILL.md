---
name: distill
description: Extract the reusable mechanisms from a conversation — the transferable "how" — and discard the task-specific details, so the method carries into future unrelated work. Self-scales to long conversations that exceed a single context pass.
argument-hint: "(optional) a prior distillation to update instead of starting fresh"
---

Separate the **reusable mechanism** from the **disposable specifics** in this conversation. Keep only what would change how a *future, unrelated* session works. Specifics (names, numbers, the task itself) are mostly disposable — the mechanisms are the asset.

## Discrimination (the core — applies in every mode)

A candidate is a *way of working* that would transfer to a completely different topic, stated in one sentence as a transferable rule. Each candidate must survive two tests:

1. **Operational:** If a future session didn't know this, would it do meaningfully worse work? (No → cut.)
2. **Distinctiveness:** Pass if EITHER (a) it's specific to how *this* work happened, not just generally good practice any competent agent already does, OR (b) it's generic but *high-recurrence* — it visibly drove the conversation across multiple distinct moments. If generic AND it appeared only once → cut.

Then **merge** survivors that are the same discipline seen from two angles. **Hard cap: 3–5 mechanisms.** More than 5 means you haven't merged or cut enough.

## Mode selection

Estimate whether the full conversation fits comfortably in one pass.

- **Fits → single pass.** Apply the discrimination above directly.
- **Too long to hold at once → segment via a temp file.** Do not silently process only what remains in context; that produces confident, partial output. Instead:
  1. Process the conversation in sequential segments. For each segment, append its candidate mechanisms (loosely — capture, don't cut) to a scratch file in the OS temp directory.
  2. When all segments are done, read the file back and apply the discrimination, merge, and 3–5 cap once over the complete set.

  The temp file is disposable working state, not output — it keeps the reduce step from drowning and makes a long distillation resumable if interrupted.

If a prior distillation was passed as an argument, seed the file with its mechanisms and process only the new material — update the existing set rather than starting over.

## Output (always)

Three labelled buckets:

- **Mechanisms worth remembering** (3–5). One line each on *why it earned its place*. If it passed via recurrence (b) rather than distinctiveness (a), say so — flag the generic-but-frequent ones as such.
- **Specifics to retain.** Only facts, decisions, or commitments that future work genuinely depends on. Ruthlessly short. Flag anything likely already known.
- **Disposable.** One line naming what you deliberately did NOT keep, so the cut is visible and the human can correct it. (When segmenting, show this only once at the end, consolidated.)

Show the result first — the distillation has value as soon as it's read. Then ask the user where to save it, if anywhere; leaving it as an on-screen summary is a fine outcome. Do not assume a storage location or write anything automatically. The human reviews the cut and decides what to keep — that review is the point.
