# HR Assist — SAP SuccessFactors API Scope (OData v2)

> **Purpose:** Identify the **minimum** set of SAP SuccessFactors APIs the HR Assist agent needs, so the connector schema stays small, the permissions stay narrow and sensitive data stays out.
> **Status:** Draft for deep-dive. **HR requirements not yet received** (the call on 2026-10-05 was delayed), so the scope below is provisional and must be confirmed against them.
> **Reference:** [SAP SuccessFactors API Reference Guide (OData V2)](https://help.sap.com/docs/successfactors-platform/sap-successfactors-api-reference-guide-odata-v2/about-sap-successfactors-apis)

> **⚠ Verify:** entity names and fields below are standard SuccessFactors OData v2 / Employee Central entities. Availability depends on which modules are licensed and enabled in the EBSCO tenant. Confirm every entity in **API Center → OData API Data Dictionary** and the tenant's `$metadata` before building the schema.

---

## 1. Summary

| # | Category | Entities | Access | Sensitivity | Phase |
|---|---|---|---|---|---|
| 1 | Identity & user resolution | `User`, `PerEmail` | Read | Medium | **1** |
| 2 | Employment & job context | `EmpEmployment`, `EmpJob` | Read | Medium | **1** |
| 3 | Organisation lookups (Foundation Objects) | `FOCompany`, `FOLocation`, `FODepartment` | Read | Low | **1** |
| 4 | Time off: balances & leave types | `EmpTimeAccountBalance`, `TimeAccount`, `TimeType`, `TimeTypeProfile` | Read | Medium | **1** |
| 5 | Holidays | `HolidayAssignment`, `HolidayCalendar`, `Holiday` | Read | Low | **1** |
| 6 | Leave requests | `EmployeeTime` | Read → **Create** | Medium | 1 (read) / 2 (create) |
| 7 | Manager & HR contact | `User` (`manager`, `hr` navigation), `EmpJobRelationships` | Read | Medium | **1** |
| 8 | Workflow & approval status | `WfRequest`, `WfRequestStep`, `TodoEntryV2` | Read | Medium | 2 |
| 9 | Metadata & picklists (supporting) | `$metadata`, `PickListValueV2` | Read | Low | **1** |
| ✖ | **Excluded**: sensitive / out of scope | See [§3](#3-excluded-apis--do-not-connect) | — | High | Never (unless HR, Legal and Security approve) |

**Principle:** the agent only reads data **about the employee who is asking**, plus non-personal reference data (holidays, leave types, org names). Only one write action is in scope (submitting a leave request), and it always needs the employee's explicit confirmation.

---

## 2. In-Scope Categories & APIs

### 2.1 Identity & User Resolution
*Links the signed-in employee (Okta / Teams email) to their SuccessFactors record.*

| Entity | What it does | Why HR Assist needs it | Key fields |
|---|---|---|---|
| `User` | Core user record (Platform). | Resolve the employee's email to their SuccessFactors `userId`, which every other call depends on. Also gives display name and status. | `userId`, `username`, `email`, `firstName`, `lastName`, `status`, `department`, `location`, `timeZone` |
| `PerEmail` | Email addresses linked to a person (Employee Central). | Fallback lookup when the login email isn't the `User.email` (e.g. a primary vs business email mismatch). | `personIdExternal`, `emailAddress`, `emailType`, `isPrimary` |

### 2.2 Employment & Job Context
*Decides **which version** of a policy applies to the employee (by country, legal entity, employee type).*

| Entity | What it does | Why HR Assist needs it | Key fields |
|---|---|---|---|
| `EmpEmployment` | Employment record: hire and seniority dates. | Answers "when did I join?" and supports tenure-based policies (e.g. leave entitlement by years of service). | `userId`, `startDate`, `originalStartDate`, `seniorityDate`, `isContingentWorker` |
| `EmpJob` | Effective-dated job information. | Gives the company, location, employee class and holiday calendar, so the agent answers with the **right local policy** instead of a generic one. | `company`, `location`, `department`, `jobTitle`, `employeeClass`, `employmentType`, `managerId`, `holidayCalendarCode`, `timeTypeProfileCode`, `workscheduleCode` |

### 2.3 Organisation Lookups (Foundation Objects)
*Turns codes into readable names. Not personal data.*

| Entity | What it does | Why HR Assist needs it |
|---|---|---|
| `FOCompany` | Legal entity, with its country. | Picks the country-specific policy document in the knowledge base. |
| `FOLocation` | Work location. | Location-specific rules (holidays, WFH, office policies). |
| `FODepartment` | Department. | Department-specific guidance and escalation routing. |

### 2.4 Time Off: Balances & Leave Types
*The most-asked personal questions: "How many leave days do I have left?"*

| Entity | What it does | Why HR Assist needs it | Key fields |
|---|---|---|---|
| `EmpTimeAccountBalance` | Current balance per time account (read-only). | **Primary source** for "how many days do I have left". It lets the agent pass test G with real data instead of declining. | `userId`, `timeAccountType`, `balance`, `timeUnit`, `asOfAccountingPeriodEnd` |
| `TimeAccount` | Time accounts (entitlement periods) per employee. | Shows the validity period and whether leave expires or carries forward. | `userId`, `accountType`, `startDate`, `endDate`, `bookingEndDate` |
| `TimeType` | Leave types defined in the tenant (casual, sick, earned, etc.). | Maps "casual leave" in conversation to the correct time type code. Lists the valid options. | `externalCode`, `externalName`, `unit`, `category` |
| `TimeTypeProfile` | Which leave types an employee may use. | Prevents the agent offering leave types the employee isn't eligible for. | `externalCode`, `timeTypes` |

### 2.5 Holidays
*"Is next Monday a public holiday?" for the employee's own location.*

| Entity | What it does | Why HR Assist needs it |
|---|---|---|
| `HolidayAssignment` | Holidays assigned to a holiday calendar, by date. | Answers holiday questions for the employee's calendar (from `EmpJob.holidayCalendarCode`). |
| `HolidayCalendar` | Holiday calendar definitions per country/location. | Picks the correct calendar. |
| `Holiday` | Individual holiday definitions (name, class). | Holiday names and whether a holiday is full or half day. |

### 2.6 Leave Requests
*Read status first (Phase 1). Create later (Phase 2, only if HR wants actions).*

| Entity | What it does | Why HR Assist needs it | Access |
|---|---|---|---|
| `EmployeeTime` | Absence / leave requests. | **Read:** "Is my leave on the 22nd approved?" **Create (Phase 2):** submit a leave request that goes into the normal SuccessFactors approval workflow, after explicit employee confirmation. | Read (Ph1), Create (Ph2) |

> **Data handling:** do **not** return free-text comments or attachments from `EmployeeTime`. They may contain medical or other sensitive details. Return only type, dates, quantity and approval status.

### 2.7 Manager & HR Contact
*Routes escalations to the right person instead of a single generic inbox.*

| Entity | What it does | Why HR Assist needs it |
|---|---|---|
| `User` → `manager` / `hr` navigation | The employee's line manager and HR representative. | "Who is my manager / HR partner?" Also routes escalation emails and tickets to the correct HR contact. |
| `EmpJobRelationships` | Other job relationships (e.g. HR business partner, matrix manager). | Finer escalation routing where the HR partner is held as a relationship type. |

> Return **names and work contact only**, never the manager's or HR partner's own personal data.

### 2.8 Workflow & Approval Status (Phase 2)

| Entity | What it does | Why HR Assist needs it |
|---|---|---|
| `WfRequest` / `WfRequestStep` | Employee Central workflow requests and their approval steps. | "Where is my request stuck / who needs to approve it?" |
| `TodoEntryV2` | Pending to-do items for a user. | Lets managers ask "what's waiting for my approval?" (only if managers are in scope). |

### 2.9 Metadata & Picklists (Supporting)

| Entity | What it does | Why HR Assist needs it |
|---|---|---|
| `$metadata` | Schema of all entities in the tenant. | Used **at design time** to generate the connector schema. Not exposed to the agent. |
| `PickListValueV2` | Picklist labels for coded fields. | Turns codes (employee class, leave status) into readable labels. |

---

## 3. Excluded APIs — Do Not Connect

These hold highly sensitive personal data, and HR Assist doesn't need them. **Leave them out of the schema and the API user's permission role.** Several are also covered by UST's policy on sensitive data, which prohibits processing national IDs, passport and driver's licence numbers, payment card numbers and patient records.

| Entity | Contains | Reason excluded |
|---|---|---|
| `PerNationalId` | National ID numbers (e.g. SSN, Aadhaar, PAN) | **Prohibited** by organisation policy |
| `EmpWorkPermit` | Passport, visa and work permit documents/numbers | **Prohibited** by organisation policy |
| `PaymentInformationV3`, `PaymentInformationDetailV3` | Bank account details | Financial data. Never needed for FAQs. |
| `EmpCompensation`, `EmpPayCompRecurring`, `EmpPayCompNonRecurring` | Salary, pay components, bonuses | Pay data. Only reconsider if HR explicitly requests it **and** Security/Legal approve. |
| `PerPersonal`, `PerGlobalInfo*` | Date of birth, gender, marital status, nationality, country-specific personal info | Not needed; special-category data |
| `PerAddressDEFLT`, `PerPhone`, `PerEmergencyContacts` | Home address, personal phone, emergency contacts | Not needed |
| `Background_*`, performance/goal forms | Education, performance ratings, goals | Out of scope for an HR FAQ assistant |
| Payroll (Employee Central Payroll) APIs | Payslips, tax data | Out of scope; very sensitive |
| Any medical / leave-of-absence documentation | Health information | Treat as patient-record-level data: **prohibited** |

---

## 4. How the Agent Should Connect (Recommended Pattern)

Don't hand the raw OData API to the agent. Put a **thin, task-shaped facade** in front of it:

```
HR Assist (Amazon Quick)
   │  Custom action connector (OpenAPI 3.0 schema, ~6–8 operations)
   ▼
API Gateway + Lambda  ── enforces "caller = data subject", strips sensitive fields, logs calls
   │  OAuth 2.0 (SAML Bearer Assertion)
   ▼
SAP SuccessFactors OData v2
```

**Why a facade:**
- **Smaller, safer schema:** a handful of clear operations instead of hundreds of OData entities. That's easier for the agent to use correctly and well within Quick's 1–100 operation limit for OpenAPI connectors.
- **Identity enforced by code, not the AI model:** the employee's `userId` comes from the **authenticated identity**, never from a parameter the model fills in. This stops "show me John's leave balance" style leaks.
- **Field filtering:** only allowlisted fields are returned (no comments, attachments or personal details).
- **Auditing:** one place to log every call for compliance.

### Proposed connector operations (draft schema)

| Operation | Type | Underlying entities | Confirmation |
|---|---|---|---|
| `getMyProfileSummary` | Read | `User`, `EmpEmployment`, `EmpJob`, `FOCompany`, `FOLocation` | No |
| `getMyLeaveBalances` | Read | `EmpTimeAccountBalance`, `TimeType` | No |
| `getMyLeaveTypes` | Read | `TimeTypeProfile`, `TimeType` | No |
| `getMyLeaveRequests` | Read | `EmployeeTime` (filtered fields) | No |
| `getMyHolidays` (date range) | Read | `EmpJob.holidayCalendarCode`, `HolidayAssignment`, `Holiday` | No |
| `getMyManagerAndHRContact` | Read | `User` (`manager`, `hr`), `EmpJobRelationships` | No |
| `submitLeaveRequest` *(Phase 2)* | **Write** | `EmployeeTime` (create) | **Yes, always** (set to *Always Ask*) |
| `getMyPendingApprovals` *(Phase 2, managers)* | Read | `WfRequest`, `TodoEntryV2` | No |

---

## 5. Authentication & Permissions

### 5.1 Authentication
| Item | Recommendation |
|---|---|
| Method | **OAuth 2.0 with SAML Bearer Assertion** (SAP's recommended method for OData APIs). SAP is retiring Basic Authentication; **⚠ Verify** the current deprecation timeline. |
| OAuth client | Register in **Admin Center → Manage OAuth2 Client Applications** with an X.509 certificate. Keep the private key in AWS Secrets Manager. |
| User context | **Preferred:** issue the token **as the employee** (`user_id` in the SAML assertion), so SuccessFactors role-based permissions limit data to that employee. **Alternative:** a technical API user, with the Lambda facade strictly filtering to the caller's `userId`. |
| Endpoint | The tenant's data centre-specific API host (e.g. `api<dc>.successfactors.com`). Confirm with the EBSCO SuccessFactors admin. |

### 5.2 Role-Based Permissions (RBP) — dedicated "HR Assist API" role
**⚠ Verify** the exact permission names in the tenant's **Manage Permission Roles**.

| Permission area | Grant | Notes |
|---|---|---|
| Employee Central API | *Employee Central HRIS OData API (read-only)*, *Employee Central Foundation OData API (read-only)* | Add the *editable* HRIS permission only when `submitLeaveRequest` goes live (Phase 2) |
| Employee data | View: Employment Details, Job Information | Target population: **Self** (with the user-context pattern) |
| Time Management | View time account balances and absences; Create absence (Phase 2) | Self only |
| Holiday calendars / Foundation Objects | View | Reference data |
| Manage Integration Tools | OData API access via OAuth; **OData API Audit Log** access for the admin | Keep Basic Auth disabled |
| **Not granted** | Compensation, Payment Information, National ID, Work Permit, Personal Information, Global Info, Payroll | Matches [§3](#3-excluded-apis--do-not-connect) |

### 5.3 Approvals needed

| # | Approval | Approver |
|---|---|---|
| AP1 | Final API scope (this document) | HR (product owner) + Security |
| AP2 | OAuth client registration and API role creation | SuccessFactors administrator |
| AP3 | Data handling: fields exposed, logging, retention | Data Protection / Compliance |
| AP4 | Write access (`submitLeaveRequest`), Phase 2 | HR + SuccessFactors admin |

---

## 6. Open Questions for the Deep-Dive

- [ ] Which SuccessFactors modules are enabled at EBSCO (Employee Central, **Time Off**, EC Payroll)? This decides whether §2.4–2.6 exist.
- [ ] Does the login email (Okta) match `User.email` or `username` in SuccessFactors?
- [ ] Are leave balances held in SuccessFactors Time Off, or in another system?
- [ ] Is the HR business partner held as `User.hr` or as an `EmpJobRelationships` type?
- [ ] Does HR want any personal-data answers at all? If not, only Categories 3, 5 and 9 are needed.
- [ ] Is a sandbox / preview SuccessFactors instance available for building and testing the connector?
- [ ] Data centre / API host and the OAuth setup owner on the EBSCO side.
