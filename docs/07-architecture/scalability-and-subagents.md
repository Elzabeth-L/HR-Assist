# HR Assist — Scalability & Sub-Agents

> **Purpose:** Describe how HR Assist could scale from a single agent to an orchestrator with domain sub-agents, and when to make that move.
> **Status:** Proposal for **Phase 2+**. v1 stays a single agent. Final domain segmentation will be proposed **after HR's question list is received**.

---

## 1. What Was Discussed on the HR Call

The idea came up on HR requirements call #1 (confirmed in the transcript, about 37–41 min):

- **Solution architect:** you can segment specialised agents, e.g. *"an agent that only handles employee benefits… an agent that only handles employee discounts… almost what you would call sub-agents… an HR specialist… could be another chatbot that it reaches out to."* Make one do its job very well, then repeat the pattern.
- **Engagement lead:** *"Maybe there is a routing agent which takes the intent, then determines, okay, it's a benefit question or it's a leave question, time and absence question, and it hands it off to that sub-agent behind the scene."* Initially, every sub-agent would do **discovery** only.
- **Engagement lead:** the segmentation *"will be something that we will do organically once we receive the question set and the document set"*. UST proposes the sub-agents, and HR reviews.
- **Solution architect:** prove a **thin slice** first (about 5 questions, one micro-domain), then add complexity. If an agent's scope creeps, that's a signal for another agent.

> Part of this discussion falls in a transcript gap (about 39:34 → 40:43). Confirm the details with the attendees if needed.

---

## 2. Agent Orchestration in Amazon Quick

A primary (**orchestrator**) agent can call other, specialised agents **as tools**, much like it calls an action connector. It recognises which domain a question belongs to and routes it accordingly.

---

## 3. Proposed Future Structure

```
                    Employee
                       │
                       ▼
          HR Assistant (orchestrator)  ← single front door, EBSCO-branded
           │           │            │
           ▼           ▼            ▼
   Policy sub-agent  Process sub-agent  (Future) Personal Data sub-agent
   SharePoint KB     HR "tribal          Only component with
                     knowledge" docs     SuccessFactors access
```

| Agent | Scope | Phase |
|---|---|---|
| **Orchestrator** | Understands intent, routes, assembles the final answer and citations | Phase 2+ |
| **Policy sub-agent** | Documented policies on **SharePoint** | Phase 2+ |
| **Process sub-agent** | "How do I…" knowledge HR is newly documenting (e.g. company car, interns, volunteer time off) | Phase 2+ |
| **Personal Data sub-agent** | Employee-specific answers (e.g. leave balance) from SuccessFactors | **Only after personalisation is approved** |

Domain sub-agents (e.g. **Benefits**, **Time & Absence**, **Discounts**) may replace or sit under the Policy / Process split. That depends on how HR's question list naturally groups.

---

## 4. Why This Matters for Scale

1. **Retrieval quality:** a narrow search scope per sub-agent keeps answers relevant as content grows.
2. **Permission isolation:** personalisation stays a **bounded, auditable** addition. Only one sub-agent ever touches employee data, which matches HR's "not personalised, considering the risks" stance.
3. **Cross-team reuse:** other departments (IT, Finance) could plug their own sub-agents into the same front door later.
4. **Cost and latency predictability:** each sub-agent loads only its own knowledge and actions.
5. **Ownership:** each sub-agent's content can be owned by an HR subject-matter expert, which addresses HR's question about domain-level admins.

---

## 5. Trade-offs

| Benefit | Cost / risk |
|---|---|
| Better relevance and isolation | More routing logic to build and maintain |
| Clear ownership per domain | Questions can be **misrouted** or span several domains |
| Easier to extend | More agents to test; regression testing per domain |

---

## 6. Recommendation

1. **v1:** a **single agent** over SharePoint + tribal-knowledge content, proven on a **thin slice** (one or two domains).
2. **Next:** introduce the orchestrator and sub-agents once the question list shows clear domains, a second domain is added, personalisation is approved, or another department wants to join.
3. **Testing:** keep a **golden question set per domain**, so routing and answers can be regression-tested every time a sub-agent changes.
