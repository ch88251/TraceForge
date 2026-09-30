# 0006. Test scaffolds are stateless drafts that can never pass

- Status: Accepted
- Date: 2026-09-29

## Context

TraceForge generates starting-point test code from Gherkin scenarios. A scaffold is not
automation: its steps are placeholders. If a scaffold could pass, or if generating one
marked a scenario as automated, TraceForge would report verification that never happened.

## Decision

- **Stateless.** `GET /api/projects/{projectId}/features/{featureId}/scaffold` returns
  generated code and saves nothing. Recording that a scenario is automated will be a
  separate, explicit step (the planned `AutomatedTest` entity).
- **Never passes.** Every generated test is skipped until someone implements it:
  `pytest.skip(...)` for Pytest and `test.fixme(...)` for Playwright.
- **Explicit traceability.** Each generated test carries its TraceForge scenario ID in a
  form the framework reports, so future runners can map results to scenarios without
  guessing from test names:
  - Pytest: `@pytest.mark.traceforge_scenario("<scenario-id>")`
  - Playwright: `annotation: { type: 'traceforge-scenario', description: '<scenario-id>' }`
- **Adapter per framework.** Generators implement a small `ScaffoldGenerator` protocol
  (`test_generation/scaffold.py`) and receive framework-neutral input. They have no
  database or HTTP access. Pytest and Playwright/TypeScript are the first two.
- **Verified by execution.** Tests run the generated Pytest code under pytest (every test
  must be skipped) and syntax-check the generated TypeScript with `node --check`.

## Registering a scaffold

Generating stays stateless. A separate, explicit action
(`POST …/scaffold/automated-tests`) records a scaffold's tests as `draft` automated tests
(see [ADR 0007](0007-automated-test-registry.md)):

- The server regenerates the scaffold and uses the generator's own test names
  (`GeneratedScaffold.tests`), so registered names always match the generated code.
  Clients do not send test names.
- Pytest: one automated test per generated function. An outline whose Examples tables
  have different headers produces several functions, and so several automated tests.
- Playwright: the test name is the scenario name. For outlines this is the outline name,
  because the actual titles are built per example row at runtime.
- Tests already registered at the same location are reported as `existing`, not errors,
  so registering again after adding scenarios only creates the new ones.
- Drafts do not count toward automation coverage.

## Gherkin → framework mapping

| Gherkin                  | Pytest                                  | Playwright                              |
|--------------------------|-----------------------------------------|-----------------------------------------|
| Feature                  | module (`test_<name>.py`), docstring    | `test.describe` (`<name>.spec.ts`)      |
| Background               | `@pytest.fixture(autouse=True)`         | `test.beforeEach`                       |
| Rule                     | noted in the test docstring             | nested `test.describe('Rule: …')`       |
| Scenario                 | `def test_<name>()`                     | `test('<name>', …)`                     |
| Scenario Outline         | `@pytest.mark.parametrize`              | `for` loop over example objects         |
| Examples, same header    | combined into one parametrize           | combined into one loop                  |
| Examples, different header | one function per header (`_examples_N`) | one loop per header, numbering continues |
| Tags                     | listed in docstrings                    | native `tag` option                     |
| Steps, tables, doc strings | comments                              | comments                                |

## Consequences

- Users must register the `traceforge_scenario` marker in their pytest configuration,
  or pytest warns about an unknown marker. The generated module docstring says how.
- Playwright titles for outline examples get an `(example N)` suffix so they stay unique.
- Running generated Playwright specs still requires the user's Playwright setup (browsers,
  config). TraceForge only checks syntax, not runtime behavior.