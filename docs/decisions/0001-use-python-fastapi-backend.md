# 0001. Use Python and FastAPI for the backend

- Status: Accepted
- Date: 2026-09-29

## Context

TraceForge needs a typed REST API, background test execution, live progress streaming,
Gherkin parsing, adapters for several test frameworks, and (later) AI-assisted generation
and evaluation. The candidates were Java/Spring Boot and Python/FastAPI.

## Decision

Use Python 3.13 with FastAPI, Pydantic v2, SQLAlchemy 2.x, and Alembic. Use `uv` for
dependency management. Enforce types with `mypy --strict`; lint and format with `ruff`.

## Consequences

- The official `gherkin` parser, Pytest, Robot Framework, and Playwright's Python API
  are all directly available.
- LLM SDKs and evaluation tooling are first-class in Python, which suits the AI roadmap.
- Static typing is enforced by tooling, not by the compiler, so `mypy --strict`
  runs as part of the definition of done.
- Module boundaries depend on convention (one package per module) rather than being
  enforced by a framework such as Spring Modulith.