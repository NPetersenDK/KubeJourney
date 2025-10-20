# Prerequisites

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm upgrade --install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace --wait
```


```bash
kubectl create secret generic azure-secret-sp \
  --from-literal=ClientID="CLIENTID" \
  --from-literal=ClientSecret="CLIENTSECRET" \
  -n external-secrets
```