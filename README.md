# Kubernetes Infrastructure Automation with Ansible

This repository provides a fully automated, idempotent solution using Ansible to provision and upgrade a Kubernetes cluster. It supports both Debian/Ubuntu and RedHat/CentOS operating system families.

The project is divided into two primary operations: creating a new cluster from scratch and performing zero-downtime upgrades on an existing cluster.

## Architecture and Execution Flow

The following diagram illustrates the execution flow for both the creation and upgrade processes.

```mermaid
graph TD
    subgraph Ansible Execution
        A[Ansible Control Node]
        A -->|create-cluster.yaml| B(Provision Cluster)
        A -->|upgrade-cluster.yaml| C(Upgrade Cluster)
    end

    subgraph Provisioning Flow
        B --> D[Prepare All Nodes]
        D --> E[Install Containerd & Tools]
        E --> F[Initialize Master Node]
        F --> G[Join Worker Nodes]
    end

    subgraph Zero-Downtime Upgrade Flow
        C --> H[Upgrade Master Node]
        H --> I[Drain Master]
        I --> J[Apply Kubeadm Upgrade]
        J --> K[Uncordon Master]
        K --> L[Upgrade Worker Nodes]
        L --> M[Delegate Drain to Master]
        M --> N[Upgrade Kubelet Config]
        N --> O[Delegate Uncordon to Master]
    end
````

## Project Directory Structure

Plaintext

```
.
├── README.md
├── ansible.cfg
├── create-cluster.yaml
├── inventory
├── roles
│   ├── cluster-prep
│   ├── install-containerd
│   ├── install-k8s-tools
│   ├── master-init
│   ├── upgrade-master
│   ├── upgrade-worker
│   └── worker-join
└── upgrade-cluster.yaml
```

## Prerequisites

- Ansible installed on the control node.
    
- SSH key-based authentication established between the control node and all target nodes.
    
- Target nodes must be running a supported Linux distribution (Debian/Ubuntu or RHEL/CentOS).
    
- Sudo privileges for the SSH user on all target nodes.
    

## Usage: Creating a New Cluster

The cluster creation process handles OS preparation, container runtime installation, Control Plane initialization, and Worker Node joining.

1. **Configure the Inventory:**
    
    Open the `inventory` file and update the IP addresses or hostnames to match your target infrastructure. Ensure your master and worker nodes are placed under the correct groups.
    
    Ini, TOML
    
    ```
    [master]
    192.168.1.10
    
    [workers]
    192.168.1.11
    192.168.1.12
    ```
    
2. **Run the Provisioning Playbook:**
    
    Execute the following command to build the cluster. No further intervention is required.
    
    Bash
    
    ```
    ansible-playbook -i inventory create-cluster.yaml
    ```
    

## Usage: Upgrading an Existing Cluster

The upgrade process follows official Kubernetes version skew policies. It upgrades the Control Plane first, followed by the Worker Nodes using a safe cordon/drain and uncordon strategy to ensure zero downtime.

1. **Verify the Inventory:**
    
    Ensure the `inventory` file contains the current IP addresses of your cluster nodes.
    
2. **Run the Upgrade Playbook:**
    
    You must provide the exact target Kubernetes version using the `k8s_version` extra variable. The version must follow the `Major.Minor.Patch` format (e.g., 1.35.0).
    
    _Note: Do not skip minor versions. If you are on 1.34.x, you must upgrade to 1.35.x._
    
    Bash
    
    ```
    ansible-playbook -i inventory upgrade-cluster.yaml -e "k8s_version=1.35.0"
    ```
    
    _The playbook is highly intelligent; it will automatically extract the minor version to update the OS package repositories, verify current API server versions to maintain idempotency, and safely delegate worker draining tasks to the master node._
    

## Roles Overview

- **cluster-prep:** Disables swap, loads necessary kernel modules, and configures sysctl parameters required by Kubernetes.
    
- **install-containerd:** Installs and configures the containerd runtime and runc.
    
- **install-k8s-tools:** Installs `kubeadm`, `kubelet`, and `kubectl` on all nodes.
    
- **master-init:** Initializes the Kubernetes control plane using `kubeadm init` and deploys the networking CNI.
    
- **worker-join:** Extracts the join token from the master and attaches worker nodes to the cluster.
    
- **upgrade-master:** Manages the control plane upgrade. Handles draining the master, upgrading the database/manifests via `kubeadm upgrade apply`, updating packages, restarting services, and uncordoning.
    
- **upgrade-worker:** Manages worker node upgrades. Delegates drain/uncordon commands to the master node, upgrades local configurations via `kubeadm upgrade node`, and restarts local services.

