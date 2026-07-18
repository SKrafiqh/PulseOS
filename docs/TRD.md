# AI-Driven Distributed Computer Lab Monitoring System

**Technical Requirements Document (TRD) — v1.0**
Companion doc to `AI_Lab_Monitoring_PRD_v3.md`. PRD = what/why. TRD = exact how — schemas, contracts, algorithms, config, error codes, acceptance criteria.

---

## Table of Contents

1. [Purpose & Relationship to PRD](#1-purpose--relationship-to-prd)
2. [Technology Versions](#2-technology-versions)
3. [Environment & Configuration](#3-environment--configuration)
4. [Client Agent — Technical Spec](#4-client-agent--technical-spec)
5. [Backend — Technical Spec](#5-backend--technical-spec)
6. [WebSocket Message Contracts](#6-websocket-message-contracts)
7. [REST API Contracts](#7-rest-api-contracts)
8. [Database — DDL](#8-database--ddl)
9. [ML Engine — Technical Spec](#9-ml-engine--technical-spec)
10. [AI Service — Technical Spec](#10-ai-service--technical-spec)
11. [Frontend — Technical Spec](#11-frontend--technical-spec)
12. [Security — Technical Spec](#12-security--technical-spec)
13. [Error Codes](#13-error-codes)
14. [Non-Functional Targets (Measurable)](#14-non-functional-targets-measurable)
15. [Testing & Acceptance Criteria](#15-testing--acceptance-criteria)
16. [Deployment & CI](#16-deployment--ci)
17. [Appendix — Config File Examples](#17-appendix--config-file-examples)

---

## 1. Purpose & Relationship to PRD

PRD defines requirements and architecture intent. TRD defines the exact contracts an implementer builds against: field names, types, status codes, algorithm parameters, and pass/fail criteria. Where PRD and TRD conflict, **TRD wins for implementation**; PRD wins for scope/intent disputes.

---

## 2. Technology Versions

| Layer | Package | Pinned Version (minimum) |
|---|---|---|
| Client Agent | Python | 3.11+ |
| Client Agent | psutil | 5.9+ |
| Client Agent | websockets | 12.0+ |
| Client Agent | PyInstaller | 6.x |
| Backend | Python | 3.11+ |
| Backend | fastapi | 0.110+ |
| Backend | uvicorn[standard] | 0.29+ |
| Backend | sqlalchemy | 2.0+ |
| Backend | pyjwt | 2.8+ |
| Backend | supabase-py | 2.x |
| ML Engine | scikit-learn | 1.4+ |
| ML Engine | pandas / numpy | 2.x / 1.26+ |
| AI Service | openai-compatible client (OpenRouter) | latest |
| Frontend | React | 18.x |
| Frontend | TypeScript | 5.x |
| Frontend | TailwindCSS | 3.x |
| Frontend | Chart.js | 4.x |
| Frontend | @tanstack/react-query | 5.x |
| DB | PostgreSQL (via Supabase) | 15+ |

---

## 3. Environment & Configuration

### 3.1 Backend `.env`

```
SUPABASE_URL=https://<project>.supabase.co
SUPABASE_SERVICE_ROLE_KEY=<secret>
SUPABASE_ANON_KEY=<public>
JWT_SECRET=<secret>
JWT_EXPIRY_MINUTES=60
OPENROUTER_API_KEY=<secret>
OPENROUTER_MODEL=<model-id>
AGENT_HEARTBEAT_TIMEOUT_SECONDS=30
SAMPLING_INTERVAL_DEFAULT_SECONDS=3
ML_EVAL_INTERVAL_SECONDS=60
CPU_THRESHOLD_PCT=90
MEM_THRESHOLD_PCT=90
DISK_THRESHOLD_PCT=95
RATE_LIMIT_DASHBOARD_RPM=60
RATE_LIMIT_AGENT_REGISTER_RPM=1
```

### 3.2 Client Agent `agent.config.json`

```json
{
  "backend_ws_url": "wss://<host>/ws/agent",
  "sampling_interval_seconds": 3,
  "heartbeat_interval_seconds": 10,
  "buffer_max_samples": 600,
  "reconnect_backoff_base_ms": 500,
  "reconnect_backoff_max_ms": 30000,
  "log_level": "INFO",
  "run_on_startup": true
}
```

---

## 4. Client Agent — Technical Spec

### 4.1 Sampling

- Collector runs on `sampling_interval_seconds` (config, default 3s).
- Uses `psutil.cpu_percent(percpu=True)`, `psutil.virtual_memory()`, `psutil.swap_memory()`, `psutil.disk_io_counters()`, `psutil.net_io_counters()`, `psutil.process_iter(['pid','name','username','cpu_percent','memory_info','num_threads'])`.
- Per-sample process list capped at top 50 by CPU% to bound payload size.

### 4.2 Buffering / Retry Queue

- Ring buffer, max `buffer_max_samples` (default 600 ≈ 30 min at 3s interval). On overflow, **drop oldest** (FIFO eviction), never drop newest.
- On disconnect: sampling continues into buffer; sender coroutine retries connect with exponential backoff: `min(base_ms * 2^attempt, max_ms)` + jitter (±20%).
- On reconnect: flush buffer oldest-first in batches of ≤50 samples per WS frame.

### 4.3 Registration Sequence

1. Read/create `machine_id` (UUID v4) in local state file on first run — persists across restarts.
2. POST `/machines/register` with `{machine_id, hostname}` → receive `{session_token}`.
3. Open WS to `backend_ws_url` with `Authorization: Bearer <session_token>`.

### 4.4 Packaging

- `pyinstaller --onefile --noconsole agent_main.py` → single `.exe`.
- Windows Startup: writes shortcut to `shell:startup` if `run_on_startup=true`.

---

## 5. Backend — Technical Spec

### 5.1 Process Model

- Single Uvicorn ASGI process for HTTP+WS (dev); `--workers N` behind a process manager for prod, with sticky routing not required since state lives in Postgres, not memory.
- Background scheduler (APScheduler or asyncio task) triggers ML eval every `ML_EVAL_INTERVAL_SECONDS`.
- Task queue (in-process `asyncio.Queue` for Phase 1; swappable for Redis/RQ at scale) decouples anomaly → AI Service calls from the eval loop.

### 5.2 Ingestion Write Path

1. WS frame received → validate against schema (Section 6.1).
2. Threshold check runs synchronously against the sample (no DB read required — thresholds are static config).
3. Sample appended to in-memory write buffer; buffer flushed to `metric_samples`/`process_samples` every 1–2s or 100 rows, whichever first (batched `INSERT`).
4. Ack returned to agent only after buffer accept (not after DB commit) to keep ingestion latency low.

---

## 6. WebSocket Message Contracts

### 6.1 Agent → Backend: `telemetry_batch`

```json
{
  "type": "telemetry_batch",
  "machine_id": "uuid",
  "samples": [
    {
      "sampled_at": "2026-07-18T10:00:00Z",
      "cpu_pct": 42.5,
      "mem_pct": 61.2,
      "swap_pct": 5.0,
      "disk_io_read_bps": 1048576,
      "disk_io_write_bps": 262144,
      "net_sent_bps": 12000,
      "net_recv_bps": 84000,
      "processes": [
        {"pid": 1234, "name": "chrome.exe", "cpu_pct": 12.1, "mem_mb": 340.2, "thread_count": 22}
      ]
    }
  ]
}
```

### 6.2 Agent → Backend: `heartbeat`
```json
{"type": "heartbeat", "machine_id": "uuid", "ts": "2026-07-18T10:00:10Z"}
```

### 6.3 Backend → Agent: `ack`
```json
{"type": "ack", "received": 1, "ts": "2026-07-18T10:00:00Z"}
```

### 6.4 Backend → Dashboard: `machine_update`
```json
{"type": "machine_update", "machine_id": "uuid", "status": "online", "cpu_pct": 42.5, "mem_pct": 61.2}
```

### 6.5 Backend → Dashboard: `alert`
```json
{"type": "alert", "alert_id": 991, "machine_id": "uuid", "source": "ml_anomaly", "severity": "warning", "message": "CPU trending toward saturation"}
```

### 6.6 Backend → Dashboard: `ai_explanation`
```json
{"type": "ai_explanation", "alert_id": 991, "content": "...", "grounding_ref": {"metric_sample_ids": [1,2,3]}}
```

---

## 7. REST API Contracts

All responses: `application/json`. Errors follow Section 13 format.

| Endpoint | Request | Response 200 | Errors |
|---|---|---|---|
| `POST /auth/login` | `{email, password}` | `{jwt, expires_at, role}` | 401 |
| `POST /machines/register` | `{machine_id, hostname}` | `{session_token}` | 400, 409 |
| `GET /machines` | — | `[{machine_id, hostname, status, cpu_pct, mem_pct, last_seen_at}]` | 401 |
| `GET /machines/{id}/history?from=&to=&granularity=` | query params | `{samples: [...]}` | 401, 404 |
| `GET /alerts?severity=&machine_id=` | query params | `[{alert_id, machine_id, severity, message, created_at, resolved_at}]` | 401 |
| `GET /predictions/{machine_id}` | — | `{predicted_cpu_pct, predicted_mem_pct, anomaly_score, is_anomalous, generated_at}` | 401, 404 |
| `POST /ai/query` | `{question}` | `{answer, grounding_ref}` | 401, 429 |
| `GET /ai/summary/lab` | — | `{content, created_at}` | 401 |
| `GET /ai/summary/weekly` | — | `{content, created_at}` | 401 |
| `GET/PUT /settings` | `{key, value}` (PUT) | `{key, value, updated_at}` | 401, 403 |
| `GET /logs?from=&to=` | query params | `[{action, actor, reference_id, created_at}]` | 401, 403 |

**Auth header:** `Authorization: Bearer <jwt>` on all endpoints except `/auth/login` and `/machines/register` (machine token instead).

---

## 8. Database — DDL

```sql
create table machines (
  machine_id uuid primary key default gen_random_uuid(),
  hostname text not null,
  auth_token_hash text not null,
  location_label text,
  status text not null default 'offline',
  last_seen_at timestamptz,
  created_at timestamptz not null default now()
);
create index idx_machines_status on machines(status);

create table metric_samples (
  id bigserial primary key,
  machine_id uuid not null references machines(machine_id),
  sampled_at timestamptz not null,
  cpu_pct real, mem_pct real, swap_pct real,
  disk_io_read_bps real, disk_io_write_bps real,
  net_sent_bps real, net_recv_bps real
);
create index idx_metric_machine_time on metric_samples(machine_id, sampled_at desc);

create table process_samples (
  id bigserial primary key,
  machine_id uuid not null references machines(machine_id),
  sampled_at timestamptz not null,
  pid int, process_name text, cpu_pct real, mem_mb real, thread_count int
);
create index idx_process_machine_time on process_samples(machine_id, sampled_at desc);
create index idx_process_name on process_samples(process_name);

create table predictions (
  id bigserial primary key,
  machine_id uuid not null references machines(machine_id),
  generated_at timestamptz not null default now(),
  horizon_minutes int not null,
  predicted_cpu_pct real, predicted_mem_pct real,
  model_used text, anomaly_score real, is_anomalous boolean default false
);
create index idx_predictions_machine_time on predictions(machine_id, generated_at desc);

create table alerts (
  id bigserial primary key,
  machine_id uuid not null references machines(machine_id),
  source text not null check (source in ('threshold','ml_anomaly')),
  severity text not null check (severity in ('warning','critical')),
  message text not null,
  created_at timestamptz not null default now(),
  resolved_at timestamptz
);
create index idx_alerts_created on alerts(created_at desc);

create table ai_summaries (
  id bigserial primary key,
  scope text not null check (scope in ('anomaly_explain','lab_summary','nl_query','weekly_digest')),
  related_alert_id bigint references alerts(id),
  content text not null,
  grounding_ref jsonb not null,
  created_at timestamptz not null default now()
);

create table audit_logs (
  id bigserial primary key,
  actor text not null,
  action text not null,
  reference_id text,
  created_at timestamptz not null default now()
);
create index idx_audit_created on audit_logs(created_at desc);

create table users (
  id uuid primary key,
  email text unique not null,
  role text not null check (role in ('admin','faculty')),
  created_at timestamptz not null default now()
);

create table configurations (
  key text primary key,
  value jsonb not null,
  updated_by uuid references users(id),
  updated_at timestamptz not null default now()
);

create table sessions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references users(id),
  issued_at timestamptz not null default now(),
  expires_at timestamptz not null,
  revoked boolean not null default false
);
create index idx_sessions_user on sessions(user_id);
```

> Note: `users` created before `configurations` here (DDL order matters — FK dependency), unlike listing order in the PRD.

---

## 9. ML Engine — Technical Spec

### 9.1 Feature Set (per machine, per evaluation window)

`[cpu_pct_ma5, mem_pct_ma5, swap_pct, disk_io_total, net_io_total, cpu_trend_slope, mem_trend_slope, process_count]`

- `ma5` = 5-sample simple moving average.
- `trend_slope` = linear regression slope of `ma5` over last 10 windows.

### 9.2 Prediction Model

- `sklearn.linear_model.LinearRegression` fit per machine on lag features `[t-1, t-2, t-3, trend_slope]` → predicts `cpu_pct`/`mem_pct` at `horizon_minutes` (default 10).
- Refit on each `ML_EVAL_INTERVAL_SECONDS` tick using trailing 24h window (min 30 samples required to produce a prediction; below that, `predictions` row omitted, no forecast on dashboard).

### 9.3 Anomaly Detection

- `sklearn.ensemble.IsolationForest(n_estimators=100, contamination=0.05, random_state=42)`, trained per machine on trailing 7-day feature history.
- Retrained daily (not per-tick) to amortize cost; per-tick call is `.decision_function()` only.
- `anomaly_score` = negative of `decision_function` output (higher = more anomalous). `is_anomalous = anomaly_score > 0.15` (tunable via `configurations`).
- Minimum 200 historical samples required before a machine's Isolation Forest is trained; until then `is_anomalous` always `false`.

### 9.4 Output Contract

Every eval tick writes exactly one `predictions` row per machine per horizon evaluated, per Section 8 schema.

---

## 10. AI Service — Technical Spec

### 10.1 Input Contract (strict)

AI Service functions accept **only**:
```json
{
  "trigger": {"alert_id": 991, "source": "ml_anomaly", "anomaly_score": 0.31},
  "grounding": {
    "metric_samples": [ "last 20 rows for the machine" ],
    "process_samples": [ "top 10 by cpu_pct at trigger time" ],
    "prior_predictions": [ "last 3 predictions rows" ]
  }
}
```
No raw unbounded telemetry dump — grounding payload capped at ~2,000 tokens to keep cost/latency bounded and force the model to work from summarized, relevant context.

### 10.2 Prompt Template (explanation)

```
System: You explain machine resource anomalies using ONLY the data provided.
Never state a fact not present in `grounding`. If data is insufficient, say so.
User: {trigger} {grounding}
Task: Explain likely root cause in <=120 words. Cite specific process/metric values used.
```

### 10.3 Output Contract

```json
{"content": "...", "grounding_ref": {"metric_sample_ids": [], "process_sample_ids": []}}
```
Stored verbatim in `ai_summaries.content` / `ai_summaries.grounding_ref`.

### 10.4 Failure Handling

- OpenRouter timeout/error → alert still delivered to dashboard (Section 6.5) **without** `ai_explanation` message; UI shows "Explanation pending" and retries once after 30s.
- No fallback to unsupported/invented text under any failure mode.

---

## 11. Frontend — Technical Spec

- Routing: React Router, one route per Section 22 PRD page.
- Data fetching: `@tanstack/react-query` — polling fallback (5s) for any view not covered by an active WS subscription; WS messages (Section 6.4–6.6) invalidate/update the relevant query cache directly.
- State: server state lives in React Query cache; only UI-local state (filters, selected machine) in component state — no global client store needed for Phase 1.
- Charts: Chart.js line charts for time-series, capped at 500 points per render (server-side downsampling via `granularity` param on `/machines/{id}/history`).

---

## 12. Security — Technical Spec

| Control | Detail |
|---|---|
| JWT | HS256, `JWT_SECRET` env, `exp` = `JWT_EXPIRY_MINUTES` (default 60), claims: `{sub: user_id, role, exp}`. |
| Machine token | Random 32-byte token, stored as SHA-256 hash in `machines.auth_token_hash`. |
| Password hashing | Delegated to Supabase Auth (bcrypt). |
| TLS | Enforced at reverse proxy / Uvicorn SSL context; plain `ws://`/`http://` rejected in prod config. |
| Rate limiting | Token-bucket per JWT/machine token: dashboard 60 rpm, agent register 1 rpm (Section 3.1 env vars). 429 on exceed. |
| CORS | Dashboard origin allow-listed explicitly; no wildcard `*` in prod. |

---

## 13. Error Codes

| HTTP | Code | Meaning |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Malformed request body. |
| 401 | `UNAUTHENTICATED` | Missing/invalid JWT or machine token. |
| 403 | `FORBIDDEN` | Valid auth, insufficient role. |
| 404 | `NOT_FOUND` | Machine/alert/resource does not exist. |
| 409 | `ALREADY_REGISTERED` | Machine ID already registered. |
| 429 | `RATE_LIMITED` | Rate limit exceeded. |
| 500 | `INTERNAL_ERROR` | Unhandled server error — logged to `audit_logs`. |

Error body shape: `{"error_code": "...", "message": "...", "detail": null}`

---

## 14. Non-Functional Targets (Measurable)

| Metric | Target | Verification |
|---|---|---|
| Dashboard update latency | <= 3s p95 from sample generation to WS push | Load test with 20 simulated agents |
| Agent CPU overhead | < 2% avg over 10 min sampling window | `psutil` self-measurement on agent host |
| Concurrent agent connections | 100+ without latency target breach | Load test with mock agents |
| Ingestion write batch flush | <= 2s or 100 rows | Backend metrics log |
| ML eval cycle | Completes within `ML_EVAL_INTERVAL_SECONDS` for 100 machines | Backend timing log |
| AI explanation turnaround | <= 10s p95 from anomaly event to `ai_explanation` push (excl. OpenRouter outage) | Timestamp diff in `ai_summaries` vs `alerts.created_at` |

---

## 15. Testing & Acceptance Criteria

1. **Registration:** fresh agent registers, appears in `GET /machines` within 1 heartbeat interval.
2. **Reconnect resilience:** kill network for 60s mid-stream -> buffered samples appear in `metric_samples` in correct order post-reconnect, none dropped (given buffer not exceeded).
3. **Threshold alert:** force CPU > `CPU_THRESHOLD_PCT` for 3 consecutive samples -> `alerts` row with `source='threshold'` created within one sampling interval.
4. **ML anomaly:** seed synthetic memory-leak pattern (steady linear increase) -> `is_anomalous=true` predictions row appears before value crosses static threshold.
5. **Grounded explanation:** anomaly alert triggers `ai_summaries` row whose `grounding_ref` IDs all exist in `metric_samples`/`process_samples`.
6. **NL query grounding:** `POST /ai/query` answer's `grounding_ref` resolves to non-empty, relevant rows for at least 5 varied test questions.
7. **Load:** 100 mock agents sustained 10 min -> p95 dashboard update latency <= 3s (Section 14).
8. **Security:** request with expired/forged JWT -> 401 on all protected endpoints; agent token cannot read another machine's data (403/404).

---

## 16. Deployment & CI

- `.github/workflows/ci.yml`: lint + unit tests on push (backend: pytest; frontend: vitest); build check for client agent PyInstaller step on tag.
- Migrations applied via Supabase CLI (`supabase db push`) against DDL in Section 8, tracked in `database/migrations/`.
- Backend deploy: `uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 2` behind TLS-terminating reverse proxy on admin host.
- Frontend deploy: `npm run build` -> static bundle served by backend or lightweight static server on same host.

---

## 17. Appendix — Config File Examples

See Section 3 for `.env` and `agent.config.json`. Production overrides live in `config/prod.env`; development overrides in `config/dev.env`, loaded by `scripts/start_dev.sh` / `scripts/start_prod.sh`.
