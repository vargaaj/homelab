# Homelab Architecture

This repository documents and deploys the homelab services running on a reused CyberPowerPC desktop. The machine runs Proxmox VE as the bare-metal operating system, with application workloads intended to run inside guests such as an Ubuntu VM running k3s.

The first workload in this repo is a small MLOps platform: MLflow for experiment tracking, Qdrant for vector storage, and Tailscale for private tailnet access without exposing LAN NodePorts or public ingress.

## Hardware

| Component | Details |
| --- | --- |
| CPU | AMD Ryzen 7 2700X, 8 cores, 3.7 GHz base, 4.35 GHz turbo, 105 W |
| Motherboard | MSI X470 Gaming Plus AM4 ATX |
| Memory | 16 GB DDR4-3000, 2 x 8 GB ADATA XPG Z1 |
| Storage | 240 GB WD Green SATA SSD and 2 TB 7200 RPM SATA HDD |
| Original GPU | XFX AMD Radeon RX 590 Fatboy 8 GB GDDR5, failed and no longer used |
| Current GPU | maxsun NVIDIA GeForce GT 710 2 GB DDR3, low-power display adapter |
| Power supply | Thermaltake 600 W 80 Plus Gold |
| Cooling | 120 mm liquid CPU cooler and six 120 mm case fans |
| Bare-metal OS | Proxmox VE |

The replacement GT 710 is mainly here for local display output. It is not expected to be useful for modern GPU compute workloads.

## System Shape

```text
Laptop / workstation
  | SSH, Git, ML clients
  | Tailscale tailnet
  v
Proxmox host
  | Ubuntu or similar Kubernetes guest
  v
k3s cluster
  |-- mlops namespace
      |-- MLflow Deployment + PVCs
      |-- Qdrant StatefulSet + PVC
      |-- Tailscale LoadBalancer Services
```

The laptop still handles local document processing and model/client work: extracting text, chunking documents, embedding chunks, calling Qdrant, and running local LLM tooling. The homelab keeps durable service state: MLflow metadata, MLflow artifacts, and Qdrant vector data.

## Repository Layout

```text
.
|-- README.md                 # Overall homelab architecture
|-- mlops/                    # Kubernetes manifests and MLOps runbook
|   |-- README.md
|   |-- kustomization.yaml
|   |-- mlflow.yaml
|   |-- namespace.yaml
|   |-- qdrant.yaml
|   `-- tailscale-services.yaml
`-- tailscale/                # Tailnet policy and Tailscale operator runbook
    |-- README.md
    `-- tailnet-policy-snippet.hujson
```

## Service Boundaries

- Proxmox owns the physical host, storage devices, and guest lifecycle.
- k3s owns Kubernetes scheduling and persistent volume claims inside the Kubernetes guest.
- MLflow stores tracking metadata and artifacts on local persistent volumes.
- Qdrant stores vector collections on local persistent storage and requires an API key Secret.
- Tailscale exposes selected Kubernetes Services only to the private tailnet.

## Runbooks

- [MLOps stack](mlops/README.md): k3s assumptions, Qdrant Secret creation, MLflow/Qdrant deployment, tests, checks, and teardown.
- [Tailscale access](tailscale/README.md): tailnet policy, OAuth client setup, Kubernetes Operator install, exposed service names, and troubleshooting.


