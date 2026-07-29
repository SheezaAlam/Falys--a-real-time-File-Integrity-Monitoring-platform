# FALYS Detection Engine

## What this is

FALYS's `/log` ingestion route used to do one thing: run a single
deterministic rule pass (`risk_classifier.py`) and store a
low/medium/high/critical severity. That still exists, but it's now one
of three signals feeding a proper detection engine, modeled on the same
pipeline shape commercial EDR/XDR products use (Defender for Endpoint,
CrowdStrike Falcon, SentinelOne, Elastic Security): collect context,
score from multiple independent angles, correlate across events, and
classify with an explicit confidence value plus explainable reasons —
not just a bare severity string.

**Nothing about the agent API changed.** Agents still `POST` the exact
same JSON payload to `/log` with the same `X-API-Key` header they always
did. All existing routes, the dashboard pages, alerts, and agent
deployment/generation flows are unmodified. The engine is entirely a
server-side addition.

## Architecture

```
server/
  engine/
    __init__.py       <- public entry point: analyze_event()
    engine.py          <- orchestrator, wires the stages below together
    context.py          <- 1. Context Collection
    rule_scoring.py     <- 2. Rule-Based Risk Scoring
    anomaly.py           <- 3. Behavioral Anomaly Detection
    correlation.py        <- 4. Event Correlation
    classifier.py          <- 5. Confidence-Based Classification
    state.py         <- shared bounded/thread-safe primitives used by
                         every stage above (LRU caches, rolling windows,
                         online mean/variance)
  engine_config_store.py  <- admin-editable weights/thresholds/tuning
                              (logs/engine_config.json)
  risk_rules_store.py     <- unchanged: admin-editable rules the RULE
                              stage consumes (sensitive paths, event-type
                              severities, critical_hosts, ...)
  server.py                <- /log calls engine.analyze_event() once per
                               event; everything else (routes, alerts,
                               dashboard) is unchanged
```

### The five stages, per event

1. **Context Collection** (`context.py`) — is this host/user new, is it
   business hours, is this the first time this user has touched this
   directory, is the host tagged as a critical asset. Pure fact
   gathering, no scoring.
2. **Rule-Based Risk Scoring** (`rule_scoring.py`) — the original
   deterministic logic (event-type baseline, sensitive-path globs,
   permission-change detection, burst-frequency), now producing a 0-100
   score instead of just a tier, admin-editable via `/api/risk-rules`
   exactly as before.
3. **Behavioral Anomaly Detection** (`anomaly.py`) — no fixed rules;
   builds a per-host statistical baseline (online mean/variance of
   events-per-minute, an hour-of-day activity histogram) and flags
   deviations from *that specific host's own history*, plus novelty
   flags for "first time this user has used this host/path."
4. **Event Correlation** (`correlation.py`) — groups related events
   within a sliding time window per host: bursts, staged tampering
   patterns (permission loosened → sensitive file touched → deleted),
   and cross-host activity by the same attributed user. Produces
   `INC-xxxxxxxxxx` incident records, listed at `GET /api/incidents`.
5. **Confidence-Based Classification** (`classifier.py`) — combines the
   three scores (weighted, admin-tunable) into one 0-100 score, maps it
   to a tier (**Legitimate / Low Risk / Suspicious / High Risk /
   Critical**), and computes a separate 0-100 **confidence** value driven
   by how mature the anomaly baseline is and how many independent
   signals agree.

### Fault tolerance

`engine.py`'s orchestrator wraps every stage individually. If any one
stage raises (bad data, an edge case in a statistical calculation,
whatever), that stage's contribution degrades to a safe default and a
warning is logged — ingestion never stops because a detector had a bug.
This mirrors how a production EDR pipeline should behave: a scoring
regression should degrade detection quality, not availability.

### What gets stored per event

`server.py`'s `/log` route now stores, in addition to the pre-existing
fields:

```json
{
  "severity": "high",                  // legacy, unchanged meaning
  "classification": "High Risk",
  "confidence": 82,
  "risk_score": 71,
  "reasons": [
    "Path matches sensitive-path rule '*shadow' (critical)",
    "File permissions changed from their last known value (escalates to 'high')",
    "Staged tampering pattern on this host: permission_change -> sensitive_hit within 120s"
  ],
  "correlation_id": "INC-11b2ad78e6",
  "signals": {"rule": 65, "anomaly": 35, "correlation": 55}
}
```

`should_alert()` (`rules.py`) is untouched — it still reads the legacy
`severity` field against the admin-configured `alert_threshold`, so
email alerting behavior is identical to before.

## New API routes

