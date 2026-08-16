# Service Interface Contracts: Document Management

**Feature**: Document Upload and Management  
**Created**: 2026-08-16  
**Specification**: [spec.md](../spec.md)  
**Data Model**: [data-model.md](../data-model.md)

## Contract Overview

This document defines the public interfaces/contracts for the document management feature. These contracts are exposed through the service layer (not directly as HTTP APIs - this is Blazor Server with server-side business logic).

---

## IFileStorageService Interface

**Purpose**: Abstraction for file storage operations, enabling future cloud migration (Azure Blob Storage, AWS S3, etc.)

**Rationale**: Separates file storage implementation from business logic. Local filesystem implementation in MVP; future cloud implementations swap in via DI configuration.

```csharp
namespace ContosoDashboard.Services
{
    /// <summary>
    /// Abstraction for file storage operations.
    /// Enables local filesystem in training, cloud storage in production.
    /// </summary>
    public interface IFileStorageService
    {
        /// <summary>
        /// Upload a file stream to storage.
        /// </summary>
        /// <param name="fileStream">File content stream</param>
        /// <param name="filePath">Relative storage path (e.g., "123/456/abc123.pdf")</param>
        /// <param name="cancellationToken">Cancellation token for async operation</param>
        /// <returns>Absolute path or URL for retrieval</returns>
        Task<string> UploadAsync(Stream fileStream, string filePath, CancellationToken cancellationToken = default);

        /// <summary>
        /// Download a file stream from storage.
        /// </summary>
        /// <param name="filePath">Relative storage path</param>
        /// <param name="cancellationToken">Cancellation token</param>
        /// <returns>File stream</returns>
        Task<Stream> DownloadAsync(string filePath, CancellationToken cancellationToken = default);

        /// <summary>
        /// Get a URL for accessing file (for preview, download links).
        /// </summary>
        /// <param name="filePath">Relative storage path</param>
        /// <returns>URL that serves file through DocumentController endpoint</returns>
        Task<string> GetUrlAsync(string filePath);

        /// <summary>
        /// Delete a file from storage permanently.
        /// </summary>
        /// <param name="filePath">Relative storage path</param>
        /// <param name="cancellationToken">Cancellation token</param>
        Task DeleteAsync(string filePath, CancellationToken cancellationToken = default);

        /// <summary>
        /// Verify file exists in storage.
        /// </summary>
        /// <param name="filePath">Relative storage path</param>
        /// <returns>True if file exists</returns>
        Task<bool> ExistsAsync(string filePath);

        /// <summary>
        /// Get file size in bytes.
        /// </summary>
        /// <param name="filePath">Relative storage path</param>
        /// <returns>File size or 0 if not found</returns>
        Task<long> GetFileSizeAsync(string filePath);
    }
}
```

### LocalFileStorageService Implementation

```csharp
namespace ContosoDashboard.Services
{
    /// <summary>
    /// Local filesystem implementation of IFileStorageService.
    /// Stores files in AppData/uploads/ directory.
    /// Used for training/offline environments.
    /// </summary>
    public class LocalFileStorageService : IFileStorageService
    {
        private readonly string _baseStoragePath; // AppData/uploads/

        public LocalFileStorageService(IConfiguration configuration)
        {
            // Read storage path from configuration or default to AppData/uploads/
            _baseStoragePath = Path.Combine(
                Directory.GetCurrentDirectory(), 
                configuration["FileStorage:BasePath"] ?? "AppData/uploads"
            );

            // Ensure directory exists
            Directory.CreateDirectory(_baseStoragePath);
        }

        /// <summary>
        /// Generate unique file path before upload to prevent conflicts.
        /// Upload sequence: Generate path → Save file → Save to DB
        /// This prevents orphaned DB records if file save fails.
        /// </summary>
        public static string GenerateFilePath(int userId, int? projectId, string fileExtension)
        {
            var guid = Guid.NewGuid().ToString("N");
            var projectFolder = projectId?.ToString() ?? "personal";
            return Path.Combine(userId.ToString(), projectFolder, $"{guid}{fileExtension}");
        }

        public async Task<string> UploadAsync(Stream fileStream, string filePath, CancellationToken cancellationToken = default)
        {
            var fullPath = Path.Combine(_baseStoragePath, filePath);
            
            // Create directory if needed
            var directory = Path.GetDirectoryName(fullPath);
            Directory.CreateDirectory(directory!);

            // Save file
            using (var fileHandle = File.Create(fullPath))
            {
                await fileStream.CopyToAsync(fileHandle, cancellationToken);
            }

            return fullPath;
        }

        public async Task<Stream> DownloadAsync(string filePath, CancellationToken cancellationToken = default)
        {
            var fullPath = Path.Combine(_baseStoragePath, filePath);
            
            if (!File.Exists(fullPath))
                throw new FileNotFoundException($"File not found: {filePath}");

            return new FileStream(fullPath, FileMode.Open, FileAccess.Read, FileShare.Read);
        }

        public async Task<string> GetUrlAsync(string filePath)
        {
            // Return controller endpoint that serves file with authorization check
            // E.g., /api/documents/download/123 (DocumentId-based, service validates access)
            return $"/api/documents/download/{Uri.EscapeDataString(filePath)}";
        }

        public async Task DeleteAsync(string filePath, CancellationToken cancellationToken = default)
        {
            var fullPath = Path.Combine(_baseStoragePath, filePath);
            if (File.Exists(fullPath))
                File.Delete(fullPath);
        }

        public async Task<bool> ExistsAsync(string filePath)
        {
            var fullPath = Path.Combine(_baseStoragePath, filePath);
            return File.Exists(fullPath);
        }

        public async Task<long> GetFileSizeAsync(string filePath)
        {
            var fullPath = Path.Combine(_baseStoragePath, filePath);
            if (!File.Exists(fullPath)) return 0;

            var info = new FileInfo(fullPath);
            return info.Length;
        }
    }
}
```

