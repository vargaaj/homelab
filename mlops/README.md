# MLOps Stack

This directory contains the Kubernetes manifests for the homelab MLOps stack.

The stack is intentionally small and single-node friendly:

- k3s provides lightweight Kubernetes.
- MLflow tracks experiments, params, metrics, and artifacts.
- SQLite stores MLflow metadata on a persistent volume.
- Qdrant stores vector embeddings and payload metadata for RAG workflows.
- Tailscale LoadBalancer Services expose MLflow and Qdrant to the tailnet.

## Manifests

| File | Purpose |
| --- | --- |
| `namespace.yaml` | Creates the `mlops` namespace. |
| `mlflow.yaml` | Creates MLflow PVCs, Deployment, and internal ClusterIP Service. |
| `qdrant.yaml` | Creates the Qdrant StatefulSet, PVC template, and internal ClusterIP Service. |
| `tailscale-services.yaml` | Creates Tailscale LoadBalancer Services for MLflow and Qdrant. |
| `kustomization.yaml` | Bundles the manifests for `kubectl apply -k`. |

## Assumptions

Run these commands from the Kubernetes node or from a machine with admin access to the k3s cluster. In this homelab, Proxmox is the bare-metal OS, so k3s should run inside an Ubuntu VM or another Kubernetes guest unless you intentionally install Kubernetes directly on the Proxmox host.

The Tailscale Kubernetes Operator should be installed before applying this stack. See [../tailscale/README.md](../tailscale/README.md).

## 1. Install k3s

On the Ubuntu Kubernetes guest:

```bash
curl -sfL https://get.k3s.io | sh -
```

Check the node and storage class:

```bash
sudo kubectl get nodes
sudo kubectl get storageclass
```

k3s includes `kubectl` and a default local-path storage provisioner, which is enough for this first single-node setup.

## 2. Install Helm

Check whether Helm is already available:

```bash
helm version
```

If it is missing, install Helm before installing the Tailscale Kubernetes Operator.

## 3. Create The Qdrant API Key Secret

Generate a long random key:

```bash
openssl rand -hex 32
```

Create the namespace and Secret:

```bash
sudo kubectl create namespace mlops --dry-run=client -o yaml | sudo kubectl apply -f -

sudo kubectl create secret generic qdrant-api-key \
  --namespace mlops \
  --from-literal api-key="<generated-key>"
```

Keep the generated key somewhere safe. Laptop-side clients will need it as `QDRANT_API_KEY`.

## 4. Deploy MLflow And Qdrant

From the repo root:

```bash
sudo kubectl apply -k mlops
```

Wait for workloads:

```bash
sudo kubectl rollout status deployment/mlflow -n mlops
sudo kubectl rollout status statefulset/qdrant -n mlops
```

Check the created resources:

```bash
sudo kubectl get pods,pvc,svc -n mlops
```

The `mlflow-tailnet` and `qdrant-tailnet` Services should eventually show Tailscale hostnames or addresses.

## 5. Find Tailnet Service Names

```bash
sudo kubectl get svc -n mlops
```

Look for the external information on:

- `mlflow-tailnet`
- `qdrant-tailnet`

They should normally resolve through MagicDNS as names like:

```text
mlflow.<your-tailnet>.ts.net
qdrant.<your-tailnet>.ts.net
```


Set client environment variables:

```bash
export MLFLOW_TRACKING_URI="http://mlflow.<your-tailnet>.ts.net:5000"
export QDRANT_URL="http://qdrant.<your-tailnet>.ts.net:6333"
export QDRANT_API_KEY="<generated-key>"
```

Check MLflow:

```bash
curl "$MLFLOW_TRACKING_URI/version"
```

Check that Qdrant rejects unauthenticated access:

```bash
curl "$QDRANT_URL"
```

Check that Qdrant accepts authenticated access:

```bash
curl -H "api-key: $QDRANT_API_KEY" "$QDRANT_URL"
```

## Useful Checks

```bash
sudo kubectl get all -n mlops
sudo kubectl get pvc -n mlops
sudo kubectl describe svc mlflow-tailnet -n mlops
sudo kubectl describe svc qdrant-tailnet -n mlops
sudo kubectl logs deployment/mlflow -n mlops
sudo kubectl logs statefulset/qdrant -n mlops
```

## Teardown

This removes the running resources:

```bash
sudo kubectl delete -k mlops
```

Stateful data may remain in persistent volumes depending on the storage reclaim policy. Do not delete PVCs or PVs unless you are intentionally deleting MLflow or Qdrant data.

## Notes

- SQLite is fine for this single-user first version, but it is not the right backend for heavy concurrent MLflow usage.
- Qdrant API keys should be sent over trusted private networks. In this setup, access is through Tailscale.
- If Postgres or object storage is added later, the laptop-side RAG flow can stay mostly the same.
