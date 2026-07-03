---
title: "Zero-Trust Identity Governance Platform on Microsoft Entra ID"
date: 2026-07-03
tags: [IAM, Entra-ID, Zero-Trust, PIM, Conditional-Access, Identity-Governance, SC-300, Azure-Security]
---

# Zero-Trust Identity Governance Platform on Microsoft Entra ID

**Eliminating long-lived static credentials and enforcing least-privilege, time-bound access for both human and non-human identities.**

`Microsoft Entra ID` `PIM` `Conditional Access` `Identity Protection` `Workload Identity Federation` `Entitlement Management` `Lifecycle Workflows` `Microsoft Graph API` `GitHub Actions OIDC` `PowerShell` `Python`

---

## Overview

This project implements an end-to-end identity governance architecture across Microsoft Entra ID, covering all four domains of Microsoft's SC-300 (Identity and Access Administrator) skills framework: identity management, authentication & access, application access, and identity governance.

The core design principle: **credentials should be short-lived, and privilege should be earned on demand — never standing.** Every component in this project traces back to that single idea, applied to a different layer of the identity stack.

**Key outcomes:**

| Metric | Before | After |
|---|---|---|
| CI/CD credential type | Long-lived client secret | OIDC token (~1hr lifetime, zero stored secrets) |
| Privileged roles as Permanent Active | 100% | 0% (all converted to PIM Eligible) |
| Average privilege activation time | Manual, ad hoc | Minutes, with enforced approval + MFA |
| Offboarding → full access revocation | Manual, days | Automated, minutes |
| MFA registration coverage | Baseline | +XX% |

---

## Architecture

```mermaid
flowchart TB
    subgraph Sources["Identity Sources"]
        A1["Human Identities<br/>(Entra ID Users/Groups)"]
        A2["Workload Identities<br/>(CI/CD, Apps, Services)"]
    end

    subgraph AuthLayer["Authentication & Access Layer"]
        B1["Conditional Access<br/>(MFA + Device Compliance)"]
        B2["Identity Protection<br/>(Risk-based Adaptive Access)"]
        B3["Workload Identity Federation<br/>(OIDC, no secrets)"]
    end

    subgraph PrivLayer["Privilege Layer"]
        C1["PIM Eligible Roles<br/>(time-bound + approval)"]
        C2["Entitlement Management<br/>(Access Packages)"]
    end

    subgraph GovLayer["Governance & Lifecycle"]
        D1["Access Reviews<br/>(periodic recertification)"]
        D2["Lifecycle Workflows<br/>(Joiner/Leaver automation)"]
    end

    subgraph ObsLayer["Observability"]
        E1["Entra ID Audit &<br/>Sign-in Logs"]
        E2["Microsoft Graph API<br/>+ Azure Workbook"]
    end

    A1 --> B1 --> C1
    A1 --> B2
    A2 --> B3 --> C1
    C1 --> D1
    A1 --> C2 --> D1
    D2 --> C1
    D2 --> C2
    D1 --> E1
    C1 --> E1
    B1 --> E1
    E1 --> E2
```

### Privilege activation flow (PIM)

```mermaid
sequenceDiagram
    actor User
    participant PIM as Entra ID PIM
    participant CA as Conditional Access
    participant Approver
    participant Log as Audit Log

    User->>PIM: Request role activation
    PIM->>CA: Enforce MFA + device compliance check
    CA-->>User: Challenge for MFA
    User->>PIM: Provide justification + MFA
    PIM->>Approver: Send approval request
    Approver->>PIM: Approve (time-bound, e.g. 4h)
    PIM->>Log: Record activation event
    PIM-->>User: Role active (auto-expires)
    Note over PIM,Log: Role automatically deactivates at expiry — no standing access
```

### Zero-secret CI/CD authentication (Workload Identity Federation)

```mermaid
sequenceDiagram
    participant GA as GitHub Actions
    participant GH as GitHub OIDC Provider
    participant Entra as Entra ID
    participant Azure as Azure Resource

    GA->>GH: Request OIDC token for this workflow run
    GH-->>GA: Short-lived signed JWT
    GA->>Entra: Present JWT (client-id, tenant-id — no secret)
    Entra->>Entra: Validate JWT against Federated Credential trust
    Entra-->>GA: Issue Azure AD access token
    GA->>Azure: Call ARM API with access token
    Note over GA,Azure: Token is scoped to this exact workflow run and repo/branch — cannot be replayed elsewhere
```

