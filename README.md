# HR Assist — AI HR Knowledge Assistant

An AI-powered assistant that helps EBSCO employees **find HR information instantly**, in everyday language, with every answer drawn from HR's own content and linked to its source. Designed and built by UST for EBSCO.

---

## The Problem

- Employees ask HR the **same questions** again and again (vacation and leave policies, volunteer time off, interns, company cars, parental leave, scholarships), usually over **Teams, email or phone**.
- Most answers **already exist on SharePoint**, but they take too many clicks to find, so employees don't search, or don't find what they need.
- Some answers aren't written down anywhere: they **live in HR team members' heads**.
- Questions often land on departmental **HR business partners**, whose role is consulting on bigger issues, and get forwarded around. There's no single place to get an answer.

## The Solution

A conversational **HR knowledge assistant** that:

| | How |
|---|---|
| **Finds the answer** | Searches HR's approved content (SharePoint policies plus HR-written "how do I…" guides) using retrieval-augmented generation: it searches first, then answers |
| **Shows its source** | Quotes the relevant text and links to the original page, so employees can check it |
| **Stays honest** | Answers only from HR content; says "I don't know" rather than guessing; points employees to their manager or HR when needed |
| **Respects boundaries** | Doesn't interpret policy or make HR decisions; never accesses sensitive personal data |
| **Lives where employees work** | Available in familiar places such as the SharePoint HR hub and/or Microsoft Teams, branded as EBSCO's own assistant |

```
Employee question ─► HR Knowledge Assistant ─► searches HR content (SharePoint + HR guides)
                                             ├─► Found     → answer + link to the source
                                             └─► Not found → "I don't know" + where to get help
```

## The Vision: Discover → Analyze → Act

| Stage | What the assistant does | Example |
|---|---|---|
| **Discover** | Finds the right HR information and cites it | "Where is the vacation policy?" |
| **Analyze** | Combines policy with the employee's own situation | "What's my vacation balance, and which leave type applies?" |
| **Act** | Takes action on the employee's behalf, with their confirmation | "Submit that leave request for me" |

The experience stays the same throughout; more capability is added behind it over time, as HR approves it.

## Technology

Two platforms are being evaluated in parallel: **Amazon Quick** (AWS's agentic AI workspace, used for the prototype) and **Microsoft Copilot Studio** (native to EBSCO's Microsoft 365 environment). The final platform will be chosen after hands-on testing of both.

## Repository Guide

| Folder | What's inside |
|---|---|
| [docs/01-overview](docs/01-overview/) | **[Project overview](docs/01-overview/project-overview.md)** (start here) · [Amazon Quick reference](docs/01-overview/amazon-quick-reference.md) · [Copilot Studio reference](docs/01-overview/copilot-studio-reference.md) |
| [docs/02-planning](docs/02-planning/) | [HR call #1 summary](docs/02-planning/hr-call-01-summary.md) · [Plan](docs/02-planning/plan-phase.md) · [Open questions](docs/02-planning/open-questions.md) · [HR requirements questions](docs/02-planning/hr-requirements-questions.md) |
| [docs/03-knowledge-base](docs/03-knowledge-base/) | [Knowledge sources](docs/03-knowledge-base/kb-source-options.md) |
| [docs/04-deployment](docs/04-deployment/) | [SharePoint guide](docs/04-deployment/sharepoint.md) · [Teams guide](docs/04-deployment/teams.md) · [Teams flow diagram](docs/04-deployment/teams-architecture.html) |
| [docs/05-testing](docs/05-testing/) | [Test plan](docs/05-testing/test-plan.md) |
| [docs/06-integrations](docs/06-integrations/) | [Outlook integration](docs/06-integrations/outlook-integration.md) · [SuccessFactors API scope](docs/06-integrations/successfactors-api-scope.md) · [SuccessFactors access & roles](docs/06-integrations/successfactors-access-roles.md) |
| [docs/07-architecture](docs/07-architecture/) | [Scalability & sub-agents](docs/07-architecture/scalability-and-subagents.md) |
| [docs/08-platform-evaluation](docs/08-platform-evaluation/) | [Evaluation approach](docs/08-platform-evaluation/evaluation-approach.md) · [Amazon Quick vs Copilot Studio](docs/08-platform-evaluation/quick-vs-copilot-studio.md) |
| [draft](draft/) | Original raw notes |
