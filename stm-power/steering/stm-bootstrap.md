# STM Bootstrap

> **TL;DR:** Creates the `stm/` introspection layer, `stm/bin/stm.py` CLI, hooks, workspace steering files, and a `.gitignore` block. Checks for Python 3.9+, conflicts, LTM availability, and existing installations before proceeding. Falls back to degraded mode without Python.

## Prerequisites

1. **Detect Python:** Try `python3 --version`, then `python --version`. Require >= 3.9. Store the working command as `python_cmd`. If unavailable, follow Degraded Bootstrap below.
2. **Check platform:** If Windows (`os.name == "nt"`), warn: "Windows is not a tested v1 target."
3. **Check conflicts:** Look for existing `.kiro/hooks/stm-*.kiro.hook` or `.kiro/steering/stm-*.md`. If found, report and ask before proceeding.
4. **Check existing `stm/`:** If it exists with `config.json` containing `"created_by": "stm-power"`, follow Case B or C below. If it exists without the marker, report conflict and ask.
5. **Check for LTM:** If `ltm/config.json` exists with `"created_by": "ltm-power"`, note LTM is available. STM will read `ltm/runtime/current-session.json` for session linking. If LTM is absent, STM manages its own session IDs. If LTM is present, reuse its `python_cmd` to prevent divergence.

## Case A — Fresh install

All paths relative to project root.

### 1. Create directories

```text
stm/store/
stm/runtime/
stm/bin/
```

### 2. Create `stm/config.json`

```json
{
  "created_by": "stm-power",
  "version": "1.1.1",
  "created_at": "<ISO-8601-now>",
  "python_cmd": "<python_cmd>",
  "schema_version": 2,
  "ltm_available": true,
  "consensus_threshold": 3,
  "consensus_score_minimum": 0.6,
  "max_observations_per_topic": 20,
  "retention_days": 90,
  "compression_token_threshold": 50000,
  "merge_candidate_min_shared_words": 3,
  "compaction_warn_bytes": 1048576,
  "compaction_hard_bytes": 5242880,
  "graduation_output_path": "learnings",
  "session_timeout_minutes": 60,
  "scopes": [],
  "scope_discovery": "auto",
  "distill_trigger": "manual",
  "auto_distill_after_observations": 10,
  "sharing_mode": "local",
  "tool_trace_enabled": true,
  "tool_trace_max_entries": 50,
  "scope_maturity_threshold": 5,
  "capture_enabled": true,
  "max_observations_per_session": 50,
  "min_capture_confidence": "low",
  "privacy_patterns": []
}
```

Set `ltm_available` based on whether `ltm/config.json` was detected.
Set `sharing_mode` based on user's choice during bootstrap (see Step 11b).

### 3. Create empty store files (0 bytes)

- `stm/store/observations.jsonl`
- `stm/store/clusters.jsonl`

### 4. Create runtime placeholders

**`stm/runtime/status.json`:**
```json
{
  "generated_at": "<ISO-8601-now>",
  "observation_count": 0,
  "summary_count": 0,
  "compressed_count": 0,
  "cluster_count": 0,
  "pending_graduations": 0,
  "last_distill": null,
  "last_compress": null,
  "compression_needed": false,
  "errors_count": 0
}
```

**`stm/runtime/current-session.json`** (used only when LTM is absent):
```json
{
  "session_id": "stm_sess_<YYYY_MM_DD>_01",
  "started_at": "<ISO-8601-now>",
  "last_event_at": null,
  "event_count": 0
}
```

**`stm/runtime/merge-candidates.json`:**
```json
[]
```

**`stm/runtime/errors.jsonl`:** (empty)

### 5. Create `stm/README.md`

