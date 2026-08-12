# ce_proxmox_autoprovision — prompt-only branch

Claude Code-driven provisioning for F5 Distributed Cloud CE (SMSv2) nodes
on Proxmox (KVM), using a single copy-paste prompt — no subagent, no
auto-loaded project context, no MCP config. Paste the prompt into any
Claude Code session and it works, without this repo needing to be your
current project directory.

**This is one of three approaches in this repo.** See the `main` branch
for the subagent-based version (persistent rules, invoke by name) and the
`fully-auto` branch for the (not-yet-built) plan to automate the F5
Console steps too. This branch is the simplest of the three — good for a
one-off run, or if you don't want to set up `CLAUDE.md`/subagent files at
all.

---

## Structure

- `docs/f5-ce-provision-prompt-only.md` — the actual prompt template, plus
  usage instructions and known pitfalls
- `docs/f5-ce-proxmox-setup.md` — full reference procedure, steps 1-9,
  same content as `main`'s copy (both branches document the same
  underlying `qm`/cloud-init process)

No `.claude/`, no `CLAUDE.md`, no `.mcp.json` on this branch — deliberately.
Everything the prompt needs is self-contained in the prompt text itself.

---

## Usage

1. In the F5 Console, do steps 1-3 yourself: create the Secure Mesh Site,
   generate the node token (valid 24h), download the qcow2 to the Proxmox
   host. Full detail in `docs/f5-ce-proxmox-setup.md`.
2. Open `docs/f5-ce-provision-prompt-only.md`, copy the prompt template,
   fill in your actual values (Proxmox host, VM ID/name, SLO IP/gateway,
   qcow2 path, token) in a scratch buffer — not saved to disk.
3. Paste the filled-in prompt into any Claude Code session.
4. Claude Code runs steps 4-9 one command at a time, gates on the Step 8
   config check before booting, and reminds you to rotate the token once
   registration succeeds in the F5 Console.

## Security notes

- The node token is a live 24h credential. It will appear in whatever
  Claude Code session you paste it into, regardless of how it's provided —
  plan to rotate it in the F5 Console immediately after successful
  registration.
- SSH access to the Proxmox host (root, key-based) must be granted
  manually by adding a generated public key to `authorized_keys` — never
  provide a private key or root password in any session.
- Never commit a filled-in copy of the prompt template with a real token,
  IP, or VM name if that information is sensitive to you.

## Status

Working — this approach successfully provisioned a live CE node. Kept here
as the minimal, no-setup option alongside the more structured approaches
on the other branches.

