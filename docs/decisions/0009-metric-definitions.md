# 0009. Metrics are calculated from explicit records with fixed, documented rules

- Status: Accepted
- Date: 2026-09-29

## Context

The dashboard must answer "what is covered, automated, passing, and failing" without
ambiguous metrics (`CLAUDE.md`: never an unqualified "coverage"). Verification state
depends on links, automation, and results that all change independently.

## Decision

- **Derived, never stored.** Verification states and ratios are calculated on each
  request. The aggregation happens in SQL (grouping, and a window function for each
  scenario's latest run), so historical results are not loaded into Python.
- **One module owns the rules** (`metrics/verification.py`): scenario outcome within a
  run, requirement state precedence, pass rate. It is pure and unit-tested. Queries
  only gather inputs.
- **Evidence is per scenario, from its latest run that contains it.** A partial rerun
  does not erase evidence for scenarios it did not include. A newer run that only
  *skipped* a scenario does replace older evidence with "not executed": skipping is
  not a pass.
- **Failures dominate.** A scenario fails if any of its tests fail in that run. A
  requirement is `failing` if any linked scenario fails, whatever its other scenarios show.
- **Evidence and automation are separate.** A passing result counts even if the test was
  never registered as an automated test. Automation coverage counts only `implemented`
  automated tests. `defined` and `not_executed` differ only in whether automation exists.
- **Deprecated requirements are out of scope** for every requirement metric.
- **No false zeros.** Ratios with a zero denominator are `null`, not 0%.
- CLAUDE.md's `verified` state (verified for a release) is not implemented. `passing` is
  the strongest state until releases are modelled.

## Consequences

- Metric changes are code changes to one module, covered by table-driven tests.
- Each dashboard load runs a handful of aggregate queries. If large projects make this
  slow, cache per project and invalidate on runs and link changes, rather than storing
  derived state.
- Trends other than pass rate need snapshots, which are not recorded yet.