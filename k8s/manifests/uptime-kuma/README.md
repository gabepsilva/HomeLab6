## Uptime Kuma manifests

Use the YAML files in this directory to deploy Uptime Kuma with NFS-backed persistence.

Order:
1. 01-namespace.yaml
2. 02-certificate.yaml
3. 02-pvc.yaml
4. 03-deployment.yaml
5. 04-service.yaml
6. 05-ingress.yaml (optional)

Example apply:
```bash
kubectl apply -f 01-namespace.yaml
kubectl apply -f 02-certificate.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-deployment.yaml
kubectl apply -f 04-service.yaml
kubectl apply -f 05-ingress.yaml  # optional
```



