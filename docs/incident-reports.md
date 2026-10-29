# Incident reports

Problems hit while building and running the lab, with cause, fix and lesson. Incident 1 is written up in full; the others are short. The build docs ([DC01](build-dc01.md), [CLIENT01](build-client01.md), [ELASTIC01](build-elastic01.md)) refer to these by number.

| # | Incident | Area |
|---|---|---|
| 1 | Detection rules silent: the domain's UTC was eight hours wrong | DC01, CLIENT01, ELASTIC01 |
| 2 | Expired evaluation licence on the domain controller | DC01 |
| 3 | Locked out of the domain administrator account | DC01 |
| 4 | A duplicate VM sharing the domain controller's disk | DC01 |
| 5 | Temporary internet access: adapter type vs guest configuration | All VMs |
| 6 | Empty integrations list in Kibana | ELASTIC01 |
| 7 | Fleet setup looping: missing encryption keys | ELASTIC01 |
| 8 | Fleet Server wizard: name collision and a misleading crash | ELASTIC01 |
| 9 | Generated install command for the wrong CPU architecture | ELASTIC01 |
| 10 | "File does not exist" that was really a permissions problem | ELASTIC01 |
| 11 | Slow, not broken: Elasticsearch start-up time | ELASTIC01 |
| 12 | Reachable but rejected: chrony and the domain controller | ELASTIC01, DC01 |
| 13 | Patching the SIEM's OS without upgrading the SIEM | ELASTIC01 |
| 14 | Agent enrolment against a self-signed certificate | CLIENT01 |
| 15 | SSH hardened to key-only login without a lockout | ELASTIC01 |

---

## 1. Detection rules silent: the domain's UTC was eight hours wrong

**Date:** 28 September 2026

### Symptoms

- Both new detection rules showed "Succeeded" on every run, but no alerts appeared after live tests.
- Discover, last 24 hours: no CLIENT01 events at all. The only data came from the SIEM server's own monitoring.
- Fleet showed CLIENT01 **Healthy** and checking in, but its CPU and memory columns showed "N/A".
- The agent's local `status` command reported everything healthy.

### Investigation

| Step | Check | Finding |
|---|---|---|
| 1 | Rule details | Rules ran successfully. "Succeeded" means the query ran, not that it matched. |
| 2 | Discover, last 24 h | No CLIENT01 data |
| 3 | Fleet | Control channel healthy; metrics missing |
| 4 | CLIENT01 agent log | "Connection actively refused" to ports 8220 and 9200, then recovery. **Hypothesis A:** test events lost during a SIEM outage (Elasticsearch is slow to start, see incident 11). |
| 5 | Re-test | Still no alerts: **A ruled out** |
| 6 | Sysmon log on CLIENT01 (`Get-WinEvent`) | Fresh process-creation events with the expected parent and child. The endpoint was fine; the problem was between Sysmon and Kibana. |
| 7 | **Hypothesis B:** stuck event-log checkpoint; restarted Elastic Agent | Clean restart, still nothing: **B ruled out** |
| 8 | The agent's **embedded OpenTelemetry collector** log (`elastic-otel-collector-*.ndjson`), a separate file | HTTP 400 `timestamp_error`: the document timestamp was "outside of ranges of currently writable indices". Elasticsearch saw the data as coming from the future. |
| 9 | Discover, **"Search entire time range"** | 1,061 CLIENT01 events, the newest stamped about **8 hours in the future** |
| 10 | One event: `@timestamp` (endpoint clock) vs `event.ingested` (Elasticsearch clock) | Eight hours apart; the real pipeline delay was about 12 seconds |
| 11 | Time settings on every machine | DC01 and CLIENT01 set to the **Pacific** time zone; CLIENT01's UTC eight hours ahead |

### Root cause

- DC01's clock **face** showed UK time, but its **time zone** was Pacific. Windows calculates UTC from local time plus the zone's offset: Pacific Daylight Time is UTC-7 and UK summer time is UTC+1, so its UTC was exactly eight hours ahead.
- DC01 was the domain's time authority and trusted only its own hardware clock. CLIENT01 synced from DC01, inherited the wrong UTC, and, being on Pacific time too, *displayed* correct-looking local time.
- Kerberos didn't complain: it only needs a client and its DC within five minutes of **each other**, and both were wrong by the same amount.
- ELASTIC01 had the correct time, so only CLIENT01's data landed "in the future".

