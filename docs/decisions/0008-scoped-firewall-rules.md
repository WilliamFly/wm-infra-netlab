# 0008. Scope firewall rules to a source subnet in harden-baseline

## Status
Accepted

## Context
`harden-baseline`'s firewall role opens every port in `harden_ufw_allowed_ports`
to any source ("Anywhere"). That's fine for the router (SSH only, meant to be
reachable from the laptop's public IP), but it's wrong for the DB VM (port 5432
should only ever be reached from private-net, 10.0.2.0/24) and the app VM
(port 8080 shouldn't be open to the whole internet either). Applying the role
as-is to either VM would either leave those ports wide open, or (with the
default deny-incoming policy) break the Postgres/app access that's currently
working.

## Decision
Add an optional `from` key to each `harden_ufw_allowed_ports` entry. When set,
it's passed as `src` to the `community.general.ufw` module, scoping that rule
to a CIDR or host. When omitted, the rule is open to any source — identical to
the role's behavior before this change, so the router's existing config needed
no edits.

## Alternatives considered
- **A separate "scoped ports" variable/list, kept apart from
  `harden_ufw_allowed_ports`.** Rejected — two lists to reason about instead
  of one, no real benefit.
- **Hardcode subnet scoping into a role-specific task (e.g. a "db" role) instead
  of extending the shared role.** Rejected — pulls firewall logic out of the
  one place it's meant to live, undermining the whole point of a shared,
  reusable `harden-baseline`.
- **Leave the role as open-to-anywhere and rely on the router's own forwarding
  rules as the only isolation boundary.** Rejected — defense in depth; the
  data-net VMs shouldn't depend solely on the router being configured
  correctly to stay unreachable from the wrong subnet.
