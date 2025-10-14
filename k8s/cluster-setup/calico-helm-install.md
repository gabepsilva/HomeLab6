# Installing Calico with Helm on Kubernetes 1.32.3

This guide provides step-by-step instructions for installing Calico with VXLAN overlay networking using Helm on a Kubernetes 1.32.3 cluster.

## Prerequisites

- Kubernetes 1.32.3 cluster with at least one master and one worker node
- `kubectl` configured to work with your cluster
- Helm 3 installed on your system

## Step 1: Add the Calico Helm Repository

```bash
helm repo add projectcalico https://docs.tigera.io/calico/charts
helm repo update
```

## Step 2: Create the Tigera Operator Namespace

```bash
kubectl create namespace tigera-operator
```

## Step 3: Create a Values File for VXLAN Overlay Networking

Create a file named `calico-values.yaml` with the following content:

```yaml
installation:
  calicoNetwork:
    # Configure IP pools based on your cluster's pod CIDR
    ipPools:
    - cidr: 10.245.0.0/16  # Update this to match your cluster's pod CIDR
      encapsulation: VXLAN  # Using VXLAN for overlay networking
      natOutgoing: Enabled
      blockSize: 26
    # Disable BGP as we're using VXLAN overlay
    bgp: Disabled
```

## Step 4: Install Calico using Helm

```bash
helm install calico projectcalico/tigera-operator \
  --version v3.30 \
  --namespace tigera-operator \
  -f calico-values.yaml
```

## Step 5: Monitor the Installation

Monitor the installation progress using:

```bash
kubectl get pods -n tigera-operator
kubectl get pods -n calico-system
```

Wait until all pods are in `Running` state.

## Step 6: Verify the Installation

Check the status of Calico components:

```bash
kubectl get tigerastatus
```

All components should show `AVAILABLE: True` and `DEGRADED: False`.

## Step 7: Verify IP Pool Configuration

Ensure the IP pool is configured correctly with VXLAN overlay:

```bash
kubectl get ippools default-ipv4-ippool -o yaml
```

Look for:
- `cidr: 10.245.0.0/16` (or your configured CIDR)
- `encapsulation: VXLAN` or `vxlanMode: Always`

## Step 8: Verify Network Connectivity

After installation, your Kubernetes nodes should transition to the `Ready` state. Verify this with:

```bash
kubectl get nodes
```

## Conclusion

You've successfully installed Calico with VXLAN overlay networking using Helm on your Kubernetes cluster. Your pods can now communicate across nodes using Calico's networking capabilities.

To learn more about configuring network policies and advanced Calico features, refer to the [official Calico documentation](https://docs.tigera.io/calico/latest/). 