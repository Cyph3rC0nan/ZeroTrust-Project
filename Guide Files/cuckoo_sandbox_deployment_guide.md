# 🔬 Cuckoo Sandbox — Local Deployment Guide

**Assigned to**: [Team Member Name]  
**Deadline**: Before April 24 (Day 8 of project plan)  
**Estimated time**: 3–5 hours  
**Priority**: HIGH

---

> [!IMPORTANT]
> **4-Hour Rule**: If you're stuck for more than 4 hours total and Cuckoo still isn't working, **STOP** and switch to **Plan B** (Section 8 at the bottom). We can't afford to lose days on this. Plan B (ANY.RUN + Hybrid Analysis) gives us equally good results for grading.

---

## 0. What You're Building (Big Picture)

```
┌─────────────────────────────────────────────────┐
│            YOUR PHYSICAL MACHINE                │
│                                                 │
│  ┌───────────────────────────────────────────┐  │
│  │     Ubuntu VM (Cuckoo Host)               │  │
│  │     - Cuckoo Sandbox server               │  │
│  │     - VirtualBox (nested)                 │  │
│  │     - Receives samples & produces reports │  │
│  │                                           │  │
│  │  ┌─────────────────────────────────────┐  │  │
│  │  │   Windows 7/10 VM (Analysis Guest)  │  │  │
│  │  │   - Malware runs HERE               │  │  │
│  │  │   - Monitored by Cuckoo agent       │  │  │
│  │  │   - Snapshot taken = clean state     │  │  │
│  │  └─────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

You submit malware to Cuckoo → it boots the Windows VM from a clean snapshot → runs the malware → records everything → shuts down → gives you a report.

---

## 1. Hardware Requirements

Make sure your machine meets these **before starting**:

| Requirement | Minimum | Recommended |
|---|---|---|
| RAM | 8 GB | 16 GB+ |
| Free disk space | 50 GB | 80 GB+ |
| CPU | 4 cores | 6+ cores |
| Virtualization | Must be enabled in BIOS | — |
| Nested virtualization | Must be supported | — |

### Check Virtualization Support

**If your physical machine runs Windows:**
- Open Task Manager → Performance → CPU → check "Virtualization: Enabled"
- If not enabled, reboot into BIOS/UEFI and enable **VT-x** (Intel) or **AMD-V** (AMD)

**If using VMware for the Ubuntu host VM:**
- VM Settings → Processors → ✅ "Virtualize Intel VT-x/EPT or AMD-V/RVI"
- This enables **nested virtualization** (VirtualBox inside VMware)

**If using VirtualBox for the Ubuntu host VM:**
- `VBoxManage modifyvm "UbuntuCuckoo" --nested-hw-virt on`

> [!WARNING]
> Nested virtualization can be slow. If your PC has less than 8 GB RAM, this setup will struggle. Consider Plan B.

---

## 2. Create the Ubuntu Host VM

### 2.1 Download Ubuntu

- Download **Ubuntu 22.04 LTS Server** ISO: https://releases.ubuntu.com/22.04/
- Server edition is lighter (no GUI overhead) — you'll access Cuckoo via web browser anyway

### 2.2 Create VM in VMware/VirtualBox

| Setting | Value |
|---|---|
| Name | `CuckooHost` |
| OS Type | Linux / Ubuntu 64-bit |
| RAM | **6 GB minimum** (8 GB if you can spare it) |
| CPU | **4 cores** |
| Disk | **60 GB** (dynamically allocated is fine) |
| Network | **Bridged Adapter** (so you can access Cuckoo web UI from your host browser) |
| Nested Virtualization | **Enabled** (critical!) |

### 2.3 Install Ubuntu

1. Boot from the ISO
2. Follow the installer — use defaults
3. Set username: `cuckoo` (or whatever you want)
4. **Enable OpenSSH** during install (makes life easier — you can SSH in from your host)
5. After install, reboot and log in

### 2.4 Post-Install Basics

```bash
# Update everything
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y net-tools curl wget git unzip

# Note your IP address (you'll need it later)
ip addr show
# Look for something like 192.168.x.x — write it down
```

---

## 3. Install VirtualBox (Inside Ubuntu VM)

Cuckoo uses VirtualBox to manage the analysis guest VM.

```bash
# Add VirtualBox repository
wget -q https://www.virtualbox.org/download/oracle_vbox_2016.asc -O- | sudo apt-key add -
echo "deb [arch=amd64] https://download.virtualbox.org/virtualbox/debian jammy contrib" | sudo tee /etc/apt/sources.list.d/virtualbox.list