---

## IDocumentService Interface

**Purpose**: Business logic for document CRUD, role-based access control, and validation

```csharp
namespace ContosoDashboard.Services
{
    /// <summary>
    /// Business logic for document management with role-based access control.
    /// All access validated at service layer (Defense in Depth).
    /// </summary>
    public interface IDocumentService
    {
        // Upload
        /// <summary>
        /// Upload new document with metadata.
        /// Performs: virus scan, file validation, role-based access check.
        /// </summary>
        Task<Document> UploadAsync(
            Stream fileStream, 
            DocumentUploadRequest request, 
            ClaimsPrincipal currentUser, 
            CancellationToken cancellationToken = default);

        // Download
        /// <summary>
        /// Get download stream for document.
        /// Validates user has access before returning stream.
        /// </summary>
        Task<(Stream stream, string fileName, string mimeType)> DownloadAsync(
            int documentId, 
            ClaimsPrincipal currentUser, 
            CancellationToken cancellationToken = default);

        // Browse
        /// <summary>
        /// Get user's documents with filtering and sorting.
        /// Returns only documents user can access.
        /// </summary>
        Task<List<DocumentSummary>> GetUserDocumentsAsync(
            int skip = 0, 
            int take = 50, 
            string? category = null, 
            int? projectId = null, 
            DateTime? dateFromUtc = null, 
            DateTime? dateToUtc = null, 
            string sortBy = "UploadDate", 
            bool ascending = false,
            ClaimsPrincipal currentUser);

        /// <summary>
        /// Get project documents for users with project access.
        /// </summary>
        Task<List<DocumentSummary>> GetProjectDocumentsAsync(
            int projectId, 
            int skip = 0, 
            int take = 50, 
            ClaimsPrincipal currentUser);

        // Search
        /// <summary>
        /// Full-text search across title, description, tags, uploader.
        /// Returns only accessible documents, results within 2 seconds.
        /// </summary>
        Task<List<DocumentSearchResult>> SearchAsync(
            string query, 
            int skip = 0, 
            int take = 50, 
            ClaimsPrincipal currentUser, 
            CancellationToken cancellationToken = default);

        // Metadata
        /// <summary>
        /// Update document metadata (title, description, category, tags).
        /// Only uploader or Project Manager can edit.
        /// </summary>
        Task<Document> UpdateMetadataAsync(
            int documentId, 
            DocumentMetadataUpdateRequest request, 
            ClaimsPrincipal currentUser);

        /// <summary>
        /// Replace document file with new version.
        /// Retains original upload date and uploader.
        /// </summary>
        Task<Document> ReplaceFileAsync(
            int documentId, 
            Stream newFileStream, 
            DocumentFileReplaceRequest request, 
            ClaimsPrincipal currentUser);

        // Delete
        /// <summary>
        /// Soft-delete document (sets Status to Deleted).
        /// Creates audit log entry.
        /// Only uploader or Project Manager can delete.
        /// </summary>
        Task DeleteAsync(int documentId, ClaimsPrincipal currentUser);

        /// <summary>
        /// Permanently remove document (hard delete).
        /// Removes from database and storage.
        /// Admin-only operation.
        /// </summary>
        Task PermanentlyDeleteAsync(int documentId, ClaimsPrincipal currentUser);

        // Access Control (internal service-to-service calls)
        /// <summary>
        /// Check if user has access to document for viewing.
        /// Used by other services, pages for authorization.
        /// </summary>
        Task<bool> UserCanViewAsync(int documentId, ClaimsPrincipal currentUser);

        /// <summary>
        /// Check if user can edit document metadata.
        /// </summary>
        Task<bool> UserCanEditAsync(int documentId, ClaimsPrincipal currentUser);

        /// <summary>
        /// Check if user can delete document.
        /// </summary>
        Task<bool> UserCanDeleteAsync(int documentId, ClaimsPrincipal currentUser);

        /// <summary>
        /// Get document details for authorized user only.
        /// Returns null if user lacks access.
        /// </summary>
        Task<Document?> GetDocumentAsync(int documentId, ClaimsPrincipal currentUser);
    }
}
```

