# CLAUDE.md — ce_proxmox_autoprovision

Single-purpose repo: provisioning F5 Distributed Cloud CE (SMSv2) nodes on
Proxmox via Claude Code. This file is loaded automatically into every
session in this repo.

Parent context (full homelab network map, Bifrost gateway, branch discipline,
etc.) lives in the `envoy_ai_gateway` repo — this repo doesn't duplicate it.
Only F5 CE-relevant facts live here.

---

## Scope

This repo provisions F5 XC CE nodes on **one** Proxmox host. It does not
touch, and has no business touching:
- The Ollama/Bifrost/Envoy VMs (managed in `envoy_ai_gateway`)
- The NUC/ESXi host (10.0.0.30) — off-limits until Phase 5, unrelated to
  this repo regardless
- Anything outside `qm` commands, cloud-init snippets, and F5 Console/API
  interactions for CE nodes

## Target Host

| Host | IP | Role |
|---|---|---|
| Proxmox host | 10.0.0.251 | Hypervisor — root SSH, key-based only |

F5 XC CE nodes provisioned here each get their own SLO IP (see individual
provisioning inputs) — there is no fixed IP for "the CE node."

---

## Non-Negotiable Rules

- **One command at a time.** Show output, wait for confirmation.
- **Ask before destructive commands** — `qm destroy`, `disk unlink --force`,
  overwriting an existing CE node's config.
- **Secrets never in git.** No live F5 node tokens, no filled-in copies of
  templates with real IPs/VM names if sensitive. Only placeholder templates
  are committed.
- **Validate before applying.** Cloud-init YAML gets checked with `python3`
  + `yaml.safe_load` and structural assertions before it's attached or the
  VM is booted.
- **Config gate before boot.** `qm config <vmid>` must match the expected
  block in `docs/f5-ce-proxmox-setup.md` Step 8 before `qm start` runs. SLO
  IP/MAC and HA mode are permanent after registration — this gate is never
  skipped or softened.
- **Diagnose before rewriting.** Read `journalctl` / actual error output
  before proposing a fix.

---

## Access & Credentials

- **Proxmox host root SSH** uses key auth only. If a new key is needed:
  generate it, print the public key, and stop — wait for the human to add
  it to `authorized_keys` out of band (console/IPMI/existing session).
  Never ask for or accept a private key or root password in chat.
- **F5 XC node tokens** are short-lived (24h), single-use for registration.
  Never pass a token as a bare shell argument — write it into the target
  file via SSH heredoc. Never print the token value back in a response.
  Every provisioning task ends with a reminder to regenerate the token in
  the F5 Console once registration succeeds.

---

## Reference Docs

- `docs/f5-ce-proxmox-setup.md` — full provisioning procedure, steps 1-9
- `.claude/agents/f5-ce-provisioner.md` — subagent that executes steps 4-9

## Current State

No CE nodes provisioned via this repo yet as of setup. Update this section
as nodes are added — VM ID, name, SLO IP, provisioned date, status.
