# Implementation Plan

Phased build order. Task-level breakdown: `TASKS.md`. Architecture reference: `SYSTEM_ARCHITECTURE.md`.

## Phase 1 — Core Monitoring

**Goal:** telemetry flows end-to-end and is visible live, with zero AI/ML involved yet.

- Supabase project setup; apply `database/migrations/0001_init.sql` (`machines`, `metric_samples`, `process_samples`).
- Client Agent: registration, heartbeat, psutil sampling, WS send, local buffer/retry queue.
- Backend: `/machines/register`, `/ws/agent` ingestion, threshold engine (static config), batched DB writes.
- Dashboard: Lab Overview, Machine Details, Live Metrics, Historical Analytics (basic).

**Exit criteria:** agent on a real Windows PC streams to backend, dashboard shows live CPU/mem within 3s, threshold alert fires correctly. See `TESTING.md` §1–3.

## Phase 2 — AI Features

**Goal:** ML predicts/detects, LLM explains — boundary enforced from day one.

- ML Engine: feature pipeline (moving average, trend slope), Linear Regression prediction, Isolation Forest anomaly detection, scheduled eval loop.
- `predictions` table populated on schedule.
- AI Service: prompt template, grounding payload construction (capped ~2000 tokens), OpenRouter integration.
- Backend: anomaly → grounding fetch → AI call → `ai_summaries` + alert pipeline (async, non-blocking).
- Alerts page unified across `threshold` + `ml_anomaly` sources.

**Exit criteria:** seeded anomaly scenario produces a grounded explanation traceable via `grounding_ref`. See `TESTING.md` §4–6.

## Phase 3 — Dashboard Completion

**Goal:** full Section-22-equivalent page set, polished UX.

- Process Explorer, AI Assistant (NL query UI), Health Score, Machine Search, Settings, System Logs.
- Full React Query integration — WS messages invalidate/update cache directly.
- `/settings` and `/logs` endpoints wired to admin-only UI.

**Exit criteria:** every page in `SYSTEM_ARCHITECTURE.md` / dashboard spec is functional against live backend data.

## Phase 4 — Deployment & Hardening

**Goal:** runs unattended across a real multi-PC lab.

- Production `.env` / `agent.config.json` finalized (see `DEPLOYMENT.md`).
- Client Agent packaged via PyInstaller, tested on 3+ physical Windows PCs.
- Rate limiting, TLS, JWT expiry, audit logging reviewed end-to-end (`SECURITY.md`).
- Load test: 100 simulated agents, verify NFR targets in `TESTING.md` §7.

**Exit criteria:** full lab deployment sustains a real class session without agent crash or degraded dashboard latency.

## Phase 5 — Future Enhancements (not required for capstone submission)

- 3D lab visualization (React Three Fiber).
- Remote remediation actions with admin confirmation + audit trail.
- Multi-lab federation, adaptive thresholds, expanded RBAC.

## Build Order Rationale

ML Engine is built in Phase 2 *before* wiring the AI Service to anything live, so the prediction/detection logic is validated on real data independent of LLM cost/latency. This also lets Phase 1 ship a fully working (if AI-less) monitoring product early, de-risking the OS-focused core of the grade before the AI layer is attempted.
