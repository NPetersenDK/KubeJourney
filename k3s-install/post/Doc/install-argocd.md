# Installing ArgoCD in the k3s cluster

## Prerequisites
- Cluster up and running with bgp enabled services
- kubectl configured to interact with the cluster

## Steps
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer", "externalTrafficPolicy": "Local"}, "metadata": {"labels": {"bgp": "pool-a", "bgp-announce": "true"}}}'
```