```markdown
# stm/ — Short-Term Memory (Introspection)

Project-local agent introspection managed by stm-power.

## Commit policy: repo-portable tooling, local-private memory

**Commit:** `stm/bin/stm.py`, `stm/config.json`, `stm/manifest.json`, this README.
**Do NOT commit:** `stm/store/`, `stm/runtime/`.

## Commands

Read `python_cmd` from `stm/config.json`.

- `<python_cmd> stm/bin/stm.py record --scope <scope> --topic <topic> --stance <stance> --claim "<claim>" --evidence "<evidence>" --confidence <level> --source <source> [--cue "<pattern>"]`
- `<python_cmd> stm/bin/stm.py topics`
- `<python_cmd> stm/bin/stm.py recall --scope <scope> [--cue "<situation>"] [--limit 3]`
- `<python_cmd> stm/bin/stm.py status`
- `<python_cmd> stm/bin/stm.py compress --identify`
- `<python_cmd> stm/bin/stm.py compress --execute --ids <obs_ids>`
- `<python_cmd> stm/bin/stm.py distill`
- `<python_cmd> stm/bin/stm.py pending-clusters`
- `<python_cmd> stm/bin/stm.py mark-graduated --cluster <id> --path <path>`
- `<python_cmd> stm/bin/stm.py mark-integrated --cluster <id> --files <file1> <file2>`
- `<python_cmd> stm/bin/stm.py trace --tool <name> --outcome <success|failure> [--detail <text>]`
- `<python_cmd> stm/bin/stm.py trace-summary [--session-only] [--limit N]`
- `<python_cmd> stm/bin/stm.py learnings [--scope <scope>] [--cue <text>] [--limit N]`
- `<python_cmd> stm/bin/stm.py sharing [--mode local|shared]`
- `<python_cmd> stm/bin/stm.py search <term> [--scope <scope>] [--days N] [--limit N]`
- `<python_cmd> stm/bin/stm.py health`
- `<python_cmd> stm/bin/stm.py validate`
- `<python_cmd> stm/bin/stm.py repair`
- `<python_cmd> stm/bin/stm.py purge-all --confirm`
- `<python_cmd> stm/bin/stm.py teardown --confirm`
- `<python_cmd> stm/bin/stm.py selftest`
```

### 6. Generate `stm/bin/stm.py`

Read `stm-script-source.md` and write its fenced code block contents to `stm/bin/stm.py`.

**Chunked write procedure** (the script is ~1178 lines — too large for a single write in some environments):

1. Write lines 1–400 (imports, helpers, tool-trace, record command) to `stm/bin/stm.py`
2. Verify file exists and line count >= 400
3. Append lines 401–800 (topics, recall, status, search, compress, distill)
4. Verify line count >= 800
5. Append lines 801–end (cluster management, learnings, sharing, health, validate, repair, purge, teardown, selftest, CLI entry point)
6. Verify line count >= 1170

**CRITICAL — Mandatory verification gate:**
```bash
<python_cmd> stm/bin/stm.py selftest --quick
```
**STOP.** Do NOT proceed to step 7 (hook installation) until selftest passes. If selftest fails, the script is incomplete or corrupted. Re-read `stm-script-source.md` and rewrite the file. A broken script will cause all subsequent hooks to fail silently.

Verify the SHA-256 hash matches the expected value in `stm-script-source.md`. If it does not match but selftest passes, proceed with a warning — hash mismatches can occur from line-ending normalization.

### 7. Install hooks

Write to `.kiro/hooks/`:

**`stm-reflect.kiro.hook`:**
```json
{
  "name": "STM Reflect — End-of-Session Review",
  "version": "1.0.0",
  "description": "Agent reviews the current session, generates observations, and checks for behavioral patterns.",
  "when": { "type": "userTriggered" },
  "then": {
    "type": "askAgent",
    "prompt": "Review the current session's work. First run `<python_cmd> stm/bin/stm.py topics` to see existing topic keys and `<python_cmd> stm/bin/stm.py recall --scope agent-behavior --limit 3` to check for relevant prior learnings. Then: (1) Identify 1-3 things that went well or poorly. If LTM is available, read `ltm/store/events.jsonl` (last 20 entries for this session) to ground observations in what actually happened. If LTM is absent, review recent git activity (`git log --oneline -5`, `git diff --stat`) and recent STM observations instead. For each insight, record via `<python_cmd> stm/bin/stm.py record` with --source agent:reflect. Include --cue when you can identify the situational trigger. (2) If you applied any graduated learnings this session, record follow-up observations with --source agent:followup --applies-to <cluster_id> --outcome <confirmed|mixed|failed>. (3) Briefly scan your observation patterns: run `<python_cmd> stm/bin/stm.py topics` and note if any scopes have zero observations — are you blind to issues there, or is that area genuinely stable? If you notice a gap, record a neutral observation noting it. Keep the whole reflection to 1-3 observations total."
  }
}
```

