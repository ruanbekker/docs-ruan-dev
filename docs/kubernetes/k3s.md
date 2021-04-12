# Kubernetes with K3S

K3s by Rancher is a lightweight kubernetes distribution, you can read more up on [k3s.io](https://k3s.io/)

## About our Setup

I will be using version 1.17.17 from the [k3s releases](https://github.com/k3s-io/k3s/releases) page, if you remove the `INSTALL_K3S_VERSION` environment variable, the installation will default to the latest version.

To view more environment variables:
- https://rancher.com/docs/k3s/latest/en/installation/install-options/

## Deploy the Master

To deploy the master node:

```bash
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" INSTALL_K3S_VERSION=v1.17.17+k3s1 sh -s -
```

Once the installation succeeds, the kubeconfig file is located at `/etc/rancher/k3s/k3s.yaml` but should already be in your environment and you should be able to list your nodes:

```bash
kubectl get nodes --output wide
```

## Join a Agent to the Cluster

To add a worker node, we need to fetch the node-token from the master node:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Then proceed the installation by passing the master node's ip address as well as the node-token, we will assume that you stored the master ip address as `MASTER_IP=x.x.x.x` and the node-token stored as `MASTER_NODE_TOKEN=xxxxx`:

```bash
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" INSTALL_K3S_VERSION=v1.17.17+k3s1 K3S_URL=https://${MASTER_IP}:6443 K3S_TOKEN=${MASTER_NODE_TOKEN}
```

Once the node joined the master you should be able to list the nodes from the master:

```bash
kubectl get nodes --output wide
```

## Access Kubernetes from Outside

The kubeconfig on the master node has the localhost address in the config, so to access kubernetes from outside the node, we need to replace the master node's public or network reachable address:

```bash
sed "s/127.0.0.1/${}/g" /etc/rancher/k3s/k3s.yaml
```

Then store that output into a file `~/.kube/config`.
