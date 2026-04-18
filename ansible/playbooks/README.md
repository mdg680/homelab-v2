# Ansible Playbooks

## pihole.yml — Pi-hole DNS records

Manages local DNS records on the Pi-hole instance running in the cluster.

### Adding a new domain

Edit `pihole.yml` and add the domain to `custom_domains`:

```yaml
custom_domains:
  - whoami.home
  - jellyfin.home
  - qbittorrent.home
  - your-new-service.home  # <-- add here
```

All domains point to the same `traefik_ip` (the load balancer IP), and Traefik routes to the correct service based on the `Host()` rule in each service's `IngressRoute`.

### Setting the Pi-hole web password

The playbook no longer manages the Pi-hole password. To set it manually:

```bash
kubectl exec -n pihole $(kubectl get pod -n pihole -l app=pihole -o jsonpath='{.items[0].metadata.name}') -- pihole setpassword
```

You will be prompted to enter a password interactively. The password is stored in the vault as `vault_pihole_password` in `ansible/inventory/group_vars/homelab_vault.yml`.

### Apply

```bash
ansible-playbook ansible/playbooks/pihole.yml
```

### Verify

```bash
nslookup your-new-service.home
```
