# Data Model: Document Upload and Management

## Entities

### Document
Represents a stored work file, including file metadata, lifecycle state, and access controls.

| Field | Type | Notes |
|---|---|---|
| DocumentId | int | Primary key, consistent with the existing integer-based id pattern |
| Title | string | Required; searchable |
| Description | string | Optional; searchable |
| Category | string | Required, values from the allowed categories |
| ProjectId | int? | Optional association to a project |
| UploaderUserId | int | User who uploaded the document |
| FileName | string | Original file name, never used in the storage path |
| StoredFileName | string | Generated unique name used for safe storage |
| StoragePath | string | Internal path or blob name |
| MimeType | string | Up to 255 chars for Office document compatibility |
| FileSizeBytes | long | Stored for validation and display |
| UploadDateUtc | DateTime | Audit and sort metadata |
| LastModifiedUtc | DateTime | Update tracking |
| IsSoftDeleted | bool | Lifecycle flag |
| DeletedAtUtc | DateTime? | Used for retention logic |
| IsPurged | bool | Prevents reactivation after permanent purge |
| Tags | string | Optional newline/comma-delimited tags |
| ContentHash | string | Optional hash for duplicate or integrity validation |
| Status | string | PendingScan, Clean, Quarantined, ScanFailed, SoftDeleted, Purged |
| ScanQueuedAtUtc | DateTime? | Time the scan message was published |
| ScanStartedAtUtc | DateTime? | Time background processing began |
| ScanCompletedAtUtc | DateTime? | Time a terminal scan result was recorded |
| ScanAttempts | int | Number of scan attempts |
| ScanCorrelationId | string | Correlates upload, queue, worker, and audit logs |
| ScanFailureReason | string | Sanitized operational reason for quarantine or failure |

### DocumentShare
Represents a document sharing relationship between a document and a user or team.

| Field | Type | Notes |
|---|---|---|
| DocumentShareId | int | Primary key |
| DocumentId | int | FK to Document |
| UserId | int? | Recipient user, when sharing to a user |
| TeamId | int? | Optional team-based share if the app later adds teams |
| SharedByUserId | int | User who initiated the share |
| SharedAtUtc | DateTime | Audit timestamp |
| CanView | bool | Defaults to true |
| CanDownload | bool | Defaults to true |
| IsActive | bool | Disabled when share is removed |

### DocumentAuditLog
Tracks meaningful document lifecycle events for operational and compliance review.

| Field | Type | Notes |
|---|---|---|
| AuditLogId | int | Primary key |
| DocumentId | int | FK to Document |
| UserId | int | User triggering the action |
| ActionType | string | Upload, download, replace, delete, restore, share |
| Description | string | Human-readable summary |
| ActionTimeUtc | DateTime | Event time |

## Relationships

- One user uploads many documents.
- Each project may have many documents.
- Each document may have zero or many share records.
- Each document may have many audit log entries.
- A document is not retrievable until it has a `Clean` scan status.
- A soft-deleted document is still retrievable for 90 days; after that it is purged and no longer queryable for normal access.

## Indexes and Query Behavior

- Index on `ProjectId` and `UploaderUserId` for project-document and personal-document queries.
- Index on `Category`, `UploadDateUtc`, and `Title` for filtering and sorting.
- Index on `IsSoftDeleted` and `DeletedAtUtc` for purge and retention processing.
- Index on `MimeType` and `FileSizeBytes` to support validation and reporting.
- Full-text or column index is recommended for search terms on title, description, uploader, and tags when the dataset grows.

## Lifecycle Rules

- Newly uploaded documents are created in `PendingScan` state and remain unavailable to users.
- A successful background scan changes the document to `Clean`; malware detection changes it to `Quarantined`; exhausted scanner failures change it to `ScanFailed`.
- Only `Clean` documents may appear in search, preview, download, sharing, project views, task attachments, or dashboard widgets.
- Delete requests flip the document to `SoftDeleted` and store the deletion timestamp.
- Restore requests are allowed only when the document remains within the 90-day window and the user has adequate permissions.
- After 90 days, a scheduled cleanup removes the document metadata and underlying file from storage.

## Background Scan Message

The web application publishes a message to Azure Queue Storage after the private file and `PendingScan` metadata have been persisted:

| Field | Type | Notes |
|---|---|---|
| DocumentId | int | Identifies the pending document |
| StoragePath | string | Opaque private storage reference |
| ContentHash | string | Optional integrity/deduplication value |
| CorrelationId | string | Shared across logs and audit records |
| Attempt | int | Worker attempt number |

An Azure Function using a Queue Storage trigger consumes this message, scans the file, and updates the document status. The function must be idempotent and must route exhausted retries to a poison queue without making the file available.
