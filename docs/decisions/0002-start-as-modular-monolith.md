# 0002. Start as a modular monolith

- Status: Accepted
- Date: 2026-09-29

## Context

The MVP scope is small, and the domain (projects, requirements, Gherkin, execution,
results, metrics) is tightly linked by traceability. Independent services would add
deployment and consistency costs without a current benefit.

## Decision

Build one backend application. Each domain module is a package under
`backend/src/traceforge/<module>/` with its own `models.py`, `schemas.py`, `service.py`,
and `router.py`. Routers call services; services own transactions and business rules.
Modules call each other through service functions, not by reaching into other
modules' routers.

## Consequences

- One deployable, one database, and simple local development.
- Modules could later be extracted into services if a real need appears.