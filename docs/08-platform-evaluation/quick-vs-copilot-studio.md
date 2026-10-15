# HR Assist — Amazon Quick vs Microsoft Copilot Studio

> **Purpose:** A side-by-side comparison for the HR assistant at EBSCO.
> **Basis:** Quick points come from our research and sandbox work; Copilot Studio points come from Microsoft's official documentation. Items marked ⚠️ still need hands-on verification.

---

## 1. Comparison

| Area | Amazon Quick | Microsoft Copilot Studio |
|---|---|---|
| **Fit with EBSCO's tools** | Separate AWS workspace; reaches Microsoft 365 through connectors | ✅ Native to Microsoft 365 (Teams, SharePoint, Outlook) |
| **SharePoint knowledge** | Supported (Professional / Enterprise tier); how permissions carry over is ⚠️ to verify | ✅ Native; answers on behalf of the user with permission trimming and sensitivity labels. ❌ Can't use encrypted or password-protected files |
| **Deployment to SharePoint / Teams** | ❌ Heavy. **Teams:** Okta → AWS Identity Center → Secrets Manager → IAM role → admin consent. **SharePoint:** custom web part + embed backend. | ✅ Likely lighter: built-in publishing to Microsoft channels with Entra sign-in. ⚠️ Verify the SharePoint option. |
| **Teams experience** | ❌ Known limits: no admin-set default agent; actions only in direct messages | ⚠️ Expected to behave like a standard Teams app |
| **Sign-in** | Okta + AWS Identity Center; every user needs a Quick licence | ✅ Entra ID (EBSCO's Microsoft sign-in) |
| **Guardrails** | Knowledge scoping (no general knowledge), system prompt, action confirmation | Ungrounded responses / web search off, content moderation, instructions, security scan |
| **Governance** | AWS tooling (IAM, CloudTrail, domain allowlists) | ✅ Central: data policies, Purview, Sentinel, environment routing, data residency, CMK |
| **AI models** | ❌ Model abstracted (modes instead of a model picker) | ✅ GPT and Anthropic Claude models selectable (admin must enable Anthropic) |
| **Work context** | Connectors bring the data in; the model infers the context | ✅ Work IQ semantic layer. ❌ Preview, extra billing, not needed for Phase 1 |
| **Outlook** | ✅ Drafting replies via connector. ❌ No email trigger, so auto-reply needs custom AWS infrastructure | ✅ Likely simpler (native Microsoft automation and email triggers) ⚠️ |
| **Future HR-system integration** | Custom OpenAPI connector; SuccessFactors design already done | Connector / custom API approach ⚠️; the same security design applies |
| **Cost model** | ✅ Predictable per-user subscription. ❌ Every user is a licensed user | ❌ Usage-based (Copilot Credits) and less predictable. ✅ Spending caps available ⚠️ |
| **Beyond chatbots** | ✅ BI dashboards, research, flows and automation | ✅ Power Platform (Power Automate, Power Apps) |
| **Maturity** | Recently rebranded; some documentation inconsistencies | ❌ Changing fast; two authoring experiences; several preview features |
| **Readiness** | ✅ Sandbox, prototype and architect support in place | ❌ Environment and architect still to be arranged |

---

## 2. Where Each Is Likely Stronger

**Copilot Studio**
- Much lower deployment effort into SharePoint and Teams
- Permission-aware SharePoint answers out of the box
- Central Microsoft governance EBSCO IT may already use
- Model choice, including Claude
- Outlook / email automation, if HR wants it later

**Amazon Quick**
- Ready to test now
- Predictable per-user cost
- Fits a wider EBSCO strategy if Quick is adopted for BI / analytics
- Aligns with EBSCO's AWS cloud partnership and the planned AWS-hosted HR-system proxy

**About the same on both**
- Grounded, cited answers and "answer only from content" settings
- Topic boundaries (no policy interpretation, etc.) rely on instructions and testing, not a built-in switch

---

## 3. To Verify Hands-On

- Answer quality on the same thin slice
- SharePoint permission trimming on both
- Time and approvals needed to publish to the SharePoint HR hub and Teams
- Real cost for EBSCO's employee count
- Models actually available in each environment
- Effort for HR content owners to keep content current
