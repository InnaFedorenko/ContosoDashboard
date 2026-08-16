# Tasks: Document Upload and Management

**Input**: Design documents from `/specs/001-document-management/`  
**Prerequisites**: plan.md, spec.md, data-model.md, quickstart.md, contracts/  
**Status**: Ready for implementation

## Format

- [ ] T001 [P] Description in path
- [ ] T012 [P] [US1] Description in path
- [ ] T014 [US1] Description in path

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Establish the document feature structure and configuration needed before implementation.

- [ ] T001 Create feature folder structure and storage locations under ContosoDashboard/Models, ContosoDashboard/Services, ContosoDashboard/Pages, and AppData/uploads
- [ ] T002 [P] Add document feature settings to ContosoDashboard/appsettings.json and ContosoDashboard/appsettings.Development.json for upload limits, file storage, and virus-scan configuration
- [ ] T003 [P] Confirm required .NET 8 project references and startup configuration in ContosoDashboard/Program.cs for service registration and file storage initialization

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that must be complete before any user story implementation.

- [ ] T004 Create Document, DocumentShare, DocumentActivity, and UploadQueue entity classes in ContosoDashboard/Models/
- [ ] T005 Update ContosoDashboard/Data/ApplicationDbContext.cs to register DbSet properties, relationships, indexes, and authorization-related query helpers
- [ ] T006 Implement IFileStorageService and LocalFileStorageService in ContosoDashboard/Services/
- [ ] T007 Implement IVirusScanService and MockVirusScanService in ContosoDashboard/Services/ for async scan queue processing and mock scan fallback
- [ ] T008 Implement IUploadQueueService and UploadQueueService in ContosoDashboard/Services/ to track offline uploads, pending sync, retries, and status transitions
- [ ] T009 Implement DocumentActivity logging support and audit event creation in ContosoDashboard/Services/DocumentActivityService.cs
- [ ] T010 [P] Create secure storage directory bootstrap and GUID-based path generation utilities in ContosoDashboard/Services/LocalFileStorageService.cs

**Checkpoint**: Foundation ready - document story implementation can begin in parallel.

---

## Phase 3: User Story 1 - Upload Document with Metadata (Priority: P1) 🎯 MVP

**Goal**: Allow authenticated users to upload documents with metadata and queue them for async virus scanning.

**Independent Test**: Upload a valid PDF, provide title/category, complete the upload flow, and confirm the document is stored and visible in My Documents.

### Implementation for User Story 1

- [ ] T011 [P] [US1] Implement document upload validation and secure file storage flow in ContosoDashboard/Services/DocumentService.cs
- [ ] T012 [P] [US1] Implement upload queueing and scan-trigger flow in ContosoDashboard/Services/DocumentService.cs and ContosoDashboard/Services/UploadQueueService.cs
- [ ] T013 [US1] Build document upload page and metadata form in ContosoDashboard/Pages/DocumentUpload.razor
- [ ] T014 [US1] Add upload progress/status indicators and offline queue messaging in ContosoDashboard/wwwroot/js/document-upload.js and ContosoDashboard/Pages/DocumentUpload.razor
- [ ] T015 [US1] Add document creation and audit logging for upload events in ContosoDashboard/Services/DocumentService.cs and ContosoDashboard/Services/DocumentActivityService.cs

**Checkpoint**: User Story 1 is fully functional and testable independently.

---

## Phase 4: User Story 2 - Browse and Filter My Documents (Priority: P1)

**Goal**: Show uploaded documents in a sortable, filterable list so users can find and organize their own work.

**Independent Test**: Navigate to My Documents, sort by upload date, apply category/project/date filters, and verify only matching records are displayed.

### Implementation for User Story 2

- [ ] T016 [P] [US2] Implement user document listing and filtering queries in ContosoDashboard/Services/DocumentService.cs
- [ ] T017 [US2] Build My Documents page with table, sorting, category/project/date filters in ContosoDashboard/Pages/MyDocuments.razor
- [ ] T018 [P] [US2] Add project-scoped document list and filtering in ContosoDashboard/Pages/ProjectDocuments.razor

**Checkpoint**: User Stories 1 and 2 work independently and can be validated together.

---

## Phase 5: User Story 3 - Download and Preview Documents (Priority: P1)

**Goal**: Let users download and preview documents they have permission to access.

**Independent Test**: Open a document, click download or preview, and verify the original file or browser preview loads without access violations.

### Implementation for User Story 3

- [ ] T019 [P] [US3] Implement document access checks and download/stream logic in ContosoDashboard/Services/DocumentService.cs
- [ ] T020 [US3] Add document detail actions for preview and download in ContosoDashboard/Pages/DocumentDetail.razor
- [ ] T021 [P] [US3] Add dashboard recent documents widget in ContosoDashboard/Pages/Index.razor and/or ContosoDashboard/Services/DashboardService.cs

**Checkpoint**: Upload, browsing, and access flows are independently usable.

---

## Phase 6: User Story 4 - Search for Documents by Title, Description, and Tags (Priority: P2)

**Goal**: Provide fast search across metadata and shared access scope.

**Independent Test**: Search for a known title, tag, project name, or uploader, and validate results are returned within 2 seconds and restricted to accessible documents.

### Implementation for User Story 4

