# ELASTIC01: Elastic SIEM build

ELASTIC01 is the lab's SIEM: a self-hosted Elastic Stack 9.5.4 (Elasticsearch, Kibana and Fleet Server) on Ubuntu Server. It collects Sysmon and Windows event logs from the endpoint CLIENT01 through Elastic Agent. I chose self-hosting over an Elastic Cloud trial to work through the infrastructure myself: package repositories, TLS, enrolment, Fleet, time sync and patching.

Problems hit along the way are written up in [incident-reports.md](incident-reports.md); the numbers below (e.g. "incident 7") refer to that file.

## At a glance

| Area | Configuration |
|---|---|
| VM | Ubuntu Server 26.04.1 LTS, x86_64, 4 vCPU, 8 GB RAM, 80 GB NVMe (thin-provisioned), plain partitioning (no LVM) |
| Network | Static 192.168.125.12/24, no gateway, DNS 192.168.125.10 (DC01), search domain lab.local |
| Stack | Elasticsearch, Kibana, Elastic Agent / Fleet Server, all 9.5.4 |
| Access | SSH with keys only (ED25519, passphrase-protected), root login disabled; VMware console as the fallback |
| Time | chrony, synced from DC01 |
| Patching | OS patched; `elasticsearch` and `kibana` held in apt so OS updates never upgrade the stack |

## 1. Base OS and network

- Ubuntu Server installed with the OpenSSH server option. The installer suggests LVM by default; I used plain partitioning instead.
- Static address in `/etc/netplan/00-installer-config.yaml`: 192.168.125.12/24, no gateway (the host-only network has none), DNS = DC01, search domain `lab.local`.
- A copy of that file is kept as `00-installer-config.yaml.bak`. It is the known-good static configuration for the temporary-internet procedure below.

### Temporary internet access (isolated network)

The lab network has no internet route. Anything new from the internet (packages, the Elastic Agent download, Kibana integrations, OS updates) needs a short, deliberate switch to VMware NAT:

1. Add DHCP to the netplan file and apply it:
   ```bash
   sudo sed -i '/dhcp6: true/i\      dhcp4: true' /etc/netplan/00-installer-config.yaml
   sudo netplan apply
   ```
   `sed -i '/pattern/i\text'` inserts a line before the matching one, here `dhcp4: true` above the existing `dhcp6: true`. `netplan apply` restarts the interface, so an SSH session over it drops (expected).
2. Switch the VM's network adapter to NAT, find the new address with `ip a` on the VM console, and reconnect SSH to that address.
3. Do the download or update.
4. **Revert the adapter to host-only first**, then restore the static file and apply it:
   ```bash
   sudo cp /etc/netplan/00-installer-config.yaml.bak /etc/netplan/00-installer-config.yaml
   sudo diff /etc/netplan/00-installer-config.yaml.bak /etc/netplan/00-installer-config.yaml && echo IDENTICAL
   sudo netplan apply
   ```
   Adapter first, because the static address is still configured: moving the VM back to the host-only network makes it reachable again immediately (incident 5).

Changing the VMware adapter type does not change the guest's own IP configuration. Both have to change together, on Windows and Linux guests alike (incident 5).

## 2. Elasticsearch

- Installed from Elastic's official APT repository. The repository's GPG key is stored in its own keyring (`/usr/share/keyrings/elasticsearch-keyring.gpg`) and the source line uses `signed-by=`, so that key is trusted **only** for Elastic's repository, not for every package source on the system.
- Security is configured automatically on first start: TLS certificates, a generated password for the `elastic` superuser, and a CA certificate at `/etc/elasticsearch/certs/http_ca.crt`.
- Verified end to end with an authenticated HTTPS request (`curl --cacert` pointing at that CA, logging in as `elastic`), which returned the cluster's information. `sudo` is needed even to read the CA: `/etc/elasticsearch` is `root:elasticsearch` with mode 750, and without it curl reports the file as missing rather than as a permissions problem (incident 10).
- The service is enabled at boot and came back on its own after a full VM reboot.

## 3. Kibana

