# Task Checklist

Granular tasks per `IMPLEMENTATION_PLAN.md` phase. Check off as completed; keep in sync with GitHub Issues/Project Board (see `CONTRIBUTING.md`).

## Phase 1 — Core Monitoring

**Database**
- [ ] Create Supabase project
- [ ] Write `database/migrations/0001_init.sql` — `machines`, `metric_samples`, `process_samples`
- [ ] Apply migration, verify indexes exist

**Client Agent**
- [ ] Scaffold `client-agent/` (Python 3.11 project)
- [ ] Implement `psutil` sampler (CPU, mem, swap, disk I/O, net I/O, top-50 processes)
- [ ] Implement machine registration (UUID generation, local state file)
- [ ] Implement WS client with heartbeat + telemetry send
- [ ] Implement ring buffer + exponential backoff reconnect
- [ ] Write `agent.config.json` loader
- [ ] PyInstaller build script, verify `.exe` runs standalone
- [ ] Windows Startup registration option

**Backend**
- [ ] Scaffold FastAPI project (`backend/app/`)
- [ ] `POST /machines/register`
- [ ] `/ws/agent` handler — validate schema, threshold check, batched write
- [ ] Threshold engine (config-driven CPU/mem/disk thresholds)
- [ ] `GET /machines`, `GET /machines/{id}/history`

**Dashboard**
- [ ] Scaffold React + TS + Tailwind project
- [ ] Lab Overview page (machine grid, status colors)
- [ ] Machine Details page
- [ ] Live Metrics (WS subscription)
- [ ] Historical Analytics (basic time-range charts)

**Exit check:** run `TESTING.md` §1–3 against a real agent + backend.

## Phase 2 — AI Features

**ML Engine**
- [ ] Scaffold `ml-engine/`
- [ ] Feature pipeline: moving average, trend slope, feature vector assembly
- [ ] Linear Regression prediction (per-machine, min 30 samples gate)
- [ ] Isolation Forest anomaly detection (min 200 samples gate, daily retrain)
- [ ] Scheduled eval loop, writes `predictions` table

**AI Service**
- [ ] Scaffold `ai-service/`
- [ ] Grounding payload builder (capped token budget)
- [ ] Prompt templates: explain, summarize, NL query
- [ ] OpenRouter client integration
- [ ] Failure handling (timeout → "pending", no invented fallback)

**Backend orchestration**
- [ ] `ml.predict()` / `ml.detect_anomaly()` internal interfaces
- [ ] `ai.explain()` / `ai.summarize()` / `ai.answer_query()` internal interfaces
- [ ] Async task queue for anomaly → AI Service dispatch
- [ ] `POST /ai/query`, `GET /ai/summary/lab`, `GET /ai/summary/weekly`
- [ ] `GET /predictions/{machine_id}`

**Dashboard**
- [ ] Alerts page (unified threshold + ml_anomaly feed)
- [ ] AI explanation display on alert detail

**Exit check:** `TESTING.md` §4–6.

## Phase 3 — Dashboard Completion

- [ ] Process Explorer page
- [ ] AI Assistant / NL query UI
- [ ] Health Score computation + display
- [ ] Machine Search
- [ ] Settings page + `GET/PUT /settings`
- [ ] System Logs page + `GET /logs`
- [ ] React Query wiring for all remaining views
- [ ] WS-driven cache invalidation for `machine_update`, `alert`, `ai_explanation`

## Phase 4 — Deployment & Hardening

- [ ] Finalize prod `.env` / `agent.config.json`
- [ ] Deploy backend + dashboard on admin host
- [ ] Roll out agent `.exe` to 3+ physical lab PCs, verify registration
- [ ] TLS termination configured (reverse proxy)
- [ ] Rate limiting verified (dashboard 60rpm, agent register 1rpm)
- [ ] Audit log review of a full test session
- [ ] Load test: 100 mock agents, verify latency/CPU targets (`TESTING.md` §7)
- [ ] Security pass (`SECURITY.md` checklist)

## Phase 5 — Future Enhancements

- [ ] 3D lab visualization spike (React Three Fiber)
- [ ] Remote remediation design doc (not implemented in capstone scope)
- [ ] Multi-lab federation design doc

## Documentation (ongoing, all phases)

- [ ] Keep `CHANGELOG.md` updated per merged PR
- [ ] Keep `README.md` quick-start accurate as setup steps change
- [ ] Update `DATABASE_SCHEMA.md` / `API_SPEC.md` on any schema/contract change
