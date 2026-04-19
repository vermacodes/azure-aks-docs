---
title: Cluster authentication concepts in Azure Kubernetes Service (AKS)
titleSuffix: Azure Kubernetes Service
description: Learn how Azure Kubernetes Service (AKS) authenticates Kubernetes API requests using Microsoft Entra ID, and how to lock down break-glass access with Conditional Access and PIM.
ms.topic: concept-article
ms.subservice: aks-security
ms.date: 04/19/2026
author: shashankbarsin
ms.author: shasb
ai-usage: ai-assisted
# Customer intent: "As a cluster operator, I want to understand how AKS authenticates callers to the Kubernetes API so that I can centralize identity in Microsoft Entra ID and govern break-glass access."
---

# Cluster authentication concepts in Azure Kubernetes Service (AKS)

This article describes how Azure Kubernetes Service (AKS) authenticates callers to the Kubernetes API — that is, who can connect to the control plane. It covers the recommended Microsoft Entra ID-based authentication path and how to lock down break-glass access.

For how AKS evaluates what an authenticated caller is *allowed to do*, see [Cluster authorization concepts](concepts-cluster-authorization.md).

For the other identity scenarios in AKS, see:

* [Managed identities in AKS](use-managed-identity.md) for cluster-to-Azure access (such as pulling images from ACR or attaching disks).
* [Microsoft Entra Workload ID overview](workload-identity-overview.md) for pod-to-Azure access (such as workloads calling Key Vault).

For an orientation across all four AKS identity scenarios, see [Access and identity options for AKS](concepts-identity.md).

## Authenticate cluster users with Microsoft Entra ID

Kubernetes itself doesn't provide an identity directory. Without an external identity provider, you'd need to manage local credentials per cluster, which doesn't scale and creates audit gaps.

We recommend deploying AKS clusters with [Microsoft Entra ID authentication for the control plane][entra-id-cp-auth]. With this integration, the cluster validates incoming Kubernetes API requests against Microsoft Entra ID and uses the caller's Entra identity for authorization decisions. Microsoft Entra ID centralizes the identity layer — any change in user or group status is automatically reflected in cluster access — and enables Conditional Access, multifactor authentication, and Privileged Identity Management.

![Cluster-level authentication for Microsoft Entra integration with AKS](media/operator-best-practices-identity/cluster-level-authentication-flow.png)

For setup, see [Enable Microsoft Entra ID authentication for the AKS control plane][entra-id-cp-auth].

## Disable local accounts and enforce Conditional Access

Local accounts use a built-in cluster admin certificate that bypasses Entra ID, breaking centralized audit and Conditional Access. In production, disable local accounts on AKS clusters so that all access flows through Microsoft Entra ID. Apply Conditional Access policies (multifactor authentication, location restrictions) and use Privileged Identity Management for break-glass access. For details, see:

* [Manage local accounts in AKS](local-accounts.md)
* [Cluster and node access control with Conditional Access](access-control-managed-azure-ad.md)
* [Cluster and node access control with Privileged Identity Management](privileged-identity-management.md)

## Requirements and limitations

* The Microsoft Entra tenant configured for cluster authentication must be the same as the tenant of the subscription that holds the AKS cluster.
* For non-interactive logins or older `kubectl` versions, use the [`kubelogin`](https://github.com/Azure/kubelogin) plugin.

## Next steps

* [Enable Microsoft Entra ID authentication for the AKS control plane][entra-id-cp-auth]
* [Cluster authorization concepts](concepts-cluster-authorization.md)
* [Managed identities in AKS](use-managed-identity.md)
* [Microsoft Entra Workload ID overview](workload-identity-overview.md)

<!-- INTERNAL LINKS -->
[entra-id-cp-auth]: entra-id-control-plane-authentication.md
