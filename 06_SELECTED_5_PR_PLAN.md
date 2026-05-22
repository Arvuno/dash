# 06_SELECTED_5_PR_PLAN.md — Dash PR Plan (Top 5)

## Selection Rationale

Five PRs are selected based on: impact (user-fixing value), feasibility (can be implemented without breaking changes), and alignment with the upstream project's direction (multi-DB routing is already an open PR, hardcoded model is a blocker).

| # | ID | Type | Impact | Effort |
|---|-----|------|--------|--------|
| 1 | Q-1 | Bug Fix | Critical | Low |
| 2 | Q-2 | Quality | Medium | Low |
| 3 | Q-3 | CI Fix | Medium | Low |
| 4 | Q-4 | Design Fix | Medium | Medium |
| 5 | P-2 | Feature | Medium | Low |

---

## PR 1: Fix hardcoded `gpt-5.4` model — make configurable via env var

### What
Extract the model ID into an environment variable `DASH_MODEL` with a safe default (`gpt-4o`).

### Files to change

**`dash/settings.py`**
```python
# Before
MODEL = OpenAIResponses(id="gpt-5.4")

# After
MODEL_ID = getenv("DASH_MODEL", "gpt-4o")
MODEL = OpenAIResponses(id=MODEL_ID)
```

**`evals/__init__.py`**
```python
# Before
JUDGE_MODEL = OpenAIResponses(id="gpt-5.4")

# After
from os import getenv
JUDGE_MODEL = OpenAIResponses(id=getenv("DASH_MODEL", "gpt-4o"))
```

**`evals/improve.py`** — Use Agno model instead of bare `OpenAI()` client, and read from env:
```python
# Before
from openai import OpenAI
client = OpenAI()
response = client.chat.completions.create(
    model="gpt-5.4",
    ...
)

# After
from agno.models.openai import OpenAIResponses
response_model = OpenAIResponses(id=getenv("DASH_MODEL", "gpt-4o"))
# ... use response_model.responses.create(...) or equivalent
```

Note: The `improve.py` logic may need to use `OpenAIResponses` which has a different API than bare `OpenAI`. The GPT-5.4 model name also won't work. For `improve.py`, use the env var `DASH_MODEL` (default `gpt-4o`) and ensure it's a model that supports JSON mode and function calling if needed.

**`example.env`** — Add `DASH_MODEL=gpt-4o` to the template.

**`README.md`** — Add `DASH_MODEL` to the Environment Variables table.

### Validation
```bash
# Should work without OPENAI_API_KEY set (will fail at API call but no import errors)
.venv/bin/python -c "from dash.settings import MODEL; print(MODEL)"
```

---

## PR 2: Align `improve.py` with Agno model pattern

### What
Refactor `evals/improve.py` to use `agno.models.openai.OpenAIResponses` consistently instead of the bare `openai.OpenAI` client.

### Why
- Consistent with the rest of the codebase
- Inherits Agno's proxy/config handling
- Avoids potential API incompatibilities

### Note
`OpenAIResponses` (the Responses API) has a different interface from `chat.completions.create`. If `improve.py` needs JSON mode, verify that `OpenAIResponses` supports it or fall back to `OpenAIResponses(id=...)` with appropriate settings. The simplest approach may be to use `Dash` team model via `agent.run()` pattern, but since `improve.py` is itself improving the system, using a separate direct model call is appropriate.

### Files to change
- `evals/improve.py` — Replace bare `OpenAI()` with `agno.models.openai.OpenAIResponses` using the same model env var from PR 1.

---

## PR 3: Fix CI — use `uv run mypy` in validate script

### What
Update `scripts/validate.sh` to use `uv run mypy` instead of bare `mypy`, ensuring it finds the package-installed version.

### Files to change

**`scripts/validate.sh`** (relevant lines):
```bash
# Before
> mypy /root/oss-pr-campaign/repos/dash --config-file pyproject.toml

# After
> uv run mypy /root/oss-pr-campaign/repos/dash --config-file pyproject.toml
```

Also update the comment header to match.

