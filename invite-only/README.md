# Invite Only

- **Room:** [Invite Only](https://tryhackme.com/room/invite-only)
- **Path:** SOC Level 1 > Threat Analysis Tools
- **Category:** Threat Analysis
- **Difficulty:** Easy
- **Date completed:** 26th of September 2026

---

## Summary

In this room I play an SOC analyst at TrySecureMe, a Managed Server Provider. An L1 analyst flagged two suspicious indicators early in the morning, and my job is to help an L3 analyst dig into them and turn them into usable threat intelligence.

Flagged indicators:

- IP: `101[.]99[.]76[.]120`
- SHA256 hash: `5d0509f68a9b7c415a726be75a078180e3f02e59866f193b0a99eee8e39c874f`

The main tool is **TryDetectThis2.0**, a threat intelligence search app on the lab machine (it works a lot like VirusTotal). The last few questions also need a bit of Google research to find the original threat report.

---

## Walkthrough

### Q1. What is the name of the file identified with the flagged SHA256 hash?

First I started the Lab Machine and launched TryDetectThis2.0 from the desktop. Then I pasted the flagged hash from the room description into the search bar. The file name showed up right at the top of the results.

[insert screenshot]

Answer: `syshelpers.exe`

---

### Q2. What is the file type associated with the flagged SHA256 hash?

I found this one in the **Details** tab.

[insert screenshot]

Answer: `Win32 EXE`

---

### Q3. What are the execution parents of the flagged hash? List the names chronologically, using a comma as a separator. Note down the hashes for later use.

This is in the **Relations** tab, under **Execution Parents**.

[insert screenshot]

The screenshot shows two parents:

- a PowerShell script - `361GJX7J`
- a Win32 EXE - `installer.exe`

The answer is both names in chronological order, separated by a comma with no space. It's worth noting these down (especially the hash of `installer.exe`), because they come back later in the room.

Answer: `361GJX7J,installer.exe`

---

### Q4. What is the name of the file being dropped? Note down the hash value for later use.

The answer is right below Execution Parents, in the same **Relations** tab.

[insert screenshot]

Answer: `AClient.exe`

---

### Q5. Research the second hash in question 3 and list the four malicious dropped files in the order they appear (from up to down), separated by commas.

The "second hash in question 3" is the hash of the `installer.exe` execution parent:

`fa102d4e3cfbe85f5189da70a52c1d266925f3efd122091cdc8fe0fc39033942`

I duplicated the browser tab first, so I wouldn't lose the results for the first hash, and then searched for the `installer.exe` hash in the new tab.

In the **Relations** tab I scrolled all the way down to **Dropped Files**. There are 20 files there, but the question only asks about the malicious ones. These are marked with a yellow triangle with an exclamation mark, so they're hard to miss. The first three are right at the top, and the fourth one is a bit further down, so you need to scroll.

[insert screenshot]
[insert screenshot2]

Answer: `searchhost.exe,syshelpers.exe,nat.vbs,runsys.vbs`

---

### Q6. Analyse the files related to the flagged IP. What is the malware family that links these files?

Now it was time to search for the flagged IP. In the room description it's **defanged** - the dots are wrapped in square brackets (`[.]`) so nobody clicks it or connects to it by accident. TryDetectThis2.0 returns nothing for the defanged version, so I had to remove the brackets first and search for the clean IP (`101.99.76.120`).

This one took me a while. I went through every tab from top to bottom without finding anything obvious. Finally, while reading the comments in the **Community** tab, I spotted a name I recognised - **AsyncRAT**, a well-known remote access trojan. That turned out to be the answer.

[insert screenshot]

Answer: `AsyncRAT`

---

### Q7. What is the title of the original report where these flagged indicators are mentioned? Use Google to find the report.

The question says to use Google, so that's what I did. I combined the answers from the last two questions into one search:

```
AsyncRAT searchhost.exe syshelpers.exe nat.vbs runsys.vbs -site:medium.com -site:github.com -"writeup"
```

I excluded medium.com, github.com and the word "writeup" on purpose. Most TryHackMe rooms already have public writeups, and I didn't want to stumble into a ready-made solution.

The first result was a direct hit:

[insert screenshot]
[insert screenshot2]

It was a Check Point Research article. Later I also noticed that the title of this report is mentioned in the second comment in the **Community** tab from Q6 - so the clue was there all along.

Answer: `FROM TRUST TO THREAT: HIJACKED DISCORD INVITES USED FOR MULTI-STAGE MALWARE DELIVERY`

---

### Q8. Which tool did the attackers use to steal cookies from the Google Chrome browser?

This question pointed me straight back to the report from Q7[^1]. A quick CTRL+F for "steal cook" took me to the right paragraph. According to the report, the attackers **adapted the open-source tool ChromeKatz** to steal cookies from up-to-date versions of Google Chrome, and also from other Chromium-based browsers like Edge and Brave.

Answer: `ChromeKatz`

---

### Q9. Which phishing technique did the attackers use? Use the report to answer the question.

Same approach as Q8 - CTRL+F for "phishing tech". The report[^1] explains that the attackers used the ClickFix phishing technique, together with multi-stage loaders and time-based evasion, to quietly deliver AsyncRAT and a customised Skuld Stealer aimed at crypto wallets.

Answer: `ClickFix`

---

### Q10. What is the name of the platform that was used to redirect a user to malicious servers?

Again, CTRL+F, this time for "redirect". The report[^1] describes how invite links that were originally shared by legitimate communities (on websites or Telegram channels) ended up redirecting users to malicious Discord servers instead. That also explains the room's name - "Invite Only".

Answer: `Discord`

---

And voilà, there you have it!

---

## Lessons Learned

- **Pivoting is the core of threat analysis.** One hash led to its parents, the parents led to more dropped files, and the IP tied everything to one malware family. Each finding is a starting point for the next search.
- **Don't skip the Community tab.** Comments from other analysts can hold the key clue (malware family, links to reports) when the technical tabs don't make it obvious.
- **Search operators are useful.** Excluding sites and words with `-site:` and `-"word"` helped me find the original source instead of other people's solutions.
- **Always go back to the original report.** Threat intel tools show *what* is malicious, but the report explains *how* the attack works - the delivery method, the tools used and the techniques behind it.

---

[^1]: Check Point Research, "From Trust to Threat: Hijacked Discord Invites Used for Multi-Stage Malware Delivery" - https://research.checkpoint.com/2025/from-trust-to-threat-hijacked-discord-invites-used-for-multi-stage-malware-delivery/