# LTM Bootstrap

This file contains the complete bootstrap workflow for setting up LTM in a project. It is loaded only when the user says "Remember this project" or equivalent bootstrap phrases.

## Prerequisites

Before bootstrapping, the agent must:

1. **Detect Python:** Try `python3 --version`, then `python --version`. Require >= 3.9. Store the working command. If neither is available, follow the Degraded Bootstrap section below.
2. **Check for Windows:** If `os.name == "nt"`, warn: "Windows is not a tested v1 target. LTM may work but hooks and shell commands may require manual adjustment."
3. **Check for conflicts:** Look for existing `.kiro/hooks/ltm-*.kiro.hook` or `.kiro/steering/ltm-*.md` files. If found, report and ask before proceeding.
4. **Check for existing `ltm/`:** If it exists with `config.json` containing `"created_by": "ltm-power"`, follow Case B (re-bootstrap) or Case C (repair) from the action plan. If it exists without the marker, report conflict and ask the user.

## Bootstrap steps (Case A — fresh install)

Create the following structure. All paths are relative to the project root.

### 1. Create directories

```
ltm/
ltm/store/
ltm/runtime/
ltm/reports/
ltm/snapshots/
ltm/bin/
```

### 2. Create `ltm/config.json`

Replace `<python_cmd>` with the detected interpreter (`python3` or `python`).

```json
{
  "created_by": "ltm-power",
  "version": "1.0.0",
  "created_at": "<ISO-8601-now>",
  "python_cmd": "<python_cmd>",
  "schema_version": 1,
  "exclude_paths": ["ltm/**", "node_modules/**", "dist/**", "build/**", ".next/**", "coverage/**"],
  "sensitive_path_patterns": [".env*", "secrets/**", "credentials/**", ".aws/**", ".ssh/**", ".kube/**", "*.pem", "*.key", "*.p12", "*.pfx", "*.crt", "*.cert", "id_rsa", "id_ed25519", "kubeconfig"],
  "session_timeout_minutes": 60,
  "max_recent_files": 15,
  "event_retention_days": 30,
  "semantic_checkpoint_warn_hours": 8
}
```

### 3. Create empty ledger files

Create these as empty files (0 bytes):
- `ltm/store/events.jsonl`
- `ltm/store/checkpoints.jsonl`
- `ltm/store/sessions.jsonl`
- `ltm/store/open_threads.jsonl`

### 4. Create placeholder runtime files

**`ltm/runtime/active-context.json`:**
```json
{
  "generated_at": "<ISO-8601-now>",
  "healthy": true,
  "active_workstream": null,
  "recent_files": [],
  "recent_sessions": [],
  "open_threads": [],
  "next_actions": [],
  "token_budget_status": "pass",
  "semantic_coverage_status": "pass",
  "inferred_workstream": {"value": null, "source": "unknown", "confidence": "low"},
  "staleness_seconds": 0
}
```

**`ltm/runtime/last-recall.md`:**
```markdown
## Recent work
No activity recorded yet.

## Open threads
None.

## Next actions
Say "Save a checkpoint" after completing meaningful work.
```

**`ltm/runtime/current-session.json`:**
```json
{
  "session_id": "sess_<YYYY_MM_DD>_01",
  "started_at": "<ISO-8601-now>",
  "last_event_at": null,
  "event_count": 0
}
```

**`ltm/runtime/health.json`:**
```json
{
  "checked_at": "<ISO-8601-now>",
  "store_freshness_hours": 0,
  "ledger_integrity": "pass",
  "runtime_freshness_hours": 0,
  "budget_status": "pass",
  "capture_coverage": "pass",
  "semantic_coverage": {"last_checkpoint_age_hours": 0, "structural_sessions_without_checkpoint": 0, "status": "pass"},
  "hook_status": "file_created",
  "overall": "healthy",
  "state": "healthy-active"
}
```

Create `.gitkeep` files in `ltm/reports/` and `ltm/snapshots/`.

### 5. Create `ltm/README.md`

```markdown
# ltm/ — Local Long-Term Memory

This directory contains project-local memory managed by ltm-power.

## Commit policy

v1 uses **repo-portable tooling with local-private memory data**.

**Safe to commit:** `ltm/bin/ltm.py`, `ltm/config.json`, `ltm/manifest.json`, this README.

**Do NOT commit:** `ltm/store/`, `ltm/runtime/`, `ltm/reports/`, `ltm/snapshots/` — these contain local memory data and are gitignored.

If the hook command was changed to use an absolute path, review `.kiro/hooks/ltm-postturn-capture.kiro.hook` before committing — absolute hook commands are machine-specific.

## Quick reference

Read `python_cmd` from `ltm/config.json` for the correct interpreter.

- Recall: `<python_cmd> ltm/bin/ltm.py files --limit 10`
- Health: `<python_cmd> ltm/bin/ltm.py health`
- Checkpoint: `<python_cmd> ltm/bin/ltm.py checkpoint --summary "..."`
- Validate: `<python_cmd> ltm/bin/ltm.py validate`
- Repair: `<python_cmd> ltm/bin/ltm.py repair`
- Purge last: `<python_cmd> ltm/bin/ltm.py purge-last --confirm`
- Purge all: `<python_cmd> ltm/bin/ltm.py purge-all --confirm`
- Teardown: `<python_cmd> ltm/bin/ltm.py teardown --confirm`
```

