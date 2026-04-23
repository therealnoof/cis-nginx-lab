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
# Make sure you're in the repo directory
cd /home/ubuntu/cis-nginx-lab

# Delete any existing apps and configs
kubectl delete -f manifests/cis-standalone/ingress-f5-sharedport-app2.yaml 2>/dev/null
kubectl delete -f manifests/cis-standalone/ingress-f5-sharedport-app1.yaml 2>/dev/null
kubectl delete -f manifests/cis-standalone/ingress-f5-app2.yaml 2>/dev/null
kubectl delete -f manifests/cis-standalone/ingress-f5.yaml 2>/dev/null
kubectl delete -f manifests/cis-standalone/as3-configmap-app2.yaml 2>/dev/null
kubectl delete -f manifests/cis-standalone/as3-configmap.yaml 2>/dev/null
kubectl delete -f manifests/cis-standalone/app2-service-as3.yaml 2>/dev/null
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

### Demo A4: Deploy Second App — Show the Overhead

**Who:** DevOps (terminal)

> "Now I need to deploy a second app. Watch what's required — I can't just deploy a Deployment and Service and be done. Whichever path I used for App 1, I have to repeat per-app BIG-IP config for App 2. DevOps has to know BIG-IP internals either way."

Use the **same option you chose in Demo A2** — don't mix AS3 and annotations for the same app.

**Option 1 — AS3 ConfigMap:**
```bash
# Deploy the second app's Service + Deployment (with AS3 labels)
kubectl apply -f manifests/cis-standalone/app2-service-as3.yaml

# Show it's running
kubectl get svc f5-hello-world-web2
kubectl get pods -l app=f5-hello-world-2

# Now update the AS3 declaration to add a second VIP
# This replaces the ConfigMap with one that has BOTH apps
kubectl apply -f manifests/cis-standalone/as3-configmap-app2.yaml
```

**Option 2 — F5 Ingress Annotations (different port per app):**
```bash
# Deploy the second app's Service + Deployment
# (AS3 labels on the Service are ignored when using annotations — harmless)
kubectl apply -f manifests/cis-standalone/app2-service-as3.yaml

# Show it's running
kubectl get svc f5-hello-world-web2
kubectl get pods -l app=f5-hello-world-2

# Deploy a second Ingress with F5 annotations for the new VIP/port
kubectl apply -f manifests/cis-standalone/ingress-f5-app2.yaml
```

Produces TWO separate BIG-IP VS entries (one per port): App 1 on 10.1.20.10:80, App 2 on 10.1.20.10:8081.

**Option 3 — F5 Ingress Annotations (shared VIP + shared port, host-routed):**

If both apps must listen on the same port (e.g., both on 80), CIS can group multiple Ingresses that share `virtual-server.f5.com/ip` + `http-port` into a **single** BIG-IP VS with an LTM traffic policy that routes by Host header (`spec.rules[].host`).

```bash
# The A2 catch-all Ingress (ingress-f5.yaml) has no host — it would
# swallow all traffic on VIP:80 and starve App 2. Replace it with the
# host-routed pair.
kubectl delete -f manifests/cis-standalone/ingress-f5.yaml 2>/dev/null

# Deploy App 2's Service + Deployment (if not already running)
kubectl apply -f manifests/cis-standalone/app2-service-as3.yaml

# Apply the host-routed Ingress pair — both on VIP:80
kubectl apply -f manifests/cis-standalone/ingress-f5-sharedport-app1.yaml
kubectl apply -f manifests/cis-standalone/ingress-f5-sharedport-app2.yaml
```

Produces ONE BIG-IP VS on 10.1.20.10:80, backed by two pools. Traffic policy on the VS picks the pool based on Host header.

**Show on BIG-IP:**
1. **Local Traffic → Virtual Servers** — in the `kubernetes` partition
   - One VS for the shared-port case (Option 3)
   - **Policies** tab on that VS — you can see the Host → pool rules CIS generated
2. **Local Traffic → Pools** — one pool per app, each with its own pod IPs

