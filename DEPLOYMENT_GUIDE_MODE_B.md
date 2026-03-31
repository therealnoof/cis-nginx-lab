# Mode B: CIS + IngressLink — Deployment Guide

## BIG-IP → NGINX Plus IC → Pods (Two-Persona Model)

> **Pre-requisite:** Complete [Steps 1-2 in the main Deployment Guide](DEPLOYMENT_GUIDE.md) first (K8s cluster + BIG-IP prep).
>
> **Traffic flow:** Client → BIG-IP VIP → NGINX IC Pods → App Pods
>
> BIG-IP handles: L4 load balancing, WAF, SSL offload, GSLB
> NGINX IC handles: L7 routing, host/path rules, canary, rate limiting

---

## Table of Contents

1. [Create Proxy Protocol iRule](#step-1-create-proxy-protocol-irule)
2. [Install NGINX Plus IC](#step-2-install-nginx-plus-ingress-controller)
3. [Deploy CIS](#step-3-deploy-cis)
4. [Create IngressLink](#step-4-create-ingresslink)
5. [Deploy Sample Applications](#step-5-deploy-sample-applications)
6. [Configure WAF](#step-6-configure-waf)
7. [Verify End-to-End](#step-7-verify-end-to-end)
8. [Teardown](#teardown)

---

## Step 1: Create Proxy Protocol iRule

IngressLink uses Proxy Protocol to pass the real client IP from BIG-IP to NGINX IC. Without this, NGINX sees all requests coming from BIG-IP's tunnel IP.

**On BIG-IP GUI:** Local Traffic → iRules → iRule List → Create

Name: `proxy_protocol_irule`

```tcl
when CLIENT_ACCEPTED {
    set proxyheader "PROXY TCP[IP::version] [IP::remote_addr] [IP::local_addr] [TCP::remote_port] [TCP::local_port]\r\n"
}
when SERVER_CONNECTED {
    TCP::respond $proxyheader
}
```

Click **Finished**.

> **Why?** BIG-IP acts as a reverse proxy at L4. The iRule injects a PROXY protocol header with the original client IP. NGINX IC reads this header (configured in Step 4a) so your app pods see the real client IP in `X-Real-IP` and `X-Forwarded-For` headers.

---

## Step 2: Install NGINX Plus Ingress Controller

NGINX Plus IC requires a license. You'll create Kubernetes secrets with your JWT token, then install via Helm.

### 2a. Create the Registry Secret

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

> **Important:** The `--docker-username` is your JWT token content (the entire token string). The `--docker-password` can be any non-empty string (NGINX registry ignores it).

### 2b. Create the License Secret

NGINX Plus IC v5.x+ requires the JWT license mounted as a Kubernetes secret (in addition to the registry pull secret above).

```bash
# Create the license secret
# IMPORTANT: --type=nginx.com/license is required — a generic secret will be rejected
kubectl create secret generic license-token \
  --namespace nginx-ingress \
  --type=nginx.com/license \
  --from-file=license.jwt=/path/to/your/nginx-repo.jwt
```

> **Note:** This is a different secret than the registry secret in Step 2a. The registry secret lets Kubernetes *pull* the image. The license secret lets NGINX Plus *run* with a valid license. You need both. The secret type **must** be `nginx.com/license` or the IC will reject it.

### 2c. Install via Helm

```bash
helm install nginx-ingress oci://ghcr.io/nginx/charts/nginx-ingress \
  --namespace nginx-ingress \
  -f manifests/nginx-plus-ic/nginx-plus-ic-values.yaml \
  --wait
```

### 2d. Verify NGINX Plus IC

```bash
# Check the pod is running
kubectl get pods -n nginx-ingress

# Expected:
# NAME                             READY   STATUS    RESTARTS   AGE
# nginx-ingress-controller-xxxxx   1/1     Running   0          30s

# Check the ClusterIP service
kubectl get svc -n nginx-ingress
```

---

## Step 3: Deploy CIS

### 3a. Install F5 CIS CRDs

CIS needs several Custom Resource Definitions (VirtualServer, IngressLink, Policy, ExternalDNS, etc.). These are included in the repo so they survive rebuilds.

```bash
# Install all F5 CIS CRDs
kubectl apply -f manifests/cis/f5-cis-crds.yaml

# Verify
kubectl get crd | grep cis.f5.com
```

### 3b. Edit CIS Deployment for Your Environment

```bash
vi manifests/cis-ingresslink/cis-deployment-ingresslink.yaml
```

Update these values:
- `--bigip-url` → your BIG-IP management IP (e.g., `https://10.1.1.4`)
- `--bigip-partition` → partition name (should be `kubernetes`)
- `--flannel-name` → VXLAN tunnel name (should be `/Common/flannel_vxlan`)

### 3c. Apply CIS Deployment

```bash
kubectl apply -f manifests/cis-ingresslink/cis-deployment-ingresslink.yaml
```

### 3d. Verify CIS Is Running

```bash
kubectl get pods -n kube-system -l app=k8s-bigip-ctlr
kubectl logs -n kube-system -l app=k8s-bigip-ctlr --tail=20

# Look for successful BIG-IP connection in the logs
```

---

## Step 4: Create IngressLink

IngressLink creates two BIG-IP virtual servers (port 80 + 443) with pools pointing to NGINX IC pod IPs.

### 4a. Apply Proxy Protocol Config for NGINX

```bash
# Tell NGINX IC to read Proxy Protocol headers from BIG-IP
kubectl apply -f manifests/cis-ingresslink/nginx-config-proxy-protocol.yaml
```

### 4b. Create the NGINX IC Service for IngressLink

```bash
# Create the ClusterIP service that IngressLink discovers via labels
kubectl apply -f manifests/cis-ingresslink/nginx-ingress-service.yaml
```

### 4c. Edit and Apply the IngressLink Resource

```bash
# Update the VIP address to match your environment
vi manifests/cis-ingresslink/ingresslink.yaml

# Apply — CIS will create the BIG-IP virtual servers
kubectl apply -f manifests/cis-ingresslink/ingresslink.yaml
```

### 4d. Verify on BIG-IP

```bash
# Check IngressLink status in K8s
kubectl get il -n nginx-ingress

# Expected:
# NAME              VIP            AGE
# vs-ingresslink    10.1.20.10    10s
```

**On BIG-IP GUI:**
1. **Local Traffic → Virtual Servers** (switch to `kubernetes` partition)
2. You should see two VS entries:
   - `ingress_link_crd_10.1.20.10_80`
   - `ingress_link_crd_10.1.20.10_443`
3. Click into either → **Resources** → check the Pool
4. Pool members should be the NGINX IC pod IPs
5. Check that the `proxy_protocol_irule` is attached under **Resources → iRules**

---

## Step 5: Deploy Sample Applications

### 5a. Deploy Coffee App (App 1)

```bash
kubectl apply -f manifests/apps/app1-coffee.yaml

# Verify pods and ingress
kubectl get pods -l app=coffee
kubectl get svc coffee-svc
kubectl get ingress coffee-ingress

# Test through the BIG-IP VIP
curl -s -H "Host: coffee.example.com" http://10.1.20.10/

# Expected: Response from one of the coffee pods
```

### 5b. Deploy Tea App (App 2) — No BIG-IP Ticket Needed

This is the "aha moment" — deploy a second service and it's instantly reachable through the same VIP.

```bash
kubectl apply -f manifests/apps/app2-tea.yaml

# Verify
kubectl get pods -l app=tea
kubectl get ingress tea-ingress

# Test immediately — should work through the same VIP!
curl -s -H "Host: tea.example.com" http://10.1.20.10/
```

> **Key point:** No BIG-IP changes were needed. DevOps deployed a new service and it was instantly reachable. NGINX IC routes by hostname (coffee.example.com → coffee pods, tea.example.com → tea pods). BIG-IP VIP and pool stayed exactly the same.

---

## Step 6: Configure WAF

### 6a. Create a WAF Policy on BIG-IP (NetOps)

1. **Security → Application Security → Security Policies → Create**
2. Policy name: `linux-high` (or use the built-in template)
3. Policy template: **Rapid Deployment Policy** (good for demos)
4. Application Language: **Unicode (utf-8)**
5. Enforcement Mode: **Blocking** (or Transparent for initial demo)
6. Click **Create Policy**

> **Tip for demos:** Use "Transparent" mode first to show logging without blocking, then switch to "Blocking" to show enforcement.

### 6b. Apply WAF Policy via CRD (DevOps)

```bash
# Apply the WAF policy CRD — CIS attaches it to the BIG-IP VS
kubectl apply -f manifests/waf/waf-policy.yaml
```

### 6c. Verify WAF

**On BIG-IP GUI:**
1. **Local Traffic → Virtual Servers** → click VS → **Security → Policies**
2. Verify the ASM policy is attached

**Test:**
```bash
# Normal request — should work fine
curl -s -H "Host: coffee.example.com" http://10.1.20.10/

# Attack request — WAF should block this
curl -s -H "Host: coffee.example.com" \
  "http://10.1.20.10/?param=<script>alert('xss')</script>"
```

Check **Security → Event Logs → Application → Requests** on BIG-IP to see the blocked attempt.

---

## Step 7: Verify End-to-End

```bash
echo "=== K8s Node ==="
kubectl get nodes

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

echo "=== Services ==="
kubectl get svc --all-namespaces | grep -E "coffee|tea|nginx"

echo "=== Coffee through VIP ==="
curl -s -H "Host: coffee.example.com" http://10.1.20.10/

echo "=== Tea through VIP ==="
curl -s -H "Host: tea.example.com" http://10.1.20.10/
```

**BIG-IP GUI checks:**
- [ ] Two Virtual Servers exist in `kubernetes` partition (port 80 + 443)
- [ ] Pool has NGINX IC pod IPs as members
- [ ] Pool members show green (healthy)
- [ ] Proxy Protocol iRule is attached
- [ ] WAF policy is attached (if Step 6 was completed)

---

## Teardown

```bash
# Remove apps
kubectl delete -f manifests/apps/

# Remove WAF policy CRD
kubectl delete -f manifests/waf/waf-policy.yaml

# Remove IngressLink
kubectl delete -f manifests/cis-ingresslink/ingresslink.yaml
kubectl delete -f manifests/cis-ingresslink/nginx-ingress-service.yaml
kubectl delete -f manifests/cis-ingresslink/nginx-config-proxy-protocol.yaml

# Remove CIS
kubectl delete -f manifests/cis-ingresslink/cis-deployment-ingresslink.yaml

# Remove shared CIS resources
kubectl delete -f manifests/cis/cis-rbac.yaml
kubectl delete secret bigip-login -n kube-system

# Remove NGINX IC
helm uninstall nginx-ingress -n nginx-ingress
kubectl delete namespace nginx-ingress

# Remove IngressLink CRD (optional)
kubectl delete -f manifests/cis-ingresslink/ingresslink-crd.yaml
```

**On BIG-IP:**
- Verify the `kubernetes` partition is empty (CIS should clean up automatically)
- Optionally delete the `proxy_protocol_irule` if no longer needed

---

## Next Steps

If you haven't tried [Mode A: CIS Standalone](DEPLOYMENT_GUIDE_MODE_A.md), give it a look — it's a simpler architecture that's useful for showing BIG-IP's direct Kubernetes integration before building up to the two-tier model.

> **To switch modes:** Tear down Mode B first (above), then follow Mode A. Running both simultaneously will conflict since they deploy different CIS configurations to the same namespace.
