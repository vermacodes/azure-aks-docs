---
title: Concepts - Access and identity in Azure Kubernetes Service (AKS)
description: Learn the four identity scenarios in Azure Kubernetes Service (AKS) — control-plane authentication and authorization, cluster identity, and workload identity — and where to find the right deep-dive doc for each.
ms.topic: concept-article
ms.subservice: aks-security
ms.date: 04/18/2026
author: shashankbarsin
ms.author: shasb
ai-usage: ai-assisted

# Customer intent: As a Kubernetes administrator, I want a clear orientation to the identity scenarios in AKS so that I can pick the right authentication and authorization model for each one.
---

# Access and identity options for Azure Kubernetes Service (AKS)

AKS uses identity in four distinct scenarios. Each scenario answers a different question and has its own configuration model. This article gives a brief introduction to each and points to the deep-dive documentation.

## The four identity scenarios in AKS

| Scenario | Question it answers | Deep-dive docs |
|---|---|---|
| **A. Control-plane authentication** | Who is the caller hitting the Kubernetes API? | [Microsoft Entra integration](#microsoft-entra-integration), [external identity providers](external-identity-provider-authentication-overview.md) |
| **B. Control-plane authorization** | What is the caller allowed to do once authenticated? | [Kubernetes API authorization concepts](concepts-kubernetes-api-authorization.md) |
| **C. Cluster identity (cluster → Azure)** | How does the AKS cluster act on Azure to manage resources on your behalf? | [Managed identities in AKS](use-managed-identity.md) |
| **D. Workload identity (pod → Azure)** | How do pods authenticate to Azure services such as Key Vault or Storage? | [Microsoft Entra Workload ID overview](workload-identity-overview.md) |

The rest of this article gives a brief orientation to each scenario.

## A. Control-plane authentication

Control-plane authentication establishes the identity of a user or service principal calling the Kubernetes API server. AKS supports:

* **Microsoft Entra ID (recommended).** Use Entra ID identities and groups to sign in to the cluster. AKS-managed Entra integration provisions and rotates the integration on your behalf. To enable, see [Use AKS-managed Microsoft Entra integration](enable-authentication-microsoft-entra-id.md).
* **Local accounts.** A built-in cluster admin certificate that bypasses Entra ID. We recommend disabling local accounts in production. See [Manage local accounts](manage-local-accounts-managed-azure-ad.md).
* **External identity providers.** Use an OIDC-compliant identity provider other than Microsoft Entra ID. See [External identity provider authentication](external-identity-provider-authentication-overview.md).

<a name='azure-ad-integration'></a>

### Microsoft Entra integration

Microsoft Entra ID is a cloud identity service that combines directory services, application access management, and identity protection. With AKS-managed Entra integration, you can grant Entra users or groups access to Kubernetes resources within a namespace or across the cluster.

![Microsoft Entra integration with AKS clusters](media/concepts-identity/aad-integration.png)

Authentication uses OpenID Connect on top of OAuth 2.0. The Kubernetes API server validates incoming tokens through a webhook against Microsoft Entra ID. The high-level flow is:

1. `kubectl` signs in the user with the Microsoft Entra client application.
1. Microsoft Entra ID issues an access token.
1. `kubectl` sends the token to the API server.
1. The API server's authentication webhook verifies the token signature against Microsoft Entra public signing keys.
1. The API server makes an authorization decision (see [Kubernetes API authorization concepts](concepts-kubernetes-api-authorization.md)).

For setup, see [Use AKS-managed Microsoft Entra integration](enable-authentication-microsoft-entra-id.md). For Conditional Access and Privileged Identity Management with cluster access, see [Cluster and node access control with Conditional Access](access-control-managed-azure-ad.md) and [Cluster and node access control with PIM](privileged-identity-management.md).

## B. Control-plane authorization

After a caller is authenticated, AKS authorizes the request using one (or both) of two models:

* **Kubernetes RBAC.** The native Kubernetes `Role` / `ClusterRole` / `RoleBinding` model evaluated by the API server. Permissions live in the cluster as Kubernetes manifests.
* **Microsoft Entra ID authorization.** An AKS authorization webhook delegates authorization decisions to Microsoft Entra ID using Azure role assignments, optionally extended with Azure ABAC conditions. Permissions are managed centrally in Microsoft Entra ID and can govern many clusters from a single role assignment at subscription, management group, or resource group scope.

For a side-by-side comparison and guidance on when to use each model, see [Kubernetes API authorization concepts](concepts-kubernetes-api-authorization.md).

<a name='kubernetes-rbac'></a>
<a name='azure-rbac-for-kubernetes-authorization'></a>
<a name='azure-rbac-to-authorize-access-to-the-aks-resource'></a>

### Authorization for the AKS resource (Azure Resource Manager)

In addition to authorizing calls to the Kubernetes API, you also need to authorize calls to Azure Resource Manager that manage the AKS resource itself — for example, scaling or upgrading the cluster, or pulling the `kubeconfig`. This is standard Azure RBAC against the `Microsoft.ContainerService` resource provider, separate from authorizing the Kubernetes API. See [Limit access to the cluster configuration file](control-kubeconfig-access.md) and the built-in roles in [Azure built-in roles](/azure/role-based-access-control/built-in-roles#containers).

## C. Cluster identity (cluster → Azure)

AKS clusters use Azure managed identities to act on Azure resources on your behalf — for example, to create load balancers, attach disks, or pull images from Azure Container Registry. The main identities are:

* **Control-plane identity.** Used by the cluster control plane to manage Azure resources for the cluster.
* **Kubelet identity.** Used by the kubelet on each node to authenticate to services such as Azure Container Registry.
* **Add-on identities.** Some AKS add-ons use their own managed identities.

For details on each identity type and how to use system-assigned vs user-assigned identities, see [Managed identities in AKS](use-managed-identity.md).

## D. Workload identity (pod → Azure)

Workload identity lets pods running in your AKS cluster authenticate to Microsoft Entra–protected Azure services (such as Key Vault, Storage, or Cosmos DB) without storing secrets in the cluster. AKS uses [Microsoft Entra Workload ID](workload-identity-overview.md), which projects a Kubernetes service account token federated to a Microsoft Entra application or user-assigned managed identity.

Don't use the deprecated [Microsoft Entra pod-managed identity](use-azure-ad-pod-identity.md) for new workloads.

## Decision guide

| Goal | Use these docs |
|---|---|
| Sign users into the cluster with Microsoft Entra ID | [Enable AKS-managed Entra integration](enable-authentication-microsoft-entra-id.md) |
| Govern who can do what in the Kubernetes API across many clusters | [Use Microsoft Entra ID authorization for the Kubernetes API](manage-entra-id-authorization.md) |
| Restrict access to specific custom resource types | [ABAC conditions in Entra ID authorization](manage-entra-id-authorization.md#restrict-custom-resource-access-using-abac-conditions-preview) |
| Author per-cluster, per-namespace permissions as Kubernetes manifests | [Use Kubernetes RBAC with Entra integration](azure-ad-rbac.md) |
| Let the cluster pull from ACR or attach disks | [Managed identities in AKS](use-managed-identity.md) |
| Let pods reach Key Vault or Storage without secrets | [Microsoft Entra Workload ID overview](workload-identity-overview.md) |
| Restrict who can download the cluster `kubeconfig` | [Limit access to cluster configuration file](control-kubeconfig-access.md) |

<a name='aks-service-permissions'></a>

## AKS service permissions reference

AKS needs certain Azure permissions to create and operate clusters on your behalf. Permissions are split between the identity creating and operating the cluster (typically a service principal or user assigned during cluster creation) and the [cluster identity](#c-cluster-identity-cluster--azure) used at runtime. For the built-in roles used to grant these permissions, see [Azure built-in roles for Containers](/azure/role-based-access-control/built-in-roles#containers). For a worked example of granting a service principal the permissions needed for a custom virtual network, see [Use a service principal with AKS](kubernetes-service-principal.md).

## Next steps

* [Best practices for authentication and authorization in AKS][operator-best-practices-identity]
* [Kubernetes API authorization concepts](concepts-kubernetes-api-authorization.md)
* [Use Microsoft Entra ID authorization for the Kubernetes API](manage-entra-id-authorization.md)
* [Managed identities in AKS](use-managed-identity.md)
* [Microsoft Entra Workload ID overview](workload-identity-overview.md)

For more information on core Kubernetes and AKS concepts, see the following articles:

* [Kubernetes / AKS clusters and workloads][aks-concepts-clusters-workloads]
* [Kubernetes / AKS security][aks-concepts-security]
* [Kubernetes / AKS virtual networks][aks-concepts-network]
* [Kubernetes / AKS storage][aks-concepts-storage]
* [Kubernetes / AKS scale][aks-concepts-scale]

<!-- LINKS - Internal -->
[aks-concepts-clusters-workloads]: concepts-clusters-workloads.md
[aks-concepts-security]: concepts-security.md
[aks-concepts-scale]: concepts-scale.md
[aks-concepts-storage]: concepts-storage.md
[aks-concepts-network]: concepts-network.md
[operator-best-practices-identity]: operator-best-practices-identity.md
