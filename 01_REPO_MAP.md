# 01_REPO_MAP.md — Dash Repository Structure

## Top-Level Files

```
dash/
├── README.md              # Project overview, quick start, architecture docs
├── CLAUDE.md              # Full project context (read this first!)
├── CONTRIBUTING.md        # Contribution guidelines
├── LICENSE                # Apache-2.0
├── pyproject.toml         # Python package config, ruff/mypy settings
├── requirements.txt       # Pinned dependencies
├── example.env            # Template for .env
├── railway.json           # Railway deployment config
├── Dockerfile             # Multi-stage Docker build
├── compose.yaml           # Docker Compose (api + db)
├── .dockerignore
├── .gitignore
└── .github/workflows/validate.yml   # CI: ruff + mypy (no DB tests)
```

## Directory Structure

```
dash/
├── app/
│   ├── __init__.py
│   ├── main.py            # FastAPI + AgentOS entry point (teams, scheduler, Slack)
│   └── config.yaml        # Quick prompts
│
├── dash/                  # Core agent logic
│   ├── __init__.py
│   ├── __main__.py        # CLI: python -m dash
│   ├── team.py            # Dash Team (leader, coordinate mode)
│   ├── settings.py        # MODEL, DB, Slack, knowledge/learning config
│   ├── paths.py           # Path constants (TABLES_DIR, BUSINESS_DIR, etc.)
│   ├── instructions.py    # System prompt builders (Leader, Analyst, Engineer)
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── analyst.py     # Analyst agent definition
│   │   └── engineer.py    # Engineer agent definition
│   ├── context/
│   │   ├── __init__.py
│   │   ├── semantic_model.py   # Table metadata → system prompt
│   │   └── business_rules.py   # Business rules/gotchas → system prompt
│   └── tools/
│       ├── __init__.py
│       ├── build.py       # Tool factory per agent (analyst vs engineer)
│       ├── introspect.py   # Runtime schema inspection tool
│       ├── save_query.py   # save_validated_query tool
│       └── update_knowledge.py  # update_knowledge tool
│
├── db/
│   ├── __init__.py        # Re-exports: db_url, get_postgres_db, get_sql_engine, etc.
│   ├── session.py         # Dual-engine setup, schema guards, knowledge DB
│   └── url.py             # Database URL builder from env vars
│
├── knowledge/             # Loaded into vector DB
│   ├── tables/            # Table metadata JSON (SaaS metrics: 6 tables)
│   │   ├── customers.json
│   │   ├── subscriptions.json
│   │   ├── plan_changes.json
│   │   ├── invoices.json
│   │   ├── usage_metrics.json
│   │   └── support_tickets.json
│   ├── queries/
│   │   └── common_queries.sql   # 10 validated query patterns
│   └── business/
│       └── metrics.json   # Metrics defs + business rules + gotchas
│
├── evals/                 # Evaluation framework
│   ├── __init__.py        # CATEGORIES dict + JUDGE_MODEL
│   ├── __main__.py        # CLI: python -m evals [--category X] [--verbose]
│   ├── run.py             # Unified eval runner (5 categories)
│   ├── smoke.py           # Smoke tests (26 tests, 8 groups)
│   ├── improve.py         # Self-improvement runner (uses CLAUDE.md)
│   └── cases/
│       ├── __init__.py
│       ├── accuracy.py    # AccuracyEval cases
│       ├── routing.py     # ReliabilityEval cases
│       ├── security.py    # AgentAsJudgeEval (binary)
│       ├── governance.py  # AgentAsJudgeEval (binary)
│       └── boundaries.py  # AgentAsJudgeEval (binary)
│
├── scripts/               # DevOps / deployment
│   ├── __init__.py
│   ├── venv_setup.sh      # Create .venv with uv
│   ├── format.sh          # ruff format + isort
│   ├── validate.sh        # ruff check + mypy
│   ├── generate_requirements.sh  # uv pip compile → requirements.txt
│   ├── generate_data.py   # Generate SaaS sample data
│   ├── load_knowledge.py  # Load knowledge into vector DB
│   ├── build_image.sh     # Multi-platform Docker build
│   ├── entrypoint.sh      # Docker entrypoint (DB wait, banner)
│   ├── railway_up.sh      # First-time Railway setup
│   ├── railway_redeploy.sh  # Redeploy to Railway
│   └── railway_env.sh    # Sync .env.production to Railway
│
└── docs/
    ├── IMPROVE_DASH.md    # Self-improvement prompt for Claude Code
    ├── SLACK_CONNECT.md   # Slack app setup guide
    └── TEST_QUESTIONS.md  # Manual test questions
```

