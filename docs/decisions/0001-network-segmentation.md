# ADR 0001: Network Segmentation Approach

## Status
Accepted

## Context
Learning production-grade network segmentation (public/private/data tiers,
NAT, firewalling) without paying for cloud while iterating. Needs to mirror
how an AWS VPC actually routes traffic (route tables per subnet) closely
enough that the concepts transfer directly later, rather than teaching a
simplified version that has to be re-learned.

## Decision
Build a 3-tier libvirt network locally:
- `public-net`  (10.0.1.0/24) — router's internet-facing side; future
  reverse proxy/LB lives here
- `private-net` (10.0.2.0/24) — app tier; no direct route to the internet
- `data-net`    (10.0.3.0/24) — DB tier; most restricted, no direct route
  to the internet, only reachable from private-net

A single router VM has 3 NICs, one per network, and performs NAT + firewall
routing between them. Each virtual network is a separate routed segment
(not a flat bridge), enforced with nftables rules on the router — mirroring
one route table per subnet, the way AWS VPC actually works.

Provisioned via Terraform (libvirt provider) + Ansible (router firewall
config, using the `harden-baseline` role).

## Consequences
- Pro: direct conceptual mapping to AWS VPC subnets/route tables — Phase 6
  (AWS port) becomes "recognize this pattern in managed form," not
  "learn networking from scratch again"
- Pro: proves real segmentation (data tier physically cannot be reached
  except via private tier), not just documentation claiming it
- Con: more setup complexity than 2 NICs or a flat network — 3 virtual
  networks, 3 firewall zones, more to get wrong while learning
- Con: router VM becomes a single point of failure for the whole lab
  (acceptable for a learning environment, would need HA in real prod)

## Alternatives considered
- **2-NIC router, data-net routed through private-net implicitly**:
  simpler, but blurs the "one route table per subnet" AWS analogy —
  rejected, since the whole point is that fidelity
- **Flat network, host-level firewalling only (no libvirt network split)**:
  doesn't actually prove isolation — a misconfigured iptables rule on one
  VM could expose everything with nothing else stopping it — rejected
- **Docker network isolation instead of libvirt networks**: real, but a
  weaker isolation boundary (shared kernel) — this project uses that layer
  separately, one level down, inside the app-tier VM
- **Skip local entirely, build straight on AWS**: rejected — costs money
  to iterate on mistakes, and local-first lets networking fundamentals be
  learned without cloud API/billing friction in the way
