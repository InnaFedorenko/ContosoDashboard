# Quickstart & Validation Guide: Document Management

**Feature**: Document Upload and Management  
**Created**: 2026-08-16  
**Purpose**: End-to-end validation scenarios proving feature requirements are met

This guide documents runnable scenarios that validate the document management feature works as specified. These are NOT full test suites - they're acceptance scenarios that verify key features end-to-end.

---

## Pre-Requisites

- [ ] Application running on `http://localhost:5000`
- [ ] Database initialized with tables (Document, DocumentShare, DocumentActivity, UploadQueue)
- [ ] Mock authentication configured with 4 test users:
  - Admin: `admin@contoso.com` (Administrator role)
  - Project Manager: `camille.nicole@contoso.com` (Project Manager role)
  - Team Lead: `floris.kregel@contoso.com` (Team Lead role)
  - Employee: `ni.kang@contoso.com` (Employee role)
- [ ] Sample projects created and users assigned to projects
- [ ] Test files ready:
  - `sample-document.pdf` (5 MB PDF)
  - `sample-report.docx` (2 MB Word document)
  - `sample-image.jpg` (1 MB image)

---

## Scenario 1: Basic Document Upload (P1)

**Goal**: Verify FR-001, FR-004, FR-005, FR-006 - User can upload single document with required metadata

**Test User**: Employee (`ni.kang@contoso.com`)

**Steps**:

1. Login as Employee
2. Navigate to Documents → Upload Document
3. Select file: `sample-document.pdf`
4. Enter metadata:
   - Title: "Q3 Budget Report" (required)
   - Description: "Initial draft for leadership review" (optional)
   - Category: "Reports" (required)
   - Associated Project: "Contoso Redesign" (optional)
   - Tags: "Q3, budget, draft" (optional)
5. Click "Upload"
6. Observe upload progress indicator showing percentage

**Expected Results**:
- [ ] Upload completes within 30 seconds (SC-005)
- [ ] Success message displayed: "Document uploaded successfully"
- [ ] Document appears in "My Documents" list (FR-011)
- [ ] All metadata displayed correctly
- [ ] Upload date/time populated automatically (FR-006)
- [ ] Uploader name shown as "Ni Kang" (FR-006)
- [ ] File size shows ~5 MB (FR-006)
- [ ] MIME type recorded as "application/pdf" (FR-006)

**Acceptance Criteria**: ✅ PASS if all results observed

---

## Scenario 2: Document Upload with Unsupported File Type (Security)

**Goal**: Verify FR-003 - System rejects unsupported file types

**Test User**: Employee (`ni.kang@contoso.com`)

**Steps**:

1. Login as Employee
2. Navigate to Documents → Upload Document
3. Select file: `malicious.exe` (or any unsupported type)
4. Attempt to upload
5. Observe system response

**Expected Results**:
- [ ] Upload rejected before file sent to server (client-side validation)
- [ ] Error message: "File type not supported. Supported types: PDF, Word, Excel, PowerPoint, text, JPEG, PNG"
- [ ] No database record created
- [ ] No file saved to disk

**Acceptance Criteria**: ✅ PASS if rejection occurs

---

## Scenario 3: Document Upload Exceeding Size Limit (Validation)

**Goal**: Verify FR-003 - System rejects files exceeding 25 MB

**Test User**: Employee (`ni.kang@contoso.com`)

**Steps**:

1. Login as Employee
2. Navigate to Documents → Upload Document
3. Select file: `large-file-26mb.zip` (exactly 26 MB)
4. Attempt to upload
5. Observe system response

**Expected Results**:
- [ ] Upload rejected with error message: "File size exceeds 25 MB limit"
- [ ] No database record created
- [ ] No partial file saved to disk

**Acceptance Criteria**: ✅ PASS if rejection occurs before server processing

---

## Scenario 3b: Virus Scanning Background Job (Security - FR-007)

**Goal**: Verify FR-007 - Uploaded files scanned for viruses before storage access. Async scanning via background job.

**Test User**: Employee (`ni.kang@contoso.com`)

**Prerequisites**:
- [ ] Mock virus scanner configured in `appsettings.Development.json`
- [ ] UploadQueue table initialized
- [ ] Background job processor running (or manual trigger available for testing)

**Test Flow - Part 1: Upload Triggers Scan Queue**

**Steps**:

