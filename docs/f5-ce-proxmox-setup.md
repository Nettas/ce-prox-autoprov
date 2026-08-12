# F5 Distributed Cloud CE (SMSv2) — Single Node on Proxmox
*VM: ce-aigtwy | IP: 10.0.0.243 | Proxmox VM ID: 103*
 
Deploys a single-node F5 Distributed Cloud Customer Edge using Secure Mesh Site v2
(SMSv2). Proxmox runs KVM under the hood, so this follows F5's official KVM
procedure but substitutes `qm` for `virt-install`.
 
---
 
## What This Is
 
F5 Distributed Cloud CE tunnels your network into F5's SaaS control plane
(Regional Edges), giving app delivery / security services without owning F5
hardware. SMSv2 is F5's simplified onboarding: generate a node token + cloud-init
from the Console, boot a VM with it, and the node self-registers.
 
---
 
## Resource Requirements (F5 Minimums)
 
| Resource | Minimum |
|---|---|
| vCPUs | 8 |
| RAM | 32 GB |
| Disk | 80 GB |
 
⚠️ Check your host's total capacity before deploying. A single CE node at minimum
spec can consume most of a homelab host's RAM — plan which other VMs need to be
shut down to make room, since F5 CE is a full data-plane workload, not something
that idles lightly under oversubscription.
 
---
 
## Prerequisites
 
- Proxmox host reachable, with a storage backend (`local-zfs` or `local-lvm`)
- F5 Distributed Cloud Console account (console.ves.volterra.io)
- CE qcow2 image downloaded from Console (see Step 1)
- Free static IP on your LAN for the SLO (Site Local Outside) interface
- `virt-install --cloud-init` requires v3.0.0+ if using bare KVM — not applicable
  when using `qm`, but noted here since F5's docs assume `virt-install`
---
 
## Step 1 — F5 Console: Create Site Object
 
1. **Multi-Cloud Network Connect** workspace → **Manage** → **Site Management** → **Secure Mesh Sites v2**
2. **Add Secure Mesh Site**
3. Name the site (e.g. `homelab-proxmox-ce`)
4. **Provider Name**: `KVM` (no native Proxmox option — Proxmox runs KVM under the hood)
5. **High Availability**: `Disabled` for single node
   > ⚠️ Cannot be changed after creation. Disabled = 1 node max, no additional nodes can ever be added to this Site.
6. Leave remaining options at default
7. **Add Secure Mesh Site**
---
 
## Step 2 — F5 Console: Generate Node Token
 
> Token is valid 24 hours from generation — don't generate until you're ready to boot the VM.
 
1. On your Site, click **...** → **Generate Node Token**
2. **Copy cloud-init** — this is a YAML block containing the token
3. Save it temporarily; you'll embed it in a cloud-init snippet on the Proxmox host
---
 
## Step 3 — F5 Console: Download CE Node Image
 
1. On your Site, **...** → **Copy Image Name** (gives a curl/wget URL) or **Download Image** directly
2. You want the **qcow2** format (not raw/vmdk) for KVM/Proxmox
3. Transfer the qcow2 to the Proxmox host, e.g. from Windows via PowerShell's built-in `scp`:
```powershell
scp "C:\path\to\f5xc-ce-<version>.qcow2" root@PROXMOX_HOST_IP:/var/lib/vz/template/iso/
```
 
Verify on the Proxmox host:
```bash
ls -lh /var/lib/vz/template/iso/f5xc-ce-<version>.qcow2
```
 
---
 
## Step 4 — Create the VM Shell
 
```bash
qm create 103 \
  --name ce-aigtwy \
  --memory 32768 \
  --cores 8 \
  --cpu host \
  --net0 virtio,bridge=vmbr0 \
  --ostype l26 \
  --scsihw virtio-scsi-single
```
 
---
 
## Step 5 — Import the qcow2 as the VM's Disk
 
```bash
qm importdisk 103 /var/lib/vz/template/iso/f5xc-ce-<version>.qcow2 local-zfs
```
 
This converts and imports the qcow2 into your storage backend as an unattached
disk (`unused0`). Output ends with:
```
unused0: successfully imported disk 'local-zfs:vm-103-disk-0'
```
 
Attach it and set boot order:
 
```bash
qm set 103 --scsi0 local-zfs:vm-103-disk-0,iothread=1
qm set 103 --boot order="scsi0;ide2;net0"
```
 
