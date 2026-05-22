# 05_PR_CANDIDATES.md — Dash PR Candidates

## Quality Issues Found

### Q-1: Hardcoded `gpt-5.4` model — likely wrong/broken (High Priority)

**Locations:**
- `dash/settings.py:20` — `MODEL = OpenAIResponses(id="gpt-5.4")`
- `evals/__init__.py:15` — `JUDGE_MODEL = OpenAIResponses(id="gpt-5.4")`
- `evals/improve.py:173` — `model="gpt-5.4"` (plain string for `OpenAI()` client)

**Problem:** GPT-5 has not been released as of May 2026. The model `gpt-5.4` does not exist in OpenAI's API. Dash cannot function without a valid model ID. Even if a future GPT-5 model is released, hardcoding the model ID means users cannot configure their own model.

**Impact:** Every agent (Leader, Analyst, Engineer, judge model in evals, improvement model) uses this non-existent model. The system is completely non-functional out of the box.

**Fix:** Extract to env var `DASH_MODEL` with a safe default like `gpt-4o`. Use the env var in all three locations.

---

### Q-2: `improve.py` uses bare `OpenAI()` client instead of Agno's model (Medium Priority)

**Location:** `evals/improve.py:166-173`

```python
from openai import OpenAI
client = OpenAI()
response = client.chat.completions.create(
    model="gpt-5.4",
    ...
)
```

**Problem:** Inconsistent with the rest of the codebase which uses `agno.models.openai.OpenAIResponses`. The bare `OpenAI()` client also doesn't respect proxy settings, token management, or other Agno-level configurations.

**Fix:** Use `OpenAIResponses(id=...)` from Agno instead, aligned with Q-1 fix.

---

### Q-3: CI workflow missing `dev` dependencies for lint/typecheck

**Location:** `.github/workflows/validate.yml:38`

```yaml
- name: Install the project with dev dependencies
  run: uv sync --extra dev
```

**Problem:** The `pyproject.toml` defines `dev` extras with `mypy`, `python-dotenv`, and `ruff`. The CI step `uv sync --extra dev` should install these. However, the `mypy` step fails because `mypy` is not in PATH — it's installed in the uv-managed Python environment, not globally. The validate script (`scripts/validate.sh`) calls `mypy` directly without using `uv run`.

**Fix:** Use `uv run mypy` instead of `mypy` in `validate.sh`, or add mypy to PATH via uv. Also worth verifying the `--extra dev` install actually works in CI.

---

### Q-4: `build_analyst_instructions()` hits DB on import — no lazy init

**Location:** `dash/settings.py:28` — module-level `create_knowledge()` call

```python
dash_knowledge = create_knowledge("Dash Knowledge", "dash_knowledge")
```

This calls `get_postgres_db()` → `PgVector(...).create()` → connects to DB on import.

**Problem:** Importing `dash.settings` (and thus `db.session`, `db/__init__`) triggers a DB connection. This means even importing the package for CLI help or type-checking requires a running DB. The smoke tests also can't be imported without a DB.

**Fix:** Use lazy initialization — create knowledge objects on first access rather than at module load time.

---

### Q-5: Business rule has incorrect `status = 'active'` filter

**Location:** `knowledge/business/metrics.json:6-8`

```json
"calculation": "SUM(mrr) FROM subscriptions WHERE status = 'active'"
```

**Problem:** The gotcha in the same file says active subscriptions use `ended_at IS NULL`, not `status = 'active'`. The metrics definition contradicts the gotcha. This could cause the agent to generate incorrect queries.

**Fix:** Update the calculation to `SUM(mrr) FROM subscriptions WHERE ended_at IS NULL` to be consistent with the gotcha.

---

## PR Candidates from Issues

### P-1: Issue #12 — Decouple agent/knowledge DB from data DB (High Value, Medium Effort)

**Issue:** Single Postgres used for runtime state, knowledge vectors, AND the analytics data DB. Users want to point Dash at a ClickHouse or other OLAP store for analytics while keeping Agno state on Postgres.

**Why valuable:** Opens Dash to production environments where analytics data lives in specialized warehouses. Would significantly broaden adoption.

**Approach:** Split `db/session.py` so `DB_*` vars control agent/knowledge storage, while new `ANALYTICS_DB_*` vars control data queries. This is exactly what PR #8 (issue #8) attempts.

---

### P-2: Issue #7 — User-configurable custom SQL schema (Medium Value, High Effort)

**Issue:** The `dash` schema name is hardcoded. Users may want to use a different schema name.

**Approach:** Make `DASH_SCHEMA` configurable via env var. The `db/session.py` already has `DASH_SCHEMA = "dash"` as a constant — it just needs to be read from env.

---

### P-3: Issue #11 — OpenAI-first provider selection with OpenRouter fallback (Medium Value, Low Effort)

**PR #11 exists but is open.** It adds `OPENROUTER_API_KEY` fallback when `OPENAI_API_KEY` is not set.

**Note:** This PR was authored before the `gpt-5.4` problem was identified. If merged, it would make the fallback use `gpt-5.4` as well. The model selection in that PR needs review.

---

### P-4: Issue #6 — VectorChord as PGVector replacement (Low Priority, Low Effort)

**Issue:** VectorChord is a PostgreSQL extension that could replace `pgvector` for some use cases.

**Effort:** Likely involves adding a new vector DB option in `db/session.py` and making it configurable.

---

## Summary Table

| ID | Type | Priority | Effort | Description |
|----|------|----------|--------|-------------|
| Q-1 | Bug | Critical | Low | Hardcoded `gpt-5.4` — make model configurable |
| Q-2 | Quality | Medium | Low | `improve.py` uses bare `OpenAI()` client |
| Q-3 | CI | Medium | Low | mypy not in PATH in CI/validate script |
| Q-4 | Design | Medium | Medium | DB connection on import — use lazy init |
| Q-5 | Bug | Low | Low | `status = 'active'` contradicts gotcha |
| P-1 | Feature | High | High | Multi-DB routing (issue #12) |
| P-2 | Feature | Medium | Low | Make `DASH_SCHEMA` configurable (issue #7) |
| P-3 | Feature | Medium | Low | OpenRouter fallback (PR #11 exists) |
| P-4 | Feature | Low | Low | VectorChord support (issue #6) |