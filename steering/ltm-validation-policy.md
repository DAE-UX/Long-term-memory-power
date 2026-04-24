# LTM Validation Policy

## Validation categories

### Bootstrap validation
- `ltm/` exists with all expected subdirectories.
- All expected files exist (`config.json`, `manifest.json`, `README.md`, ledger files, runtime files).
- `config.json` has `created_by: ltm-power`.
- Manifest is complete and contains only relative paths.

### Ledger integrity
- Every line in each JSONL file is valid JSON.
- Every record has required fields for its type.
- Timestamps are valid ISO-8601.
- No truncated trailing lines.

### Capture validation
- Events exist for recent sessions.
- Deduplication is working (no identical events within 2 seconds with same content hash).

### Recall validation
- Runtime artifacts can be generated from ledger data.
- `ltm.py files` and `ltm.py sessions` return results.
- Budget caps are respected (runtime artifacts within size limits).

### Purge validation
- Last-session purge removes only the latest session.
- Full purge resets all content while preserving structure.
- Scaffolding survives both purge modes.

### Repair validation
- Missing runtime files can be recreated.
- Partial missing files are repaired.
- Store data is preserved during repair.

### Redaction validation
- Known secret patterns are masked in test fixtures.
- Sensitive file paths are excluded from `files_sample` and checkpoint `changed_files`.

### Health check dimensions (8 total)

1. **Store freshness:** GREEN <4h, YELLOW <24h, STALE >24h (not RED — inactivity is not failure).
2. **Ledger integrity:** GREEN = all pass, RED = any fail.
3. **Runtime freshness:** GREEN <1h, YELLOW <4h, RED >4h.
4. **Budget compliance:** GREEN = under preferred, YELLOW = under hard ceiling, RED = over.
5. **Capture coverage:** GREEN = events in last hour, YELLOW = events in last 4h, STALE = no recent events.
6. **Semantic coverage:** GREEN = checkpoint <8h, YELLOW <24h, WARN = >24h or 3+ structural sessions.
7. **End-to-end probe:** GREEN = read store + generate query works, RED = fails.
8. **Hook status:** `verified` = GREEN, `file_created` = YELLOW, `manual_setup_required` = YELLOW, `failed` = RED, `disabled` = GREEN only in degraded mode.

**Overall states:** `healthy-active`, `healthy-stale`, `degraded`, `broken`.

A normal install with `hook_status: file_created` reports `degraded` for automatic capture until the hook is verified by an actual event.

## Acceptance scenario (manual walkthrough)

1. Bootstrap in a fresh project. Confirm structure.
2. Make meaningful edits. Confirm capture events written.
3. Create a checkpoint. Confirm checkpoint in ledger.
4. Simulate new session (set `last_event_at` in `current-session.json` to older than `session_timeout_minutes`, or temporarily set `session_timeout_minutes` to 1). Ask to resume. Confirm recall works.
5. Purge last session. Confirm older memory preserved.
6. Purge all memory. Confirm healthy reset.
7. Delete a runtime file. Run repair. Confirm restored.
8. Run full teardown. Confirm no orphaned files.

## Command behavior rules

- `health`: read-only. Reports missing files but does NOT create them.
- `validate`: read-only. Fails on missing required files. Does NOT create or fix anything.
- `repair`: may create missing files and directories. May remove only incomplete trailing lines. Must preserve complete JSON records even if they fail schema validation.
- `selftest`: creates temporary fixtures in a temp directory. Verifies implementation behavior independent of current project data.
