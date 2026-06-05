# 🛡️ Wazuh SOC Automation — SOAR Engine

> **Automated threat detection, IP blocking, and incident response powered by Wazuh + Criminal IP + Suricata**
>
> Group 7 — Kosal Karuna, Cho Davon, Tith Sopanha

---

## 📑 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Integration Matrix](#integration-matrix)
- [Rule Coverage](#rule-coverage)
- [Active Response Actions](#active-response-actions)
- [Criminal IP Rules](#criminal-ip-rules)
- [MITRE ATT\&CK Mapping](#mitre-attck-mapping)
- [Log & Storage Paths](#log--storage-paths)
- [Email Configuration](#email-configuration)
- [Features](#features)

---

## Overview

This SOAR engine integrates with your existing Wazuh deployment to provide:

- ✅ Automated IP blocking via `firewall-drop`
- ✅ Account lockout via `disable-account`
- ✅ File quarantine via `quarantine-file.sh`
- ✅ Criminal IP reputation enrichment
- ✅ MITRE ATT&CK–tagged alerting
- ✅ SQLite-backed deduplication and statistics
- ✅ Email notifications for critical/high alerts
- ✅ Suricata `eve.json` integration
- ✅ Auto-cleanup of expired firewall blocks

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Wazuh Manager                        │
│                                                             │
│  ossec.conf ──► local_rules.xml ──► criminal_ip_ruleset.xml │
│       │                │                     │              │
│       ▼                ▼                     ▼              │
│  Active Response   Rule Engine        Criminal IP API       │
│       │                │                     │              │
│       └────────────────┴─────────────────────┘              │
│                        │                                    │
│                   SOAR Engine                               │
│         (soar-engine.py / custom-criminalip.py)             │
│                        │                                    │
│         ┌──────────────┼──────────────┐                     │
│         ▼              ▼              ▼                     │
│    firewall-drop  disable-account  quarantine-file.sh       │
│         │              │              │                     │
│    blocked_ips.log  disable.log  quarantine.log             │
│                        │                                    │
│                   soar.db (SQLite)                          │
└─────────────────────────────────────────────────────────────┘
```

---

## Integration Matrix

### 1. `ossec.conf` Integration ✅

| `ossec.conf` Section | How the Engine Uses It |
|---|---|
| `<command name="firewall-drop">` | Calls `/var/ossec/active-response/bin/firewall-drop` |
| `<command name="disable-account">` | Calls `/var/ossec/active-response/bin/disable-account` |
| `<command name="quarantine-file">` | Calls `/var/ossec/active-response/bin/quarantine-file.sh` |
| `<active-response rules_id="101020,101002,...">` | Mapped in `ACTIVE_RESPONSE_RULES` |
| `<active-response rules_id="100500">` | Mapped in `ACCOUNT_DISABLE_RULES` |
| `<active-response rules_id="100101,100113,100302">` | Mapped in `QUARANTINE_RULES` |
| `<syslog_output level="15">` | Sends CRITICAL alerts via email |

---

### 2. `local_rules.xml` Integration ✅

| Rule ID | Engine Mapping | Action |
|---|---|---|
| `100101` | `CRITICAL_RULES` + `QUARANTINE_RULES` | 🔥 Block IP + 🔒 Quarantine file |
| `100102` | `HIGH_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |
| `100103` | `HIGH_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |
| `100104` | `HIGH_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |
| `100105` | `MEDIUM_RULES` + `FAILED_LOGIN_RULES` | 📊 Track attempts |
| `100106` | `MEDIUM_RULES` | 👁️ Monitor |
| `100107` | `MEDIUM_RULES` | 👁️ Monitor |
| `100108` | `MEDIUM_RULES` | 👁️ Monitor |
| `100109` | `MEDIUM_RULES` | 👁️ Monitor |
| `100110` | `LOW_RULES` | 📝 Log |
| `100111` | `LOW_RULES` | 📝 Log |
| `100112` | `LOW_RULES` | 📝 Log |
| `100113` | `CRITICAL_RULES` + `QUARANTINE_RULES` | 🔥 Block IP + 🔒 Quarantine |
| `100114` | `CRITICAL_RULES` | 🔥 Block IP |
| `100115` | `HIGH_RULES` | 🔥 Block IP |
| `100116` | `MEDIUM_RULES` + `FAILED_LOGIN_RULES` | 📊 Track attempts |
| `100117` | `HIGH_RULES` | 🔥 Block IP |
| `100118` | `HIGH_RULES` | 🔥 Block IP |
| `100119` | `HIGH_RULES` | 🔥 Block IP |
| `100120` | `MEDIUM_RULES` | 👁️ Monitor |
| `100200` | `MEDIUM_RULES` | 👁️ Monitor |
| `100201` | `MEDIUM_RULES` | 👁️ Monitor |
| `100203` | `HIGH_RULES` | 🔥 Block IP |
| `100204` | `CRITICAL_RULES` | 🔥 Block IP |
| `100205` | `CRITICAL_RULES` | 🔥 Block IP |
| `100206` | `MEDIUM_RULES` | 👁️ Monitor |
| `100207` | `MEDIUM_RULES` | 👁️ Monitor |
| `100208` | `HIGH_RULES` | 🔥 Block IP |
| `100209` | `LOW_RULES` + `FALSE_POSITIVE_RULES` | ⏭️ Ignored |
| `100210` | `HIGH_RULES` + `FAILED_LOGIN_RULES` | 🔥 Block IP |
| `100300` | `LOW_RULES` | 📝 Log |
| `100302` | `CRITICAL_RULES` + `QUARANTINE_RULES` | 🔥 Block IP + 🔒 Quarantine |
| `100400` | `MEDIUM_RULES` | 👁️ Monitor |
| `100500` | `LOW_RULES` + `ACCOUNT_DISABLE_RULES` | 🔐 Disable account |
| `100700` | `HIGH_RULES` | 🔥 Block IP |
| `100800` | `HIGH_RULES` | 🔥 Block IP |
| `100802` | `HIGH_RULES` | 🔥 Block IP |
| `100803` | `HIGH_RULES` | 🔥 Block IP |
| `100901` | `HIGH_RULES` + `FAILED_LOGIN_RULES` | 🔥 Block IP |
| `101002` | `HIGH_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |
| `101003` | `CRITICAL_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |
| `101020` | `HIGH_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |

---

### 3. `criminal_ip_ruleset.xml` Integration ✅

| Rule ID | Description | Engine Mapping | Action |
|---|---|---|---|
| `100623` | Base Criminal IP event | Tracked | 👁️ Monitor |
| `100624` | VPN detected | `MEDIUM_RULES` | 👁️ Monitor |
| `100625` | TOR network | `HIGH_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |
| `100626` | Proxy server | `MEDIUM_RULES` | 👁️ Monitor |
| `100627` | Dark Web activity | `HIGH_RULES` | 🔥 Block IP |
| `100628` | Critical inbound score | `HIGH_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |
| `100629` | Dangerous inbound score | `HIGH_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |
| `100630` | Moderate inbound score | `MEDIUM_RULES` | 👁️ Monitor |
| `100631` | Safe inbound score | `LOW_RULES` | 📝 Log |
| `100632` | Low inbound score | `LOW_RULES` | 📝 Log |
| `100633` | Hosting service | `MEDIUM_RULES` | 👁️ Monitor |
| `100634` | Cloud service | `MEDIUM_RULES` | 👁️ Monitor |
| `100635` | Snort IDS flagged | `MEDIUM_RULES` | 👁️ Monitor |
| `100636` | Scanner activity | `MEDIUM_RULES` | 👁️ Monitor |
| `100638` | Anonymous VPN | `MEDIUM_RULES` | 👁️ Monitor |
| `100650` | TOR + Critical score | `CRITICAL_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |
| `100651` | TOR + Dangerous score | `HIGH_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |
| `100652` | Scanner + Critical score | `CRITICAL_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |
| `100653` | Scanner + Dangerous score | `HIGH_RULES` | 🔥 Block IP |
| `100654` | Dark Web + Scanner | `HIGH_RULES` | 🔥 Block IP |
| `100655` | Dark Web + Critical score | `CRITICAL_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |
| `100656` | Snort + Critical score | `CRITICAL_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |
| `100657` | Anonymous VPN + Critical | `HIGH_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |
| `100658` | Proxy + Critical/Dangerous | `HIGH_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |
| `100660` | Outbound Moderate | `MEDIUM_RULES` | 👁️ Monitor |
| `100661` | Outbound Dangerous | `HIGH_RULES` | 🔥 Block IP |
| `100662` | Outbound Critical (exfil) | `HIGH_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |
| `100663` | Outbound TOR (C2) | `CRITICAL_RULES` + `ACTIVE_RESPONSE_RULES` | 🔥 Block IP |

---

### 4. Active Response Scripts ✅

| Script | Called By | Purpose |
|---|---|---|
| `/var/ossec/active-response/bin/firewall-drop` | `ActiveResponseManager.block_ip()` | Drop traffic from malicious IP |
| `/var/ossec/active-response/bin/disable-account` | `ActiveResponseManager.disable_account()` | Lock out compromised user account |
| `/var/ossec/active-response/bin/quarantine-file.sh` | `ActiveResponseManager.quarantine_file()` | Isolate malicious file |
| `/var/ossec/active-response/bin/cleanup-timeouts.sh` | `ActiveResponseManager.run_cleanup()` | Unblock IPs after timeout expires |

---

## Active Response Actions

### Decision Logic by Rule Level

| Alert Level | Actions Taken |
|---|---|
| **CRITICAL** (15–16) | 🔥 Block IP + 📧 Email alert + 📝 Log + 🔒 Quarantine (if file rule) |
| **HIGH** (12–14) | 🔥 Block IP + 📧 Email (once per IP) + 📝 Log |
| **MEDIUM** (8–11) | 📊 Track + 📝 Log + 📧 Email (once) |
| **LOW** (5–7) | 📝 Log + 📊 Track |
| **Failed Logins** | 📊 Track count → 🔥 Block at threshold |

### Block Timeout Policy

| Threat Category | Block Duration |
|---|---|
| TOR node detected | 2 hours |
| Critical inbound score | 1 hour |
| Dangerous inbound score | 30 minutes |
| SSH brute force | 1 minute (Wazuh default) |
| MITRE recon rules | 1 minute (Wazuh default) |
| Account lockout | 5 minutes |

---

## Criminal IP Rules

### Inbound Score Severity Ladder

```
Safe ──► Low ──► Moderate ──► Dangerous ──► Critical
 L2      L3        L6            L9           L10
```

### Composite Rule Threat Levels

```
TOR + Critical Score     ──► Level 14  🔴 AUTO-BLOCK
Dark Web + Critical      ──► Level 14  🔴 AUTO-BLOCK
Scanner + Critical       ──► Level 13  🔴 AUTO-BLOCK
Dark Web + Scanner       ──► Level 13  🔴 AUTO-BLOCK
Snort + Critical         ──► Level 13  🔴 AUTO-BLOCK
Outbound TOR (C2)        ──► Level 13  🔴 AUTO-BLOCK
TOR + Dangerous          ──► Level 12  🟠 AUTO-BLOCK
Anonymous VPN + Critical ──► Level 12  🟠 AUTO-BLOCK
Proxy + Dangerous        ──► Level 11  🟠 AUTO-BLOCK
Scanner + Dangerous      ──► Level 11  🟠 AUTO-BLOCK
```

---

## MITRE ATT&CK Mapping

| Technique ID | Technique Name | Mapped Rules |
|---|---|---|
| `T1590.005` | IP Scanning | `101020` |
| `T1595.002` | Vulnerability Scanning | `101002`, `101003` |
| `T1090.003` | Multi-hop Proxy (TOR) | `100625`, `100638`, `100650`, `100651`, `100663` |
| `T1090` | Proxy | `100626`, `100658` |
| `T1041` | Exfiltration Over C2 | `100660`, `100661`, `100662`, `100663` |
| `T1583` | Acquire Infrastructure | `100627`, `100654`, `100655` |
| `T1595` | Active Scanning | `100635`, `100636`, `100652`, `100653`, `100654` |
| `T1190` | Exploit Public-Facing App | `100800`, `101002` |
| `T1059` | Command and Scripting | `100101`, `100113`, `100204`, `100205` |
| `T1068` | Exploitation for Privilege Escalation | `100208` |
| `T1140` | Deobfuscate/Decode Files | `100203` |
| `T1204` | User Execution | `100201`, `100302` |
| `T1498` | Network DoS | `100700` |

---

## Log & Storage Paths

| Path | Purpose |
|---|---|
| `/home/wazuh-user/backup/log/soar-engine.log` | Main SOAR engine log |
| `/home/wazuh-user/backup/log/blocked_ips.log` | All IP block records |
| `/home/wazuh-user/backup/log/sent_alerts.json` | Deduplication database |
| `/home/wazuh-user/backup/log/stats.json` | Runtime statistics |
| `/home/wazuh-user/backup/log/quarantine.log` | File quarantine actions |
| `/home/wazuh-user/backup/log/firewall.log` | Firewall rule changes |
| `/home/wazuh-user/backup/log/disable-account.log` | Account lockout actions |
| `/home/wazuh-user/backup/log/cleanup.log` | Block expiry cleanup |
| `/home/wazuh-user/backup/soar.db` | SQLite persistent database |
| `/var/ossec/logs/integrations.log` | Criminal IP API debug log |
| `/var/ossec/logs/active-responses.log` | Active response execution log |

---

## Email Configuration

| Setting | Value |
|---|---|
| SMTP Username | `sop98886@gmail.com` |
| SMTP Password | `your-email-app-key` |
| Alert Recipient | `sopanha.tith@student.cadt.edu.kh` |
| Email Trigger Level | Level ≥ 10 (configurable) |
| Critical Email Trigger | Level ≥ 15 via `syslog_output` |

---

## Features

| Feature | Description |
|---|---|
| **SQLite Database** | Persistent storage of all alerts, actions, and block history |
| **Health Checks** | Automatic monitoring of all Wazuh components on startup |
| **Statistics Tracking** | Real-time counters with historical trend data |
| **Enhanced Deduplication** | Per-rule, per-IP deduplication backed by SQLite |
| **Failed Login Tracking** | Cross-session brute force attempt counting |
| **Expired Block Cleanup** | Auto-unblocks IPs after configured timeout |
| **Suricata Integration** | Parses alerts directly from `/var/log/suricata/eve.json` |
| **MITRE ATT&CK Mapping** | Full technique tagging on all 65+ rules |
| **Console Logging** | Color-formatted real-time output |
| **Email Templates** | Professional HTML alert formatting |
| **Config Validation** | Verifies all file paths and permissions on startup |
| **Criminal IP Enrichment** | Automatic IP reputation lookup on network-based alerts |
| **Composite Threat Rules** | Multi-factor correlation (TOR + Critical, DarkWeb + Scanner, etc.) |

---

## Compatibility Checklist

| Component | Status |
|---|---|
| All 65+ local rules mapped | ✅ |
| All 3 active response scripts integrated | ✅ |
| All backup/log paths respected | ✅ |
| All MITRE ATT&CK IDs preserved | ✅ |
| Email configuration applied | ✅ |
| Criminal IP ruleset (Sections 1–5) covered | ✅ |
| No existing config files modified | ✅ |
| Wazuh Manager Amazon Linux 2023 compatible | ✅ |
