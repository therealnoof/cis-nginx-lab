# F5 CIS + NGINX Plus IC Lab — Demo Guide

## SE Walkthrough: Two Deployment Modes

> **Audience:** Solutions Engineer running the demo.
>
> **Pre-requisite:** Complete the [Deployment Guide](DEPLOYMENT_GUIDE.md) for the mode(s) you want to demo.
>
> **Recommended flow:** Start with Mode A (simple, fast) then transition to Mode B (two-persona, high-impact). This tells the story: "CIS can go direct to pods — but watch what happens when we add NGINX IC."

---

## Demo Setup

Have two windows open:
- **Window 1 (left):** BIG-IP GUI — logged in as NetOps
- **Window 2 (right):** Terminal — SSH'd into K8s node as DevOps

---

## Part 1: CIS Standalone (Mode A)

> **Story:** "BIG-IP can load balance directly to Kubernetes pods. No extra components needed — just CIS watching the K8s API."

### Clean Slate Setup (Mode A)

Run these commands before starting the demo to ensure a clean state:

```bash
# Delete any existing apps and configs
kubectl delete -f manifests/cis-standalone/ingress-f5.yaml 2>/dev/null
kubectl delete -f manifests/cis-standalone/as3-configmap.yaml 2>/dev/null
kubectl delete -f manifests/cis-standalone/app-service-as3.yaml 2>/dev/null

# Wait for CIS to clean up BIG-IP
sleep 10

# Verify clean — no app pods should be running
kubectl get pods -l app=f5-hello-world

# Verify CIS is healthy
kubectl get pods -n kube-system -l app=k8s-bigip-ctlr

# Verify BIG-IP — VS list should be empty in the kubernetes/AS3 partition
```

---

### Demo A1: Show the Architecture

**Who:** NetOps (BIG-IP GUI) + DevOps (terminal)

> "Let me show you the simplest integration. BIG-IP can load balance directly to Kubernetes pods. CIS — Container Ingress Services — watches the K8s API and programs BIG-IP automatically."

**Show on terminal:**
```bash
# Show CIS is running — this is the bridge between K8s and BIG-IP
kubectl get pods -n kube-system -l app=k8s-bigip-ctlr

# No apps deployed yet
kubectl get pods
kubectl get svc
```

**Show on BIG-IP:**
1. **Local Traffic → Virtual Servers** — empty (no apps deployed yet)
2. "BIG-IP is ready and waiting. CIS is watching. Let's deploy an app."

---

### Demo A2: Deploy App and Watch BIG-IP Auto-Configure

**Who:** DevOps (terminal)

> "Now I deploy an app. Watch BIG-IP — CIS will automatically create the VIP and pool."

**Option 1 — AS3 ConfigMap:**
```bash
# Deploy the app with AS3 labels
kubectl apply -f manifests/cis-standalone/app-service-as3.yaml

# Deploy the AS3 declaration — this tells CIS to create the BIG-IP VIP
kubectl apply -f manifests/cis-standalone/as3-configmap.yaml

# Show the service and pods
kubectl get svc f5-hello-world-web
kubectl get pods -l app=f5-hello-world -o wide
```

**Option 2 — F5 Ingress Annotations:**
```bash
# Deploy the app
kubectl apply -f manifests/cis-standalone/app-service-as3.yaml

# Deploy the Ingress with F5 annotations — CIS reads these and creates the BIG-IP VIP
kubectl apply -f manifests/cis-standalone/ingress-f5.yaml

# Show the service and pods
kubectl get svc f5-hello-world-web
kubectl get pods -l app=f5-hello-world -o wide
```

**Show on BIG-IP (switch to audience):**
1. **Local Traffic → Virtual Servers** — a VS appeared automatically!
   - AS3 ConfigMap: check the `AS3` partition
   - F5 Ingress: check the `kubernetes` partition
2. Click into the **Pool → Members** — show pod IPs
3. "These are actual pod IPs on the overlay network. BIG-IP joined the Flannel VXLAN, so it routes directly to pods. CIS did all of this — no manual config."

**Test it:**
```bash
curl -s http://10.1.20.10
```

---

### Demo A3: Scale Pods — Watch BIG-IP Update

**Who:** DevOps (terminal)

> "Watch the BIG-IP pool members while I scale the app. CIS updates the pool in real-time."

```bash
# Scale from 2 to 4 replicas
kubectl scale deployment f5-hello-world-web --replicas=4

# Watch pods come up
kubectl get pods -l app=f5-hello-world -w
```

**Show on BIG-IP:** Refresh the pool members — 4 members should appear within seconds.

> "CIS saw the new pods and updated BIG-IP automatically. Scale down and the members disappear. No manual pool edits."

