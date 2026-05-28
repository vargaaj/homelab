# Homelab MLflow + Qdrant on k3s

This repo contains a small homelab MLOps stack for a single Ubuntu server:

- k3s provides lightweight Kubernetes.
- MLflow tracks experiments, params, metrics, and artifacts.
- SQLite stores MLflow metadata on a persistent volume.
- Qdrant stores vector embeddings and payload metadata for RAG.
- Tailscale exposes MLflow and Qdrant to your tailnet without opening LAN NodePorts.

Local laptop still does the local-doc work: extract text, chunk documents, embed chunks, call Qdrant, and run the local LLM. The homelab stores the durable MLflow and vector database state.


## How The Pieces Fit

The manifests live under `mlops`.

`namespace.yaml` creates the `mlops` namespace so these resources are grouped together.

`mlflow.yaml` creates:

- `mlflow-backend`, a 1Gi PVC for the SQLite database at `/mlflow/backend/mlflow.db`.
- `mlflow-artifacts`, a 10Gi PVC for MLflow artifacts at `/mlflow/artifacts`.
- A single-replica MLflow Deployment using `ghcr.io/mlflow/mlflow:v3.12.0`.
- An internal `ClusterIP` Service named `mlflow`.

`qdrant.yaml` creates:

- A single-replica Qdrant StatefulSet using `qdrant/qdrant:v1.17.1`.
- A 20Gi persistent volume for `/qdrant/storage`.
- A required Secret reference named `qdrant-api-key`.
- An internal `ClusterIP` Service named `qdrant`.

`tailscale-services.yaml` creates:

- `mlflow-tailnet`, a Tailscale `LoadBalancer` Service for MLflow on port `5000`.
- `qdrant-tailnet`, a Tailscale `LoadBalancer` Service for Qdrant on port `6333`.

The internal `ClusterIP` Services are for Kubernetes-internal traffic. The Tailscale `LoadBalancer` Services are what your laptop will use over the tailnet.

## Bring-Up Steps

Run these on the Ubuntu homelab server unless noted otherwise.

### 1. Install k3s

```bash
# Download and run the official k3s installer.
# This installs the k3s server, starts Kubernetes, and installs kubectl for k3s.
curl -sfL https://get.k3s.io | sh -
```

Check that the node is ready:

```bash
# Ask Kubernetes to list cluster nodes.
# You want to see the Ubuntu server in Ready state.
sudo kubectl get nodes

# Ask Kubernetes to list available storage classes.
# k3s should show a local-path storage class for simple local PVC storage.
sudo kubectl get storageclass
```

k3s includes `kubectl` and a default local storage provisioner, so the PVCs in this repo should work on a single-node homelab without extra storage setup.

### 2. Install Helm

Install Helm if the server does not already have it:

```bash
# Print the installed Helm version.
# If this fails with "command not found", Helm is not installed yet.
helm version
```

If that command is missing, install Helm from the official Helm install docs, then come back here.

### 3. Prepare Tailscale Operator Access

In the Tailscale admin console:

1. Make sure MagicDNS is enabled.
2. Merge `tailscale/tailnet-policy-snippet.hujson` into your tailnet policy.
3. Create an OAuth client with these write scopes:
   - Devices Core
   - Auth Keys
   - Services
4. Tag the OAuth client with `tag:k8s-operator`.

Install the operator into k3s:

```bash
# Add the official Tailscale Helm chart repository to Helm.
helm repo add tailscale https://pkgs.tailscale.com/helmcharts

# Download the latest chart metadata from configured Helm repositories.
helm repo update

# Install the Tailscale Kubernetes Operator, or upgrade it if it already exists.
# The OAuth client ID and secret let the operator create tailnet services.
helm upgrade \
  --install \
  tailscale-operator \
  tailscale/tailscale-operator \
  --namespace=tailscale \
  --create-namespace \
  --set-string oauth.clientId="<oauth-client-id>" \
  --set-string oauth.clientSecret="<oauth-client-secret>" \
  --wait
```

Check it:

```bash
# List Pods in the tailscale namespace.
# You want the operator Pod to be Running and Ready.
sudo kubectl get pods -n tailscale
```

You should also see a `tailscale-operator` machine in the Tailscale admin console.

### 4. Create The Qdrant API Key Secret

Generate a long random key:

```bash
# Generate a 32-byte random value and print it as 64 hexadecimal characters.
# This becomes the shared Qdrant API key used by your laptop.
openssl rand -hex 32
```

