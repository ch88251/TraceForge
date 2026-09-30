# 0010. Local accounts, server-side sessions, and project roles

- Status: Accepted
- Date: 2026-09-29

## Context

TraceForge can execute code on its host (ADR 0008) and holds project data that not every
user should see or change. It needs authentication and project-level authorization
before anyone else uses it. Options considered: local accounts, delegating to an OIDC
identity provider, or both. Permission models considered: Owner/Editor/Viewer, the same
plus a separate "runner" grant, or Owner/Viewer only.

## Decision

**Authentication: local accounts with server-side sessions.**
- Passwords are hashed with Argon2id (argon2-cffi defaults). Minimum length is 12; maximum is 128.
- Sessions live in Postgres. The browser holds a random 256-bit token in a `traceforge_session`
  cookie (`HttpOnly`, `SameSite=Lax`, `Secure` when `TRACEFORGE_SESSION_COOKIE_SECURE=true`).
  Only a SHA-256 hash of the token is stored, so the table cannot be replayed.
  Sessions last `TRACEFORGE_SESSION_TTL_HOURS` (default 12).
- Cookies (not bearer tokens) keep tokens out of JavaScript and work with `EventSource`,
  which cannot send an Authorization header.
- Wrong password and unknown email get the same response. Unknown emails still cost one
  Argon2 verification, so timing does not reveal which accounts exist.
- Five failed sign-ins for an email within 15 minutes lock that email for the rest of the window
  (in memory, per process). This trades a possible short lockout of a targeted account for
  protection against password guessing.
- Changing a password signs out the user's other sessions. Deactivation or an admin password
  reset signs out all of them.
- Unsafe requests (POST, PUT, PATCH, DELETE) whose `Origin` header is neither TraceForge's own
  origin nor in `TRACEFORGE_CORS_ORIGINS` are rejected (defense in depth next to `SameSite`).

**Accounts: admin-created.** There is no public sign-up. The first administrator is created
with `python -m traceforge.cli create-admin` (password from the terminal or a named
environment variable, never argv). Administrators create users with a generated temporary
password, shown once, which must be changed at first sign-in.

**Authorization: Owner / Editor / Viewer per project, plus a system admin flag.**

| Capability | Viewer | Editor | Owner | Admin |
|---|:-:|:-:|:-:|:-:|
| See the project, requirements, Gherkin, runs, metrics, traces | ✓ | ✓ | ✓ | ✓ (all projects) |
| Generate (not register) test scaffolds | ✓ | ✓ | ✓ | ✓ |
| Change requirements, Gherkin, links, automated tests | | ✓ | ✓ | ✓ |
| Start and cancel test runs (executes code) | | ✓ | ✓ | ✓ |
| Manage members, rename or delete the project | | | ✓ | ✓ |
| Manage user accounts | | | | ✓ |

- Enforcement is **deny by default** and central. Routes under `/api/projects/{project_id}`
  get a router-level dependency that requires viewer for safe methods and editor otherwise.
  Owner-only routes add an explicit owner dependency. Test-run routes resolve the run's project first.
  A route-inventory test fails if any API route is unauthenticated or a project/run route
  lacks the check.
- Non-members get **404** (the same as a missing project), so project and run IDs cannot be
  probed. Members without enough rights get **403**.
- Creating a project makes you its owner. A project always keeps at least one owner.
- Starting test runs is an editor capability rather than a separate grant. Runs are further
  confined by workspace roots (ADR 0008).
- `triggered_by` on test runs is taken from the signed-in user, never from the request.

**Audit.** Security-relevant actions go to `audit_events`:
- sign-in success and failure, blocked sign-ins, sign-out, password changes
- user creation, updates and password resets
- project creation and deletion, and membership changes
- test run start and cancellation

Secrets are never recorded. Failed sign-ins record the attempted email.

## Consequences

- Existing projects have no members after the migration. Administrators can see them all
  and add members.
- Rate limiting and the session store assume one backend process, like execution (ADR 0008).
- Project names are still unique across the whole installation, so creating a project can
  reveal that a name is taken by a project you cannot see.
- OIDC/SSO can be added later as another way to create a session. Sessions and roles do not
  depend on how the user authenticated.
- There is no API-token mechanism for CI yet. Automation must use a session.