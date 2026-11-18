# Static DHCP reservations

To apply static DHCP reservations without breaking the live network bridge or losing guest IP leases, use `virsh net-update` (hot updates without restarting the network) or perform a clean cold-cycle.

#### Method 1: Live Hot Update (Recommended — Zero Downtime)

Run these directly on your WSL Host while the VMs are running:

Bash

```
# Add static DHCP lease reservations into the active network
virsh net-update default add ip-dhcp-host \
  "<host mac='52:54:00:3a:e2:65' name='vm-master' ip='192.168.122.10'/>" \
  --live --config

virsh net-update default add ip-dhcp-host \
  "<host mac='52:54:00:b3:2b:ab' name='vm-worker-1' ip='192.168.122.11'/>" \
  --live --config

virsh net-update default add ip-dhcp-host \
  "<host mac='52:54:00:b6:8e:5f' name='vm-worker-2' ip='192.168.122.12'/>" \
  --live --config
```

Then refresh the DHCP client on each Alpine guest:

Bash

```
# Run inside vm-master, vm-worker-1, vm-worker-2
sudo udhcpc -i eth0 -n -q
```

#### Method 2: Clean Re-Definition via XML File

If editing the full network XML directly:

1\. Write the Network XML on WSL Host:

Bash

```
cat <<EOF | sudo tee /tmp/default-net.xml
<network>
  <name>default</name>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr0' stp='on' delay='0'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.100' end='192.168.122.254'/>
      <host mac='52:54:00:3a:e2:65' name='vm-master' ip='192.168.122.10'/>
      <host mac='52:54:00:b3:2b:ab' name='vm-worker-1' ip='192.168.122.11'/>
      <host mac='52:54:00:b6:8e:5f' name='vm-worker-2' ip='192.168.122.12'/>
    </dhcp>
  </ip>
</network>
EOF
```

2\. Cold-Cycle Network & VMs to Re-bind TAP Devices:

Bash

```
# Destroy running guests so their tap devices detach cleanly
virsh destroy vm-master
virsh destroy vm-worker-1
virsh destroy vm-worker-2

# Re-define and restart the libvirt network
virsh net-destroy default
virsh net-define /tmp/default-net.xml
virsh net-start default

# Start the guests back up (fresh TAP creation plugged into virbr0)
virsh start vm-master
virsh start vm-worker-1
virsh start vm-worker-2
```

3\. Verify Assigned Leases on WSL Host:

Bash

```
virsh net-dhcp-leases default
```





#### The Issue: Guest IPs Were Lost

The loss of IP connectivity was caused by combining a hypervisor-level network restart (`virsh net-destroy`) with warm reboots of the guest VMs, leaving the virtual TAP interfaces stranded.

**1. The Root Cause (TAP Interface Disconnection):**

* KVM/QEMU connects a guest VM to the host bridge (`virbr0`) using a virtual network interface called a TAP device (e.g., `vnet0`).
* When you run `virsh net-destroy default`, libvirt deletes the `virbr0` bridge and terminates the `dnsmasq` DHCP process on the host.
* When you run `virsh net-start default`, libvirt recreates `virbr0` and restarts `dnsmasq`.
* However, any already running VMs do not know the bridge was destroyed. Their TAP interfaces (`vnet0`, `vnet1`) are left dangling—unattached to the newly created `virbr0` bridge.

**2. Why the VMs Failed to Recover:**

* Executing `virsh reboot` performs a graceful shutdown inside the guest OS, but QEMU keeps the VM hypervisor process alive. Because the QEMU process never fully terminated, it never requested libvirt to destroy and recreate the `vnet` interface.
* Consequently, when the Alpine guests rebooted and broadcasted DHCP requests (`udhcpc: broadcasting discover`), the packets reached the dangling TAP interfaces and were dropped. The DHCP traffic never reached `virbr0` or the host's `dnsmasq` server.\
  <br>

