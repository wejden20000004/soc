# SOC in a Box — Docker Migration Plan (Windows Host Edition)

Adapted for the real constraint: **Docker Desktop on Windows 11** (WSL2 backend), no additional Linux VM available. The Windows 10 endpoint remains a VMware VM.

Architecture principle: everything that can live in Docker Desktop does (MISP, TheHive, Elastic stack, Fleet, n8n, Ollama). The one thing that cannot — network sniffing — runs **natively on Windows** (Suricata + Npcap) and ships its alerts into the containerized stack. Zeek is dropped (no viable Windows build); coverage is compensated by Suricata signatures + Sysmon endpoint telemetry.

```
Windows 11 host (192.168.105.100)
├── Suricata (native, Npcap)  ──> sniffs VMware VMnet8 adapter
├── Elastic Agent / Filebeat (native) ──> ships eve.json → localhost:5044
├── Docker Desktop (WSL2)
│   ├── traefik            (443 → kibana/misp/hive/n8n .soc.local)
│   ├── elasticsearch, logstash, kibana, fleet-server   [compose.siem.yml]
│   ├── misp-core, mariadb, redis, misp-modules         [compose.misp.yml]
│   ├── thehive, cassandra, es-thehive, minio           [compose.thehive.yml]
│   └── n8n, ollama                                     [compose.automation.yml]
└── VMware Workstation
    └── Windows 10 endpoint VM (Sysmon + Elastic Agent → https://192.168.105.100:8220)
```

Repo layout is unchanged from the original plan (modular compose files, `.env`, `workflows/`), plus a `windows/` folder for the Suricata and Filebeat configs.

---

## Phase 0 — Host preparation (Day 1)

**1. Install Docker Desktop** with the WSL2 backend (default).

**2. Configure WSL2** — create/edit `%UserProfile%\.wslconfig`:

```ini
[wsl2]
memory=22GB
processors=6
swap=8GB
kernelCommandLine = "sysctl.vm.max_map_count=262144"
```

Then `wsl --shutdown` and restart Docker Desktop. Notes:
- Without `memory=`, WSL2 caps at 50% of host RAM — the stack won't fit.
- Without the `kernelCommandLine` sysctl, both Elasticsearch instances crash-loop after every reboot. Verify inside WSL: `wsl -d docker-desktop sysctl vm.max_map_count`.

**3. Storage rule (non-negotiable):** all databases (Elasticsearch ×2, Cassandra, MariaDB, MinIO, Ollama models) use **named Docker volumes**, never bind mounts to `C:\` paths. Bind mounts cross the Windows↔Linux filesystem boundary at 5–10× I/O penalty and will make Elasticsearch/Cassandra unusable. Bind mounts are acceptable only for small read-only configs (logstash pipelines, traefik config).

**4. VMware sniffing prerequisite:** in Virtual Network Editor, note which host adapter carries the Windows 10 VM's traffic (`VMware Network Adapter VMnet8` for the NAT 192.168.105.0/24). Native Suricata will listen there. Because VMnet8 is a host virtual adapter, the host sees the VM's traffic that crosses it (VM↔host, VM↔internet via NAT).

**5. Hosts file** (`C:\Windows\System32\drivers\etc\hosts`):
```
127.0.0.1 kibana.soc.local misp.soc.local hive.soc.local n8n.soc.local
```

**Resource budget (32 GB host):**

| Where | What | RAM |
|---|---|---|
| WSL2/Docker | ES-SIEM 4g heap / 6g limit | 6 GB |
| WSL2/Docker | Logstash + Kibana + Fleet | 3 GB |
| WSL2/Docker | MISP stack | 3 GB |
| WSL2/Docker | TheHive stack | 5 GB |
| WSL2/Docker | n8n + traefik | 1 GB |
| WSL2/Docker | Ollama (llama3.2) | 4 GB |
| Windows native | Suricata + Filebeat | 1 GB |
| Windows native | VMware Windows 10 VM | 3 GB |
| Windows native | OS + Docker Desktop overhead | ~5 GB |

Tight but viable at 32 GB. If pressure appears, the flex points are: run Ollama **natively on Windows** instead of in Docker (you already do — it frees WSL2 RAM and n8n reaches it at `http://host.docker.internal:11434`), and cap ES heap at 3g.

---

## Phase 1 — n8n and Ollama in Docker (Day 1–2)

```yaml
# compose.automation.yml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: unless-stopped
    environment:
      - GENERIC_TIMEZONE=Africa/Tunis
      - WEBHOOK_URL=https://n8n.soc.local/
    volumes: [n8n_data:/home/node/.n8n]
    networks: [soc-backend]
    mem_limit: 768m

  ollama:
    image: ollama/ollama
    restart: unless-stopped
    volumes: [ollama_models:/root/.ollama]
    networks: [soc-backend]
    mem_limit: 5g
    # NVIDIA GPU passthrough works well on Docker Desktop/WSL2:
    # deploy: { resources: { reservations: { devices:
    #   [{ driver: nvidia, count: all, capabilities: [gpu] }] } } }
```

