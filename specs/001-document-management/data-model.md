# Data Model Design: Document Upload and Management

**Feature**: Document Upload and Management  
**Created**: 2026-08-16  
**Specification**: [spec.md](../spec.md)

## Entity Relationship Diagram

```
User (existing)
  ├── 1:N → Document (as UploadedBy)
  ├── 1:N → DocumentShare (as SharedWithUser)
  ├── 1:N → DocumentShare (as SharedByUser)
  └── 1:N → DocumentActivity (as Actor)

Project (existing)
  ├── 1:N → Document (as AssociatedProject, optional)
  └── (Document access inherited through project membership)

Document
  ├── M:1 → User (UploadedBy)
  ├── M:1 → Project (AssociatedProject, optional)
  ├── 1:N → DocumentShare
  ├── 1:N → DocumentActivity
  └── 1:N → UploadQueue (queued for sync)

DocumentShare
  ├── M:1 → Document
  ├── M:1 → User (SharedWithUser)
  └── M:1 → User (SharedByUser)

DocumentActivity (Audit Log)
  ├── M:1 → Document
  └── M:1 → User (Actor)

UploadQueue (Offline Support)
  └── 1:1 → Document (on successful sync)
```

## Entity Schemas

### Document

Represents uploaded document file with metadata.

| Column | Type | Constraints | Notes |
|--------|------|-----------|-------|
| `DocumentId` | int | PK, AutoIncrement | Consistent with User/Project keys (integer, not GUID) |
| `Title` | nvarchar(255) | NOT NULL, UNIQUE(per user) | Required metadata |
| `Description` | nvarchar(1000) | Nullable | Optional user-provided description |
| `Category` | nvarchar(50) | NOT NULL | Text value ("Project Documents", "Personal Files", "Reports", "Presentations", "Team Resources", "Other") |
| `FilePath` | nvarchar(500) | NOT NULL | GUID-based: `{userId}/{projectId or "personal"}/{guid}.{ext}` |
| `FileName` | nvarchar(255) | NOT NULL | Original filename uploaded by user (for download) |
| `FileSize` | bigint | NOT NULL | File size in bytes |
| `FileType` | nvarchar(255) | NOT NULL | MIME type (e.g., "application/pdf", allows up to 255 chars for Office docs) |
| `UploadDate` | datetime2 | NOT NULL, Default=NOW() | UTC timestamp |
| `UploadedById` | int | FK → User | Who uploaded this document |
| `AssociatedProjectId` | int | FK → Project, Nullable | Project document relates to (optional) |
| `Tags` | nvarchar(max) | Nullable | Serialized JSON array ["tag1", "tag2"] or pipe-separated "tag1\|tag2" |
| `Status` | nvarchar(20) | NOT NULL, Default='Active' | Enum: Active, Deleted (soft delete support) |
| `CreatedDate` | datetime2 | NOT NULL, Default=NOW() | Record creation time |
| `ModifiedDate` | datetime2 | Nullable | Last modification time |

**Indexes**:
- PK: DocumentId
- FK: UploadedById
- FK: AssociatedProjectId
- Composite: (UploadedById, Status) - for user's documents
- Composite: (AssociatedProjectId, Status) - for project documents
- Full-text: Title, Description, Category (for search FR-013)

**Unique Constraints**:
- (UploadedById, Title, Status) - prevent duplicate titles per user

### DocumentShare

Audit trail for document sharing relationships.

| Column | Type | Constraints | Notes |
|--------|------|-----------|-------|
| `ShareId` | int | PK, AutoIncrement | Unique share record ID |
| `DocumentId` | int | FK → Document, NOT NULL | Which document was shared |
| `SharedWithUserId` | int | FK → User, NOT NULL | Who received access to document |
| `SharedByUserId` | int | FK → User, NOT NULL | Who initiated the share |
| `SharedDate` | datetime2 | NOT NULL, Default=NOW() | When share occurred |
| `RevokedDate` | datetime2 | Nullable | When share was revoked (if ever) |

**Indexes**:
- PK: ShareId
- FK: DocumentId
- FK: SharedWithUserId
- FK: SharedByUserId
- Composite: (DocumentId, SharedWithUserId) - for access control checks

**Unique Constraints**:
- (DocumentId, SharedWithUserId) - prevent duplicate shares

### DocumentActivity

Comprehensive audit log for compliance and troubleshooting.

| Column | Type | Constraints | Notes |
|--------|------|-----------|-------|
| `ActivityId` | int | PK, AutoIncrement | Unique activity log entry |
| `DocumentId` | int | FK → Document | Which document action affected |
| `UserId` | int | FK → User | Who performed the action |
| `ActivityType` | nvarchar(50) | NOT NULL | Enum: Upload, Download, Delete, Share, MetadataChange, PreviewViewed, Permission Change |
| `ActivityDate` | datetime2 | NOT NULL, Default=NOW() | When action occurred (UTC) |
| `Details` | nvarchar(500) | Nullable | Context-specific data (e.g., "Shared with John Smith", "Changed category from Reports to Presentations") |

