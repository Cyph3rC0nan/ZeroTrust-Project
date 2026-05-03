# Day 6 — EDR Configuration (Arm the Defenses)

**Date**: April 26, 2026  
**Target**: Configure full EDR stack so it's ready to detect + auto-respond to malware on Day 7  
**Criteria served**: 3 (EDR) — core setup  
**Connection**: SSH to Ubuntu, RDP to Windows, Browser for dashboards  

---

## What We're Doing Today

```
TODAY (Day 6):
  ├── Part A: Wazuh Active Response (auto-block + auto-kill)
  ├── Part B: Velociraptor Hunts (sweep for attack artifacts from Days 4-5)
  └── Part C: Wazuh VirusTotal integration (hash-based malware detection)
                │
                │  AFTER TODAY → EDR IS FULLY ARMED
                ▼
TOMORROW (Day 7):
  Drop malware → EDR detects → auto-responds → Velociraptor extracts → Sandbox
```

---

# Part A — Wazuh Active Response

---

## A1: SSH into the Wazuh Ubuntu Instance

```bash
ssh -i your-key.pem ubuntu@<UBUNTU-PUBLIC-IP>
```

Verify Wazuh is running:
```bash
sudo systemctl status wazuh-manager
```

## A2: Configure Active Response — Brute Force IP Block

Edit the Wazuh manager config:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

### Check if `firewall-drop` command is already defined

Search the file for `firewall-drop`:
```bash
sudo grep -n "firewall-drop" /var/ossec/etc/ossec.conf
```

If it's **NOT there**, add this command definition inside `<ossec_config>`:

```xml
<command>
  <name>firewall-drop</name>
  <executable>firewall-drop</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>
```

### Add Active Response Rule 1: Block IP on Brute Force

Add this inside `<ossec_config>` (after the `<command>` blocks):

```xml
<!-- ACTIVE RESPONSE 1: Auto-block IP on brute force attack -->
<!-- Triggers on rule 5712 = multiple authentication failures from same source -->
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5712</rules_id>
  <timeout>600</timeout>
  <repeated_offenders>30,60,120</repeated_offenders>
</active-response>
```

**What this does:**
- When 5+ failed logins from the same IP trigger Wazuh **rule 5712**
- Wazuh auto-runs `firewall-drop` → adds an iptables rule to **block the attacker IP**
- Block lasts 600 seconds (10 min)
- Repeat offenders get blocked for 30, 60, then 120 minutes

## A3: Configure Active Response — Kill Suspicious Process

### First: Find the right rule IDs from your Day 4-5 attacks

Check which Wazuh rules fired during your credential dumping and PowerShell attacks:

```bash
# Search recent alerts for LSASS / credential / process access events
sudo grep -i "lsass\|mimikatz\|credential\|procdump" /var/ossec/logs/alerts/alerts.json 2>/dev/null | tail -10

# Or search the alerts log
sudo grep "rule.*level.*[0-9]" /var/ossec/logs/alerts/alerts.log | tail -30

# List high-severity rules that fired recently
sudo cat /var/ossec/logs/alerts/alerts.json | python3 -c "
import sys, json
for line in sys.stdin:
    try:
        alert = json.loads(line)
        rule = alert.get('rule', {})
        if rule.get('level', 0) >= 10:
            print(f\"Rule {rule['id']}: {rule.get('description','')} (Level {rule['level']})\")
    except: pass
" 2>/dev/null | sort -u | tail -20
```

📝 **Write down the rule IDs you see** — especially any related to:
- Process access to LSASS (Sysmon Event 10)
- Suspicious PowerShell execution
- Credential dumping tools

### Add Active Response Rule 2: Kill Process

```xml
<!-- ACTIVE RESPONSE 2: Kill suspicious process -->
<!-- Add the actual rule IDs you found above -->
<!-- Common Sysmon-based rule IDs for LSASS access: 92050, 92051, 184665 -->
<!-- Replace XXXXX with your actual rule IDs from the search above -->
<active-response>
  <command>win_route-null</command>
  <location>local</location>
  <rules_id>92050,92051</rules_id>
  <timeout>0</timeout>
</active-response>
```

> [!TIP]
> If you can't find specific rule IDs from your logs, use these common Sysmon detection rules:
> - **92050–92051**: Sysmon Event 10 — Process accessing LSASS
> - **92000–92010**: Sysmon process creation with suspicious patterns
> - **184665–184666**: Mimikatz-related detections (if your Wazuh version has them)
> 
> You can also check available rules:
> ```bash
> sudo grep -r "lsass\|credential.dump\|mimikatz" /var/ossec/ruleset/rules/ | head -20
> ```

