# HR Assist — Microsoft Teams Deployment Guide

> **Deployment pattern:** Amazon Quick **Microsoft Teams extension**. Employees chat with the HR Assist agent inside Teams.
> **Identity provider:** Okta, federated into AWS IAM Identity Center.
> **Status:** Current priority deployment channel.

This guide lists everything needed to deploy HR Assist in Microsoft Teams: licences, access, configuration values, IAM permissions, admin approvals, rollout steps, known limitations and validation.

---

## Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Connector vs Extension — Read First](#2-connector-vs-extension--read-first)
3. [Prerequisites & Licensing](#3-prerequisites--licensing)
4. [Roles & Responsibilities](#4-roles--responsibilities)
5. [Configuration Values Register](#5-configuration-values-register)
6. [Implementation Steps (Phases 1–7)](#6-implementation-steps-phases-17)
7. [Permissions & Approvals Summary](#7-permissions--approvals-summary)
8. [Rollout Strategy](#8-rollout-strategy)
9. [End-User Guide](#9-end-user-guide)
10. [Known Limitations](#10-known-limitations)
11. [Optional: Teams Connector (Outbound Actions)](#11-optional-teams-connector-outbound-actions)
12. [Security Considerations](#12-security-considerations)
13. [Validation Checklist](#13-validation-checklist)
14. [Troubleshooting](#14-troubleshooting)
15. [Open Items](#15-open-items)
16. [References](#16-references)

---

## 1. Architecture Overview

```
 Employee (Teams client)
        │  DM / @mention "Amazon Quick"
        ▼
 Microsoft Teams ──► "Amazon Quick" multi-tenant Teams app (pre-registered by AWS)
        │
        ▼
 Amazon Quick App Integrations service (regional)
        │  1. Signs in the user through Okta (OIDC app integration)
        │  2. Reads Okta client credentials from AWS Secrets Manager (via IAM role)
        │  3. Exchanges the Okta token through the IAM Identity Center Trusted Token Issuer
        ▼
 Amazon Quick account (Enterprise)
        │  User resolved by email → Quick user
        ▼
 HR Assist custom chat agent ──► HR Space / Knowledge base (Quick Index)
        │                    └──► Action connectors (Gmail, Slack)  [DMs only]
        ▼
 Amazon Bedrock (model abstracted by Quick)
```

**Trust chain:** Okta (who the user is) → IAM Identity Center trusted token issuer (AWS trusts Okta tokens) → Secrets Manager + IAM role (Quick can use the Okta client) → Quick extension access (connects everything to the M365 tenant) → M365 admin consent (Teams allows the app).

---

## 2. Connector vs Extension — Read First

Amazon Quick has **two separate** Teams features. Each has its own authentication chain, and setting up one does **not** set up the other.

| | **Extension** (this guide) | **Connector** (optional, [§11](#11-optional-teams-connector-outbound-actions)) |
|---|---|---|
| Purpose | Brings the Quick chat **into** Teams | Lets the agent **act in** Teams (post messages, manage channels, schedule meetings) |
| Where users chat | Natively inside Teams | Quick's own UI |
| Auth chain | Okta + IAM Identity Center trusted token issuer | Separate Entra ID app registration, Microsoft Graph |
| Permission scope | Narrow, bot-facing consent | Broad Graph permissions (calendars, channels, transcripts, profiles) |

> The long "Approval required" list of Graph permissions (read all transcripts, create/delete channels, etc.) belongs to the **Connector**, **not** the Extension. Don't let it hold up approval of the Extension.

---

## 3. Prerequisites & Licensing

### 3.1 Subscriptions & accounts

| # | Requirement | Details |
|---|---|---|
| P1 | **Amazon Quick Enterprise** subscription | The Microsoft Teams extension is Enterprise-only. |
| P2 | Quick account in a **supported AWS Region** | `us-east-1`, `us-west-2`, `eu-west-1`, `ap-southeast-2` |
| P3 | Quick account integrated with **AWS IAM Identity Center** | The extension relies on Identity Center identities and trusted token propagation. |
| P4 | **Okta** as the Identity Center identity source (SAML + SCIM provisioning recommended) | Users and groups must exist in Identity Center with the **same email** as in Okta. |
| P5 | Every Teams user of HR Assist is a **licensed Quick user** | Provision through Identity Center groups assigned to Quick. Licences are per user. |
| P6 | **Microsoft 365 tenant** with Teams | You need the tenant ID and a Global Admin (or a role with `AppCatalog.ReadWrite.All`) for consent. |

### 3.2 Agent readiness

| # | Requirement | Details |
|---|---|---|
| A1 | HR Assist custom chat agent created and tested in Quick | System prompt finalised |
| A2 | HR Space / knowledge base linked to the agent | Confluence knowledge base indexed and linked |
| A3 | Action connectors (Gmail, Slack) connected and permission settings reviewed | See [§12](#12-security-considerations) |
| A4 | Agent **and** its Space shared with the pilot user group (Viewer) | Users can only choose agents and Spaces they have access to. |
| A5 | Test plan passed in the Quick web UI | All test cases passed |

---

## 4. Roles & Responsibilities

| Role | Responsible for | Phases |
|---|---|---|
| **Okta Administrator** | Create the OIDC app integration, callback URIs, group assignments, Federation Broker Mode; share Client ID and Secret securely | 1 |
| **AWS / Cloud Administrator** | IAM Identity Center trusted token issuer, Secrets Manager secret, IAM role and policies | 2, 3 |
| **Amazon Quick Administrator** | Extension access, trusted token issuer setup in Quick, user/group licensing | 4 |
| **Azure / M365 Administrator** | Provide the M365 tenant ID | 5 |
| **Quick Author (agent owner)** | Deploy the extension, share the HR Assist agent and Space | 6 |
| **M365 Global Admin / Teams Admin** | Admin consent, app availability, setup (pinning) policy | 7 |
| **HR Stakeholder** | Approve content scope, pilot group and escalation targets | Throughout |
| **Security / Compliance** | Review data flows, Slack redaction rules, action trust settings | Before go-live |

---

## 5. Configuration Values Register

Record each value as you create it. **Never store secrets in this file.** Use a password manager or Secrets Manager only.

| # | Value | Produced in | Consumed in | Recorded |
|---|---|---|---|---|
| V1 | AWS Region of the Quick account | — | 1, 3, 4 | ☐ |
| V2 | Okta domain (`{yourOktaDomain}`) | 1 | 2 | ☐ |
| V3 | Okta **Client ID** | 1 | 3 (secret), 4 (Aud claim) | ☐ |
| V4 | Okta **Client Secret** (shown once only) | 1 | 3 (secret only) | ☐ *store securely* |
| V5 | **Trusted Token Issuer ARN** | 2 | 4 | ☐ |
| V6 | **Secrets Manager secret ARN** | 3 | 4 | ☐ |
| V7 | **IAM role ARN** (secrets role) | 3 | 4 | ☐ |
| V8 | **M365 tenant ID** | 5 | 4 | ☐ |
| V9 | Extension access name | 4 | 6 | ☐ |
| V10 | Pilot Okta group / Teams user group | 1, 7 | 1, 7 | ☐ |

> **Ordering note:** Phase 4 needs the M365 tenant ID from Phase 5. Get the tenant ID early, before starting Phase 4.

---

## 6. Implementation Steps (Phases 1–7)

### Phase 1 — Okta: OIDC App Integration
**Owner:** Okta Administrator

1. Okta Admin Console → **Applications → Applications → Create App Integration**.
2. Sign-in method: **OIDC – OpenID Connect**. Application type: **Web Application**.
3. **Grant type → Core grants:** enable **Authorization Code** and **Refresh Token**.
4. **Grant type → Advanced → Other grants:** enable **Implicit (hybrid)**.
5. **Sign-in redirect URIs:** add one per Region you deploy to:
   ```
   https://qbs-cell001.dp.appintegrations.<region>.prod.plato.ai.aws.dev/auth/idc-tti/callback
   ```
   Replace `<region>` with V1, e.g. `us-east-1`. Check the exact URI against the current AWS author guide before saving.
6. **Assignments → Controlled access:** select the groups that need access (start with the pilot group, V10).
7. **Assignments → Enable immediate access:** select **Enable immediate access with Federation Broker Mode** → **Save**.
8. Record the **Client ID** (V3) and **Client Secret** (V4). The secret is displayed **once only**, so pass it straight to the AWS admin over a secure channel.

**Output:** V2, V3, V4.

---

### Phase 2 — AWS IAM Identity Center: Trusted Token Issuer
**Owner:** AWS / Cloud Administrator

1. IAM Identity Center → **Settings → Authentication → Create trusted token issuer**.
2. **Issuer URL:** the bare issuer, with **no** `/.well-known/...` suffix:
   ```
   https://{yourOktaDomain}/oauth2/default
   ```
3. **Attribute mapping:** Identity provider attribute = **Email**; IAM Identity Center attribute = **Email**.
4. Create it, then record the **Trusted Token Issuer ARN** (V5).

**Output:** V5.

---

### Phase 3 — AWS Secrets Manager & IAM Role
**Owner:** AWS / Cloud Administrator

**3a. Store the Okta client credentials**
1. Secrets Manager → **Store a new secret → Other type of secret → Plaintext**:
   ```json
   {
     "client_id": "<Okta Client ID (V3)>",
     "client_secret": "<Okta Client Secret (V4)>"
   }
   ```
2. Record the **secret ARN** (V6).

**3b. Create the IAM role Quick assumes to read the secret**
1. IAM → **Roles → Create role → Custom trust policy**:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": { "Service": "<region>.prod.appintegrations.plato.aws.internal" },
         "Action": "sts:AssumeRole",
         "Condition": {}
       }
     ]
   }
   ```
2. Attach the **inline permissions policy**:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "BasePermissions",
         "Effect": "Allow",
         "Action": [
           "secretsmanager:GetSecretValue",
           "sso:DescribeTrustedTokenIssuer"
         ],
         "Resource": "*"
       }
     ]
   }
   ```
   > **Hardening (recommended):** replace `"Resource": "*"` with the specific secret ARN (V6) and trusted token issuer ARN (V5) once the setup is confirmed to work. If the secret uses a customer-managed KMS key, also grant `kms:Decrypt` on that key.
3. Record the **role ARN** (V7).

**Output:** V6, V7.

---

### Phase 4 — Amazon Quick: Extension Access
**Owner:** Amazon Quick Administrator
**Needs:** V3, V5, V6, V7, V8

1. Profile icon → **Manage account → Permissions → Extension access → New extension access**.
2. **First time only: Trusted Token Issuer Setup.** This is a one-time setting for the whole Quick account.
   - Trusted Token Issuer ARN: **V5**
   - Aud claim: Okta **Client ID (V3)**
3. Select **Microsoft Teams → Next**.
4. Fill in:
   - **Name / Description:** e.g. `HR Assist – Teams` (V9)
   - **M365 tenant ID:** V8
   - **Secrets Role ARN:** V7
   - **Secrets ARN:** V6
5. Choose **Add**.

**Output:** Extension access V9.

---

### Phase 5 — Microsoft 365: Tenant ID
**Owner:** Azure / M365 Administrator (do this **before** Phase 4)

- Azure Portal → **Microsoft Entra ID → Overview / Properties → Tenant ID**, **or**
- Microsoft 365 admin center → **Settings → Org settings → Organization profile**.

> **No Entra app registration is needed** for the Extension when Okta is the Identity Center identity source. The Okta app from Phase 1 does that job. The tenant ID only tells Quick which M365 tenant to deploy into.

**Output:** V8.

---

### Phase 6 — Amazon Quick: Deploy the Extension
**Owner:** Quick Author

1. **Connections → Extensions** → find extension access **V9** → complete the installation.
2. Deployment settings available: **Name**, **Description**, **Installation type**.
3. Make sure the **HR Assist agent and its Space are shared** with the pilot users (Viewer role).
4. Copy the **install link** for the M365 admin (Phase 7).

> There is **no setting to pin HR Assist as the default agent** for the extension. See [§10](#10-known-limitations).

---

### Phase 7 — Microsoft 365 / Teams: Admin Consent & Availability
**Owner:** M365 Global Admin (or a role holding `AppCatalog.ReadWrite.All`)

1. Open the install link from Phase 6 and sign in as Global Admin.
2. **Teams Admin Center → Teams apps → Manage apps** → find **Amazon Quick** → **Permissions** tab → **Grant admin consent**.
3. Check that the page shows *"Admin consent granted for all required permissions."*
   > AWS has already registered "Amazon Quick" as a multi-tenant Teams app, so this step **grants consent**. It does not create a bot.
4. **Recommended:** **Manage apps → Amazon Quick → Edit availability** → limit it to the **HR pilot group** (V10) instead of the whole org.
5. **Optional:** create or modify a **Teams app setup policy** to **pre-pin** Amazon Quick for the pilot group, so users don't have to search for it.

---

## 7. Permissions & Approvals Summary

### 7.1 Permissions

| System | Permission / access | Granted to | Purpose |
|---|---|---|---|
| Okta | Super/App Admin | Okta admin | Create the OIDC app, assign groups |
| AWS IAM Identity Center | Admin (`sso:CreateTrustedTokenIssuer`, etc.) | AWS admin | Create the trusted token issuer |
| AWS Secrets Manager | `secretsmanager:CreateSecret` | AWS admin | Store the Okta client credentials |
| AWS IAM | `iam:CreateRole`, `iam:PutRolePolicy` | AWS admin | Create the secrets role |
| IAM role (V7) | `secretsmanager:GetSecretValue`, `sso:DescribeTrustedTokenIssuer` | Quick App Integrations service principal | Runtime token exchange |
| Amazon Quick | Admin | Quick admin | Extension access, trusted token issuer setup |
| Amazon Quick | Author / Owner of HR Assist | Quick author | Deploy the extension, share the agent |
| Amazon Quick | Viewer on HR Assist agent and Space | Pilot employees | Use the agent |
| Microsoft 365 | Global Admin or `AppCatalog.ReadWrite.All` | M365 admin | Admin consent for Amazon Quick |
| Teams Admin Center | Teams Administrator | Teams admin | Availability and setup policies |

### 7.2 Approvals to get before go-live

| # | Approval | Approver | Status |
|---|---|---|---|
| AP1 | Use of Amazon Quick Enterprise and licences for pilot users | Budget owner / IT | ☐ |
| AP2 | Okta app integration and group assignment | Identity / IAM team | ☐ |
| AP3 | AWS resources (trusted token issuer, secret, IAM role) | Cloud governance | ☐ |
| AP4 | Admin consent for the Amazon Quick Teams app | M365 Global Admin | ☐ |
| AP5 | Pilot group and rollout scope | HR stakeholder | ☐ |
| AP6 | Data handling review: KB content, Gmail/Slack escalation, Slack redaction | Security / Compliance | ☐ |
| AP7 | Action permission settings (Always Ask vs Trust) | Security + HR | ☐ |

---

## 8. Rollout Strategy

1. **Internal test:** project team only. Run the full test plan inside Teams DMs.
2. **HR pilot:** limit Teams availability and Okta assignment to the HR pilot group. Pre-pin the app.
3. **Feedback & tuning:** adjust the system prompt, KB scope and escalation routing.
4. **Wider rollout:** extend Okta group assignment, Identity Center/Quick licensing and Teams availability together. All three must stay in sync.

---

## 9. End-User Guide

1. **Add the app:** Teams → **Apps** → search **"Quick"** → **Add** (skip this if it's pre-pinned).
2. **Open a direct message** with Amazon Quick. **Use DMs for HR questions.**
3. After your first message, click the **gear icon** → choose the **HR Assist** agent/Space.
4. Ask your question, e.g. *"How many casual leaves do I get per year?"*
5. If the bot offers to escalate to HR, **check the email preview** and confirm or decline.
6. Conversation history can be reviewed in the **Amazon Quick web app**, not in Teams.

**In channels:** type `@Amazon Quick <question>`. Channel mentions use the **default assistant, not HR Assist**, and **escalation actions don't work** there. To add the app to a channel: **… → Manage team → Apps → + Add an app → Amazon Quick**.

---

## 10. Known Limitations

| Limitation | Impact | Mitigation |
|---|---|---|
| No admin setting to fix HR Assist as the extension's default agent | Users must pick HR Assist themselves | Tell users to choose it via the gear icon in DMs |
| Channel `@mentions` always use the default "My Assistant" and all Spaces | Answers may not follow HR Assist's rules | Tell employees to use DMs for HR questions |
| **Actions (Gmail/Slack escalation) work only in DMs** | No escalation from channels | Use DMs |
| No visuals for structured data, no web search | Text-only answers | Acceptable for an HR FAQ bot |
| Cannot auto-reply in channels; not available in Teams meetings | — | — |
| History is not shown in Teams | Users check the Quick web app | Document this in the user guide |
| Unknown whether the gear-icon agent choice persists across conversations | Possible repeated selection | To be tested ([§15](#15-open-items)) |

**Workaround to avoid:** reconfiguring the org-wide default "My Assistant" to behave like HR Assist. That would affect every Quick use case across the organisation.

---

## 11. Optional: Teams Connector (Outbound Actions)

Only needed if HR Assist should **post into Teams** (for example, sending escalation alerts to an HR Teams channel instead of Slack).

| Item | Detail |
|---|---|
| Setup | Quick → **Integrations → Microsoft Teams** (action connector) |
| Auth | Separate **Microsoft Entra ID app registration** with Microsoft Graph permissions |
| Approvals | Entra admin consent for Graph permissions. These are **broad** (calendars, channels, members, transcripts, recordings, user profiles). |
| Guidance | Request only the permissions needed for the configured actions. Get a separate security review. Keep the same redaction rules used for Slack. |

---

## 12. Security Considerations

- **Client secret handling:** the Okta secret goes only into Secrets Manager. Rotate it periodically and update the secret value when you do.
- **Least privilege:** scope the IAM role's `Resource` to the specific secret and issuer ARNs (see Phase 3b).
- **Access scoping:** Okta group assignment, Quick licensing and Teams availability should all match the same approved group.
- **Agent sharing:** share HR Assist and its Space as **Viewer** only. Keep Owners to the project team.
- **Action trust:** the Slack alert is set to run silently ("Trust"). Periodically check that `#hr-escalations` posts contain **no employee-identifying details**. Keep Gmail on confirmation.
- **Sensitive data:** the agent must not forward sensitive identifiers word for word (test F1).
- **Audit:** use CloudTrail for IAM Identity Center/Secrets Manager access and the Okta system log for sign-ins.

---

## 13. Validation Checklist

| # | Check | Result |
|---|---|---|
| T1 | Pilot user can add Amazon Quick in Teams and sign in through Okta without errors | ☐ |
| T2 | Non-pilot user cannot see or use the app (availability policy works) | ☐ |
| T3 | User can choose HR Assist via the gear icon in a DM | ☐ |
| T4 | Policy question (A1) is answered with a citation in the Teams DM | ☐ |
| T5 | Escalation (B1 → E1) sends the Gmail email after confirmation and posts a redacted Slack alert | ☐ |
| T6 | Decline flow (D1) sends nothing | ☐ |
| T7 | Channel `@mention` works but uses the default assistant (expected) | ☐ |
| T8 | Whether agent choice persists in a new DM conversation is recorded | ☐ |
| T9 | Full test plan run in Teams DMs | ☐ |

---

## 14. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Okta error on sign-in: redirect URI mismatch | Callback URI missing or wrong Region | Add the exact regional callback URI in the Okta app (Phase 1.5) |
| User signs in but gets "no access" in Quick | User not in Identity Center or not licensed in Quick; email mismatch | Check SCIM provisioning and Quick group licensing; check the email attribute matches |
| Trusted token issuer errors | Issuer URL includes `/.well-known/...`, or wrong authorization server | Use `https://{oktaDomain}/oauth2/default` exactly |
| Extension access fails to save | Wrong role/secret ARN, trust policy principal Region mismatch | Check V6/V7 and the `<region>` in the trust policy |
| Amazon Quick app not visible in Teams | Consent not granted or app blocked/limited | Grant consent; check the availability and permission policies |
| HR Assist not in the gear-icon list | Agent/Space not shared with the user | Share as Viewer |
| Escalation doesn't trigger | Used in a channel mention, or action permission denied | Use a DM; check the action connector permissions |

---

## 15. Open Items

- [ ] Confirm the Quick account Region (V1) and that it supports the Teams extension.
- [ ] Confirm the Okta → IAM Identity Center provisioning (SCIM) setup and the email attribute mapping.
- [ ] Test whether the gear-icon agent choice persists across new conversations.
- [ ] Decide on the escalation channel for Teams users: Slack (current) or the Teams connector ([§11](#11-optional-teams-connector-outbound-actions)).
- [ ] Confirm Quick licensing cost for the pilot and the full rollout.
- [ ] Get Microsoft Learn citations for the Teams Admin Center steps if a fully sourced reference is needed.

---

## 16. References

- AWS — Teams extension overview: `docs.aws.amazon.com/quick/latest/userguide/teams-extension.html`
- AWS — Teams extension author guide: `docs.aws.amazon.com/quick/latest/userguide/teams-extension-author-guide.html`
- AWS — Teams extension user guide: `docs.aws.amazon.com/quick/latest/userguide/teams-extension-user-guide.html`
- AWS — Microsoft Teams integration (connector): `docs.aws.amazon.com/quick/latest/userguide/microsoft-teams-integration.html`
- A mirrored doc tree also exists under `/quicksuite/latest/...`
- Microsoft Learn — Teams Admin Center: manage apps, admin consent, app setup and availability policies.
