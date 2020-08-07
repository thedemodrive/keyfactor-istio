# keyfactor-istio

# Roadmap:
1. Istio Modification:
  [x] Integrate Keyfactor with Istiod
  [x] Support mTLS secure connection between IstioD and Keyfactor Proxy
  [x] Support configure Keyfactor Proxy address
  [x] Support configure TLS Certs : client cert, client private key, CA cert.
  [x] Create pull request for supporting pod metadata into Certificate Signing API (github.com/istio/api)
  [ ] Code changes to include Pod's metadata: PodName, PodIP (waiting for the approve from Istio on istio/api)
  [ ] Hardness Unit Test
  [ ] Create and complete the pull request based on Istio's feedbacks
  
2. Keyfactor Proxy:
  [x] Enroll Workload Certficate API for Istio integration.
  [x] Auto provisioning Server TLS Cert and Client TLS Cert (use for Istio) based on Hostname in kubernetes
  [x] Certificate Peer Verify (mTLS)
  [x] Keyfactor Metadata: TrustDomain, ServiceName, PodNamespace (parsed from CSR PEM)
  [x] Configure custom metadata mapping
  [x] Configure Keyfactor Credentials via Kubernetes Secret
  [x] Deployment: Kubernetes Manifest, Helm Chart, Docker
  [x] Service Health checking and Client TLS Cert checking.
  [ ] Hardness Unit Test
  [ ] Auto provisiong Client TLS certs for Istio via K8S API (currently, Certs are manually provisioned by users)
