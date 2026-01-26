# Project Context: gawbul-homelab

## Project Overview

This repository contains the Infrastructure as Code (IaC) for setting up a Kubernetes homelab cluster on Raspberry Pi 5 hardware. It uses **K3s** for the Kubernetes distribution, provisioned via **Ansible** on **Ubuntu Noble (24.04)**.

The project follows "Best Practice" patterns for bare-metal Kubernetes, including declarative configuration, immutable-like node management, and GitOps for all internal components.

## Architecture

*   **Cluster Name**: `eniac`
*   **Hardware**: 4x Raspberry Pi 5 (16GB RAM, 512GB microSD)
*   **Nodes**:
    *   `eniac-node1`: Control Plane + Worker
    *   `eniac-node2` - `eniac-node4`: Worker Nodes
*   **Networking**:
    *   Node Network: `192.168.50.0/24` (Static IPs `.4` - `.7`)
    *   Pod/Service CIDR: K3s Defaults (`10.42.0.0/16`, `10.43.0.0/16`)
    *   Inter-node encryption: **Wireguard (native)** via Flannel.
*   **Key Components**:
    *   **Cilium**: High-performance eBPF CNI.
    *   **MetalLB**: Layer 2 LoadBalancer.
    *   **NFS Subdir Provisioner**: Dynamic storage on NAS (`192.168.50.2`).
    *   **Flux CD**: GitOps engine (syncs with `gawbul/gawbul-gitops`).

## Development Environment & Tooling

This project uses [mise](https://mise.jdx.dev/) for tool management and task running.

### Key Tools (Managed by `mise`)
*   `ansible`, `kubectl`, `helm`, `flux2`, `k9s`, `kubectx`, `jq`, `yq`.

## Workflows & Commands

### 1. Cluster Deployment
The cluster state is applied via Ansible commands:

```bash
# Apply the full site.yaml playbook to all nodes
mise run ansible-playbook-run

# Run specific roles in isolation
mise run k3s-playbook-run
mise run local-playbook-run

# Get the cluster credentials
mise run kubeconfig-get
```

**Playbook Order (`site.yaml`):**
1.  `common`: Basic system setup (all nodes).
2.  `k3s`: K3s installation and configuration (all nodes).
3.  `local`: Local machine bootstrapping (Flux install).

### 2. GitOps (Flux)
Components like Cilium, MetalLB, and the NFS Provisioner are defined in the external `gawbul-gitops` repository and reconciled by Flux.

## Directory Structure

*   `roles/k3s/`: The primary automation role for K3s installation.
*   `roles/common/`: Shared system configuration (hostnames, kernel parameters).
*   `roles/local/`: Local machine setup and Flux bootstrapping.
*   `group_vars/all.yaml`: Centralized node definitions.
*   `kubeconfig/k3s.yaml`: Local access credentials (git-ignored).

## Conventions & Standards

*   **Idempotency**: Ansible roles must be safe to run multiple times.
*   **Externalized Manifests**: All K8s resources (apps and infra) belong in the GitOps repo, not this IaC repo.
*   **Security**: Sensitive data is managed via Ansible Vault (`flux_github_token.yaml`).
