# Command Reference

[← Back to README](README.md)

A consolidated, task-grouped list of the meaningful commands used in this lab. Routine actions (opening an app, pressing Enter) are left out, and repeated commands are listed once.

**Host key:** `[ubuntu-soc]` = Wazuh server (192.168.58.131) · `[kali-lab]` = Wazuh agent (192.168.58.130)

## Contents

1. [System baseline](#1-system-baseline-ubuntu-soc)
2. [Disk expansion](#2-disk-expansion-ubuntu-soc)
3. [Hostname and VM control](#3-hostname-and-vm-control)
4. [SSH setup](#4-ssh-setup-ubuntu-soc)
5. [Wazuh server installation](#5-wazuh-server-installation-ubuntu-soc)
6. [Service health checks](#6-service-health-checks-ubuntu-soc)
7. [Indexer troubleshooting](#7-indexer-troubleshooting-ubuntu-soc)
8. [Kali endpoint and agent](#8-kali-endpoint-and-wazuh-agent-kali-lab)
9. [SSH brute-force investigation](#9-ssh-brute-force-investigation)
10. [Wazuh rule investigation](#10-wazuh-rule-investigation-ubuntu-soc)
11. [SSH and account review](#11-ssh-and-account-review-ubuntu-soc)
12. [File Integrity Monitoring](#12-file-integrity-monitoring-kali-lab)
13. [Manager, port and Filebeat troubleshooting](#13-manager-port-and-filebeat-troubleshooting)
14. [Security Configuration Assessment](#14-security-configuration-assessment-kali-lab)

---

## 1. System baseline `[ubuntu-soc]`

```bash
lsb_release -a                       # confirm OS version (Ubuntu 24.04.4 LTS)
sudo apt update                      # refresh package lists
sudo apt upgrade -y                  # install available upgrades
sudo apt install curl wget git vim net-tools unzip -y   # lab utilities
ip addr                              # network interface and IP
ping -c 4 google.com                 # test internet connectivity
free -h                              # memory
lscpu                                # CPU details
nproc                                # CPU core count
uptime                               # load average
```

## 2. Disk expansion `[ubuntu-soc]`

```bash
df -h                                # disk usage
df -h /                              # root filesystem usage
lsblk                                # block devices and partitions
findmnt -no FSTYPE /                 # filesystem type of /
sudo apt install cloud-guest-utils -y   # provides growpart
sudo growpart /dev/sda 2             # expand partition 2
sudo resize2fs /dev/sda2             # expand the filesystem into it
```

## 3. Hostname and VM control

```bash
hostname                             # show hostname
hostnamectl                          # detailed host information
sudo hostnamectl set-hostname ubuntu-soc   # set a meaningful hostname
sudo shutdown now                    # power off cleanly before a VM snapshot
```

## 4. SSH setup `[ubuntu-soc]`

```bash
sudo apt install openssh-server -y
sudo systemctl enable --now ssh      # start now and at boot
sudo systemctl status ssh --no-pager
```

## 5. Wazuh server installation `[ubuntu-soc]`

```bash
# Download installer and config template
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.14/config.yml
ls -lh wazuh-install.sh
ls -lh config.yml

# Review and edit node IPs
cat config.yml
nano config.yml

# Install in order
sudo bash wazuh-install.sh --generate-config-files          # certificates and keys
sudo bash wazuh-install.sh --wazuh-indexer node-1           # storage and search
sudo bash wazuh-install.sh --wazuh-server wazuh-1           # manager (+ Filebeat)
sudo bash wazuh-install.sh --start-cluster                  # initialise Indexer security
sudo bash wazuh-install.sh --wazuh-dashboard dashboard      # web interface
```

## 6. Service health checks `[ubuntu-soc]`

```bash
sudo systemctl status wazuh-indexer --no-pager
sudo systemctl status wazuh-manager --no-pager
sudo systemctl status wazuh-dashboard --no-pager
```

## 7. Indexer troubleshooting `[ubuntu-soc]`

```bash
# Read the service logs
sudo journalctl -u wazuh-indexer -n 30 --no-pager
sudo journalctl -u wazuh-indexer -n 50 --no-pager
sudo journalctl -u wazuh-indexer -n 100 --no-pager | grep -iE "error|exception|fatal"

# Fix missing directory and ownership
sudo mkdir -p /var/log/wazuh-indexer
ls -ld /var/log/wazuh-indexer
id wazuh-indexer                                       # confirm service account
sudo chown wazuh-indexer:wazuh-indexer /var/log/wazuh-indexer
sudo systemctl restart wazuh-indexer
```

## 8. Kali endpoint and Wazuh agent `[kali-lab]`

```bash
# Identity
hostname
ip addr
ip -4 addr

# Fix "unable to resolve host" warning
cat /etc/hosts
sudo nano /etc/hosts                  # add: 127.0.1.1 kali-lab
sudo echo "Hostname resolution working"

# Add the Wazuh repository and signing key
sudo apt update
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update

# Install and configure the agent (version matches the server)
sudo apt install wazuh-agent=4.14.7-1
sudo nano /var/ossec/etc/ossec.conf                    # set manager address
sudo grep -A 3 "<client>" /var/ossec/etc/ossec.conf    # verify it

# Start and verify
sudo systemctl enable --now wazuh-agent
sudo systemctl status wazuh-agent --no-pager
```

## 9. SSH brute-force investigation

```bash
# From Kali: generate normal and failed logins
ssh manasi@192.168.58.131                              # [kali-lab]

# On Ubuntu: read authentication activity
sudo tail -n 10 /var/log/auth.log                      # [ubuntu-soc]
sudo grep "Failed password" /var/log/auth.log
sudo grep "Accepted password" /var/log/auth.log

# SSH service journal
sudo journalctl -u ssh --no-pager | grep "Failed password" | tail -10

# Timeline for the incident window
sudo journalctl -u ssh --since "2026-09-05 02:20:00" --until "2026-09-05 02:27:00" --no-pager | grep "Failed password"

# Did any attempt succeed?
sudo journalctl -u ssh --since "2026-09-05 02:20:00" --until "2026-09-05 02:27:00" --no-pager | grep "Accepted"

# Add auth.log to Wazuh monitoring
sudo grep -n -A 3 -B 2 "/var/log/auth.log" /var/ossec/etc/ossec.conf   # already configured?
sudo nano /var/ossec/etc/ossec.conf                    # add <localfile> block
sudo /var/ossec/bin/wazuh-analysisd -t                 # validate configuration
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager --no-pager
```

## 10. Wazuh rule investigation `[ubuntu-soc]`

```bash
# Rule 5760: single SSH authentication failure
sudo grep -R -n 'id="5760"' /var/ossec/ruleset/rules/
sudo sed -n '455,475p' /var/ossec/ruleset/rules/0095-sshd_rules.xml

# Rule 5763: brute-force correlation
sudo grep -R -n -i "brute force" /var/ossec/ruleset/rules/ | head -20
sudo sed -n '475,490p' /var/ossec/ruleset/rules/0095-sshd_rules.xml
```

## 11. SSH and account review `[ubuntu-soc]`

```bash
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin'   # effective SSH settings
sudo sshd -T | grep '^permitrootlogin'
sudo passwd -S manasi                                  # account status (locked or not)
```

## 12. File Integrity Monitoring `[kali-lab]`

```bash
# Review existing configuration
sudo sed -n '/<syscheck>/,/<\/syscheck>/p' /var/ossec/etc/ossec.conf

# Create the test directory and monitor it
mkdir -p ~/fim-lab
ls -ld ~/fim-lab
sudo nano /var/ossec/etc/ossec.conf                    # add <directories realtime="yes">

# Validate and restart
sudo /var/ossec/bin/wazuh-syscheckd -t
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent --no-pager

# First (premature) test
echo "SOC FIM test" > ~/fim-lab/test-file.txt
ls -l ~/fim-lab/

# Check whether the scan had finished
sudo grep -i "syscheck" /var/ossec/logs/ossec.log | tail -20
sudo grep -iE "File integrity monitoring scan (started|ended)" /var/ossec/logs/ossec.log | tail -10

# Clean tests, run after real-time FIM is active
echo "FIM detection test" > ~/fim-lab/final-test.txt          # create
echo "Second line added" >> ~/fim-lab/final-test.txt          # modify
echo "FIM deletion test" > ~/fim-lab/deletion-test.txt        # prepare
rm ~/fim-lab/deletion-test.txt                                # delete

# Verify alerts were generated (run on Ubuntu, the Manager)
sudo grep -i "final-test.txt" /var/ossec/logs/alerts/alerts.json | tail -5
sudo grep -i "deletion-test.txt" /var/ossec/logs/alerts/alerts.json | tail -5
```

## 13. Manager, port and Filebeat troubleshooting

```bash
# Manager [ubuntu-soc]
sudo systemctl status wazuh-manager --no-pager -l
sudo systemctl enable --now wazuh-manager

# Agent communication ports [ubuntu-soc]
sudo ss -lntp | grep -E ':1514|:1515'                  # 1514 = remoted, 1515 = authd

# Agent log [kali-lab]
sudo tail -20 /var/ossec/logs/ossec.log

# Filebeat [ubuntu-soc]
sudo filebeat test output                              # can it reach the Indexer?
sudo systemctl status filebeat --no-pager
sudo systemctl enable --now filebeat
```

## 14. Security Configuration Assessment `[kali-lab]`

```bash
# Review SCA settings
sudo sed -n '/<sca>/,/<\/sca>/p' /var/ossec/etc/ossec.conf

# Verify the effective SSH value (before)
sudo sshd -T | grep '^maxauthtries'

# Find where the value is defined
sudo grep -nE '^[[:space:]]*#?[[:space:]]*MaxAuthTries' /etc/ssh/sshd_config
sudo grep -RniH -i 'MaxAuthTries' /etc/ssh/

# Remediate
sudo nano /etc/ssh/sshd_config                         # change #MaxAuthTries 6 to MaxAuthTries 4

# Validate, verify, apply
sudo sshd -t                                           # no output = syntax OK
sudo sshd -T | grep '^maxauthtries'                    # should now show 4
sudo systemctl restart ssh
sudo systemctl status ssh --no-pager

# Trigger and check a fresh SCA scan
sudo systemctl restart wazuh-agent
sudo tail -30 /var/ossec/logs/ossec.log | grep -iE "sca|Evaluation"
```

---

## Quick Patterns Worth Remembering

| Goal | Command pattern |
|---|---|
| Check a service | `sudo systemctl status <service> --no-pager` |
| Read a service's logs | `sudo journalctl -u <service> -n 100 --no-pager` |
| Filter for problems | `... \| grep -iE "error\|exception\|fatal"` |
| Start now and at boot | `sudo systemctl enable --now <service>` |
| Validate before restarting | `wazuh-analysisd -t`, `wazuh-syscheckd -t`, `sshd -t` |
| See what is listening | `sudo ss -lntp` |
| Confirm effective SSH config | `sudo sshd -T` |
