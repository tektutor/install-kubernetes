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

## Automating Virtual machine creation process with Vagrant

Create a folder as admin
```
sudo su -
mkdir -p /root/kubernetes/scripts
cd /root/kubernetes
touch Vagrantfile
touch scripts/bootstrap.sh

wget https://vagrantcloud.com/ubuntu/boxes/jammy64/versions/20240514.0.0/providers/virtualbox.box -O jammy64.box
vagrant box add ubuntu/jammy64 ./jammy64.box

```
Create a file named Vagrantfile
```
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/ubuntu-22.04"

  # HAProxy Load Balancer
  config.vm.define "haproxy" do |haproxy|
    haproxy.vm.hostname = "haproxy.k8s.rps.com"
    haproxy.vm.network "private_network", ip: "192.168.56.10"
    haproxy.vm.provider "virtualbox" do |vb|
      vb.memory = 2048
      vb.cpus = 2
    end
    haproxy.vm.provider "libvirt" do |lv|
      lv.memory = 2048
      lv.cpus = 2
    end
  end

  # Define Master Nodes
  (1..3).each do |i|
    config.vm.define "master0#{i}" do |master|
      master.vm.hostname = "master0#{i}.k8s.rps.com"
      master.vm.network "private_network", ip: "192.168.56.1#{i}"
      master.vm.provider "virtualbox" do |vb|
        vb.memory = 131072
        vb.cpus = 10
      end
      master.vm.provider "libvirt" do |lv|
        lv.memory = 131072
        lv.cpus = 10
      end
    end
  end

  # Define Worker Nodes
  (1..3).each do |i|
    config.vm.define "worker0#{i}" do |worker|
      worker.vm.hostname = "worker0#{i}.k8s.rps.com"
      worker.vm.network "private_network", ip: "192.168.56.2#{i}"
      worker.vm.provider "virtualbox" do |vb|
        vb.memory = 131072
        vb.cpus = 10
      end
      worker.vm.provider "libvirt" do |lv|
        lv.memory = 131072
        lv.cpus = 10
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

In case, you wish to clean existing VMs
```
# Step 1: Destroy all VMs
vagrant destroy -f

# Step 2: Remove old `.vagrant` metadata and state
rm -rf .vagrant

# (Optional) Remove VirtualBox/Libvirt VMs if Vagrant left anything hanging
# For VirtualBox:
VBoxManage list vms
VBoxManage unregistervm <vm-name> --delete

# For Libvirt:
virsh list --all
virsh destroy <vm-name>
virsh undefine <vm-name> --remove-all-storage
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

Let's install HAProxy on the haproxy vm
```
sudo apt update
sudo apt install -y haproxy
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

```

SSH into first master VM
```
vagrant ssh k8s-master-1
```

Run this on the k8s-master-1 node
```
sudo kubeadm init --control-plane-endpoint "192.168.10.10:6443" --upload-certs
sudo kubeadm token create --ttl 1h --print-join-command
sudo kubeadm init phase upload-certs --upload-certs


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

In case the joins fails
```
sudo systemctl status kubelet

sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes/pki /etc/kubernetes/manifests /var/lib/etcd
sudo systemctl restart kubelet
```

Rerun this on the first master
```
kubeadm token create --print-join-command --ttl 1h 
kubeadm init phase upload-certs --upload-certs
```
