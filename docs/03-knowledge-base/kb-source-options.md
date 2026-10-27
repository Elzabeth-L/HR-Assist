# HR Assist — Knowledge Base Sourcing & Access

> **Status:** Updated after analysing HR's knowledge workbook (2026-10-26). Verified against AWS documentation where noted.
> **Scope:** Phase 1 content: SharePoint, the public benefits site (ebscochoice.com) and HR-written documents. SuccessFactors is **deferred**.

---

## 1. Summary by Source

| Source | Approach | Status |
|---|---|---|
| **SharePoint (internal HR sites)**: most sources | Native SharePoint knowledge bases, **one per HR topic area**, **admin-managed setup with ACLs** | ⏳ Site access / app permission pending |
| **ebscochoice.com** (public benefits site) | Web Crawler was tried and **blocked (HTTP 403)** → **manual PDF upload** to the Space | Decided; upload pending |
| **Personal OneDrive files** | Not usable as-is. HR moves the content into a **shared SharePoint library**, then it's indexed like any other SharePoint source. | ⏳ HR action |
| **Interactive web apps** (HR / IT portals) | Not indexable (logins and forms). Answers link to them instead. | — |
| **HR-written "tribal knowledge"** | Stored in a dedicated SharePoint library, one short document per topic | ⏳ HR writing |
| **SAP SuccessFactors** | Deferred (v1 is not personalised) | ⏸ |

---

## 2. SharePoint

### 2.1 Structure
- **One knowledge base per HR topic area** (e.g. Benefits, Compensation, Learning & Development, Onboarding, Policies), mirroring the workbook's *Topic* column. Not one giant knowledge base.
- All of them sit in **one Space that's already linked to the agent**, so every knowledge base added to the Space is automatically available to the agent with no per-knowledge-base agent setup.
- **Daily sync**, since HR is actively editing content.

### 2.2 Setup mode: admin-managed (verified)

Amazon Quick offers two SharePoint setup modes:

| | **User-managed** ("sign in with my account") | **Admin-managed** ("connect as an application", service credentials) |
|---|---|---|
| How Quick connects | As the **person** who signs in | As an **application** registered in EBSCO's Microsoft Entra tenant |
| Approval step | A Microsoft **consent pop-up** when you sign in. If the tenant blocks user consent, an M365 admin must approve it. | **No pop-up.** An M365 admin registers the app and grants **admin consent** to its permissions in Entra. |
| What gets indexed | Only what the signer can access | Only the sites granted to the application |
| **Per-employee permissions** | ❌ **Not supported.** *"All indexed content is accessible to any user who has access to the knowledge base."* | ✅ **Document-level access control (ACL)** |
| Fit for HR | Only for content every employee may see | **Chosen for all HR knowledge bases** |

### 2.3 How the admin-managed connection works

**In plain terms:** Quick gets its own **company ID card** for EBSCO's Microsoft environment instead of borrowing a person's login.

| Piece | Plain meaning | Where it's set up |
|---|---|---|
| **Signing key** | Quick's "pen" for signing its login; the secret part never leaves AWS | AWS account hosting Quick (AWS KMS) |
| **Certificate** | A sample of that signature, so Microsoft recognises it | Made from the key, uploaded to Microsoft |
| **Entra app registration** | Quick's **ID card** in EBSCO's tenant, listing what it may do | Microsoft Entra admin center |
| **Per-site grants** | Which HR sites the ID card opens | Microsoft (Graph), one per site |
| **Connection in Quick** | The ID card details typed into Quick once | Quick |

**Technical detail:**

