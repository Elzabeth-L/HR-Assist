# HR Assist — Knowledge Base Source Options & Justifications


This document compares the data sources considered for HR Assist in the Enterprise sandbox and explains how each one is used.

---

## 1. Summary & Recommendation

| Source | What it provides | How Quick uses it | Keeps content fresh | Carries permissions | Recommendation |
|---|---|---|---|---|---|
| **Confluence Cloud** | HR **policy documents** (leave, WFH, holidays, benefits, FAQs) | Native integration → indexed knowledge base | Yes, scheduled sync | Yes, document-level ACLs if admin credentials are supplied at setup | **Primary knowledge base** |
| **SAP SuccessFactors** | **Employee-specific HR data** (leave balances, leave requests, holiday calendar, manager/HR contact, job details) | Custom action connector (OpenAPI) queried live, **not** indexed | Yes, always live (read at query time) | Yes, SuccessFactors role-based permissions | **Live data source** for personal questions |

**Why both:**
- **Confluence** is where HR policies already live. Indexing it directly keeps the agent in sync with the real source of truth.
- **SuccessFactors** is the system of record for employee data. Questions like "how many leave days do I have left?" can't be answered from policy documents, so they must come from SuccessFactors live, scoped to the employee asking.

Together, the agent can answer both **"What is the policy?"** (Confluence) and **"What does it mean for me?"** (SuccessFactors).

---

## 2. Atlassian Confluence Cloud

Confluence supports **both** knowledge base creation (indexing spaces, pages and blog posts) **and** actions (create/update pages) in a single integration.

### Steps
1. **Integrations → Add → Bring data from Atlassian Confluence Cloud.**
2. Enter the site URL (`your-site.atlassian.net`) → **Next**.
3. Complete the OAuth pop-up and **accept** the permissions.
4. Under **Create knowledge base**:
   - Give the KB a name.
   - Paste the Confluence **space or page URLs** to index, one at a time.
5. Configure multimedia and file-size settings → **Create**.
6. Sync runs automatically (**daily by default**, configurable).
7. Link the KB to a **Space**, then link the Space to the **agent**.

### Things to watch
- **URL scoping:** a URL like `.../wiki/spaces/SPACEKEY/overview` is treated as a **single page**, not the whole space. To index everything under a space, use the **base space URL**.
- **Document-level ACLs:** entering Atlassian **admin credentials** at the authentication step carries Confluence's permission structure into Quick. **This can't be added later**, so decide before you create the KB.
- **Scoping large sites:** you can create several narrow KBs from the same Confluence source (for example "HR Policies" and "Engineering Docs"). An agent only searches the KBs it is linked to.

---

## 3. SAP SuccessFactors

SuccessFactors holds **structured employee records**, not policy documents. Rather than being indexed, it is connected as a **custom action connector** that the agent calls live when an employee asks a personal question.

> **⚠ Verify:** check the Quick **Integrations** list in the sandbox for a native SuccessFactors connector. If none exists, use the custom OpenAPI pattern below.

### Steps
1. **SuccessFactors admin:** register an OAuth 2.0 client (**Admin Center → Manage OAuth2 Client Applications**) and create a dedicated, read-only **"HR Assist API"** permission role limited to each employee's own data.
2. **AWS:** deploy a small facade (**API Gateway + Lambda**) that:
   - takes the employee's identity from the authenticated session (never from the AI model),
   - calls the SuccessFactors OData v2 APIs,
   - returns only allowlisted fields.
3. **Write an OpenAPI 3.0 spec** for a few task-based operations, e.g. `getMyLeaveBalances`, `getMyLeaveRequests`, `getMyHolidays`, `getMyManagerAndHRContact`.
4. **Amazon Quick → Connectors → Create for your team → OpenAPI Specification → Import Schema.**
5. Link the connector to the HR Assist agent. Keep any write action (e.g. submitting a leave request) set to **Always Ask**.

### Data in scope

| Area | SuccessFactors entities | Example question |
|---|---|---|
| Leave balances & types | `EmpTimeAccountBalance`, `TimeType`, `TimeTypeProfile` | "How many casual leaves do I have left?" |
| Leave requests | `EmployeeTime` | "Is my leave on the 22nd approved?" |
| Holidays | `HolidayAssignment`, `HolidayCalendar` | "Is next Monday a holiday at my location?" |
| Job & employment context | `EmpJob`, `EmpEmployment` | "When did I join?" / picking the right local policy |
| Manager & HR contact | `User` (manager/HR), `EmpJobRelationships` | "Who is my HR partner?" |

### Things to watch
- **Sensitive data:** never connect national IDs, passport/work permit, bank details, compensation, personal details or medical data.
- **Identity mapping:** the Okta / Teams login email must match the employee's SuccessFactors record.
- **Module availability:** leave balances need the **SuccessFactors Time Off** module to be enabled.
- **Approvals:** HR (product owner), Security and the SuccessFactors admin must sign off the API scope and permission role.

---

## 4. Other Features to Explore in the Enterprise Sandbox

The sandbox is Enterprise-tier and low-risk, so these are worth trying:

| Feature | What to try | Value for HR Assist |
|---|---|---|
| **Action connectors** | **Jira / ServiceNow** (HR tickets), **Slack / Outlook** (notifications) | Lets the same agent both **inform** (answer questions) and **act** (raise tickets, notify HR) |
| **MCP integration** | Connect a custom internal tool through Model Context Protocol instead of OpenAPI | Extensibility for in-house systems (included with Enterprise) |
| **Quick Automate** | Build a pipeline: *Confluence policy page updated → summarise the change → notify HR channel* | Covers the "pipelines" exploration item |

---

## 5. Notes on Sync Behaviour

**Confluence (indexed knowledge base):**
- Content is **indexed ahead of time**, not at query time.
- Sync is set per KB under **Sync Schedules** (Daily/Weekly/Monthly or **Sync now**). It can email on failure and has a maximum-deletion-percentage safeguard.
- Sync is **incremental**: only added, modified or deleted documents are reprocessed.

**SuccessFactors (live action connector):**
- No sync or index. Data is read **live** on each request, so balances and request statuses are always current.
- Nothing from SuccessFactors is stored in the Quick knowledge base.
