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

```bash
# Quick health check
kubectl get pods -n kube-system -l app=k8s-bigip-ctlr
kubectl get pods -n nginx-ingress  # Mode B only
```

---

## Part 1: CIS Standalone (Mode A)

> **Story:** "BIG-IP can load balance directly to Kubernetes pods. No extra components needed — just CIS watching the K8s API."

---

### Demo A1: Show the Architecture

**Who:** NetOps (BIG-IP GUI)

> "Let me show you the simplest integration. BIG-IP has a VIP, and CIS automatically populated the pool with pod IPs from Kubernetes. No NGINX, no extra layers — BIG-IP talks directly to pods via a VXLAN tunnel."

**Show on BIG-IP:**
1. **Local Traffic → Virtual Servers** — show the VS in the `AS3` partition
2. Click into the **Pool → Members** — show pod IPs
3. "These are actual pod IPs on the overlay network. BIG-IP joined the Flannel VXLAN, so it routes directly to pods."

**Show on terminal:**
```bash
# Show what CIS is watching
kubectl get pods -n kube-system -l app=k8s-bigip-ctlr

# Show the AS3 ConfigMap that defined the VIP
kubectl get configmap f5-as3-declaration -o yaml | head -30

# Show the app pods
kubectl get pods -l app=f5-hello-world -o wide

# The pod IPs should match what BIG-IP shows in the pool
```

---

### Demo A2: Scale Pods — Watch BIG-IP Update

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

### Demo A3: Show the Limitation

> "This works great for simple apps. But here's the challenge — every new app needs a new BIG-IP VIP or a new AS3 declaration. Let me show you what happens when we add NGINX Ingress Controller to the picture."

**Talking point for transition:**
> "In a microservices world, you might have 50 services. Do you want 50 BIG-IP VIPs? Or one VIP with intelligent L7 routing behind it? That's where Mode B comes in."

---

## Part 2: CIS + IngressLink (Mode B)

> **Story:** "NetOps owns BIG-IP security. DevOps deploys at Kubernetes speed. Neither waits for the other."

> **Note:** If transitioning from Mode A live, you'll need to delete the standalone CIS and deploy the IngressLink CIS. For a smooth demo, have Mode B pre-built.

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
# Show all the pieces
kubectl get pods --all-namespaces | grep -E "bigip|nginx"

# Show the IngressLink resource
kubectl get il -n nginx-ingress -o yaml

# Show NGINX IC is the only thing BIG-IP knows about
kubectl get pods -n nginx-ingress -o wide
```

---

### Demo B2: Deploy First App (Coffee)

**Who:** DevOps (terminal)

> "Now I'm DevOps. I deploy a standard Kubernetes app with a standard Ingress. I don't touch BIG-IP."

> "One thing to note about NGINX Plus IC — when multiple services share the same hostname, we use a **mergeable Ingress** pattern. There's a 'master' Ingress that defines the host, and each service gets a 'minion' Ingress that defines its path. This lets teams independently deploy services under the same domain without stepping on each other."

```bash
# Show the Ingress annotations before deploying
cat manifests/apps/app1-coffee.yaml | grep -A2 "annotations"

# You'll see two Ingress resources:
#   cafe-master-ingress — type: "master", defines host cafe.example.com (no paths)
#   coffee-ingress      — type: "minion", defines path /coffee

# Deploy coffee app (creates master + coffee minion)
kubectl apply -f manifests/apps/app1-coffee.yaml

# Watch pods come up
kubectl get pods -l app=coffee -w
# (Ctrl+C once running)

# Show the Ingress resources
kubectl get ingress

# Test through the BIG-IP VIP
curl -s -H "Host: cafe.example.com" http://10.1.20.10/coffee
```

> "Traffic flows: Client → BIG-IP VIP → NGINX IC → Coffee pods. I didn't open a ticket, I didn't log into BIG-IP. The mergeable Ingress pattern means other teams can add their services under cafe.example.com without modifying my Ingress."

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
# Show tea's Ingress annotation — it's a "minion" joining the same host
cat manifests/apps/app2-tea.yaml | grep -A2 "annotations"

# Deploy tea app (adds a minion Ingress for /tea)
kubectl apply -f manifests/apps/app2-tea.yaml

# Test immediately — no waiting!
curl -s -H "Host: cafe.example.com" http://10.1.20.10/tea
```

> "Live in seconds. Same VIP, same BIG-IP config. Tea joined as a 'minion' Ingress under the same cafe.example.com host. NGINX IC routes /coffee to coffee pods, /tea to tea pods. DevOps velocity + NetOps control."

```bash
# Hit both services
curl -s -H "Host: cafe.example.com" http://10.1.20.10/coffee
curl -s -H "Host: cafe.example.com" http://10.1.20.10/tea
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
curl -s -H "Host: cafe.example.com" http://10.1.20.10/coffee

# XSS attack — WAF blocks it
curl -s -H "Host: cafe.example.com" \
  "http://10.1.20.10/coffee?input=<script>alert(1)</script>"
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

# Show both versions
kubectl get pods -l app=coffee
kubectl get pods -l app=coffee-v2

# Traffic split based on pod count (2 v1 pods + 1 v2 pod ≈ 67/33)
for i in $(seq 1 10); do
  curl -s -H "Host: cafe.example.com" http://10.1.20.10/coffee | grep "Server name"
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
  -p '{"spec":{"virtualServerAddress":"10.1.20.10"}}'
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

## Reset Between Demos

### Reset Mode A
```bash
kubectl delete -f manifests/cis-standalone/as3-configmap.yaml
kubectl delete -f manifests/cis-standalone/app-service-as3.yaml
sleep 5
kubectl apply -f manifests/cis-standalone/app-service-as3.yaml
kubectl apply -f manifests/cis-standalone/as3-configmap.yaml
```

### Reset Mode B
```bash
kubectl delete -f manifests/apps/
kubectl delete -f manifests/waf/waf-policy.yaml
sleep 10
# Re-deploy starting from Demo B2
```

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
