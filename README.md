# Kiro Memory Powers

Two complementary Kiro powers for AI agent memory: long-term recall across sessions and short-term introspection within sessions.

**Author:** AWS | **License:** MIT

## Powers in this repo

### [ltm-power](ltm-power/) — Local Long-Term Memory

Project-local memory that persists across sessions. Tracks file changes, saves checkpoints with decisions and threads, and recalls past work cheaply.

- Say **"Remember this project."** to bootstrap.
- Say **"Pick up where we left off."** to recall.
- Say **"Save a checkpoint."** to save a milestone.

[Full LTM documentation →](ltm-power/README.md)

### [stm-power](stm-power/) — STM Introspection

Agent self-improvement through structured observation capture. Records what works and what doesn't, builds consensus across sessions, and graduates validated insights into actionable learning files.

- Say **"Remember what works."** to bootstrap.
- Say **"Reflect on this session."** to generate observations.
- Say **"Run STM lifecycle."** to compress, distill, and graduate.

[Full STM documentation →](stm-power/README.md)

## Install

In Kiro, open the Powers panel → "Add power from GitHub" → enter this repo URL. Point to the `ltm-power/` or `stm-power/` subfolder depending on which power you want.

## How they work together

- **LTM** tracks what happened — files changed, sessions, checkpoints, decisions.
- **STM** tracks what the agent learned from what happened — the evaluative layer.
- When both are installed, STM reads LTM's session IDs to link observations to sessions.
- The dependency is one-way: STM → LTM. LTM never reads from STM.
- Both powers use progressive disclosure to minimize context overhead.
