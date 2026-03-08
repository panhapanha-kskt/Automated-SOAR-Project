# 🏗️ Wazuh SOC — System Architecture & Attack Workflows

> **Complete architecture diagrams, data flow, attack scenarios, and system metrics**
>
> Group 7 — Kosal Karuna, Cho Davon, Tith Sopanha | Server: `192.168.18.43`

---

## 📑 Table of Contents

- [Confirmed Active Response Scripts](#confirmed-active-response-scripts)
- [Complete System Architecture](#complete-system-architecture)
- [Data Flow Diagram](#data-flow-diagram)
- [Attack Scenario Workflows](#attack-scenario-workflows)
- [System Components](#system-components)
- [System Metrics](#system-metrics)

---

## Confirmed Active Response Scripts

| File | Status | Location |
|---|---|---|
| `disable-account` | ✅ PRESENT | `/var/ossec/active-response/bin/disable-account` |
| `firewall-drop` | ✅ PRESENT | `/var/ossec/active-response/bin/firewall-drop` |
| `cleanup-timeouts.sh` | ✅ PRESENT | `/var/ossec/active-response/bin/cleanup-timeouts.sh` |
| `quarantine-file.sh` | ✅ PRESENT | `/var/ossec/active-response/bin/quarantine-file.sh` |

---

## Complete System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         YOUR WAZUH SERVER ARCHITECTURE                      │
│                                 192.168.18.43                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              INPUT SOURCES                                  │
├──────────────────┬──────────────────┬──────────────────┬────────────────────┤
│   Apache Logs    │     Syslog       │     Auditd       │     Journald       │
│ /var/log/httpd/  │   auth.log       │   audit.log      │   system logs      │
└────────┬─────────┴────────┬─────────┴────────┬─────────┴────────┬───────────┘
         │                  │                   │                  │
         └──────────────────┴───────────────────┴──────────────────┘
                                      │
                                      ▼
                       ┌──────────────────────────────┐
                       │      WAZUH LOG COLLECTOR     │
                       │   (Reading all log sources)  │
                       └──────────────┬───────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         RULE ENGINE & DETECTION                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    LOCAL RULES  (local_rules.xml)                     │ │
│  ├───────────────────────────────────────────────────────────────────────┤ │
│  │  ⚠️  Web Attacks:  100800, 100802  (Path Traversal)                  │ │
│  │  🔴 Critical:      100101 (Reverse Shell), 100102 (Metasploit)       │ │
│  │  🟡 Medium:        100105 (SSH Brute),     100108 (Web Tools)        │ │
│  │  🟢 Low:           100110 (Info Gathering), 100111 (Persistence)     │ │
│  │  🎯 MITRE:         101020 (T1590.005),      101002 (T1595.002)       │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │              CRIMINAL IP RULES  (criminal_ip_ruleset.xml)             │ │
│  ├───────────────────────────────────────────────────────────────────────┤ │
│  │  🚨 Critical Score:    100628  (Critical inbound)                    │ │
│  │  🌐 TOR Network:       100625  (TOR detection)                       │ │
│  │  ⚡ Scanner Activity:  100636  (Scanner detected)                    │ │
│  │  🔥 Composite:         100650  (TOR+Critical), 100652 (Scan+Crit)    │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            INTEGRATION LAYER                                │
├──────────────────┬──────────────────┬──────────────────┬────────────────────┤
│   Criminal IP    │   VirusTotal     │    Shuffle       │     Syslog         │
│   API Lookup     │   Hash Check     │    SOAR          │    Forwarding      │
└────────┬─────────┴────────┬─────────┴────────┬─────────┴────────┬───────────┘
         │                  │                   │                  │
         └──────────────────┴───────────────────┴──────────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────────┐
                    │       ENRICHED ALERTS GENERATED     │
                    │  (Original + Threat Intelligence)   │
                    └──────────────────┬──────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          ACTIVE RESPONSE LAYER                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                     ACTIVE RESPONSE TRIGGERS                          │ │
│  ├───────────────────────────────────────────────────────────────────────┤ │
│  │                                                                       │ │
│  │  Rules 100101, 100113, 100302  ──────────────► quarantine-file.sh    │ │
│  │         (Malware / Reverse Shell)               (Quarantine file)    │ │
│  │                                                                       │ │
│  │  Rule 100500  ───────────────────────────────► disable-account       │ │
│  │         (Multiple failed logins)                (Lock user)          │ │
│  │                                                                       │ │
│  │  Rules 101020, 101002, 101003, 5710, 5763  ──► firewall-drop         │ │
│  │         (Scans / Brute force)                   (Block IP)           │ │
│  │                                                                       │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                      │                                     │
│                                      ▼                                     │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                        CLEANUP MECHANISM                              │ │
│  ├───────────────────────────────────────────────────────────────────────┤ │
│  │                                                                       │ │
│  │   Cron: */5 * * * *  ──────────────────────► cleanup-timeouts.sh     │ │
│  │         (Runs every 5 minutes)               (Unblocks expired IPs   │ │
│  │                                               and re-enables accounts)│ │
│  │                                                                       │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      BACKUP & LOGGING STRUCTURE                             │
│                        /home/wazuh-user/backup/                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                             LOG FILES                                 │ │
│  ├───────────────────────────────────────────────────────────────────────┤ │
│  │  📝 quarantine.log        (All quarantine actions)                   │ │
│  │  📝 firewall.log          (All IP blocks/unblocks)                   │ │
│  │  📝 disable-account.log   (All account disables)                     │ │
│  │  📝 cleanup.log           (All cleanup operations)                   │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ┌──────────────────┬──────────────────┬──────────────────────────────────┐ │
│  │    firewall/     │    accounts/     │         nas/backup/              │ │
│  ├──────────────────┼──────────────────┼──────────────────────────────────┤ │
│  │  blocks/         │  disabled/       │  ├── firewall/  (backup)         │ │
│  │  ├── blocked_    │  ├── disabled_   │  └── accounts/  (backup)         │ │
│  │  │   YYYYMMDD    │  │   YYYYMMDD    │                                  │ │
│  │  │   .txt        │  │   .txt        │  backup-administrator@cadt/      │ │
│  │  └── *.meta      │  └── *.meta      │  └── backup/                     │ │
│  │                  │                  │      ├── firewall/               │ │
│  │                  │                  │      └── accounts/               │ │
│  └──────────────────┴──────────────────┴──────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow Diagram

```
┌─────────┐   ┌──────────┐   ┌─────────┐   ┌──────────┐   ┌──────────┐
│ Attack  │──▶│   Log    │──▶│  Rule   │──▶│ Integra- │──▶│ Enriched │
│ Source  │   │Generation│   │ Trigger │   │   tion   │   │  Alert   │
└─────────┘   └──────────┘   └─────────┘   └──────────┘   └──────────┘
                                  │                              │
                                  ▼                              ▼
                           ┌─────────────┐               ┌─────────────┐
                           │   Active    │               │ Criminal IP │
                           │  Response  │               │   Lookup    │
                           │  Command   │               └─────────────┘
                           └──────┬──────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          EXECUTION & BACKUP                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐   ┌──────────────┐   ┌─────────────────────────────┐ │
│  │firewall-drop│──▶│ Block IP in  │──▶│ Save to persistence:        │ │
│  │  executes   │   │  iptables    │   │ /backup/firewall/blocks/     │ │
│  └─────────────┘   └──────────────┘   └─────────────────────────────┘ │
│                                                                         │
│  ┌─────────────┐   ┌──────────────┐   ┌─────────────────────────────┐ │
│  │disable-acct │──▶│  Lock user   │──▶│ Save to persistence:        │ │
│  │  executes   │   │   account    │   │ /backup/accounts/disabled/   │ │
│  └─────────────┘   └──────────────┘   └─────────────────────────────┘ │
│                                                                         │
│  ┌─────────────┐   ┌──────────────┐   ┌─────────────────────────────┐ │
│  │quarantine-  │──▶│ Move file to │──▶│ Save metadata & backup      │ │
│  │file executes│   │  quarantine  │   │ to NAS and admin backup      │ │
│  └─────────────┘   └──────────────┘   └─────────────────────────────┘ │
│                                                                         │
│                    ┌─────────────────────────────────────────────────┐ │
│                    │            cleanup-timeouts.sh                  │ │
│                    │            (Runs every 5 minutes)               │ │
│                    ├─────────────────────────────────────────────────┤ │
│                    │  ✓ Checks expired IP blocks                     │ │
│                    │  ✓ Checks expired account locks                 │ │
│                    │  ✓ Unblocks / unlocks automatically             │ │
│                    │  ✓ Logs all cleanup actions                     │ │
│                    └─────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Attack Scenario Workflows

### Scenario 1 — Web Attack → IP Block

```
Attacker ──► Path Traversal ──► Apache Log ──► Rule 100800 ──► Criminal IP Lookup
  (185.220.101.20)                                                      │
                                                                        ▼
                                                               Score: Critical
                                                                        │
                                    ┌───────────────────────────────────┘
                                    ▼
                       ┌────────────────────────────┐
                       │     Active Response        │
                       │     firewall-drop add      │
                       └───────────────┬────────────┘
                                       ▼
                       ┌────────────────────────────┐
                       │  iptables -I INPUT -s      │
                       │  185.220.101.20 -j DROP    │
                       └───────────────┬────────────┘
                                       ▼
                       ┌────────────────────────────┐
                       │  Backup:                   │
                       │  /backup/firewall/blocks/  │
                       │  /nas/backup/firewall/     │
                       │  /admin/backup/firewall/   │
                       └────────────────────────────┘
```

---

### Scenario 2 — SSH Brute Force → Account Lock

```
Attacker ──► Multiple SSH failures ──► Rule 5710/5763 ──► Rule 100500 triggers
  (192.168.1.100)                        (frequency)               │
                                                                    ▼
                                                    ┌──────────────────────────┐
                                                    │  disable-account add     │
                                                    │       user: root         │
                                                    └──────────────┬───────────┘
                                                                   ▼
                                                    ┌──────────────────────────┐
                                                    │  passwd -l root          │
                                                    │  usermod -s /nologin     │
                                                    │  pkill -u root           │
                                                    └──────────────┬───────────┘
                                                                   ▼
                                                    ┌──────────────────────────┐
                                                    │  Backup:                 │
                                                    │  /backup/accounts/       │
                                                    │  disabled/root.meta      │
                                                    └──────────────────────────┘
```

---

### Scenario 3 — Malware File → Quarantine

```
Malware dropped ──► Syscheck detects ──► Rule 100101/550 ──► quarantine-file.sh
  (/tmp/evil.sh)       (file change)          (match)                │
                                                                      ▼
                                                    ┌──────────────────────────┐
                                                    │  mv /tmp/evil.sh         │
                                                    │  → /quarantine/          │
                                                    │  20250308_120000_        │
                                                    │  evil.sh.quarantined     │
                                                    └──────────────┬───────────┘
                                                                   ▼
                                                    ┌──────────────────────────┐
                                                    │  Create metadata:        │
                                                    │  - Original path         │
                                                    │  - Rule ID: 100101       │
                                                    │  - Timestamp             │
                                                    │  - File permissions      │
                                                    └──────────────┬───────────┘
                                                                   ▼
                                                    ┌──────────────────────────┐
                                                    │  Backup to:              │
                                                    │  NAS & admin backup      │
                                                    └──────────────────────────┘
```

---

### Scenario 4 — TOR + Critical IP → Immediate Block

```
Attacker ──► SSH attempt ──► Rule 5716 ──► Criminal IP API ──► is_tor: true
  (TOR exit node)                                               score: Critical
                                                                       │
                                         ┌─────────────────────────────┘
                                         ▼
                            ┌─────────────────────────┐
                            │  Rule 100650 fires      │
                            │  [CRITICAL] TOR exit    │
                            │  node + Critical score  │
                            └─────────────┬───────────┘
                                          ▼
                            ┌─────────────────────────┐
                            │  firewall-drop          │
                            │  timeout: 7200s (2hrs)  │
                            └─────────────┬───────────┘
                                          ▼
                            ┌─────────────────────────┐
                            │  📧 Email alert sent    │
                            │  🔥 IP blocked          │
                            │  📝 Logged to DB        │
                            │  🔄 Shuffle notified    │
                            └─────────────────────────┘
```

---

## System Components

| Layer | Components | Status |
|---|---|---|
| **Detection** | Local Rules (35+), Criminal IP Rules (30+) | ✅ ACTIVE |
| **Enrichment** | Criminal IP, VirusTotal, Shuffle | ✅ ACTIVE |
| **Response** | `firewall-drop`, `disable-account`, `quarantine-file` | ✅ ACTIVE |
| **Cleanup** | `cleanup-timeouts.sh` (cron every 5 min) | ✅ ACTIVE |
| **Backup** | 3-way redundancy (local, NAS, admin) | ✅ ACTIVE |
| **Logging** | 4 log files with rotation | ✅ ACTIVE |

---

## System Metrics

| Metric | Value |
|---|---|
| Total Detection Rules | 65+ (35 local + 30 Criminal IP) |
| Active Response Scripts | 4 fully functional |
| Backup Locations | 3 (Local + NAS + Admin) |
| Log Files | 4 (quarantine, firewall, account, cleanup) |
| Integration Partners | 4 (Criminal IP, VirusTotal, Shuffle, Syslog) |
| Cleanup Interval | Every 5 minutes via cron |
| MITRE Techniques Covered | 13 ATT&CK techniques |
| Rule Levels Monitored | Level 2–16 (all tiers) |

---

## Final System Status

| Capability | Status |
|---|---|
| Complete threat detection | ✅ |
| Criminal IP threat intelligence | ✅ |
| Automated IP blocking | ✅ |
| Automated account disabling | ✅ |
| Automated file quarantine | ✅ |
| Automated cleanup of expired blocks | ✅ |
| Triple-redundant backup system | ✅ |
| Comprehensive logging | ✅ |
| MITRE ATT&CK mapping | ✅ |
| Multiple third-party integrations | ✅ |
| Wazuh Amazon Linux 2023 compatible | ✅ |
