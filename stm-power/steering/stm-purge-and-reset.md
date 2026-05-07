# STM Purge and Reset

Two modes of memory removal, from least to most destructive.

## Purge all observations

Clears all observation and cluster data but keeps STM structure intact.

Read `python_cmd` from `stm/config.json`.

```bash
<python_cmd> stm/bin/stm.py purge-all          # dry-run (default)
<python_cmd> stm/bin/stm.py purge-all --confirm # execute
```

**What gets removed:**
- All content from `stm/store/observations.jsonl` and `stm/store/clusters.jsonl`.
- All files in `stm/runtime/`.

**What is preserved:**
- `stm/` directory structure, `config.json`, `manifest.json`, `README.md`, `stm/bin/stm.py`.
- All hooks and workspace steering files.

**After purge:** Recreate empty store files and placeholder runtime files. Run validation.

## Full teardown

Removes all STM artifacts from the project as if the power was never used.

```bash
<python_cmd> stm/bin/stm.py teardown          # dry-run (default)
<python_cmd> stm/bin/stm.py teardown --confirm # execute
```

**Process:**
1. Read `stm/manifest.json`.
2. Validate manifest: `created_by` must be `stm-power`. All paths must be under `stm/`, `.kiro/hooks/stm-*`, or `.kiro/steering/stm-*`. Reject absolute paths or `..` traversal.
3. Remove all listed files and directories.
4. Remove the `# --- stm-power ---` block from `.gitignore`. If delimiters not found, list lines for manual removal.
5. Remove listed hook files from `.kiro/hooks/`.
6. Remove listed steering files from `.kiro/steering/`.
7. Do NOT remove `.kiro/hooks/` or `.kiro/steering/` directories themselves.
8. Do NOT modify any file not listed in the manifest.
9. Report what was removed to stdout (compact JSON summary).

**Important:** All destructive commands default to dry-run without `--confirm`. Always show the dry-run output to the user before proceeding.

## Co-installation with LTM

- "Forget all observations" → STM `purge-all` only.
- "Forget all memory" → both STM `purge-all` AND LTM `purge-all`.
- "Remove all memory" → teardown STM first, then LTM. Order matters because STM reads from LTM at runtime.
