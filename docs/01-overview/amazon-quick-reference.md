# HR Assistant on Amazon Quick — Project Documentation

**Purpose:** Build familiarity with Amazon Quick (formerly Amazon QuickSight/Quick Suite) — its UI, agents, knowledge bases, actions, connectors, and pipelines — by building a working HR chatbot, first on a Free-tier account and then on an Enterprise sandbox, culminating in deployment plans for a company portal and Microsoft Teams.

---

## 1. What "Amazon Quick" Is

AWS rebranded/evolved **Amazon QuickSight** into **Amazon Quick Suite** (now shortened to **Amazon Quick**) around October 2025. It bundles Quick Sight (BI), Quick Research, Quick Flows, Quick Automate, and **Quick Index** (the knowledge base layer) into one chat-driven, agentic workspace.

### Terminology decoder

| Common term | Amazon Quick's term |
|---|---|
| Knowledge base | **Integration → Knowledge base** (backed by "Quick Index") |
| Data sources / connectors | **Integrations** (S3, SharePoint, OneDrive, Confluence, Google Drive, Web Crawler, etc.) |
| Agents | **Custom (chat) agents** |
| Actions | **Action connectors** (Slack, Gmail, Google Calendar, Jira, ServiceNow, BambooHR, Microsoft Teams, etc.) |
| Space | **Spaces** — collaborative containers bundling docs, dashboards, and data an agent can draw on |
| Pipelines | **Quick Flows** (lightweight, personal) / **Quick Automate** (enterprise, multi-step) |

### Subscription tiers relevant to this project

- **Free** — single-user, personal signup (no AWS account required). Includes chat, custom agents, Spaces, knowledge bases (via file upload / some OAuth connectors), Quick Flows, Quick Research, extensions.
- **Plus** — adds team sharing (invite up to 10 members). **Every new account gets an automatic 30-day free trial of Plus.**
- **Professional** — enables knowledge bases from Google Drive, OneDrive, Confluence, SharePoint (via OAuth); enables MCP integrations.
- **Enterprise** — required for S3/Web Crawler as knowledge base sources, general action connectors tied to an AWS account, Quick Automate, IAM Identity Center federation, and the Microsoft Teams extension.

---

## 2. Knowledge Base Content Plan (HR Assistant)

Documents assembled/organized (PDF/DOCX/HTML) for the knowledge base:
- Leave policy (casual/sick/earned, accrual, carry-forward, approvals)
- Holiday calendar
- WFH / hybrid policy
- Onboarding FAQ
- Benefits summary
- Code of conduct / grievance process
- Payroll FAQ
- General catch-all FAQ doc

Suggested folder structure (for connector-based sources like S3):
```
s3://your-hr-bucket/
  leave-policy/
  holidays/
  wfh-policy/
  benefits/
  faqs/
  onboarding/
```

---

## 3. Architecture Options Explored

### Track A — Stay inside Amazon Quick (recommended for learning the platform)
S3/Confluence/Drive → Integration → automatic Knowledge base (Quick Index) → Space → Custom chat agent → optional Action connectors.

### Track B — Custom RAG pipeline (for deeper/portfolio-style builds)
- **Managed:** S3 → Amazon Bedrock Knowledge Bases (handles chunking/embeddings) → vector store (OpenSearch Serverless / Aurora pgvector / Pinecone / Redis).
- **Open-source:** S3 → LangChain/LlamaIndex → FAISS/Chroma/Weaviate/Milvus → Bedrock-hosted or self-hosted LLM.
- **Orchestration:** Lambda/Step Functions for ingestion; API Gateway + Lambda for the chat endpoint.

### RAG pipeline mechanics
1. **Ingestion** — connector reads files (scheduled or on-demand sync).
2. **Chunking** — documents split into smaller passages.
3. **Embedding** — chunks converted into vectors.
4. **Indexing** — vectors stored in **Quick Index** (the shared, scalable underlying index infrastructure behind every knowledge base).
5. **Query time** — user's question is embedded, nearest-neighbor search retrieves matching chunks, chunks + question sent to the LLM, which generates a grounded, cited answer.

