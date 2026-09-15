# Research: Document Upload and Management

## Decision 1: File storage strategy

Use a dedicated storage layer with an interface-based abstraction and a local filesystem implementation for the training app.

**Why**
- The repository already emphasizes infrastructure abstraction and offline-first behavior.
- The stakeholder requirement explicitly calls for a future Azure Blob storage path without changing business logic.
- The application must avoid using user-controlled file names or writing directly into `wwwroot`.

**Decision**
- Add `IFileStorageService` with `UploadAsync`, `DeleteAsync`, `DownloadAsync`, `GetUrlAsync`, and a `LocalFileStorageService` implementation.
- Store files in a safe path such as `AppData/uploads/{userId}/{projectId or personal}/{guid}.{ext}` outside `wwwroot`.
- Preserve the upload sequence: generate unique path → save file → save metadata.

## Decision 2: Authorization model

Enforce role and project membership checks in the document service rather than in UI markup alone.

**Why**
- The app already uses claims-based role authorization and service-level checks in services such as `ProjectService` and `TaskService`.
- Document management is highly sensitive and must be protected before file download or metadata changes.

**Decision**
- Owners manage their own documents.
- Project managers manage all documents in their project.
- Administrators manage documents globally.
- Any document action is validated against the current user claims and project membership before persistence or file access.

## Decision 3: Asynchronous malware scanning policy

Use an asynchronous scan pipeline for hosted deployments: the web app persists the file as pending scan, publishes a message to Azure Queue Storage, and an Azure Function with a Queue Storage trigger performs the scan. Use a local scanner/background adapter for offline training.

**Why**
- Uploading and scanning are separate operations, so a slow scanner does not block the upload request or Blazor circuit.
- Azure Functions and Queue Storage provide retryable, independently scalable background processing for hosted deployments.
- The app is intentionally offline-first, so the same interface must support a local implementation without requiring Azure.
- The constitution requires a secure-by-default posture. A scanner failure cannot result in unsafe uploads.

**Decision**
- Validate file extension, MIME, and size before storage.
- Save accepted files to private storage with status `PendingScan`; they are not searchable, previewable, downloadable, or shareable at this point.
- Publish a queue message containing the document id and opaque storage reference, not file contents or sensitive metadata.
- An Azure Function queue trigger retrieves the file, invokes the configured scanner, and records `Clean`, `Quarantined`, or `ScanFailed`.
- Configure retries for transient failures and route exhausted messages to a poison queue. Quarantined or failed documents remain inaccessible until an authorized remediation workflow resolves them.
- In offline mode, use a local scanner when available; if scanning cannot complete, quarantine or reject the file and preserve the same access restrictions.

## Decision 4: Queue and worker reliability

Treat queue delivery as at-least-once and make scan processing idempotent.

**Why**
- Azure Queue Storage can redeliver a message after a timeout or transient failure.
- Repeated processing must not produce conflicting status changes, duplicate notifications, or duplicate audit records.

**Decision**
- Include a correlation id and attempt count in each message.
- The worker checks the current document status before processing and records a single terminal outcome.
- Use structured logs for queue, scanner, and status transitions.
- Poison-queue items require explicit operator review; they must never make pending files available.

## Decision 5: Document lifecycle

Use a soft-delete retention period with eventual permanent purge.

**Why**
- Deletion is a material action, and users need recovery support within a controlled retention period.
- The specification calls for auditability and predictable storage cleanup.

**Decision**
- Soft delete for 90 days with eligible restore access.
- Permanent purge after 90 days for both the metadata record and the underlying file.
- A reset or restore action is only available to authorized document managers during the soft-delete window.

## Decision 6: UI integration

Use the existing project and task pages as the integration surface for document actions.

**Why**
- The app’s current architecture centers around project pages, task views, and dashboard summaries.
- The feature can be added without a new app shell or frontend framework.

**Decision**
- Add upload and listing pages under `Pages/Documents`.
- Add document attachment controls in task and project views.
- Add recent documents and counts to the dashboard summary area.
- Add a shared-with-me section for recipients of shared documents.
