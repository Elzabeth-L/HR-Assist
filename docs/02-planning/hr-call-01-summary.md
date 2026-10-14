# HR Assist — HR Requirements Call #1: Summary & Decisions

> **Meeting:** First requirements discussion with the EBSCO HR team
> **Date:** ⚠ Confirm (week of 2026-10-05)
> **Attendees:** EBSCO HR leads · Engagement lead (RM) · AWS representative · Solution architect · UST team
> **Source:** AI-generated Teams transcript (contains gaps; see [§8](#8-transcript-gaps))

---

## 1. What HR Wants

A **knowledge-discovery chatbot** for questions whose answers already exist but take too many clicks to find.

**Examples:** where is the vacation policy? · next steps for hiring a summer intern · how to use volunteer time off · how to get a company car when visiting the office · parental leave policy · scholarships for employees' children · accessing employment documents.

**Content sources:**
1. **SharePoint**: documented policies.
2. **"Tribal knowledge"**: answers HR currently gives ad hoc over Teams, now being written down by HR for the first time.

**Today:** employees ask over **Teams, email or phone**, often whichever HR person they know. Every department has an **HR business partner** who gets many small questions and often forwards them. There's **no ticketing system**.

---

## 2. Phase 1 Scope Decisions

| # | Decision | Source |
|---|---|---|
| D1 | **Not personalised.** No employee-specific answers such as vacation balances, because of the risk. | HR; confirmed by AWS |
| D2 | **No policy interpretation.** Point people to the policy; employees discuss it with their manager, who can contact HR. | HR |
| D3 | **Answer + source:** show the specific relevant policy text **and** link to the source page. | HR, AWS |
| D4 | **Start where no employee data is needed**, so HR's internal discussions on data access don't hold things up. | HR |
| D5 | **English (US) first**; multilingual later. | HR |
| D6 | **Personalisation stays open for later**, subject to governance, security and the region of processing. | Engagement lead |

> **Implication:** SuccessFactors work is **deferred**.

---

## 3. Roadmap: Discover → Analyze → Act

| Stage | Meaning | Example |
|---|---|---|
| **Discover** *(v1)* | Find the right information, with citations | "Where is the vacation policy?" |
| **Analyze** *(future)* | Combine policy with the employee's situation | "What's my balance, and which leave type applies?" |
| **Act** *(future)* | Act on the employee's behalf | "Submit that leave request for me" |

Same interface throughout, with more intelligence added over time. **Time and absence** was suggested as the first, low-risk area for personalisation later.

---

## 4. Delivery Approach

- **Thin slice first:** prove it on about five questions in one narrow domain, then expand.
- **One or two domains at a time**, tested by HR subject-matter experts while the next domain is built.
- **Sub-agents per domain** (e.g. benefits, leave) with a routing agent. To be proposed once HR's question list arrives.
- **Quality:** an approved question-and-answer set to compare against, thumbs up/down feedback, guardrails ("don't make things up, say I don't know"), restricting Quick to the documents only, and a beta group before wider rollout.
- **Fallback idea:** gather the context and hand off to a human, so the employee doesn't have to repeat themselves.

---

## 5. Channel, Branding & Ownership

| Topic | Outcome | Status |
|---|---|---|
| **Where it lives** | HR assumed the **SharePoint HR hub**; Teams is also possible. HR decides. | ⏳ Open |
| **Branding** | EBSCO-branded HR assistant, not shown as "Quick". "Jarvis" was rejected. | HR to name |
| **Ownership** | Define content owners (one HR owner vs domain experts), agent-configuration owner and deployment owner. Someone must be accountable for updating content when policies change. | ⏳ Open |
| **Content curation** | Solution architect to guide whether content stays in place or is consolidated into a dedicated space | ⏳ Open |

---

## 6. Impact on Our Plan

| Area | Before the call | After the call |
|---|---|---|
| Personal data / SuccessFactors | Planned | **Deferred** |
| Knowledge source | Confluence | **SharePoint + new HR-written documents** |
| Answer style | Answer policy questions | **Quote the relevant text + link to the source**; no interpretation |
| Channel | Teams first | **Open:** HR leans towards the SharePoint HR hub |
| Escalation | Gmail + Slack | **Redesign:** no ticketing system; HR business partners use Teams and email |
| Actions (apply leave) | Optional Phase 2 | Future **"Act"** stage |
| Architecture | Single agent | Single agent for v1; sub-agents later |

---

## 7. Next Steps & Open Questions

| # | Action | Owner | Status |
|---|---|---|---|
| A1 | Share the final question list (Excel) and policy documents after HR's internal Tuesday meeting | HR | ⏳ Pending |
| A2 | Categorise questions: answered by SharePoint / needs new documentation / unclear | UST + solution architect | After A1 |
| A3 | Propose the first thin slice and the domain / sub-agent structure | UST + solution architect | After A2 |
| A4 | Set a regular meeting cadence (before noon US Eastern) | Engagement lead | After A1 |

**Open questions for HR:** where the assistant lives (SharePoint, Teams or both) · the line between quoting and interpreting a policy · content owners per domain · where unanswered questions go · which domain to start with · name and branding · testers and pilot users.

---

## 8. Transcript Gaps

Missing sections (approximate): 23:04–25:49 (channel), 30:26–34:27 (branding / language), 39:34–40:43 (sub-agents), 46:21–46:39, 51:08–52:21 (ownership), 54:21–56:21 (curation), 59:34–1:02:33. Confirm any decisions from these sections with the attendees.