- [ ] T022 [P] [US4] Implement document search query and access filtering in ContosoDashboard/Services/DocumentService.cs
- [ ] T023 [US4] Build document search page and result presentation in ContosoDashboard/Pages/DocumentSearch.razor
- [ ] T024 [P] [US4] Add quick-search hooks and navigation entries in ContosoDashboard/Shared/NavMenu.razor

**Checkpoint**: Search works independently with proper access enforcement.

---

## Phase 7: User Story 5 - Manage Document Metadata (Priority: P2)

**Goal**: Let document owners update metadata and replace outdated files without losing record ownership or audit entry history.

**Independent Test**: Edit a document title/category/tags, replace the file, and confirm the metadata and original uploader/date remain consistent.

### Implementation for User Story 5

- [ ] T025 [P] [US5] Implement metadata update and file-replacement logic in ContosoDashboard/Services/DocumentService.cs
- [ ] T026 [US5] Update document detail UI and edit actions in ContosoDashboard/Pages/DocumentDetail.razor
- [ ] T027 [P] [US5] Add audit entries for metadata and replacement events in ContosoDashboard/Services/DocumentActivityService.cs

**Checkpoint**: Document owners can manage metadata and file replacement without crossing permissions.

---

## Phase 8: User Story 6 - Delete Documents Permanently (Priority: P2)

**Goal**: Allow authorized users to delete documents after confirmation and keep the system in a consistent, auditable state.

**Independent Test**: Delete a document with confirmation, validate it is removed from storage and lists, and verify the action is logged.

### Implementation for User Story 6

- [ ] T028 [US6] Implement soft-delete and permanent delete flows in ContosoDashboard/Services/DocumentService.cs
- [ ] T029 [P] [US6] Add confirmation UI and delete action in ContosoDashboard/Pages/MyDocuments.razor and ContosoDashboard/Pages/DocumentDetail.razor
- [ ] T030 [US6] Add delete audit records and cleanup behavior for storage and access data in ContosoDashboard/Services/DocumentActivityService.cs

**Checkpoint**: Deletion and cleanup behavior is validated independently from other stories.

---

## Phase 9: User Story 7 - Share Documents with Specific Users and Teams (Priority: P3)

**Goal**: Enable controlled sharing and notifications for collaborative document access.

**Independent Test**: Share a document with a recipient, confirm they receive a notification, and verify access works from the Shared with Me view.

### Implementation for User Story 7

- [ ] T031 [P] [US7] Implement DocumentShareService rules and share persistence in ContosoDashboard/Services/DocumentShareService.cs
- [ ] T032 [US7] Integrate sharing notifications with ContosoDashboard/Services/NotificationService.cs
- [ ] T033 [US7] Add share UI and shared-documents view in ContosoDashboard/Pages/DocumentDetail.razor and ContosoDashboard/Pages/Notifications.razor

**Checkpoint**: Sharing delivery and recipient access are independently testable.

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: Final hardening across the entire feature set.

- [ ] T034 [P] Review and harden RBAC and IDOR guard logic across ContosoDashboard/Services/DocumentService.cs and ContosoDashboard/Services/DocumentShareService.cs
- [ ] T035 [P] Run the document quickstart validation scenarios in specs/001-document-management/quickstart.md and fix any end-to-end issues
- [ ] T036 [P] Verify all document pages include role checks, secure navigation, and consistent permission messaging in ContosoDashboard/Pages/
- [ ] T037 [P] Update README and feature documentation references to reflect the document upload and management workflow in README.md and StakeholderDocs/

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 Setup**: No dependencies; can start immediately
- **Phase 2 Foundational**: Depends on Phase 1 completion and blocks all user story work
- **User Story Phases 3-9**: All depend on Phase 2 completion
- **Phase 10 Polish**: Depends on all desired user story phases being complete

### User Story Dependencies

- **US1 (P1)**: No dependencies beyond foundation; MVP story
- **US2 (P1)**: Depends on US1 data and storage model; can be developed in parallel once foundation is complete
- **US3 (P1)**: Depends on upload/browse structure and access rules
- **US4 (P2)**: Depends on document metadata and access model
- **US5 (P2)**: Depends on US1 and access rules
- **US6 (P2)**: Depends on US1 and storage cleanup rules
- **US7 (P3)**: Depends on document access and notification infrastructure

### Parallel Opportunities

- All Phase 1 tasks marked [P] can run in parallel
- All Phase 2 tasks marked [P] can run in parallel
- Once foundation is complete, US1 and US2 can proceed in parallel as independent MVP streams
- US3, US4, and US5 can proceed in parallel after shared foundation is complete
- US6 and US7 can be scheduled in parallel with remaining polish tasks if staffing allows

---

## Parallel Example: User Story 1 and 2

```bash
# Launch upload and browse tasks in parallel once foundation is complete:
# - T011 [P] [US1] Implement upload validation and storage flow
# - T016 [P] [US2] Implement list and filter queries
# - T013 [US1] Build upload form
# - T017 [US2] Build My Documents list UI
```

---

## Implementation Strategy

### MVP First (User Story 1 only)

1. Complete Phase 1 and Phase 2
2. Complete User Story 1 upload flow
3. Validate upload, metadata, and queued scan flow
4. Stop and confirm the MVP is stable before moving to browse/search/share features

### Incremental Delivery

1. Ship upload + browse + preview as the first business-ready increment
2. Add search and metadata management next
3. Add delete/share workflows last, after RBAC and audit logging are stable
4. Finish with polish and cross-cutting validation against quickstart.md
