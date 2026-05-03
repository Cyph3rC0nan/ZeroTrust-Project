# Blue Team Cybersecurity Lab

**Course**: IT Skills  
**Team Members**: Kenan, Raul, Nuray, Nihat  

## Project Overview

An end-to-end Blue Team cybersecurity lab simulating a corporate network on AWS. The project demonstrates detection-to-analysis pipeline: from Active Directory hardening, through real-time SIEM monitoring and automated incident response, to forensic malware analysis and YARA rule generation.

## Architecture

| Machine | OS | Role |
|---------|------|------|
| DC01 | Windows Server 2019 | Domain Controller |
| CLIENT01 | Windows Server 2019 | Domain Workstation |
| WAZUH-SRV | Ubuntu 22.04 | Wazuh SIEM + Velociraptor EDR |
| KALI | Kali Linux | Attacker (Local) |

## Tools Used

- **Wazuh** — SIEM + Active Response (automated containment)
- **Velociraptor** — Forensic threat hunting + evidence extraction
- **Sysmon** — Endpoint telemetry (SwiftOnSecurity config)
- **ANY.RUN** — Dynamic malware sandbox
- **Hybrid Analysis** — Secondary sandbox + MITRE mapping
- **YARA** — Malware detection rule authoring
- **Ghidra** — Reverse engineering

## Project Structure

```
ITSkills/
├── AD-GPO-SS/                    # Active Directory & GPO screenshots
├── Agent-Sysmon-conf/            # Wazuh agent & Sysmon deployment screenshots
├── Attack&Logs/                  # Attack execution & Wazuh log screenshots
├── Active-response-wazuh/        # EDR active response screenshots
├── Velociraptor Deployment/      # Velociraptor server & client screenshots
├── Malware Creation/             # Malware drop, FIM, VirusTotal screenshots
├── Dynamic analysis/             # ANY.RUN & Hybrid Analysis screenshots
├── Static analysis + yara/       # Strings, PE headers, Ghidra, YARA screenshots
├── Guide Files/                  # Day-by-day implementation guides
├── Final Report.docx             # Full project report
└── README.md
```

## Key Achievements

- ✅ 7 MITRE ATT&CK techniques simulated — **100% detection rate**
- ✅ Automated Active Response: brute force IP blocking in seconds
- ✅ Full malware analysis pipeline: EDR → extraction → sandbox → IOCs → YARA
- ✅ Two validated YARA detection rules with zero false positives

## Attack Techniques Covered

| # | Technique | ATT&CK ID |
|---|-----------|-----------|
| 1 | PowerShell Execution | T1059.001 |
| 2 | Account Creation | T1136.001 |
| 3 | Scheduled Task Persistence | T1053.005 |
| 4 | Credential Dumping (LSASS) | T1003.001 |
| 5 | Brute Force | T1110.001 |
| 6 | Lateral Movement (RDP) | T1021.001 |
| 7 | Malware Drop | T1105 |