**`stm-compress.kiro.hook`:**
```json
{
  "name": "STM Compress — Summarize Observation Groups",
  "version": "1.0.0",
  "description": "Compresses raw observations into summaries, preserving 1-2 exemplar cases per group.",
  "when": { "type": "userTriggered" },
  "then": {
    "type": "askAgent",
    "prompt": "Run `<python_cmd> stm/bin/stm.py compress --identify` to find compression candidates. For each candidate group: read all raw observations, select 1-2 as exemplars (prefer observations with cue fields, user:feedback source, or highest confidence), synthesize a single summary observation that combines the claims and evidence, then record it via `<python_cmd> stm/bin/stm.py record --source agent:compress --compressed-from <id1,id2,...>`. After recording all summaries, run `<python_cmd> stm/bin/stm.py compress --execute --ids <original_ids> --exemplars <exemplar_ids>` to mark originals as compressed and exemplars as preserved. Report what was compressed and estimated token savings."
  }
}
```

**`stm-distill.kiro.hook`:**
```json
{
  "name": "STM Distill — Group, Score, and Merge",
  "version": "1.0.0",
  "description": "Runs grouping/scoring on STM observations, then prompts agent to review merge candidates.",
  "when": { "type": "userTriggered" },
  "then": {
    "type": "askAgent",
    "prompt": "Run `<python_cmd> stm/bin/stm.py distill` to group and score observations. If the output reports merge candidates, read `stm/runtime/merge-candidates.json` and for each candidate pair decide: merge (reassign to the same topic key and re-run distill) or keep separate. Report the distill summary."
  }
}
```

**`stm-graduate.kiro.hook`:**
```json
{
  "name": "STM Graduate — Synthesize Learning Proposals",
  "version": "1.0.0",
  "description": "Agent reads pending clusters, synthesizes cue patterns and learning proposals with exemplar cases.",
  "when": { "type": "userTriggered" },
  "then": {
    "type": "askAgent",
    "prompt": "Run `<python_cmd> stm/bin/stm.py pending-clusters` to get clusters ready for graduation. For each pending cluster: (1) verify it contains summary observations — skip if raw-only, (2) read all linked observations, (3) synthesize a cue_pattern from the observations' cue fields — this is the 'when you see X' trigger, (4) read the current state of affected project files, (5) synthesize a consensus claim and actionable recommendation, (6) include the exemplar case(s) in the learning file for narrative richness, (7) include application history if any follow-up observations exist, (8) generate a learning file following the Graduated Learning Template in the STM steering. After writing each file, run `<python_cmd> stm/bin/stm.py mark-graduated --cluster <id> --path <path>`. Also check for clusters flagged for reversal and generate reversal files. Report what was graduated, skipped, and what needs user review."
  }
}
```

**`stm-lifecycle.kiro.hook`:**
```json
{
  "name": "STM Lifecycle — Compress, Distill, Graduate",
  "version": "1.0.0",
  "description": "Runs the full STM lifecycle: compression, distillation, and graduation in sequence.",
  "when": { "type": "userTriggered" },
  "then": {
    "type": "askAgent",
    "prompt": "Execute the STM lifecycle. Steps: (1) Check compression via `<python_cmd> stm/bin/stm.py status` — if compression_needed, run the compress procedure. (2) Run `<python_cmd> stm/bin/stm.py distill` and review merge candidates. (3) Check for pending clusters via `<python_cmd> stm/bin/stm.py pending-clusters` — if any, run the graduate procedure. (4) Report results."
  }
}
```

