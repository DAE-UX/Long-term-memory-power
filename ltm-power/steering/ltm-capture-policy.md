# LTM Capture Policy

## What to capture

Durable, future-useful facts only:
- Files touched during agent work
- Progress milestones
- Accepted decisions with rationale
- Unresolved items and open threads
- Validation outcomes
- User-confirmed intent

## What NOT to capture

- Chain-of-thought or internal reasoning
- Full prompt transcripts
- Secrets, tokens, credentials
- Large command outputs
- Redundant save bursts

## Capture types

- **Turn events** — automatic via hook after each agent turn that changes files.
- **Checkpoints** — explicit, written when user asks or after significant progress.
- **Runtime artifacts** — regenerated summaries derived from ledger data.

## Deduplication

Content-based: skip if previous event has same `session_id`, same `files_sample` hash, same `git_status`, and was recorded within 2 seconds. Do not skip solely on time — distinct events with different files must be preserved.

## Secret redaction

**Text patterns:** `sk_live_`, `sk_test_`, `AKIA`, `ghp_`, `gho_`, `-----BEGIN`, `Bearer `, base64 strings 40+ chars.

**Variable names:** `password`, `secret`, `token`, `api_key`, `private_key`, `access_key`.

**Action:** Replace with `[REDACTED]`, set `"redacted": true`. Best-effort in v1.

## Sensitive path filtering

Paths matching `sensitive_path_patterns` in `ltm/config.json` are omitted from `files_sample` and checkpoint `changed_files`. Omitted entirely, not redacted.

Defaults: `.env*`, `secrets/**`, `credentials/**`, `.aws/**`, `.ssh/**`, `.kube/**`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.crt`, `*.cert`, `id_rsa`, `id_ed25519`, `kubeconfig`.

## Session boundaries

New session starts when previous session's last event exceeds `session_timeout_minutes` (default 60, configurable). IDs: `sess_YYYY_MM_DD_NN`.

Rollover writes a structural summary to `sessions.jsonl` (event count only). Agent enriches with semantic summary during next checkpoint.

## Checkpoint rules

Agent writes checkpoints when user asks or after significant progress. Checkpoints include a `decisions` array (no separate ledger in v1).

**REQUIRED:** Filter `changed_files` through `sensitive_path_patterns`. Route all writes through `ltm.py checkpoint --from-json`. Direct writes only as degraded fallback.
