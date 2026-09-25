# ADR 0002: Single Multi-Homed Router VM

## Status
Accepted

## Context
ADR 0001 established a 3-tier network (public/private/data) needing a
routing/NAT/firewall point between tiers. That decision didn't specify
*what* performs that routing — this ADR covers the concrete choice made
while implementing Phase 1.

## Decision
Use **one VM with three network interfaces** (one NIC per tier: public-net,
private-net, data-net), each on a fixed MAC address so cloud-init can
reliably assign static IPs regardless of interface enumeration order. This
single VM performs NAT (public-net ↔ outside) and will enforce inter-tier
firewall rules (private ↔ public, private ↔ data, data isolated except
from private) via nftables, applied in a later step.

The router VM is minimal: 1 vCPU / 1GB RAM, Ubuntu 24.04, provisioned by
Terraform (libvirt provider) with cloud-init handling identity/SSH only —
no routing behavior is baked in at this stage; forwarding + nftables rules
are a separate, later commit so each step is independently verifiable.

## Consequences
- Pro: mirrors a real single-appliance router/firewall (a home router, a
  pfSense box, a cloud NAT gateway) — one thing to reason about, one place
  all inter-tier rules live
- Pro: simpler to provision and reason about than multiple cooperating VMs
  — fewer moving parts while still proving real network segmentation
- Con: single point of failure for the entire lab's networking — if this
  VM is down, all three tiers lose connectivity to each other and the
  internet (acceptable for a learning lab; would need redundancy in real
  production)
- Con: all routing/firewall logic concentrated in one VM's config — a
  misconfiguration here has lab-wide blast radius (mitigated by keeping
  the nftables rules in version control and reviewed via the same GitOps
  discipline as everything else in this project)

## Alternatives considered
- **Separate bastion VM + separate NAT gateway VM**: splits "SSH jump
  host" and "NAT/routing" into two VMs, closer to some real AWS
  architectures (bastion ≠ NAT gateway). Rejected for Phase 1 as
  unnecessary complexity — nothing yet requires a bastion distinct from
  the router itself; can be revisited if a real need for a separate jump
  host emerges (e.g. once the router VM shouldn't also be the general
  SSH entry point)
- **pfSense/OPNsense appliance VM instead of plain Ubuntu + nftables**:
  more production-realistic (GUI, mature NAT/firewall tooling) but hides
  the mechanics behind an abstraction — rejected because the goal here is
  to actually understand IP forwarding and nftables rules by writing them,
  not to click through a firewall GUI
- **Hardware router/dedicated device**: out of scope — this project is
  explicitly about reproducing VPC-style segmentation entirely in
  software/IaC, not physical networking gear
