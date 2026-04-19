---
title: Best practices for managing authentication and authorization
titleSuffix: Azure Kubernetes Service
description: Learn the best practices for authenticating and authorizing access to the Kubernetes API in Azure Kubernetes Service (AKS) clusters.
ms.topic: best-practice
ms.subservice: aks-security
ms.date: 04/18/2026
author: shashankbarsin
ms.author: shasb
ai-usage: ai-assisted
# Customer intent: "As a cluster operator, I want best practices for authenticating and authorizing access to the Kubernetes API in AKS so that I can govern cluster access at scale and apply least privilege."
---

# Best practices for authentication and authorization in Azure Kubernetes Service (AKS)

This article focuses on best practices for **control-plane access** in AKS — that is, who can authenticate to the Kubernetes API and what they're allowed to do once they're in. For best practices on the other identity scenarios in AKS, see:

* [Managed identities in AKS](use-managed-identity.md) for cluster-to-Azure access (such as pulling images from ACR or attaching disks).
* [Microsoft Entra Workload ID overview](workload-identity-overview.md) for pod-to-Azure access (such as workloads calling Key Vault).

For an orientation across the four AKS identity scenarios, see [Access and identity options for AKS](concepts-identity.md).

You'll learn how to:

> [!div class="checklist"]
>
> * Authenticate cluster users with Microsoft Entra ID.
> * Authorize the Kubernetes API with Microsoft Entra ID role assignments at scale.
> * Use Kubernetes RBAC for fine-grained intra-cluster permissions.
> * Restrict custom resource access with ABAC conditions.
> * Disable local accounts and enforce Conditional Access and Privileged Identity Management.

## Authenticate cluster users with Microsoft Entra ID

> **Best practice guidance**
>
> Deploy AKS clusters with [AKS-managed Microsoft Entra integration][aks-aad]. Microsoft Entra ID centralizes the identity layer — any change in user or group status is automatically reflected in cluster access — and enables Conditional Access, multifactor authentication, and Privileged Identity Management.

Kubernetes itself doesn't provide an identity directory. Without an external identity provider, you'd need to manage local credentials per cluster, which doesn't scale and creates audit gaps. With AKS-managed Entra integration, the cluster validates incoming Kubernetes API requests against Microsoft Entra ID and uses the caller's Entra identity for authorization decisions.

![Cluster-level authentication for Microsoft Entra integration with AKS](media/operator-best-practices-identity/cluster-level-authentication-flow.png)

For setup, see [Use AKS-managed Microsoft Entra integration][aks-aad].

## Authorize the Kubernetes API with Microsoft Entra ID at scale

> **Best practice guidance**
>
> Use Microsoft Entra ID authorization for the Kubernetes API and assign roles at the **subscription, management group, or resource group scope** to govern many clusters from a single role assignment. Reserve Kubernetes RBAC for fine-grained intra-cluster permissions.

After a caller is authenticated, AKS authorizes them using one of two models — Kubernetes RBAC or Microsoft Entra ID authorization. We recommend Microsoft Entra ID authorization as the default for the following reasons:

* **Scale.** A single Azure role assignment at a higher scope (resource group, subscription, or management group) governs every current and future AKS cluster in that scope. With Kubernetes RBAC, you must apply manifests to every cluster individually.
* **Single identity plane.** The same Entra users and groups that govern access to your other Azure resources govern your Kubernetes API. There's no separate user directory to provision or rotate.
* **Conditional Access and PIM.** Cluster access automatically inherits your organization's existing Conditional Access policies and can be elevated just-in-time through PIM.
* **Centralized audit.** Role assignment changes are recorded in the Azure Activity Log alongside other Azure resource changes.

For built-in roles, custom role definitions, and step-by-step setup, see [Use Microsoft Entra ID authorization for the Kubernetes API](manage-entra-id-authorization.md). For a side-by-side comparison of authorization models, see [Kubernetes API authorization concepts](concepts-kubernetes-api-authorization.md).

## Use Kubernetes RBAC for fine-grained intra-cluster permissions

> **Best practice guidance**
>
> Use Kubernetes RBAC alongside Entra ID authorization for permissions that need to live as code with workload manifests, or for in-cluster service accounts.

Kubernetes RBAC is the right tool for intra-cluster, per-namespace permissions managed via GitOps and for the service accounts that workloads use to call the Kubernetes API. Use Microsoft Entra users and groups as subjects in Kubernetes `RoleBinding` and `ClusterRoleBinding` objects so that human identities still come from your central directory. For details, see [Use Kubernetes RBAC with Microsoft Entra integration](azure-ad-rbac.md).

## Restrict custom resource access with ABAC conditions

> **Best practice guidance**
>
> When you grant broad read access through Microsoft Entra ID authorization but want to restrict which custom resources (CRDs) the assignee can read, attach an Azure ABAC condition to the role assignment.

Without ABAC conditions, granting read on custom resources requires a wildcard like `Microsoft.ContainerService/managedClusters/*/read`, which covers every CRD on every cluster in scope. With ABAC, you can attach a condition that restricts access to specific CRD groups and kinds — for example, allow `templates.gatekeeper.sh` while blocking `kyverno.io` — without writing per-cluster Kubernetes RBAC manifests.

For background and step-by-step setup, see [Restrict custom resource access using ABAC conditions](manage-entra-id-authorization.md#restrict-custom-resource-access-using-abac-conditions-preview).

## Disable local accounts and enforce Conditional Access

> **Best practice guidance**
>
> In production, disable local accounts on AKS clusters so that all access flows through Microsoft Entra ID. Apply Conditional Access policies (multifactor authentication, location restrictions) and use Privileged Identity Management for break-glass access.

Local accounts use a built-in cluster admin certificate that bypasses Entra ID, breaking centralized audit and Conditional Access. Disable local accounts to ensure all cluster access is governed by your Entra ID policies. For details, see:

* [Manage local accounts in AKS](manage-local-accounts-managed-azure-ad.md)
* [Cluster and node access control with Conditional Access](access-control-managed-azure-ad.md)
* [Cluster and node access control with Privileged Identity Management](privileged-identity-management.md)

## Next steps

* [Use AKS-managed Microsoft Entra integration][aks-aad]
* [Use Microsoft Entra ID authorization for the Kubernetes API](manage-entra-id-authorization.md)
* [Use Kubernetes RBAC with Microsoft Entra integration][azure-ad-rbac]
* [Kubernetes API authorization concepts](concepts-kubernetes-api-authorization.md)
* [Managed identities in AKS](use-managed-identity.md)
* [Microsoft Entra Workload ID overview](workload-identity-overview.md)

For more information about cluster operations in AKS, see the following best practices:

* [Multi-tenancy and cluster isolation][aks-best-practices-cluster-isolation]
* [Basic Kubernetes scheduler features][aks-best-practices-scheduler]
* [Advanced Kubernetes scheduler features][aks-best-practices-advanced-scheduler]

<!-- INTERNAL LINKS -->
[aks-aad]: enable-authentication-microsoft-entra-id.md
[azure-ad-rbac]: azure-ad-rbac.md
[aks-best-practices-scheduler]: operator-best-practices-scheduler.md
[aks-best-practices-advanced-scheduler]: operator-best-practices-advanced-scheduler.md
[aks-best-practices-cluster-isolation]: operator-best-practices-cluster-isolation.md
