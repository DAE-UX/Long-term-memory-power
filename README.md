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

### [stm-power](stm-power/) — Short-Term Introspection

Within-session awareness and observation capture. Records what the agent learns during tool usage and promotes high-value observations to long-term memory.

[Full STM documentation →](stm-power/README.md)

## Install

In Kiro, open the Powers panel → "Add power from GitHub" → enter this repo URL. Point to the `ltm-power/` or `stm-power/` subfolder depending on which power you want.

## How they work together

- **STM** captures micro-learnings within a session (observations, patterns, constraints).
- **LTM** preserves durable knowledge across sessions (checkpoints, decisions, file history).
- At session boundaries, STM observations can be promoted into LTM checkpoints.
- Both powers use progressive disclosure to minimize context overhead.
