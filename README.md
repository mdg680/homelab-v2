# homelab-v2

Modern homelab infrastructure using K3s, GitOps (ArgoCD), and automated DNS management with Pi-hole.

## Overview

This repository contains the complete infrastructure-as-code for a K3s-based homelab:
- **K3s cluster** deployed via Ansible
- **ArgoCD** for GitOps application management
- **Traefik** as reverse proxy with hostname-based routing
- **Pi-hole** for network-wide DNS with custom `.home` domains
- **Services**: Jellyfin media server, monitoring, and more

All services are accessible via friendly hostnames (e.g., `jellyfin.home`, `argocd.home`) without manual host file edits.

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

Required variables:
```yaml
---
vault_ansible_user: <your_username>
vault_ansible_ssh_private_key_file: <path_to_ssh_key>
vault_ansible_become_password: <sudo_password>
vault_k3s_master_ip: <k3s_master_ip>
vault_k3s_agent_ip: <k3s_agent_ip>
vault_loadbalancer_ip: <traefik_loadbalancer_ip>
vault_ssh_public_key: <your_ssh_public_key>
```


## Directory Structure

```
homelab-v2/
├── ansible/
│   ├── ansible.cfg              # Ansible configuration
│   ├── playbooks/
│   │   ├── site.yml             # Master orchestration playbook
│   │   ├── k3s.yml              # K3s cluster deployment
│   │   ├── argocd.yml           # ArgoCD installation
│   │   └── pihole.yml           # Pi-hole DNS configuration
│   ├── inventory/
│   │   ├── hosts.yml            # Host inventory
│   │   └── group_vars/
│   │       ├── homelab_vault.yml # Encrypted sensitive variables
│   │       └── homelab.yml       # Cluster configuration
│   └── roles/                    # Reusable Ansible roles (future)
├── manifests/
│   ├── apps/                     # ArgoCD Application definitions
│   │   ├── jellyfin.yml
│   │   ├── whoami.yml
│   │   └── pihole.yml
│   └── services/                 # Service manifests
│       ├── jellyfin/
│       ├── pihole/
│       ├── whoami/
│       └── argocd/
├── dependencies/
│   └── pip/
│       └── requirements.txt      # Python dependencies
├── .vault_password               # Vault password (NOT in git)
├── .gitignore
└── README.md
```

## Deployed Services

All services are accessible via custom `.home` domains:

| Service | URL | Description |
|---------|-----|-------------|
| ArgoCD | `http://argocd.home` | GitOps deployment dashboard |
| Jellyfin | `http://jellyfin.home` | Media server |
| Pi-hole | `http://pihole.home/admin` | DNS management (password: `admin`) |
| Whoami | `http://whoami.home` | Test service for routing verification |

**Network Setup:**
- Traefik LoadBalancer IP: `192.168.1.228`
- Pi-hole DNS: Port 53 on `192.168.1.228`
- All `.home` domains resolve through Pi-hole to Traefik
- Router WAN DNS configured to use Pi-hole

## Running Playbooks

### Deploy Complete Stack

```bash
cd ansible/
ansible-playbook playbooks/site.yml
```

### Deploy Individual Components

```bash
# Deploy K3s cluster
ansible-playbook playbooks/k3s.yml

# Install ArgoCD
ansible-playbook playbooks/argocd.yml

# Configure Pi-hole DNS records
ansible-playbook playbooks/pihole.yml --vault-password-file ../.vault_password
```

### Dry Run (Check Mode)

Preview changes without applying them:

```bash
cd ansible/
ansible-playbook playbooks/site.yml --check
```

## Adding New Services

1. **Create service manifests** in `manifests/services/<service-name>/`:
   - `namespace.yml`
   - `deployment.yml`
   - `service.yml`
   - `ingressroute.yml` (for Traefik routing)
   - `kustomization.yml` (to define resource order)

2. **Create ArgoCD Application** in `manifests/apps/<service-name>.yml`:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: myapp
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: git@github.com:your-user/homelab-v2.git
       targetRevision: master
       path: manifests/services/myapp
     destination:
       server: https://kubernetes.default.svc
       namespace: myapp
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
       syncOptions:
         - CreateNamespace=true
   ```

3. **Add DNS record** to `ansible/playbooks/pihole.yml`:
   ```yaml
   custom_domains:
     - whoami.home
     - jellyfin.home
     - myapp.home  # Add your new domain
   ```

4. **Apply changes**:
   ```bash
   # Commit and push manifests
   git add manifests/
   git commit -m "feat: add myapp service"
   git push origin master
   
   # Update DNS
   ansible-playbook ansible/playbooks/pihole.yml --vault-password-file .vault_password
   ```

ArgoCD will automatically detect and deploy the new application.

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

## Pi-hole DNS Configuration

Pi-hole provides network-wide DNS resolution for `.home` domains.

### Initial Setup

After deploying Pi-hole via ArgoCD, configure custom DNS records:

```bash
ansible-playbook ansible/playbooks/pihole.yml --vault-password-file .vault_password
```

This automatically:
- Adds all domains from `custom_domains` list to Pi-hole
- Configures dnsmasq to serve custom DNS records
- Points all `.home` domains to the Traefik LoadBalancer IP

### Router Configuration

**Configure your router to use Pi-hole:**
1. Go to router WAN settings
2. Set "Connect to DNS Server automatically" to **No**
3. Set DNS Server 1 to your Pi-hole IP (e.g., `192.168.50.193`)
4. Keep DNS Server 2 as your ISP's DNS for redundancy

Pi-hole will:
- Resolve `.home` domains locally
- Forward external domains to upstream DNS (your ISP)
- Provide ad-blocking for all network devices

### Manual DNS Management

Access Pi-hole admin at `http://pihole.home/admin` (password: `admin`):
- View DNS queries and statistics
- Add/remove custom DNS records
- Configure upstream DNS servers
- Manage blocklists

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

## Troubleshooting

### DNS not resolving `.home` domains

**Check if Pi-hole DNS is working:**
```bash
dig @192.168.50.193 jellyfin.home
nslookup jellyfin.home
```

**If `nslookup` shows wrong DNS server:**
- Check router DHCP/WAN DNS settings
- Manually set DNS on your device to Pi-hole IP
- Restart network adapter: `ipconfig /release && ipconfig /renew` (Windows)

**Add new DNS records:**
```bash
# Edit custom_domains in ansible/playbooks/pihole.yml
# Then run:
ansible-playbook ansible/playbooks/pihole.yml --vault-password-file .vault_password
```

### ArgoCD applications stuck "Progressing"

**Check pod status:**
```bash
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

**Check service status:**
```bash
kubectl get svc -n <namespace>
# Look for LoadBalancer services stuck in <pending>
```

**Common fixes:**
- Change LoadBalancer to ClusterIP if accessed via Traefik
- Check if hostPath volumes have correct permissions
- Verify NFS mounts are accessible

### Service shows "404 Not Found"

**Verify IngressRoute:**
```bash
kubectl get ingressroute -n <namespace>
kubectl describe ingressroute <name> -n <namespace>
```

**Test routing with curl:**
```bash
curl -H "Host: myapp.home" http://192.168.50.193
```

**Check Traefik logs:**
```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik
```

### Can't access ArgoCD dashboard

**Get ArgoCD admin password:**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

**ArgoCD redirecting to HTTPS:**
```bash
kubectl patch cm argocd-cmd-params-cm -n argocd --type merge -p '{"data":{"server.insecure":"true"}}'
kubectl rollout restart deployment argocd-server -n argocd
```

## License

See LICENSE file