- Installed from the same APT repository.
- `server.host: 0.0.0.0` in `/etc/kibana/kibana.yml`, so a browser on the host PC can reach it (the default only listens on localhost).
- Connected to Elasticsearch through the browser enrolment flow: an enrolment token (`elasticsearch-create-enrollment-token -s kibana`, valid for 30 minutes) plus a verification code from the Kibana service log.
- **Three encryption keys are required for Fleet**, and they are not set by default:
  ```bash
  sudo /usr/share/kibana/bin/kibana-encryption-keys generate --quiet --force
  ```
  The three output lines (`xpack.encryptedSavedObjects.encryptionKey`, `xpack.reporting.encryptionKey`, `xpack.security.encryptionKey`) go into `kibana.yml`, followed by `sudo systemctl restart kibana`. Without them, Fleet's setup retries forever (incident 7).
- Kibana itself runs on plain HTTP on port 5601; only Elasticsearch uses TLS. This is a lab trade-off (see "Known gaps").

## 4. Fleet Server

- Created through Kibana: Fleet, Agents, Add Fleet Server, **Advanced** tab, with the policy name "Fleet Server policy 1". The Quick Start tab failed because Fleet had already created a default policy with the same name (incident 8).
- Elastic Agent in Fleet Server mode, installed from the **Linux x86_64** tarball to `/opt/Elastic/Agent` as a systemd service. Kibana's generated command defaulted to the aarch64 (ARM) build, which would not run on this x86_64 VM (incident 9).
  ```bash
  sudo ./elastic-agent install --fleet-server-es=<elasticsearch-url> \
    --fleet-server-service-token=<token> --fleet-server-policy=<policy-id> \
    --fleet-server-es-ca-trusted-fingerprint=<CA fingerprint> \
    --fleet-server-port=8220 --install-servers
  ```
  `--fleet-server-es-ca-trusted-fingerprint` pins the self-signed CA by its SHA-256 fingerprint, so Fleet Server verifies Elasticsearch's certificate without the CA being installed system-wide.
- Fleet Server listens on `https://192.168.125.12:8220` and showed **Healthy** in Fleet.

## 5. Endpoint enrolment (CLIENT01)

- A separate agent policy, **Windows Endpoints**, kept apart from the Fleet Server policy.
- The official **Windows** integration, which collects the Sysmon Operational channel by default (plus Forwarded, PowerShell and PowerShell Operational). Adding it needed temporary internet access, because Kibana loads its integration list from Elastic's package registry (incident 6).
- The agent on CLIENT01 was enrolled with **`--insecure`**: Fleet Server's certificate comes from Elasticsearch's self-signed CA, which CLIENT01 doesn't trust (incident 14). The traffic stays encrypted; only certificate verification is skipped. Elastic documents this as acceptable for test environments, not production. The production fix is to distribute the CA to endpoints or use a certificate from a trusted CA.
- Verified: agent Healthy in Fleet; Sysmon documents arriving in `logs-*` under `data_stream.dataset : "windows.sysmon_operational"`, including process creation (event code 1) and DNS query events.

## 6. Time synchronisation

The lab has one time chain: host PC (internet NTP) -> VMware Tools -> DC01 -> CLIENT01 (Windows domain hierarchy) and ELASTIC01 (chrony).

- `systemctl status chrony systemd-timesyncd`: chrony active, timesyncd not installed.
- `grep -rn -E "^(pool|server)|confdir|sourcedir" /etc/chrony/` showed where configuration actually lives: sources in `/etc/chrony/sources.d/`, general settings in `/etc/chrony/conf.d/`.
- Ubuntu's internet pools commented out (unreachable on host-only), and DC01 added in `/etc/chrony/sources.d/lab-dc01.sources`:
  ```
  server 192.168.125.10 iburst
  ```
- chrony could reach DC01 but rejected it. `sudo chronyc ntpdata 192.168.125.10` showed a root dispersion of about 10 s, which is what a Windows DC using its own clock announces. chrony's default `maxdistance` is 3 s; Windows clients don't enforce such a limit. Fixed on the client side, without changing what the DC announces to every machine, in `/etc/chrony/conf.d/lab-maxdistance.conf`:
  ```
  maxdistance 16
  ```
- Result: `chronyc sources` shows `^* 192.168.125.10` and `timedatectl` reports `System clock synchronized: yes` (incident 12).

## 7. Hardening and maintenance

### SSH: keys only

Password logins can be brute-forced (MITRE ATT&CK T1110); a key pair can't realistically be guessed. Password authentication was left on during the build as a documented, deferred step, then removed:

