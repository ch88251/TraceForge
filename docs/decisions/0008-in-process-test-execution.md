# 0008. In-process test execution with a reporting plugin and workspace roots

- Status: Accepted
- Date: 2026-09-29

## Context

TraceForge must run automated tests, show progress live (ADR 0003), and store results
linked to scenarios and automated tests. Running tests means executing arbitrary code
from the user's repository, so where and how that happens is a security decision.

## Decision

**Execution model**
- Runs execute on a bounded `ThreadPoolExecutor` inside the backend process
  (`TRACEFORGE_MAX_CONCURRENT_RUNS`, default 2). Each run starts the framework in a
  subprocess. There is no job queue or broker service.
- Runs have a time limit (`TRACEFORGE_RUN_TIMEOUT_SECONDS`, default 1800). Cancellation
  and timeouts stop the whole process group (SIGTERM, then SIGKILL after 5 s).
- On startup, runs left `queued` or `running` by a previous process are marked `error`
  ("Interrupted: …"). An interrupted run is never resumed silently.

**Pytest integration**
- A standalone plugin (`traceforge_pytest_plugin.py`, stdlib + pytest only) is loaded
  with `-p` via `PYTHONPATH`. It writes JSON-lines events to a dedicated pipe
  (`TRACEFORGE_EVENTS_FD`), separate from pytest's terminal output, so test output can
  never be parsed as an event. The last 200 lines of terminal output are kept as the
  run's `output_tail` for diagnosis.
- The plugin registers the `traceforge_scenario` marker, so generated scaffolds need
  no pytest configuration when run by TraceForge.
- Results are linked explicitly, never by guessing from names:
  - `scenario_id` comes from the marker, and only if the scenario belongs to the run's project.
  - `automated_test_id` comes from a registered pytest automated test for that scenario
    with the same file path, and either the same function name or no test name.

**Security boundaries**
- Test runs may only use working directories inside `TRACEFORGE_WORKSPACE_ROOTS`
  (resolved, so symlinks and `..` cannot escape). **With no roots configured, execution
  is disabled.** A custom Python executable must also be inside a workspace root.
- Test paths must be relative and cannot start with `-`, so they cannot inject pytest options.
- The test process receives a minimal environment (PATH, HOME, LANG, LC_ALL, TMPDIR, TZ).
  TraceForge's own configuration, including the database URL, is never passed to test code.

**Status semantics**
- Result statuses: `passed`, `failed`, `error` (setup/teardown/collection problems),
  `skipped` (including expected failures), and `pending` (collected but not finished).
- Run statuses:
  - `passed`: pytest exit 0 and no failures or errors. A run where every test was skipped
    is `passed`. It verifies nothing, and requirement verification is based on
    per-test results.
  - `failed`: tests ran and at least one failed or errored.
  - `error`: the run could not execute properly (collection errors, usage errors,
    no tests collected, timeout, internal failure).
  - `cancelled`: a person cancelled the run.

**Live events**
- An in-process broker delivers events from worker threads to SSE subscribers.
  Events are published after the corresponding database commit.
- A subscriber subscribes before reading the snapshot, so nothing falls between the two.
  The browser treats events as idempotent upserts and never regresses a finished result,
  so duplicates are harmless.
- Event payloads use the same snake_case field names as the REST API, rather than the
  camelCase shown in the `CLAUDE.md` example.

## Consequences

- One backend process only. Running several instances would need a shared broker and
  a job queue, which is the point to revisit this decision.
- POSIX only (process groups, `pass_fds`).
- Tests that need extra environment variables (e.g. a base URL) cannot receive them yet.
  A per-run allowlist of variables would be the next step.
- `discoverTests` from the runner contract is not implemented yet. Collection happens as
  part of each run.