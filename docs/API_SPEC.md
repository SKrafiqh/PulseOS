# API Specification

Base: FastAPI backend. All REST responses `application/json`. All protected endpoints require `Authorization: Bearer <token>` (JWT for dashboard, machine token for agent). Error format in §Errors below.

## REST Endpoints

| Method & Path | Auth | Request Body | 200 Response | Errors |
|---|---|---|---|---|
| `POST /auth/login` | none | `{email, password}` | `{jwt, expires_at, role}` | 401 |
| `POST /machines/register` | machine token | `{machine_id, hostname}` | `{session_token}` | 400, 409 |
| `GET /machines` | JWT | — | `[{machine_id, hostname, status, cpu_pct, mem_pct, last_seen_at}]` | 401 |
| `GET /machines/{id}/history?from=&to=&granularity=` | JWT | — | `{samples: [...]}` | 401, 404 |
| `GET /alerts?severity=&machine_id=` | JWT | — | `[{alert_id, machine_id, severity, message, created_at, resolved_at}]` | 401 |
| `GET /predictions/{machine_id}` | JWT | — | `{predicted_cpu_pct, predicted_mem_pct, anomaly_score, is_anomalous, generated_at}` | 401, 404 |
| `POST /ai/query` | JWT | `{question}` | `{answer, grounding_ref}` | 401, 429 |
| `GET /ai/summary/lab` | JWT | — | `{content, created_at}` | 401 |
| `GET /ai/summary/weekly` | JWT | — | `{content, created_at}` | 401 |
| `GET /settings` | JWT | — | `{key, value, updated_at}` | 401 |
| `PUT /settings` | JWT (admin) | `{key, value}` | `{key, value, updated_at}` | 401, 403 |
| `GET /logs?from=&to=` | JWT (admin) | — | `[{action, actor, reference_id, created_at}]` | 401, 403 |

Rate limits: dashboard JWT 60 req/min, agent registration 1 req/min per machine token. Exceeding either returns `429 RATE_LIMITED`.

## WebSocket Channels

### `/ws/agent` — Client Agent → Backend

**`telemetry_batch`**
```json
{
  "type": "telemetry_batch",
  "machine_id": "uuid",
  "samples": [{
    "sampled_at": "2026-07-18T10:00:00Z",
    "cpu_pct": 42.5, "mem_pct": 61.2, "swap_pct": 5.0,
    "disk_io_read_bps": 1048576, "disk_io_write_bps": 262144,
    "net_sent_bps": 12000, "net_recv_bps": 84000,
    "processes": [{"pid": 1234, "name": "chrome.exe", "cpu_pct": 12.1, "mem_mb": 340.2, "thread_count": 22}]
  }]
}
```

**`heartbeat`**
```json
{"type": "heartbeat", "machine_id": "uuid", "ts": "2026-07-18T10:00:10Z"}
```

**Backend replies with `ack`:**
```json
{"type": "ack", "received": 1, "ts": "2026-07-18T10:00:00Z"}
```

### `/ws/dashboard` — Backend → Dashboard

**`machine_update`**
```json
{"type": "machine_update", "machine_id": "uuid", "status": "online", "cpu_pct": 42.5, "mem_pct": 61.2}
```

**`alert`**
```json
{"type": "alert", "alert_id": 991, "machine_id": "uuid", "source": "ml_anomaly", "severity": "warning", "message": "CPU trending toward saturation"}
```

**`ai_explanation`**
```json
{"type": "ai_explanation", "alert_id": 991, "content": "...", "grounding_ref": {"metric_sample_ids": [1,2,3]}}
```

## Errors

| HTTP | Code | Meaning |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Malformed request body |
| 401 | `UNAUTHENTICATED` | Missing/invalid JWT or machine token |
| 403 | `FORBIDDEN` | Valid auth, insufficient role |
| 404 | `NOT_FOUND` | Resource does not exist |
| 409 | `ALREADY_REGISTERED` | Machine ID already registered |
| 429 | `RATE_LIMITED` | Rate limit exceeded |
| 500 | `INTERNAL_ERROR` | Unhandled server error, logged to `audit_logs` |

Body shape: `{"error_code": "...", "message": "...", "detail": null}`
