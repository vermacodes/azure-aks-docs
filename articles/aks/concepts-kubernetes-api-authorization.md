---
title: Kubernetes API authorization concepts in AKS
titleSuffix: Azure Kubernetes Service
description: Learn about the options for authorizing access to the Kubernetes API in Azure Kubernetes Service (AKS), including Kubernetes RBAC and Microsoft Entra ID authorization.
ms.topic: concept-article
ms.subservice: aks-security
ms.date: 04/18/2026
author: shashankbarsin
ms.author: shasb
ai-usage: ai-assisted

# Customer intent: "As a cluster operator, I want to understand the authorization options available for the Kubernetes API in AKS so that I can choose the right model for governing access at scale across my clusters."
---

# Kubernetes API authorization concepts in Azure Kubernetes Service (AKS)

After a caller is authenticated to the Kubernetes API server in your AKS cluster, AKS evaluates whether the caller is authorized to perform the requested action. AKS supports two authorization models for the Kubernetes API:

* **Kubernetes role-based access control (RBAC).** The native Kubernetes authorization model. Permissions are defined as `Role` and `ClusterRole` objects and granted to subjects through `RoleBinding` and `ClusterRoleBinding` objects stored in each cluster.
* **Microsoft Entra ID authorization.** An AKS authorization webhook that delegates authorization decisions to Microsoft Entra ID. Permissions are granted as Azure role assignments to Entra ID users, groups, or service principals, and can be optionally refined with [Azure ABAC conditions](/azure/role-based-access-control/conditions-overview).

You can use both models on the same cluster. This article describes when to use each one and the value Entra ID authorization provides for governing many clusters from a single identity plane.

## Option 1: Kubernetes RBAC

Kubernetes RBAC is the upstream Kubernetes authorization model. You author `Role` or `ClusterRole` objects that grant verbs (such as `get`, `list`, `create`) on resources (such as `pods`, `deployments`), and bind them to subjects (users, groups, or service accounts) using `RoleBinding` or `ClusterRoleBinding` objects. The Kubernetes API server's built-in RBAC authorizer evaluates these bindings on every request.

Use Kubernetes RBAC when you want:

* Fine-grained, intra-cluster access control authored as Kubernetes manifests alongside the workloads they protect.
* GitOps-managed authorization that lives in the same source of truth as your application configuration.
* Permissions for in-cluster service accounts that workloads use to call the Kubernetes API.

Kubernetes RBAC permissions are scoped to a single cluster. To apply the same policy to many clusters, you must apply the manifests to each cluster (typically through GitOps).

For background on the Kubernetes RBAC model, see the [upstream Kubernetes RBAC documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/). To use Microsoft Entra ID users and groups as subjects in Kubernetes `RoleBinding` and `ClusterRoleBinding` objects, see [Use Kubernetes RBAC with Microsoft Entra integration](azure-ad-rbac.md).

## Option 2: Microsoft Entra ID authorization for the Kubernetes API

With Entra ID authorization, AKS deploys an authorization webhook that delegates Kubernetes API authorization decisions to Microsoft Entra ID. When a request reaches the API server, the webhook calls the Entra ID `checkaccess` API to evaluate the caller's Azure role assignments (and any attached ABAC conditions) and returns an allow or deny decision.

![Entra ID authorization webhook flow for the Kubernetes API](media/concepts-identity/azure-rbac-k8s-authz-flow.png)

### Why use Entra ID authorization

Entra ID authorization gives you the following benefits over managing Kubernetes RBAC manifests on every cluster:

* **Single identity plane.** The same Microsoft Entra users, groups, and service principals that govern access to your Azure resources also govern access to your Kubernetes API. There's no separate user directory to provision or rotate.
* **Assign once, govern many clusters.** Azure role assignments can be made at **subscription, management group, or resource group scope**. A single role assignment at a resource group scope grants access to every current and future AKS cluster in that resource group. With Kubernetes RBAC, you must apply manifests to every cluster individually.
* **Conditional Access and Privileged Identity Management (PIM).** Cluster access automatically inherits your organization's existing Entra ID Conditional Access policies (such as multifactor authentication or location-based restrictions) and can be elevated just-in-time through PIM.
* **Centralized audit.** Every role assignment change is recorded in the Azure Activity Log alongside other Azure resource changes, so you have one audit trail for cluster access governance.
* **Fine-grained custom resource constraints.** With ABAC conditions, you can restrict access to specific custom resource (CRD) groups and kinds without writing per-cluster Kubernetes RBAC manifests.

