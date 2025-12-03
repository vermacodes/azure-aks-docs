---
title: Azure Kubernetes Service (AKS) application routing add-on with the Kubernetes Gateway API (preview)
description: Use the application routing add-on to manage ingress traffic on Azure Kubernetes Service (AKS) using the Kubernetes Gateway API.
ms.subservice: aks-networking
ms.custom: devx-track-azurecli, biannual
author: nshankar
ms.topic: how-to
ms.date: 11/18/2025
ms.author: nshankar
# Customer intent: As a cloud engineer, I want to deploy and configure ingress on Azure Kubernetes Service with the Kubernetes Gateway API using the application routing add-on, so that I can efficiently manage HTTP/HTTPS traffic to my applications.
---

# Configure ingress with the Kubernetes Gateway API via the application routing add-on (preview)

[!INCLUDE [preview features callout](~/reusable-content/ce-skilling/azure/includes/aks/includes/preview/preview-callout.md)]

The application routing add-on now supports the Kubernetes Gateway API for ingress traffic management. If you are using [managed NGINX][app-routing-nginx] with the legacy Ingress API, migrating to using the Kubernetes Gateway API implementation is strongly recommended.

The application routing add-on Kubernetes Gateway API implementation deploys an Istio control plane to reconcile Kubernetes Gateway API resources. However, it differs from the [Istio service mesh add-on for AKS][istio-addon] in the following ways:
* The application routing add-on's Istio control plane does not support sidecar injection or other Istio Custom Resource Definitions (CRDs). It only reconciles Gateway API resources.
* The application routing add-on's Istio control plane is not [revisioned][istio-revisions] and is upgraded in-place for both minor and patch version updates.

## Limitations