Steps: export the two existing n8n workflows to JSON → commit to `workflows/` → `docker compose up -d` → `docker exec -it ollama ollama pull llama3.2` → import workflows (IPs get fixed in Phase 6).

Ollama placement decision: **container** if the "one compose" story matters most; **native Windows app** if RAM is tight or you want GPU without toolkit setup. Either way n8n reaches it by one URL (`http://ollama:11434` vs `http://host.docker.internal:11434`).

---

## Phase 2 — SIEM stack in Docker, detection on Windows (Days 3–8)

### 2a. Containerized analytics (compose.siem.yml)

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.18.0
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms4g -Xmx4g
      - ELASTIC_PASSWORD=${ES_PASSWORD}
    volumes: [es_siem_data:/usr/share/elasticsearch/data]   # named volume!
    networks: [soc-backend]
    mem_limit: 6g

  logstash:
    image: docker.elastic.co/logstash/logstash:8.18.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
    ports: ["5044:5044"]     # Beats input — native Suricata shipper + endpoint agents
    networks: [soc-backend]
    mem_limit: 1500m

  kibana:
    image: docker.elastic.co/kibana/kibana:8.18.0
    networks: [soc-backend]

  fleet-server:
    image: docker.elastic.co/elastic-agent/elastic-agent:8.18.0
    ports: ["8220:8220"]     # published so the Windows 10 VM can enroll via host IP
    networks: [soc-backend]
```

Ports 5044 and 8220 are published on the Windows host, so anything on the 192.168.105.0/24 network (the endpoint VM, native Suricata's shipper) reaches them at **192.168.105.100**.

### 2b. Native detection layer (replaces Security Onion's Suricata/Zeek)

Do NOT attempt `network_mode: host` sniffing in Docker Desktop — "host" is the hidden WSL2 VM's network; a containerized Suricata sees nothing from VMware.

1. Install **Npcap** (WinPcap-compatible mode).
2. Install **Suricata for Windows** (official MSI). In `suricata.yaml`, set the capture interface to the VMnet8 adapter (list adapters with `suricata.exe -l`, or use the `\Device\NPF_{GUID}` from `getmac /v`).
3. Rules: install `suricata-update` (Python) or download the Emerging Threats Open ruleset manually so the `ET SCAN Nmap` signatures from report test 4.5.1 keep firing.
4. Run Suricata as a Windows service: `suricata.exe -c suricata.yaml -i <adapter> --service-install` then start it. Output: `eve.json` in `C:\Program Files\Suricata\log\`.
5. Ship eve.json into the stack — install **Filebeat for Windows** with the Suricata module enabled, output to `localhost:5044` (Logstash). Do not bind-mount the log file into a container: file-change notifications don't propagate reliably across the Windows→WSL2 boundary; a native shipper over TCP is the robust pattern.

**Zeek:** dropped. Document the trade-off honestly in the report: signature-based NIDS (Suricata) + fine-grained endpoint telemetry (Sysmon Event ID 1/3) preserve the detection scenarios actually tested (Nmap scan, phishing); behavioral network metadata is the acknowledged gap, listed as a perspective (Zeek returns the day the stack runs on a Linux host — the compose file is ready for it).

---

## Phase 3 — MISP (Days 8–10)

`misp-docker` runs fine under Docker Desktop. Same compose as the original plan (misp-core, mariadb, redis, misp-modules on `soc-backend`, MariaDB on a named volume).

Data migration from the old VM (192.168.105.130), unchanged:
1. Bring containerized MISP up behind `https://misp.soc.local`.
2. Sync Servers → add old VM → **Pull all** (events with the "scanner"/"MALWARE" tags, taxonomies).
3. Re-enable the public feeds; generate a new API key into `.env`.
4. Old VM read-only for a week, then decommission.

One Windows-specific check: the old MISP VM must be reachable *from inside* the container for the pull — containers reach the 192.168.105.0/24 network through Docker Desktop's NAT by default; test with `docker exec misp-core curl -k https://192.168.105.130`.

---

## Phase 4 — TheHive (Days 10–12)

Relocation from the Ubuntu VM's Docker to Docker Desktop:
1. Copy the compose file into `compose.thehive.yml`, pin versions, attach to `soc-backend`, remove direct published ports (proxy only).
2. Data: on the old VM, `tar` each named volume (`docker run --rm -v vol:/data -v $(pwd):/backup alpine tar czf /backup/vol.tgz -C /data .`), copy the archives to Windows, restore into fresh named volumes (`docker run --rm -v vol:/data -v ${PWD}:/backup alpine tar xzf /backup/vol.tgz -C /data`). Given the handful of test tickets, re-creating from scratch is a legitimate shortcut.
3. Optional: add **Cortex** (analyzers/responders, native MISP integration) — first concrete step toward the SOAR perspective in the report's conclusion.

---

## Phase 5 — Reverse proxy + secrets (Day 12–13)

- Traefik container on 443 routes `kibana/misp/hive/n8n.soc.local`; self-signed wildcard cert for `*.soc.local`.
- Port-collision check: nothing else on Windows may hold 443 (IIS, Skype legacy, `netstat -ano | findstr :443`).
- All credentials in `.env` (git-ignored) + committed `.env.example`.