**One cause, three symptoms:**
1. Sysmon logs were **accepted** but stored eight hours ahead, invisible to any search ending at "now", including both rules' 6-minute windows.
2. Metrics go into time series data streams, which only accept documents inside a writable time window, so they were **rejected**. Hence "N/A" in Fleet.
3. Agent monitoring logs were future-dated too, so the agent's log view in Fleet looked empty.

### Fix

Internet NTP was not an option: DC01 has no DNS forwarders and no internet route (host-only network by design). So:

1. Snapshots of DC01 and CLIENT01.
2. **DC01 first**, remotely through PowerShell Remoting:
   ```powershell
   Invoke-Command -ComputerName DC01 -ScriptBlock { Set-TimeZone -Id "GMT Standard Time"; Set-Date -Adjust -8:00:00; Get-Date; [DateTime]::UtcNow }
   ```
   `Set-TimeZone` changes only how time is displayed; `Set-Date -Adjust` moves the actual clock. (The same remote session still printed the old zone afterwards; a fresh session showed the change had applied. A running session caches the time zone it loaded at start.)
3. **CLIENT01 immediately after**, because once DC01 moved, the two were eight hours apart and Kerberos would start failing:
   ```powershell
   Set-TimeZone -Id "GMT Standard Time"
   Set-Date -Adjust -8:00:00
   w32tm /resync /force
   ```
4. Reboot DC01, then CLIENT01: this proves the fix survives a boot and clears Kerberos tickets issued with future timestamps.
5. Verified: both machines' clocks matched the host; fresh CLIENT01 events appeared in Kibana with about one second between `@timestamp` and `event.created`.
6. Re-test: **both rules alerted**, with no rule changes. Both alerts were triaged as benign positives.

### Lessons

- **"Healthy" is not the same as "working".** Health checks show that processes run and connections open, not that the data is correct.
- **When data seems missing, check whether it's there with the wrong time.** Search the entire time range, or compare `@timestamp` with `event.ingested`. That comparison is now my first check.
- **Time zone is not time.** Systems store UTC; the zone is only how it is displayed. A correct-looking clock can hide a wrong UTC.
- **In a domain, fix time at the top**, in order: DC, then members, then reboot.
- **Know where each component logs.** The key error appeared only in the embedded collector's log.
- **Proactive detection:** monitor the gap between `event.ingested` and `@timestamp` per host, and treat a domain controller syncing from its own clock as a finding.
- **Side effect:** CLIENT01 data from enrolment until the fix is very probably stored eight hours ahead. Earlier checks passed only because they searched wide windows such as "last 7 days".

---

## 2. Expired evaluation licence on the domain controller

**Symptom:** "Windows License is expired" on DC01's desktop. `slmgr /dlv` showed `Notification`, reason `0xC004F009` (grace time expired), and **one rearm left**. An expired Windows Server evaluation shuts itself down every hour, and on a domain controller that takes DNS, authentication and time away from every other VM.

**Background:** CLIENT01 had hit the same state straight after installation. There, an offline `slmgr /rearm` used up a rearm without fixing anything; online activation (`slmgr /ato`) did.

**Fix (treated as a planned change on a critical server):**
1. Snapshot, and record the static network settings before changing them.
2. VM adapter to NAT, IPv4 and DNS to automatic.
3. **Test reachability before acting:** `Test-NetConnection activation-v2.sls.microsoft.com -Port 443` returned `TcpTestSucceeded: True`.
4. `slmgr /ato`, then `slmgr /dlv`: Licensed for 180 days, **rearm count still 1**.
5. Revert: **adapter to host-only first**, then the exact static settings.
6. Check DNS: while on NAT, DC01 had registered its temporary address, so three records briefly had two A records each. A few minutes after the static IP returned, only the correct address remained, with no manual clean-up.
7. Verified from CLIENT01 after a DC01 reboot: `nltest /dsgetdc:lab.local` found DC01 with its roles (PDC, GC, KDC, time server), and both machines showed the same time.