1. Login as Employee
2. Navigate to Documents → Upload Document
3. Select file: `sample-document.pdf` (5 MB)
4. Enter metadata:
   - Title: "Q3 Budget Report"
   - Category: "Reports"
5. Click "Upload"
6. Observe success message displayed immediately (upload completes)
7. Verify file status shows "Scanning..." in My Documents
8. Check database UploadQueue table: Verify entry created with:
   - [ ] DocumentId: (auto-assigned)
   - [ ] QueueStatus: "Pending"
   - [ ] RetryCount: 0

**Expected Results - Part 1**:
- [ ] Upload completes within 30 seconds (FR-005)
- [ ] Document created with Status = "PENDING_SCAN" (not ACTIVE yet)
- [ ] Success message: "Document uploaded, virus scanning in progress..."
- [ ] Scan queue entry created in UploadQueue table
- [ ] Document visible only to uploader during scan (not in shared lists)
- [ ] Upload confirmation returned to user immediately (async scan)

**Acceptance Criteria - Part 1**: ✅ PASS if upload queues file for scanning

---

**Test Flow - Part 2: Mock Scanner Processes Queue**

**Steps** (if using sync mock scanner):

1. Wait 2-3 seconds (mock scan simulates 2s delay)
2. Refresh browser or check document status in My Documents
3. Verify document status changed from "Scanning..." to "Active"
4. Observe document now fully accessible (downloadable, previewable, searchable)
5. Check DocumentActivity table: Verify "VirusScan" event logged

**Expected Results - Part 2**:
- [ ] Mock scanner completes within 3 seconds (2s simulated scan + overhead)
- [ ] Document Status updated to "ACTIVE"
- [ ] Document visible in browse/search results
- [ ] DocumentActivity record created with ActivityType="VirusScan", Status="Clean"
- [ ] UploadQueue.QueueStatus updated to "Scanned_Clean"

**Acceptance Criteria - Part 2**: ✅ PASS if scan completes and document becomes accessible

---

**Test Flow - Part 3: Threat Detected Scenario**

**Steps** (if test framework supports threat injection):

1. Configure test to inject threat detection in mock scanner
2. Upload file: `eicar-test-file.txt` (EICAR test standard malware signature)
3. Observe system response
4. Check document Status in database: Should be "QUARANTINED"
5. Attempt to download document as uploader: Should be blocked with "File quarantined" message
6. Verify admin notification sent (check Notification records in database or notification service logs)

**Expected Results - Part 3**:
- [ ] Document Status changed to "QUARANTINED"
- [ ] Document removed from uploader's browse/download (blocked)
- [ ] Admin receives alert notification (FR-007 security requirement)
- [ ] UploadQueue.QueueStatus = "Scan_Failed"
- [ ] DocumentActivity logged with status "Threat_Detected"
- [ ] Error message clear: "File contains malware and was quarantined. Contact admin."

**Acceptance Criteria - Part 3**: ✅ PASS if threat detection blocks access and alerts admin

---

**Test Flow - Part 4: Offline Queue Processing**

**Steps** (simulate offline environment):

1. Disable network (DevTools → Offline mode or disable WiFi)
2. Upload file: `sample-report.docx` with metadata
3. Observe: "Document uploaded, queued for scan when connection restored"
4. Verify document appears locally with Status = "PENDING_SCAN"
5. Re-enable network
6. Observe: Background sync job picks up queue entry
7. Wait 2-3 seconds for scan to complete
8. Verify document Status updated to "ACTIVE"

**Expected Results - Part 4**:
- [ ] Offline upload succeeds without network (FR-005a)
- [ ] Queue entry persists in UploadQueue table during offline (FR-005b)
- [ ] On reconnection, sync service processes queue (FR-005c)
- [ ] Scan completes on restoration of connectivity (FR-005d)
- [ ] User notified when scan completes (push notification or page refresh)

**Acceptance Criteria - Part 4**: ✅ PASS if offline queue processes on reconnection

---

**Test Flow - Part 5: Scan Timeout & Retry**

**Steps** (if framework supports timeout injection):

1. Configure mock scanner to simulate timeout (>120 seconds)
2. Upload file: `large-file.iso`
3. Wait for timeout
4. Verify UploadQueue entry: RetryCount incremented
5. Check error message: "Scan timeout, will retry"
6. Trigger manual retry via admin UI (or wait for automatic retry)
7. Verify scan completes on retry