```bash
# Scale back down
kubectl scale deployment f5-hello-world-web --replicas=2
```

---

### Demo A4: Show the Limitation

> "This works great for simple apps. But here's the challenge — every new app needs a new BIG-IP VIP or a new AS3 declaration. DevOps has to know BIG-IP details like partition names and VIP addresses. Let me show you what happens when we add NGINX Ingress Controller to the picture."

**Talking point for transition:**
> "In a microservices world, you might have 50 services. Do you want 50 BIG-IP VIPs? Or one VIP with intelligent L7 routing behind it? That's where Mode B comes in."

---

## Part 2: CIS + IngressLink (Mode B)

> **Story:** "NetOps owns BIG-IP security. DevOps deploys at Kubernetes speed. Neither waits for the other."

> **Note:** If transitioning from Mode A live, you'll need to delete the standalone CIS and deploy the IngressLink CIS. For a smooth demo, have Mode B pre-built on a separate environment.

### Clean Slate Setup (Mode B)

Run these commands before starting the demo to ensure a clean state:

```bash
# Delete any existing apps and WAF policy
kubectl delete -f manifests/apps/ 2>/dev/null
kubectl delete -f manifests/waf/waf-policy.yaml 2>/dev/null

# Wait for CIS to clean up BIG-IP
sleep 10

# Verify clean — no app pods should be running
kubectl get pods

# Verify infrastructure is healthy
kubectl get pods -n kube-system -l app=k8s-bigip-ctlr
kubectl get pods -n nginx-ingress
kubectl get il -n nginx-ingress
```

---

### Demo B1: Orient the Audience

**Who:** NetOps (BIG-IP GUI)

> "Now we have a two-tier architecture. BIG-IP handles the front door — VIP, WAF, SSL. Behind it, NGINX Plus Ingress Controller handles L7 routing inside Kubernetes. CIS connects them with IngressLink."

**Show on BIG-IP:**
1. **Local Traffic → Virtual Servers** (in `kubernetes` partition)
   - Show the two IngressLink VS entries (port 80 + 443)
   - "CIS created these automatically from the IngressLink CRD"
2. Click into the **Pool → Members**
   - "Pool members are NGINX IC pod IPs — not app pod IPs. BIG-IP only talks to NGINX."

**Show on terminal:**
```bash
# Show CIS and NGINX IC are running
kubectl get pods -n kube-system -l app=k8s-bigip-ctlr
kubectl get pods -n nginx-ingress

# Show the IngressLink — this is the bridge between BIG-IP and NGINX IC
kubectl get il -n nginx-ingress

# No app services deployed yet
kubectl get svc
```

---

### Demo B2: Deploy First App (Coffee)

**Who:** DevOps (terminal)

> "Now I'm DevOps. I deploy a standard Kubernetes app. I don't touch BIG-IP. Each app gets its own hostname — coffee.example.com, tea.example.com — so teams deploy independently."

```bash
# Deploy coffee app — Deployment, Service, and Ingress
kubectl apply -f manifests/apps/app1-coffee.yaml

# Watch the service come up
kubectl get svc coffee-svc
kubectl get pods -l app=coffee -w
# (Ctrl+C once running)

# Test through the BIG-IP VIP
curl -s -H "Host: coffee.example.com" http://10.1.20.50/
```

> "Traffic flows: Client → BIG-IP VIP → NGINX IC → Coffee pods. I didn't open a ticket, I didn't log into BIG-IP. Each app gets its own hostname and Ingress — completely independent."

---

### Demo B3: Show BIG-IP Didn't Change

**Who:** NetOps (BIG-IP GUI)

> "Let me show you what happened on BIG-IP — nothing. The VIP is the same, the pool is the same. NGINX IC handles the new route."

1. **Local Traffic → Virtual Servers** — same VS, no changes
2. **Pool → Members** — still NGINX IC pod IPs
3. "BIG-IP doesn't know about coffee or tea. It just knows NGINX IC. That's the abstraction."

---

### Demo B4: Deploy Second Service — No BIG-IP Ticket

**Who:** DevOps (terminal)

> "Here's the big moment. I need to deploy a second microservice. In the old world: ticket to NetOps, wait for approval, change window. Watch what happens now."

```bash
# Deploy tea app — its own Deployment, Service, and Ingress
kubectl apply -f manifests/apps/app2-tea.yaml

# Show the new service
kubectl get svc tea-svc

# Test immediately — no waiting!
curl -s -H "Host: tea.example.com" http://10.1.20.50/
```

