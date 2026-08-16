# Implementation Plan: Document Upload and Management

**Branch**: `001-document-management` | **Date**: 2026-08-16 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-document-management/spec.md`
**Status**: Phase 1 (Design) - Ready for task decomposition

## Summary

Add centralized document upload and management capabilities to ContosoDashboard, enabling employees to securely upload, organize, browse, search, and share work-related documents. Core features include: multi-file upload with metadata, role-based access control (IDOR protection), offline upload queue support, document preview, and audit logging. Technical approach: Service layer architecture with IFileStorageService abstraction for future cloud migration. Local filesystem storage (`AppData/uploads`) with GUID-based file paths. Entity Framework Core data model with Document, DocumentShare, and DocumentActivity entities. Three entity data model (7 P1-P3 user stories, 34 functional requirements, 10 measurable success criteria).

## Technical Context

**Language/Version**: C# 12 with .NET 8  
**Primary Dependencies**: Entity Framework Core 8.0, Microsoft.AspNetCore.Authentication (claims-based), Blazor Server 8.0  
**Storage**: SQLite (training) / SQL Server (configurable), Local filesystem (`AppData/uploads/`)  
**Testing**: xUnit or MSTest (to be specified in tasks)  
**Target Platform**: Web application (Blazor Server + Razor Pages)  
**Project Type**: ASP.NET Core web application (single monolithic project)  
**Performance Goals**: Document upload <30s (25MB), list load <2s (500 docs), search <2s, preview <3s  
**Constraints**: Offline-capable with queue-based sync, IDOR protection mandatory, no cloud services (training), mock auth system only  
**Scale/Scope**: 4 user roles, 7 features (P1-P3), 3 core entities + upload queue, ~2000 LOC estimated  

**Key Decisions**:
- Offline upload queueing: Queue-based sync with automatic retry (cleared: Q1)
- Storage quotas: Out of scope for MVP (unlimited storage) (cleared: Q2)
- File storage: Local filesystem with IFileStorageService abstraction for future cloud migration
- Architecture: Service layer pattern with dependency injection, role-based authorization at service level
- Virus scanning: Azure Functions with Queue Storage triggers for async background processing (see Virus Scanning Architecture below)

### Virus Scanning Architecture (Background Job Processing)

**Requirement**: FR-007 mandates virus scanning of all uploaded files before storage access.

**Implementation Approach**: Asynchronous background job processing with Azure Functions triggered by queue events. This prevents upload blocking on scan latency and supports offline-first workflows.

**System Design**:

```
1. Upload Flow:
   Upload → Validate metadata → Store file → Create DocumentQueue entry → Queue scan message
         ↓ (async) ↓
   Return upload confirmation to user (file in PENDING_SCAN status)
   
2. Scan Flow (Azure Function triggered by Queue):
   Queue trigger → Extract document metadata → Invoke virus scanner → Update status
   ├─ Success: Status = ACTIVE, DocumentQueue status = SCANNED_CLEAN
   └─ Threat: Status = QUARANTINED, DocumentQueue status = SCAN_FAILED, Admin alert

3. Access Control:
   • Only ACTIVE documents returned in search/browse
   • PENDING_SCAN documents visible only to uploader (FR-006: metadata visible during upload)
   • QUARANTINED documents only visible to admin for manual review/deletion