**Test (no DNS needed — override Host header):**
```bash
curl -s -H 'Host: app1.lab.local' http://10.1.20.10
curl -s -H 'Host: app2.lab.local' http://10.1.20.10
```

**Show on BIG-IP (Options 1 and 2):**
1. **Local Traffic → Virtual Servers** — now TWO VS entries
   - AS3 ConfigMap: check the `AS3` partition
   - F5 Ingress: check the `kubernetes` partition
2. App 1 on port 80, App 2 on port 8081
3. Each has its own pool with its own pod IPs

**Test (Options 1 and 2):**
```bash
# App 1 — port 80
curl -s http://10.1.20.10

# App 2 — port 8081
curl -s http://10.1.20.10:8081
```

> "It works, but look at what I had to do. With AS3, I rewrote the ConfigMap JSON to add a second Application block, picked a new pool name and port. With annotations on separate ports, I wrote a second Ingress and hand-picked the VIP and port. With annotations on a shared port, I had to delete App 1's catch-all Ingress, add hostnames to both, and make sure every VS-level annotation agreed across both Ingresses. Every path has DevOps owning BIG-IP addressing, port/host design, and F5's annotation schema. That doesn't scale to 50 microservices — and hand-rolling host-based fan-out is exactly what NGINX IC does for you in Mode B."

---

### Demo A5: The Limitation — Transition to Mode B

> "Imagine 50 microservices. That's 50 AS3 application blocks, 50 pool definitions, port management, and every DevOps team needs to understand BIG-IP internals. Let me show you what happens when we add NGINX Ingress Controller to the picture."

**Talking point for transition:**
> "In a microservices world, you want one VIP with intelligent L7 routing behind it. DevOps deploys a standard Kubernetes Ingress with a hostname — no BIG-IP knowledge needed. That's Mode B."

---

## Part 2: CIS + IngressLink (Mode B)

> **Story:** "NetOps owns BIG-IP security. DevOps deploys at Kubernetes speed. Neither waits for the other."

> **Note:** If transitioning from Mode A live, you'll need to delete the standalone CIS and deploy the IngressLink CIS. For a smooth demo, have Mode B pre-built on a separate environment.

### Clean Slate Setup (Mode B)

Run these commands before starting the demo to ensure a clean state:

```bash
# Make sure you're in the repo directory
cd /home/ubuntu/cis-nginx-lab

# Delete any existing apps and WAF policy
kubectl delete -f manifests/apps/ 2>/dev/null
kubectl delete -f manifests/waf/waf-policy.yaml 2>/dev/null

# Delete any GSLB resources from a prior Part 3 run
kubectl delete -f manifests/gslb/externaldns-coffee-modeB.yaml 2>/dev/null
kubectl delete -f manifests/gslb/externaldns-tea.yaml 2>/dev/null
kubectl delete -f manifests/gslb/externaldns-coffee.yaml 2>/dev/null
kubectl delete -f manifests/gslb/virtualserver-coffee-modeB.yaml 2>/dev/null

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

### Demo B5: Scale the Ingress Controller

**Who:** DevOps (terminal) + NetOps (BIG-IP GUI)

> "What happens when we need more capacity at the ingress layer? DevOps scales the NGINX IC — and BIG-IP automatically adds the new pods to its pool. No ticket, no manual pool edits."

```bash
# Show current NGINX IC — 1 pod
kubectl get pods -n nginx-ingress

# Scale to 3 replicas
kubectl scale deployment -n nginx-ingress nginx-ingress-controller --replicas=3

