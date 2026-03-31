# F5 CIS + NGINX Plus Ingress Controller Lab

**Two deployment modes demonstrating how F5 BIG-IP Container Ingress Services (CIS) integrates with Kubernetes — standalone and with NGINX Plus Ingress Controller via IngressLink.**

NetOps owns the BIG-IP — VIPs, WAF policies, and GSLB.
DevOps owns Kubernetes — apps, routing, and (in Mode B) NGINX Plus IC.
CIS bridges the two so new services go live **without a BIG-IP ticket**.

---

## Two Deployment Modes

### Mode A: CIS Standalone (BIG-IP → Pods)

```
  Client → BIG-IP VIP → App Pod IPs (via VXLAN)

  BIG-IP handles EVERYTHING: L4 LB, L7 routing, WAF, SSL
  No NGINX IC in the path
```

**Use when:** You want BIG-IP to directly load balance to pods. Simpler setup, fewer components. Good for traditional apps or when BIG-IP needs full L7 control.

### Mode B: CIS + IngressLink (BIG-IP → NGINX IC → Pods)

```
  Client → BIG-IP VIP → NGINX IC Pods → App Pods

  BIG-IP handles: L4 LB, WAF, SSL offload, GSLB
  NGINX IC handles: L7 routing, host/path rules, canary, rate limiting
```

**Use when:** You want the two-persona model — NetOps controls BIG-IP security, DevOps controls NGINX routing at Kubernetes speed. Best for microservices and teams that need independent velocity.

---

## Architecture

```
                    MODE A (Standalone)         MODE B (IngressLink)
                    ───────────────────         ─────────────────────

                    ┌───────────────┐           ┌───────────────┐
  Internet ────────►│  BIG-IP VIP   │           │  BIG-IP VIP   │◄──── Internet
                    │  + WAF        │           │  + WAF        │
                    │  + L7 routing │           │  (L4 only)    │
                    └──────┬────────┘           └──────┬────────┘
                           │                           │
                    ───────┼────── K8s ────────────────┼──── K8s ──
                           │                           │
                           │                    ┌──────▼────────┐
                           │                    │  NGINX Plus   │
                           │                    │  Ingress Ctrl │
                           │                    │  (L7 routing) │
                           │                    └──────┬────────┘
                           │                           │
                    ┌──────▼──┐              ┌─────────▼─────────┐
                    │ App Pod │              │  /coffee  │  /tea │
                    │         │              │  App 1    │ App 2 │
                    └─────────┘              └───────────────────┘
```

---

## Project Structure

```
cis-nginx-lab/
├── README.md                                     <- You are here
├── DEPLOYMENT_GUIDE.md                           <- Full build guide (both modes)
├── DEMO_GUIDE.md                                 <- SE walkthrough (both modes)
├── .gitignore
└── manifests/
    ├── cis/
    │   ├── cis-rbac.yaml                         <- ServiceAccount + ClusterRole (shared)
    │   ├── bigip-login-secret.yaml               <- TEMPLATE — fill in creds (shared)
    │   └── f5-cis-crds.yaml                      <- All F5 CIS CRDs (shared)
    │
    ├── cis-standalone/                           <- MODE A: BIG-IP → Pods
    │   ├── cis-deployment-standalone.yaml        <- CIS with --custom-resource-mode=false
    │   ├── as3-configmap.yaml                    <- AS3 declaration for BIG-IP VIP
    │   ├── app-service-as3.yaml                  <- App 1 + Service with AS3 labels
    │   ├── app2-service-as3.yaml                 <- App 2 + Service (shows Mode A overhead)
    │   ├── as3-configmap-app2.yaml               <- AS3 declaration with both apps
    │   └── ingress-f5.yaml                       <- Alternative: F5 Ingress annotations
    │
    ├── cis-ingresslink/                          <- MODE B: BIG-IP → NGINX IC → Pods
    │   ├── cis-deployment-ingresslink.yaml       <- CIS with --custom-resource-mode=true
    │   ├── ingresslink-crd.yaml                  <- IngressLink CRD definition
    │   ├── ingresslink.yaml                      <- IngressLink resource (VIP → IC)
    │   ├── nginx-ingress-service.yaml            <- ClusterIP service for NGINX IC
    │   └── nginx-config-proxy-protocol.yaml      <- Proxy Protocol config for NGINX
    │
    ├── nginx-plus-ic/                            <- NGINX Plus IC (Mode B only)
    │   ├── nginx-plus-ic-values.yaml             <- Helm values
    │   └── nginx-ingress-clusterip-svc.yaml      <- Reference ClusterIP service
    │
    ├── apps/                                     <- Sample apps (Mode B)
    │   ├── app1-coffee.yaml                      <- Coffee app + Ingress
    │   ├── app2-tea.yaml                         <- Tea app + Ingress
    │   └── canary-coffee-v2.yaml                 <- Canary deployment
    │
    ├── gslb/                                     <- GSLB / ExternalDNS (both modes)
    │   ├── externaldns-coffee.yaml               <- Wide IP for coffee.example.com
    │   └── externaldns-tea.yaml                  <- Wide IP for tea.example.com
    │
    └── waf/
        └── waf-policy.yaml                       <- WAF policy CRD (Mode B)
```

