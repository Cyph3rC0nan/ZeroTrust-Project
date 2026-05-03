# Demo Presentation Plan — May 4th

**Team**: Kenan (Lead + Malware), Raul (Attack), Nuray (Log), Nihat (EDR)  
**Presentation**: 5 minutes + 2 min Q&A  
**Deadline**: May 3rd evening (everything done) → May 4th present  

---

## Presentation Flow (5 Minutes Total)

```
[0:00 - 0:40]  SLIDE 1-3: Kenan introduces the project (SLIDES ONLY)
[0:40 - 0:50]  SLIDE 4: "Let's see it in action" → VIDEO STARTS
[0:50 - 1:40]  VIDEO zooms to TOP-LEFT: Raul narrates Attack
[1:40 - 2:30]  VIDEO zooms to TOP-RIGHT: Nuray narrates Logs
[2:30 - 3:20]  VIDEO zooms to BOTTOM-LEFT: Nihat narrates EDR
[3:20 - 4:10]  VIDEO zooms to BOTTOM-RIGHT: Kenan narrates Malware Analysis
[4:10 - 4:20]  VIDEO zooms out → shows all 4 quadrants together
[4:20 - 4:50]  SLIDE 5-6: Kenan wraps up (results + closing)
[4:50 - 5:00]  SLIDE 7: "Thank You — Questions?"
[5:00 - 7:00]  Q&A — any member answers their domain
```

**Time per person during video: ~50 seconds each**

---

## 3-Day Execution Plan

### DAY 1 — May 1 (Today): RECORD EVERYTHING

**Everyone does this FIRST:**
1. Install **OBS Studio** (free): https://obsproject.com/download
2. Set recording to: **1920×1080**, **30fps**, **MP4 format**
3. OBS Settings → Output → Recording Format → mp4

---

#### Raul — Record Attack Footage (~5-8 min raw)

Record these on Kali terminal (ONE continuous recording or separate clips):

| Clip | What to Record | Duration |
|------|---------------|----------|
| A1 | PowerShell attack: run `payload.ps1` or encoded command on DC | 1-2 min |
| A2 | Account creation: `net user hacker P@ss... /add` + `net localgroup Administrators hacker /add` | 30 sec |
| A3 | Brute force: Hydra running against DC | 1-2 min |
| A4 | Credential dump: procdump targeting LSASS | 30 sec |
| A5 | Lateral movement: RDP from Client to DC | 1 min |