## Codebase Map (Key Symbols)

| Module | Symbol | Purpose |
|--------|--------|---------|
| `dash.team` | `dash` | Team instance (coordinate mode, leader) |
| `dash.team` | `leader_tools` | SlackTools list (only if SLACK_TOKEN) |
| `dash.agents.analyst` | `analyst` | Analyst agent instance |
| `dash.agents.engineer` | `engineer` | Engineer agent instance |
| `dash.instructions` | `build_leader_instructions()` | Composes leader prompt with Slack context |
| `dash.instructions` | `build_analyst_instructions()` | Composes Analyst prompt with semantic model + business context |
| `dash.instructions` | `build_engineer_instructions()` | Composes Engineer prompt with semantic model |
| `dash.settings` | `MODEL` | `OpenAIResponses(id="gpt-5.4")` |
| `dash.settings` | `agent_db` | `PostgresDb` instance |
| `dash.settings` | `dash_knowledge` | Knowledge base (dash_knowledge table, pgvector) |
| `dash.settings` | `dash_learnings` | Learning base (dash_learnings table, pgvector) |
| `dash.settings` | `dash_learning` | `LearningMachine(mode=AGENTIC)` |
| `dash.tools.build` | `build_analyst_tools()` | Returns `[SQLTools(ro), introspect, save_query, reasoning]` |
| `dash.tools.build` | `build_engineer_tools()` | Returns `[SQLTools(rw, schema=dash), introspect, update_knowledge, reasoning]` |
| `db.session` | `get_sql_engine()` | Cached engine with `search_path=dash,public` + schema guard |
| `db.session` | `get_readonly_engine()` | Cached engine with `default_transaction_read_only=on` |
| `db.session` | `DASH_SCHEMA` | `"dash"` |
| `db.session` | `_PUBLIC_WRITE_RE` | Regex blocking DDL/DML to public schema |
| `db.session` | `create_knowledge()` | Creates Knowledge + PgVector + PostgresDb |
| `evals` | `CATEGORIES` | Dict mapping category → runner config |
| `evals` | `JUDGE_MODEL` | `OpenAIResponses(id="gpt-5.4")` |

## Data Model (public schema)

6 tables in the SaaS metrics dataset:

| Table | Key Columns |
|-------|-------------|
| `customers` | id, company_name, industry, company_size, source, signup_date, status |
| `subscriptions` | id, customer_id, plan, mrr, seats, billing_cycle, status, started_at, ended_at, cancellation_reason |
| `plan_changes` | id, customer_id, change_type, previous_mrr, new_mrr, changed_at |
| `invoices` | id, customer_id, amount, status, issued_at, due_at |
| `usage_metrics` | id, customer_id, metric_date, api_calls, active_users, storage_gb, reports_generated |
| `support_tickets` | id, customer_id, priority, category, created_at, resolved_at, satisfaction_score |

Key gotchas baked into knowledge:
- `subscriptions.ended_at IS NULL` = still active (not `status='active'`)
- `support_tickets.satisfaction_score` ~30% NULL
- Usage metrics sampled 3-5x/month, not daily
- Annual billing: `amount = mrr * 12 * 0.9`

## Eval Categories

| Category | Eval Type | Cases | What It Tests |
|----------|-----------|-------|---------------|
| security | AgentAsJudgeEval (binary) | 4 | No credential/secret leaks |
| governance | AgentAsJudgeEval (binary) | 3 | Refuses destructive SQL |
| boundaries | AgentAsJudgeEval (binary) | 3 | Schema access boundaries |
| routing | ReliabilityEval | 5 | Routes to correct agent/tools |
| accuracy | AccuracyEval (1-10) | 4 | Data correctness + insights |

Smoke tests: 26 tests across 8 groups (warmup, simple_data, metrics, data_quality, multistep, insight, engineering, edge_cases)