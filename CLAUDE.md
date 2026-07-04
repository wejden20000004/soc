# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository state

The project folder structure exists (compose files, `logstash/`, `traefik/`, `workflows/`,
`windows/`) but most of it is scaffolding, not a working stack. `compose.automation.yml` and
`compose.siem.yml` are populated verbatim from the plan doc's YAML and should be
usable as-is (module names, mem limits, volumes). `compose.misp.yml`, `compose.thehive.yml`,
`compose.traefik.yml`, `logstash/pipeline/suricata.conf`, and `traefik/dynamic/routes.yml`
are stubs with `TODO` markers — the plan explicitly defers these to external sources
(misp-docker upstream, the old Ubuntu VM's TheHive compose file) or leaves the field
mapping/routing undecided. Treat any `TODO`/`image: TODO` line as not-yet-implemented rather
than a working default. There is no build, lint, or test tooling — this is infrastructure,
validated by standing up containers and running the Phase 8 checks, not a test suite.
Full narrative and rationale live in
[soc-in-a-box-docker-migration-plan-windows.md](soc-in-a-box-docker-migration-plan-windows.md).

## What this project is

"SOC in a Box" — migrating a 4-VM security-operations lab (SIEM, MISP, TheHive, automation)
onto a single Windows 11 host, using Docker Desktop (WSL2 backend) for everything that can be
containerized, plus two services that must run natively on Windows because they need direct
access to host-level network/OS resources:

- **Suricata** (native Windows service, via Npcap) — sniffs the VMware `VMnet8` NAT adapter
  that carries the monitored Windows 10 endpoint VM's traffic. Containerized sniffing does not
  work here: Docker Desktop's "host" network is the hidden WSL2 VM, not the Windows host, so a
  containerized Suricata sees nothing from VMware.
- **Filebeat** (native Windows service) — tails Suricata's `eve.json` and ships it to Logstash
  on `localhost:5044`. Must be a native TCP shipper, not a bind-mounted log tail: file-change
  notifications don't propagate reliably across the Windows→WSL2 filesystem boundary.

Everything else runs in Docker Desktop across four planned compose files:

| Compose file | Services |
|---|---|
| `compose.siem.yml` | elasticsearch, logstash, kibana, fleet-server |
| `compose.misp.yml` | misp-core, mariadb, redis, misp-modules |
| `compose.thehive.yml` | thehive, cassandra, es-thehive, minio (+ optional Cortex) |
| `compose.automation.yml` | n8n, ollama |

All containers share a `soc-backend` Docker network. A Traefik container terminates 443 and
routes `kibana/misp/hive/n8n.soc.local` (self-signed wildcard cert), so no other container
publishes ports directly except where an external device must reach it (see below).

Zeek is intentionally dropped (no viable Windows build) — this is a documented, deliberate
trade-off, not a gap to silently fix. Coverage is compensated by Suricata signatures + Sysmon
(Event ID 1/3) endpoint telemetry. Don't reintroduce Zeek into the Windows plan.

## Architectural rules to preserve when adding files

These constraints came from real failure modes on Docker Desktop/WSL2 and should hold for any
compose file, config, or script added to this repo:

- **Named Docker volumes only for databases** (Elasticsearch ×2, Cassandra, MariaDB, MinIO,
  Ollama models). Never bind-mount these to `C:\` paths — the Windows↔WSL2 filesystem boundary
  imposes a 5–10× I/O penalty that makes Elasticsearch/Cassandra unusable. Bind mounts are fine
  only for small read-only configs (Logstash pipelines, Traefik config).
- **Explicit memory limits everywhere**: `.wslconfig` needs `memory=`/`processors=`/`swap=`
  (WSL2 otherwise caps at 50% of host RAM), and every compose service needs `mem_limit:` — see
  the resource budget table in the plan doc for the target split (32 GB host).
- **`kernelCommandLine = "sysctl.vm.max_map_count=262144"`** must be set in `.wslconfig`, or
  both Elasticsearch instances crash-loop after every host reboot.
- **Ports 5044 (Beats) and 8220 (Fleet) are published on the Windows host**, not just internal
  to `soc-backend` — the monitored endpoint VM and native Suricata's Filebeat shipper reach them
  via the host's LAN IP (e.g. `192.168.105.100`), not a container-internal hostname.
- **`host.docker.internal`** is the Docker Desktop hostname for reaching natively-run Windows
  services (e.g. Ollama, if it's kept native instead of containerized) from inside a container.
- Credentials belong in a git-ignored `.env`, with a committed `.env.example` alongside it.

## n8n workflow rewiring

When porting the automation workflows from the old VM-based architecture, the endpoint URLs
change (old Security Onion/VM IPs → Docker service names or `host.docker.internal`); see the
Phase 6 table in the plan doc for the full old→new mapping. The ticket-creation score threshold
(>55) is a fixed business rule from the original workflows and should not change during the
port.

## Validation

The plan defines 14 mirrored tests (Phase 8) against report tests 4.3–4.9 from the original
VM deployment (connectivity, web UI reachability, log collection, an Nmap-scan detection via
Suricata, phishing-email trigger, MISP enrichment, Ollama scoring, end-to-end automation
latency, and port/firewall exposure), plus a host-reboot self-healing test. When implementing
a phase, check it against the corresponding row in that table rather than inventing new
pass criteria.
