# AD + Elastic Detection Lab

A home lab that mirrors a small corporate network: an Active Directory domain, a Windows 11 endpoint with Sysmon, and a self-hosted Elastic SIEM. I use it to build detection rules from current threat reporting, map them to MITRE ATT&CK, and prove them with safe live simulations.

**In this repository**

- `detections/` - two custom detection rules (Kibana export, NDJSON)
- `dashboards/` - a process-creation visualisation and its data view (Kibana export, NDJSON)

## Architecture

    CLIENT01  Windows 11 Enterprise          ELASTIC01  Ubuntu Server 26.04 LTS
      Sysmon 15.21 (SwiftOnSecurity config)    Elasticsearch 9.5.4
      Elastic Agent 9.5.4  ---- logs ------->  Fleet Server 9.5.4
                                               Kibana 9.5.4  <---- browser on host
    DC01  Windows Server 2025
      Active Directory (lab.local), DNS, domain time source

    All VMs on an isolated VMware host-only network (192.168.125.0/24), no internet access.

| Component | Role |
|---|---|
| DC01 | Domain controller: AD DS, DNS, time source for the lab |
| CLIENT01 | Domain-joined endpoint generating Sysmon telemetry |
| ELASTIC01 | Elasticsearch, Kibana and Fleet Server, built from Elastic's official packages |

## Detections

| Rule | What it detects | MITRE ATT&CK |
|---|---|---|
| Suspicious Encoded PowerShell from Explorer (ClickFix Pattern) | PowerShell started by explorer.exe with encoded or hidden-window arguments: the "paste into Run" chain used by ClickFix fake-CAPTCHA lures | T1204.004 Malicious Copy and Paste, T1059.001 PowerShell |
| Script Interpreter Spawned from Microsoft Office Application | Word, Excel, PowerPoint or Outlook launching PowerShell, cmd, WScript, CScript or MSHTA, typical of malicious macros | T1204.002 Malicious File, T1059 Command and Scripting Interpreter |

**Design choices**

- **Threat-informed:** ClickFix ("Malicious Copy and Paste") is in Red Canary's 2026 Threat Detection Report top 10, so the first rule targets a current technique, not a textbook one.
- **Behaviour over payload:** both rules key on the parent-child process relationship. Attackers can change the command; a macro still has to launch something from Office.
- **Field choice:** the ClickFix rule matches on `process.command_line`, not `process.args`, because `args` splits flags such as `-WindowStyle Hidden` into separate items that one wildcard can't match.
- **Baselined before deploying:** a broad "script interpreter started" query returned normal activity (for example VMware Tools starting cmd.exe), which is why the Office-parent restriction matters.

**Validation**

Each rule went through three stages: Rule Preview on the base condition (fields and values exist), Rule Preview on the full condition (no hits on normal activity: a negative control), then a live test on CLIENT01.

- ClickFix rule: a harmless base64-encoded `Write-Host` command run through Win+R.
- Office rule: no Office installed, so a renamed copy of cmd.exe (`winword.exe`) started PowerShell, reproducing the parent-child behaviour safely.

Both rules alerted, and both alerts were triaged as benign positives (planned tests). The MITRE mappings were re-checked against ATT&CK v19.1 in October 2026.

## Lessons learned

- **"Healthy" is not the same as "working".** The first tests produced no alerts while every component reported healthy. The domain controller's time zone was wrong, so endpoint events were stored eight hours in the future and fell outside every time-bounded search. Comparing `@timestamp` (endpoint clock) with `event.ingested` (SIEM clock) is now my first check when data seems missing.
- **Reachable is not usable.** The Linux SIEM server's time client reached the Windows domain controller but rejected it as too imprecise; Windows clients accepted it. Fixed on the client side rather than changing what the DC announces to every machine.
- **Patching the SIEM's OS without upgrading the SIEM:** held the Elastic packages, dry-ran the upgrade, and found that OS patching still restarts the SIEM services, so it belongs in a maintenance window.

## Lab trade-offs (deliberate, lab only)

- Host-only network with no internet: one-off downloads use a temporary NAT procedure.
- Self-signed certificates: the endpoint agent enrolled with `--insecure` (traffic still encrypted, certificate not verified).
- Kibana runs on plain HTTP inside the isolated network.
- The domain controller takes its time from the hypervisor, because no internet NTP is available.

**Planned improvements:** a dedicated least-privilege account for rule management, rule index patterns narrowed to the Windows/Sysmon data, TLS for Kibana, and handling of rule execution gaps while the lab is powered off.

## Using the exports

Exported from Kibana 9.5.4. They assume Windows Sysmon data from Elastic Agent's Windows integration in `logs-*`.

- Rules: Kibana > Security > Rules > Detection rules (SIEM) > Import rules > `detections/kibana-rules-export.ndjson`
- Visualisation: Kibana > Saved objects > Import > `dashboards/kibana-visualisation-export.ndjson` (includes its `logs-*` data view)