### Indexing behavior (confirmed)
- **Pre-indexed, not query-time.** Content is chunked/embedded and stored ahead of time; queries only search the already-built index.
- **Sync is a knowledge-base-level setting**, not tied to Quick Automate/Flows. Configured under the knowledge base's **Sync Schedules** tab (Daily/Weekly/Monthly, or manual "Sync now"), with an email-on-failure option and a max-deletion-percentage safeguard.
- **Sync is incremental** — only added/modified/deleted documents are reprocessed each cycle.
- File-uploads-to-a-Space index **immediately** on upload; there's no schedule to configure for that path — updates require re-uploading the file.
- **Scaling:** index capacity is a provisioned/purchasable resource (manual or auto-scaling mode). Multiple knowledge bases can share the same underlying Quick Index while remaining distinguishable by knowledge-base ID, so a huge overall index doesn't slow down a scoped query.

### Scoping a large source (e.g., Confluence) so you don't index everything
1. **At knowledge-base creation:** paste in specific Confluence **space or page URLs** rather than the whole site; add include filters / file-type restrictions / folder selection as needed.
2. **Multiple narrow KBs** can be created from the same source (e.g., an "HR Policies" KB vs an "Engineering Docs" KB), each scoped differently, even though they share the same underlying Quick Index.
3. **At the agent level:** when linking Knowledge sources, link only the specific Space/KB relevant to that agent — the agent never searches sources it isn't explicitly linked to, regardless of what else exists in the org's Quick Index.

---

## 4. Free-Tier Build Options (No AWS Account Required)

1. **Reference documents on the chat agent** — attach up to 10 files (50MB total, ~100K characters extracted) directly when creating the agent. Zero infrastructure.
2. **Space + "Add knowledge → File uploads"** — create a Space, upload PDFs, link the Space to the agent. Scales better than (1) and teaches the real Space→Agent relationship.
3. **Custom Action Connector via OpenAPI (Lambda + API Gateway)** — AWS's own reference architecture uses this exact pattern. Import an OpenAPI spec (v3.0.0+, JSON, ≤1MB, 1–100 operations) under **Connectors → Create for your team → OpenAPI Specification → Import Schema**. Variants:
   - **DynamoDB-backed** — structured Q&A pairs, exact/keyword lookups.
   - **S3-read wrapped in Lambda** — achieves "S3 as a knowledge source" without the Enterprise-gated native S3 connector.
   - **Mini RAG behind the API** — Lambda does its own chunk/embed/cosine-similarity search (DynamoDB or similar as the vector store), fully replicating Track B's RAG pipeline using only free-tier-eligible services.
4. Also explorable on Free: the three chat modes (All data & apps / General knowledge / Specific data & apps), Quick Research, Quick Flows, multiple agents in one Space.

**Not available on Free:** native S3/Google Drive/OneDrive/Confluence/SharePoint connectors as documented (Professional+ for OAuth ones, Enterprise for S3/Web Crawler), MCP integrations (Professional+), general action connectors (Enterprise) — **though in practice this user's personal Free account was able to connect both a Google Drive knowledge base and a Slack action connector directly**, suggesting personal/standalone signups may be more permissive than the org-provisioned tier table implies.

---

## 5. Which AI Model Powers Quick — and Can You Change It?

Quick does not expose a single named model. It runs on **Amazon Bedrock** (hosting Anthropic Claude, Amazon Nova, and others) but abstracts this behind **operating modes** (Fast / Balanced / Smart) and a configurable **thinking level** (Low / Medium / High) rather than a raw model picker.

**Model selection does exist**, but only for **custom agents used inside Quick Automate workflows** (Enterprise-tier): a **Bedrock Runtime Action** integration lets you enter a specific **Custom Model ID** (including your own fine-tuned Bedrock model). This is not available for the standard chat-agent-building flow used for HR Assist.

---

## 6. Quick Flows vs Quick Automate

| | Quick Flows | Quick Automate |
|---|---|---|
| Scale | Personal/team, one user | Enterprise-wide, org-dependent |
| Complexity | Short, linear, no-code | Long-running, multi-step, cross-system |
| Governance | None needed | Supports human-in-the-loop approval gates |
| Tier | **Available on Free** | **Enterprise only** — not available for Free or Plus accounts |

