# AGENTS.md - Coding Agent Guidelines

This file provides guidance for AI coding agents working in this Talos Kubernetes repository.

## Project Overview

This is a reproducible Talos Linux Kubernetes cluster configuration repository for home lab environments. It contains infrastructure-as-code, Kubernetes manifests, Talos machine configurations, and documentation.

## Build/Lint/Test Commands

### Documentation
```bash
# Serve docs locally with live reload
mkdocs serve

# Build static docs site
mkdocs build

# Lint markdown files
markdownlint docs/


### YAML/Kubernetes
```bash
# Lint all YAML files
yamllint .

# Validate Kubernetes manifests (dry-run)
kubectl apply --dry-run=client -f <manifest.yaml>

# Validate all manifests in a directory
kubectl apply --dry-run=client -R -f manifests/

# Format YAML with yq
yq -i '.' <file.yaml>

# Check Talos config syntax
talosctl validate --config <config.yaml> --mode <cloud-init|metal>
```

### Talos Cluster
```bash
# Generate Talos config
talosctl gen config <cluster-name> <control-plane-endpoint>

# Apply configuration to a node
talosctl apply-config --insecure --nodes <ip> --file <config.yaml>

# Bootstrap cluster
talosctl bootstrap --nodes <control-plane-ip>

# Get kubeconfig
talosctl kubeconfig --nodes <control-plane-ip>

# Check cluster health
talosctl health --nodes <control-plane-ip>

# Upgrade a node
talosctl upgrade --nodes <ip> --image <image>
```

### Kubernetes Testing
```bash
# Validate resource syntax
kubectl apply --dry-run=client -f <file>

# Test Helm chart rendering (if using Helm)
helm template <release> <chart> -f values.yaml

# Lint Helm chart
helm lint <chart-path>
```

## Repository Structure

```
.
├── docs/                    # MkDocs documentation
├── manifests/               # Kubernetes manifests (organized by namespace/function)
├── talos/                   # Talos machine configurations
│   ├── controlplane/        # Control plane node configs
│   └── workers/             # Worker node configs
├── scripts/                 # Utility scripts
├── secrets/                 # Encrypted secrets (DO NOT commit plaintext)
├── mkdocs.yml               # Documentation configuration
└── AGENTS.md                # This file
```

## Code Style Guidelines

### YAML (Kubernetes Manifests & Talos Configs)

- **Indentation**: 2 spaces, never tabs
- **Lists**: Use `-` with space before value
- **Strings**: Prefer unquoted; quote only when containing special chars
- **Comments**: Use `#` with space after; explain non-obvious values
- **Naming**:
  - Resources: lowercase with hyphens (e.g., `my-app-deployment`)
  - Labels: use `app.kubernetes.io/` standard labels
  - Annotations: group by prefix

```yaml
# Good example
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: web-apps
  labels:
    app.kubernetes.io/name: nginx
    app.kubernetes.io/component: web-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx
```

### Markdown (Documentation)

- **Headings**: Use ATX style (`#`), one H1 per file
- **Lists**: Use `-` for unordered, `1.` for ordered
- **Code blocks**: Always specify language
- **Links**: Use reference-style for repeated links
- **Admonitions**: Use Material for MkDocs syntax

```markdown
!!! note "Optional Title"
    This is a note admonition.

!!! warning
    This is a warning.
```

### Shell Scripts

- **Shebang**: Always include `#!/usr/bin/env bash`
- **Error handling**: Use `set -euo pipefail`
- **Variables**: Use `${var}` syntax; quote expansions
- **Functions**: Use `function name() { }` or `name() { }`

```bash
#!/usr/bin/env bash
set -euo pipefail

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

function main() {
    local cluster_name="${1:-default}"
    # ...
}

main "$@"
```

### File Naming Conventions

- **Kubernetes manifests**: `<resource-type>-<name>.yaml` (e.g., `deployment-nginx.yaml`)
- **Talos configs**: `node-<hostname>.yaml` or `<role>-<hostname>.yaml`
- **Scripts**: `verb-noun.sh` (e.g., `deploy-cluster.sh`, `backup-etcd.sh`)
- **Documentation**: lowercase with hyphens (e.g., `getting-started.md`)

## Secrets Management

- **NEVER** commit plaintext secrets
- Use sealed-secrets, external-secrets, or SOPS for encryption
- Mark files containing secrets in `.gitignore`
- Use Kubernetes Secrets with external secret management

```bash
# Encrypt with SOPS
sops --encrypt --kms <kms-arn> secrets/secret.yaml > secrets/secret.enc.yaml

# Decrypt for local testing (never commit)
sops --decrypt secrets/secret.enc.yaml
```

## Kubernetes Best Practices

### Labels and Annotations

Always include standard labels:
```yaml
metadata:
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: myapp-prod
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: myapp
    app.kubernetes.io/managed-by: kubectl
```

### Resource Management

- Always set `resources.requests` and `resources.limits`
- Use `ConfigMaps` for configuration, `Secrets` for sensitive data
- Use namespaces to isolate workloads
- Apply `NetworkPolicy` for security isolation

### GitOps Conventions

- Store manifests in Git as single source of truth
- Use kustomize overlays for environment differences
- Keep base configurations environment-agnostic
- Document any manual steps in `docs/`

## Talos-Specific Guidelines

### Machine Configuration

- Use `talosctl gen config` as starting point
- Store generated configs in `talos/` directory
- Never include secrets in machine configs (use patches)
- Document custom patches applied

### Cluster Management

- Always backup before upgrades
- Use `talosctl health` before and after changes
- Apply changes to one node at a time for control plane
- Document upgrade procedures in `docs/operations/`

## Error Handling

### Validation

Always validate before applying:
1. YAML syntax: `yamllint <file>`
2. Kubernetes schema: `kubectl apply --dry-run=client -f <file>`
3. Talos config: `talosctl validate --config <file> --mode <mode>`

### Rollback

Document rollback procedures for:
- Kubernetes deployments
- Talos upgrades
- Configuration changes

## Git Workflow

- Use conventional commit messages
- Reference issues in commits
- Keep commits atomic and focused
- Run linting before committing

```
feat: add nginx ingress controller
fix: correct control plane endpoint in talos config
docs: update installation guide for v1.5
chore: update talos to v1.6.0
```

## Testing

This is infrastructure code; focus on:
- Syntax validation (yamllint, kubectl dry-run)
- Configuration drift detection
- Cluster health checks
- Documentation accuracy

## Documentation Updates

When adding features or changing configuration:
1. Update relevant `docs/` pages
2. Add inline comments for complex configurations
3. Update `docs/operations/` for new procedures
4. Run `mkdocs build --strict` to validate docs
