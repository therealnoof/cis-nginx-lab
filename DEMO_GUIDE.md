# F5 CIS + NGINX Plus IC Lab — Demo Guide

## SE Walkthrough: Two-Persona Model

> **Audience:** Solutions Engineer running the demo. This guide walks through each demo scenario with talking points and commands for both personas.
>
> **Pre-requisite:** Complete the [Deployment Guide](DEPLOYMENT_GUIDE.md) first. All components should be running before you start the demo.

---

## Demo Setup Checklist

Before starting the demo, verify everything is healthy:

```bash
# Quick health check — all pods should be Running
kubectl get pods -n kube-system -l app=k8s-bigip-ctlr
kubectl get pods -n nginx-ingress
kubectl get ingresslink -n nginx-ingress
```

Have two windows open:
- **Window 1 (left):** BIG-IP GUI — logged in as NetOps
- **Window 2 (right):** Terminal — SSH'd into K8s node as DevOps

> **Tip:** If presenting on a single screen, use a browser with BIG-IP on the left and a terminal on the right. The side-by-side layout reinforces the two-persona message.

---

## Demo Flow Overview

| # | Scenario | Who Acts | What Changes |
|---|----------|----------|--------------|
| 1 | Show the VIP and architecture | NetOps (GUI) | Nothing — just orient the audience |
| 2 | Deploy first app (coffee) | DevOps (kubectl) | App pods + Ingress created |
| 3 | Show CIS auto-configured BIG-IP | NetOps (GUI) | Pool members appeared automatically |
| 4 | Deploy second service (tea) — no ticket | DevOps (kubectl) | New route, same VIP |
| 5 | Apply WAF policy | NetOps (GUI) + DevOps (kubectl) | WAF attached to VS |
| 6 | Canary deployment | DevOps (kubectl) | Traffic split, BIG-IP untouched |
| 7 | CRD-driven automation | DevOps (kubectl) | CIS auto-deploys BIG-IP config |

---

## Scenario 1: Orient the Audience

**Who:** NetOps (BIG-IP GUI)

### Talking Points

> "Let me show you the architecture. We have a BIG-IP handling external traffic — VIPs, WAF, the things NetOps cares about. Behind it, there's a Kubernetes cluster where DevOps deploys apps using NGINX Plus as the ingress controller. The glue between them is CIS — Container Ingress Services — which watches Kubernetes and automatically programs BIG-IP."

### Show in BIG-IP GUI

1. **Local Traffic → Virtual Servers** — switch to `kubernetes` partition
   - Show the VIP that was created by IngressLink
   - Click into it → show the pool
   - "CIS created this VS and pool automatically when we deployed the IngressLink resource in Kubernetes"

2. Click into the **Pool** → **Members**
   - Show the NGINX IC pod IPs
   - "These are the NGINX Plus Ingress Controller pods. CIS discovered them automatically — NetOps didn't configure them manually."

### Show in Terminal

```bash
# Show what's running in K8s
kubectl get pods --all-namespaces | grep -E "bigip|nginx"

# Show the IngressLink resource
kubectl get ingresslink -n nginx-ingress -o yaml
```

---

## Scenario 2: Deploy First App (Coffee)

**Who:** DevOps (terminal)

### Talking Points

> "Now I'm DevOps. I have an app to deploy. I don't need to know anything about BIG-IP, WAF policies, or VIPs. I just write standard Kubernetes manifests."

### Commands

```bash
# Deploy coffee app — Deployment, Service, and Ingress
kubectl apply -f manifests/apps/app1-coffee.yaml

# Watch pods come up
kubectl get pods -l app=coffee -w
# (Press Ctrl+C once both pods show Running)

# Show the Ingress
kubectl get ingress coffee-ingress
```

### Test It

```bash
# Hit the app through the BIG-IP VIP
# Replace 10.1.10.100 with your actual VIP
curl -s -H "Host: cafe.example.com" http://10.1.10.100/coffee
```

**Expected:** Response showing the coffee pod's server name and IP.

### Talking Points

> "I deployed a standard Kubernetes app with a standard Ingress resource. Traffic flows: Client → BIG-IP VIP → NGINX IC → Coffee pods. I didn't open a ticket, I didn't touch BIG-IP."

---

## Scenario 3: Show CIS Auto-Configuration on BIG-IP

**Who:** NetOps (BIG-IP GUI)

### Talking Points

> "Let me switch to the NetOps view. Watch what happened on BIG-IP without anyone touching it."