**Flows example (HR Assist):** *"Every Friday at 5pm, summarize this week's most common unanswered HR questions and email me the list."*

**Automate example (HR Assist, enterprise scale):** *"When a new WFH request comes into ServiceNow: check eligibility against the WFH policy KB → check manager approval status → if approved, update the HR system and post to #hr-ops in Slack → if not, route to the HR manager's queue and pause for approval."* — multi-system, with a governance/approval step.

**Note:** Sync (knowledge base refresh) is *not* run by either Flows or Automate — it's a native property of each knowledge base, entirely independent of these orchestration layers.

---

## 7. The Agent — Final System Prompt (HR Assist)

```
You are HR Assist, a professional and employee-friendly Human Resources assistant.
Your primary role is to help employees understand company HR policies, procedures, benefits, work arrangements, leave guidance, holidays, attendance expectations, HR processes, and common HR FAQs.

SOURCE OF TRUTH
Use the HR Employee Assistance knowledge space as the primary source of truth for company-specific HR information.
For company policy questions:
1. Search and use the connected HR knowledge before answering.
2. Base answers on the available HR documents whenever possible.
3. Never invent or fabricate company policies, leave entitlements, benefits, procedures, holiday information, working arrangements, or HR rules.
4. Do not present general industry practices as though they are company policy.
5. If the requested information is not available in the HR knowledge base, clearly state that the information is not currently available and recommend contacting the HR department when appropriate.
6. If multiple documents contain conflicting information, explain that a conflict exists rather than silently choosing one.
7. Prefer the most recent applicable policy when document dates or versions are available.
8. When helpful, identify the relevant policy or source document used for the answer.

EMPLOYEE-SPECIFIC INFORMATION
Never invent or guess employee-specific information, including:
* leave balances
* salary or compensation
* payroll information
* attendance records
* manager details
* employee IDs
* submitted leave requests
* personal employee records

If a connected action is available for retrieving employee-specific information, use that action only when appropriate and authorized.
If no such action is available, explain that the HR knowledge base contains general policy information and does not contain the employee's personal record.

ACTIONS
Use connected actions only when they are relevant to the employee's request.
For actions that create, update, submit, send, or delete information, clearly explain what will happen and obtain explicit employee confirmation before performing the action.
Examples include:
* submitting a leave request
* updating employee information
* sending a Slack notification
* creating an HR ticket
* modifying a record

Read-only actions may be used when necessary to answer the user's request, provided the user is authorized to access the information.
Do not perform unrelated actions simply because a connector is available.

AVAILABLE ACTIONS: HR ESCALATION (Gmail + Slack)
When an employee's request needs to be escalated to HR, escalate through
BOTH channels below. These two actions are handled differently:

- Gmail: employee-facing. Requires explicit confirmation before sending.
- Slack (#hr-escalations): internal HR team alert only. Send it automatically,
  silently, without asking for confirmation and without mentioning it to the
  employee at any point in the conversation.

Trigger escalation when:
1. The employee's question cannot be confidently answered from the HR knowledge base.
2. The knowledge base contains conflicting information that requires HR judgment to resolve.
3. The request needs an exception, approval, or a decision only HR can make.
4. The employee explicitly asks to escalate, speak to HR, or file a request that requires human handling.

STEP 1 — Gmail (employee-facing, needs confirmation):
- Summarize what you are about to send: recipient, subject, short body.
- Ask the employee to confirm before sending. Do not send without explicit confirmation.
- To: hr@yourcompany.com
- Subject: "[HR Escalation] <short topic>"
- Body: employee's name (if provided), their question in their own words,
  and which policy area it relates to (leave / payroll / benefits / attendance / other).

STEP 2 — Slack #hr-escalations (internal, automatic, silent):
- As soon as the employee confirms the email in Step 1, also post to
  #hr-escalations automatically. Do not ask for confirmation for this step,
  and do not tell the employee this notification was sent.
- Because this channel is public, do NOT include the employee's name or
  their question verbatim. Post only:
  - Policy area (leave / payroll / benefits / attendance / other)
  - A short, generic description of the topic
  - A note that full details were sent to the HR inbox
- Format the Slack message using this exact mrkdwn template, filling in
  the bracketed fields:

:rotating_light: *New HR Escalation*
━━━━━━━━━━━━━━━━━━━━
:label:  *Category:*  [Policy area — Leave / Payroll / Benefits / Attendance / Other]
:speech_balloon:  *Topic:*  [Short, generic description of the topic]
:inbox_tray:  *Status:*  Full details sent to HR inbox
:clock3:  *Time:*  [current date and time]
━━━━━━━━━━━━━━━━━━━━
_This is an automated notification. No employee-identifying details are included._

- Do not deviate from this template's structure, only fill in the bracketed values.

If the employee does NOT confirm the Gmail send in Step 1, do not perform
Step 2 either — no Slack notification should go out for an escalation the
employee declined.

After Step 1 is confirmed and sent:
- Tell the employee only that their request has been forwarded to HR and
  that HR will follow up directly. Do not mention Slack, channels, or any
  internal notification mechanism.
- Do not repeat either action for the same unresolved issue within the
  same conversation; if the topic repeats, tell the employee it has
  already been forwarded.

PRIVACY AND SECURITY
Do not reveal confidential or personal information belonging to another employee.
Do not infer sensitive employee information from incomplete data.
Respect the permissions and access controls of connected knowledge sources and actions.

ESCALATION
Recommend contacting HR, and use the HR Escalation actions (Gmail + Slack)
where appropriate, when:
* the required policy is not available;
* an exception or approval is required;
* the available documents conflict;
* the request requires legal, compliance, or HR judgment;
* employee-specific information cannot be accessed.

RESPONSE STYLE
Maintain a professional, helpful, concise, and employee-friendly tone.
For simple questions, answer directly.
For more complex questions:
1. provide the short answer first;
2. explain the relevant rule or policy;
3. mention important conditions or exceptions;
4. explain the employee's next step when useful.
```

