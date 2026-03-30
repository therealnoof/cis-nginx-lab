# Mode A: CIS Standalone — Deployment Guide

## BIG-IP → Pods Directly (No NGINX IC)

> **Pre-requisite:** Complete [Steps 1-2 in the main Deployment Guide](DEPLOYMENT_GUIDE.md) first (K8s cluster + BIG-IP prep).
>
> **Traffic flow:** Client → BIG-IP VIP → App Pod IPs (via VXLAN tunnel)
>
> BIG-IP handles everything: L4 load balancing, L7 routing, WAF, SSL.

---

## Table of Contents

1. [Deploy CIS](#step-1-deploy-cis)
2. [Deploy the Application](#step-2-deploy-the-application)
3. [Verify on BIG-IP](#step-3-verify-on-big-ip)
4. [Test Traffic](#step-4-test-traffic)
5. [Verify End-to-End](#step-5-verify-end-to-end)
6. [Teardown](#teardown)

---

## Step 1: Deploy CIS

### 1a. Edit CIS Deployment for Your Environment

```bash
vi manifests/cis-standalone/cis-deployment-standalone.yaml
```

Update these values:
- `--bigip-url` → your BIG-IP management IP (e.g., `https://10.1.1.4`)
- `--bigip-partition` → partition name (should be `kubernetes`)
- `--flannel-name` → VXLAN tunnel name (should be `/Common/flannel_vxlan`)

### 1b. Apply CIS Deployment

```bash
kubectl apply -f manifests/cis-standalone/cis-deployment-standalone.yaml
```

### 1c. Verify CIS Is Running

```bash
# Check pod status
kubectl get pods -n kube-system -l app=k8s-bigip-ctlr

# Expected:
# NAME                              READY   STATUS    RESTARTS   AGE
# k8s-bigip-ctlr-xxxxxxxxx-xxxxx   1/1     Running   0          30s

# Check logs for successful BIG-IP connection
kubectl logs -n kube-system -l app=k8s-bigip-ctlr --tail=20

# Look for: "Successfully connected to BIG-IP" or similar
# If you see AS3 errors, verify AS3 is installed (Step 2b in main guide)
```

---

## Step 2: Deploy the Application

You have two options for telling CIS how to configure BIG-IP. Choose one:

### Option 1: AS3 ConfigMap (Recommended)

AS3 is a declarative JSON API — you describe what you want on BIG-IP and CIS makes it happen.

```bash
# Deploy the app (Deployment + Service with AS3 labels)
kubectl apply -f manifests/cis-standalone/app-service-as3.yaml

# Deploy the AS3 declaration (creates the BIG-IP VIP)
kubectl apply -f manifests/cis-standalone/as3-configmap.yaml

# Verify pods are running
kubectl get pods -l app=f5-hello-world
```

**How the AS3 labels work:**

The Service has three special labels that map to the AS3 JSON:
```
cis.f5.com/as3-tenant: AS3        → matches the Tenant name in the JSON
cis.f5.com/as3-app: A1            → matches the Application name
cis.f5.com/as3-pool: web_pool     → matches the Pool name
```

CIS sees these labels and auto-populates `serverAddresses` in the AS3 pool with the Service's pod IPs. You don't need to list pod IPs manually — CIS keeps them in sync as pods scale up/down.

> **Note:** AS3 creates objects in a partition named after the Tenant (e.g., `AS3`), not the `--bigip-partition` value.

---

### Option 2: F5 Ingress Annotations (Alternative)

Instead of AS3 JSON, you can use a standard Kubernetes Ingress with F5-specific annotations.

```bash
# Deploy the app (if not already deployed)
kubectl apply -f manifests/cis-standalone/app-service-as3.yaml

# Deploy the Ingress with F5 annotations
kubectl apply -f manifests/cis-standalone/ingress-f5.yaml

# Verify
kubectl get ingress f5-hello-world-ingress
```

> **Note:** Ingress annotations create objects in the `--bigip-partition` (e.g., `kubernetes`). Don't use both options for the same app — they'll create conflicting VS entries.

---

## Step 3: Verify on BIG-IP

1. Log into BIG-IP GUI
2. **Local Traffic → Virtual Servers**
3. Switch partition to:
   - `AS3` (if you used Option 1)
   - `kubernetes` (if you used Option 2)
4. You should see a VS with VIP `10.1.20.10`
5. Click into it → **Resources** → **Pool**
6. Pool members should be the app pod IPs (not node IPs)

```bash
# Compare pod IPs with BIG-IP pool members
kubectl get pods -l app=f5-hello-world -o wide

# The pod IPs in the "IP" column should match BIG-IP pool members
```

---

## Step 4: Test Traffic

```bash
# Hit the VIP (replace with your VIP address)
curl -s http://10.1.20.10

# Expected: HTML response from the f5-hello-world app

# Run multiple requests to see load balancing across pods
for i in $(seq 1 6); do
  curl -s http://10.1.20.10 | grep -o "Server address:.*"
done

# You should see different pod IPs in the responses
```

---

## Step 5: Verify End-to-End

```bash
echo "=== K8s Node ==="
kubectl get nodes

echo "=== CIS Pod ==="
kubectl get pods -n kube-system -l app=k8s-bigip-ctlr

echo "=== App Pods ==="
kubectl get pods -l app=f5-hello-world -o wide

echo "=== AS3 ConfigMap ==="
kubectl get configmap f5-as3-declaration 2>/dev/null && echo "AS3 mode" || echo "Not using AS3"

echo "=== F5 Ingress ==="
kubectl get ingress f5-hello-world-ingress 2>/dev/null && echo "Ingress mode" || echo "Not using F5 Ingress"

echo "=== Traffic Test ==="
curl -s http://10.1.20.10 | head -5
```

**BIG-IP GUI checks:**
- [ ] Virtual Server exists in correct partition (`AS3` or `kubernetes`)
- [ ] Pool has app pod IPs as members
- [ ] Pool members show green (healthy)

---

## Teardown

```bash
# Remove the AS3 declaration (removes BIG-IP VS and pool)
kubectl delete -f manifests/cis-standalone/as3-configmap.yaml

# Or remove the F5 Ingress (if using Option 2)
kubectl delete -f manifests/cis-standalone/ingress-f5.yaml

# Remove the app
kubectl delete -f manifests/cis-standalone/app-service-as3.yaml

# Remove CIS
kubectl delete -f manifests/cis-standalone/cis-deployment-standalone.yaml

# Remove shared resources
kubectl delete -f manifests/cis/cis-rbac.yaml
kubectl delete secret bigip-login -n kube-system
```

**On BIG-IP:** Verify the `AS3` (or `kubernetes`) partition is empty. CIS should have cleaned up when the ConfigMap/Ingress was deleted.

---

## Next Steps

Once you've seen Mode A working, try [Mode B: CIS + IngressLink](DEPLOYMENT_GUIDE_MODE_B.md) to see the two-persona model with NGINX Plus IC.

> **To switch modes:** Tear down Mode A first (above), then follow Mode B. Running both simultaneously will conflict since they deploy different CIS configurations to the same namespace.