---

## Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| Identity Platform | Microsoft Entra ID (P2) | Core directory, PIM, Access Reviews require P2 |
| Privileged Access | Entra ID PIM | Time-bound, approval-gated role activation |
| Conditional Access | Entra ID Conditional Access | Risk-adaptive authentication enforcement |
| Risk Detection | Entra ID Identity Protection | User/sign-in risk scoring |
| Workload Identity | Federated Credentials + GitHub Actions OIDC | Secret-free CI/CD authentication |
| Entitlement Management | Access Packages, Catalogs | Self-service access with approval + expiry |
| Lifecycle Automation | Lifecycle Workflows | Automated joiner/leaver processing |
| Automation & Reporting | Microsoft Graph API (Python `msgraph-sdk`, PowerShell) | Data extraction, scripted validation |
| Observability | Azure Workbook / Power BI, Log Analytics | Metrics dashboard, audit trail |
| CI/CD | GitHub Actions | Deployment pipeline, OIDC token issuance |

---

## What This Project Demonstrates

**1. Application access management** — Zero-secret Workload Identity Federation between GitHub Actions and Entra ID via OIDC, replacing long-lived client secrets with per-run federated tokens scoped to a specific repository, branch, and environment.

**2. Adaptive authentication & access** — A layered Conditional Access policy set (baseline MFA, privileged-role hardening, risk-based response, legacy auth blocking, unmanaged device restriction) integrated with Identity Protection risk signals, with a protected break-glass account excluded from all policies.

**3. Privileged access governance** — Conversion of high-privilege roles from Permanent Active to PIM Eligible, with enforced MFA, approval workflows, and maximum activation duration — directly tied into Conditional Access for defense-in-depth.

**4. Identity lifecycle governance** — Full Access Package lifecycle (request → approval → time-bound grant → automatic expiry/recertification) and an automated Leaver workflow that disables accounts, revokes tokens, and strips group/role/PIM assignments within minutes of an offboarding trigger.

**5. Observability** — Metrics pulled via Microsoft Graph API (PIM activation logs, role assignment mix, MFA coverage, risky sign-in counts) visualized in an Azure Workbook, quantifying the before/after impact of each change.

---

## Design Decisions & Trade-offs

- **Why Workload Identity Federation over Managed Identity for CI/CD:** Managed Identity only works for Azure-hosted resources; Federation extends the same secret-free model to external platforms like GitHub Actions, making it the more general zero-credential pattern.
- **Why risk signals are configured inside Conditional Access rather than legacy Identity Protection policies:** Microsoft's current guidance consolidates risk-based conditions into Conditional Access for a single, unified policy surface rather than two separate enforcement points.
- **Why Eligible (not Permanent Active) for every sensitive role:** Standing privilege is the single largest reduction in blast radius available without impacting legitimate work — the cost is a few seconds of activation friction per use.

---

## Evidence & Screenshots

*(Embed screenshots here from your evidence folder, e.g.)*

```markdown
![PIM Eligible Role Configuration](../assets/images/03-02-pim-eligible-config.png)
*Privileged role converted from Permanent Active to Eligible, with 4-hour max activation and mandatory approval.*
```

---

## Alignment with SC-300 (Identity and Access Administrator)

| SC-300 Domain | Project Component |
|---|---|
| Implement identity management solutions | User/group provisioning via Graph API |
| Implement authentication and access management | Conditional Access policy set + Identity Protection |
| Implement access management for apps | Workload Identity Federation, App Registration scoping |
| Plan and implement identity governance | PIM, Entitlement Management, Access Reviews, Lifecycle Workflows |

---

## Repository

`[link to GitHub repo]`

## Related Project

See also: **[Cloud Security Baseline Automation & Compliance Pipeline](#)** — the infrastructure-side counterpart to this identity-side project, together forming a complete cloud security engineering narrative (infra compliance + identity governance).
