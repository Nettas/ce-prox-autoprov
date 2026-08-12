# F5 XC CE Provisioning — Prompt-Only Version

This branch is the simplest of the three approaches in this repo: a single
copy-paste prompt template you fill in and paste directly into a Claude
Code session. No subagent, no `CLAUDE.md` auto-loaded context, no MCP
config — just a prompt.

**This approach was used successfully to provision a working CE node.**
It's kept here as a standalone, minimal option for anyone who wants
something simpler than the subagent-based approach on `main`, or wants to
understand the underlying procedure without the subagent abstraction layer.

## When to use this branch vs. `main`

| | This branch (`prompt-only`) | `main` (subagent-based) |
|---|---|---|
| Setup | None — just copy the template | Requires the repo's `CLAUDE.md` + subagent files in place |
| Repeatability | You retype/re-paste the filled prompt each time | Invoke by name once set up: `Use f5-ce-provisioner: ...` |
| Guardrails | Whatever you write into the prompt each time | Baked into the subagent's instructions permanently |
| Good for | One-off runs, understanding the raw procedure, environments without Claude Code project config | Repeated provisioning, teams, anywhere you want the rules enforced consistently |

Both approaches call the same underlying `qm`/cloud-init procedure — this
isn't a different or lesser method, just a different packaging of it.

---

## Before using this

1. Complete Steps 1–3 in `docs/f5-ce-proxmox-setup.md` yourself (F5
   Console: create Site, generate node token, download qcow2 to the
   Proxmox host). Token is valid 24h — don't generate it until you're
   ready to run this prompt.
2. Fill in the placeholders below.
3. Paste the filled-in prompt into a **Claude Code** session — the token
   is a live credential and should only exist in the session that
   actually needs it.
4. **After registration succeeds in the F5 Console, regenerate the node
   token** to invalidate the one used here. This is the actual control —
   treat it as mandatory, not optional cleanup.

## ⚠️ Never commit a filled-in copy of this file

Only the template (with placeholders) belongs in git. If you fill in real
values to paste into Claude Code, do it in a scratch buffer / directly at
paste time — do not save or commit a version with the real token, IP, or
VM name if any of that is sensitive.

---

## Prompt template

```
Reference: docs/f5-ce-proxmox-setup.md in this project (F5 XC CE SMSv2
single-node on Proxmox).

Provision a new F5 XC CE node following that guide, steps 4-9. I've already
generated the node token and downloaded the qcow2 through the F5 Console
(steps 1-3 done).

Inputs:
- Proxmox host: <PROXMOX_HOST_IP>
- VM ID: <VMID>
- VM name: <NAME>
- SLO IP: <SLO_IP/24>
- SLO gateway: <GATEWAY>
- qcow2 path on Proxmox host: <PATH>
- Node token: <TOKEN>

Do the following, one command at a time, confirming output before
proceeding:
1. Check host free RAM (`free -h`), running VMs (`qm list`), AND physical
   CPU core count (`lscpu`) — sum committed vCPUs across running VMs and
   compare against physical cores, not just thread count. Flag and stop if
   headroom is insufficient for 8 vCPU / 32GB RAM / 80GB disk (the doc's
   stated minimum) on ANY of these three resources, otherwise proceed.
2. Create the VM shell (Step 4) with those resources.
3. Import the qcow2 as the disk (Step 5) and attach scsi0 — do NOT set
   boot order yet, since `ide2` doesn't exist until Step 7 and setting
   boot order early will fail with
   `invalid bootorder: device 'ide2' does not exist`.
4. Write the cloud-init snippet (Step 6) with the token and SLO IP/gateway
   filled in. Write the token via SSH heredoc directly on the host — never
   as a bare shell argument, never printed back in your response.
5. Validate the YAML with the python3/yaml.safe_load check from the doc
   before moving on.
6. Attach the cloud-init drive AND set the boot order together (Step 7) —
   `qm set <vmid> --boot order="scsi0;ide2;net0"`, quoted, per the doc's
   pitfall note on unquoted semicolons. This only works now that `ide2`
   exists.
7. Show me `qm config <vmid>` against the Step 8 "Expected" block and
   confirm it matches before I boot it.
8. Boot the VM (Step 9), then `ping -c 3 <slo_ip>` to confirm reachability.

Note: SLO IP and HA mode cannot change after registration — if the config
diff in step 7 doesn't match, stop and ask before booting.

Reminder for me: regenerate the node token in the F5 Console once
registration completes, to invalidate the one used here.
```

---

## Known pitfalls (from `docs/f5-ce-proxmox-setup.md`)

| Symptom | Cause / Fix |
|---|---|
| `invalid bootorder: device 'ide2' does not exist` | Boot order was requested before `ide2` existed — it must be set in Step 7, after the cloud-init drive is attached, not earlier |
| `unused0` left after `--delete scsi0` | `--delete` detaches but doesn't free the volume — `qm disk unlink <vmid> --idlist unused0 --force` |
| `ide2`/`net0` run as shell commands | Unquoted `;` in `--boot order=...` — always quote |
| Stuck at "Waiting for Registration" | Token expired (24h) or SLO interface can't reach internet — regenerate token, re-verify network path |
| Host resource exhaustion (RAM/disk) | 8 vCPU / 32GB RAM per node is substantial for a homelab host — check headroom before creating the VM shell |
| Host resource exhaustion (vCPU specifically) | A host can have free RAM/disk but still be short on physical cores if other VMs are running — check `lscpu`, not just `free -h` |

## Notes

- SLO IP/MAC and HA mode cannot be changed after registration.
- This template follows the same discipline as `main`'s subagent: one
  command at a time, confirm before destructive, validate config before
  handing it over — it's the same rules, just carried in the prompt text
  itself instead of a persistent agent definition.
- If provisioning a *second* CE node later, remember Step 1's HA setting
  is locked at Site creation — decide HA-enabled vs. disabled up front if
  these nodes are meant to cluster.

