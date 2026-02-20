# OpenShift Automation Install

Ansible automation for deploying OpenShift clusters on OpenShift Virtualization using Agent-Based Installer.

## Overview

This project automates:
1. **VM Creation** — Creates virtual machines on OpenShift Virtualization (KubeVirt)
2. **OpenShift Installation** — Generates Agent-Based Installer ISO and deploys OCP cluster
3. **Lifecycle Management** — Boot order changes, ISO swapping, cluster monitoring

## Prerequisites

- OpenShift Virtualization cluster (hosting platform)
- `oc` CLI logged in to the cluster
- `virtctl` CLI for VM operations
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
- Configure network settings (`machine_network`, `dns_servers`, `ntp_servers`)
- Set `external_lb: true` for external load balancer (F5/HAProxy)
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
ansible-playbook cdrom/attach-iso.yml -e @clusters/mycluster.yml
```

This will:
- Upload ISO to DataVolume (ReadWriteMany)
- Stop VMs
- Attach CDROM with ISO (bootOrder: 1)
- Start VMs

### 6. Change Boot Order (during installation)

```bash
ansible-playbook cdrom/bootorder.yml -e @clusters/mycluster.yml
```

Changes boot order to rootdisk without restarting VMs. The change takes effect when nodes restart during OCP installation.

### 7. Monitor Installation

```bash
ansible-playbook wait-ocp-install.yml -e @clusters/mycluster.yml
```

Or manually:
```bash
openshift-install agent wait-for install-complete --dir=/tmp/ocp-install/<cluster>
```

## Playbooks

### Core Playbooks

| Playbook | Description |
|----------|-------------|
| `create-virt-env.yml` | Creates VMs (no CDROM), starts, saves MAC addresses |
| `delete-virt-env.yml` | Deletes VMs (dry-run by default, use `-e remove=yes`) |
| `generate-ocp-agent-iso.yml` | Generates Agent-Based Installer ISO |
| `wait-ocp-install.yml` | Monitors installation progress |
| `diagnose-vm.yml` | Diagnoses VM configuration issues |

### CDROM Management (`cdrom/`)

| Playbook | Description |
|----------|-------------|
| `cdrom/attach-iso.yml` | Uploads ISO to DataVolume, attaches CDROM, restarts VMs |
| `cdrom/add-cdrom.yml` | Adds CDROM to VMs that don't have it |
| `cdrom/remove-cdrom.yml` | Removes CDROM from VMs |
| `cdrom/swapiso.yml` | Swaps ISO in CDROM to different DataVolume |
| `cdrom/bootorder.yml` | Changes boot order (CDROM ↔ rootdisk) |

### Playbook Options

**cdrom/bootorder.yml:**
```bash
# Default: change boot order without restart
ansible-playbook cdrom/bootorder.yml -e @clusters/mycluster.yml

# With restart
ansible-playbook cdrom/bootorder.yml -e @clusters/mycluster.yml -e restart=yes
```

**cdrom/add-cdrom.yml:**
```bash
# Add CDROM using iso_pvc_name from config
ansible-playbook cdrom/add-cdrom.yml -e @clusters/mycluster.yml

# Add CDROM with custom DataVolume
ansible-playbook cdrom/add-cdrom.yml -e @clusters/mycluster.yml -e iso_dv_name=custom-iso
```

**cdrom/swapiso.yml:**
```bash
ansible-playbook cdrom/swapiso.yml -e @clusters/mycluster.yml -e iso_dv_name=new-iso-dv
```

**wait-ocp-install.yml:**
```bash
# Wait for bootstrap only
ansible-playbook wait-ocp-install.yml -e @clusters/mycluster.yml -e stage=bootstrap

# Wait for complete installation (default)
ansible-playbook wait-ocp-install.yml -e @clusters/mycluster.yml -e stage=complete
```

## Configuration

All configuration is in `clusters/<name>.yml`:

```yaml
# Cluster identity
cluster_name: ocp1
base_domain: example.com

