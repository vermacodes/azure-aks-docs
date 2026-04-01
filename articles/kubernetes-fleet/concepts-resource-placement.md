---
title: "Introducing Azure Kubernetes Fleet Manager intelligent resource placement"
description: This article describes the concepts of Azure Kubernetes Fleet Manager intelligent resource placement
ms.date: 04/01/2026
author: sjwaight
ms.author: simonwaight
ms.service: azure-kubernetes-fleet-manager
ms.topic: concept-article
# Customer intent: As a platform admin, I want to propagate Kubernetes resources from a hub cluster to multiple member clusters, so that I can manage workloads and access control across diverse environments.
zone_pivot_groups: cluster-namespace-scope
---

# Introducing Azure Kubernetes Fleet Manager intelligent resource placement

**Applies to** :heavy_check_mark: Fleet Manager with hub cluster

Managing Kubernetes resources across multiple clusters presents significant challenges for both platform administrators and application developers. As organizations scale their Kubernetes infrastructure beyond a single cluster, they often encounter complexities related to resource distribution, consistency maintenance, and manual management overhead. The traditional approach of managing each cluster independently creates operational silos that become increasingly difficult to maintain as the fleet size grows.

Platform administrators often need to deploy Kubernetes resources onto multiple clusters for various reasons, including:

* Managing access control using roles and role bindings across multiple clusters.
* Running infrastructure applications, such as Prometheus or Flux, that need to be on all clusters.

Application developers often need to deploy Kubernetes resources onto multiple clusters for various reasons, for example:

* Deploying a video serving application into multiple clusters in different regions for a low latency watching experience.
* Deploying a shopping cart application into two paired regions for customers to continue to shop during a single region outage.
* Deploying a batch compute application into clusters with inexpensive spot node pools available.

It's tedious and potentially error-prone to create, update, and track these Kubernetes resources across multiple clusters manually. 

In this article we explore how multi-cluster users can use Fleet Manager's intelligent resource placement capability to place cluster and namespace-scoped Kubernetes resources that are staged on the [Fleet Manager hub cluster](./access-fleet-hub-cluster-kubernetes-api.md) across member clusters in a fleet. 

Fleet Manager's resource placement capability is based on the [KubeFleet CNCF project](https://kubefleet.dev/).

:::zone target="docs" pivot="cluster-scope"

## Overview of cluster-scoped resource placement

Use a ClusterResourcePlacement (CRP) to distribute a given set of cluster-scoped resource or entire namespaces from the Fleet Manager hub cluster onto one or more member cluster.

With CRP, you can:

