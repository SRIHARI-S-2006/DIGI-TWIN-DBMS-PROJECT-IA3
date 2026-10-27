# UniSphere Enterprise University ERP Backend

A production-oriented **Node.js + TypeScript + Express + MongoDB** backend for multi-university ERP deployments. The repository implements tenant isolation, campus/department/program/course scopes, JWT authentication, rotating refresh tokens, role templates, RBAC + ABAC authorization, maker-checker approvals, state-machine workflows, audit history, Swagger/OpenAPI, Docker and seed data.

## Coverage

- **31 enterprise modules**
- **171 working ERP resources**
- **377 workflow actions**
- **20 seeded institutional roles**, including the 19 principal university portals plus internal audit
- Explicit Swagger schemas and paths for every registered resource
- Generic but resource-aware CRUD behavior, validation, permissions, pagination, filtering, soft deletion and workflow history

The complete resource list is in [`docs/MODULE_CATALOG.md`](docs/MODULE_CATALOG.md). Access rules are explained in [`docs/ACCESS_CONTROL.md`](docs/ACCESS_CONTROL.md). See [`docs/WORKFLOW_EXAMPLES.md`](docs/WORKFLOW_EXAMPLES.md) for end-to-end flows and [`docs/WINDOWS_SETUP.md`](docs/WINDOWS_SETUP.md) for PowerShell setup.

## Main domains

1. **Platform & Subscription** — Platform tenancy, plans, feature controls and integration identities.
2. **Authentication & Security** — Sessions, MFA, incidents, access reviews and privileged-access controls.
3. **Users, Roles & Permissions** — Identity lifecycle, role assignments, delegations and access governance.
4. **University & Campus Structure** — Campuses, schools, departments, programs, periods and facilities.
5. **Admissions** — Admission cycles, applications, verification, merit, counselling and offers.
6. **Student Information System** — Student master data, guardians, documents, histories and clearances.
7. **Academic Governance** — Regulations, calendars, academic boards and committee decisions.
8. **Curriculum & Syllabus** — Curriculum versions, course mapping, syllabus and learning outcomes.
9. **Courses & Offerings** — Course catalogue, term offerings, sections and faculty allocation.
10. **Enrollment & Registration** — Student enrollment, course registration, transfers, add/drop and withdrawal.
11. **Timetable & Scheduling** — Timetables, slots, rooms, conflict checks and faculty substitution.
12. **Attendance & Leave** — Class sessions, student attendance, correction workflows and student leave.
13. **Learning Management System** — Course spaces, content, assignments, submissions, quizzes and gradebook.
14. **Examinations** — Exam cycles, schedules, rooms, admit cards, invigilation and secure papers.
15. **Evaluation, Grading & Results** — Mark entry, moderation, result publication, corrections and transcripts.
16. **Student Fees & Accounts** — Fee structures, invoices, payments, refunds, fines and clearances.
17. **Finance & Accounting** — Ledgers, journals, budgets, expenses, vendors, reconciliation and period closing.
18. **Scholarships & Concessions** — Scholarship schemes, applications, awards and disbursement.
19. **Human Resources** — Employees, positions, recruitment, appointments, transfers and separation.
20. **Payroll** — Salary structures, payroll runs, payslips, deductions and tax records.
21. **Leave, Attendance & Performance** — Employee time, leave, appraisal and workload management.
22. **Library** — Catalogue, copies, membership, circulation, fines and acquisitions.
23. **Hostel** — Hostels, rooms, beds, applications, allocations, visitors and incidents.
24. **Transport** — Vehicles, routes, stops, assignments, passes and maintenance.
25. **Procurement & Inventory** — Purchase requests, quotations, orders, receipts, stock and movement.
26. **Assets & Facilities** — Assets, maintenance, work orders, bookings and inspections.
27. **Research & Grants** — Proposals, projects, grants, ethics, publications, patents, scholars and labs.
28. **Placement & Career Services** — Employers, drives, jobs, profiles, applications, interviews, offers and training.
29. **Alumni** — Alumni profiles, chapters, events, donations and transcript requests.
30. **Communication, Grievance & Support** — Announcements, notifications, grievances, support, surveys and discipline.
31. **Analytics, Audit, Settings & Integrations** — Institution settings, integrations, reports, exports, privacy and retention.

## Local setup

### Prerequisites

- Node.js 22+
- MongoDB 7+
- npm

### Start with npm

```bash
cp .env.example .env
npm install
npm run dev
```

Swagger UI: `http://localhost:4000/docs`  
OpenAPI JSON: `http://localhost:4000/openapi.json`  
Health: `http://localhost:4000/health`

For guided setup, run `./scripts/setup.sh` on Linux/macOS or `./scripts/setup.ps1` from PowerShell.

