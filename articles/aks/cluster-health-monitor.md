---
title: Use Cluster Health Monitor checker in Azure Kubernetes Service (AKS) (preview)
description: Learn how the Cluster Health Monitor checker in Azure Kubernetes Service (AKS) runs data plane health checks and supports CoreDNS remediation.
ms.topic: how-to
ms.service: azure-kubernetes-service
ms.subservice: aks-monitoring
author: varora24
ms.author: vaibhavarora
ms.date: 02/19/2026
ai-usage: ai-assisted
# Customer intent: As a cluster operator, I want to enable Cluster Health Monitor in AKS so that I can run built-in data plane health checks and use CoreDNS remediation safeguards.
---

# Use Cluster Health Monitor checker in Azure Kubernetes Service (AKS) (preview)

This article shows you how to deploy Cluster Health Monitor in Azure Kubernetes Service (AKS).

Cluster Health Monitor helps you detect AKS managed component issues earlier and improve resiliency by enabling automatic remediation for unhealthy CoreDNS pods in specific scenarios. It runs periodic checks for components such as CoreDNS, metrics server, and API server, and exposes results as Prometheus metrics for alerting.

> [!IMPORTANT]
> Cluster Health Monitor is a built-in checker for managed components. It doesn't replace Azure Monitor, Managed Prometheus, or your existing monitoring stack.

[!INCLUDE [preview features callout](~/reusable-content/ce-skilling/azure/includes/aks/includes/preview/preview-callout.md)]

## Overview

Cluster Health Monitor is an AKS-managed add-on deployed in the `kube-system` namespace. It runs continuous in-cluster checks to validate critical components that affect cluster operations and upgrade reliability.

The checker evaluates the following signals:

- **DNS resolution success rate** to detect unhealthy DNS pods that can affect service discovery.
- **API server connectivity** to detect failures reaching the control plane from within the cluster.
- **Metrics server availability** to detect failures in metrics collection.

Cluster Health Monitor exposes check results as Prometheus metrics on port `9800`, so you can scrape and alert on these signals in your existing monitoring pipeline.

For DNS-specific checks, Cluster Health Monitor can automatically remediate a CoreDNS pod that it detects as stuck or unhealthy by deleting that pod. Kubernetes then recreates the pod. Cluster Health Monitor applies this remediation only under specific safeguard conditions to reduce disruption. For details, see [How CoreDNS remediation works](#how-coredns-remediation-works).

## Before you begin

- Install the Azure CLI version 2.XX.0 or later. You can run az --version to verify the version. To install or upgrade, see [Install Azure CLI][azure-cli-install].
- Install the `aks-preview` Azure CLI extension version 19.XX or later:

	```azurecli-interactive
	az extension add --name aks-preview
	```

	If you already installed the extension, update it to the latest version:

	```azurecli-interactive
	az extension update --name aks-preview
	```

- Use the AKS preview managed clusters API version 2026.01.02-preview for this feature.

## Enable Cluster Health Monitor

Cluster Health Monitor has two modes of operation:

- `enableContinuousControlPlaneAndAddonMonitor`
	- Enables continuous in-cluster monitoring of control plane and add-on components.
	- For DNS issues, AKS remediates unhealthy CoreDNS pods if they remain unhealthy for five minutes. See [How CoreDNS remediation works](#how-coredns-remediation-works) for more details.
- `enableOnDemandMonitor`
	- Enables on-demand health monitoring for nodes and control plane.
	- AKS might deploy temporary diagnostic pods on customer nodes to run one-time health checks before or after node operations.

Based on your requirements, you can enable either mode or both modes on your AKS cluster:

```azurecli-interactive
az aks create -g myResourceGroup -n myCluster --enable-continuous-control-plane-monitor
```

```azurecli-interactive
az aks create -g myResourceGroup -n myCluster --enable-ondemand-control-plane-monitor
```

## Verify Cluster Health Monitor

After you enable the feature, you can verify the deployment status and metric endpoint exposure.

1. Get cluster credentials:

	```azurecli-interactive
	az aks get-credentials --resource-group ${RESOURCE_GROUP} --name ${CLUSTER_NAME}
	```

1. Verify that the Cluster Health Monitor Deployment is running in `kube-system`:

	```bash
	kubectl get deployment -n kube-system  cluster-health-monitor
	```

## Disable Cluster Health Monitor

To disable Cluster Health Monitor, you can use the following command:

```azurecli-interactive
az aks update -g myResourceGroup -n myCluster --disable-continuous-control-plane-monitor
```

```azurecli-interactive
az aks update -g myResourceGroup -n myCluster --disable-ondemand-control-plane-monitor
```

## Understand metrics exposed by Cluster Health Monitor

Cluster Health Monitor exposes metrics on port `9800`. You can scrape these metrics with Prometheus and use them to detect add-on health issues.

### Metric statuses

Each check returns one of the following statuses:

| Status | Meaning |
|---|---|
| `Healthy` | The component is operating normally. |
| `Unhealthy` | The check failed. The error code in the metric indicates the failure reason. |
| `Unknown` | The result is inconclusive, for example during pod startup or when a dependency is unavailable. |

### Checks

| Check | What it monitors |
|---|---|
| APIServer | API server reachability by creating, reading, and deleting a test resource. |
| InternalCoreDNS | In-cluster DNS resolution through the CoreDNS service. |
| ExternalCoreDNS | Public DNS resolution through the CoreDNS service. |
| InternalCoreDNSPerPod | In-cluster DNS resolution through each CoreDNS pod. Includes pod name and namespace labels. |
| ExternalCoreDNSPerPod | Public DNS resolution through each CoreDNS pod. Includes pod name and namespace labels. |
| InternalLocalDNS | In-cluster DNS resolution through [LocalDNS](./localdns-custom.md). |
| ExternalLocalDNS | Public DNS resolution through [LocalDNS](./localdns-custom.md). |
| MetricsServer | Metrics server availability. |

## How CoreDNS remediation works

Cluster Health Monitor includes a CoreDNS remediation capability designed to restore DNS health while minimizing risk to DNS availability. It evaluates multiple CoreDNS health signals and remediates only when the cluster can safely absorb a pod restart.

### Signals used for remediation

- Per-pod CoreDNS DNS health check results from internal and external DNS checker runs.
- Duration of unhealthy state for each CoreDNS pod.
- Current health state of other CoreDNS pods in the cluster.
- Recent remediation history to enforce a cool-down period.

### The remediation process

Cluster Health Monitor remediates a CoreDNS pod only when all of the following conditions are true:

- Cluster Health Monitor detects exactly one CoreDNS pod as continuously unhealthy for at least five minutes.
- Cluster Health Monitor detects at least one other CoreDNS pod as healthy.
- No prior Cluster Health Monitor CoreDNS remediation event fired in the last one hour.

When all conditions are met, Cluster Health Monitor deletes one unhealthy CoreDNS pod. Kubernetes then recreates the pod.

This approach helps:

- Restore DNS capacity by replacing only the affected pod.
- Keep at least one healthy CoreDNS replica serving traffic during recovery.
- Avoid repeated remediation cycles during ongoing incidents.

## Next steps

- Review [Configure AKS diagnostics](aks-diagnostics.md).
- Review [Troubleshoot CoreDNS on Azure Kubernetes Service (AKS)](coredns-troubleshoot.md).
- Review [Monitor Azure Kubernetes Service (AKS)](monitor-aks.md).

<!-- LINKS -->
[azure-cli-install]: /cli/azure/install-azure-cli