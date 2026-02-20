# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible automation for deploying OpenShift clusters on OpenShift Virtualization (KubeVirt) using Agent-Based Installer. The project creates VMs, generates installation ISOs, manages CDROM lifecycle, and monitors cluster deployment.

## Project Structure

```
openshift-automation-install/
├── clusters/                       # Cluster configurations
│   └── ocp1.yml
├── templates/                      # Jinja2 templates
│   ├── agent-config.yaml.j2        # Agent config (MAC→IP mapping)
│   ├── install-config.yaml.j2      # OCP install config
│   └── vm-worker-large.yaml.j2     # VM template
├── cdrom/                          # CDROM management playbooks
│   ├── add-cdrom.yml
│   ├── attach-iso.yml
│   ├── bootorder.yml
│   ├── remove-cdrom.yml
│   └── swapiso.yml
├── create-virt-env.yml             # Create VMs
├── delete-virt-env.yml             # Delete VMs
├── generate-ocp-agent-iso.yml      # Generate ISO
├── wait-ocp-install.yml            # Monitor installation
├── diagnose-vm.yml                 # VM diagnostics
└── README.md
```

## Running Playbooks

All playbooks use cluster configuration from `clusters/<name>.yml`:

```bash
# === Core Workflow ===

# Create VMs (auto-saves MAC addresses to config file)
ansible-playbook create-virt-env.yml -e @clusters/ocp1.yml

# Generate Agent-Based Installer ISO
ansible-playbook generate-ocp-agent-iso.yml -e @clusters/ocp1.yml

# Attach ISO and start VMs
ansible-playbook cdrom/attach-iso.yml -e @clusters/ocp1.yml

# Change boot order to rootdisk (without restart - takes effect on next reboot)
ansible-playbook cdrom/bootorder.yml -e @clusters/ocp1.yml

# Monitor installation
ansible-playbook wait-ocp-install.yml -e @clusters/ocp1.yml

# === CDROM Management ===

# Add CDROM to VMs without it
ansible-playbook cdrom/add-cdrom.yml -e @clusters/ocp1.yml

# Remove CDROM from VMs
ansible-playbook cdrom/remove-cdrom.yml -e @clusters/ocp1.yml

# Swap ISO to different DataVolume
ansible-playbook cdrom/swapiso.yml -e @clusters/ocp1.yml -e iso_dv_name=new-iso

# === Utilities ===

# Diagnose VM configuration issues
ansible-playbook diagnose-vm.yml -e @clusters/ocp1.yml

# Delete VMs (dry-run by default, use -e remove=yes to actually delete)
ansible-playbook delete-virt-env.yml -e @clusters/ocp1.yml -e remove=yes
```

## Key Configuration Options

**Cluster config (`clusters/<name>.yml`):**

```yaml
cluster_name: ocp1
base_domain: example.com
namespace: proj-vms

# External LB (F5/HAProxy) - no VIPs needed, uses platform: none
external_lb: true

# Internal LB (keepalived) - requires VIPs
# external_lb: false
# api_vip: 192.168.214.100
# ingress_vip: 192.168.214.101

# DNS and NTP as lists
dns_servers:
  - 192.168.214.1
ntp_servers:
  - 192.168.214.1

# Air-gapped: local RHCOS ISO
# rhcos_iso_path: /path/to/rhcos-live.x86_64.iso
```

**Playbook options:**

- `restart=yes` — For cdrom playbooks, restart VMs after changes (default: no)
- `iso_dv_name=xxx` — Custom DataVolume name for ISO
- `stage=bootstrap|complete` — For wait-ocp-install.yml
- `remove=yes` — For delete-virt-env.yml, actually delete VMs

## Key Features

- **IaC approach**: Single config file per cluster in `clusters/`
- **Auto MAC population**: `create-virt-env.yml` saves MAC addresses back to config
- **External LB support**: `external_lb: true` uses `platform: none` (no VIPs required)
- **Air-gapped support**: `rhcos_iso_path` for local RHCOS ISO
- **DataVolume for ISO**: Uses DataVolume (not PVC) with ReadWriteMany for live migration
- **IPv4 only**: IPv6 explicitly disabled in network config
- **Protection**: `delete-virt-env.yml` requires `-e remove=yes` to delete
- **Idempotent**: Playbooks skip existing resources

## Key Dependencies

- Ansible collections: `kubernetes.core`, `kubevirt.core`, `ansible.utils`
- `oc` CLI logged in to cluster
- `virtctl` for VM console access
- `openshift-install` binary for ISO generation
- Pull secret from console.redhat.com

## Conventions

- Playbooks use active OpenShift context (oc login)
- VMs use: host-passthrough CPU, hotplug CPU/RAM, dual NIC (pod + Multus VLAN)
- ISO stored as DataVolume with ReadWriteMany access mode
- Comments in playbooks are in Polish