---

## Prerequisites

- **Ubuntu 22.04 VM** for Kubernetes (4 CPU / 8 GB RAM minimum)
- **BIG-IP VE VM** (v15.1+) with LTM + ASM provisioned, reachable from the K8s node
- **AS3 extension** installed on BIG-IP ([GitHub releases](https://github.com/F5Networks/f5-appsvcs-extension/releases))
- **NGINX Plus license** (Mode B only) — JWT token from [MyF5](https://my.f5.com)
- **Internet access** on both VMs

---

## Quick Start

```bash
# 1. Clone this repo
git clone https://github.com/therealnoof/cis-nginx-lab.git
cd cis-nginx-lab

# 2. Build the K8s cluster
git clone https://github.com/therealnoof/k8s-nginx-install.git
sudo bash k8s-nginx-install/k8s-install.sh

# 3. Choose your mode and follow the Deployment Guide:
#    Mode A (standalone): Steps 1-2, then 4A, 6A
#    Mode B (IngressLink): Steps 1-2, then 3, 4B, 5, 6B
```

---

## Guides

| Guide | Audience | Contents |
|-------|----------|----------|
| [Deployment Guide](DEPLOYMENT_GUIDE.md) | Lab builder | Shared setup: K8s cluster + BIG-IP prep |
| [Mode A: CIS Standalone](DEPLOYMENT_GUIDE_MODE_A.md) | Lab builder | BIG-IP → Pods directly |
| [Mode B: CIS + IngressLink](DEPLOYMENT_GUIDE_MODE_B.md) | Lab builder | BIG-IP → NGINX IC → Pods |
| [Demo Guide](DEMO_GUIDE.md) | SE / presenter | Walkthrough for both modes with talking points |

---

## Phase Roadmap

| Phase | Description | Status |
|-------|-------------|--------|
| **Phase 1** | Core lab — CIS standalone + CIS with IngressLink | **Available** |
| **Phase 2** | GSLB multi-site with ExternalDNS CRDs | **Available** |
| **Phase 3** | mTLS between BIG-IP and NGINX IC | Planned |
| **Phase 4** | GitOps pipeline with Argo CD | Planned |

---

## Key Concepts

| Term | What It Does |
|------|-------------|
| **CIS (Container Ingress Services)** | Controller pod that watches K8s API and programs BIG-IP automatically |
| **AS3 (Application Services 3)** | Declarative JSON API for configuring BIG-IP — CIS uses it under the hood |
| **IngressLink** | CRD that tells CIS to create a BIG-IP VIP pointing to NGINX IC pods (Mode B) |
| **NGINX Plus Ingress Controller** | Kubernetes-native ingress that routes traffic to app pods (Mode B) |
| **WAF Policy CRD** | Lets CIS attach an ASM/AWAF policy on BIG-IP to the VIP |
| **Proxy Protocol** | Passes real client IP from BIG-IP through NGINX IC to app pods (Mode B) |

---

## Mode Comparison

| Capability | Mode A (Standalone) | Mode B (IngressLink) |
|-----------|--------------------|--------------------|
| L4 Load Balancing | BIG-IP | BIG-IP |
| L7 Routing | BIG-IP | NGINX IC |
| WAF | BIG-IP (direct) | BIG-IP (in front of NGINX) |
| New app deployment | Requires new AS3/Ingress | Just add K8s Ingress — no BIG-IP change |
| Canary releases | Not native | NGINX IC handles it |
| Persona separation | Weak — BIG-IP does everything | Strong — NetOps/DevOps independent |
| Complexity | Lower | Higher (more components) |
| Best for | Traditional apps, simple setups | Microservices, team autonomy |

---

## License

This lab is provided as-is for demonstration and training purposes.
NGINX Plus requires a valid F5/NGINX license. BIG-IP VE requires a valid F5 license.
