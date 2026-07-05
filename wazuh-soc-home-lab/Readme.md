# 🛡️ Wazuh SOC Home Lab

A hands-on home lab that simulates a Security Operations Center (SOC) monitoring workflow: deploying a SIEM, collecting logs from an endpoint, simulating attacks, and analyzing the resulting security alerts.

> Built as a learning project to practice blue-team / SOC analyst fundamentals.
> 

## 🎯 Objectives

- Deploy a working SIEM platform (Wazuh) as a native multi-VM install
- Connect an endpoint agent as a log source
- Simulate common attacks: SSH brute-force + web application attacks (Shellshock, SQL injection, path traversal)
- Detect, triage, and analyze the generated security alerts
- Map observed activity to the MITRE ATT&CK framework

## 🧰 Tech Stack

| Component | Purpose |
| --- | --- |
| Wazuh Manager | Log analysis & alerting engine |
| Wazuh Indexer | Data storage (Elasticsearch-based) |
| Wazuh Dashboard | Web UI for investigation |
| Wazuh Agent | Endpoint log collector (installed on the monitored VM) |
| Filebeat | Ships alerts from the manager to the indexer |
| VirtualBox (NAT network) | Runs two Ubuntu Server VMs on one laptop |
| Ubuntu Server 25.04 / 26.04 (VMs) | SIEM server + monitored endpoint |

## 🏗️ Architecture

```
[ Endpoint VM ]  Ubuntu Server + wazuh-agent (10.0.2.5)
      │
      │  logs over 1514/1515
      ▼
[ SIEM Server VM ]  Ubuntu Server (10.0.2.3)
   Wazuh Manager  -->  Filebeat  -->  Wazuh Indexer  -->  Wazuh Dashboard
   (analysis/alerts)    (shipping)     (storage)          (https://10.0.2.3)
```

## ⚙️ Setup

**1. SIEM server — all-in-one (Manager + Indexer + Dashboard)**

```bash
# Official Wazuh installation assistant (v4.8)
curl -sO https://packages.wazuh.com/4.8/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

The assistant prints the generated `admin` password at the end — save it, then change it.

**2. Endpoint agent (on the monitored VM)**

```bash
sudo WAZUH_MANAGER='10.0.2.3' WAZUH_AGENT_NAME='agent' dpkg -i ./wazuh-agent_4.8.2-1_amd64.deb
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
```

**3. Enable auto-start of the whole stack (so a reboot doesn't break it)**

```bash
sudo systemctl enable wazuh-indexer wazuh-manager wazuh-dashboard filebeat
```

Dashboard: `https://10.0.2.3` — login `admin` / password.

## 🔥 Attack Simulation

**Scenario 1 — SSH brute-force**

An SSH brute-force was launched from the host against the endpoint VM (forwarded via VirtualBox):

```bash
# Repeated failed logins with a non-existent user = brute-force
ssh hacker@127.0.0.1 -p 2223   # repeat ~8 times, entering wrong passwords
```

**Scenario 2 — Web application attacks**

Apache was installed on the endpoint and its access log was registered as a Wazuh log source (`<localfile>` → `/var/log/apache2/access.log`). Then classic web attacks were fired against it:

```bash
# Shellshock (CVE-2014-6271) -> CRITICAL, rule 31168, level 15
curl -H "User-Agent: () { :; }; /bin/cat /etc/passwd" http://localhost/
# SQL injection attempt
curl "http://localhost/index.html?id=1' OR '1'='1"
# Path traversal attempt
curl "http://localhost/cgi-bin/../../../../etc/passwd"
```

## 📊 Results

The attack was detected end-to-end and visualized in the **Threat Hunting** dashboard:

| Metric | Value |
| --- | --- |
| Total security events | 601 |
| Authentication failures | 43 |
| Authentication success | 43 |
| Critical alerts (Rule level 15) | 1 — Shellshock (rule 31168) |
| MITRE techniques observed | Brute Force, Password Guessing, Valid Accounts, SSH, Exploit Public-Facing Application, Privilege Escalation |

## 🔍 Detection & Analysis

Alerts were reviewed in the **Threat Hunting** module. Key detections:

| Rule ID | Description | Level | MITRE ATT&CK |
| --- | --- | --- | --- |
| 5710 | Attempt to login using a non-existent user | 5 | T1110 – Brute Force |
| 5712 | SSHD brute force trying to get access to the system | 10 (high) | T1110.001 – Password Guessing |
| 5715 | SSHD authentication success | 3 | T1078 – Valid Accounts |
| 31168 | Shellshock attack detected | 15 (critical) | T1190 – Exploit Public-Facing Application; T1068 – Exploitation for Privilege Escalation |

For each alert I answered: **what happened → why it's suspicious → recommended response**.

## 🧯 Troubleshooting (real problems I solved)

- **Services didn't auto-start after a reboot** — the manager and `filebeat` were `disabled`, so alerts piled up in `/var/ossec/logs/alerts/alerts.log` but never reached the dashboard. Fixed with `systemctl enable --now`.
- **`filebeat` was inactive** — verified the pipeline with `filebeat test output` (`talk to server... OK`), then enabled it so alerts flow into the indexer.
- **Indexer wouldn't start** — a missing `/var/log/wazuh-indexer` directory; recreated it with the right owner and permissions.
- **Disk filled up** — grew the virtual disk and extended the LVM volume live (`growpart` → `pvresize` → `lvextend`).
- **Modern Ubuntu logging** — confirmed SSH failures are captured even though the process logs as `sshd-session`, reading `/var/log/auth.log` directly.

To debug the full pipeline (**agent → manager → filebeat → indexer → dashboard**) I used `wazuh-logtest`, `journalctl`, `systemctl status`, and raw `alerts.log`.

## 📚 What I Learned

- How a SIEM collects, normalizes, correlates logs and raises alerts
- The log flow: collection → normalization → correlation → alert → response
- Reading raw logs and mapping them to MITRE ATT&CK techniques
- Basic incident triage as a SOC analyst