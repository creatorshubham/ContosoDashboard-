# Document Service Contract

## Purpose

Describe the service boundary for the document upload and management workflow so the UI and data access layers remain aligned with the business rules.

## Core operations

### UploadDocumentAsync
- Input: `DocumentUploadRequest`
- Returns: `DocumentUploadResult`
- Validates file size, extension, required metadata, and active project or user context.
- Requires a generated file path and a successful local storage write before metadata persistence.
- Persists the document as `PendingScan`, publishes a scan message through `IFileScanDispatcher`, and does not expose the file until a worker records `Clean`.
- If scanning detects malware or cannot complete after retries, the document is quarantined or marked `ScanFailed` and remains unavailable.

### ProcessScanMessageAsync
- Input: `DocumentScanMessage` containing document id, opaque storage reference, correlation id, and attempt number.
- Invoked by an Azure Function Queue Storage trigger in hosted deployments, or by the local background adapter for offline training.
- Retrieves the private file, runs the configured scanner, and records one terminal result: `Clean`, `Quarantined`, or `ScanFailed`.
- Must be idempotent for duplicate queue deliveries and must never expose a file with a non-clean scan status.

### ReplaceDocumentAsync
- Input: `DocumentReplaceRequest`
- Requires authorization to manage the document.
- Records a version or lifecycle replacement action and keeps the document association intact unless intentionally changed.

### GetAuthorizedDocumentsAsync
- Input: user context and optional filters
- Returns: authorized document set only.
- Applies project membership, role-based access checks, and `Clean` scan status before returning rows.

### ShareDocumentAsync
- Input: document id, recipient ids, share metadata
- Notifies recipients and stores the share relationship.
- Requires ownership, project manager scope, or administrator rights.

### DeleteDocumentAsync
- Input: document id and current user
- Moves the record to soft-deleted state and records an audit entry.
- Allows restore only within the 90-day window.

### RestoreDocumentAsync
- Input: document id and current user
- Requires eligible managed-document rights.
- Brings active document back into normal access after validation.

## Result model

- Success or failure status
- Scan status for asynchronous upload completion
- Error code for security, validation, or authorization failures
- Audit metadata for tracking and review
- No direct file URLs exposed outside authorized endpoints
