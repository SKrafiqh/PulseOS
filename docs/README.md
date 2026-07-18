# AI-Driven Distributed Computer Lab Monitoring System

Operating Systems capstone project — 70% Operating Systems, 30% AI enhancement. Monitors CPU scheduling, process/thread state, memory (incl. virtual memory), disk I/O, and network I/O across a Windows computer lab from one dashboard, with a scikit-learn ML layer for prediction/anomaly detection and an OpenRouter LLM layer for grounded explanation only.

## Documentation Map

| Doc | Contents |
|---|---|
| `PRD.md` | Full product requirements, scope, OS concept mapping |
| `TRD.md` | Exact technical contracts (schemas, params, acceptance criteria) |
| `SYSTEM_ARCHITECTURE.md` | Layered architecture, data flow, sequence diagrams |
| `DATABASE_SCHEMA.md` | Table definitions, indexes, relationships |
| `API_SPEC.md` | REST + WebSocket contracts, error codes |
| `IMPLEMENTATION_PLAN.md` | Phased build plan |
| `TASKS.md` | Granular task checklist |
| `DEPLOYMENT.md` | Environments, deploy steps |
| `SECURITY.md` | Auth, transport, rate limiting |
| `TESTING.md` | Test plan, acceptance criteria |
| `CONTRIBUTING.md` | Branching, commits, PR process |
| `CHANGELOG.md` | Version history |

## Architecture (one paragraph)

Windows **Client Agent** (`psutil`, PyInstaller `.exe`) streams OS telemetry over WebSocket to a **FastAPI backend**, which persists it to **Supabase PostgreSQL**. A **scikit-learn ML Engine** predicts short-term CPU/memory trends and flags anomalies (Isolation Forest) directly from that telemetry. Only when the ML layer flags something does an **OpenRouter LLM** get involved — strictly to explain, summarize, or answer questions, grounded in the exact data that triggered it. A **React/TypeScript dashboard** surfaces all of it live. Full detail: `SYSTEM_ARCHITECTURE.md`.

## Repository Structure

```
ai-lab-monitor/
├─ client-agent/    backend/    frontend/
├─ ml-engine/        ai-service/
├─ database/         shared/    config/    scripts/
└─ docs/             .github/
```

## Quick Start (Development)

**Prerequisites:** Python 3.11+, Node 18+, a Supabase project, an OpenRouter API key.

```bash
# 1. Database
supabase login
supabase link --project-ref <your-project-ref>
supabase db push        # applies database/migrations/

# 2. Backend
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp ../config/dev.env .env    # fill in Supabase + OpenRouter keys
uvicorn app.main:app --reload

# 3. Frontend
cd ../frontend
npm install
npm run dev

# 4. Client Agent (on a Windows test machine)
cd ../client-agent
pip install -r requirements.txt
python agent_main.py
```

Details and full env var list: `DEPLOYMENT.md`.

## Status

Phase 1 (Core Monitoring) — see `TASKS.md` for live checklist and `CHANGELOG.md` for release history.

## License

Academic project — SIMATS Engineering, B.Tech CSE (AI & Data Science) capstone submission.