### How permissions are granted

Entra ID authorization for the Kubernetes API has two layers:

* **Role assignments.** A built-in or custom Azure role is assigned to an Entra ID identity at a chosen Azure scope. AKS provides the following built-in roles:

    | Role | Description |
    |---|---|
    | Azure Kubernetes Service RBAC Reader | Read-only access to most objects in a namespace. Doesn't allow viewing roles, role bindings, or `Secrets`. |
    | Azure Kubernetes Service RBAC Writer | Read/write access to most objects in a namespace. Doesn't allow viewing or modifying roles or role bindings. |
    | Azure Kubernetes Service RBAC Admin | Read/write access to most resources in a namespace, plus the ability to create roles and role bindings within the namespace. |
    | Azure Kubernetes Service RBAC Cluster Admin | Full control over every resource in the cluster, across all namespaces. |

    For custom permission patterns, you can author custom role definitions that target specific Kubernetes API groups using the `Microsoft.ContainerService` resource provider's data actions. For details, see [Use Microsoft Entra ID authorization for the Kubernetes API](manage-entra-id-authorization.md).

* **ABAC conditions (preview).** A role assignment can carry an optional condition that filters access based on attributes of the request. For Kubernetes API authorization, you can filter access to custom resources by their API group and kind using the following attributes:

    * `Microsoft.ContainerService/managedClusters/customResources:group`
    * `Microsoft.ContainerService/managedClusters/customResources:kind`

    For example, you can grant a wildcard read role across the cluster but attach a condition that only allows reads on `templates.gatekeeper.sh` resources. For background on Azure ABAC, see [What are Azure role assignment conditions?](/azure/role-based-access-control/conditions-overview).

### When to use Entra ID authorization

Use Entra ID authorization when you want:

* Centralized access governance across many AKS clusters.
* Conditional Access, PIM, or Activity Log audit for cluster access.
* Fine-grained custom resource access policies that don't require per-cluster Kubernetes RBAC manifests.

### Comparison

| Capability | Kubernetes RBAC | Entra ID authorization | Entra ID authorization with ABAC conditions |
|---|---|---|---|
| Identity source | Kubernetes users, groups, service accounts | Microsoft Entra ID identities | Microsoft Entra ID identities |
| Scope of a single grant | One cluster | Resource, resource group, subscription, or management group | Resource, resource group, subscription, or management group |
| Multi-cluster governance | Apply manifests to each cluster (typically GitOps) | One role assignment at a higher scope governs many clusters | One role assignment at a higher scope governs many clusters |
| Conditional Access / PIM | Not supported | Inherited from Entra ID | Inherited from Entra ID |
| Audit trail | Cluster audit log | Azure Activity Log | Azure Activity Log |
| Filter access by CRD group or kind | Author per-CRD `Role` objects | Use wildcard roles (broad) | Use ABAC condition attributes |

### Requirements and limitations

* Entra ID authorization requires managed Microsoft Entra integration on the cluster. To enable, see [Use Microsoft Entra ID in AKS](managed-azure-ad.md).
* The Microsoft Entra tenant configured for cluster authentication must be the same as the tenant of the subscription that holds the AKS cluster.
* For non-interactive logins or older `kubectl` versions, use the [`kubelogin`](https://github.com/Azure/kubelogin) plugin.
* ABAC conditions are in preview. To use ABAC conditions, enable Entra ID authorization at cluster creation. Some clusters that have Entra ID authorization enabled post-creation might not evaluate ABAC conditions in the authorization webhook.

## Next steps

* [Use Microsoft Entra ID authorization for the Kubernetes API](manage-entra-id-authorization.md) — step-by-step setup, role assignment, and ABAC conditions how-to.
* [Use Kubernetes RBAC with Microsoft Entra integration](azure-ad-rbac.md) — bind Microsoft Entra users and groups in Kubernetes RBAC `RoleBinding` objects.
* [Access and identity options for AKS](concepts-identity.md) — overview of the four identity scenarios in AKS.
* [What are Azure role assignment conditions?](/azure/role-based-access-control/conditions-overview)