**Lessons:** read the actual state before choosing a fix, and keep the scarce option (the last rearm) as the fallback. On a domain controller, a temporary network change has domain-wide effects: snapshot, record, test, revert adapter-first, then check DNS for stale records.

---

## 3. Locked out of the domain administrator account

**Symptom:** after promoting DC01, the domain administrator password was rejected repeatedly. Clipboard paste and on-screen keyboard problems were ruled out; the most likely cause was a mistyped password when it was set.

**Fix:** offline recovery through the Windows installation media. From the recovery command prompt, `utilman.exe` (the Ease of Access button on the sign-in screen) was backed up and replaced with a copy of `cmd.exe`. After a normal boot, the accessibility button opened a command prompt running as SYSTEM, and `net user Administrator <new password>` reset the password. **The swap was reversed straight afterwards.**

**Lessons:** this is the same trick attackers use for persistence: MITRE ATT&CK **T1546.008, Accessibility Features**. Having done it from the inside, I know what to look for: accessibility binaries (`utilman.exe`, `sethc.exe`) whose hash or size doesn't match the genuine file, or that are really `cmd.exe`. It also shows why physical or console access, and disk encryption, matter for servers.

---

## 4. A duplicate VM sharing the domain controller's disk

**Symptom:** a second VM configuration file appeared next to DC01's, most likely from copying VM files in File Explorer instead of using VMware's Clone feature.

**Check before acting:** VM Settings > Hard Disk showed the duplicate pointed at **the same virtual disk** as the real DC01.

**Fix:** removed the duplicate with **"Remove from Library"**, not "Delete from Disk", which would have deleted the shared real disk. Then deleted only the duplicate's own configuration files by hand, leaving the disk and the real VM's files alone. Re-verified DC01 afterwards (`dcdiag`, IP, hostname, AD queries): no damage.

**Lessons:** check what a delete would actually remove before running it. Duplicate VMs with VMware's Clone feature, which gives the copy its own disk.

---

## 5. Temporary internet access: adapter type vs guest configuration

**Context:** the lab network is VMware host-only, with no internet route. One-off downloads (Sysmon, Elastic packages, OS updates, licence activation) use a temporary switch to VMware NAT.

**Symptom (CLIENT01):** after switching the adapter to NAT, downloads failed with "The remote name could not be resolved". **Symptom (ELASTIC01):** no internet-routable address at all.

**Cause:** the VMware adapter type and the guest's own network settings are independent. Each guest kept its static configuration: CLIENT01 still used DC01 as its DNS server, an address that doesn't exist on the NAT network; ELASTIC01's netplan file had no DHCP at all.

**Fix, Windows:** set IP and DNS to automatic while on NAT; set the static values back afterwards.

**Fix, Ubuntu:** keep a backup of the static netplan file; add `dhcp4: true` and `netplan apply` for the NAT period; restore the backup afterwards and prove it with `diff`.

**Related findings:**
- `netplan apply` drops an SSH session running over the interface it reconfigures. Expected; reconnect.
- While ELASTIC01 is on NAT, its static address is unreachable, so Kibana is unreachable and CLIENT01's agent shows Offline. The NAT address (from `ip a` on the VM console) works in the meantime.
- **Revert order matters:** switch the adapter back to host-only **first**. The static address was never removed, only on the wrong virtual network, so it becomes reachable again immediately. SSH-ing to the static address before switching back always times out.

**Lesson:** a VM has two network layers, the hypervisor's and the guest's, and a change to one doesn't carry over to the other. The procedure is now written down and has been reused for every download since.

---

## 6. Empty integrations list in Kibana

**Symptom:** adding the Windows integration to CLIENT01's agent policy showed "There aren't any options available".

**Cause:** Kibana loads its integration catalogue from Elastic's package registry over the internet, and ELASTIC01 was back on the isolated network.

**Fix:** the temporary-NAT procedure (incident 5), then closed and reopened the "Add integration" panel to force a fresh fetch.

**Lesson:** "needs internet" isn't only about downloads. Any Kibana action that fetches something new does too, and an empty list can mean "can't reach the source" rather than "nothing exists".

---

## 7. Fleet setup looping: missing encryption keys

