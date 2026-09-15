# Implementation Plan: Document Upload and Management

**Branch**: `001-document-upload-management` | **Date**: 2026-09-15 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-document-upload-management/spec.md`

## Summary

Implement a secure document upload and management workflow for the ContosoDashboard web app that lets employees upload, browse, search, share, and manage documents with role-aware authorization. The design follows the existing ASP.NET Core + Blazor Server architecture, keeps file storage outside `wwwroot`, stores metadata in the EF Core data model, and preserves future compatibility with Azure Blob Storage via an `IFileStorageService` abstraction.

The feature will add document metadata and sharing models, a storage abstraction with a local filesystem implementation, authorization checks for project and role boundaries, a document service layer, an asynchronous virus-scanning pipeline, and UI pages or components for upload, list, search, and project-specific views.

## Technical Context

**Language/Version**: C# / .NET 8.0
**Primary Dependencies**: ASP.NET Core 8, Blazor Server, Entity Framework Core, SQL Server LocalDB, Bootstrap 5; Azure Functions and Azure Queue Storage for hosted asynchronous scanning
**Storage**: SQL Server LocalDB for metadata; local filesystem storage outside `wwwroot` for uploaded files; Azure Blob Storage-compatible storage abstraction for hosted deployments; Azure Queue Storage for scan work items
**Testing**: xUnit for unit/integration tests; EF Core in-memory or SQLite for repository/service validation; browser-based smoke tests for upload/search flows
**Target Platform**: Windows desktop development / local web application
**Project Type**: Single web application
**Performance Goals**: Upload under 30 seconds for 25 MB files; document lists under 2 seconds for 500 rows; document search under 2 seconds
**Constraints**: Must run offline for training; secure-by-default authorization; local filesystem and an in-process/local worker adapter for offline development; hosted deployments use Azure Functions with Queue Storage triggers for asynchronous virus scanning; unscanned files cannot be viewed or downloaded; document retention for 90 days before permanent purge
**Scale/Scope**: Small training app with a few dozen users, project membership and role rules, and document metadata queries across a single relational database

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- Pass: Secure-by-default design is required for file validation, generated storage paths, authorization at the server boundary, and quarantine behavior on failed malware scanning.
- Pass: The feature is structured around service-level authorization and an infrastructure abstraction instead of UI-level permission checks.
- Pass: The implementation will include automated tests before final acceptance, especially for authorization, validation, storage failures, scan outcomes, queue retries, and soft-delete retention behavior.
- Pass: The solution stays simple and observable by using a single document service, a single storage abstraction, and explicit lifecycle states (active, soft-deleted, purged).
- Pass: The design respects the role model already present in `UserRole` and project membership patterns.

## Project Structure

### Documentation (this feature)

```text
specs/001-document-upload-management/
├── spec.md
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── document-service-contract.md
└── checklists/
    └── requirements.md
```

### Source Code (repository root)

```text
ContosoDashboard/
├── Data/
│   ├── ApplicationDbContext.cs
│   └── ...
├── Models/
│   ├── User.cs
│   ├── Project.cs
│   ├── TaskItem.cs
│   ├── Document.cs
│   ├── DocumentShare.cs
│   └── ...
├── Services/
│   ├── IUserService.cs
│   ├── ProjectService.cs
│   ├── TaskService.cs
│   ├── NotificationService.cs
│   ├── DocumentService.cs
│   ├── DocumentScanQueue.cs
│   ├── DocumentScanStatusService.cs
│   ├── Storage/
│   │   ├── IFileStorageService.cs
│   │   ├── LocalFileStorageService.cs
│   │   └── AzureBlobStorageService.cs (future)
│   ├── Scanning/
│   │   ├── IFileScanDispatcher.cs
│   │   ├── LocalFileScanDispatcher.cs
│   │   └── AzureQueueFileScanDispatcher.cs
│   └── ...
├── Pages/
│   ├── Documents/
│   │   ├── Upload.razor
│   │   ├── Index.razor
│   │   ├── Search.razor
│   │   ├── ProjectDocuments.razor
│   │   └── SharedWithMe.razor
│   └── ...
├── Shared/
│   └── Components/
│       └── DocumentUploadDialog.razor
├── wwwroot/
│   └── uploads/ (reserved for generated public assets only; actual file storage remains outside the web root)
├── Program.cs
├── appsettings*.json
└── ContosoDashboard.csproj

