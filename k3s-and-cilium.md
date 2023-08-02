# k3s + cilium lab

1. deploy ubuntu server 22.04 to cube
1. set up SSH key access + fail2ban + ufw
1. install virtualbox - bridge networking is easier than qemu+kvm
1. create 3 VMs - 2 vcpu, 8GB RAM, 40GB disk
1. install ubuntu server 22.04 on each VM
1. set up SSH key access on VMs
1. `lvextend -r -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv`
1. take snapshot of each
1. setup kubectl on dev machine(s) - https://kubernetes.io/docs/tasks/tools/

## install k3s and init a cluster on first node, disabling features that Cilium will supplant
resources:
- https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#k8s-install-quick
- https://docs.k3s.io/installation/configuration
- https://docs.k3s.io/datastore/ha-embedded
- https://docs.k3s.io/advanced

NOTE: we explicitly turn off traefik in addition to args documented by Cilium
```
curl -sfL https://get.k3s.io | sh -s - server \
    --cluster-init \
    --flannel-backend=none \
    --disable-network-policy \
    --disable=traefik

sudo cp /etc/rancher/k3s/k3s.yaml ~
sudo chown k3s:k3s ~/k3s.yaml
echo "export KUBECONFIG=$HOME/k3s.yaml" >> ~/.bashrc
echo "alias k='kubectl'" >> ~/.bashrc
source ~/.bashrc
```

We should see the following:

```
k3s@k3s-1:~$ k get no
NAME    STATUS     ROLES                       AGE    VERSION
k3s-1   NotReady   control-plane,etcd,master   111s   v1.27.4+k3s1
```

NOTE: node status will show `NotReady`; `journalctl -u k3s | tail` will note:
```
"Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
```
This is expected as there's no pod networking facility in place until Cilium is deployed.

## install k3s on additional nodes
```
# get K3S_TOKEN value from node 1 via:
# sudo cat /var/lib/rancher/k3s/server/node-token

sudo cp /etc/rancher/k3s/k3s.yaml ~
sudo chown k3s:k3s ~/k3s.yaml
echo "export KUBECONFIG=$HOME/k3s.yaml" >> ~/.bashrc
echo "alias k='kubectl'" >> ~/.bashrc
source ~/.bashrc

curl -sfL https://get.k3s.io | K3S_TOKEN='<TOKEN GOES HERE>' sh -s - server \
    --server https://192.168.0.174:6443 \
    --flannel-backend=none \
    --disable-network-policy \
    --disable=traefik
```

## install cilium CLI all nodes
```
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

## install cilium into the cluster using the first node
```
cilium install --version 1.14.0
cilium status --wait
```

validate cilium + test:
https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#validate-the-installation