**Symptom:** after a reboot, Kibana's log repeated `Fleet setup attempt N failed ... FleetEncryptedSavedObjectEncryptionKeyRequired`.

**Cause:** Fleet stores secrets (such as API keys) as encrypted saved objects and needs Kibana's encryption keys, which aren't set by default.

**Fix:**
```bash
sudo /usr/share/kibana/bin/kibana-encryption-keys generate --quiet --force
```
Added the three generated settings to `/etc/kibana/kibana.yml`, then `sudo systemctl restart kibana`. The log showed `Fleet setup completed` straight after.

**Lesson:** read the exact error name. It said what was missing; the fix was a one-time prerequisite, not a deeper fault.

---

## 8. Fleet Server wizard: name collision and a misleading crash

**Symptom 1:** the "Add Fleet Server" **Quick Start** failed with "Agent Policy 'fleet-server-policy' already exists". Once Fleet's setup completed (incident 7), it had created a default Fleet Server policy itself, and Quick Start tried to create another with the same name.
**Fix:** the **Advanced** tab, which takes a distinct policy name ("Fleet Server policy 1").

**Symptom 2:** in the Advanced flow, "Create policy" showed a browser-side JavaScript error (`can't access property "item", t is null`), with nothing in Kibana's server log.
**Check before retrying:** Fleet > Agent policies showed the policy **had** been created. Retrying blindly would have made a duplicate. A hard refresh (Ctrl+Shift+R) cleared the stale page state, and the wizard picked up the existing policy.

**Lesson:** a UI error doesn't prove the action failed. Check the actual state before repeating a change.

---

## 9. Generated install command for the wrong CPU architecture

**Symptom:** Kibana's Fleet Server install instructions defaulted to the **Linux aarch64** (ARM) tab.

**Check:** the host CPU, and therefore the VM, is x86_64. An ARM binary wouldn't run.

**Fix:** chose **Linux x86_64** from the platform menu, which regenerated the command with the right download.

**Lesson:** check generated commands against the real target (architecture, paths, URLs) before running them. Applied straight away to CLIENT01's agent (Windows x86_64).

---

## 10. "File does not exist" that was really a permissions problem

**Symptom:** `curl --cacert /etc/elasticsearch/certs/http_ca.crt ...` as a normal user failed with "file ... does not exist", although the same path had just been read with `sudo`.

**Cause:** `/etc/elasticsearch` is owned `root:elasticsearch` with mode 750. The user isn't in the `elasticsearch` group, so it can't enter the directory at all, and curl reports that as a missing file.

**Fix:** run it with `sudo`.

**Lesson:** on Linux, "does not exist" can mean "can't get through a parent directory". Check permissions along the whole path (`ls -ld` on each directory).

---

## 11. Slow, not broken: Elasticsearch start-up time

**Symptom:** Kibana showed "Unable to connect" after the lab was started.

**Checks, in order:** `ip a` (address correct); `systemctl status elasticsearch kibana` (Elasticsearch still `activating`); `ss -tlnp | grep -E '9200|5601'` (only 9200 listening); `free -h` (memory tight but not swapping); `journalctl -u elasticsearch` (every earlier boot had also taken **5 to 12 minutes**). It finished starting seconds later.

**Fix:** none needed: waiting was the answer.

**Lesson:** tell slow from failed with evidence (service state, listening ports, resources, past logs) before changing anything. Changing configuration during a slow start would have created a real problem.

---

## 12. Reachable but rejected: chrony and the domain controller

