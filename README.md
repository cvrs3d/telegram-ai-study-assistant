# Telegram AI Study Assistant

A single-user Telegram Mini App for studying with Claude Opus. The MVP is designed for persistent chats, file analysis and generation, multi-stage tasks with revisions through chat, web search with citations, and interactive English quizzes.

## Documentation

- [Technical specification v0.2](docs/telegram_ai_study_assistant_spec_v0_2.md)
- [MVP development plan](docs/telegram_ai_study_assistant_development_plan.md)

## Local setup

The commands below apply once the initial Python 3.12 environment and lockfile are committed. Install `uv`, then run these commands from the repository root:

```bash
uv sync --locked
uv run --locked python -c "import anthropic"
```

The import check runs without an API key. Before making a request to Claude, copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY` locally. Keep `.env`, personal inputs, and generated test outputs out of version control.

The first implementation milestone is a local PDF and XLSX prototype. The Telegram Mini App launch is planned for stage 1.