## A4: Validate Config and Restart

```bash
# Test config for syntax errors BEFORE restarting
sudo /var/ossec/bin/wazuh-analysisd -t
```

If it says `Configuration OK`:

```bash
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager
```

Make sure it's `active (running)`. If it fails:
```bash
# Check what went wrong
sudo journalctl -u wazuh-manager -n 30 --no-pager
```

📸 **Screenshot**: `ossec.conf` showing the active response configuration

## A5: Test the Brute Force Auto-Block (LIVE TEST)

This is the **demo moment** — prove the auto-block works.

### From Kali, run a brute force attack:

```bash
# Create password list
echo -e "password123\nadmin\nwrongpass\nletmein\nPassword1\nqwerty123\ntest1234\nP@ssw0rd\nSpring2026\nWelcome1\nFail1\nFail2" > /tmp/passwords.txt

# Method 1: Hydra brute force
hydra -l Administrator -P /tmp/passwords.txt rdp://<DC-PUBLIC-IP> -t 4 -V

# Method 2: SMB brute force (if Hydra doesn't work)
for pass in password123 admin wrongpass letmein Password1 qwerty123 test1234 P@ssw0rd Spring2026 Welcome1 Fail1 Fail2; do
    echo "Trying: $pass"
    smbclient //<DC-PUBLIC-IP>/C$ -U "Administrator%$pass" -c "exit" 2>&1
    sleep 0.5
done
```

### What Should Happen

1. Wazuh detects the flood of failed logins (Event 4625)
2. After 5+ failures, **rule 5712** fires
3. Active response triggers → **your Kali IP gets blocked**
4. Subsequent attempts from Kali should **fail/timeout**

### Verify the Block

**On the Wazuh server (SSH):**

```bash
# Check active response log
sudo cat /var/ossec/logs/active-responses.log | tail -20
```

You should see entries like:
```
Fri Apr 26 19:30:15 UTC 2026 /var/ossec/active-response/bin/firewall-drop add - <KALI-IP> ...
```

```bash
# Check iptables for the blocked IP
sudo iptables -L -n | grep DROP
```

📸 **Screenshot**: Active response log showing IP block
📸 **Screenshot**: `iptables -L -n` showing the blocked IP

**In Wazuh Dashboard (browser):**

1. Go to **Security Events** → search for `active.response` or `firewall-drop`
2. You should see the active response event
3. Also search for rule `5712` — the brute force trigger

📸 **Screenshot**: Wazuh dashboard showing brute force alert + active response

### Unblock Your Kali IP

```bash
# So you can continue working
sudo /var/ossec/active-response/bin/firewall-drop delete - <YOUR-KALI-IP>

# Verify it's removed
sudo iptables -L -n | grep DROP
```

---

## Part A — Screenshot Checklist

| # | Screenshot | Done? |
|---|---|---|
| 1 | `ossec.conf` — active response config section | [ ] |
| 2 | Wazuh dashboard — brute force alert (rule 5712) | [ ] |
| 3 | Wazuh dashboard — active response trigger event | [ ] |
| 4 | `active-responses.log` showing IP block | [ ] |
| 5 | `iptables -L -n` showing blocked IP | [ ] |

---

# Part B — Velociraptor Threat Hunting

**Goal**: Sweep both endpoints for artifacts from the Day 4-5 attacks + extract dropped files.

---

## B1: Open Velociraptor GUI

In your browser:
```
https://<UBUNTU-PUBLIC-IP>:8000
```

Login with your admin credentials.

## B2: Hunt — Process Execution History

1. Click **Hunt Manager** (left sidebar)
2. Click **+** (New Hunt)
3. Description: `Post-Attack Process Sweep`
4. **Select Artifacts** → search and add: `Windows.System.Pslist`
5. Click **Launch**
6. Wait for results from both clients

### What to Look For

- `powershell.exe` with suspicious command-line arguments
- `procdump64.exe` (credential dumping tool)
- `rundll32.exe` accessing comsvcs.dll (LOLBAS technique)
- `schtasks.exe` (scheduled task creation)
- `net.exe` or `net1.exe` (account creation)
- `mstsc.exe` (RDP lateral movement)
- `cmd.exe` spawned by unexpected parents

📸 **Screenshot**: Hunt results highlighting suspicious processes

## B3: Hunt — Recently Executed Binaries (Amcache)