**Context:** ELASTIC01 needed a time source on the host-only network: DC01 (see the time chain in [build-dc01.md](build-dc01.md#7-time)).

**Symptom:** with DC01 configured as chrony's server, `chronyc sources` showed it as reachable but **not selected** (`^?`), and the clock stayed unsynchronised.

**Diagnosis (read-only):** `sudo chronyc ntpdata 192.168.125.10` showed every NTP test passing, but a **root dispersion of about 10 seconds**. That is what a Windows server using its own clock announces by default. chrony rejects sources whose root distance exceeds `maxdistance` (default 3 s); Windows clients have no such limit.

**Fix (client side):** `maxdistance 16` in `/etc/chrony/conf.d/lab-maxdistance.conf`, then a restart. DC01 became the selected source (`^*`), with an offset in microseconds. I didn't change DC01's announced dispersion instead, because that would affect every client in the domain.

**Lessons:** "reachable" isn't "usable": NTP clients apply quality limits, and Windows and Linux clients differ. Find where configuration actually lives before editing (chrony splits sources and settings between `sources.d` and `conf.d`). Measure before and after.

---

## 13. Patching the SIEM's OS without upgrading the SIEM

**Context:** 127 pending updates on ELASTIC01. Because Elastic's APT repository is configured, a plain `apt upgrade` would also upgrade Elasticsearch and Kibana. Elasticsearch can't be downgraded, and the stack must be upgraded in order, so that has to be a separate, planned change.

**Approach:**
1. **Two rollback paths:** a VM snapshot, and a netplan backup proven identical to the live file with `diff`.
2. **Hold the stack:** `sudo apt-mark hold elasticsearch kibana`.
3. **Prove it with a dry run** before upgrading: `apt-get -s upgrade --with-new-pkgs` showed nothing removed and no Elastic package installed.
4. **Run the upgrade from the VM console, not SSH**, because `openssh-server` and `netplan.io` were being upgraded.
5. **Verify afterwards:** the new kernel active after reboot (`uname -r`); holds in place (`dpkg -l` shows `hi`); SSH hardening, time sync and the static network intact; both agents Healthy in Fleet.

**Surprise:** Ubuntu's `needrestart` restarted `elasticsearch`, `kibana` and `elastic-agent` during the upgrade even though their packages were held.

**Lessons:** separate OS patching from application upgrades, and prove it with a dry run. Patching restarts services anyway, so plan a maintenance window. Run upgrades that touch networking or SSH from a console that doesn't depend on them. Check afterwards that earlier hardening survived.

---

## 14. Agent enrolment against a self-signed certificate

**Symptom:** Elastic Agent installed on CLIENT01 as a service, then retried enrolment endlessly with `x509: certificate signed by unknown authority`.

**Cause:** Fleet Server presents a certificate from Elasticsearch's auto-generated, self-signed CA, which Windows doesn't trust.

**Fix:** uninstall and reinstall with **`--insecure`**. Traffic is still encrypted; only verification of the certificate chain is skipped. Elastic documents this for test setups, not production. Production fix: distribute the CA to endpoints, or use a certificate from a trusted CA.

**Two smaller problems during the uninstall:**
- "uninstall must be run from outside the installed path": `cd` out of `C:\Program Files\Elastic\Agent` first.
- `Unexpected token 'uninstall'` from `"C:\...\elastic-agent.exe" uninstall`: PowerShell treats a quoted path on its own as a string. The call operator runs it: `& "C:\Program Files\Elastic\Agent\elastic-agent.exe" uninstall`.

**Lesson:** a self-signed lab CA has to be handled explicitly for every new endpoint (DC01 will need the same), and the shortcut is a documented trade-off, not a fix.

---

## 15. SSH hardened to key-only login without a lockout

**Why:** password logins can be brute-forced (MITRE ATT&CK T1110); key pairs can't realistically be guessed.

**Approach:**
1. **Prove the new path first:** an ED25519 key with a passphrase, the public key installed with correct permissions (700 on `~/.ssh`, 600 on `authorized_keys`), fingerprints compared on both sides, and a test login allowing only public-key authentication.
2. **Find where the setting really lives:** `sudo sshd -T` showed password authentication still effectively on, set by a cloud-init drop-in file.
3. **Override, don't edit:** sshd keeps the first value it reads, and drop-ins load alphabetically, so a new `10-lab-hardening.conf` (`PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `PermitRootLogin no`) takes priority over the cloud-init file.
4. **Validate before applying:** `sshd -t` (syntax), `sshd -T` (effective values), then `systemctl reload ssh`.
5. **Fallbacks open throughout:** an existing SSH session (a reload doesn't drop it) and the VMware console.
6. **Verify with new connections:** password-only attempt refused with `Permission denied (publickey).`; key login works. The setting survived a later `openssh-server` upgrade because it lives in its own file.

**Lessons:** never remove the old access path before the new one is proven, and keep a fallback open while changing access. Check the *effective* configuration, not just the obvious file.
