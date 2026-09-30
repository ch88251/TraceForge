# 0011. Project-scoped, expiring API tokens for automation

- Status: Accepted
- Date: 2026-09-29

## Context

CI needs to start test runs and read results without a person's session (ADR 0010).
Options considered: personal access tokens that act as their creator everywhere,
project-bound tokens, or both. Expiry could be required or optional.

## Decision

- **Project tokens only.** An owner creates a token for one project with the role `viewer`
  or `editor` (never `owner`). A token keeps working if its creator leaves, and is deleted
  with its project.
- **Expiry is required:** default 90 days, at most 365.
- **Format and storage:**
  - Tokens look like `tfp_` + 256 random bits (URL-safe base64). The recognizable prefix lets
    secret scanners flag leaked tokens.
  - Only the SHA-256 hash is stored, plus the first 12 characters for identification.
  - The secret is returned once, at creation.
- **Use:** `Authorization: Bearer <token>`. An explicit Authorization header is
  authoritative: an invalid token fails with 401 even if a valid session cookie is also sent.
- **One principal model.** Requests are made by a *principal*: a signed-in user or a token.
  Project-scoped routes authorize either through the same central check (ADR 0010):
  - A token has its role in its own project, and gets 404 for any other project.
  - Routes that need a person depend on the session-only user dependency, which rejects tokens
    with 403 `TOKEN_NOT_ALLOWED`. These are: accounts, project list and creation, members,
    owner actions, and token management.
- **Accountability:**
  - Actions performed with a token are audited with `actor_token_id`.
  - Test runs record the token's name and prefix in `triggered_by`.
  - Token creation and revocation are audited.
  - `last_used_at` is updated at most once a minute.
  - Rejected tokens are logged with their prefix only.
- **Revocation** is immediate and keeps the record (status `revoked`) so audit history
  still names the token.

## Consequences

- CI can only trigger TraceForge-executed runs and read data. Importing results of tests
  that CI runs itself is a separate, future capability.
- Personal access tokens can be added later if people need scripts that span projects.
  The principal model already supports another kind of principal.
- No rate limiting applies to token requests yet.