**`stm-auto-reflect.kiro.hook`:**
```json
{
  "name": "STM Auto-Reflect",
  "version": "1.1.0",
  "description": "Lightweight end-of-turn observation capture. Records one observation if something notable happened, skips otherwise.",
  "when": { "type": "agentStop" },
  "then": {
    "type": "askAgent",
    "prompt": "Did anything go notably well or poorly this turn — a decision that saved time, an approach that caused rework, a pattern you noticed, or user feedback? If you need grounding, run `<python_cmd> stm/bin/stm.py trace-summary --session-only --limit 10` to review recent tool activity. Also check `<python_cmd> stm/bin/stm.py learnings --cue \"<what you just did>\" --limit 2` to see if a graduated learning applied. If yes, record exactly one observation via `<python_cmd> stm/bin/stm.py record --scope <scope> --topic <topic> --stance <stance> --claim \"<one sentence>\" --evidence \"<what happened>\" --confidence <level> --source agent:reflect`. Include --cue if you can identify the situational trigger. If you applied a graduated learning, use --source agent:followup --applies-to <cluster_id> --outcome <confirmed|mixed|failed> instead. If nothing notable happened (routine operation, simple read, trivial change), respond with just: none."
  }
}
```

Replace all `<python_cmd>` with the detected interpreter.

### 8. Create workspace steering files

**`.kiro/steering/stm-observations.md`:**

```markdown
---
inclusion: auto
name: stm-observations
description: "STM observation recording. Guides the agent to notice and record what works and what doesn't during normal work."
---

## Record STM Observations

When you notice something during work that could be a reusable insight:

1. Run `<python_cmd> stm/bin/stm.py topics` to see existing topic keys.
2. Record via CLI: `<python_cmd> stm/bin/stm.py record --scope <scope> --topic <topic> --stance <stance> --claim "<claim>" --evidence "<evidence>" --confidence <low|medium|high> --source agent:inline [--cue "<situational pattern>"]`
   - `scope`: must match a scope from `stm/config.json` → `scopes`. If no scopes are defined yet and `scope_discovery` is `auto`, propose a scope name and the script will add it.
   - `topic`: reuse an existing topic key when the subject matches. Only create a new key when no existing topic covers the subject.
   - `stance`: supports | contradicts | supersedes | neutral
   - `claim`: one sentence — what you believe to be true
   - `evidence`: one to two sentences — what happened
   - `cue`: optional, ≤10 words — the situational pattern that triggered this decision
   - `confidence`: low | medium | high
   - For `supersedes`: add `--supersedes <obs_id>` — must be same topic and scope
3. The script validates, deduplicates, and appends atomically.
4. Prefer fewer, higher-quality observations over many low-quality ones.
5. Record contradictions when you notice them — these are as valuable as confirmations.

## When to Record

- An approach saved significant time or effort → `--stance supports`
- An approach caused rework or confusion → `--stance contradicts`
- You found a better way to do something → `--stance supersedes --supersedes <obs_id>`
- The user gave explicit feedback → `--source user:feedback --confidence high`
- You applied a graduated learning and can see the result → `--source agent:followup --applies-to <cluster_id> --outcome <confirmed|mixed|failed>`

## When NOT to Record

- Routine operations that went as expected
- One-off situations unlikely to recur

## Retrieve Before Deciding

When about to choose an approach in a scope where you've had prior observations:

```bash
<python_cmd> stm/bin/stm.py recall --scope <scope> --cue "<brief situation>" --limit 3
```

This is a "have I been here before?" check, not a pre-flight checklist. Use it when entering unfamiliar territory or revisiting a scope where you've previously struggled.

## Check Learnings Before Repeated Tasks

Before starting work that matches a known pattern, check for applicable graduated learnings:

```bash
<python_cmd> stm/bin/stm.py learnings --cue "<what you're about to do>" --limit 3
```

If a learning applies, follow its recommendation. After completing the task, record a followup observation:
```bash
<python_cmd> stm/bin/stm.py record --scope <scope> --topic <topic> --stance supports --claim "<outcome>" --evidence "<what happened>" --confidence medium --source agent:followup --applies-to <cluster_id> --outcome <confirmed|mixed|failed>
```

## Graduated Learning Template

When graduating a cluster, use this format for the output file:

~~~markdown
# {Topic Title} — STM Learning

> **TL;DR:** {consensus_claim}. Recommended action: {one-sentence summary}.

## Source

- Generated by STM introspection
- Based on {N} observations ({M} summary, {K} raw) across {P} sessions
- Consensus score: {score}
- Scope: {scope}
- Cluster: {cluster_id}

## When This Applies

