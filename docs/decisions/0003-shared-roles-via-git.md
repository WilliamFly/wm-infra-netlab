# ADR 0003: Shared Roles Referenced via Git, Not Copied

## Status
Accepted

## Context
`harden-baseline` (SSH hardening, ufw baseline, fail2ban,
unattended-upgrades, user mgmt) is needed by more than one component in
this project — first the router VM, and later the app-tier VMs in
Phase 2/3. It was initially copied directly into
`wm-infra-netlab-network-foundation/ansible/roles/harden-baseline/` to
keep moving quickly while building the router.

That copy was a mistake worth correcting immediately rather than
carrying forward: the moment a second consumer (e.g. the Rust/Node app
VMs) needed the same role, there would be two independent copies with no
link between them. A fix or improvement made in one would silently not
exist in the other — the same configuration-drift problem this project's
GitOps discipline exists to prevent, just at the role level instead of
the server level.

## Decision
Extract `harden-baseline` into its own standalone repo,
[wm-infra-netlab-harden-baseline](https://github.com/WilliamFly/wm-infra-netlab-harden-baseline),
laid out at the repo root the way Ansible expects a role to look
(`defaults/`, `tasks/`, `handlers/`, `meta/`, `templates/` — no extra
nesting). Every repo that needs it declares it in its own
`ansible/requirements.yml` as a git-sourced role:

```yaml
roles:
  - src: https://github.com/WilliamFly/wm-infra-netlab-harden-baseline
    name: harden-baseline
```

`ansible-galaxy install -r requirements.yml` fetches it fresh into that
repo's local `roles/harden-baseline/`, which is gitignored — never
committed, always pulled from the one canonical source. The `router`
role depends on it via `meta/main.yml`, rather than the playbook
listing both roles directly, so there's one obvious place declaring
that relationship.

## Consequences
- Pro: one canonical source of truth — a fix or improvement to SSH
  hardening, ufw defaults, fail2ban config, etc. is made once and
  every consuming repo gets it on their next `ansible-galaxy install`
- Pro: genuinely reusable beyond this project — the exact pitch this
  project was building toward ("clone a repo, stand up real infra in
  minutes") extends to reusing just this one role elsewhere, including
  at a future job, without dragging the rest of this project along
- Con: no version pin yet — every install currently tracks the
  default branch, so a breaking change pushed to that repo would
  propagate to every consumer on their next install. Acceptable for
  now (solo project, low change frequency); once this role stabilizes,
  add `version: <git-tag>` to the `requirements.yml` entries and start
  tagging releases
- Con: one more moving part than a plain local copy — requires
  `ansible-galaxy install` as an explicit step before a playbook will
  run, rather than roles being present the instant the repo is cloned

## Alternatives considered
- **Keep the local copy in each consuming repo**: simplest short-term,
  but reintroduces the exact drift problem described in Context —
  rejected
- **Git submodule pointing at the harden-baseline repo**: gives a
  pinned, reproducible reference, but adds submodule-specific git
  mechanics (`--recurse-submodules`, remembering to bump the pointer
  commit) on top of everything else already being learned in this
  project — rejected for now as complexity that doesn't pay for itself
  at this project's current size; worth revisiting if the number of
  shared components grows
- **Ansible Galaxy (the public hosted registry) instead of a plain git
  URL**: would work, but requires publishing the role publicly to
  Galaxy's index for something that's really just personal/portfolio
  infrastructure — a private git repo referenced directly is a better
  fit for this use case
