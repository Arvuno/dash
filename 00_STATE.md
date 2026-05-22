# 00_STATE.md — Dash OSS PR Campaign

## Repo Identity

| Field | Value |
|-------|-------|
| **Upstream** | `agno-agi/dash` (GitHub) |
| **Fork** | `okwn/dash` (GitHub, freshly forked) |
| **Local** | `/root/oss-pr-campaign/repos/dash` |
| **License** | Apache-2.0 |
| **Archived** | No |
| **Language** | Python 3.12+ |
| **Stars** | 2,068 |
| **Forks** | 233 (upstream) |

## What Dash Is

A **self-learning data agent** that delivers insights from a B2B SaaS dataset. Built on Agno's agent framework with a team of specialists (Analyst + Engineer) coordinated by a leader. Queries PostgreSQL via SQLTools, uses a dual-schema architecture (`public` = company data read-only, `dash` = agent-managed views/tables), and improves via a learning machine that records error patterns.

Key files:
- `dash/team.py` — Team leader (coordinate mode)
- `dash/agents/analyst.py` — SQL query agent (read-only)
- `dash/agents/engineer.py` — Schema/view builder (writes to dash schema)
- `dash/instructions.py` — System prompts for all 3 agents
- `db/session.py` — Dual-engine setup with schema guards
- `knowledge/` — Table metadata, validated queries, business rules

## CI/CD

GitHub Actions workflow at `.github/workflows/validate.yml`:
- Triggers on push to `main` and on PRs to `main`
- Python 3.12, uses `uv` for package management
- Steps: ruff format check, ruff lint, mypy type check
- No test execution in CI (tests require DB + API keys)

## Open Issues (9 total, all open)

| # | Title |
|---|-------|
| 15 | Does Dash address "Silent Wrong Answer"? |
| 12 | Decoupling runtime state, knowledge db and the data db |
| 11 | feat: add OpenAI-first provider selection with OpenRouter fallback |
| 10 | feat: add Slack integration with slack_dash agent |
| 8 | feat: add multi analytics database routing for SQL and introspection |
| 7 | [Feature]: User-configurable custom SQL schema |
| 6 | [Feature]: VectorChord extension as a replacement for PGVector |
| 5 | feat: ops warehouse, incident tools, monitoring bridge |
| 4 | Security Contact Request - Vulnerability Disclosure |

## Open PRs (4 total, all open)

| # | Title |
|---|-------|
| 11 | feat: add OpenAI-first provider selection with OpenRouter fallback |
| 10 | feat: add Slack integration with slack_dash agent |
| 8 | feat: add multi analytics database routing for SQL and introspection |
| 5 | feat: ops warehouse, incident tools, monitoring bridge |

Note: All 4 PRs are also tracked as issues (same numbers). These appear to be feature PRs filed against the repo.

## Git History

Fork and upstream are in sync — no local commits yet, no diff vs upstream/main.

Branches on upstream:
- `main` (default)
- `dash-v2`
- `docs/slack-setup`
- `feat/slack-integration`
- `kyleaton-patch-1`
- `update/agno-2.5.2`

## Key Tech Decisions

1. **Model**: `OpenAIResponses(id="gpt-5.4")` — hardcoded in `dash/settings.py`
2. **Package manager**: `uv` (used in all scripts and CI)
3. **Knowledge**: Vector search via `pgvector` + `OpenAIEmbedder(id="text-embedding-3-small")`
4. **Schema enforcement**: PostgreSQL `default_transaction_read_only` for Analyst; SQLAlchemy event listener regex guard for Engineer
5. **Evals**: 5 categories — accuracy, routing, security, governance, boundaries; plus smoke tests
6. **Self-improvement loop**: `docs/IMPROVE_DASH.md` — run smoke tests, fix instructions/knowledge, repeat up to 5 rounds