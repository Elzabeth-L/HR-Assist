# HR Assist — SAP SuccessFactors API Scope (OData v2)

> **Purpose:** Define the **minimum** set of SAP SuccessFactors APIs the HR Assist agent needs, with the permissions, authentication and safeguards for each, so the connector schema stays small, access stays narrow and sensitive data stays out.
> **Source:** *SAP SuccessFactors API Reference Guide*, Document Version 2H 2026 (2026-10-02). Page numbers are given as **(p. N)**.
> **Status:** Verified against the API reference guide. **Scope is provisional until HR requirements are received.** Entity availability still depends on the modules and Provisioning settings enabled in the EBSCO tenant (p. 622).

---

## Contents

- [1. Key Findings from the Deep-Dive](#1-key-findings-from-the-deep-dive)
- [2. How to Read the API Catalogue](#2-how-to-read-the-api-catalogue)
- [3. Required APIs](#3-required-apis)
- [4. Optional APIs](#4-optional-apis)
- [5. Not Required APIs](#5-not-required-apis)
- [6. Sensitive APIs — Flagged](#6-sensitive-apis--flagged)
- [7. Authentication](#7-authentication)
- [8. Permissions (Role-Based Permissions)](#8-permissions-role-based-permissions)
- [9. Connector Design](#9-connector-design)
- [10. Limits, Logging \& Operations](#10-limits-logging--operations)
- [11. Approvals](#11-approvals)
- [12. Open Questions for EBSCO](#12-open-questions-for-ebsco)

---

## 1. Key Findings from the Deep-Dive

| # | Finding | Why it matters | Ref |
|---|---|---|---|
| F1 | Employee Central APIs run in **User Mode** (role-based permissions apply) or **Admin Mode** (full access, **overrides RBP**). The *Employee Central HRIS OData API (read-only)* permission is Admin Mode and lets the caller query **all** HRIS objects. SAP recommends not granting it to end users. | HR Assist must use **User Mode** permissions, so each employee only sees their own data. **Do not grant** the Admin Mode API permissions. | p. 621, 624–625 |
| F2 | The `User` entity contains highly sensitive fields (e.g. `ssn`, `salary`, `dateOfBirth`, `ethnicity`, `citizenship`). The **Employee Export** permission returns **all fields** and **overrides field-level permissions**. | **Never grant Employee Export.** Use *User Search* plus field-level permissions, and always send an explicit `$select` of safe fields. | p. 454–456 |
| F3 | A role's **Export** permission without a target population lets the caller query **all records** with no row-level check. | Every HR Assist permission must have target population **Self**. | p. 87 |
| F4 | OAuth tokens are issued for the SuccessFactors **user ID in the SAML assertion's `NameID`**. | The connector can request a token **as the asking employee**, so SuccessFactors enforces "self only" itself. | p. 26 |
| F5 | `EmpTimeAccountBalance` requires **both** `userId` and `timeAccountType` in `$filter` (codes are case-sensitive). It returns balances **calculated from accrual runs already performed** (no simulation), one record per valid time account. | The connector must first look up the employee's account types. Answers should say "as of" the date used. | p. 794–796 |
| F6 | Creating an `EmployeeTime` (leave request) only triggers the approval workflow if `workflowConfirmed=true` is sent **and** the caller does **not** have *Admin access to MDF OData API*. | Without this, a leave request could **bypass manager approval**. Mandatory control for Phase 2. | p. 1347 |
| F7 | `WfRequest` needs the admin permission *Manage Workflow Requests*. `MyPendingWorkflow` returns workflows for the **current login user** only. | Use `MyPendingWorkflow` for "where's my request?" and avoid the admin entity. | p. 1629–1630, 1659 |
| F8 | `FOCompany`, `FODepartment` and `FODivision` are **migrated (MDF) Foundation Objects**, while `FOLocation` is a classic Foundation Object. They need different permissions. | Grant *MDF Foundation Objects* (user mode) and *Manage Foundation Objects Types* (view), not the Admin Mode FO APIs. | p. 625–626, 883, 933–943 |
| F9 | **HTTP Basic Authentication is deprecated and will be retired.** SAP recommends **OIDC** (via SAP Identity Authentication) or **OAuth 2.0 with SAML bearer assertion**. Don't use `/oauth/idp`. | Design for OAuth/OIDC from day one. | p. 13, 26, 30 |
| F10 | API **audit logs can contain payloads with sensitive data**; retention is up to 90 days. | Restrict audit-log access; don't enable "all payloads" in production. | p. 53–56 |
| F11 | Rate limits are **shared across all integrations in the tenant** (e.g. tenant quota of 600 requests/min, `$batch` 25/min); exceeding them returns **429 + Retry-After**. | Cache reference data; respect Retry-After; avoid chatty calls. | p. 40–49, 262 |

---

## 2. How to Read the API Catalogue

APIs are grouped by **what HR Assist needs to do**, then split into three tiers:

| Tier | Meaning |
|---|---|
| **Required** ([§3](#3-required-apis)) | Needed for the core use case: read-only, personal HR answers for the asking employee (leave balances, leave types, holidays, leave request status, job context, manager/HR contact). |
| **Optional** ([§4](#4-optional-apis)) | Only needed if HR asks for the feature (e.g. applying for leave, workflow tracking, manager approvals) or to enrich answers. |
| **Not required** ([§5](#5-not-required-apis)) | Out of scope, admin-only, deprecated or prohibited. Never connect these. |

Every API carries a **sensitivity flag**:

| Flag | Level | Meaning |
|---|---|---|
| 🟢 | **Low** | Reference or configuration data; no personal information |
| 🟡 | **Medium** | The employee's own, non-sensitive work data (job, leave balance, holidays) |
| 🟠 | **High** | Contains or can expose sensitive fields. Use only with a strict `$select`, a field allowlist and self-only enforcement (see [§6](#6-sensitive-apis--flagged)). |
| 🔴 | **Prohibited** | Highly sensitive personal data. Never connected (see [§6](#6-sensitive-apis--flagged)). |

> `GET` = read; `POST` = create or run a function. All OData paths are relative to `https://<api-server>/odata/v2/`.

---

## 3. Required APIs

### 3.1 Authentication & Setup

| API | Method | What it does | Used for | Flag | Ref |
|---|---|---|---|---|---|
| `/oauth/token` | POST | Exchanges a signed SAML assertion (the employee's `userId` + API key) for an access token valid 24 h | Every call: get a token **as the asking employee** | 🟠 | p. 28 |
| `$metadata` | GET | Describes the tenant's entities and fields | **Design time only:** build and validate the connector schema | 🟢 | p. 88 ff. |

### 3.2 Identify the Employee
*"Who is asking?" Links the Teams/Okta login to a SuccessFactors user ID.*

| API | Method | What it does | Used for | Flag | Ref |
|---|---|---|---|---|---|
| `User` | GET | Employment record with organisational and some personal info. **Also holds `ssn`, `salary`, `dateOfBirth`, `ethnicity`, `citizenship`.** | Resolve email → `userId`; display name. Strict `$select`: `userId, username, email, firstName, lastName, displayName, status, department, location, timeZone` | 🟠 | p. 453–456 |

### 3.3 Employment & Job Context
*Decides which version of a policy applies to the employee.*

| API | Method | What it does | Used for | Flag | Ref |
|---|---|---|---|---|---|
| `EmpJob` | GET | Effective-dated job information | Company, country, location, employment type, manager, **holiday calendar**, **time type profile**, work schedule | 🟡 | p. 754 |
| `EmpEmployment` | GET | Employment dates | "When did I join?" and tenure-based entitlements (`startDate`, `originalStartDate`, `seniorityDate`, `serviceDate`) | 🟡 | p. 735 |
| `FOCompany` | GET | Legal entity (MDF Foundation Object) | Selecting the country-specific policy | 🟢 | p. 933 |
| `FOLocation` | GET | Work location (Foundation Object) | Location-specific rules | 🟢 | p. 883 |

### 3.4 Check Leave Balances
*"How many leave days do I have left?"*

| API | Method | What it does | Used for | Flag | Ref |
|---|---|---|---|---|---|
| `TimeAccount` | GET | The employee's time accounts and their validity / booking periods | Find the employee's account types (needed by the balance call); explain expiry | 🟡 | p. 1431 |
| `EmpTimeAccountBalance` | GET | Calculated balance per valid time account as of a date. **Requires** `userId` + `timeAccountType` filters. | The leave-balance answer | 🟡 | p. 794–796 |
| `TimeAccountType` | GET | Template and name of each time account type | Readable names (e.g. "Casual Leave 2026") | 🟢 | p. 1453 |

### 3.5 Check Eligible Leave Types
*"What kinds of leave can I take?"*

| API | Method | What it does | Used for | Flag | Ref |
|---|---|---|---|---|---|
| `TimeTypeProfile` | GET | The employee's leave-type profile (from job info) | Find which leave types apply | 🟢 | p. 1516 |
| `AvailableTimeType` | GET | Leave types allowed within a profile | List only the leave types the employee may take | 🟢 | p. 1343 |
| `TimeType` | GET | Leave types configured in the tenant | Map "casual leave" to the right code; unit (days/hours) | 🟢 | p. 1491 |

### 3.6 Check Holidays
*"Is next Monday a holiday at my location?"*

| API | Method | What it does | Used for | Flag | Ref |
|---|---|---|---|---|---|
| `HolidayCalendar` | GET | Holiday calendar per country/location | Pick the employee's calendar (`EmpJob.holidayCalendarCode`) | 🟢 | p. 1420 |
| `HolidayAssignment` | GET | Holiday dates in a calendar, with class FULL / HALF / NONE | The holiday answer | 🟢 | p. 1412 |
| `Holiday` | GET | Holiday definitions | Holiday names | 🟢 | p. 1389 |

### 3.7 Check My Leave Requests
*"Is my leave on the 22nd approved?"*

| API | Method | What it does | Used for | Flag | Ref |
|---|---|---|---|---|---|
| `EmployeeTime` | GET | The employee's absences: type, dates, quantity and `approvalStatus`. **Also has `comment`, attachment and leave-of-absence (`loa*`) fields.** | Leave request status. Strict `$select`: `externalCode, userId, timeType, startDate, endDate, quantityInDays, quantityInHours, approvalStatus` | 🟠 | p. 1347 |

### 3.8 Find My Manager & HR Contact
*"Who is my manager / HR partner?" Also used to route escalations.*

| API | Method | What it does | Used for | Flag | Ref |
|---|---|---|---|---|---|
| `User` → `manager`, `hr` (navigation) | GET | The employee's manager and HR representative | Name and business email only, via `$expand` + `$select` | 🟡 | p. 453 ff. |

---

## 4. Optional APIs

### 4.1 Apply for Leave *(only if HR wants actions)*

| API | Method | What it does | Used for | Flag | Ref |
|---|---|---|---|---|---|
| `EmployeeTime` | POST (insert) | Creates an absence request. **Must** send `workflowConfirmed=true`, and the caller must **not** hold *Admin access to MDF OData API*, otherwise the approval workflow can be bypassed. | "Apply 2 days casual leave from the 22nd", after explicit employee confirmation (*Always Ask*) | 🟠 | p. 1347 |
| `AvailableTimeType` | GET | See §3.5 | Validate the leave type before submitting | 🟢 | p. 1343 |

### 4.2 Withdraw or Cancel a Leave Request *(only if HR wants actions)*

| API | Method | What it does | Used for | Flag | Ref |
|---|---|---|---|---|---|
| `WorkflowAllowedActionList` | GET | Shows which actions (withdraw, comment, etc.) the user may take on a workflow step | Check that the employee is allowed to withdraw | 🟢 | p. 1646 |
| `withdrawWfRequest` | POST | Function import that withdraws a pending workflow request, if allowed | "Withdraw my pending leave request" | 🟡 | p. 1666 |
| `EmployeeTime` | Merge / Delete | Updates or deletes an absence | Cancelling an **already-approved** leave. ⚠ **Verify** how EBSCO configures cancellation (a cancellation workflow may apply: `cancellationWorkflowRequestId`). | 🟠 | p. 1347 |

### 4.3 Track My Requests & Tasks

| API | Method | What it does | Used for | Flag | Ref |
|---|---|---|---|---|---|
| `MyPendingWorkflow` | GET | Workflows where the **current login user** is a contributor (pending / sent back) or CC (completed) | "Where is my request stuck?" | 🟡 | p. 1659 |
| `TodoEntryV2` | GET | The user's to-do items (category, status, due date, link) | "What HR tasks do I have pending?" | 🟡 | p. 404 |
| `TodoEntryV2` → `wfRequestNav/wfRequestUINav` | GET | Workflow details as shown on the Workflow Details page | A readable summary of a pending request | 🟡 | p. 1642 |

### 4.4 Manager Approvals *(only if managers are in scope)*

| API | Method | What it does | Used for | Flag | Ref |
|---|---|---|---|---|---|
| `approveWfRequest` | POST | Approves a workflow step the user is authorised to approve | "Approve Priya's leave request" | 🟠 | p. 1661 |
| `sendbackWfRequest`, `commentWfRequest` | POST | Sends back or comments on a workflow request | Manager follow-ups | 🟡 | p. 1662–1664 |
| `rejectWfRequest` | POST | Rejects a workflow request. **Needs** the admin permission *Manage Workflow Requests*. | Not recommended; reject in SuccessFactors directly | 🟠 | p. 1663 |

> Manager actions read and act on **other employees'** requests, so they need a separate design and security review.

### 4.5 Richer Leave Details

| API | Method | What it does | Used for | Flag | Ref |
|---|---|---|---|---|---|
| `TimeAccountDetail` | GET | Bookings on a time account (accruals, leave taken, manual adjustments) | "Why did my balance change?" | 🟡 | p. 1438 |
| `EmployeeTimeCalendar` | GET | Day-by-day breakdown of an absence (days and hours) | "How many days did my leave actually use?" | 🟡 | p. 1378 |
| `TimeAccountSummary` | GET | Accrual and entitlement summary per time account | Detailed entitlement explanations | 🟡 | p. 1570 |
| `WorkSchedule` | GET | The employee's working pattern | "Do I work Saturdays?"; half-day logic | 🟢 | p. 1519 |

### 4.6 Organisation & Routing Details

| API | Method | What it does | Used for | Flag | Ref |
|---|---|---|---|---|---|
| `FODepartment`, `FODivision` | GET | Department and division names (MDF FO) | Department-specific guidance; escalation routing | 🟢 | p. 939, 943 |
| `EmpJobRelationships` | GET | Other relationships (e.g. HR business partner, matrix manager) | Route escalations to the right HR person, if held here | 🟡 | p. 772 |
| `PerEmail` | GET | Email addresses (key: `personIdExternal` + `emailType`) | Fallback if the login email isn't `User.email`. **Business email only.** | 🟡 | p. 1257 |

### 4.7 Supporting Utilities

| API | Method | What it does | Used for | Flag | Ref |
|---|---|---|---|---|---|
| `/oauth/validate` | GET | Checks whether an access token is still valid | Token cache housekeeping | 🟢 | p. 29 |
| MDF picklists (`PickListValueV2`) | GET | Labels for coded values | Readable labels for codes | 🟢 | p. 325 |
| `$batch` | POST | Bundles several requests into one call (≤ 180) | Fewer calls when building a profile summary | 🟢 | p. 175, 262 |

---

## 5. Not Required APIs

### 5.1 Prohibited — Sensitive Personal Data 🔴
Never connected and never in the permission role. [§6](#6-sensitive-apis--flagged) explains why.

| API | Contains | Ref |
|---|---|---|
| `PerNationalId`, `PerNationalIdWithValidityPeriod` | National IDs (e.g. SSN, Aadhaar, PAN) | p. 1271–1274 |
| `EmpWorkPermit` | Passport / visa / work permit details | p. 781 |
| `PaymentInformationDetailV3*`, `Bank` | Bank account details | p. 1187 ff. |
| `EmpCompensation`, `EmpPayCompRecurring`, `EmpPayCompNonRecurring` | Salary and pay components | p. 720, 785–789 |
| Payroll, Deductions and Advances entities | Payslips, deductions, salary advances | p. 667, 698, 1217 |
| `PerPersonal`, `PerGlobalInfo<Country>`, `PerBiographicalInfoLoc<Country>` | Date of birth, gender, marital status, nationality, local personal info | p. 1254, 1265, 1283 |
| `PerAddressDEFLT`, `PerPhone`, `PerEmergencyContacts`, `PerPersonRelationship`, `PerSocialAccount` | Home address, personal phones, dependants, social accounts | p. 1250–1301 |
| `EmpBeneficiary`, Global Benefits enrolment entities | Beneficiaries, dependants, insurance enrolment | p. 718, 990 ff. |
| `EmpEmploymentTermination`, `PersonEmpTerminationInfo` | Termination details | p. 742, 1260 |
| `Photo` | Employee photos | p. 301 |

### 5.2 Admin-Only or Over-Privileged

| API | Why not | Ref |
|---|---|---|
| `WfRequest`, `WfRequestStep`, `WfRequestParticipator`, `WfRequestComments` | Need *Manage Workflow Requests* (admin) and expose **all** workflows. Use `MyPendingWorkflow` instead. | p. 1629–1640 |
| RBP entities (`RBPRole`, `RBPRule`, `DynamicGroup`, etc.) | Permission administration | p. 344–382 |
| `User` / `Emp*` / `Per*` **upsert** operations | Changing employee master data is out of scope | p. 623 |
| `TimeAccountDetail` **create** (manual adjustments), `TimeAccount` create/update, `TimeAccountPayout` | HR admin operations that change balances | p. 1431, 1438, 1567 |
| `HireDateChange`, `convertAssignmentIdExternal`, `checkUserPermissions` | Admin utilities | p. 555–557, 798 |

### 5.3 Out-of-Scope Modules

| Area | Ref |
|---|---|
| Time Recording, Time Sheet | p. 1571, 1604 |
| Performance & Goals, Calibration, Succession & Development, Job Profile Builder | p. 569, 1752, 2526, 3032 |
| Recruiting, Onboarding, Learning | p. 1819, 2419, 2758 |
| Position Management, Service Center, Execution Manager, Employee Profile (badges, background) | p. 1311, 1325, 1705, 1725 |

### 5.4 Deprecated or Unsuitable Access Methods

| Item | Why not | Ref |
|---|---|---|
| HTTP Basic Authentication | Deprecated and being retired | p. 13, 30 |
| `/oauth/idp` (assertion generation) | Insecure and deprecated | p. 26 |
| SFAPI / SOAP APIs | Deprecated for new integrations | p. 7, 620 |
| Compound Employee API | Built for bulk replication, not per-employee lookups | p. 620 |
| Admin Mode API permissions | Override RBP and return all data | p. 621–626 |

---

## 6. Sensitive APIs — Flagged

### 6.1 High-sensitivity APIs we DO use 🟠
These are needed, but they contain or can expose sensitive fields. Each one needs the listed safeguard.

| API | Risk | Mandatory safeguard |
|---|---|---|
| `/oauth/token` | A token grants access to the employee's data for 24 h | Sign assertions only for the authenticated caller; private key in AWS Secrets Manager; never log tokens |
| `User` | The same entity holds `ssn`, `salary`, `dateOfBirth`, `ethnicity`, `citizenship`, `gender`, `homePhone` | Fixed `$select` of safe fields; **never** grant *Employee Export*; field-level view only for safe fields |
| `EmployeeTime` (read) | `comment`, attachment and `loa*` fields may reveal medical or personal reasons | Fixed `$select` of allowlisted fields; drop everything else in the Lambda |
| `EmployeeTime` (create / cancel) | Could book leave for someone else or skip approval if misused | Self-only `userId`; `workflowConfirmed=true`; no *Admin access to MDF OData API*; *Always Ask* in Quick |
| `approveWfRequest`, `rejectWfRequest` | Act on **other employees'** requests | Only if managers are in scope, after a separate security review |

### 6.2 Prohibited APIs 🔴
All APIs in [§5.1](#51-prohibited--sensitive-personal-data-). They hold data covered by UST's policy on sensitive data (national IDs, passport and driver's licence numbers, payment card numbers, patient records) or other special-category data (bank, salary, health, family, demographic, termination).

### 6.3 Indirect Exposure to Watch

| Path | Risk | Control |
|---|---|---|
| `$expand` from an allowed entity (e.g. `EmpJob` → compensation, `User` → person navigations) | Pulls data from a prohibited entity | Only allowlisted navigations in the Lambda; RBP on the target entity still applies (p. 627) |
| Per-employee tokens for managers and HR staff | Their other roles give wider access (team or whole company) | The Lambda forces `userId = caller` on every query |
| Custom fields (`cust_*`) | May hold sensitive information | Excluded by default; review each with EBSCO |
| API audit logs | Payloads may contain personal data (kept up to 90 days) | Log payloads for edit errors only; restrict audit-log access (p. 53–56) |

---

## 7. Authentication

| Option | How it works | Fit for HR Assist | Ref |
|---|---|---|---|
| **OIDC via SAP Identity Authentication (IAS)** | If the SuccessFactors tenant uses IAS: client ID/secret in IAS → `/oauth2/token` → exchange the ID token (JWT bearer) for an API access token (1-hour default lifetime). | Preferred by SAP **if EBSCO's tenant is integrated with IAS**. Confirm with EBSCO. | p. 13–17 |
| **OAuth 2.0 with SAML bearer assertion** | Register an OAuth client (X.509 certificate) → build a SAML assertion with `NameID` = SuccessFactors user ID and `api_key` → `POST /oauth/token` → access token valid **24 hours**. | **Recommended default.** Assertions are signed by our backend (private key in AWS Secrets Manager) **for the asking employee's user ID**, so RBP limits data to that employee. | p. 18–29 |
| HTTP Basic Authentication | User name + company ID + password | **Not allowed.** Deprecated and being retired. | p. 13, 30 |

**OAuth client registration** (Admin Center → API Center → *OAuth Configuration for OData* → *Register Client Application*; needs *Manage OAuth2 Client Applications*) (p. 19–20):
- Upload the **X.509 public key** (RSA-2 / SHA-2). Keep the private key only in AWS Secrets Manager.
- **Bind to Users:** decide deliberately. If no users are bound, *all business users* can request tokens, which the per-employee token pattern needs. If you bind specific users, only they can.
- **Never** generate assertions with the deprecated `/oauth/idp` API (p. 26).

**Token handling:** cache tokens per user until expiry (`/oauth/validate` checks validity, p. 29). Never log tokens.

**Network:** optionally restrict API access by IP address to the AWS egress IPs (p. 34). Use the tenant's data-centre API host, e.g. `https://api<dc>.sapsf.com/odata/v2/` (p. 7).

---

## 8. Permissions (Role-Based Permissions)

Create a dedicated **"HR Assist – Employee Self (API)"** role. Grant it to the target employees with target population **Self**.

| Area | Grant (User Mode) | Do NOT grant | Ref |
|---|---|---|---|
| User entity | *General User Permission → User Search*; Employee Data field-level **view** for the safe fields only | *Employee Export*, *Employee Import / Import Employee Data* | p. 454–456 |
| Person / Employment (HRIS) | *User Permissions → Employee Central Effective Dated Entities* (Job Info view); *Employee Data* (Employment Details view; *Business Email Address* view) | *Employee Central HRIS OData API (read-only / editable)*: Admin Mode | p. 623–625, 1258 |
| Foundation Objects | *Manage Foundation Objects Types* (view, for `FOLocation`); *MDF Foundation Objects* (view, for Company / Department / Division) | *Employee Central Foundation OData API (read-only / editable)*; *Admin Access to MDF OData API* | p. 625–626 |
| Time Off | Time Management object permissions: view time accounts, balances, absences, holiday calendars (self). **Phase 2:** create absence (self). | *Admin access to MDF OData API*: also **bypasses the leave approval workflow** | p. 1338, 1347 |
| Workflow | None needed for `MyPendingWorkflow` | *Manage Workflows → Manage Workflow Requests* | p. 1630, 1659 |
| Integration | *Manage OAuth2 Client Applications* (admin only, setup) · *Access to OData API Audit Log* (named admins only) | *Allow Admin to Access OData/REST API through Basic Authentication* | p. 19, 31, 53 |

> **⚠ Verify** the exact names of the Time Management permissions in EBSCO's *Manage Permission Roles*. The guide refers to them under *Permissions Required for Time Management* (p. 1339).

---

## 9. Connector Design

```
HR Assist (Amazon Quick)
   │  Custom action connector (OpenAPI 3.0, ~6–8 operations)
   ▼
API Gateway + Lambda facade
   │  • Identity from the authenticated session (never from the AI model)
   │  • SAML assertion for that employee's userId → OAuth token (cached ≤ 24 h)
   │  • Fixed $select/$filter per operation; field allowlist on responses
   │  • Retry with backoff on 429; audit logging
   ▼
SAP SuccessFactors OData v2
```

| Operation | Type | Underlying calls | Confirmation |
|---|---|---|---|
| `getMyProfileSummary` | Read | `User` (safe `$select`), `EmpEmployment`, `EmpJob`, `FOCompany`, `FOLocation` | No |
| `getMyLeaveBalances` | Read | `TimeAccount` (find types) → `EmpTimeAccountBalance`, `TimeAccountType` | No |
| `getMyLeaveTypes` | Read | `TimeTypeProfile` → `AvailableTimeType` → `TimeType` | No |
| `getMyLeaveRequests` | Read | `EmployeeTime` (allowlisted fields) | No |
| `getMyHolidays` (date range) | Read | `EmpJob.holidayCalendarCode` → `HolidayAssignment`, `Holiday` | No |
| `getMyManagerAndHRContact` | Read | `User` → `manager`, `hr`; `EmpJobRelationships` | No |
| `submitLeaveRequest` *(Phase 2)* | **Write** | `EmployeeTime` insert with `workflowConfirmed=true` | **Always** (*Always Ask*) |
| `getMyPendingWorkflows` *(Phase 2)* | Read | `MyPendingWorkflow`, `TodoEntryV2` | No |

---

## 10. Limits, Logging & Operations

| Topic | Detail | Ref |
|---|---|---|
| Rate limits | Tenant-level and resource-level policies, shared across **all** integrations (example: tenant quota 600/min; `$batch` 25/min). Headers `RateLimit-Policy`, `RateLimit`, `Retry-After`. Soft or hard mode; 429 when exceeded. | p. 40–49 |
| OData v2 limits | `in` clause ≤ 1000 values; `$expand` ≤ 10 levels; ≤ 1000 records per upsert; ≤ 180 requests per `$batch`; page size up to 1000. | p. 262, 46 |
| Best practices | Avoid excessive single-record queries; implement retry logic; use compression; avoid excessive multithreading. | p. 263–271 |
| Caching | Cache reference data (time types, holiday calendars, FOs) to save quota. Never cache personal data beyond the request. | — |
| Audit logs | API Center audit logs record transactions and optionally payloads (**may include sensitive data**); retention up to 90 days. Enable payloads for **edit errors only**; restrict access. | p. 53–56 |

---

## 11. Approvals

| # | Approval | Approver |
|---|---|---|
| AP1 | Final API scope (this document) | HR (product owner) + Security |
| AP2 | Authentication method (OAuth SAML bearer vs OIDC/IAS) and OAuth client registration | SuccessFactors administrator + Security |
| AP3 | "HR Assist – Employee Self (API)" permission role, target population **Self** | SuccessFactors administrator |
| AP4 | Data handling: fields exposed, logging, retention | Data Protection / Compliance |
| AP5 | Phase 2 write access (`submitLeaveRequest`) and workflow behaviour | HR + SuccessFactors administrator |

---

## 12. Open Questions for EBSCO

Each question lists the answers we're **likely** to hear and what each one means for HR Assist. Confirm the actual answers with EBSCO.

### 12.1 Is Employee Central Time Off enabled?
*Needed for leave balances, leave types and leave requests.*

| Likely answer | What it means for us |
|---|---|
| **Yes**: employees already apply for leave in SuccessFactors (most likely) | Leave questions can be answered from SuccessFactors. Ask EBSCO for the list of **time account type codes** (e.g. `VACATION`, `SICK_LEAVE`), because the balance query needs them. |
| **No**: leave is managed in another system (payroll or a local HR tool) | Leave balances and requests **can't** come from SuccessFactors. Either connect that other system or keep personal leave questions out of scope. |

- [ ] Answer:

### 12.2 Is the SuccessFactors tenant integrated with SAP Identity Authentication (IAS)?
*Decides how our connector signs in to SuccessFactors.*

| Likely answer | What it means for us |
|---|---|
| **Yes**: IAS is set up and passes logins through to Okta (likely, as SAP has been moving customers onto IAS) | We can sign in using **OIDC through IAS**. The **IAS administrator** becomes another contact. |
| **No**: Okta connects to SuccessFactors directly (single sign-on) | We use **OAuth 2.0 with SAML assertions**: our backend signs a short assertion for the asking employee and exchanges it for an access token. |

- [ ] Answer:

### 12.3 What is the API server / data centre for the tenant?
*Every API call goes to this address.*

| Likely answer | What it means for us |
|---|---|
| A host such as `api<number>.sapsf.com` or `api<number>.successfactors.com`, a **company ID**, and a region (US, EU or APJ) | Used to build every API URL and the token request. The region may also raise **data-residency** questions for Security. |

- [ ] Answer:

### 12.4 Does the Okta login email match the employee's record in SuccessFactors?
*We must reliably link the person chatting in Teams to their SuccessFactors record.*

| Likely answer | What it means for us |
|---|---|
| **The business email matches**, but the SuccessFactors **`userId` is the employee number** (most common) | The connector first looks up the employee by email to find their `userId`, then uses that `userId` for everything else. |
| **`username` / `userId` is the email itself** | A simpler, direct mapping. |
| **Emails don't always match** (aliases, rehires, contractors) | We need an agreed matching rule or a mapping table. If there's no reliable match, the assistant must refuse personal lookups. |

- [ ] Answer:

### 12.5 Where are HR business partners held?
*Lets escalations go to the right HR person instead of one shared inbox.*

| Likely answer | What it means for us |
|---|---|
| **Set up as a job relationship** (e.g. type "HR Manager" / "HR Business Partner"), which also fills `User.hr` | The assistant can tell employees who their HR contact is and **route escalations to that person**. |
| **Not maintained** | Keep escalations going to the **central HR inbox**. Routing by category can be added later. |

- [ ] Answer:

### 12.6 Are any custom fields (`cust_*`) on `EmployeeTime` or `EmpJob` relevant or sensitive?
*Tenants usually add their own fields, which can be useful or risky.*

| Likely answer | What it means for us |
|---|---|
| **Yes, a few exist** (e.g. work arrangement, band or shift on `EmpJob`; reason codes, notes or attachments on `EmployeeTime`) | Review each field. **Useful** ones (e.g. WFH eligibility) can be added to the allowlist; **free-text, medical or pay-related** ones stay excluded. |
| **None** | No change to the scope. |

- [ ] Answer:

### 12.7 Is there a preview / test SuccessFactors instance?
*We need a safe place to build and test the connector.*

| Likely answer | What it means for us |
|---|---|
| **Yes, a preview (test) tenant** (most common) | Build and test there first, then repeat the setup (OAuth client, permission role) in production. |
| **Yes, but it holds a copy of real production data** | It needs the **same security approvals and data handling** as production. |
| **No** | Testing would touch production. Use a small pilot group with read-only access only, and get explicit approval. |

- [ ] Answer:

### 12.8 Does HR want personal-data answers at all?
*Decides how much of this integration is needed.*

| Likely answer | What it means for us |
|---|---|
| **Phased** (most likely): policy questions first, then leave balances and holidays; salary, payroll and benefits enrolment stay out | Phase 1 might need only reference data (holidays, leave types). Personal lookups come next, once Security signs off. |
| **Yes, from launch** | All **Required** APIs in §3 are needed, and the permission role and approvals must be ready before go-live. |
| **No, policy questions only** | Only the reference-data APIs (company/location lookups, leave types and holidays) are needed, or no SuccessFactors connection at all. |

- [ ] Answer:

### 12.9 Who owns the SuccessFactors OAuth and permission-role setup?
*Most setup steps need this person.*

| Likely answer | What it means for us |
|---|---|
| An **EBSCO SuccessFactors / HRIS administrator** or an **SAP implementation partner** | This is the contact for registering the OAuth client, creating the permission role and group, and enabling audit logs. Note that Provisioning changes can only be made by SAP or the implementation partner. |

- [ ] Answer:
