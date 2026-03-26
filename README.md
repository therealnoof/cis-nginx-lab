# F5 CIS + NGINX Plus Ingress Controller Lab

**Two-persona lab demonstrating how F5 BIG-IP and NGINX Plus Ingress Controller work together using Container Ingress Services (CIS) with IngressLink.**

NetOps owns the BIG-IP — VIPs, WAF policies, and GSLB.
DevOps owns Kubernetes — NGINX Plus IC, app deployments, and routing.
CIS bridges the two so new services go live **without a BIG-IP ticket**.

---

## Architecture

```
                          ┌──────────────────────────────────────────┐
                          │              BIG-IP VE                   │
                          │                                          │
                          │  ┌──────────┐   ┌──────────────────┐    │
  Internet ──────────────►│  │   VIP    │──►│   WAF Policy     │    │
                          │  │ (VS)     │   │   (ASM / AWAF)   │    │
                          │  └────┬─────┘   └──────────────────┘    │
                          │       │                                  │
                          │       │  Pool members auto-managed       │
                          │       │  by CIS (IngressLink)            │
                          └───────┼──────────────────────────────────┘
                                  │
                    ──────────────┼──────────────────────────────
                    Kubernetes    │   Single-node kubeadm cluster
                                  │
                          ┌───────▼──────────┐
                          │   CIS Pod        │ ◄── Watches K8s API
                          │   (f5-bigip-ctlr)│     for IngressLink /
                          │                  │     CRDs, updates BIG-IP
                          └──────────────────┘
                                  │
                          ┌───────▼──────────┐
                          │  NGINX Plus IC   │ ◄── ClusterIP service
                          │  (Ingress Ctrl)  │     (no NodePort needed)
                          └───────┬──────────┘
                                  │
                     ┌────────────┼────────────┐
                     │            │            │
               ┌─────▼───┐ ┌─────▼───┐ ┌─────▼───┐
               │  App 1  │ │  App 2  │ │ Canary  │
               │ (coffee)│ │  (tea)  │ │ (coffee │
               │         │ │         │ │  v2)    │
               └─────────┘ └─────────┘ └─────────┘

  NetOps ► manages BIG-IP VIP + WAF via GUI
  DevOps ► manages everything below the line via kubectl / Helm
```

---

## Project Structure

```
cis-nginx-lab/
├── README.md                              <- You are here
├── DEPLOYMENT_GUIDE.md                    <- Full build: bare metal → working demo
├── DEMO_GUIDE.md                          <- SE walkthrough (both personas)
├── .gitignore                             <- Keeps secrets out of git
└── manifests/
    ├── cis/
    │   ├── cis-rbac.yaml                  <- ServiceAccount + ClusterRole for CIS
    │   ├── bigip-login-secret.yaml        <- TEMPLATE — fill in your creds
    │   └── cis-deployment.yaml            <- CIS controller deployment
    ├── nginx-plus-ic/
    │   ├── nginx-plus-ic-values.yaml      <- Helm values for NGINX Plus IC
    │   └── nginx-ingress-clusterip-svc.yaml <- ClusterIP service for IC pods
    ├── ingresslink/
    │   ├── ingresslink-crd.yaml           <- IngressLink CRD definition
    │   └── ingresslink.yaml               <- IngressLink resource (ties BIG-IP → IC)
    ├── apps/
    │   ├── app1-coffee.yaml               <- Sample app: coffee (v1)
    │   ├── app2-tea.yaml                  <- Sample app: tea
    │   └── canary-coffee-v2.yaml          <- Canary: coffee v2 (for split traffic demo)
    └── waf/
        └── waf-policy.yaml                <- WAF policy CRD reference for BIG-IP
```

---

## Prerequisites

- **Ubuntu 22.04 VM** for Kubernetes (4 CPU / 8 GB RAM minimum)
- **BIG-IP VE VM** (v15.1+) with LTM + ASM provisioned, reachable from the K8s node
- **NGINX Plus license** — you will need a JWT token from [MyF5](https://my.f5.com)
- **Internet access** on both VMs (for pulling images and Helm charts)
- Familiarity with `kubectl` basics and BIG-IP TMUI

---

## Quick Start

```bash
# 1. Clone this repo
git clone https://github.com/therealnoof/cis-nginx-lab.git
cd cis-nginx-lab

# 2. Build the K8s cluster (uses the companion install script)
#    See DEPLOYMENT_GUIDE.md Step 1 for full details
git clone https://github.com/therealnoof/k8s-nginx-install.git
sudo bash k8s-nginx-install/k8s-install.sh

# 3. Follow the Deployment Guide end-to-end
#    DEPLOYMENT_GUIDE.md walks through CIS, NGINX Plus IC, and sample apps

# 4. Run the demo using the Demo Guide
#    DEMO_GUIDE.md has the SE-ready walkthrough
```

---

## Guides

| Guide | Audience | Contents |
|-------|----------|----------|
| [Deployment Guide](DEPLOYMENT_GUIDE.md) | Lab builder | Full build from bare Ubuntu to working demo |
| [Demo Guide](DEMO_GUIDE.md) | SE / presenter | Two-persona walkthrough with talking points |

---

## Phase Roadmap

| Phase | Description | Status |
|-------|-------------|--------|
| **Phase 1** | Core lab — CIS + NGINX Plus IC + IngressLink + sample apps | **Available** |
| **Phase 2** | GSLB multi-site with DNS CRDs | Planned |
| **Phase 3** | mTLS between BIG-IP and NGINX IC | Planned |
| **Phase 4** | GitOps pipeline with Argo CD | Planned |

---

## Key Concepts

| Term | What It Does |
|------|-------------|
| **CIS (Container Ingress Services)** | Controller pod that watches K8s API and programs BIG-IP automatically |
| **IngressLink** | CRD that tells CIS to create a BIG-IP VIP pointing to NGINX IC pods |
| **NGINX Plus Ingress Controller** | Kubernetes-native ingress that routes traffic to app pods |
| **WAF Policy CRD** | Lets CIS attach an ASM/AWAF policy on BIG-IP to the VIP — NetOps controls policy, DevOps references it |

---

## License

This lab is provided as-is for demonstration and training purposes.
NGINX Plus requires a valid F5/NGINX license. BIG-IP VE requires a valid F5 license.
