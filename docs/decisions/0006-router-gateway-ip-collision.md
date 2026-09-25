# ADR 0006: Router Gateway IPs Moved Off .1 to Avoid Colliding With
Libvirt's Own Bridge Address

## Status
Accepted

## Context
The router VM's `private-net` and `data-net` interfaces were assigned
`10.0.2.1` and `10.0.3.1` — the intuitive "gateway is dot-one" choice.
Unknown at the time: **libvirt automatically assigns that same `.1`
address to its own bridge interface on the host** for any network
defined with an `addresses` CIDR (confirmed via `ip addr show virbr5`/
`virbr6` on the host, both showing `10.0.2.1`/`10.0.3.1`). Two different
machines — the host and the router VM — were claiming the identical IP
on the same L2 segment the entire time.

This went unnoticed through the NAT-proof (Phase 1) and DB-reachability
(`nc -zv`, a bare TCP handshake) tests — those either didn't route
through the conflicting address in a way that mattered, or got lucky
with ARP resolution. It surfaced as a real symptom once the app VM tried
a full request/response cycle to Postgres through the router as an
actual gateway hop (`GET /visits` hanging indefinitely from outside the
app VM, while working instantly from `localhost` on the app VM itself).

## Decision
Router's `private-net` and `data-net` addresses moved to `.254`
(`10.0.2.254`, `10.0.3.254`), leaving `public-net`'s `10.0.1.10`
unchanged (that network's `.1` is claimed by libvirt's bridge too, but
the router's `.10` never collided there in the first place — this bug
was specific to the two networks where the router's address happened to
match libvirt's reserved one). Every affected VM's default-route
`via:` was updated to match. Going forward, **no VM in this project
should ever be assigned `.1` on a libvirt-managed network** — treat it
as reserved by libvirt itself, not available for use, the same way
`.0` (network address) and `.255` (broadcast) already are.

## Consequences
- Pro: eliminates a class of bug that is silent and genuinely hard to
  diagnose — no error message anywhere pointed at this; it presented as
  an app-level "hang," which could easily have been mistaken for a code
  bug or a Postgres connection-pool issue instead of an addressing
  collision
- Pro: cheap to prevent going forward — just a naming convention
  (`.254`, or any address that isn't `.1`) for every future VM's
  gateway/router-facing address in this project
- Con: required rebuilding the router, DB, and app VMs rather than a
  live-only fix, since the goal is the Terraform/cloud-init source of
  truth being correct, not just the running state — consistent with
  this project's IaC discipline, but real time cost in the moment

## Alternatives considered
- **Live-patch the running VMs only (netplan edit + apply, no rebuild)**:
  rejected — would leave the actual bug (wrong address baked into the
  cloud-init templates) unfixed in the repos; the next `terraform
  destroy`/`apply` of any of these VMs would silently reintroduce the
  exact same conflict
- **Keep `.1` and instead change libvirt's own bridge address**: not
  practical — libvirt derives its bridge address directly from the
  network's declared `addresses` CIDR; working around that would mean
  fighting the provider's own conventions instead of just choosing an
  address it doesn't already claim
