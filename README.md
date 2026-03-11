# homelab-v2

New homelab setup mainly relying on K3s-related tech stack to manage services.

## Overview

This repository contains Ansible playbooks and configuration files for deploying and managing a K3s Kubernetes cluster on a homelab infrastructure. It uses Ansible Vault to securely store sensitive information like IP addresses, usernames, and SSH keys.

## Prerequisites

- Python 3.8+
- Virtual environment (venv)
- SSH access to cluster hosts
- Ansible Vault password file (`.vault_password`)

## Setup

### 1. Clone the repository

```bash
git clone <repo-url>
cd homelab-v2
```

### 2. Create and activate virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r dependencies/pip/requirements.txt
```

### 4. Configure Ansible Vault

The vault password should be stored in `.vault_password` at the repository root:

```bash
echo "your-secure-password" > .vault_password
chmod 600 .vault_password
```

**Note:** `.vault_password` is already in `.gitignore` and should NEVER be committed to version control.

### 5. Configure vault variables

Edit the encrypted vault file with your sensitive data:

```bash
ansible-vault edit ansible/inventory/group_vars/homelab_vault.yml --vault-password-file=.vault_password
```

Add required variables:
```yaml
---
vault_ansible_user: <your_username>
vault_ansible_ssh_private_key_file: <path_to_ssh_key>
vault_ansible_become_password: <sudo_password>
```


## Directory Structure

```
homelab-v2/
├── ansible/
│   ├── ansible.cfg              # Ansible configuration
│   ├── playbooks/
│   │   ├── site.yml             # Master orchestration playbook
│   │   └── k3s.yml              # K3s cluster deployment playbook
│   ├── inventory/
│   │   ├── hosts.yml            # Host inventory
│   │   └── group_vars/
│   │       └── homelab_vault.yml # Encrypted sensitive variables
│   └── roles/                    # Reusable Ansible roles (future)
├── dependencies/
│   └── pip/
│       └── requirements.txt      # Python dependencies
├── .vault_password               # Vault password (NOT in git)
├── .gitignore
└── README.md
```

## Running Playbooks

### Dry Run (Check Mode)

Preview changes without applying them:

```bash
cd ansible/
ansible-playbook playbooks/site.yml --check
```

### Run Playbook

Execute the full deployment:

```bash
cd ansible/
ansible-playbook playbooks/site.yml
```

### Linting and Testing

```bash
# Lint Ansible playbooks
ansible-lint

# Check YAML syntax
yamllint ansible/

# Syntax check specific playbook
ansible-playbook playbooks/k3s.yml --syntax-check
```

## Interacting with K3s

### On the master node via SSH

```bash
ssh <master_host>
/usr/local/bin/k3s kubectl get nodes
/usr/local/bin/k3s kubectl get pods -A
```

## Dependencies

Core dependencies:
- **ansible**: Infrastructure automation
- **ansible-lint**: Playbook linting
- **yamllint**: YAML validation
- **ansible-compat**: Compatibility utilities

See `dependencies/pip/requirements.txt` for full list.

## License

See LICENSE file
