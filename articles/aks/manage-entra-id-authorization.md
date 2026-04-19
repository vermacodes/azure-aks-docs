---
title: Use Microsoft Entra ID authorization for the Kubernetes API in AKS
titleSuffix: Azure Kubernetes Service
description: Learn how to authorize Kubernetes API access in Azure Kubernetes Service (AKS) using Microsoft Entra ID role assignments and ABAC conditions.
ms.topic: how-to
ms.subservice: aks-security
ms.custom: devx-track-azurecli
ms.date: 04/18/2026
ms.author: shasb
author: shashankbarsin
ai-usage: ai-assisted

# Customer intent: "As a cluster operator, I want to authorize access to the Kubernetes API using Microsoft Entra ID identities and granular conditions, so that I can govern cluster access at scale across many clusters from a single identity plane."
---

# Use Microsoft Entra ID authorization for the Kubernetes API in AKS

This article shows how to authorize calls to the Kubernetes API in Azure Kubernetes Service (AKS) using Microsoft Entra ID identities. Entra ID authorization for the Kubernetes API uses Azure RBAC role assignments under the hood and can be optionally extended with Azure ABAC conditions to filter access to specific custom resource types.

For a conceptual overview of the available Kubernetes API authorization options in AKS, see [Cluster authorization concepts](concepts-cluster-authorization.md).

> [!NOTE]
> When you use [integrated authentication between Microsoft Entra ID and AKS](managed-azure-ad.md), you can use Microsoft Entra users, groups, or service principals as subjects in [Kubernetes role-based access control (Kubernetes RBAC)][kubernetes-rbac]. With Entra ID authorization, you don't need to separately manage user identities and credentials for Kubernetes. However, you still need to set up and manage Entra ID role assignments and any Kubernetes RBAC bindings separately.

## Before you begin

