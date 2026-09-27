# ItsyBitsy - TryHackMe Writeup

- **Room:** [ItsyBitsy](https://tryhackme.com/room/itsybitsy)
- **Path:** SOC Level 1
- **Difficulty:** Medium
- **Category** ELK Alert Triage
- **Completed:** 27th of September 2026

---

## Scenario

During normal SOC monitoring, analyst John saw an IDS alert pointing to possible C2 (command and control) communication from a user named Browne from the HR department. A suspicious file containing the pattern `THM:{ ________ }` was accessed.

A week of HTTP connection logs was pulled for the investigation and loaded into the `connection_logs` index in Kibana. The goal is to go through these logs, find the link and the content of the file, and answer the questions.

---

## Q1 - How many events were returned for the month of March 2022?

After logging into Kibana, I first had to find my way to the logs. From the home page I opened the menu (**☰** in the top left corner), went to **Analytics → Discover** and picked the `connection_logs` data view from the dropdown in the top left.

At first Discover showed **0 results**. That's because Kibana by default only looks at a recent time range (like the last 15 minutes), and these logs are from 2022. Since the question asks about March 2022, I set the time picker to cover the whole month (1st of March 2022 - 31st of March 2022) and the events showed up.

<img width="945" height="241" alt="image" src="https://github.com/user-attachments/assets/b8551d62-2465-4453-93af-1857edc9e8ce" />


Answer: `1482`

---

## Q2 - What is the IP associated with the suspected user in the logs?

To see which IPs show up in the logs, I clicked the **source_ip** field in the field list on the left. Kibana shows the top values of a field along with how often they appear.

There were only 2 IPs. One of them made up almost all of the traffic, while the other one appeared in only **0.4%** of events. An IP that stands out this much from normal traffic is worth a closer look, so this was my main suspect. Let's investigate!

<img width="406" height="413" alt="image" src="https://github.com/user-attachments/assets/39af59fc-09ce-4afc-bdfd-638b9818b57e" />


Answer: `192.166.65.54`

---

## Q3 - The user's machine used a legit Windows binary to download a file from the C2 server. What is the name of the binary?

I filtered the logs down to the suspicious IP from Q2, which left only 2 events. I went through the details of both of them. Since the question is about what the user's machine used, I focused on the fields describing the client side of the connection - and the **user_agent** field had the value `bitsadmin`.

I didn't know it at first, so I googled it. **bitsadmin** is a built-in Windows command-line tool for managing BITS (Background Intelligent Transfer Service) jobs - it's meant for downloading and uploading files in the background. Because it's a legitimate, signed Microsoft tool, attackers like to abuse it to download malware without raising suspicion. This technique is known as "living off the land" (LOLBin).

<img width="2559" height="604" alt="image" src="https://github.com/user-attachments/assets/00be213c-b296-40b3-a5a5-284aae302afe" />


Answer: `bitsadmin`

---

## Q4 - The infected machine connected with a famous filesharing site in this period, which also acts as a C2 server used by the malware authors to communicate. What is the name of the filesharing site?

The answer was already visible in the same events from Q3. The **host** field showed where the machine connected to - a well-known text sharing site, often abused by attackers to host payloads or commands.

Answer: `pastebin.com`

---

## Q5 - What is the full URL of the C2 to which the infected host is connected?

Again, in the same event from Q3, the **uri** field showed the path that was requested: `/yTg0Ah6a`. Putting the host from Q4 and this path together gives the full URL.

Answer: `pastebin.com/yTg0Ah6a`

---

## Q6 - A file was accessed on the filesharing site. What is the name of the file accessed?

To answer this one, I had to leave Kibana and open the URL from Q5 in the browser. The paste was still up, and its name was shown at the top of the page.

<img width="945" height="369" alt="image" src="https://github.com/user-attachments/assets/1874481a-9348-4d8b-b6ca-c94d1e3aa822" />


Answer: `secret.txt`

---

## Q7 - The file contains a secret code with the format THM{_____}.

The content of the paste was visible on the same page as in Q6, and it contained the flag.

Answer: `THM{SECRET__CODE}`

---

## Summary

A short but nice exercise in finding the needle in the haystack. The key steps were:

- setting the right time range in Kibana (otherwise you see nothing),
- using field statistics to spot an IP that stands out from normal traffic,
- reading the **user_agent**, **host** and **uri** fields to rebuild what the machine actually did.

The main takeaway: attackers don't always need custom malware - a legit Windows tool like bitsadmin and a public site like Pastebin can be enough to download files and talk to a C2, which makes this kind of activity easy to miss if you don't look closely.