#### The Mitigation:

#### Phase 1: Dynamic IP Recovery (Restoring Connectivity)

To recover dynamic DHCP leases, the host network daemon, firewall rules, and guest hypervisor processes had to be cleanly cold-cycled and re-synchronized.

**Step 1: Re-initialize the Host Virtual Network & Daemons (WSL Host)**

Force a full reload of the Libvirt daemon and restart the network to ensure `dnsmasq` binds correctly to `virbr0`:

Bash

```
# Restart libvirt daemon services on Ubuntu host
sudo systemctl restart libvirtd virtqemud 2>/dev/null || sudo service libvirtd restart

# Cleanly cycle the default network
sudo virsh net-destroy default 2>/dev/null || true
sudo virsh net-start default

# Verify dnsmasq process is actively listening on virbr0
ps aux | grep dnsmasq | grep virbr0
```

**Step 2: Unblock Host Firewall Rules (WSL Host)**

Ensure `iptables` does not drop incoming DHCP discovery packets (UDP 67/68) or forward traffic across `virbr0`:

Bash

```
sudo iptables -I INPUT -i virbr0 -p udp --dport 67 -j ACCEPT
sudo iptables -I INPUT -i virbr0 -p udp --dport 68 -j ACCEPT
sudo iptables -I FORWARD -i virbr0 -j ACCEPT
```

**Step 3: Perform a Full Cold Power-Cycle of All Guests (WSL Host)**

Hard-stopping the VMs forces QEMU to release the orphaned TAP handles and create fresh virtual interfaces linked directly to `virbr0` on boot:

Bash

```
# Hard-stop running QEMU processes
virsh destroy vm-master
virsh destroy vm-worker-1
virsh destroy vm-worker-2

# Power on the guests with fresh virtual TAP bindings
virsh start vm-master
virsh start vm-worker-1
virsh start vm-worker-2

# Verify TAP devices (vnet0, vnet1, vnet2) are attached to virbr0
ip link show master virbr0
```

**Step 4: Re-trigger DHCP Client in Guest OS (Inside Guests)**

Access each VM via serial console (`virsh console <vm-name>`) and force `udhcpc` to request a fresh dynamic lease:

Bash

```
# Ensure /etc/network/interfaces is configured for DHCP
cat <<EOF | sudo tee /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
EOF

# Force immediate DHCP discovery with verbose output
sudo udhcpc -v -i eth0 -n

# Verify dynamic IP assignment
ip a show dev eth0
```

#### Phase 2: Applying Static IP Reservations (Without Network Downtime)

Once dynamic connectivity was fully restored, the static IP mappings were applied safely without destroying the network or disconnecting the active TAP devices.

**Step 1: Add Live Host Reservations via `virsh net-update` (WSL Host)**

Using `virsh net-update` injects DHCP mappings directly into both the live running `dnsmasq` memory state and the persistent XML configuration:

Bash

```
virsh net-update default add ip-dhcp-host \
  "<host mac='52:54:00:3a:e2:65' name='vm-master' ip='192.168.122.10'/>" \
  --live --config

virsh net-update default add ip-dhcp-host \
  "<host mac='52:54:00:b3:2b:ab' name='vm-worker-1' ip='192.168.122.11'/>" \
  --live --config

virsh net-update default add ip-dhcp-host \
  "<host mac='52:54:00:b6:8e:5f' name='vm-worker-2' ip='192.168.122.12'/>" \
  --live --config
```

**Step 2: Renew Guest DHCP Leases (Inside Guests)**

Instruct the guest OS to release its old dynamic IP and acquire the newly reserved static IP:

Bash

```
# Run inside vm-master, vm-worker-1, and vm-worker-2
sudo udhcpc -i eth0 -n -q
```

**Step 3: Validate Final Network State (WSL Host)**

Verify all nodes acquired their designated static IPs:

Bash

```
virsh net-dhcp-leases default
```



