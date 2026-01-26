# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Kubernetes homelab setup running on Raspberry Pi 5 cluster (4 nodes named eniac-node1 through eniac-node4). The infrastructure is provisioned using **K3s** and Ansible, featuring a GitOps-driven architecture via Flux CD.

## Cluster Architecture

**Cluster Name:** eniac
**Control Plane:** eniac-node1
**Worker Nodes:** eniac-node1, eniac-node2, eniac-node3, eniac-node4
**Network:** 192.168.50.4-7 (static IPs), Pod CIDR: 10.42.0.0/16, Service CIDR: 10.43.0.0/16 (K3s defaults)
**Domain:** home.gawbul.io

The cluster uses:

- **K3s** as the Kubernetes distribution.
- **Flannel (Wireguard-native)** for secure inter-node networking.
- **Cilium** (via Flux) as the primary CNI.
- **MetalLB** (via Flux) for Layer 2 LoadBalancing.
- **NFS CSI driver** (via Flux) for network storage (NFS server at 192.168.50.2).
- **Flux CD** for GitOps deployment from gawbul/gawbul-gitops repository.

## Tooling & Environment

All tools are managed via **mise** (defined in `mise.toml`). Key tools include:

- ansible, kubectl, helm, flux2, k9s, kubectx
- Utilities: jq, yq, shellcheck

## Common Development Commands

### Deploy & Manage Cluster

```bash
# Run the full Ansible playbook to deploy/update the cluster
mise run ansible-playbook-run

# Run only the K3s installation role
mise run k3s-playbook-run

# Run only the local bootstrapping (Flux install)
mise run local-playbook-run

# Fetch the kubeconfig from the control plane
mise run kubeconfig-get

# Run arbitrary command on all nodes via SSH
mise run ssh-command-run "systemctl status k3s"
```

### Direct Cluster Access

```bash
# Set up local kubeconfig environment variable
export KUBECONFIG=kubeconfig/k3s.yaml

# Use kubectl normally
kubectl get nodes
kubectl get pods -A

# Use k9s for interactive cluster management
k9s
```

## Ansible Playbook Structure

The main playbook is `site.yaml`, which applies roles in this order:

1. **common** → All nodes (basic system setup, hostnames, cgroups).
2. **k3s** → All nodes (K3s server/agent installation and configuration).
3. **local** → Localhost (Flux bootstrapping).

**Variables:**

- `group_vars/all.yaml`: Cluster-wide settings (node definitions).
- `roles/k3s/defaults/main.yaml`: K3s specific versions and disabled components.

## Troubleshooting

### Checking Component Status

```bash
# On nodes (via SSH)
systemctl status k3s
journalctl -u k3s -f
```

### Local Hostname Resolution

Ensure your local machine can resolve the node hostnames (e.g., `eniac-node1.home.gawbul.io`) via your local DNS (e.g., Pi-hole).

### Common Errors
