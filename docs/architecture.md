# Architecture

TraceForge is a modular monolith (see [ADR 0002](decisions/0002-start-as-modular-monolith.md)).

```text
frontend/  React + TypeScript + Vite, TanStack Query, React Router
   │  /api (proxied by Vite in development)
backend/   FastAPI app — src/traceforge/<module>/{models,schemas,service,router}.py
   │
PostgreSQL (docker/compose.yml), schema managed by Alembic migrations
```

## Backend layering

- **router**: HTTP only. Parses the request, calls the service, and shapes the response.
- **service**: business rules, validation that needs the database, and transaction boundaries.
- **models**: SQLAlchemy ORM mappings.
- **schemas**: Pydantic request and response models.
- `errors.py`: `ApiError` subclasses, mapped to the structured error body.

## Modules

| Module         | Owns                                                         |
|----------------|--------------------------------------------------------------|
| `projects`     | Projects                                                     |
| `requirements` | Requirements                                                 |
| `gherkin`      | Features, scenarios, Gherkin parsing and validation (`parsing.py` is pure and has no DB access) |
| `traceability` | Requirement ↔ scenario links and traceability queries        |

Dependencies point one way: `traceability` → `gherkin`, `requirements` → `projects`.
`gherkin` reads the link table directly for one check: blocking removal of linked
scenarios.

## Frontend organization

- `src/features/<feature>/`: types, TanStack Query hooks (`api.ts`), pages, and components.
- `src/shared/`: the API client and generic components.
- Server state lives in TanStack Query; local form state lives in components.

## Testing

- Backend: pytest integration tests against a real PostgreSQL test database.
  The schema is built by running the Alembic migrations, and each test runs in a
  transaction that is rolled back.
- Frontend: Vitest and Testing Library, with `fetch` stubbed at the network boundary.