**Expected Results - Part 5**:
- [ ] Timeout triggers after 120 seconds (or configured threshold)
- [ ] UploadQueue.QueueStatus = "Error"
- [ ] UploadQueue.RetryCount incremented
- [ ] ErrorMessage populated (e.g., "Timeout after 120s")
- [ ] Manual retry available via admin UI
- [ ] Automatic retry scheduled (configurable backoff)
- [ ] On successful retry, document becomes ACTIVE

**Acceptance Criteria - Part 5**: ✅ PASS if scan retries handle timeouts

---

**Architecture Validation - Database State**

Verify UploadQueue and Document tables after scenarios:

```sql
-- UploadQueue should contain entries for all uploads
SELECT DocumentId, QueueStatus, RetryCount, ErrorMessage FROM UploadQueue
ORDER BY QueuedDate DESC;

-- Document status progression: PENDING_SCAN → ACTIVE (or QUARANTINED)
SELECT DocumentId, Title, Status, UploadDate FROM Document
ORDER BY UploadDate DESC;

-- DocumentActivity logs all scan events
SELECT DocumentId, ActivityType, Details, ActivityDate FROM DocumentActivity
WHERE ActivityType = 'VirusScan'
ORDER BY ActivityDate DESC;
```

**Expected Rows**:
- 5+ entries in UploadQueue (all uploads from test scenarios)
- Most documents have Status = "ACTIVE"
- Threat test document has Status = "QUARANTINED"
- DocumentActivity contains VirusScan entries for each document

**Configuration Validation**

Verify appsettings.Development.json contains:

```json
{
  "VirusScanning": {
    "Enabled": true,
    "ScannerType": "Mock",
    "MaxConcurrentScans": 5,
    "ScanTimeoutSeconds": 120,
    "MockScanDelayMs": 2000
  }
}
```

---

## Scenario 4: Browse My Documents with Filtering (P1)

**Goal**: Verify FR-011, FR-012 - User can browse, sort, filter personal documents

**Test User**: Employee (`ni.kang@contoso.com`)  
**Pre-requisite**: Upload 5+ documents in different categories/projects in previous scenarios

**Steps**:

1. Login as Employee
2. Navigate to Documents → My Documents
3. Verify list displays documents with columns: Title, Category, Upload Date, File Size, Project
4. Click "Upload Date" column header to sort (descending)
5. Verify most recently uploaded documents appear first
6. Click filter icon, select Category = "Reports"
7. Verify only "Reports" category documents displayed
8. Apply Project filter = "Contoso Redesign"
9. Verify filtered results

**Expected Results**:
- [ ] List loads within 2 seconds (SC-006)
- [ ] Column headers display: Title, Category, Upload Date, File Size, Associated Project (FR-011)
- [ ] Sort by Upload Date descending works correctly (FR-012)
- [ ] Category filter works correctly (FR-012)
- [ ] Project filter works correctly (FR-012)
- [ ] Date range filter works (e.g., "Last 30 days") (FR-012)
- [ ] Combined filters work together

**Acceptance Criteria**: ✅ PASS if all sorting/filtering works

---

## Scenario 5: Search Documents (P2)

**Goal**: Verify FR-013, FR-014 - Full-text search finds documents

**Test User**: Employee (`ni.kang@contoso.com`)  
**Pre-requisite**: Multiple documents uploaded with varying titles, descriptions, tags

**Steps**:

1. Login as Employee
2. Navigate to Documents → Search
3. Enter search term: "budget"
4. Submit search
5. Verify results return within 2 seconds
6. Click on search result to view document details
7. Search for uploader name: "Ni Kang"
8. Verify documents uploaded by Ni Kang appear
9. Search for project name: "Contoso Redesign"
10. Verify documents associated with project appear

**Expected Results**:
- [ ] Search returns within 2 seconds (SC-007)
- [ ] Title matches highlighted in results
- [ ] Description matches highlighted
- [ ] Tag matches highlighted
- [ ] "No documents found" displayed if no results
- [ ] Only documents user has access to displayed (FR-014, FR-010 IDOR protection)
- [ ] Uploader name search works
- [ ] Project name search works

**Acceptance Criteria**: ✅ PASS if search functionality works

---

## Scenario 6: Download Document (P1)

**Goal**: Verify FR-015 - User can download document with original filename

**Test User**: Employee (`ni.kang@contoso.com`)

**Steps**:

