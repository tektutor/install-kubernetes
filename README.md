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
sudo apt install virt-manager, guestfs-tools qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-viewer -y

sudo adduser root kvm
sudo systemctl enable --now libvirtd
sudo systemctl status libvirtd
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
