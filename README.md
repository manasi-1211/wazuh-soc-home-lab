# Wazuh SOC Home Lab

## SIEM Deployment, Security Monitoring and Incident Investigation

This is a personal cybersecurity lab that I built to gain practical experience with **Wazuh SIEM, Linux security monitoring, alert investigation and security hardening**.

I wanted to move beyond cybersecurity theory and build a small SOC environment where I could generate security events, see how Wazuh detected them, investigate the alerts and document what I found.

---

## What I Used

- Wazuh 4.14.7
- Ubuntu 24.04
- Kali Linux
- VMware Workstation
- Wazuh Agent
- journalctl
- grep
- SSH
- MITRE ATT&CK
- CIS Benchmarks

---

## Lab Setup

I created the lab using two virtual machines:

- **Ubuntu 24.04** – Wazuh Manager, Indexer, Dashboard and Filebeat
- **Kali Linux** – monitored endpoint

The Kali Linux machine was connected to Wazuh using the Wazuh Agent so that I could generate security events and investigate them from the Wazuh Dashboard.

### Basic Lab Flow

```text
Kali Linux
    |
    | Wazuh Agent
    |
    v
Ubuntu 24.04
    |
    +-- Wazuh Manager
    +-- Wazuh Indexer
    +-- Wazuh Dashboard
    +-- Filebeat
    |
    v
Security Alerts
    |
    v
Alert T riage & Investigation
