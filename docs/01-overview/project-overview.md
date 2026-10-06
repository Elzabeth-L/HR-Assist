# HR Assist — Project Overview

> **What this is:** a plain-language explanation of the HR Assist project: the problem, the solution, how it works, where we are and what's next. Technical detail is included only where it helps understanding.
> **Last updated:** 2026-10-06
> **Status:** Prototype complete · Formal HR requirements pending

---

## 1. In One Paragraph

**HR Assist** is an AI-powered HR chatbot for EBSCO employees, built by UST on **Amazon Quick** (AWS's AI assistant platform). Employees ask HR questions in everyday language, inside tools they already use (**Microsoft Teams** first), and get instant answers drawn from the company's **official HR policies**. Where the answer depends on the employee's own record (e.g. a leave balance), the assistant will look it up in **SAP SuccessFactors**. When it can't answer confidently, it **hands the question to the HR team** instead of guessing.

---

## 2. The Problem We're Solving

> HR's formal requirements haven't been received yet. The problem statement below is our **working understanding** and will be refined once HR confirms it.

| Today's challenge | Impact |
|---|---|
| HR receives a high volume of **repetitive questions** (leave rules, holidays, WFH, benefits) | HR time goes into answering the same questions instead of higher-value work |
| HR policies are **spread across documents and pages** and hard to search | Employees can't find answers themselves, or find outdated versions |
| Answers depend on **location, employee type or tenure** | Generic answers are often wrong for the individual |
| Personal questions ("how many leave days do I have left?") need someone to **look up the HR system** | Delays for employees and more manual work for HR |
| No consistent way to **escalate** a question that needs human judgment | Questions get lost or answered inconsistently |

**In short:** employees need fast, accurate, personalised HR answers, and HR needs to spend less time on routine questions without losing control over sensitive or judgment-based cases.

---

## 3. Who It's For

| Group | Role in the project |
|---|---|
| **EBSCO employees** | **Users.** They ask questions and get answers. |
| **EBSCO HR team** | **Product owner.** Defines scope, owns policy content, receives escalations, approves behaviour. |
| **EBSCO IT / admins** | Provide access and approvals: Okta, AWS, Microsoft 365, SuccessFactors, Confluence. |
| **UST team** | Designs, builds, tests and deploys the solution. |

---

## 4. What We're Building

### 4.1 What HR Assist does

| Capability | Example | Data source | Status |
|---|---|---|---|
| **Answers policy questions** with a reference to the source | "How many casual leaves do I get per year?" | HR policy documents (**Confluence**) | Prototype built (on Google Drive) |
| **Answers personal HR questions** | "How many leave days do I have left?" / "Is my leave approved?" | **SAP SuccessFactors** (live lookup) | Planned: API scope verified |
| **Gives location-aware answers** | "Is next Monday a holiday?" for the employee's own location | SuccessFactors job data + holiday calendar | Planned |
| **Escalates to HR** | Questions it can't answer, exceptions, conflicting policies, explicit requests | Email to HR (Gmail) + internal HR alert (Slack) | Prototype built |
| **Takes simple actions** (later) | Submit a leave request, after the employee confirms | SuccessFactors | Future phase, if HR wants it |

### 4.2 What HR Assist will NOT do
- **Make up** policies, entitlements or personal data. If it doesn't know, it says so and offers to escalate.
- Make HR decisions (exceptions, approvals, grievances, legal or performance matters). These always go to a person.
- Show one employee's data to another.
- Access highly sensitive data such as **national ID numbers, passport details, bank details, salary or medical information**.

---

## 5. How It Works

### 5.1 The simple version

```
 Employee asks in Teams
        │
        ▼
 HR Assist (Amazon Quick) confirms who the employee is (company login, Okta)
        │
        ├──► Policy question  → searches the official HR policy documents → answers with a source reference
        ├──► Personal question → looks up the employee's own record in SuccessFactors → answers
        └──► Can't answer / needs HR → drafts an email to HR, asks the employee to confirm, sends it
        │
        ▼
 Answer appears in Teams
```

### 5.2 A bit more technical

| Component | What it is | Role in HR Assist |
|---|---|---|
| **Amazon Quick** | AWS's agentic AI workspace (formerly QuickSight / Quick Suite) | Hosts the HR Assist **custom chat agent**, its knowledge base and its actions |
| **Knowledge base (Quick Index)** | Managed search index of documents | HR policies from **Confluence** are split into passages, converted into searchable vectors and indexed **ahead of time**, then kept current by scheduled, incremental sync |
| **Retrieval-augmented generation (RAG)** | "Search first, then answer" | For each question, the most relevant policy passages are retrieved and given to the AI model, so answers are **grounded in company documents** and cite their source |
| **Amazon Bedrock** | AWS's managed AI model service | Generates the answer. The model is managed by Quick. |
| **System prompt (guardrails)** | The agent's standing instructions | Enforces: use HR documents as the source of truth, never invent policy or personal data, flag conflicts, escalate when unsure, confirm before sending anything |
| **Action connectors** | Integrations that let the agent *do* things | **Gmail** (escalation email, needs employee confirmation), **Slack** (anonymised internal alert to HR), **SuccessFactors** (planned, via a custom connector) |
| **SAP SuccessFactors connector** (planned) | A small AWS API layer (API Gateway + Lambda) in front of the SuccessFactors OData APIs | Returns **only the asking employee's** data and **only approved fields**. The employee's identity comes from their login, never from the AI. |
| **Okta + AWS IAM Identity Center** | Company login + AWS identity service | Confirms who the employee is, so there's no separate login and the right permissions apply |
| **Microsoft Teams extension** | Quick's official Teams app | Puts HR Assist inside Teams chat |

### 5.3 Why answers can be trusted
1. **Grounded:** answers come from retrieved HR documents, not the AI's general knowledge.
2. **Cited:** the source policy is referenced, so employees and HR can check it.
3. **Honest about gaps:** missing or conflicting information is stated openly and escalated.
4. **No guessing on personal data:** personal answers come only from SuccessFactors, for the signed-in employee.
5. **Human in the loop:** nothing is sent on the employee's behalf without their confirmation.
6. **Tested:** a structured test plan covers normal answers, escalation, confirmation, sensitive data, made-up information, duplicate escalations, conflicting documents and misuse of actions.

---

## 6. How Employees Will Access It

| Channel | Description | Status |
|---|---|---|
| **Microsoft Teams** | Employees chat with HR Assist directly in Teams. Sign-in is handled by the company login (Okta). | **Current priority.** Setup and approvals documented. |
| **SharePoint (company portal)** | HR Assist embedded on an HR page in SharePoint | **Paused** until Teams is validated. Design documented. |

**Known Teams limitation:** employees need to **message the bot directly** (not @mention it in channels) and select HR Assist once. Channel mentions use Quick's general assistant, and escalation actions only work in direct messages.

---

## 7. Delivery Approach

| Phase | Scope | Status |
|---|---|---|
| **0. Prototype** | HR Assist agent on a Free-tier Quick account; sample HR policies (Google Drive); Gmail + Slack escalation; guardrails; test plan | ✅ Done |
| **1. Requirements** | Discovery call with HR; confirm problem, users, scope, boundaries, escalation and success measures | ⏳ Pending: requirements not yet received |
| **2. Enterprise sandbox** | Move to Quick Enterprise; index HR policies from **Confluence**; run the test plan | ⏳ Waiting on sandbox access |
| **3. Teams pilot** | Deploy via the Teams extension to an HR pilot group | Planned |
| **4. SuccessFactors (read)** | Live personal answers: leave balances, leave requests, holidays, manager/HR contact | Planned: API scope verified |
| **5. Actions & rollout** | Optional leave submission (with confirmation), smarter escalation routing, wider rollout | Future |
| **Optional** | SharePoint embedding | Paused |

---

## 8. Current Status (as of 2026-10-06)

**Done**
- Working HR Assist prototype with grounded, cited answers and HR escalation (Gmail + Slack).
- Agent guardrails defined; 15-case test plan written.
- Teams and SharePoint deployment designs documented (configuration, permissions, approvals).
- Non-technical Teams flow diagram prepared.
- SuccessFactors API scope verified against the SAP API reference guide (needed APIs, user-mode permissions, authentication, excluded sensitive data).
- Requirement-discovery questions prepared for HR.

**Waiting on**
- **HR requirements:** the requirements call was delayed.
- **Amazon Quick Enterprise sandbox access.**
- Access to the **Confluence HR pages**, plus contacts for the **Okta, Microsoft 365 and SuccessFactors** admins.

---

## 9. Key Decisions

| Decision | Status |
|---|---|
| Build platform: **Amazon Quick** | Current choice. Microsoft Copilot Studio noted as an alternative, not evaluated. |
| Policy knowledge source: **Confluence** (indexed) | Recommended |
| Personal data source: **SAP SuccessFactors** (live, via custom connector) | Recommended; scope to be confirmed with HR and Security |
| First channel: **Microsoft Teams** | Agreed priority |
| SharePoint embedding | Paused |
| Sensitive data (national IDs, passport, bank, salary, medical) | **Excluded** |
| Agent writes (e.g. leave submission) | Deferred; always with employee confirmation |

---

## 10. Risks & Dependencies

| Risk / dependency | Impact | Mitigation |
|---|---|---|
| HR requirements delayed | Scope can't be finalised | Prototype and designs prepared in parallel; questions ready |
| Sandbox access pending | Can't test Confluence or Teams | Escalate the access request; Free-tier prototype in the meantime |
| Sensitive HR data in SuccessFactors | Privacy and compliance exposure | Minimal API scope, field allowlisting, self-only access, Security sign-off |
| Per-user Quick licensing | Cost grows with users | Pilot first; controlled rollout |
| Teams limitations (agent choice, DM-only actions) | User confusion | Clear user guidance; pre-pinned app |
| Outdated policy content | Wrong answers | HR owns content; scheduled sync; source citations |

---

## 11. How We'll Measure Success (proposed, to confirm with HR)
- Share of routine HR questions answered without HR involvement.
- Fewer repetitive questions reaching the HR inbox.
- Accuracy against the test plan, and HR spot-checks of real answers.
- Employee satisfaction with answers.
- Escalations reach the right HR contact, with nothing lost.

---

## 12. Glossary

| Term | Meaning |
|---|---|
| **Agent** | The HR Assist chatbot configured in Amazon Quick |
| **Knowledge base** | The searchable collection of HR policy documents the agent answers from |
| **Space** | A Quick container that groups knowledge for an agent |
| **RAG** | Retrieval-augmented generation: search the documents first, then answer from them |
| **Action / connector** | An integration that lets the agent do something (send an email, look up a record) |
| **Escalation** | Handing a question to a human in HR |
| **SuccessFactors** | SAP's HR system of record (employee, job and time-off data) |
| **Okta / IAM Identity Center** | Company login and the AWS service that trusts it |

---

## 13. Change Log

| Date | Change |
|---|---|
| 2026-10-06 | First version. Problem statement is a working assumption pending HR requirements. |
| 2026-10-06 | SuccessFactors API scope verified against the SAP API reference guide (2H 2026). Key finding: use User Mode (self-only) permissions, not Admin Mode. |
