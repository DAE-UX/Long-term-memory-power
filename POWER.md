---
name: "ltm-power"
displayName: "Local Long-Term Memory"
description: "Project-local long-term memory and recall for Kiro. Scaffolds an ltm/ recall layer, captures state after meaningful work, enables cheap recall for resume-style tasks, and supports selective or full memory reset."
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
---

# Local Long-Term Memory

Give your Kiro agent project-local memory that persists across sessions. The agent tracks what files changed, saves checkpoints with decisions and open threads, and recalls past work cheaply — without re-exploring the entire project.

## Getting started

Say **"Remember this project."** to bootstrap the memory system.

## How recall works

Memory uses progressive disclosure with three tiers:

- **Tier 1** (~50 tokens): Recent files and session summaries. Enough to orient the agent in most cases.
- **Tier 2** (~200 tokens): Focused search across checkpoints, decisions, and threads.
- **Tier 3** (~500 tokens): Full session detail for deep investigation.

The agent starts at Tier 1 and escalates only when needed. Raw ledger files are never injected into context.

## Onboarding

### Step 1: Validate Python

Check for a working Python interpreter:

```bash
python3 --version   # try first
python --version    # fallback
```

Require Python 3.9 or later. Store the working command as `python_cmd`. If neither is available, follow the degraded bootstrap path in `ltm-bootstrap.md` — the system will work in manual mode without automatic capture.

### Step 2: Check for existing memory

If `ltm/` already exists at the project root, check `ltm/config.json` for `"created_by": "ltm-power"`:
- If healthy: report existing memory and offer recall/purge options.
- If damaged: run repair from `ltm-failure-recovery.md`.

### Step 3: Bootstrap

If `ltm/` does not exist, read `ltm-bootstrap.md` and execute the full bootstrap workflow.

## Available commands

| You say | What happens |
|---------|-------------|
| "Remember this project." | Bootstrap the LTM scaffold |
| "Pick up where we left off." | Recall recent work |
| "Save a checkpoint." | Write a milestone summary |
| "Forget the last session." | Remove only the latest session |
| "Forget everything." | Clear all memory, keep structure |
| "Remove LTM from this project." | Full teardown |
| "Validate memory." | Run health check |
| "Repair memory." | Fix missing or damaged files |

## When to Load Steering Files

- Bootstrapping or setting up LTM → `ltm-bootstrap.md`
- Writing ltm.py to a project → `ltm-script-source.md`
- Understanding capture rules → `ltm-capture-policy.md`
- Performing recall or resume → `ltm-recall-workflow.md`
- Purging or resetting memory → `ltm-purge-and-reset.md`
- Validating or testing memory → `ltm-validation-policy.md`
- Recovering from errors → `ltm-failure-recovery.md`
