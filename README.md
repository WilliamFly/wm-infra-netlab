# wm-infra-netlab

A multi-phase, hands-on lab for learning production-grade network
segmentation, CI/CD, and zero-downtime deployment — built locally first
(free, fast iteration), then ported to AWS once the architecture is proven.

This repo is the **orchestration hub**: architecture docs, ADRs (the
reasoning behind every major decision), and links to the component repos
that actually implement each piece. It does not contain application code
or infrastructure code itself — see [Repos](#repos) below.

## Why this exists

Understanding *why* a given DevOps/security practice is used — not just
copying a checklist — for designing infrastructure as a DevOps engineer.
Each phase is a real, working deliverable, and every non-trivial decision
is documented as an ADR so the reasoning is preserved, not just the result.

## Architecture (high level)

Target end-state — not all of this exists yet; see Phases below for
what's actually built so far:

```
                    [Reverse Proxy / LB]
                    /        \
              [App: Blue]  [App: Green]
                    \        /
                  [Database]
```

Three-tier network (public / private / data), a router VM doing NAT +
firewalling between them, mirroring how an AWS VPC routes traffic between
subnets — see [ADR 0001](docs/decisions/0001-network-segmentation.md) for
the full reasoning.

Full diagram and component breakdown: [`docs/architecture.md`](docs/architecture.md)
*(coming as each phase is built)*

## Phases

| Phase | Description | Status |
|---|---|---|
| 1 | Network foundation — libvirt 3-tier network + router VM (Terraform + Ansible) | **Complete** — segmentation, NAT, and forwarding verified end-to-end with real traffic |
| 2 | Rust app — VM path & Docker path | VM path **complete** — app deployed and verified end-to-end (`/`, `/health`, `/visits` all working through the router). Docker path not started |
| 3 | Node app — VM path & Docker path | Not started |
| 4 | CI/CD pipeline (build, migrate, pull-based deploy) | Not started |
| 5 | NIDS / packet capture layer | Not started |
| 6 | Port to AWS (VPC, remote state, IAM, ALB) | Not started |

## Repos

| Repo | Purpose |
|---|---|
| `wm-infra-netlab` | This repo — docs, ADRs, orchestration |
| [`wm-infra-netlab-network-foundation`](https://github.com/WilliamFly/wm-infra-netlab-network-foundation) | Phase 1 — Terraform + libvirt networks, router VM |
| [`wm-infra-netlab-harden-baseline`](https://github.com/WilliamFly/wm-infra-netlab-harden-baseline) | Shared Ansible role — SSH/firewall/fail2ban hardening |
| [`wm-infra-netlab-db`](https://github.com/WilliamFly/wm-infra-netlab-db) | Shared Postgres VM — single owner, apps connect by IP |
| [`wm-infra-netlab-app-rust`](https://github.com/WilliamFly/wm-infra-netlab-app-rust) | Rust app (VM path + Docker path) |
| `wm-infra-netlab-app-node` | Node app (VM path + Docker path) |

*(Links added as each repo is created.)*

## Decisions

All architecture decisions are recorded as ADRs in
[`docs/decisions/`](docs/decisions/), numbered sequentially. Decisions are
never edited after acceptance — if a decision changes, a new ADR supersedes
the old one and both are updated to reflect that.

- [0001 — Network Segmentation Approach](docs/decisions/0001-network-segmentation.md)
- [0002 — Single Multi-Homed Router VM](docs/decisions/0002-single-router-vm.md)
- [0003 — Shared Roles Referenced via Git, Not Copied](docs/decisions/0003-shared-roles-via-git.md)
- [0004 — Shared Stateful Resources Get Their Own Repo](docs/decisions/0004-shared-db-own-repo.md)
- [0005 — Isolated-Tier VMs Can't Live-Install Packages](docs/decisions/0005-isolated-tier-provisioning.md)
- [0006 — Router Gateway IPs Moved Off .1 to Avoid Libvirt Bridge Collision](docs/decisions/0006-router-gateway-ip-collision.md)
- [0007 — Explicit Disk Sizing Required for Every Cloned VM Volume](docs/decisions/0007-explicit-disk-sizing.md)

## Running this yourself

Clone order matters — later repos depend on earlier ones being applied
first:

1. `wm-infra-netlab-network-foundation` — networks + router. Apply this
   first; nothing else works without it.
2. `wm-infra-netlab-harden-baseline` — nothing to clone/apply directly;
   pulled automatically via `ansible-galaxy` by repos that need it.
3. `wm-infra-netlab-db` — the shared database. Needs step 1 applied and
   reachable first. See that repo's README for a manual provisioning
   step currently required (ADR 0005).
4. `wm-infra-netlab-app-rust` — needs steps 1 and 3 applied first.

Each repo's own README has exact `terraform apply` / `ansible-playbook`
instructions for that piece.

## Security note

This is a learning/portfolio lab, not a hardened production system. It
demonstrates security-relevant patterns (network segmentation, least
privilege, secrets handling) but has not been audited and should not be
used to host real sensitive data or traffic.