1. New Hunt → Description: `Amcache - Executed Binaries`
2. Select Artifact: `Windows.Detection.Amcache`
3. Launch → wait for results

This shows **every executable that ran on the system**, even after deletion. This will reveal:
- `procdump64.exe`
- `payload.ps1` execution
- Any other attack tools

📸 **Screenshot**: Amcache results showing attack tools

## B4: Hunt — Scheduled Tasks

1. New Hunt → Description: `Scheduled Task Audit`
2. Select Artifact: `Windows.System.TaskScheduler`
3. Launch → wait for results

Look for the malicious tasks you created:
- `WindowsUpdate` → points to `C:\Temp\persistence.bat`
- `SystemHealthCheck` → runs `health.ps1`

📸 **Screenshot**: Task scheduler results showing attack persistence

## B5: Hunt — User Account Audit

1. New Hunt → Description: `User Account Audit`
2. Select Artifact: `Windows.Sys.Users`
3. Launch → wait for results

Should show the backdoor accounts:
- `hacker` (if not deleted yet)
- `svc_backup` (if not deleted yet)

📸 **Screenshot**: User list showing backdoor accounts

## B6: Hunt — Network Connections

1. New Hunt → Description: `Network Connection Audit`
2. Select Artifact: `Windows.Network.Netstat`
3. Launch → wait for results

Shows active connections — useful for finding C2 or lateral movement connections.

📸 **Screenshot**: Network connections from endpoints

## B7: Extract Dropped Attack Files

This is critical — pull the attack artifacts from the endpoints for evidence.

### Extract each dropped file:

For each file in the tracking table:

1. Click on **DC01** in the client list
2. Go to **Collected Artifacts** → **New Collection**
3. Search for: `Generic.Collectors.File`
4. In the parameters, set the **path** to:

**File 1**: `C:\Temp\payload.ps1`
**File 2**: `C:\Temp\persistence.bat`  
**File 3**: `C:\Users\Public\health.ps1`
**File 4**: `C:\Temp\Procdump\procdump64.exe`

> [!TIP]
> You can collect multiple files in one collection by using a glob pattern:
> ```
> C:\Temp\**
> ```
> This grabs everything in `C:\Temp` recursively.

5. Click **Launch**
6. Once collected, go to the **Results** tab
7. Click the download icon to save the files

📸 **Screenshot**: Velociraptor file collection results (showing extracted files with hashes)

### Export Hunt Results

For each hunt above:
1. Click on the hunt → **Results** tab
2. Click **Export** (CSV or JSON)
3. Save to your artifacts folder

```
Create folder: artifacts/20260426_edr_config/
├── hunt_pslist.csv
├── hunt_amcache.csv
├── hunt_tasks.csv
├── hunt_users.csv
├── hunt_netstat.csv
├── extracted_files/
│   ├── payload.ps1
│   ├── persistence.bat
│   ├── health.ps1
│   └── procdump64.exe
└── screenshots/
    ├── pslist_results.png
    ├── amcache_results.png
    ├── tasks_results.png
    ├── users_results.png
    ├── netstat_results.png
    └── file_collection.png
```

---

## Part B — Screenshot Checklist

| # | Screenshot | Done? |
|---|---|---|
| 1 | Hunt Manager showing all hunts | [ ] |
| 2 | Pslist — suspicious processes highlighted | [ ] |
| 3 | Amcache — attack tools visible | [ ] |
| 4 | Scheduled tasks — malicious tasks | [ ] |
| 5 | User accounts — backdoor accounts | [ ] |
| 6 | Network connections | [ ] |
| 7 | File collection — extracted attack files with hashes | [ ] |

---

# Part C — Wazuh VirusTotal Integration (Optional but Powerful)

This makes the Day 7 malware drop much more impactful — Wazuh will check file hashes against VirusTotal when new files are created.

---

## C1: Get a Free VirusTotal API Key