| Piece | What it is | Why it's needed |
|---|---|---|
| **Signing key** (AWS KMS) | An asymmetric key in the AWS account that hosts Quick. The private key never leaves KMS. | Quick uses it to **sign its login** to Microsoft, like a digital signature |
| **Certificate** | The public half of the signing key, wrapped in a certificate | Uploaded to the Entra app so Microsoft can **verify Quick's signature** |
| **Entra app registration** | Quick's **identity** inside EBSCO's Microsoft tenant, with *application* permissions and admin consent | Lets Quick read SharePoint **as an app**, not as a person |
| **Per-site grants** (`Sites.Selected`) | An explicit grant of each HR site to that app | The app can read **only** the approved sites |
| **Connection in Quick** | Four values entered once: **Tenant ID, Client ID, KMS key ARN, certificate thumbprint** (+ SharePoint domain) | Ties Quick to the app; reusable for every knowledge base (*Connected account*) |

**During each sync:** Quick signs a request with the KMS key → Entra checks it against the certificate and issues an app token → Quick reads the granted sites **and their permission lists** → content and permissions are indexed.

**When an employee asks:** a **separate, Quick-managed** Microsoft app uses the **employee's own Microsoft sign-in** to check, document by document, what they can open (see §2.5).

**Two different Microsoft apps are involved:**

| | **Indexing app** (registered by the M365 admin) | **Permission-check app** (managed by AWS / Quick) |
|---|---|---|
| Created by | EBSCO's M365 admin, in EBSCO's tenant | AWS. It appears in EBSCO's tenant as an **Enterprise application** once consent is given; no configuration needed |
| Used for | **Syncing:** reading content and permission lists from the granted sites | **Answering:** checking what the **asking employee** can open |
| Acts as | The **application** itself (application permissions) | **The employee** (delegated permissions) |
| Permissions | `Sites.Selected`, `User.Read.All`, `GroupMember.Read.All` | `Sites.Read.All`, `Files.Read.All`, `User.Read`, `offline_access` (read-only; limited to what the employee can already access) |
| Consent | Admin consent (required) | Each employee once, **or** organisation-wide admin consent |

