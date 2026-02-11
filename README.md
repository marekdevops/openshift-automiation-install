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

### 1. Create VMs

Edit `group_vars/virt_env1.yml` with your VM definitions, then:

```bash
ansible-playbook create-virt-env.yml -e @group_vars/virt_env1.yml
```

Note the MAC addresses from the output (vlan214 interface).

### 2. Configure OpenShift Cluster

Edit `group_vars/ocp_cluster1.yml`:
- Set MAC addresses from step 1
- Configure IPs, cluster name, domain
- Set API and Ingress VIPs

### 3. Generate Agent ISO

```bash
ansible-playbook generate-ocp-agent-iso.yml -e @group_vars/ocp_cluster1.yml
```

### 4. Boot VMs from ISO

Upload the generated ISO (`/tmp/ocp-install/<cluster>/agent.x86_64.iso`) to a PVC and attach it to VMs.

### 5. Monitor Installation

```bash
ansible-playbook wait-ocp-install.yml -e @group_vars/ocp_cluster1.yml
```

## Playbooks

| Playbook | Description |
|----------|-------------|
| `create-virt-env.yml` | Creates VMs (skips existing) |
| `delete-virt-env.yml` | Deletes VMs (dry-run by default, use `-e remove=yes`) |
| `generate-ocp-agent-iso.yml` | Generates Agent-Based Installer ISO |
| `wait-ocp-install.yml` | Monitors installation progress |

## Configuration Files

| File | Description |
|------|-------------|
| `group_vars/virt_env1.yml` | VM definitions (CPU, RAM, disk, network) |
| `group_vars/ocp_cluster1.yml` | OpenShift cluster config (nodes, IPs, MACs) |
| `templates/vm-worker-large.yaml.j2` | VM template for KubeVirt |
| `templates/install-config.yaml.j2` | OpenShift install-config template |
| `templates/agent-config.yaml.j2` | Agent config with static network |

## VM Features

- Host-passthrough CPU model
- CPU hotplug (up to 64 sockets)
- Memory hotplug (up to 1Ti)
- Dual NIC: pod network + Multus VLAN
- Multiqueue for network/disk
- Live migration support
