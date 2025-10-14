# NFS Server Runbook (Concise)

## Host
- Hostname: samba-nfs
- FQDN: samba-nfs.i.psilva.org
- Address: 10.10.0.160/20
- OS: Ubuntu 24.04 LTS
- Ports: 2049, 111 (TCP/UDP)

## Install
```bash
sudo apt update
sudo apt install -y nfs-kernel-server nfs-common rpcbind
```

## Directories
```bash
sudo mkdir -p /srv/nfs/k8s-nfs
sudo chown ubuntu:ubuntu /srv/nfs/k8s-nfs
sudo chmod 755 /srv/nfs/k8s-nfs
```

## Exports (current)
Edit `/etc/exports`:
```
/srv/nfs/k8s-nfs	10.10.0.0/20(rw,sync,no_subtree_check,no_root_squash,insecure,fsid=0)
/srv/nfs/k8s-nfs	10.20.0.0/20(rw,sync,no_subtree_check,no_root_squash,insecure,fsid=0)
```
Notes:
- `fsid=0` makes this the NFSv4 root. For v4, mount `10.10.0.160:/`.
- Avoid typos: option list must not contain stray spaces (e.g., `insecure,fsid=0`).

## Apply and Verify
```bash
sudo systemctl enable rpcbind nfs-kernel-server
sudo systemctl start rpcbind nfs-kernel-server

# Apply exports
sudo exportfs -ra

# Verify
showmount -e 10.10.0.160
sudo exportfs -v
```

## Client Mount Examples
```bash
# NFSv3 (path-based)
sudo mount -t nfs -o vers=3 10.10.0.160:/srv/nfs/k8s-nfs /mnt/nfs-k8s

# NFSv4 (root is fsid=0)
sudo mount -t nfs -o vers=4 10.10.0.160:/ /mnt/nfs-k8s
```

Optional `/etc/fstab` entries:
```
10.10.0.160:/srv/nfs/k8s-nfs /mnt/nfs-k8s nfs defaults,_netdev 0 0   # v3
10.10.0.160:/                 /mnt/nfs-k8s nfs defaults,_netdev 0 0   # v4
```

## Maintenance
```bash
# Re-export after edits
sudo exportfs -ra

# Reload service (if needed)
sudo systemctl reload nfs-kernel-server || sudo systemctl restart nfs-kernel-server

# Show active exports
showmount -e 10.10.0.160
sudo exportfs -v

# Health
systemctl is-active nfs-kernel-server rpcbind
rpcinfo -p 10.10.0.160
```

## Troubleshooting
- Permission denied from a 10.20.x client: ensure `10.20.0.0/20` is in `/etc/exports` and run `sudo exportfs -ra`.
- NFSv4 path fails: when `fsid=0` is on `/srv/nfs/k8s-nfs`, mount `:/` instead of the full path.
- Check logs: `journalctl -u nfs-kernel-server`, `journalctl -u rpcbind`, and `journalctl -n 200 | egrep -i 'mountd|nfsd|refused|unmatched'`.

## Current State (verified)
- Exports active: `/srv/nfs/k8s-nfs` to `10.10.0.0/20` and `10.20.0.0/20`
- Services: `nfs-kernel-server` active, `rpcbind` active
- Directory: `/srv/nfs/k8s-nfs` exists (owned by `ubuntu:ubuntu`, 755)
