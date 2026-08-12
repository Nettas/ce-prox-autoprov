# Provisioning Caveats

Real-world gotchas discovered while running `docs/f5-ce-proxmox-setup.md`
against 10.0.0.251, that the doc itself doesn't (yet) call out. Append to
this file as new caveats surface on future nodes.

---

## 1. Step 5's boot-order command fails — `ide2` doesn't exist yet

**Doc says (Step 5):**
```bash
qm set <vmid> --scsi0 local-zfs:vm-<vmid>-disk-0,iothread=1
qm set <vmid> --boot order="scsi0;ide2;net0"
```

**What actually happens:**
```
update VM 105: -boot order=scsi0;ide2;net0
invalid bootorder: device 'ide2' does not exist'
```

Proxmox validates that every device named in `--boot order=...` already
exists on the VM. `ide2` (the cloud-init drive) isn't created until **Step
7** (`qm set <vmid> --ide2 local-zfs:cloudinit`), so running the boot-order
command as written, in Step 5, always fails.

**Workaround used (confirmed with the human before deviating):** skip the
boot-order command in Step 5, complete Step 6 (cloud-init snippet) and Step
7 (attach `ide2` + `cicustom`) as normal, then run the *exact same*
`qm set <vmid> --boot order="scsi0;ide2;net0"` command immediately after
Step 7, right before the Step 8 config gate. No change to the command
itself — only its position in the sequence.

**Still open:** doc should be corrected to move this command to right after
Step 7, not left as-is in Step 5. Not fixed yet — flagging here until
someone updates `f5-ce-proxmox-setup.md` directly.

---

## 2. Host is CPU-oversubscribed at F5's stated minimum

Target host (10.0.0.251) is a desktop-class i7-9700K: **8 physical cores,
no hyperthreading** (`lscpu`: 1 thread/core). F5's stated CE minimum is 8
vCPU (`--cores 8 --cpu host`).

At time of VM 105 provisioning, VM 102 (`k8s-gateway`) was already running
with 4 cores committed. Adding CE's 8 cores brought total *committed*
vCPUs to 12 against 8 physical cores — before CE even boots, let alone
under data-plane load, which the doc itself warns "is a full data-plane
workload, not something that idles lightly under oversubscription."

RAM (46Gi free of 62Gi) and disk (858G free) both had comfortable headroom
— only vCPU was tight.

**Resolution:** flagged and stopped per procedure; human explicitly
accepted the oversubscription and instructed to proceed anyway. VM 105
booted and registered network-reachable without an apparent issue, but
this was accepted risk, not validated headroom. Future nodes on this host
will compound the problem — there's no room for a second CE node at F5's
stated minimum without reducing cores elsewhere first.

---

## 3. Token-in-heredoc still lands in the tool-call transcript

The procedure's stated goal is to avoid the token in *shell history* and in
*response text*. Using an SSH heredoc achieves both of those. It does not,
however, avoid the token appearing in the agent's tool-call parameters
within the session transcript — the heredoc content has to be transmitted
somehow, and the Bash tool call itself is logged. Worth knowing this is a
narrower guarantee ("no shell history, no printed-back response") than
"token never appears anywhere in any log."

This is exactly why the procedure ends every run with a reminder to
regenerate the token in the F5 Console — treat the one used in this session
as burned.
