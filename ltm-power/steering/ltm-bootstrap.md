# LTM Bootstrap

> **TL;DR:** Creates the `ltm/` memory layer, `ltm/bin/ltm.py` CLI, a capture hook, workspace steering files, and a `.gitignore` block. Organized into 4 phases with verification gates. Falls back to degraded mode without Python.

## Prerequisites

1. **Detect Python:** Try `python3 --version`, then `python --version`. Require >= 3.9. Store as `python_cmd`. If unavailable → Degraded Bootstrap below.
2. **Check platform:** If Windows (`os.name == "nt"`), warn: "Windows is not a tested v1 target."
3. **Check conflicts:** Look for existing `.kiro/hooks/ltm-*.kiro.hook` or `.kiro/steering/ltm-*.md`. If found, report and ask.
4. **Check existing `ltm/`:** If it exists with `config.json` containing `"created_by": "ltm-power"` → Case B or C below. Without the marker → report conflict and ask.

---

## Case A — Fresh install

<phase name="structure">

## Phase 1: Structure (directories + config + ledgers)

### 1. Create directories

Use shell `mkdir -p` to ensure all directories exist:

```bash
mkdir -p ltm/store ltm/runtime ltm/reports ltm/snapshots ltm/bin
```

### 2. Create `ltm/config.json`

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

If the write tool rejects empty content, use shell `touch` or write a single newline:

```bash
touch ltm/store/events.jsonl ltm/store/checkpoints.jsonl ltm/store/sessions.jsonl ltm/store/open_threads.jsonl
```

### 4. Create runtime placeholders

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
  "hook_status": "freshly_installed",
  "overall": "healthy",
  "state": "healthy-active"
}
```

Create `.gitkeep` in `ltm/reports/` and `ltm/snapshots/`.

### 5. Create `ltm/README.md`

```markdown
# ltm/ — Local Long-Term Memory

Project-local memory managed by ltm-power.

## Commit policy: repo-portable tooling, local-private memory

**Commit:** `ltm/bin/ltm.py`, `ltm/config.json`, `ltm/manifest.json`, this README.
**Do NOT commit:** `ltm/store/`, `ltm/runtime/`, `ltm/reports/`, `ltm/snapshots/`.

If the hook uses an absolute path, review `.kiro/hooks/ltm-postturn-capture.kiro.hook` before committing.

## Commands

Read `python_cmd` from `ltm/config.json`.

- `<python_cmd> ltm/bin/ltm.py files --limit 10`
- `<python_cmd> ltm/bin/ltm.py health`
- `<python_cmd> ltm/bin/ltm.py checkpoint --summary "..."`
- `<python_cmd> ltm/bin/ltm.py validate`
- `<python_cmd> ltm/bin/ltm.py repair`
- `<python_cmd> ltm/bin/ltm.py purge-last --confirm`
- `<python_cmd> ltm/bin/ltm.py purge-all --confirm`
- `<python_cmd> ltm/bin/ltm.py teardown --confirm`
```

**STOP — Verify Phase 1:** Confirm all 5 directories exist and all files from steps 2-5 are present before continuing.

</phase>

<phase name="script">

## Phase 2: Script

### 6. Generate `ltm/bin/ltm.py`

Read `ltm-script-source.md` and write its fenced code block contents to `ltm/bin/ltm.py`.

**Verification:** Run `<python_cmd> ltm/bin/ltm.py selftest --quick`. If selftest passes, the script is correct regardless of hash. If the SHA-256 hash also matches the expected value in `ltm-script-source.md`, record it in the manifest. If the hash doesn't match but selftest passes, proceed — write tool artifacts (extra newlines from chunked writes) can cause hash mismatches without affecting functionality.

**STOP — Verify Phase 2:** Selftest must pass before continuing.

</phase>

<phase name="integration">

## Phase 3: Integration (hook, steering, gitignore, manifest)

These steps can be done in any order.

### 7. Create capture hook

Write to `.kiro/hooks/ltm-postturn-capture.kiro.hook`:

```json
{
  "name": "LTM Post-Turn Capture",
  "version": "1.0.0",
  "description": "Records agent activity to LTM memory store after each turn",
  "when": { "type": "agentStop" },
  "then": {
    "type": "runCommand",
    "command": "<python_cmd> ltm/bin/ltm.py capture-turn"
  }
}
```

### 8. Create workspace steering files

**`.kiro/steering/ltm-operations.md`:**

```markdown
---
inclusion: auto
name: ltm-operations
description: "LTM memory operations. Activates when the user asks to resume work, continue previous tasks, recall past decisions, or reference project memory."
---

