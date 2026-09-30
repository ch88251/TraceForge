# 0003. Use Server-Sent Events for live execution updates

- Status: Accepted
- Date: 2026-09-29

## Context

Users need to watch test runs progress in real time. Updates flow from server to browser;
commands such as cancelling a run are rare and fit ordinary REST calls.

## Decision

Stream run events (`TEST_RUN_STARTED`, `TEST_PASSED`, …) to the browser over
Server-Sent Events. Commands use REST, e.g. `POST /api/test-runs/{runId}/cancel`.
During the MVP, events are published through a simple in-process mechanism.

## Consequences

- SSE is plain HTTP, reconnects automatically in the browser (`EventSource`), and
  passes through proxies easily.
- There is no client-to-server channel on the stream. If bidirectional interaction is
  needed later, revisit WebSockets.
- In-process publishing assumes a single backend instance; a broker would be required
  to scale out.