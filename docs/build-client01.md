# CLIENT01: Windows 11 endpoint build

CLIENT01 is the lab's monitored endpoint: a domain-joined Windows 11 Enterprise VM running Sysmon, with Elastic Agent shipping its logs to the SIEM on ELASTIC01. The detection rules are tested on it.

Problems hit along the way are written up in [incident-reports.md](incident-reports.md); the numbers below (e.g. "incident 5") refer to that file.

## At a glance

| Area | Configuration |
|---|---|
| VM | Windows 11 Enterprise (evaluation), 2 vCPU, 4 GB RAM, 60 GB NVMe (thin-provisioned), UEFI with Secure Boot and a virtual TPM 2.0 |
| Network | VMware host-only, static 192.168.125.11/24, no gateway, DNS 192.168.125.10 (DC01) |
| Domain | Joined to `lab.local` |
| Logging | Sysmon 15.21 with the SwiftOnSecurity configuration |
| Shipping | Elastic Agent 9.5.4, Windows integration, enrolled in Fleet on ELASTIC01 |
| Time | GMT Standard Time, synced from DC01 through the domain hierarchy |

## 1. VM creation and install

- Evaluation ISO from the Microsoft Evaluation Center. VMware's Easy Install was skipped, as for DC01 ("I will install the operating system later", then attach the ISO).
- **Windows 11 requires Secure Boot and a TPM 2.0**, unlike Windows Server. Adding a virtual TPM made VMware Workstation ask how to encrypt the VM. I chose **"Only the files needed to support a virtual TPM"** rather than encrypting the whole VM. That encryption password matters: without it, the VM can't be powered on.
- VMware's wizard defaulted to 2 processors with 1 core each; I changed it to 1 socket with 2 cores to match DC01.
- Installed through Windows Setup with "I don't have a product key", and "I don't have internet" during first-run setup, since the network is host-only.

## 2. Licence

- Straight after the first boot, the desktop showed "Windows License is expired", and `slmgr /dlv` showed reason `0xC004F009` (grace time expired). An offline `slmgr /rearm` didn't fix it and used up one of only two rearms.
- What worked was online activation over a temporary NAT connection: `slmgr /ato`, then `slmgr /xpr` to confirm the expiry date (90 days from activation). The same approach later fixed DC01 without spending a rearm (incident 2).
- One rearm remains, to be used before the evaluation expires.

## 3. Network and domain join

- Static IP 192.168.125.11/24, no gateway, **DNS = DC01 (192.168.125.10)**. The host-only network's default DNS can't resolve `lab.local`, and a domain join depends on finding the domain controller through DNS.
- Tested with `ping 192.168.125.10` and `nslookup lab.local` first. The first attempt failed only because DC01 was powered off. DC01 has to be running before any other VM can do anything domain-related.
- Renamed the computer and joined the domain in one step (System Properties > Computer Name > Change: new name plus domain, one restart).
- Verified inside the VM: full computer name `CLIENT01.lab.local`, domain `lab.local`.

## 4. Sysmon

Sysmon (from Microsoft Sysinternals) adds detailed security events to the Windows event log, such as process creation with full command lines (event ID 1) and DNS queries (event ID 22). The SwiftOnSecurity configuration is a widely used, well-documented baseline. Olaf Hartong's sysmon-modular, which tags events with MITRE ATT&CK techniques, is the planned next step.

- Downloads (over temporary NAT, see below): `https://download.sysinternals.com/files/Sysmon.zip` (Sysmon 15.21) and the SwiftOnSecurity `sysmonconfig-export.xml` from `https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml`.
- Install:
  ```powershell
  New-Item -ItemType Directory -Path "C:\Tools\Sysmon" -Force
  Expand-Archive -Path "$env:USERPROFILE\Downloads\Sysmon.zip" -DestinationPath "C:\Tools\Sysmon" -Force
  Copy-Item "$env:USERPROFILE\Downloads\sysmonconfig-export.xml" -Destination "C:\Tools\Sysmon"
  cd C:\Tools\Sysmon
  .\Sysmon64.exe -accepteula -i .\sysmonconfig-export.xml
  ```
  `-accepteula` accepts the licence without a prompt; `-i` installs the service and driver with the given configuration. The output confirmed configuration schema 4.50 validated against Sysmon's schema 4.91, and both `Sysmon64` and `SysmonDrv` installed and started.