1. Login as Employee
2. Navigate to My Documents
3. Click "Download" on "sample-document.pdf"
4. Verify browser download
5. Check downloaded file:
   - [ ] Filename is "sample-document.pdf" (original name preserved)
   - [ ] File content matches original
   - [ ] File size matches

**Expected Results**:
- [ ] Download completes within 30 seconds (SC-005)
- [ ] Original filename preserved (FR-015)
- [ ] File integrity maintained (checksum matches if saved before)

**Acceptance Criteria**: ✅ PASS if download works with correct filename

---

## Scenario 7: Document Preview (PDF & Images) (P1)

**Goal**: Verify FR-016 - In-browser preview works for PDF and images

**Test User**: Employee (`ni.kang@contoso.com`)

**Steps**:

1. Login as Employee
2. Navigate to My Documents
3. Click "Preview" on `sample-document.pdf`
4. Verify PDF displays in browser (not download required)
5. Click "Preview" on `sample-image.jpg`
6. Verify image displays in browser

**Expected Results**:
- [ ] PDF preview loads within 3 seconds (SC-008)
- [ ] PDF viewer displays in browser without download
- [ ] Image preview loads within 3 seconds
- [ ] Image displays inline
- [ ] Users cannot preview unsupported formats (e.g., Word docs show "Preview not available")

**Acceptance Criteria**: ✅ PASS if preview works for PDF and images

---

## Scenario 8: Edit Document Metadata (P2)

**Goal**: Verify FR-017 - Document owner can edit metadata

**Test User**: Employee (`ni.kang@contoso.com`) - uploader of "Q3 Budget Report"

**Steps**:

1. Login as Employee
2. Navigate to My Documents
3. Click document "Q3 Budget Report"
4. Click "Edit Metadata"
5. Change:
   - Title from "Q3 Budget Report" → "Q3 Budget Report - FINAL"
   - Category from "Reports" → "Presentations"
   - Add tags: "final, approved"
6. Click "Save"
7. Verify metadata updates in document list
8. Search for original tag to verify it's updated

**Expected Results**:
- [ ] Title updated (FR-017)
- [ ] Category updated (FR-017)
- [ ] Tags updated (FR-017)
- [ ] Document searchable by new metadata
- [ ] Edit history recorded in DocumentActivity

**Acceptance Criteria**: ✅ PASS if metadata updates persist

---

## Scenario 9: Replace Document File (P2)

**Goal**: Verify FR-018 - Document owner can replace file while preserving metadata

**Test User**: Employee (`ni.kang@contoso.com`)

**Steps**:

1. Login as Employee
2. Navigate to My Documents
3. Click document "Q3 Budget Report - FINAL"
4. Click "Replace File"
5. Select new file: `sample-report.docx` (2 MB Word document)
6. Click "Replace"
7. Verify document details:
   - [ ] Title unchanged: "Q3 Budget Report - FINAL"
   - [ ] Category unchanged: "Presentations"
   - [ ] Tags unchanged
   - [ ] Upload Date unchanged (original date preserved)
   - [ ] Uploader unchanged (still "Ni Kang")
   - [ ] File Size updated to ~2 MB
   - [ ] File Type updated to "application/vnd.openxmlformats-officedocument.wordprocessingml.document"

**Expected Results**:
- [ ] File replaced successfully (FR-018)
- [ ] Original upload date preserved (FR-018)
- [ ] Uploader unchanged (FR-018)
- [ ] Metadata unchanged except file-related fields

**Acceptance Criteria**: ✅ PASS if file replacement works

---

## Scenario 10: Delete Document with Confirmation (P2)

**Goal**: Verify FR-019, FR-020 - Document owner can delete with confirmation

**Test User**: Employee (`ni.kang@contoso.com`)

**Steps**:

1. Login as Employee
2. Navigate to My Documents
3. Click document "sample-document.pdf" (temporary test document)
4. Click "Delete"
5. Observe confirmation dialog: "This action cannot be undone"
6. Click "Cancel"
7. Verify document still appears in list
8. Click "Delete" again
9. Click "Confirm Delete"
10. Verify document removed from list
11. Verify document cannot be accessed via search or direct link

**Expected Results**:
- [ ] Confirmation dialog appears (FR-019)
- [ ] Cancel preserves document
- [ ] Confirm delete removes document from "My Documents" (FR-020)
- [ ] Document file deleted from storage (FR-020)
- [ ] Database record deleted or marked deleted (FR-020)
- [ ] Activity log records deletion event (FR-025)