# Install VirtualBox
sudo apt update
sudo apt install -y virtualbox-7.0

# Verify installation
vboxmanage --version
# Should output something like 7.0.x
```

> If VirtualBox fails to install from the repo, download the `.deb` directly from https://www.virtualbox.org/wiki/Linux_Downloads

```bash
# Add your user to the vboxusers group
sudo usermod -aG vboxusers $USER
```

---

## 4. Install Cuckoo Sandbox Dependencies

### 4.1 System Dependencies

```bash
sudo apt install -y python3 python3-pip python3-dev python3-venv \
    libffi-dev libssl-dev libxml2-dev libxslt1-dev \
    libjpeg-dev zlib1g-dev \
    mongodb postgresql postgresql-contrib \
    tcpdump apparmor-utils libcap2-bin \
    ssdeep libfuzzy-dev \
    volatility3
```

### 4.2 Configure tcpdump Permissions

Cuckoo uses tcpdump to capture network traffic. It needs special permissions:

```bash
sudo setcap cap_net_raw,cap_net_admin=eip $(which tcpdump)

# Verify
getcap $(which tcpdump)
# Should show: /usr/bin/tcpdump cap_net_admin,cap_net_raw=eip
```

### 4.3 Configure PostgreSQL (Cuckoo's Database)

```bash
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Create Cuckoo database
sudo -u postgres psql -c "CREATE USER cuckoo WITH PASSWORD 'cuckoo';"
sudo -u postgres psql -c "CREATE DATABASE cuckoo OWNER cuckoo;"
```

### 4.4 Start MongoDB (for Web UI)

```bash
sudo systemctl start mongod
sudo systemctl enable mongod
```

> If `mongod` service doesn't exist, install MongoDB separately:
> ```bash
> # Import MongoDB GPG key and add repo (MongoDB 6.0)
> curl -fsSL https://pgp.mongodb.com/server-6.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-6.0.gpg --dearmor
> echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
> sudo apt update
> sudo apt install -y mongodb-org
> sudo systemctl start mongod
> sudo systemctl enable mongod
> ```

---

## 5. Install Cuckoo Sandbox

### 5.1 Create a Python Virtual Environment

```bash
# Create a dedicated directory
mkdir ~/cuckoo-env && cd ~/cuckoo-env

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Upgrade pip
pip install --upgrade pip setuptools wheel
```

### 5.2 Install Cuckoo

```bash
# Install Cuckoo via pip
pip install cuckoo

# If the above fails or you want the latest, install from GitHub:
# pip install git+https://github.com/cuckoosandbox/cuckoo.git
```

> [!NOTE]
> If you encounter errors about Python version incompatibility, Cuckoo 2.x requires Python 2.7 in some older versions. In that case, use the **CAPEv2** fork instead (it's Python 3 compatible and actively maintained):
> ```bash
> # Alternative: CAPEv2 (modern Cuckoo fork)
> cd ~
> git clone https://github.com/kevoreilly/CAPEv2.git
> cd CAPEv2
> pip install -r requirements.txt
> ```
> If you go the CAPEv2 route, follow their docs at https://capev2.readthedocs.io/ — the concepts are identical to Cuckoo but commands differ slightly. **Let the team know which one you installed.**

### 5.3 Initialize Cuckoo (if using original Cuckoo)

```bash
# Initialize Cuckoo's working directory
cuckoo init

# This creates ~/.cuckoo/ with all config files
```

### 5.4 Download Cuckoo Community Signatures

```bash
cuckoo community
# Downloads community-contributed detection signatures
```

---

## 6. Create the Windows Analysis VM

This is the VM where malware will actually run.

### 6.1 Get a Windows Image

**Option A — Windows 10 Evaluation (Free, Legal)**:
- Download from: https://www.microsoft.com/en-us/evalcenter/evaluate-windows-10-enterprise
- 90-day evaluation, no product key needed

**Option B — Windows 7 (Lighter, Better for Malware Analysis)**:
- If you have an ISO available, Windows 7 is lighter and more malware tends to target it
- Turn off automatic updates after install

### 6.2 Create the VM in VirtualBox

```bash
# Create the VM via command line
vboxmanage createvm --name "cuckoo-win10" --ostype Windows10_64 --register

