<div align="center">

# 👻 Boogeyman 2

*After having a severe attack from the Boogeyman, Quick Logistics LLC improved its security defences. However, the Boogeyman returns with new and improved tactics, techniques and procedures.*

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Category](https://img.shields.io/badge/Category-Memory%20Forensics-blue)
![Completed](https://img.shields.io/badge/Completed-3rd%20of%20October%202026-green)

[![Open Room | TryHackMe](https://img.shields.io/badge/Open%20Room-TryHackMe-red?style=for-the-badge&logo=tryhackme)](https://tryhackme.com/room/boogeyman2)

*SOC Level 1 Path > Boogeyman 2*

</div>

<br>

---

<h2 align="center">📋 Summary</h2>

> The Boogeyman is back, this time going after HR. Maxine from Quick Logistics LLC got a "job application" email with a resume attached. The Word document had a macro that downloaded a JavaScript payload and ran it with wscript.exe. That script pulled down a binary which connected back to the attacker's C2 server, and the attacker then set up a daily scheduled task to keep access. I worked through the phishing email, the macro (olevba) and a memory dump of Maxine's workstation (Volatility) to rebuild the whole chain.

**Attack chain at a glance:**

```
Phishing email (fake resume) -> Resume_WesleyTaylor.doc with VBA macro
  -> macro downloads update.png from files.boogeymanisback.lol, saves as C:\ProgramData\update.js
  -> wscript.exe runs update.js (stage 2)
  -> update.js downloads update.exe, saved as C:\Windows\Tasks\updater.exe
  -> updater.exe connects to C2 at 128.199.95.189:8080
  -> schtasks creates a daily "Updater" task running PowerShell from a registry value (persistence)
```

**Tools used:** Email client, md5sum, olevba, Volatility 3 (pstree, cmdline, netscan, memmap), strings, grep

<br>

---

## Contents

- [Task 1: Introduction](#task-1-introduction)
- [Task 2: Spear Phishing Human Resources](#task-2-spear-phishing-human-resources)
- [Attack Timeline](#attack-timeline)
- [Lessons Learned](#lessons-learned)

<br>

---

## Task 1: Introduction

The goal of this room is to analyse the new TTPs of the Boogeyman threat group after they came back for a second round.

I was given two artefacts in `/home/ubuntu/Desktop/Artefacts`:

- a copy of the phishing email
- a memory dump of the victim's workstation (`WKSTN-2961.raw`)

The main tools on the VM were **Volatility 3** (for digging through the memory dump) and **olevba** (for pulling VBA macros out of Office documents).

<br>

---

## Task 2: Spear Phishing Human Resources

The security team flagged some suspicious commands on Maxine's workstation, which started the investigation. My job: figure out what happened and how bad it is.

<br>

---

### What email was used to send the phishing email?

I opened the phishing email and checked the `From` field. A random Outlook address pretending to be a job applicant - a classic way to get HR to open an attachment, since opening resumes is literally their job.

**✅ Answer:**
```
westaylor23@outlook.com
```

<br>

---

### What is the email of the victim employee?

Straight from the `To` field of the same email.

**✅ Answer:**
```
maxine.beck@quicklogisticsorg.onmicrosoft.com
```

<br>

---

### What is the name of the attached malicious document?

The attachment was visible in the same email - a "resume" in the old `.doc` format, which still supports macros.

**✅ Answer:**
```
Resume_WesleyTaylor.doc
```

<br>

---

### What is the MD5 hash of the malicious attachment?

I saved the attachment onto the VM, went to the Artefacts directory and ran `md5sum` on it.

```bash
md5sum Resume_WesleyTaylor.doc
```

**✅ Answer:**
```
52c4384a0b9e248b95804352ebec6c5b
```

<br>

---

### What URL is used to download the stage 2 payload based on the document's macro?

Since it's a Word document sent to HR, a macro was the first thing I suspected. I ran `olevba` on it to pull out the VBA code.

```bash
olevba Resume_WesleyTaylor.doc
```

The macro downloads a file from the attacker's server. It's named `update.png` to look like a harmless image, but it gets saved and run as a script.

**✅ Answer:**
```
https://files.boogeymanisback.lol/aa2a9c53cbb80416d3b47d85538d9971/update.png
```

<br>

---

### What is the name of the process that executed the newly downloaded stage 2 payload?

The same `olevba` output showed what the macro does after the download - it runs the file with **wscript.exe**, the Windows Script Host.

**✅ Answer:**
```
wscript.exe
```

<br>

---

### What is the full file path of the malicious stage 2 payload?

Also in the macro, right next to the `wscript.exe` call. The "image" is saved as a `.js` file in `ProgramData`, a folder that's writable and doesn't draw much attention.

**✅ Answer:**
```
C:\ProgramData\update.js
```

<br>

---

### What is the PID of the process that executed the stage 2 payload?

From here on I switched to the memory dump. I used Volatility's `pstree` plugin to list the processes and `grep` to filter for `wscript.exe`.

```bash
vol -f WKSTN-2961.raw windows.pstree | grep wscript.exe
```

**✅ Answer:**
```
4260
```

<br>

---

### What is the parent PID of the process that executed the stage 2 payload?

I ran the same command without `grep`. In the tree view, the parent sits one level above `wscript.exe`, so I could read its PID from there.

```bash
vol -f WKSTN-2961.raw windows.pstree
```

**✅ Answer:**
```
1124
```

<br>

---

### What URL is used to download the malicious binary executed by the stage 2 payload?

This URL sits on the same attacker server and in the same folder as the stage 2 payload from question 5 - only the file is different. This time it's `update.exe`, the actual binary.

**✅ Answer:**
```
https://files.boogeymanisback.lol/aa2a9c53cbb80416d3b47d85538d9971/update.exe
```

<br>

---

### What is the PID of the malicious process used to establish the C2 connection?

Based on the URL, I was looking for something named like `update.exe`. I searched the process list for it and found it running under a slightly different name - **updater.exe**.

```bash
vol -f WKSTN-2961.raw windows.pstree | grep update
```

**✅ Answer:**
```
6216
```

<br>

---

### What is the full file path of the malicious process used to establish the C2 connection?

To see where the binary was running from, I used the `cmdline` plugin, which shows the full command line of every process.

```bash
vol -f WKSTN-2961.raw windows.cmdline.CmdLine
```

It was dropped into `C:\Windows\Tasks`, another folder that looks like a normal system location.

**✅ Answer:**
```
C:\Windows\Tasks\updater.exe
```

<br>

---

### What is the IP address and port of the C2 connection initiated by the malicious binary?

For network connections I used the `netscan` plugin and looked for `updater.exe` in the results.

```bash
vol -f WKSTN-2961.raw windows.netscan | grep updater
```

> 📝 Format: IP address:port

**✅ Answer:**
```
128.199.95.189:8080
```

<br>

---

### What is the full file path of the malicious email attachment based on the memory dump?

This one was already in the `cmdline` output from earlier. Word was started with the attachment as an argument, opened straight from Outlook's temporary attachment cache. The `(002)` in the name shows it was saved more than once.

**✅ Answer:**
```
C:\Users\maxine.beck\AppData\Local\Microsoft\Windows\INetCache\Content.Outlook\WQHGZCFI\Resume_WesleyTaylor (002).doc
```

<br>

---

### The attacker implanted a scheduled task right after establishing the C2 callback. What is the full command used by the attacker to maintain persistent access?

This one was a bit harder. My first idea was to dump the memory of the C2 process itself, since the attacker's commands would have gone through it.

```bash
vol -f WKSTN-2961.raw windows.memmap --dump --pid 6216
```

Then I realised I could just search the whole memory dump for `schtasks` with `strings` and `grep`, which was much faster.

```bash
strings WKSTN-2961.raw | grep schtasks
```

That gave me the full command. It creates a task called **Updater** that runs every day at 09:00 and starts a hidden PowerShell. PowerShell reads a base64 payload from the `debug` value in the registry (`HKCU:\Software\Microsoft\Windows\CurrentVersion`) and runs it. So the actual malicious code is hidden in the registry, not in a file on disk.

**✅ Answer:**
```
schtasks /Create /F /SC DAILY /ST 09:00 /TN Updater /TR 'C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -NonI -W hidden -c \"IEX ([Text.Encoding]::UNICODE.GetString([Convert]::FromBase64String((gp HKCU:\Software\Microsoft\Windows\CurrentVersion debug).debug)))\"'
```

<br>

---

## Attack Timeline

| # | Stage | What happened | Evidence |
|---|-------|---------------|----------|
| 1 | Initial Access | Spear-phishing email from `westaylor23@outlook.com` to HR (Maxine) with a fake resume | Phishing email |
| 2 | Execution | Maxine opens `Resume_WesleyTaylor.doc` from Outlook, VBA macro runs | olevba, cmdline |
| 3 | Delivery (stage 2) | Macro downloads `update.png` and saves it as `C:\ProgramData\update.js` | olevba |
| 4 | Execution (stage 2) | `wscript.exe` (PID 4260) runs `update.js` | olevba, pstree |
| 5 | Delivery (binary) | `update.js` downloads `update.exe`, saved as `C:\Windows\Tasks\updater.exe` | cmdline |
| 6 | Command & Control | `updater.exe` (PID 6216) connects to `128.199.95.189:8080` | netscan |
| 7 | Persistence | Daily "Updater" scheduled task runs hidden PowerShell with a payload stored in the registry | strings on memory dump |

<br>

---

<div align="center">

**And voilà, there you have it! 🎉**

</div>

<br>

---

## Lessons Learned

- **HR is an easy target.** Opening attachments from strangers is part of their job, so a fake resume is a perfect lure.
- **Old Office formats are risky.** `.doc` files can carry macros. Blocking macros in files from the internet would have stopped this attack at step one.
- **olevba is quick and powerful.** One command showed the download URL, the save path and the process used to run the payload.
- **File names lie.** `update.png` was a script, and `update.exe` turned into `updater.exe`. Search for partial names, not exact ones.
- **Attackers hide in normal-looking folders.** `C:\ProgramData` and `C:\Windows\Tasks` look legit at a glance, which is exactly why they were used.
- **Memory dumps tell the whole story.** With just `pstree`, `cmdline` and `netscan` in Volatility, I could rebuild the processes, file paths and the C2 connection.
- **Sometimes the simple tool wins.** I started with a process memory dump for the persistence question, but `strings` + `grep` on the whole image got the answer faster.
- **Persistence can hide in the registry.** The scheduled task itself looked boring - the real payload was a base64 blob in a registry value. Always check what a scheduled task actually runs.