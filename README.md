# Kubernetes Cluster Setup with Ansible

This project automates the deployment of a Kubernetes cluster using Ansible playbooks with cloud-init configuration.

## Prerequisites

- Ansible installed on control machine
- Ubuntu 22.04 nodes (1 master, 2+ workers)
- SSH access to all nodes
- Minimum requirements:
  - Master node: 4GB RAM, 2 CPUs
  - Worker nodes: 2GB RAM, 2 CPUs each

## Project Structure

```
.
├── ansible.cfg                 # Ansible configuration
├── inventory/
│   └── production.yaml         # Production inventory file
├── playbooks/
│   ├── 00.site.yaml            # Contains all playbooks for site configuration
│   ├── 01.os-config.yaml       # OS basic configuration
│   ├── 02.docker.yaml          # Docker installation
│   └── 03.kubernetes.yaml      # Kubernetes cluster setup
└── test/
    └── molecule/               # Molecule test configuration
```

## Configuration

### 1. Ansible Configuration

The `ansible.cfg` file includes common settings:
- Default inventory path: `./inventory`
- SSH connection optimization
- Fact caching for performance
- Auto privilege escalation with sudo

### 2. Inventory Setup

Edit `inventory/production.yaml` to match your environment:

```yaml
---
all:
  children:
    master:
      hosts:
        master-node:
          ansible_host: 192.168.1.10
          ansible_user: ubuntu
    workers:
      hosts:
        worker1:
          ansible_host: 192.168.1.11
          ansible_user: ubuntu
        worker2:
          ansible_host: 192.168.1.12
          ansible_user: ubuntu
  vars:
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

Replace:
- `ansible_host`: IP addresses of your nodes
- `ansible_user`: SSH user (default: ubuntu)
- `ansible_ssh_private_key_file`: Path to your SSH private key

### 3. Playbook Overview

#### 01.os-config.yaml
Configures OS settings required for Kubernetes:
- Disables ufw
- Disables swap
- Loads kernel modules (br_netfilter, overlay)
- Configures sysctl parameters

#### 02.docker.yaml
Installs Docker and containerd:
- Adds Docker repository
- Installs Docker packages
- Configures Docker service
- Adds user to docker group

#### 03.kubernetes.yaml
Sets up Kubernetes cluster:
- Installs kubeadm, kubectl, kubelet (v1.34)
- Initializes master node
- Joins worker nodes to cluster
- Configures pod network (CIDR: 192.168.0.0/16)

## Usage

### Run All Playbooks

Since `ansible.cfg` specifies the default inventory path, you can run playbooks without `-i` flag:

```bash
ansible-playbook playbooks/01.os-config.yaml
ansible-playbook playbooks/02.docker.yaml
ansible-playbook playbooks/03.kubernetes.yaml
```

Or run all at once:

```bash
ansible-playbook playbooks/00.site.yaml
```

Alternatively, specify inventory explicitly:

```bash
ansible-playbook -i inventory/production.yml playbooks/0.site.yaml
```

### Run Individual Playbooks

```bash
# OS configuration only
ansible-playbook playbooks/01.os-config.yaml

# Docker installation only
ansible-playbook playbooks/02.docker.yaml

# Kubernetes setup only
ansible-playbook playbooks/03.kubernetes.yaml
```

### Verify Installation

After deployment, verify the cluster on the master node:

```bash
ssh ubuntu@192.168.1.10
kubectl get nodes
kubectl get pods -A
```

## Testing with Molecule

The project includes Molecule tests for local development:

```bash
# Install test dependencies
pip install molecule molecule-vagrant

# Run tests
cd test
molecule test
```

Molecule creates 3 VMs:
- 1 master node (4GB RAM, 2 CPUs)
- 2 worker nodes (2GB RAM, 2 CPUs each)

## Variables

### test_env

The `test_env` variable controls whether certain tasks are executed:
- **Default**: `false` (undefined)
- **Test mode**: `true` (set in molecule configuration)

When `test_env=true`, tasks with `when: not test_env` are skipped. This is useful for testing without modifying system settings like SELinux or firewall.

In production, omit this variable or set it to `false`.

## Network Configuration

The Kubernetes cluster uses the following network settings:
- Pod network CIDR: `192.169.0.0/16` (Default Calico CIDR)
- Service network: Default Kubernetes settings

To change the pod network, modify the `--pod-network-cidr` parameter in `playbooks/03.kubernetes.yaml`:

```yaml
- name: Initialize Kubernetes cluster
  ansible.builtin.command: kubeadm init --pod-network-cidr=192.168.0.0/16
```

## Troubleshooting

### SSH Connection Issues

If you encounter SSH connection problems:

```bash
# Test SSH connectivity
ansible all -m ping

# Use verbose mode
ansible-playbook playbooks/01.os-config.yaml -vvv
```

### Cluster Initialization Failed

Check master node logs:

```bash
ssh ubuntu@192.168.1.10
sudo journalctl -u kubelet -f
```

### Worker Node Join Failed

Regenerate join command on master:

```bash
ssh ubuntu@192.168.1.10
sudo kubeadm token create --print-join-command
```

## Version Information

- Kubernetes: v1.34 (latest stable)
- Docker: Latest from official repository
- Ubuntu: 22.04 LTS
