# Architecture Decision Records

## ADR-001: Host networking for AdGuard Home only

**Context:** Both AdGuard Home and UniFi Network Application need to bind well-known
ports. The original plan proposed `network_mode: host` for both.

**Decision:** Use `network_mode: host` for AdGuard Home only. UniFi Network Application
and MongoDB share a Docker bridge network (`unifi-net`), with UniFi's ports mapped
explicitly.

**Rationale:**
- AdGuard Home must own port 53 on the host; host networking is the simplest way to
  achieve this and avoids hairpin NAT issues with DNS.
- UniFi needs an internal connection to MongoDB. With host networking, Docker DNS
  (container name resolution) is unavailable, so UniFi would have to connect to
  `127.0.0.1:27017`. A bridge network lets UniFi reach MongoDB via the service name
  `unifi-db`, which is the documented and tested path for the linuxserver image.
- MongoDB is not exposed to the LAN at all — only reachable from within `unifi-net`.

---

## ADR-002: Reuse the N150's IP address for the Pi 5

**Context:** UniFi APs discover their controller via the inform URL
(`http://<controller-ip>:8080/inform`). Changing the controller IP requires either
SSH into every AP to run `set-inform`, or DNS/DHCP tricks.

**Decision:** Assign the Pi 5 the same static IP the N150 currently holds.

**Rationale:**
- APs will reconnect to the new controller automatically after it boots.
- AdGuard Home also lives on this IP; Firewalla's DNS forwarder just needs one IP.
- The N150 should be powered off (or given a new IP) before the Pi 5 boots with the
  reused address.

---

## ADR-003: AdGuard admin UI on port 8081

**Context:** AdGuard Home defaults its admin UI to port 80. UniFi's HTTP portal redirect
also uses port 8880/8843, and future services may want port 80.

**Decision:** Configure `bind_host` and `bind_port` in `AdGuardHome.yaml` so the admin
UI listens on port 8081.

**Rationale:**
- Avoids port conflicts today and leaves port 80 free for a potential reverse proxy later.
- Port 3000 is used by AdGuard only during its first-run wizard, then the configured
  port takes over.

---

## ADR-004: Pinned image tags, no Watchtower

**Context:** Automated container updates (Watchtower, Ouroboros) can cause breaking
changes, especially with UniFi major version bumps that require MongoDB migrations.

**Decision:** Pin image tags in `docker-compose.yml`. Upgrades are manual:
bump tag, commit, `docker compose pull && docker compose up -d`.

**Rationale:**
- UniFi version upgrades have historically required manual DB backups beforehand.
- Pinned tags make the deployed state reproducible from git history.
- A future Watchtower or Renovate integration can be added once upgrade confidence
  is established.

---

## ADR-005: Single Pi 5 (no USB-Ethernet dual NIC)

**Context:** Running two network services on one host could warrant separate NICs to
isolate traffic or provide a management network.

**Decision:** Use a single NIC with a single IP.

**Rationale:**
- Both services are LAN-only and low-throughput.
- The Pi 5's onboard Gigabit Ethernet is more than sufficient.
- Adds no hardware cost or cabling complexity.
- A USB-Ethernet adapter can be added later if VLAN trunking or a management network
  is needed.

---

## ADR-006: AdGuardHome.yaml gitignored, template committed instead

**Context:** The plan calls for "declarative config committed to git" for reproducibility.
However, the live `AdGuardHome.yaml` contains a bcrypt-hashed admin password and may
contain sensitive local DNS rewrites (internal hostnames, IP addresses) or per-client
configuration with device names.

**Decision:** Gitignore the live `AdGuardHome.yaml`. Commit a sanitized
`AdGuardHome.yaml.example` template with placeholder values.

**Rationale:**
- Keeps secrets out of git history even for a private repo (defense in depth).
- The template captures all structural decisions (port 8081, upstream DNS provider,
  cache settings, filter lists) so the repo remains reproducible — copy the template,
  fill in secrets, and start.
- AdGuard's first-run wizard is a valid alternative to the template, so the config
  is not strictly required in git for a working deploy.
- If full config-as-code is desired later, an Ansible vault or SOPS-encrypted file
  can replace the template approach.

---

## ADR-007: UniFi services gated behind a Compose profile

**Context:** The two services don't have to run on the same host. UniFi may run
elsewhere (e.g. a more powerful x86 box or UniFi OS Server), while the Pi handles only
DNS. Forcing MongoDB + UniFi to start on every `docker compose up` wastes resources and
adds failure surface when only AdGuard Home is wanted.

**Decision:** Put `unifi-db` and `unifi-network-application` behind a `unifi` Compose
profile. Default `docker compose up -d` starts AdGuard Home only; the full stack
requires `docker compose --profile unifi up -d`.

**Rationale:**
- AdGuard-only is the common minimal deployment; it should be the zero-flag default.
- The UniFi service definitions, config, and docs stay in the repo for later use —
  nothing is removed, just made opt-in.
- The day-2 backup script checks whether `unifi-db` is running before attempting a
  `mongodump`, so it works correctly in either mode.
- Lets a single repo serve both "DNS-only Pi" and "full consolidation" topologies.
