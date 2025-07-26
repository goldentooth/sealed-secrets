# Sealed Secrets

GitOps configuration for Sealed Secrets Controller in the Goldentooth Kubernetes cluster, enabling encrypted secrets storage in Git repositories.

## Overview

Sealed Secrets allows you to store encrypted secrets in Git repositories safely. The Sealed Secrets Controller runs in your cluster and can decrypt sealed secrets, but only the controller can decrypt them, not anyone with access to the Git repository.

## Purpose

### GitOps-Safe Secret Management
- **Git Storage**: Store encrypted secrets directly in Git repositories
- **Asymmetric Encryption**: Public key encryption, private key decryption in cluster
- **Version Control**: Full version history and GitOps workflow for secrets
- **Cluster-Specific**: Secrets can only be decrypted by the target cluster

### Development Workflow
- **Developer Experience**: Simple CLI tool for secret encryption
- **CI/CD Integration**: Automated secret management in pipelines
- **Multi-Environment**: Different encryption keys per cluster
- **Audit Trail**: Complete change tracking via Git history

## Architecture

### Components
- **Sealed Secrets Controller**: Runs in cluster, manages decryption
- **kubeseal CLI**: Client tool for encrypting secrets
- **Custom Resource**: SealedSecret CRD for encrypted secret storage
- **Private Key**: Cluster-specific key for decryption

### Encryption Flow
```
Plain Secret → kubeseal CLI → SealedSecret → Git → Controller → Secret
```

## Installation

### Controller Deployment
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets
spec:
  project: default
  source:
    repoURL: https://github.com/goldentooth/sealed-secrets
    path: .
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
```

### CLI Installation
```bash
# Install kubeseal CLI
curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/kubeseal-0.18.0-linux-amd64.tar.gz"
tar -xvzf kubeseal-0.18.0-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# Verify installation
kubeseal --version
```

## Usage

### Creating Sealed Secrets

#### From Command Line
```bash
# Create a secret and seal it
echo -n mypassword | kubectl create secret generic mysecret --dry-run=client --from-file=password=/dev/stdin -o yaml | kubeseal -o yaml > mysealedsecret.yaml

# Apply the sealed secret
kubectl apply -f mysealedsecret.yaml
```

#### From Existing Secret
```bash
# Create regular Kubernetes secret file
cat > secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: app-credentials
  namespace: default
data:
  username: bXl1c2Vy  # base64 encoded
  password: bXlwYXNzd29yZA==  # base64 encoded
EOF

# Seal the secret
kubeseal -f secret.yaml -w sealed-secret.yaml

# Remove the plain secret file
rm secret.yaml
```

### SealedSecret Resource
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: app-credentials
  namespace: default
spec:
  encryptedData:
    username: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEQAx...
    password: AgAKAoiTy200K+Mo4OK4oyEcbI8J9+l18Z...
  template:
    metadata:
      name: app-credentials
      namespace: default
    type: Opaque
```

## Common Patterns

### Database Credentials
```bash
# Create database secret
kubectl create secret generic postgres-credentials \
  --from-literal=username=dbuser \
  --from-literal=password=dbpass123 \
  --from-literal=host=postgres.goldentooth.net \
  --dry-run=client -o yaml | \
kubeseal -o yaml > postgres-sealed-secret.yaml
```

### API Keys
```bash
# Create API key secret
kubectl create secret generic api-keys \
  --from-literal=github-token=ghp_xxxxxxxxxxxx \
  --from-literal=slack-webhook=https://hooks.slack.com/services/xxx \
  --dry-run=client -o yaml | \
kubeseal -o yaml > api-keys-sealed-secret.yaml
```

### TLS Certificates
```bash
# Create TLS secret from certificate files
kubectl create secret tls app-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  --dry-run=client -o yaml | \
kubeseal -o yaml > app-tls-sealed-secret.yaml
```

## Multi-Environment Setup

### Environment-Specific Encryption
```bash
# Get public key for specific cluster
kubeseal --fetch-cert --controller-namespace=kube-system > prod-public-key.pem

# Encrypt for specific environment
echo -n "prod-password" | kubectl create secret generic app-secret \
  --dry-run=client --from-file=password=/dev/stdin -o yaml | \
kubeseal --cert=prod-public-key.pem -o yaml > app-secret-prod.yaml
```

### Directory Structure
```
secrets/
├── dev/
│   ├── app-credentials-sealed.yaml
│   └── database-sealed.yaml
├── staging/
│   ├── app-credentials-sealed.yaml
│   └── database-sealed.yaml
└── prod/
    ├── app-credentials-sealed.yaml
    └── database-sealed.yaml
```

## Namespace and Scoping

### Cluster-Wide Scope
```yaml
# Can be unsealed in any namespace
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: cluster-wide-secret
  annotations:
    sealedsecrets.bitnami.com/cluster-wide: "true"
```

### Namespace-Wide Scope
```yaml
# Can be unsealed in any namespace with same name
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: namespace-wide-secret
  annotations:
    sealedsecrets.bitnami.com/namespace-wide: "true"
```

### Strict Scope (Default)
```yaml
# Can only be unsealed in the exact namespace
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: strict-secret
  namespace: production
```

## Key Management

### Controller Key Rotation
```bash
# Check current controller key
kubectl get secret -n kube-system sealed-secrets-key -o yaml

# Manual key rotation (creates new key, keeps old for decryption)
kubectl annotate secret -n kube-system sealed-secrets-key \
  sealedsecrets.bitnami.com/rotate="true"

# List all active keys
kubeseal --controller-namespace=kube-system --list
```

### Backup and Recovery
```bash
# Backup the controller private key
kubectl get secret -n kube-system sealed-secrets-key -o yaml > sealed-secrets-key-backup.yaml

# Restore key to new cluster
kubectl apply -f sealed-secrets-key-backup.yaml
```

## Integration with GitOps

### ArgoCD Application
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-with-secrets
spec:
  source:
    repoURL: https://github.com/goldentooth/app-manifests
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

### Directory Structure
```
app-manifests/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── app-secrets-sealed.yaml
    │   └── kustomization.yaml
    └── prod/
        ├── app-secrets-sealed.yaml
        └── kustomization.yaml
```

## Monitoring and Troubleshooting

### Controller Status
```bash
# Check controller pod status
kubectl get pods -n kube-system -l name=sealed-secrets-controller

# View controller logs
kubectl logs -n kube-system deployment/sealed-secrets-controller

# Check for decryption errors
kubectl get events --field-selector reason=Failed
```

### Secret Validation
```bash
# Verify sealed secret can be decrypted
kubectl apply -f sealed-secret.yaml --dry-run=server

# Check if resulting secret was created
kubectl get secret app-credentials -o yaml

# Validate secret data
kubectl get secret app-credentials -o jsonpath='{.data.password}' | base64 -d
```

## Security Considerations

### Key Security
- **Private Key Protection**: Controller private key never leaves the cluster
- **Key Rotation**: Regular rotation of encryption keys
- **Backup Security**: Secure storage of key backups
- **Access Control**: RBAC for sealed secrets controller

### Best Practices
- **Environment Isolation**: Separate keys per environment
- **Least Privilege**: Minimal permissions for service accounts
- **Audit Logging**: Track all secret decryption operations
- **Git Security**: Use signed commits for sealed secret changes

### Limitations
- **Key Compromise**: If private key is compromised, all sealed secrets are vulnerable
- **Single Point of Failure**: Controller must be running to decrypt secrets
- **Re-encryption Required**: Key rotation requires re-encrypting all sealed secrets
- **Base64 Decoding**: Sealed secrets can be decoded to reveal structure (but not content)
