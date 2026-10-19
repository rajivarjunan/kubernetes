# K3s PKI + log stack (8 workload pods)

Designed for a single-node K3s lab. Workload count: step-ca 1; cert-manager controller, webhook and cainjector 3; trust-manager 1; Loki 1; Grafana 1; Alloy 1. Total = 8. K3s system pods and existing Headlamp are additional.

## Security / prerequisites
- Keep the root CA private key on the encrypted USB. Never put it in YAML or Kubernetes.
- Create a matching root CA certificate and have the offline root sign an intermediate CA CSR first.
- Enable K3s Secret encryption at rest before creating the step-ca secret. A Kubernetes Secret is not automatically encrypted at rest.
- Confirm storage: kubectl get storageclass; kubectl get pods -A
- Use this as a lab, not a production HA CA. The intermediate signing key is online in the cluster.

## A. Install certificate controllers (3+1 pods)
    helm upgrade --install cert-manager oci://quay.io/jetstack/charts/cert-manager --version v1.21.2 --namespace cert-manager --create-namespace --set crds.enabled=true --set replicaCount=1 --set webhook.replicaCount=1 --set cainjector.replicaCount=1 -f helm-values/cert-manager-values.yaml
    helm upgrade --install trust-manager jetstack/trust-manager --version 0.25.0 --namespace cert-manager -f helm-values/trust-manager-values.yaml
    kubectl get pods -n cert-manager

## B. Create step-ca secret (do not commit the files)
Stage these files locally with restrictive permissions: rootCA.crt.pem, intermediateCA.crt.pem, intermediateCA.key.pem, and password (for decrypting the intermediate key).
    chmod 600 ./pki-input/intermediateCA.key.pem ./pki-input/password
    kubectl create namespace pki
    kubectl -n pki create secret generic step-ca-secrets --from-file=root_ca.crt=./pki-input/rootCA.crt.pem --from-file=intermediate_ca.crt=./pki-input/intermediateCA.crt.pem --from-file=intermediate_ca_key=./pki-input/intermediateCA.key.pem --from-file=password=./pki-input/password
Then:
    kubectl apply -f manifests/10-step-ca.yaml
    kubectl -n pki rollout status statefulset/step-ca

This creates one step-ca pod with a 2Gi database PVC and ClusterIP service only. It expects service DNS step-ca.pki.svc.cluster.local.

## C. Issue a test certificate
Edit manifests/20-pki-issuer-and-test-certificate.yaml:
1. Replace REPLACE_WITH_BASE64_ROOT_CA_CERT with the base64 encoding of the public root certificate, one line.
2. Replace example.home.arpa with a DNS name you control and configure Traefik/DNS so HTTP-01 validation can reach it from inside the cluster.
3. Replace the placeholder email.
Then apply:
    kubectl apply -f manifests/20-pki-issuer-and-test-certificate.yaml
    kubectl -n pki describe certificate pki-test

If HTTP-01 cannot reach the hostname, stop and configure a valid DNS-01 solver or a suitable Smallstep issuer integration. Do not turn off TLS verification.

## D. Distribute trust
    kubectl apply -f manifests/30-trust-bundle.yaml
Label any namespace that should receive the public bundle:
    kubectl label namespace pki pki-trust=enabled --overwrite
The bundle is public CA data only. Mount the resulting ConfigMap in apps that need to trust your CA.

## E. Install logs (3 pods)
Inspect chart versions first and record/pin the versions used:
    helm repo add grafana-community https://grafana-community.github.io/helm-charts
    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo update
    helm search repo grafana-community/loki
    helm search repo grafana/grafana
    helm search repo grafana/alloy

Install using the values:
    helm upgrade --install loki grafana-community/loki --namespace logging --create-namespace -f helm-values/loki-values.yaml
    kubectl create secret generic grafana-admin -n logging --from-literal=admin-user=admin --from-literal=admin-password='CHANGE_THIS_TO_A_LONG_UNIQUE_PASSWORD'
    helm upgrade --install grafana grafana/grafana --namespace logging -f helm-values/grafana-values.yaml
    helm upgrade --install alloy grafana/alloy --namespace logging -f helm-values/alloy-values.yaml
    kubectl get pods,pvc -n logging

Grafana is ClusterIP-only. View it locally:
    kubectl -n logging port-forward svc/grafana 3000:80
Browse to http://127.0.0.1:3000. Loki has no built-in auth; keep it internal. Alloy collects Kubernetes pod logs. Host journal and K3s API audit logs need separate configuration.

## Validation
    kubectl get pods -A
    kubectl get certificates -A
    kubectl -n logging logs daemonset/alloy

Expected workload pods: 8. Existing Headlamp and K3s system pods are not included.
