# Cert-Manager with Cloudflare DNS Challenge

This directory contains the cert-manager configuration for automatic TLS certificate management using Let's Encrypt and Cloudflare DNS-01 challenge.

## Components

- **cert-manager Helm Chart**: Installed via ArgoCD with CRDs enabled
- **ClusterIssuers**: 
  - `letsencrypt-staging`: For testing (uses staging API, higher rate limits)
  - `letsencrypt-prod`: For production certificates
- **Cloudflare API Token**: Synced from Azure Key Vault via External Secrets

## Setup

1. Ensure `cloudflare-api-key` exists in Azure Key Vault
2. Update email address in `cloudflare-issuer.yaml` (both issuers)
3. Apply the ArgoCD application: `kubectl apply -f ../cert-manager.yaml`

## Usage

### Testing with Staging (Recommended First)

Add this annotation to your Ingress:
```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
spec:
  tls:
  - hosts:
    - your-domain.example.com
    secretName: your-domain-tls-staging
```

### Production Certificates

Once tested, switch to production:
```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - your-domain.example.com
    secretName: your-domain-tls
```

## Verification

```bash
# Check cert-manager pods
kubectl get pods -n cert-manager

# Check certificate status
kubectl get certificate -A

# Check certificate details
kubectl describe certificate <certificate-name> -n <namespace>

# Check ClusterIssuers
kubectl get clusterissuer
```

## Cloudflare API Token Requirements

The Cloudflare API token needs these permissions:
- Zone - DNS - Edit
- Zone - Zone - Read

Scoped to the zones you want to manage certificates for.

## Rate Limits

- **Staging**: 30,000 registrations per week
- **Production**: 50 certificates per registered domain per week

Always test with staging first!
