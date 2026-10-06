# HR Assist — SAP SuccessFactors API Scope (OData v2)

> **Purpose:** Define the **minimum** set of SAP SuccessFactors APIs the HR Assist agent needs, with the permissions, authentication and safeguards for each, so the connector schema stays small, access stays narrow and sensitive data stays out.
> **Source:** *SAP SuccessFactors API Reference Guide*, Document Version 2H 2026 (2026-10-02). Page numbers are given as **(p. N)**.
> **Status:** Verified against the API reference guide. **Scope is provisional until HR requirements are received.** Entity availability still depends on the modules and Provisioning settings enabled in the EBSCO tenant (p. 622).

---

## Contents

- [1. Key Findings from the Deep-Dive](#1-key-findings-from-the-deep-dive)
- [2. Summary of In-Scope APIs](#2-summary-of-in-scope-apis)
- [3. In-Scope Categories \& APIs](#3-in-scope-categories--apis)
- [4. Excluded APIs — Do Not Connect](#4-excluded-apis--do-not-connect)
- [5. Authentication](#5-authentication)
- [6. Permissions (Role-Based Permissions)](#6-permissions-role-based-permissions)
- [7. Connector Design](#7-connector-design)
- [8. Limits, Logging \& Operations](#8-limits-logging--operations)
- [9. Approvals](#9-approvals)
- [10. Open Questions for EBSCO](#10-open-questions-for-ebsco)

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

## 2. Summary of In-Scope APIs

| # | Category | Entities | Access | Sensitivity | Phase |
|---|---|---|---|---|---|
| 1 | Identity & user resolution | `User` (limited `$select`), `PerEmail` | Read | Medium | **1** |
| 2 | Employment & job context | `EmpEmployment`, `EmpJob` | Read | Medium | **1** |
| 3 | Organisation lookups | `FOCompany`, `FODepartment`, `FODivision` (MDF), `FOLocation` | Read | Low | **1** |
| 4 | Leave balances & types | `EmpTimeAccountBalance`, `TimeAccount`, `TimeAccountType`, `TimeType`, `TimeTypeProfile` → `AvailableTimeType` | Read | Medium | **1** |
| 5 | Holidays | `HolidayCalendar`, `HolidayAssignment`, `Holiday` | Read | Low | **1** |
| 6 | Leave requests | `EmployeeTime` (allowlisted fields) | Read (Ph1) → Create (Ph2) | Medium | 1 / 2 |
| 7 | Manager & HR contact | `User` → `manager`, `hr` navigation; `EmpJobRelationships` | Read | Medium | **1** |
| 8 | My workflows & to-dos | `MyPendingWorkflow`, `TodoEntryV2` | Read | Medium | 2 |
| 9 | Supporting | `$metadata`, MDF picklists (`PickListValueV2`) | Read | Low | **1** |

---

## 3. In-Scope Categories & APIs

### 3.1 Identity & User Resolution
*Links the signed-in employee (Okta / Teams email) to their SuccessFactors user ID.*

| Entity | What it does | Use in HR Assist | Safe fields (`$select`) | Ref |
|---|---|---|---|---|
| `User` | Represents an employment: organisational info plus some personal info. Non-effective-dated (current values only). | Resolve email → `userId`; name for greetings | `userId`, `username`, `email`, `firstName`, `lastName`, `displayName`, `status`, `department`, `location`, `timeZone` | p. 453 |
| `PerEmail` | Employee email addresses (key: `personIdExternal` + `emailType`). | Fallback lookup if the login email isn't `User.email` | `personIdExternal`, `emailAddress`, `emailType`, `isPrimary` (**business email only**) | p. 1257–1258 |

> **Never** select or expand `ssn`, `salary`, `salaryLocal`, `dateOfBirth`, `gender`, `ethnicity`, `citizenship`, `nationality`, `married`, `homePhone` or similar fields on `User`.

### 3.2 Employment & Job Context
*Decides which version of a policy applies (country, legal entity, employee type, schedule).*

| Entity | What it does | Use in HR Assist | Key fields | Ref |
|---|---|---|---|---|
| `EmpEmployment` | Employment record (key: `personIdExternal` + `userId`). | "When did I join?"; tenure-based entitlements | `userId`, `startDate`, `originalStartDate`, `seniorityDate`, `serviceDate` | p. 735 |
| `EmpJob` | Effective-dated job information. | Pick the right local policy, holiday calendar and leave profile | `company`, `countryOfCompany`, `location`, `department`, `division`, `jobTitle`, `employmentType`, `isFulltimeEmployee`, `standardHours`, `managerId`, `holidayCalendarCode`, `timeTypeProfileCode`, `workscheduleCode` | p. 754 |

> Don't select pay-related fields on `EmpJob` (e.g. `payGrade`, `costCenter`).

### 3.3 Organisation Lookups (Foundation Objects)
*Turns codes into readable names. Not personal data.*

| Entity | Type | Use in HR Assist | Ref |
|---|---|---|---|
| `FOCompany` | MDF FO | Legal entity and country, to pick the country-specific policy | p. 933 |
| `FODepartment`, `FODivision` | MDF FO | Department/division names; escalation routing | p. 939, 943 |
| `FOLocation` | Classic FO | Location-specific rules (holidays, office policy) | p. 883 |

### 3.4 Leave Balances & Leave Types
*"How many leave days do I have left?" Requires Employee Central **Time Off** to be enabled (p. 1338).*

| Entity | What it does | Use in HR Assist | Ref |
|---|---|---|---|
| `EmpTimeAccountBalance` | Calculates the balance per valid time account as of a date (default today). Returns `timeAccount`, `timeAccountType`, `timeUnit` (days/hours), `balance`, `userId`, `accountClosed`. | **Primary source** for leave balances. Query: `$filter=userId eq '<self>' and timeAccountType in (...)`, optional `balanceAsOfDate=YYYY-MM-DD`. | p. 794–796 |
| `TimeAccount` | The employee's time accounts with validity and booking periods (`accountType`, `startDate`, `endDate`, `bookingStartDate`, `bookingEndDate`, closed flag). | Find the employee's account types for the balance call; explain validity / expiry | p. 1431 |
| `TimeAccountType` | Template for time accounts. | Readable names for account types | p. 1453 |
| `TimeType` | Leave types configured in the tenant. | Map "casual leave" to the right code; list valid types | p. 1491 |
| `TimeTypeProfile` → `AvailableTimeType` | The time types an employee may take (from their job info). | Only offer leave types the employee is eligible for | p. 1343, 1516–1517 |

### 3.5 Holidays
*"Is next Monday a holiday?" for the employee's own calendar (`EmpJob.holidayCalendarCode`).*

| Entity | What it does | Ref |
|---|---|---|
| `HolidayCalendar` | Defines the public holidays for an employee; filterable by `country`. | p. 1420 |
| `HolidayAssignment` | Holiday dates in a calendar, with `holidayClass` (FULL / HALF / NONE). | p. 1412–1413 |
| `Holiday` | Holiday definitions (names). | p. 1389 |

### 3.6 Leave Requests

| Entity | What it does | Use in HR Assist | Ref |
|---|---|---|---|
| `EmployeeTime` | Absences (vacation, sick leave, other paid time off). Supports Query, Insert, Merge, Replace, Upsert, Delete. | **Phase 1 (read):** "Is my leave on the 22nd approved?" **Phase 2 (create):** submit a leave request after explicit confirmation, with `workflowConfirmed=true`. | p. 1347 |

**Allowlisted fields:** `externalCode`, `userId`, `timeType`, `startDate`, `endDate`, `quantityInDays`, `quantityInHours`, `approvalStatus`.
**Never return:** `comment`, attachments, or the leave-of-absence fields (`loa*`). These may reveal medical or other sensitive information.

### 3.7 Manager & HR Contact

| Entity | What it does | Use in HR Assist | Ref |
|---|---|---|---|
| `User` → `manager`, `hr` | Navigation to the employee's manager and HR representative. | "Who is my manager / HR partner?"; route escalations to the right HR contact | p. 453 ff. |
| `EmpJobRelationships` | Additional relationships (key: `userId` + `relationshipType` + `startDate`), e.g. HR business partner, matrix manager. | Finer escalation routing if HR partners are held here | p. 772 |

> Return **name and business contact only** for the manager and HR partner.

### 3.8 My Workflows & To-Dos (Phase 2)

| Entity | What it does | Use in HR Assist | Ref |
|---|---|---|---|
| `MyPendingWorkflow` | Workflow requests where the **current login user** is a contributor (pending / sent back) or CC (completed). | "Where is my request?"; managers: "what's waiting for my approval?" | p. 1659 |
| `TodoEntryV2` | To-do items for a user (category, status, due date, deep link). | Remind employees of pending HR tasks | p. 404 |

### 3.9 Supporting

| Item | Use | Ref |
|---|---|---|
| `$metadata` | Design-time only: generate and validate the connector schema. Not exposed to the agent. | p. 88 ff. |
| MDF picklists (`PickListValueV2`) | Turn codes (e.g. approval status) into readable labels | p. 325 |
| API Center → **OData API Data Dictionary** | Check entities and fields available in the EBSCO tenant | p. 51 |

---

## 4. Excluded APIs — Do Not Connect

Leave these out of the schema **and** the permission role. Several are also covered by UST's policy on sensitive data, which prohibits processing national IDs, passport and driver's licence numbers, payment card numbers and patient records.

| Entity / area | Contains | Reason | Ref |
|---|---|---|---|
| `PerNationalId`, `PerNationalIdWithValidityPeriod` | National IDs (e.g. SSN, Aadhaar, PAN) | **Prohibited** by organisation policy | p. 1271–1274 |
| `EmpWorkPermit` | Passport / visa / work permit details | **Prohibited** by organisation policy | p. 781 |
| Payment Information (`PaymentInformationDetailV3*`, `Bank`) | Bank account details | Financial data | p. 1187 ff. |
| `EmpCompensation`, `EmpPayCompRecurring`, `EmpPayCompNonRecurring`, Payroll, Deductions, Advances | Salary, pay, deductions | Pay data. Only if HR explicitly requests it **and** Security/Legal approve | p. 698, 720, 785–789, 1217 |
| `PerPersonal`, `PerGlobalInfo<Country>`, `PerBiographicalInfoLoc<Country>` | Date of birth, gender, marital status, nationality, local personal info | Special-category data; not needed | p. 1254, 1265, 1283 |
| `PerAddressDEFLT`, `PerPhone`, `PerEmergencyContacts`, `PerPersonRelationship`, `PerSocialAccount` | Home address, phones, dependants, social accounts | Not needed | p. 1250–1301 |
| `EmpBeneficiary`, Global Benefits enrolment entities | Beneficiaries, dependants, insurance enrolment | Sensitive; benefits **policy** questions are answered from the knowledge base instead | p. 718, 990 ff. |
| `EmpEmploymentTermination`, `PersonEmpTerminationInfo` | Termination details | Out of scope | p. 742, 1260 |
| Performance & Goals, Calibration, Succession, Recruiting, Background entities | Ratings, goals, candidates, education | Out of scope for an HR FAQ assistant | p. 569, 1710, 2526, 2758, 3032 |
| `WfRequest` (admin) | All workflow requests | Needs admin permission. Use `MyPendingWorkflow` | p. 1630 |
| Sensitive `User` fields | `ssn`, `salary`, `dateOfBirth`, `ethnicity`, `citizenship`, etc. | Excluded via `$select` and field-level permissions | p. 453 ff. |

---

## 5. Authentication

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

## 6. Permissions (Role-Based Permissions)

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

## 7. Connector Design

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

## 8. Limits, Logging & Operations

| Topic | Detail | Ref |
|---|---|---|
| Rate limits | Tenant-level and resource-level policies, shared across **all** integrations (example: tenant quota 600/min; `$batch` 25/min). Headers `RateLimit-Policy`, `RateLimit`, `Retry-After`. Soft or hard mode; 429 when exceeded. | p. 40–49 |
| OData v2 limits | `in` clause ≤ 1000 values; `$expand` ≤ 10 levels; ≤ 1000 records per upsert; ≤ 180 requests per `$batch`; page size up to 1000. | p. 262, 46 |
| Best practices | Avoid excessive single-record queries; implement retry logic; use compression; avoid excessive multithreading. | p. 263–271 |
| Caching | Cache reference data (time types, holiday calendars, FOs) to save quota. Never cache personal data beyond the request. | — |
| Audit logs | API Center audit logs record transactions and optionally payloads (**may include sensitive data**); retention up to 90 days. Enable payloads for **edit errors only**; restrict access. | p. 53–56 |

---

## 9. Approvals

| # | Approval | Approver |
|---|---|---|
| AP1 | Final API scope (this document) | HR (product owner) + Security |
| AP2 | Authentication method (OAuth SAML bearer vs OIDC/IAS) and OAuth client registration | SuccessFactors administrator + Security |
| AP3 | "HR Assist – Employee Self (API)" permission role, target population **Self** | SuccessFactors administrator |
| AP4 | Data handling: fields exposed, logging, retention | Data Protection / Compliance |
| AP5 | Phase 2 write access (`submitLeaveRequest`) and workflow behaviour | HR + SuccessFactors administrator |

---

## 10. Open Questions for EBSCO

- [ ] Is **Employee Central Time Off** enabled? (Needed for balances, leave types and requests.)
- [ ] Is the SuccessFactors tenant integrated with **SAP Identity Authentication (IAS)**? This decides OIDC vs OAuth SAML bearer.
- [ ] What is the **API server / data centre** for the tenant?
- [ ] Does the **Okta login email** match `User.email` / `username` in SuccessFactors?
- [ ] Are HR business partners held in `User.hr` or as an `EmpJobRelationships` type?
- [ ] Are any **custom fields** (`cust_*`) on `EmployeeTime` / `EmpJob` relevant or sensitive?
- [ ] Is there a **preview / test** SuccessFactors instance for building the connector?
- [ ] Does HR want personal-data answers at all? If not, only Categories 3, 5 and 9 are needed.
