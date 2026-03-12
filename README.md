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

### Local kubectl (Recommended)

The K3s playbook automatically fetches the kubeconfig from the master node and saves it to `~/.kube/config` on your workstation during deployment.

**Prerequisites:**
- `kubectl` installed locally on your workstation

**Install kubectl:**
```bash
# macOS
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Or use your package manager
sudo apt install kubectl  # Debian/Ubuntu
```

**After running the K3s playbook:**
```bash
# Test connectivity
kubectl cluster-info
kubectl get nodes
kubectl get pods -A
```

### Via SSH to the master node (Alternative)

If kubectl is not installed locally, you can run commands directly on the master:

```bash
ssh <master_host>
/usr/local/bin/k3s kubectl get nodes
/usr/local/bin/k3s kubectl get pods -A
```

## ArgoCD

### Configure GitHub Access for Private Repositories

If your GitHub repository is private, ArgoCD needs SSH credentials to access it.

**Create the SSH credentials secret:**

```bash
# Create secret directly from your private key file (never in a manifest)
kubectl create secret generic github-repo-creds \
  -n argocd \
  --from-file=sshPrivateKey=$HOME/.ssh/id_ed25519 \
  --dry-run=client -o yaml | kubectl apply -f -

# Label it for ArgoCD
kubectl patch secret github-repo-creds -n argocd \
  -p '{"metadata":{"labels":{"argocd.argoproj.io/secret-type":"repository"}}}'

# Patch in the Git URL (replace with your username)
kubectl patch secret github-repo-creds -n argocd \
  -p '{"stringData":{"type":"git","url":"git@github.com:<your-username>/homelab-v2.git"}}'
```

**Then deploy applications:**

```bash
kubectl apply -f manifests/apps/jellyfin.yml
```

**Important Security Notes:**
- Never commit SSH private keys to version control
- The secret is stored in Kubernetes (encrypted if sealed secrets are configured)
- For production, consider using deploy keys or service accounts instead of personal SSH keys

## Dependencies

**Ansible Dependencies (installed via pip):**
- **ansible**: Infrastructure automation
- **ansible-lint**: Playbook linting
- **yamllint**: YAML validation
- **ansible-compat**: Compatibility utilities

See `dependencies/pip/requirements.txt` for full list.

**Local Dependencies (for cluster interaction):**
- **kubectl**: Kubernetes command-line tool (required for local cluster interaction after K3s deployment)
- **ssh**: For SSH access to remote hosts (usually pre-installed)

## License

See LICENSE file