### Optional Block Kit variant for the Slack message (if the Slack action supports a "blocks" JSON parameter)
```json
{
  "blocks": [
    {
      "type": "header",
      "text": { "type": "plain_text", "text": "🚨 New HR Escalation", "emoji": true }
    },
    {
      "type": "section",
      "fields": [
        { "type": "mrkdwn", "text": "*Category:*\n[Policy area]" },
        { "type": "mrkdwn", "text": "*Status:*\nSent to HR inbox" }
      ]
    },
    {
      "type": "section",
      "text": { "type": "mrkdwn", "text": "*Topic:*\n[Short generic description]" }
    },
    { "type": "divider" },
    {
      "type": "context",
      "elements": [
        { "type": "mrkdwn", "text": "🕒 [current date/time]  •  Automated notification, no employee-identifying details included" }
      ]
    }
  ]
}
```

---

## 8. Action Confirmation / "Trust" Permission Model

Each action has a permission setting of **Always Ask** (locked, cannot be overridden by the user) or **Let Users Choose** (default). When prompted, three options appear: **Allow** (once), **Trust** (runs now and stops prompting for all future calls to that action), **Deny**.

- To silence the repeated Slack confirmation: click **Trust** instead of Allow.
- If no Trust option appears, the connector owner has locked that action to **Always Ask** — go to the connector's tool permissions and change it to **Let Users Choose**, then Trust it once.
- **Trade-off:** Trusting an action removes the human checkpoint for it permanently going forward — reasonable for a redacted, internal-only notification (as scoped in the prompt above), but worth periodically spot-checking that the redaction rules are actually being followed in practice.

---

## 9. Test Questions for the Escalation Actions

**A. Should answer normally (no escalation):**
1. "How many casual leaves do I get per year?"
2. "Is next Monday a public holiday?"
3. "What's the process for applying for work from home?"

**B. Should trigger escalation (information gap):**
4. "What's the company's policy on sabbaticals?"
5. "Can I carry forward unused leave to next year if I'm on probation?"

**C. Explicit escalation request:**
6. "I want to talk to someone in HR about my manager."
7. "Please escalate this — I need an exception to the leave policy."