- Verified:
  ```powershell
  Get-Service Sysmon64
  Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5
  ```
  The service was Running, and the log already held process creation (ID 1) and DNS query (ID 22) events.
- VMware Tools was installed first for host-to-guest copy and paste. Its CD has a single `setup.exe` for both architectures.

### Temporary internet access on a Windows guest

Switching the VMware adapter from host-only to NAT does **not** change the guest's own settings. CLIENT01 kept its static IP and kept using DC01 as its DNS server, an address that doesn't exist on the NAT network, so downloads failed with "The remote name could not be resolved" (incident 5). The working procedure:

1. Adapter to NAT, then set both IP and DNS assignment to **Automatic (DHCP)** in Windows Settings.
2. Download.
3. Set IP and DNS back to the static values, switch the adapter back to host-only, and confirm with `ipconfig /all`.

## 5. Elastic Agent

- Downloaded from the Kibana Fleet "Add agent" flow, choosing the **Windows x86_64** zip (`elastic-agent-9.5.4-windows-x86_64.zip`) after checking the architecture.
- Enrolled under its own agent policy, **Windows Endpoints**, which uses the official **Windows** integration. It collects the Sysmon Operational channel by default, plus Forwarded, PowerShell and PowerShell Operational.
- Installed as a service in `C:\Program Files\Elastic\Agent`, enrolled to Fleet Server at `https://192.168.125.12:8220`.
- **Enrolment needed `--insecure`**: Fleet Server's certificate comes from Elasticsearch's self-signed CA, which Windows doesn't trust. The first attempt also needed a clean uninstall before reinstalling (incident 14).
- Verified: agent Healthy in Fleet (9.5.4), and Sysmon events searchable in Kibana under `data_stream.dataset : "windows.sysmon_operational"`.

Useful endpoint-side checks:

```powershell
# Is Sysmon logging?
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10 | Select-Object TimeCreated, Id, Message | Format-List
# Agent health
& "C:\Program Files\Elastic\Agent\elastic-agent.exe" status
# Agent and collector logs, newest first
Get-ChildItem "C:\Program Files\Elastic\Agent\data\elastic-agent-*\logs\" -Filter *.ndjson | Sort-Object LastWriteTime -Descending
# Restart the agent
Restart-Service -Name "Elastic Agent" -Force
```

`&` is PowerShell's call operator: without it, a quoted path is treated as a plain string, not a command to run. In Agent 9.5.4 the data inputs run inside an embedded OpenTelemetry collector that writes its own `elastic-otel-collector-*.ndjson` log. Indexing errors from Elasticsearch appeared **only** there, not in the main agent log (incident 1).

## 6. Detection testing

Both detection rules were tested here with harmless reproductions: an encoded `Write-Host` command run through Win+R for the ClickFix rule, and a copy of `cmd.exe` renamed to `winword.exe` for the Office rule. Both alerted and were triaged as benign positives. The test copy was deleted afterwards (`Remove-Item "$env:USERPROFILE\Desktop\winword.exe"`). The rules, test method and results are in [build-elastic01.md](build-elastic01.md#8-detection-engineering).

## 7. Time

CLIENT01 was on Pacific time, inherited along with DC01's wrong UTC, so every event it sent was stamped eight hours in the future (incident 1). After DC01 was fixed:

```powershell
Set-TimeZone -Id "GMT Standard Time"   # display time zone: UK
Set-Date -Adjust -8:00:00              # move the actual clock back 8 hours so UTC is correct
w32tm /resync /force                   # re-sync from DC01
[DateTime]::UtcNow                     # check UTC directly
Restart-Computer                       # clears Kerberos tickets issued with future timestamps
```

Verified with `Invoke-Command -ComputerName DC01 -ScriptBlock { Get-Date }; Get-Date`: both clocks matched the host. Fresh CLIENT01 events then appeared in Kibana at the correct time.

## Known gaps

- **Evaluation licence:** expires 6 December 2026, one rearm left.
- **`--insecure` enrolment:** production would distribute the CA or use a trusted certificate.
- **Sysmon configuration:** the SwiftOnSecurity baseline; ATT&CK-tagged sysmon-modular is planned.
- **Historical data:** events from enrolment until the time fix on 28 September 2026 are very probably stored eight hours ahead of real time.