### Show in BIG-IP GUI

1. **Local Traffic → Pools** (in `kubernetes` partition)
   - Show pool members — they should include the NGINX IC pod IPs
   - "CIS automatically updated the pool with the IC pod IPs. If we scale the IC up or down, CIS updates the pool in real-time."

2. **Local Traffic → Virtual Servers** → click into the VS → **Resources**
   - Show the pool assignment
   - "The virtual server automatically routes to the NGINX IC pool"

### Talking Points

> "This is the key value: NetOps still owns the VIP — they can see it, monitor it, attach policies. But the pool membership is automated. No manual updates, no stale configs, no tickets going back and forth."

---

## Scenario 4: Deploy Second Service — No BIG-IP Ticket

**Who:** DevOps (terminal)

### Talking Points

> "Here's where it gets interesting. I need to deploy a second microservice. In the old world, this means a ticket to NetOps, waiting for pool changes, maybe a change window. Watch what happens now."

### Commands

```bash
# Deploy tea app
kubectl apply -f manifests/apps/app2-tea.yaml

# Verify pods
kubectl get pods -l app=tea

# Test immediately — no waiting
curl -s -H "Host: cafe.example.com" http://10.1.10.100/tea
```

**Expected:** Response from a tea pod — immediately, through the same VIP.

### Talking Points

> "The tea service was live in seconds. Same VIP, same BIG-IP config. NGINX IC handles the routing — /coffee goes to coffee pods, /tea goes to tea pods. NetOps didn't have to do anything. This is how DevOps velocity and NetOps control can coexist."

### Show Both Side-by-Side

```bash
# Hit both services in quick succession
curl -s -H "Host: cafe.example.com" http://10.1.10.100/coffee
curl -s -H "Host: cafe.example.com" http://10.1.10.100/tea
```

> "Same VIP, same BIG-IP virtual server, two completely different services. NGINX IC does the Layer 7 routing. BIG-IP does the Layer 4 delivery and security."

---

## Scenario 5: Apply WAF Policy

**Who:** NetOps (BIG-IP GUI) + DevOps (terminal)

### Talking Points

> "Now NetOps wants to add WAF protection. They create and manage the policy on BIG-IP — that's their domain expertise. DevOps just references the policy name in a CRD."

### NetOps: Create/Verify WAF Policy on BIG-IP

1. **Security → Application Security → Security Policies**
   - Show the `linux-high` policy (or create one from the Rapid Deployment template)
   - "NetOps creates and tunes the WAF policy. They own the security posture."

### DevOps: Reference the WAF Policy via CRD

```bash
# Look at what we're about to apply
cat manifests/waf/waf-policy.yaml

# Apply the WAF policy CRD
kubectl apply -f manifests/waf/waf-policy.yaml
```

### Talking Points

> "Look at that YAML. DevOps doesn't configure WAF rules, signatures, or enforcement modes. They just reference the policy name that NetOps gave them. CIS reads this CRD and attaches the policy to the BIG-IP virtual server."

### Show WAF in Action

```bash
# Normal request — should work fine
curl -s -H "Host: cafe.example.com" http://10.1.10.100/coffee

# Attack request — WAF should block this
curl -s -H "Host: cafe.example.com" \
  "http://10.1.10.100/coffee?input=<script>alert(1)</script>"
```

### Show on BIG-IP

1. **Local Traffic → Virtual Servers** → click VS → **Security → Policies**
   - Show ASM policy is attached
2. **Security → Event Logs → Application → Requests**
   - Show the blocked XSS attempt in the WAF logs

### Talking Points

> "DevOps didn't change their app, didn't add a sidecar, didn't modify any Ingress annotations. NetOps applied security at the BIG-IP layer, and CIS wired it up. Clean separation of concerns."

---

## Scenario 6: Canary Deployment

**Who:** DevOps (terminal)

### Talking Points

> "DevOps needs to do a canary release of coffee v2. They want to send 10% of traffic to the new version while keeping 90% on v1. This is entirely a DevOps operation — the BIG-IP VIP doesn't change at all."

### Commands

```bash
# Show current state — 2 coffee v1 pods
kubectl get pods -l app=coffee

# Deploy canary (coffee v2) — 1 replica
kubectl apply -f manifests/apps/canary-coffee-v2.yaml

# Show both versions running side by side
kubectl get pods -l version=v1
kubectl get pods -l version=v2
```

