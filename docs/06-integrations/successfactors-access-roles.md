# HR Assist — SuccessFactors Access, Roles & Connector Credentials

> **Purpose:** Explain **where** access to SuccessFactors data is controlled for HR Assist, **who** sets up each part, and **how**. The aim is that each employee can only ever see their own data.
> **Source:** *SAP SuccessFactors API Reference Guide*, 2H 2026 (2026-10-02). Page numbers are given as **(p. N)**.
> **Status:** Design. Confirm the exact permission names in EBSCO's tenant.

---

## Contents

- [1. Key Concepts](#1-key-concepts)
- [2. Where Access Is Controlled (Four Layers)](#2-where-access-is-controlled-four-layers)
- [3. Layer 1 — SuccessFactors Permission Role](#3-layer-1--successfactors-permission-role)
- [4. Connector Credentials: X.509 Certificate \& API Key](#4-connector-credentials-x509-certificate--api-key)
- [5. Layer 2 — AWS Lambda Facade](#5-layer-2--aws-lambda-facade)
- [6. Layer 3 — OpenAPI Schema](#6-layer-3--openapi-schema)
- [7. Layer 4 — Amazon Quick](#7-layer-4--amazon-quick)
- [8. Responsibilities Summary](#8-responsibilities-summary)
- [9. Setup Checklist](#9-setup-checklist)

---

## 1. Key Concepts

| Term | Meaning |
|---|---|
| **User Mode** | The API checks the calling user's **role-based permissions (RBP)**: which entities, which fields and which people they may see. HR Assist uses **only** User Mode. (p. 621) |
| **Admin Mode** | The caller holds an "Admin Mode" API permission that **overrides RBP** and returns everything. Meant for technical integration users. **Never used by HR Assist.** (p. 621, 624–625) |
| **Target population** | *Whose* data a role can see. For HR Assist: **Self** (the employee themselves). |
| **Export permission** | A bulk-extraction permission. **Employee Export** returns *all* `User` fields (including `ssn`, `salary`, `dateOfBirth`) and overrides field-level permissions. An entity Export permission without a target population skips row-level checks entirely. **Never granted.** (p. 87, 454–456) |
| **Per-employee token** | The connector requests an access token **as the asking employee** (their SuccessFactors user ID goes in the SAML assertion's `NameID`), so SuccessFactors applies that employee's permissions. (p. 26) |

> **Important caveat:** a per-employee token carries **all** of that employee's existing roles, not just the HR Assist role. Managers usually also have a "My Team" role, and HR staff may see the whole company. **Layer 2 (AWS) must therefore force every query to the caller's own user ID.** SuccessFactors permissions are the second line of defence, not the only one.

---

## 2. Where Access Is Controlled (Four Layers)

| Layer | Where | What it controls | Set up by |
|---|---|---|---|
| **1. SuccessFactors permission role** | SuccessFactors Admin Center | Which entities and fields an employee's token may read or write; target population **Self** | EBSCO SuccessFactors admin |
| **2. AWS Lambda facade** | UST-built backend (API Gateway + Lambda) | Forces every query to the caller's own `userId`; fixed queries; strips fields not on the allowlist; holds the private key in Secrets Manager | UST |
| **3. OpenAPI schema** | Connector definition imported into Quick | Which operations exist at all. No operation accepts a `userId`, so the AI can't request another person's data. | UST |
| **4. Amazon Quick** | Quick console | Action confirmation (*Always Ask* for any write action); who can use the HR Assist agent (sharing) | Quick admin / agent owner |

> **AWS IAM roles are a separate thing.** They only control what the Lambda may do *inside AWS* (read the secret, write logs). They have no effect on SuccessFactors data.

---

## 3. Layer 1 — SuccessFactors Permission Role

**Owner:** EBSCO SuccessFactors / HRIS administrator.

1. **Create a permission group:** *Admin Center → Manage Permission Groups*. Add the employees who will use HR Assist, e.g. **"HR Assist Pilot Users"** (later, all employees).
2. **Create a permission role:** *Admin Center → Manage Permission Roles → Create New*, e.g. **"HR Assist – Employee Self (API)"**.
3. **Grant only these permissions** (User Mode, view only unless noted):

| Area | Permission |
|---|---|
| User | *General User Permission → User Search*; Employee Data field-level **view** for safe fields (name, email, department, location) |
| Employment | Employee Central Effective Dated Entities: **Job Information**; Employee Data: **Employment Details** |
| Email | Employee Data → HR Information → **Business Email Address** |
| Foundation Objects | **Manage Foundation Objects Types** (view, for Location); **MDF Foundation Objects** (view, for Company / Department / Division) |
| Time Off | Time Management object permissions: view time accounts, balances, absences, holiday calendars. **Phase 2 only:** create absence. ⚠ Verify the exact names in the tenant. |

4. **Do NOT grant:**
   - *Employee Export*, *Employee Import / Import Employee Data*;
   - *Employee Central HRIS / Foundation OData API (read-only / editable)*: these are Admin Mode;
   - *Admin access to MDF OData API*: this also **bypasses the leave approval workflow** (p. 1347);
   - *Manage Workflow Requests*;
   - *Allow Admin to Access OData/REST API through Basic Authentication*;
   - anything covering compensation, payment information, national ID, work permit, or personal and family information.
5. **Grant the role:** *Grant role to* = the permission group from step 1; **target population = Granted User (Self)**.
6. **Check:** confirm that no other role held by the pilot users includes Export or Admin Mode API permissions.

---

## 4. Connector Credentials: X.509 Certificate & API Key

### 4.1 What they are

To call SuccessFactors without a password, our backend proves its identity with a **key pair**:

| Item | What it is | Where it lives |
|---|---|---|
| **Private key** | Secret half of the key pair. Used by our backend to **sign** each SAML assertion ("this request is for employee X"). | **Only** in AWS Secrets Manager. Never shared, emailed or committed. |
| **X.509 certificate (public key)** | Public half. SuccessFactors uses it to **verify** that assertions really came from us. Safe to share. | Uploaded to SuccessFactors when the OAuth client is registered |
| **API key (client ID)** | Identifier SuccessFactors issues when the OAuth client is registered. Included in every assertion and token request. | Stored with the connector configuration (Secrets Manager) |

Think of it as a **wax seal**: we keep the seal (private key), SuccessFactors keeps a picture of it (certificate) so it can recognise our stamp, and the API key is the name on our account.

### 4.2 How it works at runtime

1. An employee asks HR Assist a personal question.
2. The Lambda builds a short SAML assertion: `NameID` = the employee's SuccessFactors user ID, plus the **API key**. It **signs** the assertion with the **private key**.
3. The Lambda sends it to `POST /oauth/token`. SuccessFactors checks the signature against the **uploaded certificate**.
4. SuccessFactors returns an **access token for that employee**, valid 24 hours (p. 28). The Lambda caches it per user.
5. The Lambda calls the OData APIs with that token. SuccessFactors applies the employee's permissions.

### 4.3 Where the certificate comes from (two options)

| Option | How | Pros / cons | Ref |
|---|---|---|---|
| **A. We generate it (recommended)** | UST generates the key pair (e.g. with OpenSSL, RSA-2048+, SHA-256), stores the **private key in Secrets Manager**, and gives EBSCO **only the certificate**. | The private key never leaves our control. | p. 20–22 |
| **B. SuccessFactors generates it** | The admin uses SuccessFactors' built-in generator on the registration screen. Both keys are shown; the **private key must be saved before registering** and then handed to us securely. | Quicker, but the private key passes through people and channels. | p. 23–25 |

### 4.4 Registering the OAuth client

**Owner:** EBSCO SuccessFactors admin (needs *Manage Integration Tools → Manage OAuth2 Client Applications*, p. 19).

1. *Admin Center → API Center → OAuth Configuration for OData → Register Client Application.*
2. Enter: Application Name (e.g. `HR Assist`), Application URL (identification only), **X.509 certificate** (from UST).
3. **Bind to Users:** leave **unbound**, so that *business users* (employees) can be issued tokens. That's required for per-employee tokens (p. 20).
4. Save. SuccessFactors shows the **API key**; the admin shares it with UST.
5. **Rotation:** if certificate validity is enabled (default 365 days), plan renewal before expiry (p. 25).

> Don't use the deprecated `/oauth/idp` API to generate assertions (p. 26). Basic Authentication must not be used (p. 13, 30).

---

## 5. Layer 2 — AWS Lambda Facade

**Owner:** UST.

- Takes the employee's identity **only from the authenticated session** (Teams/Okta), maps email → SuccessFactors `userId`, and **forces `userId eq '<self>'`** on every query.
- Uses a **fixed `$select`** per operation and drops any field not on the allowlist.
- Keeps the private key and API key in **AWS Secrets Manager**. The Lambda's **IAM role** may only read those secrets and write logs.
- Caches tokens per user (≤ 24 h). Never logs tokens or personal data.
- For leave submission (Phase 2): always sends `workflowConfirmed=true`, so the normal approval workflow runs (p. 1347).

---

## 6. Layer 3 — OpenAPI Schema

**Owner:** UST.

- Defines only task-shaped operations (e.g. `getMyLeaveBalances`, `getMyHolidays`, `submitLeaveRequest`).
- **No operation has a `userId` or "employee" parameter.** "Me" is implied by the session.
- Response schemas list only allowlisted fields.

---

## 7. Layer 4 — Amazon Quick

**Owner:** Quick admin / HR Assist agent owner.

- Import the OpenAPI schema as a custom action connector and link it to HR Assist.
- Set write actions (e.g. `submitLeaveRequest`) to **Always Ask**, so employees must confirm every time.
- Share the HR Assist agent only with the approved user group.

---

## 8. Responsibilities Summary

| Task | EBSCO SuccessFactors admin | UST | Quick admin |
|---|---|---|---|
| Permission group and role (target population Self) | ✅ | Specifies | |
| Generate key pair and certificate | (option B) | ✅ (option A) | |
| Store private key in Secrets Manager | | ✅ | |
| Register OAuth client, issue API key | ✅ | Provides certificate | |
| Lambda facade (self-only enforcement, field allowlist) | | ✅ | |
| OpenAPI schema | | ✅ | |
| Import connector, action confirmation, agent sharing | | Supports | ✅ |
| Audit log access and certificate renewal | ✅ | Reminds | |

---

## 9. Setup Checklist

- [ ] Permission group "HR Assist Pilot Users" created
- [ ] Permission role "HR Assist – Employee Self (API)" created with only the listed permissions
- [ ] Role granted to the group with target population **Self**
- [ ] Pilot users checked for other roles with Export / Admin Mode API permissions
- [ ] Key pair generated by UST; private key in AWS Secrets Manager
- [ ] Certificate sent to EBSCO; OAuth client registered (unbound); API key received
- [ ] Lambda enforces self-only queries and field allowlist; tested with a manager and an HR user account
- [ ] Write actions set to *Always Ask* in Quick
- [ ] Certificate expiry date recorded and renewal planned