---

## IDocumentShareService Interface

```csharp
namespace ContosoDashboard.Services
{
    /// <summary>
    /// Document sharing and access control.
    /// </summary>
    public interface IDocumentShareService
    {
        /// <summary>
        /// Share document with specific user.
        /// Only document uploader can share.
        /// Triggers in-app notification to recipient.
        /// </summary>
        Task<DocumentShare> ShareWithUserAsync(
            int documentId, 
            int recipientUserId, 
            ClaimsPrincipal currentUser);

        /// <summary>
        /// Share document with entire team.
        /// Notifies all team members.
        /// </summary>
        Task ShareWithTeamAsync(
            int documentId, 
            int teamId, 
            ClaimsPrincipal currentUser);

        /// <summary>
        /// Get documents shared with user.
        /// </summary>
        Task<List<DocumentSummary>> GetSharedWithMeAsync(
            int skip = 0, 
            int take = 50, 
            ClaimsPrincipal currentUser);

        /// <summary>
        /// Revoke share access.
        /// </summary>
        Task RevokeShareAsync(int shareId, ClaimsPrincipal currentUser);

        /// <summary>
        /// Check if document is shared with specific user.
        /// </summary>
        Task<bool> IsSharedWithUserAsync(int documentId, int userId);
    }
}
```

---

## Offline Upload Queue Interface

```csharp
namespace ContosoDashboard.Services
{
    /// <summary>
    /// Manages offline upload queue for when user has no internet connection.
    /// Queue-based sync approach: User can upload while offline, sync when reconnected.
    /// </summary>
    public interface IUploadQueueService
    {
        /// <summary>
        /// Queue document for upload (when offline).
        /// Stores metadata and temp file path locally.
        /// </summary>
        Task<UploadQueue> QueueUploadAsync(
            DocumentUploadRequest request, 
            Stream fileStream, 
            ClaimsPrincipal currentUser);

        /// <summary>
        /// Get user's pending uploads.
        /// Shows status (Pending, Syncing, Synced, Failed).
        /// </summary>
        Task<List<UploadQueueSummary>> GetPendingUploadsAsync(ClaimsPrincipal currentUser);

        /// <summary>
        /// Attempt to sync all pending uploads for user.
        /// Called when connection is restored.
        /// Retries failed uploads up to max attempt limit.
        /// </summary>
        Task SyncPendingUploadsAsync(int userId, CancellationToken cancellationToken = default);

        /// <summary>
        /// Manually retry failed upload.
        /// </summary>
        Task<bool> RetryFailedUploadAsync(int queueId, ClaimsPrincipal currentUser);

        /// <summary>
        /// Cancel queued upload and clean up.
        /// </summary>
        Task CancelQueuedUploadAsync(int queueId, ClaimsPrincipal currentUser);
    }
}
```

---

## IVirusScanService Interface

**Purpose**: Abstraction for virus scanning operations. Enables mock scanning in training environments and real scanning via Azure Functions in production.

**Rationale**: 
- Separates virus scan implementation from document upload logic
- Supports both synchronous mock scanning (training) and async Azure Functions (production)
- Allows flexible retry logic and offline queue processing

