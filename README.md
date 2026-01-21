# gawbul-homelab

Infrastructure as Code (IaC) for my home lab Kubernetes cluster, running on Raspberry Pi 5 hardware.

This project automates the provisioning of a **K3s** cluster using **Ansible** and integrates with **Flux CD** for GitOps-based application management.

## Architecture

*   **Hardware:** 4x Raspberry Pi 5 (16GB RAM, 64GB+ A2 MicroSD)
*   **OS:** Ubuntu 24.04 LTS (Noble Numbat)
*   **Kubernetes Distro:** [K3s](https://k3s.io/) (Lightweight, ARM64 optimized)
*   **Network:**
    *   **CNI:** [Cilium](https://cilium.io/) (eBPF-based networking, kube-proxy replacement)
    *   **LoadBalancer:** [MetalLB](https://metallb.universe.tf/) (Layer 2 mode)
    *   **Encryption:** WireGuard (Native Flannel backend)
*   **Storage:** NFS (via `nfs-subdir-external-provisioner` connected to Synology NAS)
*   **GitOps:** [Flux](https://fluxcd.io/)

### Node Layout

| Hostname | Role | IP | Description |
| :--- | :--- | :--- | :--- |
| `eniac-node1` | Control Plane + Worker | `192.168.50.4` | API Server, Etcd, Workloads |
| `eniac-node2` | Worker | `192.168.50.5` | Workloads |
| `eniac-node3` | Worker | `192.168.50.6` | Workloads |
| `eniac-node4` | Worker | `192.168.50.7` | Workloads |

## Getting Started

### Prerequisites

This project uses [mise](https://mise.jdx.dev/) to manage all tooling versions (`ansible`, `kubectl`, `flux`, etc.).

1.  **Install mise:**
    ```bash
    curl https://mise.run | sh
    ```
2.  **Install dependencies:**
    ```bash
    mise install
    ```
3.  **Setup Secrets:**
    *   Create a `.vault_pass` file with the Ansible Vault password.
    *   Ensure `flux_github_token.yaml` exists (encrypted).

### Bootstrapping the Cluster

The entire cluster can be provisioned with a few commands.

1.  **Configure Local Networking:**
    Update your local `/etc/hosts` to resolve node hostnames.
    ```bash
    mise run etc-hosts-set
    ```

2.  **Deploy Infrastructure:**
    Run the Ansible playbook to configure the OS, install K3s, and bootstrap Flux.
    ```bash
    mise run ansible-playbook-run
    ```

3.  **Access the Cluster:**
    Fetch the admin `kubeconfig` to your local machine.
    ```bash
    mise run kubeconfig-get
    export KUBECONFIG=kubeconfig/k3s.yaml
    kubectl get nodes
    ```

## Development Workflows

*   **Run only K3s setup:** `mise run k3s-playbook-run`
*   **Run only Local bootstrap:** `mise run local-playbook-run`
*   **Run SSH command on all nodes:** `mise run ssh-command-run "uptime"`

## GitOps Structure

*   **Infrastructure Repo (This Repo):** Handles OS configuration, K3s installation, and initial Flux bootstrap.
*   **GitOps Repo ([gawbul/gawbul-gitops](https://github.com/gawbul/gawbul-gitops)):** Handles all Kubernetes resources (CNI, Storage, Apps) via Flux.

## License

MIT
