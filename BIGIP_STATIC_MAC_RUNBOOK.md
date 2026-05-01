# BIG-IP VE Static MAC Pinning — Runbook

**Lab:** cis-nginx-lab | **Owner:** Noof (F5 SE)
**Date drafted:** 2026-05-01 | **Status:** Pre-work, no changes made yet

---

## 1. Why This Matters

BIG-IP VE licenses are cryptographically bound to the management interface MAC address.
If a hypervisor reassigns the vNIC MAC on reboot or live-migration, the license check
fails and the data plane stops processing traffic. Beyond licensing, this lab runs a
Flannel VXLAN overlay (`flannel_vxlan`) whose VTEP MAC is annotated directly on each
Kubernetes worker node — MAC churn silently breaks pod-to-BIG-IP routing without any
obvious error. GSLB sync-group membership and CIS pool-member health checks also rely
on stable network identity: ARP entries go stale, DHCP reservations miss, and HA
failover MAC-masquerade configs can conflict. Pinning once removes the entire class of
"works until next reboot" problems.

---

## 2. Record Current MACs Before Touching Anything (Pre-Change)

```bash
# On BIG-IP (run from tmsh or bash — do both to cross-check)
tmsh show net interface                  # per-interface MAC + stats
tmsh show sys hardware                   # chassis-level NIC info
ip link show                             # Linux kernel view — ground truth

# Capture the VXLAN tunnel MAC specifically (critical for K8s annotation)
tmsh show net tunnels tunnel flannel_vxlan all-properties | grep -i mac

# Save a UCS backup before any changes
tmsh save sys ucs /var/local/ucs/pre_mac_pin_$(date +%Y%m%d).ucs
```

Paste the MAC for every vNIC into a scratch note alongside the current hypervisor VM
name and the interface role (mgmt / internal / external / HA).

---

## 3. Pre-Change Checklist

- [ ] UCS saved (step above)
- [ ] Current MACs recorded for all vNICs
- [ ] License type confirmed — run `tmsh show sys license | grep -E "active|Registration"`.
      If it's an eval/time-limited license, contact F5 before changing the mgmt MAC.
- [ ] If HA pair: schedule maintenance window; notify standby unit; confirm config-sync
      is current (`tmsh show cm sync-status`)
- [ ] Identify which NIC role carries the mgmt IP — that MAC is the license anchor.
      In this lab the mgmt network is `10.1.1.0/24`; internal data-plane is `10.1.20.0/24`.

---

## 4. Hypervisor-Side Steps — Pin the MAC

### vSphere / ESXi (via vCenter UI)

1. Power off the BIG-IP VM (required; vCenter blocks MAC edits on running VMs).
2. VM → Edit Settings → select vNIC → expand "MAC Address" → change from
   **Automatic** to **Manual**.
3. Enter the desired MAC. **Use the `00:50:56` OUI prefix** (VMware's reserved range
   for manually assigned MACs — avoids conflicts with auto-assigned `00:0C:29` and
   `00:50:56:80:00:00`–`FF:FF:FF` dynamic pool).
   Example: `00:50:56:ab:cd:01` (mgmt), `00:50:56:ab:cd:02` (internal), etc.
4. Verify every MAC is unique within the broadcast domain before saving.
5. Power on the VM.

### vSphere / ESXi (via govc CLI — scriptable)

```bash
# List current NICs and MACs
govc vm.info -json <VM_NAME> | jq '.VirtualMachines[].Config.Hardware.Device[]
  | select(.MacAddress?) | {label:.DeviceInfo.Label, mac:.MacAddress}'

# Power off
govc vm.power -off <VM_NAME>

# Set MAC on a specific device key (get key from vm.info output, e.g. 4000 = NIC 1)
govc vm.customize -vm <VM_NAME> ...   # (govc doesn't expose direct MAC edit)
# Preferred: use PowerCLI or vCenter UI for MAC assignment; govc is better for
# verification than for editing MACs directly.

# Power on
govc vm.power -on <VM_NAME>
```

### KVM / libvirt (virsh)

```bash
# Dump the domain XML
virsh dumpxml <domain-name> > bigip_domain_backup.xml

# Open for editing — find <interface type='network'> blocks
virsh edit <domain-name>
# In each <interface> block, set or change:
#   <mac address='52:54:00:ab:cd:01'/>
# KVM convention: use 52:54:00 OUI for static assignments.
# Save and exit — change takes effect on next boot (or hot-unplug/replug for
# interfaces that support it, but safer to reboot).

virsh reboot <domain-name>
```

### Proxmox (qm CLI)

