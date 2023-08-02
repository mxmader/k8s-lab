# k3s lab

deploy ubuntu server 22.04 to cube
set up SSH key access + fail2ban + ufw
create 3 VMs - 2 vcpu, 8GB RAM, 40GB disk
install ubuntu server 22.04 on each VM
set up SSH key access on VMs
take snapshot
setup kubectl on dev machine(s) - https://kubernetes.io/docs/tasks/tools/

install cilium CLI on first node:
# TODO: put in a `set -e` script
```
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

install k3s and init a cluster on first node, disabling features that Cilium will supplant.
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
```

NOTE: node status will show `NotReady`; `journalctl -u k3s` will note:
```
"Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
```
This is expected as there's no pod networking facility in place until Cilium is deployed.

install k3s on additional nodes:
# TODO: make additional nodes run server components (etcd et al)
```
# get K3S_TOKEN value from node 1 via:
# sudo cat /var/lib/rancher/k3s/server/node-token

curl -sfL https://get.k3s.io | K3S_TOKEN=<TOKEN GOES HERE> sh -s - server \
    --server https://192.168.0.174:6443
    
# TODO: need these?    
#    --flannel-backend=none \
#    --disable-network-policy \
#    --disable=traefik
```

install cilium into the cluster using the first node:
```
sudo chmod 0644 /etc/rancher/k3s/k3s.yaml
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
cilium install --version 1.14.0
cilium status --wait
```

validate cilium + test:
https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#validate-the-installation

