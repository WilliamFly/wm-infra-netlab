# 0009. Docker-published ports bypass ufw; enforce via DOCKER-USER chain

## Status
Accepted

## Context
While verifying the Rust app's Docker path (port 8081, published alongside
the existing VM-native deploy on 8080), we tested whether ufw's source-subnet
scoping (`from: 10.0.2.0/24`, via harden-baseline) actually applied to a
Docker-published port the same way it applies to a plain systemd process.
It did not — a request sourced from 10.0.3.1 (outside the allowed range)
was correctly blocked on 8080 but succeeded on 8081.

Root cause: Docker manages published container ports through its own
DOCKER-USER/DOCKER iptables chains, inserted into the FORWARD chain ahead
of ufw. ufw's port rules govern the INPUT chain; a published Docker port
is reached via NAT + forwarding, so ufw never gets a say over it.

Fixing this correctly took three iterations, each surfacing a real,
non-obvious netfilter interaction:

1. **Matching the wrong port.** An initial rule matched `--dport 8081`
   directly, but by the time a packet reaches FORWARD, Docker's PREROUTING
   DNAT has already rewritten the destination port to the container's
   internal port (8080). The rule never matched anything. Fix: match the
   pre-NAT port via `-m conntrack --ctorigdstport 8081` instead.

2. **RETURN vs ACCEPT.** `RETURN` only stops processing the DOCKER-USER
   chain and falls through to the rest of FORWARD — it depends on
   something later accepting the packet. Docker's own chains only
   explicitly accept container-*originated* traffic, not inbound traffic
   to a published port, so this relied on FORWARD's default policy being
   ACCEPT. That was true until `ufw` was reinstalled (see below) and set
   FORWARD's default policy to DROP, at which point traffic that
   correctly RETURNed from DOCKER-USER had nothing left to accept it. Fix:
   use `-j ACCEPT` directly, which is terminal regardless of chain.

3. **Reply packets dropped by our own rule.** A reply packet (e.g. a
   container's SYN-ACK) re-enters DOCKER-USER with the container's raw
   bridge IP (e.g. 172.18.0.2) as its source — Docker's un-DNAT (rewriting
   it back to the host's address) happens later, in POSTROUTING, after
   FORWARD has already been evaluated. A per-port rule matching source
   subnet alone therefore doesn't match legitimate replies, which fell
   through into our own catch-all DROP rule. Fix: accept
   `RELATED,ESTABLISHED` traffic unconditionally, before the per-port
   rules — the same pattern Docker's own DOCKER-CT chain already uses.

A separate, unrelated issue surfaced mid-investigation: installing
`iptables-persistent` (intended to persist the DOCKER-USER rules across
reboots) silently removed the `ufw` package as a side effect — they
conflict on this Ubuntu version. This is why FORWARD's default policy
changed mid-debugging (from ufw's own absence, to ufw being reinstalled
with its stock `DEFAULT_FORWARD_POLICY="DROP"`). Fix: drop
`iptables-persistent` entirely; persist the DOCKER-USER rules instead via
a small systemd service (`docker-user-firewall`) that rebuilds them on
every boot, independent of ufw.

## Decision
The `docker-host` role now installs a `docker_published_ports`-driven
systemd service that, on every boot:
1. Flushes and rebuilds the DOCKER-USER chain
2. Accepts RELATED,ESTABLISHED traffic unconditionally (first rule)
3. For each configured port: ACCEPTs NEW connections matching
   `ctorigdstport` + the configured source CIDR, then DROPs everything
   else for that port

Verified end-to-end with real traffic and `conntrack -L` inspection: both
the disallowed-source case (blocked on both the VM-native and Docker
paths) and the allowed-source case (working on both paths) behave
identically now, closing the gap ufw could never have covered on its own.

## Alternatives considered
- **Disable Docker's iptables management entirely
  (`"iptables": false` in daemon.json) and hand-manage all NAT/forwarding.**
  Rejected — loses Docker's automatic port-publishing, a much larger and
  more error-prone undertaking than targeted DOCKER-USER rules.
- **`iptables-persistent`/`netfilter-persistent` for rule persistence.**
  Rejected — conflicts with and silently removes `ufw` on this Ubuntu
  version, confirmed directly (`dpkg -l` showed ufw as `rc` — removed,
  not purged — after installing it).
- **Bind published ports to a specific host IP instead of `0.0.0.0`.**
  Rejected on its own — restricts which local IP the port listens on, not
  which remote source can reach it; doesn't address the actual gap.
- **Pull in the third-party `ufw-docker` script.** Rejected — the point of
  this project is understanding the mechanism directly; the equivalent
  logic here is simple enough to own and was worth debugging by hand.
