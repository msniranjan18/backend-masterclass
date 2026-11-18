# Nested VM

1\. Install Virtualization & Networking Dependencies Install QEMU emulator tools, the Libvirt daemon, network routing utilities, and download the Alpine ISO inside `vm-master`:

Bash

```
sudo apk update && sudo apk add \
  qemu-img \
  qemu-system-x86_64 \
  libvirt-daemon \
  libvirt-client \
  virt-install \
  dbus \
  dnsmasq \
  iptables \
  iproute2

sudo rc-update add dbus default
sudo rc-update add libvirtd default
sudo rc-service dbus start
sudo rc-service libvirtd start

sudo wget https://dl-cdn.alpinelinux.org/alpine/v3.21/releases/x86_64/alpine-standard-3.21.0-x86_64.iso \
  -O /var/lib/libvirt/images/alpine-standard-3.21.0-x86_64.iso
```

2\. Load the TUN/TAP Kernel Module Enable virtual network device creation by loading the `tun` driver:

Bash

```
sudo modprobe tun
echo "tun" | sudo tee -a /etc/modules
```

3\. Configure Isolated Subnet to Avoid Host Route Collision Define and activate an isolated `192.168.100.0/24` virtual network so it does not conflict with your WSL host's `192.168.122.0/24` network:

Bash

```
cat <<EOF | sudo tee /etc/libvirt/qemu/networks/default.xml
<network>
  <name>default</name>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <ip address='192.168.100.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.100.100' end='192.168.100.254'/>
    </dhcp>
  </ip>
</network>
EOF

sudo virsh net-destroy default 2>/dev/null || true
sudo virsh net-undefine default 2>/dev/null || true
sudo virsh net-define /etc/libvirt/qemu/networks/default.xml
sudo virsh net-start default
sudo virsh net-autostart default
```

4\. Deploy Nested VM via QEMU Software Emulation Launch the nested guest using `--virt-type qemu` (bypassing Hyper-V L2 VMX entry faults) and immediately attach to the serial console:

Bash

```
sudo virt-install \
  --virt-type qemu \
  --name vm-nested-1 \
  --ram 512 \
  --vcpus 1 \
  --disk path=/var/lib/libvirt/images/vm-nested-1.qcow2,size=5,bus=virtio \
  --os-variant generic \
  --cdrom /var/lib/libvirt/images/alpine-standard-3.21.0-x86_64.iso \
  --network network=default,model=virtio \
  --graphics none \
  --noautoconsole && sudo virsh console vm-nested-1
```

run setup-alpine to configure the os.
