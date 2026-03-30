# F5 CIS + NGINX Plus IC Lab — Deployment Guide

## Shared Infrastructure: Kubernetes + BIG-IP

> **Audience:** Lab builder. This guide covers the shared infrastructure both modes need. Then pick your mode:
>
> | Mode | Guide | What It Does |
> |------|-------|-------------|
> | **A** | [Mode A: CIS Standalone](DEPLOYMENT_GUIDE_MODE_A.md) | BIG-IP → Pods directly |
> | **B** | [Mode B: CIS + IngressLink](DEPLOYMENT_GUIDE_MODE_B.md) | BIG-IP → NGINX IC → Pods |
>
> **Complete Steps 1-2 below first**, then follow the mode-specific guide.

---

## Table of Contents

1. [Build the Kubernetes Cluster](#step-1-build-the-kubernetes-cluster)
2. [Prepare BIG-IP](#step-2-prepare-big-ip)
3. [Troubleshooting](#troubleshooting)

---

## Prerequisites

| Component | Requirement |
|-----------|-------------|
| **K8s VM** | Ubuntu 22.04, 4 CPU, 8 GB RAM, 50 GB disk |
| **BIG-IP VE** | v15.1+ on a separate VM, LTM + ASM provisioned |
| **AS3 Extension** | Installed on BIG-IP ([GitHub releases](https://github.com/F5Networks/f5-appsvcs-extension/releases)) |
| **Network** | K8s node and BIG-IP management interface must be routable to each other |
| **NGINX Plus License** | JWT token from [MyF5](https://my.f5.com) — **Mode B only** |
| **Internet** | Both VMs need internet access for pulling images |

---

## Step 1: Build the Kubernetes Cluster

We use the companion install script to stand up a single-node kubeadm cluster with containerd, Flannel CNI, and Helm.

```bash
# SSH into your Ubuntu VM as root (or use sudo)

# Clone the install script
git clone https://github.com/therealnoof/k8s-nginx-install.git

# Run it (optionally pass a K8s version, default is 1.32.0)
chmod +x k8s-nginx-install/k8s-install.sh
bash k8s-nginx-install/k8s-install.sh

# The script sets KUBECONFIG automatically. Verify:
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl get nodes
```

**Expected output:**
```
NAME        STATUS   ROLES           AGE   VERSION
ubuntu-vm   Ready    control-plane   1m    v1.32.0
```

> **Note:** If prompted about a kernel upgrade during the script, choose "keep the local version currently installed."

**Verify Helm is available:**
```bash
helm version
```

---

## Step 2: Prepare BIG-IP

These steps are done in the **BIG-IP GUI** (TMUI) by NetOps.

### 2a. Ensure Required Modules Are Provisioned

1. Log into BIG-IP GUI → **System → Resource Provisioning**
2. Verify these are set to at least **Nominal**:
   - **Local Traffic (LTM)** — required for virtual servers and pools
   - **Application Security (ASM)** — required for WAF policies

### 2b. Install the AS3 Extension

CIS uses **AS3 (Application Services 3)** to push declarations to BIG-IP. Without the AS3 RPM installed, CIS will crash with: `AS3 RPM is not installed on BIGIP`.

1. Download the latest AS3 RPM from [F5 GitHub AS3 releases](https://github.com/F5Networks/f5-appsvcs-extension/releases)
2. In BIG-IP GUI: **iApps → Package Management LX → Import**
3. Upload the RPM file and wait for it to show as installed

**Or install via CLI:**

```bash
# Download AS3 RPM to BIG-IP (check GitHub for latest version)
curl -LO https://github.com/F5Networks/f5-appsvcs-extension/releases/download/v3.54.0/f5-appsvcs-3.54.0-6.noarch.rpm

# Install it via the REST API
curl -sk -u admin:YOUR_PASSWORD \
  https://10.1.1.4/mgmt/shared/iapp/package-management-tasks \
  -X POST -d '{"operation":"INSTALL","packageFilePath":"/var/config/rest/downloads/f5-appsvcs-3.54.0-6.noarch.rpm"}'
```

**Verify AS3 is running:**

```bash
curl -sk -u admin:YOUR_PASSWORD https://10.1.1.4/mgmt/shared/appsvcs/info | python3 -m json.tool

# You should see version info in the JSON response
```

### 2c. Create a Partition for CIS

CIS manages objects in a dedicated partition to avoid conflicts with existing BIG-IP config.

1. **System → Users → Partition List → Create**
2. Name: `kubernetes`
3. Click **Finished**

### 2d. Configure VXLAN Tunnel (for Pod-to-BIG-IP Routing)

CIS with `--pool-member-type=cluster` needs BIG-IP to route directly to pod IPs. This requires a VXLAN tunnel.

**On BIG-IP CLI (SSH or GUI Shell):**

```bash
# Create VXLAN profile
tmsh create net tunnels vxlan fl-vxlan port 8472 flooding-type none

# Create VXLAN tunnel
tmsh create net tunnels tunnel flannel_vxlan key 1 profile fl-vxlan \
  local-address 10.1.20.10  # ← BIG-IP internal self-IP (NOT the mgmt IP)

# Create a self-IP on the tunnel (must be in the Flannel pod CIDR)
tmsh create net self flannel-self address 10.244.255.254/16 \
  allow-service all vlan flannel_vxlan

# Save the config
tmsh save sys config
```

> **Why?** Flannel uses VXLAN to encapsulate pod traffic. BIG-IP needs a tunnel endpoint in the same VXLAN so it can send traffic directly to pod IPs (not just node IPs). The self-IP `10.244.255.254` is an unused address in the pod CIDR range.

### 2e. Ensure K8s Node Is on BIG-IP Internal Network

The VXLAN tunnel requires the K8s node and BIG-IP's tunnel `local-address` to be on the **same L2 network**. BIG-IP's tunnel runs on the data plane (TMM), not the management interface — so the K8s node must have an interface on the same subnet as BIG-IP's internal self-IP.

```bash
# Check if the K8s node can reach BIG-IP's internal self-IP
ping -c 3 10.1.20.10

# If ping fails, the K8s node needs an IP on the 10.1.20.0/24 network.
# If the node has a second NIC (e.g., ens6), assign it an IP:

tee /etc/netplan/60-internal.yaml > /dev/null << 'NETEOF'
network:
  version: 2
  ethernets:
    ens6:
      addresses:
        - 10.1.20.20/24
NETEOF

chmod 600 /etc/netplan/60-internal.yaml
netplan apply

# Verify
ping -c 3 10.1.20.10
```

> **Cloud VMs:** Do NOT use `match: macaddress` in the netplan config. Cloud providers (AWS, Azure, etc.) may reassign MAC addresses on reboot, which will break the interface matching and leave the NIC without an IP. Use the interface name (`ens6`) instead.

> **Important:** If your K8s node has multiple NICs, Flannel must use the interface on the BIG-IP internal network for VXLAN. Update the Flannel DaemonSet to specify the interface:

```bash
kubectl edit daemonset -n kube-flannel kube-flannel-ds

# Find the kube-flannel container args and add --iface=ens6:
#   args:
#     - --ip-masq
#     - --kube-subnet-mgr
#     - --iface=ens6
```

After saving, Flannel will restart and begin using the internal network for VXLAN encapsulation.

### 2f. Create BIG-IP Node in Kubernetes

Create a K8s node entry for BIG-IP so Flannel knows about the VXLAN tunnel endpoint.

**First, get the BIG-IP VTEP MAC address:**

```bash
# On BIG-IP CLI:
tmsh show net tunnels tunnel flannel_vxlan all-properties | grep -i mac

# Example output:
# MAC Address    02:1c:aa:c6:9a:3c
```

> **Critical:** The MAC address in the K8s node annotation below **must exactly match** the MAC shown by BIG-IP. If they don't match, the VXLAN tunnel won't pass traffic and BIG-IP pool members will show as down.

**Then create the node on the K8s server:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Node
metadata:
  name: bigip1
  annotations:
    flannel.alpha.coreos.com/backend-data: '{"VtepMAC":"<BIG-IP-VTEP-MAC>"}'
    flannel.alpha.coreos.com/backend-type: "vxlan"
    flannel.alpha.coreos.com/kube-subnet-manager: "true"
    flannel.alpha.coreos.com/public-ip: "10.1.20.10"
spec:
  podCIDR: "10.244.255.0/24"
EOF
```

> Replace `<BIG-IP-VTEP-MAC>` with the actual MAC address from the step above. Replace `10.1.20.10` with your BIG-IP internal self-IP.

### 2g. Verify VXLAN Tunnel Connectivity

```bash
# From the K8s node, verify you can reach BIG-IP management API
curl -sk -u admin:YOUR_PASSWORD https://10.1.1.4/mgmt/tm/sys/version | python3 -m json.tool

# Verify BIG-IP can ping a pod IP (run on BIG-IP CLI)
# First get a pod IP: kubectl get pods -o wide
# Then on BIG-IP:
#   tmsh run util ping -c 3 <POD_IP>
#
# If ping fails, check:
#   1. K8s node is on the same subnet as BIG-IP tunnel local-address
#   2. Flannel is using the correct interface (--iface flag)
#   3. VTEP MAC in K8s node annotation matches BIG-IP tunnel MAC
#   4. BIG-IP FDB has entry: tmsh show net fdb tunnel flannel_vxlan
```

### 2f. Create CIS Shared Resources

These are the same for both modes:

```bash
# Clone this repo on the K8s node (if not already done)
git clone https://github.com/therealnoof/cis-nginx-lab.git
cd cis-nginx-lab

# Create the BIG-IP login secret
kubectl create secret generic bigip-login \
  --namespace kube-system \
  --from-literal=username=admin \
  --from-literal=password=YOUR_BIGIP_PASSWORD

# Apply CIS RBAC (ServiceAccount, ClusterRole, ClusterRoleBinding)
kubectl apply -f manifests/cis/cis-rbac.yaml
```

---

## Next: Choose Your Mode

| | Mode A | Mode B |
|-|--------|--------|
| **Guide** | [CIS Standalone](DEPLOYMENT_GUIDE_MODE_A.md) | [CIS + IngressLink](DEPLOYMENT_GUIDE_MODE_B.md) |
| **Traffic path** | BIG-IP → Pods | BIG-IP → NGINX IC → Pods |
| **Best for** | Simple apps, quick demo | Microservices, two-persona demo |
| **Extra components** | None | NGINX Plus IC |
| **Time to build** | ~15 min | ~30 min |

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| CIS CrashLoopBackOff | Bad BIG-IP credentials, unreachable BIG-IP, or missing AS3 | Check `kubectl logs -n kube-system -l app=k8s-bigip-ctlr`. Verify secret, URL, and AS3 install. |
| `AS3 RPM is not installed on BIGIP` | AS3 extension not installed | Install AS3 RPM (see Step 2b) |
| NGINX IC ImagePullBackOff | Bad JWT token or wrong registry secret | Re-create `nginx-registry-secret` with correct JWT |
| NGINX IC CrashLoop: `license-token not found` | Missing license secret | Create `license-token` secret with `--type=nginx.com/license` |
| NGINX IC CrashLoop: `must be of the type nginx.com/license` | Secret type is wrong | Delete and recreate with `--type=nginx.com/license` |
| VS exists but pool is empty | VXLAN tunnel not set up | Verify tunnel: `tmsh show net tunnels tunnel flannel_vxlan` |
| curl to VIP times out | BIG-IP VIP address not routable | Verify VIP is on a reachable network |
| `kubectl` commands fail | KUBECONFIG not set | `export KUBECONFIG=/etc/kubernetes/admin.conf` |
