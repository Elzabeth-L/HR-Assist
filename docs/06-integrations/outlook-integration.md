# HR Assist — Outlook Integration: Auto-Reply Feasibility & Draft Capability

> **Purpose:** Assess whether Amazon Quick can (a) **automatically reply** to emails arriving in an HR mailbox, or (b) **draft replies** for a human to review and send.
> **Status:** Research. Not in v1 scope. Verify in the Enterprise sandbox before relying on it.

---

## Contents

- [1. Summary](#1-summary)
- [2. Auto-Reply to an HR Mailbox](#2-auto-reply-to-an-hr-mailbox)
- [3. Draft-Email Capability](#3-draft-email-capability)
- [4. Connector vs Extension](#4-connector-vs-extension)
- [5. Recommendation](#5-recommendation)
- [6. Verification Steps (Sandbox)](#6-verification-steps-sandbox)
- [7. Open Questions](#7-open-questions)

---

## 1. Summary

| Capability | Feasible? | How | Fit for HR's risk stance |
|---|---|---|---|
| **Fully automatic reply** to emails in an HR mailbox | Not natively. It needs custom infrastructure. | Microsoft Graph webhook → Lambda → Quick agent → reply via Graph | ❌ Poor fit: no human review |
| **Draft reply** for a human to review and send | ✅ Yes (documented) | Outlook **action connector** used by a chat agent | ✅ Good fit: human in the loop |

---

## 2. Auto-Reply to an HR Mailbox

### 2.1 Findings

| Source | Finding | Confidence |
|---|---|---|
| Official Quick Automate documentation | Only **two trigger types**: a predefined **schedule**, or programmatic invocation via the **`StartAutomationJob` API**. There's **no documented "new email arrived" trigger** (unlike Microsoft Power Automate). | Confirmed |
| Third-party (non-AWS) guide | Mentions an "event-based" trigger for **Agents** (not Automate) when new documents land in a Space or a connector receives new data | ⚠ Unverified; not clearly applicable to email |

### 2.2 What a true auto-reply would require

With no native email trigger, event-driven auto-reply needs a **custom bridge**:

```
New email in HR mailbox
   │  Microsoft Graph change notification (webhook subscription on the mailbox)
   ▼
AWS Lambda
   │  Calls the Quick agent programmatically
   ▼
Amazon Quick agent (drafts the answer)
   │
   ▼
Reply sent via Microsoft Graph / Outlook connector
```

This is real, buildable infrastructure (Graph subscription and renewal, Lambda, permissions to read and send from the mailbox). It isn't a Quick configuration setting.

### 2.3 Risk assessment

- A fully autonomous reply (no human review) **conflicts with HR's stance** from the requirements call: "considering the risks", "not personalised".
- Emails to HR may contain personal or sensitive details, which raises data-handling concerns for automated processing.
- **Not recommended** for this project phase.

---

## 3. Draft-Email Capability

### 3.1 Findings

AWS's official Outlook integration guide: *"When you connect Quick to Outlook, it summarizes long email threads, drafts replies with relevant context, schedules meetings, and starts downstream workflows."*

### 3.2 How it would work

1. An HR team member, in a Quick chat, asks: *"Draft a reply to this HR email about X."*
2. The agent uses the **Outlook action connector** to read the context and create a draft.
3. A **real draft appears in the Drafts folder**.
4. The HR team member **reviews, edits and sends** it themselves.

This gives **human-in-the-loop** by design, with **no new infrastructure**.

### 3.3 Not confirmed

- Whether Quick's Outlook **extension** offers Microsoft Copilot-style **inline drafting inside an open compose window**. Only the **Drafts-folder** pattern above is confirmed. Don't assume inline-compose parity.

---

## 4. Connector vs Extension

| | **Outlook action connector** | **Outlook extension** |
|---|---|---|
| What it is | Lets a Quick agent **act in Outlook** (read threads, draft replies, schedule meetings) | Puts the **Quick chat inside Outlook** |
| Question it answers | *What can Quick do?* | *Where does a person access Quick?* |
| Needed for drafting? | ✅ Yes | ❌ No. Drafting works from any chat surface (Quick web, Teams, etc.). |

---

## 5. Recommendation

- **Don't build auto-reply** in this phase.
- If HR wants email support, offer the **draft-for-review** pattern to the HR team (e.g. HR business partners drafting answers to common questions), using the Outlook action connector.
- Revisit event-driven automation only if HR explicitly asks for it and approves the risks.

---

## 6. Verification Steps (Sandbox)

1. **Automate triggers:** Amazon Quick Automate → create or open a **Project** → **Deployment** page → **Create Trigger** → *Step 1: Set Trigger*. Record which trigger options actually appear.
2. **Draft capability:** connect the Outlook action connector to a test agent → ask it to draft a reply to a test email → confirm the draft lands in **Drafts** and nothing is sent automatically.
3. **Action permissions:** confirm the "send" action (if any) can be set to **Always Ask**, or disabled.

---

## 7. Open Questions

- [ ] **Which mailbox?** A connector reads the mailbox of **whoever authenticates**. An HR-inbox use case would need HR's own account or a shared HR mailbox, not a project team member's mailbox.
- [ ] **Security approval:** connecting Quick to Microsoft 365 (mail, Teams) needs a corporate IT security review, covering what data is sent to AWS and the read/write scope.
