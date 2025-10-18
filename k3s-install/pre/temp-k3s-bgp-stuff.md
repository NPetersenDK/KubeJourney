# Announce Traefik via BGP

```bash
kubectl patch svc traefik -n kube-system -p '{"metadata":{"labels":{"bgp":"pool-a","bgp-announce":"true"}},"spec":{"externalTrafficPolicy":"Local"}}'
```

Works with services/temp-nginx-test-traefik.yaml