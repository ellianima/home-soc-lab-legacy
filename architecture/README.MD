#  SOC Lab Architecture

## Network Overview

| Component | OS | IP Address | Role |
|-----------|-----|-----------|------|
| Kali Linux VM | Kali Rolling | 192.168.1.8 (static) | SOC Manager |
| Ubuntu VM | Ubuntu 22.04 LTS | 192.168.1.5 (DHCP) | Monitored Endpoint (Linux Agent) |
| Windows 11 Laptop | Windows 11 | 192.168.1.7 (DHCP) | Monitored Endpoint (Windows Agent) |
| Home Router | ZTE F670L | 192.168.1.1 | Gateway / DHCP Server |

**Subnet:** `192.168.1.0/24`  
**Hypervisor:** Oracle VirtualBox (Host: Windows 11 Laptop)  
**Network Mode:** Bridged Adapter (all VMs on same LAN as physical host)

---

## Component Roles

### Kali Linux — SOC Manager (`192.168.1.8`)

The central nerve of the lab. All security tooling lives here.

| Tool | Version | Purpose |
|------|---------|---------|
| Wazuh Manager | 4.7.5 | SIEM — collects, correlates, alerts |
| Elasticsearch | 8.x | Log storage and indexing backend |
| Kibana / Wazuh Dashboard | 8.x | Alert visualization and dashboards |
| Zeek | 8.0.5 | Passive network traffic capture and analysis |
| TheHive | 5.2 (Docker) | Incident response case management |

**Key ports:**
- `1514/tcp` — Wazuh agent log ingestion
- `1515/tcp` — Wazuh agent enrollment
- `9200/tcp` — Elasticsearch API
- `5601/tcp` — Wazuh/Kibana dashboard
- `9000/tcp` — TheHive web interface

###  Ubuntu VM — Linux Agent (`192.168.1.5`)

**Hostname:** `ubuntu-soc-agent/Marshmallow-Cat-Glitter-Sparkles`  
Linux server endpoint. Wazuh agent installed, reports to Kali manager. Used for:
- Linux log monitoring (`/var/log/auth.log`, `/var/log/syslog`)
- File integrity monitoring (FIM)
- Attack simulation target (Phase 4)

### Windows 11 Laptop — Windows Agent (`192.168.1.7`)

**Hostname:** `Strawberry-Plump-Jelly-Silly-Fun`  
**Status:** ACTIVE  
Physical host machine also acting as a monitored endpoint. Wazuh agent installed. Used for:
- Windows Event Log monitoring (4624 logon, 4625 failed logon, 4688 process creation)
- Sysmon integration (planned Phase 4)
- Real Windows endpoint behavior

---

## Data Flow

```
[Ubuntu Agent]  ──TCP 1514──►
                               [Wazuh Manager : Kali 192.168.1.8]
[Windows Agent] ──TCP 1514──►     │
                                   ├── Stores logs → Elasticsearch
                                   ├── Fires alerts → Wazuh Dashboard
                                   └── Cases opened → TheHive

[Kali eth0] ──passive sniff──► [Zeek] ──► conn.log / dns.log / http.log
```

---

## Tool Interaction Map

```
Wazuh Manager
   ├── receives agent events (syslog, auth, process, FIM)
   ├── correlates against built-in + custom rules
   ├── fires alerts to Wazuh Dashboard (Kibana)
   └── (planned) forwards alerts to TheHive via API

Zeek (independent)
   ├── listens on eth0 in promiscuous mode
   ├── generates structured logs: conn.log, dns.log, http.log, notice.log
   └── logs stored at /opt/zeek/logs/current/

TheHive 5.2 (Docker)
   ├── receives incident cases (manual or automated)
   ├── tracks observables (IPs, hashes, domains)
   └── documents analyst actions and resolution
```

---

## OSI Layer Coverage

| OSI Layer | Tool Monitoring It |
|-----------|--------------------|
| Layer 7 — Application | Wazuh (app logs), Zeek (HTTP/DNS) |
| Layer 4 — Transport | Zeek (conn.log — TCP/UDP sessions) |
| Layer 3 — Network | Zeek (IP traffic), Wazuh (firewall rules) |
| Layer 2 — Data Link | Zeek (ARP — planned Suricata) |
| Endpoint (all layers) | Wazuh agents (Windows + Linux) |

---

