# Install Kubernetes in Ubuntu 24.04 64-bit LTS

## Install KVM Hypervisor in Ubuntu
```
sudo apt update
sudo apt install virt-manager, guestfs-tools qemu-kvm libvirt-daemon-system \
     libvirt-clients bridge-utils virt-viewer cloud-image-utils vagrant vagrant-libvirt -y

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
cd kubernetes
touch Vagrantfile
touch scripts/bootstrap.sh
```
Create a file named Vagrantfile
<pre>
Vagrant.configure("2") do |config|
  IMAGE = "generic/ubuntu2204"
  MASTER_COUNT = 3
  WORKER_COUNT = 2
  VM_MEMORY = 4096
  VM_CPUS = 2
  NET_PREFIX = "192.168.56"

  config.vm.box = IMAGE

  config.vm.provider :libvirt do |v|
    v.memory = VM_MEMORY
    v.cpus = VM_CPUS
  end

  config.vm.provision "shell", path: "scripts/bootstrap.sh"

  # Master Nodes
  (1..MASTER_COUNT).each do |i|
    node_name = "master%02d.k8s.rps.com" % i
    config.vm.define node_name do |node|
      node.vm.hostname = node_name
      node.vm.network "private_network", ip: "#{NET_PREFIX}.1#{i}"
    end
  end

  # Worker Nodes
  (1..WORKER_COUNT).each do |i|
    node_name = "worker%02d.k8s.rps.com" % i
    config.vm.define node_name do |node|
      node.vm.hostname = node_name
      node.vm.network "private_network", ip: "#{NET_PREFIX}.2#{i}"
    end
  end
end
</pre>

Let's create the scripts/boostrap.sh
<pre>
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
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

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
</pre>

Make sure the script is executable
```
chmod +x scripts/bootstrap.sh
```

Let's create the VMs using Vagrant now
```
cd kubernetes
vagrant up
```

Let's configure the host machine to support bridged network for the VMs to communicate with each other directly. 
Create a file /etc/netplan/01-netcfg.yaml
<pre>
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0: {}
  bridges:
    br0:
      interfaces: [enp1s0]
      dhcp4: true  
</pre>

Run this on the host machine
```
sudo netplan apply
```

On the host machine, let's configure the HAProxy 
```
defaults
    mode tcp
    timeout connect 10s
    timeout client 1m
    timeout server 1m
    option tcplog

frontend kubernetes-api
    bind *:6443
    default_backend k8s_masters

backend k8s_masters
    balance roundrobin
    option tcp-check
    server master01 192.168.56.11:6443 check
    server master02 192.168.56.12:6443 check
    server master03 192.168.56.13:6443 check
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
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Run this on master-1 VM
```
sudo kubeadm init --control-plane-endpoint "k8s-cluster.local:6443" --upload-certs
```

Run this on the host machine
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install Calico network plugin from the host machine
```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml
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
```




## Let's create VMs for 3 master and 3 worker nodes

Download Debian-12
```
wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-12.2.0-amd64-netinst.iso -o /var/lib/libvirt/images/debian-12.iso
```

Create a virtual machine for Master 1 with Ubuntu 24.04
```
sudo virt-install \
  --name master-1 \
  --ram 131072 \
  --disk path=/var/lib/libvirt/images/debian12-vm.qcow2,size=1000,format=qcow2 \
  --vcpus 12 \
  --os-type linux \
  --os-variant debian12 \
  --network default \
  --graphics vnc \
  --console pty,target_type=serial \
  --cdrom /var/lib/libvirt/images/debian-12.iso \
  --boot cdrom,hd
```


Create a virtual machine for Master 2 with Ubuntu 24.04
```
virt-install \
  --name master-2 \
  --memory 131072 \
  --vcpus 12 \
  --disk path=/var/lib/libvirt/images/master-2.qcow2,format=qcow2 \
  --os-variant=ubuntu24.04 \
  --cdrom /var/lib/libvirt/images/ubuntu-24.04.2-live-server-amd64.iso \
  --network network=default,model=virtio \
  --graphics none \
  --console pty,target_type=serial \
  --boot kernel=/var/lib/libvirt/ubuntu24-netboot/vmlinuz,initrd=/var/lib/libvirt/ubuntu24-netboot/initrd,kernel_args="console=ttyS0,115200n8 interface=auto boot=casper automatic-ubiquity"
```

Create a virtual machine for Master 3 with Ubuntu 24.04
```
virt-install \
  --name master-3 \
  --memory 131072 \
  --vcpus 12 \
  --disk path=/var/lib/libvirt/images/master-3.qcow2,format=qcow2 \
  --os-variant=ubuntu24.04 \
  --cdrom /var/lib/libvirt/images/ubuntu-24.04.2-live-server-amd64.iso \
  --network network=kubernetes,model=virtio \
  --graphics vnc \
  --serial pty \
  --console pty 
```

Create a virtual machine for Worker 1 with Ubuntu 24.04
```
virt-install \
  --name worker-1 \
  --memory 131072 \
  --vcpus 12 \
  --disk path=/var/lib/libvirt/images/worker-1.qcow2,format=qcow2 \
  --os-variant=ubuntu24.04 \
  --cdrom /var/lib/libvirt/images/ubuntu-24.04.2-live-server-amd64.iso \
  --network network=kubernetes,model=virtio \
  --graphics vnc \
  --serial pty \
  --console pty 
```

Create a virtual machine for Worker 2 with Ubuntu 24.04
```
virt-install \
  --name worker-2 \
  --memory 131072 \
  --vcpus 12 \
  --disk path=/var/lib/libvirt/images/worker-2.qcow2,format=qcow2 \
  --os-variant=ubuntu24.04 \
  --cdrom /var/lib/libvirt/images/ubuntu-24.04.2-live-server-amd64.iso \
  --network network=kubernetes,model=virtio \
  --graphics vnc \
  --serial pty \
  --console pty 
```

Create a virtual machine for Worker 3 with Ubuntu 24.04
```
virt-install \
  --name worker-3 \
  --memory 131072 \
  --vcpus 12 \
  --disk path=/var/lib/libvirt/images/worker-3.qcow2,format=qcow2 \
  --os-variant=ubuntu24.04 \
  --cdrom /var/lib/libvirt/images/ubuntu-24.04.2-live-server-amd64.iso \
  --network network=kubernetes,model=virtio \
  --graphics vnc \
  --serial pty \
  --console pty 
```
