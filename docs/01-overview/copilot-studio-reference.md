# Microsoft Copilot Studio — Reference for HR Assist

> **Purpose:** Key Copilot Studio facts relevant to the HR assistant, taken from Microsoft's official documentation (checked October 2026).
> **Note:** Microsoft documents two authoring experiences (the "standard harness" and the newer "GitHub Copilot harness"). Setting names can differ slightly between them, so verify in the sandbox.

---

## 1. What It Is

Copilot Studio is Microsoft's platform for building AI agents. It's part of Microsoft 365 / Power Platform, the same ecosystem as EBSCO's Teams, SharePoint and Outlook. Agents use **knowledge sources** (e.g. SharePoint), **tools / connectors** for actions, and a selectable **AI model**, and can be published to Microsoft channels such as Teams.

---

## 2. SharePoint as a Knowledge Source

| Behaviour | Detail |
|---|---|
| Answers on behalf of the user | With "Authenticate with Microsoft" (the default for Teams), generative answers are made *on behalf of the user* chatting with the agent |
| Permission trimming | The user needs at least **Read** access to the site or content; otherwise the agent gives "no response" |
| Sensitivity labels | Only label-permitted content is surfaced; the highest label of the sources used is shown |
| Limitation | Encrypted files (sensitivity labels with encryption, Double Key Encryption) and password-protected files **can't be used** |

---

## 3. Work IQ

**What it is:** *"The intelligence layer that grounds Microsoft Copilot and your agents in real-time, shared context across your organization."* It has three layers:

| Layer | Role |
|---|---|
| **Data** | Unifies signals from files, emails, meetings, chats and business apps across Microsoft 365 |
| **Memory** | Builds a persistent understanding of how people and teams work |
| **Inference** | Lets agents reason and act through Work IQ MCP tools |

**What it requires:**
- It is added to an agent as a **tool** (Tools → Add tool → MCP → Work IQ (Preview)) and needs a connection and sign-in.
- One-time tenant setup by a **Global Administrator**, plus a **usage-based billing plan** (Azure subscription). Usage is billed in **Copilot Credits**, under a separate spending policy.
- It is **in preview** (*"not meant for production use"*) and **read-only** unless an admin turns on write operations.

**Relevance:** not needed for Phase 1, which answers from SharePoint content. It's relevant to future personalised stages (understanding an employee's emails and chats), and those would need a privacy review.

---

## 4. Guardrails (Agent Level)

| Guardrail | Setting | Purpose for HR Assist |
|---|---|---|
| Answer only from HR content | Turn **off** "Allow ungrounded responses" (general model knowledge) | No invented answers |
| No public web answers | Turn **off** "Use information from the web" | Keeps answers internal |
| Scoped knowledge | Add only the approved HR SharePoint sites | Answers only from approved sources |
| Citations | Built in | "Quote + source link" |
| Content moderation | Level **Lowest → Highest** (default **High**) | Filters harmful content |
| Instructions | Agent instructions (system prompt) | "Quote, don't interpret", "say I don't know" |
| User credentials | Tools can run with the user's own credentials | No over-privileged access |
| Security scan | Warnings and risk findings before publishing | Catches data-exposure risks early |

> **Limitation:** content moderation covers **predefined harmful-content categories only**. There's no setting to block **arbitrary topics** (e.g. "no policy interpretation"). These rules must come from instructions, scoped knowledge and testing.

---

## 5. Security & Governance (Admin Level)