> "Live in seconds. Same VIP, same BIG-IP config. Tea has its own hostname — completely independent from coffee. NGINX IC routes by hostname: coffee.example.com → coffee pods, tea.example.com → tea pods. DevOps velocity + NetOps control."

```bash
# Hit both services through the same VIP
curl -s -H "Host: coffee.example.com" http://10.1.20.50/
curl -s -H "Host: tea.example.com" http://10.1.20.50/
```

---

### Demo B5: Apply WAF Policy

**Who:** NetOps (GUI) + DevOps (terminal)

> "NetOps wants to add WAF protection. They create the policy on BIG-IP — that's their expertise. DevOps just references the policy name in a CRD."

**NetOps — show on BIG-IP:**
1. **Security → Application Security → Security Policies**
   - Show the WAF policy
   - "NetOps creates and tunes this. They own the security posture."

**DevOps — apply the reference:**
```bash
# Look at what we're applying — just a policy name reference
cat manifests/waf/waf-policy.yaml

# Apply it
kubectl apply -f manifests/waf/waf-policy.yaml
```

**Test WAF:**
```bash
# Normal request — works fine
curl -s -H "Host: coffee.example.com" http://10.1.20.50/

# XSS attack — WAF blocks it
curl -s -H "Host: coffee.example.com" \
  "http://10.1.20.50/?input=<script>alert(1)</script>"
```

**Show on BIG-IP:** Security → Event Logs → Application → Requests

> "DevOps didn't change their app. NetOps didn't touch Kubernetes. Clean separation of concerns."

---

### Demo B6: Canary Deployment

**Who:** DevOps (terminal)

> "DevOps needs to canary a new version. This is 100% a DevOps operation — BIG-IP doesn't change at all."

```bash
# Deploy coffee v2 alongside v1
kubectl apply -f manifests/apps/canary-coffee-v2.yaml

# Show both versions running
kubectl get pods -l app=coffee
kubectl get pods -l app=coffee-v2

# Traffic split based on pod count (2 v1 pods + 1 v2 pod ≈ 67/33)
for i in $(seq 1 10); do
  curl -s -H "Host: coffee.example.com" http://10.1.20.50/ | grep "Server name"
done
```

**Show on BIG-IP:** VS and pool are unchanged.

> "The canary is invisible to BIG-IP. It happens inside Kubernetes. DevOps moves fast, NetOps' security posture is untouched."

```bash
# Rollback or promote
kubectl scale deployment coffee-v2 --replicas=2
kubectl scale deployment coffee --replicas=0
# Or: kubectl delete -f manifests/apps/canary-coffee-v2.yaml
```

---

### Demo B7: CRD-Driven Automation

**Who:** DevOps (terminal)

> "CIS responds to CRD changes in real-time. Watch the BIG-IP VIP change when I patch the IngressLink."

```bash
# Show current VIP
kubectl get il -n nginx-ingress

# Patch the VIP address
kubectl patch il vs-ingresslink -n nginx-ingress \
  --type='merge' \
  -p '{"spec":{"virtualServerAddress":"10.1.10.101"}}'

# BIG-IP VS updates within seconds
```

**Show on BIG-IP:** Refresh Virtual Servers — address changed.

```bash
# Revert
kubectl patch il vs-ingresslink -n nginx-ingress \
  --type='merge' \
  -p '{"spec":{"virtualServerAddress":"10.1.20.50"}}'
```

---

## Closing Talking Points

> **Mode A vs Mode B:**
> "CIS standalone is simple and powerful — BIG-IP talks directly to pods. But in a microservices world with 10+ services and separate NetOps/DevOps teams, IngressLink shines. One BIG-IP VIP, infinite services behind NGINX IC."
>
> **Two-Persona Model:**
> "NetOps keeps full control of BIG-IP — VIPs, WAF, TLS, GSLB. DevOps deploys at Kubernetes speed. CIS is the bridge."
>
> **Infrastructure as Code:**
> "Everything is declarative YAML — Git-friendly, CI/CD-ready, auditable. This works with BIG-IP, not just cloud-native tools."

---

## Troubleshooting During Demo

| Issue | Quick Fix |
|-------|-----------|
| curl hangs | Check VIP address. `curl -v` to see where it stalls. |
| No response from app | `kubectl get pods` — ensure pods are Running. |
| BIG-IP pool empty | `kubectl logs -n kube-system -l app=k8s-bigip-ctlr --tail=50` |
| WAF not blocking | Check BIG-IP policy enforcement mode (Transparent vs Blocking) |
| NGINX shows BIG-IP IP, not client IP | Verify Proxy Protocol iRule and `nginx-config-proxy-protocol.yaml` |
| AS3 partition not appearing | Check ConfigMap labels: `f5type: virtual-server`, `as3: "true"` |
