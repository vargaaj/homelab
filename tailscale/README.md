# Tailscale Access

This directory documents the Tailscale pieces used to expose selected Kubernetes Services to the private tailnet.

The goal is to reach MLflow and Qdrant from trusted tailnet devices without opening public ingress, LAN NodePorts, or router port forwards.

## Files

| File | Purpose |
| --- | --- |
| `tailnet-policy-snippet.hujson` | Tailnet policy snippet for the Kubernetes Operator tags and grants. |
| `../mlops/tailscale-services.yaml` | Kubernetes LoadBalancer Services handled by the Tailscale Operator. |

## Tailnet Policy

In the Tailscale admin console:

1. Make sure MagicDNS is enabled.
2. Merge `tailnet-policy-snippet.hujson` into the tailnet policy.
3. Review the `grants` section before applying it.

The included snippet allows every tailnet member to reach resources tagged `tag:k8s`. That is simple for a first homelab version. Tighten it later to a specific user, group, or device tag if the tailnet grows.

## OAuth Client

Create an OAuth client for the Kubernetes Operator with these write scopes:

- Devices Core
- Auth Keys
- Services

Tag the OAuth client with:

```text
tag:k8s-operator
```

Do not commit the OAuth client secret. Pass it to Helm during operator install or store it in your preferred secret manager.

## Install The Kubernetes Operator

Run this from a machine with admin access to the k3s cluster:

```bash
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm repo update

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

Check the operator:

```bash
sudo kubectl get pods -n tailscale
```

You should also see a `tailscale-operator` machine in the Tailscale admin console.

## Exposed Services

The MLOps manifests create two Tailscale LoadBalancer Services:

| Service | Port | Target |
| --- | --- | --- |
| `mlflow-tailnet` | `5000` | MLflow tracking server |
| `qdrant-tailnet` | `6333` | Qdrant HTTP API |

Check their status with:

```bash
sudo kubectl get svc -n mlops
```

Expected MagicDNS-style names are:

```text
mlflow.<your-tailnet>.ts.net
qdrant.<your-tailnet>.ts.net
```

## Troubleshooting

Check the operator Pod:

```bash
sudo kubectl get pods -n tailscale
sudo kubectl logs -n tailscale deploy/operator
```

Check the tailnet Services:

```bash
sudo kubectl describe svc mlflow-tailnet -n mlops
sudo kubectl describe svc qdrant-tailnet -n mlops
```

If hostnames do not appear, check the OAuth client scopes, the operator tag, the tailnet policy, and whether MagicDNS is enabled.

## Security Notes

- Tailnet exposure is private, but it is still network exposure. Keep Qdrant API keys secret.
- The sample policy is intentionally broad for an early homelab. Narrow `src` in the grants section when needed.
- Do not commit OAuth client secrets, generated auth keys, kubeconfigs, or real Kubernetes Secret manifests.