**Indexes**:
- PK: ActivityId
- FK: DocumentId
- FK: UserId
- Composite: (DocumentId, ActivityDate) - for document audit trail
- Composite: (UserId, ActivityDate) - for user activity tracking

### UploadQueue

Stores pending uploads for offline sync support (FR-005a-d).

| Column | Type | Constraints | Notes |
|--------|------|-----------|-------|
| `QueueId` | int | PK, AutoIncrement | Unique queue entry |
| `DocumentId` | int | FK → Document, Nullable | Related document (NULL while queued, filled on successful sync) |
| `UserId` | int | FK → User | Who queued this upload |
| `DocumentMetadata` | nvarchar(max) | NOT NULL | JSON-serialized: {title, description, category, projectId, tags} |
| `FileData` | nvarchar(500) | NOT NULL | Temporary local file path while offline |
| `QueueStatus` | nvarchar(20) | NOT NULL, Default='Pending' | Enum: Pending, Syncing, Synced, Failed |
| `RetryCount` | int | NOT NULL, Default=0 | Number of sync attempts |
| `ErrorMessage` | nvarchar(500) | Nullable | Last error message if sync failed |
| `QueuedDate` | datetime2 | NOT NULL, Default=NOW() | When queued locally |
| `SyncedDate` | datetime2 | Nullable | When successfully synced to server |

**Indexes**:
- PK: QueueId
- FK: UserId
- FK: DocumentId
- Composite: (UserId, QueueStatus) - for user's pending uploads

---

## Design Decisions

### Integer Keys (not GUIDs)
**Decision**: Use integer PK/FK for DocumentId consistent with existing User/Project tables  
**Rationale**: Consistency with established schema, reduces storage overhead, sufficient cardinality for training application

### Text Categories (not Enums)
**Decision**: Category column stores text values ("Project Documents", "Team Resources", etc.) instead of integer enum  
**Rationale**: Flexibility for future category additions, easier ad-hoc SQL queries, simplified UI data binding

### GUID-based File Paths
**Decision**: Store files with GUID-based names: `{userId}/{projectId}/{guid}.{ext}`  
**Rationale**: Prevents path traversal attacks, avoids filename collisions, enables duplicate file detection

### JSON Serialization for Tags
**Decision**: Tags stored as JSON array in nvarchar(max) column  
**Rationale**: Flexibility for variable number of tags per document, simpler than junction table, sufficient for query filtering

### Soft Delete Pattern
**Decision**: Status column (Active/Deleted) enables soft deletes for audit trail  
**Rationale**: Maintains audit logs even after deletion, enables accidental recovery, required for compliance

### Separate UploadQueue Table
**Decision**: Separate queue table (not a status column on Document) tracks offline uploads  
**Rationale**: Cleaner separation of concerns, prevents partial Document records, enables retry logic independently

---

## SQL Schema Generation Notes

For Entity Framework Core:
```csharp
// Models required:
public class Document { /* properties above */ }
public class DocumentShare { /* properties above */ }
public class DocumentActivity { /* properties above */ }
public class UploadQueue { /* properties above */ }

// DbContext additions:
public DbSet<Document> Documents { get; set; }
public DbSet<DocumentShare> DocumentShares { get; set; }
public DbSet<DocumentActivity> DocumentActivities { get; set; }
public DbSet<UploadQueue> UploadQueues { get; set; }

// OnModelCreating() configuration:
// - Set primary keys
// - Configure foreign keys with OnDelete behavior
// - Add indexes (composite, FK, full-text)
// - Add unique constraints
// - Set column constraints (NOT NULL, defaults, max lengths)
```

---

## Relationships & Constraints

### Data Integrity Rules

1. **Document Ownership**: Document.UploadedById must reference valid User. Cannot be NULL.
2. **Project Association**: If Document.AssociatedProjectId is NOT NULL, document is project-scoped; user must be project member to access.
3. **Share Permissions**: Can only share with users who are NOT the uploader.
4. **Activity Audit**: Every state change (upload, download, delete, share, metadata change) must create ActivityLog entry.
5. **Delete Behavior**: Deleting document cascades to DocumentShare (access revoked) and creates Delete activity log.
6. **Queue Sync**: UploadQueue entry must resolve to Document record on successful sync, or remain in Failed state for manual retry.

### Authorization Rules (Service Layer Enforced)

- **View**: User can see document if: (1) uploaded by user, OR (2) document is in their projects, OR (3) document was explicitly shared with user
- **Edit Metadata**: Only uploader or Project Manager (if project-scoped) can edit
- **Delete**: Only uploader or Project Manager can delete
- **Share**: Only uploader or Project Manager can share
- **Download**: Any user with View access can download
- **Preview**: Any user with View access can preview

---

**Data Model Complete**: Ready for implementation tasks
