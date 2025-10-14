# Jenkins Deployment on Kubernetes with Helm

This guide provides step-by-step instructions to deploy Jenkins on Kubernetes using Helm and a custom values file.

## Prerequisites

- Kubernetes cluster (v1.19+)
- Helm 3.x installed
- kubectl configured to access your cluster
- Sufficient cluster resources (minimum 2GB RAM, 1 CPU core)
- Longhorn storage class available
- nginx ingress controller installed
- cert-manager installed
- OnePassword Connect Operator installed

## Quick Start

### 1. Add Jenkins Helm Repository

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
```

### 2. Create Namespace

```bash
kubectl create namespace jenkins
```

### 3. Create OnePassword Secret

```bash
kubectl apply -f jenkins-admin-secret.yaml
```

### 4. Deploy Jenkins with Full Values

```bash
helm install jenkins jenkins/jenkins \
  --namespace jenkins \
  --values values.yaml \
  --wait
```

### 5. Create Ingress (Manual Workaround)

The Helm chart ingress configuration has issues, so we create it manually:

```bash
kubectl apply -f jenkins-ingress.yaml
```

### 6. Access Jenkins

#### Via Ingress (Production)
- URL: https://jenkins.i.psilva.org
- Username: Use OnePassword
- Password: Use OnePassword

#### Port Forward (for testing)
```bash
kubectl --namespace jenkins port-forward svc/jenkins 8080:8080
```
Then access Jenkins at: http://localhost:8080

**Working Credentials:**
- Username: Use OnePassword
- Password: Use OnePassword

## Current Deployment Status

✅ **Working Components:**
- Jenkins StatefulSet running successfully (Latest LTS 2.504.2)
- OnePassword integration for admin credentials
- Longhorn persistent storage (20Gi)
- Service endpoints accessible
- Manual ingress with SSL/TLS
- Admin authentication configured via OnePassword

✅ **Successfully Deployed:**
- Jenkins version: 2.504.2-lts (Latest LTS)
- Credentials: OnePassword integration
- Storage: Longhorn 20Gi
- SSL: cert-manager with letsencrypt-cloudflare-issuer
- Plugins: Essential plugins installed (35+ plugins including Kubernetes, Git, Docker, BlueOcean, Pipeline tools)

## Installed Plugins

Jenkins now includes essential plugins for modern CI/CD workflows:

### Core Pipeline & Workflow
- `kubernetes` (v4353.vb_47977da_9417) - Kubernetes integration
- `workflow-aggregator` (v608.v67378e9d3db_1) - Pipeline suite
- `pipeline-stage-view` - Visual pipeline stages
- `pipeline-graph-analysis` - Pipeline visualization
- `blueocean` - Modern Jenkins UI

### Source Control & Integration  
- `git` (v5.7.0) - Git integration
- `github` (v1.43.0) - GitHub integration
- `github-branch-source` - GitHub branch discovery
- `pipeline-github-lib` - GitHub pipeline libraries

### Build Tools & Containers
- `docker-workflow` (v621.va_73f881d9232) - Docker pipeline steps
- `kubernetes` - Kubernetes agents and deployment
- `ant` & `gradle` - Build tool support

### UI & Visualization
- `blueocean` suite - Modern pipeline visualization
- `dashboard-view` - Custom dashboards
- `build-pipeline-plugin` - Pipeline views

### Utilities & Notifications
- `build-timeout` - Build timeouts
- `timestamper` - Build timestamps  
- `ws-cleanup` - Workspace cleanup
- `mailer` & `email-ext` - Email notifications
- `junit` - Test result publishing

## Configuration Files

### values.yaml
Full production configuration with:
- Latest Jenkins LTS version (2.504.2)
- OnePassword credentials integration
- Longhorn storage integration
- Resource limits configured
- Security settings
- Agent support enabled
- Essential plugins (35+ including Kubernetes, Git, Docker, BlueOcean)
- No plugin version locks (uses latest compatible)

### jenkins-admin-secret.yaml
OnePassword integration:
- Fetches credentials from `vaults/Dev/items/JenkinsAdmin`
- Creates Kubernetes secret automatically
- Username: `jenkinsAdmin`
- Password: Retrieved from OnePassword

### jenkins-ingress.yaml
Manual ingress configuration:
- nginx ingress controller
- SSL/TLS with cert-manager
- Automatic certificate generation

## Useful Commands

### Check Status
```bash
kubectl get pods,svc,pvc,ingress -n jenkins
```

### View Logs
```bash
kubectl logs -f statefulset/jenkins -n jenkins -c jenkins
```

### Check OnePassword Secret
```bash
kubectl get onepassworditem jenkins-admin-secret -n jenkins
kubectl get secret jenkins-admin-secret -n jenkins
```

### Upgrade Jenkins
```bash
helm upgrade jenkins jenkins/jenkins \
  --namespace jenkins \
  --values values.yaml
```

### Uninstall Jenkins
```bash
helm uninstall jenkins --namespace jenkins
kubectl delete -f jenkins-ingress.yaml
kubectl delete -f jenkins-admin-secret.yaml
kubectl delete namespace jenkins
```

## Troubleshooting

### Common Issues

1. **Pod stuck in Pending**: Check Longhorn storage availability
2. **Ingress not working**: Verify nginx ingress controller and cert-manager
3. **SSL certificate issues**: Check cert-manager logs and cluster issuer
4. **OnePassword secret not created**: Check OnePassword Connect Operator

### Debug Commands
```bash
# Check pod status and events
kubectl describe pod jenkins-0 -n jenkins

# Check persistent volume claims
kubectl get pvc -n jenkins

# Check ingress status
kubectl describe ingress jenkins-ingress -n jenkins

# Check certificate status
kubectl get certificate -n jenkins

# Check OnePassword item status
kubectl describe onepassworditem jenkins-admin-secret -n jenkins
```

## Storage Configuration

Jenkins uses Longhorn distributed storage:
- **Storage Class**: `longhorn`
- **Size**: 20Gi
- **Access Mode**: ReadWriteOnce
- **Backup**: Automatic via Longhorn snapshots

## Security Considerations

- ✅ Admin credentials from OnePassword vault
- ✅ RBAC enabled
- ✅ TLS/SSL via cert-manager
- ✅ Security context configured
- ✅ Latest Jenkins LTS version (2.504.2)
- ⚠️ Configure additional authentication if needed
- ⚠️ Review plugin security regularly

## Next Steps

1. Access Jenkins via https://jenkins.i.psilva.org
2. Login with OnePassword credentials
3. Install additional plugins as needed via UI
4. Configure build agents
5. Set up CI/CD pipelines

## Additional Resources

- [Jenkins Helm Chart Documentation](https://github.com/jenkinsci/helm-charts)
- [Jenkins Configuration as Code](https://jenkins.io/projects/jcasc/)
- [Longhorn Documentation](https://longhorn.io/docs/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [OnePassword Connect Operator](https://github.com/1Password/onepassword-operator)
