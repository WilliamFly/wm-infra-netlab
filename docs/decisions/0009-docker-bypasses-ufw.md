# 0009. Docker-published ports bypass ufw; enforce via DOCKER-USER chain

## Status
Accepted

## Context
While verifying the Rust app's Docker path (port 8081, published alongside
the existing VM-native deploy on 8080), we tested whether ufw's source-subnet
scoping (`from: 10.0.2.0/24`, set via harden-baseline) actually applied to
both. It did not. A curl sourced from 10.0.3.1 (outside the allowed range)
was correctly blocked on 8080 but succeeded on 8081.

Root cause: Docker manages published container ports through its own
DOCKER-USER and DOCKER iptables chains, inserted into the FORWARD chain.
ufw's port rules govern the INPUT chain — a published Docker port is reached
via NAT + forwarding, not INPUT, so ufw never gets a say. DOCKER-USER was
empty by default, so nothing restricted the forwarded traffic at all.

## Decision
Extend the shared `docker-host` role with an explicit `docker_published_ports`
list. For each entry, insert a RETURN rule (allow) into DOCKER-USER scoped to
the given source, followed by a DROP rule for that port from any other
source. This directly closes the gap ufw cannot reach, and is installed
alongside Docker itself so any future consumer of this role (e.g. the Node
app's Docker path) gets the same protection by declaring its own
`docker_published_ports`.

## Alternatives considered
- **Disable Docker's iptables management (`"iptables": false` in
  daemon.json) and hand-manage all NAT/forwarding rules.** Rejected — loses
  Docker's automatic port-publishing entirely, a much larger and more
  error-prone undertaking than adding targeted DOCKER-USER rules.
- **Bind published ports to a specific host IP instead of 0.0.0.0
  (`"10.0.2.20:8081:8080"`).** Rejected on its own — restricts which local
  IP the port listens on, not which remote source can reach it; doesn't
  solve the actual problem (confirmed by testing — the exposure isn't about
  which host IP, it's about DOCKER-USER not filtering by source at all).
- **Pull in the third-party `ufw-docker` script.** Rejected — the point of
  this project is understanding the mechanism, not depending on an external
  tool; the fix here is simple enough to own directly.
