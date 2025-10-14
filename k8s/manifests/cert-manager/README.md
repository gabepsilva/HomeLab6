# Automating Let's Encrypt Certificates with Cert-Manager and DNS-01

This document outlines the steps to set up automated TLS certificate management for internal services using cert-manager, Let's Encrypt, and the DNS-01 challenge with Cloudflare.

## Prerequisites

*   A Kubernetes cluster.
*   `kubectl` installed and configured to connect to your cluster.
*   An Ingress controller (like Nginx Ingress) installed in the cluster.
*   A domain name managed by Cloudflare (`psilva.org` in this example).

---

## Section 1: Core Cert-Manager Setup (DNS-01 with Cloudflare)

This section covers the one-time setup of cert-manager and its configuration to use Cloudflare for DNS-01 challenges.

### 1. Install Cert-Manager

Cert-manager is a Kubernetes add-on that automates the management and issuance of TLS certificates.

Apply the official manifest:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.2/cert-manager.yaml
```

Verify the installation by checking the pods in the `cert-manager` namespace:

```bash
kubectl get pods -n cert-manager
```
Wait until all pods are in the `Running` state.

### 2. Configure Cloudflare API Access

Cert-manager needs API access to modify DNS records in your Cloudflare zone.

#### a. Create a Cloudflare API Token

1.  Log in to your Cloudflare Dashboard.
2.  Go to **My Profile** -> **API Tokens**.
3.  Click **Create Token**.
4.  Find the **"Edit zone DNS"** template and click **Use template**.
5.  **Permissions:** Keep `Zone | DNS | Edit`.
6.  **Zone Resources:** Select `Specific zone` and choose your domain (`psilva.org`).
7.  Optionally configure Client IP Address Filtering and TTL.
8.  Click **Continue to summary** and then **Create Token**.
9.  **Copy the generated API token immediately.**

#### b. Create Kubernetes Secret

Manifest file: `manifests/cert-manager/core-deployment/cloudflare-api-token-secret.yaml`

Create this file with the following content, replacing `<YOUR_CLOUDFLARE_API_TOKEN>` with the actual token:

```yaml
# manifests/cert-manager/core-deployment/cloudflare-api-token-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  # Place the secret in the cert-manager namespace as it's used by the ClusterIssuer.
  namespace: cert-manager
type: Opaque
stringData:
  api-token: <YOUR_CLOUDFLARE_API_TOKEN>
```

Apply the secret to your cluster:

```bash
# Ensure you have created the file above before running this
kubectl apply -f manifests/cert-manager/core-deployment/cloudflare-api-token-secret.yaml
```

**Important:** This secret is required for renewals. Do not delete it after initial setup.

### 3. Create ClusterIssuer

This resource tells cert-manager how to use Let's Encrypt and the Cloudflare secret.

Manifest file: `manifests/cert-manager/core-deployment/letsencrypt-clusterissuer.yaml`

Create this file:
```yaml
# manifests/cert-manager/core-deployment/letsencrypt-clusterissuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  # Name used to reference this issuer in Certificate resources
  name: letsencrypt-cloudflare-issuer
spec:
  acme:
    # Use the Let's Encrypt production server
    # Use https://acme-staging-v02.api.letsencrypt.org/directory for testing to avoid rate limits
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <YOUR_EMAIL_ADDRESS> # Replace with your email (for expiry notices)
    privateKeySecretRef:
      # Secret name used to store the ACME account private key (will be created automatically)
      name: letsencrypt-cloudflare-issuer-account-key
    solvers:
    - dns01:
        cloudflare:
          # Reference the secret created in the previous step
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            # Must be in the same namespace as the secret (cert-manager)
            key: api-token
```

**Important:** Replace `<YOUR_EMAIL_ADDRESS>` with your actual email.

Apply the ClusterIssuer:

```bash
kubectl apply -f manifests/cert-manager/core-deployment/letsencrypt-clusterissuer.yaml
```

You can check the status of the issuer:

```bash
kubectl describe clusterissuer letsencrypt-cloudflare-issuer
```
Look for a condition `Type=Ready, Status=True`.

---

## Section 2: Example Deployment with Wildcard Certificate (`*.i.psilva.org`)

This section shows how to deploy an application (Nginx) and secure it using a wildcard certificate managed by the `ClusterIssuer` created above.

### 1. Create Nginx Deployment and Service

Manifest file: `manifests/cert-manager/wildcard-example/nginx-wildcard-deployment.yaml`

```yaml
# manifests/cert-manager/wildcard-example/nginx-wildcard-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-wildcard-demo
  namespace: default # Example in default namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-wildcard
  template:
    metadata:
      labels:
        app: nginx-wildcard
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-wildcard-service
  namespace: default # Example in default namespace
spec:
  selector:
    app: nginx-wildcard
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### 2. Request the Wildcard Certificate

Manifest file: `manifests/cert-manager/wildcard-example/wildcard-certificate.yaml`

