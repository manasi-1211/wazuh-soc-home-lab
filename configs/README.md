# Configuration Snippets

Sanitised excerpts of the configuration changes made in this lab. They show the exact settings used, without exposing anything sensitive.

| File | Host | What it is | Part |
|---|---|---|---|
| [`config.yml`](config.yml) | ubuntu-soc | Wazuh installer node configuration (single host) | 1 |
| [`ossec-agent-snippets.xml`](ossec-agent-snippets.xml) | kali-lab | Manager address, FIM test directory, SCA settings | 1, 3, 4 |
| [`ossec-manager-snippets.xml`](ossec-manager-snippets.xml) | ubuntu-soc | `auth.log` added as a monitored log source | 2 |
| [`sshd-hardening.conf`](sshd-hardening.conf) | kali-lab | `MaxAuthTries` change from 6 to 4 | 4 |

## What is deliberately not included

- The Wazuh administrator password and any generated credentials
- Certificates, private keys and the `wazuh-install-files.tar` archive
- Complete configuration files (only the changed sections are shown)

These files are for documentation. Do not copy them over a working `ossec.conf` or `sshd_config`; add the relevant block to your own file instead.
