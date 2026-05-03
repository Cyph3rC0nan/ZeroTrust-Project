# Blue-Team Lab — Execution Plan (Targeting 25/25)

**Date**: April 16, 2026  
**Time remaining**: ~2 weeks (until ~April 30)  
**Target grade**: 25 / 25 (Outstanding)  
**Philosophy**: Show it, don't over-engineer it. Demonstrate capability, not perfection.

---

## Grading Context

> [!NOTE]
> The rubric was AI-generated and the teacher won't be surgically strict. The goal is to **visibly demonstrate** each level-5 criterion — not to build a production-ready SOC. If something looks right, functions in a demo, and has documentation/screenshots behind it, it counts. We aim for 5/5 everywhere, but with smart shortcuts.

---

## What Level 5 Actually Requires (Realistic Reading)

| Criterion | Level 5 Text | What We Actually Need to Show |
|---|---|---|
| 1. AD Setup | "Fully functional corporate-like AD environment" | OUs, users, groups, 3–4 GPOs that make sense. Looks like a real company. |
| 2. AD Attacks & Log Analysis | "Professional analysis linking attacks to detections" | Run attacks → show Wazuh caught them → map to MITRE ATT&CK. Clean write-up. |
| 3. EDR | "Full EDR workflow with automated containment" | **Velociraptor** for endpoint hunting/forensics + **Wazuh active response** for automated containment (IP block, process kill). Together = full EDR workflow. |
| 4. Malware Analysis | "Professional-grade analysis with full technical breakdown" | Sandbox report + static analysis notes + IOCs + YARA rule. Doesn't need to be a 20-page thesis. |
| 5. Presentation & Documentation | "Polished demo with professional documentation" | Clean report with screenshots, architecture diagram, and a smooth demo. |

---

## Current State → What's Left

| Component | Status | Work Needed |
|---|---|---|
| AD DC + Client joined | ✅ Done | Add OUs, users, GPOs |
| Wazuh + Dashboard | ✅ Ready on start | Enroll agents, configure log ingestion |
| VPC / Subnets | ✅ Done | — |
| Wazuh agents on Windows | ❌ | Install + configure |
| Sysmon | ❌ | Install on both Windows machines |
| Velociraptor server | ❌ | Deploy on Wazuh Ubuntu instance |
| Velociraptor clients | ❌ | Install on DC + Client |
| Attacks | ❌ | Run 6 from local Kali |
| EDR / Active Response | ❌ | Wazuh active response + Velociraptor hunts |
| Malware Sandbox | ❌ | Cuckoo local or ANY.RUN fallback |
| Documentation | ❌ | Report + runbook + demo |

---

## Execution Plan — 14 Days

---

### Phase 1: AD Polish + SIEM Pipeline (Days 1–3)

> Gets Criterion 1 to level 5, sets foundation for Criteria 2 & 3

#### Day 1 — AD Corporate Structure

**Goal**: Make the AD look like a real small company

- [ ] Create OUs:
  - `Corp > IT Department`
  - `Corp > HR`
  - `Corp > Management`
  - `Corp > Servers`
- [ ] Create 6–8 user accounts with realistic names (e.g., `j.smith`, `a.johnson`)
- [ ] Create security groups: `IT-Admins`, `HR-Staff`, `Standard-Users`
- [ ] Assign users to groups and OUs
- [ ] Create GPOs:
  1. **Password Policy** — min 12 chars, complexity, lockout after 5 fails
  2. **Audit Policy** — logon events, process creation, privilege use
  3. **PowerShell Logging** — Script Block + Module Logging enabled
  4. **Restricted Access** — deny CMD for Standard-Users (shows hardening intent)
- [ ] 📸 Screenshot: AD Users & Computers showing the OU tree
- [ ] 📸 Screenshot: GPO Management Console showing linked GPOs
- [ ] Git commit

> [!TIP]
> **Level 5 checkpoint**: If someone opens AD and sees organized OUs, named users in groups, and linked GPOs — that looks "corporate-like." Done.

#### Day 2 — Wazuh Agent Enrollment + Sysmon

**Goal**: Both Windows machines reporting to Wazuh with rich telemetry

