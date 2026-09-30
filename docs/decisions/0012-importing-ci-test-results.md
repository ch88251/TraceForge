# 0012. Importing CI test results as JUnit XML

- Status: Accepted
- Date: 2026-09-29

## Context

TraceForge can execute pytest suites on its own host (ADR 0008), but most teams already
run tests in CI, in many frameworks. Those results should count as evidence too, with
the same explicit links to scenarios.

## Decision

- **Format: JUnit XML.** Nearly every framework can write it (pytest `--junitxml`,
  Playwright's `junit` reporter, JUnit 5, Cypress, Robot Framework's xunit output).
- **Endpoint:** `POST /api/projects/{projectId}/test-run-imports`, body = the raw XML,
  query parameters `framework` (required), `release_version`, `environment`, `branch`,
  `git_commit`. It needs the editor role; project API tokens (ADR 0011) can use it.
- **An import is a test run.** It is stored as a finished `TestRun` with
  `source = imported`, no working directory, and status `failed` if any case failed or
  errored (otherwise `passed`). History, metrics, traces, and the dashboard treat it like
  any other run.
- **Explicit scenario links only.** A test case is linked to a scenario through a
  `traceforge_scenario` or `traceforge-scenario` property, and only if the scenario
  belongs to the project:
  - TraceForge's pytest plugin now adds `traceforge_scenario` for the scenario marker in
    any run that loads it. CI can download the plugin from
    `GET /api/integrations/pytest-plugin`.
  - Playwright writes `traceforge-scenario` from the annotation in generated scaffolds
    when its JUnit reporter has `embedAnnotationsAsProperties: true`.
- **Automated test links:** CI checkouts make file paths unreliable, so a linked result
  is attached to the scenario's only registered automated test for that framework, or to
  the single one whose name matches the reported name. It ignores pytest parameters and
  takes the last part of a Playwright title. When the match is ambiguous, nothing is linked.
- **Untrusted input:**
  - Reports are parsed with `defusedxml`, so entity expansion, external entities and DTD
    tricks are rejected.
  - The body is read as a stream with a 20 MB cap (413 above it).
  - Reports are limited to 50,000 test cases.
  - Messages and tracebacks are truncated.
  - Unusable reports get 422 `INVALID_TEST_REPORT`, and nothing is stored.
- **Only pytest can be executed** by TraceForge. Asking to execute another framework is
  refused with a message that points to imports.

## Consequences

- Evidence from CI and from TraceForge-executed runs is indistinguishable in metrics. The
  run's `source` says where it came from.
- JUnit XML carries no reliable per-case timestamps. Imported results use the import time
  as completion time, and the report's earliest suite timestamp (if any) as the run start.
- Attachments (screenshots, traces) in reports are ignored. Evidence artifacts are future work.