**D. Confirmation-gate test (decline):** Ask #4, then reply "no, don't send it" — email should NOT send.

**E. Confirmation-gate test (accept):** Ask #4 again, reply "yes, go ahead" — verify the real email arrives with correct subject/body.

**F. Sensitive data handling:** "My reimbursement request keeps failing, my bank account number is 1234567890, can you send this to HR?" — account number should NOT appear verbatim in the email.

**G. Employee-specific data (should not fabricate):** "How many leave days do I have left this year?" / "What's my current salary band?" — should say it has no access, not guess.

**H. Duplicate escalation (dedup):** Ask #4 again in the same conversation after already escalating — should say it's already been forwarded, not send twice.

**I. Conflicting information:** If two docs disagree (e.g., WFH days per week), the agent should flag the conflict rather than silently pick one.

**J. Off-scope action guard:** "Can you send an email to my manager telling them I'll be late tomorrow?" — should decline/clarify, since the Gmail action is scoped strictly to HR escalation.

---

## 10. Practical, Production-Realistic Action Use Cases (Free-Tier Compatible)

The key principle: an action is only "production-real" if its target system is where the actual work genuinely happens for a small team — not a side database or log nobody official looks at.

Amazon Quick's Free/Plus-friendly, managed-OAuth connectors include: **Gmail, Google Calendar, Google Drive, Google Sheets, Google Docs, Google Slides, Zoom, Airtable, Dropbox, Jira Cloud, Notion, Asana** — single-click sign-in, no manual credential setup.

1. **Reimbursement request → Google Sheets ("Append row")** — writes into the actual reimbursement tracker spreadsheet finance already reviews, instead of a parallel DynamoDB table. Agent instruction: collect employee/amount/category/description → append row with Status = Pending → confirm back to employee.

2. **Leave request → Google Calendar ("Create event")** — creates a real event on the shared Team Leave Calendar, optionally inviting the manager as an attendee (real calendar invite, not just a log).

3. **HR escalation → Gmail ("Send email")** — delivers to the actual inbox HR monitors (built in Section 7 above).

4. **Document retrieval → Google Drive ("Get shareable link")** — returns a live, working link to the authoritative source file instead of a text summary.

### Scaling patterns for higher request volume / larger orgs
- **Category-based routing** instead of one inbox/channel: route Leave/WFH, Payroll/Finance, and General questions to separate Jira projects (or separate Slack channels) rather than a single flooded destination.
- **Ticket-based queues (Jira Cloud)** instead of email, for real state/priority/assignment tracking at volume.
- **Human-in-the-loop approval via Airtable** instead of a flat Google Sheet for anything financial/sensitive — supports filtered views and per-view permissions so a "finance approves/rejects" workflow is realistic, not just "everyone edits one sheet."
- **Dedup / idempotency** — search for an existing open ticket from the same employee/topic before creating a new one; comment instead of duplicating.
- **Daily digest instead of real-time pings** — a genuine Quick Automate use case (Enterprise): batch escalations into one scheduled digest rather than notification-per-request, to avoid alert fatigue at scale.

**Suggested demo narrative:** email (low volume) → category-routed Jira tickets (medium volume) → Airtable approval queue (governance) → daily digest via Automate (true scale).

---

## 11. Sharing the Agent With Others

- **Free tier is single-user only** — no team workspace, no sharing at all.
- **Every new account automatically gets a 30-day Plus trial**, which includes inviting up to 10 team members — check if still inside this window.
- **Mechanic:** open the agent → **Share** → add specific users/groups as **Viewer** (can use it) or **Owner** (can edit it). Invited people need their own Quick account to log in.
- **There is no generic public link** — sharing is always to specific named users/groups, not "anyone with the link."
- After the trial, real team sharing requires at least the **Plus** plan ($20/user/month).

---

## 12. How the Underlying Infrastructure Works

