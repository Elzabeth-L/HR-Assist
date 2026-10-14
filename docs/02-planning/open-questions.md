# HR Assist — Open Questions & Clarifications


This document lists the questions raised at the start of the HR Assist project. It groups them by theme and tracks which ones have been answered.

---

## 1. Project & Engagement Questions

These are for the project sponsor or lead to answer. The platform documentation can't answer them.

| # | Question | Status | Notes |
|---|---|---|---|
| 1.1 | Platform confirmation: are we building on **Amazon Quick**? | Confirmed | Build is on Amazon Quick (formerly QuickSight / Quick Suite). |
| 1.2 | Are the expectations for the engagement clear? | Answered | Phase 1 = knowledge discovery, not personalised, no policy interpretation (HR call #1). Success criteria still to agree. |
| 1.3 | Which team will we be working with? | Answered | EBSCO HR (HR leads and the HR working group), the engagement lead (RM), AWS and a solution architect. |
| 1.4 | Who do we contact for requirements? | Answered | The EBSCO HR leads. |
| 1.5 | Who do we contact for general queries? | Answered | The engagement lead (RM). |
| 1.6 | What is the goal for the next connect? | Answered | Review HR's question list; propose the first thin slice and domain structure; agree the channel. |

---

## 2. Technical Questions

| # | Question | Status | Where it is answered |
|---|---|---|---|
| 2.1 | **How do we build it?** | Answered | Knowledge base → Space → custom chat agent → action connectors. |
| 2.2 | **How do we deploy it?** | Partly answered | SharePoint HR hub embedding and the Teams extension are both documented. HR is leaning towards SharePoint; HR decides. |
| 2.3 | **How do we keep the knowledge base up to date?** | Answered | Sync is a knowledge-base setting (Daily/Weekly/Monthly or manual "Sync now"). It is incremental. Files uploaded straight to a Space index immediately and must be re-uploaded to update. |
| 2.4 | **What kinds of data sources can we bring in?** | Answered | Quick supports S3, Confluence, SharePoint, OneDrive, Google Drive, Web Crawler and file uploads. **Phase 1 uses SharePoint + HR-written documents.** |
| 2.5 | **How can users reach the chatbot from other apps without logging in to Quick? (authentication)** | Answered (design) | Portal: trusted identity federation plus embed URLs, with optional JIT user provisioning. Teams: Okta → IAM Identity Center trusted token issuer → Quick extension. |
| 2.6 | **How does the pipeline work behind the scenes?** | Answered | Ingestion → chunking → embedding → indexing in Quick Index → retrieval at query time → Bedrock LLM. |
| 2.7 | **How does it avoid hallucinating?** | Answered | Answers are grounded in retrieved KB chunks and include citations. The system prompt forbids inventing policies or employee data, requires conflicts to be flagged, and requires escalation when information is missing. |

---

## 3. Summary

- **Answered:** the technical questions (2.1, 2.3–2.7).
- **Answered after HR call #1:** engagement and stakeholder questions (1.2–1.6).
- **In progress:** deployment (2.2). The channel is HR's decision (SharePoint HR hub vs Teams).
