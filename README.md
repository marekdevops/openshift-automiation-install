# OpenShift Automation Install

Ansible automation for deploying OpenShift clusters on OpenShift Virtualization using Agent-Based Installer.

## Overview

This project automates:
1. **VM Creation** — Creates virtual machines on OpenShift Virtualization (KubeVirt)
2. **OpenShift Installation** — Generates Agent-Based Installer ISO and deploys OCP cluster

## Prerequisites

- OpenShift Virtualization cluster (hosting platform)
- `oc` CLI logged in to the cluster
- `openshift-install` binary (download from [mirror.openshift.com](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/))
- Pull secret from [console.redhat.com](https://console.redhat.com/openshift/install/pull-secret)
- Ansible with collections:
  ```bash
  ansible-galaxy collection install kubernetes.core kubevirt.core ansible.utils
  ```

## Quick Start

### 1. Create Cluster Configuration

Copy and edit the example configuration:

```bash
cp clusters/ocp1.yml clusters/mycluster.yml
```

Edit `clusters/mycluster.yml`:
- Set `cluster_name`, `base_domain`
- Configure network settings (`machine_network`, `api_vip`, `ingress_vip`)
- Define nodes with hostnames, roles, and IPs

### 2. Create VMs

```bash
ansible-playbook create-virt-env.yml -e @clusters/mycluster.yml
```

This will:
- Create VMs in OpenShift Virtualization (without CDROM)
- Start VMs and wait for them to run
- **Automatically save MAC addresses** to the config file

### 3. Verify Configuration

Check `clusters/mycluster.yml` — MAC addresses are now populated. Verify:
- IP addresses are correct
- Hostnames are correct

### 4. Generate Agent ISO

```bash
ansible-playbook generate-ocp-agent-iso.yml -e @clusters/mycluster.yml
```

Output: `/tmp/ocp-install/<cluster>/agent.x86_64.iso`

### 5. Attach ISO and Restart VMs

```bash
ansible-playbook attach-iso.yml -e @clusters/mycluster.yml
```

This will:
- Upload ISO to PVC
- Stop VMs
- Attach CDROM with ISO
- Set boot order (CDROM first)
- Start VMs

### 6. Monitor Installation

```bash
ansible-playbook wait-ocp-install.yml -e @clusters/mycluster.yml
```

## Playbooks

| Playbook | Description |
|----------|-------------|
| `create-virt-env.yml` | Creates VMs (no CDROM), starts, saves MAC addresses |
| `delete-virt-env.yml` | Deletes VMs (dry-run by default, use `-e remove=yes`) |
| `generate-ocp-agent-iso.yml` | Generates Agent-Based Installer ISO |
| `attach-iso.yml` | Uploads ISO, attaches CDROM, restarts VMs |
| `wait-ocp-install.yml` | Monitors installation progress |

## Configuration

All configuration is in `clusters/<name>.yml`:

```yaml
# Cluster identity
cluster_name: ocp1
base_domain: example.com

# VM settings
namespace: proj-vms
vm_defaults:
  cpu_sockets: 8
  memory: 32Gi
  disk_size: 120Gi

# Network
machine_network: 192.168.214.0/24
api_vip: 192.168.214.100      # F5/LB VIP for API
ingress_vip: 192.168.214.101  # F5/LB VIP for Apps
external_lb: true             # Using external load balancer

# Nodes (MAC auto-populated by create-virt-env.yml)
nodes:
  - name: ocp1-master-0
    hostname: master-0
    role: master
    mac: ""                   # <- auto-filled
    ip: 192.168.214.10
```

## DNS Requirements

Configure DNS before installation:

```
api.<cluster>.<domain>     → api_vip
api-int.<cluster>.<domain> → api_vip
*.apps.<cluster>.<domain>  → ingress_vip
```

## Load Balancer Configuration

For F5/HAProxy, configure pools:

| VIP | Port | Backend Pool |
|-----|------|--------------|
| api_vip | 6443 | All masters |
| api_vip | 22623 | All masters |
| ingress_vip | 80 | All workers (or masters if no workers) |
| ingress_vip | 443 | All workers (or masters if no workers) |

## VM Features

- Host-passthrough CPU model
- CPU hotplug (configurable max)
- Memory hotplug (configurable max)
- Dual NIC: pod network + Multus VLAN
- Multiqueue for network/disk
- Live migration support