Create the Kubernetes Secret:

```bash
# Generate YAML for the mlops namespace without creating it directly, then apply it.
# The pipe sends the generated YAML into kubectl apply.
# The final "-" means "read the manifest from standard input".
sudo kubectl create namespace mlops --dry-run=client -o yaml | sudo kubectl apply -f -

# Create a Kubernetes Secret named qdrant-api-key in the mlops namespace.
# Qdrant reads the api-key value from this Secret when the Pod starts.
sudo kubectl create secret generic qdrant-api-key \
  --namespace mlops \
  --from-literal api-key="<generated-key>"
```

Keep this key somewhere safe. Your laptop will need it as `QDRANT_API_KEY`.

### 5. Deploy MLflow And Qdrant

From this repo directory:

```bash
# Apply the Kustomize bundle in mlops.
# This creates or updates the MLflow, Qdrant, PVC, Service, and Tailscale Service resources.
sudo kubectl apply -k mlops
```

Wait for workloads to become ready:

```bash
# Wait until the MLflow Deployment has one available updated Pod.
sudo kubectl rollout status deployment/mlflow -n mlops

# Wait until the Qdrant StatefulSet has updated its Pod.
sudo kubectl rollout status statefulset/qdrant -n mlops
```

Check the resources:

```bash
# List Pods, PersistentVolumeClaims, and Services in the mlops namespace.
# This shows whether the workloads are running, storage is bound, and Services exist.
sudo kubectl get pods,pvc,svc -n mlops
```

The `mlflow-tailnet` and `qdrant-tailnet` Services should eventually show Tailscale hostnames or addresses in their status.

### 6. Find The Tailnet Service Names

Run:

```bash
# List Services in the mlops namespace.
# Look for external information on mlflow-tailnet and qdrant-tailnet.
sudo kubectl get svc -n mlops
```

Look at the external hostname/address for:

- `mlflow-tailnet`
- `qdrant-tailnet`

They should normally resolve through MagicDNS as something like:

```text
mlflow.<your-tailnet>.ts.net
qdrant.<your-tailnet>.ts.net
```

### 7. Test From Your Laptop

Set environment variables on the laptop:

```bash
# Tell MLflow clients where the tracking server is.
export MLFLOW_TRACKING_URI="http://mlflow.<your-tailnet>.ts.net:5000"

# Tell Qdrant clients where the vector database HTTP API is.
export QDRANT_URL="http://qdrant.<your-tailnet>.ts.net:6333"

# Give Qdrant clients the API key you created in Kubernetes.
export QDRANT_API_KEY="<generated-key>"
```

Check MLflow:

```bash
# Call MLflow's lightweight version endpoint through Tailscale.
curl "$MLFLOW_TRACKING_URI/version"
```

Check that Qdrant rejects unauthenticated access:

```bash
# Call Qdrant without the API key.
# This should fail or return an unauthorized-style response.
curl "$QDRANT_URL"
```

Check that Qdrant accepts authenticated access:

```bash
# Call Qdrant with the api-key header.
# This should return a normal Qdrant response.
curl -H "api-key: $QDRANT_API_KEY" "$QDRANT_URL"
```


## Useful Checks

```bash
# Show all common Kubernetes resources in the mlops namespace.
sudo kubectl get all -n mlops

# Show persistent storage claims and whether they are Bound.
sudo kubectl get pvc -n mlops

# Show detailed status/events for the Tailscale MLflow Service.
sudo kubectl describe svc mlflow-tailnet -n mlops

# Show detailed status/events for the Tailscale Qdrant Service.
sudo kubectl describe svc qdrant-tailnet -n mlops

# Show logs from the MLflow Deployment.
sudo kubectl logs deployment/mlflow -n mlops

# Show logs from the Qdrant StatefulSet.
sudo kubectl logs statefulset/qdrant -n mlops
```

## Teardown

This removes the running resources:

```bash
# Delete the resources created by the mlops Kustomize bundle.
# Be careful: this stops MLflow and Qdrant.
sudo kubectl delete -k mlops
```

Stateful data may remain in persistent volumes depending on the storage reclaim policy. Do not delete PVCs or PVs unless you are intentionally deleting MLflow/Qdrant data.

## Notes

- SQLite is fine for this single-user v1, but it is not the right backend for heavy concurrent MLflow usage.
- Qdrant API keys should be sent over trusted/private networks. In this setup, access is through Tailscale.
- If you later add Postgres or object storage, the laptop-side RAG flow can stay mostly the same.

