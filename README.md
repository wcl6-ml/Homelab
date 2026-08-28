# Homelab: K3s + GitOps Infrastructure

A self-hosted Kubernetes homelab built to practice and demonstrate production-grade platform engineering: GitOps delivery, automated image updates, and observability, all running on two repurposed laptops.

---

## TL;DR

- **Cluster:** K3s, 2 nodes (repurposed laptops), managed declaratively via GitOps
- **Delivery:** FluxCD reconciles this repo to the cluster - no manual `kubectl apply`
- **Image updates:** Renovate watches Docker Hub for new image tags and opens a PR; merging triggers Flux to roll out the new version
- **Networking:** Services exposed publicly via Cloudflare Tunnel - no port forwarding, no open inbound ports
- **Observability:** Prometheus + Grafana (kube-prometheus-stack) for cluster and app metrics
- **Live apps:** [linkding](https://ldpi.weichunglai.com) (bookmarks), [audiobookshelf](https://audiobooks.weichunglai.com) (audiobook server)

---

## Architecture

```
   Image pushed
        │
        ▼
   Renovate (scans for new tags, opens PR)
        │  [manual review + merge]
        ▼
   Git repo (this repo, source of truth)
        │
        ▼
   FluxCD (reconciles cluster state to match repo)
        │
        ▼
   K3s cluster (2 nodes: ASUS X555, Acer Swift)
        │
        ▼
   Cloudflare Tunnel ──► Public DNS
                          ├─ ldpi.weichunglai.com       (linkding)
                          └─ audiobooks.weichunglai.com (audiobookshelf)

   kube-prometheus-stack ──► Grafana dashboards (cluster + app metrics)
```

---

## Hardware & Cluster Topology

The cluster runs on two old consumer laptops repurposed as K3s nodes:

| Node | Hardware | Role | RAM |
|---|---|---|---|
| Acer Swift | Laptop (original single-node setup) | K3s server | 4GB |
| ASUS X555 | Laptop (added later) | K3s agent | 8GB |

I originally ran everything single-node on the Acer Swift, which meant the cluster was leaning on swap under load. I added the ASUS X555 as a second node specifically to move off swap-dependent scheduling and get real experience with multi-node resource distribution, pod scheduling, and node affinity in K3s — problems that don't show up on a single-node cluster.

**On the repo's cluster naming:** the repo has two Flux-managed clusters, `staging` and `ml-projects`. Despite the name, `staging` is the actual persistent cluster running linkding and audiobookshelf,  it isn't a pre-prod stage in a promotion pipeline, it's just named for how it started. `ml-projects` was a separate, placeholder for applications with GPU on another node.

---

## GitOps Delivery (FluxCD)

Every app in `apps/` is reconciled into the cluster by Flux, the cluster state always matches what's committed to `main`. Structure:

```
apps/
├── base/          # shared manifests (deployment, service, storage, etc.)
├── staging/       # overlays for the persistent cluster (linkding, audiobookshelf)
└── ml-projects/   # overlays for the experimental cluster
clusters/
├── staging/       # Flux Kustomizations + flux-system bootstrap for this cluster
└── ml-projects/
infrastructure/
└── controllers/   # Renovate controller, deployed the same GitOps way as apps
monitoring/
└── controller/    # kube-prometheus-stack (Prometheus + Grafana)
```

Each cluster has its own `flux-system` bootstrap and its own `apps.yaml` / `infrastructure.yaml` / `monitoring.yaml` Kustomizations, so app config, infra tooling, and monitoring are all deployed and reconciled the same declarative way, nothing is applied by hand.

---

## Automated Image Updates (Renovate)

For images I build myself and push to Docker Hub, Renovate watches for new tags and opens a PR against this repo bumping the image version. I review and merge manually, Flux then picks up the change and rolls out the new image automatically. This gave me hands-on experience with the full loop of **image build → registry → automated PR → GitOps reconciliation**, which is the same pattern used for shipping model-serving containers in production.

This flow was piloted end-to-end in the `pyproject-helloworld` app (under `apps/ml-projects/`).

---

## Networking (Cloudflare Tunnel)

Both public apps are exposed through **Cloudflare Tunnel** rather than port forwarding, no inbound ports are opened on my home network. Tunnel config and DNS routing are managed declaratively per-app (`cloudflare.yaml` / `cloudflare-secret.yaml`, which are encypted using `sops`).

- `linkding` → https://ldpi.weichunglai.com
- `audiobookshelf` → https://audiobooks.weichunglai.com

---

## Observability

`kube-prometheus-stack` is deployed via Flux (`monitoring/controller/`) providing:
- Cluster-level metrics (node resource usage, pod health)
- Grafana dashboards

---

## Tech Stack

| Layer | Tool |
|---|---|
| Orchestration | K3s |
| GitOps / CD | FluxCD |
| Image automation | Renovate |
| Networking | Cloudflare Tunnel |
| Monitoring | Prometheus, Grafana (kube-prometheus-stack) |
| Manifests | Kustomize (base + per-cluster overlays) |

---

## Repo Layout

```
apps/            # application manifests (base + overlays per cluster)
clusters/        # per-cluster Flux bootstrap + top-level Kustomizations
infrastructure/  # cluster tooling deployed via GitOps (Renovate)
monitoring/      # Prometheus/Grafana stack config
scripts/         # setup scripts
```
