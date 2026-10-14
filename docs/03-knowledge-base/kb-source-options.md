# HR Assist — Knowledge Base Sources

> **Status:** Updated after HR requirements call #1 (2026-10-14). Phase 1 content is **SharePoint** plus **new HR-written documents**. SuccessFactors is **deferred**.

This document describes the content sources for the HR assistant and how each one is connected to Amazon Quick.

---

## 1. Summary

| Source | What it provides | How Quick uses it | Phase |
|---|---|---|---|
| **SharePoint (HR sites / policy pages)** | Documented HR **policies** (vacation, parental leave, volunteer time off, scholarships, etc.) | Native SharePoint integration → indexed knowledge base | **Phase 1, primary** |
| **HR-written knowledge documents** ("tribal knowledge") | "How do I…" answers HR currently gives ad hoc (e.g. company cars, interns) | Written by HR, stored in a dedicated SharePoint library and indexed with the rest | **Phase 1** |
| **SAP SuccessFactors** | Employee-specific data (leave balances, requests, holidays) | Custom action connector, queried live | ⏸ **Deferred** (v1 not personalised) |
| ~~Confluence~~ | Not mentioned by EBSCO HR as a content source | — | Not in scope |

---

## 2. SharePoint (Primary)

### 2.1 Steps
1. **Admin prerequisites:** Quick plan that supports SharePoint knowledge bases (Professional / Enterprise), and the Microsoft 365 / Entra approval required by the SharePoint integration. ⚠ Verify the exact requirements in the sandbox.
2. **Integrations → Add → Microsoft SharePoint (Online)** → authenticate and accept the permissions.
3. **Create knowledge base:** name it (e.g. `HR Policies`) and select **only the HR sites, libraries or folders** that hold approved HR content. Don't index the whole tenant.
4. Set the **sync schedule** (e.g. daily) and enable failure email notifications.
5. Link the knowledge base to a **Space**, then link the Space to the **agent**.

### 2.2 Things to watch
- **Scope tightly:** index only HR-approved locations, to avoid drafts, duplicates or outdated versions.
- **Permissions:** if document-level permissions are carried over, employees only get answers from content they can already see. Decide this at setup. ⚠ Verify the behaviour.
- **Source links:** citations should point to the SharePoint page employees normally use, so "answer + source link" works.
- **Several narrow knowledge bases** (e.g. one per domain) make future sub-agents easier.

---

## 3. HR-Written Knowledge Documents ("Tribal Knowledge")

Answers that today "live in HR's heads" need to be written down before the assistant can use them.

| Recommendation | Why |
|---|---|
| Store them in a **dedicated SharePoint library** (e.g. "HR Assistant Knowledge"), one short document per topic | Indexed by the same connector; links work; HR can edit them directly |
| Use a **simple template**: question(s) · answer · steps · contact · owner · last reviewed | Consistent answers; clear ownership |
| Assign an **owner per domain** and a **review date** | Keeps content current when processes change |
| Alternative for testing: **file uploads** to a Quick Space | Fast for a thin slice, but updates need re-uploading |

---

## 4. Analysing HR's Question List

When HR's question list (Excel) arrives, categorise each question:

| Category | Meaning | Action |
|---|---|---|
| **A. Covered by SharePoint** | The answer exists in a policy page | Make sure the page is in the indexed scope |
| **B. Needs new documentation** | Tribal knowledge; no written source | HR writes a knowledge document (template above) |
| **C. Unclear / out of scope** | Personalised, interpretive or ambiguous | Clarify with HR, or route to the manager / HR |

---

## 5. Sync Behaviour

- Content is **indexed ahead of time**, not at query time.
- Sync is set per knowledge base under **Sync Schedules** (Daily / Weekly / Monthly or **Sync now**). It can email on failure and has a maximum-deletion-percentage safeguard.
- Sync is **incremental**: only added, modified or deleted documents are reprocessed.
- File uploads to a Space are indexed immediately, but updates need re-uploading.

---

## 6. Deferred: SAP SuccessFactors

Employee-specific answers ("how many leave days do I have left?") would come live from SuccessFactors through a custom connector. That's out of Phase 1 because HR confirmed **no personalisation** for now. The API scope, permissions and design are kept as research for the future **Analyze / Act** stages.