```

**Azure Infrastructure**:

| Component | Purpose | Configuration |
|-----------|---------|----------------|
| **Azure Storage Queue** | Async task queue for scan jobs | Queue name: `document-scan-queue` |
| **Azure Function (Timer + Queue)** | Processes scan messages | Runtime: .NET 8 (isolated), Trigger: QueueTrigger |
| **Azure ClamAV Integration** | Virus definition source | Via Azure Container Instances or ClamAV REST API |
| **Document Queue Table** | MVP: SQL Server table tracking scan status | Entity: UploadQueue with QueueStatus column |

**Local Development/Testing**:

For training environments (no cloud), implement mock scanning:
- `IVirusScanService` interface with `ScanFileAsync()` method
- Mock implementation: Simulate scan delay (2s) + return CLEAN status
- Real Azure Functions deployment: Use actual ClamAV integration when deploying to cloud
- Configuration: `appsettings.Development.json` switches between mock and real scanners

**Implementation Tasks**:

| Task | Service Layer | Azure Component | Status |
|------|---------------|-----------------|--------|
| Define IVirusScanService interface | Services/IVirusScanService.cs | N/A (local only) | TBD |
| Implement mock scanner | Services/MockVirusScanService.cs | N/A | TBD |
| Update DocumentService to invoke scan | Services/DocumentService.cs | N/A | TBD |
| Create UploadQueue processor | Services/UploadQueueService.cs | N/A | TBD |
| Document status states (PENDING_SCAN, ACTIVE, QUARANTINED) | Models/Document.cs (Status enum) | N/A | TBD |
| Azure Function: DocumentScanFunction | N/A (cloud deployment) | Azure Function with QueueTrigger | TBD |
| Queue integration tests | Tests/VirusScanIntegrationTests.cs | Emulated Queue Storage | TBD |

**Workflow Details**:

**Step 1: Upload Triggers Scan Queue**
```csharp
// Services/DocumentService.cs (pseudo-code)
public async Task<DocumentSummary> UploadAsync(DocumentUploadRequest request, User uploader)
{
    // 1. Validate file (size, type)
    // 2. Generate secure file path
    // 3. Store file to AppData/uploads/{userId}/{guid}.{ext}
    // 4. Create Document entity with Status = PENDING_SCAN
    // 5. Save to database
    // 6. Queue scan message: {DocumentId, FilePath, FileName, FileSize}
    // 7. Return DocumentSummary (file in PENDING_SCAN state, visible to uploader)
}
```

**Step 2: Azure Function Processes Queue**
```
Trigger: QueueTrigger receives message from document-scan-queue
Input: DocumentScanMessage { DocumentId, FilePath, FileName, FileSize }
Process:
  1. Load Document from database (check exists & PENDING_SCAN status)
  2. Retrieve file from AppData/uploads/
  3. Call virus scanner via IVirusScanService
  4. Update Document.Status based on result:
     ✓ CLEAN: Status = ACTIVE, UploadQueue.QueueStatus = SCANNED_CLEAN
     ✗ THREAT: Status = QUARANTINED, UploadQueue.QueueStatus = SCAN_FAILED
     ✗ ERROR: Status = PENDING_SCAN, UploadQueue.QueueStatus = SCAN_PENDING_RETRY
  5. Log audit event (DocumentActivity record)
  6. If threat detected: alert Admin via NotificationService
Output: Document status updated; user sees ACTIVE doc when scan completes
```

**Step 3: User Experience**
- **During upload**: "File uploaded, scanning in progress..." (PENDING_SCAN)
- **After clean scan**: Document appears in MyDocuments, searchable, downloadable (ACTIVE)
- **After threat detected**: Admin notified, user sees "File quarantined - admin review required" (QUARANTINED)

**Offline Handling**:

When offline before scan completes:
1. Document stored locally with PENDING_SCAN status
2. Scan queue message persisted in UploadQueue table
3. On reconnection: Background sync service processes queue
4. Document status updated when connection restored
5. Notification pushed to user when scan completes

**Configuration** (appsettings.json):

```json
{
  "VirusScanning": {
    "Enabled": true,
    "ScannerType": "Mock",  // "Mock" for dev, "AzureFunctions" for cloud
    "MaxConcurrentScans": 5,
    "ScanTimeoutSeconds": 120,
    "QuarantineNotificationEmail": "security@contoso.com",
    "MockScanDelayMs": 2000
  },
  "AzureStorageQueue": {
    "ConnectionString": "UseDevelopmentStorage=true",  // Emulator for dev
    "QueueName": "document-scan-queue"
  }
}
```

**Testing Strategy**:

| Scenario | Implementation | Validation |
|----------|-----------------|-----------|
| Clean file scan | Mock scanner returns CLEAN | Document status = ACTIVE after scan |
| Threat detected | Mock scanner returns THREAT | Document status = QUARANTINED, admin notified |
| Scan timeout | Mock scanner throws TimeoutException | UploadQueue retry count incremented |
| Queue message lost | Rescan triggered manually via admin UI | Document re-queued for scanning |
| Offline upload + scan | Queue message persists, sync processes on reconnect | Document becomes ACTIVE when sync completes |
| Concurrent uploads | 5 concurrent scans processed | All documents scanned without blocking |

**Migration Path to Production**:

1. **MVP (Training)**: Mock scanner in Services/ layer, UploadQueue table
2. **Phase 2 (Cloud)**: Deploy DocumentScanFunction to Azure Functions, connect to ClamAV REST API
3. **Phase 3 (Scale)**: Upgrade to Azure Container Instances for custom scanner VM
4. **Optional Enhancement**: Storage account lifecycle management (move quarantined files to archive)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### ContosoDashboard Constitution v1.0.0 Alignment

| Principle | Status | Notes |
|-----------|--------|-------|
| **I. Spec-Driven Development** | ✅ PASS | Feature fully specified with 7 user stories, 34 FR, 10 success criteria. No deviations from approved spec allowed. |
| **II. Security-First by Design** | ✅ PASS | Designed with: FR-007 (virus scanning), FR-008 (secure storage), FR-009 (RBAC), FR-010 (IDOR protection), service-level auth checks. |
| **III. Role-Based Access & User Isolation** | ✅ PASS | FR-009 enforces role-based upload/management. FR-010 prevents cross-user data access. Service layer validates user context before data access. |
| **IV. Service Layer Architecture** | ✅ PASS | Business logic isolated in `/Services/DocumentService`, `IFileStorageService`, `DocumentShareService`. No business logic in pages. Dependencies injected. |
| **V. Code Quality & Documentation** | ✅ PASS | Implementation tasks will include: XML docs for public methods, inline comments for complex logic, PascalCase naming, no commented-out code. |

**Constitution Gates**: ✅ ALL PASS - No violations or justification required.

**Post-Design Re-check**: (To be completed after Phase 1 design artifacts are finalized)

## Project Structure

### Documentation (this feature)

```text
specs/001-document-management/
├── spec.md                  # Feature specification
├── plan.md                  # This file (implementation plan)
├── research.md              # Phase 0 research findings (TBD)
├── data-model.md            # Phase 1 data model design (TBD)
├── quickstart.md            # Phase 1 validation/quickstart guide (TBD)
├── contracts/               # Phase 1 interface contracts (TBD)
│   └── document-api.md
└── checklists/
    └── requirements.md      # Quality validation checklist
