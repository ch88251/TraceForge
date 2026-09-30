# 0007. Automated tests are explicit records with a draft/implemented status

- Status: Accepted
- Date: 2026-09-29

## Context

Automation coverage ("which scenarios are automated?") must be based on explicit
records, not inferred from test names or from the fact that a scaffold was generated.
Future test runners also need to map framework results back to TraceForge scenarios.

## Decision

- An `AutomatedTest` row records that executable automation for one scenario exists
  at a location: `framework`, `file_path`, optional `test_name`, `language`, and
  `repository`. A scenario may have many automated tests (e.g. an API test and a UI test).
- The same scenario + framework + file + test name cannot be registered twice
  (a unique index with `NULLS NOT DISTINCT`, so a missing test name counts as a value).
- `status` is `draft` or `implemented`, in place of the `automated` boolean sketched in
  `CLAUDE.md`. It means the same thing, reads more clearly, and leaves room for more states
  (such as `retired` once test results reference automated tests).
- **A scenario counts as automated only when it has at least one `implemented` test.**
  Registering a scaffold as `draft` does not raise automation coverage.
- `framework` is a fixed list enforced by a `CHECK` constraint:
  `pytest`, `playwright`, `cypress`, `junit5`, `cucumber_jvm`, `robot_framework`, `other`.
  Adding one requires a migration, the same as the requirement enums.
- The scenario-removal guard from [ADR 0005](0005-feature-source-is-authoritative.md) now
  also covers automated tests. The error code changed from `SCENARIO_HAS_REQUIREMENT_LINKS`
  to `SCENARIO_IN_USE`, and the details list requirement-link and automated-test counts
  per scenario. Nothing outside this repository consumed the old code.

## Consequences

- Whether a test is "implemented" is a human claim until it is executed. Once test runs exist,
  execution evidence (e.g. "implemented but never run" or "still skipped") should be shown
  next to this status rather than replacing it.
- Deleting a feature still cascades to its automated tests. When test results reference
  automated tests, deletion should become retirement so that history is preserved.