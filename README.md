# 🛡️ Endpoint Security & Monitoring with Wazuh EDR

> **Internee.pk Cybersecurity Internship — Task #1**

## 📌 Objective

Protect Internee.pk's devices and servers from malware and unauthorized access by deploying an Endpoint Detection and Response (EDR) solution using **Wazuh**, monitoring file integrity and user activity, and automating alerts for suspicious behavior.

---

## 🏗️ Architecture Overview

```
┌────────────────────────────────────────Q─────────────┐
│                   MONITORED ENDPOINTS               │
│                                                     │
│   Windows Host          Linux Server                │
│   ┌──────────┐          ┌──────────┐                │
│   │  Sysmon  │          │  Auditd  │                │
│   │  Agent   │          │  Agent   │                │
│   └────┬─────┘          └────┬─────┘                │
└────────┼────────────────────┼─────────────────────-─┘
         │    Encrypted        │
         │    Log Stream       │
         ▼                     ▼
┌─────────────────────────────────────────────────────┐
│                  WAZUH MANAGER                      │
│                                                     │
│   ┌─────────────┐    ┌──────────────────────┐       │
│   │  Wazuh      │    │  Threat Intelligence  │       │
│   │  Indexer    │    │  (MalwareBazaar Feed) │       │
│   └─────────────┘    └──────────────────────┘       │
│                                                     │
│   ┌─────────────────────────────────────────┐       │
│   │         Custom Detection Rules          │       │
│   │  • Brute Force  • Malware  • Privilege  │       │
│   └─────────────────────────────────────────┘       │
│                                                     │
│   ┌─────────────┐    ┌──────────────────────┐       │
│   │   Wazuh     │    │   Alert Engine       │       │
│   │  Dashboard  │    │   (Email/Webhook)    │       │
│   └─────────────┘    └──────────────────────┘       │
└─────────────────────────────────────────────────────┘
```

---

## 🔧 Components Used

| Component | Purpose | Version |
|-----------|---------|---------|
| Wazuh Manager | Central log collection & analysis | 4.7.x |
| Wazuh Agent | Endpoint monitoring | 4.7.x |
| Wazuh Dashboard | Web UI for alerts & visualization | 4.7.x |
| Sysmon | Windows process & network monitoring | 15.x |
| MalwareBazaar | Threat intelligence feed | API v1 |

---

## 📁 Repository Structure

```
endpoint-security-monitoring/
│
├── README.md                        # Project overview (this file)
│
├── configs/
│   ├── wazuh-agent.conf             # Wazuh agent configuration
│   ├── ossec.conf                   # Wazuh manager configuration
│   └── sysmon-config.xml            # Sysmon monitoring rules
│
├── rules/
│   ├── custom-rules.xml             # Custom Wazuh detection rules
│   └── fim-rules.xml                # File Integrity Monitoring rules
│
├── scripts/
│   ├── malwarebazaar-feed.py        # Threat intel automation script
│   └── alert-webhook.py            # Webhook alerting script
│
└── docs/
    ├── setup-guide.md               # Step-by-step installation guide
    └── alert-samples.md             # Sample alerts triggered
```

---

## ⚙️ Configuration

### 1. Wazuh Agent (`configs/wazuh-agent.conf`)
Configured to:
- Monitor Windows Event Logs (System, Security, Application)
- Track Sysmon logs for process creation and network connections
- Perform File Integrity Monitoring on critical directories

### 2. Sysmon (`configs/sysmon-config.xml`)
Configured to capture:
- Process creation with full command lines
- Network connections (inbound & outbound)
- File creation and modification
- Registry changes

### 3. Custom Detection Rules (`rules/custom-rules.xml`)
Rules written to detect:
- **Brute force attacks** — multiple failed logins within 60 seconds
- **Privilege escalation** — unauthorized admin activity
- **Malware indicators** — known bad hashes from MalwareBazaar
- **Lateral movement** — unusual remote connections
- **Data exfiltration** — large outbound data transfers

---

## 🔍 Detection Rules Explained

### Rule: Brute Force Detection
Triggers when more than 5 failed login attempts occur within 60 seconds from the same source IP. Severity level: **High (12)**.

### Rule: Malware Hash Match
Cross-references file hashes against MalwareBazaar's database. Any match triggers an immediate critical alert. Severity level: **Critical (15)**.

### Rule: Privilege Escalation
Detects when a non-admin user runs commands with elevated privileges or modifies admin groups. Severity level: **High (12)**.

### Rule: Suspicious PowerShell
Flags encoded PowerShell commands, download cradles, and known malicious PowerShell patterns. Severity level: **High (13)**.

---

## 🧬 Threat Intelligence Integration

The `scripts/malwarebazaar-feed.py` script:

1. Queries the MalwareBazaar API every 6 hours
2. Downloads the latest malware hashes (SHA256)
3. Formats them into Wazuh-compatible IOC lists
4. Automatically reloads Wazuh rules to apply updates

**Data Source:** [MalwareBazaar by abuse.ch](https://bazaar.abuse.ch/)

---

## 🚨 Alerting System

Alerts are configured at three severity tiers:

| Level | Severity | Action |
|-------|----------|--------|
| 7–9 | Medium | Logged to dashboard |
| 10–12 | High | Email notification sent |
| 13–15 | Critical | Immediate webhook + email |

---

## 📊 Sample Alerts

| Alert | Rule ID | Severity | Description |
|-------|---------|----------|-------------|
| Brute Force Detected | 100002 | High | 10 failed SSH logins in 30s from 192.168.1.105 |
| Malware Hash Match | 100005 | Critical | Known ransomware hash detected on endpoint |
| Privilege Escalation | 100003 | High | Non-admin user added to Administrators group |
| Suspicious PowerShell | 100004 | High | Encoded PowerShell execution detected |
| FIM Alert | 100001 | Medium | Critical system file modified: /etc/passwd |

---

## 🚀 Setup Guide

See [`docs/setup-guide.md`](docs/setup-guide.md) for full installation instructions.

**Quick summary:**
1. Deploy Wazuh Manager on Ubuntu Server (4GB RAM, 50GB disk)
2. Install Wazuh Agent on endpoints
3. Deploy Sysmon on Windows endpoints with provided config
4. Apply custom detection rules
5. Configure MalwareBazaar feed script (cron job)
6. Set up alerting webhooks

---

## 🧠 Key Learnings

- **EDR vs Antivirus:** EDR solutions like Wazuh go beyond signature-based detection by monitoring behavior in real time
- **Log centralization** is critical for correlating events across multiple endpoints
- **Custom rules** allow tuning detection to your specific environment, reducing false positives
- **Threat intelligence feeds** keep detection capabilities current against emerging malware
- **File Integrity Monitoring (FIM)** provides an early warning for ransomware and unauthorized changes

---

## 📚 References

- [Wazuh Official Documentation](https://documentation.wazuh.com)
- [Sysmon by Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [MalwareBazaar by abuse.ch](https://bazaar.abuse.ch/)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)

---

## 👤 Author

**Cybersecurity Intern @ Internee.pk**  
Task #1 — Endpoint Security & Monitoring  