# Watch new pods come up
kubectl get pods -n nginx-ingress -w
# (Ctrl+C once all 3 are Running)
```

**Show on BIG-IP:**
1. Refresh **Pool → Members** — 3 NGINX IC pod IPs now
2. "CIS detected the new IC pods and updated the BIG-IP pool in real-time. BIG-IP now load balances across all 3 ingress controllers."

**Test — traffic spreads across IC pods:**
```bash
curl -s -H "Host: coffee.example.com" http://10.1.20.50/
curl -s -H "Host: tea.example.com" http://10.1.20.50/
```

> "Scaling the ingress layer is a DevOps operation. BIG-IP adapts automatically. Apps keep running, no downtime, no config changes. Scale back down and BIG-IP removes the members just as fast."

```bash
# Scale back to 1 for the rest of the demo
kubectl scale deployment -n nginx-ingress nginx-ingress-controller --replicas=1
```

---

### Demo B6: Apply WAF Policy

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

### Demo B7: Canary Deployment

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

### Demo B8: CRD-Driven Automation

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

## Part 3: GSLB with ExternalDNS (Both Modes)

> **Story:** "BIG-IP DNS provides global server load balancing across sites. CIS automates the Wide IP and GSLB pool creation — DevOps deploys an ExternalDNS CRD and BIG-IP DNS starts answering queries."

> **Pre-requisite:** GSLB requires BIG-IP DNS (GTM) provisioned, with Data Centers, GTM Servers, and a DNS Listener configured. CIS deployments must include `--gtm-bigip-url` flags. See [Step 3 in DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md#step-3-optional-gslb-setup-for-externaldns-demos) for the click-by-click setup.

---

### Demo G1: Show BIG-IP DNS Before

**Who:** NetOps (BIG-IP GUI)

> "BIG-IP DNS handles global server load balancing. Right now, no Wide IPs are configured — let's have CIS create them automatically from Kubernetes."

**Show on BIG-IP:**
1. **DNS → GSLB → Wide IPs** — empty (or show existing ones)
2. **DNS → GSLB → Servers** — show the GTM Server objects (these are the prerequisites NetOps set up)
3. "NetOps configured the data centers and servers. Now DevOps can create Wide IPs by deploying a simple CRD."

---

### Demo G2: Deploy ExternalDNS CRD

**Who:** DevOps (terminal)

> "I need coffee.example.com to resolve globally via BIG-IP DNS. I deploy an ExternalDNS CRD — that's it."

**Mode A:** CIS watches standard Ingresses and has a per-app VS with `host: coffee.example.com` already — just apply the ExternalDNS CRD.
```bash
cat manifests/gslb/externaldns-coffee.yaml
kubectl apply -f manifests/gslb/externaldns-coffee.yaml
kubectl get externaldns
```

**Mode B:** CIS runs in CRD mode and doesn't watch Ingresses. You need a `VirtualServer` CRD to anchor the hostname on a BIG-IP VS, then a single-site ExternalDNS bound to that VS.
```bash
# VirtualServer — creates a dedicated per-app VS at 10.1.20.60 (edit the file for your VIP)
kubectl apply -f manifests/gslb/virtualserver-coffee-modeB.yaml

# ExternalDNS — single-site pool pointing at SiteB_Server
kubectl apply -f manifests/gslb/externaldns-coffee-modeB.yaml

kubectl get virtualserver,externaldns -n default
```
> **Mode B traffic path:** the per-app VS pool targets the NGINX IC Service, so GSLB-resolved requests traverse NGINX IC the same way IngressLink requests do — just via a different BIG-IP VIP. Assumes proxy-protocol is disabled on NGINX IC; if you re-enable it for WAF demos, this VS needs a matching iRule.

**Show on BIG-IP DNS:**
1. **DNS → GSLB → Wide IPs** — `coffee.example.com` appeared!
2. Click into it → **Pools** — show the GSLB pool
3. Click into the pool → **Members** — show the LTM virtual server as a pool member
4. "CIS created the Wide IP, the GSLB pool, and linked it to the LTM virtual server. All from one YAML file."

> "DevOps didn't log into BIG-IP DNS. They didn't configure GSLB pools or health monitors manually. One CRD, and the app is globally resolvable."

---

### Demo G3: Test DNS Resolution

**Who:** DevOps (terminal)

> "Let's query BIG-IP DNS directly and see it respond with our VIP address."

```bash
# Query BIG-IP DNS for coffee.example.com
# Replace 10.1.1.4 with your BIG-IP DNS listener IP
dig @10.1.1.4 coffee.example.com A +short

