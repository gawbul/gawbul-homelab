# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Kubernetes homelab setup running on Raspberry Pi 5 cluster (4 nodes named eniac-node1 through eniac-node4). The infrastructure is provisioned using Ansible and follows the "Kubernetes The Hard Way" approach by Kelsey Hightower, setting up Kubernetes from scratch rather than using managed distributions.

## Cluster Architecture

**Cluster Name:** eniac
**Control Plane:** eniac-node1 (also runs workloads)
**Worker Nodes:** eniac-node1, eniac-node2, eniac-node3, eniac-node4
**Network:** 192.168.50.4-7 (static IPs), Pod CIDR: 10.200.0.0/16
**Domain:** kubernetes.local

The cluster uses:
- **etcd** for state management (on control plane)
- **containerd** as container runtime with **runc**
- **CNI plugins** for networking
- **CoreDNS** for cluster DNS (ClusterIP: 10.32.0.10)
- **local-path-provisioner** as default storage class
- **NFS CSI driver** for network storage (NFS server at 192.168.50.2)
- **Metrics Server** for resource metrics
- **Flux CD** for GitOps deployment from gawbul/gawbul-gitops repository

## Tooling & Environment

All tools are managed via **mise** (defined in `mise.toml`). Key tools include:
- ansible, kubectl, helm, flux2, k9s, kubectx
- Container tools: cosign, container-structure-test, skaffold
- Config tools: jq, yq, pkl, shellcheck

Environment variables (set in mise.toml):
- `HOMELAB_NODES`: node hostnames
- `HOMELAB_IPS`: node IP addresses
- `K8S_CLUSTER_NAME`: "eniac"
- `K8S_CONTROL_PLANE_NODE`: "eniac-node1"

## Common Development Commands

### Initial Setup & Certificate Generation

```bash
# Set hostnames in /etc/hosts
mise run etc-hosts-set

# Generate CA certificate
mise run ssl-ca-certificate-generate

# Generate all SSL certificates (admin, nodes, service accounts, etc.)
mise run ssl-certificates-generate

# Generate all kubeconfig files
mise run kubeconfig-generate

# Generate encryption config for secrets at rest
mise run encryption-config-generate
```

### Deploy & Manage Cluster

```bash
# Run Ansible playbook to deploy/update cluster
mise run ansible-playbook-run

# Run arbitrary command on all nodes via SSH
mise run ssh-command-run command="systemctl status kubelet"
```

### Working with Flux CD

```bash
# Uninstall Flux
mise run flux-uninstall

# Generate age encryption keys for SOPS
mise run age-generate-keys

# Create SOPS secret in cluster
mise run sops-secret-create
```

### Direct Cluster Access

```bash
# Set up local kubeconfig
mise run kubeconfig-local-generate

# Use kubectl normally
kubectl get nodes
kubectl get pods -A

# Use k9s for interactive cluster management
k9s
```

## Ansible Playbook Structure

The main playbook is `site.yaml`, which applies roles in this order:

1. **common** → All nodes except localhost (basic system setup)
2. **k8s_all_nodes** → Control planes + workers (container runtime, CNI, kubelet)
3. **k8s_control_planes** → Control plane only (etcd, kube-apiserver, kube-controller-manager, kube-scheduler)
4. **k8s_worker_nodes** → Workers (kubelet configuration)
5. **local** → Localhost (cluster bootstrapping, CoreDNS, storage, Flux)

**Inventory:** The `hosts` file defines node groups. Note that eniac-node1 is in both `k8s_control_planes` and `k8s_worker_nodes`.

**Variables:**
- `group_vars/all.yaml`: Cluster-wide settings (K8s versions, node definitions, network config)
- `group_vars/local.yaml`: Local bootstrapping settings (GitOps repo, NFS config, versions)
- `group_vars/k8s_*.yaml`: Role-specific variables

**Ansible Vault:** Uses `.vault_pass` file for encrypted variables (configured in `ansible.cfg`). The `flux_github_token.yaml` is vault-encrypted and passed as extra vars.

## Important Configuration Details

### Certificates Location
All certificates and keys are stored in the `certs/` directory. Required certificates:
- `ca.crt` / `ca.key`: Root CA
- `admin.crt` / `admin.key`: Admin user
- `{node}.crt` / `{node}.key`: Per-node certificates
- `kube-proxy.crt`, `kube-scheduler.crt`, `kube-controller-manager.crt`, `kube-api-server.crt`: Component certificates
- `service-accounts.key`: For service account token signing
- `front-proxy-client.crt`: For API aggregation

### Kubeconfig Files
Generated kubeconfig files are in the `kubeconfig/` directory:
- `admin.kubeconfig`: Cluster admin access
- `{node}.kubeconfig`: Per-node kubelet auth
- `kube-proxy.kubeconfig`, `kube-scheduler.kubeconfig`, `kube-controller-manager.kubeconfig`: Component auth

### OpenSSL Configuration
Certificate generation uses `config/ca.conf` with sections defining:
- Subject Alternative Names (SANs) for API server
- Distinguished Names for each component
- Key usage and extended key usage attributes

### Data Encryption at Rest
Secrets and ConfigMaps are encrypted using the configuration in `encryption-config.yaml`, generated from `config/encryption-configuration.yaml` template with a random AES key.

## GitOps Workflow

The cluster uses Flux CD for continuous deployment:
- **Source Repository:** This repo (gawbul/gawbul-homelab) - infrastructure code
- **GitOps Repository:** gawbul/gawbul-gitops - application manifests
- **Flux Path:** clusters/eniac
- **Flux Components:** source-controller, kustomize-controller, helm-controller, notification-controller, image-reflector-controller, image-automation-controller

Changes to the GitOps repo are automatically reconciled by Flux. This repo handles the underlying infrastructure.

## Software Versions

Check `group_vars/all.yaml` for current versions:
- Kubernetes: 1.34.1
- containerd: 2.2.0
- runc: 1.3.3
- CNI plugins: 1.8.0
- etcd: 3.6.5
- crictl: 1.34.0

The cluster runs on Ubuntu Noble (24.04) arm64.

## Troubleshooting

### Checking Component Status
```bash
# On nodes (via SSH)
systemctl status kubelet
systemctl status containerd
systemctl status etcd  # control plane only

# From local machine
kubectl get componentstatuses
kubectl get nodes
kubectl describe node <node-name>
```

### Certificate Issues
Certificates are valid for 10 years (3653 days). If regenerating, ensure all dependent kubeconfig files are also regenerated using `mise run kubeconfig-generate`.

### DNS Issues
CoreDNS runs in kube-system namespace with ClusterIP 10.32.0.10. Verify with:
```bash
kubectl get svc -n kube-system coredns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### Storage Issues
- Default storage: local-path-provisioner
- Network storage: NFS CSI driver pointing to 192.168.50.2:/volume1/kubernetes
- Check storage class: `kubectl get storageclass`
