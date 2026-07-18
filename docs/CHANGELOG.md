# Changelog

Format follows [Keep a Changelog](https://keepachangelog.com/). This project uses documentation-version tags (`docs-vX`) since implementation has not yet shipped a runnable release.

## [Unreleased]

### Added
- Nothing yet — Phase 1 implementation in progress (see `TASKS.md`).

## [docs-v3] — 2026-07-18

### Added
- Split monolithic PRD/TRD into dedicated docs: `DATABASE_SCHEMA.md`, `API_SPEC.md`, `SYSTEM_ARCHITECTURE.md`, `IMPLEMENTATION_PLAN.md`, `TASKS.md`, `README.md`, `DEPLOYMENT.md`, `SECURITY.md`, `TESTING.md`, `CONTRIBUTING.md`.
- `configurations`, `users`, `sessions` tables added to schema.
- Rate limiting, CORS, and secrets-management guidance added to `SECURITY.md`.

### Changed
- DDL table creation order corrected: `users` must precede `configurations` (FK dependency) — was previously listed in reverse in the PRD narrative.

## [docs-v2] — 2026-07-18 (TRD.md)

### Added
- Technical Requirements Document: exact WebSocket/REST schemas, full DDL, ML model parameters (Linear Regression lag features, Isolation Forest `contamination=0.05`), AI Service strict input/output contract with grounding token cap, error code table, measurable non-functional targets, 8 numbered acceptance tests.

## [docs-v1] — 2026-07-18 (PRD v3)

### Added
- Full architectural review pass correcting the LLM/ML responsibility split: OpenRouter LLM may explain/summarize/recommend/answer only; scikit-learn ML Engine owns all prediction and anomaly detection.
- Dedicated "Operating System Concepts Mapping" chapter.
- Supabase platform breakdown (Postgres / Auth / Realtime / REST / Storage) replacing generic "PostgreSQL" references.
- Expanded Client Agent design (config file, retry queue, low-CPU-footprint targets).
- Mermaid architecture, data-flow, and sequence diagrams.
- Scalability section targeting 100+ concurrent machines.
- Phased milestones (Phase 1–5) replacing flat milestone list.
- Antigravity/MCP implementation notes (Supabase MCP, GitHub MCP, Stitch MCP).

### Changed
- 3D Lab Visualization demoted from core feature to Phase 2/5 future scope.

## [pdf-v2] — 2026-07-18 (initial docx PRD, v2)

### Added
- First architecturally-corrected PRD establishing ML-before-LLM ordering, with embedded architecture/sequence/deployment diagrams (rendered as images in the `.docx`).

## [pdf-v1] — 2026-07-18 (initial docx PRD, v1)

### Added
- Original PRD draft: problem statement, objectives, scope, functional/non-functional requirements, initial (later corrected) architecture with OpenRouter positioned for anomaly detection.
