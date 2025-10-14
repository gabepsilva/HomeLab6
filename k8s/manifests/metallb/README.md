# Installing MetalLB on Kubernetes with Helm

This guide walks through installing MetalLB using Helm with an IP address range of `10.0.1.0-10.0.1.50` (configured for networks with 10.0.0.1/20 subnet).

## Prerequisites

- Kubernetes cluster (version 1.23 or later)
- Helm v3
- `kubectl` access to your cluster

## Installation Steps

### 1. Enable strictARP in kube-proxy ConfigMap

MetalLB requires `strictARP` to be enabled in the kube-proxy configuration:

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

### 2. Create Namespace with Pod Security Admission Labels

Create the MetalLB namespace with the required pod security admission labels:

```bash
kubectl create namespace metallb-system
kubectl label namespace metallb-system \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged
```

### 3. Install MetalLB via Helm

Add the MetalLB Helm repository:

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
```

Install MetalLB:

```bash
helm install metallb metallb/metallb -n metallb-system
```

### 4. Configure IP Address Pool

Create an IP address pool configuration file named `metallb-ipaddresspool.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.10.14.1-10.10.14.255
```

Apply the configuration:

```bash
kubectl apply -f metallb-ipaddresspool.yaml
```

### 5. Configure L2 Announcement

Create an L2 advertisement configuration file named `metallb-l2advertisement.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
```

Apply the configuration:

```bash
kubectl apply -f metallb-l2advertisement.yaml
```

## Verification

Check that MetalLB pods are running:

```bash
kubectl get pods -n metallb-system
```

Test MetalLB by creating a LoadBalancer service:

```bash
kubectl create deployment nginx2 --image=nginx
kubectl expose deployment nginx2 --port=80 --type=LoadBalancer
kubectl get services
```

The service should receive an IP address from the configured range.