### Test the Canary

```bash
# Run multiple requests — you should see responses from both v1 and v2
# With 2 v1 pods and 1 v2 pod, roughly 2/3 go to v1 and 1/3 to v2
for i in $(seq 1 10); do
  curl -s -H "Host: cafe.example.com" http://10.1.10.100/coffee | grep "Server name"
done
```

### Talking Points

> "We have both versions running. Traffic is split at the NGINX IC level based on pod count — 2 v1 pods and 1 v2 pod gives us roughly a 67/33 split. For more precise control, you'd use NGINX Plus IC's VirtualServer CRD with weight-based routing."
>
> "The key point: look at BIG-IP. Nothing changed. The VIP is the same, the pool is the same (still pointing to NGINX IC pods), the WAF policy is still active. DevOps is doing canary releases at full speed while NetOps' security posture is completely untouched."

### Show on BIG-IP GUI

1. **Local Traffic → Virtual Servers** — VS hasn't changed
2. **Pool → Members** — still the same NGINX IC pod IPs (not the app pods)
3. "The canary is invisible to BIG-IP. It happens inside the cluster."

### Rollback or Promote

```bash
# If v2 looks good — scale it up and scale v1 down
kubectl scale deployment coffee-v2 --replicas=2
kubectl scale deployment coffee --replicas=0

# Or rollback — just delete v2
# kubectl delete -f manifests/apps/canary-coffee-v2.yaml
```

---

## Scenario 7: CRD-Driven Automation

**Who:** DevOps (terminal)

### Talking Points

> "Let me show how CIS responds to CRDs in real-time. When DevOps applies Kubernetes resources with the right CRD types, CIS automatically configures BIG-IP. No manual steps, no tickets, no delay."

### Live Demo: Change the IngressLink VIP

```bash
# Show current IngressLink
kubectl get ingresslink -n nginx-ingress -o yaml

# Edit the VIP address (simulating a VIP change)
# In a real scenario, NetOps would coordinate this
kubectl patch ingresslink nginx-ingress-link -n nginx-ingress \
  --type='merge' \
  -p '{"spec":{"virtualServerAddress":"10.1.10.101"}}'

# Check BIG-IP — the VS address should update within seconds
```

### Show on BIG-IP GUI

1. Refresh **Local Traffic → Virtual Servers**
2. Show the VS address changed automatically
3. "CIS is watching the K8s API in real-time. Any CRD change gets pushed to BIG-IP within seconds."

### Revert for Next Demo Run

```bash
# Reset the VIP back
kubectl patch ingresslink nginx-ingress-link -n nginx-ingress \
  --type='merge' \
  -p '{"spec":{"virtualServerAddress":"10.1.10.100"}}'
```

---

## Closing Talking Points

> **Two-Persona Model:**
> "NetOps keeps full control of BIG-IP — VIPs, WAF policies, TLS, GSLB. DevOps deploys at Kubernetes speed without waiting for BIG-IP changes. CIS is the bridge that makes both teams productive."
>
> **Automation:**
> "Everything we showed is declarative YAML. This means it can go into a Git repo, pass through CI/CD, and be audited. Infrastructure as Code isn't just for cloud — it works with BIG-IP too."
>
> **Security:**
> "WAF policies are applied at the BIG-IP layer, managed by the people who understand application security. DevOps can move fast without moving past security."

---

## Reset Between Demos

If you need to clean up and start fresh for another demo run:

```bash
# Remove apps (keep infrastructure in place)
kubectl delete -f manifests/apps/
kubectl delete -f manifests/waf/waf-policy.yaml

# Wait a few seconds for CIS to clean up BIG-IP
sleep 10

# Verify apps are gone
kubectl get pods -l app=coffee
kubectl get pods -l app=tea

# Re-deploy starting from Scenario 2
```

---

## Troubleshooting During Demo

| Issue | Quick Fix |
|-------|-----------|
| curl hangs | Check VIP address. Try `curl -v` to see where it stalls. |
| No response from app | `kubectl get pods` — ensure app pods are Running. Check Ingress: `kubectl describe ingress` |
| BIG-IP pool empty | `kubectl logs -n kube-system -l app=k8s-bigip-ctlr --tail=50` — look for errors |
| WAF not blocking | Check policy enforcement mode on BIG-IP (Transparent vs Blocking) |
| Wrong pod responding | Check Ingress host/path rules. Ensure `ingressClassName: nginx` is set. |