## Recall

1. Run `<python_cmd> ltm/bin/ltm.py files --limit 10` and `<python_cmd> ltm/bin/ltm.py sessions --limit 5`.
2. For specific past work: `<python_cmd> ltm/bin/ltm.py search "term"` or `<python_cmd> ltm/bin/ltm.py checkpoints --days 3`.
3. For a specific session: `<python_cmd> ltm/bin/ltm.py show <session_id>`.
4. Fallback: read `ltm/runtime/active-context.json` and `ltm/runtime/last-recall.md`.
5. Optional: `git status --short` and `git log --oneline -5`.
6. Use recall to orient. Do not explore broadly if recall is sufficient.

## Checkpoints

1. Read recent events from `ltm/store/events.jsonl` (last 20 entries).
2. Prepare checkpoint JSON with summary, changed_files, decisions, open_threads, next_actions.
3. Run `<python_cmd> ltm/bin/ltm.py checkpoint --from-json <path>`.
4. Fallback: write directly to `ltm/store/checkpoints.jsonl` per `ltm-memory-format.md`.
5. Regenerate: `<python_cmd> ltm/bin/ltm.py regenerate`.

## Caps

- `last-recall.md`: max 400 words.
- `active-context.json`: max 8KB (preferred 2KB).
- Never inject raw ledger files into context.

## Maintenance

- Health: `<python_cmd> ltm/bin/ltm.py health`
- Validate: `<python_cmd> ltm/bin/ltm.py validate`
- Repair: `<python_cmd> ltm/bin/ltm.py repair`
- Purge last: `<python_cmd> ltm/bin/ltm.py purge-last --confirm`
- Purge all: `<python_cmd> ltm/bin/ltm.py purge-all --confirm`
- Teardown: `<python_cmd> ltm/bin/ltm.py teardown --confirm`
- Selftest: `<python_cmd> ltm/bin/ltm.py selftest`

Read `python_cmd` from `ltm/config.json`.
```

**`.kiro/steering/ltm-memory-format.md`:**

```markdown
---
inclusion: fileMatch
fileMatchPattern: "ltm/**/*"
---

## LTM record formats

### events.jsonl
Each line: {"id": "evt_NNNNNN", "ts": "ISO-8601", "type": "file_write|checkpoint|manual_note", "summary": "", "files_changed_count": N, "files_sample": ["path", ...], "branch": "string", "session_id": "sess_...", "source": "hook:agentStop|agent:checkpoint|user:manual", "git_status": "ok|unavailable|not_repo|timeout|error", "tags": [], "redacted": false}

### checkpoints.jsonl
Each line: {"id": "chk_NNNNNN", "ts": "ISO-8601", "summary": "text", "changed_files": [], "decisions": [{"decision": "text", "rationale": "text"}], "open_threads": ["thread_id"], "next_actions": ["text"], "session_id": "sess_..."}

### sessions.jsonl
Each line: {"session_id": "sess_...", "started_at": "ISO-8601", "ended_at": "ISO-8601", "summary": "text", "recent_files": [], "checkpoints_created": [], "unresolved_items": [], "next_recommended_action": "text"}

### open_threads.jsonl
Each line: {"thread_id": "thread_NNNNNN", "ts_opened": "ISO-8601", "summary": "text", "status": "open|resolved", "linked_files": [], "last_touched": "ISO-8601"}

