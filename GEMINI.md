# Project Context: gawbul-homelab

## Project Overview

This repository contains the Infrastructure as Code (IaC) for setting up a Kubernetes homelab cluster on Raspberry Pi 5 hardware. It follows the "Kubernetes The Hard Way" philosophy (adapted from Kelsey Hightower's guide), manually provisioning components to provide a deep understanding of Kubernetes internals.

The infrastructure is provisioned using **Ansible** and deployed to nodes running **Ubuntu Noble (24.04)**.

## Architecture

*   **Cluster Name**: `eniac`
*   **Hardware**: 4x Raspberry Pi 5
*   **Nodes**:
    *   `eniac-node1`: Control Plane + Worker
    *   `eniac-node2` - `eniac-node4`: Worker Nodes
*   **Networking**:
    *   Node Network: `192.168.50.0/24` (Static IPs `192.168.50.4` - `.7`)
    *   Pod CIDR: `10.200.0.0/16`
    *   Service CIDR: `10.32.0.0/24`
*   **Key Components**:
    *   **etcd**: Distributed key-value store.
    *   **containerd**: Container runtime (via `runc`).
    *   **CoreDNS**: Cluster DNS.
    *   **CNI Plugins**: Networking.
    *   **Flux CD**: GitOps for application delivery (syncs with `gawbul/gawbul-gitops`).

## Development Environment & Tooling

This project uses [mise](https://mise.jdx.dev/) for tool management and task running.

### Key Tools (Managed by `mise`)
*   **Infrastructure**: `ansible`, `kubectl`, `helm`, `flux2`
*   **Utilities**: `k9s`, `kubectx`, `jq`, `yq`
*   **Verification**: `cosign`, `container-structure-test`

### Environment Variables
Key variables are defined in `mise.toml`:
*   `HOMELAB_NODES`: List of node hostnames.
*   `HOMELAB_IPS`: List of node IPs.
*   `K8S_CONTROL_PLANE_NODE`: The primary control plane node.

## Workflows & Commands

All major tasks are encapsulated as `mise` scripts.

### 1. Initial Setup & Certificate Generation
Before deploying, certificates and configuration files must be generated:

```bash
# Set local /etc/hosts for node resolution
mise run etc-hosts-set

# Generate Certificate Authority and all component certificates
mise run ssl-ca-certificate-generate
mise run ssl-certificates-generate

# Generate Kubeconfig files for all components (admin, kubelet, scheduler, etc.)
mise run kubeconfig-generate

# Generate encryption configuration for secrets at rest
mise run encryption-config-generate
```

### 2. Deployment (Ansible)
The cluster state is applied via Ansible playbooks.

```bash
# Apply the site.yaml playbook to all nodes
mise run ansible-playbook-run
```

**Playbook Order (`site.yaml`):**
1.  `common`: Basic system setup (all nodes).
2.  `k8s_all_nodes`: Container runtime, CNI, kubelet (all k8s nodes).
3.  `k8s_control_planes`: Control plane components (etcd, apiserver, scheduler, controller-manager).
4.  `k8s_worker_nodes`: Worker specific config (kube-proxy).
5.  `local`: Local machine bootstrapping (CoreDNS, storage classes, Flux).

### 3. Cluster Access & Management
```bash
# Generate a local kubeconfig to access the cluster
mise run kubeconfig-local-generate

# Verify node status
kubectl get nodes

# Interactive management
k9s
```

### 4. GitOps (Flux)
Applications are managed via Flux CD.
*   **Bootstrapping**: Handled in the `local` Ansible role.
*   **Secrets**: Encrypted with SOPS/Age.
    *   Generate keys: `mise run age-generate-keys`
    *   Create cluster secret: `mise run sops-secret-create`

## Directory Structure

*   `roles/`: Ansible roles defining the configuration logic.
*   `group_vars/`: Variable definitions for Ansible groups (`all.yaml` contains versions).
*   `certs/`: Generated SSL certificates and keys (git-ignored).
*   `kubeconfig/`: Generated kubeconfig files (git-ignored).
*   `config/`: Configuration templates (OpenSSL, Encryption).
*   `hosts`: Ansible inventory file.

## Conventions & Standards

*   **Security**:
    *   Ansible Vault is used for sensitive variables (requires `.vault_pass`).
    *   Kubernetes secrets are encrypted at rest.
    *   GitOps secrets are managed via SOPS.
*   **Versioning**: Software versions (Kubernetes, containerd, etc.) are centralized in `group_vars/all.yaml`.
*   **Idempotency**: Ansible tasks should be idempotent to allow safe re-runs.
