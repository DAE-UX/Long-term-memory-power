# STM Script Source

This file contains the canonical `stm.py` source code. It is loaded only when the agent needs to write the script to a project — during bootstrap or repair.

**Expected SHA-256:** `729079db337a1526211c977c0589d00759202a2006c5d22492e38ebc775aa620`

Write the contents of the fenced code block below exactly to `stm/bin/stm.py`. After writing, verify the SHA-256 hash matches the expected value above. If it does not match, report a generation error.

```python
#!/usr/bin/env python3
"""stm.py — STM Introspection CLI for Kiro.

Single-file, stdlib-only observation capture, compression, distillation,
and graduation tool. Reads/writes JSONL stores under stm/store/ and
runtime artifacts under stm/runtime/.
"""
import argparse, datetime, hashlib, json, os, re, sys, tempfile, unittest
from pathlib import Path

VERSION = "1.0.13"
ROOT = Path("stm")
STORE = ROOT / "store"
RUNTIME = ROOT / "runtime"
BIN = ROOT / "bin"
CONFIG_PATH = ROOT / "config.json"
MANIFEST_PATH = ROOT / "manifest.json"
OBSERVATIONS = STORE / "observations.jsonl"
CLUSTERS = STORE / "clusters.jsonl"
STATUS_PATH = RUNTIME / "status.json"
CUR_SESSION = RUNTIME / "current-session.json"
MERGE_CANDIDATES = RUNTIME / "merge-candidates.json"
ERRORS_PATH = RUNTIME / "errors.jsonl"
LTM_SESSION = Path("ltm/runtime/current-session.json")
LTM_CONFIG = Path("ltm/config.json")

VALID_STANCES = {"supports", "contradicts", "supersedes", "neutral"}
VALID_CONFIDENCE = {"low", "medium", "high"}
VALID_SOURCES = {"agent:inline", "agent:reflect", "agent:compress",
                 "user:feedback", "agent:followup"}
VALID_OUTCOMES = {"confirmed", "mixed", "failed"}

EXIT_OK = 0
EXIT_DEGRADED = 2
EXIT_INVALID = 3
EXIT_IO_ERROR = 4
EXIT_USAGE = 64

_PRIVACY_PATTERNS = [
    re.compile(r'sk_live_\S+'),
    re.compile(r'sk_test_\S+'),
    re.compile(r'AKIA[A-Z0-9]{16,}'),
    re.compile(r'ghp_[A-Za-z0-9]+'),
    re.compile(r'gho_[A-Za-z0-9]+'),
    re.compile(r'xox[bpras]-\S+'),
    re.compile(r'-----BEGIN\s'),
    re.compile(r'Bearer\s+[A-Za-z0-9._~+/=-]{20,}'),
    re.compile(r'(?i)(password|passwd|pwd|secret|api_key|apikey|access_key|private_key|auth_token)\s*[:=]\s*\S+'),
    re.compile(r'://[^:]+:[^@]+@'),
    re.compile(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'),
    re.compile(r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b'),
    re.compile(r'\b\d{3}-\d{2}-\d{4}\b'),
    re.compile(r'\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b'),
]


# ── helpers ──────────────────────────────────────────────────────────────────

def _now():
    return datetime.datetime.now(datetime.timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")

def _load_config():
    try:
        return json.loads(CONFIG_PATH.read_text())
    except Exception:
        return {}

def _save_config(config):
    _atomic_write_text(CONFIG_PATH, json.dumps(config, indent=2))

def _read_jsonl(path, skip_bad=True):
    records = []
    if not path.exists():
        return records
    for line in path.read_text().splitlines():
        line = line.strip()
        if not line:
            continue
        try:
            records.append(json.loads(line))
        except json.JSONDecodeError:
            if not skip_bad:
                raise
    return records

def _append_jsonl(path, record):
    path.parent.mkdir(parents=True, exist_ok=True)
    with open(path, "a") as f:
        f.write(json.dumps(record, separators=(",", ":")) + "\n")

def _write_jsonl(path, records):
    path.parent.mkdir(parents=True, exist_ok=True)
    tmp = path.with_suffix(path.suffix + ".tmp")
    with open(tmp, "w") as f:
        for r in records:
            f.write(json.dumps(r, separators=(",", ":")) + "\n")
    os.replace(tmp, path)

def _atomic_write_text(path, text):
    path.parent.mkdir(parents=True, exist_ok=True)
    tmp = path.with_suffix(path.suffix + ".tmp")
    tmp.write_text(text, encoding="utf-8")
    os.replace(tmp, path)

def _next_id(path, prefix):
    records = _read_jsonl(path)
    max_n = 0
    for r in records:
        rid = r.get("id", "") or r.get("cluster_id", "")
        if rid.startswith(prefix):
            try:
                max_n = max(max_n, int(rid.split("_")[-1]))
            except ValueError:
                pass
    return f"{prefix}{max_n + 1:06d}"

def _obs_fingerprint(topic, stance, claim):
    data = json.dumps({"t": topic, "s": stance, "c": claim}, sort_keys=True)
    return hashlib.sha256(data.encode()).hexdigest()[:16]

def _out(data):
    sys.stdout.write(json.dumps(data, indent=None, separators=(",", ":")) + "\n")

def _err(msg):
    sys.stderr.write(f"stm: {msg}\n")

def _log_error(error_type, detail):
    try:
        _append_jsonl(ERRORS_PATH, {"ts": _now(), "type": error_type, "detail": detail})
    except Exception:
        pass

def _update_status(**kwargs):
    try:
        status = json.loads(STATUS_PATH.read_text()) if STATUS_PATH.exists() else {}
    except Exception:
        status = {}
    status.update(kwargs)
    _atomic_write_text(STATUS_PATH, json.dumps(status, indent=2))

def _resolve_session(config):
    ltm_available = config.get("ltm_available", False)
    if ltm_available and LTM_SESSION.exists():
        try:
            return json.loads(LTM_SESSION.read_text()).get("session_id", "unknown")
        except Exception:
            pass
    ltm_now = LTM_CONFIG.exists()
    if ltm_now != ltm_available:
        config["ltm_available"] = ltm_now
        _save_config(config)
        if ltm_now and LTM_SESSION.exists():
            try:
                return json.loads(LTM_SESSION.read_text()).get("session_id", "unknown")
            except Exception:
                pass
    timeout_min = config.get("session_timeout_minutes", 60)
    try:
        session = json.loads(CUR_SESSION.read_text())
    except Exception:
        session = _new_session()
        _atomic_write_text(CUR_SESSION, json.dumps(session, indent=2))
        return session["session_id"]
    last = session.get("last_event_at")
    if last:
        try:
            last_dt = datetime.datetime.fromisoformat(last.replace("Z", "+00:00"))
            if (datetime.datetime.now(datetime.timezone.utc) - last_dt).total_seconds() > timeout_min * 60:
                session = _new_session()
                _atomic_write_text(CUR_SESSION, json.dumps(session, indent=2))
        except Exception:
            pass
    return session["session_id"]

def _new_session():
    now = datetime.datetime.now(datetime.timezone.utc)
    date_prefix = f"stm_sess_{now.strftime('%Y_%m_%d')}_"
    max_seq = 0
    # Check current session file first (cheap)
    if CUR_SESSION.exists():
        try:
            sid = json.loads(CUR_SESSION.read_text()).get("session_id", "")
            if sid.startswith(date_prefix):
                try:
                    max_seq = max(max_seq, int(sid.split("_")[-1]))
                except ValueError:
                    pass
        except Exception:
            pass
    # Scan only last 50 observations for today's sessions (bounded cost)
    obs = _read_jsonl(OBSERVATIONS)
    for o in obs[-50:]:
        sid = o.get("session_id", "")
        if sid.startswith(date_prefix):
            try:
                max_seq = max(max_seq, int(sid.split("_")[-1]))
            except ValueError:
                pass
    return {
        "session_id": f"{date_prefix}{max_seq + 1:02d}",
        "started_at": _now(),
        "last_event_at": None,
        "event_count": 0,
    }

def _update_session_event(session_id):
    if not CUR_SESSION.exists():
        return
    try:
        session = json.loads(CUR_SESSION.read_text())
        if session.get("session_id") == session_id:
            session["last_event_at"] = _now()
            session["event_count"] = session.get("event_count", 0) + 1
            _atomic_write_text(CUR_SESSION, json.dumps(session, indent=2))
    except Exception:
        pass

def _inline_compact():
    """Remove already-compressed observations from the store file to reclaim space."""
    obs = _read_jsonl(OBSERVATIONS)
    keep = [o for o in obs if not o.get("compressed", False)]
    removed = len(obs) - len(keep)
    if removed > 0:
        _write_jsonl(OBSERVATIONS, keep)
        _update_status(compression_needed=False)
        _err(f"inline compaction: removed {removed} compressed observations from store")


# ── record ───────────────────────────────────────────────────────────────────

def cmd_record(args):
    config = _load_config()

    # Enhancement 7: master kill switch
    if not config.get("capture_enabled", True):
        _out({"status": "disabled"})
        return

    errors = []
    if not args.scope: errors.append("--scope is required")
    if not args.topic: errors.append("--topic is required")
    if args.stance not in VALID_STANCES:
        errors.append(f"--stance must be one of: {', '.join(sorted(VALID_STANCES))}")
    if not args.claim: errors.append("--claim is required")
    if not args.evidence: errors.append("--evidence is required")
    if args.confidence not in VALID_CONFIDENCE:
        errors.append(f"--confidence must be one of: {', '.join(sorted(VALID_CONFIDENCE))}")
    if args.source not in VALID_SOURCES:
        errors.append(f"--source must be one of: {', '.join(sorted(VALID_SOURCES))}")
    if args.stance == "supersedes" and not args.supersedes:
        errors.append("--supersedes is required when stance is 'supersedes'")
    if args.applies_to and not args.outcome:
        errors.append("--outcome is required when --applies-to is set")
    if args.outcome and args.outcome not in VALID_OUTCOMES:
        errors.append(f"--outcome must be one of: {', '.join(sorted(VALID_OUTCOMES))}")
    if args.topic and not re.match(r'^[a-z0-9]+(-[a-z0-9]+)*$', args.topic):
        errors.append("--topic must be kebab-case")
    if errors:
        for e in errors: _err(e)
        _log_error("validation", "; ".join(errors))
        sys.exit(EXIT_INVALID)

    # Enhancement 7: confidence gate
    min_conf = config.get("min_capture_confidence", "low")
    conf_order = {"low": 0, "medium": 1, "high": 2}
    if conf_order.get(args.confidence, 0) < conf_order.get(min_conf, 0):
        _out({"status": "below_threshold", "confidence": args.confidence, "minimum": min_conf})
        return

    if args.supersedes:
        obs = _read_jsonl(OBSERVATIONS)
        target = [o for o in obs if o.get("id") == args.supersedes]
        if not target:
            _err(f"supersedes target {args.supersedes} not found")
            sys.exit(EXIT_INVALID)
        if target[0].get("topic") != args.topic or target[0].get("scope") != args.scope:
            _err("supersedes target must share same topic and scope")
            sys.exit(EXIT_INVALID)

    if args.applies_to:
        clusters = _read_jsonl(CLUSTERS)
        target = [c for c in clusters if c.get("cluster_id") == args.applies_to]
        if not target:
            _err(f"applies_to target {args.applies_to} not found")
            sys.exit(EXIT_INVALID)
        if target[0].get("status") != "graduated":
            _err("applies_to target must be a graduated cluster")
            sys.exit(EXIT_INVALID)

    cue = args.cue
    if cue and len(cue.split()) > 10:
        _err(f"cue truncated to 10 words (was {len(cue.split())})")
        cue = " ".join(cue.split()[:10])

    session_id = _resolve_session(config)

    # Enhancement 7: session cap
    max_per_session = config.get("max_observations_per_session", 50)
    session_obs = sum(1 for o in _read_jsonl(OBSERVATIONS)
                      if o.get("session_id") == session_id and not o.get("compressed"))
    if session_obs >= max_per_session:
        _out({"status": "capped", "session_id": session_id, "count": session_obs})
        return

    fp = _obs_fingerprint(args.topic, args.stance, args.claim)
    for o in _read_jsonl(OBSERVATIONS):
        if o.get("session_id") == session_id and not o.get("compressed", False):
            if _obs_fingerprint(o.get("topic",""), o.get("stance",""), o.get("claim","")) == fp:
                _out({"status": "duplicate", "existing_id": o.get("id")})
                return

    scopes = config.get("scopes", [])
    scope_names = [s if isinstance(s, str) else s.get("name","") for s in scopes]
    if args.scope not in scope_names:
        if config.get("scope_discovery", "auto") == "auto":
            scopes.append(args.scope)
            config["scopes"] = scopes
            _save_config(config)
        else:
            _err(f"scope '{args.scope}' not in config")
            sys.exit(EXIT_INVALID)

    obs_id = _next_id(OBSERVATIONS, "obs_")
    compressed_from = [x.strip() for x in args.compressed_from.split(",")] if args.compressed_from else None

    observation = {
        "id": obs_id, "ts": _now(), "session_id": session_id,
        "scope": args.scope, "topic": args.topic, "stance": args.stance,
        "claim": args.claim, "evidence": args.evidence, "cue": cue,
        "confidence": args.confidence, "source": args.source,
        "tags": [t.strip() for t in args.tags.split(",")] if args.tags else [],
        "applies_to": args.applies_to, "outcome": args.outcome,
        "supersedes": args.supersedes, "invalidated": False,
        "invalidated_by": None, "compressed": False,
        "compressed_from": compressed_from, "exemplar": False,
    }

    # Enhancement 1: privacy redaction
    for pat in _PRIVACY_PATTERNS:
        observation["claim"] = pat.sub("[REDACTED]", observation["claim"])
        observation["evidence"] = pat.sub("[REDACTED]", observation["evidence"])
    for pat_str in config.get("privacy_patterns", []):
        try:
            compiled = re.compile(pat_str)
            observation["claim"] = compiled.sub("[REDACTED]", observation["claim"])
            observation["evidence"] = compiled.sub("[REDACTED]", observation["evidence"])
        except re.error:
            _log_error("privacy_pattern", f"invalid regex: {pat_str}")
    # Skip if claim is entirely redacted
    claim_stripped = re.sub(r'\[REDACTED\]', '', observation["claim"]).strip()
    if not claim_stripped:
        _out({"status": "redacted", "reason": "claim entirely private"})
        return

    _append_jsonl(OBSERVATIONS, observation)
    _update_session_event(session_id)

    if args.supersedes:
        obs_all = _read_jsonl(OBSERVATIONS)
        for i, o in enumerate(obs_all):
            if o.get("id") == args.supersedes:
                obs_all[i]["invalidated"] = True
                obs_all[i]["invalidated_by"] = obs_id
                break
        _write_jsonl(OBSERVATIONS, obs_all)

    if args.applies_to and args.outcome:
        clusters = _read_jsonl(CLUSTERS)
        for i, c in enumerate(clusters):
            if c.get("cluster_id") == args.applies_to:
                clusters[i]["application_count"] = c.get("application_count", 0) + 1
                followups = [o for o in _read_jsonl(OBSERVATIONS)
                             if o.get("applies_to") == args.applies_to and o.get("outcome")]
                confirmed = sum(1 for o in followups if o["outcome"] == "confirmed")
                clusters[i]["application_success_rate"] = round(confirmed / len(followups), 2) if followups else None
                break
        _write_jsonl(CLUSTERS, clusters)

    all_obs = _read_jsonl(OBSERVATIONS)
    _update_status(
        observation_count=len(all_obs),
        summary_count=sum(1 for o in all_obs if o.get("compressed_from")),
        compressed_count=sum(1 for o in all_obs if o.get("compressed", False)),
    )
    try:
        sz = OBSERVATIONS.stat().st_size if OBSERVATIONS.exists() else 0
        hard_limit = config.get("compaction_hard_bytes", 5242880)
        warn_limit = config.get("compaction_warn_bytes", 1048576)
        if sz > hard_limit:
            _inline_compact()
        elif sz > warn_limit:
            _update_status(compression_needed=True)
    except OSError:
        pass

    _out({"status": "recorded", "id": obs_id, "session_id": session_id})


# ── topics / recall / status ─────────────────────────────────────────────────

def cmd_topics(args):
    obs = _read_jsonl(OBSERVATIONS)
    active = [o for o in obs if not o.get("compressed", False)]
    topics = {}
    for o in active:
        key = f"{o.get('scope','?')}/{o.get('topic','?')}"
        if key not in topics:
            topics[key] = {"scope": o.get("scope"), "topic": o.get("topic"), "count": 0, "stances": {}}
        topics[key]["count"] += 1
        stance = o.get("stance", "?")
        topics[key]["stances"][stance] = topics[key]["stances"].get(stance, 0) + 1
    _out(sorted(topics.values(), key=lambda x: -x["count"]))

def cmd_recall(args):
    limit = min(args.limit or 3, 5)
    clusters = _read_jsonl(CLUSTERS)
    obs = _read_jsonl(OBSERVATIONS)
    results = []
    graduated = [c for c in clusters if c.get("status") == "graduated"]
    if args.scope:
        graduated = [c for c in graduated if c.get("scope") == args.scope]
    if args.cue:
        cue_words = set(args.cue.lower().split())
        for c in graduated:
            pattern_words = set((c.get("cue_pattern") or "").lower().split())
            c["_score"] = len(cue_words & pattern_words)
        graduated.sort(key=lambda x: -x.get("_score", 0))
    for c in graduated[:limit]:
        exemplar_summary = None
        for eid in (c.get("exemplar_ids") or []):
            e = next((o for o in obs if o.get("id") == eid), None)
            if e:
                exemplar_summary = f"{e.get('claim','')} (cue: {e.get('cue','none')})"
                break
        results.append({
            "cluster_id": c.get("cluster_id"), "consensus_claim": c.get("consensus_claim"),
            "cue_pattern": c.get("cue_pattern"),
            "application_success_rate": c.get("application_success_rate"),
            "exemplar_summary": exemplar_summary,
        })
    if len(results) < limit:
        high_conf = [o for o in obs if not o.get("compressed") and o.get("confidence") == "high"
                     and not o.get("invalidated")]
        if args.scope:
            high_conf = [o for o in high_conf if o.get("scope") == args.scope]
        if args.cue:
            cue_words = set(args.cue.lower().split())
            for o in high_conf:
                o["_score"] = len(cue_words & set((o.get("cue") or "").lower().split()))
            high_conf.sort(key=lambda x: -x.get("_score", 0))
        for o in high_conf[:limit - len(results)]:
            results.append({"observation_id": o.get("id"), "claim": o.get("claim"),
                            "cue": o.get("cue"), "confidence": o.get("confidence"),
                            "stance": o.get("stance")})
    _out(results)

def cmd_status(args):
    obs = _read_jsonl(OBSERVATIONS)
    clusters = _read_jsonl(CLUSTERS)
    config = _load_config()
    active = [o for o in obs if not o.get("compressed", False)]
    pending = [c for c in clusters if c.get("status") == "pending"]
    graduated = [c for c in clusters if c.get("status") == "graduated"]
    try:
        sz = OBSERVATIONS.stat().st_size if OBSERVATIONS.exists() else 0
    except OSError:
        sz = 0
    status = {
        "observation_count": len(obs), "active_observations": len(active),
        "compressed_observations": sum(1 for o in obs if o.get("compressed")),
        "summary_observations": sum(1 for o in obs if o.get("compressed_from")),
        "cluster_count": len(clusters), "pending_graduations": len(pending),
        "graduated_count": len(graduated), "store_bytes": sz,
        "compression_needed": sz > config.get("compaction_warn_bytes", 1048576),
        "ltm_available": config.get("ltm_available", False),
        "scope_count": len(config.get("scopes", [])),
    }
    _update_status(**status)
    _out(status)


# ── search ───────────────────────────────────────────────────────────────────

def cmd_search(args):
    term = args.term.lower().replace("-", "_").replace(" ", "_")
    limit = min(args.limit or 5, 20)
    max_bytes = 4096
    obs = _read_jsonl(OBSERVATIONS)
    if args.days:
        cutoff = (datetime.datetime.now(datetime.timezone.utc)
                  - datetime.timedelta(days=args.days)).strftime("%Y-%m-%dT%H:%M:%SZ")
        obs = [o for o in obs if o.get("ts", "") >= cutoff]
    if args.scope:
        obs = [o for o in obs if o.get("scope") == args.scope]
    results = []
    total_bytes = 0
    for o in reversed(obs):
        if len(results) >= limit:
            break
        text = json.dumps(o).lower().replace("-", "_").replace(" ", "_")
        if term in text:
            entry = {
                "id": o.get("id"), "ts": o.get("ts"),
                "scope": o.get("scope"), "topic": o.get("topic"),
                "stance": o.get("stance"), "claim": o.get("claim", "")[:160],
                "confidence": o.get("confidence"),
                "compressed": o.get("compressed", False),
            }
            entry_bytes = len(json.dumps(entry))
            if total_bytes + entry_bytes > max_bytes:
                break
            results.append(entry)
            total_bytes += entry_bytes
    _out(results)


# ── compress ─────────────────────────────────────────────────────────────────

def cmd_compress(args):
    obs = _read_jsonl(OBSERVATIONS)
    active = [o for o in obs if not o.get("compressed") and not o.get("compressed_from")]
    if args.identify:
        groups = {}
        for o in active:
            key = (o.get("scope",""), o.get("topic",""))
            groups.setdefault(key, []).append(o)
        candidates = []
        for (scope, topic), group_obs in groups.items():
            if len(group_obs) >= 3:
                scored = []
                for o in group_obs:
                    score = (2 if o.get("cue") else 0) + (3 if o.get("source") == "user:feedback" else 0) + (1 if o.get("confidence") == "high" else 0)
                    scored.append((score, o))
                scored.sort(key=lambda x: -x[0])
                candidates.append({"scope": scope, "topic": topic, "count": len(group_obs),
                                   "ids": [o["id"] for o in group_obs],
                                   "proposed_exemplars": [s[1]["id"] for s in scored[:2]]})
        _out({"candidates": candidates})
        return
    if args.execute:
        if not args.ids:
            _err("--ids required for --execute"); sys.exit(EXIT_USAGE)
        ids_to_compress = [x.strip() for x in args.ids.split(",")]
        exemplar_ids = [x.strip() for x in args.exemplars.split(",")] if args.exemplars else []
        all_obs = _read_jsonl(OBSERVATIONS)
        for i, o in enumerate(all_obs):
            if o.get("id") in ids_to_compress:
                all_obs[i]["exemplar" if o["id"] in exemplar_ids else "compressed"] = True
        _write_jsonl(OBSERVATIONS, all_obs)
        _out({"compressed": len(ids_to_compress) - len(exemplar_ids), "exemplars_preserved": len(exemplar_ids)})
        return
    _err("use --identify or --execute"); sys.exit(EXIT_USAGE)

# ── distill ──────────────────────────────────────────────────────────────────

def cmd_distill(args):
    config = _load_config()
    obs = _read_jsonl(OBSERVATIONS)
    active = [o for o in obs if not o.get("compressed", False)]
    groups = {}
    for o in active:
        key = (o.get("scope",""), o.get("topic",""))
        groups.setdefault(key, []).append(o)
    clusters = _read_jsonl(CLUSTERS)
    cluster_map = {c["cluster_id"]: c for c in clusters}
    existing_topics = {(c["scope"], c["topic"]): c["cluster_id"] for c in clusters}
    new_clusters, updated = [], []
    merge_candidates = []
    threshold = config.get("consensus_threshold", 3)
    score_min = config.get("consensus_score_minimum", 0.6)
    for (scope, topic), group_obs in groups.items():
        obs_ids = [o["id"] for o in group_obs]
        supports = sum(1 for o in group_obs if o.get("stance") == "supports")
        contradicts = sum(1 for o in group_obs if o.get("stance") == "contradicts")
        total = supports + contradicts
        score = (supports - contradicts) / total if total > 0 else 0.0
        existing_cid = existing_topics.get((scope, topic))
        if existing_cid:
            c = cluster_map[existing_cid]
            c.update({"observation_ids": obs_ids, "supports_count": supports,
                      "contradicts_count": contradicts, "consensus_score": round(score, 3),
                      "last_updated": _now()})
            if c.get("status") == "graduated" and score < 0:
                c["status"] = "invalidated"
            elif c.get("status") != "graduated":
                has_summaries = any(o.get("compressed_from") for o in group_obs)
                c["status"] = "pending" if total >= threshold and score >= score_min and has_summaries else "accumulating"
            updated.append(c)
        else:
            cid = _next_id(CLUSTERS, "clst_")
            has_summaries = any(o.get("compressed_from") for o in group_obs)
            status = "pending" if total >= threshold and score >= score_min and has_summaries else "accumulating"
            new_clusters.append({
                "cluster_id": cid, "topic": topic, "scope": scope,
                "observation_ids": obs_ids, "supports_count": supports,
                "contradicts_count": contradicts, "consensus_score": round(score, 3),
                "consensus_claim": None, "recommendation": None, "cue_pattern": None,
                "status": status, "last_updated": _now(), "graduated": False,
                "graduated_to": None, "integrated_files": None, "exemplar_ids": None,
                "application_count": 0, "application_success_rate": None,
            })
            existing_topics[(scope, topic)] = cid
    # Merge candidate detection
    stop_words = {"the","a","an","is","are","was","were","be","been","being","have","has",
                  "had","do","does","did","will","would","could","should","may","might",
                  "shall","can","to","of","in","for","on","with","at","by","from","and",
                  "or","not","no","but"}
    min_shared = config.get("merge_candidate_min_shared_words", 3)
    all_topics = list(existing_topics.keys())
    for i, (s1, t1) in enumerate(all_topics):
        for j, (s2, t2) in enumerate(all_topics):
            if j <= i or s1 != s2: continue
            shared = (set(t1.split("-")) - stop_words) & (set(t2.split("-")) - stop_words)
            if len(shared) >= min_shared:
                merge_candidates.append({"topic_a": t1, "topic_b": t2, "scope": s1,
                                         "shared_words": list(shared),
                                         "cluster_a": existing_topics[(s1,t1)],
                                         "cluster_b": existing_topics[(s2,t2)]})
    _atomic_write_text(MERGE_CANDIDATES, json.dumps(merge_candidates, indent=2))
    updated_ids = {c["cluster_id"] for c in updated}
    final = [next(u for u in updated if u["cluster_id"] == c["cluster_id"]) if c["cluster_id"] in updated_ids else c for c in clusters] + new_clusters
    _write_jsonl(CLUSTERS, final)
    _update_status(cluster_count=len(final),
                   pending_graduations=sum(1 for c in final if c.get("status") == "pending"),
                   last_distill=_now())
    _out({"clusters_total": len(final), "new_clusters": len(new_clusters),
          "updated_clusters": len(updated),
          "pending_graduations": sum(1 for c in final if c.get("status") == "pending"),
          "invalidated": sum(1 for c in final if c.get("status") == "invalidated"),
          "merge_candidates": len(merge_candidates)})


# ── cluster management ───────────────────────────────────────────────────────

def cmd_pending_clusters(args):
    _out([c for c in _read_jsonl(CLUSTERS) if c.get("status") == "pending"])

def cmd_mark_graduated(args):
    clusters = _read_jsonl(CLUSTERS)
    for i, c in enumerate(clusters):
        if c.get("cluster_id") == args.cluster:
            clusters[i].update({"status": "graduated", "graduated": True,
                                "graduated_to": args.path, "last_updated": _now()})
            _write_jsonl(CLUSTERS, clusters)
            _out({"status": "graduated", "cluster_id": args.cluster, "path": args.path})
            return
    _err(f"cluster {args.cluster} not found"); sys.exit(EXIT_INVALID)

def cmd_mark_integrated(args):
    clusters = _read_jsonl(CLUSTERS)
    for i, c in enumerate(clusters):
        if c.get("cluster_id") == args.cluster:
            clusters[i].update({"integrated_files": args.files, "last_updated": _now()})
            _write_jsonl(CLUSTERS, clusters)
            _out({"status": "integrated", "cluster_id": args.cluster, "files": args.files})
            return
    _err(f"cluster {args.cluster} not found"); sys.exit(EXIT_INVALID)

# ── health / validate / repair ───────────────────────────────────────────────

def cmd_health(args):
    config = _load_config()
    h = {"checked_at": _now()}
    # Store integrity
    integrity = "pass"
    for p in [OBSERVATIONS, CLUSTERS]:
        if p.exists():
            for line in p.read_text().splitlines():
                line = line.strip()
                if line:
                    try: json.loads(line)
                    except json.JSONDecodeError: integrity = "fail"; break
    h["store_integrity"] = integrity
    # Schema spot check
    obs = _read_jsonl(OBSERVATIONS)
    schema_issues = sum(1 for o in obs[:50]
                        if not all(k in o for k in ("id","scope","topic","stance","claim","evidence"))
                        or (o.get("stance") and o["stance"] not in VALID_STANCES))
    h["schema_compliance"] = "pass" if schema_issues == 0 else "fail"
    # Referential integrity
    obs_ids = {o.get("id") for o in obs}
    ref_issues = sum(1 for o in obs if o.get("supersedes") and o["supersedes"] not in obs_ids)
    for o in obs:
        if o.get("compressed_from"):
            for cid in o["compressed_from"]:
                if cid not in obs_ids:
                    ref_issues += 1
    clusters = _read_jsonl(CLUSTERS)
    for c in clusters:
        for oid in c.get("observation_ids", []):
            if oid not in obs_ids:
                ref_issues += 1
    h["referential_integrity"] = "pass" if ref_issues == 0 else f"fail ({ref_issues} issues)"
    # Session state
    session_ok = True
    if config.get("ltm_available"):
        if LTM_SESSION.exists():
            try: json.loads(LTM_SESSION.read_text()); h["ltm_link"] = "ok"
            except Exception: h["ltm_link"] = "degraded"; session_ok = False
        else:
            h["ltm_link"] = "degraded"
    else:
        h["ltm_link"] = "not_configured"
        if CUR_SESSION.exists():
            try: json.loads(CUR_SESSION.read_text())
            except Exception: session_ok = False
    h["session_state"] = "pass" if session_ok else "degraded"
    # Application tracking
    h["low_success_clusters"] = [c["cluster_id"] for c in clusters
                                  if c.get("application_count",0) >= 3
                                  and c.get("application_success_rate") is not None
                                  and c["application_success_rate"] < 0.5]
    # Cluster state
    bad = [c for c in clusters if c.get("status") == "graduated" and c.get("consensus_score",1) < 0]
    h["cluster_state"] = "pass" if not bad else f"warn ({len(bad)} invalidation candidates)"
    # Overall
    if integrity == "fail" or h["schema_compliance"] == "fail":
        h["overall"] = "broken"
    elif h["session_state"] == "degraded" or "fail" in str(h.get("referential_integrity","")):
        h["overall"] = "degraded"
    else:
        h["overall"] = "healthy"
    _out(h)

def cmd_validate(args):
    issues = []
    for p in [ROOT, STORE, RUNTIME, CONFIG_PATH]:
        if not p.exists(): issues.append(f"missing: {p}")
    for p in [OBSERVATIONS, CLUSTERS]:
        if not p.exists():
            issues.append(f"missing store: {p}")
        else:
            for i, line in enumerate(p.read_text().splitlines()):
                line = line.strip()
                if not line: continue
                try:
                    r = json.loads(line)
                    if p == OBSERVATIONS:
                        for f in ("id","scope","topic","stance","claim","evidence"):
                            if f not in r: issues.append(f"{p}:{i+1} missing '{f}'")
                        if r.get("stance") and r["stance"] not in VALID_STANCES:
                            issues.append(f"{p}:{i+1} invalid stance")
                    if p == CLUSTERS:
                        for f in ("cluster_id","topic","scope"):
                            if f not in r: issues.append(f"{p}:{i+1} missing '{f}'")
                except json.JSONDecodeError:
                    issues.append(f"{p}:{i+1} invalid JSON")
    if CONFIG_PATH.exists():
        try:
            c = json.loads(CONFIG_PATH.read_text())
            if c.get("created_by") != "stm-power": issues.append("config.json: created_by is not stm-power")
        except json.JSONDecodeError:
            issues.append("config.json: invalid JSON")
    _out({"valid": len(issues) == 0, "issues": issues})
    if issues: sys.exit(1)

def cmd_repair(args):
    repaired = []
    for d in [ROOT, STORE, RUNTIME, BIN]:
        if not d.exists(): d.mkdir(parents=True, exist_ok=True); repaired.append(f"created directory: {d}")
    for p in [OBSERVATIONS, CLUSTERS]:
        if not p.exists():
            p.write_text(""); repaired.append(f"created empty: {p}")
        else:
            lines = p.read_text().splitlines()
            clean = []
            for line in lines:
                line = line.strip()
                if not line: continue
                try: json.loads(line); clean.append(line)
                except json.JSONDecodeError: repaired.append(f"removed truncated line from {p}")
            p.write_text("\n".join(clean) + ("\n" if clean else ""))
    if not STATUS_PATH.exists():
        _atomic_write_text(STATUS_PATH, json.dumps({"generated_at": _now()}, indent=2)); repaired.append(f"created: {STATUS_PATH}")
    if not CUR_SESSION.exists():
        _atomic_write_text(CUR_SESSION, json.dumps(_new_session(), indent=2)); repaired.append(f"created: {CUR_SESSION}")
    if not MERGE_CANDIDATES.exists():
        _atomic_write_text(MERGE_CANDIDATES, "[]"); repaired.append(f"created: {MERGE_CANDIDATES}")
    if not ERRORS_PATH.exists():
        ERRORS_PATH.write_text(""); repaired.append(f"created: {ERRORS_PATH}")
    _out({"repaired": repaired})


# ── purge / teardown ─────────────────────────────────────────────────────────

def cmd_purge_all(args):
    plan = {"clears": ["observations.jsonl", "clusters.jsonl", "runtime/*"]}
    if not args.confirm:
        _out({"dry_run": True, "plan": plan}); return
    for p in [OBSERVATIONS, CLUSTERS]: p.write_text("")
    for p in RUNTIME.glob("*"):
        if p.is_file(): p.unlink()
    _atomic_write_text(CUR_SESSION, json.dumps(_new_session(), indent=2))
    _atomic_write_text(STATUS_PATH, json.dumps({"generated_at": _now(), "observation_count": 0}, indent=2))
    _atomic_write_text(MERGE_CANDIDATES, "[]")
    ERRORS_PATH.write_text("")
    _out({"purged": plan})

def cmd_teardown(args):
    if not MANIFEST_PATH.exists():
        _err("manifest.json not found — falling back to prefix-based removal")
        plan = {"fallback": True, "paths": [str(ROOT)]}
        if not args.confirm:
            _out({"dry_run": True, "plan": plan}); return
        import shutil
        if ROOT.exists(): shutil.rmtree(ROOT)
        for p in Path(".kiro/hooks").glob("stm-*.kiro.hook"): p.unlink()
        for p in Path(".kiro/steering").glob("stm-*.md"): p.unlink()
        _out({"torn_down": plan}); return
    manifest = json.loads(MANIFEST_PATH.read_text())
    if manifest.get("created_by") != "stm-power":
        _err("manifest created_by is not stm-power — aborting"); sys.exit(1)
    all_paths = manifest.get("files", []) + manifest.get("hooks", []) + manifest.get("steering", [])
    for p in all_paths:
        if os.path.isabs(p) or ".." in p:
            _err(f"unsafe path in manifest: {p} — aborting"); sys.exit(1)
        if not (p.startswith("stm/") or p.startswith(".kiro/hooks/stm-") or p.startswith(".kiro/steering/stm-")):
            _err(f"path outside allowlist: {p} — aborting"); sys.exit(1)
    plan = {"files": all_paths, "managed_patches": manifest.get("managed_patches", [])}
    if not args.confirm:
        _out({"dry_run": True, "plan": plan}); return
    for p in all_paths:
        pp = Path(p)
        if pp.exists(): pp.unlink()
    for d in sorted([ROOT/"bin", ROOT/"runtime", ROOT/"store", ROOT], key=lambda x: -len(str(x))):
        if d.exists() and d.is_dir() and not any(d.iterdir()): d.rmdir()
    for patch in manifest.get("managed_patches", []):
        if patch.get("file") != ".gitignore": continue
        gi = Path(".gitignore")
        if gi.exists():
            content = gi.read_text()
            start, end = patch.get("start_delimiter",""), patch.get("end_delimiter","")
            if start in content and end in content:
                gi.write_text((content[:content.index(start)] + content[content.index(end)+len(end):]).strip() + "\n")
    _out({"torn_down": plan})

# ── selftest ─────────────────────────────────────────────────────────────────

def cmd_selftest(args):
    class STMSelfTest(unittest.TestCase):
        def setUp(self):
            self.tmpdir = tempfile.mkdtemp()
            self.orig = os.getcwd()
            os.chdir(self.tmpdir)
            for d in [STORE, RUNTIME, BIN]: d.mkdir(parents=True, exist_ok=True)
            for p in [OBSERVATIONS, CLUSTERS]: p.write_text("")
            ERRORS_PATH.write_text("")
            CONFIG_PATH.write_text(json.dumps({
                "created_by": "stm-power", "version": "1.0.0", "schema_version": 1,
                "ltm_available": False, "consensus_threshold": 3,
                "consensus_score_minimum": 0.6, "scopes": ["test-scope"],
                "scope_discovery": "auto", "session_timeout_minutes": 60,
                "compaction_warn_bytes": 1048576, "merge_candidate_min_shared_words": 3,
            }))
            CUR_SESSION.write_text(json.dumps({
                "session_id": "stm_sess_2026_01_01_01", "started_at": "2026-01-01T00:00:00Z",
                "last_event_at": None, "event_count": 0,
            }))
            STATUS_PATH.write_text("{}"); MERGE_CANDIDATES.write_text("[]")
        def tearDown(self):
            os.chdir(self.orig)
            import shutil; shutil.rmtree(self.tmpdir, ignore_errors=True)
        def test_01_jsonl_roundtrip(self):
            _append_jsonl(OBSERVATIONS, {"id": "obs_000001", "ts": "2026-01-01T00:00:00Z",
                "scope": "test-scope", "topic": "test-topic", "stance": "supports",
                "claim": "test", "evidence": "test"})
            self.assertEqual(len(_read_jsonl(OBSERVATIONS)), 1)
        def test_02_rejects_bad_stance(self):
            class A:
                scope="test-scope"; topic="test-topic"; stance="invalid"; claim="t"; evidence="t"
                confidence="medium"; source="agent:inline"; cue=None; tags=""; supersedes=None
                applies_to=None; outcome=None; compressed_from=None
            with self.assertRaises(SystemExit): cmd_record(A())
        def test_03_dedup(self):
            class A:
                scope="test-scope"; topic="test-topic"; stance="supports"; claim="same"
                evidence="ev"; confidence="medium"; source="agent:inline"; cue=None; tags=""
                supersedes=None; applies_to=None; outcome=None; compressed_from=None
            cmd_record(A()); cmd_record(A())
            self.assertEqual(len(_read_jsonl(OBSERVATIONS)), 1)
        def test_04_distill(self):
            for i in range(3):
                _append_jsonl(OBSERVATIONS, {"id": f"obs_{i+1:06d}", "ts": "2026-01-01T00:00:00Z",
                    "session_id": f"s{i}", "scope": "test-scope", "topic": "test-topic",
                    "stance": "supports", "claim": f"c{i}", "evidence": f"e{i}",
                    "compressed": False, "compressed_from": None, "invalidated": False, "exemplar": False})
            cmd_distill(type("A",(),{})())
            self.assertGreaterEqual(len(_read_jsonl(CLUSTERS)), 1)
        def test_05_teardown_rejects_abs(self):
            MANIFEST_PATH.write_text(json.dumps({"created_by":"stm-power","files":["/etc/passwd"],"hooks":[],"steering":[],"managed_patches":[]}))
            with self.assertRaises(SystemExit): cmd_teardown(type("A",(),{"confirm":True})())
        def test_06_teardown_rejects_traversal(self):
            MANIFEST_PATH.write_text(json.dumps({"created_by":"stm-power","files":["stm/../../etc/passwd"],"hooks":[],"steering":[],"managed_patches":[]}))
            with self.assertRaises(SystemExit): cmd_teardown(type("A",(),{"confirm":True})())
        def test_07_repair(self):
            STATUS_PATH.unlink(missing_ok=True)
            cmd_repair(type("A",(),{"_":None})())
            self.assertTrue(STATUS_PATH.exists())
        def test_08_scope_discovery(self):
            class A:
                scope="new-scope"; topic="test-topic"; stance="supports"; claim="tc"
                evidence="te"; confidence="medium"; source="agent:inline"; cue=None; tags=""
                supersedes=None; applies_to=None; outcome=None; compressed_from=None
            cmd_record(A())
            self.assertIn("new-scope", [s if isinstance(s,str) else s.get("name") for s in _load_config().get("scopes",[])])
    loader = unittest.TestLoader()
    suite = loader.loadTestsFromTestCase(STMSelfTest)
    runner = unittest.TextTestRunner(stream=sys.stderr, verbosity=2 if not getattr(args,"quick",False) else 1)
    result = runner.run(suite)
    _out({"selftest": "pass" if result.wasSuccessful() else "fail", "tests_run": result.testsRun})
    if not result.wasSuccessful(): sys.exit(1)

# ── CLI entry point ──────────────────────────────────────────────────────────

def main():
    p = argparse.ArgumentParser(prog="stm.py", description="STM Introspection CLI")
    sub = p.add_subparsers(dest="command")
    rp = sub.add_parser("record")
    rp.add_argument("--scope", required=True); rp.add_argument("--topic", required=True)
    rp.add_argument("--stance", required=True); rp.add_argument("--claim", required=True)
    rp.add_argument("--evidence", required=True); rp.add_argument("--confidence", required=True)
    rp.add_argument("--source", required=True); rp.add_argument("--cue", default=None)
    rp.add_argument("--tags", default=""); rp.add_argument("--supersedes", default=None)
    rp.add_argument("--applies-to", dest="applies_to", default=None)
    rp.add_argument("--outcome", default=None)
    rp.add_argument("--compressed-from", dest="compressed_from", default=None)
    sub.add_parser("topics")
    rcp = sub.add_parser("recall")
    rcp.add_argument("--scope", default=None); rcp.add_argument("--cue", default=None)
    rcp.add_argument("--limit", type=int, default=3)
    sub.add_parser("status")
    srp = sub.add_parser("search")
    srp.add_argument("term"); srp.add_argument("--limit", type=int, default=5)
    srp.add_argument("--days", type=int, default=None); srp.add_argument("--scope", default=None)
    cmp = sub.add_parser("compress")
    cmp.add_argument("--identify", action="store_true"); cmp.add_argument("--execute", action="store_true")
    cmp.add_argument("--ids", default=None); cmp.add_argument("--exemplars", default=None)
    sub.add_parser("distill"); sub.add_parser("pending-clusters")
    mgp = sub.add_parser("mark-graduated")
    mgp.add_argument("--cluster", required=True); mgp.add_argument("--path", required=True)
    mip = sub.add_parser("mark-integrated")
    mip.add_argument("--cluster", required=True); mip.add_argument("--files", nargs="+", required=True)
    sub.add_parser("health"); sub.add_parser("validate"); sub.add_parser("repair")
    pap = sub.add_parser("purge-all"); pap.add_argument("--confirm", action="store_true")
    tdp = sub.add_parser("teardown"); tdp.add_argument("--confirm", action="store_true")
    stp = sub.add_parser("selftest"); stp.add_argument("--quick", action="store_true")
    args = p.parse_args()
    if not args.command: p.print_help(); sys.exit(1)
    cmds = {"record": cmd_record, "topics": cmd_topics, "recall": cmd_recall,
            "status": cmd_status, "search": cmd_search, "compress": cmd_compress,
            "distill": cmd_distill,
            "pending-clusters": cmd_pending_clusters, "mark-graduated": cmd_mark_graduated,
            "mark-integrated": cmd_mark_integrated, "health": cmd_health,
            "validate": cmd_validate, "repair": cmd_repair, "purge-all": cmd_purge_all,
            "teardown": cmd_teardown, "selftest": cmd_selftest}
    try: cmds[args.command](args)
    except Exception as e: _err(str(e)); sys.exit(1)

if __name__ == "__main__":
    main()

```