| Control | What it does |
|---|---|
| Data policies (DLP) | Govern authentication, knowledge sources, connectors, HTTP requests, publishing channels and triggers |
| Environment routing | Gives makers a separate, safe environment to build in |
| Publishing control | Tenant-wide switch to stop publishing agents that use generative AI |
| Data residency | Keeps data in its geography; cross-region data movement can be disabled |
| Audit | Maker and agent activity in **Microsoft Purview**; alerts in **Microsoft Sentinel** |
| Encryption | Customer-managed keys (CMK); Customer Lockbox (doesn't cover audit telemetry) |
| Agent 365 | Agents as Entra identities with Conditional Access and access governance |
| Cost control | Caps on Copilot Credit spending |

---

## 6. Model Selection

### 6.1 Available models (standard harness, United States)

| Provider | Models | Status |
|---|---|---|
| OpenAI | GPT Chat Latest (default), GPT-4.1, GPT-5 Chat, GPT-5.5 Chat; others in preview | GA / preview |
| **Anthropic** | **Claude Sonnet 4.6** (General); **Claude Opus 4.6 / 4.7 / 4.8** (Deep) | **GA** |

In most other regions the same models are marked **cross-geo**: data might be processed outside the region.

### 6.2 Settings needed to use Anthropic models

| # | Setting | Where | Role |
|---|---|---|---|
| 1 | Enable **Anthropic** as an AI provider operating as a Microsoft subprocessor, and assign the users/groups allowed to use it | Microsoft 365 admin center → Copilot → Settings → AI providers operating as Microsoft subprocessors | AI Administrator or Global Administrator |
| 2 | Turn on **external models** for the environment | Power Platform admin center | Power Platform / tenant admin |
| 3 | *(Preview / experimental models only)* "Preview and experimental AI models" + "Move data across regions" | Power Platform admin center | Tenant admin |

- **Defaults:** Anthropic is **on** for most commercial tenants (including the US), **off** in the EU, EFTA and UK, and not available in government or sovereign clouds.
- **If only GPT models appear**, setting 1 or 2 is likely off, or the user isn't in the allowed group.

### 6.3 Azure AI Foundry models

The "bring your own model" feature applies to **prompt tools only**, not the agent's main model, and needs a `/chat/completions` endpoint. Claude models in Foundry use the Anthropic **Messages API** (`/anthropic/v1/messages`), so this route doesn't apply. **Use the native Anthropic option instead.**

### 6.4 Data handling

- Anthropic is a **Microsoft subprocessor**; the Microsoft Product Terms and Data Protection Addendum apply.
- **Avoid "Anthropic models with Data Retention"** (separate opt-in): Anthropic stores inputs and outputs outside Microsoft's agreement.
- Anthropic models are **excluded from the EU Data Boundary** and in-country processing commitments.

---

## 7. Sources

- [Security and governance](https://learn.microsoft.com/en-us/microsoft-copilot-studio/security-and-governance)
- [Add SharePoint as a knowledge source](https://learn.microsoft.com/en-us/microsoft-copilot-studio/knowledge-add-sharepoint)
- [Knowledge sources summary](https://learn.microsoft.com/en-us/microsoft-copilot-studio/knowledge-copilot-studio)
- [Add a generative answers node (content moderation)](https://learn.microsoft.com/en-us/microsoft-copilot-studio/nlu-boost-node)
- [Data, privacy, and security for web search](https://learn.microsoft.com/en-us/microsoft-copilot-studio/data-privacy-security-web-search)
- [Work IQ in Copilot Studio (preview)](https://learn.microsoft.com/en-us/microsoft-copilot-studio/use-work-iq)
- [Work IQ MCP overview](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/mcp/overview)
- [Enable your tenant for Work IQ](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/enable-work-iq)
- [Select a primary AI model](https://learn.microsoft.com/en-us/microsoft-copilot-studio/authoring-select-agent-model)
- [Choose an external model](https://learn.microsoft.com/en-us/microsoft-copilot-studio/authoring-select-external-response-model)
- [Anthropic models in Microsoft Online Services](https://learn.microsoft.com/en-us/microsoft-365/copilot/connect-to-ai-subprocessor)
- [Bring your own model for prompts](https://learn.microsoft.com/en-us/microsoft-copilot-studio/bring-your-own-model-prompts)
- [Deploy and use Claude models in Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/how-to/use-foundry-models-claude)
