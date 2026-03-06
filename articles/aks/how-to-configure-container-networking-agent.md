---
title: Deploy and use Container Networking Agent on AKS
description: Learn how to deploy Container Networking Agent as an AKS extension to troubleshoot networking issues in Azure Kubernetes Service (AKS) clusters.
author: shaifaligargmsft
ms.author: shaifaligarg
ms.date: 02/23/2026
ms.topic: how-to
ms.service: azure-kubernetes-service
---

# Deploy and use Container Networking Agent on AKS

In this article, you learn how to deploy Container Networking Agent on your Azure Kubernetes Service (AKS) cluster, configure authentication and identity, and use the agent to troubleshoot networking issues.

Container Networking Agent is an AI-powered diagnostic assistant that runs as an in-cluster web application. You describe networking problems in natural language, and the agent runs diagnostic commands (`kubectl`, `cilium`, `hubble`) against your cluster. It returns structured, evidence-backed reports with root cause analysis and remediation guidance.

Container Networking Agent helps you troubleshoot:

- DNS failures, including CoreDNS misconfigurations, network policies blocking DNS traffic, NodeLocal DNS issues, and Cilium FQDN egress restrictions.
- Packet drops, including NIC-level RX drops, kernel packet loss, socket buffer overflow, SoftIRQ saturation, and ring buffer exhaustion across cluster nodes.
- Kubernetes networking issues, including pod connectivity failures, service port misconfigurations, network policy conflicts, missing endpoints, and Hubble flow analysis.
- Cluster resource queries for quick answers about pods, services, deployments, nodes, and namespaces.

Container Networking Agent operates with read-only access to your cluster. It doesn't modify running workloads, configurations, or network policies. Remediation guidance is advisory only.

You deploy Container Networking Agent as an AKS extension (`microsoft.containernetworkingagent`). It integrates with the Azure resource management plane, and you manage it through the `az k8s-extension` CLI commands.

**Supported regions:** centralus, eastus, eastus2, uksouth, westus2.

## Prerequisites

Before you begin, ensure you have the following tools, permissions, and information.

### Tools required