```yaml
# manifests/cert-manager/wildcard-example/wildcard-certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-i-psilva-org-cert
  # Secret will be created in this namespace.
  namespace: default # Example in default namespace
spec:
  # The name of the secret where the certificate will be stored
  secretName: wildcard-i-psilva-org-tls
  issuerRef:
    name: letsencrypt-cloudflare-issuer # Reference the ClusterIssuer
    kind: ClusterIssuer
  dnsNames:
  - "*.i.psilva.org"
  # Optionally add the apex domain if needed:
  # - i.psilva.org
```

### 3. Create Ingress Resource

Manifest file: `manifests/cert-manager/wildcard-example/nginx-wildcard-ingress.yaml`

```yaml
# manifests/cert-manager/wildcard-example/nginx-wildcard-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-wildcard-ingress
  namespace: default # Example in default namespace
  annotations:
    # Replace "nginx" with your ingress controller class if different
    kubernetes.io/ingress.class: "nginx"
    # Tells cert-manager to use this issuer for this ingress (recommended)
    cert-manager.io/cluster-issuer: letsencrypt-cloudflare-issuer
spec:
  tls:
  - hosts:
    # Define which host(s) this TLS section applies to
    - "nginx.i.psilva.org" # Example host covered by the wildcard
    # Reference the secret that cert-manager will create/manage
    secretName: wildcard-i-psilva-org-tls
  rules:
  - host: "nginx.i.psilva.org"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-wildcard-service # Service from Step 1
            port:
              number: 80
```

### 4. Apply Manifests

Apply the manifests for the wildcard example:

```bash
# Apply all manifests in the wildcard-example directory
kubectl apply -f manifests/cert-manager/wildcard-example/
```

Alternatively, apply individually:
```bash
kubectl apply -f manifests/cert-manager/wildcard-example/nginx-wildcard-deployment.yaml
kubectl apply -f manifests/cert-manager/wildcard-example/wildcard-certificate.yaml
kubectl apply -f manifests/cert-manager/wildcard-example/nginx-wildcard-ingress.yaml
```

### 5. Verification

*   **DNS:** Ensure `nginx.i.psilva.org` points to your Ingress controller's external IP/LoadBalancer.
*   **Certificate Status:** Monitor the `Certificate` resource:
    ```bash
    kubectl describe certificate wildcard-i-psilva-org-cert -n default
    ```
    Wait for `Status: Conditions: Type: Ready, Status: True`.
*   **Secret:** Check if the secret is created:
    ```bash
    kubectl get secret wildcard-i-psilva-org-tls -n default
    ```
*   **Access:** Try accessing `https://nginx.i.psilva.org`.

---

## Section 3: Example Deployment with Specific Domain Certificate (`specific.i.psilva.org`)

This section shows how to deploy another application and secure it with its own specific certificate, using the same `ClusterIssuer`.

### 1. Create Nginx Deployment and Service

Manifest file: `manifests/cert-manager/specific-domain-example/nginx-specific-deployment.yaml`

```yaml
# manifests/cert-manager/specific-domain-example/nginx-specific-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-specific-demo
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-specific
  template:
    metadata:
      labels:
        app: nginx-specific
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-specific-service
  namespace: default
spec:
  selector:
    app: nginx-specific
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### 2. Request the Specific Certificate

Manifest file: `manifests/cert-manager/specific-domain-example/specific-certificate.yaml`

```yaml
# manifests/cert-manager/specific-domain-example/specific-certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: specific-i-psilva-org-cert
  namespace: default
spec:
  secretName: specific-i-psilva-org-tls # New secret name
  issuerRef:
    name: letsencrypt-cloudflare-issuer # Same issuer
    kind: ClusterIssuer
  dnsNames:
  - "specific.i.psilva.org" # The specific domain
```

### 3. Create Ingress Resource

Manifest file: `manifests/cert-manager/specific-domain-example/nginx-specific-ingress.yaml`

```yaml
# manifests/cert-manager/specific-domain-example/nginx-specific-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-specific-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: letsencrypt-cloudflare-issuer
spec:
  tls:
  - hosts:
    - "specific.i.psilva.org"
    secretName: specific-i-psilva-org-tls # Use the new secret
  rules:
  - host: "specific.i.psilva.org"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-specific-service # Use the new service
            port:
              number: 80
```

### 4. Apply Manifests

Apply the manifests for the specific domain example:

```bash
# Apply all manifests in the specific-domain-example directory
kubectl apply -f manifests/cert-manager/specific-domain-example/
```

Alternatively, apply individually:
```bash
kubectl apply -f manifests/cert-manager/specific-domain-example/nginx-specific-deployment.yaml
kubectl apply -f manifests/cert-manager/specific-domain-example/specific-certificate.yaml
kubectl apply -f manifests/cert-manager/specific-domain-example/nginx-specific-ingress.yaml
```

### 5. Verification

*   **DNS:** Ensure `specific.i.psilva.org` points to your Ingress controller's external IP/LoadBalancer.
*   **Certificate Status:** Monitor the `Certificate` resource:
    ```bash
    kubectl describe certificate specific-i-psilva-org-cert -n default
    ```
    Wait for `Status: Conditions: Type: Ready, Status: True`.
*   **Secret:** Check if the secret is created:
    ```bash
    kubectl get secret specific-i-psilva-org-tls -n default
    ```
*   **Access:** Try accessing `https://specific.i.psilva.org`. 