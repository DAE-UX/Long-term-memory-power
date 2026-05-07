# STM Capture Policy

> **TL;DR:** Rules for when and how the agent records observations. Prioritizes quality over quantity. Defines source types, stance semantics, and deduplication behavior.

## Observation Sources

| Source | When used | Typical confidence |
|--------|-----------|-------------------|
| `agent:inline` | Agent notices something during normal work | medium |
| `agent:reflect` | End-of-session reflection (stm-reflect hook) | medium–high |
| `agent:compress` | Summary observation from compression | high |
| `user:feedback` | User explicitly says something worked or didn't | high |
| `agent:followup` | Agent applied a graduated learning and observed the result | varies |

## Stance Semantics

- **supports** — evidence that a claim is true. "This approach worked."
- **contradicts** — evidence that a claim is false. "This approach failed."
- **supersedes** — a better approach replaces an earlier one. Requires `--supersedes <obs_id>`. Must share same topic and scope.
- **neutral** — an observation without a directional claim. "Noticed this pattern." Used for gap-filling and exploratory notes.

## Quality Guidelines

Record observations that are:
- **Specific** — "Reading all 3 related files before editing reduced corrections to zero" not "reading files first is good."
- **Evidenced** — include what actually happened, not just what you believe.
- **Scoped** — declare the bounded context where this applies.
- **Cued** — include the situational trigger when possible (≤10 words). This is what makes learnings retrievable later.

Do NOT record:
- Routine operations that went as expected.
- One-off situations unlikely to recur.
- Observations that duplicate an existing same-session observation (the script deduplicates by hash of topic+stance+claim).

## Capture Frequency

Typical session: 2–5 observations. More is acceptable during complex or novel work. Fewer is fine for routine sessions.

The reflect hook generates 1–3 observations per invocation. Inline capture during work adds 0–3 more. Total per session: ~3–8 observations.

## Deduplication

The `record` command hashes `(topic, stance, claim)` and checks against same-session observations. Duplicates within a session are skipped silently.

Cross-session duplicates are intentionally allowed — the same insight observed in multiple sessions is genuine consensus signal, not noise.

## Topic Reuse

Before creating a new topic key, run `stm.py topics` to see existing keys. Reuse an existing topic when the subject matches. Only create a new key when no existing topic covers the subject.

Topic keys are kebab-case: `batch-file-reads-before-edits`, `inline-validation-patterns`, `test-first-approach`.

## Scope Management

When `scope_discovery` is `"auto"` in config, the `record` command accepts any kebab-case scope name and adds new scopes to `config.json` automatically.

When `scope_discovery` is `"manual"`, the scope must already exist in `config.json`. Unknown scopes are rejected.

## Token Cost

- Inline capture: ~150–250 tokens per observation (formulate + record).
- With cue field: ~180–280 tokens.
- Topic lookup before capture: ~20 tokens output.
- Total per-session capture cost: ~500–1,000 tokens for 3–5 observations.
