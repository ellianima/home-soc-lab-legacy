# 🔧 Wazuh Agent Troubleshooting Guide
 
## Problem: Agent Persistent "Never Connected" / Duplicate Name Loop
 
**Symptoms:**
- Agent shows `Never connected` in `agent_control -l`
- Wazuh logs show: `Duplicate name 'AgentName', rejecting enrollment`
- Agent service running on endpoint but manager keeps rejecting it
- `client.keys` deletion doesn't fix it
- Agent re-registers with a new ID but still never connects
 
**Root Cause:**  
Wazuh stores agent records in **multiple persistence layers simultaneously**. Deleting from one layer doesn't clear the others — the manager reconstructs the agent record on startup from whichever layer still has data.
 
**Persistence layers (all must be cleared):**
```
/var/ossec/queue/db/global.db      ← SQLite DB (primary)
/var/ossec/etc/client.keys         ← Agent key file
/var/ossec/queue/agents-timestamp  ← Agent timestamp record
/var/ossec/queue/fts/fts-queue     ← FTS queue
```
 
---
 
## Full Fix — Step by Step
 
### Step 1 — Stop Wazuh Manager
 
```bash
sudo systemctl stop wazuh-manager
```
 
### Step 2 — Find All Persistence Files
 
```bash
sudo grep -rl "AgentName" /var/ossec/ 2>/dev/null
```
 
Replace `AgentName` with your actual agent hostname.
 
### Step 3 — Delete From SQLite Database
 
```bash
# Find the correct DB (NOT /var/ossec/var/db/global.db — that one is empty)
sudo sqlite3 /var/ossec/queue/db/global.db "SELECT id,name FROM agent;"
 
# Delete the agent
sudo sqlite3 /var/ossec/queue/db/global.db "DELETE FROM agent WHERE name='AgentName';"
 
# Verify deletion
sudo sqlite3 /var/ossec/queue/db/global.db "SELECT id,name FROM agent;"
```
 
### Step 4 — Clean All Other Persistence Layers
 
```bash
sudo sed -i '/AgentName/d' /var/ossec/etc/client.keys
sudo sed -i '/AgentName/d' /var/ossec/queue/agents-timestamp
sudo sed -i '/AgentName/d' /var/ossec/queue/fts/fts-queue
```
 
### Step 5 — Verify Everything Is Clean
 
```bash
sudo grep "AgentName" \
  /var/ossec/etc/client.keys \
  /var/ossec/queue/agents-timestamp \
  /var/ossec/queue/fts/fts-queue 2>/dev/null
```
 
Should return nothing.
 
### Step 6 — Start Manager
 
```bash
sudo systemctl start wazuh-manager
```
 
### Step 7 — Re-enroll Agent (Endpoint Side)
 
**Windows:**
```powershell
NET STOP WazuhSvc
del "C:\Program Files (x86)\ossec-agent\client.keys"
NET START WazuhSvc
```
 
**Linux:**
```bash
sudo systemctl stop wazuh-agent
sudo rm /var/ossec/etc/client.keys
sudo systemctl start wazuh-agent
```
 
### Step 8 — Verify Agent Active
 
Wait 60 seconds then check:
 
```bash
sudo /var/ossec/bin/agent_control -l
```
 
Agent should appear as `Active`.
 
---
 
## Problem: Agent Service in Zombie State (Windows)
 
**Symptoms:**
```
NET START  → "already started"
Restart    → "cannot stop"
NET STOP   → "service is not started"
```
 
**Fix:**
```powershell
NET STOP WazuhSvc
NET START WazuhSvc
```
 
The stop/start cycle clears the zombie state even if the intermediate messages look contradictory.
 
---
 
## Problem: Agent Connecting Then Immediately Dropping
 
**Symptoms in agent log:**
```
Closing connection to server ([IP]:1514/tcp)
Trying to connect to server ([IP]:1514/tcp)
```
 
**Diagnosis:**
```powershell
# Check agent log (Windows)
type "C:\Program Files (x86)\ossec-agent\ossec.log" | Select-Object -Last 30
 
# Check manager log (Kali)
sudo tail -f /var/ossec/logs/ossec.log | grep -i "agentname\|error\|invalid"
```
 
**Common causes:**
- Stale/mismatched key in `client.keys`
- Agent trying to re-enroll instead of using existing key
- Manager IP changed after initial enrollment
 
**Fix:** Follow the full persistent agent removal steps above.
 
---
 
## Problem: Kali Loses Internet After Static IP Config
 
**Symptom:** `eth0` has no IPv4 address after setting static IP via GUI.
 
**Root cause:** NetworkManager GUI saved Manual method but left address fields empty.
 
**Fix:**
```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 192.168.1.8/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8,8.8.4.4"
 
sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
 
# Verify
ip a show eth0
ping -c 3 google.com
```
 
---
 
## Problem: Wazuh Dashboard 500 Error After Service Restarts
 
**Symptom:** `https://192.168.1.8` returns `{"statusCode":500,"error":"Internal Server Error"}`
 
**Cause:** Wazuh indexer (Elasticsearch) takes 2-3 minutes to fully initialize after restart.
 
**Fix:** Wait 3 minutes after restarting services, then refresh.
 
**Correct restart order:**
```bash
sudo systemctl restart wazuh-manager
sudo systemctl restart wazuh-indexer
sudo systemctl restart wazuh-dashboard
```
 
---
 
## Key Ports Reference
 
| Port | Protocol | Purpose |
|------|----------|---------|
| 1514 | TCP | Agent log ingestion |
| 1515 | TCP | Agent auto-enrollment (authd) |
| 55000 | TCP | Wazuh REST API |
| 9200 | TCP | Elasticsearch/Indexer API |
| 443 | TCP | Wazuh Dashboard (HTTPS) |
 
---
 
## Important File Locations
 
| File | Purpose |
|------|---------|
| `/var/ossec/queue/db/global.db` | Primary agent database (SQLite) |
| `/var/ossec/etc/client.keys` | Agent key store |
| `/var/ossec/queue/agents-timestamp` | Agent timestamp records |
| `/var/ossec/logs/ossec.log` | Manager log — check this first |
| `/var/ossec/etc/rules/local_rules.xml` | Custom detection rules |
| `C:\Program Files (x86)\ossec-agent\ossec.log` | Windows agent log |
| `C:\Program Files (x86)\ossec-agent\client.keys` | Windows agent key |
 
---