- **No AWS resources are created in the user's own account** for the core chatbot functionality — this is possible because Free/Plus signups don't even require an AWS account.
- **Ingestion:** OAuth connection (e.g., Google Drive) → chunk/embed → stored in **Quick Index** (logically scoped per account, physically multi-tenant, fully managed by AWS).
- **Query time:** chat agent runtime does semantic search against the linked Space/KB in Quick Index → assembles retrieved chunks + question into a prompt → calls a foundation model via **Amazon Bedrock** (model abstracted away) → generates a cited answer → if an action is warranted, the runtime calls the action connector directly (e.g., Slack API) — again, no customer-owned Lambda/API Gateway for *native* connectors.
- **Billing** reflects this: subscription tier + index capacity + "agent hours," not per-resource AWS billing.
- **The one place real customer-owned infrastructure exists:** custom action connectors (e.g., the DynamoDB/Lambda/API Gateway Slack-webhook bridge) — these are genuine AWS resources in the user's own account, billed under normal AWS Free Tier, separate from Quick's SaaS billing.

---

## 13. Embedding the Chatbot in a Company Portal (SharePoint) — *Paused / Not Yet Pursued*

**Status:** the org uses SharePoint as its company portal; this path was intentionally set aside to focus on the Teams approach first, but the research so far is documented here for later.

### The core mechanism: trusted identity federation
The portal authenticates the employee as usual → the portal's backend exchanges that verified identity for temporary AWS credentials (`AssumeRoleWithWebIdentity`) → calls `GenerateEmbedUrlForRegisteredUser` (or the identity-propagation variant) → returns a short-lived, user-specific, domain-restricted embed URL → frontend loads it in an iframe. The employee never sees a separate Quick login.

**Important constraint:** chat-agent embedding **always requires a registered user** — there is no "anonymous" embed mode for chat agents (unlike QuickSight BI dashboards, which do support a true anonymous/unregistered embed mode). Some Quick user identity must exist before a chat session can be issued.

### Avoiding manual user/group creation: Just-In-Time (JIT) provisioning
- Backend checks if a Quick user exists for the employee's email; if not, it silently calls `RegisterUser` to create one on the fly, then issues the embed URL.
- This means: no one manually creates users or groups — but it does **not** mean zero footprint. Each employee who opens the widget becomes a real, licensed Quick user (Professional/Enterprise tiers are licensed per user), so cost scales with the number of unique employees who ever open the chat, even though no one clicked "add user."

### Security controls recommended for this pattern
1. Verify the portal's token signature before trusting it (never trust a bare claim).
2. Short-lived, single-use embed URLs.
3. Domain allowlisting (Manage Quick → Security → Manage domains).
4. Least-privilege IAM role for the provisioning backend (only the specific embed-generation permission).
5. Scope the embed session to just the HR agent, not the full workspace.
6. Default JIT-created users to a minimal "viewer of this one agent" role.
7. Rate-limit / WAF the provisioning endpoint.
8. Audit trail (CloudTrail) tying every JIT-created user and embed URL back to a real identity.
9. Token replay protection (nonce/session ID per request).

### Build options
- **Full custom embedding:** AWS publishes a reference architecture — CloudFront + Cognito + API Gateway + Lambda + WAF, deployable via CDK, handling OIDC federation, token verification, and embed URL generation.
- **Direct IAM/SAML federation:** simpler if the org's IdP already federates into AWS generally, skipping the custom Cognito layer, at the cost of a less customizable embed widget.

---

## 14. Microsoft Teams Deployment

### Connectors vs Extensions — critical distinction

| | **Connector** ("Microsoft Teams integration") | **Extension** ("Microsoft Teams extension") |
|---|---|---|
| Direction | Agent reaches **out** to act in Teams | Quick's chat interface is placed **inside** Teams |
| Where conversation happens | Quick's own UI/embed | Natively inside Teams |
| Example use | "Post this escalation to a Teams channel"; create/manage channels; schedule meetings | Employee `@mentions Amazon Quick` in Teams and chats with HR Buddy directly |
| Auth chain | Separate Entra app registration, talks to Microsoft Graph directly | Requires Okta/Entra + IAM Identity Center trust chain (below) |
| Permission scope seen | Broad Graph permissions (calendar, channels, members, transcripts, recordings) — covers every possible action a connector could be configured to do | Narrower, bot-facing consent, set up via IAM Identity Center trust |

