# 0004. Implement Pytest as the first test runner

- Status: Accepted (implemented as described in [ADR 0008](0008-in-process-test-execution.md))
- Date: 2026-09-29

## Context

Execution must be framework-neutral behind a `TestRunner` contract
(`validateConfiguration`, `discoverTests`, `execute`, `cancel`, `parseResults`).
One concrete runner is needed first to validate that contract.

## Decision

Build the Pytest runner first. It runs as a subprocess and reports results through
JUnit XML and/or a small pytest plugin that emits per-test events.

## Consequences

- The runner can be tested entirely within the backend's own toolchain.
- The contract must not leak Pytest-specific concepts, so that Playwright, Robot Framework,
  and JUnit runners can follow without changes to the core.