**Acceptance Criteria**: ✅ PASS if deletion works with confirmation

---

## Scenario 11: Role-Based Access Control (Security)

**Goal**: Verify FR-009, FR-010 - RBAC and IDOR protection

**Test User**: Employee (`ni.kang@contoso.com`)  
**Target User**: Project Manager (`camille.nicole@contoso.com`)

**Steps**:

1. Login as Employee
2. Upload document: "Employee Secret.txt" → Category "Personal Files" → No Project Association
3. Logout
4. Login as Project Manager
5. Navigate to Documents → Search
6. Search for "Employee Secret"
7. Verify Project Manager cannot see Employee's personal document

**Expected Results**:
- [ ] Project Manager cannot view Employee's personal files
- [ ] Attempt to access document via direct URL (`/documents/123`) → Access Denied (FR-010)
- [ ] Only authorized users (document owner, project members, or explicitly shared) can access
- [ ] Service layer validates access (Defense in Depth per Constitution)

**Acceptance Criteria**: ✅ PASS if access control enforced

---

## Scenario 12: Share Document (P3)

**Goal**: Verify FR-021, FR-022, FR-023 - Document sharing with notifications

**Test User**: Employee (`ni.kang@contoso.com`) - document owner  
**Recipient**: Team Lead (`floris.kregel@contoso.com`)

**Steps**:

1. Login as Employee
2. Navigate to My Documents
3. Click document "Q3 Budget Report - FINAL"
4. Click "Share"
5. Enter recipient name: "Floris Kregel"
6. Click "Share"
7. Verify success message
8. Logout
9. Login as Team Lead
10. Check notifications → Verify "Document shared with you" notification (FR-022)
11. Navigate to Documents → "Shared with Me"
12. Verify "Q3 Budget Report - FINAL" appears (FR-023)
13. Click document, download/preview to verify access

**Expected Results**:
- [ ] Share completes successfully (FR-021)
- [ ] Recipient receives in-app notification (FR-022)
- [ ] Shared document appears in recipient's "Shared with Me" section (FR-023)
- [ ] Recipient can view/download/preview (FR-023)
- [ ] DocumentShare record created with sharing audit trail
- [ ] DocumentActivity logs share event

**Acceptance Criteria**: ✅ PASS if sharing works with notifications

---

## Scenario 13: Offline Upload Queue (P1 - Offline Support)

**Goal**: Verify FR-005a-d - Offline upload queue functionality

**Test Environment**: Simulate offline by disabling network

**Test User**: Employee (`ni.kang@contoso.com`)

**Steps**:

1. Login as Employee
2. Disconnect network (disable WiFi/Ethernet or use browser DevTools → Offline mode)
3. Navigate to Documents → Upload Document
4. Select file: `sample-report.docx`
5. Enter metadata as in Scenario 1
6. Click "Upload"
7. Observe status: "Queued for upload" with spinning indicator (FR-005b)
8. Verify document appears in "Queued Uploads" section showing status
9. Reconnect network
10. Observe automatic sync attempt (FR-005c)
11. Verify status changes to "Uploaded"
12. Verify document now appears in "My Documents" with correct metadata

**Expected Results**:
- [ ] Upload queued when offline (FR-005a)
- [ ] "Queued for upload" status displayed (FR-005b)
- [ ] Automatic sync when connection restored (FR-005c)
- [ ] Manual retry option available if sync fails (FR-005d)
- [ ] Queue status visible in UI
- [ ] Failed uploads show error message with retry button

**Acceptance Criteria**: ✅ PASS if offline queueing works

---

## Scenario 14: Dashboard Recent Documents Widget (P1 Integration)

**Goal**: Verify FR-024 - Dashboard shows recent documents

**Test User**: Employee (`ni.kang@contoso.com`)  
**Pre-requisite**: Multiple documents uploaded in previous scenarios

**Steps**:

1. Login as Employee
2. Navigate to Dashboard (home page)
3. Find "Recent Documents" widget
4. Verify displays last 5 documents uploaded by user

**Expected Results**:
- [ ] "Recent Documents" widget displays on dashboard (FR-024)
- [ ] Shows last 5 documents chronologically (FR-024)
- [ ] Each entry shows: Title, Upload Date, Size (FR-024)
- [ ] Clicking entry navigates to document detail

