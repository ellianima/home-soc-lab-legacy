# 🔧 Troubleshooting Log — March 31, 2026

**Environment:** Wazuh SOC Lab — Kali Linux Manager + Windows 11 Agent
**Session:** MITRE ATT&CK + FIM Fix
**Analyst:** ellianima

---

## Issue 1 — MITRE ATT&CK Dashboard Showing No Results

### Symptom

MITRE ATT&CK panel shows "No results" despite security alerts firing correctly. Running `net user mizuena /add` generates Rule 60109/60110 alerts but Technique and Tactic columns are blank.

### Investigation Steps

**Step 1 — Confirm rule file has MITRE tags**

```bash
sudo grep -A 15 "60109\|60110" /var/ossec/ruleset/rules/0580-win-security_rules.xml
```

Result: Rule 60109 contains `<id>T1098</id>` — tag exists ✅

**Step 2 — Check raw alert JSON for MITRE field**

```bash
sudo grep -A 30 "60109" /var/ossec/logs/alerts/2026/Mar/ossec-alerts-31.json | head -40
```

Result: Alert JSON has NO `mitre` key. Field being silently dropped.

**Step 3 — Check analysisd logs for MITRE errors**

```bash
sudo grep -i "mitre" /var/ossec/logs/ossec.log | tail -20
```

Result:
```
2026/03/26 05:15:30 wazuh-analysisd: ERROR: Response from the Mitre database cannot be parsed.
2026/03/26 05:15:30 wazuh-analysisd: ERROR: Mitre matrix information could not be loaded.
```

**Step 4 — Inspect MITRE database**

```bash
sudo sqlite3 /var/ossec/var/db/mitre.db ".tables"
# Error: file is not a database

sudo ls -la /var/ossec/var/db/mitre.db
# -rw-rw---- 1 root wazuh 13598720 May 29 2024 mitre.db
# File exists but is corrupted — not valid SQLite
```

### Root Cause

`/var/ossec/var/db/mitre.db` was corrupted. `wazuh-analysisd` failed to load MITRE matrix at startup and **silently dropped all MITRE enrichment** from every alert since March 26, 2026.

### Failed Fix Attempts

**Attempt 1 — Add `<mitre_alerts>yes</mitre_alerts>` to ossec.conf**

```xml
<alerts>
    <log_alert_level>3</log_alert_level>
    <email_alert_level>12</email_alert_level>
    <mitre_alerts>yes</mitre_alerts>  <!-- INVALID ELEMENT -->
</alerts>
```

Result: Manager crashed on restart
```
ERROR: (1230): Invalid element in the configuration
CRITICAL: Configuration error at 'etc/ossec.conf'
```

Fix: Removed invalid element, manager restarted successfully.

Lesson: Always run before restarting:
```bash
sudo /var/ossec/bin/wazuh-control configtest
```

**Attempt 2 — Download mitre.db from GitHub raw URL**

```bash
sudo curl -o /var/ossec/var/db/mitre.db \
  https://raw.githubusercontent.com/wazuh/wazuh/v4.7.5/src/analysisd/mitre_db/mitre.db
```

Result: Only downloaded 14 bytes — GitHub raw URL for binary files returns redirect, not actual file.

**Attempt 3 — Run mitre_updater.py**

```bash
sudo python3 /var/ossec/framework/scripts/mitre_updater.py
```

Result: Script does not exist in Wazuh 4.7.5.

### Successful Fix — Reinstall wazuh-manager

```bash
sudo systemctl stop wazuh-manager
sudo rm /var/ossec/var/db/mitre.db
sudo apt-get install --reinstall wazuh-manager
```

Result: Package reinstalled, fresh `mitre.db` restored.

⚠️ Side effect: `apt-get install --reinstall` without version pin upgraded manager from 4.7.5 → 4.14.4

Verification:
```bash
sudo sqlite3 /var/ossec/var/db/mitre.db ".tables"
# alias  contributor  data_source  defense_bypassed  effective_permission
# group  impact  metadata  mitigate  mitigation  permission  phase
# platform  reference  software  system_requirement  tactic  technique  use
# ✅ All tables present
```

---

## Issue 2 — Accidental Wazuh Version Upgrade (4.7.5 → 4.14.4)

### Symptom

After reinstalling wazuh-manager, dashboard shows version mismatch warning:
```
[API version] Wazuh API and Wazuh dashboard version mismatch.
API version: 4.14.4. App version: 4.7.5.
```

### Root Cause

`apt-get install --reinstall wazuh-manager` without version pin grabbed latest available package (4.14.4) instead of current version (4.7.5).

### Fix — Upgrade all components to 4.14.4

Decision: Lab environment → upgrade everything rather than downgrade.

Upgrade order (critical): **Indexer → Manager → Dashboard**

```bash
sudo apt-get install --only-upgrade wazuh-indexer
sudo apt-get install --only-upgrade wazuh-dashboard
sudo systemctl restart wazuh-indexer
sudo systemctl restart wazuh-manager
sudo systemctl restart wazuh-dashboard
```

---

## Issue 3 — Dashboard Crash After Upgrade (SSL Cert Path Mismatch)

### Symptom

`wazuh-dashboard` crash-loops after upgrade. Journal shows:
```
Error: ENOENT: no such file or directory
open '/etc/wazuh-dashboard/certs/dashboard-key.pem'
```

### Root Cause

New 4.14.4 dashboard config (`opensearch_dashboards.yml`) expects shorter cert filenames without `wazuh-` prefix. Existing certs from 4.7.5 installation kept original names.

