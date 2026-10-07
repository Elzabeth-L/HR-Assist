# HR Assist — AI HR Chatbot on Amazon Quick

An AI-powered HR assistant that answers employee HR questions instantly, inside Microsoft Teams, using the company's official HR policies and HR system. Built by UST for EBSCO on **Amazon Quick** (AWS).

> **Status (2026-10-06):** Prototype complete · Waiting on HR requirements and Enterprise sandbox access

---

## The Problem

- HR spends a lot of time answering the **same questions** about leave, holidays, WFH, benefits and payroll.
- Policies are **spread across documents** and hard for employees to find. Answers often depend on location or employee type.
- Personal questions ("how many leave days do I have left?") need someone in HR to **look up the HR system**.
- There's no consistent way to **route** questions that need human judgment.

*(Working understanding. It will be refined once HR shares formal requirements.)*

## The Solution

**HR Assist**, a custom chat agent on Amazon Quick that:

| | How |
|---|---|
| **Answers policy questions** with source references | Searches HR policy documents indexed from **Confluence** (retrieval-augmented generation) |
| **Answers personal questions** *(planned)* | Looks up the asking employee's own record in **SAP SuccessFactors**, live and limited to approved fields |
| **Escalates to HR** when unsure | Drafts an email to HR, sends it only after the employee confirms, and posts an anonymised alert to HR's Slack channel |
| **Lives where employees work** | **Microsoft Teams** (priority); SharePoint embedding as a later option |
| **Stays safe** | Never invents policy or personal data; no access to national IDs, bank, salary or medical data; company login (Okta) for identity |

```
Employee (Teams) → HR Assist (Amazon Quick) ─┬─► HR policies (Confluence)        → cited answer
                                             ├─► Employee record (SuccessFactors) → personal answer
                                             └─► Escalation (Gmail + Slack)       → HR follows up
```

## Status

| Workstream | Status |
|---|---|
| Prototype: agent, knowledge base, Gmail + Slack escalation, test plan | ✅ Done |
| HR requirements | ⏳ Pending |
| Enterprise sandbox + Confluence knowledge base | ⏳ Waiting on access |
| Microsoft Teams deployment | Designed, current priority |
| SAP SuccessFactors integration | API scope verified against SAP API reference |
| SharePoint embedding | Designed, paused |

## Repository Guide

| Folder | What's inside |
|---|---|
| [docs/01-overview](docs/01-overview/) | **[Project overview](docs/01-overview/project-overview.md)** (start here) · [Amazon Quick reference](docs/01-overview/amazon-quick-reference.md) (platform research and the agent's system prompt) |
| [docs/02-planning](docs/02-planning/) | [Plan](docs/02-planning/plan-phase.md) · [Open questions](docs/02-planning/open-questions.md) · [HR requirements questions](docs/02-planning/hr-requirements-questions.md) |
| [docs/03-knowledge-base](docs/03-knowledge-base/) | [Data sources: Confluence & SuccessFactors](docs/03-knowledge-base/kb-source-options.md) |
| [docs/04-deployment](docs/04-deployment/) | [Teams guide](docs/04-deployment/teams.md) · [Teams flow diagram](docs/04-deployment/teams-architecture.html) · [SharePoint guide](docs/04-deployment/sharepoint.md) |
| [docs/05-testing](docs/05-testing/) | [Test plan](docs/05-testing/test-plan.md) |
| [docs/06-integrations](docs/06-integrations/) | [SuccessFactors API scope](docs/06-integrations/successfactors-api-scope.md) · [SuccessFactors access & roles](docs/06-integrations/successfactors-access-roles.md) |
| [draft](draft/) | Original raw notes |