| Route | Method | Purpose |
|---|---|---|
| `/api/engine-config` | GET | Read the engine's weights/thresholds/tuning |
| `/api/engine-config` | POST | Patch weights/thresholds (same unknown-key-rejection pattern as `/api/risk-rules`) |
| `/api/incidents` | GET | Recent correlated incidents (`?limit=N`) |

`/api/stats` gained a `classification_counts` field (5-tier breakdown)
alongside the existing `severity_counts` (4-tier) — both are populated
from every event going forward; older events default to
`Legitimate`/`low` rather than skewing the new breakdown.

A new dashboard page, **Detections** (`/detections`), shows the
classified event table (with expandable reasons and a confidence bar)
and the correlated-incidents feed.

## Tuning

Everything scoring-related is data, not code, editable at runtime:

- **`risk_rules_store.py`** (`/api/risk-rules`) — sensitive-path globs,
  event-type baseline severities, permission-change severity, the
  frequency-burst rule, `alert_threshold`, and the new `critical_hosts`
  list.
- **`engine_config_store.py`** (`/api/engine-config`) — signal weights
  (`rule`/`anomaly`/`correlation`), classification-tier score
  boundaries, the anomaly stage's z-score threshold and business-hours
  window, and the correlation stage's window/burst thresholds.

Both follow the same pattern: a `DEFAULT_*` dict, a JSON file under
`logs/`, and a `set_*()` that only accepts known keys and merges rather
than overwrites nested sub-dicts.

## Scaling to thousands of agents on a production Linux server

The current implementation is intentionally still a **single Python
process with in-memory state** — that's appropriate for the scale this
codebase is at today (a Flask dev server, JSONL file storage). Before
pointing this at thousands of real endpoints, the following upgrades
matter, roughly in the order they'll start to hurt:

1. **Run under a real WSGI server**, not `app.run()`. `gunicorn` with
   the `gevent` or `gthread` worker class handles the blocking I/O
   pattern here (many short-lived POSTs) far better than Flask's
   built-in dev server, e.g.:
   `gunicorn -w 4 -k gthread --threads 8 -b 0.0.0.0:8088 server:app`
   Note the Server-Sent-Events endpoint (`/api/stream`) needs a worker
   class that supports long-lived connections (`gevent`/`eventlet`), or
   its own dedicated process.

2. **Move event storage off a single JSONL file.** `load_events()`
   currently re-reads and re-parses the entire file on every dashboard
   request — fine at thousands of events, not at the volume thousands
   of agents will produce in days. Point it at a real time-series-
   friendly store (Postgres/TimescaleDB, or Elasticsearch/OpenSearch
   given this is explicitly an Elastic-Security-style use case) and keep
   `load_events()`'s call sites as the only place that needs to change.

3. **Move the engine's in-memory state to a shared store once you run
   more than one worker process/machine.** Every `LRUDict`/
   `RollingWindow`/`OnlineStats` in `engine/state.py` lives in one
   process's memory. That's correct and fast for a single process, but
   two gunicorn workers (or two servers behind a load balancer) would
   each maintain *separate, inconsistent* baselines and correlation
   state for the same host. The fix is to swap `state.py`'s primitives
   for Redis-backed equivalents (Redis natively supports capped sorted
   sets for rolling windows, hashes for per-host counters, and
   `INCR`/`EXPIRE` for rate counting) behind the same function
   signatures — `context.py`, `anomaly.py`, and `correlation.py` were
   written against `state.py`'s interface specifically so this swap
   doesn't require touching the detection logic itself. Alternatively,
   route each agent's traffic to the same worker via consistent hashing
   on `host` at the load balancer, so per-host state never needs to be
   shared — simpler, but couples your scaling story to your LB config.

4. **The agent discovery/announce mechanism (`discovery.py`) is UDP
   broadcast-based**, which works on a single flat LAN segment but not
   across subnets/VLANs or in a cloud VPC. At real fleet scale, agents
   should be configured with a real, stable server address (DNS name
   behind a load balancer) rather than relying on broadcast discovery,
   which then becomes a same-subnet fallback rather than the primary
   mechanism.

5. **Secrets currently live in `config.py`** (`AGENT_API_KEY`,
   `BREVO_API_KEY`, `ADMIN_PASSWORD`, `SECRET_KEY`, `DISCOVERY_TOKEN`).
   Before a real multi-host deployment, move these to environment
   variables or a secrets manager and rotate every value currently
   checked into that file.

6. **Per-agent API keys.** Right now every agent shares one
   `AGENT_API_KEY`. At fleet scale you'll want to revoke a single
   compromised endpoint without rotating the secret for every other
   agent — that means per-agent (or per-deployment-batch) keys, checked
   against a store instead of one constant.

None of the above blocks running this today at a moderate scale (dozens
to low hundreds of agents on one server, one process) — they're the
concrete next steps, in priority order, for "thousands of agents."
