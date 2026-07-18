# Testing

## Strategy

| Layer | Method |
|---|---|
| Backend | `pytest` — unit tests per router/service, integration tests against a test Supabase project |
| ML Engine | `pytest` — unit tests on synthetic time-series fixtures (known trend, known anomaly) |
| Frontend | `vitest` + React Testing Library — component tests; manual QA for WS-driven live views |
| Client Agent | `pytest` — sampler output shape, buffer eviction logic, backoff timing (mocked clock) |
| End-to-end | Manual + scripted load test — see Acceptance Criteria below |

CI runs backend/frontend unit tests on every push; see `CONTRIBUTING.md` and `DEPLOYMENT.md` §CI.

## Acceptance Criteria (Numbered — map 1:1 to `TASKS.md` exit checks)

1. **Registration:** fresh agent registers, appears in `GET /machines` within one heartbeat interval.
2. **Reconnect resilience:** kill network for 60s mid-stream → buffered samples appear in `metric_samples` in correct order post-reconnect, none dropped (buffer not exceeded).
3. **Threshold alert:** force CPU > `CPU_THRESHOLD_PCT` for 3 consecutive samples → `alerts` row with `source='threshold'` created within one sampling interval.
4. **ML anomaly:** seed synthetic memory-leak pattern (steady linear increase) → `is_anomalous=true` predictions row appears before value crosses the static threshold.
5. **Grounded explanation:** anomaly alert triggers an `ai_summaries` row whose `grounding_ref` IDs all exist in `metric_samples`/`process_samples`.
6. **NL query grounding:** `POST /ai/query` answer's `grounding_ref` resolves to non-empty, relevant rows for at least 5 varied test questions.
7. **Load:** 100 mock agents sustained 10 minutes → p95 dashboard update latency ≤ 3s.
8. **Security:** request with expired/forged JWT → 401 on all protected endpoints; agent token cannot read another machine's data (403/404).

## Non-Functional Targets

| Metric | Target | Verification |
|---|---|---|
| Dashboard update latency | ≤ 3s p95, sample → WS push | Load test, 20 simulated agents |
| Agent CPU overhead | < 2% avg over 10 min | `psutil` self-measurement on agent host |
| Concurrent agent connections | 100+ without latency breach | Load test, mock agents |
| Ingestion write batch flush | ≤ 2s or 100 rows | Backend metrics log |
| ML eval cycle | Completes within `ML_EVAL_INTERVAL_SECONDS` for 100 machines | Backend timing log |
| AI explanation turnaround | ≤ 10s p95, anomaly → `ai_explanation` push (excl. OpenRouter outage) | Timestamp diff, `ai_summaries` vs `alerts.created_at` |

## Test Data / Fixtures

- **Synthetic memory-leak fixture:** linearly increasing `mem_pct` over 200+ samples, used for AC #4. Lives in `backend/tests/fixtures/mem_leak_series.json`.
- **Mock agent script:** `scripts/mock_agent.py` — spins up N fake WS clients emitting randomized-but-plausible telemetry, used for AC #7.
- **NL query test set:** 5+ varied questions in `backend/tests/fixtures/nl_queries.json`, covering current-state, historical, and comparative questions, used for AC #6.

## Manual QA Checklist (pre-demo)

- [ ] All dashboard pages load without console errors against live backend
- [ ] Live Metrics updates visibly within 3s of a real agent sample
- [ ] Alerts page shows both `threshold` and `ml_anomaly` sourced alerts distinctly
- [ ] AI Assistant answers a live question with a citation the admin can verify
- [ ] Health Score changes visibly when a machine crosses into `warning`/`critical`