# Configure VM settings
vboxmanage modifyvm "cuckoo-win10" \
    --memory 2048 \
    --cpus 2 \
    --vram 64 \
    --acpi on \
    --boot1 dvd --boot2 disk \
    --nic1 hostonly --hostonlyadapter1 vboxnet0

# Create virtual hard disk
vboxmanage createhd --filename ~/VirtualBox\ VMs/cuckoo-win10/cuckoo-win10.vdi --size 40000

# Add storage controllers
vboxmanage storagectl "cuckoo-win10" --name "SATA" --add sata
vboxmanage storageattach "cuckoo-win10" --storagectl "SATA" --port 0 --device 0 --type hdd --medium ~/VirtualBox\ VMs/cuckoo-win10/cuckoo-win10.vdi

# Attach Windows ISO
vboxmanage storageattach "cuckoo-win10" --storagectl "SATA" --port 1 --device 0 --type dvddrive --medium /path/to/windows10.iso
```

### 6.3 Create Host-Only Network

```bash
# Create host-only network interface
vboxmanage hostonlyif create
# This creates vboxnet0

# Configure the network
vboxmanage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1 --netmask 255.255.255.0
```

### 6.4 Install Windows

```bash
# Start the VM (with GUI — you'll need display access for Windows install)
# If you're SSH'd in, you may need to use VBoxHeadless + RDP or install Ubuntu Desktop temporarily
vboxmanage startvm "cuckoo-win10" --type gui
```

During Windows installation:
1. Install Windows normally
2. **Set a simple password** or no password for the user
3. **Computer name**: `analysis-pc` (or anything)

### 6.5 Configure the Windows Guest (CRITICAL STEPS)

After Windows is installed and running, do ALL of the following **inside the Windows VM**:

#### Disable Security Features
```powershell
# Run in PowerShell as Administrator inside the Windows VM:

# Disable Windows Defender
Set-MpPreference -DisableRealtimeMonitoring $true
# Also: Windows Security > Virus & Threat Protection > Manage Settings > Turn OFF everything

# Disable Windows Firewall
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False

# Disable Windows Update
sc stop wuauserv
sc config wuauserv start= disabled

# Disable UAC
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" /v EnableLUA /t REG_DWORD /d 0 /f
```

#### Set Static IP
```
Control Panel > Network > Change Adapter Settings > 
Right-click Ethernet > Properties > IPv4:

IP:      192.168.56.101
Mask:    255.255.255.0
Gateway: 192.168.56.1
DNS:     8.8.8.8
```

#### Install Python (for Cuckoo Agent)
1. Download **Python 3.10+** installer from https://www.python.org/downloads/ inside the Windows VM
2. Install with **"Add to PATH"** checked
3. Verify: open CMD → `python --version`

#### Install the Cuckoo Agent
The agent runs inside the Windows VM and communicates with the Cuckoo host.

```powershell
# Download the agent from the Cuckoo host
# From inside the Windows VM, open a browser and go to:
# http://192.168.56.1:8090  (or wherever your Cuckoo is running — we'll set this up)

# OR manually copy the agent file
# The agent is at: ~/.cuckoo/agent/agent.py (on the Ubuntu host)
# Copy it to the Windows VM desktop
```

**Copy `agent.py` to the Windows VM:**
- The file is located at `~/.cuckoo/agent/agent.py` on the Ubuntu host
- Transfer it to the Windows VM (use a shared folder, Python HTTP server, or USB)

Quick transfer method from Ubuntu host:
```bash
# On the Ubuntu host, serve the file:
cd ~/.cuckoo/agent/
python3 -m http.server 8888 --bind 192.168.56.1
# Then inside the Windows VM, open browser to http://192.168.56.1:8888/agent.py
# Save it to Desktop
```

**Set agent.py to auto-start:**
1. Press `Win + R` → type `shell:startup` → Enter
2. Copy `agent.py` into the Startup folder
3. **Run it once manually now**: double-click `agent.py` — it should open a console window and start listening

#### Install Additional Software (makes analysis richer)
Inside the Windows VM, install:
- **Adobe Acrobat Reader** (for PDF malware testing)
- **Microsoft Office** (if available — for macro malware)
- **7-Zip**
- **A web browser** (Chrome/Firefox)

> [!TIP]
> The more "real" the Windows VM looks, the better the analysis. Some malware checks if the machine looks like a sandbox (no software installed = suspicious). Installing common software helps evade anti-sandbox tricks.

### 6.6 Take the Snapshot (CRITICAL!)

Once everything is configured and the agent is running:

```bash
# Back on the Ubuntu host
# Shut down the Windows VM cleanly first (Start > Shut Down inside Windows)