**Cue pattern:** {cue_pattern}

**Scope:** {scope description}

**Does NOT apply to:** {scope exclusions}

## Claim

{consensus_claim}

## Evidence Summary

{Bulleted list of evidence from supporting observations}

## Exemplar Case

{The most vivid exemplar observation — full claim + evidence + cue.}

## Contradicting Evidence

{Bulleted list from contradicting observations, or "None."}

## Recommendation

### Goal

{What the change achieves — stated as a testable outcome}

### Execution Plan

1. {Step 1}
2. {Step 2}

### Verification

| ID | Check | Pass Criteria |
|----|-------|---------------|
| {ID}-01 | {What to verify} | {How to confirm} |

## Application History

{If applied: N times, success rate, last outcome. If not: "Not yet tested in practice."}

## Reversibility

- Confidence distribution: {count per level}
- Contradicting observations: {count}
- This learning can be invalidated by future observations or poor application outcomes.
~~~

Read `python_cmd` from `stm/config.json`.
```

**`.kiro/steering/stm-memory-format.md`:**

```markdown
---
inclusion: fileMatch
fileMatchPattern: "stm/**/*"
---

## STM record formats

### observations.jsonl
Each line: {"id": "obs_NNNNNN", "ts": "ISO-8601", "session_id": "sess_...|stm_sess_...", "scope": "string", "topic": "kebab-case", "stance": "supports|contradicts|supersedes|neutral", "claim": "string", "evidence": "string", "cue": "string|null", "confidence": "low|medium|high", "source": "agent:inline|agent:reflect|agent:compress|user:feedback|agent:followup", "tags": [], "applies_to": "clst_id|null", "outcome": "confirmed|mixed|failed|null", "supersedes": "obs_id|null", "invalidated": false, "invalidated_by": "obs_id|null", "compressed": false, "compressed_from": ["obs_id"]|null, "exemplar": false}

