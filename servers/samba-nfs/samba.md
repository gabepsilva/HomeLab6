
## Samba Server Runbook (Concise)

### Host
- Hostname: samba-nfs (FQDN: samba-nfs.i.psilva.org)
- Address: 10.10.0.160/20
- OS: Ubuntu 24.04 LTS
- Services: SMB/CIFS (445,139), NFS (2049,111)

### Install
```bash
sudo apt update
sudo apt install -y samba samba-common-bin nfs-kernel-server nfs-common rpcbind
```

### Directories (examples)
```bash
sudo mkdir -p /srv/samba/{gabriel,media,public,syncthing-gpool1}
sudo chown -R gabriel:gabriel /srv/samba
sudo chmod -R 755 /srv/samba
```

### Samba Config (minimal)
Edit `/etc/samba/smb.conf`, define shares as needed (example):
```
[gabriel]
   path = /srv/samba/gabriel
   browseable = yes
   read only = no
   guest ok = no
```
Apply and enable:
```bash
sudo systemctl enable --now smbd nmbd
sudo testparm -s
```

### Users
```bash
sudo smbpasswd -a gabriel
```

### NFS (see nfs.md for details)
- Exports are managed separately; this host also serves NFS.

### Client Access
- SMB: `smb://samba-nfs.i.psilva.org` (or `smb://10.10.0.160`)
- NFS (path examples): `10.10.0.160:/srv/samba/[share]`

### Maintenance
```bash
# Status
systemctl is-active smbd nmbd

# Restart
sudo systemctl restart smbd nmbd

# Logs
journalctl -u smbd -n 200 --no-pager
```

### Troubleshooting
- Connection refused: ensure `smbd` and `nmbd` are active.
- Auth failures: reset Samba password with `sudo smbpasswd username`.
- Permission denied: check directory ownership under `/srv/samba`.

### Notes
- FQDN alias: `samba-nfs.i.psilva.org` resolves to `10.10.0.160`.
