# Contributing to Dash

Thanks for your interest in contributing! This guide covers the basics.

## Development Setup

```sh
git clone https://github.com/agno-agi/dash.git && cd dash
cp example.env .env
# Edit .env and add your OPENAI_API_KEY

./scripts/venv_setup.sh && source .venv/bin/activate

docker compose up -d --build
python scripts/generate_data.py
python scripts/load_knowledge.py
```

## Workflow

1. Fork the repo and create a branch from `main`.
2. Make your changes.
3. Run the checks:
   ```sh
   ./scripts/format.sh     # ruff format + import sorting
   ./scripts/validate.sh   # ruff lint + mypy type checking
   ```
4. Run evals if your change touches agent behavior:
   ```sh
   python -m evals --verbose
   ```
5. Open a pull request against `main`.

## Code Style

- **Formatter/linter**: ruff (line length 120)
- **Type checking**: mypy with `check_untyped_defs` and `no_implicit_optional`
- Keep imports sorted (handled by `./scripts/format.sh`).
- CI runs format check, lint, and type check on every push and PR.

## Architecture Notes

Before diving in, review the [CLAUDE.md](CLAUDE.md) file for a detailed overview of the project structure, dual-schema design, and key patterns.

A few things to keep in mind:

- The `public` schema is **read-only** — never modified by agents.
- The `dash` schema is owned by the Engineer agent.
- Tool functions use a closure/factory pattern (`create_*_tool()` in `dash/tools/`).
- Instructions are composed dynamically in `dash/instructions.py`.

## Adding an Eval Category

1. Create a case file in `evals/cases/`.
2. Register it in `evals/__init__.py`.
3. Run `python -m evals --category <name>` to verify.

## Pull Request Guidelines

- Keep PRs focused — one concern per PR.
- Include a short description of **what** changed and **why**.
- If your change affects agent behavior, include sample queries and expected output.
- Make sure CI is green before requesting review.

## Reporting Issues

Use [GitHub Issues](https://github.com/agno-agi/dash/issues). Include:

- Steps to reproduce
- Expected vs. actual behavior
- Environment details (OS, Python version, Docker version)

## License

By contributing, you agree that your contributions will be licensed under the [Apache License 2.0](LICENSE).