**What employees accept** (if org-wide consent isn't given): allowing Quick to use their **existing Microsoft sign-in** to **read what they already have access to** in SharePoint, so it can filter answers for them. It doesn't give Quick their password, and it can't reach anything they can't already open. It lasts about 90 days (refresh token).

### 2.4 Setup steps and who does them

| # | Step | Who | Where |
|---|---|---|---|
| 1 | Create the **signing key** and generate the **certificate** (AWS CLI + OpenSSL) | AWS admin of the account hosting Quick | AWS KMS |
| 2 | **Register the Quick app**: application permissions `Sites.Selected` (Graph + SharePoint), `User.Read.All`, `GroupMember.Read.All`; **grant admin consent**; upload the certificate | **M365 Global Administrator or Privileged Role Administrator** (Entra roles; the SharePoint Administrator role isn't enough) | Microsoft Entra |
| 3 | **Grant each HR site** to the app (a Graph call per site, using a temporary helper app that's deleted afterwards) | M365 Global Administrator | Microsoft Graph |
| 4 | Allow Quick to **use the signing key** | Quick administrator (Admin Pro) | Quick → Manage account → AWS resources |
| 5 | **Create each knowledge base** in Quick (see below) | Builder (Author Pro) | Quick |
| 6 | **Grant organisation-wide consent** for Quick's permission-check app (link shown in step 5) | M365 administrator | Microsoft Entra |

**Step 5: the "Create SharePoint knowledge base" wizard.** The *wizard* is Quick's step-by-step setup screen: **Knowledge → Microsoft SharePoint Online → +**. Connecting and creating a knowledge base happen in this same wizard.

| Wizard step | What you do |
|---|---|
| 1. Authentication | Choose **"Connect with service credentials"** and enter the four values. **Only the first time;** after that, pick the saved **Connected account**. |
| 2. Content | Name the knowledge base; enter the **granted site URLs** (up to 350 sites/paths) |
| 3. Additional settings | **Tick "Control document access with ACLs"** (every knowledge base; can't be changed later). ⚠️ **Clear the "Filter content by date" start date**, which defaults to the last year, so older HR policies aren't skipped. |

**Values entered in step 1 and who provides them:**

| Value | Comes from | Provided by |
|---|---|---|
| Tenant ID | EBSCO's Microsoft tenant | M365 admin |
| Client ID | The indexing app registration | M365 admin |
| KMS key ARN | The signing key | AWS admin of the Quick account (or the project team, if it has AWS console access) |
| Certificate thumbprint | Generated from the key | Whoever creates the key |
| SharePoint domain | e.g. `https://<tenant>.sharepoint.com` | M365 admin |

**Site grants can be added later.** Grant the initial site list up front. Any site added later needs its own grant **before** it's added to a knowledge base; a site that hasn't been granted fails.

> **Security review note:** AWS's per-site grant gives the app the **"fullcontrol"** role on each site, which is needed to read permission settings. Expect IT to ask about this.

**Common questions:**
- **Is there a "tenant access request" from Quick?** No. The consent pop-up seen when connecting belongs to **user-managed** setup. In admin-managed setup it's replaced by the admin **registering the app and granting admin consent** in Entra.
- **Who grants what?** App registration and admin consent: **Global Administrator or Privileged Role Administrator**. Per-site grants: **Global Administrator**. The SharePoint Administrator role alone isn't enough. It's usually the same M365 / identity team.
- **When to send the site list?** **With the initial request**, so registration and site grants happen together. More sites can be granted later.

### 2.5 How permissions are enforced (verified)

With ACLs enabled, access is checked **for each employee, on every question**:

1. **Pre-retrieval filtering:** only passages allowed by the **permission lists saved at the last sync** are considered.
2. **Real-time verification:** Quick checks the **employee's current SharePoint access** to each candidate document. *"Only documents the user is currently authorized to access are included in the response."*

**What this means:**
- The check is **per document**. Employees still get answers from documents they **can** open; documents they can't open are simply left out (no answer and no citation from them). If nothing remains, the agent says "I don't have that information".
- **Sync:** each knowledge base syncs on its schedule (e.g. daily) or with **Sync now**. Every sync is incremental (new, changed or deleted documents) **and refreshes the permission lists**.
  - **Removed access** takes effect immediately (live check).
  - **Newly granted access** may only show up after the next sync (inferred from the two-step design).
- It applies to **everyone, including the builder or admin**. QA for restricted content must be done by someone who **has** access.
- **Employee sign-in for the live check:** the permission-check app needs each employee's delegated Microsoft permission (`Sites.Read.All`, `Files.Read.All`, `User.Read`, `offline_access`).
  - **Without org-wide consent:** each employee sees a one-time "Sign in to SharePoint" / consent prompt (valid about 90 days). If EBSCO blocks user consent, they'd see an error instead.
  - **With org-wide consent:** an M365 admin approves it once **on behalf of all employees**, so no prompts appear.

> **In plain terms:** user-managed setup is like photocopying the filing cabinet with one person's key and leaving the copies for everyone. Admin-managed setup with ACLs is a librarian who checks each employee's badge before handing over a document.

### 2.6 Chosen Setup

| Decision | Choice | Why |
|---|---|---|
| Setup mode | **Admin-managed for every HR SharePoint knowledge base**, including all-employee content | One consistent model; protects content whose permissions change later |
| Document-level access | **"Control document access with ACLs" ticked** on every knowledge base | Can't be added later without rebuilding |
| App permission | **`Sites.Selected`** per HR site (not all sites) | Least privilege |
| Employee sign-in | **Organisation-wide admin consent** for the permission-check app | No prompts or "app blocked" errors |
| Date filter | **Start date cleared** | Include HR policies older than a year |
| Uploaded files | **Only content every employee may see** (e.g. public benefits PDFs) | Uploaded files carry no SharePoint permissions |

### 2.7 How access requests should be framed

Personal access requests to some sensitive sites were **declined**. The connector doesn't need anyone's personal membership. The request is for an **app-level, per-site grant**:

> *"Please grant the HR Assist SharePoint connector application access to this specific site (Sites.Selected)."*

### 2.8 Requests to EBSCO IT

- [ ] **AWS admin** (account hosting Quick): create the KMS signing key and certificate, or give the team permission to do it; share the **key ARN** and **certificate thumbprint**.
- [ ] **M365 Global / Privileged Role Admin:** register the Quick Entra app (application permissions `Sites.Selected`, `User.Read.All`, `GroupMember.Read.All`), grant admin consent, upload the certificate; share the **tenant ID** and **client ID**.
- [ ] **M365 Global Admin:** grant **each HR site** in the workbook's source list (including restricted sites) to the app.
- [ ] **Quick admin:** allow Quick to use the KMS key.
- [ ] **M365 admin:** grant **organisation-wide consent** for Quick's permission-check app.
- [ ] **Security review** of the data flow (what is indexed, what is sent to AWS, the per-site "fullcontrol" grant), together with the Microsoft 365 connector ticket.

---

## 3. ebscochoice.com (Public Benefits Site)

### 3.1 What was tried
| Setting | Value |
|---|---|
| Authentication | No authentication (public site) |
| Seed URLs | Site root, the wellness section, and specific PDF links from the workbook |
| Crawl depth | 3 (default 1), with linked pages followed |
| Domain filter | Include `ebscochoice.com` and `ebscochoice.com/wp-content/uploads/.*` |

### 3.2 Result
All seed URLs returned **HTTP 403 / VALIDATION_ERROR**; 0 pages visited or indexed. Every request was rejected before fetching.

**Diagnosis:** the site blocks the crawler at its end (bot protection / firewall), not a configuration error. AWS docs require target sites to be *"not behind a firewall"* and reachable from the Quick instance's AWS Region. A WHOIS lookup didn't identify the owner (privacy-protected registration).

### 3.3 Decision: manual upload
Download the specific PDFs listed in the workbook and **upload them into the Space**, named for traceability, e.g. `2026-Change-in-Status-Events (from ebscochoice.com).pdf`.

**Why:** only a handful of documents; fixing the block depends on an unknown external owner; Phase 1 is deliberately low-effort; and HR already reviews these documents manually, so re-uploading on change is a small extra step.

### 3.4 If revisited later
1. Ask the **benefits content owner** (who owns most of the workbook rows for this site) who administers it.
2. Ask the site owner to allowlist **Quick's crawler user agent**, `amazon-Quick-on-behalf-of-<UUID>`, rather than broad AWS IP ranges. The crawler also honours `robots.txt` rules for that user agent.
3. Turn on **"Enable file attachment crawling"** so linked PDFs are indexed.

---

## 4. Personal OneDrive Files

Not a stable source: they're tied to one person and some are unofficial. HR moves the content into a **shared SharePoint library** (not a personal OneDrive), where it's indexed the same way as other SharePoint content.

---

## 5. Sync Behaviour

- Content is **indexed ahead of time**, not at query time; ACLs are synced on each crawl and re-checked live per question.
- Sync is set per knowledge base (Daily / Weekly / Monthly or **Sync now**) and is **incremental**.
- **Uploaded files** (e.g. the benefits PDFs) are indexed immediately, but updates need re-uploading.

---

## 6. Sources (AWS)

- [Microsoft SharePoint knowledge base integration](https://docs.aws.amazon.com/quick/latest/userguide/sharepoint-knowledge-base.html)
- [User-managed setup](https://docs.aws.amazon.com/quick/latest/userguide/sharepoint-kb-user-managed.html)
- [Admin-managed setup (service credentials)](https://docs.aws.amazon.com/quick/latest/userguide/sharepoint-kb-admin-managed.html)
- [Document-level access controls](https://docs.aws.amazon.com/quick/latest/userguide/sharepoint-kb-acl.html)
- [Set up service credentials](https://docs.aws.amazon.com/quick/latest/userguide/sharepoint-kb-admin-config.html)
- [Create the knowledge base in Amazon Quick](https://docs.aws.amazon.com/quick/latest/userguide/sharepoint-kb-admin-connection.html)
- [Web Crawler integration](https://docs.aws.amazon.com/quick/latest/userguide/web-crawler-integration.html)
