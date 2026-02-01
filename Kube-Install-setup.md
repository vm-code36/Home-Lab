

## So this is the Kube installation setup guide using kubeadm from scratch 

### Steps to Setup on a bare Matel Server

- **Step 0**: Pre checks to setup Enable IPTables and Disable swap on all Nodes

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Swap off 
sudo swapoff -a
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
```


- **Step 1**: Install containerd Runtime On All The Nodes 

```
bash

sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install containerd.io

sudo systemctl daemon-reload
sudo systemctl enable containerd --now
sudo systemctl start containerd.service

echo "Containerd runtime installed successfully"

# Generate the default containerd configuration
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup 

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Restart containerd to apply changes
sudo systemctl restart containerd

```

- **Step 2**: Install & Configure **Crictl** to use Containerd

**crictl** is a CLI utility to interact with the containers created by the container runtime.

```
bash

# Install crictl
curl -LO https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-amd64.tar.gz
sudo tar zxvf crictl-${CRICTL_VERSION}-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-${CRICTL_VERSION}-linux-amd64.tar.gz

# Configure crictl to use containerd
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

echo "crictl installed and configured successfully"

```

- **Step 3**: Install **Kubeadm** & **Kubelet** & **Kubectl** on all Nodes

```
KUBERNETES_VERSION=v1.34

curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list

# apt update will add kubeadm, kubelet, kubectl

sudo apt-get update -y

# gets the exact version number

KUBERNETES_INSTALL_VERSION=$(
  apt-cache madison kubeadm |
  awk -F'|' '/1.34./ {gsub(/ /,"",$2); print $2; exit}'
)

# apt install of kubeadm, kubelet, kubectl

sudo apt-get install -y \
  kubeadm=$KUBERNETES_INSTALL_VERSION \
  kubelet=$KUBERNETES_INSTALL_VERSION \
  kubectl=$KUBERNETES_INSTALL_VERSION

# add hold to the packages to prevent upgrades

sudo apt-mark hold kubelet kubeadm kubectl

# update node ip to KUBELET_EXTRA_ARGS=--node-ip=<local ip> this will ensure valid ip all the time

# finding ip
sudo apt-get install -y jq

local_ip="$(ip --json addr show eth0 | jq -r '.[0].addr_info[] | select(.family == "inet") | .local')"

# updating ip to kubelet

cat > /etc/default/kubelet << EOF

KUBELET_EXTRA_ARGS=--node-ip=$local_ip

EOF

```

--- 

### Until now all node configurations are completed, now we have to Setup **Master Node**

- Step 4: 