### Secret redaction
Check for: sk_live_, sk_test_, AKIA, ghp_, gho_, -----BEGIN, Bearer, base64 strings 40+ chars. Replace with [REDACTED].
```

Replace all `<python_cmd>` with the detected interpreter.

### 9. Append to `.gitignore`

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
  "ltm_py_hash": "<SHA-256 or 'selftest-verified'>",
  "files": [
    "ltm/README.md", "ltm/config.json", "ltm/manifest.json",
    "ltm/store/events.jsonl", "ltm/store/checkpoints.jsonl",
    "ltm/store/sessions.jsonl", "ltm/store/open_threads.jsonl",
    "ltm/runtime/active-context.json", "ltm/runtime/last-recall.md",
    "ltm/runtime/current-session.json", "ltm/runtime/health.json",
    "ltm/bin/ltm.py"
  ],
  "hooks": [".kiro/hooks/ltm-postturn-capture.kiro.hook"],
  "steering": [".kiro/steering/ltm-operations.md", ".kiro/steering/ltm-memory-format.md"],
  "managed_patches": [{"file": ".gitignore", "start_delimiter": "# --- ltm-power ---", "end_delimiter": "# --- /ltm-power ---"}]
}
```

</phase>

<phase name="verification">

## Phase 4: Verification

### 11. Run health and selftest

```bash
<python_cmd> ltm/bin/ltm.py health
<python_cmd> ltm/bin/ltm.py selftest --quick
```

### 12. Verify hook activation

Ask the user to confirm "LTM Post-Turn Capture" appears in Kiro's Agent Hooks panel. If not, provide manual setup instructions. Update `hook_status` in `health.json` to `verified` or `manual_setup_required`.

### 13. Report success

- "I'll track meaningful agent turns automatically."
- "Say 'Save a checkpoint' after milestones."
- "Say 'What do you remember?' to recall."
- "Say 'Validate memory' if recall seems stale."
- "Say 'Forget the last session' or 'Forget everything' to purge."

**IMPORTANT:** Complete all bootstrap steps before generating reports or documentation about the process.

</phase>

---

## Case B — Healthy existing install

1. Read `ltm/config.json` to confirm ownership.
2. Run `<python_cmd> ltm/bin/ltm.py health`.
3. Report: "This project already has long-term memory. Say 'What do you remember?' for a summary, or 'Forget everything' to reset."

## Case C — Damaged existing install

1. Read `ltm/config.json` to confirm ownership.
2. Report: "Found existing memory with missing or corrupted files. Repairing without losing data."
3. Run repair, then health check. Report results.

---

## Resume (interrupted bootstrap)

If bootstrap was interrupted mid-process:

1. If `ltm/bin/ltm.py` exists → run `<python_cmd> ltm/bin/ltm.py validate` to see what's missing.
2. If `ltm/bin/ltm.py` doesn't exist → check directory listing against Phase 1 expectations.
3. Skip completed steps. Resume from the first incomplete phase.
4. Do not re-run steps that already succeeded.

---

## Degraded bootstrap (no Python)

1. Create directories: `mkdir -p ltm/store ltm/runtime ltm/reports ltm/snapshots`. No `ltm/bin/`.
2. Create `ltm/config.json` with `"python_cmd": null`, `"script_status": "unavailable"`, `"mode": "degraded-agent-managed"`.
3. Create empty ledgers and runtime placeholders.
4. Create `ltm/README.md` and `ltm/manifest.json` (no `ltm_py_hash`).
5. Do NOT create the capture hook.
6. Create workspace steering for agent-driven manual capture/recall.
7. Set `hook_status: "disabled"` in `health.json`.
8. Append `.gitignore` block.
9. Report: "LTM installed in degraded mode. Automatic capture unavailable until Python 3.9+ is installed."

---

## Script source

Read `ltm-script-source.md` for the canonical `ltm.py` source and SHA-256 hash.
