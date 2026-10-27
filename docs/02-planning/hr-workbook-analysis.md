# HR Assist — HR Knowledge Workbook Analysis

> **Source:** HR Knowledge Assistant SME Discovery workbook (Excel), shared by the HR team.
> **Purpose:** Explain what the workbook contains, how each part is used in the build, and the data-quality issues to fix before indexing.

---

## 1. Workbook Structure

| Sheet | Contents | Used for |
|---|---|---|
| **Master Question List** | About 90 fully worked Q&A rows (topic, question, answer, source link, handling type, owner, review status) | Knowledge base content + test set |
| **Backlog Questions** | About 18 questions not yet prioritised | Later phases |
| **Escalations** | Routing rules: which topics go to which HR role or team | **Agent instructions** |
| **Notes** | Always-escalate topic list; conflict / duplicate notes | **Agent instructions** + clean-up |
| **High-Value Opps** | Empty template | — |
| **Lists** | Dropdown reference values (topics, handling types) | Reference |
| **Testing Feedback** | Empty post-launch QA template | Testing |

---

## 2. Two Kinds of Content

The workbook mixes two fundamentally different things, which must be handled differently:

| Type | What it is | Where it goes |
|---|---|---|
| **Retrievable knowledge** | Q&A content tied to source documents | **Indexed** into the knowledge base |
| **Agent behaviour rules** | The Escalations sheet, the always-escalate list, and every row marked *Route to a person* or *Ask follow-up* | **Not indexed.** Translated into the agent's **instructions** as routing logic |

**Always-escalate topics (never answered by the assistant):** harassment, discrimination, retaliation, workplace conflict, ethics complaints, manager misconduct, payroll disputes, wage complaints, individual compensation concerns, policy exception requests.

---

## 3. Data-Quality Findings

| Finding | Detail | Action |
|---|---|---|
| **Fragmented sources** | One public website (`ebscochoice.com`), **10+ internal SharePoint site collections** (HR hub, learning, talent acquisition, compensation, information security, HR, manager resources, contractors, culture & inclusion, intranet), interactive web apps and personal OneDrive files | Plan sourcing per type |
| **Non-indexable sources** | Interactive web apps (e.g. HR / IT service portals) are logins and forms, not content | Link to them in answers; don't index |
| **Personal OneDrive files** | Tied to one person and fragile; one is explicitly labelled as having no source document | HR moves them into a shared SharePoint library first |
| **Outdated sources** | Several rows are marked *Source Needs Update = Yes* (e.g. intern manager orientation, onboarding hardware guide, internships page) | HR refreshes them **before** indexing |
| **Duplicates** | HR has cross-referenced near-identical questions (e.g. three wellness / EAP questions) | Merge into one answer per topic |

---

## 4. Recommended Next Actions

1. HR refreshes the sources flagged as outdated.
2. HR moves personal OneDrive content into a shared SharePoint library.
3. Merge the duplicate questions.
4. Group the questions by **Topic** to define one knowledge base per topic area.
5. Turn the Escalations and Notes sheets into agent instructions.
6. Use the Master Question List as the **approved answer set** for testing.
