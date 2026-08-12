# ce_proxmox_autoprovision

Claude Code-driven provisioning for F5 Distributed Cloud CE (SMSv2) nodes on
Proxmox (KVM). 

## Structure

- CLAUDE.md - auto-loaded context: rules, scope, target host
- .mcp.json - xc-mcp registration (token via env var, pending tool verification)
- .claude/agents/f5-ce-provisioner.md - subagent, executes provisioning steps 4-9
- docs/f5-ce-proxmox-setup.md - full reference procedure, steps 1-9

## Usage

1. In the F5 Console, do steps 1-3 yourself: create the Secure Mesh Site,
   generate the node token (valid 24h), download the qcow2 to the Proxmox
   host. Full detail in docs/f5-ce-proxmox-setup.md.
2. Open Claude Code in this repo.
3. Invoke the subagent with your inputs:

   Use f5-ce-provisioner: VM ID 106, name ce-tertiary, SLO IP 10.0.0.246/24,
   gateway 10.0.0.1, qcow2 at /var/lib/vz/template/iso/f5xc-ce-1.32.1.qcow2,
   token <paste>

4. The agent runs steps 4-9 one command at a time, gates on the Step 8 config
   check before booting, and reminds you to rotate the token once
   registration succeeds in the F5 Console.

## Security notes

- The node token is a live 24h credential. It will appear in this Claude
  Code session's context/logs regardless of how it's provided — plan to
  rotate it in the F5 Console immediately after successful registration.
- SSH access to the Proxmox host (root, key-based) must be granted manually
  by adding the agent-generated public key to authorized_keys — the agent
  will never request or accept a private key/password.
- Nothing in this repo should ever contain a filled-in token, real SLO IP
  tied to a sensitive deployment, or private key material.

## Status

Bootstrap — CLAUDE.md and subagent added. xc-mcp integration pending tool
verification. Not yet run against a live provisioning task in this repo.
