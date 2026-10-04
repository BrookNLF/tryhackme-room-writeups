<div align="center">

# 👻 Boogeyman 3

*Due to the previous attacks of Boogeyman, Quick Logistics LLC hired a managed security service provider to handle its Security Operations Center. Little did they know, the Boogeyman was still lurking and waiting for the right moment to return.*

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Category](https://img.shields.io/badge/Category-Threat%20Hunting-blue)
![Completed](https://img.shields.io/badge/Completed-2nd%20of%20October%202026-green)

[![Open Room | TryHackMe](https://img.shields.io/badge/Open%20Room-TryHackMe-red?style=for-the-badge&logo=tryhackme)](https://tryhackme.com/room/boogeyman3)

*SOC Level 1 Path > Boogeyman 3*

</div>

<br>

---

<h2 align="center">📋 Summary</h2>

> The final round of the Boogeyman series - and the biggest one. This time the attacker went after the CEO, Evan Hutchinson, with a phishing email carrying an ISO file. Evan reported the email, but the stage 1 payload had already run. From there the attacker copied a malicious DLL to a hidden spot, set up a scheduled task, bypassed UAC with fodhelper.exe, downloaded Mimikatz from GitHub and dumped credentials. With those, they found a script on a file share with hardcoded domain credentials, moved to a second workstation, dumped the local admin hash, reached the domain controller, ran DCSync, and finally tried to download ransomware. I traced all of it in Kibana using Sysmon logs.

**Attack chain at a glance:**

```
Phishing email to CEO -> ISO with stage 1 payload -> mshta.exe (PID 6392)
  -> xcopy copies review.dat (malicious DLL) to %TEMP%
  -> rundll32.exe runs review.dat -> C2 at 165.232.170.151:80
  -> PowerShell creates scheduled task "Review" (persistence)
  -> fodhelper.exe UAC bypass -> high privileges
  -> Mimikatz downloaded from GitHub -> itadmin NTLM hash (pass-the-hash)
  -> IT_Automation.ps1 read from \\WKSTN-1327\ITFiles -> allan.smith password
  -> Invoke-Command to WKSTN-1327 (wsmprovhost.exe) -> administrator hash dumped
  -> DCSync on the domain controller (administrator + backupda)
  -> ransomboogey.exe downloaded from ff.sillytechninja.io
```

**Tools used:** Kibana (ELK stack), Sysmon logs (Event ID 1 and 3), KQL

<br>

---

## Contents

- [Task 1: Introduction](#task-1-introduction)
- [Task 2: The Chaos Inside](#task-2-the-chaos-inside)
- [Attack Timeline](#attack-timeline)
- [Lessons Learned](#lessons-learned)

<br>

---

## Task 1: Introduction

After two Boogeyman attacks, Quick Logistics LLC hired a managed security service provider to run its SOC. But the Boogeyman never really left - it was waiting for the right moment to come back.

The goal of this room is to analyse the new TTPs of the Boogeyman group. This time there's no email file or memory dump to dig through - everything happens in an **Elastic Stack** running on the lab machine, which I accessed straight from the browser (login `elastic` / `elastic`).

<br>

---

## Task 2: The Chaos Inside

**The scenario:** The Boogeyman had already compromised one employee without tripping any alarms and stayed quiet, waiting. Using that employee's email access, the attackers went after the CEO, Evan Hutchinson. The email looked a bit off, but Evan opened the attachment anyway. When nothing seemed to happen, he reported the email to the security team.

The team checked Evan's workstation and found the attachment in his Downloads folder - an ISO file - along with a file inside it. They believed the incident happened between the 29th and 30th of August 2023. My job: figure out what happened and how bad it is.

Before touching anything else, I set the time range in Kibana. I set it from **29th of August 2023, 00:00** to **30th of August 2023, 23:30**, so I limited the logs to that window. Without it I'd either miss events or drown in noise. Setting the time range is always my first step.

Two Sysmon event IDs did most of the work in this room:

- **Event ID 1** - process creation (what ran, with what command line, and who started it)
- **Event ID 3** - network connection (what talked to where)

<br>

---

### What is the PID of the process that executed the initial stage 1 payload?

I started with process creation events and looked for PowerShell, since attackers love using it for payload delivery.

```
winlog.event_id: 1 and process.name: powershell.exe
```

Going through the results, I noticed the PowerShell processes were started by **mshta.exe** (`process.parent.name`). mshta.exe runs HTA files and is a well-known LOLBin - a legit Windows tool that attackers abuse. That made it the launching point of the attack. The `process.parent.pid` field gave me its PID.

**✅ Answer:**
```
6392
```

<br>

---

### The stage 1 payload attempted to implant a file to another location. What is the full command-line value of this execution?

"Implant a file to another location" sounded like a copy, so I looked for built-in copy tools like `copy` or `xcopy`.

```
winlog.event_id: 1 and process.name: xcopy.exe
```

That gave me an `xcopy.exe` run copying `review.dat` from the mounted ISO (`D:\`) into Evan's Temp folder. Temp is a quiet spot where a file is less likely to be noticed.

**✅ Answer:**
```
"C:\Windows\System32\xcopy.exe" /s /i /e /h D:\review.dat C:\Users\EVAN~1.HUT\AppData\Local\Temp\review.dat
```

<br>

---

### The implanted file was eventually used and executed by the stage 1 payload. What is the full command-line value of this execution?

Next I searched for anything using `review.dat`. In the results I noticed `fodhelper.exe` - a tool often used for UAC bypass (more on that later). I checked its parent process, and the parent's command line showed how `review.dat` was actually run:

It was loaded by **rundll32.exe** with the `DllRegisterServer` export. So despite the `.dat` extension, `review.dat` is a DLL.

**✅ Answer:**
```
"C:\Windows\System32\rundll32.exe" D:\review.dat,DllRegisterServer
```

<br>

---

### The stage 1 payload established a persistence mechanism. What is the name of the scheduled task created by the malicious script?

I filtered process creation events for the stage 1 file name (`projectfinancial*`) to see everything the payload started.

Among the results was a PowerShell command referencing `review.dat`. At first it looked like just another mention of the file, but a closer look showed it was building a scheduled task - it used `New-ScheduledTaskAction` and other task cmdlets, with `rundll32.exe` running `review.dat` as the action. The task name was right there in the command.

**✅ Answer:**
```
Review
```

<br>

---

### The execution of the implanted file inside the machine has initiated a potential C2 connection. What is the IP and port used by this connection?

I already knew `rundll32.exe` was the process running the malicious DLL, so I checked its network activity using Sysmon network events.

```
winlog.event_id: 3 and process.name: rundll32.exe
```

There was an outgoing connection from `rundll32.exe` to an external IP over port 80. Plain HTTP on port 80 is a common way for C2 traffic to blend in with normal web traffic.

> 📝 Format: IP:port

**✅ Answer:**
```
165.232.170.151:80
```

<br>

---

### The attacker has discovered that the current access is a local administrator. What is the name of the process used by the attacker to execute a UAC bypass?

I'd already seen `fodhelper.exe` while working on question 3, so I filtered for it directly.

```
winlog.event_id: 1 and process.name: fodhelper.exe
```

The logs showed `fodhelper.exe` being used to run `rundll32.exe` with `review.dat`. I did a quick search to confirm: fodhelper.exe auto-elevates without showing a UAC prompt, and it reads a registry key the current user can change. Attackers put their own command in that key, launch fodhelper.exe, and their command runs with high privileges - no prompt at all.

**✅ Answer:**
```
fodhelper.exe
```

<br>

---

### Having a high privilege machine access, the attacker attempted to dump the credentials inside the machine. What is the GitHub link used by the attacker to download a tool for credential dumping?

The go-to tool for dumping credentials from memory (LSASS) is **Mimikatz**, so I searched for it.

```
winlog.event_id: 1 and process.command_line: *mimikatz*
```

One of the results was a PowerShell command downloading a ZIP file straight from the official Mimikatz GitHub releases page.

**✅ Answer:**
```
https://github.com/gentilkiwi/mimikatz/releases/download/2.2.0-20220919/mimikatz_trunk.zip
```

<br>

---

### After successfully dumping the credentials inside the machine, the attacker used the credentials to gain access to another machine. What is the username and hash of the new credential pair?

I kept the same Mimikatz filter. It returned two important events:

- **older one** - the PowerShell download from the previous question
- **newer one** - `mimikatz.exe` actually running

The command line of the newer event used `sekurlsa::pth` - **pass-the-hash**. Instead of a password, the attacker passes an NTLM hash to log in as another user. I opened the event's `message` field to read it properly: `OriginalFilename` confirmed it was `mimikatz.exe`, and the command line showed the user (`itadmin`) and the NTLM hash.

> 📝 Format: username:hash

**✅ Answer:**
```
itadmin:F84769D250EB95EB2D7D8B4A1C5613F2
```

<br>

---

### Using the new credentials, the attacker attempted to enumerate accessible file shares. What is the name of the file accessed by the attacker from a remote share?

To look for file share enumeration, I filtered on the two shells attackers usually use.

```
winlog.event_id: 1 and process.name: (cmd.exe or powershell.exe)
```

Going through the command lines, one PowerShell event stood out - it was reading a `.ps1` script directly from a share called `ITFiles` on a remote machine, `WKSTN-1327`. A script on an IT share is a classic place to find hardcoded passwords.

**✅ Answer:**
```
IT_Automation.ps1
```

<br>

---

### After getting the contents of the remote file, the attacker used the new credentials to move laterally. What is the new set of credentials discovered by the attacker?

I stayed on the same filter and kept reading. One event used a `$Credential` variable, which is always worth a closer look.

The command built a `PSCredential` object with a hardcoded username and a plaintext password turned into a secure string - so the password was sitting there in clear text. The attacker clearly got these from `IT_Automation.ps1` and used them to run commands on another machine.

> 📝 Format: username:password

**✅ Answer:**
```
QUICKLOGISTICS\allan.smith:Tr!ckyP@ssw0rd987
```

<br>

---

### What is the hostname of the attacker's target machine for its lateral movement attempt?

Same event as before. The command used `Invoke-Command` with the `-ComputerName` parameter, which runs commands on a remote machine through PowerShell Remoting. The computer name was the target.

**✅ Answer:**
```
WKSTN-1327
```

<br>

---

### Using the malicious command executed by the attacker from the first machine to move laterally, what is the parent process name of the malicious command executed on the second compromised machine?

Now I switched to the second machine and looked at its process creation events.

```
winlog.event_id: 1 and host.name: WKSTN-1327
```

The suspicious commands there had **wsmprovhost.exe** as their parent. That's the process that hosts PowerShell Remoting sessions on the receiving end - exactly what `Invoke-Command` from the first machine would create.

**✅ Answer:**
```
wsmprovhost.exe
```

<br>

---

### The attacker then dumped the hashes in this second machine. What is the username and hash of the newly dumped credentials?

Back to Mimikatz, this time on the second machine.

```
winlog.event_id: 1 and host.name: WKSTN-1327 and process.command_line: *mimikatz*
```

I found another pass-the-hash command with a `/user:` parameter. I double-checked the `process.parent.args` and `process.command_line` fields, and both showed the same username and NTLM hash - this time for the local `administrator` account.

> 📝 Format: username:hash

**✅ Answer:**
```
administrator:00f80f2538dcb54e7adc715c0e7091ec
```

<br>

---

### After gaining access to the domain controller, the attacker attempted to dump the hashes via a DCSync attack. Aside from the administrator account, what account did the attacker dump?

DCSync in Mimikatz is run with `lsadump::dcsync`, so I searched for that.

```
winlog.event_id: 1 and process.command_line: *dcsync*
```

The results showed `mimikatz.exe` running DCSync with an explicit `/user:` parameter. DCSync pretends to be a domain controller and asks the real one to hand over password hashes. One run targeted `administrator`, and another one targeted a different account - **backupda**, which by its name looks like a backup domain admin.

**✅ Answer:**
```
backupda
```

<br>

---

### After dumping the hashes, the attacker attempted to download another remote file to execute ransomware. What is the link used by the attacker to download the ransomware binary?

I'd already seen a process called `ransomboogey.exe` while scrolling through the logs, so I used it as my search term.

```
winlog.event_id: 1 and process.command_line: *ransomboogey*
```

One of the PowerShell events used `iwr` (`Invoke-WebRequest`) to download the file and save it as `ransomboogey.exe` in `evan.hutchinson`'s user folder. That's the last step - after taking over the domain, the attacker was about to deploy ransomware.

**✅ Answer:**
```
http://ff.sillytechninja.io/ransomboogey.exe
```

<br>

---

## Attack Timeline

| # | Stage | What happened | Evidence |
|---|-------|---------------|----------|
| 1 | Initial Access | Phishing email with an ISO sent to the CEO, Evan Hutchinson | Room scenario |
| 2 | Execution | Stage 1 payload runs through `mshta.exe` (PID 6392), starting PowerShell | Sysmon Event ID 1 |
| 3 | Defense Evasion | `xcopy` copies `review.dat` (DLL) from `D:\` to `%TEMP%` | Sysmon Event ID 1 |
| 4 | Execution | `rundll32.exe` loads `review.dat` with `DllRegisterServer` | Sysmon Event ID 1 |
| 5 | Persistence | Scheduled task `Review` created through PowerShell | Sysmon Event ID 1 |
| 6 | Command & Control | `rundll32.exe` connects to `165.232.170.151:80` | Sysmon Event ID 3 |
| 7 | Privilege Escalation | UAC bypass with `fodhelper.exe` | Sysmon Event ID 1 |
| 8 | Credential Access | Mimikatz downloaded from GitHub, `itadmin` hash dumped and used (pass-the-hash) | Sysmon Event ID 1 |
| 9 | Discovery | `IT_Automation.ps1` read from `\\WKSTN-1327\ITFiles`, exposing `allan.smith`'s password | Sysmon Event ID 1 |
| 10 | Lateral Movement | `Invoke-Command` to `WKSTN-1327`, commands run under `wsmprovhost.exe` | Sysmon Event ID 1 |
| 11 | Credential Access | Local `administrator` hash dumped on `WKSTN-1327` | Sysmon Event ID 1 |
| 12 | Credential Access | DCSync on the domain controller - `administrator` and `backupda` hashes | Sysmon Event ID 1 |
| 13 | Impact | `ransomboogey.exe` downloaded from `ff.sillytechninja.io` | Sysmon Event ID 1 |

<br>

---

<div align="center">

**And voilà, there you have it! 🎉**

</div>

<br>

---

## Lessons Learned

- **Set the time range first.** It sounds boring, but in Kibana it's the difference between finding the attack and drowning in logs.
- **Sysmon Event ID 1 is the MVP.** Process creation with full command lines and parent processes answered almost every question in this room.
- **Parent processes tell the story.** mshta.exe starting PowerShell, or wsmprovhost.exe starting commands, are big red flags on their own.