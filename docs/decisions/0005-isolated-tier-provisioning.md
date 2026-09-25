# ADR 0005: Isolated-Tier VMs Can't Live-Install Packages — Temporary
Egress Workaround, Packer Planned as the Real Fix

## Status
Accepted (workaround) — Packer migration tracked as follow-up, not yet
implemented

## Context
Provisioning the Postgres VM on `data-net` via cloud-init's
`packages:`/`runcmd:` steps failed: `apt` couldn't reach the internet,
because `data-net` has no forwarding rule to `public-net` at all (only
`private→public` and `private→data` exist — see ADR 0001). This is the
isolation working exactly as designed, but it means **any** VM on an
isolated tier can't live-install packages during first boot — a real
conflict between "the tier has no internet access" and "cloud-init's
default provisioning model assumes it does."

## Decision
Short-term: the router's Ansible role gained a toggle,
`router_temp_allow_data_egress` (default `false`), that adds/removes a
single, clearly-labeled, explicitly-scoped `ufw route allow` rule
(`data-net → public-net`) on demand. The workflow for provisioning a new
data-net VM is: flip the toggle true, run the router playbook, finish
package installation/config manually over SSH (cloud-init only gets one
attempt at boot, it won't retry once the network's fixed), flip the
toggle back to false, run the router playbook again to close the hole.
Both directions are idempotent — the rule is guaranteed present when
true and guaranteed absent when false, never something manually
tracked.

Long-term (not yet built): bake required packages (Postgres, etc.)
directly into the VM image with **Packer**, so `data-net` VMs never need
live internet access at all — cloud-init then only does lightweight
config (creating the app database/role, editing `pg_hba.conf`), not a
package install. This was already the planned direction from the
`infra-lab` project's original README and is consistent with this
project's broader immutable-infrastructure principle (nothing should be
installed or resolved on a target at deploy time — see the CI/CD
discussion this project is built around). Once built, this ADR's
temporary-egress mechanism becomes dead code that can be removed
entirely, not standing infrastructure that needs to keep existing
alongside the "real" fix.

## Consequences
- Pro (short-term): unblocks provisioning immediately without building a
  full Packer pipeline before Phase 2 can proceed
- Pro (short-term): the temporary rule is genuinely temporary — the
  toggle mechanism makes it impossible to accidentally leave the
  isolated tier with standing internet access
- Con (short-term): provisioning a new data-net VM currently requires a
  manual, multi-step dance (flip toggle, re-run playbook, SSH in and
  finish setup by hand, flip toggle back, re-run playbook) rather than a
  single `terraform apply` — this is real friction, and the whole reason
  this ADR treats Packer as the actual answer rather than something to
  polish this workaround into
- Pro (long-term, once Packer lands): removes the isolated-tier
  provisioning problem categorically, for every future data-net (and
  potentially private-net) VM, not just Postgres

## Alternatives considered
- **Give data-net standing internet access**: rejected outright — this
  is exactly the blast-radius problem ADR 0001 exists to prevent; a
  compromised DB tier with outbound internet access is a real
  exfiltration path
- **Internal apt mirror/proxy reachable from data-net without giving it
  general internet access**: a legitimate real-world pattern (companies
  with genuinely air-gapped networks often do exactly this), but more
  infrastructure than this project needs at its current size — Packer
  solves the same problem more simply here, since there's no requirement
  for data-net VMs to install packages *after* first boot, only at
  creation time
- **Leave the temporary-egress mechanism as the permanent answer**:
  rejected — it works, but it's a process people have to remember to
  follow correctly every time a new isolated-tier VM is built, which is
  exactly the kind of manual step this project's GitOps/IaC discipline
  is meant to eliminate
