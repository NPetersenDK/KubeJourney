# Installing ArgoCD in the k3s cluster

## Prerequisites
- Cluster up and running with bgp enabled services
- kubectl configured to interact with the cluster
- ArgoCD CLI installed in cluster and CLI.

- Run the following on all nodes to prepare for Longhorn:
```bash
apt install open-iscsi nfs-common cryptsetup
systemctl enable iscsid
systemctl start iscsid
modprobe iscsi_tcp
```
- Download Longhornctl CLI: https://longhorn.io/docs/1.10.0/advanced-resources/longhornctl/install-longhornctl/
- Run: ```longhornctl check preflight``` to verify node readiness.

## Steps
- Use: https://longhorn.io/docs/1.10.0/deploy/install/install-with-argocd/
- Apply backend/longhorn.yml to install Longhorn via ArgoCD.