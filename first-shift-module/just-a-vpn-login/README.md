# Just a VPN Login

- **Room:** Just a VPN Login
- **Module:** [First Shift](https://tryhackme.com/module/first-shift)
- **Category:** CTI
- **Difficulty:** Easy
- **Date completed:** 15th of September 2026

## Summary

An internal SOC alert flags a VPN login for susan.martin@probablyfine.thm from Singapore. Susan is genuinely traveling there for a conference, so it looks fine at first - until she confirms she never logged in, and mentions installing a "security check" tool while on public wifi. That tool turns out to be a stealer binary. The room walks through using TryDetectThis (a threat intel platform, works like VirusTotal) to check the suspicious IP, the file hash, and a linked threat intel report to build the full picture.

## Walkthrough

### Step 1: What is the ASN number related to the IP?

Paste the IP from the room description (`37.19.201.132`) into TryDetectThis.

<img width="694" height="147" alt="image" src="https://github.com/user-attachments/assets/6790a68c-342a-4296-89e1-945761ac7eef" />

**Answer:** `212238`

### Step 2: Which service is offered from this IP?

No lookup needed here, the answer is already in the room description: "Unusual VPN login... from 37.19.201.132".
**Answer:** `vpn`

### Step 3: What is the filename of the file related to the hash?

Paste the hash from the room description into TryDetectThis:
`b8e02f2bc0ffb42e8cf28e37a26d8d825f639079bf6d948f8debab6440ee5630`

Check File Details -> File Name.

<img width="356" height="217" alt="image" src="https://github.com/user-attachments/assets/ab8b1408-7797-4c19-bebd-b21bd2e6de12" />

**Answer:** `zY9sqWs.exe`

### Step 4: What is the threat signature that Microsoft assigned to the file?

Scroll to Vendor Analysis and search (Ctrl+F) for Microsoft.

<img width="945" height="258" alt="image" src="https://github.com/user-attachments/assets/87bf7499-aa3f-4af6-84da-7c66cba8d678" />

**Answer:** `Trojan:Win32/LummaStealer.PM!MTB`

### Step 5: One of the contacted domains is part of a large malicious infrastructure cluster. Based on its HTTPS certificate, how many domains are linked to the same campaign?

Under File Communicated Behavior -> Contacted Domains, there are 3 domains with detections. Checking each one individually in TryDetectThis, `gadgethgfub.icu` is the one with real results. Its certificate listed a large number of alternative names, so I pasted the full list into Claude and had it count them.

<img width="945" height="583" alt="image" src="https://github.com/user-attachments/assets/eb43607c-5f8e-47a8-9ed2-0dabba4eb90b" />

<img width="945" height="933" alt="image" src="https://github.com/user-attachments/assets/eb156b26-8cd0-44c4-a6a2-2a25987bb6b9" />

<img width="945" height="774" alt="image" src="https://github.com/user-attachments/assets/3e1a5a52-a8cd-48d5-82cc-22c2ef488565" />

**Answer:** `151`

### Step 6: The file matches one of the YARA rules made by "kevoreilly". What line is present in the rule's "condition" field?

Back on the hash overview, under Detections and Reports there's a rule called Lumma by kevoreilly, linking to GitHub:
[kevoreilly/CAPEv2 - Lumma.yar](https://github.com/kevoreilly/CAPEv2/blob/master/data/yara/CAPE/Lumma.yar)

<img width="904" height="374" alt="image" src="https://github.com/user-attachments/assets/94570a86-af28-416c-8c76-0c64cfe3ce4a" />

The condition field is the last line of the rule.

<img width="945" height="509" alt="image" src="https://github.com/user-attachments/assets/6b43de0d-9a63-4cf7-9700-d09771799bb7" />

**Answer:** `uint16(0) == 0x5a4d and any of them`

### Step 7: The file is also mentioned in a threat intel report. What is the title of the report mentioning this hash?

Same Detections and Reports section, under the Reports tab.

<img width="945" height="274" alt="image" src="https://github.com/user-attachments/assets/276bc513-b651-4e77-9d09-cb44da4c52a7" />

**Answer:** `Behind the Curtain: How Lumma Affiliates Operate`

Report link: [recordedfuture.com - Behind the Curtain: How Lumma Affiliates Operate](https://www.recordedfuture.com/research/behind-the-curtain-how-lumma-affiliates-operate)

### Step 8: Which team did the author of the malware start collaborating with in early 2024?

Ctrl+F through the report.

<img width="945" height="201" alt="image" src="https://github.com/user-attachments/assets/7b5662b4-1ad5-40e5-aa64-b7fa3e49a01f" />

**Answer:** `GhostSocks`

### Step 9: A Mexican-based affiliate related to the malware family also uses other infostealers. Which mentioned infostealer targets Android systems?

Ctrl+F for "Android" in the report.

<img width="945" height="159" alt="image" src="https://github.com/user-attachments/assets/ce755515-a9e6-4203-890c-e1f02552dfea" />

**Answer:** `CraxsRAT`

### Step 10: The report states that the affiliates behind the malware use the services of AnonRDP. Which Mitre ATT&CK sub-technique does this align with?

Scroll to Appendix C - MITRE ATT&CK Techniques table at the bottom, looking for the VPN-related entry.

<img width="945" height="617" alt="image" src="https://github.com/user-attachments/assets/080c3621-aad1-479c-b7f3-9dd41c05b83c" />

**Answer:** `T1583.003` (Resource Development: Acquire Infrastructure: Virtual Private Server)

And voila, there you have it!

## Lessons Learned
- Ctrl+F is genuinely one of the most useful skills when digging through a tool like VirusTotal (or TryDetectThis here) - vendor lists, detection names, and long reports are packed with information, and searching for a keyword saves a lot of time versus scrolling and reading everything manually.
- VirusTotal itself (and similar threat intel platforms) is a tool security professionals reach for constantly - checking IPs, file hashes, and domains against it is a routine part of the job, not just something used in CTFs.