# Take the snapshot — Cuckoo will restore to THIS state before each analysis
vboxmanage snapshot "cuckoo-win10" take "clean_snapshot" --description "Clean state with agent running, AV disabled"

# Verify
vboxmanage snapshot "cuckoo-win10" list
```

> [!CAUTION]
> **Do NOT skip the snapshot!** Cuckoo needs this to revert the VM to a clean state between analyses. Without it, Cuckoo won't work.

---

## 7. Configure Cuckoo

### 7.1 Edit Main Configuration

```bash
nano ~/.cuckoo/conf/cuckoo.conf
```

Key settings to change:
```ini
[cuckoo]
machinery = virtualbox
memory_dump = yes

[resultserver]
ip = 192.168.56.1
port = 2042

[database]
connection = postgresql://cuckoo:cuckoo@localhost/cuckoo
```

### 7.2 Configure VirtualBox Machinery

```bash
nano ~/.cuckoo/conf/virtualbox.conf
```

```ini
[virtualbox]
mode = headless
path = /usr/bin/vboxmanage
interface = vboxnet0

[cuckoo-win10]
label = cuckoo-win10
platform = windows
ip = 192.168.56.101
snapshot = clean_snapshot
interface = 
resultserver_ip = 192.168.56.1
resultserver_port = 2042
tags = 
options = 
```

### 7.3 Configure Network Routing

So malware can reach the internet (for C2 detection):

```bash
nano ~/.cuckoo/conf/routing.conf
```

```ini
[routing]
route = internet
internet = ens33
# ^ Replace ens33 with your Ubuntu host's main network interface name
# Check with: ip route | grep default
```

Enable IP forwarding and NAT on the Ubuntu host:

```bash
# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1

# Make it persistent
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# Set up NAT (replace ens33 with your actual interface)
sudo iptables -t nat -A POSTROUTING -o ens33 -s 192.168.56.0/24 -j MASQUERADE
sudo iptables -A FORWARD -i vboxnet0 -o ens33 -j ACCEPT
sudo iptables -A FORWARD -i ens33 -o vboxnet0 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Save iptables rules
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

### 7.4 Configure Reporting

```bash
nano ~/.cuckoo/conf/reporting.conf
```

```ini
[mongodb]
enabled = yes
host = 127.0.0.1
port = 27017
db = cuckoo

[reporthtml]
enabled = yes
```

---

## 8. Start Cuckoo & Test

### 8.1 Start Cuckoo

You need **3 terminal windows/tabs** (use `tmux` or multiple SSH sessions):

**Terminal 1 — Cuckoo Core:**
```bash
cd ~/cuckoo-env
source venv/bin/activate
cuckoo
```

**Terminal 2 — Cuckoo Web UI:**
```bash
cd ~/cuckoo-env
source venv/bin/activate
cuckoo web --host 0.0.0.0 --port 8080
```

**Terminal 3 — Cuckoo API (optional but useful):**
```bash
cd ~/cuckoo-env
source venv/bin/activate
cuckoo api --host 0.0.0.0 --port 8090
```

### 8.2 Access the Web UI

From your **physical machine's browser**, go to:
```
http://<ubuntu-vm-ip>:8080
```

You should see the Cuckoo dashboard.

### 8.3 Run a Test Analysis

1. Open the Cuckoo web UI
2. Click **Submit** → choose a file
3. **Test with a harmless file first** (e.g., a simple `.txt` or `calc.exe`)
4. Hit Submit → watch the analysis run
5. After ~2 minutes, check the **Results** page

**What to look for:**
- ✅ VM boots from snapshot
- ✅ File gets submitted to the VM
- ✅ Analysis runs for the configured timeout
- ✅ Results/report page shows behavioral data

### 8.4 Test with EICAR (Safe Malware Test File)

```bash
# Download EICAR test file (this is NOT real malware — it's an industry-standard test string)
echo 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' > /tmp/eicar.com

# Submit via API:
curl -F "file=@/tmp/eicar.com" http://localhost:8090/tasks/create/file
```