| Tool | Version | Check command | Install guide |
|------|---------|---------------|---------------|
| Azure CLI | >= 2.77.0 | `az version` | [Install the Azure CLI](/cli/azure/install-azure-cli) |
| kubectl | latest | `kubectl version --client` | `az aks install-cli` |
| jq | any | `jq --version` | [Download jq](https://jqlang.github.io/jq/download/) |
| Git | any | `git --version` | [Install Git](https://git-scm.com/downloads) |

### Azure permissions required

- **Contributor** on the target resource group.
- **User Access Administrator** on the target resource group (for RBAC role assignments).
- Access to create Azure OpenAI resources in your subscription.
- Access to create Entra ID App Registrations (for production authentication).

### Information you'll need

| Parameter | Description |
|-----------|-------------|
| Azure Subscription ID | Your target subscription. |
| Resource group name | Existing or to be created during deployment. |
| Azure region | A supported region: centralus, eastus, eastus2, uksouth, westus2. |
| AKS cluster name | Create a new cluster or use an existing one. |
| `SERVICE_MANAGEMENT_REFERENCE` | Service management reference (for example, a Service Tree ID) required by your tenant for App Registration setup (Step 7). To find this value for an existing app, run: `az ad app show --id <app-id> --query serviceManagementReference -o tsv`. |

### Cluster requirements

- An active Azure subscription. If you don't have one, create a [free account](https://azure.microsoft.com/free/) before you begin.
- An AKS cluster with [workload identity](/azure/aks/workload-identity-overview) and [OIDC issuer](/azure/aks/use-oidc-issuer) enabled, running a [supported Kubernetes version](/azure/aks/supported-kubernetes-versions). You create a new cluster in [Step 2](#step-2-create-an-aks-cluster) if needed.
- (Recommended) [Azure CNI powered by Cilium](/azure/aks/azure-cni-powered-by-cilium) with [Advanced Container Networking Services (ACNS)](/azure/aks/advanced-container-networking-services-overview) enabled, for full diagnostic capabilities including Hubble flow analysis and Cilium policy diagnostics.
- Minimum node size: `Standard_D4_v3` or equivalent (three nodes recommended).
- An [Azure OpenAI](/azure/ai-services/openai/) resource with a deployed model (for example, GPT-4o or later). You create this resource in [Step 3](#step-3-create-an-azure-openai-resource-and-deploy-a-model) if needed.
- The AKS cluster requires outbound connectivity to the Azure OpenAI endpoint over HTTPS (port 443).
- The cluster must be able to pull container images from `acnpublic.azurecr.io`.

> [!TIP]
> Container Networking Agent works on clusters without Cilium or ACNS, but with reduced diagnostic capabilities. On non-ACNS clusters, the agent provides DNS, packet drop, and standard Kubernetes networking diagnostics. Hubble flow analysis and Cilium policy diagnostics aren't available.

## Deploy Container Networking Agent

### Step 1: Set environment variables and create a resource group

Define the core configuration variables and create an Azure resource group to organize all the resources you'll deploy in subsequent steps (AKS cluster, Azure OpenAI service, managed identity). Replace the placeholder values with your own.

```azurecli-interactive
export SUBSCRIPTION_ID="<your-subscription-id>"
export LOCATION="eastus2"
export RESOURCE_GROUP="<your-resource-group>"
export CLUSTER_NAME="<your-aks-cluster-name>"

az login
az account set --subscription $SUBSCRIPTION_ID
```

Create a resource group if one doesn't already exist using the [`az group create`](/cli/azure/group#az-group-create) command.

```azurecli-interactive
az group create \
    --name $RESOURCE_GROUP \
    --location $LOCATION
```

### Step 2: Create an AKS cluster

Container Networking Agent runs inside an AKS cluster and requires several cluster features to function. If you don't already have an AKS cluster, create one using the [`az aks create`](/cli/azure/aks#az-aks-create) command with the following features enabled:

- **Workload identity and OIDC issuer** — Required for secure, secretless authentication between the agent pod and Azure services (OpenAI, AKS API).
- **Cilium dataplane** — Required for network policy enforcement and Cilium policy diagnostics.
- **ACNS (Advanced Container Networking Services)** — Required for Hubble flow analysis and advanced networking diagnostics.

```azurecli-interactive
az aks create \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --location $LOCATION \
    --node-count 3 \
    --node-vm-size Standard_D4_v3 \
    --network-plugin azure \
    --network-plugin-mode overlay \
    --network-dataplane cilium \
    --enable-acns \
    --enable-oidc-issuer \
    --enable-workload-identity \
    --generate-ssh-keys
```

> [!NOTE]
> If you already have a cluster without workload identity and OIDC issuer, enable them using [`az aks update`](/cli/azure/aks#az-aks-update):
>
> ```azurecli-interactive
> az aks update \
>     --resource-group $RESOURCE_GROUP \
>     --name $CLUSTER_NAME \
>     --enable-oidc-issuer \
>     --enable-workload-identity
> ```

Get the cluster credentials using the [`az aks get-credentials`](/cli/azure/aks#az-aks-get-credentials) command.

```azurecli-interactive
az aks get-credentials \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --overwrite-existing
```

### Step 3: Create an Azure OpenAI resource and deploy a model

Container Networking Agent uses Azure OpenAI Service to power its AI reasoning. When you describe a networking issue, the agent sends your query and the collected diagnostic evidence to Azure OpenAI for analysis and response generation. In this step, you create an Azure OpenAI resource and deploy a model (for example, GPT-4o or later) that the agent uses at runtime.

> [!NOTE]
> If you already have an Azure OpenAI resource with a deployed model, skip the resource and model creation commands below. Instead, set the following environment variables to match your existing resource and continue to the next step:
>
> ```azurecli-interactive
> export OPENAI_SERVICE_NAME="<your-existing-openai-service-name>"
> export OPENAI_DEPLOYMENT_NAME="<your-existing-deployment-name>"
> export AZURE_OPENAI_ENDPOINT=$(az cognitiveservices account show \
>     --name $OPENAI_SERVICE_NAME \
>     --resource-group $RESOURCE_GROUP \
>     --query "properties.endpoint" -o tsv)
>
> echo "OpenAI Endpoint: $AZURE_OPENAI_ENDPOINT"
> echo "Deployment Name: $OPENAI_DEPLOYMENT_NAME"
> ```

#### [Azure CLI - Run commands individually](#tab/step3-cli)

If you don't already have an Azure OpenAI resource, create one using the [`az cognitiveservices account create`](/cli/azure/cognitiveservices/account#az-cognitiveservices-account-create) command.

```azurecli-interactive
export OPENAI_SERVICE_NAME="<your-openai-service-name>"
export OPENAI_DEPLOYMENT_NAME="<your-deployment-name>"
export OPENAI_MODEL_NAME="<your-model-name>"
export OPENAI_MODEL_VERSION="<your-model-version>"

az cognitiveservices account create \
    --name $OPENAI_SERVICE_NAME \
    --resource-group $RESOURCE_GROUP \
    --location $LOCATION \
    --kind "OpenAI" \
    --sku "S0" \
    --custom-domain $OPENAI_SERVICE_NAME
```

Wait for provisioning to complete, then deploy the model using the [`az cognitiveservices account deployment create`](/cli/azure/cognitiveservices/account/deployment#az-cognitiveservices-account-deployment-create) command.

> [!TIP]
> The following loop polls the provisioning state every 10 seconds and proceeds automatically once the resource is ready. This is more reliable than a fixed `sleep` delay.

```azurecli-interactive
echo "Waiting for Azure OpenAI resource to be provisioned..."
while true; do
  STATE=$(az cognitiveservices account show \
      --name $OPENAI_SERVICE_NAME \
      --resource-group $RESOURCE_GROUP \
      --query "properties.provisioningState" -o tsv 2>/dev/null)
  if [[ "$STATE" == "Succeeded" ]]; then
    echo "Resource provisioned successfully."
    break
  fi
  echo "  Current state: ${STATE:-Creating}. Retrying in 10 seconds..."
  sleep 10
done

az cognitiveservices account deployment create \
    --name $OPENAI_SERVICE_NAME \
    --resource-group $RESOURCE_GROUP \
    --deployment-name $OPENAI_DEPLOYMENT_NAME \
    --model-name $OPENAI_MODEL_NAME \
    --model-version $OPENAI_MODEL_VERSION \
    --model-format OpenAI \
    --sku-name "GlobalStandard" \
    --sku-capacity 1000
```

Retrieve the endpoint URL using the [`az cognitiveservices account show`](/cli/azure/cognitiveservices/account#az-cognitiveservices-account-show) command.

```azurecli-interactive
export AZURE_OPENAI_ENDPOINT=$(az cognitiveservices account show \
    --name $OPENAI_SERVICE_NAME \
    --resource-group $RESOURCE_GROUP \
    --query "properties.endpoint" -o tsv)

echo "OpenAI Endpoint: $AZURE_OPENAI_ENDPOINT"
```

#### [Script - Run the automated setup script](#tab/step3-script)

Follow these three steps to create and run the setup script.

**1. Create the script file:**

```azurecli-interactive
touch setup-azure-openai.sh && chmod +x setup-azure-openai.sh
```

**2. Copy the following content into `setup-azure-openai.sh`:**

Open the file in your preferred editor and paste the entire script below. The script accepts command-line options (run with `--help` to see all options) or reads from environment variables set in Step 1. At minimum, set `-g` (resource group), `-n` (service name), and `-l` (location), or ensure `RESOURCE_GROUP`, `OPENAI_SERVICE_NAME`, and `LOCATION` are exported.

```bash
#!/bin/bash

# Azure OpenAI Deployment Script
# Creates an Azure OpenAI service and deploys a specified Azure OpenAI model with the required configuration

set -euo pipefail

# Color codes for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Default configuration
DEFAULT_DEPLOYMENT_NAME="${AZURE_OPENAI_DEPLOYMENT:-gpt-5}"
DEFAULT_MODEL_NAME="${AZURE_OPENAI_MODEL_NAME:-gpt-5}"
DEFAULT_MODEL_VERSION="${AZURE_OPENAI_MODEL_VERSION:-2025-08-07}"
DEFAULT_API_VERSION="${AZURE_OPENAI_API_VERSION:-2024-12-01-preview}"
DEFAULT_SKU_NAME="${AZURE_OPENAI_SKU_NAME:-GlobalStandard}"
DEFAULT_SKU_CAPACITY="${AZURE_OPENAI_SKU_CAPACITY:-1000}"  # Capacity (TPM for Standard, PTU for Provisioned)

# Function to print colored output
print_info() {
    echo -e "${BLUE}[INFO]${NC} $1"
}

print_success() {
    echo -e "${GREEN}[SUCCESS]${NC} $1"
}

print_warning() {
    echo -e "${YELLOW}[WARNING]${NC} $1"
}

print_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# Function to check if Azure CLI is installed
check_azure_cli() {
    if ! command -v az &> /dev/null; then
        print_error "Azure CLI is not installed. Please install it first:"
        print_info "Visit: https://learn.microsoft.com/cli/azure/install-azure-cli"
        exit 1
    fi
}

# Function to check if user is logged in to Azure
check_azure_login() {
    if ! az account show &> /dev/null; then
        print_error "Not logged in to Azure. Please run 'az login' first."
        exit 1
    fi
}

# Function to validate parameters
validate_parameters() {
    if [[ -z "${RESOURCE_GROUP:-}" ]]; then
        print_error "Resource group name is required"
        print_info "You can set it via -g option or AZURE_OPENAI_RESOURCE_GROUP environment variable"
        exit 1
    fi
    
    if [[ -z "${OPENAI_SERVICE_NAME:-}" ]]; then
        print_error "OpenAI service name is required"
        print_info "You can set it via -n option or AZURE_OPENAI_SERVICE_NAME environment variable"
        exit 1
    fi
    
    if [[ -z "${LOCATION:-}" ]]; then
        print_error "Location is required"
        print_info "You can set it via -l option or AZURE_OPENAI_LOCATION environment variable"
        exit 1
    fi

    # Validate model availability dynamically
    print_info "Validating availability of model '${MODEL_NAME}' version '${MODEL_VERSION}' in '${LOCATION}'..."
    
    # Fetch available versions for the model in the specified location
    local available_versions
    if ! available_versions=$(az cognitiveservices model list -l "${LOCATION}" \
        --query "[?kind=='OpenAI' && model.name=='${MODEL_NAME}'].model.version" \
        -o tsv 2>/dev/null); then
        print_error "Failed to fetch model list from Azure. Please check your connection and location '${LOCATION}'."
        exit 1
    fi
    
    if [[ -z "$available_versions" ]]; then
        print_error "Model '${MODEL_NAME}' is not available in region '${LOCATION}'."
        print_info "Please check supported regions for this model."
        exit 1
    fi

    # Check if exact version exists
    if ! echo "$available_versions" | grep -Fxq "${MODEL_VERSION}"; then
        print_error "Version '${MODEL_VERSION}' for model '${MODEL_NAME}' is not available in region '${LOCATION}'."
        print_info "Available versions: $(echo "$available_versions" | tr '\n' ',' | sed 's/,$//; s/,/, /g')"
        exit 1
    else
        print_success "Model '${MODEL_NAME}' version '${MODEL_VERSION}' is available."
    fi
}

# Function to check if resource group exists, create if not
setup_resource_group() {
    print_info "Checking if resource group '${RESOURCE_GROUP}' exists..."
    
    if az group show --name "${RESOURCE_GROUP}" &> /dev/null; then
        print_info "Resource group '${RESOURCE_GROUP}' already exists"
    else
        print_info "Creating resource group '${RESOURCE_GROUP}' in location '${LOCATION}'..."
        az group create \
            --name "${RESOURCE_GROUP}" \
            --location "${LOCATION}" \
            --output table
        print_success "Resource group created successfully"
    fi
}

# Function to check if OpenAI service exists
create_openai_service() {
    print_info "Checking if OpenAI service '${OPENAI_SERVICE_NAME}' exists..."
    
    if az cognitiveservices account show \
        --name "${OPENAI_SERVICE_NAME}" \
        --resource-group "${RESOURCE_GROUP}" &> /dev/null; then
        print_info "OpenAI service '${OPENAI_SERVICE_NAME}' already exists"
    else
        print_info "Creating Azure OpenAI service '${OPENAI_SERVICE_NAME}'..."
        az cognitiveservices account create \
            --name "${OPENAI_SERVICE_NAME}" \
            --resource-group "${RESOURCE_GROUP}" \
            --location "${LOCATION}" \
            --kind "OpenAI" \
            --sku "S0" \
            --custom-domain "${OPENAI_SERVICE_NAME}" \
            --output table
        print_success "OpenAI service created successfully"
        
        # Wait for service to be fully provisioned
        print_info "Waiting for service to be ready..."
        sleep 30
    fi
}

# Function to list all deployments in the resource group for debugging quota
list_current_deployments() {
    print_info "Scanning resource group '${RESOURCE_GROUP}' for existing deployments..."
    local accounts
    accounts=$(az cognitiveservices account list -g "${RESOURCE_GROUP}" --query "[?kind=='OpenAI'].name" -o tsv 2>/dev/null || echo "")
    
    if [[ -z "$accounts" ]]; then
        print_info "No OpenAI accounts found in this resource group."
        return
    fi
    
    for account in $accounts; do
        echo -e "\nDeployments for service: ${BLUE}${account}${NC}"
        az cognitiveservices account deployment list \
            -g "${RESOURCE_GROUP}" \
            -n "$account" \
            --query "[].{Name:name, Model:properties.model.name, Version:properties.model.version, SKU:sku.name, Capacity:sku.capacity}" \
            -o table
    done
}

# Function to check quota availability
check_quota() {
    print_info "Checking quota availability for ${SKU_NAME}..."
    
    local quota_name
    case "${SKU_NAME}" in
        "GlobalStandard")
            quota_name="OpenAI.GlobalStandard.${MODEL_NAME}"
            ;;
        "Standard")
             quota_name="OpenAI.Standard.${MODEL_NAME}"
            ;;
        "ProvisionedManaged")
            quota_name="Provisioned Managed Throughput Unit"
            ;;
        "GlobalProvisionedManaged")
            quota_name="Global Provisioned Managed Throughput Unit"
            ;;
        "DataZoneProvisionedManaged")
            quota_name="Data Zone Provisioned Managed Throughput Unit"
            ;;
        *)
            print_warning "Skipping quota check for SKU '${SKU_NAME}' (not auto-verified)"
            return 0
            ;;
    esac

    # Fetch usage data using Python for reliable JSON parsing
    local quota_values
    quota_values=$(az cognitiveservices usage list --location "${LOCATION}" --output json 2>/dev/null | python3 -c "
import sys, json
try:
    data = json.load(sys.stdin)
    # Search for quota by name or localized name
    quota = next((x for x in data if x['name']['value'] == '${quota_name}' or x['name']['localizedValue'] == '${quota_name}'), None)
    if quota:
        print(f\"{quota['currentValue']} {quota['limit']}\")
except Exception:
    pass
")

    if [[ -z "$quota_values" ]]; then
        print_warning "Could not determine quota usage for '${quota_name}'. Proceeding..."
        return 0
    fi

    # Parse values
    local current_value
    local limit_value
    current_value=$(echo "$quota_values" | awk '{print $1}')
    limit_value=$(echo "$quota_values" | awk '{print $2}')
    
    # Remove decimal places for integer arithmetic
    current_value=${current_value%.*}
    limit_value=${limit_value%.*}
    
    print_info "Quota '${quota_name}': Using ${current_value}/${limit_value}"
    
    if (( current_value + SKU_CAPACITY > limit_value )); then
        print_error "Insufficient quota!"
        print_error "Requesting ${SKU_CAPACITY} but only $(( limit_value - current_value )) is available."
        print_error "Current usage: ${current_value}, Limit: ${limit_value}"
        
        echo
        print_warning "Detailed usage report for resource group '${RESOURCE_GROUP}':"
        list_current_deployments
        echo
        
        print_info "Tip: Use a different region, a different SKU, or request a quota increase."
        exit 1
    fi
}

# Function to deploy model
deploy_model() {
    print_info "Checking if model deployment '${DEPLOYMENT_NAME}' exists..."
    
    if az cognitiveservices account deployment show \
        --name "${OPENAI_SERVICE_NAME}" \
        --resource-group "${RESOURCE_GROUP}" \
        --deployment-name "${DEPLOYMENT_NAME}" &> /dev/null; then
        print_info "Model deployment '${DEPLOYMENT_NAME}' already exists"
    else
        # Check quota
        check_quota
        
        print_info "Deploying ${MODEL_NAME} model as '${DEPLOYMENT_NAME}' with ${SKU_NAME} SKU..."
        print_warning "Note: ${MODEL_NAME} requires provisioned throughput which has higher costs than standard deployments"
        
        # Attempt deployment with error handling
        if az cognitiveservices account deployment create \
            --name "${OPENAI_SERVICE_NAME}" \
            --resource-group "${RESOURCE_GROUP}" \
            --deployment-name "${DEPLOYMENT_NAME}" \
            --model-name "${MODEL_NAME}" \
            --model-version "${MODEL_VERSION}" \
            --model-format "OpenAI" \
            --sku-capacity "${SKU_CAPACITY}" \
            --sku-name "${SKU_NAME}" \
            --output table; then
            
            print_success "Model deployment completed successfully"
            
            # Wait for deployment to be ready
            print_info "Waiting for deployment to be ready..."
            sleep 60
        else
            print_error "Model deployment failed!"
            print_info ""
            print_info "Common reasons for failure:"
            print_info "1. Insufficient provisioned throughput quota"
            print_info "2. Model not available in the selected region"
            print_info "3. Invalid capacity or SKU combination"
            print_info ""
            print_info "To request provisioned throughput quota:"
            print_info "1. Go to Azure portal > Cognitive Services > Quotas"
            print_info "2. Select your subscription and region"
            print_info "3. Request quota increase for 'Provisioned Throughput Units'"
            print_info "4. Wait for approval (can take 1-2 business days)"
            print_info ""
            print_info "Alternative: Try using gpt-4o with Standard SKU instead:"
            print_info "  $0 -m 'gpt-4o' -v '2024-11-20' --sku-name 'Standard' --sku-capacity '20'"
            return 1
        fi
    fi
}

# Function to get service details
get_service_details() {
    print_info "Retrieving service configuration details..."
    
    # Get endpoint
    ENDPOINT=$(az cognitiveservices account show \
        --name "${OPENAI_SERVICE_NAME}" \
        --resource-group "${RESOURCE_GROUP}" \
        --query "properties.endpoint" \
        --output tsv)
    
    # Get API key
    API_KEY=$(az cognitiveservices account keys list \
        --name "${OPENAI_SERVICE_NAME}" \
        --resource-group "${RESOURCE_GROUP}" \
        --query "key1" \
        --output tsv)
    
    print_success "Service details retrieved successfully"
}

# Function to generate environment configuration
generate_env_config() {
    local config_file="${1:-azure-openai-config.env}"
    
    print_info "Generating environment configuration file: ${config_file}"
    
    cat > "${config_file}" << EOF
# Azure OpenAI Configuration (Generated by setup-azure-openai.sh)
# Created: $(date)
# Service: ${OPENAI_SERVICE_NAME}
# Resource Group: ${RESOURCE_GROUP}
# Location: ${LOCATION}

# Azure OpenAI Configuration (Required)
AZURE_OPENAI_ENDPOINT=${ENDPOINT}
AZURE_OPENAI_DEPLOYMENT=${DEPLOYMENT_NAME}
AZURE_OPENAI_CHAT_DEPLOYMENT_NAME=${DEPLOYMENT_NAME}
AZURE_OPENAI_API_VERSION=${API_VERSION}

# Azure OpenAI API Key (Keep this secure!)
AZURE_OPENAI_API_KEY=${API_KEY}

# Azure Resource Information
AZURE_OPENAI_SERVICE_NAME=${OPENAI_SERVICE_NAME}
AZURE_OPENAI_RESOURCE_GROUP=${RESOURCE_GROUP}
AZURE_OPENAI_LOCATION=${LOCATION}
EOF
    
    # Set secure permissions on the config file
    chmod 600 "${config_file}"
    
    print_success "Configuration file created: ${config_file}"
    print_warning "IMPORTANT: Keep the configuration file secure as it contains API keys!"
}

# Function to display usage information
usage() {
    cat << EOF
Usage: $0 [OPTIONS]

This script creates an Azure OpenAI service and deploys a GPT model.

OPTIONS:
    -g, --resource-group NAME    Azure resource group name (required)
    -n, --service-name NAME      OpenAI service name (required)
    -l, --location LOCATION      Azure region (required)
    -d, --deployment-name NAME   Model deployment name (default: ${DEFAULT_DEPLOYMENT_NAME})
    -m, --model-name NAME        Model name (default: ${DEFAULT_MODEL_NAME})
    -v, --model-version VERSION  Model version (default: ${DEFAULT_MODEL_VERSION})
    -a, --api-version VERSION    API version (default: ${DEFAULT_API_VERSION})
    -s, --sku-name SKU           Deployment SKU name (default: ${DEFAULT_SKU_NAME})
    -c, --sku-capacity CAPACITY  SKU capacity (default: ${DEFAULT_SKU_CAPACITY})
    -o, --output-file FILE       Output configuration file (default: azure-openai-config.env)
    -h, --help                   Show this help message

EXAMPLES:
    # Create with default settings (GlobalStandard deployment)
    $0
    
    # Create with custom resource group and service name
    $0 -g "my-rg" -n "my-openai-service"
    
    # Create in UK South region (default for GPT-5)
    $0 -l "uksouth"
    
    # Use different capacity (TPM for Standard, PTU for Provisioned)
    $0 --sku-capacity "50"
    
    # Use Provisioned Managed SKU (requires quota)
    $0 --sku-name "ProvisionedManaged" --sku-capacity "100"
    
    # Generate config to custom file
    $0 -o "config.env"

SUPPORTED REGIONS for GPT-5 (2025-08-07):
    - uksouth (default)
    - eastus2
    - westus
    - westus3
    - northcentralus
    - koreacentral
    - japaneast
    - uaenorth
    - swedencentral
    - switzerlandnorth

NOTE: 
    - Requires Azure CLI to be installed and authenticated
    - GPT-5 uses GlobalStandard deployment by default (pay-per-token)
    - GlobalStandard provides 1000 TPM quota for GPT-5
    - Default capacity is set to 1000 TPM
    - Provisioned throughput deployment requires special quota approval
EOF
}

# Main function
main() {
    # Parse command line arguments
    while [[ $# -gt 0 ]]; do
        case $1 in
            -g|--resource-group)
                RESOURCE_GROUP="$2"
                shift 2
                ;;
            -n|--service-name)
                OPENAI_SERVICE_NAME="$2"
                shift 2
                ;;
            -l|--location)
                LOCATION="$2"
                shift 2
                ;;
            -d|--deployment-name)
                DEPLOYMENT_NAME="$2"
                shift 2
                ;;
            -m|--model-name)
                MODEL_NAME="$2"
                shift 2
                ;;
            -v|--model-version)
                MODEL_VERSION="$2"
                shift 2
                ;;
            -a|--api-version)
                API_VERSION="$2"
                shift 2
                ;;
            -s|--sku-name)
                SKU_NAME="$2"
                shift 2
                ;;
            -c|--sku-capacity)
                SKU_CAPACITY="$2"
                shift 2
                ;;
            -o|--output-file)
                OUTPUT_FILE="$2"
                shift 2
                ;;
            -h|--help)
                usage
                exit 0
                ;;
            *)
                print_error "Unknown option: $1"
                usage
                exit 1
                ;;
        esac
    done
    
    # Set defaults if not provided
    RESOURCE_GROUP="${RESOURCE_GROUP:-${AZURE_OPENAI_RESOURCE_GROUP:-}}"
    OPENAI_SERVICE_NAME="${OPENAI_SERVICE_NAME:-${AZURE_OPENAI_SERVICE_NAME:-}}"
    LOCATION="${LOCATION:-${AZURE_OPENAI_LOCATION:-}}"
    DEPLOYMENT_NAME="${DEPLOYMENT_NAME:-$DEFAULT_DEPLOYMENT_NAME}"
    MODEL_NAME="${MODEL_NAME:-$DEFAULT_MODEL_NAME}"
    MODEL_VERSION="${MODEL_VERSION:-$DEFAULT_MODEL_VERSION}"
    API_VERSION="${API_VERSION:-$DEFAULT_API_VERSION}"
    SKU_NAME="${SKU_NAME:-$DEFAULT_SKU_NAME}"
    SKU_CAPACITY="${SKU_CAPACITY:-$DEFAULT_SKU_CAPACITY}"
    OUTPUT_FILE="${OUTPUT_FILE:-azure-openai-config.env}"
    
    # Check prerequisites
    check_azure_cli
    check_azure_login

    # Validate parameters
    validate_parameters

    # Display configuration
    echo
    print_info "Azure OpenAI Deployment Configuration:"
    echo -e "  Resource Group: ${GREEN}${RESOURCE_GROUP}${NC}"
    echo -e "  Service Name: ${GREEN}${OPENAI_SERVICE_NAME}${NC}"
    echo -e "  Location: ${GREEN}${LOCATION}${NC}"
    echo -e "  Deployment Name: ${GREEN}${DEPLOYMENT_NAME}${NC}"
    echo -e "  Model: ${GREEN}${MODEL_NAME}${NC} (${GREEN}${MODEL_VERSION}${NC})"
    echo -e "  SKU: ${GREEN}${SKU_NAME}${NC} (Capacity: ${GREEN}${SKU_CAPACITY}${NC})"
    echo -e "  API Version: ${GREEN}${API_VERSION}${NC}"
    echo -e "  Output File: ${GREEN}${OUTPUT_FILE}${NC}"
    echo
    
    # Confirm execution
    echo -n "Do you want to proceed with these settings? (y/N): "
    read -r proceed
    if [[ ! "$proceed" =~ ^[Yy]$ ]]; then
        echo -e "${YELLOW}Deployment cancelled by user.${NC}"
        exit 0
    fi

    # Run deployment steps
    print_info "Starting Azure OpenAI deployment..."
    echo
    
    setup_resource_group
    create_openai_service
    deploy_model
    get_service_details
    generate_env_config "$OUTPUT_FILE"
    
    echo
    print_success "Azure OpenAI deployment completed successfully!"
    print_info "Configuration details have been written to the environment file."
    echo
    print_info "You can now use these settings in your application:"
    print_info "source ${OUTPUT_FILE}"
    echo
    if [[ "${SKU_NAME}" =~ "Provisioned" ]]; then
        print_warning "IMPORTANT: This deployment uses provisioned throughput billing!"
        print_warning "You will be charged for the provisioned capacity even when not in use."
        print_warning "Monitor your usage in the Azure portal to avoid unexpected costs."
        echo
    fi
    print_warning "Remember to keep your API keys secure and never commit them to version control!"
}

# Run main function with all arguments
main "$@"
```

**3. Run the script:**

```azurecli-interactive
./setup-azure-openai.sh -g "$RESOURCE_GROUP" -n "$OPENAI_SERVICE_NAME" -l "$LOCATION"
```

> [!NOTE]
> Use `source` (instead of `./`) so the exported environment variables (`AZURE_OPENAI_ENDPOINT`, `OPENAI_DEPLOYMENT_NAME`, `OPENAI_SERVICE_NAME`) remain available in your current shell for subsequent steps.

#### [Portal - Use the Azure portal UI](#tab/step3-portal)

Create an Azure OpenAI resource and deploy a model directly from the Azure portal.

Follow the step-by-step guide on Microsoft Learn:

> [Create and deploy an Azure OpenAI Service resource - Azure portal](/azure/ai-foundry/openai/how-to/create-resource?pivots=web-portal)

The guide walks you through:

1. Creating an Azure OpenAI resource in your subscription and resource group.
2. Deploying a model (for example, GPT-4.1 or later) to the resource.
3. Retrieving the endpoint URL and resource name.

> [!IMPORTANT]
> After creating the resource and deploying a model through the portal, set the following environment variables in your terminal before continuing to the next step. These values are required by subsequent deployment steps.
>
> ```azurecli-interactive
> export OPENAI_SERVICE_NAME="<your-openai-service-name>"       # The name you gave your Azure OpenAI resource
> export OPENAI_DEPLOYMENT_NAME="<your-model-deployment-name>"  # The deployment name shown in the portal
> export AZURE_OPENAI_ENDPOINT="<your-openai-endpoint>"         # For example, https://<name>.openai.azure.com/
>
> echo "OpenAI Endpoint: $AZURE_OPENAI_ENDPOINT"
> echo "Deployment Name: $OPENAI_DEPLOYMENT_NAME"
> ```

---

### Steps 4-6: Create a managed identity, assign RBAC roles, and configure federated credentials

Container Networking Agent needs an Azure identity to authenticate with Azure services at runtime. In these steps, you create a managed identity, grant it the minimum required permissions to access Azure OpenAI and AKS resources, and link it to the Kubernetes service account that the agent pod runs under. This approach uses [AKS workload identity](/azure/aks/workload-identity-overview) to eliminate the need for secrets or connection strings in your cluster.

#### [Azure CLI](#tab/step456-cli)

**Step 4: Create a managed identity**

Create a user-assigned managed identity that serves as the agent's Azure identity. The agent pod uses this identity to authenticate with Azure OpenAI for AI reasoning and with the AKS API for cluster diagnostics. Use the [`az identity create`](/cli/azure/identity#az-identity-create) command.

```azurecli-interactive
export IDENTITY_NAME="container-networking-agent-reader-identity"

az identity create \
    --name $IDENTITY_NAME \
    --resource-group $RESOURCE_GROUP \
    --location $LOCATION \
    --tags purpose=k8s-workload-identity
```

Get the identity client ID and principal ID using the [`az identity show`](/cli/azure/identity#az-identity-show) command.

```azurecli-interactive
export AZURE_CLIENT_ID=$(az identity show \
    --name $IDENTITY_NAME \
    --resource-group $RESOURCE_GROUP \
    --query "clientId" -o tsv)

export IDENTITY_PRINCIPAL_ID=$(az identity show \
    --name $IDENTITY_NAME \
    --resource-group $RESOURCE_GROUP \
    --query "principalId" -o tsv)

echo "Client ID: $AZURE_CLIENT_ID"
echo "Principal ID: $IDENTITY_PRINCIPAL_ID"
```

**Step 5: Assign RBAC roles**

Grant the managed identity the minimum permissions it needs to operate. Each role is scoped to the narrowest resource possible, following the principle of least privilege. Use the [`az role assignment create`](/cli/azure/role/assignment#az-role-assignment-create) command. The managed identity needs the following roles:

| Role | Scope | Purpose |
|------|-------|---------|
| `Cognitive Services OpenAI User` | Azure OpenAI resource | Allows the agent to make inference calls to the deployed model. |
| `Azure Kubernetes Service Cluster User Role` | AKS cluster | Allows the agent to access AKS cluster properties. |
| `Azure Kubernetes Service Contributor Role` | AKS cluster | Allows the agent to read and write AKS cluster information for MCP AKS operations. |
| `Reader` | Resource group | Allows the agent to read resource group metadata. |

```azurecli-interactive
# Cognitive Services OpenAI User on the OpenAI resource
az role assignment create \
    --assignee $IDENTITY_PRINCIPAL_ID \
    --role "Cognitive Services OpenAI User" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.CognitiveServices/accounts/$OPENAI_SERVICE_NAME"

# Azure Kubernetes Service Cluster User Role on the AKS cluster
az role assignment create \
    --assignee $IDENTITY_PRINCIPAL_ID \
    --role "Azure Kubernetes Service Cluster User Role" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ContainerService/managedClusters/$CLUSTER_NAME"

# Azure Kubernetes Service Contributor Role on the AKS cluster
az role assignment create \
    --assignee $IDENTITY_PRINCIPAL_ID \
    --role "Azure Kubernetes Service Contributor Role" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ContainerService/managedClusters/$CLUSTER_NAME"

# Reader on the resource group
az role assignment create \
    --assignee $IDENTITY_PRINCIPAL_ID \
    --role "Reader" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"
```

> [!IMPORTANT]
> Azure RBAC role assignments can take up to 10 minutes to propagate. If the agent pod shows `401` or `403` errors after installation, wait a few minutes and restart the pod.

**Step 6: Configure federated credentials**

Create a federated credential that links the managed identity to the Kubernetes service account used by the agent pod. This link enables [AKS workload identity](/azure/aks/workload-identity-overview): at runtime, AKS injects a federated token into the agent pod, and the agent exchanges this token for a Microsoft Entra ID access token to call Azure OpenAI and other Azure services. No secrets or connection strings are stored in the cluster. Use the [`az identity federated-credential create`](/cli/azure/identity/federated-credential#az-identity-federated-credential-create) command.

```azurecli-interactive
export OIDC_ISSUER_URL=$(az aks show \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --query "oidcIssuerProfile.issuerUrl" -o tsv)

az identity federated-credential create \
    --name "container-networking-agent-k8s-fed-cred" \
    --identity-name $IDENTITY_NAME \
    --resource-group $RESOURCE_GROUP \
    --issuer $OIDC_ISSUER_URL \
    --subject "system:serviceaccount:kube-system:container-networking-agent-reader" \
    --audiences "api://AzureADTokenExchange"
```

#### [Script](#tab/step456-script)

Follow these three steps to create and run the setup script.

**1. Create the script file:**

```azurecli-interactive
touch setup-managed-identity-federated.sh && chmod +x setup-managed-identity-federated.sh
```

**2. Copy the following content into `setup-managed-identity-federated.sh`:**

Open the file in your preferred editor and paste the entire script below. The script reads from environment variables set in previous steps. At minimum, ensure `LOCATION`, `RESOURCE_GROUP`, `AKS_CLUSTER_NAME`, and `LLM_NAME` (your Azure OpenAI resource name) are exported. The script requires `jq` for JSON parsing.

```bash
#!/bin/bash

# Script to create Azure User-Assigned Managed Identity with Federated Credentials
# for Kubernetes Service Account Authentication
# This script follows Azure best practices for workload identity federation
# 
# Target: Service Account 'container-networking-agent-reader' in 'kube-system' namespace (updated to match Helm chart)
# 
# This script is idempotent - safe to run multiple times

set -e

# Configuration - modify these variables as needed or set as environment variables
IDENTITY_NAME="${IDENTITY_NAME:-container-networking-agent-reader-identity}"
SERVICE_ACCOUNT_NAMESPACE="${SERVICE_ACCOUNT_NAMESPACE:-kube-system}"
SERVICE_ACCOUNT_NAME="${SERVICE_ACCOUNT_NAME:-container-networking-agent-reader}"
FEDERATED_CREDENTIAL_NAME="${FEDERATED_CREDENTIAL_NAME:-container-networking-agent-k8s-fed-cred}"

# Get current context
SUBSCRIPTION_ID=$(az account show --query id -o tsv 2>/dev/null || echo "")

# Validate all required variables
: "${IDENTITY_NAME:?ERROR: IDENTITY_NAME is required}"
: "${LOCATION:?ERROR: LOCATION is required}"
: "${RESOURCE_GROUP:?ERROR: RESOURCE_GROUP is required}"
: "${AKS_CLUSTER_NAME:?ERROR: AKS_CLUSTER_NAME is required}"
: "${SERVICE_ACCOUNT_NAMESPACE:?ERROR: SERVICE_ACCOUNT_NAMESPACE is required}"
: "${SERVICE_ACCOUNT_NAME:?ERROR: SERVICE_ACCOUNT_NAME is required}"
: "${LLM_NAME:?ERROR: LLM_NAME (Azure OpenAI resource name) is required}"
: "${SUBSCRIPTION_ID:?ERROR: Could not get current Azure subscription}"

# Optional: separate resource group for the Azure OpenAI resource; default to main RESOURCE_GROUP
LLM_RESOURCE_GROUP="${LLM_RESOURCE_GROUP:-$RESOURCE_GROUP}"

# Colors for output
readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[1;33m'
readonly BLUE='\033[0;34m'
readonly CYAN='\033[0;36m'
readonly NC='\033[0m' # No Color

# Logging functions with timestamps
log_info() {
    echo -e "$(date '+%Y-%m-%d %H:%M:%S') ${GREEN}[INFO]${NC} $1" >&2
}

log_warn() {
    echo -e "$(date '+%Y-%m-%d %H:%M:%S') ${YELLOW}[WARN]${NC} $1" >&2
}

log_error() {
    echo -e "$(date '+%Y-%m-%d %H:%M:%S') ${RED}[ERROR]${NC} $1" >&2
}

log_step() {
    echo -e "$(date '+%Y-%m-%d %H:%M:%S') ${BLUE}[STEP]${NC} $1" >&2
}

log_debug() {
    if [[ "${DEBUG:-}" == "true" ]]; then
        echo -e "$(date '+%Y-%m-%d %H:%M:%S') ${CYAN}[DEBUG]${NC} $1" >&2
    fi
}

# Function to check if Azure CLI is logged in and has required permissions
check_azure_prerequisites() {
    log_step "Checking Azure CLI prerequisites..."
    
    if ! command -v az >/dev/null 2>&1; then
        log_error "Azure CLI is not installed. Please install it first:"
        log_error "https://learn.microsoft.com/cli/azure/install-azure-cli"
        exit 1
    fi
    
    if ! az account show >/dev/null 2>&1; then
        log_error "Not logged into Azure CLI. Please run 'az login' first."
        exit 1
    fi
    
    local current_subscription
    current_subscription=$(az account show --query id -o tsv)
    if [[ "$current_subscription" != "$SUBSCRIPTION_ID" ]]; then
        log_warn "Current subscription ($current_subscription) doesn't match target ($SUBSCRIPTION_ID)"
        log_info "Setting subscription to $SUBSCRIPTION_ID"
        az account set --subscription "$SUBSCRIPTION_ID"
    fi
    
    # Check if resource group exists
    if ! az group show --name "$RESOURCE_GROUP" >/dev/null 2>&1; then
        log_error "Resource group '$RESOURCE_GROUP' does not exist"
        exit 1
    fi

    # If OpenAI lives in a different RG, verify that one too
    if [[ "$LLM_RESOURCE_GROUP" != "$RESOURCE_GROUP" ]]; then
        if ! az group show --name "$LLM_RESOURCE_GROUP" >/dev/null 2>&1; then
            log_error "OpenAI resource group '$LLM_RESOURCE_GROUP' (LLM_RESOURCE_GROUP) does not exist"
            exit 1
        fi
        log_info "✅ Detected separate OpenAI resource group: $LLM_RESOURCE_GROUP"
    fi
    
    # Check if AKS cluster exists
    if ! az aks show --resource-group "$RESOURCE_GROUP" --name "$AKS_CLUSTER_NAME" >/dev/null 2>&1; then
        log_error "AKS cluster '$AKS_CLUSTER_NAME' does not exist in resource group '$RESOURCE_GROUP'"
        exit 1
    fi
    
    log_info "✅ Azure CLI prerequisites verified"
}

# Function to check and enable AKS workload identity if needed
check_aks_workload_identity() {
    log_step "Checking AKS workload identity configuration..."
    
    local cluster_info
    cluster_info=$(az aks show --resource-group "$RESOURCE_GROUP" --name "$AKS_CLUSTER_NAME" --query "{oidcIssuerEnabled: oidcIssuerProfile.enabled, workloadIdentityEnabled: securityProfile.workloadIdentity.enabled, oidcIssuerUrl: oidcIssuerProfile.issuerUrl}" -o json 2>/dev/null)

    local oidc_enabled
    oidc_enabled=$(echo "$cluster_info" | jq -r '.oidcIssuerEnabled // false')
    local workload_identity_enabled
    workload_identity_enabled=$(echo "$cluster_info" | jq -r '.workloadIdentityEnabled // false')
    local oidc_issuer_url
    oidc_issuer_url=$(echo "$cluster_info" | jq -r '.oidcIssuerUrl // empty')
    
    log_debug "OIDC Issuer Enabled: $oidc_enabled"
    log_debug "Workload Identity Enabled: $workload_identity_enabled"
    log_debug "OIDC Issuer URL: $oidc_issuer_url"
    
    if [[ "$oidc_enabled" != "true" || "$workload_identity_enabled" != "true" || -z "$oidc_issuer_url" ]]; then
        log_warn "AKS cluster does not have workload identity properly enabled"
        log_info "Enabling OIDC issuer and workload identity on AKS cluster..."
        
        az aks update \
            --resource-group "$RESOURCE_GROUP" \
            --name "$AKS_CLUSTER_NAME" \
            --enable-oidc-issuer \
            --enable-workload-identity \
            --output none
            
        log_info "✅ AKS workload identity enabled"
        
        # Re-fetch the OIDC issuer URL
        oidc_issuer_url=$(az aks show --resource-group "$RESOURCE_GROUP" --name "$AKS_CLUSTER_NAME" --query "oidcIssuerProfile.issuerUrl" -o tsv 2>/dev/null)
    fi
    
    if [[ -z "$oidc_issuer_url" ]]; then
        log_error "Could not retrieve OIDC issuer URL after enabling workload identity"
        exit 1
    fi
    
    log_info "✅ AKS OIDC issuer URL: $oidc_issuer_url"
    echo "$oidc_issuer_url"
}

# Function to create user-assigned managed identity
create_managed_identity() {
    log_step "Creating user-assigned managed identity '$IDENTITY_NAME'..."
    
    if az identity show --name "$IDENTITY_NAME" --resource-group "$RESOURCE_GROUP" >/dev/null 2>&1; then
        log_info "✅ Managed identity '$IDENTITY_NAME' already exists"
        return 0
    fi
    
    log_info "Creating new managed identity '$IDENTITY_NAME' in resource group '$RESOURCE_GROUP'"
    az identity create \
        --name "$IDENTITY_NAME" \
        --resource-group "$RESOURCE_GROUP" \
        --location "$LOCATION" \
        --tags purpose=k8s-workload-identity service-account="$SERVICE_ACCOUNT_NAMESPACE/$SERVICE_ACCOUNT_NAME" \
        --output none
    
    log_info "✅ Managed identity '$IDENTITY_NAME' created successfully"
}

# Function to get managed identity details
get_identity_details() {
    local identity_info
    identity_info=$(az identity show \
        --name "$IDENTITY_NAME" \
        --resource-group "$RESOURCE_GROUP" \
        --query "{clientId: clientId, principalId: principalId, resourceId: id}" \
        -o json 2>/dev/null)
    
    if [[ -z "$identity_info" ]]; then
        log_error "Could not retrieve managed identity details"
        exit 1
    fi
    
    echo "$identity_info"
}

# Function to create federated credential
create_federated_credential() {
    local oidc_issuer_url="$1"
    local client_id="$2"
    
    log_step "Creating federated credential for Kubernetes service account..."
    
    local subject="system:serviceaccount:${SERVICE_ACCOUNT_NAMESPACE}:${SERVICE_ACCOUNT_NAME}"
    
    log_info "Federated credential details:"
    log_info "  Name: $FEDERATED_CREDENTIAL_NAME"
    log_info "  Subject: $subject"
    log_info "  Issuer: $oidc_issuer_url"
    log_info "  Audiences: api://AzureADTokenExchange"
    
    # Check if federated credential already exists
    local existing_credential
    existing_credential=$(az identity federated-credential list \
        --identity-name "$IDENTITY_NAME" \
        --resource-group "$RESOURCE_GROUP" \
        --query "[?name=='$FEDERATED_CREDENTIAL_NAME'].name" \
        -o tsv 2>/dev/null)
    
    if [[ -n "$existing_credential" ]]; then
        log_info "✅ Federated credential '$FEDERATED_CREDENTIAL_NAME' already exists"
        
        # Verify the existing credential has correct settings
        local existing_subject
        existing_subject=$(az identity federated-credential show \
            --identity-name "$IDENTITY_NAME" \
            --resource-group "$RESOURCE_GROUP" \
            --name "$FEDERATED_CREDENTIAL_NAME" \
            --query "subject" -o tsv 2>/dev/null)
            
        if [[ "$existing_subject" != "$subject" ]]; then
            log_warn "Existing federated credential has different subject: $existing_subject"
            log_info "Expected subject: $subject"
            log_info "Updating federated credential..."
            
            # Delete and recreate with correct settings
            az identity federated-credential delete \
                --identity-name "$IDENTITY_NAME" \
                --resource-group "$RESOURCE_GROUP" \
                --name "$FEDERATED_CREDENTIAL_NAME" \
                --yes \
                --output none
                
            # Create new one (fall through to creation logic below)
        else
            return 0
        fi
    fi
    
    log_info "Creating federated credential..."
    az identity federated-credential create \
        --identity-name "$IDENTITY_NAME" \
        --resource-group "$RESOURCE_GROUP" \
        --name "$FEDERATED_CREDENTIAL_NAME" \
        --issuer "$oidc_issuer_url" \
        --subject "$subject" \
        --audiences "api://AzureADTokenExchange" \
        --output none
    
    log_info "✅ Federated credential created successfully"
}

# Detect a running pod's service account (by app label) and optionally adjust SERVICE_ACCOUNT_NAME
detect_running_service_account() {
    if [[ -z "${APP_LABEL:-}" ]]; then
        log_debug "APP_LABEL not set; skipping running pod service account detection"
        return 0
    fi
    if ! command -v kubectl >/dev/null 2>&1; then
        log_warn "kubectl not available; cannot auto-detect running pod service account"
        return 0
    fi
    log_step "Detecting running pod service account for app label 'app=${APP_LABEL}' in namespace '${SERVICE_ACCOUNT_NAMESPACE}'..."
    local pod_name
    pod_name=$(kubectl -n "${SERVICE_ACCOUNT_NAMESPACE}" get pods -l "app=${APP_LABEL}" -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || true)
    if [[ -z "$pod_name" ]]; then
        log_warn "No pod found with label app=${APP_LABEL} (namespace ${SERVICE_ACCOUNT_NAMESPACE}); skipping detection"
        return 0
    fi
    local detected_sa
    detected_sa=$(kubectl -n "${SERVICE_ACCOUNT_NAMESPACE}" get pod "$pod_name" -o jsonpath='{.spec.serviceAccountName}' 2>/dev/null || true)
    if [[ -z "$detected_sa" ]]; then
        log_warn "Could not determine service account for pod $pod_name"
        return 0
    fi
    if [[ "$detected_sa" != "$SERVICE_ACCOUNT_NAME" ]]; then
        log_warn "Configured SERVICE_ACCOUNT_NAME='$SERVICE_ACCOUNT_NAME' differs from running pod service account '$detected_sa' (pod: $pod_name)"
        if [[ "${AUTO_ADJUST_SERVICE_ACCOUNT:-}" == "true" ]]; then
            SERVICE_ACCOUNT_NAME="$detected_sa"
            log_info "Adjusted SERVICE_ACCOUNT_NAME to detected value: $SERVICE_ACCOUNT_NAME"
        else
            log_info "Set AUTO_ADJUST_SERVICE_ACCOUNT=true to automatically switch to '$detected_sa'"
        fi
    else
        log_info "Detected running pod $pod_name using expected service account '$SERVICE_ACCOUNT_NAME'"
    fi
}

# Ensure the Kubernetes service account has the correct workload identity annotation
reconcile_service_account_annotation() {
    if ! command -v kubectl >/dev/null 2>&1; then
        log_warn "kubectl not available; skipping service account annotation reconciliation"
        return 0
    fi
    log_step "Reconciling service account annotation azure.workload.identity/client-id for ${SERVICE_ACCOUNT_NAMESPACE}/${SERVICE_ACCOUNT_NAME}..."
    if ! kubectl -n "${SERVICE_ACCOUNT_NAMESPACE}" get sa "${SERVICE_ACCOUNT_NAME}" >/dev/null 2>&1; then
        log_warn "Service account ${SERVICE_ACCOUNT_NAMESPACE}/${SERVICE_ACCOUNT_NAME} not found in cluster"
        return 0
    fi
    local current_annotation
    current_annotation=$(kubectl -n "${SERVICE_ACCOUNT_NAMESPACE}" get sa "${SERVICE_ACCOUNT_NAME}" -o jsonpath='{.metadata.annotations.azure\.workload\.identity/client-id}' 2>/dev/null || true)
    if [[ -z "$current_annotation" ]]; then
        log_warn "Service account missing azure.workload.identity/client-id annotation"
        if [[ "${AUTO_FIX:-}" == "true" ]]; then
            log_info "Patching annotation with client-id: $1"
            if kubectl -n "${SERVICE_ACCOUNT_NAMESPACE}" annotate sa "${SERVICE_ACCOUNT_NAME}" "azure.workload.identity/client-id=$1" --overwrite >/dev/null 2>&1; then
                log_info "✅ Annotation added"
            else
                log_warn "Failed to patch annotation"
            fi
        else
            log_info "Run: kubectl -n ${SERVICE_ACCOUNT_NAMESPACE} annotate sa ${SERVICE_ACCOUNT_NAME} azure.workload.identity/client-id=$1 --overwrite"
        fi
    elif [[ "$current_annotation" != "$1" ]]; then
        log_warn "Service account annotation client-id mismatch. Current: $current_annotation Expected: $1"
        if [[ "${AUTO_FIX:-}" == "true" ]]; then
            log_info "Overwriting annotation with expected client-id"
            if kubectl -n "${SERVICE_ACCOUNT_NAMESPACE}" annotate sa "${SERVICE_ACCOUNT_NAME}" "azure.workload.identity/client-id=$1" --overwrite >/dev/null 2>&1; then
                log_info "✅ Annotation updated"
            else
                log_warn "Failed to update annotation"
            fi
        else
            log_info "Set AUTO_FIX=true to auto-correct or run manually: kubectl -n ${SERVICE_ACCOUNT_NAMESPACE} annotate sa ${SERVICE_ACCOUNT_NAME} azure.workload.identity/client-id=$1 --overwrite"
        fi
    else
        log_info "✅ Service account annotation already correct"
    fi
}

# Prepare and export environment variables needed by 'make deploy' / 'make helm-deploy'
prepare_deploy_environment() {
    log_step "Preparing deployment environment variables..."
    local client_id="$1"
    local subscription_id="$2"
    local tenant_id
    tenant_id=$(az account show --query tenantId -o tsv 2>/dev/null || echo "")

    # Resolve OpenAI endpoint if not set
    if [[ -z "${AZURE_OPENAI_ENDPOINT:-}" ]]; then
        local resolved_endpoint
        resolved_endpoint=$(az cognitiveservices account show -n "$LLM_NAME" -g "$LLM_RESOURCE_GROUP" --query properties.endpoint -o tsv 2>/dev/null || true)
        if [[ -n "$resolved_endpoint" ]]; then
            AZURE_OPENAI_ENDPOINT="$resolved_endpoint"
            log_info "Resolved AZURE_OPENAI_ENDPOINT=$AZURE_OPENAI_ENDPOINT"
        else
            log_warn "Could not auto-resolve AZURE_OPENAI_ENDPOINT (set manually)"
        fi
    fi

    # Default values / placeholders
    AKS_RESOURCE_GROUP="${AKS_RESOURCE_GROUP:-$RESOURCE_GROUP}"
    AZURE_OPENAI_API_VERSION="${AZURE_OPENAI_API_VERSION:-2025-03-01-preview}"
    AZURE_OPENAI_DEPLOYMENT="${AZURE_OPENAI_DEPLOYMENT:-<set-your-model-deployment>}"
    AZURE_OPENAI_SCOPE="${AZURE_OPENAI_SCOPE:-api://$client_id/.default}"

    # Export for child processes (e.g., make helm-deploy invoked from this script)
    export AZURE_CLIENT_ID="$client_id" \
           AZURE_TENANT_ID="$tenant_id" \
           AZURE_SUBSCRIPTION_ID="$subscription_id" \
           AZURE_OPENAI_ENDPOINT \
           AZURE_OPENAI_DEPLOYMENT \
           AZURE_OPENAI_API_VERSION \
           AZURE_OPENAI_SCOPE \
           AKS_CLUSTER_NAME \
           AKS_RESOURCE_GROUP \
           LOAD_BALANCER_IP

    log_info "Exported deployment environment variables to current process"

    # Optionally write an env file for sourcing later
    if [[ "${WRITE_ENV_FILE:-true}" == "true" ]]; then
        local env_file="${ENV_OUTPUT_FILE:-.cna-deploy.env}"
        cat > "$env_file" <<EOF
# Auto-generated by setup-managed-identity-federated.sh
AZURE_CLIENT_ID=$AZURE_CLIENT_ID
AZURE_TENANT_ID=$AZURE_TENANT_ID
AZURE_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID
AZURE_OPENAI_ENDPOINT=${AZURE_OPENAI_ENDPOINT:-}
AZURE_OPENAI_DEPLOYMENT=$AZURE_OPENAI_DEPLOYMENT
AZURE_OPENAI_API_VERSION=$AZURE_OPENAI_API_VERSION
AZURE_OPENAI_SCOPE=$AZURE_OPENAI_SCOPE
AKS_CLUSTER_NAME=$AKS_CLUSTER_NAME
AKS_RESOURCE_GROUP=$AKS_RESOURCE_GROUP
LOAD_BALANCER_IP=${LOAD_BALANCER_IP:-}
EOF
        log_info "Wrote environment file: $env_file (source it with: set -a; . $env_file; set +a)"
    fi
}

# Persist resolved values into config.env so Make targets get them directly
update_env_files() {
    log_step "Persisting values to config.env..."
    local tenant_id
    tenant_id=$(az account show --query tenantId -o tsv 2>/dev/null || echo "")

    local cfg_file="config.env"

    # Helper to upsert key=value in a file (creates file if missing)
    upsert_kv() {
        local file="$1"; shift
        local key="$1"; shift
        local val="$1"; shift
        [[ -z "$key" ]] && return 0
        [[ ! -f "$file" ]] && touch "$file"
        if grep -E "^${key}=" "$file" >/dev/null 2>&1; then
            # Replace line
            sed -i.bak "/^${key}=/c\\${key}=${val}" "$file" || true
        else
            echo "${key}=${val}" >> "$file"
        fi
    }

    # Backup originals once per run
    [[ -f "$cfg_file" && ! -f "${cfg_file}.bak_run" ]] && cp "$cfg_file" "${cfg_file}.bak_run"

    # Upsert config values
    upsert_kv "$cfg_file" AZURE_CLIENT_ID "$AZURE_CLIENT_ID"
    upsert_kv "$cfg_file" AZURE_TENANT_ID "$tenant_id"
    upsert_kv "$cfg_file" AZURE_SUBSCRIPTION_ID "$AZURE_SUBSCRIPTION_ID"
    upsert_kv "$cfg_file" AKS_CLUSTER_NAME "$AKS_CLUSTER_NAME"
    upsert_kv "$cfg_file" AKS_RESOURCE_GROUP "$AKS_RESOURCE_GROUP"
    upsert_kv "$cfg_file" AZURE_OPENAI_DEPLOYMENT "$AZURE_OPENAI_DEPLOYMENT"
    upsert_kv "$cfg_file" AZURE_OPENAI_API_VERSION "$AZURE_OPENAI_API_VERSION"
    upsert_kv "$cfg_file" AZURE_OPENAI_ENDPOINT "${AZURE_OPENAI_ENDPOINT:-}"
    # Optional scope
    [[ -n "$AZURE_OPENAI_SCOPE" ]] && upsert_kv "$cfg_file" AZURE_OPENAI_SCOPE "$AZURE_OPENAI_SCOPE"

    log_info "Updated $cfg_file with current identity & OpenAI settings"
}

# Retry helper with exponential backoff for role assignment operations
# Usage: retry_with_backoff <max_attempts> <command...>
# Retries up to max_attempts times with exponential backoff (2^attempt seconds, capped at 60s)
retry_with_backoff() {
    local max_attempts="$1"
    shift
    local attempt=1
    local delay=8  # initial delay in seconds

    while [[ $attempt -le $max_attempts ]]; do
        if "$@" 2>/dev/null; then
            return 0
        fi
        
        if [[ $attempt -eq $max_attempts ]]; then
            log_error "Command failed after $max_attempts attempts: $*"
            return 1
        fi
        
        log_warn "Attempt $attempt/$max_attempts failed, retrying in ${delay}s... (waiting for service principal propagation)"
        sleep "$delay"
        
        # Exponential backoff: 2, 4, 8, 16, 32 (capped at 60)
        delay=$((delay * 2))
        [[ $delay -gt 60 ]] && delay=60
        attempt=$((attempt + 1))
    done
    
    return 1
}

# Function to create RBAC role assignments
create_rbac_assignments() {
    local client_id="$1"
    local principal_id="$2"
    
    log_step "Creating RBAC role assignments..."
    
    # Wait a moment for the managed identity service principal to propagate in Azure AD
    log_info "Waiting for managed identity service principal to propagate..."
    sleep 5
    
    # Define scopes and roles
    local aks_scope="/subscriptions/$SUBSCRIPTION_ID/resourcegroups/$RESOURCE_GROUP/providers/Microsoft.ContainerService/managedClusters/$AKS_CLUSTER_NAME"
    local rg_scope="/subscriptions/$SUBSCRIPTION_ID/resourcegroups/$RESOURCE_GROUP"
    
    # AKS Contributor Role assignment (for AKS cluster management operations like az aks show)
    log_info "Assigning 'Azure Kubernetes Service Contributor Role' to AKS cluster..."
    if az role assignment list --assignee "$principal_id" --scope "$aks_scope" --role "Azure Kubernetes Service Contributor Role" --query "[0].id" -o tsv | grep -q .; then
        log_info "✅ AKS Contributor Role already assigned"
    else
        if retry_with_backoff 5 az role assignment create \
            --assignee "$principal_id" \
            --role "Azure Kubernetes Service Contributor Role" \
            --scope "$aks_scope" \
            --description "Allows managed identity to read and write AKS cluster information for MCP AKS operations" \
            --output none; then
            log_info "✅ AKS Contributor Role assigned successfully"
        else
            log_error "Failed to assign AKS Contributor Role after retries"
            return 1
        fi
    fi
    
    # AKS Cluster User Role assignment (for kubeconfig access)
    log_info "Assigning 'Azure Kubernetes Service Cluster User Role' to AKS cluster..."
    if az role assignment list --assignee "$principal_id" --scope "$aks_scope" --role "Azure Kubernetes Service Cluster User Role" --query "[0].id" -o tsv | grep -q .; then
        log_info "✅ AKS Cluster User Role already assigned"
    else
        if retry_with_backoff 5 az role assignment create \
            --assignee "$principal_id" \
            --role "Azure Kubernetes Service Cluster User Role" \
            --scope "$aks_scope" \
            --description "Allows managed identity to access AKS cluster for ${SERVICE_ACCOUNT_NAME} service account" \
            --output none; then
            log_info "✅ AKS Cluster User Role assigned successfully"
        else
            log_error "Failed to assign AKS Cluster User Role after retries"
            return 1
        fi
    fi
    
    # Reader role assignment at resource group level (for resource discovery)
    log_info "Assigning 'Reader' role to resource group..."
    if az role assignment list --assignee "$principal_id" --scope "$rg_scope" --role "Reader" --query "[0].id" -o tsv | grep -q .; then
        log_info "✅ Reader role already assigned"
    else
        if retry_with_backoff 5 az role assignment create \
            --assignee "$principal_id" \
            --role "Reader" \
            --scope "$rg_scope" \
            --description "Allows managed identity to read resources for ${SERVICE_ACCOUNT_NAME} service account" \
            --output none; then
            log_info "✅ Reader role assigned successfully"
        else
            log_error "Failed to assign Reader role after retries"
            return 1
        fi
    fi
    
    # Azure OpenAI (Cognitive Services) role assignment (Cognitive Services OpenAI User)
    log_info "Assigning 'Cognitive Services OpenAI User' role on Azure OpenAI account '$LLM_NAME' (RG: $LLM_RESOURCE_GROUP)..."
    local openai_scope=""
    if openai_scope=$(az cognitiveservices account show -n "$LLM_NAME" -g "$LLM_RESOURCE_GROUP" --query id -o tsv 2>/dev/null); then
        if az role assignment list --assignee "$principal_id" --scope "$openai_scope" --role "Cognitive Services OpenAI User" --query "[0].id" -o tsv | grep -q .; then
            log_info "✅ Cognitive Services OpenAI User role already assigned"
        else
            # Attempt direct create with retry; fallback to role definition id if name lookup fails
            if retry_with_backoff 5 az role assignment create \
                --assignee "$principal_id" \
                --role "Cognitive Services OpenAI User" \
                --scope "$openai_scope" \
                --description "Allows managed identity to invoke Azure OpenAI (inference)" \
                --output none; then
                log_info "✅ Cognitive Services OpenAI User role assigned successfully"
            else
                # Resolve role definition id explicitly as fallback
                local role_id
                role_id=$(az role definition list --name "Cognitive Services OpenAI User" --query "[0].id" -o tsv 2>/dev/null || echo "")
                if [[ -n "$role_id" ]]; then
                    log_info "Retrying role assignment with role definition id $role_id"
                    if retry_with_backoff 5 az role assignment create \
                        --assignee "$principal_id" \
                        --role "$role_id" \
                        --scope "$openai_scope" \
                        --description "Allows managed identity to invoke Azure OpenAI (inference)" \
                        --output none; then
                        log_info "✅ Cognitive Services OpenAI User role assigned successfully"
                    else
                        log_warn "Could not assign OpenAI role after retries (check permissions)"
                    fi
                else
                    log_warn "Could not resolve role id for 'Cognitive Services OpenAI User'"
                fi
            fi
        fi
    else
        log_warn "Azure OpenAI account '$LLM_NAME' not found in resource group '$LLM_RESOURCE_GROUP'; skipping OpenAI role assignment"
    fi

    # Wait for role assignments to propagate (AKS + OpenAI)
    log_info "Waiting for role assignments to propagate..."
    sleep 10
}

# Function to verify the setup
verify_setup() {
    local client_id="$1"
    local principal_id="$2"
    
    log_step "Verifying setup..."
    
    # Verify managed identity exists
    if az identity show --name "$IDENTITY_NAME" --resource-group "$RESOURCE_GROUP" >/dev/null 2>&1; then
        log_info "✅ Managed identity exists"
    else
        log_error "❌ Managed identity not found"
        return 1
    fi
    
    # Verify federated credential exists
    if az identity federated-credential show --identity-name "$IDENTITY_NAME" --resource-group "$RESOURCE_GROUP" --name "$FEDERATED_CREDENTIAL_NAME" >/dev/null 2>&1; then
        log_info "✅ Federated credential exists"
    else
        log_error "❌ Federated credential not found"
        return 1
    fi
    
    # Verify role assignments
    local aks_scope="/subscriptions/$SUBSCRIPTION_ID/resourcegroups/$RESOURCE_GROUP/providers/Microsoft.ContainerService/managedClusters/$AKS_CLUSTER_NAME"
    local cluster_contributor_role_count
    cluster_contributor_role_count=$(az role assignment list --assignee "$principal_id" --scope "$aks_scope" --role "Azure Kubernetes Service Contributor Role" --query "length(@)" -o tsv)
    local cluster_user_role_count
    cluster_user_role_count=$(az role assignment list --assignee "$principal_id" --scope "$aks_scope" --role "Azure Kubernetes Service Cluster User Role" --query "length(@)" -o tsv)
    local reader_role_count
    reader_role_count=$(az role assignment list --assignee "$principal_id" --scope "/subscriptions/$SUBSCRIPTION_ID/resourcegroups/$RESOURCE_GROUP" --role "Reader" --query "length(@)" -o tsv)
    
    if [[ "$cluster_contributor_role_count" -gt 0 ]]; then
        log_info "✅ AKS Contributor Role assignment verified"
    else
        log_error "❌ AKS Contributor Role assignment not found"
    fi
    
    if [[ "$cluster_user_role_count" -gt 0 ]]; then
        log_info "✅ AKS Cluster User Role assignment verified"
    else
        log_error "❌ AKS Cluster User Role assignment not found"
    fi
    
    if [[ "$reader_role_count" -gt 0 ]]; then
        log_info "✅ Reader role assignment verified"
    else
        log_error "❌ Reader role assignment not found"
    fi
    
    return 0
}

# Function to display setup summary
display_summary() {
    local client_id="$1"
    local principal_id="$2"
    local resource_id="$3"
    
    # Attempt to resolve Azure OpenAI endpoint if not already set
    if [[ -z "${AZURE_OPENAI_ENDPOINT:-}" ]]; then
        local resolved_openai_endpoint
        resolved_openai_endpoint=$(az cognitiveservices account show -n "$LLM_NAME" -g "$LLM_RESOURCE_GROUP" --query properties.endpoint -o tsv 2>/dev/null || true)
        if [[ -n "$resolved_openai_endpoint" ]]; then
            AZURE_OPENAI_ENDPOINT="$resolved_openai_endpoint"
            log_info "Resolved Azure OpenAI endpoint: $AZURE_OPENAI_ENDPOINT"
        fi
    fi
    # Default AKS_RESOURCE_GROUP to RESOURCE_GROUP if not provided
    if [[ -z "${AKS_RESOURCE_GROUP:-}" ]]; then
        AKS_RESOURCE_GROUP="$RESOURCE_GROUP"
    fi
    
    echo
    echo -e "${BLUE}Azure OpenAI Resource:${NC}"
    echo "  LLM_NAME: $LLM_NAME"
    if [[ "$LLM_RESOURCE_GROUP" == "$RESOURCE_GROUP" ]]; then
        echo "  LLM_RESOURCE_GROUP: $LLM_RESOURCE_GROUP (same as RESOURCE_GROUP)"
    else
        echo "  LLM_RESOURCE_GROUP: $LLM_RESOURCE_GROUP (separate resource group)"
    fi
    echo
    echo -e "${GREEN}==================================================================================${NC}"
    echo -e "${GREEN}                    Azure Managed Identity Setup Complete${NC}"
    echo -e "${GREEN}==================================================================================${NC}"
    echo
    echo -e "${BLUE}Identity Details:${NC}"
    echo "  Name: $IDENTITY_NAME"
    echo "  Client ID: $client_id"
    echo "  Principal ID: $principal_id"
    echo "  Resource ID: $resource_id"
    echo
    echo -e "${BLUE}Kubernetes Integration:${NC}"
    echo "  Service Account: $SERVICE_ACCOUNT_NAMESPACE/$SERVICE_ACCOUNT_NAME"
    echo "  Federated Credential: $FEDERATED_CREDENTIAL_NAME"
    echo
    echo -e "${BLUE}Environment Configuration:${NC}"
    echo "  Resource Group: $RESOURCE_GROUP"
    echo "  AKS Cluster: $AKS_CLUSTER_NAME"
    echo "  Subscription: $SUBSCRIPTION_ID"
    echo "  Location: $LOCATION"
    echo
    echo -e "${CYAN}Environment Variables for Applications:${NC}"
    echo "export AZURE_CLIENT_ID=\"$client_id\""
    echo "export AZURE_SUBSCRIPTION_ID=\"$SUBSCRIPTION_ID\""
    echo "export AZURE_TENANT_ID=\"$(az account show --query tenantId -o tsv)\""
    echo
    echo -e "${CYAN}Environment Variables for make deploy:${NC}"
    echo "# Required (deployment)"
    echo "export AZURE_CLIENT_ID=\"$client_id\""
    echo "export AZURE_TENANT_ID=\"$(az account show --query tenantId -o tsv)\""
    echo "export AZURE_SUBSCRIPTION_ID=\"$SUBSCRIPTION_ID\""
    echo "# Required (runtime)"
    echo "export AZURE_OPENAI_ENDPOINT=\"${AZURE_OPENAI_ENDPOINT:-<set-your-openai-endpoint>}\""
    echo "export AZURE_OPENAI_DEPLOYMENT=\"${AZURE_OPENAI_DEPLOYMENT:-<your-model-deployment-name>}\""
    echo "export AZURE_OPENAI_API_VERSION=\"${AZURE_OPENAI_API_VERSION:-2025-03-01-preview}\""
    echo "# Cluster context for Helm chart"
    echo "export AKS_CLUSTER_NAME=\"$AKS_CLUSTER_NAME\""
    echo "export AKS_RESOURCE_GROUP=\"$AKS_RESOURCE_GROUP\""
    echo "# Optional"
    echo "export AZURE_OPENAI_SCOPE=\"${AZURE_OPENAI_SCOPE:-api://$client_id/.default}\"  # adjust if using custom scope"
    echo "export LOAD_BALANCER_IP=\"${LOAD_BALANCER_IP:-}\"  # optional static LB IP"
    echo
    echo -e "${CYAN}Quick apply (copy/paste):${NC}"
    echo "# Copy the lines above or source this script output in your shell before: make deploy"
    echo
    echo -e "${CYAN}Service Account Annotation:${NC}"
    echo "azure.workload.identity/client-id: $client_id"
    echo
    echo -e "${YELLOW}Next Steps:${NC}"
    echo "1. Ensure your pod spec includes the service account:"
    echo "   serviceAccountName: $SERVICE_ACCOUNT_NAME"
    echo "2. Verify workload identity is enabled in your deployment"
    echo "3. Test authentication from within a pod using Azure SDK"
    echo
    echo -e "${GREEN}Setup completed successfully! 🎉${NC}"
    echo
}

# Function to handle cleanup on script exit
cleanup_on_exit() {
    local exit_code=$?
    if [[ $exit_code -ne 0 ]]; then
        log_error "Script failed with exit code $exit_code"
        echo
        echo -e "${YELLOW}Troubleshooting Tips:${NC}"
        echo "1. Ensure you have sufficient Azure permissions (Contributor + User Access Administrator)"
        echo "2. Verify AKS cluster exists and is accessible"
        echo "3. Check Azure CLI is logged in: az account show"
        echo "4. Run with DEBUG=true for more verbose output"
        echo
    fi
}

# Main execution function
main() {
    # Set up error handling
    trap cleanup_on_exit EXIT
    
    echo -e "${GREEN}==================================================================================${NC}"
    echo -e "${GREEN}    Azure Managed Identity with Federated Credentials Setup${NC}"
    echo -e "${GREEN}    Target: Kubernetes Service Account '$SERVICE_ACCOUNT_NAME' in '$SERVICE_ACCOUNT_NAMESPACE'${NC}"
    echo -e "${GREEN}==================================================================================${NC}"
    echo
    
    # Display configuration
    echo -e "${BLUE}Configuration:${NC}"
    echo "  Identity Name: $IDENTITY_NAME"
    echo "  Resource Group: $RESOURCE_GROUP"
    echo "  AKS Cluster: $AKS_CLUSTER_NAME"
    echo "  Location: $LOCATION"
    echo "  Service Account: $SERVICE_ACCOUNT_NAMESPACE/$SERVICE_ACCOUNT_NAME"
    echo "  Subscription: $SUBSCRIPTION_ID"
    echo
    
    # Execute setup steps
    check_azure_prerequisites
    local oidc_issuer_url
    oidc_issuer_url=$(check_aks_workload_identity)
    create_managed_identity
    
    # Get identity details
    local identity_details
    identity_details=$(get_identity_details)
    local client_id
    client_id=$(echo "$identity_details" | jq -r '.clientId')
    local principal_id
    principal_id=$(echo "$identity_details" | jq -r '.principalId')
    local resource_id
    resource_id=$(echo "$identity_details" | jq -r '.resourceId')
    
    log_info "Using managed identity: $client_id (Principal: $principal_id)"
    
    # Setup federated credentials and RBAC
    # Optionally detect actual running pod service account (if APP_LABEL provided) before creating federated credential
    detect_running_service_account
    # Ensure annotation matches client id
    reconcile_service_account_annotation "$client_id"
    # Export and persist env vars needed for make deploy
    prepare_deploy_environment "$client_id" "$SUBSCRIPTION_ID"
    update_env_files
    create_federated_credential "$oidc_issuer_url" "$client_id"
    create_rbac_assignments "$client_id" "$principal_id"
    
    # Verify and display results
    if verify_setup "$client_id" "$principal_id"; then
        display_summary "$client_id" "$principal_id" "$resource_id"
    else
        log_error "Setup verification failed. Please check the configuration manually."
        exit 1
    fi
}

# Script entry point - only run main if script is executed directly
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
```

**3. Configure and run the script:**

Copy the environment template file, fill in your values, source it, and then run the setup script:

```azurecli-interactive
# Copy the environment template
cp deployment/setup-managed-identity.env.template deployment/setup-managed-identity.env

# Edit the file with your actual values:
#   RESOURCE_GROUP    — your resource group name (from Step 1)
#   AKS_CLUSTER_NAME  — your AKS cluster name (from Step 2)
#   LOCATION          — your Azure region (from Step 1)
#   LLM_NAME          — your Azure OpenAI resource name (from Step 3, same as OPENAI_SERVICE_NAME)

# Source the environment file to load all variables into your shell
source deployment/setup-managed-identity.env

# Run the managed identity and federated credential setup script
./deployment/setup-managed-identity-federated.sh
```

> [!NOTE]
> Use `source` (instead of `./`) for the environment file so the exported environment variables remain available in your current shell for subsequent steps. The script creates a managed identity, configures federated credentials, assigns RBAC roles, and generates a `.cna-deploy.env` file with all resolved values.

---

### Step 7: Create an App Registration for Entra ID authentication

Container Networking Agent is a web application that requires user sign-in. In this step, you create a Microsoft Entra ID App Registration that enables OAuth2/OIDC-based authentication. The App Registration issues tokens that the agent validates when users access the chat interface through their browser.

Container Networking Agent supports two methods for user sign-in:

| Method | Use case |
|--------|----------|
| Simple username login | Development and testing environments only. |
| Microsoft Entra ID (MSAL) | Production environments with OAuth2/OIDC through an App Registration (recommended). |

> [!NOTE]
> For development or testing environments where you plan to use simple username login, you can skip this step. Microsoft Entra ID authentication is recommended for production deployments.

#### [Azure CLI](#tab/step7-cli)

To enable Microsoft Entra ID user authentication in production, create an App Registration using the [`az ad app create`](/cli/azure/ad/app#az-ad-app-create) command.

```azurecli-interactive
export APP_DISPLAY_NAME="container-networking-agent-oauth2-user-auth-<your-alias>"
export APP_REDIRECT_URI="http://localhost:8000/auth/callback"  # For local/port-forward testing

# Optional: Required by some tenants - set your Service Tree ID if needed
# export SERVICE_MANAGEMENT_REFERENCE="<your-service-tree-id>"

# Create app registration with ID token and access token issuance
# NOTE: Ensure you have Owner or Contributor permissions on the app registration
#       you plan to use so you can add federated credentials later.
# If your tenant requires SERVICE_MANAGEMENT_REFERENCE, add: --service-management-reference $SERVICE_MANAGEMENT_REFERENCE
export APP_CLIENT_ID=$(az ad app create \
    --display-name $APP_DISPLAY_NAME \
    --web-redirect-uris $APP_REDIRECT_URI \
    --sign-in-audience AzureADMyOrg \
    --enable-id-token-issuance true \
    --enable-access-token-issuance true \
    --query "appId" -o tsv)

echo "App Registration Client ID (ENTRA_CLIENT_ID): $APP_CLIENT_ID "

# Get tenant ID
export TENANT_ID=$(az account show --query tenantId -o tsv)
echo "Tenant ID: $TENANT_ID"
```

Add the required Microsoft Graph delegated permissions (`openid`, `profile`, `User.Read`, `offline_access`) using the [`az ad app permission add`](/cli/azure/ad/app/permission#az-ad-app-permission-add) command.

```azurecli-interactive
az ad app permission add \
    --id $APP_CLIENT_ID \
    --api 00000003-0000-0000-c000-000000000000 \
    --api-permissions \
        e1fe6dd8-ba31-4d61-89e7-88639da4683d=Scope \
        14dad69e-099b-42c9-810b-d002981feec1=Scope \
        37f7f235-527c-4136-accd-4a02d197296e=Scope \
        7427e0e9-2fba-42fe-b0c0-848c9e6a8182=Scope

echo "App Registration Client ID: $APP_CLIENT_ID"
echo "Tenant ID: $TENANT_ID"
```

#### [Script](#tab/step7-script)

Follow these three steps to create and run the setup script.

**1. Create the script file:**

```azurecli-interactive
touch setup-azure-app-registration.sh && chmod +x setup-azure-app-registration.sh
```

**2. Copy the following content into `setup-azure-app-registration.sh`:**

Open the file in your preferred editor and paste the entire script below. The script reads from environment variables. At minimum, ensure `LOCATION`, `RESOURCE_GROUP`, `AKS_CLUSTER_NAME`, and `SERVICE_MANAGEMENT_REFERENCE` are exported. Optionally set `APP_REDIRECT_URI` (defaults to `http://localhost:8000/auth/callback`).

```bash
#!/bin/bash

# Draft script scaffold for automating Azure App Registration creation and configuration
# for the Container Networking Agent authentication flow. The script will gradually gain functions that:
#   - validate Azure CLI context and permissions
#   - create or reuse the Entra ID application registration
#   - configure redirect URIs, secrets, and permissions
#   - emit environment variables for downstream services

set -e

# Configuration - modify these variables as needed or set as environment variables
IDENTITY_NAME="${IDENTITY_NAME:-container-networking-agent-reader-identity}"
SERVICE_ACCOUNT_NAMESPACE="${SERVICE_ACCOUNT_NAMESPACE:-kube-system}"
SERVICE_ACCOUNT_NAME="${SERVICE_ACCOUNT_NAME:-container-networking-agent-reader}"
FEDERATED_CREDENTIAL_NAME="${FEDERATED_CREDENTIAL_NAME:-container-networking-agent-k8s-fed-cred}"
APP_DISPLAY_NAME="${APP_DISPLAY_NAME:-container-networking-agent-oauth2-user-auth}"
APP_REDIRECT_URI="${APP_REDIRECT_URI:-http://localhost:8000/auth/callback}"
APP_LOGOUT_URL="${APP_LOGOUT_URL:-}"
APP_GRAPH_SCOPES="${APP_GRAPH_SCOPES:-}"
AKS_SERVER_APP_ID="${AKS_SERVER_APP_ID:-6dae42f8-4368-4678-94ff-3960e28e3630}"
AKS_SERVER_SCOPES="${AKS_SERVER_SCOPES:-}"
SERVICE_MANAGEMENT_REFERENCE="${SERVICE_MANAGEMENT_REFERENCE:-}"

# Validate all required variables
: "${IDENTITY_NAME:?ERROR: IDENTITY_NAME is required}"
: "${LOCATION:?ERROR: LOCATION is required}"
: "${RESOURCE_GROUP:?ERROR: RESOURCE_GROUP is required}"
: "${AKS_CLUSTER_NAME:?ERROR: AKS_CLUSTER_NAME is required}"
: "${SERVICE_ACCOUNT_NAMESPACE:?ERROR: SERVICE_ACCOUNT_NAMESPACE is required}"
: "${SERVICE_ACCOUNT_NAME:?ERROR: SERVICE_ACCOUNT_NAME is required}"
: "${SERVICE_MANAGEMENT_REFERENCE:?ERROR: SERVICE_MANAGEMENT_REFERENCE is required}"

# Colors for output
readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[1;33m'
readonly BLUE='\033[0;34m'
readonly NC='\033[0m'

# Default Microsoft Graph delegated scopes to request for the web application.
GRAPH_DEFAULT_SCOPES=("openid" "profile" "offline_access" "User.Read")

# Default AKS server delegated scopes to request for the application.
AKS_SERVER_DEFAULT_SCOPES=("user.read")

# Known Microsoft Graph delegated scope IDs for quick lookup. Values copied from
# https://learn.microsoft.com/graph/permissions-reference and Azure CLI examples.
declare -A GRAPH_SCOPE_ID_MAP=(
    [openid]="37f7f235-527c-4136-accd-4a02d197296e"
    [profile]="14dad69e-099b-42c9-810b-d002981feec1"
    [offline_access]="7427e0e9-2fba-42fe-b0c0-848c9e6a8182"
    [User.Read]="e1fe6dd8-ba31-4d61-89e7-88639da4683d"
)

declare -A AKS_SCOPE_ID_MAP=(
    [user.read]="34a47c2f-cd0d-47b4-a93c-2c41130c671c"
)

# Resolve a Microsoft Graph delegated permission value (e.g. "User.Read") into
# the GUID required by `az ad app permission add`. Falls back to a Graph lookup
# if the permission is not pre-populated above.
resolve_graph_scope_id() {
    local scope="$1"

    if [[ -z "$scope" ]]; then
        return 1
    fi

    if [[ -n "${GRAPH_SCOPE_ID_MAP[$scope]:-}" ]]; then
        echo "${GRAPH_SCOPE_ID_MAP[$scope]}"
        return 0
    fi

    local resolved
    resolved=$(az ad sp show \
        --id 00000003-0000-0000-c000-000000000000 \
        --query "oauth2Permissions[?value=='$scope'].id | [0]" \
        -o tsv 2>/dev/null || true)

    if [[ -n "$resolved" ]]; then
        GRAPH_SCOPE_ID_MAP[$scope]="$resolved"
        echo "$resolved"
        return 0
    fi

    return 1
}

resolve_aks_scope_id() {
    local scope="$1"

    if [[ -z "$scope" ]]; then
        return 1
    fi

    if [[ -n "${AKS_SCOPE_ID_MAP[$scope]:-}" ]]; then
        echo "${AKS_SCOPE_ID_MAP[$scope]}"
        return 0
    fi

    return 1
}

# Tag major steps during execution for clarity.
log_step() {
    echo -e "${BLUE}[STEP]${NC} $1" >&2
}

# Emit informational messages in green.
log_info() {
    echo -e "${GREEN}[INFO]${NC} $1" >&2
}

# Highlight warnings in yellow.
log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1" >&2
}

# Report fatal errors in red and exit the script.
log_error() {
    echo -e "${RED}[ERROR]${NC} $1" >&2
}

# Ensure a required CLI is available before proceeding.
require_command() {
    local name="$1"
    if ! command -v "$name" >/dev/null 2>&1; then
        log_error "Required command '$name' not found in PATH."
        exit 1
    fi
}

# Step 1: confirm Azure CLI context is ready for app registration work.
validate_prerequisites() {
    log_step "Step 1: Validate Azure CLI login state and permissions"

    require_command az

    if ! az account show >/dev/null 2>&1; then
        log_error "Azure CLI is not logged in. Run 'az login' (and 'az account set') before continuing."
        exit 1
    fi

    local subscription_id tenant_id account_name
    subscription_id=$(az account show --query id -o tsv 2>/dev/null || true)
    tenant_id=$(az account show --query tenantId -o tsv 2>/dev/null || true)
    account_name=$(az account show --query name -o tsv 2>/dev/null || true)

    if [[ -z "$subscription_id" || -z "$tenant_id" ]]; then
        log_error "Unable to resolve current Azure subscription or tenant."
        exit 1
    fi

    log_info "Active subscription: ${account_name:-unknown} ($subscription_id)"
    log_info "Tenant: $tenant_id"

    # Check if resource group exists
    if ! az group show --name "$RESOURCE_GROUP" >/dev/null 2>&1; then
        log_error "Resource group '$RESOURCE_GROUP' does not exist"
        exit 1
    fi

    # Check if AKS cluster exists
    if ! az aks show --resource-group "$RESOURCE_GROUP" --name "$AKS_CLUSTER_NAME" >/dev/null 2>&1; then
        log_error "AKS cluster '$AKS_CLUSTER_NAME' does not exist in resource group '$RESOURCE_GROUP'"
        exit 1
    fi
    
    log_info "✅ Azure CLI prerequisites verified"    
}

# Step 2: gather app registration configuration (names, URIs, scopes).
collect_configuration() {
    log_step "Step 2: Collect app registration configuration"

    local tenant_id
    tenant_id=$(az account show --query tenantId -o tsv 2>/dev/null || true)
    if [[ -z "$tenant_id" ]]; then
        log_error "Unable to resolve current tenant while collecting configuration."
        exit 1
    fi

    # Resolve redirect URIs, allowing optional comma-separated overrides.
    local redirect_inputs=()
    if [[ -n "${APP_REDIRECT_URIS:-}" ]]; then
        IFS=',' read -r -a redirect_inputs <<< "${APP_REDIRECT_URIS}"
    else
        redirect_inputs=("$APP_REDIRECT_URI")
    fi

    declare -g -a APP_REDIRECT_URIS_SELECTED=()
    local value trimmed
    for value in "${redirect_inputs[@]}"; do
        trimmed="$(echo "$value" | xargs)"
        if [[ -n "$trimmed" ]]; then
            APP_REDIRECT_URIS_SELECTED+=("$trimmed")
        fi
    done

    if [[ ${#APP_REDIRECT_URIS_SELECTED[@]} -eq 0 ]]; then
        log_error "At least one redirect URI must be supplied (APP_REDIRECT_URI or APP_REDIRECT_URIS)."
        exit 1
    fi

    # Determine logout URL (optional).
    declare -g APP_LOGOUT_URI_SELECTED="$(echo "$APP_LOGOUT_URL" | xargs)"

    # Resolve Microsoft Graph delegated scopes.
    declare -g -a APP_GRAPH_SCOPES_SELECTED=()
    if [[ -n "$APP_GRAPH_SCOPES" ]]; then
        IFS=',' read -r -a APP_GRAPH_SCOPES_SELECTED <<< "$APP_GRAPH_SCOPES"
        for i in "${!APP_GRAPH_SCOPES_SELECTED[@]}"; do
            APP_GRAPH_SCOPES_SELECTED[$i]="$(echo "${APP_GRAPH_SCOPES_SELECTED[$i]}" | xargs)"
        done
    else
        APP_GRAPH_SCOPES_SELECTED=("${GRAPH_DEFAULT_SCOPES[@]}")
    fi

    # Resolve AKS server delegated scopes.
    declare -g -a APP_AKS_SCOPES_SELECTED=()
    if [[ -n "$AKS_SERVER_SCOPES" ]]; then
        IFS=',' read -r -a APP_AKS_SCOPES_SELECTED <<< "$AKS_SERVER_SCOPES"
        for i in "${!APP_AKS_SCOPES_SELECTED[@]}"; do
            APP_AKS_SCOPES_SELECTED[$i]="$(echo "${APP_AKS_SCOPES_SELECTED[$i]}" | xargs | tr '[:upper:]' '[:lower:]')"
        done
    else
        APP_AKS_SCOPES_SELECTED=("${AKS_SERVER_DEFAULT_SCOPES[@]}")
    fi

    # Persist other configuration values for subsequent steps.
    declare -g APP_TENANT_ID="$tenant_id"
    declare -g APP_OBJECT_ID=""   # will be populated when app is created or discovered
    declare -g APP_CLIENT_ID=""   # will be populated alongside APP_OBJECT_ID
    declare -g APP_REGISTRATION_STATUS="pending"
    declare -g APP_REDIRECT_UPDATE_STATUS="pending"
    declare -g APP_FEDERATED_CREDENTIAL_STATUS="pending"
    declare -g APP_GRAPH_PERMISSION_STATUS="pending"
    declare -g APP_GRAPH_ADMIN_CONSENT_STATUS="not_requested"
    declare -g APP_AKS_PERMISSION_STATUS="pending"
    declare -g APP_SERVICE_MANAGEMENT_STATUS="pending"

    log_info "Collected configuration for tenant $APP_TENANT_ID"
    log_info "  Display name: $APP_DISPLAY_NAME"
    log_info "  Redirect URIs: ${APP_REDIRECT_URIS_SELECTED[*]}"
    if [[ -n "$APP_LOGOUT_URI_SELECTED" ]]; then
        log_info "  Logout URL: $APP_LOGOUT_URI_SELECTED"
    fi
    log_info "  Graph scopes: ${APP_GRAPH_SCOPES_SELECTED[*]}"
}

# Step 3: create or reuse the application registration and capture identifiers.
ensure_app_registration() {
    log_step "Step 3: Create or reuse Azure App Registration"

    require_command az

    local object_id="${APP_OBJECT_ID:-}"
    local client_id="${APP_CLIENT_ID:-}"
    local source=""

    # Allow users to provide the client/object IDs upfront.
    if [[ -n "$client_id" ]]; then
        local provided_info
        provided_info=$(az ad app show --id "$client_id" --query "{objectId:id, appId:appId, displayName:displayName}" -o tsv 2>/dev/null || true)
        if [[ -z "$provided_info" ]]; then
            log_error "APP_CLIENT_ID was supplied but no app registration was found."
            exit 1
        fi
    read -r object_id client_id APP_DISPLAY_NAME <<< "$provided_info"
    source="provided-client"
    elif [[ -n "$object_id" ]]; then
        local provided_object
        provided_object=$(az ad app show --id "$object_id" --query "{objectId:id, appId:appId, displayName:displayName}" -o tsv 2>/dev/null || true)
        if [[ -z "$provided_object" ]]; then
            log_error "APP_OBJECT_ID was supplied but no app registration was found."
            exit 1
        fi
    read -r object_id client_id APP_DISPLAY_NAME <<< "$provided_object"
    source="provided-object"
    else
        # Attempt to find an existing registration by display name.
        local match_count
        match_count=$(az ad app list --display-name "$APP_DISPLAY_NAME" --query "length(@)" -o tsv 2>/dev/null || echo 0)

        if [[ "$match_count" -gt 1 ]]; then
            log_warn "Multiple app registrations share the display name '$APP_DISPLAY_NAME'. Re-using the first result; consider setting APP_CLIENT_ID explicitly."
        fi

        if [[ "$match_count" -ge 1 ]]; then
            local existing_info
            existing_info=$(az ad app list --display-name "$APP_DISPLAY_NAME" --query "[0].{objectId:id, appId:appId, displayName:displayName}" -o tsv 2>/dev/null || true)
            if [[ -n "$existing_info" ]]; then
                read -r object_id client_id APP_DISPLAY_NAME <<< "$existing_info"
                source="existing"
            fi
        fi
    fi

    if [[ -z "$object_id" || -z "$client_id" ]]; then
        log_info "Creating new app registration '$APP_DISPLAY_NAME'."

        local create_args=(
            az ad app create
            --display-name "$APP_DISPLAY_NAME"
            --sign-in-audience AzureADMyOrg
            --enable-id-token-issuance true
            --enable-access-token-issuance true
        )

        if [[ ${#APP_REDIRECT_URIS_SELECTED[@]} -gt 0 ]]; then
            create_args+=(--web-redirect-uris)
            create_args+=("${APP_REDIRECT_URIS_SELECTED[@]}")
        fi

        if [[ -n "$APP_LOGOUT_URI_SELECTED" ]]; then
            create_args+=(--web-logout-url "$APP_LOGOUT_URI_SELECTED")
        fi

        if [[ -n "$SERVICE_MANAGEMENT_REFERENCE" ]]; then
            create_args+=(--service-management-reference "$SERVICE_MANAGEMENT_REFERENCE")
        fi

        log_info "Invoking Azure CLI to create app registration: ${create_args[*]} --query {objectId:id, appId:appId} -o tsv"

        local create_output
        if ! create_output=$("${create_args[@]}" --query "{objectId:id, appId:appId}" -o tsv 2>&1); then
            log_error "Failed to create the Azure App Registration. Azure CLI response: $create_output"
            exit 1
        fi

        log_info "Azure CLI create output: $create_output"

        read -r object_id client_id <<< "$create_output"
        source="created"
    else
        log_info "Re-using app registration '$APP_DISPLAY_NAME' (${source:-existing})."
    fi

    APP_OBJECT_ID="$object_id"
    APP_CLIENT_ID="$client_id"

    case "$source" in
        "created") APP_REGISTRATION_STATUS="created" ;;
        "existing") APP_REGISTRATION_STATUS="existing" ;;
        "provided-client") APP_REGISTRATION_STATUS="provided via APP_CLIENT_ID" ;;
        "provided-object") APP_REGISTRATION_STATUS="provided via APP_OBJECT_ID" ;;
        *) APP_REGISTRATION_STATUS="existing" ;;
    esac

    log_info "App object ID: $APP_OBJECT_ID"
    log_info "App client ID: $APP_CLIENT_ID"

    if [[ -n "$SERVICE_MANAGEMENT_REFERENCE" ]]; then
        log_info "Ensuring service management reference is set to '$SERVICE_MANAGEMENT_REFERENCE'"
        local sm_output
        if sm_output=$(az ad app update --id "$APP_OBJECT_ID" --service-management-reference "$SERVICE_MANAGEMENT_REFERENCE" 2>&1); then
            APP_SERVICE_MANAGEMENT_STATUS="configured"
            log_info "Service management reference update response: $sm_output"
        else
            APP_SERVICE_MANAGEMENT_STATUS="failed"
            log_warn "Unable to set service management reference. Azure CLI response: $sm_output"
        fi
    else
        APP_SERVICE_MANAGEMENT_STATUS="skipped"
        log_info "SERVICE_MANAGEMENT_REFERENCE not provided; skipping service management reference configuration."
    fi
}

# Step 4: configure redirect and logout URLs on the app registration.
configure_redirect_settings() {
    log_step "Step 4: Configure redirect/logout URLs"

    require_command az

    if [[ -z "$APP_OBJECT_ID" ]]; then
        log_error "APP_OBJECT_ID is not set. Step 3 must succeed before configuring redirect URIs."
        exit 1
    fi

    if [[ ${#APP_REDIRECT_URIS_SELECTED[@]} -eq 0 ]]; then
        log_error "No redirect URIs collected. Ensure Step 2 populated APP_REDIRECT_URIS_SELECTED."
        exit 1
    fi

    local update_args=(
        az ad app update
        --id "$APP_OBJECT_ID"
        --web-redirect-uris
    )

    update_args+=("${APP_REDIRECT_URIS_SELECTED[@]}")

    if [[ -n "$APP_LOGOUT_URI_SELECTED" ]]; then
        update_args+=(--web-logout-url "$APP_LOGOUT_URI_SELECTED")
    fi

    if ! "${update_args[@]}" >/dev/null 2>&1; then
        log_error "Failed to update redirect/logout URLs. Verify your permissions and input values."
        exit 1
    fi

    APP_REDIRECT_UPDATE_STATUS="configured"

    log_info "Applied ${#APP_REDIRECT_URIS_SELECTED[@]} redirect URI(s) to app $APP_OBJECT_ID"
    if [[ -n "$APP_LOGOUT_URI_SELECTED" ]]; then
        log_info "Set logout URL to $APP_LOGOUT_URI_SELECTED"
    else
        log_warn "Logout URL left unset (optional)"
    fi
}

# Step 5: ensure workload identity federated credential exists for the AKS service account.
ensure_federated_credential() {
    log_step "Step 5: Configure workload identity federated credential"

    require_command az

    if [[ -z "$APP_OBJECT_ID" ]]; then
        log_error "APP_OBJECT_ID is not set. Run Step 3 successfully before configuring federated credentials."
        exit 1
    fi

    local issuer audience subject
    issuer=$(az aks show --resource-group "$RESOURCE_GROUP" --name "$AKS_CLUSTER_NAME" --query "oidcIssuerProfile.issuerUrl" -o tsv 2>/dev/null || true)
    if [[ -z "$issuer" ]]; then
        log_error "Unable to retrieve OIDC issuer URL from AKS cluster '$AKS_CLUSTER_NAME'. Ensure workload identity is enabled."
        exit 1
    fi

    audience="api://AzureADTokenExchange"
    subject="system:serviceaccount:${SERVICE_ACCOUNT_NAMESPACE}:${SERVICE_ACCOUNT_NAME}"

    local existing
    existing=$(az ad app federated-credential list --id "$APP_OBJECT_ID" --query "[?name=='$FEDERATED_CREDENTIAL_NAME']" -o tsv 2>/dev/null || true)

    if [[ -n "$existing" ]]; then
        log_info "Federated credential '$FEDERATED_CREDENTIAL_NAME' already exists for app $APP_OBJECT_ID."
        
        # Verify the existing credential has correct settings
        local existing_issuer existing_subject
        local cred_details
        cred_details=$(az ad app federated-credential show --id "$APP_OBJECT_ID" --federated-credential-id "$FEDERATED_CREDENTIAL_NAME" -o json 2>/dev/null || echo "{}")
        existing_issuer=$(echo "$cred_details" | grep -o '"issuer":[[:space:]]*"[^"]*"' | sed 's/"issuer":[[:space:]]*"\([^"]*\)"/\1/')
        existing_subject=$(echo "$cred_details" | grep -o '"subject":[[:space:]]*"[^"]*"' | sed 's/"subject":[[:space:]]*"\([^"]*\)"/\1/')
        
        if [[ "$existing_issuer" != "$issuer" ]] || [[ "$existing_subject" != "$subject" ]]; then
            log_warn "Existing federated credential has incorrect configuration:"
            log_warn "  Current issuer:  $existing_issuer"
            log_warn "  Expected issuer: $issuer"
            log_warn "  Current subject:  $existing_subject"
            log_warn "  Expected subject: $subject"
            log_info "Deleting and recreating federated credential with correct configuration..."
            
            # Delete the existing credential
            az ad app federated-credential delete \
                --id "$APP_OBJECT_ID" \
                --federated-credential-id "$FEDERATED_CREDENTIAL_NAME" \
                2>/dev/null || log_warn "Failed to delete existing credential, will try to create anyway"
            
            # Fall through to creation logic below
        else
            log_info "Federated credential configuration is correct."
            APP_FEDERATED_CREDENTIAL_STATUS="existing"
            return
        fi
    fi

    log_info "Creating federated credential '$FEDERATED_CREDENTIAL_NAME' for subject '$subject'."

    local credential_file
    credential_file=$(mktemp)
    trap 'rm -f "$credential_file"' RETURN

    cat >"$credential_file" <<EOF
{
  "name": "$FEDERATED_CREDENTIAL_NAME",
  "issuer": "$issuer",
  "subject": "$subject",
  "description": "AKS service account federation for ${SERVICE_ACCOUNT_NAMESPACE}/${SERVICE_ACCOUNT_NAME}",
  "audiences": [
    "$audience"
  ]
}
EOF

    local create_cmd=(
        az ad app federated-credential create
        --id "$APP_OBJECT_ID"
        --parameters "$credential_file"
    )

    log_info "Invoking Azure CLI: ${create_cmd[*]}"

    local create_output
    if ! create_output=$("${create_cmd[@]}" 2>&1); then
        log_error "Failed to create federated credential. Azure CLI response: $create_output"
        log_error "Verify Azure CLI version and permissions before retrying."
        exit 1
    fi

    APP_FEDERATED_CREDENTIAL_STATUS="created"
    log_info "Federated credential '$FEDERATED_CREDENTIAL_NAME' created successfully."
}

# Step 6: assign Microsoft Graph delegated permissions and optionally trigger admin consent.
configure_graph_permissions() {
    log_step "Step 6: Configure Microsoft Graph delegated permissions"

    require_command az

    if [[ -z "$APP_OBJECT_ID" ]]; then
        log_error "APP_OBJECT_ID is not set. Run Step 3 successfully before configuring Graph permissions."
        exit 1
    fi

    if [[ ${#APP_GRAPH_SCOPES_SELECTED[@]} -eq 0 ]]; then
        log_error "No Graph scopes collected. Ensure Step 2 populated APP_GRAPH_SCOPES_SELECTED."
        exit 1
    fi

    log_info "Assigning Microsoft Graph delegated scopes: ${APP_GRAPH_SCOPES_SELECTED[*]}"

    for scope in "${APP_GRAPH_SCOPES_SELECTED[@]}"; do
        local scope_id
        if ! scope_id=$(resolve_graph_scope_id "$scope"); then
            log_error "Unable to resolve Microsoft Graph delegated permission '$scope' to a GUID."
            log_error "Use 'az ad sp show --id 00000003-0000-0000-c000-000000000000 --query \"oauth2Permissions[].{value:value,id:id}\"' to inspect available scopes."
            exit 1
        fi

        log_info "Resolved scope '$scope' to id $scope_id"

        local permission_cmd=(
            az ad app permission add
            --id "$APP_OBJECT_ID"
            --api 00000003-0000-0000-c000-000000000000
            --api-permissions "${scope_id}=Scope"
        )

        log_info "Invoking Azure CLI: ${permission_cmd[*]}"

        local permission_output
        if ! permission_output=$("${permission_cmd[@]}" 2>&1); then
            log_error "Failed to add Graph scope '$scope'. Azure CLI response: $permission_output"
            log_error "Verify your directory permissions (needs Application Administrator or higher)."
            exit 1
        fi
    done

    APP_GRAPH_PERMISSION_STATUS="assigned"

    if [[ ${#APP_AKS_SCOPES_SELECTED[@]} -gt 0 ]]; then
        log_info "Assigning AKS server delegated scopes for API $AKS_SERVER_APP_ID: ${APP_AKS_SCOPES_SELECTED[*]}"

        for scope in "${APP_AKS_SCOPES_SELECTED[@]}"; do
            local scope_id
            if ! scope_id=$(resolve_aks_scope_id "$scope"); then
                log_error "Unable to resolve AKS server delegated permission '$scope' to a GUID."
                log_error "Update AKS_SCOPE_ID_MAP in the script or set AKS_SERVER_SCOPES to supported values."
                exit 1
            fi

            log_info "Resolved AKS scope '$scope' to id $scope_id"

            local permission_cmd=(
                az ad app permission add
                --id "$APP_OBJECT_ID"
                --api "$AKS_SERVER_APP_ID"
                --api-permissions "${scope_id}=Scope"
            )

            log_info "Invoking Azure CLI: ${permission_cmd[*]}"

            local permission_output
            if ! permission_output=$("${permission_cmd[@]}" 2>&1); then
                log_error "Failed to add AKS scope '$scope'. Azure CLI response: $permission_output"
                log_error "Verify your directory permissions (needs Application Administrator or higher)."
                exit 1
            fi
        done

        APP_AKS_PERMISSION_STATUS="assigned"
    else
        APP_AKS_PERMISSION_STATUS="skipped"
        log_warn "No AKS server scopes configured; skipping AKS permission assignment."
    fi

    if [[ "${APP_GRANT_GRAPH_ADMIN_CONSENT:-false}" == "true" ]]; then
        log_info "Attempting admin consent for assigned scopes."
        if az ad app permission grant --id "$APP_OBJECT_ID" --api 00000003-0000-0000-c000-000000000000 >/dev/null 2>&1; then
            if az ad app permission admin-consent --id "$APP_OBJECT_ID" >/dev/null 2>&1; then
                APP_GRAPH_ADMIN_CONSENT_STATUS="granted"
                log_info "Admin consent granted successfully."
            else
                APP_GRAPH_ADMIN_CONSENT_STATUS="grant_failed"
                log_warn "Admin consent command failed. You may need higher privileges or manual approval."
            fi
        else
            APP_GRAPH_ADMIN_CONSENT_STATUS="grant_failed"
            log_warn "Unable to grant delegated permissions automatically; manual admin consent may be required."
        fi
    else
        APP_GRAPH_ADMIN_CONSENT_STATUS="not_requested"
        log_info "Skipping admin consent (APP_GRANT_GRAPH_ADMIN_CONSENT=false)."
    fi
}

# Step 7: emit a summary of the collected (and soon-to-be generated) configuration values.
output_summary() {
    log_step "Step 7: Output configuration and next actions"

    local tenant_display="${APP_TENANT_ID:-<pending>}"
    local object_id_display="${APP_OBJECT_ID:-<pending>}"
    local client_id_display="${APP_CLIENT_ID:-<pending>}"
    local registration_status="${APP_REGISTRATION_STATUS:-pending}"
    local redirect_status="${APP_REDIRECT_UPDATE_STATUS:-pending}"
    local federated_status="${APP_FEDERATED_CREDENTIAL_STATUS:-pending}"
    local graph_status="${APP_GRAPH_PERMISSION_STATUS:-pending}"
    local graph_admin_status="${APP_GRAPH_ADMIN_CONSENT_STATUS:-not_requested}"
    local service_mgmt_status="${APP_SERVICE_MANAGEMENT_STATUS:-pending}"

    log_info "Azure tenant: ${tenant_display}"
    log_info "App display name: ${APP_DISPLAY_NAME}"
    log_info "App registration status: ${registration_status}"
    log_info "Redirect configuration status: ${redirect_status}"
    log_info "Federated credential status: ${federated_status}"
    log_info "Graph delegated permission status: ${graph_status}"
    log_info "Graph admin consent status: ${graph_admin_status}"
    log_info "AKS delegated permission status: ${APP_AKS_PERMISSION_STATUS:-pending}"
    log_info "Service management reference status: ${service_mgmt_status}"

    if [[ ${#APP_REDIRECT_URIS_SELECTED[@]} -gt 0 ]]; then
        log_info "Redirect URIs:"
        for uri in "${APP_REDIRECT_URIS_SELECTED[@]}"; do
            echo "  - ${uri}"
        done
    else
        log_warn "No redirect URIs recorded yet"
    fi

    if [[ -n "${APP_LOGOUT_URI_SELECTED}" ]]; then
        log_info "Logout URL: ${APP_LOGOUT_URI_SELECTED}"
    else
        log_warn "Logout URL not provided (optional)"
    fi

    if [[ ${#APP_GRAPH_SCOPES_SELECTED[@]} -gt 0 ]]; then
        log_info "Microsoft Graph delegated scopes: ${APP_GRAPH_SCOPES_SELECTED[*]}"
    else
        log_warn "Graph scopes not yet determined"
    fi

    if [[ ${#APP_AKS_SCOPES_SELECTED[@]} -gt 0 ]]; then
        log_info "AKS server delegated scopes: ${APP_AKS_SCOPES_SELECTED[*]}"
    else
        log_warn "AKS server scopes not yet determined"
    fi

    log_info "App object ID: ${object_id_display}"
    log_info "App client ID: ${client_id_display} <----- ENTRA_CLIENTID"

    log_info "Federated credential name (Step 5 target): ${FEDERATED_CREDENTIAL_NAME}"
    log_info "Service account namespace/name: ${SERVICE_ACCOUNT_NAMESPACE}/${SERVICE_ACCOUNT_NAME}"
}

main() {
    validate_prerequisites
    collect_configuration
    ensure_app_registration
    configure_redirect_settings
    ensure_federated_credential
    configure_graph_permissions
    output_summary
}

if [[ "${BASH_SOURCE[0]}" == "$0" ]]; then
    main "$@"
fi
```

**3. Run the script:**

```azurecli-interactive
export AKS_CLUSTER_NAME="$CLUSTER_NAME"  # Map cluster name from Step 2
export APP_REDIRECT_URI="https://<your-app-hostname>/auth/callback"
./setup-azure-app-registration.sh
```

> [!NOTE]
> Use `source` (instead of `./`) so the exported environment variables (`APP_CLIENT_ID`, `TENANT_ID`) remain available in your current shell for subsequent steps.

---

### Step 8: Install the extension

With all prerequisites in place — AKS cluster, Azure OpenAI resource, managed identity with RBAC roles and federated credentials, and (optionally) an App Registration — you can now install the Container Networking Agent as an AKS cluster extension. The extension deploys the agent pod into the `kube-system` namespace and configures it with the identity, OpenAI, and authentication settings from the previous steps.

Install the extension using the [`az k8s-extension create`](/cli/azure/k8s-extension#az-k8s-extension-create) command. Use the command that matches your cluster configuration.

> [!TIP]
> The `--version` flag is optional. If you omit it, the extension installs the latest available version automatically.

**For clusters with ACNS and Cilium dataplane:**

```azurecli-interactive
# Map variables from previous steps to the names used by the extension
export AZURE_TENANT_ID=$(az account show --query tenantId -o tsv)
export AZURE_SUBSCRIPTION_ID=$SUBSCRIPTION_ID
export AZURE_OPENAI_DEPLOYMENT=$OPENAI_DEPLOYMENT_NAME

az k8s-extension create \
    --cluster-name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --cluster-type managedClusters \
    --name containernetworkingagent \
    --extension-type microsoft.containernetworkingagent \
    --version 0.0.9 \ 
    --release-train stable \
    --auto-upgrade-minor-version false \
    --scope cluster \
    --configuration-settings image.repository=acnpublic.azurecr.io/unlisted/containernetworking/container-networking-agent \
    --configuration-settings image.tag=v0.0.8 \
    --configuration-settings config.AZURE_CLIENT_ID=$AZURE_CLIENT_ID \
    --configuration-settings config.AZURE_TENANT_ID=$AZURE_TENANT_ID \
    --configuration-settings config.AZURE_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID \
    --configuration-settings config.AKS_CLUSTER_NAME=$CLUSTER_NAME \
    --configuration-settings config.AKS_RESOURCE_GROUP=$RESOURCE_GROUP \
    --configuration-settings config.ENTRA_TENANT_ID=$AZURE_TENANT_ID \
    --configuration-settings config.ENTRA_CLIENT_ID=$APP_CLIENT_ID \
    --configuration-settings config.AZURE_OPENAI_DEPLOYMENT=$AZURE_OPENAI_DEPLOYMENT \
    --configuration-settings config.AZURE_OPENAI_API_VERSION=2025-03-01-preview \
    --configuration-settings config.AZURE_OPENAI_ENDPOINT=$AZURE_OPENAI_ENDPOINT \
    --configuration-settings config.AKS_MCP_ENABLED_COMPONENTS=kubectl \
    --configuration-settings config.DISABLE_TELEMETRY=false
```

**For clusters without ACNS (Hubble disabled):**

```azurecli-interactive
# Map variables from previous steps to the names used by the extension (skip if already set above)
export AZURE_TENANT_ID=$(az account show --query tenantId -o tsv)
export AZURE_SUBSCRIPTION_ID=$SUBSCRIPTION_ID
export AZURE_OPENAI_DEPLOYMENT=$OPENAI_DEPLOYMENT_NAME

az k8s-extension create \
    --cluster-name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --cluster-type managedClusters \
    --name containernetworkingagent \
    --extension-type microsoft.containernetworkingagent \
    --version 0.0.9 \
    --release-train stable \
    --auto-upgrade-minor-version false \
    --scope cluster \
    --configuration-settings image.repository=acnpublic.azurecr.io/unlisted/containernetworking/container-networking-agent \
    --configuration-settings image.tag=v0.0.8 \
    --configuration-settings config.AZURE_CLIENT_ID=$AZURE_CLIENT_ID \
    --configuration-settings config.AZURE_TENANT_ID=$AZURE_TENANT_ID \
    --configuration-settings config.AZURE_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID \
    --configuration-settings config.AKS_CLUSTER_NAME=$CLUSTER_NAME \
    --configuration-settings config.AKS_RESOURCE_GROUP=$RESOURCE_GROUP \
    --configuration-settings config.ENTRA_TENANT_ID=$AZURE_TENANT_ID \
    --configuration-settings config.ENTRA_CLIENT_ID=$APP_CLIENT_ID \
    --configuration-settings config.AZURE_OPENAI_DEPLOYMENT=$AZURE_OPENAI_DEPLOYMENT \
    --configuration-settings config.AZURE_OPENAI_API_VERSION=2025-03-01-preview \
    --configuration-settings config.AZURE_OPENAI_ENDPOINT=$AZURE_OPENAI_ENDPOINT \
    --configuration-settings config.AKS_MCP_ENABLED_COMPONENTS=kubectl \
    --configuration-settings config.DISABLE_TELEMETRY=false \
    --configuration-settings hubble.enabled=false
```

> [!NOTE]
> On clusters without ACNS, set `hubble.enabled=false` and `config.AKS_MCP_ENABLED_COMPONENTS=kubectl`. The agent still provides DNS, packet drop, and standard Kubernetes networking diagnostics. Hubble flow analysis and Cilium policy diagnostics aren't available.

### Step 9: Verify the extension installation

Check the extension provisioning state using the [`az k8s-extension show`](/cli/azure/k8s-extension#az-k8s-extension-show) command.

```azurecli-interactive
az k8s-extension show \
    --cluster-name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --cluster-type managedClusters \
    --name containernetworkingagent \
    --query "provisioningState" -o tsv
```

The output should show `Succeeded`.

Verify the agent pods are running:

```azurecli-interactive
kubectl get pods -n kube-system | grep container-networking-agent
```

Verify that one or more pods show `Running` status.

### Step 10: Access the agent

Forward the service port to your local machine:

```azurecli-interactive
kubectl port-forward svc/container-networking-agent-service -n kube-system 8080:80
```

Open your browser and go to `http://localhost:8080`. Sign in and start asking networking questions.

## Validate the deployment

After you deploy Container Networking Agent, verify that the deployment is healthy.

### Check pod status

```azurecli-interactive
kubectl get pods -n kube-system | grep container-networking-agent
```

**Expected output:** One or more pods in `Running` status with `1/1` ready containers.

### Check health endpoints

Forward the service port if you haven't already:

```azurecli-interactive
kubectl port-forward svc/container-networking-agent-service -n kube-system 8080:80
```

Check the health, readiness, and liveness endpoints:

```azurecli-interactive
# Health check
curl http://localhost:8080/health

# Readiness probe
curl http://localhost:8080/ready

# Liveness probe
curl http://localhost:8080/live
```

All three endpoints should return HTTP 200 responses.

### Check pod logs

```azurecli-interactive
kubectl logs -n kube-system -l app=container-networking-agent --tail=100
```

A healthy deployment shows:

- Successful Azure OpenAI connectivity validation.
- Agent warmup pool initialization (default: three pre-warmed agents).
- No authentication or connection errors.

### Verify extension state

Verify the extension state in Azure using the [`az k8s-extension show`](/cli/azure/k8s-extension#az-k8s-extension-show) command.

```azurecli-interactive
az k8s-extension show \
    --cluster-name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --cluster-type managedClusters \
    --name containernetworkingagent \
    --query "{provisioningState:provisioningState, version:version}" -o table
```

The `provisioningState` value should show `Succeeded` with a version number.

### Verify service account

Confirm the service account exists and has the correct workload identity annotation:

```azurecli-interactive
kubectl get serviceaccount container-networking-agent-reader -n kube-system -o yaml
```

The annotation `azure.workload.identity/client-id` should match your managed identity client ID.

## Use Container Networking Agent

After deployment and validation, access the agent through the web chat interface.

### Access the chat interface

1. Forward the service port:

   ```azurecli-interactive
   kubectl port-forward svc/container-networking-agent-service -n kube-system 8080:80
   ```

1. Open `http://localhost:8080` in your browser.

1. Sign in using either simple username login (development) or Microsoft Entra ID (production), depending on your configuration.

### Start a diagnostic conversation

Type a question or describe a networking problem in the chat input. The agent follows a structured diagnostic workflow:

1. **Classify**: Determines the issue category (DNS, connectivity, network policy, service routing, or packet drops).
1. **Collect evidence**: Runs the appropriate diagnostic commands against your cluster.
1. **Analyze**: Examines the collected evidence to identify anomalies and root causes.
1. **Report**: Returns a structured report with evidence tables, root cause analysis, and remediation commands.

### Sample prompts

| Scenario | Prompt |
|----------|--------|
| DNS failure | *"A pod in namespace `my-app` cannot resolve any DNS names"* |
| Packet drops | *"I see packet drops on node `aks-nodepool1-12345678-vmss000000`"* |
| Service unreachable | *"My client pod cannot connect to the backend-service in namespace `production`"* |
| Network policy blocking traffic | *"Pods in namespace `frontend` cannot communicate with the `backend` namespace"* |
| Cluster-wide DNS failure | *"All DNS is broken in the cluster"* |
| Proactive health check | *"Check network health on node `my-node`"* |

### Diagnostic workflows

- **DNS troubleshooting**: The agent checks CoreDNS pod health, service endpoints, and CoreDNS configuration (including custom ConfigMaps). It also checks NodeLocal DNS status, DNS resolution from multiple paths, and network policies that might block DNS traffic.
- **Packet drop analysis**: The agent deploys a lightweight debug DaemonSet to collect host-level network statistics. It examines NIC ring buffer utilization, kernel softnet statistics, per-CPU SoftIRQ distribution, socket buffer saturation, and network interface statistics. Delta measurements detect active drops versus historical counters.
- **Kubernetes networking diagnostics**: The agent examines pod status and scheduling, service configuration and endpoint registration, network policies (both Kubernetes and Cilium), Hubble flows, and service-to-pod port mapping.

### Diagnostic output

Each diagnostic response includes:

- A summary of the issue and its status.
- An evidence table showing each check, its result, and whether it passed or failed.
- Analysis of what's working and what's broken.
- Root cause identification with specific evidence citations.
- Exact commands to fix the issue and verify the fix.

### Session and conversation limits

| Limit | Default |
|-------|---------|
| Chat context window | ~15 exchanges |
| Messages per conversation | 100 |
| Conversations per user | 20 |
| Session idle timeout | 30 minutes |
| Session absolute timeout | 8 hours |

Start a new conversation for unrelated issues to keep context fresh.

## Cluster access and security

Container Networking Agent uses a dedicated service account (`container-networking-agent-reader`) with a read-only ClusterRole in the `kube-system` namespace. The RBAC configuration uses the principle of least privilege:

- **Read access** to core Kubernetes resources: Pods, Services, Nodes, Namespaces, ConfigMaps, Events, Deployments, ReplicaSets, DaemonSets, StatefulSets, Ingresses, NetworkPolicies, Endpoints, EndpointSlices, PersistentVolumes, and PersistentVolumeClaims.
- **Read access** to Cilium CRDs: CiliumNetworkPolicies, CiliumEndpoints, CiliumIdentities, CiliumLoadBalancerIPPools, CiliumL2AnnouncementPolicies, and other Cilium resources.
- **Read access** to metrics: Node and Pod metrics via metrics-server.
- **Limited exec access**: The agent uses `pods/exec` only for diagnostic commands (Cilium status and endpoint information).
- **No write access**: The agent can't create, update, or delete any cluster resource.

The agent pod makes outbound HTTPS calls to your Azure OpenAI endpoint. If you use egress restrictions (network policies, Azure Firewall, or NSGs), allow outbound traffic from the `kube-system` namespace to your Azure OpenAI endpoint on port 443.

For packet drop diagnostics, the agent deploys a lightweight debug DaemonSet (`retina-debug-daemonset`) in `kube-system` that requires `hostNetwork` and `NET_ADMIN` capabilities. This DaemonSet is shared across diagnostic sessions and cleaned up automatically.

The agent doesn't persist diagnostic data externally. The pod stores session data (chat history, agent assignments) in memory, and this data is lost if the pod restarts.

## Manage the extension

### Update the extension

Update the extension to a new version using the [`az k8s-extension update`](/cli/azure/k8s-extension#az-k8s-extension-update) command.

```azurecli-interactive
az k8s-extension update \
    --cluster-name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --cluster-type managedClusters \
    --name containernetworkingagent \
    --version <new-version>
```

### Uninstall the extension

Remove the extension using the [`az k8s-extension delete`](/cli/azure/k8s-extension#az-k8s-extension-delete) command.

```azurecli-interactive
az k8s-extension delete \
    --cluster-name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --cluster-type managedClusters \
    --name containernetworkingagent
```

## Troubleshoot

If you encounter issues during deployment, setup, or while using Container Networking Agent, see [Troubleshoot Container Networking Agent on AKS](./troubleshoot-container-networking-agent.md). The troubleshooting guide covers common scenarios including extension installation failures, identity and permissions errors, Azure OpenAI connectivity issues, authentication errors, pod startup failures, and diagnostic capability issues.

## Next steps

- [Troubleshoot Container Networking Agent on AKS](./troubleshoot-container-networking-agent.md)
- [Container Networking Agent overview](./container-networking-agent-overview.md)
- [Advanced Container Networking Services overview](/azure/aks/advanced-container-networking-services-overview)
- [Azure CNI powered by Cilium](/azure/aks/azure-cni-powered-by-cilium)
