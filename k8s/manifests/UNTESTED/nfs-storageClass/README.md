##
## HAVING ISSUES WITH NFS OVER VIRTIO FOLDER!, THIS WAS UNDEPLOYED!
##


## NFS StorageClass on Kubernetes (Helm + kubectl)

This guide shows how to set up a dynamic NFS-backed StorageClass using the NFS Subdir External Provisioner. It uses Helm for installation and kubectl for verification/management.

- NFS server: `samba-nfs.i.psilva.org`
- You will need the exported NFS directory path on that server (example used below: `/export/k8s`).

### Prerequisites
- kubectl configured with the correct context for your cluster
- Helm v3 installed
- Network connectivity and DNS resolution from cluster nodes to `samba-nfs.i.psilva.org`
- The NFS export on the server is readable/writable by your worker nodes

### 1) Inspect cluster state (optional but recommended)
```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get storageclass
```

### 2) Create a namespace for the provisioner
```bash
kubectl create namespace nfs-storage --dry-run=client -o yaml | kubectl apply -f -
```

### 3) Add the Helm repo and update
We will use the community chart from the Kubernetes SIGs.
```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update
```

### 4) Install the NFS Subdir External Provisioner via Helm


The example below also creates a StorageClass named `nfs-client` and sets it as the default StorageClass.

```bash
helm upgrade --install nfs-client nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace nfs-storage \
  --set nfs.server=samba-nfs.i.psilva.org \
  --set nfs.path=/srv/nfs/k8s-nfs \
  --set storageClass.name=nfs-client \
  --set storageClass.defaultClass=false \
  --set storageClass.allowVolumeExpansion=true \
  --set storageClass.reclaimPolicy=Retain \
  --set storageClass.archiveOnDelete=true
```

Notes:
- `archiveOnDelete=true` will keep a copy of data under an `archived-` folder when PVCs are deleted.
- If you do not want this StorageClass to be the default, set `storageClass.defaultClass=false`.

### 5) Verify installation
```bash
kubectl get pods -n nfs-storage -o wide
kubectl get deploy,rs,svc -n nfs-storage
kubectl get storageclass
```

You should see a `nfs-client` StorageClass (or your chosen name). If it is default, it will have `(default)` next to it.

### 6) (Optional) Make `nfs-client` the default StorageClass via kubectl
If you chose not to set it as default during Helm install, you can patch it later:
```bash
kubectl annotate storageclass nfs-client storageclass.kubernetes.io/is-default-class="true" --overwrite
```

If an existing StorageClass is default, remove the annotation there first:
```bash
kubectl annotate storageclass <existing-default> storageclass.kubernetes.io/is-default-class- --overwrite
```

### 7) Test with a sample PVC and Pod
Apply a simple test to ensure dynamic provisioning works.

Create `test-pvc.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-client-test-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-client
  resources:
    requests:
      storage: 1Gi
```

Create `test-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-client-test-pod
spec:
  containers:
    - name: app
      image: busybox:stable
      command: ["sh", "-c", "echo NFS works! > /data/hello.txt && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: nfs-client-test-pvc
```

Apply and validate:
```bash
kubectl apply -f test-pvc.yaml
kubectl apply -f test-pod.yaml

kubectl get pvc
kubectl get pods
kubectl exec -it nfs-client-test-pod -- cat /data/hello.txt
```

On success, you should see `NFS works!` and the corresponding directory created on the NFS server under the configured export path.

Cleanup test objects when done:
```bash
kubectl delete -f test-pod.yaml -f test-pvc.yaml
```

### 8) Troubleshooting tips
- Ensure worker nodes can resolve and reach `samba-nfs.i.psilva.org` on NFS ports (typically TCP/UDP 2049; firewall/NAT rules may apply).
- Verify the NFS export path and permissions on the server. The provisioner writes subdirectories for each claim.
- Check controller logs:
  ```bash
  kubectl logs -n nfs-storage deploy/nfs-client-nfs-subdir-external-provisioner
  ```
- If using NetworkPolicies, allow traffic from the provisioner pod to the NFS server.

### 9) Uninstall
Remove the Helm release (will not delete PV data on the NFS server unless you manually remove it):
```bash
helm uninstall nfs-client -n nfs-storage
kubectl delete namespace nfs-storage
```

If you created a default StorageClass and want to revert, set another StorageClass to default or remove the annotation.


