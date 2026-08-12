# CLAUDE.md — ce_proxmox_autoprovision

Single-purpose repo: provisioning F5 Distributed Cloud CE (SMSv2) nodes on
Proxmox via Claude Code. This file is loaded automatically into every
session in this repo.

If you're using this repo as a template for your own environment, treat
every `<PLACEHOLDER>` below as something to fill in with your own values —
nothing in this file should reference a specific person's network layout.

---

## Scope

This repo provisions F5 XC CE nodes on **one** Proxmox host. It does not
touch, and has no business touching:
- Any other VM or service not directly involved in CE provisioning
- Any host not explicitly named as the provisioning target for a given task
- Anything outside `qm` commands, cloud-init snippets, and F5 Console/API
  interactions for CE nodes

If you're running this alongside other infrastructure-as-code repos, keep
broader network/service context in those repos — this one should stay
narrowly scoped to CE provisioning so its instructions stay easy to follow
and audit.

## Target Host

| Host | IP | Role |
|---|---|---|
| Proxmox host | `<PROXMOX_HOST_IP>` | Hypervisor — root SSH, key-based only |

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
- **Check vCPU headroom, not just RAM/disk.** A host can show comfortable
  free RAM while still being oversubscribed on physical cores if other VMs
  are already running — check both before provisioning.
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
  the F5 Console once registration succeeds. Note: a heredoc avoids shell
  history and printed response text, but the tool-call transcript itself
  still carries the token — token rotation after use is the actual
  safeguard, not just the heredoc technique.

---

## Reference Docs

- `docs/f5-ce-proxmox-setup.md` — full provisioning procedure, steps 1-9
- `.claude/agents/f5-ce-provisioner.md` — subagent that executes steps 4-9
- `docs/provisioning-caveats.md` — real-world gotchas discovered running
  this procedure, not yet folded into the main doc

## Current State — Provisioned Nodes

Track nodes you've provisioned through this repo here. Example row shown —
replace with your own, delete the example once you have real entries:

| VM ID | Name | SLO IP | Provisioned | Status |
|---|---|---|---|---|
| `<example>` | `<example>` | `<example>` | `<YYYY-MM-DD>` | `<example>` |

See `docs/provisioning-caveats.md` for gotchas hit during past runs.

