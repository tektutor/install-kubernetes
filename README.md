# Install Kubernetes in Ubuntu 24.04 64-bit LTS

## Install docker in Ubuntu
```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker $USER
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker

docker --version
docker images
```

## Install KVM Hypervisor in Ubuntu
```
sudo apt update
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils -y
sudo adduser root kvm
sudo systemctl enable --now libvirtd
sudo systemctl status libvirtd
sudo apt install virt-manager -y
sudo apt install -y guestfs-tools
```

## Let's create VMs for 3 master and 3 worker nodes


Create a disk image
```
qemu-img create -f qcow2 /var/lib/libvirt/images/master-1.qcow2 1000G
qemu-img create -f qcow2 /var/lib/libvirt/images/master-2.qcow2 1000G
qemu-img create -f qcow2 /var/lib/libvirt/images/master-3.qcow2 1000G
qemu-img create -f qcow2 /var/lib/libvirt/images/worker-1.qcow2 1000G
qemu-img create -f qcow2 /var/lib/libvirt/images/worker-2.qcow2 1000G
qemu-img create -f qcow2 /var/lib/libvirt/images/worker-3.qcow2 1000G
```

Create a file virt-net.xml
```
<network>
  <name>kubernetes</name>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='kubernetes' stp='on' delay='0'/>
  <domain name='kubernetes'/>
  <ip address='192.168.100.1' netmask='255.255.255.0'>
  </ip>
</network>  
```

Run the below command
```
sudo virsh net-define --file virt-net.xml
sudo virsh net-autostart kubernetes
sudo virsh net-start kubernetes
sudo virsh net-list
```

Expected output
<pre>
sudo virsh net-define --file virt-net.xml
Network kubernetes defined from virt-net.xml
</pre>

Create a virtual machine for Master 1 with Ubuntu 24.04
```
virt-install \
  --name master-1 \
  --memory 131072 \
  --vcpus 12 \
  --disk path=/var/lib/libvirt/images/master-1.qcow2,format=qcow2 \
  --os-variant=ubuntu24.04 \
  --cdrom /var/lib/libvirt/images/ubuntu-24.04.2-live-server-amd64.iso \
  --network network=kubernetes,model=virtio \
  --graphics vnc \
  --serial pty \
  --console pty 
```
