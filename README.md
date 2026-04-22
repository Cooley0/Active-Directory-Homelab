# Active Directory Home Lab | SOC & Attack Simulation

## Network Diagram
![Network Diagram](Network%20Diagram.png)

## Overview

I built a fully functional Active Directory home lab environment to simulate real-world enterprise attack scenarios and detect them using Splunk SIEM. This project covers the full cycle (environment setup, attack simulation, and log analysis), mirroring what a SOC Analyst might encounter on a day to day basis.

---

## Architecture

| Machine | OS | Role | IP |
|---|---|---|---|
| ADDC01 | Windows Server 2022 | Domain Controller (joseph.local) | 192.168.10.7 |
| target-PC | Windows 11 Pro | Domain-joined workstation | 192.168.10.100 |
| Splunk Server | Ubuntu Server 24.04.4 | SIEM / Log aggregation | 192.168.10.10 |
| Kali Linux | Kali Linux | Attacking machine | 192.168.10.250 |

All VMs were setup locally using VirtualBox on a NAT Network - No cloud services were used.

---

## Tools & Technologies

- **Active Directory Domain Services (AD DS)** - Identity and access management
- **Splunk Enterprise** - SIEM for log collection and threat detection
- **Splunk Universal Forwarder** - Log forwarding from endpoints to Splunk
- **Sysmon (Olaf config)** - Generated customized Sysmon configurations
- **Hydra** - Brute force attack simulation
- **Atomic Red Team** - MITRE ATT&CK-mapped attack simulation

---

### Part 1 - Splunk Setup & Log Forwarding

I Installed **Splunk Enterprise** on the Ubuntu Server VM and configured it to recieve logs from endpoints. Created a custom 'input.conf' file on the Windows machines using the Olaf Sysmon Configuration to forward these event sources into a custom index named 'endpoint':

- 'WinEventLog://Application'
- 'WinEventLog://Security'
- 'WinEventLog://System'
- 'WinEventLog://Microsoft-Windows-Sysmon/Operational'

I set the Splunk receiving port to the default **9997**

### input.conf Configuration
![input.conf](01%20Input.conf%20Configuration.png)

### Splunk Indexes (endpoint index created)
![Splunk Indexes](02%20Splunk%20Indexes.png)

### Splunk Recieving Port (9997)
![Receiving Port](03%20Splunk%20Receiving%20Port.png)

### Splunk Recieving Events from target-PC (host + sources confirmed)
![Host Confirmed](04%20Splunk%20Events%20from%20target-PC.png)

![Sources Confirmed](05%20Splunk%20Event%20Sources.png)

### Splunk Receiving Events from Both Hosts (target-PC and ADDC01)
![Both Hosts](06%20Splunk%20Both%20Hosts.png)

---

### Part 2 - Active Directory Setup

I Installed **Active Directory Domain Services** on a Windows Server 2022 VM and promoted the server to a Domain Controller for my domain 'joseph.local'. During the promotion, i configured the NTDS database, log files, and SYSVOL paths.

Attackers frequently target Domain Controllers because they store everything AD-related, including password hashes in the NTDS.dit file. This makes the DC the highest value target in any Windows enterprise environment.

After promotion, I created two **Organization Units** to simulate a realistic company structure:
- **IT** - contains user: Terry Smith (tsmith)
- **HR** - contains user: Jenny smith (jsmith)

Finally, I joined the Windows 11 Pro VM (target-PC) to the 'joseph.local' domain and verified by logging in as one of the newly created domain users (jsmith).

### Installing AD DS on Windows Server 2022
![AD DS Install](07%20AD%20DS%20Installation.png)

### NTDS / SYSVOL Paths During Domain Controller Promotion
![NTDS Paths](08%20NTDS%20-%20SYSVOL%20Paths.png)

### Creating Organizational Units and Users in AD
![AD Users](09%20Creating%20AD%20Users.png)

### Windows 11 Joined to joseph.local Domain
![Domain Join](10%20Windows%2011%20joining%20domain.png)

### Logging into Domain User (Jenny Smith) from target-PC
![Domain Login](11%20Domain%20login%20(Jenny%20Smith).png)

---

## Part 3 - Brute Force Attack Simulation (Hydra)

I Used the **Kali Linux** attack machine to simulate a brute force RDP attack against domain user Terry Smith (tsmith) on target-PC.

I created a custom 'passwords.txt' wordlist using the top 20 passwords from 'rockyou.txt' with the target password added at the bottom - simulating a targeted credential attack.

### Kali Linux IP Configuration (192.168.10.250)
![Kali IP](12%20Kali%20Linux%20IP%20Config.png)

### Enabling RDP for Domain Users on target-PC
![RDP Users](13%20Enabling%20RDP.png)

### passwords.txt Wordlist
![Password List](14%20Password%20List.png)

