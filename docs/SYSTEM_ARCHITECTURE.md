# System Architecture

Operating Systems capstone project. 70% OS / 30% AI enhancement. Full requirements: `../PRD.md`. Exact contracts: `API_SPEC.md`, `DATABASE_SCHEMA.md`.

## Layered Architecture

Six layers: distributed client → application (backend) → data platform (Supabase) → deterministic ML → explanatory LLM → presentation. Non-negotiable ordering: **raw telemetry → ML prediction/detection → LLM explanation**. The LLM never receives raw telemetry to predict from — only ML output plus grounding context to narrate.

```mermaid
flowchart TB
    subgraph CLIENT["Distributed Client Layer"]
        A1["Windows PC 1\nClient Agent (.exe)"]
        A2["Windows PC N\nClient Agent (.exe)"]
    end
    subgraph APP["Application Layer"]
        B["FastAPI Backend\nWebSocket Ingestion · REST · JWT Auth\nThreshold Engine · Orchestrator · Workers"]
    end
    subgraph DATA["Supabase Platform"]
        C1["PostgreSQL: machines, telemetry,\nalerts, audit_logs, users, sessions"]
        C2["Supabase Auth"]
        C3["predictions & ai_summaries"]
    end
    subgraph ML["ML Layer (scikit-learn)"]
        D["Linear Regression · Isolation Forest\n(Prediction + Anomaly Detection)"]
    end
    subgraph AI["AI Layer (OpenRouter)"]
        E["Explain · Summarize · Recommend\n(NEVER predicts or detects)"]
    end
    F["React Dashboard"]

    A1 & A2 -- "WSS: telemetry + heartbeat" --> B
    B -- writes --> C1
    B -- auth --> C2
    B -- "reads history" --> D
    D -- "stores" --> C3
    D -- "anomaly + forecast" --> E
    E -- "reads grounding context" --> C1
    E -- "stores explanation" --> C3
    B -- "REST + WS" --> F
    C3 --> F
```

## Component Responsibilities

| Component | Tech | Responsibility |
|---|---|---|
| Client Agent | Python + psutil, PyInstaller | Collect OS telemetry, register identity, heartbeat, buffer/retry |
| Backend | FastAPI + Uvicorn | WS ingestion, REST, JWT auth, threshold engine, orchestration, workers |
| Data Platform | Supabase (Postgres + Auth + Realtime + REST) | System of record — see §Supabase Role |
| ML Engine | scikit-learn | Prediction (Linear Regression) + anomaly detection (Isolation Forest) |
| AI Service | OpenRouter LLM | Grounded explanation, summaries, NL query — no prediction |
| Dashboard | React + TS + React Query | All UI views |

## Data Flow

```mermaid
flowchart LR
    A["Client Agent\n(psutil, 2-5s)"] --> B["WS Ingestion"]
    B --> C["Threshold Engine\n(sync)"]
    C --> D["Supabase Postgres\n(persist)"]
    D --> E["ML Engine\n(async predict+detect)"]
    E --> F["AI Service\n(async explain)"]
    F --> G["Dashboard\n(WS push)"]
```

Threshold alerts fire synchronously in-line; ML + LLM stages run asynchronously via background workers and never block ingestion.

## Sequence: Registration → Heartbeat → Telemetry

```mermaid
sequenceDiagram
    participant CA as Client Agent
    participant BE as Backend
    participant DB as Supabase Postgres
    CA->>BE: Register (machine_id, auth token)
    BE->>DB: Upsert machine record
    BE-->>CA: 200 OK + session token
    CA->>BE: Heartbeat (every 10s)
    CA->>BE: Telemetry batch
    BE->>DB: Insert metric/process samples
    BE-->>CA: Ack
    Note over CA: [connection drops]
    CA->>CA: Buffer locally, backoff retry
    CA->>BE: Reconnect + flush buffer
```

## Sequence: Anomaly → ML → LLM → Alert

```mermaid
sequenceDiagram
    participant BE as Backend
    participant ML as ML Engine
    participant DB as Supabase Postgres
    participant AI as AI Service
    participant UI as Dashboard
    BE->>ML: Scheduled eval tick
    ML->>DB: Fetch history
    ML->>ML: Predict + Isolation Forest check
    ML->>DB: Store predictions row
    alt anomaly_score > threshold
        ML-->>BE: Emit anomaly event
        BE->>DB: Fetch grounding context
        BE->>AI: Request explanation (grounded)
        AI-->>BE: Root cause + recommendation
        BE->>DB: Store ai_summaries + alert
        BE->>UI: Push alert + explanation
    end
```

## Supabase Role

| Capability | Used For | Phase 1 status |
|---|---|---|
| PostgreSQL | System of record, all tables | Core |
| Auth | Admin/faculty login, JWT issuance | Core |
| Realtime | Supplemental push for `alerts` table changes | Selective — primary telemetry still on custom WS for sampling-cadence control |
| REST (PostgREST) | Internal tooling/debugging | Supporting |
| Storage | Exported reports, Phase 2 lab layout images | Future scope, not used |

Under the hood the engine is standard PostgreSQL — schema/indexing decisions (`DATABASE_SCHEMA.md`) are ordinary relational design, not Supabase-specific.

## Operating System Concepts Mapping

Core academic justification — every feature maps to an OS concept.

| Feature | OS Concept |
|---|---|
| CPU utilization monitoring | CPU Scheduling |
| Process enumeration | Process Management |
| Thread count | Multithreading / Thread Scheduling |
| Memory usage tracking | Memory Allocation |
| Swap monitoring | Virtual Memory |
| Disk I/O throughput | Disk Scheduling |
| File/directory access | File System Management |
| Network throughput | I/O Management |
| Agent metric collection via psutil | System Calls / Kernel Interfaces |
| Multi-machine telemetry | Network Communication / Distributed Systems |
| Persisted time-series | Resource Accounting |
| Concurrent WS handling | Concurrency / I/O Multiplexing |
| Threshold engine | Real-Time Resource Monitoring |
| Registration + heartbeat | Process/Device State Management |
| Local buffer + retry queue | Producer-Consumer Buffering |
| Background workers | Job Scheduling / Batch Processing |

## ML / LLM Boundary Enforcement

Backend orchestrator exposes two distinct interfaces:
- `ml.predict(machine_id)` / `ml.detect_anomaly(machine_id)` — ML Engine only.
- `ai.explain(anomaly)` / `ai.summarize(scope)` / `ai.answer_query(text)` — AI Service only.

No interface lets a caller request a prediction from the AI Service. Enforced at the API contract level, not convention. See `SECURITY.md` for credential isolation between services.