Config expected:
```
dashboard-key.pem
dashboard.pem
```

Actual files present:
```
wazuh-dashboard-key.pem
wazuh-dashboard.pem
root-ca.pem
```

### Fix — Update config to match existing filenames

```bash
sudo nano /etc/wazuh-dashboard/opensearch_dashboards.yml
```

Changed:
```yaml
server.ssl.key: "/etc/wazuh-dashboard/certs/dashboard-key.pem"
server.ssl.certificate: "/etc/wazuh-dashboard/certs/dashboard.pem"
```

To:
```yaml
server.ssl.key: "/etc/wazuh-dashboard/certs/wazuh-dashboard-key.pem"
server.ssl.certificate: "/etc/wazuh-dashboard/certs/wazuh-dashboard.pem"
```

Rationale: Editing config is safer than renaming cert files. Other services may reference the existing filenames.

---

## Issue 4 — Dashboard Still Failing After SSL Fix (IPv6 vs IPv4)

### Symptom

Dashboard still crash-loops after SSL fix. Journal shows:
```
[ConnectionError]: connect ECONNREFUSED ::1:9200
```

### Root Cause

Dashboard config uses `localhost` which resolves to `::1` (IPv6) on this system. Wazuh indexer only listens on `127.0.0.1` (IPv4). Connection refused because service not bound to IPv6.

Root: `/etc/hosts` maps both `127.0.0.1` and `::1` to `localhost`. Modern Linux systems prefer IPv6, causing the mismatch.

### Fix — Force IPv4 explicitly

```bash
sudo nano /etc/wazuh-dashboard/opensearch_dashboards.yml
```

Changed:
```yaml
opensearch.hosts: https://localhost:9200
```

To:
```yaml
opensearch.hosts: https://127.0.0.1:9200
```

Result: ✅ Dashboard started successfully. All three services running on Wazuh 4.14.4.

---

## Issue 5 — FIM Events Not Showing in Dashboard

### Symptom

After adding `C:\Users\Public` to syscheck config and creating test file, FIM dashboard shows "No results match your search criteria."

### Investigation

**Step 1 — Verify config was saved correctly**

```powershell
type "C:\Program Files (x86)\ossec-agent\ossec.conf" | findstr "Users\Public"
# <directories realtime="yes" report_changes="yes">C:\Users\Public</directories> ✅
```

**Step 2 — Verify agent is running**

```powershell
Get-Service WazuhSvc
# Status: Running ✅
```

**Step 3 — Check if manager is receiving FIM events (live tail)**

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.json | grep -i "syscheck\|fim\|Public"
```

Then on Windows:
```powershell
New-Item -Path "C:\Users\Public\fim_test2.txt" -ItemType File
```

Result: Manager received event immediately:
```json
"description": "File added to the system."
"path": "c:\\users\\public\\fim_test2.txt"
"mode": "realtime"
"event": "added"
```

Manager IS receiving events. Issue was dashboard display lag / index.

### Root Cause

FIM requires two-phase detection:
1. First scan → builds baseline database of existing files
2. Subsequent events → compares against baseline → fires alert

First test file was created before baseline scan completed. Second file (`fim_test2.txt`) created after baseline → detected correctly.

### Fix

Restart agent to force baseline scan, then create new test file:

```powershell
NET STOP WazuhSvc
NET START WazuhSvc
New-Item -Path "C:\Users\Public\fim_test2.txt" -ItemType File
```

Result: ✅ FIM alert appeared in dashboard
```
Path: c:\users\public\fim_test2.txt
Action: added
Rule: File added to the system (Rule 554, Level 5)
```

---

## Summary Table

| Issue | Root Cause | Fix | Prevention |
|-------|-----------|-----|------------|
| MITRE blank | Corrupted `mitre.db` — silent failure since Mar 26 | Reinstall wazuh-manager | Health checks on MITRE DB; monitor analysisd logs |
| Invalid config crash | Added non-existent XML element | Remove element, restart | Always run `configtest` before restart |
| Accidental upgrade | No version pin on `apt-get reinstall` | Upgraded all components to 4.14.4 | Always pin: `package=version` |
| SSL cert path mismatch | New version expects different filenames | Updated config paths | Check cert paths after every upgrade |
| IPv6 connection refused | `localhost` resolves to `::1`, indexer on IPv4 | Use `127.0.0.1` explicitly | Use explicit IPs in all config files |
| FIM no events | Baseline not yet built when first test ran | Restart agent + recreate file | Wait for baseline scan after agent restart |

---

## Commands Reference

```bash
# Config validation (ALWAYS before restart)
sudo /var/ossec/bin/wazuh-control configtest

# Check MITRE database health
sudo sqlite3 /var/ossec/var/db/mitre.db ".tables"
sudo grep -i "mitre" /var/ossec/logs/ossec.log | tail -20

# Live alert monitoring
sudo tail -f /var/ossec/logs/alerts/alerts.json | grep -i "syscheck\|mitre\|Public"

# Agent status
sudo /var/ossec/bin/agent_control -l

# Service restart order
sudo systemctl restart wazuh-indexer
sudo systemctl restart wazuh-manager
sudo systemctl restart wazuh-dashboard

# Version-pinned reinstall (safe)
sudo apt-get install --reinstall wazuh-manager=4.7.5-1
```

---

*Cerveaux Labs — Home SOC Operations*
*Analyst: ellianima*
*github.com/ellianima/home-soc-lab*