---

## 9. Plan B — If Cuckoo Doesn't Work (Equally Valid for Grading)

> [!IMPORTANT]
> If you've spent 4+ hours and Cuckoo still isn't running, switch to this immediately. **These are REAL sandbox platforms used by professional SOCs.** The teacher will not penalize us for using industry tools.

### Option 1: ANY.RUN (Best Reports, Free Tier)
- Go to https://any.run
- Create a **free community** account
- Upload malware samples → watch them execute live → get full reports
- **Free tier**: 5 submissions/day, public results
- Reports include: process trees, network activity, MITRE ATT&CK mapping, screenshots

### Option 2: Hybrid Analysis (by CrowdStrike)
- Go to https://www.hybrid-analysis.com
- Create a free account
- Upload samples → get detailed behavioral reports
- Reports include: file system changes, registry changes, network IOCs

### Option 3: Joe Sandbox Cloud
- https://www.joesandbox.com
- Free community edition available
- Very detailed reports

### What to Do with Plan B:
1. Submit **2 malware samples** (from MalwareBazaar: https://bazaar.abuse.ch/)
2. Save the **full PDF/HTML reports** from the sandbox
3. Extract IOCs from the reports
4. Take **screenshots** of the sandbox running the samples
5. Document everything the same way we would with Cuckoo

---

## 10. Deliverables Checklist

When you're done, confirm the following to the team:

- [ ] **Which sandbox is running?** (Cuckoo / CAPEv2 / ANY.RUN / Hybrid Analysis)
- [ ] **Can you submit a sample and get a report?** (Yes/No)
- [ ] **Screenshot**: Dashboard or main interface running
- [ ] **Screenshot**: A test analysis report (even with a benign file)
- [ ] **IP address / URL** to access the sandbox (if Cuckoo — so we can submit samples remotely)
- [ ] **Write down any issues** you ran into (we'll include troubleshooting in our documentation)

---

## 11. Troubleshooting (Common Issues)

| Problem | Solution |
|---|---|
| `vboxmanage: command not found` | VirtualBox not installed. Reinstall or check PATH. |
| VM won't start — "VT-x not available" | Enable virtualization in BIOS. For nested: enable nested virt on the outer VM. |
| Cuckoo says "machine not available" | Check `virtualbox.conf` — label must match the VM name exactly. Snapshot name must match. |
| Agent not reachable | Check Windows VM IP is `192.168.56.101`, agent.py is running, firewall is OFF. |
| Analysis times out | Agent isn't running in the VM, or network between host and guest is broken. Ping `192.168.56.101` from Ubuntu host. |
| `pip install cuckoo` fails | Try Python 2.7 venv for Cuckoo 2.x, or switch to CAPEv2 (Python 3). |
| MongoDB won't start | Check logs: `sudo journalctl -u mongod`. Usually a permissions issue — try `sudo chown -R mongodb:mongodb /var/lib/mongodb`. |
| No internet from analysis VM | Check iptables NAT rules and IP forwarding (`sysctl net.ipv4.ip_forward`). |
| Web UI shows blank page | MongoDB probably not running. Check with `sudo systemctl status mongod`. |

---

## 12. Quick Reference — Key Paths & Commands

```bash
# Cuckoo config files
~/.cuckoo/conf/cuckoo.conf       # Main config
~/.cuckoo/conf/virtualbox.conf   # VM settings
~/.cuckoo/conf/reporting.conf    # Report settings
~/.cuckoo/conf/routing.conf      # Network routing

# Start Cuckoo
source ~/cuckoo-env/venv/bin/activate
cuckoo                           # Core engine
cuckoo web --host 0.0.0.0       # Web UI
cuckoo api --host 0.0.0.0       # API

# VirtualBox commands
vboxmanage list vms              # List all VMs
vboxmanage snapshot "cuckoo-win10" list  # List snapshots
vboxmanage startvm "cuckoo-win10"       # Start VM
vboxmanage controlvm "cuckoo-win10" poweroff  # Force stop

# Submit sample via CLI
cuckoo submit /path/to/malware.exe
```

---

> [!NOTE]
> **Questions?** Reach out to the team chat immediately if you're stuck. Don't spend hours Googling. We have 2 weeks — every hour counts.

**Good luck! 🚀**
