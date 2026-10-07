# Incident Report: SSH Brute-Force Detection (Controlled Lab Test)

[← Back to README](../README.md) | [Full investigation write-up](../docs/02-ssh-bruteforce-investigation.md)

| Field | Detail |
|---|---|
| **Report ID** | IR-LAB-001 |
| **Report type** | Post-incident report (controlled detection test) |
| **Analyst** | Manasi |
| **Date of activity** | 5 September 2026 |
| **Detection platform** | Wazuh 4.14.7 |
| **Classification** | Credential access attempt: brute force (MITRE ATT&CK T1110) |
| **Severity (Wazuh)** | Level 10 |
| **Final status** | Closed. No evidence of successful compromise |
| **Environment** | Isolated home lab. No production systems involved |

> This was a deliberate, controlled simulation run in a lab I own. It is written as a real incident report to practise SOC reporting.

---

## 1. Executive Summary

Wazuh detected repeated failed SSH login attempts against the `ubuntu-soc` server from the host `192.168.58.130` (`kali-lab`), targeting the account `manasi`. The activity triggered correlation Rule 5763 (Level 10, mapped to MITRE T1110 Brute Force).

I validated the alert against the original SSH logs, identified the source and targeted account, and checked the incident window for successful authentication. **No successful login was found.** The incident was closed as a controlled brute-force detection test with no evidence of compromise.

---

## 2. Scope and Affected Assets

| Asset | Role | IP | Involvement |
|---|---|---|---|
| `kali-lab` | Wazuh agent / controlled test host | 192.168.58.130 | **Source** of failed attempts |
| `ubuntu-soc` | Wazuh server / SSH target | 192.168.58.131 | **Target** of failed attempts |
| Account `manasi` | Local user | n/a | Targeted account |

---

## 3. Detection

| Item | Detail |
|---|---|
| Individual detection | Rule **5760**, Level 5: *sshd: authentication failed* |
| Correlation detection | Rule **5763**, Level 10: *sshd: brute force trying to get access to the system* |
| Rule logic | 8 matching Rule 5760 events, from the same source IP, within 120 seconds |
| Decoder | `sshd` |
| Log source | `journald` and `/var/log/auth.log` |
| MITRE ATT&CK | T1110, Brute Force (tactic: Credential Access) |

**How it was found:** Wazuh generated the Rule 5763 alert after repeated Rule 5760 failures. I reviewed it in the Wazuh Dashboard (Threat Hunting and Events views).

---

## 4. Timeline

| Date / time | Event | Source |
|---|---|---|
| 4 Sept 2026 | Legitimate successful SSH login by `manasi` from 192.168.58.130 (baseline, not part of the incident) | Ubuntu auth log |
| 5 Sept 2026, ~02:22:47 | First failed SSH password attempts from 192.168.58.130 appear in the investigated window | Ubuntu SSH journal |
| 5 Sept 2026, ~02:22:47 to ~02:25:00 | Repeated `Failed password` events for `manasi` from 192.168.58.130 | Ubuntu SSH journal |
| During the above | Rule 5760 fires on individual failures; Rule 5763 (Level 10) fires on correlation | Wazuh |
| After detection | Investigated: source IP, user, decoder, original log, surrounding events | Wazuh Dashboard |
| After detection | Searched the window for `Accepted` events: **none found** | Ubuntu SSH journal |

**Evidence-handling note:** The visible terminal lines span slightly more than 120 seconds, and the Wazuh surrounding view can display repeated timestamp representations. I therefore do not claim those exact lines were the eight events used by the correlation rule. The Rule 5763 alert itself confirms the correlation condition was met.

---

## 5. Indicators

| Type | Value |
|---|---|
| Source IP | 192.168.58.130 |
| Targeted user | `manasi` |
| Target host / service | `ubuntu-soc`, SSH (sshd) |
| Authentication method | Password |
| Log pattern | `Failed password for manasi from 192.168.58.130 ...` |
| Wazuh rules | 5760, 5763 |

Note: `agent.name = ubuntu-soc` shows where the log was *collected*. The attacker-side address is `data.srcip = 192.168.58.130`.