# You should see the VIP address (e.g., 10.1.20.10 or 10.1.20.50)
```

> "BIG-IP DNS is answering with the VIP. In production, you'd delegate the DNS zone to BIG-IP so all clients resolve through it. With multiple sites, BIG-IP DNS would return the closest or healthiest VIP."

---

### Demo G4: Add Second Site (Multi-Site GSLB)

**Who:** DevOps (terminal)

> "Now watch what happens when we have both sites running the same app. BIG-IP DNS load balances across both — active-active GSLB."

```bash
# The coffee ExternalDNS already has two pool entries:
#   - SiteA_Server (Mode A BIG-IP)
#   - SiteB_Server (Mode B BIG-IP)
# Both are active if both sites have the app deployed

# Query DNS multiple times — you should see different VIPs
dig @10.1.1.4 coffee.example.com A +short
dig @10.1.1.4 coffee.example.com A +short
dig @10.1.1.4 coffee.example.com A +short
```

**Show on BIG-IP DNS:**
1. **Wide IP → Pools** — show both pools with green status
2. "BIG-IP DNS is health-checking both sites. If one goes down, DNS automatically stops returning that VIP. Instant failover, no manual intervention."

> "This is the full picture: CIS manages LTM virtual servers from Kubernetes CRDs, and CIS manages GTM Wide IPs from ExternalDNS CRDs. NetOps controls the GSLB topology and health policies. DevOps just deploys YAML."

---

### Demo G5: Clean Up GSLB

```bash
# Mode A artifacts
kubectl delete -f manifests/gslb/externaldns-coffee.yaml 2>/dev/null

