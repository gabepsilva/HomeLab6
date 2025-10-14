# Kubernetes Monitoring Stack

This directory contains the configuration and setup instructions for monitoring your Kubernetes homelab using Prometheus and Grafana.

## Overview

The monitoring stack includes:
- **Prometheus**: Metrics collection and storage
- **Grafana**: Visualization and dashboards
- **Node Exporter**: Node-level metrics
- **Alertmanager**: Alerting (optional)

## Prerequisites

- Helm 3.x installed
- Kubernetes cluster running
- Sufficient storage for metrics retention

## Installation

### 1. Add Helm Repository

```bash
# Add the Prometheus community Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 2. Configure Values

A pre-configured `monitoring-values.yaml` file is provided in this directory with homelab-optimized settings. You can customize it as needed:

**Important**: Before installation, update the Grafana admin password in the values file:

```bash
# Edit the values file to set your secure password
nano monitoring-values.yaml
```

Change the line:
```yaml
adminPassword: "your-secure-password-here"  # Change this!
```

### 3. Install the Monitoring Stack

```bash
# Install Prometheus and Grafana
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f monitoring-values.yaml
```

### 4. Verify Installation

```bash
# Check if all pods are running
kubectl get pods -n monitoring

# Check services
kubectl get svc -n monitoring
```

## Accessing Grafana

### Port Forward (for testing)

```bash
# Forward Grafana port locally
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Then access Grafana at: http://localhost:3000

- **Username**: `admin`
- **Password**: The password you set in `monitoring-values.yaml`

### Ingress Setup (recommended for production)

To expose Grafana through your ingress controller with SSL/TLS, apply the provided YAML files:

```bash
# Apply the certificate first
kubectl apply -f grafana-certificate.yaml

# Then apply the ingress
kubectl apply -f grafana-ingress.yaml
```

**Note**: Update `grafana.yourdomain.com` in both files to match your domain before applying.

## Configuration

### Storage Requirements

Adjust storage sizes in `monitoring-values.yaml` based on your needs:

- **Prometheus**: 50Gi (adjust based on retention period and cluster size)
- **Grafana**: 10Gi (for dashboards and configuration)
- **Alertmanager**: 10Gi (for alert history)

### Resource Limits

The current configuration uses conservative resource requests suitable for homelab environments. Adjust based on your cluster capacity:

```yaml
prometheus:
  prometheusSpec:
    resources:
      requests:
        cpu: 500m      # Adjust based on your cluster
        memory: 2Gi    # Adjust based on your cluster
      limits:
        cpu: 1000m     # Optional: set limits
        memory: 4Gi    # Optional: set limits
```

### Retention Policy

Default retention is set to 15 days. Adjust based on your storage capacity:

```yaml
prometheus:
  prometheusSpec:
    retention: 15d  # Options: 1d, 7d, 15d, 30d, etc.
```

## Useful Dashboards

After installation, Grafana comes with several pre-configured dashboards:

1. **Kubernetes Cluster Monitoring**: Overview of cluster health
2. **Node Exporter**: Individual node metrics
3. **Kubernetes Pod Monitoring**: Pod-level metrics
4. **Kubernetes Deployment**: Deployment monitoring

## Troubleshooting

### Check Pod Status

```bash
kubectl get pods -n monitoring
kubectl describe pod <pod-name> -n monitoring
```

### Check Logs

```bash
kubectl logs -n monitoring <pod-name>
```

### Check Storage

```bash
kubectl get pvc -n monitoring
kubectl describe pvc <pvc-name> -n monitoring
```

### Common Issues

1. **Storage Issues**: Ensure your storage class supports the required access modes
2. **Resource Constraints**: Increase resource limits if pods are being evicted
3. **Network Issues**: Check if your ingress controller is properly configured

## Uninstallation

To remove the monitoring stack:

```bash
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```

## Additional Resources

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Kube-Prometheus-Stack Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
