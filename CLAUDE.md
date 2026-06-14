# CLAUDE.md

Guidance for Claude Code working in this repo. Keep this file short; the
canonical, in-depth references are **AGENTS.md** (architecture + dev guide) and
**CONTRIBUTING.md** (contribution rules). Read those before non-trivial work.

## What this is

Hermes Agent — a self-improving AI agent (Python ≥3.11, managed with `uv`).
Core conversation loop lives in `run_agent.py`; the interactive CLI in `cli.py`.

## Setup

```bash
uv venv .venv --python 3.11
source .venv/bin/activate
uv pip install -e ".[all,dev]"
```

## Commands

```bash
scripts/run_tests.sh                  # tests — ALWAYS use this, never bare pytest
scripts/run_tests.sh tests/agent/...  # narrow to a dir/file/test
ruff check .                          # lint (blocking in CI; only PLW1514 enforced)
ty check                              # typecheck (advisory)
```

`run_tests.sh` enforces CI parity (unset API keys, UTC, C.UTF-8, subprocess
isolation per test). Calling `pytest` directly diverges from CI — don't.

## Orientation

- `run_agent.py` — `AIAgent`, the agent loop (large).
- `cli.py` / `hermes_cli/` — interactive CLI + subcommands.
- `model_tools.py`, `toolsets.py` — tool orchestration and toolset definitions.
- `tools/` — tool implementations, auto-discovered via `tools/registry.py`.
- `gateway/platforms/` — messaging adapters (Telegram, Discord, Slack, …).
- `plugins/`, `skills/` — extension points (see AGENTS.md for the contracts).

## Conventions

- Dependencies are **exact-pinned** in `pyproject.toml` (supply-chain policy).
  Change a pin, then regenerate the lockfile with `uv lock`. Don't add ranges.
- Provider/backend-specific deps go in an extra + lazy-install, not core deps.
- Run the full test suite before pushing.