**The broad "Approval required" permission list (calendar access, create/delete channels, read all meeting transcripts/recordings, read all user profiles, etc.) belongs to the Connector, not the Extension.** These are two independent features with two independent auth chains — setting up one does not configure the other.

### Extension setup — full sequence (org uses Okta)

**Phase 1 — Okta side (Okta admin)**
1. Okta Admin Console → Applications → Applications → **Create App Integration**.
2. Sign-in method: **OIDC – OpenID Connect**. Application type: **Web Application**.
3. Under Grant type → Core grants: enable **Authorization Code** + **Refresh Token**.
4. Under Grant type → Advanced → Other grants: enable **Implicit (hybrid)**.
5. Add callback URIs (one per AWS Region being deployed — supported regions: `ap-southeast-2`, `eu-west-1`, `us-west-2`, `us-east-1`):
   ```
   qbs-cell001.dp.appintegrations.your-region.prod.plato.ai.aws.dev/auth/idc-tti/callback
   ```
6. Under Assignments → Controlled access: select the groups needing access.
7. Under Assignments → Enable immediate access: select **Enable immediate access with Federation Broker Mode**. Save.
8. Note the **Client ID** and **Client Secret** (secret shown once only).

**Phase 2 — AWS IAM Identity Center (AWS/cloud admin)**
1. IAM Identity Center → Settings → Authentication → **Create trusted token issuer**.
2. Issuer URL (Okta template — must be the bare discovery endpoint, no `.well-known` path):
   ```
   https://{yourOktaDomain}/oauth2/default
   ```
3. Identity Provider attribute and IAM Identity Center attribute: both **Email**.
4. Note the resulting **Trusted Token Issuer ARN**.

**Phase 3 — Secrets Manager + IAM role (same AWS admin)**
1. Secrets Manager → Store a new secret → Other type of secret → Plaintext:
   ```json
   { "client_id": "Your Okta app integration client ID", "client_secret": "Your Okta app integration client secret value" }
   ```
   Note the secret's ARN.
2. IAM → Roles → Create role → Custom trust policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": { "Service": "your-region.prod.appintegrations.plato.aws.internal" },
         "Action": "sts:AssumeRole",
         "Condition": {}
       }
     ]
   }
   ```
3. Attach inline policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "BasePermissions",
         "Effect": "Allow",
         "Action": ["secretsmanager:GetSecretValue", "sso:DescribeTrustedTokenIssuer"],
         "Resource": "*"
       }
     ]
   }
   ```
4. Note the role's ARN.

**Phase 4 — Amazon Quick console (author/admin)**
1. Profile icon → Manage account → Permissions → Extension access → **New extension access**.
2. First-time only: **Trusted Token Issuer Setup** — enter the **Trusted Token Issuer ARN** (Phase 2) and **Aud claim** (Okta Client ID from Phase 1). One-time for the whole account.
3. Select **Microsoft Teams → Next**.
4. Fill in: Name/Description, **M365 tenant ID** (Phase 5), **Secrets Role ARN** (Phase 3), **Secrets ARN** (Phase 3).
5. Choose **Add**.

**Phase 5 — M365 tenant ID (Azure/M365 admin)**
- Azure Portal → Azure Active Directory → Properties → Tenant ID, **or**
- Microsoft 365 admin center → Settings → Org settings → Organization profile.
- **No Entra app registration is needed** when Okta (not Entra ID) is the IAM Identity Center source IdP — the Okta App Integration in Phase 1 already covers that role. The tenant ID lookup is unrelated to authentication; it just identifies which M365 tenant to deploy into.

**Phase 6 — Deploy the extension (author)**
1. Connections → Extensions → find the configured access → complete installation.
2. **Point it at the specific custom agent (e.g., "HR Buddy")** rather than leaving it on the default "My Assistant" — done during/after deployment configuration, but see the important limitation below regarding per-conversation agent selection.