### Hydra Brute Force Attack - Successful Credential Discovery
![Hydra Success](15%20Hydra%20Successful%20Attack.png)

"hydra -l tsmith -P passwords.txt rdp://192.168.10.100 -V -I"

- '-l tsmith' - single username to target
- '-P passwords.txt' - password list the command is given to use
- 'rdp://192.168.10.100' - attack via RDP protocol against target-PC IP
- '-V' - verbose mode, shows every attempt in real time
- '-I' - ignore the previous restore file and start fresh



> **Note:** Crowbar was initially attempted for this step but I encountered compatibility issues with Windows 11's RDP implementation. Hydra was used as an alternative and produced identical results for detection purposes.

---

## Part 4 - Brute Force Detection in Splunk

After the Hydra attack, I searched Splunk for activity related to 'tsmith' within the last 15 minutes using the endpoint index.

**EventCode 4625 - Failed logon:** 50 failed login attempts were detected against tsmith. The rapid succession of timestamps confirms this was automated brute force activity, not a human mistyping error.

**EventCode 4624 - Successful Logon:** After the failed attempts, a successful logon was recorded. Travelling deeper into the log revealed the source workstation name as **kali** and the source network address as **192.168.10.250** - confirming the attacker machine had successfully logged on.

### Splunk Search - tsmith Events
![tsmith Search](16%20Splunk%20Search%20(tsmith).png)

### EventCode Breakdown (50x EventCode 4625 - Failed Log on)
![EventCode 4625](17%20EventCode%20Breakdown.png)

### EventCode 4625 Logs - Rapidly Failed Attempts
![Failed Logons](18%20EventCode%204625%20Failed%20Logon%20Attempts.png)

### EventCode 4624 - Successful Logon from Kali (192.168.10.250)
![Successful Logon](19%20EventCode%204624%20Successful%20Logon%20from%20Kali.png)

### Logon Source Confirmed - Kali Machine IP
![Kali Source](20%20Logon%20Source%20Confirmed.png)

---

## Part 5 - Atomic Red Team Attack Simulation

I installed **Atomic Red Team** on target-PC to simulate MITRE ATT&CK mapped attacks and generate telemetry for Splunk detection. All attack techniques reference the MITRE ATT&CK framework at [attack.mitre.org](https://attack.mitre.org).

#### Atomic Red Team Attack 1 - T1136.001 | Create Local Account

I ran 'Invoke-AtomicTest T1136.001' which simulated an attacker creating a new local administrator account named NewLocalUser as a persistence technique - mimicking what a real attacker would do after gaining initial access to maintain their foothold.

**MITRE Tactic:** Persistence
**MITRE Technique:** T1136.001 - Create account: Local Account

#### Atomic Red Team Execution - T1136.001
![ART Install](21%20AtomicRedTeam%20Installation.png)

![T1136 Command](22%20T1136.001%20Command.png)

![T1136 Result](23%20T1136.001%20Results.png)

#### Splunk Detection - NewLocalUser Account Creation
![NewLocalUser Splunk](24%20Splunk%20Detection%20NewLocalUser.png)

---

### Atomic Red Team Attack 2 - T1059.001 | Powershell Command and Scripting Interpreter

I ran 'Invoke-AtomicTest T1059.001' which simulated an attacker using PowerShell as a command and scripting interpreter - one of the most commonly abused techniques in real-world attacks.

**MITRE Tactic:** Execution
**MITRE Technique:** T1059.001 - Command and Scripting Interpreter: PowerShell

#### Atomic Red Team Execution - T1059.001
![T1059 Command](25%20T1059.001%20Command.png)

![T1059 Result](26%20T1059.001%20Results.png)

#### Splunk Detection - PowerShell Telemetry
![PowerShell Splunk](27%20Splunk%20Detection%20PowerShell%20Telemetry.png)

#### MITRE Technique IDs Detected in Splunk
![Technique IDs](28%20MITRE%20Technique%20IDs.png)

---

## Key Skills Demonstrated

- Active Directory deployment and domain configuration
- Splunk SIEM setup, index creation, and log ingestion
- Sysmon deployment and configuration for endpoint telemetry
- Brute force attack simulation and credential analysis
- MITRE ATT&CK technique mapping (T1136.001, T1059.001)
- Threat detection and log analysis using Splunk SPL queries
- Multi-VM network configuration in VirtualBox
- Troubleshooting real-world compatibility issues between tools and OS versions

---

## Attacks Simulated & Detected

| Attack | Tool | MITRE ID | Detected in Splunk | 
|---|---|---|---|
| RDP Brute Force | Hydra | T1110.001 | EventCode 4625 (50 failed), 4624 (success) |
| Local Account Creation | Atomic Red Team | T1136.001 | NewLocalUser account events |
| PowerShell Execution | Atomic Red Team | T1059.001 | 169 Sysmon/Powershell events |
