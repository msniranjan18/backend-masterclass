# k8s Cluster Setup - Hard Way

## Kubernetes the Hard Way — Alpine 3.21 on your KVM VMs

Topology (from your `virsh` output):

| Role                 | Hostname    | IP             | Pod CIDR (worker) |
| -------------------- | ----------- | -------------- | ----------------- |
| Control plane + etcd | vm-master   | 192.168.122.10 | —                 |
| Worker 1             | vm-worker-1 | 192.168.122.11 | 10.200.1.0/24     |
| Worker 2             | vm-worker-2 | 192.168.122.12 | 10.200.2.0/24     |

Service CIDR: `10.32.0.0/24` · K8s version: `v1.31.0` (bump `K8S_VER` below if you want newer)

**Important note on Alpine before you invest hours in this:** the referenced repo (msniranjan18/kubernetes-the-hard-way, forked from mmumshad's) targets Debian + systemd. Two things differ meaningfully on Alpine and are handled below:

1. **No systemd** → every long-running component (etcd, apiserver, controller-manager, scheduler, kubelet, kube-proxy, containerd) runs as an **OpenRC** service instead of a systemd unit.
2. **musl libc, not glibc** → official `kubernetes` binaries are statically-linked Go binaries with `CGO_ENABLED=0`, so they normally run fine on musl. But verify this early (Phase 3 has a checkpoint) rather than discovering a problem after PKI setup.

Everything under "HOST" runs on your Mac/Linux machine (`manav@MSNiranjan`). Everything under a VM name runs inside that VM.

***

### Phase 0 — SSH access to the VMs (you said this isn't set up yet)

#### 0.1 Get console access to configure SSH the first time

bash

```bash
# HOST
virsh console vm-master
# log in as root at the console (Alpine default install usually has no root password set,
# or whatever you set during setup — if you never set one, boot into single-user / use
# `virsh console` right after install to set it: passwd root)
```

#### 0.2 Inside each VM (vm-master, vm-worker-1, vm-worker-2) via console

sh

```sh
# VM (all three, one at a time via virsh console)
apk update
apk add openssh openssh-server sudo bash curl wget openssl
rc-update add sshd
rc-service sshd start

# Allow root login with keys (simplifies scp'ing certs later; disable after if you prefer)
sed -i 's/^#PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
rc-service sshd restart

# Set hostname to match the DHCP lease name if not already set
echo "vm-master" > /etc/hostname   # (or vm-worker-1 / vm-worker-2 on those VMs)
hostname -F /etc/hostname
```

Exit the console with `Ctrl+]`.

#### 0.3 Push your SSH key from HOST

bash

```bash
# HOST
[ -f ~/.ssh/id_ed25519 ] || ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

for ip in 192.168.122.10 192.168.122.11 192.168.122.12; do
  ssh-copy-id -i ~/.ssh/id_ed25519.pub root@$ip
done

# sanity check
for ip in 192.168.122.10 192.168.122.11 192.168.122.12; do
  ssh root@$ip 'hostname && cat /etc/alpine-release'
done
```

#### 0.4 Make life easier — SSH config + /etc/hosts everywhere

bash

```bash
# HOST: ~/.ssh/config
cat >> ~/.ssh/config <<'EOF'
Host vm-master
  HostName 192.168.122.10
  User root
Host vm-worker-1
  HostName 192.168.122.11
  User root
Host vm-worker-2
  HostName 192.168.122.12
  User root
EOF
```

bash

```bash
# Run this block against HOST and all 3 VMs (paste into each root shell, or via ssh loop)
cat >> /etc/hosts <<'EOF'
192.168.122.10 vm-master
192.168.122.11 vm-worker-1
192.168.122.12 vm-worker-2
EOF
```

bash

```bash
# quick way to push /etc/hosts to all 3 VMs from HOST
for ip in 192.168.122.10 192.168.122.11 192.168.122.12; do
  ssh root@$ip "grep -q vm-master /etc/hosts || cat >> /etc/hosts <<'EOF'
192.168.122.10 vm-master
192.168.122.11 vm-worker-1
192.168.122.12 vm-worker-2
EOF"
done
```

***

### Phase 1 — OS-level prep (run on vm-master, vm-worker-1, vm-worker-2)

sh

```sh
# ALL VMs
apk update && apk upgrade
apk add bash curl wget vim tar iptables ip6tables ipset conntrack-tools ethtool socat \
        ca-certificates util-linux e2fsprogs

# Disable swap (Alpine usually has none by default, but be safe)
swapoff -a
sed -i '/swap/d' /etc/fstab

# Kernel modules needed by the container runtime + kube-proxy
cat >> /etc/modules <<'EOF'
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

# sysctl for bridged traffic + ip forwarding
cat > /etc/sysctl.d/99-kubernetes.conf <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl -p /etc/sysctl.d/99-kubernetes.conf

# OpenRC's cgroup mounting service (needed by containerd)
rc-update add cgroups boot
rc-service cgroups start
```

***

### Phase 2 — Container runtime: containerd + runc + CNI plugins (all 3 VMs)

sh

```sh
# ALL VMs
apk add containerd runc cni-plugins cri-tools

mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# Alpine has no systemd, so set the cgroup driver to cgroupfs (must match kubelet's later)
sed -i 's/SystemdCgroup = true/SystemdCgroup = false/' /etc/containerd/config.toml

rc-update add containerd default
rc-service containerd start
rc-service containerd status
```

**Checkpoint — do this now, before touching certs:**

sh

```sh
# on vm-master
ctr version
crictl --runtime-endpoint unix:///run/containerd/containerd.sock version
```

If both print clean version output, containerd/runc/musl are fine — proceed. If you see `Exec format error` or dynamic-linking errors, stop and tell me; we'd switch to `gcompat`/glibc shim or a musl-built alternative before going further.

***

### Phase 3 — Download Kubernetes + etcd binaries (all 3 VMs)

sh

```sh
# ALL VMs
K8S_VER=v1.31.0
ETCD_VER=v3.5.15

cd /tmp
for bin in kubectl kube-apiserver kube-controller-manager kube-scheduler kubelet kube-proxy; do
  wget  "https://dl.k8s.io/release/${K8S_VER}/bin/linux/amd64/${bin}"
  chmod +x $bin
  mv $bin /usr/local/bin/
done

wget "https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz"
tar xzf etcd-${ETCD_VER}-linux-amd64.tar.gz
mv etcd-${ETCD_VER}-linux-amd64/etcd* /usr/local/bin/

kubectl version --client
kube-apiserver --version
kubelet --version
kube-proxy --version
etcd --version
```

(Workers don't strictly need `kube-apiserver`/`etcd`/`kube-controller-manager`/`kube-scheduler`, and master doesn't need `kube-proxy`/`kubelet` to run — but it's harmless and simpler to fetch the same set everywhere.)\
\
\
<br>



#### **Issue- Kubelet was having issue**

```
vm-worker-2:~# kubelet version
-sh: kubelet: not found
vm-worker-2:~#
vm-worker-2:~# which kubelet
/usr/local/bin/kubelet
```

`kubelet` is dynamically linked against glibc (`libc.so.6`, `libresolv.so.2`), and something's already partially providing `/lib64/ld-linux-x86-64.so.2` (so `gcompat` may already be present, or you tried it), but `libresolv.so.2` is still missing.

#### Mitigation:

The cleanest fix is the sgerrand `glibc` package (it ships `libresolv.so.2` and unlike `gcompat` which is a lighter shim that doesn't cover everything):

{% code overflow="wrap" lineNumbers="true" expandable="true" %}
```sh
apk add --no-cache ca-certificates wget
wget -O /etc/apk/keys/sgerrand.rsa.pub https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub

GLIBC_VER=2.35-r1
wget https://github.com/sgerrand/alpine-pkg-glibc/releases/download/${GLIBC_VER}/glibc-${GLIBC_VER}.apk
wget https://github.com/sgerrand/alpine-pkg-glibc/releases/download/${GLIBC_VER}/glibc-bin-${GLIBC_VER}.apk

apk add glibc-${GLIBC_VER}.apk glibc-bin-${GLIBC_VER}.apk

kubelet --version
```
{% endcode %}

***

### Phase 4 — Certificate Authority + TLS certs (run on HOST — you already have openssl)

bash

```bash
# HOST

mkdir -p ~/k8s-hard-way/certs && cd ~/k8s-hard-way/certs

MASTER_IP=192.168.122.10
WORKER1_IP=192.168.122.11
WORKER2_IP=192.168.122.12
SERVICE_IP=10.32.0.1   # kubernetes svc's ClusterIP, first addr of SERVICE_CIDR

# --- CA ---
openssl genrsa -out ca.key 2048
openssl req -x509 -new -sha256 -nodes -key ca.key -days 3653 \
  -subj "/CN=KUBERNETES-CA/O=Kubernetes" -out ca.crt

gen_cert () {
  name=$1; subj=$2; sans=$3
  openssl genrsa -out ${name}.key 2048
  openssl req -new -key ${name}.key -subj "$subj" -out ${name}.csr
  if [ -n "$sans" ]; then
    openssl x509 -req -in ${name}.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
      -out ${name}.crt -days 365 -extfile <(printf "subjectAltName=%s" "$sans")
  else
    openssl x509 -req -in ${name}.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
      -out ${name}.crt -days 365
  fi
}

# admin (cluster-admin client cert)
gen_cert admin "/CN=admin/O=system:masters"

# kube-controller-manager
gen_cert kube-controller-manager "/CN=system:kube-controller-manager/O=system:kube-controller-manager"

# kube-scheduler
gen_cert kube-scheduler "/CN=system:kube-scheduler/O=system:kube-scheduler"

# kube-proxy
gen_cert kube-proxy "/CN=system:kube-proxy/O=system:node-proxier"

# kubelet client certs — one per worker, CN must be system:node:<hostname>
gen_cert vm-worker-1 "/CN=system:node:vm-worker-1/O=system:nodes" \
  "IP:${WORKER1_IP}"
gen_cert vm-worker-2 "/CN=system:node:vm-worker-2/O=system:nodes" \
  "IP:${WORKER2_IP}"

# kube-apiserver server cert — needs every name/IP clients will use to reach it
gen_cert kubernetes "/CN=kubernetes/O=Kubernetes" \
  "DNS:kubernetes,DNS:kubernetes.default,DNS:kubernetes.default.svc,DNS:kubernetes.default.svc.cluster.local,DNS:vm-master,IP:${MASTER_IP},IP:${SERVICE_IP},IP:127.0.0.1"

# service-account signing key pair (no CA signing needed — apiserver/controller-manager just need the keypair)
openssl genrsa -out service-account.key 2048
openssl req -new -x509 -sha256 -key service-account.key -subj "/CN=service-accounts" \
  -days 3653 -out service-account.crt

ls
```

Distribute:

bash

```bash
# HOST
scp ca.crt ca.key kubernetes.crt kubernetes.key service-account.crt service-account.key \
    kube-controller-manager.crt kube-controller-manager.key \
    kube-scheduler.crt kube-scheduler.key \
    root@vm-master:/tmp/

scp ca.crt vm-worker-1.crt vm-worker-1.key kube-proxy.crt kube-proxy.key \
    root@vm-worker-1:/tmp/
scp ca.crt vm-worker-2.crt vm-worker-2.key kube-proxy.crt kube-proxy.key \
    root@vm-worker-2:/tmp/
```

On each VM, move certs into place:

sh

```sh
# vm-master
mkdir -p /var/lib/kubernetes/pki
mv /tmp/{ca.crt,ca.key,kubernetes.crt,kubernetes.key,service-account.crt,service-account.key,\
kube-controller-manager.crt,kube-controller-manager.key,kube-scheduler.crt,kube-scheduler.key} \
  /var/lib/kubernetes/pki/
```

sh

```sh
#!/bin/bash

# vm-worker-1 (and same pattern on vm-worker-2 with its own name)
mkdir -p /var/lib/kubelet/pki /var/lib/kube-proxy/pki
mv /tmp/ca.crt /var/lib/kubelet/pki/
mv /tmp/vm-worker-1.crt /var/lib/kubelet/pki/kubelet.crt
mv /tmp/vm-worker-1.key /var/lib/kubelet/pki/kubelet.key
mv /tmp/ca.crt /var/lib/kube-proxy/pki/ca.crt 2>/dev/null || cp /var/lib/kubelet/pki/ca.crt /var/lib/kube-proxy/pki/ca.crt
mv /tmp/kube-proxy.crt /var/lib/kube-proxy/pki/
mv /tmp/kube-proxy.key /var/lib/kube-proxy/pki/
```

{% code overflow="wrap" lineNumbers="true" expandable="true" %}
```sh
# vm-worker-2
#(and same pattern on vm-worker-1 with its own name)
mkdir -p /var/lib/kubelet/pki /var/lib/kube-proxy/pki
mv /tmp/ca.crt /var/lib/kubelet/pki/
mv /tmp/vm-worker-2.crt /var/lib/kubelet/pki/kubelet.crt
mv /tmp/vm-worker-2.key /var/lib/kubelet/pki/kubelet.key
mv /tmp/ca.crt /var/lib/kube-proxy/pki/ca.crt 2>/dev/null || cp /var/lib/kubelet/pki/ca.crt /var/lib/kube-proxy/pki/ca.crt
mv /tmp/kube-proxy.crt /var/lib/kube-proxy/pki/
mv /tmp/kube-proxy.key /var/lib/kube-proxy/pki/
```
{% endcode %}

***

### Phase 5 — Kubeconfigs (generate on HOST, distribute)

bash

{% code overflow="wrap" lineNumbers="true" expandable="true" %}
```bash
# HOST, still in ~/k8s-hard-way/certs
MASTER_IP=192.168.122.10

kconf () {
  name=$1; cert=$2; key=$3; user=$4
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt --embed-certs=true \
    --server=https://${MASTER_IP}:6443 --kubeconfig=${name}.kubeconfig
  kubectl config set-credentials ${user} \
    --client-certificate=${cert} --client-key=${key} --embed-certs=true \
    --kubeconfig=${name}.kubeconfig
  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way --user=${user} --kubeconfig=${name}.kubeconfig
  kubectl config use-context default --kubeconfig=${name}.kubeconfig
}

kconf kube-controller-manager kube-controller-manager.crt kube-controller-manager.key system:kube-controller-manager
kconf kube-scheduler kube-scheduler.crt kube-scheduler.key system:kube-scheduler
kconf kube-proxy kube-proxy.crt kube-proxy.key system:kube-proxy
kconf vm-worker-1 vm-worker-1.crt vm-worker-1.key system:node:vm-worker-1
kconf vm-worker-2 vm-worker-2.crt vm-worker-2.key system:node:vm-worker-2

# admin kubeconfig — use for your own kubectl on HOST later; server = localhost via SSH tunnel or MASTER_IP directly
kconf admin admin.crt admin.key admin

scp kube-controller-manager.kubeconfig kube-scheduler.kubeconfig root@vm-master:/var/lib/kubernetes/
scp kube-proxy.kubeconfig vm-worker-1.kubeconfig root@vm-worker-1:/tmp/
scp kube-proxy.kubeconfig vm-worker-2.kubeconfig root@vm-worker-2:/tmp/
```
{% endcode %}

sh

<pre class="language-sh" data-overflow="wrap" data-line-numbers><code class="lang-sh"><strong># vm-worker-1
</strong>mv /tmp/vm-worker-1.kubeconfig /var/lib/kubelet/kubeconfig
mv /tmp/kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
# same on vm-worker-2 with vm-worker-2.kubeconfig
</code></pre>

{% code overflow="wrap" lineNumbers="true" expandable="true" %}
```sh
# vm-worker-2
mv /tmp/vm-worker-2.kubeconfig /var/lib/kubelet/kubeconfig
mv /tmp/kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
# same on vm-worker-2 with vm-worker-2.kubeconfig
```
{% endcode %}

***

### Phase 6 — Data encryption config for Secrets (HOST → vm-master)

bash

{% code title="" overflow="wrap" lineNumbers="true" %}
```bash
# HOST

# Generate Key
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

# Generate Config using the key
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF

# Copy to master
scp encryption-config.yaml root@vm-master:/var/lib/kubernetes/
```
{% endcode %}

***

### Phase 7 — etcd on vm-master (OpenRC service)

sh

{% code title="" overflow="wrap" lineNumbers="true" %}
```sh
# vm-master
mkdir -p /etc/etcd /var/lib/etcd
cp /var/lib/kubernetes/pki/ca.crt /var/lib/kubernetes/pki/kubernetes.crt /var/lib/kubernetes/pki/kubernetes.key /etc/etcd/
chmod 700 /var/lib/etcd

cat > /etc/init.d/etcd <<'EOF'
#!/sbin/openrc-run
name="etcd"
command="/usr/local/bin/etcd"
command_args="--name vm-master \
  --cert-file=/etc/etcd/kubernetes.crt \
  --key-file=/etc/etcd/kubernetes.key \
  --peer-cert-file=/etc/etcd/kubernetes.crt \
  --peer-key-file=/etc/etcd/kubernetes.key \
  --trusted-ca-file=/etc/etcd/ca.crt \
  --peer-trusted-ca-file=/etc/etcd/ca.crt \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://192.168.122.10:2380 \
  --listen-peer-urls https://192.168.122.10:2380 \
  --listen-client-urls https://192.168.122.10:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.122.10:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster vm-master=https://192.168.122.10:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd"
command_background="yes"
pidfile="/run/etcd.pid"
output_log="/var/log/etcd.log"
error_log="/var/log/etcd.err"
depend() { need net; after containerd; }
EOF

chmod +x /etc/init.d/etcd

rc-update add etcd default
rc-service etcd start
rc-service etcd status

ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/kubernetes.crt --key=/etc/etcd/kubernetes.key
```
{% endcode %}

***

### Phase 8 — Control plane on vm-master (OpenRC services)

sh

{% code overflow="wrap" lineNumbers="true" %}
```sh
# vm-master
mkdir -p /var/lib/kubernetes/logs

# --- kube-apiserver ---
cat > /etc/init.d/kube-apiserver <<'EOF'
#!/sbin/openrc-run
name="kube-apiserver"
command="/usr/local/bin/kube-apiserver"
command_args="--advertise-address=192.168.122.10 \
  --allow-privileged=true \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --client-ca-file=/var/lib/kubernetes/pki/ca.crt \
  --etcd-cafile=/var/lib/kubernetes/pki/ca.crt \
  --etcd-certfile=/var/lib/kubernetes/pki/kubernetes.crt \
  --etcd-keyfile=/var/lib/kubernetes/pki/kubernetes.key \
  --etcd-servers=https://127.0.0.1:2379 \
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \
  --kubelet-certificate-authority=/var/lib/kubernetes/pki/ca.crt \
  --kubelet-client-certificate=/var/lib/kubernetes/pki/kubernetes.crt \
  --kubelet-client-key=/var/lib/kubernetes/pki/kubernetes.key \
  --runtime-config=api/all=true \
  --service-account-key-file=/var/lib/kubernetes/pki/service-account.crt \
  --service-account-signing-key-file=/var/lib/kubernetes/pki/service-account.key \
  --service-account-issuer=https://192.168.122.10:6443 \
  --service-cluster-ip-range=10.32.0.0/24 \
  --tls-cert-file=/var/lib/kubernetes/pki/kubernetes.crt \
  --tls-private-key-file=/var/lib/kubernetes/pki/kubernetes.key \
  --v=2"
command_background="yes"
pidfile="/run/kube-apiserver.pid"
output_log="/var/lib/kubernetes/logs/apiserver.log"
error_log="/var/lib/kubernetes/logs/apiserver.err"
depend() { need net; after etcd; }
EOF
chmod +x /etc/init.d/kube-apiserver

# --- kube-controller-manager ---
cat > /etc/init.d/kube-controller-manager <<'EOF'
#!/sbin/openrc-run
name="kube-controller-manager"
command="/usr/local/bin/kube-controller-manager"
command_args="--bind-address=0.0.0.0 \
  --cluster-cidr=10.200.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/var/lib/kubernetes/pki/ca.crt \
  --cluster-signing-key-file=/var/lib/kubernetes/pki/ca.key \
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \
  --root-ca-file=/var/lib/kubernetes/pki/ca.crt \
  --service-account-private-key-file=/var/lib/kubernetes/pki/service-account.key \
  --service-cluster-ip-range=10.32.0.0/24 \
  --use-service-account-credentials=true \
  --v=2"
command_background="yes"
pidfile="/run/kube-controller-manager.pid"
output_log="/var/lib/kubernetes/logs/controller-manager.log"
error_log="/var/lib/kubernetes/logs/controller-manager.err"
depend() { need net; after kube-apiserver; }
EOF
chmod +x /etc/init.d/kube-controller-manager

# --- kube-scheduler ---
cat > /etc/init.d/kube-scheduler <<'EOF'
#!/sbin/openrc-run
name="kube-scheduler"
command="/usr/local/bin/kube-scheduler"
command_args="--kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig --v=2"
command_background="yes"
pidfile="/run/kube-scheduler.pid"
output_log="/var/lib/kubernetes/logs/scheduler.log"
error_log="/var/lib/kubernetes/logs/scheduler.err"
depend() { need net; after kube-apiserver; }
EOF
chmod +x /etc/init.d/kube-scheduler

rc-update add kube-apiserver default
rc-update add kube-controller-manager default
rc-update add kube-scheduler default
rc-service kube-apiserver start
rc-service kube-controller-manager start
rc-service kube-scheduler start

sleep 3
rc-service kube-apiserver status
curl -sk https://127.0.0.1:6443/version
```
{% endcode %}

#### RBAC so kube-apiserver can reach kubelets

sh

{% code overflow="wrap" lineNumbers="true" %}
```sh
# vm-master
export KUBECONFIG=/var/lib/kubernetes/kube-controller-manager.kubeconfig
# actually apply as admin — copy admin.kubeconfig here too, or run this from HOST once kubectl works (Phase 9)
```
{% endcode %}

(Do this from HOST after Phase 9, it's easier — see below.)

***

### Phase 9 — kubectl access from HOST

bash

{% code overflow="wrap" lineNumbers="true" %}
```bash
# HOST
cp ~/k8s-hard-way/certs/admin.kubeconfig ~/.kube/config
kubectl config use-context default
kubectl get componentstatuses 2>/dev/null
kubectl cluster-info

# Apply the kubelet-authorization RBAC (standard KTHW manifests)
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups: [""]
    resources: ["nodes/proxy","nodes/stats","nodes/log","nodes/spec","nodes/metrics"]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```
{% endcode %}

***

### Phase 10 — Workers: containerd already done (Phase 2); now kubelet + kube-proxy

Repeat on **vm-worker-1** (`POD_CIDR=10.200.1.0/24`) and **vm-worker-2** (`POD_CIDR=10.200.2.0/24`), substituting the hostname/CIDR each time.

sh

{% code overflow="wrap" lineNumbers="true" expandable="true" %}
```sh
# vm-worker-1
mkdir -p /var/lib/kubelet /var/lib/kube-proxy /var/lib/kubernetes/pki
cp /var/lib/kubelet/pki/ca.crt /var/lib/kubernetes/pki/ca.crt

cat > /var/lib/kubelet/kubelet-config.yaml <<EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubelet/pki/ca.crt"
authorization:
  mode: Webhook
cgroupDriver: cgroupfs
containerRuntimeEndpoint: unix:///run/containerd/containerd.sock
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "10.200.1.0/24"
resolvConf: "/etc/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/pki/kubelet.crt"
tlsPrivateKeyFile: "/var/lib/kubelet/pki/kubelet.key"
EOF

cat > /etc/init.d/kubelet <<'EOF'
#!/sbin/openrc-run
name="kubelet"
command="/usr/local/bin/kubelet"
command_args="--config=/var/lib/kubelet/kubelet-config.yaml \
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --hostname-override=vm-worker-1 \
  --v=2"
command_background="yes"
pidfile="/run/kubelet.pid"
output_log="/var/lib/kubernetes/logs/kubelet.log"
error_log="/var/lib/kubernetes/logs/kubelet.err"
depend() { need net; after containerd; }
EOF
chmod +x /etc/init.d/kubelet

cat > /var/lib/kube-proxy/kube-proxy-config.yaml <<EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF

cat > /etc/init.d/kube-proxy <<'EOF'
#!/sbin/openrc-run
name="kube-proxy"
command="/usr/local/bin/kube-proxy"
command_args="--config=/var/lib/kube-proxy/kube-proxy-config.yaml --v=2"
command_background="yes"
pidfile="/run/kube-proxy.pid"
output_log="/var/lib/kubernetes/logs/kube-proxy.log"
error_log="/var/lib/kubernetes/logs/kube-proxy.err"
depend() { need net; after kubelet; }
EOF
chmod +x /etc/init.d/kube-proxy

mkdir -p /var/lib/kubernetes/logs
rc-update add kubelet default
rc-update add kube-proxy default
rc-service kubelet start
rc-service kube-proxy start
rc-service kubelet status
```
{% endcode %}

{% code title="" overflow="wrap" lineNumbers="true" expandable="true" %}
```sh
# vm-worker-2
mkdir -p /var/lib/kubelet /var/lib/kube-proxy /var/lib/kubernetes/pki
cp /var/lib/kubelet/pki/ca.crt /var/lib/kubernetes/pki/ca.crt

cat > /var/lib/kubelet/kubelet-config.yaml <<EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubelet/pki/ca.crt"
authorization:
  mode: Webhook
cgroupDriver: cgroupfs
containerRuntimeEndpoint: unix:///run/containerd/containerd.sock
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "10.200.2.0/24"
resolvConf: "/etc/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/pki/kubelet.crt"
tlsPrivateKeyFile: "/var/lib/kubelet/pki/kubelet.key"
EOF

cat > /etc/init.d/kubelet <<'EOF'
#!/sbin/openrc-run
name="kubelet"
command="/usr/local/bin/kubelet"
command_args="--config=/var/lib/kubelet/kubelet-config.yaml \
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --hostname-override=vm-worker-2 \
  --v=2"
command_background="yes"
pidfile="/run/kubelet.pid"
output_log="/var/lib/kubernetes/logs/kubelet.log"
error_log="/var/lib/kubernetes/logs/kubelet.err"
depend() { need net; after containerd; }
EOF
chmod +x /etc/init.d/kubelet

cat > /var/lib/kube-proxy/kube-proxy-config.yaml <<EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF

cat > /etc/init.d/kube-proxy <<'EOF'
#!/sbin/openrc-run
name="kube-proxy"
command="/usr/local/bin/kube-proxy"
command_args="--config=/var/lib/kube-proxy/kube-proxy-config.yaml --v=2"
command_background="yes"
pidfile="/run/kube-proxy.pid"
output_log="/var/lib/kubernetes/logs/kube-proxy.log"
error_log="/var/lib/kubernetes/logs/kube-proxy.err"
depend() { need net; after kubelet; }
EOF

chmod +x /etc/init.d/kube-proxy

mkdir -p /var/lib/kubernetes/logs

rc-update add kubelet default
rc-update add kube-proxy default

rc-service kubelet start
rc-service kube-proxy start

rc-service kubelet status
rc-service kube-proxy status

```
{% endcode %}

From HOST:

bash

{% code overflow="wrap" lineNumbers="true" %}
```bash
kubectl get nodes
# vm-worker-1, vm-worker-2 should appear as NotReady until CNI (Phase 11) is up
```
{% endcode %}

***

### Phase 11 — Pod networking (routed, no CNI plugin needed — matches KTHW's simplest approach)

Since there's no cloud router here, add static routes on the KVM host (or on each worker) so pod traffic between worker1/worker2 CIDRs is routable across the `default` libvirt bridge (virbr0).

bash

```bash
# HOST (the KVM hypervisor) — routes so pod-to-pod traffic across nodes works
sudo ip route add 10.200.1.0/24 via 192.168.122.11
sudo ip route add 10.200.2.0/24 via 192.168.122.12
```

On each worker, configure containerd's CNI to use a simple bridge with that node's pod CIDR:

sh

{% code overflow="wrap" expandable="true" %}
```sh
# vm-worker-1 (adjust subnet to 10.200.2.0/24 on vm-worker-2)
mkdir -p /etc/cni/net.d
cat > /etc/cni/net.d/10-bridge.conf <<'EOF'
{
  "cniVersion": "1.0.0",
  "name": "bridge",
  "type": "bridge",
  "bridge": "cnio0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "ranges": [[{"subnet": "10.200.1.0/24"}]],
    "routes": [{"dst": "0.0.0.0/0"}]
  }
}
EOF

rc-service containerd restart
rc-service kubelet restart

rc-service containerd status
rc-service kubelet status

```
{% endcode %}

<pre class="language-sh" data-title="" data-overflow="wrap" data-line-numbers data-expandable="true"><code class="lang-sh"># vm-worker-2 (adjust subnet to 10.200.2.0/24 on vm-worker-2)
mkdir -p /etc/cni/net.d
cat > /etc/cni/net.d/10-bridge.conf &#x3C;&#x3C;'EOF'
{
  "cniVersion": "1.0.0",
  "name": "bridge",
  "type": "bridge",
  "bridge": "cnio0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "ranges": [[{"subnet": "10.200.2.0/24"}]],
    "routes": [{"dst": "0.0.0.0/0"}]
  }
}
EOF

<strong>rc-service containerd restart
</strong>rc-service kubelet restart

rc-service containerd status
rc-service kubelet status

</code></pre>



Download CNI on both worker using:

{% code title="" overflow="wrap" lineNumbers="true" expandable="true" %}
```bash
# Stop the current wget (Ctrl+C)

# Run this instead:
CNI_VER=1.5.1
mkdir -p /opt/cni/bin

curl -L -o cni-plugins.tgz "https://github.com/containernetworking/plugins/releases/download/v${CNI_VER}/cni-plugins-linux-amd64-v${CNI_VER}.tgz"

# Extract
tar -C /opt/cni/bin -xzvf cni-plugins.tgz

# Cleanup
rm cni-plugins.tgz

# Verify
ls /opt/cni/bin/
```
{% endcode %}

{% code title="" overflow="wrap" lineNumbers="true" expandable="true" %}
```
rc-service kubelet restart
```
{% endcode %}







Add pod-CIDR routes between the two workers directly too (belt-and-braces, useful if you ever move off virbr0):

sh

```sh
# vm-worker-1
ip route add 10.200.2.0/24 via 192.168.122.12
# vm-worker-2
ip route add 10.200.1.0/24 via 192.168.122.11
```

***

### Phase 12 — CoreDNS

bash

```bash
# HOST
kubectl apply -f https://raw.githubusercontent.com/mmumshad/kubernetes-the-hard-way/master/deployments/coredns.yaml
kubectl -n kube-system get pods -l k8s-app=kube-dns
```

***

### Phase 13 — Smoke test

bash

```bash
# HOST
kubectl get nodes -o wide
kubectl create deployment nginx --image=nginx
kubectl get pods -o wide -w   # wait for Running

kubectl create secret generic test-secret --from-literal=mykey=myvalue
kubectl get secrets test-secret -o yaml

kubectl expose deployment nginx --port=80 --type=ClusterIP
kubectl run busybox --image=busybox:1.36 --restart=Never -- sleep 3600
kubectl exec busybox -- wget -qO- nginx   # should return the nginx welcome HTML
```

If nginx runs, gets an IP in the right pod CIDR, and DNS resolves `nginx` from busybox, your from-scratch cluster is working end to end.

***

### Where you'll most likely hit friction on Alpine specifically

* **musl vs glibc**: flagged in Phase 3 — if the official k8s binaries don't run, fall back to `apk add gcompat` or build from source with `CGO_ENABLED=0` (already the default upstream build, so this is unlikely but worth knowing).
* **OpenRC logs**: no `journalctl` — tail `/var/lib/kubernetes/logs/*.log` and `*.err` directly when a service won't come up.
* **iptables**: Alpine's `iptables` package may default to nft-backend; if kube-proxy's iptables mode misbehaves, check `apk info iptables` and consider `apk add iptables-legacy` and switching alternatives via `update-alternatives` (or Alpine's `iptables --version` to confirm backend).