**Acceptance Criteria**: ✅ PASS if widget displays

---

## Scenario 15: Audit Logging (Compliance)

**Goal**: Verify FR-025 - Document activities logged for audit

**Test User**: Admin (`admin@contoso.com`)

**Steps**:

1. Perform various document operations in previous scenarios (upload, download, delete, share)
2. Login as Admin
3. Navigate to Admin → Audit Log (if available in UI)
4. Filter by: Document Activity
5. Verify entries for Upload, Download, Delete, Share, MetadataChange operations

**Expected Results**:
- [ ] Upload operations logged (FR-025)
- [ ] Download operations logged (FR-025)
- [ ] Delete operations logged (FR-025)
- [ ] Share operations logged (FR-025)
- [ ] Metadata change operations logged (FR-025)
- [ ] Activity includes: Who, What, When, Document ID
- [ ] Logs help with compliance and troubleshooting

**Acceptance Criteria**: ✅ PASS if audit trail complete

---

## Data Model Validation

**Goal**: Verify database schema matches design

**Steps**:

1. Connect to database
2. Verify tables exist:
   - [ ] Document table with columns: DocumentId, Title, Category, FilePath, FileSize, FileType, UploadDate, UploadedById, AssociatedProjectId, Tags, Status
   - [ ] DocumentShare table with: ShareId, DocumentId, SharedWithUserId, SharedByUserId, SharedDate
   - [ ] DocumentActivity table with: ActivityId, DocumentId, UserId, ActivityType, ActivityDate, Details
   - [ ] UploadQueue table with: QueueId, DocumentId, UserId, DocumentMetadata, FileData, QueueStatus, RetryCount, ErrorMessage, QueuedDate, SyncedDate

3. Verify relationships:
   - [ ] Document.UploadedById → User.UserId (FK)
   - [ ] Document.AssociatedProjectId → Project.ProjectId (FK)
   - [ ] DocumentShare.DocumentId → Document.DocumentId (FK)
   - [ ] DocumentShare.SharedWithUserId → User.UserId (FK)
   - [ ] DocumentShare.SharedByUserId → User.UserId (FK)
   - [ ] DocumentActivity.DocumentId → Document.DocumentId (FK)
   - [ ] DocumentActivity.UserId → User.UserId (FK)
   - [ ] UploadQueue.DocumentId → Document.DocumentId (FK, nullable initially)
   - [ ] UploadQueue.UserId → User.UserId (FK)

**Acceptance Criteria**: ✅ PASS if schema matches design

---

## Performance Validation

**Goal**: Verify performance meets success criteria

| Metric | Target | Test Method |
|--------|--------|-------------|
| Upload (<30s for 25MB) | SC-005 | Upload 25MB file, measure time |
| List load (<2s for 500 docs) | SC-006 | Navigate to My Documents with 500+ docs, measure load time |
| Search response (<2s) | SC-007 | Execute search with 500+ docs, measure response time |
| Preview load (<3s) | SC-008 | Click preview on PDF/image, measure load time |

**Acceptance Criteria**: ✅ PASS if all metrics met

---

## Validation Summary

| Scenario | Feature | Status |
|----------|---------|--------|
| 1 | Upload with metadata (P1) | [ ] PASS |
| 2 | Reject unsupported files (Security) | [ ] PASS |
| 3 | Reject oversized files (Validation) | [ ] PASS |
| 4 | Browse with filtering (P1) | [ ] PASS |
| 5 | Search documents (P2) | [ ] PASS |
| 6 | Download document (P1) | [ ] PASS |
| 7 | Preview PDF/images (P1) | [ ] PASS |
| 8 | Edit metadata (P2) | [ ] PASS |
| 9 | Replace file (P2) | [ ] PASS |
| 10 | Delete with confirmation (P2) | [ ] PASS |
| 11 | RBAC & IDOR protection (Security) | [ ] PASS |
| 12 | Share with notifications (P3) | [ ] PASS |
| 13 | Offline upload queue (P1 - Offline) | [ ] PASS |
| 14 | Dashboard widget (P1 Integration) | [ ] PASS |
| 15 | Audit logging (Compliance) | [ ] PASS |
| Data | Schema validation | [ ] PASS |
| Performance | All SCs met | [ ] PASS |

**Overall Status**: [ ] READY FOR RELEASE (all scenarios PASS)

---

**Quickstart Guide Complete**: Use these scenarios to validate feature implementation
