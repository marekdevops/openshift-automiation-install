# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible automation for deploying OpenShift clusters on OpenShift Virtualization (KubeVirt) using Agent-Based Installer. The project creates VMs, generates installation ISOs, and monitors cluster deployment.

## Running Playbooks

All playbooks use cluster configuration from `clusters/<name>.yml`:

```bash
# Create VMs (auto-saves MAC addresses to config file)
ansible-playbook create-virt-env.yml -e @clusters/ocp1.yml

# Delete VMs (dry-run by default)
ansible-playbook delete-virt-env.yml -e @clusters/ocp1.yml
# Actually delete:
ansible-playbook delete-virt-env.yml -e @clusters/ocp1.yml -e remove=yes

# Generate Agent-Based Installer ISO
ansible-playbook generate-ocp-agent-iso.yml -e @clusters/ocp1.yml

# Monitor installation
ansible-playbook wait-ocp-install.yml -e @clusters/ocp1.yml
```

## Architecture

**Configuration:**
- `clusters/<name>.yml` — Single source of truth for each cluster (VM specs, network, nodes with MAC/IP)
- MAC addresses are auto-populated by `create-virt-env.yml`

**VM Management:**
- `create-virt-env.yml` — Creates VMs, waits for them to start, saves MAC addresses to config
- `delete-virt-env.yml` — Removes VMs defined in config (dry-run protection)
- `templates/vm-worker-large.yaml.j2` — Jinja2 template for VMs

**OpenShift Installation:**
- `generate-ocp-agent-iso.yml` — Generates `install-config.yaml`, `agent-config.yaml`, and bootable ISO
- `wait-ocp-install.yml` — Waits for bootstrap and installation completion
- `templates/install-config.yaml.j2` — OpenShift cluster configuration
- `templates/agent-config.yaml.j2` — Static network config per host (MAC → IP mapping)

## Key Features

- **IaC approach**: Single config file per cluster in `clusters/`
- **Auto MAC population**: `create-virt-env.yml` saves MAC addresses back to config
- **External LB support**: `external_lb: true` uses `loadBalancer.type: UserManaged` for F5/HAProxy
- **Protection**: `delete-virt-env.yml` requires `-e remove=yes` to actually delete
- **Idempotent**: `create-virt-env.yml` skips existing VMs

## Key Dependencies

- Ansible collections: `kubernetes.core`, `kubevirt.core`, `ansible.utils`
- `openshift-install` binary for ISO generation
- Pull secret from console.redhat.com

## Conventions

- Playbooks use active OpenShift context (no kubeconfig path needed)
- VMs are labeled `managed-by=ansible` for tracking
- Network: pod network (masquerade) + Multus VLAN bridge (vlan214)