---

## Phase 6 — Rewire the n8n workflows (Days 13–15)

| Old (VM architecture) | New (Docker Desktop) |
|---|---|
| SSH poll into Security Onion for alerts | HTTP Request → `http://elasticsearch:9200/suricata-*/_search`, query `@timestamp > now-1m AND event.kind:alert` |
| MISP `https://192.168.105.130` | `https://misp-core` |
| Ollama `http://192.168.105.100:11434` | `http://ollama:11434` (container) or `http://host.docker.internal:11434` (native) |
| TheHive `http://192.168.105.136:9000` | `http://thehive:9000` |
| Gmail SMTP | unchanged |

`host.docker.internal` is the Docker Desktop magic hostname for "the Windows host" — use it for anything n8n must reach that runs natively (e.g. Ollama if kept native). Keep the score>55 ticket threshold unchanged. Import, run one manual execution per workflow, then activate.

---

## Phase 7 — Endpoint re-enrollment (Day 15)

The Windows 10 VM's Elastic Agent moves from Security Onion Fleet (192.168.105.10:8220) to the containerized Fleet:

1. New Kibana → Fleet → Windows policy (system + Sysmon channel `Microsoft-Windows-Sysmon/Operational`) → enrollment token.
2. On the endpoint VM:
   ```powershell
   .\elastic-agent.exe uninstall
   .\elastic-agent.exe install --url=https://192.168.105.100:8220 --enrollment-token=<token> --insecure
   ```
3. Sysmon untouched. Verify: agent Healthy in Fleet; Event ID 1/3 documents visible in Kibana; `Test-NetConnection 192.168.105.100 -Port 8220` succeeds (mirror of report test 4.9.3).
4. Windows Defender Firewall on the host must allow inbound 8220/5044/443 (Docker Desktop usually adds rules; verify).

---

## Phase 8 — Validation: re-run the 14 tests (Days 16–18)

| Report test | Windows-Docker equivalent | Pass criterion |
|---|---|---|
| 4.3.1 connectivity | endpoint VM pings 192.168.105.100; `docker compose ps` | all healthy |
| 4.3.2 web UIs | 4 UIs via `*.soc.local` | reachable over HTTPS |
| 4.4 log collection | endpoint agent → Fleet → ES | Sysmon logs in Kibana |
| 4.5.1 Nmap scan | nmap from/toward the endpoint across VMnet8; native Suricata watches | `ET SCAN Nmap` alert lands in ES via Filebeat |
| 4.5.2 phishing email | unchanged (IMAP trigger) | workflow triggers |
| 4.6 MISP enrichment | n8n → misp-core | "scanner"/"MALWARE" tags |
| 4.7 Ollama analysis | n8n → ollama | coherent scores (~60/~80) |
| 4.8 automation chain | full pipeline | TheHive ticket + email < 5 s |
| 4.9 infrastructure | `docker compose ps`; `netstat -ano` on host | 443/5044/8220 only |

Extra Windows-specific test: **reboot the host** and confirm everything self-heals — Docker Desktop auto-start, `restart: unless-stopped` on all services, Suricata and Filebeat installed as Windows services, and `vm.max_map_count` persisted via `.wslconfig`.

---

## Risks specific to the Windows host

| Risk | Mitigation |
|---|---|
| Containerized sniffing silently sees nothing | Never sniff from a container on Docker Desktop; native Suricata only. Smoke-test with `suricata.exe -i <adapter>` + a ping before wiring the pipeline |
| ES crash-loop after reboot | `kernelCommandLine` sysctl in `.wslconfig` (survives `wsl --shutdown`) |
| Terrible DB performance | Named volumes only; audit compose files for accidental `C:\` bind mounts on data dirs |
| WSL2 eats or starves RAM | Explicit `memory=` in `.wslconfig` + `mem_limit` on every service |
| Endpoint can't reach 8220/5044 | Windows Firewall inbound rules; VMware NAT allows VM→host by default |
| eve.json tail breaks across FS boundary | Ship with native Filebeat over TCP 5044, never a bind-mounted tail |
| Zeek coverage gap | Documented trade-off; Sysmon + Suricata cover the tested scenarios; Zeek listed as perspective for Linux deployment |

---

## Timeline

| Days | Phase |
|---|---|
| 1 | Docker Desktop + .wslconfig + Npcap |
| 1–2 | n8n + Ollama |
| 3–8 | Elastic stack in Docker + native Suricata/Filebeat pipeline |
| 8–10 | MISP + pull-sync |
| 10–12 | TheHive relocation (+ Cortex) |
| 12–13 | Traefik + secrets |
| 13–15 | Workflow rewiring + endpoint re-enrollment |
| 16–18 | 14-test validation + reboot test + decommission VMs |

End state: 4 VMs → 1 VM (the monitored Windows endpoint). Everything else is `docker compose up` on the Windows host plus two native Windows services (Suricata, Filebeat) — with the honest caveat, worth one paragraph in the report, that a production SME deployment would run the same compose files on a Linux server, where the detection layer also containerizes and Zeek comes back.