### Why this works
`uv run` executes the command in the context of the uv-managed environment where mypy is installed as part of the `--extra dev` dependencies.

### Validation
```bash
./scripts/format.sh  # Should pass
./scripts/validate.sh  # Should pass without "mypy: command not found"
```

---

## PR 4: Lazy initialization for knowledge/DB objects

### What
Avoid hitting the database at import time. Change `dash_knowledge`, `dash_learnings`, `dash_learning`, and `agent_db` to be lazily initialized on first access.

### Files to change

**`dash/settings.py`** — Replace module-level initialization with lazy factory:

```python
from functools import lru_cache

@lru_cache(maxsize=1)
def get_agent_db():
    return get_postgres_db()

@lru_cache(maxsize=1)
def get_dash_knowledge():
    return create_knowledge("Dash Knowledge", "dash_knowledge")

# ... etc

# Module-level refs become properties/cached calls
agent_db = get_agent_db()
```

But this still evaluates at import time. A true lazy approach:

```python
# Module-level: just the function
_agent_db = None

def get_agent_db():
    global _agent_db
    if _agent_db is None:
        _agent_db = get_postgres_db()
    return _agent_db
```

And in `dash/team.py` and `dash/agents/*.py`, use `get_agent_db()` function calls instead of the `agent_db` global.

However, the simplest pragmatic fix without major refactor is to keep the current approach but document that importing `dash` requires a DB. The real fix (lazy init across the board) is a larger refactor.

**Alternative (smaller scope):** Document the DB requirement clearly. Add a try/except around knowledge creation so the module loads even without DB, falling back to None. Agents gracefully degrade.

**Chosen approach:** Go with the global-guard pattern — wrap knowledge/DB initialization in try/except, set to None on failure, and have each usage site check for None.

```python
# dash/settings.py
try:
    agent_db = get_postgres_db()
    dash_knowledge = create_knowledge("Dash Knowledge", "dash_knowledge")
    dash_learnings = create_knowledge("Dash Learnings", "dash_learnings")
    dash_learning = LearningMachine(knowledge=dash_learnings, learned_knowledge=LearnedKnowledgeConfig(mode=LearningMode.AGENTIC))
except Exception:
    agent_db = None
    dash_knowledge = None
    dash_learnings = None
    dash_learning = None
```

Then in each agent definition, check if knowledge is available before passing to the Agent constructor.

---

## PR 5: Make `DASH_SCHEMA` configurable via env var

### What
Replace the hardcoded `DASH_SCHEMA = "dash"` in `db/session.py` with an env var `DASH_SCHEMA` defaulting to `"dash"`.

### Files to change

**`db/session.py`**
```python
from os import getenv

DASH_SCHEMA = getenv("DASH_SCHEMA", "dash")
```

**`example.env`** — Add `DASH_SCHEMA=dash` to the template.

**`README.md`** — Add `DASH_SCHEMA` to the Environment Variables table.

### Why this matters
Users who already have a schema named "dash" for other purposes can rename Dash's schema without patching code. This is a simple change with clear user benefit.

### Validation
```bash
DASH_SCHEMA=my_schema .venv/bin/python -c "from db.session import DASH_SCHEMA; print(DASH_SCHEMA)"
# Should print: my_schema
```

---

## Implementation Order

1. **PR 1 (Q-1)** — Blocked only by bad model ID, unblocks all testing
2. **PR 3 (Q-3)** — CI fix, easy to validate
3. **PR 5 (P-2)** — Small, isolated, easy
4. **PR 2 (Q-2)** — Depends on PR 1 (uses same env var)
5. **PR 4 (Q-4)** — Larger refactor, do last

---

## Out of Scope (for this batch)

- **Multi-DB routing (P-1/issue #12)** — Large feature, requires significant DB session refactor, good as a follow-up
- **OpenRouter fallback (P-3/PR #11)** — PR already exists; needs Q-1 fix applied first (model name)
- **VectorChord (P-4/issue #6)** — Low priority
- **Silent wrong answer (issue #15)** — Design discussion needed, not a code PR