```csharp
namespace ContosoDashboard.Services
{
    /// <summary>
    /// Virus scanning abstraction for uploaded files.
    /// Training: Mock scanner (fast, no actual scanning)
    /// Production: Azure Functions with ClamAV integration
    /// </summary>
    public interface IVirusScanService
    {
        /// <summary>
        /// Scan file at given path for viruses/malware.
        /// Supports both sync (mock) and async (Azure Function) implementations.
        /// </summary>
        /// <param name="documentId">Document record ID for tracking</param>
        /// <param name="filePath">Full path to file in storage</param>
        /// <param name="fileName">Original file name for logging</param>
        /// <param name="fileSize">File size in bytes for logging</param>
        /// <param name="cancellationToken">Cancellation token</param>
        /// <returns>VirusScanResult with status and details</returns>
        Task<VirusScanResult> ScanFileAsync(
            int documentId, 
            string filePath, 
            string fileName, 
            long fileSize, 
            CancellationToken cancellationToken = default);

        /// <summary>
        /// Queue document for async scanning (via Azure Function).
        /// Used when offline or for batch processing.
        /// </summary>
        /// <param name="documentId">Document record ID</param>
        /// <param name="filePath">Full path to file</param>
        /// <param name="fileName">Original file name</param>
        /// <param name="fileSize">File size</param>
        /// <returns>Queue ID for tracking</returns>
        Task<int> QueueScanAsync(int documentId, string filePath, string fileName, long fileSize);

        /// <summary>
        /// Check if file is safe for access (called before download/preview).
        /// Returns false if PENDING_SCAN or QUARANTINED.
        /// </summary>
        Task<bool> IsFileSafeAsync(int documentId);
    }
}
```

### VirusScanResult Model

```csharp
namespace ContosoDashboard.Services
{
    /// <summary>
    /// Result of virus scan operation.
    /// </summary>
    public class VirusScanResult
    {
        /// <summary>
        /// Scan status: CLEAN, THREAT_DETECTED, ERROR, or PENDING_SCAN
        /// </summary>
        public VirusScanStatus Status { get; set; }

        /// <summary>
        /// Human-readable message (e.g., "File contains Win.Trojan.X" or "Scan timeout after 120s")
        /// </summary>
        public string Message { get; set; }

        /// <summary>
        /// Threat name if detected (e.g., "Eicar-Test-File")
        /// </summary>
        public string? ThreatName { get; set; }

        /// <summary>
        /// Timestamp of scan
        /// </summary>
        public DateTime ScannedAt { get; set; }

        /// <summary>
        /// Azure Function execution ID (if using cloud scanner)
        /// </summary>
        public string? ExecutionId { get; set; }
    }

    /// <summary>
    /// Virus scan status enumeration
    /// </summary>
    public enum VirusScanStatus
    {
        /// <summary>File passed scan - safe for access</summary>
        Clean,

        /// <summary>Threat/malware detected - file quarantined</summary>
        ThreatDetected,

        /// <summary>Scan operation failed or timed out - retry later</summary>
        Error,

        /// <summary>Scan not yet completed - file access blocked</summary>
        PendingScan
    }
}
```

### MockVirusScanService Implementation

For training/offline environments, this mock implementation simulates scanning:

```csharp
namespace ContosoDashboard.Services
{
    /// <summary>
    /// Mock virus scanner for training environments.
    /// Simulates 2-second scan delay, always returns CLEAN status.
    /// Uses queue for consistency with async scanning flow.
    /// </summary>
    public class MockVirusScanService : IVirusScanService
    {
        private readonly IUploadQueueService _queueService;
        private readonly ILogger<MockVirusScanService> _logger;
        private readonly ApplicationDbContext _context;

        public MockVirusScanService(
            IUploadQueueService queueService, 
            ILogger<MockVirusScanService> logger, 
            ApplicationDbContext context)
        {
            _queueService = queueService;
            _logger = logger;
            _context = context;
        }

        public async Task<VirusScanResult> ScanFileAsync(
            int documentId, 
            string filePath, 
            string fileName, 
            long fileSize, 
            CancellationToken cancellationToken = default)
        {
            _logger.LogInformation($"Mock scanning file {fileName} ({fileSize} bytes)");

            // Simulate scan delay (2 seconds)
            await Task.Delay(2000, cancellationToken);

            // Always return clean in mock (training data is safe)
            var result = new VirusScanResult
            {
                Status = VirusScanStatus.Clean,
                Message = "Mock scan passed",
                ThreatName = null,
                ScannedAt = DateTime.UtcNow,
                ExecutionId = Guid.NewGuid().ToString()
            };

            // Update document status to ACTIVE
            var document = await _context.Documents.FindAsync(documentId);
            if (document != null)
            {
                document.Status = "ACTIVE";
                await _context.SaveChangesAsync(cancellationToken);
            }

            return result;
        }

        public async Task<int> QueueScanAsync(
            int documentId, 
            string filePath, 
            string fileName, 
            long fileSize)
        {
            // Mock: Return queue ID immediately
            _logger.LogInformation($"Queued mock scan for document {documentId}");
            return documentId; // Use document ID as queue ID for simplicity
        }

        public async Task<bool> IsFileSafeAsync(int documentId)
        {
            var document = await _context.Documents.FindAsync(documentId);
            return document?.Status == "ACTIVE";
        }
    }
}
```

