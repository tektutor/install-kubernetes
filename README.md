# Install Kubernetes in Ubuntu 24.04 64-bit LTS

## Install KVM Hypervisor in Ubuntu
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

## Automating Virtual machine creation process with Vagrant

Create a folder as admin
```
sudo su -
mkdir -p /root/kubernetes/scripts
cd /root/kubernetes
touch Vagrantfile
touch scripts/bootstrap.sh
```
Create a file named Vagrantfile
```
Vagrant.configure("2") do |config|
  IMAGE = "generic/ubuntu2204"
  MASTER_COUNT = 3
  WORKER_COUNT = 3
  VM_MEMORY = 131072
  VM_CPUS = 10
  NET_PREFIX = "192.168.56"

  config.vm.box = IMAGE

  config.vm.provision "shell", path: "scripts/bootstrap.sh"

  # Master Nodes
  (1..MASTER_COUNT).each do |i|
    node_name = "master%02d.k8s.rps.com" % i
    ip = "#{NET_PREFIX}.1#{i}"
    config.vm.define node_name do |node|
      node.vm.hostname = node_name
      node.vm.network "private_network", ip: ip, libvirt__network_name: "vagrant-libvirt"

      node.vm.provider :libvirt do |libvirt|
        libvirt.memory = VM_MEMORY
        libvirt.cpus = VM_CPUS
        libvirt.storage_pool_name = "default"
      end
    end
  end

  # Worker Nodes
  (1..WORKER_COUNT).each do |i|
    node_name = "worker%02d.k8s.rps.com" % i
    ip = "#{NET_PREFIX}.2#{i}"
    config.vm.define node_name do |node|
      node.vm.hostname = node_name
      node.vm.network "private_network", ip: ip, libvirt__network_name: "vagrant-libvirt"

      node.vm.provider :libvirt do |libvirt|
        libvirt.memory = VM_MEMORY
        libvirt.cpus = VM_CPUS
        libvirt.storage_pool_name = "default"
      end
    end
  end
end
```

Let's create the scripts/boostrap.sh
```
#!/bin/bash

# Disable swap
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# Install dependencies
apt-get update
apt-get install -y apt-transport-https curl ca-certificates gnupg lsb-release containerd

# Containerd config
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd

# Add Kubernetes repo
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

# Install Kubernetes components
sudo rm -f /etc/apt/sources.list.d/kubernetes.list
sudo rm -f /usr/share/keyrings/kubernetes-archive-keyring.gpg
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  gpg --dearmor | sudo tee /etc/apt/keyrings/kubernetes-apt-keyring.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

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

Make sure the script is executable
```
chmod +x scripts/bootstrap.sh
```

Let's create the VMs using Vagrant now
```
cd kubernetes
virsh pool-destroy default  # Only if it's not running anything
virsh pool-undefine default

virsh pool-define-as default dir - - - - "/var/lib/libvirt/images"
virsh pool-autostart default
virsh pool-start default

vagrant up
```

Let's configure the host machine to support bridged network for the VMs to communicate with each other directly. 
Create a file /etc/netplan/01-netcfg.yaml
<pre>
network:
  version: 2
  renderer: networkd
  ethernets:
    eno2: {}
  bridges:
    br0:
      interfaces: [eno2]
      dhcp4: true  
</pre>

Run this on the host machine
```
ls -l /etc/netplan/

# This is mandatory before issuing netplan commands, otherwise we will lose network connectivity to the machine
# Especially if this is a remote machine, this is mandatory
sudo chmod 600 /etc/netplan/01-netcfg.yaml

sudo netplan try
sudo netplan apply
ifconfig eno2
ping 8.8.8.8
```

Listing all Vagrants VMS
```
vagrant global-status
cd /root/kubernetes
vagrant ssh 
```

Run this on all Virtual Machines
```
# Install Docker or containerd
sudo apt update
sudo apt install -y apt-transport-https curl
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd

# Add Kubernetes repo
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] \
https://apt.kubernetes.io/ kubernetes-xenial main" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Let's install HAProxy on the host machine
```
sudo apt update
sudo apt install -y haproxy
```

Edit /etc/haproxy/haproxy.cfg and replace the bottom section with the following:
<pre>
defaults
    mode tcp
    timeout connect 10s
    timeout client 1m
    timeout server 1m
    log global
    option tcplog

frontend kubernetes-api
    bind *:6443
    default_backend k8s_masters

backend k8s_masters
    balance roundrobin
    option tcp-check
    server master1 192.168.56.11:6443 check
    server master2 192.168.56.12:6443 check
    server master3 192.168.56.13:6443 check  
</pre>

Restart HAProxy
```
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

SSH into first master VM
```
vagrant ssh k8s-master-1
```

Run this on the k8s-master-1 node
```
sudo kubeadm init --control-plane-endpoint "<HOST_IP>:6443" --upload-certs

# Install Calico network plugin from the host machine
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml
```

Run this on the first master VM
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Join the rest of the masters and workers with the join token from first master node