1. **Prove the new path before removing the old one.** Generated an ED25519 key with a passphrase on the host PC, added the public key to the admin user's `~/.ssh/authorized_keys` (directory 700, file 600; sshd ignores keys with looser permissions), compared the fingerprint on both sides, and logged in with `-o PreferredAuthentications=publickey -o PasswordAuthentication=no`.
2. **Find where the setting really lives.** `sudo sshd -T` (effective configuration) still showed `passwordauthentication yes`. `/etc/ssh/sshd_config` includes `sshd_config.d/*.conf` near the top, and `50-cloud-init.conf` there sets `PasswordAuthentication yes`.
3. **Write a drop-in that is read first.** sshd uses the first value it reads, and included files load alphabetically, so a new `/etc/ssh/sshd_config.d/10-lab-hardening.conf` overrides the cloud-init file without editing it:
   ```
   PasswordAuthentication no
   KbdInteractiveAuthentication no
   PermitRootLogin no
   ```
4. `sudo sshd -t` (syntax check, no output = valid), `sudo sshd -T` (shows the new effective values), then `sudo systemctl reload ssh`. An existing SSH session and the VMware console stayed open as fallbacks throughout.
5. Verified with new connections: a password-only attempt gets `Permission denied (publickey).`; a key login works (incident 15).

### OS patching without upgrading the SIEM

A plain `apt upgrade` would also upgrade Elasticsearch and Kibana from Elastic's repository. Elasticsearch can't be downgraded and the stack has to be upgraded in a set order (Elasticsearch, then Kibana, then Fleet Server and agents), so stack upgrades must be separate, planned changes.

- Snapshot first, and prove the netplan backup is identical to the live file.
- Hold the stack, while still offline: `sudo apt-mark hold elasticsearch kibana`. Elastic Agent is a tarball install, so apt never touches it.
- After `apt update` (127 upgradable packages), a dry run proved nothing Elastic would change and nothing would be removed:
  ```bash
  apt-get -s upgrade --with-new-pkgs | grep -E '^Remv |^Inst (elasticsearch|kibana) '
  ```
  (no output = safe).
- The upgrade ran from the **VM console, not SSH**, because `openssh-server` and `netplan.io` were among the packages: a dropped session mid-upgrade could leave apt half-finished.
- Afterwards: `dpkg -l` shows `hi` (hold, installed) for both Elastic packages; reboot for the new kernel; `uname -r` confirms it.
- Re-checked that earlier work survived: SSH hardening (`sshd -T`), chrony source, holds, the static network, and both agents Healthy in Fleet (incident 13).

**Side effect worth knowing:** Ubuntu's `needrestart` restarted `elasticsearch`, `kibana` and `elastic-agent` during the upgrade even though the packages were held. OS patching still interrupts the SIEM, so it belongs in a maintenance window.

### Operational notes

- **Start order:** DC01 first (DNS and time), then ELASTIC01, then CLIENT01. Shut down in reverse.
- **Slow start:** on this 8 GB VM, Elasticsearch takes 5 to 12 minutes to reach "started" after boot, and Kibana shows "Unable to connect" until then. Check `systemctl status elasticsearch kibana` and listening ports (`ss -tlnp | grep -E '9200|5601'`) before troubleshooting (incident 11).

## 8. Detection engineering

### Visualisation

A Lens line chart, "CLIENT01 - Process Creation Events by Process (Sysmon)": data view `logs-*`, query `data_stream.dataset : "windows.sysmon_operational" and event.code : "1"`, `@timestamp` on the x-axis, count of records on the y-axis, broken down by the top 9 values of `process.name`. Exported in [`dashboards/`](../dashboards/).

### Rules

Both rules are custom query rules (KQL), created manually under Security > Rules > Detection rules (SIEM), severity Medium (risk score 47), running every 5 minutes with 1 minute of extra look-back. Exported in [`detections/`](../detections/).

#### Rule 1: Suspicious Encoded PowerShell from Explorer (ClickFix Pattern)

```
event.code : "1" and process.name : ("powershell.exe" or "pwsh.exe") and process.parent.name : "explorer.exe" and process.command_line : (*-enc* or *-EncodedCommand* or *-windowstyle*hidden* or *-w*hidden* or *iex* or *downloadstring* or *invoke-expression*)
```