> ⚠️ Quote the `order` value — unquoted `;` is interpreted by bash as a command
> separator, causing `ide2`/`net0` to run as bogus shell commands (harmless, but
> the boot order won't be fully set).
 
The imported qcow2 lands at F5's default 80GB — matches the minimum spec, no
resize needed. If you do need to grow it:
```bash
qm resize 103 scsi0 <new-size>G
```
and add a `runcmd` block to the cloud-init file (see Step 6) to extend the
filesystem on boot:
```yaml
runcmd:
  - [ sh, -c, test -e /usr/bin/fsextend && /usr/bin/fsextend || true ]
```
 
---
 
## Step 6 — Write the Cloud-Init Snippet
 
```bash
mkdir -p /var/lib/vz/snippets
nano /var/lib/vz/snippets/f5xc-cloud-init.yaml
```
 
```yaml
#cloud-config
# F5 Distributed Cloud Secure Mesh Site v2 - node bootstrap
# Read by cloud-init on first boot, writes /etc/vpm/user_data, which the F5 CE
# image's provisioning agent reads to self-register against the F5 Distributed
# Cloud control plane.
write_files:
  - path: /etc/vpm/user_data
    # token: one-time node registration token from F5 Console (valid 24h)
    # slo_ip/slo_gateway: static IP config for the Site Local Outside (SLO)
    # interface -- the interface that reaches the public internet to register
    # with F5's Regional Edges. Omit slo_dns to fall back to public/default DNS.
    content: |
      token: <PASTE_TOKEN_HERE>
      slo_ip: 10.0.0.243/24
      slo_gateway: 10.0.0.1
    owner: root
    permissions: '0644'
```
 
> ⚠️ The SLO IP cannot be changed after the node registers — get it right before booting.
> ⚠️ VM name must be DNS-1035 compliant: lowercase alphanumeric + `-`, start with a letter, no `.`
 
Paste the real token directly on the host (not through any chat/tool) to avoid
putting a live credential in a transcript. Validate the YAML before moving on:
 
```bash
python3 -c "
import yaml
with open('/var/lib/vz/snippets/f5xc-cloud-init.yaml') as f:
    doc = yaml.safe_load(f)
assert 'write_files' in doc
wf = doc['write_files'][0]
assert wf['path'] == '/etc/vpm/user_data'
print('YAML valid')
"
```
 
---
 
## Step 7 — Attach Cloud-Init Drive
 
```bash
qm set 103 --ide2 local-zfs:cloudinit
qm set 103 --cicustom "user=local:snippets/f5xc-cloud-init.yaml"
```
 
---
 
## Step 8 — Verify Full Config
 
```bash
qm config 103
```
 
Expected:
- `boot: order=scsi0;ide2;net0`
- `cicustom: user=local:snippets/f5xc-cloud-init.yaml`
- `cores: 8`
- `memory: 32768`
- `scsi0: local-zfs:vm-103-disk-0,iothread=1,size=80000M`
- `ide2: local-zfs:vm-103-cloudinit,media=cdrom`
---
 
## Step 9 — Boot and Verify Registration
 
```bash
qm start 103
```
 
Check network reachability:
```bash
ping -c 3 10.0.0.243
```
 
In F5 Console, navigate to your Site's dashboard. **System Health** should
progress: **Waiting for Registration** → **Provisioning** → **100%**, with
**Data Plane** / **Control Plane** both showing up.
 
---
 
## Common Pitfalls
 
| Symptom | Cause / Fix |
|---|---|
| `unused0` left after `--delete scsi0` | `--delete` detaches but doesn't free the volume — run `qm disk unlink 103 --idlist unused0 --force` |
| `ide2`/`net0` run as shell commands | Unquoted `;` in `--boot order=...` — always quote: `--boot order="scsi0;ide2;net0"` |
| Node stuck at "Waiting for Registration" | Token expired (24h limit) or SLO interface can't reach the internet — regenerate token, re-verify network path |
| Host resource exhaustion | 8 vCPU / 32GB RAM per node is substantial for a homelab — plan which other VMs to shut down before boot |
 
---
 
## Reference
 
- Node cannot be resized after deployment (vertical scaling not supported)
- SLO IP/MAC cannot change after registration
- HA mode (Enabled/Disabled) cannot change after Site creation
- Full F5 docs: [Deploy Secure Mesh Site v2 on KVM (ClickOps)](https://docs.cloud.f5.com/docs-v2/multi-cloud-network-connect/how-to/site-management/deploy-sms-kvm-clickops)
---
*Last updated: July 23, 2026 — Phase 5 prep, testing CE on desktop Proxmox (10.0.0.251)*
 
