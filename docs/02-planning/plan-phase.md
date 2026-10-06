# HR Assist — Planning Phase


This document records the current state of the HR Assist build, the options under consideration, and the questions that must be settled before moving on.

---

## 1. Current State

| Item | Status |
|---|---|
| Chat agent (HR Assist) | Built on a personal Free-tier Amazon Quick account |
| Knowledge base | **Google Drive** (connected through OAuth) |
| Actions | Gmail (employee-facing escalation) and Slack `#hr-escalations` (internal alert) |

---

## 2. Access Channels (How Employees Reach the Bot)

| Option | Description | Mechanism | Status |
|---|---|---|---|
| **Organisation / HR portal embed** | Embed the Quick chatbot in an existing employee portal (the org uses SharePoint) | Trusted identity federation → `GenerateEmbedUrlForRegisteredUser` → iframe; optional JIT user provisioning | Researched, **paused** |
| **Microsoft Teams** | Quick chat inside Teams through the Microsoft Teams extension | Okta OIDC app → IAM Identity Center trusted token issuer → Secrets Manager + IAM role → Quick extension access → M365 admin consent | **Current priority** |
| **Slack** | Install the Quick app in Slack | Quick's Slack app / extension | Idea only, not yet explored |

**Open question:** does the organisation have an HR or company portal that all employees can reach? The answer decides whether portal embedding is worth pursuing.

---

## 3. Chatbot Platform Options

| Platform | Notes |
|---|---|
| **Amazon Quick** | Current build platform. Native KB (Quick Index), Spaces, custom agents and action connectors. |
| **Microsoft Copilot Studio** | Alternative to evaluate. It fits naturally with Teams/SharePoint/M365. Not explored yet. |

---

## 4. Knowledge Base Direction

- **Now:** Google Drive (Free tier).
- **Target:** **Confluence**, indexing only the relevant HR pages/spaces rather than the whole site.
  - Scope it by pasting specific space or page URLs when you create the KB, or build several narrow KBs from the same source.

---

## 5. Potential System Integration

- **SAP SuccessFactors**: a possible HR system of record for employee-specific data (leave balances, employee records). This would let the agent answer questions that the system prompt currently tells it not to guess. The integration route (native connector, OpenAPI custom connector or MCP) has not been decided.

---

## 6. Questions to Resolve

| # | Question | Owner | Status |
|---|---|---|---|
| 6.1 | Sandbox access (Enterprise account) | Project lead / AWS admin | Open |
| 6.2 | Formal requirements | Stakeholders | Open |
| 6.3 | Confluence as KB: approve indexing only the relevant HR pages | HR / Confluence owners | Open |
| 6.4 | Is there an org-wide employee portal for embedding? | IT / HR | Open |
| 6.5 | Is SAP SuccessFactors in use, and can it be integrated? | HR systems team | Open |
| 6.6 | Platform choice: Amazon Quick vs Copilot Studio | Project lead | Open |

---

## 7. Deployment Options (Summary)

1. **Microsoft Teams integration**: in progress.
2. **SharePoint embedding**: paused until the Teams approach is validated.
