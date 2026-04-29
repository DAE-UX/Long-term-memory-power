---
name: "ltm-power"
displayName: "Local Long-Term Memory"
description: "Project-local long-term memory and recall for Kiro. Scaffolds an ltm/ recall layer, captures state after meaningful work, enables cheap recall for resume-style tasks, and supports selective or full memory reset, validation, repair, and in-place updates."
author: "AWS"
keywords:
  - "memory"
  - "long term memory"
  - "ltm"
  - "remember this project"
  - "resume work"
  - "pick up where we left off"
  - "project recall"
  - "last session"
  - "continue previous work"
  - "purge memory"
  - "reset memory"
  - "forget"
  - "what do you remember"
  - "update ltm"
  - "update memory"
---

# Local Long-Term Memory

Project-local memory that persists across sessions. Tracks file changes, saves checkpoints with decisions and threads, and recalls past work cheaply — no re-exploration needed.

## Getting started

Say **"Remember this project."**

## How recall works

Three tiers of progressive disclosure:

- **Tier 1** (~50 tokens): Recent files and sessions. Handles most cases.
- **Tier 2** (~200 tokens): Search across checkpoints, decisions, threads.
- **Tier 3** (~500 tokens): Full session detail.

Starts at Tier 1, escalates only when needed. Raw ledgers never enter context.

## Onboarding

### Step 1: Validate Python

```bash
python3 --version   # try first
python --version    # fallback
```

Require 3.9+. Store as `python_cmd`. If unavailable, follow degraded path in `ltm-bootstrap.md`.

### Step 2: Check existing memory

If `ltm/` exists, check `ltm/config.json` for `"created_by": "ltm-power"`:
- Healthy → report and offer recall/purge.
- Damaged → repair via `ltm-failure-recovery.md`.

### Step 3: Bootstrap

If no `ltm/`, read `ltm-bootstrap.md` and execute.

## Commands

| You say | What happens |
|---------|-------------|
| "Remember this project." | Bootstrap |
| "Pick up where we left off." | Recall |
| "Save a checkpoint." | Milestone summary |
| "Forget the last session." | Purge latest session |
| "Forget everything." | Clear all memory |
| "Remove LTM from this project." | Full teardown |
| "Validate memory." | Health check |
| "Repair memory." | Fix damaged files |
| "Update LTM." | Update the LTM tooling to the latest version |

## When to Load Steering Files

- Bootstrapping → `ltm-bootstrap.md`
- Writing ltm.py → `ltm-script-source.md`
- Capture rules → `ltm-capture-policy.md`
- Recall/resume → `ltm-recall-workflow.md`
- Purge/reset → `ltm-purge-and-reset.md`
- Validation → `ltm-validation-policy.md`
- Recovery → `ltm-failure-recovery.md`
- Updating LTM tooling → `ltm-update.md`

## License & Support

**License:** MIT — see [LICENSE](../LICENSE)

**Author:** AWS

**Issues & feedback:** [github.com/DAE-UX/Long-term-memory-power](https://github.com/DAE-UX/Long-term-memory-power/issues)
- Updating LTM tooling → `ltm-update.md`
