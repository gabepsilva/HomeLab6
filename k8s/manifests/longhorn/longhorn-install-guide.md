# Installing Longhorn on Kubernetes using Helm

This guide provides step-by-step instructions for installing Longhorn on a Kubernetes cluster using Helm.

## Prerequisites

- A Kubernetes cluster (v1.18 or later)
- Helm 3 installed
- kubectl configured to communicate with your cluster
- Each node in the cluster must have the following components:
  - `open-iscsi` installed and `iscsid` daemon running
  - `bash`
  - `curl`
  - `findmnt`
  - `grep`
  - `awk`
  - `blkid`
  - `lsblk`
  - `mount`
  - `mkfs.ext4`
  - `docker` or container runtime

### Installing Prerequisites on Debian/Ubuntu

For Debian/Ubuntu-based systems, you can install the required dependencies with:

```bash
# Update package lists
sudo apt update

# Install the required packages
sudo apt install -y open-iscsi curl util-linux grep gawk mount e2fsprogs

# Enable and start the iscsid service
sudo systemctl enable iscsid
sudo systemctl start iscsid
```

For Docker installation, follow the [official Docker installation guide](https://docs.docker.com/engine/install/ubuntu/).

## Installation Steps

### 1. Add the Longhorn Helm repository

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
```

### 2. Configure Storage Options (Optional)

To specify custom storage directories, you can use a values.yaml file during installation:

```yaml
defaultSettings:
  defaultDataPath: /path/to/your/storage
```

### 3. Install Longhorn

```bash
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace -f values.yaml
```

### 4. Verify the Installation

Wait for all the Longhorn pods to be running:

```bash
kubectl -n longhorn-system get pods -w
```

### 5. Access the Longhorn UI

Set up port forwarding to access the Longhorn UI:

```bash
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8000:80
```

Then navigate to `http://localhost:8000` in your web browser.

Alternatively, you can create an Ingress to expose the Longhorn UI externally.



### This command shows which folder LongHorn is using 
```bash
 kubectl -n longhorn-system get nodes.longhorn.io -o jsonpath='{.items[*].spec.disks}'

```

## Post-Installation Configuration

### Adding Additional Storage Disks

To add additional disks to specific nodes:

```bash
kubectl edit node.longhorn.io <node-name>
```

Then add additional disks under the `spec.disks` section:

```yaml
spec:
  disks:
    custom-disk:
      path: /mnt/extra-storage
      allowScheduling: true
      storageReserved: 0
```

# Uninstall Longhorn


```
kubectl -n longhorn-system patch settings.longhorn.io deleting-confirmation-flag --type='json' -p='[{"op": "replace", "path": "/value", "value": "true"}]'

helm uninstall longhorn -n longhorn-system

kubectl delete namespace longhorn-system
```


# Longhorn Commands

# Viewing Statically Created Longhorn Volumes with kubectl

To view statically created Longhorn volumes using kubectl, you need to look at the PersistentVolume (PV) and PersistentVolumeClaim (PVC) resources. Here are the commands you can use:

## List all PersistentVolumes

```bash
kubectl get pv
```

To see more details about Longhorn volumes specifically:

```bash
kubectl get pv -o wide | grep longhorn
```

For detailed information about a specific PV:

```bash
kubectl describe pv <pv-name>
```

## List all PersistentVolumeClaims

```bash
kubectl get pvc --all-namespaces
```

To filter for a specific namespace:

```bash
kubectl get pvc -n <namespace>
```

## View Longhorn-specific resources

Longhorn creates custom resources for its volumes:

```bash
kubectl get volumes.longhorn.io -n longhorn-system
```

To get detailed information about a specific Longhorn volume:

```bash
kubectl describe volumes.longhorn.io <volume-name> -n longhorn-system
```

## View the Longhorn nodes

```bash
kubectl get nodes.longhorn.io -n longhorn-system
```

## Check Longhorn engine instances

```bash
kubectl get engines.longhorn.io -n longhorn-system
```

These commands will help you identify and inspect statically created Longhorn volumes in your Kubernetes cluster.
