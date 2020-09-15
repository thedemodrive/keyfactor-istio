# Setup guidelines for Keyfactor Kubernetes Integration

## Prerequisite

- These steps require you to have a cluster running a compatible version of Kubernetes. You can use any supported platform, for example [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) or others specified by the [platform-specific setup instructions](https://istio.io/docs/setup/platform-setup/).
- Require Kubernetes Version > **1.18** (eg: minikube start --memory=10000 --cpus=4 --kubernetes-version=1.18.6)
- Require Helm 3 to install Keyfactor-Proxy. [Helm 3 Instruction](https://helm.sh/docs/intro/install/)
- Download & extract release bundle: [**Click here to download release bundle keyfactor-v2-beta **](https://github.com/thedemodrive/keyfactor-istio/releases/download/keyfactor-v2-beta/keyfactor-v2-beta.zip).
- Add the `istioctl` client at `./release/istioctl-*` to your path (Linux or macOS or Windows):
  - OSX: `istioctl-osx`
  - Linux: `istioctl-linux-amd64`
  - Windows: `istioctl-win.exe`

## Install Keyfactor-K8S-Proxy

1. Update file `credentials.yaml`

```YAML
# Endpoint of Keyfactor Platform
endPoint: ""

# Name of certificate authorization for enroll Istio's Workload Certificate
caName: ""

# Using for authentication header: "Basic ...."
authToken: ""

# API path for enroll new certificate from Keyfactor
enrollPath: ""

# Certificate Template for enroll Istio's Workload Certificate: Default is Istio
caTemplate: "Istio"

# ApiKey from Api Setting, for enroll Istio's Workload Certificate
appKey: ""

# ApiKey for auto provisioning TLS server / client certificates
provisioningAppKey: ""

# CA Template for auto provisioning TLS server / client certificates
provisioningTemplate: "Istio"
```

2. Create kubernetes secrets `keyfactor-credentials` contains Keyfactor's credentials

```bash
kubectl create namespace keyfactor
kubectl create secret generic keyfactor-credentials -n keyfactor --from-file credentials.yaml
```

3. Update Keyfactor Proxy helm's values `helm-values.yaml` to install Keyfactor-K8S-Proxy

```Yaml
# Number of replication for Keyfactor-Proxy
replicaCount: 1
keyfactor:
  # Name of kubernetes secret contains credentials.yaml
  secretName: keyfactor-credentials
  # Config name mapping Keyfactor's Custom Metadata, turn off field by remove field
  # Pattern: [Istio Metadata Field] : [Keyfactor Metadata Name]
  # Supported Istio's Metadata: ClusterID, ServiceName, PodName, PodIP, PodNamespace, TrustDomain
  metadataMapping:
    ClusterID: Cluster
    ServiceName: Service
    PodName: PodName
    PodIP: PodIP
    PodNamespace: PodNamespace
    TrustDomain: TrustDomain # Value Example: cluster.local
  # Enable auto provisioning TLS's client certificates
  # Using for secure connection between Istio <> Keyfactor K8S Agent
  enableAutoProvisioningIstioCert: true
  istioNamespace: istio-system # Namespace to install Istio
  istioSecretName: custom-ca-tls # Name of secret contains TLS Client Certs
```

> Note: Keyfactor K8S Proxy will auto provisioning TLS Client Certificates for Istio Config 4. Install Keyfactor K8S Proxy via Helm 3

```bash
helm install keyfactor-k8s -n keyfactor ./release/keyfactor-k8s-0.0.1-rc.tgz -f helm-values.yaml --wait
```

## Test Keyfactor Integrate With Kubernetes - Keyfactor Certificate Signer

1. Create CSR (Certificate Signing Request)
   The following scripts show how to generate PKI private key and CSR

```bash
openssl genrsa -out keyfactor.key 2048
openssl req -new -key keyfactor.key -out keyfactor.csr
```

2. Create `csr-example.yaml`

```Yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: <REPLACE_CSR_NAME>
  annotations:
    ServiceName: "ServiceName1234" # Example metadata
    ClusterID: "ClusterID123" # Example metadata
    TrustDomain: "cluster.local" # Example metadata
    PodName: "ABC/XYZ" # Example metadata
    PodNamespace: "Namespace" # Example metadata
    PodIP: "192.168.3.2" # Example metadata
spec:
  request: <REPLACE_CSR_HERE>
  usages:
    - client auth
    - server auth
  signerName: "keyfactor.com/<REPLACE_ANY_NAME>"
```

3. Submit CSR to Kubernetes by CLI

```bash
kubectl apply -f ./csr-example.yaml
```

4. Approve CSR

```bash
kubectl certificate approve <REPLACE_CSR_NAME>
```

5. Check CSR status

```bash
kubectl get csr
```

![Example](https://github.com/thedemodrive/keyfactor-istio/raw/master/img/CSR%20Status.png)

> Note: The condition of CSR should be Approved,Issued

6. Show detail signed certificate

```bash
kubectl get csr <REPLACE_CSR_NAME> -o=jsonpath='{.status.certificate}'
```

> Note: Certificate is encoded by Base64
