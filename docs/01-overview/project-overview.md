# HR Assist — Project Overview

> **What this is:** a plain-language explanation of the HR Assist project: the problem, the solution, how it works, where we are and what's next. Technical detail is included only where it helps understanding.
> **Last updated:** 2026-10-14
> **Status:** Requirements call #1 held · Phase 1 scope agreed · Enterprise sandbox available · Waiting on HR's question list

---

## 1. In One Paragraph

**HR Assist** (final EBSCO name to be chosen by HR) is an AI-powered **HR knowledge assistant** for EBSCO employees, built by UST. Employees ask HR questions in everyday language and get the answer straight away: the relevant text from HR's own content, plus a link to the source. Phase 1 is about **discovery**: helping people find information that already exists but is hard to locate. It is **not personalised** and doesn't interpret policy. Later phases may add personalised answers and actions, once HR approves. The prototype and current design use **Amazon Quick** (AWS); the final platform choice is subject to confirmation.

---

## 2. The Problem We're Solving

*Confirmed on HR requirements call #1.*

| Today's challenge | Impact |
|---|---|
| Employees ask HR the **same questions** repeatedly (vacation policy, volunteer time off, interns, company cars, parental leave, scholarships) | HR time goes on routine questions |
| Answers mostly **exist on SharePoint** but take **too many clicks** to find | Employees don't search, or don't find the answer |
| Some answers are **undocumented "tribal knowledge"** | Only certain HR people can answer them |
| Questions arrive **ad hoc via Teams, email or phone**, often to any HR person the employee knows | Inconsistent answers; questions get forwarded around |
| **HR business partners** (one per department) receive many small questions | Distracts them from consulting on bigger issues |
| **No ticketing system** | No tracking of HR questions |

**In short:** employees need one easy place to find HR information, and HR needs to stop answering the same questions one by one.

---

## 3. Who It's For

| Group | Role in the project |
|---|---|
| **EBSCO employees** | **Users.** They ask questions and find answers. |
| **EBSCO HR team** (HR leads and the HR working group) | **Product owner.** Defines scope, provides the question list and content, decides the channel and branding, owns content accuracy. |
| **HR subject-matter experts** | Own and test the content for their domains |
| **Engagement lead** | Overall engagement and coordination between EBSCO, UST and AWS |
| **AWS and solution architecture advisors** | Platform guidance and solution patterns |
| **UST team** | Designs, builds, tests and deploys the solution |

---

## 4. What We're Building

### 4.1 Phase 1: what the assistant does

| Capability | Example | Source |
|---|---|---|
| **Finds the right policy or process** | "Where can I find the vacation policy?" / "How do I use volunteer time off?" | HR policies on **SharePoint** |
| **Answers "how do I" questions** that were never written down | "How do I get a company car when visiting the office?" | New HR-written knowledge documents |
| **Quotes the relevant text and links to the source** | Shows the policy section plus the SharePoint link | Citations from the retrieved document |
| **Says when it doesn't know** | Points to the manager or HR instead of guessing | Guardrails in the agent's instructions |

### 4.2 What it will NOT do in Phase 1
- **Personalised answers:** no leave balances or employee records.
- **Interpret policy:** it points to and quotes the policy; employees discuss interpretation with their manager or HR.
- **Make things up:** it only answers from HR's content.
- **Act on the employee's behalf** (e.g. apply for leave). That's a possible future stage.
- **Access sensitive data** (national IDs, passport, bank, salary, medical).

---

## 5. How It Works

### 5.1 The simple version

```
 Employee asks a question (SharePoint HR hub and/or Teams)
        │
        ▼
 EBSCO HR Assistant
        │  searches only HR's approved content
        ├──► Found      → relevant policy text + link to the source
        └──► Not found  → says so and points to the manager / HR (fallback design TBD)
```

### 5.2 A bit more technical (current design, based on Amazon Quick)

| Component | What it is | Role |
|---|---|---|
| **AI assistant platform** (currently **Amazon Quick**) | Agentic AI workspace that hosts chat agents, knowledge bases and actions | Hosts the HR assistant's agent and its knowledge base |
| **Knowledge base / search index** | Managed search index | HR content from **SharePoint** (plus HR-written documents) is split into passages, indexed **ahead of time** and kept current by scheduled, incremental sync |
| **Retrieval-augmented generation (RAG)** | "Search first, then answer" | The most relevant passages are retrieved and given to the AI model, so answers are **grounded** in HR content and **cite** their source |
| **Scoped to HR content only** | Platform setting / instructions | The agent doesn't use the AI's general knowledge for answers |
| **System prompt (guardrails)** | The agent's standing instructions | Use HR content as the only source; quote, don't interpret; never invent; say "I don't know" |
| **AI model service** (currently **Amazon Bedrock**) | Managed large language models | Generates the answer |

### 5.3 Why answers can be trusted
1. **Grounded and cited:** every answer comes from HR content and links to it.
2. **No interpretation:** it quotes the policy rather than explaining it.
3. **Honest about gaps:** it says when it doesn't know.
4. **Tested against an approved answer set:** HR's own questions and approved answers.
5. **Feedback loop:** thumbs up/down from users; HR subject-matter experts test each domain.
6. **Pilot first:** a small group before wider rollout.

---

## 6. How Employees Will Access It