### clusters.jsonl
Each line: {"cluster_id": "clst_NNNNNN", "topic": "kebab-case", "scope": "string", "observation_ids": ["obs_id"], "supports_count": N, "contradicts_count": N, "consensus_score": float, "consensus_claim": "string|null", "recommendation": "string|null", "cue_pattern": "string|null", "status": "accumulating|pending|graduated|invalidated", "last_updated": "ISO-8601", "graduated": false, "graduated_to": "path|null", "integrated_files": ["path"]|null, "exemplar_ids": ["obs_id"]|null, "application_count": 0, "application_success_rate": null}
```

Replace all `<python_cmd>` with the detected interpreter.

### 9. Append to `.gitignore`

If `sharing_mode` is `"local"` (default):
```gitignore
# --- stm-power ---
stm/store/*.jsonl
stm/runtime/*
# --- /stm-power ---
```

If `sharing_mode` is `"shared"`:
```gitignore
# --- stm-power ---
stm/runtime/*
# --- /stm-power ---
```

### 10. Create `stm/manifest.json`

```json
{
  "created_by": "stm-power",
  "version": "1.1.1",
  "created_at": "<ISO-8601-now>",
  "stm_py_hash": "<SHA-256 of stm/bin/stm.py>",
  "files": [
    "stm/README.md", "stm/config.json", "stm/manifest.json",
    "stm/store/observations.jsonl", "stm/store/clusters.jsonl",
    "stm/runtime/status.json", "stm/runtime/current-session.json",
    "stm/runtime/merge-candidates.json", "stm/runtime/errors.jsonl",
    "stm/runtime/tool-trace.jsonl",
    "stm/bin/stm.py"
  ],
  "hooks": [
    ".kiro/hooks/stm-reflect.kiro.hook",
    ".kiro/hooks/stm-compress.kiro.hook",
    ".kiro/hooks/stm-distill.kiro.hook",
    ".kiro/hooks/stm-graduate.kiro.hook",
    ".kiro/hooks/stm-lifecycle.kiro.hook",
    ".kiro/hooks/stm-auto-reflect.kiro.hook",
    ".kiro/hooks/stm-tool-trace.kiro.hook"
  ],
  "steering": [
    ".kiro/steering/stm-observations.md",
    ".kiro/steering/stm-memory-format.md"
  ],
  "managed_patches": [
    {"file": ".gitignore", "start_delimiter": "# --- stm-power ---", "end_delimiter": "# --- /stm-power ---"}
  ]
}
```

### 11. Run Scope Discovery

If `scope_discovery` is `"auto"`:
1. Scan the project root for common structural patterns:
   - `src/` → suggests a `source-edits` scope
   - `test/` or `tests/` → suggests a `test-edits` scope
   - Config files → suggests a `config-edits` scope
   - CI/CD files → suggests a `ci-cd` scope
   - Documentation files → suggests a `documentation` scope
   - Infrastructure files → suggests an `infrastructure` scope
2. Always include two universal scopes:
   - `agent-behavior` — how the agent interacts with the user
   - `cross-file-consistency` — multi-file edits where consistency matters
3. Present proposed scopes to the user for confirmation
4. Write confirmed scopes to `config.json`

### 11b. Ask Sharing Preference

Ask the user: "Should introspection data be shared with the team (committed to repo) or kept local (gitignored)?"

- If **shared**: set `sharing_mode` to `"shared"` in config. Explain: "Observations will be committed to the repo. All collaborators contribute to and benefit from the same introspection data. Conflicts are rare because observations use append-only JSONL with unique IDs."
- If **local** (default): set `sharing_mode` to `"local"`. Explain: "Observations stay on your machine. Each developer has their own private introspection."

Adjust the `.gitignore` block written in step 9 based on the choice.

### 11c. Install Tool-Trace Hook

**`.kiro/hooks/stm-tool-trace.kiro.hook`:**
```json
{
  "name": "STM Tool Trace — Passive Activity Capture",
  "version": "1.0.0",
  "description": "Records tool invocations to stm/runtime/tool-trace.jsonl for reflection grounding.",
  "when": { "type": "postToolUse", "toolTypes": ["write", "shell"] },
  "then": {
    "type": "runCommand",
    "command": "<python_cmd> stm/bin/stm.py trace --tool \"$TOOL_NAME\" --outcome \"$TOOL_OUTCOME\""
  }
}
```

Note: If LTM is present and already captures tool activity via its `capture-turn` hook, the tool-trace provides supplementary current-session data. Both can coexist without conflict.

### 11d. Create Learnings Directory

```bash
mkdir -p learnings
```

This is where graduated learning files will be written. The directory should be committed to the repo regardless of sharing mode — learnings are always meant to be project-visible.

### 12. Verify

```bash
<python_cmd> stm/bin/stm.py health
<python_cmd> stm/bin/stm.py selftest --quick
```

### 13. Report success

- "I'll track what works and what doesn't as I go."
- "Say 'Reflect on this session' to generate observations from recent work."
- "Say 'Run STM lifecycle' to compress, distill, and graduate insights."
- "Say 'What topics have you observed?' to see current observation topics."
- "Say 'What learnings apply here?' to check for relevant graduated insights."
- "Say 'Share introspection with the team' to enable shared mode."
- "Say 'Forget all observations' to purge."

---

## Case B — Healthy existing install

1. Read `stm/config.json` to confirm ownership.
2. Run `<python_cmd> stm/bin/stm.py health`.
3. Report: "This project already has STM introspection. Say 'What topics have you observed?' for a summary, or 'Forget all observations' to reset."

## Case C — Damaged existing install

1. Read `stm/config.json` to confirm ownership.
2. Report: "Found existing STM with missing or corrupted files. Repairing without losing data."
3. Run repair, then health check. Report results.

---

## Degraded bootstrap (no Python)

1. Create `stm/` structure (store/, runtime/). No `stm/bin/`.
2. Create `stm/config.json` with `"python_cmd": null`, `"mode": "degraded-agent-managed"`.
3. Create empty store files and runtime placeholders.
4. Create `stm/README.md` and `stm/manifest.json` (no `stm_py_hash`).
5. Do NOT create hooks (agent manages observations directly via file writes).
6. Create workspace steering for agent-driven manual capture.
7. Append `.gitignore` block.
8. Report: "STM installed in degraded mode. CLI unavailable until Python 3.9+ is installed."

---

## Script source

Read `stm-script-source.md` for the canonical `stm.py` source and SHA-256 hash.
