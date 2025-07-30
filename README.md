# Install Kubernetes in Ubuntu 24.04 64-bit LTS

## Install KVM Hypervisor in Ubuntu Server
```
sudo apt update
sudo apt-get install -y curl gnupg software-properties-common
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update

sudo apt-get install -y qemu-system libvirt-daemon-system libvirt-clients ebtables dnsmasq \
                        libxslt-dev libxml2-dev libvirt-dev zlib1g-dev ruby-dev \
                        bridge-utils libguestfs-tools gcc make virt-manager guestfs-tools \
                        qemu-kvm virt-viewer cloud-image-utils vagrant -y

sudo usermod -aG libvirt $(whoami)
sudo usermod -aG kvm $(whoami)     
vagrant plugin install vagrant-libvirt
vagrant plugin list

sudo apt install net-tools neovim tree iputils-ping tmux git

sudo adduser root kvm
sudo systemctl enable --now libvirtd
sudo systemctl status libvirtd
```

## Let's create virtual machines using KVM manually

Create a folder as admin
```
sudo su -
```

Install the below in all Virtual Machines except HAProxy VM
```
#!/bin/bash

apt install -y sudo

# Disable swap
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# Install dependencies
apt-get update

sudo apt update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd


apt-get install -y apt-transport-https curl ca-certificates gnupg  gpg lsb-release containerd
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo apt update
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd

containerd config dump | grep SystemdCgroup

# Containerd config
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd

# Enable kernel modules and sysctl settings
modprobe overlay
modprobe br_netfilter

tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system

```

Edit sudo vim /etc/crictl.yaml

```
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: true
```



Run this on all Virtual Machines
```
# Install Docker or containerd
sudo apt update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
containerd config dump | grep SystemdCgroup
```

Let's install HAProxy on the haproxy vm
```
sudo apt update
sudo apt install -y haproxy ufw
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list


```

Edit /etc/haproxy/haproxy.cfg and replace the bottom section with the following:
<pre>
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon
    maxconn 2048

defaults
    log     global
    mode    tcp
    option  tcplog
    timeout connect 10s
    timeout client  1m
    timeout server  1m

# Frontend for Kubernetes API
frontend k8s_api
    bind *:6443
    default_backend k8s_masters

# Backend: Kubernetes Master Nodes
backend k8s_masters
    mode tcp
    balance roundrobin
    option tcp-check
    default-server inter 5s fall 3 rise 2

    server master01 192.168.56.11:6443 check
    server master02 192.168.56.12:6443 check
    server master03 192.168.56.13:6443 check
</pre>

Restart HAProxy
```
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

Open these ports on the HAProxy VM
```
sudo ufw allow 6443/tcp
sudo ufw allow 22/tcp        # if SSH is needed
sudo ufw allow 8404/tcp      # only if enabling HAProxy stats UI
sudo iptables -A INPUT -p tcp --dport 6443 -j ACCEPT


# Allow SSH access
sudo ufw allow ssh

# Enable the firewall. You will be prompted to confirm.
sudo ufw enable

# Kubernetes API Server
sudo ufw allow 6443/tcp

# etcd server client API (for inter-master communication)
sudo ufw allow 2379:2380/tcp

# Kubelet API
sudo ufw allow 10250/tcp

# Kube-scheduler
sudo ufw allow 10259/tcp

# Kube-controller-manager
sudo ufw allow 10257/tcp

# NodePort services (default range for application exposure)
sudo ufw allow 30000:32767/tcp
sudo ufw allow 30000:32767/udp
```

SSH into first master VM
```
vagrant ssh k8s-master-1
```

Run this on master1
```
hostnamectl set-hostname master01.k8s.rps.com
```

Run this on master2
```
hostnamectl set-hostname master02.k8s.rps.com
```

Run this on master3
```
hostnamectl set-hostname master03.k8s.rps.com
```

Add the below in /etc/hosts on all nodes
```
192.168.56.10 haproxy.k8s.rps.com
192.168.56.11 master01.k8s.rps.com
192.168.56.12 master02.k8s.rps.com
192.168.56.13 master03.k8s.rps.com
192.168.56.21 worker01.k8s.rps.com
192.168.56.22 worker02.k8s.rps.com
192.168.56.23 worker03.k8s.rps.com
```

Run this on the k8s-master-1 node
```
hostnamectl set-hostname master01.k8s.rps.com

sudo kubeadm init --control-plane-endpoint "192.168.56.10:6443" --upload-certs --pod-network-cidr=10.244.0.0/16
sudo kubeadm token create --ttl 1h --print-join-command
sudo kubeadm init phase upload-certs --upload-certs
crictl ps --output '{{range .containers}}{{.metadata.name}}{{"\t"}}{{.state.String}}{{"\n"}}{{end}}'
```

Troubleshooting pod crash after kubeadm init
```
containerd config dump | grep SystemdCgroup
cat /var/lib/kubelet/config.yaml | grep cgroupDriver
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward

# Stop kubelet to prevent it from restarting etcd
sudo systemctl stop kubelet

# Delete the data directory (default is /var/lib/etcd)
sudo rm -rf /var/lib/etcd

sudo rm -rf /etc/kubernetes/
sudo rm -rf /var/lib/kubelet
sudo rm -rf /var/lib/etcd
sudo rm -rf /etc/cni/net.d

# Start kubelet again
sudo systemctl start kubelet
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```
Also edit sudo vim /etc/systemd/resolved.conf
```
[Resolve]
# The DNS server IP addresses to use.
DNS=8.8.8.8 8.8.4.4
# The DNS servers will be used for resolving domain names and other resource records.
# The list should contain no more than three servers.
# Use spaces to separate multiple addresses.
```

Restart service
```
sudo systemctl restart systemd-resolved
```


# Install Calico network plugin from the host machine
```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/calico.yaml
```

Run this on the first master VM
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Join the rest of the masters and workers with the join token from first master node

In case the joins fails
```
sudo systemctl status kubelet

sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes/pki /etc/kubernetes/manifests /var/lib/etcd
sudo rm -rf /etc/kubernetes/
sudo rm -rf /var/lib/kubelet
sudo rm -rf /var/lib/etcd
sudo systemctl restart kubelet
```

Open up these ports on all master
```
# Kubernetes API server (needed by kubelets, HAProxy, kubectl, etc.)
sudo ufw allow 6443/tcp

# etcd server-client communication (between masters)
sudo ufw allow 2379:2380/tcp

# kubelet API (used for logs, exec, probes, etc.)
sudo ufw allow 10250/tcp

# kube-scheduler metrics (optional)
sudo ufw allow 10259/tcp

# kube-controller-manager metrics (optional)
sudo ufw allow 10257/tcp
```

Rerun this on the first master
```
kubeadm token create --print-join-command --ttl 1h 
kubeadm init phase upload-certs --upload-certs
```