- [ ] Start Wazuh Ubuntu instance
- [ ] Install Wazuh agent on DC → enroll → verify in dashboard
- [ ] Install Wazuh agent on Client → enroll → verify in dashboard
- [ ] Download and install **Sysmon** on both machines (use [SwiftOnSecurity config](https://github.com/SwiftOnSecurity/sysmon-config))
- [ ] Edit `ossec.conf` on both agents to ingest Sysmon logs:
  ```xml
  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
  ```
- [ ] Restart Wazuh agents
- [ ] 📸 Screenshot: Wazuh dashboard showing 2 agents online
- [ ] 📸 Screenshot: Sysmon events flowing into Wazuh
- [ ] Git commit

#### Day 3 — Velociraptor Deployment + Log Validation

**Goal**: Deploy Velociraptor EDR + prove the full pipeline works end-to-end

**Velociraptor Server (on Wazuh Ubuntu instance):**
- [ ] Download Velociraptor Linux binary on the Ubuntu instance:
  ```bash
  wget https://github.com/Velocidex/velociraptor/releases/latest/download/velociraptor-v0.73.3-linux-amd64
  chmod +x velociraptor-v0.73.3-linux-amd64
  ```
- [ ] Generate server + client configs:
  ```bash
  ./velociraptor-v0.73.3-linux-amd64 config generate -i
  ```
  (Use defaults, set the frontend hostname to the Ubuntu instance's private IP)
- [ ] Start the Velociraptor server:
  ```bash
  ./velociraptor-v0.73.3-linux-amd64 frontend -v --config server.config.yaml
  ```
- [ ] Access Velociraptor GUI at `https://<ubuntu-ip>:8889`
- [ ] 📸 Screenshot: Velociraptor dashboard (empty, ready for clients)

**Velociraptor Client (on both Windows machines via RDP):**
- [ ] Download Velociraptor Windows binary on DC and Client
- [ ] Copy the `client.config.yaml` from the server to both Windows machines
- [ ] Install as a service:
  ```powershell
  .\velociraptor.exe service install --config client.config.yaml
  ```
- [ ] Verify both clients appear in Velociraptor server GUI
- [ ] 📸 Screenshot: Velociraptor showing 2 clients connected

**Log Validation:**
- [ ] Generate test events:
  - Create a new user via PowerShell
  - RDP from Client to DC
  - Run `whoami`, `ipconfig`, `net user` (basic recon commands)
- [ ] Verify events appear in **Wazuh dashboard**
- [ ] Run a quick **VQL hunt** in Velociraptor to confirm it works:
  ```sql
  SELECT * FROM info()
  ```
- [ ] Create or customize a **Wazuh dashboard view**: security alerts, top events, agent status
- [ ] Document baseline: "Before attacks, this is what normal looks like"
- [ ] 📸 Screenshot: Wazuh dashboard with events
- [ ] 📸 Screenshot: Velociraptor hunt results
- [ ] Git commit

> **After Day 3**: Criterion 1 ✅ (level 5) | Criterion 2 & 3 foundations set | Velociraptor + Wazuh both operational

---

### Phase 2: Attack Simulation + Detection + EDR (Days 4–7)

> Gets Criterion 2 to level 5, Criterion 3 to level 5
> **Pipeline**: Attacks (4-5) → EDR Config (6) → Malware Drop with armed EDR + Write-up (7)

#### Day 4 — Snapshot + Attacks 1–3

- [ ] Create AWS snapshots: `snapshot_20260421_pre-attack`
- [ ] Ensure Kali can reach cloud VMs (security group: allow Kali's public IP on specific ports)

**Attack 1 — T1059.001: PowerShell Execution**
- Run suspicious PowerShell (download cradle, encoded command)
- Capture: Wazuh alert + Sysmon Event ID 1 + Velociraptor process artifact

**Attack 2 — T1136.001: Create Local Account**
- `net user hacker Password123! /add`
- Capture: Wazuh alert (rule 5902 or similar) + Windows Security Event 4720

**Attack 3 — T1053.005: Scheduled Task Persistence**
- Create scheduled task that runs a payload
- Capture: Sysmon Event + Wazuh alert

- [ ] After each attack, run a **Velociraptor hunt** to collect forensic evidence:
  - `Windows.System.TaskScheduler` (for scheduled tasks)
  - `Windows.System.Pslist` (for running processes)
- [ ] Store all artifacts in `/artifacts/YYYYMMDD_<technique>/`
- [ ] Git commit after each attack

#### Day 5 — Attacks 4–6

**Attack 4 — T1003.001: Credential Dumping**
- Use Mimikatz or `procdump` on LSASS
- Capture: Sysmon process access event + Wazuh alert

**Attack 5 — T1110.001: Brute Force**
- Brute force RDP or SMB login from Kali (use Hydra or CrackMapExec)
- Capture: Multiple 4625 (failed logon) events + Wazuh alert

**Attack 6 — T1021.001: Lateral Movement via RDP**
- RDP from compromised account to DC
- Capture: logon event chain across both machines

- [ ] Run **Velociraptor hunts** after attacks:
  - `Windows.EventLogs.LogonSessions` (for lateral movement evidence)
  - `Windows.Detection.Yara.NTFS` (scan for dropped binaries)
- [ ] Store all artifacts
- [ ] Git commit

#### Day 6 — EDR Configuration (Arm the Defenses)

**Goal**: Configure full EDR so it's ready to detect + auto-respond to the malware drop on Day 7

**Part A — Wazuh Active Response (Automated Containment):**
- [ ] Configure **Wazuh Active Response** rule 1:
  - Trigger: 5+ failed logon attempts from same IP
  - Action: Firewall block (auto-block attacker IP)
  - Test it: run brute force → watch IP get blocked automatically
- [ ] Configure **Active Response** rule 2:
  - Trigger: Mimikatz detection or suspicious process
  - Action: Kill process
- [ ] 📸 Screenshot: active response config in `ossec.conf`
- [ ] 📸 Screenshot: Wazuh log showing "Active response triggered"

**Part B — Velociraptor Threat Hunting (Sweep for Attack Artifacts from Days 4-5):**
- [ ] Create a **VQL hunt** that sweeps both endpoints for IOCs from the attacks:
  ```sql
  -- Example: Find all processes that spawned from PowerShell
  SELECT Name, Pid, Ppid, CommandLine, Username
  FROM pslist()
  WHERE Name =~ 'powershell'
  ```
- [ ] Run `Windows.Detection.Amcache` to find recently executed binaries
- [ ] Run `Windows.System.ARP` to show network connections during attacks
- [ ] Export Velociraptor hunt results as evidence (CSV or JSON)
- [ ] 📸 Screenshot: Velociraptor hunt showing attack artifacts
- [ ] 📸 Screenshot: Velociraptor client detail page showing endpoint info
- [ ] Git commit

> [!TIP]
> After Day 6, EDR is fully armed. Day 7 is when we drop malware and let the EDR catch it live.

#### Day 7 — Malware Drop Simulation (EDR Armed) + Attack Write-up

**Goal**: Drop malware with EDR fully active → EDR detects + responds → extract via Velociraptor → this binary goes to sandbox on Day 8

> [!IMPORTANT]
> **This is the full pipeline moment.** EDR is now armed from Day 6. When malware hits the endpoint, the whole chain fires: Sysmon logs it → Wazuh alerts → Active Response blocks/kills → Velociraptor extracts the binary → binary goes to sandbox.

**Part A — Malware Drop Simulation:**
- [ ] Download a known malware sample from [MalwareBazaar](https://bazaar.abuse.ch/) on Kali (well-documented PE sample, e.g., AgentTesla, Formbook)
- [ ] Transfer the sample to the DC endpoint (SCP, Evil-WinRM, or direct download)
- [ ] Place it on disk: `C:\Temp\malware_sample.exe`
- [ ] **Execute the sample** (or simulate execution) — let EDR catch it:
  - Sysmon Event ID 1 (process create) + Event ID 11 (file create)
  - Wazuh alert fires on suspicious execution
  - **Active Response triggers**: process killed / IP blocked
- [ ] 📸 Screenshot: Wazuh alert showing malware detected
- [ ] 📸 Screenshot: Active Response log showing containment action
- [ ] Record file hashes: `Get-FileHash C:\Temp\malware_sample.exe -Algorithm SHA256`

**Part B — Velociraptor Extracts the Malware:**
- [ ] In Velociraptor GUI → select DC01 → **New Collection**
- [ ] Collect artifact: `Generic.Collectors.File`
  - Path: `C:\Temp\malware_sample.exe`
  - This **downloads the binary from the endpoint to the Velociraptor server**
- [ ] Also collect: `Windows.Detection.Amcache` (shows the malware was executed)
- [ ] Export the collected binary from Velociraptor → save for Day 8 sandbox
- [ ] 📸 Screenshot: Velociraptor file collection showing the extracted malware
- [ ] 📸 Screenshot: Velociraptor showing the file details (hash, size, path)

> **The extracted binary is now ready for sandbox analysis on Day 8.**

**Part C — Attack Analysis Write-up (All 6 Attacks + Malware Drop):**
- [ ] For each of the 6 attacks + malware drop, create a one-page summary:

  ```
  Technique: T1059.001 — PowerShell Execution
  ATT&CK Tactic: Execution
  What was done: [description]
  Detection Evidence:
    - Wazuh Alert: Rule ID XXXX, Level XX
    - Sysmon Event: Event ID 1, Process Create
    - Windows Event: Event ID 4688
  MITRE Mapping: ✅ Execution → T1059.001
  Detection Verdict: DETECTED / PARTIALLY DETECTED / MISSED
  Screenshot: [link]
  ```

- [ ] Create a **detection coverage matrix**:

  | Attack | Technique | Wazuh Detected? | Sysmon Logged? | Velociraptor? | Active Response? |
  |---|---|---|---|---|---|
  | 1 | T1059.001 | ✅ | ✅ | ✅ Process artifact | — |
  | 2 | T1136.001 | ✅ | ✅ | ✅ User enum | — |
  | 5 | T1110.001 | ✅ | ✅ | ✅ Logon sessions | ✅ IP blocked |
  | 7 | Malware Drop | ✅ | ✅ | ✅ **Binary extracted** | ✅ Process killed |

- [ ] Write **EDR Response Playbook** documenting the combined workflow:
  1. **Detection**: Wazuh alerts on malware execution
  2. **Automated Response**: Active response kills process / blocks source
  3. **Investigation**: Velociraptor extracts the malware binary from endpoint
  4. **Analysis**: Binary sent to sandbox for dynamic + static analysis
  5. **Output**: IOCs + YARA rules generated
  6. **Recovery**: Restore from snapshot if needed
- [ ] Git commit

> **After Day 7**: Criterion 2 ✅ (level 5) | Criterion 3 ✅ (level 5) | Malware binary extracted and ready for sandbox

---

### Phase 3: Malware Analysis (Days 8–10)

> Gets Criterion 4 to level 5
> **Key**: The malware sample comes from **Velociraptor extraction on Day 7** — NOT a fresh download. This proves the full pipeline.

#### Day 8 — Sandbox Setup + Submit Extracted Malware

**Strategy**: Try Cuckoo. If it takes more than 3–4 hours to install, switch to Plan B immediately.

- [ ] **Plan A**: Set up Cuckoo Sandbox on local VM
  - Install Cuckoo on Ubuntu host
  - Configure Windows analysis VM inside Cuckoo
  - Test with a benign file
- [ ] **Plan B** (if Cuckoo fails): Use **ANY.RUN** free community + **Hybrid Analysis**
  - Register accounts
  - These are legitimate sandbox platforms — they produce professional reports
- [ ] Take the **malware binary extracted by Velociraptor on Day 7**
  - Download it from Velociraptor server
  - This is the sample that the EDR detected and responded to
- [ ] Submit the extracted sample to sandbox
- [ ] 📸 Screenshot: sandbox running the EDR-captured malware

> [!TIP]
> **For the demo/report narrative**: "Our EDR (Wazuh + Velociraptor) detected malware on the endpoint, auto-killed the process, and we used Velociraptor to extract the binary for forensic analysis in our sandbox." This is the story the teacher wants to see.

#### Day 9 — Dynamic + Static Analysis

**Dynamic analysis (sandbox reports):**
- [ ] Capture full sandbox report for both samples
- [ ] Document per sample:
  - File hash (MD5, SHA256)
  - Process tree (what it spawned)
  - Network activity (DNS queries, C2 IPs)
  - File system changes (dropped files, modified registry keys)
  - Classification / verdict

**Static analysis:**
- [ ] Run `strings` on sample → extract interesting strings (URLs, IPs, commands)
- [ ] Analyze PE headers with `pestudio` or `pefile`
- [ ] Optional: open in Ghidra, screenshot the main function or a C2 routine
- [ ] Document findings

> [!TIP]
> For static analysis, you don't need deep reverse engineering. Run `strings`, show PE info, open Ghidra and screenshot 1–2 interesting functions. That's "full technical breakdown" level for a student project.

#### Day 10 — IOCs + YARA Rules

- [ ] Compile **IOC list** for each sample:
  ```
  Sample: evil.exe
  MD5: abc123...
  SHA256: def456...
  C2: 192.168.x.x:4444
  DNS: evil-domain.com
  Dropped file: C:\Temp\payload.dll
  Registry: HKCU\Software\Microsoft\Windows\CurrentVersion\Run\evil
  ```

- [ ] Write **2 YARA rules** (one per sample):
  ```yara
  rule Trojan_Sample1 {
      meta:
          description = "Detects Sample1 trojan based on unique strings"
          author = "Team"
          date = "2026-04-27"
      strings:
          $s1 = "evil-domain.com"
          $s2 = { 4D 5A 90 00 }
          $s3 = "cmd.exe /c"
      condition:
          uint16(0) == 0x5A4D and 2 of ($s*)
  }
  ```

- [ ] Test YARA rules against samples (show they match)
- [ ] 📸 Screenshot: YARA rule matching the sample
- [ ] Git commit all artifacts

> **After Day 10**: Criterion 4 ✅ (level 5)

---

### Phase 4: Documentation & Presentation (Days 11–14)

> Gets Criterion 5 to level 5

#### Day 11 — Final Report

- [ ] Write structured report (8–12 pages), sections:
  1. **Executive Summary** (half page)
  2. **Lab Architecture** — include a clean diagram (I can generate one for you)
  3. **Active Directory Design** — OU structure, GPOs, user accounts
  4. **SIEM & Log Pipeline** — Wazuh + Sysmon + agents, how data flows
  5. **Endpoint Detection & Response** — Velociraptor deployment + VQL hunts + Wazuh active response
  6. **Attack Simulation Results** — the 6 attacks with detection evidence
  7. **Detection Coverage Matrix** — table from Day 7 (now includes Velociraptor column)
  8. **Malware Analysis** — sandbox reports + static analysis + IOCs
  9. **YARA Rules** — full rules with explanation
  10. **Lessons Learned & Recommendations**
- [ ] **Every section must have at least 1 screenshot or visual**

#### Day 12 — Incident Runbook

- [ ] Write a **6-page incident response runbook** covering:
  - **Scenario 1**: Brute force detected → auto-blocked → manual follow-up
  - **Scenario 2**: Malware executed → sandbox analysis → containment
  - Each scenario covers: Detection → Triage → Containment → Eradication → Recovery
- [ ] Include flowcharts or step-by-step numbered procedures
- [ ] This document alone shows "professional documentation"

#### Day 13 — Demo Preparation

- [ ] Prepare 10–15 minute demo showing the full chain:
  ```
  Attack (from Kali)
  → SIEM detection (Wazuh dashboard alert)
  → Endpoint telemetry (Sysmon process tree)
  → Active response (auto-block / auto-kill malware)
  → Velociraptor extracts malware binary from endpoint
  → Binary submitted to sandbox for analysis
  → Sandbox report → IOCs → YARA rule
  ```
- [ ] Create slides (10–15 slides max) or prepare live demo
- [ ] Practice the demo flow — each team member presents their part
- [ ] Record demo video (screen recording) as backup

#### Day 14 — Final Polish & Submission

- [ ] Finalize Git repo:
  - Clean `README.md` with full setup instructions
  - `/docs` — report + runbook
  - `/artifacts` — organized by date and test
  - `/configs` — all config files (ossec.conf, sysmon config, GPO exports)
  - `/scripts` — any automation scripts used
- [ ] Review all screenshots are included
- [ ] Final team walkthrough
- [ ] Submit

> **After Day 14**: Criterion 5 ✅ (level 5) | All criteria complete

---

## Score Projection: 25/25

| Criterion | What We Show | Why It's Level 5 |
|---|---|---|
| 1. AD Setup | OUs, users, groups, 4 GPOs, screenshots | Looks like a real company's AD — "corporate-like" ✅ |
| 2. Attacks & Logs | 6 attacks, MITRE-mapped, detection matrix | Analysis links every attack to its detection — "professional analysis" ✅ |
| 3. EDR | **Velociraptor** (hunting + forensics) + **Wazuh active response** (auto-block + auto-kill) + playbook | Two-tool EDR stack + automated containment + documented workflow ✅ |
| 4. Malware Analysis | Sandbox report + strings + PE analysis + IOCs + YARA | Sandbox + static + YARA = "full technical breakdown" ✅ |
| 5. Documentation | 12-page report + runbook + demo video + clean repo | Screenshots + visuals + smooth demo = "polished & professional" ✅ |

---

## Smart Shortcuts (Show, Don't Over-Engineer)

1. **AD GPOs**: 4 GPOs linked to OUs is enough. Don't build 15 policies. A password policy + audit policy + PowerShell logging + one restriction = corporate-like.

2. **Attacks**: 6 attacks is the sweet spot. Don't run 12. Pick attacks that Wazuh/Sysmon **will** detect (so your matrix looks good). Avoid obscure techniques that produce no alerts.

3. **EDR / Velociraptor**: Install Velociraptor server + clients, run 3–4 VQL hunts during attacks, export results. Don't build custom VQL from scratch — use the built-in artifact library (e.g., `Windows.System.Pslist`, `Windows.Detection.Amcache`). It has 200+ ready-made artifacts.

4. **Wazuh Active Response**: 2 active response rules (IP block + process kill) is "automated containment." Don't try to build a full SOAR platform.

5. **Malware Analysis**: If Cuckoo is a pain, ANY.RUN gives you **better-looking reports** than Cuckoo anyway (they're pretty and professional). The teacher won't care which sandbox you used — they care that you have a report.

6. **YARA Rules**: 2 simple YARA rules that actually match your samples. Don't try to write enterprise detection rules.

7. **Documentation**: Screenshots are your best friend. A report with 15 good screenshots looks 10x more impressive than one with dense text.

8. **Demo**: Live demo is impressive but risky. Have a **recorded video backup**. If something breaks live, play the video.

---

## Negotiation Position (If Scores Come in Lower Than Expected)

If the teacher pushes back on any criterion:

| If They Say | You Say |
|---|---|
| "AD isn't corporate enough" | "We have OUs, groups, realistic users, and 4 enforced GPOs — this mirrors a small business setup" |
| "Attacks weren't analyzed professionally" | "Each attack is mapped to MITRE ATT&CK with detection evidence and a coverage matrix" |
| "No real EDR" | "We run Velociraptor for endpoint hunting and forensic artifact collection, plus Wazuh active response for automated containment (IP blocking, process termination). That's a two-tool EDR stack with a documented response playbook." |
| "Malware analysis isn't detailed enough" | "We have dynamic sandbox reports, static analysis, extracted IOCs, and verified YARA detection rules" |
| "Documentation isn't polished" | "We have a structured report with architecture diagrams, screenshots, an incident runbook, and a recorded demo" |

> [!IMPORTANT]
> **Key negotiation leverage**: Everything is in a Git repo with timestamped commits, screenshots, and artifacts. That level of rigor is rare in student projects. If the teacher gives anything less than 23, the repo alone proves the work was done.

---

## Daily Summary (Quick Reference)

| Day | Date | Focus | Criterion |
|---|---|---|---|
| 1 | Apr 17 | AD OUs, users, GPOs | 1 |
| 2 | Apr 18 | Wazuh agents + Sysmon | 2, 3 |
| 3 | Apr 19 | Velociraptor deployment + log validation | 2, 3 |
| 4 | Apr 20 | Attacks 1–3 | 2 |
| 5 | Apr 21 | Attacks 4–6 | 2 |
| 6 | Apr 22 | EDR config: Wazuh active response + Velociraptor hunts | 3 |
| 7 | Apr 23 | **Malware drop (EDR armed)** + Velociraptor extraction + write-up | 2, 3, 4 |
| 8 | Apr 24 | Sandbox setup + submit EDR-extracted malware | 4 |
| 9 | Apr 25 | Dynamic + static analysis | 4 |
| 10 | Apr 26 | IOCs + YARA rules | 4 |
| 11 | Apr 27 | Final report | 5 |
| 12 | Apr 28 | Incident runbook | 5 |
| 13 | Apr 29 | Demo preparation | 5 |
| 14 | Apr 30 | Final polish + submit | All |

---

## Full Pipeline Flow (How Everything Connects)

```
Days 4-5: Run 6 attacks → Wazuh + Sysmon log everything
                │
Day 6:    Configure EDR (active response + Velociraptor hunts for attack artifacts)
                │  EDR IS NOW FULLY ARMED
                │
Day 7:    Drop malware on endpoint → EDR detects → auto-kills → Velociraptor extracts binary
                │
Days 8-10: Extracted binary → Sandbox → Dynamic analysis → Static analysis → IOCs → YARA
                │
Days 11-14: Document everything → Report → Runbook → Demo showing the full chain
```
