<!-- Sync Impact Report: Constitution v1.0.0 Created
- Initial version establishment for ContosoDashboard project
- Ratified: 2026-08-16
- Core principles: Spec-Driven Development, Security-First, Role-Based Design, Service Layer Architecture, Code Quality
-->

# ContosoDashboard Constitution

## Core Principles

### I. Spec-Driven Development (SDD)
All features MUST be specified using the Spec Kit workflow before implementation begins. Specifications MUST include: user stories, acceptance criteria, design artifacts, and task breakdown. Code implementation follows approved specifications; deviations MUST be documented and re-approved. Features without specifications MUST NOT be implemented.

**Rationale**: Ensures stakeholder alignment, prevents scope creep, and maintains consistent quality across the multi-feature dashboard application.

### II. Security-First by Design
Every feature MUST implement security at all layers: authentication, authorization, data validation, and service-level checks. Role-based access control (RBAC) MUST be enforced via `[Authorize]` attributes and service layer validation. User data isolation MUST be verified for each feature (IDOR protection mandatory). Security headers and defensive practices MUST be included in all implementations.

**Rationale**: This training application demonstrates security best practices; learners rely on this model for production understanding.

### III. Role-Based Access & User Isolation
All protected pages and services MUST enforce role-based authorization through claims-based identity. Features MUST verify user's role and project/team membership before granting data access. Cross-user data access MUST be prevented at both page and service layers (defense in depth). Authorization logic MUST be centralized in service layer to prevent duplication.

**Rationale**: Contoso's dashboard serves multiple roles (Admin, Project Manager, Team Lead, Employee); user isolation is non-negotiable for data integrity and trust.

### IV. Service Layer Architecture
Business logic MUST reside exclusively in service classes (located in `/Services/`). Pages and components MUST NOT contain business logic. Services MUST be dependency-injected and unit-testable. Data access MUST occur only through services, with role and user context validated before returning results.

**Rationale**: Maintains clean separation of concerns, improves testability, and ensures consistent security enforcement across all UI touchpoints.

### V. Code Quality & Documentation
All new code MUST include inline comments for complex logic. Public methods and classes MUST include XML documentation. Naming conventions MUST be clear and consistent (PascalCase for classes/methods, camelCase for variables). Unused code and commented-out code MUST NOT be committed. Code reviews MUST verify compliance before merging.

**Rationale**: Training code serves as a learning model; clarity and documentation are essential for knowledge transfer.

## Technical Requirements

**Technology Stack**:
- Framework: .NET 8 / Blazor Server (fixed)
- Database: Entity Framework Core with SQLite (training) or SQL Server (configurable)
- Authentication: Claims-based with cookie middleware (mock in training, replaceable in production)
- UI Framework: Razor components with CSS Grid/Flexbox

**File Organization**:
- Data models: `/Models/`
- Database context: `/Data/`
- Business logic services: `/Services/`
- UI components & pages: `/Pages/` and `/Shared/`
- Static assets: `/wwwroot/`

## Development Workflow

**Feature Implementation Cycle**:
1. Create feature specification (Spec Kit workflow) with stakeholder approval
2. Design data models and database schema per specification
3. Implement service layer with role-based authorization
4. Create Razor components/pages that consume services
5. Test across all user roles to verify access control
6. Code review with security and architecture focus
7. Merge and document in project wiki

**Code Review Gates**:
- All PRs MUST be reviewed before merging
- Reviews MUST verify: specification compliance, security implementation, code quality, test coverage (if applicable)
- Breaking changes MUST be documented and versioned

## Governance

This Constitution supersedes all conflicting practices and represents the agreed-upon engineering standards for ContosoDashboard. Amendments MUST be documented with rationale, reviewed by the team, and approved before enforcement.

All PRs and code reviews MUST verify compliance with these principles. Complexity introduced by security or access control requirements MUST be justified and documented. Deviations from the Constitution require written approval and amendment tracking.

**Amendment Process**: Proposed changes to this Constitution MUST include: current principle/rule, proposed change, rationale, impact assessment, and migration plan (if applicable).

---

**Version**: 1.0.0 | **Ratified**: 2026-08-16 | **Last Amended**: 2026-08-16
