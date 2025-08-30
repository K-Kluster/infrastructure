# Kasia GitOps Infrastructure

A declarative infrastructure repository that defines the entire Kasia platform and applications (Kaspa nodes, indexers, and supporting services) using [GitOps](https://www.gitops.tech/) principles. Every change to the cluster is described as code and managed through [Argo CD](https://argo-cd.readthedocs.io/).

---

## Why GitOps?

- **Single source of truth** — Kubernetes objects, Helm values, and secrets (encrypted with SOPS) live in Git, giving a full audit trail.
- **Declarative & repeatable** — Desired state is captured in manifests; apply the same state to any cluster.
- **Automated reconciliation** — Argo CD continuously compares the cluster to Git and converges them.
- **Safe collaboration** — Pull requests, review, and history help avoid configuration drift and mistakes.

---

## Repository Overview

```text
infrastructure/
├── app/                     # Application layer (Kaspa node & indexer)
│   ├── indexer/             # Base + overlays for mainnet, next, testnet
│   └── node/                # Base + overlays for mainnet and testnet
└── plateform/               # Cluster-wide services
    ├── argocd/              # Argo CD installation & configuration
    ├── cert-manager/        # Certificate management with Cloudflare DNS
    ├── longhorn/            # Persistent storage (dashboard, auth)
    ├── system/              # Misc. system addons (e.g., reflector)
    └── traefik/             # Ingress controller & TLS store
```

### Key Technologies

| Area              | Tooling / Notes                                                                                          |
| ----------------- | -------------------------------------------------------------------------------------------------------- |
| GitOps controller | Argo CD using the “app-of-apps” pattern                                                                  |
| Configuration     | Native Kubernetes manifests + Kustomize overlays                                                         |
| Secret management | [KSOPS](https://github.com/viaduct-ai/kustomize-sops) with [SOPS](https://github.com/mozilla/sops) & AGE |
| Ingress & routing | Traefik with custom dashboards and middleware                                                            |
| Certificates      | cert-manager issuing wildcard certs via Cloudflare DNS                                                   |
| Storage           | Longhorn distributed block storage                                                                       |
| Workloads         | Kaspa `kaspad` nodes and Kasia indexers for mainnet/testnet                                              |

---

## Architecture

- **App-of-apps** — `plateform/plateform.yaml` and `app/app.yaml` are Argo CD `Application` objects that bootstrap the rest of the tree.
- **Kustomize overlays** — `base/` contains common manifests; environment overlays (`mainnet`, `testnet`, `next`) apply patches.
- **Secrets via KSOPS** — `*.enc.yaml` files are encrypted with SOPS/AGE and rendered through KSOPS at deploy time.

---

## Getting Started

### Prerequisites

- Kubernetes cluster (v1.24+) with cluster-admin access
- `kubectl`, `kustomize` (or `kubectl kustomize`), and the Argo CD CLI
- [SOPS](https://github.com/mozilla/sops) and an AGE key with access to the encrypted secrets
- Optional: `helm`, `ksops`, and `argocd` CLI installed locally

### 1) Clone and configure secrets

```bash
git clone https://github.com/K-Kluster/infrastructure.git
cd infrastructure

# Point SOPS to your AGE private key
export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt
```

Encrypted files are under `plateform/**/**/*.enc.yaml` and `plateform/argocd/github-repository.enc.yaml` (paths may vary by overlay).
Ensure your AGE key can decrypt them, or replace with your own.

### 2) Bootstrap Argo CD

Apply the Argo CD installation and its config (including KSOPS):

```bash
kubectl apply -k plateform/argocd
```

After the pods are ready, log in (either expose the service or port-forward):

```bash
argocd login <server>
# or, for a quick local session:
# kubectl -n argocd port-forward svc/argocd-server 8080:443
# argocd login localhost:8080 --insecure
```

### 3) Bootstrap the platform

Create the root `Application` that manages cluster services:

```bash
kubectl apply -f plateform/plateform.yaml
```

Then sync the **plateform** application in the Argo CD UI/CLI.
This deploys Traefik, cert-manager, Longhorn, and system components.

### 4) Deploy Kasia applications

Apply the application-level root:

```bash
kubectl apply -f app/app.yaml
```

Argo CD will create:

- **node** — Kaspa nodes with mainnet and testnet overlays
- **indexer** — Kasia indexer StatefulSets, each pointed at the corresponding Kaspa node

### 5) Customize environments

Use:

- `app/node/{mainnet,testnet}` and
- `app/indexer/{mainnet,next,testnet}`

for overlay-specific patches (service names, URLs, resources, etc.).

Update images or configuration by editing the respective manifest and committing to `main`.

### 6) Ongoing operations

- Argo CD continually reconciles the cluster to this repository.
- **Secrets:** update `*.enc.yaml` with `sops` and commit.
- Add new services under `plateform/` or `app/` and reference them from the appropriate `kustomization.yaml`.

---

## Contributing

1. Fork the repo and create a feature branch.
2. Modify manifests or overlays. Use `sops` for any secret changes.
3. Validate locally with `kustomize build <path>`.
4. Open a pull request describing the change.
5. After merge, Argo CD syncs the cluster automatically.

---

## License

Distributed under the ISC License. See `LICENSE` for details.

---

Happy GitOps! 🚀
