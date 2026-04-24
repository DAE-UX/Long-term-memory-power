# LTM Purge and Reset

Three modes of memory removal, from least to most destructive.

## Purge last session

Removes only the most recent session. Older memory is preserved.

Read `python_cmd` from `ltm/config.json`.

```bash
<python_cmd> ltm/bin/ltm.py purge-last          # dry-run (default)
<python_cmd> ltm/bin/ltm.py purge-last --confirm # execute
```

**Session selection priority:**
1. Current session from `current-session.json` if it has events.
2. Otherwise, the session with the latest `ended_at` in `sessions.jsonl`.
3. Fallback: the latest `session_id` by timestamp in `events.jsonl`.

**What gets removed:**
- That session's entry from `sessions.jsonl`.
- Events linked only to that session from `events.jsonl`.
- Checkpoints linked only to that session from `checkpoints.jsonl`.
- Open threads that originated solely in that session with no older dependency.

**What is preserved:**
- Older sessions, accepted decisions, older threads.
- All hooks, steering files, scripts, and structure.

**After purge:** Regenerate runtime artifacts and run validation.

## Purge all memory

Clears all memory data but keeps the LTM structure intact.

```bash
<python_cmd> ltm/bin/ltm.py purge-all          # dry-run (default)
<python_cmd> ltm/bin/ltm.py purge-all --confirm # execute
```

**What gets removed:**
- All content from all JSONL files in `ltm/store/`.
- All files in `ltm/runtime/` and `ltm/reports/`.

**What is preserved:**
- `ltm/` directory structure, `config.json`, `manifest.json`, `README.md`, `ltm/bin/ltm.py`.
- All hooks and workspace steering files.

**After purge:** Recreate empty JSONL files and placeholder runtime files. Recreate `ltm/reports/` directory. Write `ltm/reports/purge-all-report-<timestamp>.md`. Run validation.

## Full teardown

Removes all LTM artifacts from the project as if the power was never used.

```bash
<python_cmd> ltm/bin/ltm.py teardown          # dry-run (default)
<python_cmd> ltm/bin/ltm.py teardown --confirm # execute
```

**Process:**
1. Read `ltm/manifest.json`.
2. Validate manifest: `created_by` must be `ltm-power`. All paths must be under `ltm/`, `.kiro/hooks/ltm-*`, or `.kiro/steering/ltm-*`. Reject absolute paths or `..` traversal.
3. Remove all listed files and directories.
4. Remove the `# --- ltm-power ---` block from `.gitignore`. If delimiters not found, list lines for manual removal.
5. Remove listed hook files from `.kiro/hooks/`.
6. Remove listed steering files from `.kiro/steering/`. If user modified a steering file (check for `ltm-power` marker), warn before deleting.
7. Do NOT remove `.kiro/hooks/` or `.kiro/steering/` directories themselves.
8. Do NOT modify any file not listed in the manifest.
9. Report what was removed to stdout (compact JSON summary).

**Important:** All destructive commands default to dry-run without `--confirm`. Always show the dry-run output to the user before proceeding.