### 6. Generate `ltm/bin/ltm.py`

Write the literal Python source below to `ltm/bin/ltm.py`. After writing, compute the SHA-256 hash of the file. The expected hash is recorded at the end of this section. If the hashes don't match, report a generation error.

The full `ltm.py` source is provided in the next section of this file.

### 7. Create the capture hook

Write to `.kiro/hooks/ltm-postturn-capture.kiro.hook`:

```json
{
  "name": "LTM Post-Turn Capture",
  "version": "1.0.0",
  "description": "Records agent activity to LTM memory store after each turn",
  "when": {
    "type": "agentStop"
  },
  "then": {
    "type": "runCommand",
    "command": "<python_cmd> ltm/bin/ltm.py capture-turn"
  }
}
```

Replace `<python_cmd>` with the detected interpreter.

### 8. Create workspace steering files

**`.kiro/steering/ltm-operations.md`** — see the workspace steering content in the action plan §10.1. Replace all `<python_cmd>` placeholders with the detected interpreter.

**`.kiro/steering/ltm-memory-format.md`** — see the workspace steering content in the action plan §10.2.

### 9. Append to `.gitignore`

Append this block to the project's `.gitignore` (create the file if it doesn't exist):

```gitignore
# --- ltm-power ---
ltm/store/*.jsonl
ltm/runtime/*
ltm/reports/*
ltm/snapshots/*
# --- /ltm-power ---
```

### 10. Create `ltm/manifest.json`

```json
{
  "created_by": "ltm-power",
  "version": "1.0.0",
  "created_at": "<ISO-8601-now>",
  "ltm_py_hash": "<SHA-256 of ltm/bin/ltm.py>",
  "files": [
    "ltm/README.md",
    "ltm/config.json",
    "ltm/manifest.json",
    "ltm/store/events.jsonl",
    "ltm/store/checkpoints.jsonl",
    "ltm/store/sessions.jsonl",
    "ltm/store/open_threads.jsonl",
    "ltm/runtime/active-context.json",
    "ltm/runtime/last-recall.md",
    "ltm/runtime/current-session.json",
    "ltm/runtime/health.json",
    "ltm/bin/ltm.py"
  ],
  "hooks": [
    ".kiro/hooks/ltm-postturn-capture.kiro.hook"
  ],
  "steering": [
    ".kiro/steering/ltm-operations.md",
    ".kiro/steering/ltm-memory-format.md"
  ],
  "managed_patches": [
    {
      "file": ".gitignore",
      "start_delimiter": "# --- ltm-power ---",
      "end_delimiter": "# --- /ltm-power ---"
    }
  ]
}
```

### 11. Verify installation

```bash
<python_cmd> ltm/bin/ltm.py health
<python_cmd> ltm/bin/ltm.py selftest --quick
```

### 12. Verify hook activation

Ask the user to confirm the "LTM Post-Turn Capture" hook appears in Kiro's Agent Hooks panel. If it does not appear, provide manual setup instructions. Record `hook_status` in `health.json` as `verified` or `manual_setup_required`.

### 13. Report success

Tell the user:
- "I'll track meaningful agent turns automatically."
- "Say 'Save a checkpoint' after milestones."
- "Say 'What do you remember?' or 'Pick up where we left off' to recall."
- "Say 'Validate memory' if recall seems stale."
- "Say 'Forget the last session' or 'Forget everything' to purge."

## Degraded bootstrap (Python unavailable)

If neither `python3` nor `python` >= 3.9 is available:

1. Create `ltm/` directory structure (store/, runtime/, reports/, snapshots/). Do NOT create `ltm/bin/`.
2. Create `ltm/config.json` with `"python_cmd": null`, `"script_status": "unavailable"`, `"mode": "degraded-agent-managed"`.
3. Create empty JSONL ledger files and placeholder runtime files.
4. Create `ltm/README.md`.
5. Create `ltm/manifest.json` (without `ltm_py_hash`).
6. Do NOT create the `agentStop` `runCommand` hook.
7. Create workspace steering files that explain agent-driven manual capture/recall.
8. Set `hook_status: "disabled"` and `script_status: "unavailable"` in `health.json`.
9. Append `.gitignore` block.
10. Report: "LTM is installed in degraded mode. Automatic capture is unavailable until Python 3.9+ is installed."

---

## `ltm.py` source

Read `ltm-script-source.md` and write its contents to `ltm/bin/ltm.py`. That steering file contains the canonical script source and the expected SHA-256 hash for verification.
