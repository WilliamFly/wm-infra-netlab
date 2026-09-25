# ADR 0004: Shared Stateful Resources Get Their Own Repo

## Status
Accepted

## Context
The Postgres database is shared by more than one app (the Rust app now,
the Node app later). It was initially going to live inside
`wm-infra-netlab-app-rust`'s Terraform, following that repo's build
naturally — but a shared, stateful resource owned by only one of its
consumers creates a real conflict, not just an organizational
inconvenience: if `wm-infra-netlab-app-node` later needed the same
database, it would either have to duplicate the DB's Terraform (risking
two states fighting over one VM/IP) or connect to a resource it has no
ownership of and no ability to safely `destroy`/rebuild without
affecting the other app.

## Decision
Extract the database into its own repo,
[wm-infra-netlab-db](https://github.com/WilliamFly/wm-infra-netlab-db).
It is the single owner of the Postgres VM — the only repo that runs
`terraform apply`/`destroy` against it. Every consuming app repo
connects by IP (`db_host`, currently `10.0.3.20`) via plain variables,
never by referencing `wm-infra-netlab-db`'s Terraform state.

## Consequences
- Pro: exactly one owner for a shared resource — no risk of two states
  both believing they manage the same VM
- Pro: an app repo can be destroyed and rebuilt freely without any risk
  to the shared database or other apps depending on it
- Con: connection details (IP, credentials) have to be kept in sync
  manually across repos for now — each consuming repo's
  `terraform.tfvars` needs the same `db_app_password` as
  `wm-infra-netlab-db`'s. Acceptable at this project's size; a secrets
  manager (Vault, AWS Secrets Manager) would remove this manual sync in
  a larger/production setup — tracked as future work once Phase 6 (AWS)
  is underway

## Alternatives considered
- **Keep the DB in `app-rust`, have `app-node` connect to it later**:
  rejected — same "which repo actually owns this" ambiguity the
  `harden-baseline` extraction (ADR 0003) already solved once for a
  different kind of shared resource
- **A single `wm-infra-netlab-shared` repo for all cross-app
  infrastructure**: considered, but too vague a scope for now — "shared"
  isn't a real category on its own; naming the repo for what it actually
  is (`db`) stays clearer as long as there's only one such resource
