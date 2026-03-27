# F5 CIS + NGINX Plus IC Lab — Deployment Guide

## Full Build: Two Deployment Modes

> **Audience:** Lab builder — follow these steps to build the environment from scratch.
>
> **Choose your path:**
> - **Mode A (CIS Standalone):** BIG-IP → Pods directly. Steps 1 → 2 → 4A → 6A
> - **Mode B (CIS + IngressLink):** BIG-IP → NGINX IC → Pods. Steps 1 → 2 → 3 → 4B → 5 → 6B → 7

---

## Table of Contents

1. [Build the Kubernetes Cluster](#step-1-build-the-kubernetes-cluster) — Both modes
2. [Prepare BIG-IP](#step-2-prepare-big-ip) — Both modes
3. [Install NGINX Plus IC](#step-3-install-nginx-plus-ingress-controller) — Mode B only
4. [Install F5 CIS](#step-4-install-f5-cis) — Choose 4A or 4B
5. [Create IngressLink](#step-5-create-ingresslink) — Mode B only
6. [Deploy Applications](#step-6-deploy-applications) — Choose 6A or 6B
7. [Configure WAF](#step-7-configure-waf) — Mode B only
8. [Verify End-to-End](#step-8-verify-end-to-end) — Both modes
9. [Troubleshooting](#troubleshooting)

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

> **Required for:** Both modes

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

> **Required for:** Both modes

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

### 2e. Create Proxy Protocol iRule (Mode B only)

If you plan to use IngressLink (Mode B), create this iRule on BIG-IP so NGINX IC receives the real client IP:

**On BIG-IP GUI:** Local Traffic → iRules → iRule List → Create

Name: `Proxy_Protocol_iRule`

```tcl
when CLIENT_ACCEPTED {
    set proxyheader "PROXY TCP[IP::version] [IP::remote_addr] [IP::local_addr] [TCP::remote_port] [TCP::local_port]\r\n"
}
when SERVER_CONNECTED {
    TCP::respond $proxyheader
}
```

> **Why?** Without Proxy Protocol, NGINX IC sees all requests coming from BIG-IP's tunnel IP. The iRule injects the real client IP into the Proxy Protocol header, and NGINX reads it.

### 2f. Verify Network Connectivity

```bash
# From the K8s node, verify you can reach BIG-IP management API
# Use the MANAGEMENT IP (10.1.1.4), not the internal self-IP (10.1.20.10)
# BIG-IP REST API requires authentication (-u flag)
curl -sk -u admin:YOUR_PASSWORD https://10.1.1.4/mgmt/tm/sys/version | python3 -m json.tool

# You should see BIG-IP version info in the JSON response
# If you get "Expecting value" error, you're hitting the wrong IP or missing -u creds
```

---

## Step 3: Install NGINX Plus Ingress Controller

> **Required for:** Mode B (IngressLink) only — skip this step for Mode A

NGINX Plus IC requires a license. You'll create Kubernetes secrets with your JWT token, then install via Helm.

### 3a. Create the Registry Secret

```bash
# Create the namespace first
kubectl create namespace nginx-ingress

# Create a Docker registry secret using your NGINX Plus JWT token
# Get your JWT from https://my.f5.com → NGINX → Subscriptions → JWT
kubectl create secret docker-registry nginx-registry-secret \
  --namespace nginx-ingress \
  --docker-server=private-registry.nginx.com \
  --docker-username=$(cat /path/to/your/nginx-repo.jwt) \
  --docker-password=none
```

> **Important:** The `--docker-username` is your JWT token content (the entire token string). The `--docker-password` can be any non-empty string (NGINX registry ignores it). Replace `/path/to/your/nginx-repo.jwt` with the actual path to your JWT file.

### 3b. Create the License Secret

NGINX Plus IC v5.x+ requires the JWT license mounted as a Kubernetes secret (in addition to the registry pull secret above). Without this, the IC pod will crash with: `license secret: could not find nginx-ingress/license-token`.

```bash
# Create the license secret
# IMPORTANT: --type=nginx.com/license is required — a generic secret will be rejected
kubectl create secret generic license-token \
  --namespace nginx-ingress \
  --type=nginx.com/license \
  --from-file=license.jwt=/path/to/your/nginx-repo.jwt
```

> **Note:** This is a different secret than the registry secret in Step 3a. The registry secret lets Kubernetes *pull* the image. The license secret lets NGINX Plus *run* with a valid license. You need both. The secret type **must** be `nginx.com/license` or the IC will reject it.

### 3c. Install via Helm

```bash
# Install NGINX Plus IC using the Helm values file
helm install nginx-ingress oci://ghcr.io/nginx/charts/nginx-ingress \
  --namespace nginx-ingress \
  -f manifests/nginx-plus-ic/nginx-plus-ic-values.yaml \
  --wait
```

### 3d. Verify NGINX Plus IC

```bash
# Check the pod is running
kubectl get pods -n nginx-ingress

# Expected output:
# NAME                             READY   STATUS    RESTARTS   AGE
# nginx-ingress-controller-xxxxx   1/1     Running   0          30s

# Check the ClusterIP service was created
kubectl get svc -n nginx-ingress

# Expected: TYPE should be ClusterIP (not LoadBalancer or NodePort)
```

---

## Step 4: Install F5 CIS

> **Choose 4A or 4B based on your deployment mode.**

### Shared Setup (Both Modes)

```bash
# Create the BIG-IP login secret
kubectl create secret generic bigip-login \
  --namespace kube-system \
  --from-literal=username=admin \
  --from-literal=password=YOUR_BIGIP_PASSWORD

# Apply CIS RBAC (ServiceAccount, ClusterRole, ClusterRoleBinding)
kubectl apply -f manifests/cis/cis-rbac.yaml
```

---

### Step 4A: Deploy CIS — Standalone Mode

> **Mode A:** BIG-IP VIP → App Pods directly

```bash
# Edit the deployment for your environment:
#   --bigip-url         → your BIG-IP management IP
#   --bigip-partition   → partition name (should be "kubernetes")
#   --flannel-name      → VXLAN tunnel name (should be "/Common/fl-tunnel")
vi manifests/cis-standalone/cis-deployment-standalone.yaml

# Apply the CIS deployment
kubectl apply -f manifests/cis-standalone/cis-deployment-standalone.yaml
```

**Key flags for standalone mode:**
- `--custom-resource-mode=false` — watches Ingress resources and AS3 ConfigMaps
- `--agent=as3` — uses AS3 to push declarations to BIG-IP
- `--pool-member-type=cluster` — routes to pod IPs via VXLAN

**Verify:**
```bash
kubectl get pods -n kube-system -l app=k8s-bigip-ctlr
kubectl logs -n kube-system -l app=k8s-bigip-ctlr --tail=20
```

**→ Continue to [Step 6A](#step-6a-deploy-app--standalone-mode)**

---

### Step 4B: Deploy CIS — IngressLink Mode

> **Mode B:** BIG-IP VIP → NGINX IC → App Pods

```bash
# Install the IngressLink CRD (if not already present)
kubectl get crd ingresslinks.cis.f5.com 2>/dev/null || \
  kubectl apply -f manifests/cis-ingresslink/ingresslink-crd.yaml

# Edit the deployment for your environment
vi manifests/cis-ingresslink/cis-deployment-ingresslink.yaml

# Apply the CIS deployment
kubectl apply -f manifests/cis-ingresslink/cis-deployment-ingresslink.yaml
```

**Key flags for IngressLink mode:**
- `--custom-resource-mode=true` — watches F5 CRDs (IngressLink, VirtualServer, etc.)
- No `--agent=as3` — CIS manages AS3 internally for CRDs
- `--pool-member-type=cluster` — routes to pod IPs via VXLAN

**Verify:**
```bash
kubectl get pods -n kube-system -l app=k8s-bigip-ctlr
kubectl logs -n kube-system -l app=k8s-bigip-ctlr --tail=20
```

**→ Continue to [Step 5](#step-5-create-ingresslink)**

---

## Step 5: Create IngressLink

> **Required for:** Mode B only

IngressLink creates two BIG-IP virtual servers (port 80 + 443) with pools pointing to NGINX IC pod IPs.

### 5a. Apply Proxy Protocol Config

```bash
# Tell NGINX IC to read Proxy Protocol headers from BIG-IP
kubectl apply -f manifests/cis-ingresslink/nginx-config-proxy-protocol.yaml
```

### 5b. Create the NGINX IC Service for IngressLink

```bash
# Create the ClusterIP service that IngressLink discovers via labels
kubectl apply -f manifests/cis-ingresslink/nginx-ingress-service.yaml
```

### 5c. Edit and Apply the IngressLink Resource

```bash
# Update the VIP address to match your environment
vi manifests/cis-ingresslink/ingresslink.yaml

# Apply it — CIS will create the BIG-IP virtual servers
kubectl apply -f manifests/cis-ingresslink/ingresslink.yaml
```

### 5d. Verify on BIG-IP

```bash
# Check IngressLink status
kubectl get il -n nginx-ingress

# Expected:
# NAME              VIP            AGE
# vs-ingresslink    10.1.10.100    10s
```

**On BIG-IP GUI:**
1. **Local Traffic → Virtual Servers** (switch to `kubernetes` partition)
2. You should see two VS entries:
   - `ingress_link_crd_10.1.10.100_80`
   - `ingress_link_crd_10.1.10.100_443`
3. Click into either → **Resources** → check the Pool
4. Pool members should be the NGINX IC pod IPs

**→ Continue to [Step 6B](#step-6b-deploy-apps--ingresslink-mode)**

---

## Step 6: Deploy Applications

### Step 6A: Deploy App — Standalone Mode

> **Mode A:** BIG-IP VIP → App Pods directly

You have two approaches — choose one:

**Option 1: AS3 ConfigMap** (recommended for demos)

```bash
# Deploy the app + service with AS3 labels
kubectl apply -f manifests/cis-standalone/app-service-as3.yaml

# Deploy the AS3 ConfigMap — CIS will create the BIG-IP VIP
kubectl apply -f manifests/cis-standalone/as3-configmap.yaml

# Verify pods
kubectl get pods -l app=f5-hello-world

# Test through the BIG-IP VIP
curl -s http://10.1.10.100
```

> **How it works:** The Service has special labels (`cis.f5.com/as3-tenant`, `cis.f5.com/as3-app`, `cis.f5.com/as3-pool`) that tell CIS to fill the AS3 pool with this Service's pod IPs. CIS sends the AS3 declaration to BIG-IP, which creates the VIP and pool.

**Option 2: F5 Ingress Annotations** (alternative)

```bash
# Deploy the app (if not already deployed from Option 1)
kubectl apply -f manifests/cis-standalone/app-service-as3.yaml

# Deploy the Ingress with F5 annotations
kubectl apply -f manifests/cis-standalone/ingress-f5.yaml

# Test through the VIP
curl -s http://10.1.10.100
```

> **Note:** Don't use both Options 1 and 2 for the same app — they'll conflict. The AS3 partition is named after the Tenant ("AS3"), while the Ingress uses `--bigip-partition` ("kubernetes").

**On BIG-IP GUI:**
1. **Local Traffic → Virtual Servers**
2. Check the `AS3` partition (for Option 1) or `kubernetes` partition (for Option 2)
3. You should see a VS with VIP `10.1.10.100`
4. Pool members = app pod IPs (not node IPs — this is ClusterIP/VXLAN mode)

**→ Skip to [Step 8](#step-8-verify-end-to-end)**

---

### Step 6B: Deploy Apps — IngressLink Mode

> **Mode B:** BIG-IP VIP → NGINX IC → App Pods

```bash
# Deploy coffee app (App 1)
kubectl apply -f manifests/apps/app1-coffee.yaml

# Verify pods and ingress
kubectl get pods -l app=coffee
kubectl get ingress coffee-ingress

# Test through the BIG-IP VIP
curl -s -H "Host: cafe.example.com" http://10.1.10.100/coffee
```

**Deploy second service — no BIG-IP ticket needed:**

```bash
# Deploy tea app (App 2) — live immediately through the same VIP!
kubectl apply -f manifests/apps/app2-tea.yaml

# Test — should work instantly
curl -s -H "Host: cafe.example.com" http://10.1.10.100/tea
```

> **Key point:** No BIG-IP changes were needed for the second app. NGINX IC handles L7 routing. This is the "no ticket" value prop.

**→ Continue to [Step 7](#step-7-configure-waf)**

---

## Step 7: Configure WAF

> **Required for:** Mode B (applies WAF to the IngressLink VIP). For Mode A, you can attach WAF directly on BIG-IP via the GUI.

### 7a. Create a WAF Policy on BIG-IP

1. **Security → Application Security → Security Policies → Create**
2. Policy name: `linux-high` (or use the built-in template)
3. Policy template: **Rapid Deployment Policy** (good for demos)
4. Application Language: **Unicode (utf-8)**
5. Enforcement Mode: **Blocking** (or Transparent for initial demo)
6. Click **Create Policy**

### 7b. Apply WAF Policy via CRD

```bash
# Apply the WAF policy CRD — CIS attaches it to the BIG-IP VS
kubectl apply -f manifests/waf/waf-policy.yaml
```

### 7c. Verify WAF

```bash
# Normal request — should work
curl -s -H "Host: cafe.example.com" http://10.1.10.100/coffee

# Attack request — WAF should block this
curl -s -H "Host: cafe.example.com" \
  "http://10.1.10.100/coffee?param=<script>alert('xss')</script>"
```

---

## Step 8: Verify End-to-End

### Mode A Verification

```bash
echo "=== CIS Pod ==="
kubectl get pods -n kube-system -l app=k8s-bigip-ctlr

echo "=== App Pods ==="
kubectl get pods -l app=f5-hello-world

echo "=== AS3 ConfigMap ==="
kubectl get configmap f5-as3-declaration

echo "=== Traffic Test ==="
curl -s http://10.1.10.100
```

### Mode B Verification

```bash
echo "=== CIS Pod ==="
kubectl get pods -n kube-system -l app=k8s-bigip-ctlr

echo "=== NGINX IC Pods ==="
kubectl get pods -n nginx-ingress

echo "=== IngressLink ==="
kubectl get il -n nginx-ingress

echo "=== App Pods ==="
kubectl get pods -l app=coffee
kubectl get pods -l app=tea

echo "=== Ingresses ==="
kubectl get ingress --all-namespaces

echo "=== Traffic Tests ==="
curl -s -H "Host: cafe.example.com" http://10.1.10.100/coffee
curl -s -H "Host: cafe.example.com" http://10.1.10.100/tea
```

**BIG-IP GUI checks (both modes):**
- [ ] Virtual Server exists in the correct partition
- [ ] Pool has correct member IPs (pod IPs for both modes)
- [ ] Pool members show green (healthy)
- [ ] WAF policy attached (if configured)

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| CIS CrashLoopBackOff | Bad BIG-IP credentials, unreachable BIG-IP, or missing AS3 | Check `kubectl logs -n kube-system -l app=k8s-bigip-ctlr`. Verify secret, URL, and AS3 install. |
| `AS3 RPM is not installed on BIGIP` | AS3 extension not installed | Install AS3 RPM (see Step 2b) |
| NGINX IC ImagePullBackOff | Bad JWT token or wrong registry secret | Re-create `nginx-registry-secret` with correct JWT |
| NGINX IC CrashLoop: `license-token not found` | Missing license secret | Create `license-token` secret with `--type=nginx.com/license` (Step 3b) |
| NGINX IC CrashLoop: `must be of the type nginx.com/license` | Secret type is wrong | Delete and recreate with `--type=nginx.com/license` |
| No VS on BIG-IP (standalone) | AS3 ConfigMap labels wrong or CIS not watching namespace | Check ConfigMap labels: `f5type: virtual-server` and `as3: "true"` |
| No VS on BIG-IP (IngressLink) | IngressLink not applied or CIS not in CRD mode | Check `kubectl get il -n nginx-ingress`. Verify `--custom-resource-mode=true` |
| VS exists but pool is empty | VXLAN tunnel not set up | Verify tunnel: `tmsh show net tunnels tunnel fl-tunnel` |
| curl to VIP times out | BIG-IP VIP address not routable | Verify VIP is on a reachable network |
| NGINX shows BIG-IP IP instead of client IP | Proxy Protocol not configured | Apply `nginx-config-proxy-protocol.yaml` and verify iRule on BIG-IP |
| `kubectl` commands fail | KUBECONFIG not set | `export KUBECONFIG=/etc/kubernetes/admin.conf` |

---

## Teardown

### Mode A Teardown

```bash
kubectl delete -f manifests/cis-standalone/as3-configmap.yaml
kubectl delete -f manifests/cis-standalone/app-service-as3.yaml
kubectl delete -f manifests/cis-standalone/cis-deployment-standalone.yaml
kubectl delete -f manifests/cis/cis-rbac.yaml
kubectl delete secret bigip-login -n kube-system
```

### Mode B Teardown

```bash
kubectl delete -f manifests/apps/
kubectl delete -f manifests/waf/waf-policy.yaml
kubectl delete -f manifests/cis-ingresslink/ingresslink.yaml
kubectl delete -f manifests/cis-ingresslink/nginx-ingress-service.yaml
kubectl delete -f manifests/cis-ingresslink/nginx-config-proxy-protocol.yaml
kubectl delete -f manifests/cis-ingresslink/cis-deployment-ingresslink.yaml
kubectl delete -f manifests/cis/cis-rbac.yaml
kubectl delete secret bigip-login -n kube-system
helm uninstall nginx-ingress -n nginx-ingress
kubectl delete namespace nginx-ingress
kubectl delete -f manifests/cis-ingresslink/ingresslink-crd.yaml
```

**On BIG-IP:** Delete the `kubernetes` and `AS3` partition contents (CIS should clean up when deployments are deleted, but verify manually).
