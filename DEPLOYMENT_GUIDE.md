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
3. [(Optional) GSLB Setup for ExternalDNS Demos](#step-3-optional-gslb-setup-for-externaldns-demos)
4. [Troubleshooting](#troubleshooting)

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

> **Required for both modes.** Mode A needs VXLAN to reach app pod IPs directly. Mode B needs VXLAN to reach NGINX IC pod IPs. If you want to skip VXLAN setup, you can use `--pool-member-type=nodeport` instead, but this requires changing services to NodePort type and is not covered in this guide.

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

# Step 1: Assign the IP immediately
ip addr add 10.1.20.20/24 dev ens6

# Step 2: Make it persistent with a systemd service
# (We use systemd instead of netplan because cloud-init can interfere
# with netplan configs and cloud providers may change MAC addresses
# on reboot, causing netplan interface matching to fail.)
tee /etc/systemd/system/ens6-ip.service > /dev/null << 'SVCEOF'
[Unit]
Description=Assign static IP to ens6
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/sbin/ip addr add 10.1.20.20/24 dev ens6
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
SVCEOF

systemctl daemon-reload
systemctl enable ens6-ip.service

# Verify
ping -c 3 10.1.20.10
```

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
>
> **After any BIG-IP reboot:** The VTEP MAC address can change on reboot (especially on cloud-hosted BIG-IP VE). After rebooting BIG-IP, always re-run the `tmsh show` command above and update the K8s node annotation if the MAC has changed. Symptoms of a stale MAC: BIG-IP pool members show down, BIG-IP can't ping pod IPs, no VXLAN packets on `tcpdump -i ens6 udp port 8472`.

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

## Step 3 (Optional): GSLB Setup for ExternalDNS Demos

> **When to follow this:** You've completed Mode A and/or Mode B and want to run Part 3 (GSLB with ExternalDNS) from the Demo Guide. Skip if you're not demoing GSLB.
>
> **What this covers:** BIG-IP DNS (GTM) side configuration — which is clicks in the BIG-IP GUI — plus a three-line uncomment in the CIS deployment manifest to point CIS at the GTM.

### 3a. Provision BIG-IP DNS (GTM) Module

On the BIG-IP you want to act as the GTM:

- **System → Resource Provisioning** → set **DNS** to **Nominal** → **Submit** → the BIG-IP reboots.

> **Lab shortcut:** GTM can run on the same BIG-IP as LTM. The modules coexist. For a two-site demo, pick one BIG-IP (Mode A's or Mode B's) to also act as the GTM — the other BIG-IP only needs LTM.

### 3b. Create Data Centers

A Data Center is a logical container representing a physical site. Create one per LTM site you want in GSLB.

On the GTM BIG-IP:

- **DNS → GSLB → Data Centers → Create**
  - Name: `SiteA_DC`
  - Location / description: optional
- Repeat for `SiteB_DC`.

### 3c. Create GTM Servers

A GTM Server represents a BIG-IP LTM endpoint that GTM will health-check and use as a pool member.

- **DNS → GSLB → Servers → Create**
  - Name: `SiteA_Server` — **must match `dataServerName` in `manifests/gslb/externaldns-coffee.yaml`**
  - Product: `BIG-IP System`
  - Data Center: `SiteA_DC`
  - Devices → Add: the IP address of the Mode A LTM (any self-IP that GTM can reach)
  - Health Monitors: `bigip` (standard)
  - **Configuration dropdown → Advanced → Virtual Server Discovery: `Enabled`** — without this, GTM's VS list stays empty and CIS fails when it tries to put the LTM VS into the GSLB pool. The error looks like `Virtual Server Resource not Available in BIG-IP` in CIS logs.
- Repeat for `SiteB_Server` → `SiteB_DC` → Mode B LTM IP, with Virtual Server Discovery also enabled.

> If you use names other than `SiteA_Server` / `SiteB_Server`, edit the `dataServerName` fields in the `manifests/gslb/externaldns-*.yaml` files to match — the names must line up exactly, with the `/Common/` partition prefix.

### 3d. Configure a DNS Listener

Clients hit the DNS Listener to resolve names. Without one, BIG-IP DNS answers nothing.

- **DNS → Delivery → Listeners → Create**
  - Name: `gslb_listener`
  - Destination: a free IP on your client-facing subnet (e.g., `10.1.20.53`)
  - Port: `53`
  - Service Profile: `dns` (default)
- Create a second listener for TCP if the GUI splits UDP and TCP (some versions do).

### 3e. Enable GTM Args in the CIS Deployment

CIS needs to know where the GTM BIG-IP lives. Edit your mode's CIS deployment manifest and uncomment the three `--gtm-bigip-*` lines:

- **Mode A:** `manifests/cis-standalone/cis-deployment-standalone.yaml` (around line 74)
- **Mode B:** `manifests/cis-ingresslink/cis-deployment-ingresslink.yaml` (around line 70)

Before:
```yaml
# - "--gtm-bigip-url=https://10.1.1.6"
# - "--gtm-bigip-username=$(BIGIP_USERNAME)"
# - "--gtm-bigip-password=$(BIGIP_PASSWORD)"
```

After (replace with your GTM BIG-IP management IP):
```yaml
- "--gtm-bigip-url=https://10.1.1.6"
- "--gtm-bigip-username=$(BIGIP_USERNAME)"
- "--gtm-bigip-password=$(BIGIP_PASSWORD)"
```

The `$(BIGIP_USERNAME)` / `$(BIGIP_PASSWORD)` env vars already resolve from the `bigip-login` secret you created in Step 2f — no extra secret needed when GTM shares the same credentials as LTM. If GTM uses separate credentials, create a second secret (e.g., `bigip-gtm-login`) and wire up additional env vars in the CIS Deployment.

> **Mode A note:** Mode A's CIS deployment also has a comment reminding you that ExternalDNS requires `--custom-resource-mode=true`. Ensure that flag is present (not commented out) before CIS can watch ExternalDNS CRDs.

Apply and restart CIS:
```bash
kubectl apply -f manifests/cis-ingresslink/cis-deployment-ingresslink.yaml
# (or the Mode A path)

kubectl rollout restart deployment/k8s-bigip-ctlr -n kube-system
kubectl rollout status  deployment/k8s-bigip-ctlr -n kube-system
```

### 3f. Verify CIS Reached GTM

```bash
kubectl logs -n kube-system -l app=k8s-bigip-ctlr --tail=100 | grep -iE 'gtm|wideip|externaldns'
```

You should see CIS log successful connection to GTM. Red flags: `401 Unauthorized`, `connection refused`, or repeated `Post https://<gtm-ip>/mgmt/...` errors — those point at credentials, network reach, or the GTM URL being wrong.

### 3g. Deploy a Test ExternalDNS CRD

**Mode A:**
```bash
kubectl apply -f manifests/apps/app1-coffee.yaml       # needs a VS with host=coffee.example.com
kubectl apply -f manifests/gslb/externaldns-coffee.yaml
kubectl get externaldns
```
In Mode A, CIS reads the standard `coffee-ingress` and creates a per-app LTM VS with `coffee.example.com` as its hostname. ExternalDNS binds to that VS and pushes the Wide IP.

**Mode B:** CIS in Mode B runs with `--custom-resource-mode=true` and does not watch standard Ingresses — it only sees CRDs. IngressLink alone doesn't expose per-app hostnames to CIS, so `externaldns-coffee.yaml` has nothing to bind to. You need a `VirtualServer` CRD that gives CIS a hostname to anchor on:

```bash
# Coffee app's Deployment + Service (Ingress is present but not used by CIS here)
kubectl apply -f manifests/apps/app1-coffee.yaml

# VirtualServer CRD — CIS creates a per-app LTM VS at its own VIP
kubectl apply -f manifests/gslb/virtualserver-coffee-modeB.yaml

# Single-site ExternalDNS that references SiteB_Server only
kubectl apply -f manifests/gslb/externaldns-coffee-modeB.yaml

kubectl get virtualserver,externaldns
```

On BIG-IP GUI:
- **Local Traffic → Virtual Servers** → `coffee.example.com_kubernetes` (or similar — CIS-generated name) appears at the VIP in the VirtualServer CRD, SEPARATE from the IngressLink VSes.
- **DNS → GSLB → Wide IPs** → `coffee.example.com` appears.
- Click in → **Pools** → the GTM pool contains `SiteB_Server`, which advertises the per-app VS above.

> **Mode B traffic path:** the per-app VS is a second BIG-IP entry point into the same data plane — its pool targets the NGINX IC Service, not the app pods directly. GSLB-resolved clients still traverse NGINX IC (Host-based routing to `coffee-svc`), just via a different VIP than IngressLink. If you re-enable proxy-protocol later (for WAF / client-IP demos), this VS will need a matching iRule or NGINX IC will reset its connections.

Test DNS resolution from any host that can reach the listener IP:
```bash
dig @10.1.20.53 coffee.example.com +short
```
Returns the LTM VIP per the Wide IP's load balance method.

**Done** — you can now run Part 3 (Demos G1-G5) in `DEMO_GUIDE.md`.

> **Troubleshooting GSLB:**
> - **Wide IP doesn't appear:** CIS logs (step 3f command) are the first place to look. In Mode B, the most common cause is missing VirtualServer CRD — without it, CIS has no hostname to bind the ExternalDNS to and silently no-ops. Check `kubectl get virtualserver` in the same namespace.
> - **Wide IP exists but pool members show `down`:** GTM can't reach the LTM VIPs. Check the GTM Server's device IP and the network path from the GTM BIG-IP to each LTM.
> - **`dig` returns `SERVFAIL` or `NXDOMAIN`:** the DNS Listener is misconfigured or not on the expected IP. Verify with `tmsh list ltm virtual | grep dns` on the GTM BIG-IP.

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
