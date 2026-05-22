# Phishing Simulation and Detection Lab

![Security](https://img.shields.io/badge/Category-Cybersecurity-red)
![Level](https://img.shields.io/badge/Level-Intermediate-orange)
![Lab](https://img.shields.io/badge/Environment-Home%20Lab-blue)
![Status](https://img.shields.io/badge/Status-Completed-green)
![SIEM](https://img.shields.io/badge/SIEM-Splunk-black)
![OS](https://img.shields.io/badge/Attacker-Kali%20Linux-purple)
![Firewall](https://img.shields.io/badge/Firewall-pfSense-blue)
![Server](https://img.shields.io/badge/Victim-Windows%20Server%202025-lightgrey)

-----

## Project Overview

A hands-on cybersecurity home lab project that simulates a real-world phishing attack from end to end. This project covers the full attack lifecycle  from crafting and delivering a phishing email, to analyzing email headers, detecting malicious activity using a SIEM, and performing structured incident response. A pfSense firewall segments the network between the attacker and victim machines, mirroring how enterprise networks are protected. Built entirely on an isolated internal network using VirtualBox, this lab mirrors how real Security Operations Centers (SOCs) detect and respond to phishing threats in enterprise environments.

-----

## Objectives

- Simulate a phishing campaign using GoPhish and MailHog on an isolated internal network
- Segment the lab network using pfSense with WAN, LAN, and OPT1 interfaces
- Block and monitor phishing traffic at the firewall level using pfSense rules
- Analyze phishing email headers using MXToolbox to identify SPF, DKIM, and DMARC failures
- Capture and analyze network traffic using Wireshark to identify credential theft in real time
- Ingest and correlate Windows Sysmon logs in Splunk to detect phishing-related activity
- Build detection rules using Splunk SPL queries mapped to real attacker behavior
- Perform structured incident response using the PICERL framework
- Document Indicators of Compromise (IOCs) and produce a formal incident report

-----

## Lab Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                   VirtualBox Lab Environment                     │
│                                                                  │
│   ┌──────────────────────────────────────────────────────────┐   │
│   │                     pfSense Firewall                     │   │
│   │                                                          │   │
│   │    WAN                LAN                  OPT1          │   │
│   │                  192.168.2.1          192.168.1.1        │   │
│   └──────┬───────────────┬───────────────────┬──────────────┘    │
│          │               │                   │                   │
│     [Internet]    ┌──────┴──────┐     ┌──────┴──────┐            │
│                   │ KALI LINUX  │     │ WIN SERVER  │            │
│                   │192.168.2.5  │     │192.168.1.1  │            │
│                   │ (Attacker)  │     │  (Victim)   │            │
│                   │             │     │             │            │
│                   │ GoPhish     │     │ Sysmon      │            │
│                   │ MailHog     │     │ Splunk Fwd  │            │
│                   │ Wireshark   │     │             │            │
│                   │ Splunk SIEM │     │             │            │
│                   └─────────────┘     └─────────────┘            │
│                                                                  │
│   Attack Flow:                                                   │
│   Kali ──▶ pfSense (OPT1→LAN rules) ──▶ Windows Victim          │
│                                                                  │
│   Detection Flow:                                                │
│   pfSense Logs + Sysmon ──▶ Splunk ──▶ Alert                    │
│   Wireshark ──▶ HTTP Stream ──▶ Credentials Captured            │
│   MXToolbox ──▶ Header Analysis ──▶ SPF DKIM DMARC Failures     │
│                                                                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

-----

## Tools and Technologies

|Tool                      |Purpose                                            |
|--------------------------|---------------------------------------------------|
|pfSense                   |Network segmentation and firewall traffic control  |
|GoPhish                   |Phishing campaign simulation                       |
|MailHog                   |Local fake SMTP mail server                        |
|Splunk 10.2.2             |SIEM log collection and analysis                   |
|Splunk Universal Forwarder|Ships Windows logs to Splunk                       |
|Sysmon                    |Windows process and network event logging          |
|Wireshark                 |Network traffic capture and analysis               |
|MXToolbox                 |Email header analysis and SPF DKIM DMARC inspection|
|VirtualBox                |Isolated lab environment                           |

-----

## Lab Environment

|Machine       |OS                 |IP Address               |Interface     |Role               |
|--------------|-------------------|-------------------------|--------------|-------------------|
|pfSense       |pfSense CE         |192.168.2.1 / 192.168.1.1|WAN, LAN, OPT1|Firewall and router|
|Kali Linux    |Kali Linux         |192.168.2.5              |OPT1          |Attacker           |
|Windows Server|Windows Server 2025|192.168.1.1              |LAN           |Victim             |

**Network:** VirtualBox with pfSense routing between segments fully isolated, no internet required

-----

## Installation and Setup

### Configuring pfSense in VirtualBox

pfSense acts as the gateway and firewall between the attacker and victim networks.

**Step 1: Configure pfSense VM with 3 network adapters in VirtualBox:**

```
Adapter 1 (WAN):  
Adapter 2 (LAN):  Internal Adapter 
Adapter 3 (OPT1): Internal Adapter 
```

**Step 2: Boot pfSense and assign interfaces:**

```
WAN  → em0
LAN  → em1 (victim network)
OPT1 → em2 (attacker network)
```

**Step 3: Set static IPs on each interface:**

```
LAN  IP: 192.168.1.1/24
OPT1 IP: 192.168.2.1/24
```

**Step 4: Access pfSense web UI from Kali:**

```
http://192.168.2.1
Username: admin
Password: pfsense
```

-----

### Configuring pfSense Network Segmentation

**Step 1: Enable OPT1 interface:**

```
Interfaces > OPT1 > Enable Interface
IPv4 Configuration: Static
IPv4 Address: 192.168.2.1/24
Save and Apply
```

**Step 2: Configure for LAN (victim network):**

```
Services r > LAN
Range: 192.168.1.0/24
Save
```

-----

### Configuring pfSense Firewall Rules

**Rule 1: Allow OPT1 (Kali) to reach LAN (Windows) on port 80:**

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

**Rule 2: Allow LAN (Windows) to reach Splunk on Kali:**

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

**Rule 3: Allow LAN to reach MailHog on Kali:**

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

**Rule 4: Block all other LAN to OPT1 traffic:**

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

**Enable logging on all rules:**

```
Edit each rule > Check Log packets matched by this rule > Save
```

-----

### Installing MailHog on Kali Linux

**Step 1: Install Go:**

```bash
sudo apt update
sudo apt install golang -y
```

**Step 2: Install MailHog:**

```bash
go install github.com/mailhog/MailHog@latest
```

**Step 3: Start MailHog:**

```bash
~/go/bin/MailHog
```

**Step 4: Verify output:**

```
[SMTP] LISTENING on 0.0.0.0:1025
[APIv1] LISTENING on 0.0.0.0:8025
```

**Step 5: Access MailHog from Windows VM:**

```
http://192.168.2.5:8025
```

-----

### Installing GoPhish on Kali Linux

**Step 1: Download GoPhish:**

```bash
cd ~/Desktop
wget https://github.com/gophish/gophish/releases/download/v0.12.1/gophish-v0.12.1-linux-64bit.zip
```

**Step 2: Unzip and prepare:**

```bash
unzip gophish-v0.12.1-linux-64bit.zip -d gophish
cd gophish
chmod +x gophish
```

**Step 3: Run GoPhish:**

```bash
sudo ./gophish
```

**Step 4: Note auto-generated password from terminal:**

```
msg="Please login with the username admin and password XXXXXXXXX"
```

**Step 5: Open GoPhish dashboard on Kali:**

```
https://127.0.0.1:3333
Username: admin
Password: from terminal output
```

-----

### Installing Sysmon on Windows Server 2025

**Step 1: Download Sysmon from Microsoft Sysinternals**

**Step 2: Open PowerShell as Administrator:**

```powershell
cd C:\sysmon
.\sysmon64.exe -accepteula -i
```

**Step 3: Verify Sysmon is running:**

```powershell
Get-Service sysmon64
```

Expected output:

```
Status   Name
-------  ----
Running  Sysmon64
```

-----

### Installing Splunk on Kali Linux

**Step 1: Install Splunk:**

```bash
sudo dpkg -i splunk.deb
sudo /opt/splunk/bin/splunk start --accept-license
```

**Step 2: Access Splunk:**

```
http://127.0.0.1:8000
```

**Step 3: Configure receiving port 9997:**

```
Settings > Forwarding and Receiving > Configure Receiving > New Receiving Port > 9997
```

-----

### Installing Splunk Universal Forwarder on Windows Server 2025

**Step 1: Install with receiving indexer:**

```
Receiving Indexer: 192.168.2.5
Port:              9997
```

**Step 2: Create inputs.conf:**

```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```

```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = sysmon
sourcetype = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
disabled = false

[WinEventLog://Security]
index = windws
sourcetype = WinEventLog:Security
disabled = false

[WinEventLog://Application]
index = widws
sourcetype = WinEventLog:Application
disabled = false
```

**Step 3: Restart forwarder:**

```powershell
Restart-Service SplunkForwarder
```

-----

## Phase 1: Simulate

### GoPhish Configuration

**Sending Profile:**

```
Name:  MailHog
From:  itsupport@domain.com
Host:  127.0.0.1:1025
```

**Target Group:**

```
Name:        Lab Targets
First Name:  Elvis
Last Name:   (victim name)
Email:       servercis@lab.com
```

**Email Template:**

```
Name:             Password Reset
Envelope Sender:  itsupport@domain.com
Subject:          Urgent - Your Password expires today
```

```html
<p>Dear Elvis,</p>
<p>Your account password expires in 24 hours.
Click below to reset it immediately:</p>
<p><a href="{{.URL}}">Reset My Password</a></p>
<p>IT Support Team</p>
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
Name:            Phishing Detection Test
Email Template:  Password Reset
Landing Page:    Fake Office 365 Login
URL:             http://192.168.2.5
Sending Profile: MailHog
Groups:          Lab Targets
```

### Phase 1 Results

```
Email Sent          YES
Email Opened        YES
Link Clicked        YES
Credentials         CAPTURED
```

Phishing link generated by GoPhish:

```
http://192.168.2.5/?rid=XXXXXXX
```

The rid parameter is a unique tracking ID GoPhish assigns per victim to track clicks and credential submissions.

-----

## Phase 2: Detect

### 2A — Email Header Analysis Using MXToolbox

After receiving the phishing email in MailHog, the raw email headers were extracted and analyzed using MXToolbox Header Analyzer at mxtoolbox.com/EmailHeaders.aspx

**Headers extracted from the phishing email:**

```
Content-Transfer-Encoding: quoted-printable
Content-Type: text/html; charset=UTF-8
Date: Fri, 22 May 2026 01:04:25 +0100
From: itsupport@domain.com
Message-Id: <177940826506034983...@kali>
Mime-Version: 1.0
Received: from kali by mailhog.example (MailHog)
Return-Path: <it@lab.local>
Subject: Urgent - Your Password expires today
To: "server cis" <servercis@lab.com>
X-Mailer: gophish
```

**Key findings from header analysis:**

|Header Field|Value                       |Finding                                              |
|------------|----------------------------|-----------------------------------------------------|
|From        |itsupport@domain.com        |Spoofed sender address                               |
|Return-Path |it@lab.local                |Mismatch with From field — red flag                  |
|X-Mailer    |gophish                     |Identifies GoPhish as sending tool                   |
|Received    |from kali by mailhog.example|Reveals true sending server                          |
|Relay Delay |0 seconds                   |Email sent and received instantly — no legitimate MTA|

**SPF, DKIM and DMARC Analysis Results:**

|Check             |Result|Meaning                            |
|------------------|------|-----------------------------------|
|DMARC Compliant   |FAIL  |Email fails DMARC policy           |
|SPF Alignment     |FAIL  |Sender IP not authorized           |
|SPF Authenticated |FAIL  |No valid SPF record for lab.local  |
|DKIM Alignment    |FAIL  |No DKIM signature present          |
|DKIM Authenticated|FAIL  |No DKIM-Signature header found     |
|DMARC Policy      |None  |Policy set to none — no enforcement|

**DMARC Record found for domain.com:**

```
v=DMARC1; p=none; pct=100; sp=none; adkim=r; aspf=r; fo=1
```

Policy is set to none meaning emails that fail DMARC are not rejected or quarantined — a common misconfiguration in real environments.

**SPF finding:**

```
spf:lab.local
Sorry, we could not find any name servers for lab.local
```

lab.local is an internal domain with no public DNS records — confirming this is a spoofed internal sender with no legitimate SPF record.

**Conclusion from header analysis:**

This email is a phishing email. The From address is spoofed, the Return-Path does not match, there is no DKIM signature, SPF fails, and the X-Mailer header reveals GoPhish was used to send it. In a real environment this email would be quarantined or rejected if DMARC policy was set to quarantine or reject.

-----

### 2B: pfSense Firewall Detection

During the phishing campaign pfSense logs all traffic between Kali and Windows.

**View live firewall logs:**

```
Status > System Logs > Firewall
```

|Log Entry                 |What It Means                      |
|--------------------------|-----------------------------------|
|Pass OPT1 to LAN port 80  |Victim clicked phishing link       |
|Pass LAN to OPT1 port 9997|Splunk forwarder sending logs      |
|Block LAN to OPT1 any     |Blocked unexpected outbound traffic|

-----

### 2C: Wireshark Detection

Start Wireshark on Kali:

```bash
sudo wireshark
```

Apply filter:

```
http or smtp
```

After victim clicks phishing link:

```
Find POST request > Right click > Follow > HTTP Stream
```

Credentials visible in plain text:

```
email=victim%40company.com&password=Password123
```

-----

### 2D: Splunk Detection

**SPL Search 1 — All Sysmon Events Clean View:**

```
index=main sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational"
| rex field=_raw "EventID>(?<EventID>\d+)<"
| rex field=_raw "Image>(?<Image>[^<]+)<"
| rex field=_raw "DestinationIp>(?<DestIP>[^<]+)<"
| rex field=_raw "User>(?<User>[^<]+)<"
| table _time, EventID, Image, DestIP, User
```

**SPL Search 2: Detect Failed Logins After Phishing:**

```
index=main sourcetype="WinEventLog:Security"
EventCode=4625
| table _time, Computer, Account_Name, Failure_Reason, IpAddress
```


## Phase 3: Respond

### PICERL Incident Response Framework

#### Preparation

- VirtualBox lab built with pfSense firewall separating attacker and victim networks
- pfSense configured with WAN, LAN, and OPT1 interfaces and firewall rules with logging enabled
- GoPhish and MailHog installed and configured on Kali Linux
- Sysmon installed on Windows Server 2025 with event logging enabled
- Splunk Universal Forwarder configured to ship logs to Splunk SIEM on Kali
- Wireshark ready to capture live network traffic on Kali
- MXToolbox identified as the tool for email header analysis
- Splunk detection searches written and tested before simulation

#### Identification

The following evidence confirmed a phishing attack occurred:

- GoPhish dashboard showed email sent, opened, link clicked, and credentials submitted
- MXToolbox header analysis confirmed SPF fail, DKIM fail, DMARC fail, and X-Mailer showing gophish
- Return-Path it@lab.local did not match From address itsupport@domain.com — confirmed spoofing
- pfSense firewall logs showed HTTP connection from LAN (victim) to OPT1 (attacker) on port 80
- Wireshark captured HTTP POST request containing victim credentials in plain text
- Splunk Sysmon EventID 3 detected outbound connection from Windows Server to 192.168.2.5
- Splunk Security EventCode 4625 showed failed login attempts following the phishing click

#### Containment

- pfSense LAN to OPT1 rules updated to block all traffic from victim machine immediately
- Victim VM network adapter disabled in VirtualBox to fully isolate the machine
- Attacker IP 192.168.2.5 blocked at pfSense firewall level
- Phishing URL http://192.168.2.5 blocked using pfSense firewall rule
- All active sessions on victim machine terminated

#### Eradication

- Reviewed all processes on victim VM using Sysmon logs in Splunk
- Checked Windows registry run keys for persistence mechanisms
- Checked scheduled tasks for malicious entries
- Verified no malware was dropped during simulation
- Compromised credentials flagged for immediate reset

#### Recovery

- Victim VM snapshot restored to clean pre-attack state
- pfSense firewall rules reviewed and tightened
- Verified Sysmon and Splunk Forwarder still running after restore
- Confirmed no attacker artifacts remained on the system
- Final Splunk search confirmed no further suspicious activity

---

## Screenshots
Gophish running on terminal

<img width="1920" height="1080" alt="Screenshot 2026-05-21 225230" src="https://github.com/user-attachments/assets/2bc41464-3931-4fde-a304-05deba7fc0c6" />

Mailhug running on terminal

<img width="1920" height="1080" alt="Screenshot 2026-05-21 225204" src="https://github.com/user-attachments/assets/a757fda3-7b69-4352-aa4d-4ab71526ff3b" />

Users and Groups

<img width="1920" height="1080" alt="Screenshot 2026-05-22 032338" src="https://github.com/user-attachments/assets/c1d6b70e-6672-47a5-8e1b-6d1bd7a1f2bd" />

Email Template

<img width="1920" height="1080" alt="Screenshot 2026-05-22 032407" src="https://github.com/user-attachments/assets/5127178a-a7c1-4581-8964-ba8b772d1499" />

Landing Page

<img width="1920" height="1080" alt="Screenshot 2026-05-22 032432" src="https://github.com/user-attachments/assets/cef156c8-7ca5-486a-b2ee-febfe23c453a" />

Sending Profile

<img width="1920" height="1080" alt="Screenshot 2026-05-22 032454" src="https://github.com/user-attachments/assets/3c4564d3-f344-4dde-980c-99f6aec97424" />

Attack Simulation 

<img width="1920" height="1080" alt="Screenshot 2026-05-22 014943" src="https://github.com/user-attachments/assets/8f43c812-e26f-4108-a588-a5ccba841b30" />

Victim receives email

<img width="1920" height="1080" alt="Screenshot 2026-05-22 015016" src="https://github.com/user-attachments/assets/fb4eacd8-ca03-4e87-93ef-917fd46b1b29" />

Inputs Credentials

<img width="1920" height="1080" alt="Screenshot 2026-05-21 225805" src="https://github.com/user-attachments/assets/c963d9ca-cefc-461c-b43d-28167a1bd19b" />

Phishing Attack Dashboard

<img width="1920" height="1080" alt="Screenshot 2026-05-22 015124" src="https://github.com/user-attachments/assets/55702f88-a59c-4526-ad84-cd46a413e871" />

Gophish receives victims credentials

<img width="1920" height="1080" alt="Screenshot 2026-05-22 015225" src="https://github.com/user-attachments/assets/5fbe5f2d-a100-40f9-af56-f12986906bc0" />

Analyzing email header

<img width="1920" height="1080" alt="Screenshot 2026-05-22 020814" src="https://github.com/user-attachments/assets/b3a505e5-389a-427e-b3fe-2f1df492dc3e" />

<img width="1920" height="1080" alt="Screenshot 2026-05-22 020841" src="https://github.com/user-attachments/assets/581c6a0b-6975-47d5-b9fc-295da4ab16b7" />

<img width="1920" height="1080" alt="Screenshot 2026-05-22 020859" src="https://github.com/user-attachments/assets/b5cf510b-1cc0-4b8e-b862-9798dead1840" />

Wireshark Captures victims credentials

<img width="1920" height="1080" alt="Screenshot 2026-05-22 015333" src="https://github.com/user-attachments/assets/1a71de4d-6538-4a89-bf74-a8a7fe326137" />

---

#### Lessons Learned

- HTTP credential submission is visible in plain text — HTTPS must be enforced on all services
- SPF, DKIM, and DMARC failures are clear indicators of email spoofing
- DMARC policy should be set to quarantine or reject — not none
- pfSense network segmentation successfully isolated attacker and victim traffic
- Sysmon EventID 3 is a reliable early indicator of phishing link clicks
- X-Mailer header revealing gophish is a strong indicator of simulated phishing tooling
- Correlating MXToolbox, pfSense, GoPhish, Wireshark, and Splunk gives a complete attack picture

-----

## Indicators of Compromise (IOCs)

|Type             |Value                                   |
|-----------------|----------------------------------------|
|Attacker IP      |192.168.2.5                             |
|Victim IP        |192.168.1.1                             |
|Phishing URL     |http://192.168.2.5                      |
|Email From       |itsupport@domain.com                    |
|Email Return-Path|it@lab.local                            |
|Email Subject    |Urgent - Your Password expires today    |
|X-Mailer         |gophish                                 |
|Sending Server   |kali via mailhog.example                |
|SPF Result       |FAIL                                    |
|DKIM Result      |FAIL — no signature                     |
|DMARC Result     |FAIL — policy none                      |
|Sysmon EventID   |3 — network connection to 192.168.2.5   |
|pfSense Log      |HTTP pass LAN to OPT1 port 80           |
|Wireshark        |HTTP POST with credentials in plain text|

-----

## Incident Timeline

```
T+00:00  GoPhish campaign launched from Kali (192.168.2.5)
T+00:01  Phishing email delivered to victim via MailHog
T+00:01  Email headers show X-Mailer: gophish
T+00:02  Victim opens phishing email in browser
T+00:02  Email headers extracted and analyzed in MXToolbox
T+00:02  SPF, DKIM, DMARC all confirmed FAIL
T+00:03  Victim clicks Reset My Password link
T+00:03  pfSense logs HTTP connection LAN to OPT1 port 80
T+00:03  HTTP POST detected in Wireshark
T+00:03  Sysmon EventID 3 fires — connection to 192.168.2.5
T+00:04  Credentials captured in GoPhish dashboard
T+00:05  Splunk alert triggered — SOC investigation begins
T+00:06  pfSense rule updated — victim traffic blocked
T+00:07  Victim VM isolated from network
T+00:08  Incident report documentation started
```

-----

## Difficulties and Troubleshooting
1. MailHog Inbox Empty
GoPhish campaigns were launching successfully but no emails appeared in MailHog. After investigation, the Sending Profile Host was configured to port 8025 (web UI) instead of port 1025 (SMTP). Corrected the Host to 127.0.0.1:1025 and used swaks to manually send a test email to confirm MailHog was receiving before relaunching the campaign.

2. Phishing Link Stopped Working After pfSense Was Added
After segmenting the network with pfSense, clicking the phishing link on Windows Server returned no response. pfSense blocks all inter-interface traffic by default. Created explicit allow rules on pfSense permitting HTTP port 80 from OPT1 to LAN, MailHog port 8025, and Splunk Forwarder port 9997, which restored full connectivity.

3. IP and Interface Misconfiguration
Multiple campaign failures occurred because OPT1 and LAN interfaces were incorrectly mapped causing routing to break completely. Resolved by stopping all VMs, remapping interfaces correctly in pfSense, assigning static IPs to each machine, and verifying connectivity with ping before proceeding.

4. No Suspicious Events in Splunk
Splunk searches returned only normal Splunk Forwarder activity with no phishing-related events visible. The searches were being run before any simulation was triggered. Resolved by running a complete end-to-end campaign first — launching GoPhish, clicking the link on Windows Server, submitting credentials — then checking Splunk immediately after.

---

## Skills Demonstrated

- Network segmentation using pfSense with WAN, LAN, and OPT1 interfaces
- Firewall rule creation and traffic blocking using pfSense
- Phishing campaign setup and execution using GoPhish
- Email header analysis and SPF DKIM DMARC inspection using MXToolbox
- Network traffic analysis and credential extraction using Wireshark
- SIEM log ingestion and SPL query writing using Splunk
- Windows endpoint telemetry using Sysmon
- Incident response documentation using the PICERL framework
- Isolated internal lab network configuration using VirtualBox

-----

## Legal Disclaimer

> All simulations were performed exclusively within a personally owned and controlled isolated lab environment. Performing phishing simulations against real individuals or organizations without explicit written authorization is illegal and unethical.

-----

## References

- [pfSense Documentation](https://docs.netgate.com/pfsense/en/latest)
- [GoPhish Documentation](https://docs.getgophish.com)
- [MXToolbox Header Analyzer](https://mxtoolbox.com/EmailHeaders.aspx)
- [Sysmon - Microsoft Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [Splunk SPL Reference](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference)
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)
- [MailHog](https://github.com/mailhog/MailHog)

-----

> Built as part of a hands-on cybersecurity skills development journey targeting a SOC Analyst role.