DocumentScan.Functions/
├── DocumentScanFunction.cs       # Azure Queue Storage-triggered worker
├── ScanMessage.cs                # Queue message contract
├── host.json
└── DocumentScan.Functions.csproj
```

**Structure Decision**: Use the existing single-application ASP.NET Core + Blazor Server structure for the web experience and add a narrowly scoped `DocumentScan.Functions` worker for hosted asynchronous scanning. The web app communicates with the worker through `IFileScanDispatcher` and a Queue Storage message, so local training mode can substitute `LocalFileScanDispatcher` without loading Azure dependencies. This preserves the offline training path while defining a production-ready Azure Functions boundary.

## Complexity Tracking

No constitution violations require a special exception for this feature.

## Phase Plan

### Phase 0 — Research and design validation
- Confirm the exact authorization model for document access, sharing, and project membership.
- Define the hosted scan workflow using Azure Queue Storage and an Azure Functions queue trigger, including retry, poison-queue, quarantine, and status-update behavior.
- Define the offline adapter that uses a locally available antivirus scanner or rejects/quarantines when scanning cannot complete.
- Confirm file metadata and retention rules for the `Document` entity and soft-delete lifecycle.
- Decide whether project document pages and dashboard widgets are server-rendered or component-based Blazor views.

### Phase 1 — Domain and storage design
- Add `Document`, `DocumentShare`, and metadata/audit entities to `Models`.
- Extend `ApplicationDbContext` with `DbSet<Document>` entries and indexes for project, uploader, category, scan status, and soft-delete state.
- Add `IFileStorageService` and a local implementation that writes files to a dedicated path outside `wwwroot`.
- Add scan status fields and an `IFileScanDispatcher` abstraction so the web app can enqueue scan work without depending directly on Azure SDKs.
- Define service methods for upload, replace, download, share, delete, restore, and list/search by project and role.

### Phase 2 — Authorization and persistence
- Enforce permissions in the document service using current claims, project membership, and role checks.
- Ensure only authorized users can view, download, share, replace, or delete document records.
- Validate file type, MIME, 25 MB size limit, and generated GUID-based paths before persistence.
- Handle storage failures and database failures atomically to avoid orphaned files or inconsistent metadata.
- Persist each accepted upload as `PendingScan`, enqueue a scan message containing only the document id and storage reference, and keep it unavailable to users until a clean result is recorded.

### Phase 3 — Background virus scanning
- Implement an Azure Queue Storage message contract with document id, storage path/blob name, content hash, attempt metadata, and correlation id.
- Implement an isolated .NET Azure Function with a Queue Storage trigger that retrieves the pending file through the storage abstraction, runs the configured antivirus scanner, and writes `Clean`, `Quarantined`, or `ScanFailed` status back through an authenticated application endpoint/service.
- Keep the function independent from the Blazor UI and ensure it has least-privilege access to the queue, private file storage, and document-status endpoint.
- Configure bounded retries and poison-message handling. Repeated scanner or dependency failures must move the document to quarantine or an explicit failed state and must not publish it as available.
- Make processing idempotent: duplicate queue deliveries must not duplicate audit entries, notifications, or status transitions after a terminal result.
- Emit structured logs and audit events for enqueue, scan start, clean result, malware detection, scanner failure, retry, quarantine, and poison-queue handling.
- For offline training, select `LocalFileScanDispatcher` through configuration. It may invoke the locally available scanner synchronously or use a local background queue, but it must preserve the same status transitions and deny access until scanning completes.

### Phase 4 — User experience and workflow integration
- Add upload flow with validation, progress indication, and success/error states.
- Show a pending-scan state after upload and explain that the document becomes available only after successful scanning.
- Add list/search/share screens and a “Shared with Me” section.
- Integrate document attachment and display into task and project views.
- Add recent-document and document-count widgets to the dashboard.

### Phase 5 — Validation and hardening
- Test upload validation, authorization boundaries, file storage failure, soft-delete retention, and purge behavior.
- Test queue publication, Azure Function retry and poison-message handling, scanner outcomes, idempotent duplicate delivery, and access denial while a document is `PendingScan` or `ScanFailed`.
- Verify project and dashboard permission views.
- Validate performance under the 2-second search/list targets and 30-second upload target.
- Confirm no direct file serving from `wwwroot` and no user-controlled path usage.