1. Go to [virustotal.com](https://www.virustotal.com/) and create a free account
2. Go to your profile → **API Key**
3. Copy the API key

## C2: Configure Wazuh to Use VirusTotal

SSH into the Ubuntu instance:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add this block inside `<ossec_config>`:

```xml
<!-- VirusTotal Integration -->
<integration>
  <name>virustotal</name>
  <api_key>YOUR_VIRUSTOTAL_API_KEY_HERE</api_key>
  <rule_id>554,550</rule_id>
  <alert_format>json</alert_format>
</integration>
```

> [!NOTE]
> - Rule **554** = File added to the system (Sysmon Event 11)
> - Rule **550** = Integrity monitoring — file modified
> - When these rules fire, Wazuh will send the file hash to VirusTotal
> - If VirusTotal knows the hash as malicious → Wazuh generates a **high-severity alert**

### Also enable Sysmon file monitoring rules

Make sure Sysmon Event ID 11 (FileCreate) rules are active:

```bash
# Check if Sysmon rules exist
sudo ls /var/ossec/ruleset/rules/ | grep sysmon
```

If you see `0800-sysmon_id_11.xml` or similar, they're already there.

## C3: Enable File Integrity Monitoring on Temp Directory

To monitor `C:\Temp` for new files (where malware will be dropped tomorrow):

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add or verify this section exists:

```xml
<syscheck>
  <directories check_all="yes" realtime="yes">C:\Temp</directories>
</syscheck>
```

## C4: Restart Wazuh

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager
```

## C5: Quick Test (Optional)

To verify the integration works, create a test file on the DC:

```powershell
# On DC via RDP — create a harmless file
echo "test" > C:\Temp\testfile.txt
```

Check Wazuh dashboard for a file integrity event. If you see it → the monitoring pipeline works.

> [!TIP]
> **Why this matters for Day 7**: When you drop the malware sample tomorrow, Wazuh will:
> 1. Detect the file creation via Sysmon Event ID 11
> 2. Send the hash to VirusTotal
> 3. VirusTotal returns "malicious" (since it's a known sample from MalwareBazaar)
> 4. Wazuh generates a **critical alert** with VirusTotal results
> 5. Active Response can then trigger based on this alert
>
> This is the "EDR detects malware" moment you want for the demo.

---

## End-of-Day Verification

### Confirm EDR is fully armed:

| Component | Status | How to Check |
|---|---|---|
| Wazuh Active Response — IP block | ✅ Tested | `active-responses.log` shows block |
| Wazuh Active Response — Process kill | ✅ Configured | Rule in `ossec.conf` |
| Velociraptor — Process hunting | ✅ Working | Hunt results show processes |
| Velociraptor — File extraction | ✅ Working | Attack files extracted |
| Velociraptor — Scheduled tasks | ✅ Working | Malicious tasks found |
| Wazuh — VirusTotal integration | ✅ Configured | API key in config |
| Wazuh — File monitoring | ✅ Configured | `C:\Temp` monitored |

**EDR is now armed. Tomorrow's malware drop will trigger the full pipeline.**

---

## ✅ Day 6 Completion Checklist

- [ ] **Part A — Wazuh Active Response**
  - [ ] `firewall-drop` command verified/added
  - [ ] Brute force IP block rule configured
  - [ ] Process kill rule configured
  - [ ] Config validated (`wazuh-analysisd -t`)
  - [ ] Wazuh restarted successfully
  - [ ] Live brute force test from Kali
  - [ ] IP block confirmed in `active-responses.log`
  - [ ] IP block confirmed in `iptables`
  - [ ] Kali IP unblocked
  - [ ] Screenshots taken (5)

- [ ] **Part B — Velociraptor Hunts**
  - [ ] Hunt: Process list (`Pslist`)
  - [ ] Hunt: Amcache (executed binaries)
  - [ ] Hunt: Scheduled tasks
  - [ ] Hunt: User accounts
  - [ ] Hunt: Network connections
  - [ ] File extraction: all 4 dropped files collected
  - [ ] Hunt results exported (CSV/JSON)
  - [ ] Screenshots taken (7)

- [ ] **Part C — VirusTotal Integration**
  - [ ] VirusTotal API key obtained
  - [ ] Integration added to `ossec.conf`
  - [ ] File integrity monitoring on `C:\Temp`
  - [ ] Wazuh restarted
  - [ ] Quick test done

- [ ] All artifacts saved to `artifacts/20260426_edr_config/`
- [ ] Git commit pushed

---

## What's Next: Day 7

Tomorrow is **the full pipeline moment**:

```
1. Download malware sample from MalwareBazaar (on Kali)
2. Transfer to DC endpoint → place in C:\Temp\
3. EDR detects:
   ├── Sysmon Event 11 (file created)
   ├── Wazuh alert (VirusTotal says "malicious")
   └── Active Response triggers (kill/block)
4. Velociraptor extracts the binary from endpoint
5. Binary saved → ready for sandbox on Day 8
```

**The story**: *"Our EDR detected the malware via hash reputation, auto-responded, and we used Velociraptor to forensically extract the sample for sandbox analysis."*