# Mode B artifacts (VS + single-site ExternalDNS + tea)
kubectl delete -f manifests/gslb/externaldns-coffee-modeB.yaml 2>/dev/null
kubectl delete -f manifests/gslb/virtualserver-coffee-modeB.yaml 2>/dev/null
kubectl delete -f manifests/gslb/externaldns-tea.yaml 2>/dev/null
```

**Show on BIG-IP DNS:** Wide IPs are removed automatically. In Mode B the per-app VS at `10.1.20.60` also disappears — GSLB artifacts leave no trace.

---

## Closing Talking Points

> **Mode A vs Mode B:**
> "CIS standalone is simple and powerful — BIG-IP talks directly to pods. But in a microservices world with 10+ services and separate NetOps/DevOps teams, IngressLink shines. One BIG-IP VIP, infinite services behind NGINX IC."
>
> **Two-Persona Model:**
> "NetOps keeps full control of BIG-IP — VIPs, WAF, TLS, GSLB. DevOps deploys at Kubernetes speed. CIS is the bridge."
>
> **GSLB:**
> "ExternalDNS CRDs let DevOps define global DNS entries the same way they define apps — declarative YAML. BIG-IP DNS handles the health checking, topology routing, and failover. Multi-site resilience from a single CRD."
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

### Pool members showing DOWN (but CIS successfully pushed config)

This is the most common "it looks deployed but traffic fails" scenario. CIS authenticated to BIG-IP and created the pool — you can see the member IPs in the GUI — but every member is red. The config plane worked; the data plane can't reach the pods.

**What it means:** BIG-IP cannot route to the pod CIDR. In this lab both modes use CIS in ClusterIP mode, so BIG-IP must join the Flannel VXLAN overlay to talk directly to pod IPs (Mode A = app pod IPs, Mode B = NGINX IC pod IPs).

**Checks — run in this order:**

1. **Confirm CIS actually pushed members** (rules out a config-plane issue):
   ```bash
   kubectl logs -n kube-system -l app=k8s-bigip-ctlr --tail=100 | grep -iE 'post|as3|error'
   ```
   If you see `[AS3] posting... success`, the control plane is fine — the problem is network reachability.

2. **Check the Flannel VXLAN tunnel on BIG-IP** (this is the #1 cause in this lab):
   ```
   tmsh show net tunnels tunnel flannel_vxlan all-properties
   tmsh show net fdb tunnel flannel_vxlan
   tmsh show net arp | grep <pod-cidr>
   ```
   Empty FDB / no ARP entries for the pod CIDR means BIG-IP never joined the overlay. CIS writes FDB entries via the k8s API — check the CIS deployment args for `--flannel-name=fl-vxlan` and a matching `net tunnel` on BIG-IP.

3. **Compare VTEP MACs on both sides of the tunnel** (critical in AWS — reboots/replacements regenerate Flannel VTEP MACs, and the stale side keeps pointing at the old one). There are **two independent drift cases**, with different fixes — check both:

   **K8s side — what Flannel is currently advertising:**
   ```bash
   # VTEP MAC per node, straight from the annotation CIS reads
   kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.flannel\.alpha\.coreos\.com/backend-data}{"\n"}{end}'

   # Cross-check on the node itself
   ip -d link show flannel.1
   bridge fdb show dev flannel.1
   ```
   The `VtepMAC` field in `backend-data` is the source of truth for *worker* nodes — CIS reads it from the annotation and pushes it to BIG-IP. For the `bigip1` fake node, the annotation is what *workers* use to reach BIG-IP, so it must match BIG-IP's actual tunnel MAC.

   **BIG-IP side — what's actually in the tunnel:**
   ```
   # FDB: one entry per worker node
   tmsh show net fdb tunnel flannel_vxlan

   # Tunnel's own MAC (the BIG-IP side of the VXLAN)
   tmsh show net tunnels tunnel flannel_vxlan all-properties | grep -iE 'mac|discard'
   ```

   **Case 3a — Worker node VTEP drift (worker rebooted/replaced):**
   The MAC in BIG-IP's FDB for that node no longer matches the `VtepMAC` in its annotation. CIS hasn't re-synced after the AWS-side change — force it:
   ```bash
   kubectl rollout restart deployment/k8s-bigip-ctlr -n kube-system
   kubectl logs -n kube-system -l app=k8s-bigip-ctlr --tail=100 | grep -iE 'fdb|flannel|vtep'
   ```
   Re-run `tmsh show net fdb tunnel flannel_vxlan` after the restart — MACs should now match and pool members should turn green.

   **Case 3b — BIG-IP VTEP drift (BIG-IP rebooted):**
   BIG-IP's `flannel_vxlan` tunnel MAC (from `tmsh show net tunnels ...`) doesn't match the `VtepMAC` in the `bigip1` node annotation. Workers encapsulate return traffic with the stale MAC; BIG-IP decaps, sees a frame for a MAC that isn't its tunnel, and drops it — you'll see a climbing **Incoming Discard Packets** counter on the tunnel. CIS does **not** manage the `bigip1` annotation, so restarting CIS won't help. Re-annotate manually with the current tunnel MAC:
   ```bash
   # Use the MAC from: tmsh show net tunnels tunnel flannel_vxlan all-properties
   kubectl annotate node bigip1 \
     flannel.alpha.coreos.com/backend-data='{"VtepMAC":"<current-bigip-tunnel-mac>"}' \
     --overwrite
   ```
   Incoming discards should stop climbing immediately and pool members should go green.

4. **Verify the self-IP on the tunnel is in the pod CIDR range:**
   ```
   tmsh list net self | grep -A3 flannel
   ```
   Self-IP must be on the Flannel subnet (e.g., `10.244.20.1/16`) — not the node network.

5. **Health monitor hitting the wrong port:**
   - **Mode A:** monitor should target the app's containerPort (e.g., `8080`), not the Service port.
   - **Mode B:** monitor should target the NGINX IC pod's port (`80`/`443`), not the IngressLink VS port. Check the IngressLink CR's `monitor` block.

6. **NetworkPolicy or pod-level firewall:**
   ```bash
   kubectl get networkpolicy -A
   ```
   Any policy that doesn't allow-list the BIG-IP self-IP on the VXLAN will silently drop health checks.

**Quick sanity test from BIG-IP shell:**
```
ping <pod-ip>          # L3 reachability via VXLAN
curl <pod-ip>:<port>   # L4/L7 reachability (mimics the monitor)
```
If ping fails → tunnel/FDB/MAC mismatch (steps 2–4). If ping works but curl fails → monitor port or NetworkPolicy (steps 5–6).
