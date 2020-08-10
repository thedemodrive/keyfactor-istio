# Setup guidelines

![Network Architecture](https://github.com/thedemodrive/keyfactor-istio/raw/master/Screen%20Shot%202020-08-07%20at%202.36.06%20PM.png)
`
## Prerequisite

- These steps require you to have a cluster running a compatible version of Kubernetes. You can use any supported platform, for example [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) or others specified by the [platform-specific setup instructions](https://istio.io/docs/setup/platform-setup/).
- Require Helm 3 to install Keyfactor-Proxy. [Helm 3 Instruction](https://helm.sh/docs/intro/install/)
- Download & extract release bundle: [**Click here to download release bundle v2.alpha.1 **](https://github.com/thedemodrive/keyfactor-istio/releases/download/v2.alpha.2/v2alpha2.zip).
- cd `./v2alpha1`
- Add the `istioctl` client at `./release/istioctl-*` to your path (Linux or macOS or Windows):
  - OSX: `istioctl-osx`
  - Linux: `istioctl-linux-amd64`
  - Windows: `istioctl-win.exe`

## Install Keyfactor-Proxy
1. Update file `credentials.yaml`
```YAML
# Keyfactor Endpoint
endPoint: ""

# Name of certificate authorization
caName: ""

# Using for authentication header
authToken: ""

# API path for enroll new certificate from Keyfactor
enrollPath: "KeyfactorAPI/Enrollment/CSR"

# Certificate Template for enroll the new one: Default is Istio
caTemplate: "Istio"

# ApiKey from Api Setting, enroll certificate for Istio
appKey: ""

# ApiKey for provisioning TLS server and client certificates
provisioningAppKey: ""

# CA Template for provisioning TLS server and client certificates
provisioningTemplate: "Istio"
```
2. Create kubernetes secrets `keyfactor-secret` contains Keyfactor's credentials

```bash
kubectl create namespace keyfactor
kubectl create secret generic keyfactor-secret -n keyfactor --from-file=./credentials.yaml
```
3. Update helm's values `proxy-config.yaml` to install Keyfactor-Proxy
```Yaml
# Number of replication for Keyfactor-Proxy
replicaCount: 1
keyfactor:
  # Name of kubernetes secret contains credentials.yaml
  secretName: "keyfactor-secret"
  # Config name mapping Keyfactor's Custom Metadata, turn off field by remove item
  metadataMapping:
    # Name of cluster
    - name: ClusterID
      fieldName: Cluster # Name of Keyfactor's metadata field
    # Name of service
    - name: Service
      fieldName: Service # Name of Keyfactor's metadata field
    # Name of Pod
    - name: PodName
      fieldName: PodName # Name of Keyfactor's metadata field
    # Pod IP
    - name: PodIP
      fieldName: PodIP # Name of Keyfactor's metadata field
    # Eg: cluster.local
    - name: TrustDomain
      fieldName: TrustDomain # Name of Keyfactor's metadata field
    # Namespace of pod
    - name: PodNamespace
      fieldName: PodNamespace # Name of Keyfactor's metadata field
```
4. Install Keyfactor-Proxy via Helm 3

```bash
helm install k -n keyfactor --values proxy-config.yaml ./release/keyfactor-k8s-0.1.0.tgz 
```
`NOTE: after install keyfactor-proxy by helm, **the address of Keyfactor-Proxy** will be print in console. It's used for CUSTOM_CA_ADDR in istio-config.yaml`
![Helm install successful console](https://github.com/thedemodrive/keyfactor-istio/raw/master/helm-console.png)
## Install Istio

1. Get TLS Client Cert from Keyfactor-Proxy and prepare for Istio

```
# Get name of first pod
export POD_NAME=$(kubectl get pods --namespace keyfactor -l "app.kubernetes.io/name=keyfactor-k8s" -o jsonpath="{.items[0].metadata.name}")

kubectl cp keyfactor/$POD_NAME:certs/client.crt custom-ca.crt
kubectl cp keyfactor/$POD_NAME:certs/client.key custom-ca.key
kubectl cp keyfactor/$POD_NAME:certs/root-cert.pem root-cert.pem

kubectl create namespace istio-system
# Configure certificates for Istio
kubectl create secret generic cacerts -n istio-system --from-file ./root-cert.pem --from-file ./custom-ca.crt --from-file ./custom-ca.key
```
2. Update `istio-config.yaml`, set ENV `CUSTOM_CA_ADDR` of Pilot Component
```Yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
spec:
  hub: thedemodrive
  tag: v2.alpha.2
  installPackagePath: "charts"
  profile: "demo"
  components:
    pilot:
      enabled: true
      k8s:
        env:
          - name: CUSTOM_CA_ADDR
            # The address of Keyfactor-Proxy, It's printed on console after installing Keyfactor-Proxy
            value: "k-keyfactor-k8s.keyfactor.svc.cluster.local:8932"
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
  - [ ] Create and complete the pull request based on Istio's feedbacks
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