* The application routing Gateway API implementation and the [Istio service mesh add-on][istio-addon] cannot be enabled simultaneously. You must disable one first in order to enable the other.
* The application routing Gateway API implementation uses the same [resource customization allowlist][resource-customization-allowlist] as the Istio add-on for validating ConfigMap customizations for `Gateway` resources. Customizations not on the allowlist are disallowed and blocked via add-on managed webhooks.
* [Azure DNS and TLS certificate management][app-routing-dns-tls] via the application routing add-on is currently not supported for the Kubernetes Gateway API. You can follow the steps in the [Transport Layer Security (TLS) ingress gateway](#configure-a-tls-ingress-gateway) to configure a `Gateway` to perform TLS termination.
* Configuring HTTPS ingress access to HTTPS services - i.e Server Name Indication (SNI) Passthrough - via the `TLSRoute` resource is currently unsupported.

## Prerequisites

* AZ CLI version.
* After you [enable the application routing Gateway API implementation](#enable-the-application-routing-gateway-api-implementation), you must install the [Managed Gateway API CRDs][managed-gateway-api] prior to creating any Kubernetes Gateway API resources. Use of self-managed Gateway API CRDs with the application routing add-on is unsupported.

## Enable the application routing Gateway API implementation

Run the following command to enable the application routing Gateway API implementation:

```azurecli-interactive

```

You should see `istiod` pods in the `aks-istio-system` namespace:

```bash
kubectl get pods -n aks-istio-system
```

```output
NAME                      READY   STATUS    RESTARTS   AGE
istiod-54b4ff45cf-htph8   1/1     Running   0          3m15s
istiod-54b4ff45cf-wlvgd   1/1     Running   0          3m
```

You should also see the `validatingwebhookconfiguration` get deployed:

```bash
kubectl get validatingwebhookconfiguration
```

```output
NAME                                        WEBHOOKS   AGE
aks-node-validating-webhook                 1          117m
azure-service-mesh-ccp-validating-webhook   1          4m2s
```

## Install Managed Gateway API CRDs

Follow the instructions in the [Managed Gateway API document][managed-gateway-api] to install the Kubernetes Gateway API CRDs onto your cluster. Because the Managed Gateway API installation requires another implementation to be enabled first, you must enable the application routing Gateway API implementation prior to, or simultaneously with, enabling the Managed Gateway API CRDs. Note that use of self-managed Gateway API CRDs with the application routing add-on is unsupported.

After installing the CRDs, you should also see the Istio gateway customization ConfigMap get created:

```bash
kubectl get cm -n aks-istio-system
```

```output
NAME                                  DATA   AGE
...
istio-gateway-class-defaults          2      43s
...
```

## Configure ingress using a Kubernetes Gateway

### Deploy sample application

First, deploy the sample `httpbin` application in the `default` namespace:

```bash
export ISTIO_RELEASE="release-1.27"
kubectl apply -f https://raw.githubusercontent.com/istio/istio/$ISTIO_RELEASE/samples/httpbin/httpbin.yaml
```

### Create Kubernetes Gateway and HTTPRoute

Next, deploy a Gateway API configuration in the `default` namespace with the `gatewayClassName` set to `istio`. 

```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  gatewayClassName: istio
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httpbin
spec:
  parentRefs:
  - name: httpbin-gateway
  hostnames: ["httpbin.example.com"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /get
    backendRefs:
    - name: httpbin
      port: 8000
EOF
```

> [!NOTE]
> The example above creates an external ingress load balancer service that's accessible from outside the cluster. You can add [annotations][annotation-customizations] to create an [internal load balancer][azure-internal-lb] and customize other load balancer settings.

Verify that a `Deployment`, `Service`, `HorizontalPodAutoscaler`, and `PodDisruptionBudget` get created for `httpbin-gateway`:

```bash
kubectl get deployment httpbin-gateway-istio
```

```output
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
httpbin-gateway-istio   2/2     2            2           6m41s
```

```bash
kubectl get service httpbin-gateway-istio
```

```output
NAME                    TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)                        AGE
httpbin-gateway-istio   LoadBalancer   10.0.54.96   <external-ip>    15021:30580/TCP,80:32693/TCP   7m13s
```

```bash
kubectl get hpa httpbin-gateway-istio
```

```output
NAME                    REFERENCE                          TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
httpbin-gateway-istio   Deployment/httpbin-gateway-istio   cpu: 3%/80%   2         5         2          8m13s
```

```bash
kubectl get pdb httpbin-gateway-istio
```

```output
NAME                    MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
httpbin-gateway-istio   1               N/A               1                     9m1s
```

### Send request to sample application

Finally, try sending a `curl` request to the `httpbin` application. First, set the `INGRESS_HOST` environment variable:

```bash
kubectl wait --for=condition=programmed gateways.gateway.networking.k8s.io httpbin-gateway
export INGRESS_HOST=$(kubectl get gateways.gateway.networking.k8s.io httpbin-gateway -ojsonpath='{.status.addresses[0].value}')
```

Then, try sending an HTTP request to `httpbin`:

```bash
curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST/get"
```

You should see an `HTTP 200` response.

### Securing ingress traffic with the Kubernetes Gateway API

The application routing add-on supports syncing secrets from Azure Key Vault (AKV) for securing Gateway API ingress traffic with TLS termination. Follow the steps below to create certificates and keys to terminate TLS traffic at the Gateway.

#### Required client/server certificates and keys

1. Create a root certificate and private key for signing the certificates for sample services:

```bash
mkdir httpbin_certs
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout httpbin_certs/example.com.key -out httpbin_certs/example.com.crt
```

2. Generate a certificate and a private key for `httpbin.example.com`:

```bash
openssl req -out httpbin_certs/httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout httpbin_certs/httpbin.example.com.key -subj "/CN=httpbin.example.com/O=httpbin organization"
openssl x509 -req -sha256 -days 365 -CA httpbin_certs/example.com.crt -CAkey httpbin_certs/example.com.key -set_serial 0 -in httpbin_certs/httpbin.example.com.csr -out httpbin_certs/httpbin.example.com.crt
```

#### Configure a TLS ingress gateway

##### Set up Azure Key Vault and sync secrets to the cluster

1. Create Azure Key Vault

    You need an [Azure Key Vault resource][akv-quickstart] to supply the certificate and key inputs to the application routing add-on.

    ```bash
    export AKV_NAME=<azure-key-vault-resource-name>  
    az keyvault create --name $AKV_NAME --resource-group $RESOURCE_GROUP --location $LOCATION
    ```
    
2. Enable [Azure Key Vault provider for Secret Store CSI Driver][akv-addon] add-on on your cluster.

    ```bash
    az aks enable-addons --addons azure-keyvault-secrets-provider --resource-group $RESOURCE_GROUP --name $CLUSTER
    ```
    
3. If your Key Vault is using Azure RBAC for the permissions model, follow the instructions [here][akv-rbac-guide] to assign an Azure role of Key Vault Secrets User for the add-on's user-assigned managed identity. Alternatively, if your key vault is using the vault access policy permissions model, authorize the user-assigned managed identity of the add-on to access Azure Key Vault resource using access policy:
    
    ```bash
    OBJECT_ID=$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER --query 'addonProfiles.azureKeyvaultSecretsProvider.identity.objectId' -o tsv | tr -d '\r')
    CLIENT_ID=$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER --query 'addonProfiles.azureKeyvaultSecretsProvider.identity.clientId')
    TENANT_ID=$(az keyvault show --resource-group $RESOURCE_GROUP --name $AKV_NAME --query 'properties.tenantId')
    
    az keyvault set-policy --name $AKV_NAME --object-id $OBJECT_ID --secret-permissions get list
    ```

4. Create secrets in Azure Key Vault using the certificates and keys.

    ```bash
    az keyvault secret set --vault-name $AKV_NAME --name test-httpbin-key --file httpbin_certs/httpbin.example.com.key
    az keyvault secret set --vault-name $AKV_NAME --name test-httpbin-crt --file httpbin_certs/httpbin.example.com.crt
    ```

5. Use the following manifest to deploy SecretProviderClass to provide Azure Key Vault specific parameters to the CSI driver.
    
    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: secrets-store.csi.x-k8s.io/v1
    kind: SecretProviderClass
    metadata:
      name: httpbin-credential-spc
    spec:
      provider: azure
      secretObjects:
      - secretName: httpbin-credential
        type: kubernetes.io/tls
        data:
        - objectName: test-httpbin-key
          key: tls.key
        - objectName: test-httpbin-crt
          key: tls.crt
      parameters:
        useVMManagedIdentity: "true"
        userAssignedIdentityID: $CLIENT_ID 
        keyvaultName: $AKV_NAME
        cloudName: ""
        objects:  |
          array:
            - |
              objectName: test-httpbin-key
              objectType: secret
              objectAlias: "test-httpbin-key"
            - |
              objectName: test-httpbin-crt
              objectType: secret
              objectAlias: "test-httpbin-crt"
        tenantId: $TENANT_ID
    EOF
    ```

    Alternatively, to reference a certificate object type directly from Azure Key Vault, use the following manifest to deploy SecretProviderClass. In this example, `test-httpbin-cert-pxf` is the name of the certificate object in Azure Key Vault. Refer to [obtain certificates and keys][akv-csi-driver-obtain-cert-and-keys] section for more information. 
    
    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: secrets-store.csi.x-k8s.io/v1
    kind: SecretProviderClass
    metadata:
      name: httpbin-credential-spc
    spec:
      provider: azure
      secretObjects:
      - secretName: httpbin-credential
        type: kubernetes.io/tls
        data:
        - objectName: test-httpbin-key
          key: tls.key
        - objectName: test-httpbin-crt
          key: tls.crt
      parameters:
        useVMManagedIdentity: "true"
        userAssignedIdentityID: $CLIENT_ID 
        keyvaultName: $AKV_NAME
        cloudName: ""
        objects:  |
          array:
            - |
              objectName: test-httpbin-cert-pfx  #certificate object name from keyvault
              objectType: secret
              objectAlias: "test-httpbin-key"
            - |
              objectName: test-httpbin-cert-pfx #certificate object name from keyvault
              objectType: cert
              objectAlias: "test-httpbin-crt"
        tenantId: $TENANT_ID
    EOF
    ``` 

6. Use the following manifest to deploy a sample pod. The secret store CSI driver requires a pod to reference the SecretProviderClass resource to ensure secrets sync from Azure Key Vault to the cluster.
    
    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: secrets-store-sync-httpbin
    spec:
      containers:
        - name: busybox
          image: mcr.microsoft.com/oss/busybox/busybox:1.33.1
          command:
            - "/bin/sleep"
            - "10"
          volumeMounts:
          - name: secrets-store01-inline
            mountPath: "/mnt/secrets-store"
            readOnly: true
      volumes:
        - name: secrets-store01-inline
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "httpbin-credential-spc"
    EOF
    ```

    - Verify that the `httpbin-credential` secret is created in the `default` namespace as defined in the SecretProviderClass resource.
    
        ```bash
        kubectl describe secret/httpbin-credential
        ```       
        Example output:
        ```output
        Name:         httpbin-credential
        Namespace:    default
        Labels:       secrets-store.csi.k8s.io/managed=true
        Annotations:  <none>
        
        Type:  kubernetes.io/tls
        
        Data
        ====
        tls.crt:  1180 bytes
        tls.key:  1675 bytes
        ```

##### Deploy TLS Gateway

1. Create a Kubernetes Gateway that references the `httpbin-credential` secret under the TLS configuration:

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: gateway.networking.k8s.io/v1
    kind: Gateway
    metadata:
      name: httpbin-gateway
    spec:
      gatewayClassName: istio
      listeners:
      - name: https
        hostname: "httpbin.example.com"
        port: 443
        protocol: HTTPS
        tls:
          mode: Terminate
          certificateRefs:
          - name: httpbin-credential
        allowedRoutes:
          namespaces:
            from: Selector
            selector:
              matchLabels:
                kubernetes.io/metadata.name: default
    EOF
    ```

    Then, create a corresponding `HTTPRoute` to configure the gateway's ingress traffic routes:

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: httpbin
    spec:
      parentRefs:
      - name: httpbin-gateway
      hostnames: ["httpbin.example.com"]
      rules:
      - matches:
        - path:
            type: PathPrefix
            value: /status
        - path:
            type: PathPrefix
            value: /delay
        backendRefs:
        - name: httpbin
          port: 8000
    EOF
    ```

    Get the gateway address and port:

    ```bash
    kubectl wait --for=condition=programmed gateways.gateway.networking.k8s.io httpbin-gateway
    export INGRESS_HOST=$(kubectl get gateways.gateway.networking.k8s.io httpbin-gateway -o jsonpath='{.status.addresses[0].value}')
    export SECURE_INGRESS_PORT=$(kubectl get gateways.gateway.networking.k8s.io httpbin-gateway -o jsonpath='{.spec.listeners[?(@.name=="https")].port}')
    ```

2. Send an HTTPS request to access the `httpbin` service:

    ```bash
    curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
    --cacert httpbin_certs/example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
    ```

    You should see the httpbin service return the 418 I’m a Teapot code.

## Versioning and Upgrades

The application routing Gateway API implementation deploys and upgrades the Istio control plane based on the AKS cluster Kubernetes version **for both minor version and patch version upgrades**.

The Istio version is the maximum supported Istio minor version for the AKS version, which can be found in the follwoing table. For instance, if you are on AKS version `1.29`, the maximum supported Istio minor version that is installed is `1.27`. Keep in mind that the maximum supported Istio version for a given Kubernetes version could differ between [Long-Term Support (LTS) clusters][aks-lts] and non-LTS clusters.

| Istio version | Upstream release | AKS release | End of life | Compatible AKS versions | Compatible AKS LTS versions |
|--------------|-------------------|--------------|---------|-------------|-----------------------|-----------------------|
| 1.27 | Aug 2025 | Sept 2025 | ~May 2026 (expected) | 1.29, 1.30, 1.31, 1.32, 1.33, 1.34 | 1.29, 1.30, 1.31, 1.32, 1.33, 1.34 |

### Upgrades

Upgrades of Istio control plane for the application routing Gateway API implementation occur in-place and are triggered in the following two scenarios:

- Manual: The AKS cluster is upgraded to a new version which has a higher maximum supported Istio version corresponding to it. The Istio control plane will be upgraded to the higher minor version as part of the AKS cluster upgrade.
- Automatic: A new Istio version is released for AKS and is now the maximum supported Istio version for the AKS cluster version. The Istio control plane on your cluster will automatically be upgraded to the new minor version after the release is rolled out to your region. Follow the [AKS release notes][aks-release-notes] and [AKS release tracker][aks-release-tracker] to track new Istio version releases and see when the new version has rolled out to your region.

> [!NOTE]
> Patch version upgrades of the Istio control plane are triggered automatically. Minor version upgrades can be triggered either automatically or manually depending on the AKS version and timing of Istio minor version releases.

## Resource customizations

The application routing Gateway API implementation supports customization of the `Gateway` resources via annotations and ConfigMaps. Application routing uses the same resource customization allowlist as the Istio service mesh add-on for Gateway API resource customization. Follow the steps in the [Istio add-on Gateway API docs][istio-gateway-resource-customization] to configure resources generated for the `Gateways` and to see which fields fall under the [allowlist][resource-customization-allowlist].

## Disable the application routing Gateway API implementation

Run the following command to disable the application routing Gateway API implementation:

```azurecli-interactive

```

### Cleanup Resources

Run the following commands to delete the `Gateway` and `HttpRoute`:

```bash
kubectl delete gateways.gateway.networking.k8s.io httpbin-gateway
kubectl delete httproute httpbin
```

If you created a ConfigMap to customize your `Gateway`, run the following command to delete the ConfigMap:

```bash
kubectl delete configmap gw-options
```

If you created a SecretProviderClass and secret to use for TLS termination, delete the following resources as well:

```bash
kubectl delete secret httpbin-credential
kubectl delete pod secrets-store-sync-httpbin
kubectl delete secretproviderclass httpbin-credential-spc
```

<!-- LINKS - internal -->

[annotation-customization]: istio-gateway-api.md#annotation-customizations
[app-routing-nginx]: app-routing.md
[app-routing-dns-tls]: app-routing-dns-ssl.md
[aks-lts]: long-term-support.md
[akv-rbac-guide]: /azure/key-vault/general/rbac-guide#using-azure-rbac-secret-key-and-certificate-permissions-with-key-vault
[azure-internal-lb]: ./internal-lb.md
[aks-release-tracker]: ./release-tracker.md
[istio-addon]: istio-about.md
[istio-gateway-resource-customization]: istio-gateway-api.md#resource-customizations
[resource-customization-allowlist]: istio-gateway-api.md#resource-customization-allowlist
[managed-gateway-api]: managed-gateway-api.md
[istio-release-calendar]: istio-support-policy.md#service-mesh-add-on-release-calendar

<!-- LINKS - external -->
[aks-release-notes]: https://github.com/azure/aks/releases
[istio-revisions]: https://istio.io/latest/blog/2021/revision-tags/
