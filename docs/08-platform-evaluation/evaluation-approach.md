# HR Assist — Platform Evaluation Approach

> **Purpose:** Explain why HR Assist is being evaluated on two platforms, **Amazon Quick** and **Microsoft Copilot Studio**, and how the comparison will be run.
> **Status:** In progress. A recommendation will be shared with HR **only after** both platforms have been tested hands-on.

---

## 1. Why Two Platforms

- EBSCO uses **AWS** as its cloud partner (server migration) but runs its **workplace tools on Microsoft**: Teams, SharePoint and Outlook.
- Amazon Quick has been the prototype platform. Copilot Studio is native to the Microsoft 365 environment where EBSCO's HR content and employees already are.
- The two platforms offer similar services on paper. The real differences (effort, cost, answer quality) only show up in hands-on testing.

---

## 2. What Stays the Same on Either Platform

These are **platform-neutral** and carry over unchanged:

- HR's requirements: knowledge discovery, not personalised, no policy interpretation, cited answers, English (US) first
- Content: SharePoint policies + HR-written knowledge documents
- HR's question list and the approved answer set used for testing
- Guardrail rules (answer only from HR content, quote and link, say "I don't know")
- The Discover → Analyze → Act roadmap

Only the **build, deployment and integration** layers are platform-specific.

---

## 3. Hypotheses to Test

| # | Hypothesis (favouring Copilot Studio) | How to test |
|---|---|---|
| H1 | Microsoft's built-in understanding of work context (Work IQ) gives agents context with less connector work | Review the setup effort; note that Work IQ is in preview and not needed for Phase 1 |
| H2 | It uses fewer tokens / costs less than pulling data in through connectors | Compare usage and cost for the same questions |
| H3 | Sign-in and Microsoft 365 connections are simpler | Compare the setup effort for SharePoint and Teams on both |
| H4 | Guardrails are easier to set up within one ecosystem | Configure the same guardrail rules on both |
| H5 | There's a wider model choice (incl. Anthropic Claude) | Check which models each sandbox actually offers |

---

## 4. Method

1. **Same thin slice on both:** about five HR questions from one domain, over the **same SharePoint documents**.
2. **Same guardrail rules** and the same instructions, as far as each platform allows.
3. **Test against HR's approved answers.** Record accuracy, citations and "I don't know" behaviour.
4. **Deploy to the same channel** (SharePoint HR hub and/or Teams), and record the effort and approvals needed.
5. **Score both** using the sheet below, and summarise the trade-offs.

### 4.1 Quick Proof of Concept (in progress)

While Copilot Studio access is being arranged, a proof of concept is being built in the **Amazon Quick Enterprise sandbox**, so both platforms can later be demonstrated side by side.

| PoC focus | Detail |
|---|---|
| **Teams first** | Can the Quick agent be deployed to, and chatted with in, Teams? What Microsoft 365 data can it access? |
| **SharePoint embedding** | Not part of the PoC: it needs custom SharePoint code, which isn't a suitable proof-of-concept scenario |
| **Outlook** | Not required for the chatbot itself; only relevant to a possible email use case |
| **Built-in connectors only** | Microsoft 365 connectors rather than custom connectors (e.g. SuccessFactors) |
| **Personal-account test** | A personal Gmail account connected to a Free-tier Quick account could read, send and receive mail through the Quick interface. The Outlook connector is expected to behave similarly once access is enabled. |

**Progress so far:**
- Behaviour-first agent built (instructions, routing, escalation) and tested; vague-concern handling refined.
- Empty Space created and linked to the agent.
- Web Crawler on the public benefits site was blocked (HTTP 403), so those PDFs will be uploaded manually.
- SharePoint needs **admin-managed** setup (document-level permissions) and per-site app grants.

---

## 5. Scoring Sheet

Score each criterion 1–5 for each platform.

| Criterion | What to check | Quick | Copilot Studio |
|---|---|---|---|
| Answer quality | Accuracy vs approved answers; correct citations; "I don't know" behaviour | | |
| SharePoint knowledge | Setup effort, sync, user-permission trimming | | |
| Deployment | Effort and approvals to reach the SharePoint HR hub / Teams | | |
| Guardrails | Answering only from content, no web, "quote, don't interpret" | | |
| Security & governance | Audit, data policies, data residency | | |
| Model choice | Models available and how they're enabled | | |
| Cost | Licensing model + usage for the same questions | | |
| Admin & ownership | Ease for HR content owners to maintain | | |
| Outlook support | Drafting for review; triggers | | |
| Future roadmap | Ease of the Analyze / Act stages (HR-system integration, actions) | | |

---

## 6. Environment Needs

| Item | Amazon Quick | Copilot Studio |
|---|---|---|
| Sandbox / environment | ✅ Enterprise sandbox available | ⏳ To be arranged with Microsoft |
| Solution architect | ✅ Engaged (to be involved once connector access is enabled) | ⏳ To be arranged |
| Model access | Models abstracted by Quick | Admin must enable Anthropic and external models for a full model choice |
| Microsoft 365 connector | ⏳ **Blocked pending corporate IT security review.** The sandbox can't yet connect to Teams, Outlook or SharePoint. Enabling the Microsoft 365 connector needs a security ticket and review (including what data is sent to AWS). | Native to Microsoft 365 |

---

## 7. Open Questions

- [ ] When will the Microsoft environment be available, and which tenant (trial, EBSCO dev tenant, or partner-provided)?
- [ ] Can both sandboxes use real EBSCO SharePoint HR content, or only sample documents?
- [ ] Does the Quick track continue at full pace in the meantime?
- [ ] How long does the evaluation run, and who makes the final decision?
- [ ] Does the evaluation affect any timeline HR expects?
- [ ] When will the security review of the Quick ↔ Microsoft 365 connector be completed?