| Channel | Description | Status |
|---|---|---|
| **SharePoint HR hub** | Assistant embedded on the HR homepage in SharePoint | **Option HR leans towards.** Design documented. |
| **Microsoft Teams** | Employees chat with the assistant in Teams (company login via Okta) | **Option.** Setup and approvals documented. |

⏳ **HR will decide** where it lives: one channel or both.

---

## 7. Roadmap & Delivery Approach

### 7.1 Discover → Analyze → Act

| Stage | Meaning | Example | Status |
|---|---|---|---|
| **1. Discover** | Find information accurately, with citations | "Where is the vacation policy?" | **Phase 1 (current)** |
| **2. Analyze** | Combine policy with the employee's situation | "What's my vacation balance, and which leave applies?" | Future: needs HR approval, governance and an HR-system integration (e.g. SuccessFactors) |
| **3. Act** | Act on the employee's behalf | "Submit that leave request for me" | Future |

### 7.2 How Phase 1 is delivered

| Step | Scope | Status |
|---|---|---|
| 0. Prototype | Free-tier agent, sample policies, escalation, guardrails, test plan | ✅ Done |
| 1. Requirements | Call #1: scope agreed | ✅ Done |
| 2. Environment | Enterprise sandbox account | ✅ Available |
| 3. Question list | HR shares the final questions (Excel) and documents | ⏳ Pending |
| 4. Analysis | Categorise questions: answered by SharePoint / needs new documentation / unclear | Next |
| 5. Thin slice | Prove the assistant on about five questions in one or two domains | Planned |
| 6. Expand by domain | HR experts test each domain while the next is built; sub-agents per domain if needed | Planned |
| 7. Pilot → rollout | Small employee group, then all employees | Planned |

---

## 8. Current Status (as of 2026-10-14)

**Done**
- Free-tier prototype (grounded, cited answers; escalation; test plan).
- Requirements call #1 held: Phase 1 scope agreed.
- **Enterprise sandbox access received.**
- SharePoint and Teams deployment designs documented.
- Research completed: Outlook integration, Quick Desktop scope, sub-agent architecture, and the SuccessFactors API scope (deferred).

**Waiting on**
- HR's **final question list** and documents (after HR's internal review).
- HR's decision on the **channel** and **branding**.

---

## 9. Key Decisions

| Decision | Status |
|---|---|
| Build platform | Prototype and current design on **Amazon Quick**; final platform **not yet confirmed** |
| Phase 1 = **knowledge discovery**, **not personalised** | ✅ Agreed with HR |
| **No policy interpretation**: quote the relevant text + link to the source | ✅ Agreed (where the line sits is to be confirmed) |
| Content: **SharePoint** + new HR-written documents | ✅ Agreed |
| **English (US) first** | ✅ Agreed |
| **EBSCO-branded** assistant | ✅ Agreed (name TBD by HR) |
| Channel: SharePoint HR hub vs Teams | ⏳ HR to decide |
| Fallback when unanswered (no ticketing system) | ⏳ To design |
| SuccessFactors / personalisation | ⏸ Deferred |
| Sub-agents per domain | Proposed for later; after the question list |
| Sensitive data (national IDs, passport, bank, salary, medical) | **Excluded** |

---

## 10. Risks & Dependencies

| Risk / dependency | Impact | Mitigation |
|---|---|---|
| Question list delayed | Can't scope the thin slice | Designs and prototype ready; analysis approach agreed |
| Content gaps (undocumented knowledge) | Assistant can't answer | HR writing new documents; questions categorised by coverage |
| No clear content owner | Answers go stale after policy changes | Define owners per domain; scheduled sync |
| Channel undecided | Deployment work can't start | Both SharePoint and Teams designs documented |
| Platform not confirmed | Rework if a different platform is chosen | Keep content, question sets and guardrails platform-neutral |
| Quoting vs interpreting is unclear | Answers may overstep HR's intent | Confirm the rule with HR; enforce it in the system prompt and tests |

---

## 11. How We'll Measure Success (proposed, to confirm with HR)
- Accuracy against HR's approved question-and-answer set.
- Share of questions answered with a correct source link.
- Fewer routine questions reaching HR business partners.
- Positive user feedback (thumbs up / down).

---

## 12. Glossary

| Term | Meaning |
|---|---|
| **Agent** | The HR assistant configured in the AI platform |
| **Knowledge base** | The searchable collection of HR content the agent answers from |
| **Tribal knowledge** | HR answers not written down anywhere yet |
| **RAG** | Retrieval-augmented generation: search the content first, then answer from it |
| **Thin slice** | A small first scope (a few questions, one domain) used to prove the approach |
| **Sub-agent** | A specialised agent for one domain (e.g. benefits), called by a main routing agent |
| **SuccessFactors** | SAP's HR system of record (deferred for Phase 1) |

---

## 13. Change Log

| Date | Change |
|---|---|
| 2026-10-06 | First version. Problem statement was a working assumption pending HR requirements. |
| 2026-10-06 | SuccessFactors API scope verified against the SAP API reference guide (2H 2026). |
| 2026-10-14 | Updated after HR requirements call #1. Scope is now **knowledge discovery, not personalised, no policy interpretation**; content is **SharePoint + HR-written documents**; channel is open (SharePoint HR hub vs Teams); SuccessFactors **deferred**; Discover → Analyze → Act roadmap added. |
| 2026-10-14 | Enterprise sandbox access received. Platform described as the current design (not final). Personal names replaced with roles. |
