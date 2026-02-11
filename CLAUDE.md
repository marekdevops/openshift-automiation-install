# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible automation for deploying OpenShift clusters on OpenShift Virtualization (KubeVirt) using Agent-Based Installer. The project creates VMs, generates installation ISOs, and monitors cluster deployment.

## Running Playbooks

```bash
# Create VMs (skips existing ones)
ansible-playbook create-virt-env.yml -e @group_vars/virt_env1.yml

# Delete VMs (dry-run by default)
ansible-playbook delete-virt-env.yml -e @group_vars/virt_env1.yml
# Actually delete:
ansible-playbook delete-virt-env.yml -e @group_vars/virt_env1.yml -e remove=yes

# Generate Agent-Based Installer ISO
ansible-playbook generate-ocp-agent-iso.yml -e @group_vars/ocp_cluster1.yml

# Monitor installation
ansible-playbook wait-ocp-install.yml -e @group_vars/ocp_cluster1.yml
```

## Architecture

**VM Management:**
- `create-virt-env.yml` — Creates VMs from `virt_env1.yml`, never modifies existing VMs
- `delete-virt-env.yml` — Removes only VMs defined in config, dry-run protection
- `templates/vm-worker-large.yaml.j2` — Jinja2 template for large worker VMs

**OpenShift Installation:**
- `generate-ocp-agent-iso.yml` — Generates `install-config.yaml`, `agent-config.yaml`, and bootable ISO
- `wait-ocp-install.yml` — Waits for bootstrap and installation completion
- `templates/install-config.yaml.j2` — OpenShift cluster configuration
- `templates/agent-config.yaml.j2` — Static network config per host (MAC → IP mapping)

**Configuration:**
- `group_vars/virt_env1.yml` — VM definitions (names, CPU, RAM, disk)
- `group_vars/ocp_cluster1.yml` — OCP cluster config (nodes, MACs, IPs, VIPs, domain)

## Key Dependencies

- Ansible collections: `kubernetes.core`, `kubevirt.core`, `ansible.utils`
- `openshift-install` binary for ISO generation
- Pull secret from console.redhat.com

## Conventions

- Playbooks use active OpenShift context (no kubeconfig path needed, just `oc login` first)
- VMs are labeled `managed-by=ansible` for tracking
- ISO PVC referenced as `tools-iso-pvc` by default
- Network: pod network (masquerade) + Multus VLAN bridge