- **MITRE ATT&CK:** Execution (TA0002): User Execution: Malicious Copy and Paste (T1204.004); Command and Scripting Interpreter: PowerShell (T1059.001).
- **Why ClickFix:** "Malicious Copy and Paste" is in Red Canary's 2026 Threat Detection Report top 10. Fake CAPTCHA and "fix this error" pages trick users into pasting a command into the Run dialog.
- **Why `explorer.exe` as the parent:** anything run from Win+R is started by Explorer, which is the defining trait of the paste-and-run chain.
- **Why `process.command_line`, not `process.args`:** `process.args` splits the command into separate array items (`-WindowStyle`, `Hidden`), so one wildcard can't match the phrase; the full command line can.
- **Why two techniques:** ATT&CK mapping is many-to-many. The rule observes both the user-execution step and the interpreter.
- **Known false positives:** help-desk instructions that have users paste PowerShell into the Run dialog.

#### Rule 2: Script Interpreter Spawned from Microsoft Office Application

```
event.code : "1" and process.parent.name : ("winword.exe" or "excel.exe" or "powerpnt.exe" or "outlook.exe") and process.name : ("powershell.exe" or "cmd.exe" or "wscript.exe" or "cscript.exe" or "mshta.exe")
```

- **MITRE ATT&CK:** User Execution: Malicious File (T1204.002); Command and Scripting Interpreter (T1059, the parent technique, because the rule covers several interpreters).
- **Why:** Office applications have almost no legitimate reason to start a script interpreter, so the parent-child pair is high-signal whatever the command.
- **Baselined first:** a broad "interpreter started" preview returned normal activity such as VMware Tools (`vmtoolsd.exe`) starting `cmd.exe`, which is why the Office-parent restriction matters.
- **Known false positives:** document automation add-ins and macro-based business workflows.

### Validation

Each rule went through three stages:

1. **Rule Preview on the base condition** (e.g. PowerShell started by Explorer): proves the fields and values exist, so realistic hits are expected.
2. **Rule Preview on the full condition** over normal activity: zero hits expected. This is the negative control.
3. **Live test on CLIENT01**, using harmless reproductions of the behaviour:
   - **Rule 1:** Win+R, then `powershell -NoExit -EncodedCommand <base64>`, where the payload is a harmless `Write-Host` line. `-EncodedCommand` expects base64 of **UTF-16LE** text, generated like this:
     ```python
     import base64
     cmd = 'Write-Host "ClickFix detection test"'
     print(base64.b64encode(cmd.encode("utf-16-le")).decode())
     ```
   - **Rule 2:** no Office is installed, so a copy of `cmd.exe` renamed to `winword.exe` was run and used to start `powershell`. Sysmon records the parent as `winword.exe`. (The renamed copy prints "cannot find message text for message number 0x2350": cmd.exe loads its messages from a resource file named after the executable, `cmd.exe.mui`, and there is no `winword.exe.mui`. Cosmetic only.) The test copy was deleted afterwards.

### Results (28 September 2026)

| Rule | Parent -> child | Alert | Triage |
|---|---|---|---|
| Rule 1 (ClickFix) | explorer.exe -> powershell.exe (`-NoExit -EncodedCommand ...`) | Created about 2 minutes after the test | Benign positive (planned test) |
| Rule 2 (Office) | winword.exe -> powershell.exe | Created about 3 minutes after the test | Benign positive (planned test) |

The first round of tests produced **no alerts at all**. DC01 and CLIENT01 were storing UTC eight hours in the future, so the events fell outside every time-bounded search. After fixing the domain's time, the re-test alerted with no rule changes (incident 1).

The MITRE mappings were re-checked after Kibana moved to ATT&CK v19.1, using Kibana's MITRE ATT&CK coverage page: all six IDs are still valid.

## Known gaps and next steps

Deliberate lab trade-offs, with what production would do instead:

- **Kibana on plain HTTP:** enable TLS for Kibana.
- **Self-signed CA and `--insecure` enrolment:** distribute the CA or use a trusted certificate.
- **Rules created as the `elastic` superuser** (visible as `created_by` in the export): use a dedicated, least-privilege detection-engineering account.
- **Rules use Kibana's default index pattern list:** narrow them to the Windows/Sysmon data for speed and precision.
- **Rule execution gaps** while the lab is powered off: in production these mean missed events; look at Kibana's gap monitoring and remediation.
- **Timestamp override** (`event.ingested`) to tolerate ingestion delays, weighing that not every source sets it.
- **Historical data:** CLIENT01 events from enrolment until the time fix on 28 September 2026 are very probably stored eight hours ahead of real time.

Next detections: PowerShell Remoting / WinRM lateral movement (T1021.006), LSASS access (Sysmon event ID 10), Olaf Hartong's sysmon-modular configuration for ATT&CK-tagged events, and onboarding DC01 as a second endpoint.
