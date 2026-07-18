# Security

## Authentication

| Actor | Mechanism | Detail |
|---|---|---|
| Admin/faculty (dashboard) | JWT via Supabase Auth | HS256, `JWT_SECRET` env, `exp` = `JWT_EXPIRY_MINUTES` (default 60). Claims: `{sub: user_id, role, exp}`. |
| Client Agent | Per-machine bearer token | Random 32-byte token issued at `/machines/register`, stored server-side as SHA-256 hash in `machines.auth_token_hash`. Never stored in plaintext. |
| Passwords | Delegated to Supabase Auth | bcrypt hashing, not implemented in-house. |

## Transport Security

- TLS enforced on **all** connections: agent↔backend, backend↔dashboard, backend↔Supabase, backend↔OpenRouter.
- WebSocket connections use **WSS only** — plain `ws://` rejected in production config.
- Plain `http://` rejected in production config; TLS terminated at reverse proxy or Uvicorn SSL context.

## Credential Isolation

- OpenRouter and Supabase service-role keys held **server-side only** — never exposed to the client agent binary or the dashboard JS bundle.
- Dashboard uses the Supabase **anon key** only, scoped by row-level security where applicable; privileged operations go through the backend, not directly to Supabase from the browser.

## Authorization / Least Privilege

- A machine token can only write telemetry for **its own** `machine_id` — attempting to write another machine's data returns 403/404, verified by `TESTING.md` §8.
- `role: admin` required for `PUT /settings` and `GET /logs`; `role: faculty` is read-only across all endpoints.

## Rate Limiting

Token-bucket per credential:

| Credential | Limit |
|---|---|
| Dashboard JWT | 60 requests/minute |
| Agent registration token | 1 request/minute |

Exceeding either returns `429 RATE_LIMITED` (see `API_SPEC.md`).

## Audit Logging

Every alert, prediction event, and AI-generated explanation is recorded to `audit_logs` (`action`, `actor`, `reference_id`, `created_at`) — see `DATABASE_SCHEMA.md`. This is what makes AI output traceable: every `ai_summaries.grounding_ref` can be cross-checked against the exact rows it claims to be grounded in.

## AI-Specific Security Notes

- The AI Service's input contract is capped (~2,000 tokens of grounding context) and structurally cannot request a prediction — see `SYSTEM_ARCHITECTURE.md` §ML/LLM Boundary Enforcement. This isn't just an accuracy safeguard, it's also a cost/DoS control: an unbounded prompt from untrusted telemetry could otherwise be used to run up API cost.
- LLM output is never executed or used to trigger an action automatically — it is display-only text on the dashboard (no remote-execution feature exists in Phase 1; see `PRD.md` Future Scope for the gated Phase 2+ proposal).

## CORS

Dashboard origin explicitly allow-listed in backend config. No wildcard `*` origin in production.

## Secrets Management

- Real secrets live only in `config/prod.env` / `config/dev.env` (gitignored). Only `.env.example` (with placeholder values) is committed.
- Rotate `JWT_SECRET` and machine tokens if a leak is suspected; rotating `JWT_SECRET` invalidates all active dashboard sessions immediately (by design).

## Known Limitations (Phase 1)

- No RBAC beyond `admin`/`faculty` — see `PRD.md` Future Scope for planned expansion.
- No automatic secret rotation — manual process only.
- Rate limiting is in-process (per backend instance) — not yet distributed; acceptable at single-admin-host scale (Section: `SYSTEM_ARCHITECTURE.md` deployment topology).