* You need the Azure CLI version 2.24.0 or later installed and configured. Run `az --version` to find the version. If you need to install or upgrade, see [Install Azure CLI][install-azure-cli].
* You need `kubectl`, with a minimum version of [1.18.3](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.18.md#v1183).
* You need managed Microsoft Entra integration enabled on your cluster before you can add Entra ID authorization for the Kubernetes API. If you need to enable managed Microsoft Entra integration, see [Use Microsoft Entra ID in AKS](managed-azure-ad.md).
* For custom role definitions that need to cover custom resources (CRDs), use `Microsoft.ContainerService/managedClusters/*/read`. For built-in resources, you can use the specific API groups, such as `Microsoft.ContainerService/apps/deployments/read`. To filter access to specific CRD groups or kinds, use the [ABAC conditions](#restrict-custom-resource-access-using-abac-conditions-preview) section later in this article.
* New role assignments can take *up to five minutes* to propagate and be updated by the authorization server.
* Entra ID authorization for the Kubernetes API requires that the Microsoft Entra tenant configured for authentication is the same as the tenant for the subscription that holds your AKS cluster.

<a name='create-a-new-aks-cluster-with-managed-azure-ad-integration-and-azure-rbac-for-kubernetes-authorization'></a>

## Create a new AKS cluster with managed Microsoft Entra integration and Entra ID authorization

1. Create an Azure resource group using the [`az group create`][az-group-create] command.

    ```azurecli-interactive
    export RESOURCE_GROUP=<resource-group-name>
    export LOCATION=<azure-region>

    az group create --name $RESOURCE_GROUP --location $LOCATION
    ```

1. Create an AKS cluster with managed Microsoft Entra integration and Entra ID authorization using the [`az aks create`][az-aks-create] command.

    ```azurecli-interactive
    export CLUSTER_NAME=<cluster-name>

    az aks create \
        --resource-group $RESOURCE_GROUP \
        --name $CLUSTER_NAME \
        --enable-aad \
        --enable-azure-rbac \
        --generate-ssh-keys
    ```

    Your output should look similar to the following example output:

    ```output
    "AADProfile": {
        "adminGroupObjectIds": null,
        "clientAppId": null,
        "enableAzureRbac": true,
        "managed": true,
        "serverAppId": null,
        "serverAppSecret": null,
        "tenantId": "****-****-****-****-****"
    }
    ```

## Enable Entra ID authorization on an existing AKS cluster

* Enable Entra ID authorization for the Kubernetes API on an existing AKS cluster using the [`az aks update`][az-aks-update] command with the `--enable-azure-rbac` flag.

    ```azurecli-interactive
    az aks update --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --enable-azure-rbac
    ```

    > [!NOTE]
    > To use [ABAC conditions for custom resources](#restrict-custom-resource-access-using-abac-conditions-preview), enable Entra ID authorization at cluster creation rather than on an existing cluster. Some clusters that have Entra ID authorization enabled post-creation might not pick up the ABAC condition evaluation feature in the authorization webhook.

## Disable Entra ID authorization for the Kubernetes API

* Remove Entra ID authorization for the Kubernetes API from an existing AKS cluster using the [`az aks update`][az-aks-update] command with the `--disable-azure-rbac` flag.

    ```azurecli-interactive
    az aks update --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --disable-azure-rbac
    ```

## AKS built-in roles

AKS provides the following built-in roles:

| Role                                | Description  |
|-------------------------------------|--------------|
| Azure Kubernetes Service RBAC Reader  | Allows read-only access to see most objects in a namespace. It doesn't allow viewing roles or role bindings. This role doesn't allow viewing `Secrets`, since reading the contents of Secrets enables access to ServiceAccount credentials in the namespace, which would allow API access as any ServiceAccount in the namespace (a form of privilege escalation).  |
| Azure Kubernetes Service RBAC Writer | Allows read/write access to most objects in a namespace. This role doesn't allow viewing or modifying roles or role bindings. However, this role allows accessing `Secrets` and running Pods as any ServiceAccount in the namespace, so it can be used to gain the API access levels of any ServiceAccount in the namespace. |
| Azure Kubernetes Service RBAC Admin  | Allows admin access, intended to be granted within a namespace. Allows read/write access to most resources in a namespace (or cluster scope), including the ability to create roles and role bindings within the namespace. This role doesn't allow write access to resource quota or to the namespace itself. |
| Azure Kubernetes Service RBAC Cluster Admin  | Allows super-user access to perform any action on any resource. It gives full control over every resource in the cluster and in all namespaces. |

## Create role assignments for cluster access

### [Azure CLI](#tab/azure-cli)

1. Get your AKS resource ID using the [`az aks show`][az-aks-show] command.

    ```azurecli
    AKS_ID=$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --query id --output tsv)
    ```

2. Create a role assignment using the [`az role assignment create`][az-role-assignment-create] command. `<AAD-ENTITY-ID>` can be a username or the client ID of a service principal. The following example creates a role assignment for the *Azure Kubernetes Service RBAC Admin* role.

    ```azurecli-interactive
    az role assignment create --role "Azure Kubernetes Service RBAC Admin" --assignee <AAD-ENTITY-ID> --scope $AKS_ID
    ```

    > [!NOTE]
    > You can create the *Azure Kubernetes Service RBAC Reader* and *Azure Kubernetes Service RBAC Writer* role assignments scoped to a specific namespace within the cluster using the [`az role assignment create`][az-role-assignment-create] command and setting the scope to the desired namespace.
    >
    > ```azurecli-interactive
    > az role assignment create --role "Azure Kubernetes Service RBAC Reader" --assignee <AAD-ENTITY-ID> --scope $AKS_ID/namespaces/<namespace-name>
    > ```

### [Azure portal](#tab/azure-portal)

1. In the [Azure portal](https://portal.azure.com/), navigate to your AKS cluster.
1. In the service menu, select **Access control (IAM)** > **Add role assignment**.
1. On the **Role** tab, select the desired role, such as *Azure Kubernetes Service RBAC Admin*, and then select **Next**.
1. On the **Members** tab, configure the following settings:

    * **Assign access to**: Select **User, group, or service principal**.
    * **Members**: Select **+ Select members**, search for and select the desired members, and then select **Select**.

1. Select **Review + assign** > **Assign**.

    > [!NOTE]
    > In Azure portal, after creating role assignments scoped to a desired namespace, you won't be able to see "role assignments" for namespace [at a scope][list-role-assignments-at-a-scope-at-portal]. You can find it by using the [`az role assignment list`][az-role-assignment-list] command, or [list role assignments for a user or group][list-role-assignments-for-a-user-or-group-at-portal], which you assigned the role to.
    >
    > ```azurecli-interactive
    > az role assignment list --scope $AKS_ID/namespaces/<namespace-name>
    > ```

---

## Create custom roles definitions

The following example custom role definition allows a user to only read deployments and nothing else. For the full list of possible actions, see [Microsoft.ContainerService operations](/azure/role-based-access-control/resource-provider-operations#microsoftcontainerservice).

1. To create your own custom role definitions, copy the following file, replacing `<YOUR SUBSCRIPTION ID>` with your own subscription ID, and then save it as `deploy-view.json`.

    ```json
    {
        "Name": "AKS Deployment Reader",
        "Description": "Lets you view all deployments in cluster/namespace.",
        "Actions": [],
        "NotActions": [],
        "DataActions": [
            "Microsoft.ContainerService/managedClusters/apps/deployments/read"
        ],
        "NotDataActions": [],
        "assignableScopes": [
            "/subscriptions/<YOUR SUBSCRIPTION ID>"
        ]
    }
    ```

2. Create the role definition using the [`az role definition create`][az-role-definition-create] command, setting the `--role-definition` to the `deploy-view.json` file you created in the previous step.

    ```azurecli-interactive
    az role definition create --role-definition @deploy-view.json 
    ```

3. Assign the role definition to a user or other identity using the [`az role assignment create`][az-role-assignment-create] command.

    ```azurecli-interactive
    az role assignment create --role "AKS Deployment Reader" --assignee <AAD-ENTITY-ID> --scope $AKS_ID
    ```

## Restrict custom resource access using ABAC conditions (preview)

[!INCLUDE [preview features callout](~/reusable-content/ce-skilling/azure/includes/aks/includes/preview/preview-callout.md)]

ABAC conditions let you filter Entra ID role assignments to specific custom resource (CRD) groups and kinds. Use ABAC conditions when you want to grant a broad read role across the Kubernetes API but restrict which CRDs the assignee can access, without writing per-cluster Kubernetes RBAC manifests. For background on Azure ABAC, see [What are Azure role assignment conditions?](/azure/role-based-access-control/conditions-overview).

### When to use ABAC conditions

Use this feature when you want to:

* Grant read access to most Kubernetes resources but restrict which CRD groups or kinds an assignee can list or get.
* Centrally enforce custom resource access boundaries from Microsoft Entra ID without managing Kubernetes RBAC `Role` and `RoleBinding` objects on every cluster.
* Distinguish between CRDs published by different operators (for example, allow `templates.gatekeeper.sh` while blocking `kyverno.io`).

### Prerequisites

* The cluster must have Entra ID authorization enabled. Enable it **at cluster creation** using `--enable-azure-rbac`. Some clusters that have Entra ID authorization enabled post-creation might not pick up ABAC condition evaluation.
* Conditions must use condition syntax version `2.0`.
* The role assignment must include a Data Action that targets the Kubernetes API, such as `Microsoft.ContainerService/managedClusters/*/read` or `Microsoft.ContainerService/managedClusters/customresources/read`.

### Available condition attributes

The following request attributes are available when authoring conditions for the Kubernetes API on an AKS cluster:

| Attribute | Description |
|---|---|
| `Microsoft.ContainerService/managedClusters/customResources:group` | The API group of the custom resource being accessed (for example, `templates.gatekeeper.sh`). |
| `Microsoft.ContainerService/managedClusters/customResources:kind` | The kind of the custom resource being accessed (for example, `constrainttemplates`). |

### Add an ABAC condition to a role assignment

The following example creates a role assignment using the *AKS CRD Reader* custom role with `Microsoft.ContainerService/managedClusters/*/read` and attaches a condition that only allows access to `constrainttemplates` in the `templates.gatekeeper.sh` group.

1. Save the following condition to a file named `abac-condition.txt`. The condition allows non-custom-resource reads to pass through unchanged, and restricts custom resource reads to a specific group and kind.

    ```text
    (
     (
      !(ActionMatches{'Microsoft.ContainerService/managedClusters/customresources/read'})
     )
     OR
     (
      @Request[Microsoft.ContainerService/managedClusters/customResources:group] StringEqualsIgnoreCase 'templates.gatekeeper.sh'
      AND
      @Request[Microsoft.ContainerService/managedClusters/customResources:kind] StringEqualsIgnoreCase 'constrainttemplates'
     )
    )
    ```

1. Create the role assignment with the condition using the [`az role assignment create`][az-role-assignment-create] command.

    ```azurecli-interactive
    az role assignment create \
        --role "AKS CRD Reader" \
        --assignee <AAD-ENTITY-ID> \
        --scope $AKS_ID \
        --condition "$(cat abac-condition.txt)" \
        --condition-version "2.0" \
        --description "Allow reads on Gatekeeper constrainttemplates only"
    ```

You can also add a condition through the Azure portal. On the **Add role assignment** page, select the **Conditions** tab, then select **Add condition** and use the visual editor to build the expression.

### Verify the condition

After the role assignment propagates (up to five minutes), confirm the assignee can read the allowed CRD but not other CRDs.

```bash
# Get a Microsoft Entra access token for the AKS audience.
TOKEN=$(az account get-access-token --resource 6dae42f8-4368-4678-94ff-3960e28e3630 --query accessToken --output tsv)
APISERVER=$(kubectl config view --minify --output jsonpath='{.clusters[0].cluster.server}')

# Allowed by the condition: returns 200 (or 404 if the CRD isn't installed).
curl -s -o /dev/null -w "%{http_code}\n" \
    -H "Authorization: Bearer $TOKEN" \
    "$APISERVER/apis/templates.gatekeeper.sh/v1/constrainttemplates"

# Blocked by the condition (different group): returns 403.
curl -s -o /dev/null -w "%{http_code}\n" \
    -H "Authorization: Bearer $TOKEN" \
    "$APISERVER/apis/kyverno.io/v1/clusterpolicies"
```

### Troubleshoot

If the ABAC condition isn't enforced as expected, check the following:

* **Cluster creation flag.** Confirm Entra ID authorization was enabled at cluster creation. Some clusters that had it enabled post-creation might not evaluate ABAC conditions in the authorization webhook.
* **Condition version.** The role assignment must specify `--condition-version "2.0"`.
* **Attribute spelling.** The attribute names use camelCase: `customResources:group`, `customResources:kind`. The action match uses lowercase: `customresources/read`.
* **Propagation delay.** Wait up to five minutes after creating or updating the role assignment.

## Use Entra ID authorization with `kubectl`

1. Make sure you have the [Azure Kubernetes Service Cluster User](/azure/role-based-access-control/built-in-roles#azure-kubernetes-service-cluster-user-role) built-in role, and then get the kubeconfig of your AKS cluster using the [`az aks get-credentials`][az-aks-get-credentials] command.

    ```azurecli-interactive
    az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME
    ```

2. You can now use `kubectl` to manage your cluster. For example, you can list the nodes in your cluster using `kubectl get nodes`.

    ```azurecli-interactive
    kubectl get nodes
    ```

    Example output:

    ```output
    NAME                                STATUS   ROLES   AGE    VERSION
    aks-nodepool1-93451573-vmss000000   Ready    agent   3h6m   v1.15.11
    aks-nodepool1-93451573-vmss000001   Ready    agent   3h6m   v1.15.11
    aks-nodepool1-93451573-vmss000002   Ready    agent   3h6m   v1.15.11
    ```

## Use Entra ID authorization with `kubelogin`

AKS created the [`kubelogin`](https://github.com/Azure/kubelogin) plugin to help unblock scenarios such as non-interactive logins, older `kubectl` versions, or using SSO across multiple clusters without the need to sign in to a new cluster.

1. Use the `kubelogin` plugin by running the following command:

    ```azurecli-interactive
    export KUBECONFIG=/path/to/kubeconfig
    kubelogin convert-kubeconfig
    ```

2. You can now use `kubectl` to manage your cluster. For example, you can list the nodes in your cluster using `kubectl get nodes`.

    ```azurecli-interactive
    kubectl get nodes
    ```

    Example output:

    ```output
    NAME                                STATUS   ROLES   AGE    VERSION
    aks-nodepool1-93451573-vmss000000   Ready    agent   3h6m   v1.15.11
    aks-nodepool1-93451573-vmss000001   Ready    agent   3h6m   v1.15.11
    aks-nodepool1-93451573-vmss000002   Ready    agent   3h6m   v1.15.11
    ```

## Clean up resources

### Delete role assignment

### [Azure CLI](#tab/azure-cli)

1. List role assignments using the [`az role assignment list`][az-role-assignment-list] command.

    ```azurecli-interactive
    az role assignment list --scope $AKS_ID --query [].id --output tsv
    ```

2. Delete role assignments using the [`az role assignment delete`][az-role-assignment-delete] command.

    ```azurecli-interactive
    az role assignment delete --ids <LIST OF ASSIGNMENT IDS>
    ```

### [Azure portal](#tab/azure-portal)

1. Navigate to your AKS cluster and select **Access control (IAM)** > **Role assignments**.
2. Select the role assignment you want to delete, and then select **Delete** > **Yes**.

---

### Delete role definition

* Delete the custom role definition using the [`az role definition delete`][az-role-definition-delete] command.

    ```azurecli-interactive
    az role definition delete --name "AKS Deployment Reader"
    ```

### Delete resource group and AKS cluster

### [Azure CLI](#tab/azure-cli)

* Delete the resource group and AKS cluster using the [`az group delete`][az-group-delete] command.

    ```azurecli-interactive
    az group delete --name $RESOURCE_GROUP --yes --no-wait
    ```

### [Azure portal](#tab/azure-portal)

1. Navigate to the resource group that contains your AKS cluster and select **Delete resource group**.
2. On the **Delete a resource group** page, enter the resource group name, and then select **Delete** > **Delete**.

---

## Next steps

To learn more about AKS authentication, authorization, Kubernetes RBAC, and Azure RBAC, see:

* [Cluster authorization concepts for AKS](concepts-cluster-authorization.md)
* [Access and identity options for AKS](/azure/aks/concepts-identity)
* [What is Azure RBAC?](/azure/role-based-access-control/overview)
* [What are Azure ABAC conditions?](/azure/role-based-access-control/conditions-overview)
* [Microsoft.ContainerService operations](/azure/role-based-access-control/resource-provider-operations#microsoftcontainerservice)

<!-- LINKS - Internal -->
[aks-support-policies]: support-policies.md
[aks-faq]: faq.md
[az-extension-add]: /cli/azure/extension#az-extension-add
[az-extension-update]: /cli/azure/extension#az-extension-update
[az-feature-list]: /cli/azure/feature#az-feature-list
[az-feature-register]: /cli/azure/feature#az-feature-register
[az-aks-install-cli]: /cli/azure/aks#az-aks-install-cli
[az-aks-create]: /cli/azure/aks#az-aks-create
[az-aks-show]: /cli/azure/aks#az-aks-show
[list-role-assignments-at-a-scope-at-portal]: /azure/role-based-access-control/role-assignments-list-portal#list-role-assignments-at-a-scope
[list-role-assignments-for-a-user-or-group-at-portal]: /azure/role-based-access-control/role-assignments-list-portal#list-role-assignments-for-a-user-or-group
[az-role-assignment-create]: /cli/azure/role/assignment#az-role-assignment-create
[az-role-assignment-delete]: /cli/azure/role/assignment#az-role-assignment-delete
[az-role-assignment-list]: /cli/azure/role/assignment#az-role-assignment-list
[az-provider-register]: /cli/azure/provider#az-provider-register
[az-group-create]: /cli/azure/group#az-group-create
[az-group-delete]: /cli/azure/group#az-group-delete
[az-aks-update]: /cli/azure/aks#az-aks-update
[managed-aad]: ./managed-azure-ad.md
[install-azure-cli]: /cli/azure/install-azure-cli
[az-role-definition-create]: /cli/azure/role/definition#az-role-definition-create
[az-role-definition-delete]: /cli/azure/role/definition#az-role-definition-delete
[az-aks-get-credentials]: /cli/azure/aks#az-aks-get-credentials
[kubernetes-rbac]: kubernetes-rbac-entra-id.md