### Start with Docker

```bash
cp .env.example .env
docker compose up --build
```

### Seed a demonstration university

```bash
npm run seed
```

Demo credentials:

```text
University slug: demo-university
Email: admin@demo-university.edu
Password: Admin@12345
```

Change all demonstration credentials and secrets before any shared or production deployment.

## Provision the first university through Swagger

Call `POST /api/v1/auth/bootstrap` with the value configured in `BOOTSTRAP_KEY`:

```json
{
  "bootstrapKey": "change-this-bootstrap-key",
  "university": {
    "name": "Enterprise University",
    "slug": "enterprise-university",
    "code": "EU",
    "plan": "ENTERPRISE"
  },
  "admin": {
    "email": "admin@enterprise.edu",
    "password": "StrongPassword@123",
    "firstName": "University",
    "lastName": "Administrator"
  }
}
```

Then use the returned access token with Swagger's **Authorize** button.

## Resource API pattern

Every resource receives these endpoints:

```text
GET    /api/v1/erp/{module}/{resource}
POST   /api/v1/erp/{module}/{resource}
GET    /api/v1/erp/{module}/{resource}/{id}
PATCH  /api/v1/erp/{module}/{resource}/{id}
DELETE /api/v1/erp/{module}/{resource}/{id}
POST   /api/v1/erp/{module}/{resource}/{id}/actions/{action}
```

Example admission application:

```json
{
  "status": "DRAFT",
  "campusId": "64f000000000000000000001",
  "programId": "64f000000000000000000010",
  "ownerUserId": "64f000000000000000000100",
  "data": {
    "applicationNo": "APP-2026-0001",
    "applicantUserId": "64f000000000000000000100",
    "cycleId": "64f000000000000000000200",
    "programId": "64f000000000000000000010",
    "campusId": "64f000000000000000000001",
    "personalDetails": { "fullName": "Example Applicant" },
    "academicDetails": { "percentage": 88.4 }
  }
}
```

Submit it through:

```text
POST /api/v1/erp/admissions/applications/{id}/actions/submit
```

## Authorization model

Permissions use `module.resource.action`, such as:

```text
admissions.applications.verify
attendance.attendance-sessions.lock
results.results.publish
student-finance.refunds.approve
payroll.payroll-runs.approve
```

A role assignment also defines its access scope:

```text
PLATFORM -> UNIVERSITY -> CAMPUS -> SCHOOL -> DEPARTMENT -> PROGRAM -> COURSE
                                                       -> SELF / DEPENDENT
```

The backend applies scope restrictions to list queries and checks them again for single-record operations. Explicit denies override wildcard allows. Sensitive fields require `read-sensitive` permission.

## Maker-checker approvals

High-risk actions create an approval request rather than immediately changing the final state. Approvers use:

```text
GET  /api/v1/approvals
POST /api/v1/approvals/{approvalId}/decision
```

The requester cannot approve their own request. Some workflows require two independent approvals, including result publication, payroll, purchase orders, research grants and accounting-period operations.

## Core enterprise controls

- Tenant ID comes from the verified token, never solely from the request body.
- University status is verified on every authenticated request.
- Role assignments support effective dates and temporary delegation.
- Locked workflow states reject normal updates unless the role has an override permission.
- Delete operations are soft deletes.
- Passwords and refresh tokens are hashed.
- Refresh-token rotation includes reuse detection and family revocation.
- Failed login attempts trigger temporary account lockout.
- Audit entries include tenant, user, session, request ID, action and before/after snapshots.
- Secret and highly restricted fields are redacted without `read-sensitive` permission.
- API responses include correlation IDs for troubleshooting.

## Project structure

```text
src/
├── common/          # errors, middleware, authorization utilities
├── config/          # environment, database and logging
├── docs/            # generated OpenAPI document
├── modules/
│   ├── auth/        # login, refresh rotation and bootstrap
│   ├── iam/         # users, roles and scoped assignments
│   ├── tenants/     # university tenants
│   ├── erp/         # 31-module resource registry and engine
│   ├── approvals/   # maker-checker requests and decisions
│   ├── audit/       # immutable mutation history
│   └── dashboard/   # summary analytics
├── routes/
└── scripts/
```

## Verification commands

```bash
npm run build
npm test
npm run docs:export
```

## Production recommendations

Before production, add your institution's payment gateway, object storage, email/SMS provider, background job queue and observability stack. Run MongoDB as a replica set, use a secrets manager, enable TLS, apply field-level encryption for highly restricted data, introduce database backups and disaster recovery, and place the API behind a managed gateway or reverse proxy.

This repository is a complete backend foundation. Institution-specific statutory forms, accounting rules, grading regulations, biometric protocols and payment reconciliation logic should be configured or extended according to the target university's policies.
