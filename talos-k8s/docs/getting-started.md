# Getting Started

This guide covers the initial setup of your Talos Kubernetes cluster.

## Prerequisites

- `talosctl` installed on your workstation
- Network access to the target node (10.10.0.194)
- Talos OS installed on the target machine

## Generate Configuration

Run the following command to generate the cluster configuration:

```bash
talosctl gen config <cluster-name> https://10.10.0.194:6443
```

Replace `<cluster-name>` with your desired cluster name (e.g., `homelab`).

### Generated Files

| File | Description |
|------|-------------|
| `controlplane.yaml` | Configuration for control plane nodes |
| `worker.yaml` | Configuration for worker nodes |
| `talosconfig` | Client configuration for `talosctl` |

## Next Steps

1. Apply the control plane configuration to your node
2. Bootstrap the cluster
3. Retrieve the kubeconfig

See the [Installation Guide](installation/index.md) for detailed steps.
