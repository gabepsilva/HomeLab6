# NGINX Ingress Controller Installation

This directory contains configurations for deploying the NGINX Ingress Controller on Kubernetes v1.32.3.

## Prerequisites

- Kubernetes cluster running v1.32.3
- Helm v3 installed
- `kubectl` configured to communicate with your cluster

## Installation Steps

1. Add the NGINX Ingress repository:
   ```bash
   helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
   helm repo update
   ```

2. Install NGINX Ingress Controller:
   ```bash
   helm install nginx-ingress ingress-nginx/ingress-nginx -f values.yaml -n ingress-nginx --create-namespace
   ```

3. Verify the installation:
   ```bash
   kubectl get pods -n ingress-nginx
   kubectl get svc -n ingress-nginx
   ```

## Configuration

The `values.yaml` file contains custom configurations for the NGINX Ingress Controller:

- 2 controller replicas for high availability
- LoadBalancer service type with Local external traffic policy
- Resource requests and limits for controller and default backend
- Metrics enabled for Prometheus integration
- Common proxy configurations

