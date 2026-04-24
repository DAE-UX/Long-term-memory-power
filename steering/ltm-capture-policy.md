# LTM Capture Policy

## What to capture

Record only durable, future-useful facts:

- Files touched during agent work
- Meaningful progress milestones
- Accepted decisions and their rationale
- Unresolved items and open threads
- Validation outcomes
- User-confirmed intent

## What NOT to capture

- Hidden chain-of-thought or internal reasoning
- Full prompt transcripts
- Secrets, tokens, or environment credentials
- Giant command outputs
- Redundant save bursts

## Capture granularity

Prefer small factual records over verbose narratives. The system uses:

- **Turn-level events** — automatic, written by the capture hook after each agent turn that changes files.
- **Checkpoints** — explicit, written by the agent when the user asks or after significant progress.
- **Runtime artifacts** — regenerated compact summaries derived from ledger data.

## Deduplication

Content-based idempotency: if the previous event has the same `session_id`, same `files_sample` hash, same `git_status`, and was recorded within 2 seconds, skip it as a duplicate. Do not skip solely because an event was recorded within 2 seconds — distinct consecutive events touching different files must be preserved.

## Secret redaction

Before writing any store record, check for likely secrets:

**Text patterns:** `sk_live_`, `sk_test_`, `AKIA`, `ghp_`, `gho_`, `-----BEGIN`, `Bearer `, base64-like strings of 40+ characters in sensitive contexts.

**Variable names:** `password`, `secret`, `token`, `api_key`, `private_key`, `access_key`.

**Action:** Replace matched values with `[REDACTED]`. Add `"redacted": true` to the event record.

Redaction is best-effort in v1. Do not rely on it as a security boundary.

## Sensitive path filtering

File paths matching `sensitive_path_patterns` in `ltm/config.json` are omitted from `files_sample` in events and from `changed_files` in checkpoints. Sensitive paths are omitted entirely, not stored as `[REDACTED_PATH]`.

Default sensitive patterns: `.env*`, `secrets/**`, `credentials/**`, `.aws/**`, `.ssh/**`, `.kube/**`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.crt`, `*.cert`, `id_rsa`, `id_ed25519`, `kubeconfig`.

## Session boundaries

A new session starts when a capture event occurs and the previous session's last event exceeds `session_timeout_minutes` (default 60 minutes, configurable in `ltm/config.json`).

Session IDs follow the format: `sess_YYYY_MM_DD_NN` (date + sequence number).

The capture script manages session rollover automatically. When rolling over, it writes a structural session summary to `sessions.jsonl` with the event count. The agent enriches this with a semantic summary during the next checkpoint.

## Checkpoint rules

The agent writes checkpoints when:
- The user explicitly asks ("Save a checkpoint").
- The agent judges significant progress has been made.

Checkpoints include a `decisions` array field. Decisions are folded into checkpoints in v1 — there is no separate `decisions.jsonl` ledger.

Checkpoint `changed_files` must be filtered through `sensitive_path_patterns` before writing.

All checkpoint writes go through `ltm.py checkpoint --from-json` for validation. Direct ledger writes are fallback-only when `ltm.py` is unavailable.
