# Troubleshooting Log

[← Previous: Security Configuration Assessment](04-security-configuration-assessment.md) | [README](../README.md)

Real problems I hit while building and testing the lab, and how I diagnosed each one. Every entry follows the same pattern: **symptom → investigation → root cause → fix → lesson**.

## Index

| # | Problem | Root cause | Area |
|---|---|---|---|
| 1 | Dashboard installation failed | Indexer security not initialised | Deployment |
| 2 | Dashboard: "server is not ready yet" | Indexer failed (log directory missing, then wrong ownership) | Deployment |
| 3 | `sudo` warning on Kali: "unable to resolve host" | `/etc/hosts` used the old hostname | Endpoint setup |
| 4 | First FIM test produced no alert | Test ran before real-time FIM was active | FIM |
| 5 | Kali agent could not send events | Wazuh Manager was stopped | FIM |
| 6 | Alerts not reaching the Dashboard | Filebeat was stopped | FIM |

---

## 1. Dashboard installation failed

**Symptom:** The first attempt to install the Wazuh Dashboard failed.

**Investigation:** I read the installer's error message instead of retrying.

**Root cause:** The Indexer security configuration had not been initialised.

**Fix:**
```bash
sudo bash wazuh-install.sh --start-cluster
sudo bash wazuh-install.sh --wazuh-dashboard dashboard
```

**Lesson:** Installation order matters. The installer's error message named the missing prerequisite.

---

## 2. Dashboard: "Wazuh dashboard server is not ready yet"

**Symptom:** The Dashboard web page loaded but displayed that the server was not ready.

**Investigation:**
```bash
sudo systemctl status wazuh-dashboard --no-pager
sudo systemctl status wazuh-indexer --no-pager
sudo journalctl -u wazuh-indexer -n 100 --no-pager | grep -iE "error|exception|fatal"
```

| Check | Finding |
|---|---|
| Dashboard service | Active, so not the cause |
| Indexer service | **Failed** |
| Indexer logs | Could not open `/var/log/wazuh-indexer/gc.log` |

The directory was first missing. After creating it, the error became a permissions problem.

**Root cause:** Missing log directory, then incorrect ownership for the Indexer's service account.

**Fix:**
```bash
sudo mkdir -p /var/log/wazuh-indexer
id wazuh-indexer                                      # UID 997, GID 984
sudo chown wazuh-indexer:wazuh-indexer /var/log/wazuh-indexer
sudo systemctl restart wazuh-indexer
sudo systemctl status wazuh-indexer --no-pager
```

**Lesson:** The symptom appeared in the Dashboard, but the cause was one layer down in the Indexer. Work backwards from the symptom and use logs and permissions, rather than restarting repeatedly without evidence.

---

## 3. `sudo`: "unable to resolve host kali-lab"

**Symptom:** Every `sudo` command on Kali printed a hostname resolution warning.

**Investigation:** `cat /etc/hosts` showed the local mapping still used the previous hostname.

**Root cause:** The hostname had been changed but `/etc/hosts` was not updated.

**Fix:** Edited `/etc/hosts`:
```
127.0.1.1 kali-lab
```
Then confirmed with `sudo echo "Hostname resolution working"`.

**Lesson:** A hostname lives in more than one place. Check that they agree.

---

## 4. First FIM test produced no alert

**Symptom:** A file was created in the monitored directory but nothing appeared in the Dashboard.

**Investigation:**
```bash
sudo grep -i "syscheck" /var/ossec/logs/ossec.log | tail -20
sudo grep -iE "File integrity monitoring scan (started|ended)" /var/ossec/logs/ossec.log | tail -10
```
The log showed the directory was configured for real-time monitoring, but the initial FIM scan had started and **not yet finished**.

**Root cause:** The test file was created before real-time monitoring was fully active.

**Fix:** Wait for the scan to complete, confirm real-time monitoring has started, then run the test.

**Lesson:** Timing matters. Verify a monitoring feature is *running* before testing it.

---

## 5. Kali agent could not send events: Manager stopped

**Symptom:** Still no events after the scan finished. The Kali agent log showed connection failures to TCP 1514 and 1515.

**Investigation:**
```bash
sudo systemctl status wazuh-manager --no-pager -l
```
The Manager was **inactive (dead)**.

**Root cause:** The Wazuh Manager service was not running, so there was nothing on the server to receive agent events, even though the agent service on Kali was healthy.

**Fix:**
```bash
sudo systemctl enable --now wazuh-manager
sudo ss -lntp | grep -E ':1514|:1515'
```

**Verification:** Port 1514 listening via `wazuh-remoted`, port 1515 via `wazuh-authd`. The Kali log then showed it connected to 192.168.58.131:1514 and came online.

**Lesson:** Checking the ports separated a *Manager communication* problem from an *FIM configuration* problem.

---

## 6. Alerts not reaching the Dashboard: Filebeat stopped

**Symptom:** With the Manager running, events still needed to reach the Dashboard.

**Investigation:**
```bash
sudo filebeat test output
sudo systemctl status filebeat --no-pager
```

| Check | Finding |
|---|---|
| `filebeat test output` | Successfully reached `https://192.168.58.131:9200` (DNS, connection, TLS, communication all passed) |
| Filebeat service | **Inactive (dead)** |

**Root cause:** The connection to the Indexer was fine; the service itself simply was not running. Alerts could be generated locally but were never forwarded.

**Fix:**
```bash
sudo systemctl enable --now filebeat
```
`enable` also makes Filebeat start at boot so this doesn't recur after a reboot.

**Lesson:** Checking the local alerts file (`/var/ossec/logs/alerts/alerts.json`) confirms whether an event was generated, which tells you whether the problem is detection or delivery.

---

## General Troubleshooting Approach

1. **Read the evidence first.** Check service status, then logs, before restarting anything.
2. **Work along the data path.**
   `Agent → Manager → Filebeat → Indexer → Dashboard`
   Test each link in turn.
3. **Separate configuration from connectivity.** Ports, service state and config validation are different questions.
4. **Validate before applying.** Use the built-in config tests (`wazuh-analysisd -t`, `wazuh-syscheckd -t`, `sshd -t`).
5. **Verify the fix.** Confirm the service is active and the original symptom has gone.
6. **Record it.** Writing down the cause and lesson makes the problem faster to solve next time.
