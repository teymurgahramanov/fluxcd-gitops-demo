# GitOps FluxCD Example

This repository is intended for reference purposes only. It provides a small example of how a FluxCD GitOps repository can be organized for multiple Kubernetes clusters.

## What This Shows

Flux watches this Git repository and applies the manifests for one cluster path. Each cluster points Flux to shared infrastructure, infrastructure apps, and application manifests. The base files hold common settings, environment overlays adjust dev/prod behavior, and cluster overlays add cluster-specific values such as hostnames.

In short:

1. A cluster is bootstrapped with Flux.
2. Flux reads the matching `clusters/<cluster-name>` path.
3. Infrastructure is reconciled first.
4. Infrastructure apps are reconciled after the core infrastructure.
5. Application releases are reconciled independently from infrastructure with continuous retry.

## Repository Layout

```text
gitops-fluxcd-example/
|-- apps/                    # Application HelmReleases
|-- clusters/                # Per-cluster Flux definitions
|   |-- prod-cluster-1/
|   |-- prod-cluster-2/
|   `-- dev-cluster/
`-- infrastructure/          # Shared platform components
    |-- apps/
    |-- crds/
    |-- essentials/
    `-- resources/
```

## Clusters

| Cluster | Environment |
| --- | --- |
| `prod-cluster-1` | `production` |
| `prod-cluster-2` | `production` |
| `dev-cluster` | `development` |

## Day 0

1. Create the Vault namespace, service account, token secret, and token review binding before bootstrapping Flux.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: vault
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-auth
  namespace: vault
---
apiVersion: v1
kind: Secret
metadata:
  name: vault-auth-token
  namespace: vault
  annotations:
    kubernetes.io/service-account.name: vault-auth
type: kubernetes.io/service-account-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
  namespace: vault
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: vault-auth
  namespace: vault
EOF
```

2. Configure the Kubernetes auth method in Vault using the output of this command.

```bash
echo -e "\n" && \
  echo -e "Token Reviewer JWT:\n" && \
  kubectl get secret vault-auth-token -n vault --output 'go-template={{ .data.token }}' | base64 --decode && \
  echo -e "\n" &&\
  echo -e "CA Certificate:\n" && \
  kubectl config view --raw --minify --flatten --output 'jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode
```

3. Bootstrap FluxCD and wait for reconciliation to finish.

```bash
flux bootstrap git \
  --url=https://github.com/example-org/gitops-fluxcd-example.git \
  --token-auth=true \
  --username=$FLUXCD_GIT_USERNAME \
  --password=$FLUXCD_GIT_TOKEN \
  --path=clusters/$CLUSTER_NAME
```

