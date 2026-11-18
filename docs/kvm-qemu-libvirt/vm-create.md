# VM Create

#### 1. Install KVM, Libvirt & Virtualization Packages

Update your package lists and install the full KVM virtualization stack along with command-line tools:

Bash

```
sudo apt update && sudo apt install -y \
  qemu-kvm \
  libvirt-daemon-system \
  libvirt-clients \
  bridge-utils \
  virt-install \
  virtinst \
  cpu-checker
```

#### 2. Enable Libvirt Services & Configure User Permissions

Add your current user to the `libvirt` and `kvm` groups so you can manage VMs without prefixing every command with `sudo`, and start the daemon:

Bash

```
sudo usermod -aG libvirt,kvm $USER
sudo systemctl enable --now libvirtd

# Refresh group membership in the current terminal session
newgrp libvirt
```

#### 3. Configure and Start the Default NAT Network

Ensure Libvirt's default virtual bridge (`virbr0`) on `192.168.122.0/24` is defined, active, and set to autostart:

Bash

```
sudo virsh net-autostart default
sudo virsh net-start default 2>/dev/null || true
sudo virsh net-list --all
```

#### 4. Download the Target OS ISO

Download an ISO (such as Alpine or Ubuntu Cloud/Server) directly into the default Libvirt storage pool:

Bash

```
# Example: Download Alpine Standard ISO
sudo wget https://dl-cdn.alpinelinux.org/alpine/v3.21/releases/x86_64/alpine-standard-3.21.0-x86_64.iso \
  -O /var/lib/libvirt/images/alpine-standard-3.21.0-x86_64.iso
```

#### 5. Create and Launch the Virtual Machine

Run `virt-install` to allocate the virtual disk, bind the network, expose host CPU features (`host-passthrough` for nested virtualization support), and attach directly to the serial console:

Bash

```
sudo virt-install \
  --name vm-master \
  --ram 2048 \
  --vcpus 2 \
  --cpu host-passthrough \
  --disk path=/var/lib/libvirt/images/vm-master.qcow2,size=10,bus=virtio \
  --os-variant generic \
  --cdrom /var/lib/libvirt/images/alpine-standard-3.21.0-x86_64.iso \
  --network network=default,model=virtio \
  --graphics none \
  --console pty,target_type=serial
```

_(Press `Ctrl + ]` at any point to detach from the guest serial console back to the Ubuntu host terminal)._

6\. Useful Everyday Lifecycle Commands

Bash

```
# List all running and inactive VMs
virsh list --all

# Connect to the serial console of a running VM
virsh console vm-master

# Start / Shutdown / Force-stop a VM
virsh start vm-master
virsh shutdown vm-master
virsh destroy vm-master

# Delete VM definition and its storage disk completely
virsh undefine vm-master --remove-all-storage
```
