# Database Schema

Platform: Supabase PostgreSQL 15+. See `SYSTEM_ARCHITECTURE.md` §Supabase Role for platform vs. data-model distinction.

## Entity Relationship Summary

```
machines 1─N metric_samples
machines 1─N process_samples
machines 1─N predictions
machines 1─N alerts
alerts   1─N ai_summaries   (related_alert_id nullable — some summaries aren't alert-triggered)
users    1─N sessions
users    1─N configurations (as editor, via updated_by)
```

## Tables

### `machines`
Registry of every Windows lab PC.

| Column | Type | Constraints |
|---|---|---|
| machine_id | uuid | PK, default `gen_random_uuid()` |
| hostname | text | not null |
| auth_token_hash | text | not null — SHA-256 of per-machine token |
| location_label | text | nullable — desk/room label, Phase 2 spatial view |
| status | text | not null, default `'offline'` — `online` / `idle` / `offline` |
| last_seen_at | timestamptz | updated on every heartbeat |
| created_at | timestamptz | not null, default `now()` |

Index: `idx_machines_status (status)`

### `metric_samples`
System-level OS telemetry, one row per sample per machine.

| Column | Type | Constraints |
|---|---|---|
| id | bigserial | PK |
| machine_id | uuid | FK → machines |
| sampled_at | timestamptz | not null |
| cpu_pct | real | |
| mem_pct | real | |
| swap_pct | real | |
| disk_io_read_bps | real | |
| disk_io_write_bps | real | |
| net_sent_bps | real | |
| net_recv_bps | real | |

Index: `idx_metric_machine_time (machine_id, sampled_at desc)` — primary access pattern is "latest N samples for machine X."

### `process_samples`
Per-process telemetry, one row per process per sample.

| Column | Type | Constraints |
|---|---|---|
| id | bigserial | PK |
| machine_id | uuid | FK → machines |
| sampled_at | timestamptz | not null |
| pid | int | |
| process_name | text | |
| cpu_pct | real | |
| mem_mb | real | |
| thread_count | int | |

Indexes: `idx_process_machine_time (machine_id, sampled_at desc)`, `idx_process_name (process_name)` for search.

### `predictions`
ML Engine output only. **Never written by the AI Service.**

| Column | Type | Constraints |
|---|---|---|
| id | bigserial | PK |
| machine_id | uuid | FK → machines |
| generated_at | timestamptz | not null, default `now()` |
| horizon_minutes | int | not null |
| predicted_cpu_pct | real | |
| predicted_mem_pct | real | |
| model_used | text | `linear_regression` / `isolation_forest` |
| anomaly_score | real | |
| is_anomalous | boolean | default `false` |

Index: `idx_predictions_machine_time (machine_id, generated_at desc)`

### `alerts`
Threshold and ML-derived alerts, unified feed.

| Column | Type | Constraints |
|---|---|---|
| id | bigserial | PK |
| machine_id | uuid | FK → machines |
| source | text | check in `('threshold','ml_anomaly')` |
| severity | text | check in `('warning','critical')` |
| message | text | not null |
| created_at | timestamptz | not null, default `now()` |
| resolved_at | timestamptz | nullable |

Index: `idx_alerts_created (created_at desc)`

### `ai_summaries`
AI Service output only. **Never a source of predictions.**

| Column | Type | Constraints |
|---|---|---|
| id | bigserial | PK |
| scope | text | check in `('anomaly_explain','lab_summary','nl_query','weekly_digest')` |
| related_alert_id | bigint | FK → alerts, nullable |
| content | text | not null |
| grounding_ref | jsonb | not null — IDs of source rows, always populated |
| created_at | timestamptz | not null, default `now()` |

### `audit_logs`
Immutable action trail.

| Column | Type | Constraints |
|---|---|---|
| id | bigserial | PK |
| actor | text | `system` or admin user id |
| action | text | e.g. `alert_raised`, `ai_explanation_generated` |
| reference_id | text | nullable |
| created_at | timestamptz | not null, default `now()` |

Index: `idx_audit_created (created_at desc)`

### `users`
Managed jointly with Supabase Auth.

| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK — mirrors Supabase Auth user id |
| email | text | unique, not null |
| role | text | check in `('admin','faculty')` |
| created_at | timestamptz | not null, default `now()` |

### `configurations`
Runtime-tunable settings (thresholds, intervals) — see `SECURITY.md` for who may write.

| Column | Type | Constraints |
|---|---|---|
| key | text | PK |
| value | jsonb | not null |
| updated_by | uuid | FK → users |
| updated_at | timestamptz | not null, default `now()` |

### `sessions`
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK, default `gen_random_uuid()` |
| user_id | uuid | FK → users |
| issued_at | timestamptz | not null, default `now()` |
| expires_at | timestamptz | not null |
| revoked | boolean | not null, default `false` |

Index: `idx_sessions_user (user_id)`

## DDL

Full `CREATE TABLE` statements live in `database/migrations/0001_init.sql`. Table creation order (FK dependency): `machines` → `metric_samples` / `process_samples` / `predictions` / `alerts` → `ai_summaries` → `audit_logs` → `users` → `configurations` → `sessions`.

## Retention & Partitioning Notes

`metric_samples` and `process_samples` are the highest-volume tables. At 100 machines × 3s interval, expect ~2.4M `metric_samples` rows/day. Partition by `sampled_at` (monthly) once volume passes ~10M rows; not required for Phase 1 lab-scale deployment.
