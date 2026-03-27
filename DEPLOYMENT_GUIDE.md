# F5 CIS + NGINX Plus IC Lab — Deployment Guide

## Full Build: Ubuntu → Kubernetes → CIS → NGINX Plus IC → Sample Apps

> **Audience:** Lab builder — follow these steps end-to-end to build the lab environment from scratch.

---

## Table of Contents

1. [Build the Kubernetes Cluster](#step-1-build-the-kubernetes-cluster)
2. [Prepare BIG-IP](#step-2-prepare-big-ip)
3. [Install NGINX Plus Ingress Controller](#step-3-install-nginx-plus-ingress-controller)
4. [Install F5 CIS](#step-4-install-f5-cis)
5. [Create CIS VirtualServer](#step-5-create-cis-virtualserver)
6. [Deploy Sample Applications](#step-6-deploy-sample-applications)
7. [Configure BIG-IP WAF](#step-7-configure-big-ip-waf)
8. [Verify End-to-End](#step-8-verify-end-to-end)
9. [Troubleshooting](#troubleshooting)

---

## Prerequisites

| Component | Requirement |
|-----------|-------------|
| **K8s VM** | Ubuntu 22.04, 4 CPU, 8 GB RAM, 50 GB disk |
| **BIG-IP VE** | v15.1+ on a separate VM, LTM + ASM provisioned |
| **Network** | K8s node and BIG-IP management interface must be routable to each other |
| **NGINX Plus License** | JWT token from [MyF5](https://my.f5.com) |
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
```

> **Why?** Flannel uses VXLAN to encapsulate pod traffic. BIG-IP needs a tunnel endpoint in the same VXLAN so it can send traffic directly to pod IPs (not just node IPs). The self-IP `10.244.255.254` is an unused address in the pod CIDR range.

**On the Kubernetes node**, create a BIG-IP node entry so Flannel knows about the tunnel:

```bash
# Get the BIG-IP VTEP MAC address
# On BIG-IP: tmsh show net tunnels tunnel flannel_vxlan all-properties
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

---

## Step 3: Install NGINX Plus Ingress Controller

NGINX Plus IC requires a license. You'll create a Kubernetes secret with your JWT token, then install via Helm.

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
# Add/update the NGINX Helm chart
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

CIS is the controller that watches Kubernetes and programs BIG-IP.

### 4a. Create the BIG-IP Login Secret

```bash
# Create the secret with your BIG-IP admin credentials
kubectl create secret generic bigip-login \
  --namespace kube-system \
  --from-literal=username=admin \
  --from-literal=password=YOUR_BIGIP_PASSWORD
```

> **Security note:** We use `kubectl create secret` instead of applying the YAML template so credentials never touch disk. In production, use a secrets manager.

### 4b. Apply CIS RBAC

```bash
# Create the ServiceAccount, ClusterRole, and ClusterRoleBinding
kubectl apply -f manifests/cis/cis-rbac.yaml
```

### 4c. Edit and Deploy CIS

Before applying, edit the CIS deployment to match your environment:

```bash
# Open the file and update these values:
#   --bigip-url         → your BIG-IP management IP
#   --bigip-partition   → partition name (should be "kubernetes")
vi manifests/cis/cis-deployment.yaml

# Apply the deployment
kubectl apply -f manifests/cis/cis-deployment.yaml
```

### 4d. Verify CIS

```bash
# Check the CIS pod is running
kubectl get pods -n kube-system -l app=k8s-bigip-ctlr

# Expected:
# NAME                              READY   STATUS    RESTARTS   AGE
# k8s-bigip-ctlr-xxxxxxxxx-xxxxx   1/1     Running   0          30s

# Check CIS logs for successful BIG-IP connection
kubectl logs -n kube-system -l app=k8s-bigip-ctlr --tail=20

# Look for: "Successfully connected to BIG-IP" or similar
# If you see auth errors, double-check the bigip-login secret
```

---

## Step 5: Create CIS VirtualServer

The VirtualServer CRD tells CIS to create a BIG-IP virtual server with a pool pointing to the NGINX IC service.

### 5a. Edit the VirtualServer Resource

```bash
# Update the VIP address to match your environment
vi manifests/cis/virtualserver.yaml

# Change virtualServerAddress to the IP you want the BIG-IP VIP on
# This IP must be in a network that BIG-IP can serve (typically a VLAN)
```

### 5b. Apply the VirtualServer CRD

```bash
kubectl apply -f manifests/cis/virtualserver.yaml
```

### 5c. Verify CIS Created the BIG-IP Config

```bash
# Check VirtualServer status
kubectl get vs -n nginx-ingress

# Expected:
# NAME               HOST   VIRTUALSERVERADDRESS   ...   AGE
# nginx-ingress-vs          10.1.10.100                  10s

# On BIG-IP GUI, verify:
# 1. Go to Local Traffic → Virtual Servers → Virtual Server List
# 2. Change partition dropdown to "kubernetes"
# 3. You should see a new VS with the VIP address
# 4. Click into it → Resources → check the Pool
# 5. Pool members should be the NGINX IC pod IPs
```

---

## Step 6: Deploy Sample Applications

### 6a. Deploy Coffee App (App 1)

```bash
kubectl apply -f manifests/apps/app1-coffee.yaml

# Verify pods and service
kubectl get pods -l app=coffee
kubectl get svc coffee-svc
kubectl get ingress coffee-ingress
```

### 6b. Test Through the Pipeline

```bash
# Test through the BIG-IP VIP (replace with your VIP address)
curl -s -H "Host: cafe.example.com" http://10.1.10.100/coffee

# Expected: Response from one of the coffee pods showing server name/address
```

### 6c. Deploy Tea App (App 2)

This demonstrates the "no ticket needed" value — deploy a second service and it's immediately reachable.

```bash
kubectl apply -f manifests/apps/app2-tea.yaml

# Verify
kubectl get pods -l app=tea
kubectl get ingress tea-ingress

# Test — should work immediately through the same VIP!
curl -s -H "Host: cafe.example.com" http://10.1.10.100/tea
```

> **Key point for the demo:** No BIG-IP changes were needed. DevOps deployed a new service and it was instantly reachable through the existing VIP. CIS + NGINX IC handled everything.

---

## Step 7: Configure BIG-IP WAF

These steps are done by **NetOps** in the BIG-IP GUI.

### 7a. Create a WAF Policy on BIG-IP

1. **Security → Application Security → Security Policies → Create**
2. Policy name: `linux-high` (or use the built-in template)
3. Policy template: **Rapid Deployment Policy** (good for demos)
4. Application Language: **Unicode (utf-8)**
5. Enforcement Mode: **Blocking** (or Transparent for initial demo)
6. Click **Create Policy**

> **Tip for demos:** Use "Transparent" mode first to show logging without blocking, then switch to "Blocking" to show enforcement.

### 7b. Apply WAF Policy via CRD

DevOps applies the WAF policy reference (they don't configure WAF details):

```bash
# Apply the WAF policy CRD
kubectl apply -f manifests/waf/waf-policy.yaml

# CIS reads this and attaches the WAF policy to the BIG-IP VS
```

### 7c. Verify WAF on BIG-IP

1. **Local Traffic → Virtual Servers** (in `kubernetes` partition)
2. Click on the virtual server → **Security → Policies**
3. Verify the ASM policy is attached
4. Test with a simple attack:

```bash
# This should be blocked (or logged) by WAF
curl -s -H "Host: cafe.example.com" \
  "http://10.1.10.100/coffee?param=<script>alert('xss')</script>"
```

---

## Step 8: Verify End-to-End

Run through this checklist to confirm everything is working:

```bash
# --- Kubernetes checks ---
echo "=== K8s Nodes ==="
kubectl get nodes

echo "=== CIS Pod ==="
kubectl get pods -n kube-system -l app=k8s-bigip-ctlr

echo "=== NGINX IC Pods ==="
kubectl get pods -n nginx-ingress

echo "=== CIS VirtualServer ==="
kubectl get vs -n nginx-ingress

echo "=== App Pods ==="
kubectl get pods -l app=coffee
kubectl get pods -l app=tea

echo "=== Ingresses ==="
kubectl get ingress --all-namespaces

echo "=== Services ==="
kubectl get svc --all-namespaces | grep -E "coffee|tea|nginx"

# --- Traffic checks ---
echo "=== Coffee through VIP ==="
curl -s -H "Host: cafe.example.com" http://10.1.10.100/coffee

echo "=== Tea through VIP ==="
curl -s -H "Host: cafe.example.com" http://10.1.10.100/tea
```

**BIG-IP GUI checks:**
- [ ] Virtual Server exists in `kubernetes` partition
- [ ] Pool has NGINX IC pod IPs as members
- [ ] Pool members show green (healthy)
- [ ] WAF policy is attached (if Step 7 was completed)

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| CIS pod in CrashLoopBackOff | Bad BIG-IP credentials or unreachable BIG-IP | Check `kubectl logs -n kube-system -l app=k8s-bigip-ctlr`. Verify secret and URL. |
| NGINX IC pod ImagePullBackOff | Bad JWT token or wrong registry secret | Verify `kubectl get secret nginx-registry-secret -n nginx-ingress -o yaml`. Re-create with correct JWT. |
| No VS on BIG-IP | CIS not watching the right namespace or VirtualServer CRD not applied | Check CIS args for `--namespace`. Check `kubectl get vs -n nginx-ingress`. |
| VS exists but pool is empty | VXLAN tunnel not set up or service name doesn't match | Verify tunnel (`tmsh show net tunnels tunnel flannel_vxlan`). Check VirtualServer pool service name matches NGINX IC service. |
| curl to VIP times out | BIG-IP VIP address not routable from your client | Verify VIP is on a reachable network. Check BIG-IP route table. |
| WAF not blocking | Policy in Transparent mode or not attached to VS | Check Security → Application Security → Policy status. Switch to Blocking. |
| `kubectl` commands fail | KUBECONFIG not set | Run `export KUBECONFIG=/etc/kubernetes/admin.conf` |

---

## Teardown

To clean up the lab without destroying the K8s cluster:

```bash
# Remove apps
kubectl delete -f manifests/apps/

# Remove WAF policy CRD
kubectl delete -f manifests/waf/waf-policy.yaml

# Remove CIS VirtualServer and CIS itself
kubectl delete -f manifests/cis/virtualserver.yaml
kubectl delete -f manifests/cis/cis-deployment.yaml
kubectl delete -f manifests/cis/cis-rbac.yaml
kubectl delete secret bigip-login -n kube-system

# Remove NGINX IC
helm uninstall nginx-ingress -n nginx-ingress
kubectl delete namespace nginx-ingress
```

**On BIG-IP:** Delete the `kubernetes` partition contents (CIS should have cleaned up when the deployment was deleted, but verify manually).
