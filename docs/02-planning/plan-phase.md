# HR Assist — Planning Phase

> **Last updated:** 2026-10-14 (after HR requirements call #1)

This document records the current state of the HR Assist build, the options under consideration, and the questions that must be settled before moving on.

---

## 1. Current State

| Item | Status |
|---|---|
| Prototype agent | Built on a personal Free-tier Amazon Quick account (pre-requirements) |
| Prototype knowledge base | Google Drive (sample policies) |
| Prototype actions | Gmail escalation + Slack alert. **To be revisited:** EBSCO has no ticketing system and HR questions currently go through Teams and email. |
| Requirements | Call #1 held: **knowledge discovery, not personalised, no policy interpretation** |
| HR knowledge workbook | ✅ Received and analysed (about 90 Q&A rows, routing rules, always-escalate topics) |
| Sandbox agent | Behaviour-first agent (instructions + routing) built in the Enterprise sandbox; empty Space linked |

---

## 2. Phase 1 Scope

- **In scope:** finding HR policies and processes; quoting the relevant text with a source link; "I don't know" fallback; English (US); EBSCO branding.
- **Out of scope:** personalised answers (leave balances etc.), policy interpretation, actions on the employee's behalf.
- **Content:** SharePoint HR content + new HR-written "tribal knowledge" documents.

---

## 3. Access Channels (How Employees Reach the Assistant)

| Option | Description | Mechanism | Status |
|---|---|---|---|
| **SharePoint HR hub** | Assistant embedded on the HR homepage | SPFx web part + embed backend → registered-user embed URL → iframe | **HR leaning towards this.** Design documented. |
| **Microsoft Teams** | Quick chat inside Teams through the Teams extension | Okta OIDC → IAM Identity Center trusted token issuer → Secrets Manager + IAM role → extension access → M365 admin consent | Option. Design documented. |

⏳ **Decision owner:** HR. One channel or both.

---

## 4. Chatbot Platform

| Platform | Notes |
|---|---|
| **Amazon Quick** | Prototype platform; **under evaluation**. EBSCO is also evaluating Quick more widely (e.g. BI). |
| **Microsoft Copilot Studio** | **Under evaluation in parallel**: native to EBSCO's Microsoft 365 environment. Microsoft environment and solution architect to be arranged. |

---

## 5. Knowledge Base Direction

- **Prototype:** Google Drive (Free tier).
- **Phase 1:** **SharePoint**, indexing only HR-approved sites and libraries, plus a dedicated library for HR-written knowledge documents.
- **Approach:** categorise HR's questions into *covered by SharePoint* / *needs new documentation* / *unclear*.

---

## 6. Future Integrations (Deferred)

- **SAP SuccessFactors:** personalised answers and actions (Analyze / Act stages). Research complete (API scope, roles, OpenAPI vs MCP). Deferred until HR approves personalisation.
- **Outlook:** a draft-reply-for-review capability exists via the Outlook connector. Auto-reply isn't recommended.

---

## 7. Questions to Resolve

| # | Question | Owner | Status |
|---|---|---|---|
| 7.1 | Final question list and policy documents | HR working group | ✅ Received |
| 7.2 | Channel: SharePoint HR hub, Teams or both | HR | Open |
| 7.3 | Where quoting a policy ends and interpreting it begins | HR | Open |
| 7.4 | Fallback when unanswered (no ticketing system) | HR + UST | ✅ Routing rules from the workbook |
| 7.5 | Content owners per domain; who updates content when policies change | HR | Open |
| 7.6 | First thin-slice domain(s) | HR + UST | After 7.1 |
| 7.7 | Assistant name and branding | HR | Open |
| 7.8 | Enterprise sandbox access | Engagement lead / AWS admin | ✅ Granted |

---

## 8. Deployment Options (Summary)

1. **SharePoint HR hub embedding:** preferred by HR (pending confirmation).
2. **Microsoft Teams extension:** alternative or additional channel.

---

## 9. Next Steps

1. **HR documents:** HR shares the policy documents and FAQ list → UST analyses them → **discovery sessions are held based on that question list** (no discovery meetings until then).
2. **Quick proof of concept:** continue in the Enterprise sandbox. Microsoft 365 connector access depends on a corporate IT security review (ticket raised).
3. **Copilot Studio:** the environment is being arranged in parallel.
4. **Industry research (while waiting):** review AI-assisted HR solutions on the market (e.g. **Leena AI**) and product demos, to find feature ideas HR may not have considered.
5. **Workbook follow-ups:** upload the benefits PDFs to the Space; send one request to EBSCO IT (admin-managed Quick app registration with `Sites.Selected`, per-site read grants for all HR sites, organisation-wide consent for the permission-check app); HR refreshes outdated sources and moves OneDrive content to SharePoint; group questions into one knowledge base per topic; extend the routing table as the question list grows.
6. **Testing:** re-run the escalation scenarios once the instructions are final, and test per-employee SharePoint permissions before launch.
