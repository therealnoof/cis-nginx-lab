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
tmsh create net tunnels tunnel fl-tunnel key 1 profile fl-vxlan \
  local-address 10.1.20.10  # ← BIG-IP internal self-IP (NOT the mgmt IP)

# Create a self-IP on the tunnel (must be in the Flannel pod CIDR)
tmsh create net self flannel-self address 10.244.255.254/16 \
  allow-service all vlan fl-tunnel

# Save the config
tmsh save sys config
```

> **Why?** Flannel uses VXLAN to encapsulate pod traffic. BIG-IP needs a tunnel endpoint in the same VXLAN so it can send traffic directly to pod IPs (not just node IPs). The self-IP `10.244.255.254` is an unused address in the pod CIDR range.

**On the Kubernetes node**, create a BIG-IP node entry so Flannel knows about the tunnel:

```bash
# Get the BIG-IP VTEP MAC address
# On BIG-IP: tmsh show net tunnels tunnel fl-tunnel all-properties
# Look for the MAC address field

# Create the BIG-IP node annotation in K8s
cat <<EOF | kubectl apply -f -
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

> Replace `<BIG-IP-VTEP-MAC>` with the actual MAC address and `10.1.20.10` with your BIG-IP internal self-IP.

### 2e. Verify Network Connectivity

```bash
# From the K8s node, verify you can reach BIG-IP management API
# Use the MANAGEMENT IP (10.1.1.4), not the internal self-IP (10.1.20.10)
# BIG-IP REST API requires authentication (-u flag)
curl -sk -u admin:YOUR_PASSWORD https://10.1.1.4/mgmt/tm/sys/version | python3 -m json.tool

# You should see BIG-IP version info in the JSON response
# If you get "Expecting value" error, you're hitting the wrong IP or missing -u creds
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
| VS exists but pool is empty | VXLAN tunnel not set up | Verify tunnel: `tmsh show net tunnels tunnel fl-tunnel` |
| curl to VIP times out | BIG-IP VIP address not routable | Verify VIP is on a reachable network |
| `kubectl` commands fail | KUBECONFIG not set | `export KUBECONFIG=/etc/kubernetes/admin.conf` |
