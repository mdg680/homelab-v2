# qBittorrent + WireGuard

qBittorrent deployment with a WireGuard sidecar that tunnels all torrent traffic through a VPN.

## Architecture

All containers in the pod share the same network namespace. Traffic flow:

```
internet <--> WireGuard (wg0) <--> qBittorrent
```

- **`wireguard-init`** (initContainer) — brings up `wg0` before qBittorrent starts
- **`wireguard`** (sidecar) — monitors the tunnel and re-establishes it if it drops
- **`qbittorrent`** — all outbound traffic is transparently routed through `wg0`

### Kill switch

An iptables kill switch is configured via `PostUp`/`PreDown` in `wg0.conf`. It rejects any outbound packet that does not leave through `wg0` the moment the tunnel comes up. If WireGuard crashes without a clean shutdown, the `PostUp` rule remains in place and blocks all traffic, preventing any leakage on the plain connection.

## Storage

| PVC | Mount | NFS Path |
|-----|-------|----------|
| `qbittorrent-config-pvc` | `/config` | `/volume1/Metadata/qbittorrent/config` |
| `qbittorrent-downloads-pvc` | `/downloads` | `/volume1/Downloads` |
| `qbittorrent-media-pvc` | `/media` | `/volume1/Media` |
| `qbittorrent-data-pvc` | `/data` | `/volume1/Data` |

## Sealed Secrets setup

Sealed Secrets encrypts Kubernetes secrets with the cluster's public key so they are safe to commit to git. Only the cluster can decrypt them.

### Install the controller

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml
```

Verify it's running:

```bash
kubectl get pods -n kube-system | grep sealed-secrets
```

### Install the kubeseal CLI

Download the correct binary — do not use the generic `/releases/latest/download/kubeseal-linux-amd64` URL as it may return a 404 page instead of the binary:

```bash
VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest | grep '"tag_name"' | cut -d'"' -f4)
sudo curl -sL "https://github.com/bitnami-labs/sealed-secrets/releases/download/${VERSION}/kubeseal-$(echo $VERSION | tr -d v)-linux-amd64.tar.gz" \
  | sudo tar -xz -C /usr/local/bin kubeseal
sudo chmod +x /usr/local/bin/kubeseal
kubeseal --version
```

> If `kubeseal` gives `/usr/local/bin/kubeseal: line 1: Not: command not found`, the binary is a 404 error page. Re-download using the versioned URL above.

### Important: namespace is baked into the seal

SealedSecrets are scoped to a specific namespace. If you rename the namespace, the seal becomes invalid and you must re-seal with the new namespace name. After renaming:

1. Recreate `secret.yml` with the updated `namespace:` field
2. Re-seal it (see below)
3. Release the old PVs so they can rebind (see Troubleshooting)

## Updating the WireGuard config

The WireGuard config is stored as a `SealedSecret` — encrypted with the cluster's public key so it is safe to commit to git.

**1. Edit `secret.yml` with your VPN provider's WireGuard config:**

```yaml
stringData:
  wg0.conf: |
    [Interface]
    PrivateKey = <your-private-key>
    Address = <vpn-assigned-ip>/32
    DNS = <vpn-dns>
    PostUp = iptables -I OUTPUT ! -o wg0 -m mark ! --mark $(wg show wg0 fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
    PreDown = iptables -D OUTPUT ! -o wg0 -m mark ! --mark $(wg show wg0 fwmark) -m addrtype ! --dst-type LOCAL -j REJECT

    [Peer]
    PublicKey = <server-public-key>
    AllowedIPs = 0.0.0.0/0, ::/0
    Endpoint = <vpn-endpoint>:<port>
    PersistentKeepalive = 25
```

**2. Re-seal it:**

```bash
grep -v '^#' secret.yml | kubeseal \
  --controller-namespace kube-system \
  --controller-name sealed-secrets-controller \
  --format yaml > sealed-secret.yml
```

**3. Delete the plaintext file and commit `sealed-secret.yml`:**

```bash
rm secret.yml
git add sealed-secret.yml
git commit -m "chore: update wireguard sealed secret"
```

> `secret.yml` is in `.gitignore` and must never be committed.

## Accessing the Web UI

- **In-cluster:** `http://qbittorrent.qbittorrent:8080`
- **Via Traefik:** `http://qbittorrent.home`
- **NodePort:** `http://<node-ip>:30081`

Default credentials: `admin` / `adminadmin` — change these immediately on first login.

### Verify VPN is active

```bash
kubectl exec -n qbittorrent deployment/qbittorrent -c qbittorrent -- curl -s https://ipinfo.io/ip
```

The returned IP should be your VPN provider's exit node, not your home IP.

## Troubleshooting

### PVs stuck in `Released` state after namespace rename

When the namespace changes, old PVCs are deleted but the PVs retain a `claimRef` to the old namespace and won't bind to new PVCs. Fix by clearing the `claimRef`:

```bash
for pv in qbittorrent-config-pv qbittorrent-data-pv qbittorrent-downloads-pv qbittorrent-media-pv; do
  kubectl patch pv $pv --type=merge -p '{"spec":{"claimRef":null}}'
done
```

Verify PVs return to `Available`:

```bash
kubectl get pv | grep qbittorrent
```

### NFS mount access denied

If pods fail with `mount.nfs: access denied by server`, the NFS export is missing on the NAS.

Verify exports:
```bash
showmount -e 192.168.1.146
```

For each missing path, add a shared folder on Synology DSM:
- **Control Panel → Shared Folders → Create** (create the folder if it doesn't exist)
- **Edit → NFS Permissions → Create** with:
  - Hostname/IP: `192.168.1.0/24`
  - Privilege: `Read/Write`
  - Squash: `No mapping`
  - Security: `sys`
  - Allow connections from non-privileged ports: ✓
  - Allow users to access mounted subfolders: ✓

Then bounce the pods to retry the mount:
```bash
kubectl delete pods -n qbittorrent --all
```

### NodePort already allocated

If applying the Service fails with `provided port is already allocated`, check what ports are in use:

```bash
kubectl get svc -A | grep NodePort
```

Then update `nodePort` in `service.yml` to a free port in the `30000-32767` range.

### ErrUnsealFailed — no key could decrypt secret

The SealedSecret was sealed against a different namespace or cluster. Re-seal with the correct namespace (see **Sealed Secrets setup** above).