### Azure Functions Implementation (Production)

For cloud deployment, implement Azure Function with ClamAV:

```
DocumentScanFunction.cs (Azure Function project)
├─ Trigger: QueueTrigger("document-scan-queue")
├─ Input: DocumentScanMessage { DocumentId, FilePath, FileName, FileSize }
├─ Process:
│  1. Load document from database
│  2. Download file from AppData/uploads/
│  3. Call ClamAV via REST API (or container instance)
│  4. Update Document.Status based on result
│  5. Log audit event
│  6. Alert admin if threat detected
└─ Output: Document status changed, user notified

Configuration (local.settings.json for emulator):
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    "CLAMAV_ENDPOINT": "http://localhost:3310"
  }
}
```

**Workflow Summary**:

1. **Upload**: DocumentService.UploadAsync() → Create Document with Status=PENDING_SCAN → Queue scan message
2. **Local Dev**: Mock scanner processes queue immediately (2s delay)
3. **Cloud**: Azure Function picks up queue message → Calls ClamAV → Updates Document.Status
4. **Result**: Document transitions to ACTIVE (clean) or QUARANTINED (threat) after scan completes
5. **Offline**: Scan queue message persists in UploadQueue table until connectivity restored

---

## Data Transfer Objects (DTOs)


### DocumentUploadRequest
```csharp
public class DocumentUploadRequest
{
    public string Title { get; set; }
    public string? Description { get; set; }
    public string Category { get; set; } // "Project Documents", "Reports", etc.
    public int? AssociatedProjectId { get; set; }
    public List<string>? Tags { get; set; }
}
```

### DocumentMetadataUpdateRequest
```csharp
public class DocumentMetadataUpdateRequest
{
    public string? Title { get; set; }
    public string? Description { get; set; }
    public string? Category { get; set; }
    public List<string>? Tags { get; set; }
}
```

### DocumentFileReplaceRequest
```csharp
public class DocumentFileReplaceRequest
{
    public string? FileName { get; set; } // Optional, can detect from stream
}
```

### DocumentSummary (for list views)
```csharp
public class DocumentSummary
{
    public int DocumentId { get; set; }
    public string Title { get; set; }
    public string Category { get; set; }
    public DateTime UploadDate { get; set; }
    public long FileSize { get; set; }
    public string? AssociatedProjectName { get; set; }
    public string UploaderName { get; set; }
}
```

### DocumentSearchResult (for search results)
```csharp
public class DocumentSearchResult : DocumentSummary
{
    public string? MatchContext { get; set; } // Snippet showing where search term matched
}
```

### UploadQueueSummary
```csharp
public class UploadQueueSummary
{
    public int QueueId { get; set; }
    public string DocumentTitle { get; set; }
    public string QueueStatus { get; set; } // "Pending", "Syncing", "Synced", "Failed"
    public int RetryCount { get; set; }
    public string? ErrorMessage { get; set; }
    public DateTime QueuedDate { get; set; }
}
```

---

## Implementation Guidelines

### Authorization Pattern

Every service method MUST validate user access:

```csharp
// Example: In DocumentService.GetDocumentAsync()
public async Task<Document?> GetDocumentAsync(int documentId, ClaimsPrincipal currentUser)
{
    var document = await _context.Documents.FindAsync(documentId);
    if (document == null) return null;

    // Check access: uploader, project member, or explicitly shared
    var userId = int.Parse(currentUser.FindFirst(ClaimTypes.NameIdentifier)?.Value ?? "0");
    
    if (document.UploadedById == userId) return document; // Owner

    if (document.AssociatedProjectId.HasValue)
    {
        // Check if user is project member
        var isMember = await _context.ProjectMembers
            .AnyAsync(pm => pm.ProjectId == document.AssociatedProjectId && pm.UserId == userId);
        if (isMember) return document;
    }

    // Check if explicitly shared
    var isShared = await _context.DocumentShares
        .AnyAsync(ds => ds.DocumentId == documentId && ds.SharedWithUserId == userId);
    if (isShared) return document;

    return null; // Access denied (soft-fail, return null)
}
```

### File Upload Sequence

```
1. Generate unique file path: GenerateFilePath(userId, projectId, extension)
2. Save file to disk via IFileStorageService.UploadAsync()
3. Create Document record in database with FilePath from step 1
4. Create DocumentActivity audit log entry
5. Return Document to caller
// If step 2-3 fails, disk file is orphaned but can be cleaned by admin
// If step 3 fails, no database record created (preferred state)
```

---

**Interface Contracts Complete**: Ready for implementation
