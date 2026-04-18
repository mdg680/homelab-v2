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

### Apply

```bash
ansible-playbook ansible/playbooks/pihole.yml
```

### Verify

```bash
nslookup your-new-service.home
```
