# Provisioning Caveats

Real-world gotchas discovered while running `docs/f5-ce-proxmox-setup.md`,
that the doc itself may not fully call out yet. Append to this file as new
caveats surface on future nodes.

---

## 1. Step 5's boot-order command fails — `ide2` doesn't exist yet

**Status: fixed in `docs/f5-ce-proxmox-setup.md`** (boot order now set in
Step 7, not Step 5). Keeping this entry for context on why the doc reads
the way it does now.

**What happened originally:** the doc's Step 5 included
`qm set <vmid> --boot order="scsi0;ide2;net0"` immediately after attaching
`scsi0`. Proxmox validates that every device named in `--boot order=...`
already exists on the VM. `ide2` (the cloud-init drive) isn't created
until Step 7 (`qm set <vmid> --ide2 local-zfs:cloudinit`), so running the
boot-order command as originally written, in Step 5, always failed with:

```
invalid bootorder: device 'ide2' does not exist
```

**Resolution:** moved the boot-order command to the end of Step 7, right
after `ide2` and `cicustom` are set. No change to the command itself, only
its position in the sequence.

---

## 2. Host CPU oversubscription — RAM/disk headroom isn't the whole picture

Encountered on a desktop-class host with 8 physical cores and no
hyperthreading (`lscpu` showed 1 thread/core). F5's stated CE minimum is
8 vCPU.

At the time, another VM on the same host was already running with several
cores committed. Adding CE's 8 cores brought total *committed* vCPUs above
the host's physical core count — before CE even booted, let alone under
data-plane load, which the reference doc itself warns doesn't idle lightly
under oversubscription.

RAM and disk both had comfortable headroom in this case — only vCPU was
tight. The original resource check only looked at `free -h` / `qm list`
memory figures, which don't surface vCPU oversubscription at all.

**Resolution:** the subagent's Step 1 resource check now explicitly runs
`lscpu` for physical core count and sums committed vCPUs across running
VMs, not just RAM/disk. If you hit this, the trade-off is real: proceeding
with an oversubscribed host is a judgment call the human should make
explicitly, not something to route around silently. Provisioning still
succeeded network-reachable-wise in this case, but that's accepted risk,
not validated headroom — a second node at F5's minimum on the same host
without freeing capacity elsewhere first would very likely make things
worse, not just tight.

---

## 3. Token-in-heredoc has a narrower privacy guarantee than it sounds like

The procedure's stated goal is to avoid the token appearing in *shell
history* and in *printed response text*. Using an SSH heredoc achieves
both of those specifically. It does not avoid the token appearing in the
agent's tool-call parameters within the session transcript itself — the
heredoc content has to be transmitted somehow, and the tool call carrying
it is logged as part of the session.

Worth being precise about this: "no shell history, no printed-back
response" is a narrower guarantee than "token never appears anywhere in
any log." This is exactly why the procedure ends every run with a mandatory
reminder to regenerate the token in the F5 Console — treat the token used
in any given provisioning session as burned once that session is done,
regardless of how carefully it was handled during the session.

