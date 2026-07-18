# Contributing

Single-contributor academic capstone, but this repo follows standard team-project conventions so it reads as production-realistic and stays organized across the project's phases.

## Branching

- `main` — always deployable, matches the current Phase exit criteria in `IMPLEMENTATION_PLAN.md`.
- `feature/<short-description>` — one branch per task in `TASKS.md` (e.g. `feature/isolation-forest-anomaly`).
- `fix/<short-description>` — bug fixes.

## Commit Messages

Conventional Commits format:

```
<type>(<scope>): <short summary>

feat(ml-engine): add Isolation Forest anomaly scoring
fix(backend): correct threshold comparison operator (was <=, should be <)
docs(api-spec): add rate limit table
chore(ci): pin scikit-learn version
```

Types: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`.

## Pull Request Checklist

- [ ] Linked to a `TASKS.md` item or GitHub Issue.
- [ ] Unit tests added/updated for changed logic.
- [ ] If schema changed: `DATABASE_SCHEMA.md` updated + migration file added.
- [ ] If API contract changed: `API_SPEC.md` updated.
- [ ] `CHANGELOG.md` entry added under `[Unreleased]`.
- [ ] CI passing (`.github/workflows/ci.yml`).

## Code Style

- Python: `black` + `ruff`, type hints required on public functions.
- TypeScript: `eslint` + `prettier`, strict mode on.
- No secrets in code or commit history — see `SECURITY.md` §Secrets Management.

## Adding a New Metric or Alert Rule

Per `IMPLEMENTATION_PLAN.md`'s modularity goal, this should require touching only:
1. Client Agent sampler (collect it).
2. `metric_samples`/`process_samples` schema if a new column is needed (`DATABASE_SCHEMA.md` + migration).
3. Threshold engine config (`configurations` table) if it's threshold-based, or ML feature set (`SYSTEM_ARCHITECTURE.md` §ML Engine) if it's anomaly-based.
4. Dashboard chart/table binding.

If a change touches more than these four points, flag it — it likely violates the module boundaries this project is built on.

## Reporting Issues

Use GitHub Issues, one issue per `TASKS.md` line item where practical. Label with phase (`phase-1`, `phase-2`, ...) and component (`client-agent`, `backend`, `ml-engine`, `ai-service`, `frontend`, `docs`).