---

## 6. Investigation Steps

1. **Reviewed the alert:** Rule 5763, Level 10, T1110.
2. **Validated at source:** confirmed the failures in Ubuntu's own SSH journal (`journalctl -u ssh`), not just in the SIEM.
3. **Identified key fields:** source IP, target user, decoder, original log, surrounding events.
4. **Read the detection logic:** reviewed Rules 5760 and 5763 in `0095-sshd_rules.xml` to understand why the alert fired.
5. **Built a timeline:** from SSH journal entries for the incident window.
6. **Checked for success:** searched the same window for `Accepted` authentication events.
7. **Reviewed configuration and account:** checked effective SSH settings (`sshd -T`) and account status (`passwd -S`).

---

## 7. Findings

| Question | Answer |
|---|---|
| Was the activity real? | Yes. Failures were confirmed in Ubuntu's SSH logs |
| What was the source? | 192.168.58.130 (`kali-lab`, controlled lab host) |
| What was targeted? | SSH service on `ubuntu-soc`, account `manasi` |
| Did it succeed? | **No.** No `Accepted` events in the incident window |
| Was the account compromised? | No evidence. Account had a password set and was not locked |
| SSH configuration | Password authentication enabled; root password login not permitted |

---

## 8. Impact Assessment

| Area | Assessment |
|---|---|
| Confidentiality | No evidence of unauthorised access |
| Integrity | No evidence of changes |
| Availability | No impact |
| Overall | **No impact.** Detection test only |

---

## 9. Response Actions

| Stage | Action | Decision |
|---|---|---|
| Identify | Reviewed Rule 5763 alert | Confirmed repeated SSH failures |
| Validate | Checked the Ubuntu SSH journal | Failures real, from 192.168.58.130 |
| Contain | Considered blocking the source | **No permanent block.** The source is my own controlled lab host |
| Remediate | Reviewed SSH and account settings | No lab change. Production recommendations below |
| Recover | Checked for successful access | None found |
| Document | Recorded evidence and reasoning | This report |

---

## 10. Recommendations (If This Were a Production System)

| Priority | Recommendation | Reason |
|---|---|---|
| High | Use SSH key-based authentication and disable password authentication | Removes the credential type being guessed |
| High | Set `MaxAuthTries` to 4 or lower | Limits guesses per connection (CIS recommendation) |
| Medium | Configure Wazuh Active Response or a tool such as fail2ban to block repeat offenders | Automates containment |
| Medium | Restrict SSH access to known source addresses or a VPN | Reduces exposure |
| Medium | Alert on Rule 5763 followed by any `Accepted` login from the same IP | Catches a brute-force that succeeds |
| Low | Review lockout and password policy | Defence in depth |

A related hardening exercise (CIS check 36144, `MaxAuthTries`) is documented in [Part 4](../docs/04-security-configuration-assessment.md). It was performed on the Kali endpoint.

---

## 11. Conclusion

A controlled SSH password-guessing scenario was successfully detected by Wazuh. Individual failures were identified by Rule 5760 and correlated by Rule 5763 into a Level 10 alert mapped to MITRE ATT&CK T1110. The alert was validated against the original logs, the source and target were identified, and no successful authentication was found. The incident is closed as a controlled detection test with no evidence of compromise.

---

## Evidence References

| Figure | Description |
|---|---|
| [Fig 7](../images/ssh-bruteforce/fig07-ssh-successful-login.png) | Baseline successful SSH login |
| [Fig 8](../images/ssh-bruteforce/fig08-ssh-failed-logins-journal.png) | Failed logins in Ubuntu SSH journal |
| [Fig 10](../images/ssh-bruteforce/fig10-rule-5760-events.png) | Rule 5760 events |
| [Fig 11](../images/ssh-bruteforce/fig11-document-details-ssh-failure.png) | Alert fields: source IP, user, decoder |
| [Fig 12](../images/ssh-bruteforce/fig12-rule-5763-bruteforce-alert.png) | Rule 5763 alert, Level 10, T1110 |
| [Fig 13](../images/ssh-bruteforce/fig13-surrounding-documents.png) | Surrounding events |
