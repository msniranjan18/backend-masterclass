# VM Clone

`virsh-clone` will copy everything stored on disk, but there are two critical details regarding running state and hardware identity to understand:

#### 1. **What Will Be Cloned**

* Installed OS & Packages: All tools installed in `vm-master` (`qemu`, `libvirt`, `dnsmasq`, `iptables`).
* Nested VM Files: The virtual disk `/var/lib/libvirt/images/vm-nested-1.qcow2`, ISOs, network definitions, and Libvirt domain XML configurations.
* Configurations & Users: All network configs, SSH keys, and system services.

#### 2. What Will NOT Be Cloned (Live Memory State)

A standard disk clone captures only data committed to the storage layer.

* The nested VM (`vm-nested-1`) will not be actively running in RAM inside the cloned master when it first boots up.
* In the new clone, `vm-nested-1` will appear in `virsh list --all` in a `shut off` state. You will simply start it using `sudo virsh start vm-nested-1`.

#### 3. The In-Flight / Live Disk Risk (Important)

If you clone `vm-master` while `vm-nested-1` is actively writing to its disk, the nested `.qcow2` file inside the clone might suffer from filesystem inconsistency (similar to pulling the power cord).

**Recommended Safe Process:**

1.  Inside `vm-master`: Shut down the nested VM cleanly:

    Bash

    ```
    sudo virsh shutdown vm-nested-1
    ```
2.  On your WSL Host: Shut down `vm-master` before cloning:

    Bash

    ```
    sudo virsh shutdown vm-master
    ```
3.  On your WSL Host: Run `virt-clone`:

    Bash

    ```
    sudo virt-clone \
      --original vm-master \
      --name vm-worker-1 \
      --auto-clone
    ```

`virt-clone` creates an exact copy of the disk image and automatically generates a new, unique MAC address and UUID for the new master so the two VMs don't collide on your network.\
<br>

{% code overflow="wrap" lineNumbers="true" expandable="true" %}
```
(base) manav@MSNiranjan:/var/lib/libvirt$ virsh list --all
 Id   Name          State
-----------------------------
 1    vm-master     running
 2    vm-worker-1   running
 3    vm-worker-2   running
```
{% endcode %}





If you login to the cloned VMs, you'll notic that the cloned vm has same many similar settings or configuration like hostname, IPs addresses.\
1\. Identical Network Namespace / Subnet on Both Hosts

Both `vm-master` and your clone have an identical, isolated `virbr0` NAT bridge on `192.168.100.0/24`.

* The internal `dnsmasq` DHCP server on each master starts handing out leases beginning at `192.168.100.100`.
* Because both hosts run completely independent, isolated local networks, they do not know about each other's IP assignments.

2\. Identical MAC Address and DHCP Client Identifier (DUID)

Because `vm-master` was cloned bit-for-bit, the nested VM definition inside the clone:

* Has the exact same virtual MAC address on its `vda`/`eth0` interface (`/etc/libvirt/qemu/vm-nested-1.xml`).
* Has the exact same DHCP Client ID / DUID stored in `/etc/machine-id` or `/var/lib/dhcp/` inside the nested Alpine image.

When the nested VM requests an IP, the local `dnsmasq` DHCP server recognizes the MAC address and leases the identical first available IP (e.g., `192.168.100.100`).





Whenever you clone a Linux VM (like `vm-worker-1` or `vm-worker-2`), execute these commands inside the newly cloned machine to clean up duplicated identities and prevent network/security collisions.

#### 1. Set the New Hostname

Replace `NEW_HOSTNAME` (e.g., `vm-worker-1`) with the actual name of your node:

Bash

```
export NEW_NODE_NAME="vm-worker-1"

# Update hostname and hosts files
echo "$NEW_NODE_NAME" | sudo tee /etc/hostname
sudo sed -i "s/vm-master/$NEW_NODE_NAME/g" /etc/hosts

# Apply immediately
sudo hostname -F /etc/hostname
```

#### 2. Reset System Machine ID & DUID

Regenerating the machine ID ensures DHCP servers and system services treat the node as a distinct client:

Bash

```
# Clear machine-id (if present on your distro)
if [ -f /etc/machine-id ]; then
    sudo rm -f /etc/machine-id
    sudo dbus-uuidgen --ensure=/etc/machine-id
fi

# Reset DHCP client leases so it requests a fresh lease
sudo rm -f /var/lib/dhcp/* /var/lib/dhcpcd/* /var/lib/udhcpc/* 2>/dev/null || true
```

#### 3. Regenerate Host SSH Keys

Because the clone has the identical private host keys as `vm-master`, generate fresh host keys to prevent SSH spoofing warnings:

Bash

```
# Delete existing host keys
sudo rm -f /etc/ssh/ssh_host_*

# Generate new host keys
sudo ssh-keygen -A

# Restart SSH service
sudo rc-service sshd restart
```

#### 4. Differentiate the Nested Subnet (Optional but Recommended)

If you run nested VMs inside this worker node, change its internal Libvirt subnet so it does not collide with `vm-master`'s `192.168.100.0/24`:

Bash

```
# Example: Use 192.168.101.0/24 for worker-1 (or 102 for worker-2)
cat <<EOF | sudo tee /etc/libvirt/qemu/networks/default.xml
<network>
  <name>default</name>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <ip address='192.168.101.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.101.100' end='192.168.101.254'/>
    </dhcp>
  </ip>
</network>
EOF

sudo virsh net-destroy default 2>/dev/null || true
sudo virsh net-define /etc/libvirt/qemu/networks/default.xml
sudo virsh net-start default
```

#### 5. Reload Shell Prompt

Apply the changes to your current terminal session immediately:

Bash

```
exec su - $USER
```
