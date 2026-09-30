# 0013. Evidence files: local storage, strict uploads, safe downloads

- Status: Accepted
- Date: 2026-09-29

## Context

Results need their evidence (screenshots, videos, traces, logs, reports) next to them,
reachable from runs and requirement traces. Evidence arrives from people and CI, so
every file is untrusted, and some types (HTML, SVG) can run script if a browser renders
them on TraceForge's origin.

## Decision

- **Storage:**
  - Metadata lives in `test_artifacts`: run, optional result, type, name, content type, size,
    SHA-256, and the uploading user or token.
  - Bytes are stored on local disk under `TRACEFORGE_ARTIFACT_ROOT`, behind an `ArtifactStorage`
    interface, so S3-compatible storage can be added without touching callers.
  - Storage keys are `{project}/{run}/{artifact}` (IDs only). File names are never used as paths.
  - Writes are atomic (temp file + rename), and a failed insert removes the file.
- **Retention:** files live as long as their run. Deleting an artifact deletes its file.
  Deleting a project deletes its whole directory. There is no time-based expiry.
- **Uploads** (`POST /api/test-runs/{runId}/artifacts`, editor role, tokens allowed):
  - The body is the raw file, streamed to disk with a size cap (`TRACEFORGE_MAX_ARTIFACT_BYTES`,
    default 100 MB).
  - The content type must be on an allowlist. SVG, executables and anything unknown are refused.
  - Images and videos must start with the right file signature, so HTML labelled `image/png`
    is refused.
  - Empty files are refused.
- **Downloads** (`GET …/artifacts/{id}/content`, viewer role):
  - Only images, videos and plain text are served inline. Everything else is sent with
    `Content-Disposition: attachment`.
  - Every response carries `X-Content-Type-Options: nosniff` and
    `Content-Security-Policy: sandbox; default-src 'none'…`, so even an opened HTML report
    cannot run script or load anything.
- **CI bundles** (`POST /api/projects/{projectId}/test-run-imports` with `application/zip`):
  - The bundle holds the JUnit report plus the files its test cases reference with
    `[[ATTACHMENT|path]]` (as Playwright's junit reporter writes). Those files become evidence
    on the matching results.
  - The zip is hostile input:
    - Entries are read only when referenced.
    - References are resolved relative to the report, and any that would leave the bundle
      (`..`, absolute paths, backslashes, drive letters) are ignored.
    - Limits: at most 10,000 entries, 2 GB expanded, and a compression-ratio check against
      zip bombs. The real decompressed size is counted while reading, because headers can lie.
    - The bundle itself is capped by `TRACEFORGE_MAX_IMPORT_BUNDLE_BYTES` (default 200 MB).
  - Attachments that are missing or refused become `warnings` in the response. They never
    silently disappear, and the rest of the import still succeeds.

## Consequences

- Evidence lives on one server's disk, so it must be backed up with it, and running several
  backend instances needs shared or object storage. That is the reason for the storage interface.
- Upload bodies are written with blocking I/O in chunks. This is fine for one process, and
  worth revisiting if uploads become heavy.
- TraceForge-executed runs do not collect files yet. Tests can upload evidence through the API.