```

### Source Code (ContosoDashboard project)

Feature implements within existing monolithic ASP.NET Core structure:

```text
ContosoDashboard/
├── Models/                          # Data entities
│   ├── Document.cs                  # NEW: Document entity
│   ├── DocumentShare.cs             # NEW: Share relationship
│   ├── DocumentActivity.cs          # NEW: Audit log entity
│   ├── UploadQueue.cs               # NEW: Offline upload queue
│   ├── User.cs                      # EXISTING: Use for FK relationships
│   └── Project.cs                   # EXISTING: Use for FK relationships
│
├── Data/
│   └── ApplicationDbContext.cs      # MODIFY: Add DbSet<Document>, DbSet<DocumentShare>, etc.
│
├── Services/                        # Business logic layer
│   ├── DocumentService.cs           # NEW: Document CRUD, role-based access
│   ├── IFileStorageService.cs       # NEW: Interface for file storage abstraction
│   ├── LocalFileStorageService.cs   # NEW: Local filesystem implementation
│   ├── DocumentShareService.cs      # NEW: Sharing logic
│   ├── DocumentActivityService.cs   # NEW: Audit logging
│   └── [existing services]          # NotificationService (integrate for share notifications)
│
├── Pages/                           # UI components & Razor pages
│   ├── MyDocuments.razor            # NEW: User's documents with filter/sort
│   ├── ProjectDocuments.razor       # NEW: Project-scoped document view
│   ├── DocumentUpload.razor         # NEW: Upload component
│   ├── DocumentDetail.razor         # NEW: Document preview & metadata
│   ├── DocumentSearch.razor         # NEW: Search interface
│   └── [existing pages]
│
├── wwwroot/
│   ├── css/
│   │   └── documents.css            # NEW: Styling for document UI
│   └── js/
│       └── document-upload.js       # NEW: Upload progress & queue UI
│
└── AppData/
    └── uploads/                     # NEW: Local file storage directory
        └── {userId}/{projectId}/{guid}.{ext}
```

**Structure Decision**: Single monolithic project structure (existing). Document feature integrates into established layers: Models (data entities), Data (EF Core DbSets), Services (business logic), Pages (UI). File storage in `AppData/uploads/` outside `wwwroot/`. No architecture changes required.

## Complexity Tracking

> **No Constitution violations - No justification required**

All design decisions align with ContosoDashboard Constitution v1.0.0 principles. No additional complexity justification needed.

---

## Phase 0: Research & Validation

**Status**: ✅ COMPLETE - No unknowns remain from specification clarification session

### Research Tasks Resolved

| Task | Status | Finding |
|------|--------|---------|
| Offline upload handling | ✅ Resolved | Queue-based sync with automatic retry (Q1 clarification) |
| Storage quota enforcement | ✅ Resolved | Out of scope for MVP; unlimited storage in v1 (Q2 clarification) |
| Virus scanning integration point | ✅ Identified | FR-007 requires scan before storage; integration details deferred to implementation tasks |
| Notification system availability | ✅ Confirmed | Existing NotificationService can be used for sharing notifications |
| File storage pattern | ✅ Specified | GUID-based paths: `{userId}/{projectId}/{guid}.{ext}` prevents path traversal attacks |

---

## Phase 1: Design & Contracts

### Design Artifacts Generated

All design artifacts below are created by this plan phase and ready for task decomposition via `/speckit.tasks`.

**Status**: ✅ Design artifacts created - See linked files below

#### 1. Data Model (data-model.md)

[See linked file: data-model.md](data-model.md)

#### 2. Interface Contracts (contracts/)

[See linked files in contracts/ directory](contracts/)

#### 3. Quickstart Validation Guide (quickstart.md)

[See linked file: quickstart.md](quickstart.md)

---

## Completion Status

✅ **Phase 0 (Research)**: Complete - All clarifications resolved  
✅ **Phase 1 (Design)**: Complete - Data model, contracts, and quickstart artifacts created  
✅ **Constitution Check**: PASS - All 5 principles aligned  
✅ **Ready for Phase 2**: Task decomposition via `/speckit.tasks`

**Plan Created**: 2026-08-16  
**Status**: Ready for implementation task generation
