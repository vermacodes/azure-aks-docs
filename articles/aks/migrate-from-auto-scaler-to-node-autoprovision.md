---
title: Migrate for Cluster Autoscaler to Node auto provisioning
description: Learn about how to migrate your Azure Kubernetes Service (AKS) cluster from cluster autoscaler to node autoprovisioning.
ms.topic: how-to
ms.custom: devx-track-azurecli
ms.date: 09/01/2025
ms.author: wilsondarko
author: wdarko1

#Customer intent: As a cluster operator or developer, I want to automatically provision and manage the optimal VM configuration for my AKS workloads, so that I can efficiently scale my cluster while minimizing resource costs and complexities.

---

# Migrate from Cluster Autoscaler to Node auto provisioning

This document provides instructions to migrate your existing Azure Kubernetes Service (AKS) cluster from using [cluster autoscaler][cluster-autoscaler] to [node auto provisioning][nap-main-doc]. 

Node auto provisioning (NAP) uses pending pod resource requirements to decide the optimal virtual machine configuration to run those workloads in the most efficient and cost-effective manner.

Node auto provisioning is based on the open source [Karpenter](https://karpenter.sh) project, and the [AKS Karpenter provider][aks-karpenter-provider], which is also open source. Node auto provisioning automatically deploys, configures, and manages Karpenter on your AKS clusters.

### Prerequisites

| Prerequisite                         | Notes                                                                                                        |
|--------------------------------------|--------------------------------------------------------------------------------------------------------------|
| **Azure Subscription**               | If you don't have an Azure subscription, you can create a [free account](https://azure.microsoft.com/free).  |
| **Azure CLI**| `2.76.0` or later. To find the version, run `az --version`. For more information about installing or upgrading the Azure CLI, see [Install Azure CLI][azure cli].|                      

## Limitations

- You can't enable node autoprovisioning in a cluster where node pools have cluster autoscaler enabled
- NAP currently does not support
   - Windows node pools
   - IPv6 clusters
   - Service Principals
   - HTTP Proxy

## Pre-migration checklist

- Confirm cluster eligibility for node auto provisioning
- Right-size workloads for consolidation
  - Set proper resource requests/limits, replicas, and pod disruption budgets (PDBs) to allow for a gradual migration. This migration method will require properly set PDBs to ensure well-managed disruption of your workloads. 
- Verify your system node pool
  - AKS requires a system node pool for system components (such as CoreDNS, Karpenter, etc.)

### Disable cluster autoscaler safely

If cluster autoscaler is enabled cluster-wide, disable it at the cluster level using the `--disable-cluster-autoscaler` flag. Nodes aren’t removed when you disable cluster autoscaler, so your capacity stays steady.

```
az aks update --resource-group myResourceGroup --name myAKSCluster --disable-cluster-autoscaler
```

If cluster autoscaler is only enabled on select node pools, you also have the option to disable cluster autoscaler for specific node pools using the `-disable-cluster-autoscaler` flag.

```
# Disable CAS on a specific pool
az aks nodepool update \
  --resource-group myResourceGroup \
  --cluster-name myAKSCluster \
  --name mypool1 \
  --disable-cluster-autoscaler
```

You can also set the node count of your node pool to a pinned count, as you begin the migration to node auto provisioning. The following az aks nodepool scale command pins the node count of node pool `mypool1` in cluster `myAKSCluster` to five (5). 

```
# (Optional) Pin to a safe desired count before the switch
az aks nodepool scale \
  --resource-group myResourceGroup \
  --cluster-name myAKSCluster \
  --name mypool1 \
  --node-count 5
```

## Enable node autoprovisioning

### Enable node autoprovisioning on an existing cluster

- Enable node autoprovisioning on an existing cluster using the `az aks update` command and set `--node-provisioning-mode` to `Auto`.

    ```azurecli-interactive
    az aks update --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP_NAME --node-provisioning-mode Auto
    ```

### Define your first NodePool and AKSNodeClass

After enabling node autoprovisioning on your cluster, you can create a basic NodePool and AKSNodeClass to start provisioning nodes. Here's a simple example:

```yaml
#nodepool-default.yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    metadata:
      labels:
        intent: apps
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot, on-demand]
        - key: karpenter.azure.com/sku-family
          operator: In
          values: [D]
      expireAfter: Never
  limits:
    cpu: 100
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 0s
---
apiVersion: karpenter.azure.com/v1beta1
kind: AKSNodeClass
metadata:
  name: default
  annotations:
    kubernetes.io/description: "General purpose AKSNodeClass for running Ubuntu2204 nodes"
spec:
  imageFamily: Ubuntu2204
```

This example creates a basic NodePool that:
- Supports both spot and on-demand instances
- Uses D-series VMs
- Sets a CPU limit of 100
- Enables consolidation when nodes are empty or underutilized

You can now deploy the custom resources to your cluster with the following `kubectl` command:

```
kubectl apply -f nodepool-default.yaml
```


## Migrate workloads from fixed pools to node auto provisioning managed nodes

Now scale down user pools gradually (keep the system pool):

```
# For each user pool, step down in small increments (respect PDBs)
az aks nodepool scale \
  --resource-group <RG> \
  --cluster-name <CLUSTER> \
  --name <USER_POOL> \
  --node-count <LOWER_DESIRED>
```

As pods evict, node auto provisioning provisions replacement nodes per your NodePool and AKSNodeClass rules. If a user pool must go to zero, remember you can only do that on user pools (not system pool), and with cluster autoscaler disabled - which has been disabled in an earlier step.

>[!NOTE]
> We recommend a gradual scale down in waves, and watch replicas/PDBs to avoid dips in availability.

### Clean up old autoscaling 

- If you are using managed AKS cluster autoscaler only, you have already disabled cluster autoscaler with the above steps. 
- If you are using self-hosted cluster autoscaler installed in kube-system, scale the cluster autoscaler pods to zero and remove.

```
kubectl -n kube-system scale deploy/cluster-autoscaler --replicas=0
kubectl -n kube-system delete deploy/cluster-autoscaler
```

## Fine tune node auto provisioning post-migration

After you have completed your migration there are more capabilities to fine tune your cluster.

- **Manage disruption behavior** - Tune disruption `consolidationPolicy` and `consolidateAfter` windows to balance cost vs. virtual machine churn. To learn more, visit our [NAP Disruption documentation][nap-disruption-doc]
- **Multiple NodePools** - Split by workload class (e.g., Spot vs On-Demand, GPU vs CPU) and use requirements, weights, and taints to control placement. To learn more, visit our [NAP NodePool documentation][nap-nodepool-doc]
- **Networking** - For more information of managing networking experiences like custom virtual networks, visit our [NAP netowrking documentation][nap-networking-doc]
- **Observability** Stream Karpenter events and expose NAP control-plane metrics via Azure Monitor managed Prometheus. For more visit our [NAP public documentation][nap-observability]


## Disabling node autoprovisioning

Node auto provisioning can only be disabled when:

- There are no existing node autoprovisioning-managed nodes. Use `kubectl get nodes -l karpenter.sh/nodepool` to view node autoprovisioning-managed nodes.
- All existing karpenter.sh/NodePools have their `spec.limits.cpu` field set to 0.

### Steps to disable node autoprovisioning

1. Set all karpenter.sh/NodePools `spec.limits.cpu` field to 0. This action prevents new nodes from being created, but doesn't disrupt currently running nodes.

> [!NOTE]
> If you don't care about ensuring that every pod that was running on a node autoprovisioning node is migrated safely to a non-node autoprovisioning node,
> you can skip steps 2 and 3 and instead use the `kubectl delete node` command for each node autoprovisioning-managed node.
>
> **Skipping steps 2 and 3 is not recommended, as it might leave some pods pending and doesn't honor Pod Disruption Budgets (PDBs).**
>
> **Don't run `kubectl delete node` on any nodes that aren't managed by node autoprovisioning.**

2. Add the `karpenter.azure.com/disable:NoSchedule` taint to every karpenter.sh/NodePool.
   ```yaml
   apiVersion: karpenter.sh/v1
   kind: NodePool
   metadata:
     name: default
   spec:
     template:
       spec:
         ...
         taints:
           - key: karpenter.azure.com/disable,
             effect: NoSchedule
   ```
   
   This action starts the process of migrating the workloads on the node autoprovisioning-managed nodes to non-NAP nodes, honoring Pod Disruption Budgets (PDBs) and disruption limits. Pods migrate to non-NAP nodes if they can fit. If there isn't enough fixed-size capacity, some node autoprovisioning-managed nodes remain.

4. Scale up existing fixed-size ManagedCluster noode pools, or create new fixed-size node pools, to take the load from the node auto provisioning-managed nodes.
   As these nodes are added to the cluster the node auto provisioning-managed nodes are drained, and work is migrated to the fixed-scale nodes.

5. Confirm that all node auto provisioning-managed nodes are deleted, using `kubectl get nodes -l karpenter.sh/nodepool`. If node autoprovisioning-managed nodes still exist, the cluster likely lacks fixed-scale capacity. Add more nodes so the remaining workloads can be migrated.
6. Update the node provisioning mode parameter of the ManagedCluster to `Manual`.

    #### [Azure CLI](#tab/azure-cli)

      ```azurecli-interactive
      az aks update \
          --name $CLUSTER_NAME \
          --resource-group $RESOURCE_GROUP_NAME \
          --node-provisioning-mode Manual
      ```

    #### [ARM template](#tab/arm)

    ```azurecli-interactive
    az deployment group create --resource-group $RESOURCE_GROUP_NAME --template-file ./nap.json
    ```

    The `nap.json` file should contain the following ARM template. The value of the `properties.nodeProvisioningProfile.mode` field set to `Manual`
    is what's performing the disablement:

    ```JSON
    {
      "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
      "contentVersion": "1.0.0.0",
      "metadata": {},
      "parameters": {},
      "resources": [
        {
          "type": "Microsoft.ContainerService/managedClusters",
          "apiVersion": "2025-05-01",
          "sku": {
            "name": "Base",
            "tier": "Standard"
          },
          "name": "napcluster",
          "location": "uksouth",
          "identity": {
            "type": "SystemAssigned"
          },
          "properties": {
            "networkProfile": {
                "networkPlugin": "azure",
                "networkPluginMode": "overlay",
                "networkPolicy": "cilium",
                "networkDataplane":"cilium",
                "loadBalancerSku": "Standard"
            },
            "dnsPrefix": "napcluster",
            "agentPoolProfiles": [
              {
                "name": "agentpool",
                "count": 3,
                "vmSize": "standard_d2s_v3",
                "osType": "Linux",
                "mode": "System"
              }
            ],
            "nodeProvisioningProfile": {
              "mode": "Manual"
            }
          }
        }
      ]
    }
    ```





---
<!-- LINKS - internal -->
[aks-view-master-logs]: monitor-aks.md#aks-control-planeresource-logs
[azure-cli-extensions]: /cli/azure/azure-cli-extensions-overview
[azure cli]: /cli/azure/get-started-with-azure-cli
[az-extension-add]: /cli/azure/extension#az-extension-add
[az-extension-update]: /cli/azure/extension#az-extension-update
[planned-maintenance#schedule-configuration-types-for-planned-maintenance]: /azure/aks/planned-maintenance#schedule-configuration-types-for-planned-maintenance
[az-aks-create]: /cli/azure/aks#az-aks-create
[az-aks-get-credentials]: /cli/azure/aks#az-aks-get-credentials
[az-aks-install-cli]: /cli/azure/aks#az-aks-install-cli
[auto-upgrade]: /azure/aks/auto-upgrade-cluster#cluster-auto-upgrade-channels
[auto-mode]: /azure/templates/microsoft.containerservice/managedclusters?pivots=deployment-language-bicep#managedclusternodeprovisioningprofile
[node-os-upgrade-channel]: /azure/aks/auto-upgrade-node-os-image#available-node-os-upgrade-channels
[azure-support]: /azure/azure-portal/supportability/how-to-create-azure-support-request
[vm-overview]: /azure/virtual-machines/sizes/overview
[nap-main-doc]: /azure/aks/node-autoprovision
[nap-disruption-doc]: /azure/aks/node-autoprovision-disruption
[nap-nodepool-doc]: /azure/aks/node-autoprovision-node-pools
[nap-networking-doc]: /azure/aks/node-autoprovision-networking
[nap-observability]: /azure/aks/node-autoprovision#node-auto-provisioning-metrics
[cluster-autoscaler]: /azure/aks/cluster-autoscaler

<!-- LINKS - external -->
[aks-karpenter-provider]: https://github.com/Azure/karpenter-provider-azure
[aks-karpenter-provider-issues]: https://github.com/Azure/karpenter-provider-azure/issues
[kubectl]: https://kubernetes.io/docs/reference/kubectl/
[kubectl-apply]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply
[kubectl-get]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#get
[AKS-repo]: https://github.com/Azure/AKS/issues
