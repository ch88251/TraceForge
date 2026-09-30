# 0005. Feature source text is authoritative; scenarios are identified by name

- Status: Accepted (guard extended to automated tests by [ADR 0007](0007-automated-test-registry.md))
- Date: 2026-09-29

## Context

Gherkin is authored as whole `.feature` files. TraceForge also needs structured
scenarios with stable IDs, so requirements can be linked to them and, later, tests,
runs, and results can reference them. When a user edits a feature file, the
application must decide which existing scenario each parsed scenario corresponds to.

## Decision

- A feature's `source_text` is authoritative. Every save parses it with the official
  Cucumber Gherkin parser and re-syncs the `features` and `scenarios` rows. The
  original text, including comments and formatting, is stored unchanged.
- Scenarios are matched to existing rows **by name within the feature**. A matched
  scenario keeps its ID; an unmatched parsed scenario gets a new ID; an existing scenario
  with no match is deleted.
- Scenario names must therefore be non-empty and unique within a feature. A save that
  breaks this rule is rejected.
- A save that would delete a scenario linked to any requirement is rejected
  (409 `SCENARIO_HAS_REQUIREMENT_LINKS`). Renaming a linked scenario counts as deleting it.
  The user must unlink first, so traceability is never lost as a side effect.
- Deleting a whole feature is an explicit action and cascades to its scenarios and links.

## Alternatives considered

- **Edit scenarios individually** in the database and generate the feature text from them:
  this loses comments and formatting and fights how teams author Gherkin.
- **Embed a TraceForge ID tag in the source** (e.g. `@tf:1234`): this survives renames, but
  it pollutes users' files and breaks when tags are copied or removed. It could be added later
  as an optional, stronger identity hint.
- **Match by position or fuzzy text similarity**: this is unpredictable, and reordering
  scenarios would silently move links to the wrong scenario.

## Consequences

- Renaming a linked scenario takes three steps: unlink, rename, relink. A dedicated
  rename operation can be added if this becomes a pain point.
- Once test results reference scenarios, the same deletion guard should extend to
  scenarios with automated tests or execution history.