# K3s with Cilium BGP Installation Guide

## Prerequisites

Install required packages:

```bash
apt install curl sudo
```

## Install K3s

Install K3s with Cilium-compatible settings (no Flannel, no kube-proxy, no servicelb):

```bash
curl -sfL https://get.k3s.io | sh -s - --flannel-backend=none --disable-network-policy --cluster-init
```

Wait a few seconds before continuing:

```bash
sleep 10
```

## Configure Kubeconfig

Set up kubeconfig access:

```bash
sudo chmod 600 /etc/rancher/k3s/k3s.yaml
echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> $HOME/.bashrc
source $HOME/.bashrc
mkdir -p $HOME/.kube
sudo cp -i /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
echo "export KUBECONFIG=$HOME/.kube/config" >> $HOME/.bashrc
source $HOME/.bashrc
```

Verify the installation:

```bash
kubectl get pods --all-namespaces
```

## Install Cilium CLI

Commands from: https://docs.cilium.io/en/stable/installation/k3s/

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

## Install Cilium

Set your API server details and install Cilium with BGP control plane enabled:

```bash
API_SERVER_IP=10.0.14.21
API_SERVER_PORT=6443
cilium install \
  --set k8sServiceHost=${API_SERVER_IP} \
  --set k8sServicePort=${API_SERVER_PORT} \
  --set kubeProxyReplacement=true \
  --set bgpControlPlane.enabled=true \
  --set ipam.operator.clusterPoolIPv4PodCIDRList=172.20.0.0/16 \
  --set ipv4NativeRoutingCIDR=172.20.0.0/16
```

Wait for Cilium to be ready (drink coffee while Cilium installs):

```bash
cilium status --wait
```

## Verify Cilium BGP Resources

Check that BGP resources are available:

```bash
kubectl api-resources | grep -i ciliumBGP
```

Expected output:

```
ciliumbgpadvertisements             cbgpadvert                          cilium.io/v2                        false        CiliumBGPAdvertisement
ciliumbgpclusterconfigs             cbgpcluster                         cilium.io/v2                        false        CiliumBGPClusterConfig
ciliumbgpnodeconfigoverrides        cbgpnodeoverride                    cilium.io/v2                        false        CiliumBGPNodeConfigOverride
ciliumbgpnodeconfigs                cbgpnode                            cilium.io/v2                        false        CiliumBGPNodeConfig
ciliumbgppeerconfigs                cbgppeer                            cilium.io/v2                        false        CiliumBGPPeerConfig
ciliumbgppeeringpolicies            bgpp                                cilium.io/v2alpha1                  false        CiliumBGPPeeringPolicy
```

Verify cluster status:

```bash
kubectl get pods --all-namespaces
kubectl get nodes
kubectl get pods -A
```

## Configure BGP

Set BGP peering policy on node:

```bash
kubectl label nodes k3s-nd-01 bgp-policy=default
```

## Apply BGP Configuration

Go to BGP folder and apply the configuration files:

```bash
kubectl apply -f ippool.yaml
kubectl apply -f cluster.yaml
kubectl apply -f advertisement.yaml
kubectl apply -f peer.yaml
```

## Deploy Test Service

Go to services folder and apply the nginx test:

```bash
kubectl apply -f nginx-test.yaml
```

## Verify BGP Status

Check BGP peers and routes:

```bash
cilium bgp peers
```

You should see 2 advertised routes to peers.

Expected output:

```
root@k3s-nd-01:/home/nikolaj/services# cilium bgp peers
Node        Local AS   Peer AS   Peer Address   Session State   Uptime   Family         Received   Advertised
k3s-nd-01   65001      65001     10.0.14.1      established     13s      ipv4/unicast   19         2
```

## Add Second Node for High Availability (Optional)

To create a highly available cluster with redundancy, add a second node that functions as both control plane and worker.

### Step 1: Get Node Token from First Node

On the first node, retrieve the token needed to join additional nodes:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Copy this token - you'll need it for the second node.

### Step 2: Install K3s on Second Node

On the second node, install K3s as a server (control plane + worker):

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --server https://10.0.14.21:6443 \
  --token fffffffff \
  --flannel-backend=none \
  --disable-network-policy
```

Replace:
- `<FIRST_NODE_IP>` - IP address of your first K3s node (e.g., `10.0.14.21`)
- `<NODE_TOKEN>` - The token you copied from the first node

Example:
```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --server https://10.0.14.21:6443 \
  --token K10abcdef1234567890::server:abcdef1234567890 \
  --flannel-backend=none \
  --disable-network-policy
```

Wait for the installation to complete:

```bash
sleep 10
```

### Step 3: Configure Kubeconfig on Second Node

Set up kubeconfig access on the second node:

```bash
sudo chmod 600 /etc/rancher/k3s/k3s.yaml
echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> $HOME/.bashrc
source $HOME/.bashrc
mkdir -p $HOME/.kube
sudo cp -i /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
echo "export KUBECONFIG=$HOME/.kube/config" >> $HOME/.bashrc
source $HOME/.bashrc
```

### Step 4: Verify Both Nodes

From either node, verify that both nodes are in the cluster:

```bash
kubectl get nodes -o wide
```

Expected output showing both nodes with control-plane, etcd, and master roles:

```
NAME         STATUS   ROLES                       AGE   VERSION
k3s-nd-01    Ready    control-plane,etcd,master   30m   v1.28.x+k3s1
k3s-nd-02    Ready    control-plane,etcd,master   5m    v1.28.x+k3s1
```

Check that all pods are running on both nodes:

```bash
kubectl get pods -A -o wide
```

### Step 5: Configure BGP on Second Node

Label the second node for BGP peering:

```bash
kubectl label nodes k3s-nd-02 bgp-policy=default
```

### Step 6: Verify BGP Status on Both Nodes

Check BGP peers from the first node:

```bash
cilium bgp peers
```

Expected output showing both nodes:

```
Node        Local AS   Peer AS   Peer Address   Session State   Uptime   Family         Received   Advertised
k3s-nd-01   65001      65001     10.0.14.1      established     5m       ipv4/unicast   19         2
k3s-nd-02   65001      65001     10.0.14.1      established     2m       ipv4/unicast   19         2
```