**Tips for Raul:**
- Make terminal font BIG (18-20pt) so text is readable when zoomed out
- Use a dark terminal with green/white text (looks professional)
- Don't rush — each command should be visible for 2-3 seconds
- Total raw footage: ~5-8 minutes (we'll speed it up to ~50 seconds)

**Save as**: `attack_raw.mp4`

---

#### Nuray — Record Log Analysis Footage (~5-8 min raw)

Record these on the Wazuh Dashboard (browser):

| Clip | What to Record | Duration |
|------|---------------|----------|
| L1 | Wazuh Dashboard overview: show agents, total alerts | 30 sec |
| L2 | Search for PowerShell alerts → click one → show details | 1-2 min |
| L3 | Search for account creation (Event 4720) → show details | 1 min |
| L4 | Search for brute force alerts (Rule 5712) → show the flood of 4625 events | 1-2 min |
| L5 | Search for lateral movement (Event 4624 Type 10) → show details | 1 min |

**Tips for Nuray:**
- Zoom browser to 110-125% so text is readable
- Use the Wazuh search bar clearly — type slowly so viewers can see
- Click on alerts to show the JSON details
- Show the alert level and rule description

**Save as**: `logs_raw.mp4`

---

#### Nihat — Record EDR Footage (~5-8 min raw)

Record these across Wazuh Active Response + Velociraptor:

| Clip | What to Record | Duration |
|------|---------------|----------|
| E1 | Show Active Response config in ossec.conf (terminal) | 30 sec |
| E2 | Show brute force happening → then Active Response blocks IP | 1-2 min |
| E3 | Show the blocked IP in iptables or Wazuh active-response log | 30 sec |
| E4 | Velociraptor: show connected clients | 30 sec |
| E5 | Velociraptor: run a hunt (e.g., Pslist or file collection) → show results | 1-2 min |
| E6 | Velociraptor: show the extracted malware file in collected artifacts | 1 min |

**Tips for Nihat:**
- For Velociraptor, zoom the browser so hunt results are readable
- Show the "New Hunt" → select artifact → launch → results flow
- For Active Response, show BEFORE (attack works) → AFTER (attack blocked)

**Save as**: `edr_raw.mp4`

---

#### Kenan — Record Malware Analysis Footage (~5-8 min raw)

Record across ANY.RUN + Kali (static analysis):

| Clip | What to Record | Duration |
|------|---------------|----------|
| M1 | ANY.RUN: upload the sample → start analysis | 30 sec |
| M2 | ANY.RUN: live analysis playing (the desktop where malware runs) | 1-2 min |
| M3 | ANY.RUN: process tree view | 30 sec |
| M4 | ANY.RUN: network activity (HTTP/DNS requests) | 30 sec |
| M5 | ANY.RUN: MITRE ATT&CK mapping | 30 sec |
| M6 | Kali: strings command running on sample → show URLs/IPs found | 1 min |
| M7 | Kali: YARA rule matching the sample (yara rule.yar sample.exe) | 30 sec |

**Tips for Kenan:**
- ANY.RUN recordings look very impressive — the live desktop is visually striking
- For strings: pipe through grep to show only interesting results
- For YARA: show the rule briefly, then run it and show "MATCH"

**Save as**: `malware_raw.mp4`

---

#### Deadline: Everyone sends their raw MP4 to Kenan by **May 1 midnight**

---

### DAY 2 — May 2: EDIT VIDEO + BUILD SLIDES

---

#### Part A: Video Editing (assign to whoever is best — or Kenan)

**Tool: CapCut Desktop** (FREE, easy, no watermark)
- Download: https://www.capcut.com/download
- Works on Windows, very beginner-friendly

**Step-by-Step Editing Guide:**

**Step 1: Create the 4-Quadrant Layout**

1. Open CapCut → New Project → set to **1920×1080**
2. Import all 4 raw videos
3. Create the base layout:
   - Drag `attack_raw.mp4` to timeline → **Position**: X=-480, Y=-270, Scale 50% (top-left)
   - Drag `logs_raw.mp4` to timeline → **Position**: X=480, Y=-270, Scale 50% (top-right)
   - Drag `edr_raw.mp4` to timeline → **Position**: X=-480, Y=270, Scale 50% (bottom-left)
   - Drag `malware_raw.mp4` to timeline → **Position**: X=480, Y=270, Scale 50% (bottom-right)

4. Add colored borders between quadrants:
   - Add a text/shape overlay with thin white/cyan lines dividing the 4 sections

5. Add labels to each quadrant:
   - Top-left: "🔴 ATTACK SIMULATION"
   - Top-right: "📊 LOG ANALYSIS"
   - Bottom-left: "🛡️ EDR RESPONSE"
   - Bottom-right: "🔬 MALWARE ANALYSIS"

**Step 2: Create Zoom Transitions (THE KEY EFFECT)**

The zoom effect is done using CapCut's **Keyframe animation**:

**Phase 1 — All 4 quadrants visible (0:00 - 0:03)**
- Show all 4 running simultaneously for 3 seconds (impressive opening)

**Phase 2 — Zoom to Attack / top-left (0:03 - 0:53)**
- At 0:03: Add keyframe → Scale 100%, Position centered
- At 0:04: Add keyframe → Scale 200%, Position X=480, Y=270 (zooms into top-left)
- Attack plays for 50 seconds zoomed in
- At 0:52: Add keyframe → start zooming out
- At 0:53: Add keyframe → Scale 100%, Position centered (back to 4-quad)

**Phase 3 — Zoom to Logs / top-right (0:53 - 1:43)**
- At 0:53: visible all 4 for 1 sec
- At 0:54: Zoom to Scale 200%, Position X=-480, Y=270 (top-right)
- Logs play for 50 seconds
- At 1:42: start zoom out
- At 1:43: back to 4-quad

**Phase 4 — Zoom to EDR / bottom-left (1:43 - 2:33)**
- Same pattern: zoom to Scale 200%, Position X=480, Y=-270
- EDR plays for 50 seconds

**Phase 5 — Zoom to Malware / bottom-right (2:33 - 3:23)**
- Zoom to Scale 200%, Position X=-480, Y=-270
- Malware analysis plays for 50 seconds

**Phase 6 — Final zoom out (3:23 - 3:30)**
- Zoom back to Scale 100% showing all 4 quadrants
- Hold for 5-7 seconds (dramatic ending)
- Fade to black

> ALTERNATIVE EASIER METHOD: If keyframing is too complex, use CapCut's 
> built-in "Ken Burns" effect or simply cut between full-screen clips:
> - Just show attack clip full screen → transition → logs full screen → etc.
> - Add a small "quadrant indicator" overlay in the corner showing which 
>   section is active
> - This is much easier and still looks professional

**Step 3: Speed Up the Clips**

Each person's raw footage is ~5-8 minutes but needs to fit in ~50 seconds:
- Right-click clip → Speed → select **6x-8x speed**
- Or use "Speed Curve" for variable speed (slow on important moments, fast on typing)
- Test playback — commands should still be briefly readable

**Step 4: Add Audio**

- **Mute all original audio** (right-click each clip → Mute)
- The narration will be LIVE during presentation (not in the video)
- Add subtle **background music**:
  - YouTube Audio Library: https://studio.youtube.com/channel/UC/music (free, royalty-free)
  - Search for: "technology", "cybersecurity", "tension"
  - Good picks: ambient electronic, low-key cinematic
  - Set volume to **15-20%** (very quiet — just ambiance, not distracting)

**Step 5: Add Text Overlays**

Add brief text labels at key moments:
- When attack starts: "⚔️ PowerShell Recon Script Deployed"
- When brute force starts: "⚔️ Hydra Brute Force — 500 attempts/min"
- When IP is blocked: "🛡️ BLOCKED — Active Response triggered in 3 seconds"
- When YARA matches: "✅ YARA Rule Match — Malware Detected"

These text callouts make the video self-explanatory even without narration.

**Step 6: Export**

- Export as **1080p MP4**, 30fps
- Target file size: under 100MB (for embedding in slides)
- Total video length: **3 minutes 30 seconds**

**Save as**: `demo_video_final.mp4`

---

#### Part B: Build Slides (Use NotebookLM + Google Slides)

**How to use NotebookLM for this:**

1. Go to https://notebooklm.google.com/
2. Create a new notebook
3. Upload your **Final Report.docx** as a source
4. Ask NotebookLM:
   > "Based on this cybersecurity lab report, create an 8-slide presentation 
   > outline for a 5-minute demo aimed at a cybersecurity jury. The opening 
   > must frame the problem: organizations invest heavily in perimeter defense 
   > (firewalls, WAFs, cloud security) but the real danger begins AFTER initial 
   > access — when an attacker is already inside the network, moving laterally, 
   > dropping malware, stealing credentials, and establishing persistence. 
   > Most breaches go undetected for 200+ days. Our project addresses exactly 
   > this gap: detecting and responding to post-exploitation threats in real-time. 
   > Include: the threat problem, our solution architecture, the 4-phase pipeline 
   > (attack → detection → response → analysis), results, and a closing slide. 
   > Keep text minimal — bullet points only."
5. Use the generated outline to build slides in **Google Slides** or **PowerPoint**

**Slide Structure (8 slides):**

| Slide | Title | Content | Who Speaks | Time |
|-------|-------|---------|------------|------|
| 1 | Title Slide | Project name, team names, date, dark cyber aesthetic | — | 3 sec |
| 2 | The Real Threat | "The attacker is already inside" — 3 stats + the problem statement (see below) | Kenan | 20 sec |
| 3 | Our Solution | Architecture diagram + "We built a detection-to-analysis pipeline" + tool logos | Kenan | 15 sec |
| 4 | Live Demo | "Watch the full pipeline in action" → VIDEO PLAYS | ALL | 3:30 |
| 5 | Results | Detection matrix 7/7, "100% detection rate", key stats | Kenan | 10 sec |
| 6 | Lessons & Impact | 3 bullet points + "What we'd add in production" | Kenan | 10 sec |
| 7 | Thank You + Q&A | "Questions?" + team names + contact | — | — |
| 8 | (Backup) | Extra screenshots if video fails | — | — |

**Slide 2 Content — "The Real Threat" (THE HOOK):**
```
Title: "The Real Danger Isn't Getting In. It's What Happens After."

• Average dwell time: 204 days before breach detection (IBM X-Force)
• 68% of breaches involve post-exploitation techniques (Verizon DBIR)
• Firewalls, WAFs, and cloud security stop initial access —
  but who detects the attacker INSIDE your network?

→ We built a lab that answers this question.
```

**Slide Design Tips:**
- Use a **dark theme** (black/dark blue background — fits cybersecurity aesthetic)
- Font: **Roboto** or **Inter** (clean, modern)
- Colors: cyan (#00D4FF), green (#00FF88), red (#FF4444) on dark background
- Minimal text — max 4 bullet points per slide
- Use the architecture diagram from your report on Slide 2
- The video on Slide 4 should be **embedded** (Google Slides: Insert → Video → Upload)

---

### DAY 3 — May 3: REHEARSE + Q&A PREP

---

#### Part A: Rehearsal (Do this 3-4 times minimum)

**Run-through checklist:**

1. **Timing**: Use a stopwatch. Each person practices their 50-second narration over the video
2. **Transitions**: Practice the handoff between speakers (who talks when)
3. **Video sync**: Make sure each person starts talking when their quadrant zooms in
4. **Backup plan**: If video doesn't play → have screenshots ready on extra slides

**Narration Scripts (each person memorizes their ~50 seconds):**

---

**Kenan — Introduction (0:00 - 0:45, slides 1-3)**

*(Slide 1 — Title — 3 seconds, just let them read it)*

*(Slide 2 — The Real Threat — 20 seconds)*
> "Good morning. Before we show you our project, let me ask a question: 
> where does the real danger begin in a cyberattack? Most organizations 
> invest heavily in perimeter defenses — firewalls, WAFs, cloud security. 
> But the uncomfortable truth is: the real damage happens AFTER initial access. 
> When an attacker is already inside your network — moving laterally, dumping 
> credentials, dropping malware, establishing persistence — and your infrastructure 
> can't see it. The average breach goes undetected for over 200 days. 
> Our project addresses exactly this blind spot."

*(Slide 3 — Our Solution — 15 seconds)*
> "We built a full detection-to-analysis pipeline on AWS — a Windows Active 
> Directory domain, monitored by Wazuh SIEM with Sysmon telemetry, and 
> Velociraptor for forensic hunting. We simulated real attacks, detected 
> every one, auto-responded, and forensically analyzed extracted malware. 
> Let me show you the full pipeline in action."

---

**Raul — Attack Narration (while Attack quadrant is zoomed)**
> "Here I'm executing real attack techniques from our Kali machine. First, 
> a PowerShell reconnaissance script that collects system information. Then 
> I create a backdoor admin account called 'hacker'. Next, a brute force 
> attack using Hydra — sending hundreds of password guesses. Finally, I dump 
> credentials from LSASS memory using Procdump. All attacks are mapped to 
> the MITRE ATT&CK framework."

---

**Nuray — Log Analysis Narration (while Logs quadrant is zoomed)**
> "Every attack Raul executed was detected by our SIEM. Here in the Wazuh 
> Dashboard we can see the alerts. The PowerShell execution triggered Sysmon 
> Event ID 1 with full command-line logging. The account creation generated 
> Windows Event 4720. The brute force created a flood of Event 4625 failed 
> logon alerts. We achieved a 100% detection rate across all 7 attack 
> techniques."

---

**Nihat — EDR Narration (while EDR quadrant is zoomed)**
> "Detection alone isn't enough — we need response. Our Wazuh Active Response 
> automatically blocked the brute force attacker's IP within seconds. You can 
> see the iptables rule being added automatically. On the Velociraptor side, 
> we ran forensic hunts across all endpoints — collecting process lists, 
> checking for persistence, and most importantly, extracting the malware 
> binary from the infected endpoint for analysis."

---

**Kenan — Malware Analysis Narration (while Malware quadrant is zoomed)**
> "After extracting the malware with Velociraptor, I submitted it to ANY.RUN 
> sandbox. Here you can see the malware executing live — it's making network 
> connections to C2 servers, modifying the registry for persistence, and 
> attempting to exfiltrate data. I also ran static analysis — strings extraction, 
> PE header analysis, and Ghidra decompilation. From all this, we extracted 
> IOCs and wrote two YARA rules that successfully detect this malware family."

---

**Kenan — Closing (4:20 - 4:50, slides)**
> "To summarize: we detected 7 out of 7 attacks, automated response for 
> brute force and malware, and completed a full forensic analysis pipeline — 
> from detection to YARA rules. Thank you. We're happy to take questions."

---

#### Part B: Q&A Preparation

**15 Expected Questions + Prepared Answers:**

**Q1: Why did you choose Wazuh over other SIEMs like Splunk?**
> Wazuh is open-source, has built-in Active Response for automated containment, 
> and integrates natively with VirusTotal. Splunk is enterprise-grade but costly 
> and overkill for a lab environment.

**Q2: Why use two tools (Wazuh + Velociraptor) instead of one?**
> They serve different roles. Wazuh excels at real-time alerting and automated 
> response. Velociraptor excels at forensic hunting and remote file extraction. 
> Together they cover both the automated and investigative sides of EDR.

**Q3: What is Sysmon and why is it important?**
> Sysmon is a Windows driver that logs detailed system activity — process creation 
> with command-line arguments, network connections, file creation, and registry 
> changes. Without it, we'd only have basic Windows event logs which miss a lot 
> of attacker activity.

**Q4: How does Active Response work technically?**
> When Wazuh detects a rule match (e.g., 5 failed logins = rule 5712), it 
> executes a script on the agent or server. For brute force, it runs 
> firewall-drop which adds an iptables rule blocking the attacker's IP.

**Q5: Could an attacker bypass your detection?**
> Yes — sophisticated attackers could disable Sysmon, use fileless techniques, 
> or tunnel through allowed protocols. Our setup detects known techniques 
> but a real APT would require additional layers like network IDS, EDR 
> behavioral analysis, and deception technology.

**Q6: Why didn't you use Cuckoo Sandbox?**
> We attempted Cuckoo and CAPE deployment but encountered significant dependency 
> issues. We pivoted to ANY.RUN which provided equivalent behavioral analysis 
> with professional-quality reports in minutes instead of hours of setup.

**Q7: What are YARA rules?**
> YARA is a pattern-matching tool for malware detection. We write rules that 
> describe strings, byte patterns, or structural features unique to a malware 
> family. If a file matches the rule, it's flagged as malicious.

**Q8: What was the most dangerous attack you simulated?**
> Credential dumping (T1003.001) — dumping LSASS memory gives attackers 
> plaintext passwords and NTLM hashes for every logged-in user, including 
> domain admins. This is often the pivot point in real breaches.

**Q9: How would this scale to a real company?**
> Wazuh supports thousands of agents. We'd add network segmentation, 
> SOAR integration for automated playbooks, threat intelligence feeds, 
> and a dedicated SOC team for 24/7 monitoring.

**Q10: What is MITRE ATT&CK and how did you use it?**
> It's a knowledge base of adversary techniques. We mapped each attack 
> to a specific technique ID (e.g., T1059.001 = PowerShell). This ensures 
> our detection covers known real-world attack patterns.

**Q11: What's the difference between dynamic and static analysis?**
> Dynamic = running the malware in a sandbox and observing its behavior 
> (network calls, file drops, registry changes). Static = examining the 
> file without executing it (strings, PE headers, disassembly).

**Q12: Did the malware actually cause damage?**
> No — it was run in ANY.RUN's sandbox (isolated environment). On our DC, 
> the EDR detected it before execution. In the sandbox, it attempted 
> C2 communication and registry persistence but was fully contained.

**Q13: What would you do differently if you started over?**
> Deploy network IDS (Suricata/Zeek) for east-west traffic monitoring, 
> use SOAR for automated playbook orchestration, and implement network 
> segmentation from the start.

**Q14: How do you ensure no false positives in your YARA rules?**
> We tested rules against clean system binaries (ls, bash, etc.) — zero 
> matches. In production, you'd test against a large corpus of legitimate 
> software before deployment.

**Q15: What GPOs did you configure and why?**
> Four: Password Policy (prevent weak passwords), Audit Policy (enable 
> event logging for detection), PowerShell Logging (capture script content), 
> CMD Restriction (reduce attack surface for non-IT users).

**Who answers what:**
- Raul: Q5, Q8, Q10 (attack-related)
- Nuray: Q1, Q3, Q15 (SIEM/logging-related)
- Nihat: Q2, Q4, Q9 (EDR-related)
- Kenan: Q6, Q7, Q11, Q12, Q13, Q14 (malware/general)

---

## Tools Summary

| Tool | Purpose | Cost | Link |
|------|---------|------|------|
| **OBS Studio** | Screen recording | Free | obsproject.com |
| **CapCut Desktop** | Video editing (4-quadrant + zoom) | Free | capcut.com |
| **Google Slides** | Presentation slides | Free | slides.google.com |
| **NotebookLM** | Generate slide content from report | Free | notebooklm.google.com |
| **YouTube Audio Library** | Background music | Free | studio.youtube.com/music |

---

## Task Assignment Summary

| Person | Day 1 (May 1) | Day 2 (May 2) | Day 3 (May 3) |
|--------|---------------|---------------|---------------|
| **Raul** | Record attack footage → send MP4 | Review video | Rehearse narration |
| **Nuray** | Record log footage → send MP4 | Review video | Rehearse narration |
| **Nihat** | Record EDR footage → send MP4 | Review video, help with slides | Rehearse narration |
| **Kenan** | Record malware footage, build slides | Edit video in CapCut, finalize slides | Lead rehearsal, finalize everything |

**Critical Deadlines:**
- May 1 midnight: All 4 raw recordings sent to Kenan
- May 2 evening: Video edited + slides done
- May 3 evening: 3+ full rehearsals completed, Q&A practiced
- May 4 morning: Present! 🎯
