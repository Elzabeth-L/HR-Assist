# HR Assist — SharePoint Embedding Deployment Guide

> **Deployment pattern:** Embed the Amazon Quick HR Assist chat agent in a **SharePoint Online** page. Users sign in once to SharePoint and never see a separate Quick login.
> **Status:** **Paused.** Researched but not yet pursued.

This guide covers the architecture, prerequisites, configuration, permissions, approvals, security and validation for embedding HR Assist in SharePoint.

> Items marked **⚠ Verify** must be checked against current AWS and Microsoft documentation before implementation.

---

## Contents

- [1. Architecture Overview](#1-architecture-overview)
- [2. Key Constraints](#2-key-constraints)
- [3. Design Decisions](#3-design-decisions)
- [4. Prerequisites](#4-prerequisites)
- [5. Roles \& Responsibilities](#5-roles--responsibilities)
- [6. Configuration Values Register](#6-configuration-values-register)
- [7. Implementation Steps](#7-implementation-steps)
  - [Phase 1 — Amazon Quick: Account Configuration](#phase-1--amazon-quick-account-configuration)
  - [Phase 2 — AWS IAM: Embed Role](#phase-2--aws-iam-embed-role)
  - [Phase 3 — Embed Backend (API)](#phase-3--embed-backend-api)
  - [Phase 4 — Microsoft Entra ID: App Registration](#phase-4--microsoft-entra-id-app-registration)
  - [Phase 5 — SPFx Web Part](#phase-5--spfx-web-part)
  - [Phase 6 — SharePoint Admin: Deploy \& Approve](#phase-6--sharepoint-admin-deploy--approve)
  - [Phase 7 — Publish on the HR Page](#phase-7--publish-on-the-hr-page)
- [8. Permissions \& Approvals](#8-permissions--approvals)
  - [8.1 Permissions](#81-permissions)
  - [8.2 Approvals before go-live](#82-approvals-before-go-live)
- [9. Security Controls](#9-security-controls)
- [10. Cost \& Rollout](#10-cost--rollout)
- [11. Validation Checklist](#11-validation-checklist)
- [12. Troubleshooting](#12-troubleshooting)
- [13. Fallback: Link-Out Instead of Embed](#13-fallback-link-out-instead-of-embed)
- [14. Open Items](#14-open-items)
- [15. References](#15-references)

## 1. Architecture Overview

```
 Employee browser (signed in to SharePoint via Entra ID / Okta)
   ▼
 SharePoint page ──► HR Assist SPFx web part
   │  gets an Entra access token for the "HR Assist Embed API" (no extra login)
   ▼
 AWS embed backend (CloudFront + WAF → API Gateway → Lambda)
   │  1. Validate the Entra token
   │  2. Map email → Quick user (JIT RegisterUser if missing)
   │  3. GenerateEmbedUrlForRegisteredUser (AllowedDomains = SharePoint)
   ▼
 Web part renders the short-lived embed URL in an iframe
   ▼
 Amazon Quick ──► HR Assist agent ──► HR knowledge base ──► Bedrock
```

## 2. Key Constraints

| # | Constraint | Implication |
|---|---|---|
| C1 | Chat-agent embedding needs a **registered Quick user**; there is no anonymous mode | Every employee needs a Quick user, pre-provisioned or created JIT |
| C2 | Embed URLs are **short-lived and user-specific** | The built-in SharePoint *Embed* web part won't work; a custom SPFx web part must fetch a fresh URL each load |
| C3 | **Domain allowlisting** is enforced | The SharePoint domain must be allowlisted in Quick and passed in the embed call |
| C4 | Licensing is **per user** | Cost grows with every unique employee who opens the chat |
| C5 | SharePoint restricts iframes/scripts (HTML Field Security, CSP) | Bundle the embedding SDK in the SPFx package. **⚠ Verify** CSP settings |

## 3. Design Decisions

| # | Decision | Options | Recommendation |
|---|---|---|---|
| D1 | Identity federation | (a) Lambda validates the Entra token, calls Quick with its own role · (b) Entra as IAM OIDC provider + `AssumeRoleWithWebIdentity` · (c) `GenerateEmbedUrlForRegisteredUserWithIdentity` via IAM Identity Center | (a) simplest; (c) if the Teams trusted token issuer exists. **⚠ Verify** (c) supports chat agents |
| D2 | User provisioning | Pre-provision via Identity Center groups, or JIT `RegisterUser` | Pre-provision for cost control |
| D3 | Build approach | AWS reference architecture (CDK) or direct IAM/SAML federation | AWS reference architecture (CDK) |
| D4 | Embed scope | HR Assist agent only, or full workspace | **HR Assist only** |
| D5 | Placement | HR Help page, HR hub site, or sitewide | One **HR Help** page |

## 4. Prerequisites

| # | Requirement | Details |
|---|---|---|
| P1 | **Amazon Quick Enterprise** | Required for embedding and IAM-integrated users |
| P2 | AWS account | Rights to deploy API Gateway, Lambda, WAF, CloudFront, IAM, DynamoDB |
| P3 | **SharePoint Online** | Tenant or site-collection App Catalog |
| P4 | **Microsoft Entra ID** | SharePoint sign-in (federated with Okta if applicable) |
| P5 | SPFx toolchain | Node.js, Yeoman SharePoint generator. **⚠ Verify** supported versions |
| P6 | Quick licences | For all target users |
| P7 | HR Assist agent ready | Built, tested, knowledge base linked, shared with target group as **Viewer**, agent ID recorded |
| P8 | Actions in embed | **⚠ Verify** that Gmail/Slack escalation actions work in embedded sessions |

## 5. Roles & Responsibilities

| Role | Responsible for | Phases |
|---|---|---|
| Quick Administrator | Domain allowlist, users/licences, agent sharing | 1 |
| AWS / Cloud Administrator | IAM role, backend infrastructure, CloudTrail | 2, 3 |
| Backend Developer | Token validation, JIT provisioning, embed URL generation | 3 |
| Entra ID Administrator | App registration, API scope, user assignment | 4 |
| SPFx Developer | Web part that fetches and renders the embed URL | 5 |
| SharePoint Administrator | App Catalog deployment, API access approval, CSP/HTML Field Security | 6 |
| HR Site Owner | Add the web part to the HR page | 7 |
| Security / Compliance | Threat model, pen test, data flow review | Before go-live |
| Budget Owner | Approve per-user licensing cost | Before go-live |

## 6. Configuration Values Register

**Never store secrets here.**

| # | Value | Produced in | Used in | ☐ |
|---|---|---|---|---|
| S1 | AWS account ID and Quick Region | — | 2, 3 | ☐ |
| S2 | Quick namespace (usually `default`) | — | 3 | ☐ |
| S3 | HR Assist agent ID / ARN | 1 | 3 | ☐ |
| S4 | SharePoint domain `https://<tenant>.sharepoint.com` | — | 1, 3 | ☐ |
| S5 | Embed backend IAM role ARN | 2 | 3 | ☐ |
| S6 | Embed API base URL | 3 | 5 | ☐ |
| S7 | Entra tenant ID | 4 | 3 | ☐ |
| S8 | Entra Application (client) ID of the embed API | 4 | 3, 5 | ☐ |
| S9 | Application ID URI + scope, e.g. `api://hr-assist-embed/Embed.Access` | 4 | 5 | ☐ |
| S10 | Target user group | 1, 4 | 1, 6 | ☐ |
| S11 | Embed session lifetime (minutes) | 3 | 3 | ☐ |

## 7. Implementation Steps

### Phase 1 — Amazon Quick: Account Configuration
**Owner:** Quick Administrator

1. Manage Quick → **Security → Manage domains** → add `https://<tenant>.sharepoint.com` (S4).
2. Provision the target group (S10) through IAM Identity Center, or enable JIT (D2).
3. Set new/JIT users to the lowest (viewer) role.
4. Share the HR Assist agent and its Space with the target group as **Viewer**.
5. Record the agent ID/ARN (S3).

### Phase 2 — AWS IAM: Embed Role
**Owner:** AWS / Cloud Administrator

Permissions policy (template; **⚠ Verify** action names and ARN formats for chat-agent embedding):
```json
{ "Version": "2012-10-17", "Statement": [
  { "Sid": "GenerateEmbedUrl", "Effect": "Allow",
    "Action": ["quicksight:GenerateEmbedUrlForRegisteredUser"],
    "Resource": "arn:aws:quicksight:<region>:<account-id>:user/<namespace>/*",
    "Condition": { "ForAllValues:StringEquals": {
      "quicksight:AllowedEmbeddingDomains": ["https://<tenant>.sharepoint.com"] } } },
  { "Sid": "JitProvisioning", "Effect": "Allow",
    "Action": ["quicksight:DescribeUser", "quicksight:RegisterUser"],
    "Resource": "arn:aws:quicksight:<region>:<account-id>:user/<namespace>/*" }
] }
```
- Drop `JitProvisioning` if users are pre-provisioned.
- Option (b): add an IAM OIDC provider for `https://login.microsoftonline.com/<tenant-id>/v2.0` (audience S8) and allow `sts:AssumeRoleWithWebIdentity`.
- Add CloudWatch Logs permissions for Lambda. Record the role ARN (S5).

### Phase 3 — Embed Backend (API)
**Owner:** Backend Developer + AWS Administrator

Stack: **CloudFront + WAF → API Gateway → Lambda** (AWS reference architecture, CDK).

Lambda request flow:
1. **Validate the Entra token:** signature (JWKS), `iss`, `aud` = S8, expiry, scope `Embed.Access`.
2. **Replay protection:** reject reused nonces (DynamoDB with TTL).
3. **Resolve the user:** email → `DescribeUser`; if missing and JIT is enabled → `RegisterUser` (viewer role) and log it.
4. **Generate the embed URL** with `GenerateEmbedUrlForRegisteredUser`:
   - `UserArn` = resolved user
   - `ExperienceConfiguration` = chat experience scoped to HR Assist (S3). **⚠ Verify** the exact key
   - `AllowedDomains` = [S4]
   - `SessionLifetimeInMinutes` = S11 (e.g. 60)
5. **Return** the URL. Never cache it.

Hardening: WAF rate limits, API Gateway throttling, CORS limited to S4, CloudWatch logs, CloudTrail. Record the API URL (S6).

### Phase 4 — Microsoft Entra ID: App Registration
**Owner:** Entra ID Administrator

1. **App registrations → New registration:** `HR Assist Embed API` (single tenant).
2. **Expose an API:** Application ID URI `api://hr-assist-embed`, scope `Embed.Access`.
3. Make sure access tokens include the `email`/`upn` claim.
4. Enterprise applications → **Assignment required = Yes** → assign the target group (S10).
5. Record S7, S8, S9.

### Phase 5 — SPFx Web Part
**Owner:** SPFx Developer

1. Scaffold the web part `HR Assist Chat`.
2. Declare the API permission in `config/package-solution.json`:
   ```json
   "webApiPermissionRequests": [
     { "resource": "HR Assist Embed API", "scope": "Embed.Access" }
   ]
   ```
3. Use `AadHttpClient` to call `POST {S6}/embed-url` with a fresh nonce.
4. Render the URL with `amazon-quicksight-embedding-sdk`, **bundled** in the package (not loaded from a CDN).
5. Show a friendly error and an HR contact link if the embed fails. Fetch a new URL on every load.
6. Build the `.sppkg` package.

### Phase 6 — SharePoint Admin: Deploy & Approve
**Owner:** SharePoint Administrator

1. Upload the `.sppkg` to the **App Catalog** → **Deploy** (all sites or the HR site only).
2. SharePoint admin center → **Advanced → API access** → approve `HR Assist Embed API / Embed.Access`.
3. If required, allow the Quick embed domain under **HTML Field Security**. **⚠ Verify** the host domain.
4. Check the tenant's CSP / trusted script sources. **⚠ Verify.**

### Phase 7 — Publish on the HR Page
**Owner:** HR Site Owner

1. Add the **HR Assist Chat** web part to the **HR Help** page.
2. Limit page permissions to the pilot audience.
3. Publish.

## 8. Permissions & Approvals

### 8.1 Permissions

| System | Permission | Granted to | Purpose |
|---|---|---|---|
| Amazon Quick | Admin | Quick admin | Domains, users, sharing |
| Amazon Quick | Viewer on HR Assist agent + Space | Target employees | Use the agent |
| AWS IAM | `quicksight:GenerateEmbedUrlForRegisteredUser` (domain-conditioned) | Embed backend role | Issue embed URLs |
| AWS IAM | `quicksight:DescribeUser`, `quicksight:RegisterUser` | Embed backend role (JIT only) | Provision users |
| AWS IAM | `sts:AssumeRoleWithWebIdentity` | Entra OIDC trust (option b only) | Federation |
| Entra ID | Application Administrator | Entra admin | App registration |
| Entra ID | `Embed.Access` delegated scope | SharePoint Online Client Extensibility principal | SPFx → API calls |
| SharePoint | SharePoint Administrator | SP admin | Deploy package, approve API access |
| SharePoint | Site Owner | HR site owner | Add the web part |

### 8.2 Approvals before go-live

| # | Approval | Approver | ☐ |
|---|---|---|---|
| AP1 | Per-user Quick licensing (incl. JIT growth) | Budget owner | ☐ |
| AP2 | AWS backend resources and IAM role | Cloud governance | ☐ |
| AP3 | Entra app registration and scope | Identity team | ☐ |
| AP4 | SPFx package deployment | SharePoint admin / change board | ☐ |
| AP5 | SharePoint API access (`Embed.Access`) | SharePoint admin | ☐ |
| AP6 | Security review / pen test | Security | ☐ |
| AP7 | Data handling (KB content, escalations, logs) | Compliance / HR | ☐ |
| AP8 | Page placement and audience | HR stakeholder | ☐ |

## 9. Security Controls

- **Identity:** verify the Entra token signature, never bare claims; nonce-based replay protection; Entra "Assignment required" (Phases 3, 4).
- **Embed URLs:** short-lived, single-use, scoped to HR Assist only, allowlisted to the SharePoint domain (Phases 1, 3, 5).
- **Least privilege:** embed-only IAM role; JIT users get a minimal viewer role (Phases 1, 2).
- **API protection:** WAF rate limiting, API Gateway throttling, CORS limited to SharePoint (Phase 3).
- **Audit:** CloudTrail record of every JIT user and embed URL (Phase 3).

## 10. Cost & Rollout

**Cost**
- Every unique employee who opens the widget becomes a **licensed Quick user**.
- AWS backend costs (API Gateway, Lambda, WAF, CloudFront, DynamoDB) are low at portal traffic levels.
- Control cost by limiting the audience, pre-provisioning groups and removing inactive users.

**Rollout**
1. **Dev:** test SharePoint site + non-production Quick account.
2. **Pilot:** HR team only.
3. **Security review** and fixes.
4. **Wider rollout:** expand the audience and monitor licence counts.

## 11. Validation Checklist

| # | Check | ☐ |
|---|---|---|
| T1 | Pilot user opens the HR page → chat loads with no extra login | ☐ |
| T2 | Non-assigned user → "no access" message, no embed URL issued | ☐ |
| T3 | Reused URL or URL opened on another domain → rejected | ☐ |
| T4 | Tampered/expired token → 401 | ☐ |
| T5 | Replayed nonce → rejected | ☐ |
| T6 | First-time user → JIT user created with viewer role and logged | ☐ |
| T7 | Embed shows only HR Assist | ☐ |
| T8 | Policy question answered with a citation | ☐ |
| T9 | Escalation flow works in the embed. **⚠ Verify** | ☐ |
| T10 | WAF rate limit triggers under load | ☐ |

## 12. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Blank iframe / refused to connect | Domain not allowlisted or `AllowedDomains` mismatch | Use the exact `https://<tenant>.sharepoint.com` in both places |
| "API access not approved" | Request pending in SharePoint admin center | Approve under **Advanced → API access** |
| 401 from backend | Wrong `aud`/`iss`, v1 vs v2 token | Align validation with the token |
| Embed URL expired | URL cached or used late | Fetch a fresh URL on every load |
| "No permission" in chat | Agent/Space not shared | Share as Viewer |
| SDK script blocked | CSP / external CDN | Bundle the SDK |

## 13. Fallback: Link-Out Instead of Embed

If the embed isn't approved, add a button on the HR page that opens HR Assist in the Quick web app in a new tab. Users sign in through SSO (Identity Center + Okta). No backend, SPFx or API approval is needed, but users still need Quick licences.

## 14. Open Items

- [ ] Decide whether to pursue this now or after the Teams extension is validated.
- [ ] Confirm JIT user licensing at scale with the AWS account team.
- [ ] **⚠ Verify** the chat-agent `ExperienceConfiguration` and `...WithIdentity` support.
- [ ] **⚠ Verify** escalation actions in embedded sessions.
- [ ] Decide D1 (federation) and D2 (provisioning).
- [ ] Review SharePoint CSP / HTML Field Security with the SharePoint admin.

## 15. References

- AWS — Amazon Quick user guide, embedding: `docs.aws.amazon.com/quick/latest/userguide/`
- AWS — API reference: `GenerateEmbedUrlForRegisteredUser`, `GenerateEmbedUrlForRegisteredUserWithIdentity`, `RegisterUser`
- AWS — Reference architecture for embedding Quick chat (CDK)
- Microsoft Learn — SPFx: connect to Entra ID-secured APIs (`AadHttpClient`, `webApiPermissionRequests`), API access management, HTML Field Security
