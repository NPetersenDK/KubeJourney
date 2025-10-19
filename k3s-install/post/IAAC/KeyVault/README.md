# Prerequisites

- An existing Azure Key Vault.
- An App Registration in Entra ID
- Permissions granted to the App Registration to access the Key Vault secrets.
  - I used "Key Vault Secrets Offficer" role assignment. It allowed me to read and list secrets.

# How
- Get the following values from your Azure environment:
  - TENANT_ID
  - CLIENT_ID
  - CLIENT_SECRET
  - KEY_VAULT_URL

- Run the following command: 
```bash
kubectl create secret generic azure-secret-sp \
  --from-literal=ClientID="CLIENTID" \
  --from-literal=ClientSecret="CLIENTSECRET" \
  -n external-secrets
```
- Change the values in `az-clustersecretstore.yaml` file with your TENANT_ID and KEY_VAULT_URL

- Apply the ClusterSecretStore manifest:
```bash
kubectl apply -f k3s-install/post/IAAC/KeyVault/az-clustersecretstore.yaml
```
- Apply the ExternalSecret manifest:
```bash
kubectl apply -f k3s-install/post/IAAC/KeyVault/az-externalsecrets.yaml
```

Debug:

See the status of the ClusterSecretStore
```bash
kubectl describe clusterSecretStore azure-secret-store
```

Check the status of all ExternalSecrets
```bash
kubectl get externalsecrets -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'
```

See sync status:
```bash
kubectl get externalsecret all-keyvault-secrets
```

Get secrets:
```bash
kubectl get secret all-keyvault-secrets
```
Describe secret:
```bash
kubectl describe secret all-keyvault-secrets
```

Use in Pod:
```yaml
env:
  - name: MY_SECRET
    valueFrom:
      secretKeyRef:
        name: all-keyvault-secrets
        key: <your-keyvault-secret-name>
```