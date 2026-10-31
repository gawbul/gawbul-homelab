# AGENTS.md

This file provides guidance and project context for AI agents (Claude Code, Gemini / Antigravity, Cursor, etc.) working in this repository.

## Project Overview

This repository contains Infrastructure as Code (IaC) for setting up and managing a Kubernetes homelab cluster on Raspberry Pi 5 hardware. It uses **K3s** for the Kubernetes distribution, provisioned via **Ansible** on **Ubuntu Noble (24.04 LTS)**.

The project follows bare-metal Kubernetes best practices, including declarative configuration, idempotent node management, and GitOps for all in-cluster components.

## Cluster Architecture

- **Cluster Name:** `eniac`
- **Hardware:** 4x Raspberry Pi 5 (16GB RAM, 512GB microSD)
- **Domain:** `home.gawbul.io`
- **Network CIDRs:**
  - Node Network: `192.168.50.0/24` (Static IPs `.4` – `.7`)
  - Pod CIDR: `10.42.0.0/16` (K3s default)
  - Service CIDR: `10.43.0.0/16` (K3s default)

### Node Layout

| Hostname | Role | IP | Description |
| :--- | :--- | :--- | :--- |
| `eniac-node1` | Control Plane + Worker | `192.168.50.4` | API Server, Etcd, Workloads |
| `eniac-node2` | Worker | `192.168.50.5` | Workloads |
| `eniac-node3` | Worker | `192.168.50.6` | Workloads |
| `eniac-node4` | Worker | `192.168.50.7` | Workloads |

### Key Components

- **K3s:** Lightweight, ARM64-optimized Kubernetes distribution.
- **Flannel (WireGuard-native):** Secure inter-node overlay encryption.
- **Cilium:** High-performance eBPF CNI and kube-proxy replacement (deployed via Flux).
- **MetalLB:** Layer 2 LoadBalancer (deployed via Flux).
- **NFS Subdir Provisioner / CSI Driver:** Dynamic persistent storage on NAS (`192.168.50.2`, deployed via Flux).
- **Flux CD:** GitOps engine syncing manifests from the [`gawbul/gawbul-gitops`](https://github.com/gawbul/gawbul-gitops) repository.

## Tooling & Environment

Tool versions and tasks are managed via [mise](https://mise.jdx.dev/) (defined in `mise.toml`).

Key tools:
- `ansible`, `kubectl`, `helm`, `flux2`, `k9s`, `kubectx`, `sops`, `age`
- Utilities: `jq`, `yq`, `shellcheck`

## Common Development Commands

All standard workflows are wrapped in `mise` tasks:

### Deploy & Manage Cluster

```bash
# Apply the full site.yaml playbook to all nodes
mise run ansible-playbook-run

# Run only the K3s installation role
mise run k3s-playbook-run

# Run only local bootstrapping (Flux install)
mise run local-playbook-run

# Fetch kubeconfig from the control plane (and rewrite endpoint to hostname)
mise run kubeconfig-get

# Run arbitrary command on all nodes via SSH
mise run ssh-command-run "systemctl status k3s"
```

### Direct Cluster Access

```bash
# Point kubectl to the downloaded cluster config
export KUBECONFIG=kubeconfig/k3s.yaml

# Run kubectl commands
kubectl get nodes
kubectl get pods -A

# Interactive management
k9s
```

### Secret Management (SOPS / Age)

```bash
# Generate age encryption keys
mise run age-generate-keys

# Create the sops-age secret in the cluster (flux-system namespace)
mise run sops-secret-create
```

## Repository & Playbook Structure

```
.
├── group_vars/
│   └── all.yaml              # Centralized cluster & node definitions
├── hosts                     # Ansible inventory
├── kubeconfig/
│   └── k3s.yaml              # Local kubeconfig credentials (git-ignored)
├── roles/
│   ├── common/               # Shared OS setup (hostnames, kernel params, cgroups)
│   ├── k3s/                  # K3s server/agent installation and systemd service
│   └── local/                # Local machine bootstrapping and Flux install
├── flux_github_token.yaml    # Ansible Vault-encrypted token for Flux
├── mise.toml                 # Tool definitions and runner tasks
├── README.md                 # Project overview and quickstart
└── site.yaml                 # Main Ansible playbook entrypoint
```

### Playbook Execution Order (`site.yaml`)

1. **`common`** → Runs on all nodes (hostnames, system configuration, cgroups).
2. **`k3s`** → Runs on all nodes (installs server on `eniac-node1`, agent on `eniac-node2..4`).
3. **`local`** → Runs on localhost (bootstraps Flux CD into the cluster).

Configuration variables:
- `group_vars/all.yaml`: Node inventory mappings, IPs, and cluster domain.
- `roles/k3s/defaults/main.yaml`: K3s version, installation args, disabled packaged components (servicelb, traefik, etc.).

## Conventions & Standards

- **Idempotency:** All Ansible tasks and roles must be safe to execute multiple times without unintended side effects.
- **Separation of Concerns (Externalized Manifests):**
  - This repo (IaC) is strictly for OS configuration, K3s provisioning, and initial Flux bootstrapping.
  - All Kubernetes workloads, networking configs, storage drivers, and application manifests belong in the [`gawbul/gawbul-gitops`](https://github.com/gawbul/gawbul-gitops) repository.
- **Security & Secrets:**
  - Sensitive credentials in this repo are encrypted with Ansible Vault (`flux_github_token.yaml` via `.vault_pass`).
  - In-cluster secrets are managed with SOPS and age.
  - Never commit plaintext secrets, `.vault_pass`, or local `kubeconfig/` credentials.

## Troubleshooting

### Checking K3s Status

```bash
# Check service status on nodes (via SSH)
systemctl status k3s

# Follow service logs
journalctl -u k3s -f
```

### Local Hostname Resolution

Ensure your local workstation can resolve cluster node hostnames (e.g. `eniac-node1.home.gawbul.io`) through local DNS (e.g., Pi-hole or local router).