# VM settings
namespace: proj-vms
iso_pvc_name: agent-iso-pvc
multus_network: default/vlan214-net

vm_defaults:
  cpu_sockets: 8
  memory: 32Gi
  disk_size: 120Gi
  max_cpu_sockets: 16
  max_memory: 64Gi

# Network
machine_network: 192.168.214.0/24
gateway: 192.168.214.1

dns_servers:
  - 192.168.214.1

ntp_servers:
  - 192.168.214.1

# External load balancer (F5/HAProxy) - no VIPs needed
external_lb: true

# Internal load balancer (keepalived) - VIPs required
# external_lb: false
# api_vip: 192.168.214.100
# ingress_vip: 192.168.214.101

# Nodes (MAC auto-populated by create-virt-env.yml)
nodes:
  - name: ocp1-master-0
    hostname: master-0
    role: master
    mac: ""                   # <- auto-filled
    ip: 192.168.214.10

  - name: ocp1-master-1
    hostname: master-1
    role: master
    mac: ""
    ip: 192.168.214.11

  - name: ocp1-master-2
    hostname: master-2
    role: master
    mac: ""
    ip: 192.168.214.12

  - name: ocp1-worker-0
    hostname: worker-0
    role: worker
    mac: ""
    ip: 192.168.214.20

# Installation options
single_node: false
fips_enabled: false
```

## Air-Gapped Installation

For environments without internet access, you can provide a local RHCOS ISO:

```yaml
# In cluster config
rhcos_iso_path: /path/to/rhcos-live.x86_64.iso
```

Download RHCOS ISO from [mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/)

**Note:** For fully air-gapped environments, you also need a mirror registry with OCP release images. The simplest approach is to generate the agent ISO on a connected host and transfer it.

## DNS Requirements

Configure DNS before installation:

### With External Load Balancer (`external_lb: true`)

DNS points directly to your F5/HAProxy VIP:

```
api.<cluster>.<domain>     → F5 VIP
api-int.<cluster>.<domain> → F5 VIP
*.apps.<cluster>.<domain>  → F5 VIP
```

### With Internal Load Balancer (`external_lb: false`)

DNS points to cluster-managed VIPs:

```
api.<cluster>.<domain>     → api_vip
api-int.<cluster>.<domain> → api_vip
*.apps.<cluster>.<domain>  → ingress_vip
```

## Load Balancer Configuration

For F5/HAProxy, configure pools:

| Service | Port | Backend Pool |
|---------|------|--------------|
| API | 6443 | All masters |
| Machine Config | 22623 | All masters |
| HTTP Ingress | 80 | All workers (or masters if no workers) |
| HTTPS Ingress | 443 | All workers (or masters if no workers) |

## VM Features

- **CPU**: Host-passthrough model with hotplug support
- **Memory**: Hotplug support (configurable max)
- **Network**: Dual NIC (pod network + Multus VLAN), IPv4 only
- **Storage**: VirtIO disk with multiqueue
- **Migration**: Live migration support (requires RWX storage)

## Troubleshooting

### VM not registering with rendezvous host

Check on the problematic VM:
```bash
virtctl console <vm-name> -n <namespace>

# Inside VM
systemctl status agent
journalctl -u agent -f
ping <rendezvous-ip>
```

### MAC address mismatch

Verify MAC in config matches actual VM:
```bash
# Actual MAC
oc get vmi <vm-name> -n <namespace> -o jsonpath='{.status.interfaces}' | jq .

# Config MAC
cat clusters/<cluster>.yml | grep -A5 <vm-name>
```

### Boot order issues

Check current boot order:
```bash
oc get vm <vm-name> -n <namespace> -o yaml | grep -A5 bootOrder
```

### DataVolume stuck in provisioning

Check CDI logs:
```bash
oc get dv -n <namespace>
oc describe dv <dv-name> -n <namespace>
```