```bash
# List current NIC config
qm config <VMID>

# Pin MAC on net0 (mgmt) — Proxmox will reject duplicates in the same cluster
qm set <VMID> --net0 virtio=00:50:56:ab:cd:01,bridge=vmbr0

# Pin net1 (internal data-plane, 10.1.20.x)
qm set <VMID> --net1 virtio=00:50:56:ab:cd:02,bridge=vmbr1

# Verify
qm config <VMID> | grep ^net

# Reboot if the VM is running
qm reboot <VMID>
```

---

## 5. BIG-IP-Side Verification After Reboot

```bash
# Confirm the OS sees the expected MACs
ip link show

# Confirm TMM / TMOS view matches
tmsh show net interface

# Re-check the VXLAN tunnel MAC — this value must match the K8s node annotation
tmsh show net tunnels tunnel flannel_vxlan all-properties | grep -i mac

# Confirm license is still active (most critical check)
tmsh show sys license | grep -E "active|Licensed|Registration|ends"
# Expected: "Licensed" and a future end date. If you see errors, see §7 Rollback.
```

---

## 6. Post-Change: Update the Kubernetes VTEP Annotation

The VXLAN VTEP MAC is **hardcoded in a Kubernetes node annotation**. If the BIG-IP
tunnel MAC changed, the annotation must be patched or pod traffic will silently drop.

```bash
# 1. Get the new tunnel MAC from BIG-IP
NEW_MAC=$(ssh admin@10.1.1.4 'tmsh show net tunnels tunnel flannel_vxlan \
  all-properties | grep -i mac | awk "{print \$NF}"')
echo "New VTEP MAC: $NEW_MAC"

# 2. Patch the annotation on every worker node that routes through BIG-IP
#    (the annotation value is JSON — preserve the PublicIP field)
kubectl annotate node <worker-node-name> \
  "flannel.alpha.coreos.com/backend-data={\"VtepMAC\":\"${NEW_MAC}\"}" \
  --overwrite

# 3. Verify CIS picks up the updated FDB entry (allow ~30 s for reconcile)
ssh admin@10.1.1.4 'tmsh show net fdb tunnel flannel_vxlan'
# Expect to see the worker node IP → new MAC mapping

# 4. Smoke-test pod connectivity from BIG-IP
ssh admin@10.1.1.4 'ping -c 3 10.244.0.1'   # adjust to a live pod IP
```

Relevant files in this repo that reference the VTEP MAC / tunnel config:
- `DEPLOYMENT_GUIDE.md` lines 203–235 — annotation format + MAC query command
- `DEMO_GUIDE.md` lines 669–738 — MAC drift troubleshooting (two specific failure cases)
- `manifests/cis-standalone/cis-deployment-standalone.yaml` line 61 — `--flannel-name` flag
- `manifests/cis-ingresslink/cis-deployment-ingresslink.yaml` line 60 — same flag

No Terraform or Ansible exists in this repo; all changes are manual + kubectl.

---

## 7. What Else Breaks on Unexpected MAC Churn

- **VE License** — bound to mgmt NIC MAC; license validation fails → data plane halts.
- **HA failover MAC masquerade** — floating MACs are derived from the base NIC MAC; if
  the base changes, gratuitous ARPs for VIPs go out with the wrong source and peers may
  reject the failover.
- **GSLB sync-group membership** — sync-group peers identify each other partly by
  stable device identity; MAC instability can trigger re-election or split-brain.
- **External ARP / static DHCP reservations** — any upstream switch ARP cache or DHCP
  server reservation keyed to the old MAC stops working until cleared.
- **CIS pool-member health** — if VXLAN VTEP MAC drifts, Flannel FDB entries go stale
  and BIG-IP can no longer reach pod IPs, failing all health monitors.
- **Prometheus / SNMP monitoring** — interface-level stats are keyed by ifIndex/MAC in
  some collectors; churn causes metric gaps or duplicate series.

---

## 8. Rollback

### If the license fails after an intentional MAC change

```bash
# Option A — Revert the hypervisor MAC to the original value (see §4)
# Then on BIG-IP, force a license re-check:
tmsh run sys failover standby    # if HA, fail to standby first
tmsh load sys license            # re-reads from F5 license server (needs internet)
# OR if air-gapped:
# Re-activate via GUI: System → License → Re-activate

# Option B — If MAC change is permanent and correct, the license must be reissued.
# Contact F5 Support with:
#   - Original registration key (tmsh show sys license | grep Registration)
#   - Old MAC and new MAC for the mgmt interface
#   - Reason for change
# F5 will reissue to the new MAC anchor. ETA is typically same-day for SE labs.
```

### If VTEP MAC drift breaks pod routing before you can patch the annotation

```bash
# Quick restore: re-annotate with the correct current MAC (see §6)
# Or revert hypervisor MAC (§4) and reboot — the old annotation will be valid again.
```

---

*End of runbook. No files were modified in this repo to produce this document.*
