# ADR 0007: Explicit Disk Sizing Required for Every Cloned VM Volume

## Status
Accepted

## Context
Every VM in this project clones its disk from the same base Ubuntu
24.04 cloud image via a `libvirt_volume` with `base_volume_id` set, but
no `size` argument. Without one, the cloned volume silently inherits the
base image's own (small) virtual size — about **2.4GB** — regardless of
what the VM is actually meant to do.

This surfaced twice, in two different ways:
- The app VM hit `No space left on device` mid-`rustup` install — the
  Rust toolchain plus a `cargo build --release` dependency tree needs
  far more than 2.4GB
- The router VM was quietly sitting at 93% disk usage (`2.2G` used of
  `2.4G`) from routine package installs and log accumulation
  (fail2ban, journald, unattended-upgrades), never having errored
  outright but heading toward the same wall

Neither VM's Terraform ever declared a disk size on purpose — it was an
absence, not a considered default, and would have kept silently
recurring on every future VM in this project (the DB VM, at the time
this was found, was still on the same undersized default too).

## Decision
Every VM's disk-cloning `libvirt_volume` resource now sets `size`
explicitly, in bytes, via a dedicated Terraform variable
(`app_disk_size_gb`, `db_disk_size_gb`, `router_disk_size_gb`) rather
than relying on the base image's implicit size. Chosen defaults:
- **20GB** for VMs that install substantial software or accumulate
  persistent data (the app VM's Rust toolchain/build artifacts, the DB
  VM's Postgres data/WAL)
- **10GB** for lighter-weight VMs (the router — no heavy installs, just
  headroom for logs over its lifetime)

Ubuntu cloud images auto-grow their root partition to fill whatever
disk they're given (cloud-init's `growpart`/`resizefs` modules, on by
default) — so declaring a larger volume size is sufficient on its own;
no additional cloud-init configuration is needed to make the resize
actually happen.

## Consequences
- Pro: removes a class of failure that presents as confusing,
  seemingly-unrelated errors (a missing linker, a stalled `apt install`)
  rather than an obvious "disk full" message pointing at the real cause
- Pro: cheap fix, no ongoing cost — local disk is plentiful; there's no
  real reason to have shipped with the tiny default in the first place
- Con: every new VM added to this project needs someone to consciously
  pick a disk size rather than it being safe to omit — worth carrying
  forward as a checklist item for any future VM resource
- Con: fixing this on already-running VMs required destroying and
  recreating them (Terraform can't grow a volume a VM is actively using
  without a reboot at minimum, and the partition inside needs
  cloud-init's growpart to run again, which only happens automatically
  on a genuinely fresh boot) — real time cost in the moment, but
  necessary to keep the Terraform source of truth authoritative

## Alternatives considered
- **A single shared disk-size variable across all VM types**: simpler,
  but wastes disk for VMs that don't need it and under-provisions ones
  that do — rejected in favor of per-VM-type variables with sensible
  defaults, still overridable per environment
- **Resize the running VMs' disks live instead of destroying/recreating**:
  possible via `virsh blockresize` + manual `growpart`/`resize2fs`
  commands, but more error-prone and doesn't verify the Terraform source
  of truth is actually correct — rejected in favor of the same
  destroy/recreate discipline already used for the IP-collision fix
  (ADR 0006)
