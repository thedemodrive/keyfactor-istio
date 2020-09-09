# Setup guidelines

![Network Architecture](https://github.com/thedemodrive/keyfactor-istio/raw/master/Screen%20Shot%202020-08-07%20at%202.36.06%20PM.png)
`
## Prerequisite

- These steps require you to have a cluster running a compatible version of Kubernetes. You can use any supported platform, for example [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) or others specified by the [platform-specific setup instructions](https://istio.io/docs/setup/platform-setup/).
- Require Kubernetes Version >= 1.18
- Require Helm 3 to install Keyfactor-Proxy. [Helm 3 Instruction](https://helm.sh/docs/intro/install/)
- Download & extract release bundle: [**Click here to download release bundle keyfactor-v2-beta **](https://github.com/thedemodrive/keyfactor-istio/releases/download/keyfactor-v2-beta/keyfactor-v2-beta.zip).
- Add the `istioctl` client at `./release/istioctl-*` to your path (Linux or macOS or Windows):
  - OSX: `istioctl-osx`
  - Linux: `istioctl-linux-amd64`
  - Windows: `istioctl-win.exe`

## Install Keyfactor-Proxy
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
> Note: Keyfactor K8S Proxy will auto provisioning TLS Client Certificates for Istio Config
4. Install Keyfactor K8S Proxy via Helm 3

```bash
helm install keyfactor-k8s -n keyfactor ./release/keyfactor-k8s-0.0.1-rc.tgz -f helm-values.yaml --wait
```
> Note: After keyfactor-k8s-proxy is installed by helm, Istio's configuration yaml file will be printed.

![Helm install successful console](https://github.com/thedemodrive/keyfactor-istio/raw/master/Helm%20Result.png)

## Test Keyfactor Integrate With Kubernetes - Keyfactor Certificate Signer
1. Create CSR (Certificate Signing Request)

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
![Example](https://github.com/thedemodrive/keyfactor-istio/raw/master/CSR%20Status.png)
> Note: The condition of CSR should be Approved,Issued

6. Show detail signed certificate
```bash
kubectl get csr <REPLACE_CSR_NAME> -o=jsonpath='{.status.certificate}'
```
> Note: Certificate is encoded by Base64

## Install Istio

1. Update `istio-config.yaml` by using printed values on above step.
```Yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
spec:
  hub: thedemodrive
  tag: keyfactor-v2-beta2
  installPackagePath: "charts"
  profile: "demo"
  values:
    pilot:
      secretVolumes:
        - name: <SECRET_NAME>
          secretName: <SECRET_NAME>
          mountPath: /etc/istio/<SECRET_NAME>
  meshConfig:
    ca:
      # Use istiod_side to specify CA Server integrate to Istiod side or Agent side
      istiodSide: true
      # Address of the CA server implementing the Istio CA gRPC API.
      # Can be IP address or a fully qualified DNS name with port
      # Eg: custom-ca.default.svc.cluster.local:8932, 192.168.23.2:9000
      # If specified, Istio will authorize and forward the CSRs from the workloads to the specified external CA
      # using the Istio CA gRPC API.
      address: <KEYFACTOR_K8S_ADDRESS>:<PORT>
      # timeout for forward CSR requests from Istiod to External CA
      # Default: 30s
      requestTimeout: 30s
      # Default TLS mode is MUTUAL, It's require to specific TLS client certificate files.
      # According sistiodSide = true, TLS files could be mounted from Kubernetes Secret
      # configurable via values.pilot.secretVolumes
      tlsSettings:
        mode: MUTUAL # Supported values: DISABLE, MUTUAL
        clientCertificate: "/etc/istio/<SECRET_NAME>/client-cert.pem"
        privateKey: "/etc/istio/<SECRET_NAME>/client-key.pem"
        caCertificates: "/etc/istio/<SECRET_NAME>/cacert.pem"
        sni: <KEYFACTOR_K8S_ADDRESS>
        subjectAltNames: []
```
3. Install Istio with `istio-config.yaml`

```Bash
istioctl install -f istio-config.yaml
```

## Setup example Microservices

Deploy Book-Info microservice example of Istio ([references](https://istio.io/docs/examples/bookinfo/))
![Book Info Sample](https://istio.io/latest/docs/examples/bookinfo/withistio.svg)

- Turn on Istio auto-inject for namespace **default**

  ```bash
  kubectl label namespace default istio-injection=enabled
  ```

- Deploy an example of istio ([Book-Info](https://istio.io/docs/examples/bookinfo/))

  ```bash
  kubectl apply -f ./samples/bookinfo/platform/kube/bookinfo.yaml
  ```

- Configure a gateway for the Book-Info sample

  ```bash
  kubectl apply -f ./samples/bookinfo/networking/bookinfo-gateway.yaml
  ```

- Configure mTLS destination rules

  ```bash
  kubectl apply -f ./samples/bookinfo/networking/destination-rule-all-mtls.yaml
  ```

- Lock down mutual TLS for the entire mesh

  ```bash
  kubectl apply -f ./samples/peer-authentication.yaml
  ```

## HOW TO VERIFY THE TRAFFIC IS USING MUTUAL TLS ENCRYPTION

Lock down mutual TLS for the entire mesh

```bash
kubectl apply -f ./samples/peer-authentication.yaml
```

### Create the namespace "insidemesh" and deploy a sleep container **with sidecars**

```bash
kubectl create ns insidemesh
kubectl label namespace insidemesh istio-injection=enabled
kubectl apply -f ./samples/sleep/sleep.yaml -n insidemesh
```

Verify the setup by sending an http request (using curl command) from sleep pod (namespace: insidemesh) to productpage.default:9080:

1. To check if the certificate get from productpage.default is issued by KeyfactorCA

```bash
kubectl exec $(kubectl get pod -l app=sleep -n insidemesh -o jsonpath={.items..metadata.name}) -c sleep -n insidemesh -- openssl s_client -showcerts -connect productpage.default:9080
```

2. Request using curl

```bash
kubectl exec $(kubectl get pod -l app=sleep -n insidemesh -o jsonpath={.items..metadata.name}) -c sleep -n insidemesh -- curl http://productpage.default:9080 -s -o /dev/null -w "sleep.insidemesh to http://productpage.default:9080: -> HTTP_STATUS: %{http_code}\n"
```

> Note: every workload **deployed with sidecar** can access Book Info services (HTTP_STATUS = 200)

### Create another namespace "outsidemesh" and deploy a sleep container **without a sidecar**

```bash
kubectl create ns outsidemesh
kubectl apply -f samples/sleep/sleep.yaml -n outsidemesh
```

Verify the setup by sending an http request (using curl command) from sleep pod (namespace: outsidemesh) to productpage.default:9080:

```bash
kubectl exec $(kubectl get pod -l app=sleep -n outsidemesh -o jsonpath={.items..metadata.name}) -c sleep -n outsidemesh -- curl http://productpage.default:9080 -s -o /dev/null -w "sleep.outsidemesh to http://productpage.default:9080: -> HTTP_STATUS: %{http_code}\n"
```

> Note: every workload **deployed without sidecar** cannot access Book Info services (HTTP_STATUS = 000)

# Tasks Checklist:
1. CSR Forwarder (Istio modification):
  - [x] Integrate Keyfactor with Istiod
  - [x] Support mTLS secure connection between IstioD and Keyfactor Proxy
  - [x] Support configure Keyfactor Proxy address
  - [x] Support configure TLS Certs : client cert, client private key, CA cert.
  - [x] Hardness Unit Test
  - [x] Create and complete the pull request based on Istio's feedbacks
  - [ ] Get approval
2. Code change to include Pod's Metadata (Istio modification): 2 fields: PodName and PodIP
  - [x] Implement
  - [x] Create pull request for supporting pod metadata into Certificate Signing API (github.com/istio/api)
  - [ ] Get approval
3. Keyfactor Proxy:
  - [x] Enroll Workload Certficate API for Istio integration.
  - [x] Auto provisioning Server TLS Cert and Client TLS Cert (use for Istio) based on Hostname in kubernetes
  - [x] Certificate Peer Verify (mTLS)
  - [x] Keyfactor Metadata: TrustDomain, ServiceName, PodNamespace (parsed from CSR PEM)
  - [x] Configure custom metadata mapping
  - [x] Configure Keyfactor Credentials via Kubernetes Secret
  - [x] Deployment: Kubernetes Manifest, Helm Chart, Docker
  - [x] Service Health checking and Client TLS Cert checking.
  - [ ] Hardness Unit Test
  - [ ] Auto provisiong Client TLS certs for Istio via K8S API (currently, Certs are manually provisioned by users)
