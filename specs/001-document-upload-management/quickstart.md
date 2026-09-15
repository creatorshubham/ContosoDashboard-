# Quick Start: Document Upload and Management

## Local setup

1. Open the repository root in a terminal.
2. Ensure the .NET 8 SDK is installed and the default LocalDB/SQL Server configuration is available.
3. Run the Web app from the `ContosoDashboard` folder:

```powershell
cd ContosoDashboard
dotnet run
```

4. Sign in as an existing mock user in the dashboard.

## Manual validation scenarios

### Upload flow

1. Navigate to the Documents area.
2. Choose a supported file under 25 MB.
3. Provide a title, category, optional description, and project association if applicable.
4. Confirm the upload and confirm the file is saved outside `wwwroot`.
5. Validate the document first appears as scan-pending and is not downloadable.
6. After the local scanner/background adapter reports a clean result, validate the document appears in the current user’s document list and the project view if applicable.

### Validation failures

1. Attempt to upload an unsupported extension.
2. Attempt to upload a file over 25 MB.
3. Attempt to upload a file while local malware scanning is unavailable or fails.
4. Confirm the upload is rejected or quarantined and no file is served to the user.

### Hosted background scan flow

1. Configure the hosted deployment with Azure Queue Storage and the Azure Functions queue-trigger worker.
2. Upload a valid file and verify the web app persists it as `PendingScan` and publishes a message containing the document id and opaque storage reference.
3. Verify the function consumes the message, scans the private file, and records a terminal status.
4. Verify duplicate delivery is idempotent, transient failures retry, and exhausted retries reach the poison queue without exposing the file.

### Share and access

1. Upload a document as an owner.
2. Share it with another user.
3. Validate the recipient sees the document in the Shared with Me view.
4. Confirm access is denied for users without permission.

### Lifecycle controls

1. Delete a document.
2. Validate the soft-delete state and 90-day retention behavior.
3. Restore the document before the retention window closes.
4. Confirm permanent purge after 90 days.

## Expected compliance

- Files are stored outside `wwwroot`.
- User-supplied names are not used as direct filesystem paths.
- Role-based checks run on document operations.
- Search results only include authorized documents.
- A 25 MB and extension allow-list are enforced.