* Select which Kubernetes resources to distribute. These can be cluster-scoped Kubernetes resources defined using [Kubernetes Group Version Kind (GVK)](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.33/#api-groups) references, or a namespace, which distributes the namespace and all its resources.
* Specify placement policies to select member clusters. These policies can explicitly select clusters by names, or dynamically select clusters based on cluster labels and properties. 
* Specify rollout strategies to safely roll out any updates of the selected Kubernetes resources to multiple target clusters.
* View the rollout progress for each target cluster.

For scenarios requiring fine-grained control over individual namespace-scoped resources within a namespace, see [namespace-scoped resource placement](./concepts-namespace-scoped-resource-propagation.md?pivots=namespace-scope#overview-of-namespace-scoped-resource-placement), which enables distribution of specific resources rather than entire namespaces.

:::zone-end

:::zone target="docs" pivot="namespace-scope"

## Overview of namespace-scoped resource placement

Use a ResourcePlacement (RP) to distribute a given set of resources within a specific namespace from the Fleet Manager hub cluster onto one or more member cluster. ResourcePlacement provides fine-grained control over how specific resources within a namespace are distributed across member clusters.

> [!IMPORTANT]
> `ResourcePlacement` uses the `placement.kubernetes-fleet.io/v1beta1` API version and is currently in preview. Some features demonstrated in this article, such as `selectionScope` in `ClusterResourcePlacement`, are also part of the v1beta1 API and aren't available in the v1 API.

**Key characteristics:**

- **Namespace-scoped**: Both `ResourcePlacement` and the resources it selects exist within the same namespace.
- **Selective**: selects specific resources within a namespace by type, name, or labels rather than entire namespaces.
- **Declarative**: Uses the same placement policies as `ClusterResourcePlacement` for consistent behavior.

A `ResourcePlacement` consists of three core components:

- **Resource Selectors**: Define which namespace-scoped resources to include.
- **Placement Policy**: Determine target clusters using `PickAll`, `PickFixed`, or `PickN` strategies.
- **Rollout Strategy**: Control how changes propagate across selected clusters.

## When to use ResourcePlacement

`ResourcePlacement` is ideal for scenarios requiring granular control over namespace-scoped resources:

- **Selective resource distribution**: Deploy specific ConfigMaps, Secrets, or Services without affecting the entire namespace.
- **Multi-tenant environments**: Allow different teams to manage their resources independently within shared namespaces.
- **Configuration management**: Distribute environment-specific configurations across different cluster environments.
- **Compliance and governance**: Apply different policies to different resource types within the same namespace.
- **Progressive rollouts**: Safely deploy resource updates across clusters with zero-downtime strategies.

In multi-cluster environments, workloads often consist of both cluster-scoped and namespace-scoped resources that need to be distributed across different clusters. While `ClusterResourcePlacement` (CRP) handles cluster-scoped resources effectively, entire namespaces and their contents, there are scenarios where you need more granular control over namespace-scoped resources within existing namespaces.

`ResourcePlacement` (RP) was designed to address this gap by providing:

- **Namespace-scoped resource management**: Target specific resources within a namespace without affecting the entire namespace.
- **Operational flexibility**: Allow teams to manage different resources within the same namespace independently.
- **Complementary functionality**: Work alongside CRP to provide a complete multi-cluster resource management solution.

> [!NOTE]
> `ResourcePlacement` can be used together with `ClusterResourcePlacement` in namespace-only mode. For example, you can use CRP to deploy the namespace, while using RP for fine-grained management of specific resources like environment-specific ConfigMaps or Secrets within that namespace.

:::zone-end

## Placement policy key components

A placement policy, regardless of scope (cluster or namespace) consists of the following components:

- **Resource Selectors**: which resources to include via `resourceSelectors`.
- **Placement Policy**: define how to pick clusters via `placeType` using one of `PickAll`, `PickFixed`, or `PickN` strategies.
- **Rollout Strategy**: control how resources rollout across selected clusters via a `strategy`.

## Resource selection

Select resources using one or more `resourceSelectors` in a placement. Each resource selector can specify:

* **Group, Version, Kind (GVK)**: The type of Kubernetes resource to select.
* **Name**: The name of a specific resource.
* **Label selectors**: Labels to match multiple resources.

:::zone target="docs" pivot="cluster-scope"

### Namespace selection scope (preview)

When using CRP to select an entire namespace, you can use the `selectionScope` field to control whether to include all the child resources in the namespace, or just place an empty namespace.

* **Default behavior** (when `selectionScope` is not specified): distributes the namespace and all resources within it.
* **`NamespaceOnly`**: distributes only the namespace resource, without any resources within the namespace. This is useful when you want to establish namespaces across clusters while managing individual resources separately using [`ResourcePlacement`](./concepts-namespace-scoped-resource-propagation.md).

> [!IMPORTANT]
> The `selectionScope` field is available in the `placement.kubernetes-fleet.io/v1beta1` API version as a preview feature. It is not available in the `placement.kubernetes-fleet.io/v1` API.

The following example shows how to propagate only the namespace without its contents using the v1beta1 API:

```yaml
apiVersion: placement.kubernetes-fleet.io/v1beta1
kind: ClusterResourcePlacement
metadata:
  name: namespace-only-crp
spec:
  resourceSelectors:
    - group: ""
      kind: Namespace
      name: my-app
      version: v1
      selectionScope: NamespaceOnly
  policy:
    placementType: PickAll
```

This approach enables a workflow where platform administrators use `ClusterResourcePlacement` to establish namespaces, while application teams use [`ResourcePlacement`](./concepts-namespace-scoped-resource-propagation.md) for fine-grained control over specific resources within those namespaces.

:::zone-end

## Placement policy types

The following placement policy types are available for controlling the how clusters are selected by Fleet Manager resource placement:

* **[PickFixed](#pickfixed-placement-type)** places the resource onto a specific list of member clusters by name.
* **[PickAll](#pickall-placement-type)** places the resource onto all member clusters, or all member clusters that meet a criteria. This policy is useful for placing infrastructure workloads, like cluster monitoring or reporting applications.
* **[PickN](#pickn-placement-type)** is the most flexible placement option and allows for selection of clusters based on affinity or topology spread constraints and is useful when spreading workloads across multiple appropriate clusters to ensure availability is desired.

### PickFixed placement type

If you want to deploy a workload to a known set of member clusters, you can use a `PickFixed` placement policy to select the clusters by name.

`clusterNames` is the only valid policy option for this placement type.

The following example shows how to deploy the `test-deployment` namespace onto member clusters `cluster1` and `cluster2`.

```yaml
apiVersion: placement.kubernetes-fleet.io/v1
kind: ClusterResourcePlacement
metadata:
  name: crp-fixed
spec:
  policy:
    placementType: PickFixed
    clusterNames:
    - cluster1
    - cluster2
  resourceSelectors:
    - group: ""
      kind: Namespace
      name: test-deployment
      version: v1
```

### PickAll placement type

You can use a `PickAll` placement type to deploy a workload across all member clusters in the fleet or to a subset of clusters that match criteria you set.

When creating this type of placement the following cluster affinity types can be specified:

- **requiredDuringSchedulingIgnoredDuringExecution**: as this policy is required during scheduling, it **filters** the clusters based on the specified criteria.

The following example shows how to deploy a `prod-deployment` namespace and all of its objects across all clusters labeled with `environment: production`:

```yaml
apiVersion: placement.kubernetes-fleet.io/v1
kind: ClusterResourcePlacement
metadata:
  name: crp-pickall
spec:
  policy:
    placementType: PickAll
    affinity:
        clusterAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
                clusterSelectorTerms:
                - labelSelector:
                    matchLabels:
                        environment: production
  resourceSelectors:
    - group: ""
      kind: Namespace
      name: prod-deployment
      version: v1
```

### PickN placement type

The `PickN` placement type is the most flexible option and allows for placement of resources into a configurable number of clusters based on both affinities and topology spread constraints.

When creating this type of placement the following cluster affinity types can be specified:

* **requiredDuringSchedulingIgnoredDuringExecution**: as this policy is required during scheduling, it **filters** the clusters based on the specified criteria.
* **preferredDuringSchedulingIgnoredDuringExecution**: as this policy is preferred, but not required during scheduling, it **ranks** clusters based on specified criteria.

You can set both required and preferred affinities. Required affinities prevent placement to clusters that don't match, and preferred affinities provide ordering of valid clusters.

#### `PickN` with affinities

Using affinities with a `PickN` placement policy functions similarly to using affinities with pod scheduling. 

The following example shows how to deploy a workload onto three clusters. Only clusters with the `critical-allowed: "true"` label are valid placement targets, and preference is given to clusters with the label `critical-level: 1`:

```yaml
apiVersion: placement.kubernetes-fleet.io/v1
kind: ClusterResourcePlacement
metadata:
  name: crp-pickn-01
spec:
  resourceSelectors:
    - ...
  policy:
    placementType: PickN
    numberOfClusters: 3
    affinity:
        clusterAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
              weight: 20
              preference:
              - labelSelector:
                  matchLabels:
                    critical-level: 1
            requiredDuringSchedulingIgnoredDuringExecution:
                clusterSelectorTerms:
                - labelSelector:
                    matchLabels:
                      critical-allowed: "true"
```

#### `PickN` with topology spread constraints

You can use topology spread constraints to force the division of the cluster placements across topology boundaries to satisfy availability requirements. For example, use these constraints to split placements across regions or update rings. You can also configure topology spread constraints to prevent scheduling if the constraint can't be met (`whenUnsatisfiable: DoNotSchedule`) or schedule as best possible (`whenUnsatisfiable: ScheduleAnyway`).

The following example shows how to spread a given set of resources out across multiple regions and attempts to schedule across member clusters with different update days.

```yaml
apiVersion: placement.kubernetes-fleet.io/v1
kind: ClusterResourcePlacement
metadata:
  name: crp-pickn-02
spec:
  resourceSelectors:
    - ...
  policy:
    placementType: PickN
    topologySpreadConstraints:
    - maxSkew: 2
      topologyKey: region
      whenUnsatisfiable: DoNotSchedule
    - maxSkew: 2
      topologyKey: updateDay
      whenUnsatisfiable: ScheduleAnyway
```

For more information, see the [KubeFleet documentation on topology spread constraints][crp-topo].

## Placement policy options

The table summarizes the available scheduling policy fields for each placement type.

|       Policy Field          | PickFixed | PickAll | PickN |
|-----------------------------|-----------|---------|-------|
| `placementType`             | ✅ | ✅ | ✅ |
| `affinity`                  | ❌ | ✅ | ✅ |
| `clusterNames`              | ✅ | ❌ | ❌ |
| `numberOfClusters`          | ❌ | ❌ | ✅ |
| `topologySpreadConstraints` | ❌ | ❌ | ✅ | 

### Selecting clusters based on labels and properties 

#### Available labels and properties to select clusters  

When using the `PickN` and `PickAll` placement types, you can use the following labels and properties as part of your policies.

##### Labels

The following labels are automatically added to all member clusters and can be used for target cluster selection in resource placement policies. 

| Label | Description |
|----------|-------------|
| fleet.azure.com/location | Azure Region of the cluster (westus) |
| fleet.azure.com/resource-group | Azure Resource Group of the cluster (rg_prodapps_01) |
| fleet.azure.com/subscription-id | Azure Subscription Identifier the cluster resides in. Formatted as UUID/GUID. |
| fleet.azure.com/cluster-name | The name of the cluster |
| fleet.azure.com/member-name | The name of the Fleet member corresponding to the cluster |

You can also use any custom labels you apply to your clusters.

##### Properties

The following properties are available for use as part of placement policies. 

CPU and memory properties are represented as [Kubernetes resource units](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-units-in-kubernetes).

Cost properties are decimals, which represent a per-hour cost in US Dollars for the Azure compute utilized for nodes within the cluster. Cost is based on Azure public pricing.

| Property Name | Description |
|----------|-------------|
| kubernetes-fleet.io/node-count | Available nodes on the member cluster. |
| resources.kubernetes-fleet.io/total-cpu | Total CPU resource units of cluster. | 
| resources.kubernetes-fleet.io/allocatable-cpu | Allocatable CPU resource units of cluster. |
| resources.kubernetes-fleet.io/available-cpu | Available CPU resource units of cluster. |
| resources.kubernetes-fleet.io/total-memory | Total memory resource unit of cluster. |
| resources.kubernetes-fleet.io/allocatable-memory | Allocatable memory resource units of cluster. |
| resources.kubernetes-fleet.io/available-memory | Available memory resource units of cluster. |
| kubernetes.azure.com/per-cpu-core-cost | The per-CPU core cost of the cluster.  |
| kubernetes.azure.com/per-gb-memory-cost | The per-GiB memory cost of the cluster. | 
| kubernetes.azure.com/vm-sizes/{vm-sku-name}/capacity | The available number of nodes of type [vm-sku-name][vm-sku-name] in the cluster*.<br/>Example VM SKU name: NV16as_v4.<br/>* In preview via v1beta1 API. |

#### Specifying selection matching criteria

When using cluster properties in a policy criteria, you specify:

* **Name**: Name of the property, which is one the properties [listed in properties](#properties) in this article. 

* **Operator**: An operator used to express the condition between the constraint/desired value and the observed value on the cluster. The following operators are currently supported:

    * `Gt` (Greater than): a cluster's observed value of the given property must be greater than the value in the condition before it can be picked for resource placement.
    * `Ge` (Greater than or equal to): a cluster's observed value of the given property must be greater than or equal to the value in the condition before it can be picked for resource placement.
    * `Lt` (Less than): a cluster's observed value of the given property must be less than the value in the condition before it can be picked for resource placement.
    * `Le` (Less than or equal to): a cluster's observed value of the given property must be less than or equal to the value in the condition before it can be picked for resource placement.
    * `Eq` (Equal to): a cluster's observed value of the given property must be equal to the value in the condition before it can be picked for resource placement.
    * `Ne` (Not equal to): a cluster's observed value of the given property must be not equal to the value in the condition before it can be picked for resource placement.

    If you use the operator `Gt`, `Ge`, `Lt`, `Le`, `Eq`, or `Ne`, the list of values in the condition should have exactly one value.

* **Values:** A list of values, which are possible values of the property.

Fleet evaluates each cluster based on the properties specified in the condition. Failure to satisfy conditions listed under `requiredDuringSchedulingIgnoredDuringExecution` excludes this member cluster from resource placement.

> [!NOTE]
> If a member cluster doesn't possess the property expressed in the condition, it will automatically fail the condition.

Here's an example placement policy to select only clusters with five or more nodes.

```yaml
apiVersion: placement.kubernetes-fleet.io/v1
kind: ClusterResourcePlacement
metadata:
  name: crp
spec:
  resourceSelectors:
    - ...
  policy:
    placementType: PickAll
    affinity:
        clusterAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
                clusterSelectorTerms:
                - propertySelector:
                    matchExpressions:
                    - name: "kubernetes-fleet.io/node-count"
                      operator: Ge
                      values:
                      - "5"
```

#### How property ranking works

When `preferredDuringSchedulingIgnoredDuringExecution` is used, a property sorter ranks all the clusters in the fleet based on their values in an ascending or descending order. The weights used for ordering are calculated based on the value specified.

A property sorter consists of:

* **Name**: Name of the cluster property.
* **Sort order**: Sort order can be either `Ascending` or `Descending`. When `Ascending` order is used, member clusters with lower observed values are preferred. When `Descending` order is used, member clusters with higher observed value are preferred.

##### Descending order

For sort order Descending, the proportional weight is calculated using the formula:

```
((Observed Value - Minimum observed value) / (Maximum observed value - Minimum observed value)) * Weight
```

For example, let's say you want to rank clusters based on the property of available CPU capacity in descending order and that you have a fleet of three clusters with the following available CPU:

| Cluster | Available CPU capacity |
| -------- | ------- |
| `cluster-a` | 100 |
| `cluster-b` | 20 |
| `cluster-c` | 10 |

In this case, the sorter computes the following weights:

| Cluster | Available CPU capacity | Calculation | Weight |
| -------- | ------- | ------- | ------- | 
| `cluster-a` | 100 | (100 - 10) / (100 - 10) | 100% |
| `cluster-b` | 20 | (20 - 10) / (100 - 10) | 11.11% |
| `cluster-c` | 10 | (10 - 10) / (100 - 10) | 0% |

##### Ascending order

For sort order Ascending, the proportional weight is calculated using the formula:

```
(1 - ((Observed Value - Minimum observed value) / (Maximum observed value - Minimum observed value))) * Weight
```

For example, let's say you want to rank clusters based on their per-CPU-core-cost in ascending order and that you have a fleet of three clusters with the following CPU core costs:

| Cluster | Per-CPU core cost |
| -------- | ------- |
| `cluster-a` | 1 |
| `cluster-b` | 0.2 |
| `cluster-c` | 0.1 |

In this case, the sorter computes the following weights:

| Cluster | Per-CPU core cost | Calculation | Weight |
| -------- | ------- | ------- | ------- | 
| `cluster-a` | 1 | 1 - ((1 - 0.1) / (1 - 0.1)) | 0% |
| `cluster-b` | 0.2 | 1 - ((0.2 - 0.1) / (1 - 0.1)) | 88.89% |
| `cluster-c` | 0.1 | 1 - (0.1 - 0.1) / (1 - 0.1) | 100% |

## Resource snapshots

Fleet Manager keeps a history of the 10 most recently used placement scheduling policies, along with resource versions the placement has selected.

These snapshots can be used with [staged rollout strategies][fleet-staged-rollout] to control the version deployed.

For more information, see the [documentation on snapshots][fleet-snapshots].

## Encapsulating resources using envelope objects

When propagating resources to member clusters following the [hub and spoke model](./concepts-multi-cluster-workload-management.md), it's important to understand that the hub cluster itself is also a Kubernetes cluster. Any resource you want to propagate would first be applied directly to the hub cluster, which can lead to some potential side effects:

1. **Unintended Side Effects**: Resources like ValidatingWebhookConfigurations, MutatingWebhookConfigurations, or Admission Controllers would become active on the hub cluster, potentially intercepting and affecting hub cluster operations.

2. **Security Risks**: RBAC resources (Roles, ClusterRoles, RoleBindings, ClusterRoleBindings) intended for member clusters could grant unintended permissions on the hub cluster.

3. **Resource Limitations**: ResourceQuotas, FlowSchema, or LimitRanges defined for member clusters would take effect on the hub cluster. While those side effects generally do no harm, there might be cases where you want to avoid these constraints on the hub cluster.

To avoid those unnecessary side effects, Azure Kubernetes fleet manager provides custom resources to wrap objects to solve these problems. The envelope object itself is applied to the hub, but the resources it contains are only extracted and applied when they reach the member clusters. In this way, one can define resources that should be propagated without actually deploying their contents on the hub cluster. For more information, see the documentation on [envelope objects][envelope-object].

## Using Tolerations

`ClusterResourcePlacement` objects support the specification of tolerations, which apply to the `ClusterResourcePlacement` object. Each toleration object consists of the following fields:

* `key`: The key of the toleration.
* `value`: The value of the toleration.
* `effect`: The effect of the toleration, such as `NoSchedule`.
* `operator`: The operator of the toleration, such as `Exists` or `Equal`.

Each toleration is used to tolerate one or more specific taints applied on the `ClusterResourcePlacement`. Once all taints on a [`MemberCluster`](./concepts-fleet.md#what-are-member-clusters) are tolerated, the scheduler can then propagate resources to the cluster. You can't update or remove tolerations from a `ClusterResourcePlacement` object once created.

For more information, see the [documentation on tolerations][fleet-tolerations].

## Configuring rollout strategy

Fleet uses a rolling update strategy to control how updates are rolled out across clusters.

In the following example, the fleet scheduler rolls out updates to each cluster sequentially, waiting at least `unavailablePeriodSeconds` between clusters. Rollout status is considered successful if all resources were correctly applied to the cluster. Rollout status checking doesn't cascade to child resources, so for example, it doesn't confirm that pods created by a deployment become ready.

```yaml
apiVersion: placement.kubernetes-fleet.io/v1
kind: ClusterResourcePlacement
metadata:
  name: crp
spec:
  resourceSelectors:
    - ...
  policy:
    ...
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
      unavailablePeriodSeconds: 60
```

For more information, see the [documentation on rollout strategies][fleet-rollout].

## Determine placement status

The Fleet scheduler provides two ways to view placement status depending on your access level and requirements:

* **ClusterResourcePlacement status**: View placement status directly on the cluster-scoped `ClusterResourcePlacement` object. Use this approach when you have cluster-level permissions and need to view status for any placement across the fleet. This approach is the primary method for platform administrators.

* **ClusterResourcePlacementStatus (preview)**: View placement status through a namespace-scoped `ClusterResourcePlacementStatus` object. Use this approach when you want to enable namespace users to view placement status without granting cluster-level permissions. This approach requires using the v1beta1 API and setting `statusReportingScope: NamespaceAccessible` on the `ClusterResourcePlacement`. For more information, see the [ClusterResourcePlacementStatus section](#clusterresourceplacementstatus-preview).

Both approaches provide the following information:

* The conditions that currently apply to the placement, which include if the placement was successfully completed.
* A placement status section for each member cluster, which shows the status of deployment to that cluster.

### Viewing ClusterResourcePlacement status

The following example shows viewing status directly from a `ClusterResourcePlacement` that deployed the `test` namespace and the `test-1` ConfigMap into two member clusters using `PickN`. The placement was successfully completed and the resources were placed into the `aks-member-1` and `aks-member-2` clusters.

You can view this information using the `kubectl describe crp <name>` command.

```bash
kubectl describe crp crp-1
```

```output
Name:         crp-1
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  placement.kubernetes-fleet.io/v1
Kind:         ClusterResourcePlacement
Metadata:
  ...
Spec:
  Policy:
    Number Of Clusters:  2
    Placement Type:      PickN
  Resource Selectors:
    Group:
    Kind:                  Namespace
    Name:                  test
    Version:               v1
  Revision History Limit:  10
Status:
  Conditions:
    Last Transition Time:  2023-11-10T08:14:52Z
    Message:               found all the clusters needed as specified by the scheduling policy
    Observed Generation:   5
    Reason:                SchedulingPolicyFulfilled
    Status:                True
    Type:                  ClusterResourcePlacementScheduled
    Last Transition Time:  2023-11-10T08:23:43Z
    Message:               All 2 cluster(s) are synchronized to the latest resources on the hub cluster
    Observed Generation:   5
    Reason:                SynchronizeSucceeded
    Status:                True
    Type:                  ClusterResourcePlacementSynchronized
    Last Transition Time:  2023-11-10T08:23:43Z
    Message:               Successfully applied resources to 2 member clusters
    Observed Generation:   5
    Reason:                ApplySucceeded
    Status:                True
    Type:                  ClusterResourcePlacementApplied
  Placement Statuses:
    Cluster Name:  aks-member-1
    Conditions:
      Last Transition Time:  2023-11-10T08:14:52Z
      Message:               Successfully scheduled resources for placement in aks-member-1 (affinity score: 0, topology spread score: 0): picked by scheduling policy
      Observed Generation:   5
      Reason:                ScheduleSucceeded
      Status:                True
      Type:                  ResourceScheduled
      Last Transition Time:  2023-11-10T08:23:43Z
      Message:               Successfully Synchronized work(s) for placement
      Observed Generation:   5
      Reason:                WorkSynchronizeSucceeded
      Status:                True
      Type:                  WorkSynchronized
      Last Transition Time:  2023-11-10T08:23:43Z
      Message:               Successfully applied resources
      Observed Generation:   5
      Reason:                ApplySucceeded
      Status:                True
      Type:                  ResourceApplied
    Cluster Name:            aks-member-2
    Conditions:
      Last Transition Time:  2023-11-10T08:14:52Z
      Message:               Successfully scheduled resources for placement in aks-member-2 (affinity score: 0, topology spread score: 0): picked by scheduling policy
      Observed Generation:   5
      Reason:                ScheduleSucceeded
      Status:                True
      Type:                  ResourceScheduled
      Last Transition Time:  2023-11-10T08:23:43Z
      Message:               Successfully Synchronized work(s) for placement
      Observed Generation:   5
      Reason:                WorkSynchronizeSucceeded
      Status:                True
      Type:                  WorkSynchronized
      Last Transition Time:  2023-11-10T08:23:43Z
      Message:               Successfully applied resources
      Observed Generation:   5
      Reason:                ApplySucceeded
      Status:                True
      Type:                  ResourceApplied
  Selected Resources:
    Kind:       Namespace
    Name:       test
    Version:    v1
    Kind:       ConfigMap
    Name:       test-1
    Namespace:  test
    Version:    v1
Events:
  Type    Reason                     Age                    From                                   Message
  ----    ------                     ----                   ----                                   -------
  Normal  PlacementScheduleSuccess   12m (x5 over 3d22h)    cluster-resource-placement-controller  Successfully scheduled the placement
  Normal  PlacementSyncSuccess       3m28s (x7 over 3d22h)  cluster-resource-placement-controller  Successfully synchronized the placement
  Normal  PlacementRolloutCompleted  3m28s (x7 over 3d22h)  cluster-resource-placement-controller  Resources have been applied to the selected clusters
```

### ClusterResourcePlacementStatus (preview)

> [!IMPORTANT]
> The `ClusterResourcePlacementStatus` resource and `StatusReportingScope` field are available in the `placement.kubernetes-fleet.io/v1beta1` API version as a preview feature. They aren't available in the `placement.kubernetes-fleet.io/v1` API.

The `ClusterResourcePlacementStatus` is a namespace-scoped resource that provides the placement status of a corresponding cluster-scoped `ClusterResourcePlacement` object so it's accessible to users of the namespace who don't have cluster-level rights.

For namespace users without cluster-level permissions, you can view the same placement status information through a namespace-scoped `ClusterResourcePlacementStatus` object. This approach requires the `ClusterResourcePlacement` to be configured with `statusReportingScope: NamespaceAccessible` using the v1beta1 API.

Key features:

* **Namespace-scoped access**: Allows users with namespace-level permissions to view placement status without requiring cluster-scoped access.
* **Status mirroring**: Contains the same placement status information as the parent `ClusterResourcePlacement` but is accessible within a specific namespace.
* **Optional feature**: Only created when `StatusReportingScope` is set to `NamespaceAccessible`. Once set, `StatusReportingScope` is immutable.

When `StatusReportingScope` is set to `NamespaceAccessible` for a `ClusterResourcePlacement`, only one namespace resource selector is allowed, and can't be changed after creation.

#### Configuring ClusterResourcePlacementStatus

To use this feature, specify the v1beta1 API version in your `ClusterResourcePlacement`:

```yaml
apiVersion: placement.kubernetes-fleet.io/v1beta1
kind: ClusterResourcePlacement
metadata:
  name: crp-with-status-reporting
spec:
  statusReportingScope: NamespaceAccessible
  resourceSelectors:
    - group: ""
      kind: Namespace
      name: my-app
      version: v1
  policy:
    placementType: PickAll
```

#### Viewing ClusterResourcePlacementStatus

You can view the status using the `kubectl describe` command:

```bash
kubectl describe clusterresourceplacementstatuses.v1beta1.placement.kubernetes-fleet.io crp-with-status-reporting -n my-app
```

The output contains the same status information as the `ClusterResourcePlacement` but is accessible to users with only namespace-level permissions.

For more information, see the [documentation on how to understand the placement result][fleet-status].

## Placement change triggers

The Fleet scheduler prioritizes the stability of existing workload placements. This prioritization can limit the number of changes that cause a workload to be removed and rescheduled. The following scenarios can trigger placement changes:

* Placement policy changes in the `ClusterResourcePlacement` object can trigger removal and rescheduling of a workload.
  * Scale out operations (increasing `numberOfClusters` with no other changes) places workloads only on new clusters and doesn't affect existing placements.
* Cluster changes, including:
  * A new cluster becoming eligible can trigger placement if the new cluster meets the placement policy, for example, a `PickAll` policy.
  * A cluster with a placement is removed from the fleet. Depending on the policy, the scheduler attempts to place all affected workloads on remaining clusters without affecting existing placements.

Resource-only changes (updating the resources or updating the `ResourceSelector` in the `ClusterResourcePlacement` object) roll out gradually in existing placements but do **not** trigger rescheduling of the workload.

:::zone-end

:::zone target="docs" pivot="namespace-scope"

`ResourcePlacement` (RP) was designed to address this gap by providing:

- **Namespace-scoped resource management**: Target specific resources within a namespace without affecting the entire namespace.
- **Operational flexibility**: Allow teams to manage different resources within the same namespace independently.
- **Complementary functionality**: Work alongside CRP to provide a complete multi-cluster resource management solution.

> [!NOTE]
> `ResourcePlacement` can be used together with `ClusterResourcePlacement` in namespace-only mode. For example, you can use CRP to deploy the namespace, while using RP for fine-grained management of specific resources like environment-specific ConfigMaps or Secrets within that namespace.

### Real-world namespace usage patterns

While CRP assumes that namespaces represent application boundaries, real-world usage patterns are often more complex. Organizations frequently use namespaces as team boundaries rather than application boundaries, leading to several challenges that `ResourcePlacement` directly addresses:

**Multi-application namespaces**: In many organizations, a single namespace contains multiple independent applications owned by the same team. These applications might have:

- Different lifecycle requirements (one application might need frequent updates while another remains stable).
- Different cluster placement needs (development vs. production applications).
- Independent scaling and resource requirements.
- Separate compliance or governance requirements.

**Individual scheduling decisions**: Many workloads, particularly AI/ML jobs, require individual scheduling decisions:

- **AI Jobs**: Machine learning workloads often consist of short-lived, resource-intensive jobs that need to be scheduled based on cluster resource availability, GPU availability, or data locality.
- **Batch Workloads**: Different batch jobs within the same namespace might target different cluster types based on computational requirements.

**Complete application team control**: `ResourcePlacement` provides application teams with direct control over their resource placement without requiring platform team intervention:

- **Self-service operations**: Teams can manage their own resource distribution strategies.
- **Independent deployment cycles**: Different applications within a namespace can have independent rollout schedules.
- **Granular override capabilities**: Teams can customize resource configurations per cluster without affecting other applications in the namespace.

This granular approach ensures that `ResourcePlacement` can adapt to diverse organizational structures and workload patterns while maintaining the simplicity and power of the Fleet scheduling framework.

:::zone-end

## Key differences between ResourcePlacement and ClusterResourcePlacement

The following table highlights the key differences between `ResourcePlacement` and `ClusterResourcePlacement`:

| Aspect | ResourcePlacement (RP) | ClusterResourcePlacement (CRP) |
|--------|------------------------|--------------------------------|
| **Scope** | Namespace-scoped resources only | Cluster-scoped resources (especially namespaces and their contents) |
| **Resource** | Namespace-scoped API object | Cluster-scoped API object |
| **Selection Boundary** | Limited to resources within the same namespace as the RP | Can select any cluster-scoped resource |
| **Typical Use Cases** | AI/ML Jobs, individual workloads, specific ConfigMaps/Secrets that need independent placement decisions | Application bundles, entire namespaces, cluster-wide policies |
| **Team Ownership** | Can be managed by namespace owners/developers | Typically managed by platform operators |

Both `ResourcePlacement` and `ClusterResourcePlacement` share the same core capabilities for all other aspects not listed in the differences table.

## Working with ClusterResourcePlacement

`ResourcePlacement` is designed to work in coordination with `ClusterResourcePlacement` (CRP) to provide a complete multi-cluster resource management solution. Understanding this relationship is crucial for effective fleet management.

### Namespace prerequisites

> [!IMPORTANT]
> `ResourcePlacement` can only place namespace-scoped resources to clusters that already have the target namespace. We recommend using `ClusterResourcePlacement` for namespace establishment.

**Typical workflow**:

1. **Platform Admin**: Uses `ClusterResourcePlacement` to deploy namespaces across the fleet.
2. **Application Teams**: Use `ResourcePlacement` to manage specific resources within those established namespaces.

The following examples show how to coordinate CRP and RP:

> [!NOTE]
> The following examples use the `placement.kubernetes-fleet.io/v1beta1` API version. The `selectionScope: NamespaceOnly` field is a preview feature available in v1beta1 and isn't available in the v1 API.

**Platform Admin**: First, create the namespace using `ClusterResourcePlacement`:

```yaml
apiVersion: placement.kubernetes-fleet.io/v1beta1
kind: ClusterResourcePlacement
metadata:
  name: app-namespace-crp
spec:
  resourceSelectors:
    - group: ""
      kind: Namespace
      name: my-app
      version: v1
      selectionScope: NamespaceOnly # only namespace itself is placed, no resources within the namespace
  policy:
    placementType: PickAll # If placement type is not PickAll, the application teams needs to know what are the clusters they can place their applications.
```

**Application Team**: Then, manage specific resources within the namespace using `ResourcePlacement`:

```yaml
apiVersion: placement.kubernetes-fleet.io/v1beta1
kind: ResourcePlacement
metadata:
  name: app-configs-rp
  namespace: my-app
spec:
  resourceSelectors:
    - group: ""
      kind: ConfigMap
      version: v1
      labelSelector:
        matchLabels:
          app: my-application
  policy:
    placementType: PickFixed
    clusterNames:
    - cluster1
    - cluster2
```

### Best practices

When using `ResourcePlacement` with `ClusterResourcePlacement`, follow these best practices:

- **Establish namespaces first**: Always ensure namespaces are deployed via CRP before creating `ResourcePlacement` objects.
- **Monitor dependencies**: Use Fleet monitoring to ensure namespace-level CRPs are healthy before deploying dependent RPs.
- **Coordinate policies**: Align CRP and RP placement policies to avoid conflicts (for example, if CRP places namespace on clusters A, B, C, RP can target any subset of those clusters).
- **Team boundaries**: Use CRP for platform-managed resources (namespaces, RBAC) and RP for application-managed resources (app configs, secrets).

This coordinated approach ensures that `ResourcePlacement` provides the flexibility teams need while maintaining the foundational infrastructure managed by platform operators.

## Resource selection, placement, and rollout

`ResourcePlacement` uses the same placement patterns as `ClusterResourcePlacement`:

- **[Placement types](./concepts-resource-placement.md#placement-types)**: `PickAll`, `PickFixed`, and `PickN` strategies work identically for both APIs.
- **[Rollout strategy](./concepts-rollout-strategy.md)**: Control how updates propagate across clusters with the same rolling update mechanisms.
- **[Status and observability](./howto-understand-placement.md)**: Monitor deployment progress using `kubectl describe resourceplacement <name> -n <namespace>`.
- **[Advanced features](./concepts-resource-placement.md)**: Use tolerations, resource overrides, topology spread constraints, and affinity rules.

The key difference is in **resource selection** scope. While `ClusterResourcePlacement` typically selects entire namespaces and their contents, `ResourcePlacement` provides fine-grained control over individual namespace-scoped resources.

For complete details on these capabilities, refer to the [ClusterResourcePlacement documentation](./concepts-resource-placement.md#resource-selection).

:::zone-end

## Next steps

* [Use cluster resource placement to deploy workloads across multiple clusters](./quickstart-resource-propagation.md).
* [Using ResourcePlacement to deploy namespace-scoped resources](./concepts-namespace-scoped-resource-propagation.md).
* [Intelligent cross-cluster Kubernetes resource placement based on member clusters properties](./intelligent-resource-placement.md).
* [Controlling eviction and disruption for cluster resource placement](./concepts-eviction-disruption.md).
* [Defining a rollout strategy for a cluster resource placement](./concepts-rollout-strategy.md).
* [Cluster resource placement FAQs](./faq.md#cluster-resource-placement-faqs).

<!-- LINKS - external -->
[fleet-github]: https://kubefleet.dev/docs/
[envelope-object]: ./quickstart-envelope-reserved-resources.md
[crp-topo]: https://kubefleet.dev/docs/how-tos/topology-spread-constraints/
[fleet-rollout]: ./concepts-rollout-strategy.md
[fleet-staged-rollout]: ./concepts-rollout-strategy.md#staged-update-strategy-preview
[fleet-tolerations]: ./use-taints-tolerations.md
[fleet-snapshots]: ./concepts-placement-snapshots.md
[fleet-status]: ./howto-understand-placement.md
[vm-sku-name]: /azure/virtual-machines/vm-naming-conventions