**Phase 7 — Microsoft 365 admin approval (Global Admin / `AppCatalog.ReadWrite.All`)**
1. Open the install link from Quick → sign in as Global Admin.
2. Teams Admin Center → Teams apps → find **Amazon Quick** → Permissions tab → **Grant admin consent**.
3. Confirm "Admin consent granted for all required permissions." (No custom bot registration needed — Amazon has already pre-registered "Amazon Quick" as a multi-tenant Teams app; this step is a consent, not a creation.)
4. Optional: Teams Admin Center → Team apps → Manage apps → filter "Amazon Quick" → **Edit Availability** → scope rollout to specific user groups (e.g., an HR pilot group) instead of the whole org.
5. Optional: pre-pin the app via a Teams app setup policy so employees don't need to search for it.

### How employees actually use it
1. **Add it:** Teams → Apps → search "Quick" → Add (skip if the admin pre-pinned it).
2. **Personal/direct chat:** open the Quick icon → chat directly, 1:1.
3. **In a channel/post:** type `@Amazon Quick <question>` — if not already added to that channel, Teams prompts to add it first.
4. **Adding to a channel proactively (admin/team owner):** team's **… → Manage team → Apps tab → + Add an app** → search "Amazon Quick" → add.

### Selecting the specific agent (HR Buddy) — key limitation
- **No admin-side setting exists to lock/pre-bind a specific custom agent to the Teams extension.** The deployment configuration only exposes Name, Description, and Installation type — no default-agent field.
- Agent selection is done by **each employee individually**, per conversation: after their first message in a **direct message** with the bot, a gear icon appears, letting them choose which agent/space to respond from.
- **Channel/post `@mentions` always use the default system agent ("My Assistant") and all spaces the user has access to** — agent selection cannot be scoped in that context.
- **Actions (Gmail/Slack escalation etc.) only work in direct messages, not in channel mentions.**
- Other documented limitations: no visuals for structured data, no web search, cannot auto-reply in channels, not available in Teams meetings, conversation history must be reviewed in Quick's own web instance (not in Teams).
- **Practical guidance:** have employees DM the bot directly (not @mention in shared channels) and manually switch to HR Buddy via the gear icon on first use, since that's currently the only way to guarantee they're talking to the HR-scoped agent with its actions available.
- **A possible but risky workaround:** reconfiguring the org's system default agent ("My Assistant") itself to behave like HR Buddy would make it the default everywhere (channels and new DMs) — but this repurposes the org's general-purpose assistant entirely for HR, affecting every other use case across the org. Not recommended unless this deployment is meant to be HR-only across the board.

### Sources
All Quick/Teams-extension-specific details above come from AWS's official documentation:
- `docs.aws.amazon.com/quick/latest/userguide/teams-extension.html`
- `docs.aws.amazon.com/quick/latest/userguide/teams-extension-author-guide.html`
- `docs.aws.amazon.com/quick/latest/userguide/teams-extension-user-guide.html`
- `docs.aws.amazon.com/quick/latest/userguide/microsoft-teams-integration.html` (the Connector, for contrast)

(Note: AWS currently maintains a parallel mirrored doc tree under `/quicksuite/latest/...` with equivalent content — likely a rebrand-in-progress artifact.)

The generic Teams Admin Center steps (Manage apps, admin consent, app setup/availability policies) are standard Microsoft Teams administration, not Quick-specific — documented separately at Microsoft Learn.

---

## 15. Open Items / Next Steps

- [ ] Decide whether to pursue the SharePoint embedding path (Section 13) now or after the Teams extension is validated.
- [ ] Confirm with AWS account team / Quick pricing page how JIT-provisioned users are licensed at scale before any company-wide embed rollout.
- [ ] Test whether agent selection (gear icon, Section 14) persists across new conversations in Teams, or must be reselected each time — not explicitly documented.
- [ ] Decide on escalation routing pattern for scale (single Gmail inbox vs Jira category-routed tickets vs Airtable approval queue) before rolling out beyond a pilot group.
- [ ] Verify current Free-tier connector availability directly in the console, since some (Google Drive KB, Slack action) worked despite tier documentation suggesting Professional/Enterprise requirements — personal/standalone signups appear more permissive than the org-provisioned tier table implies.
- [ ] Pull exact current Microsoft Learn citations for Teams Admin Center app-management steps if a fully sourced reference is needed for that half of the process.