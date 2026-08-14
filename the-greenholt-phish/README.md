# The Greenholt Phish

- **Room:** [The Greenholt Phish](https://tryhackme.com/room/phishingemails5fgjlzxc)
- **Module:** SOC Level 1 - Phishing Analysis
- **Difficulty:** Easy (Premium room)
- **Date completed:** 14th of August 2026

## Summary

A sales executive at Greenholt PLC gets an email that looks like it's from a known customer, asking for a money transfer and carrying an unsolicited attachment. Generic greeting, urgent money request, an attachment nobody asked for - classic phishing red flags. The email gets escalated to the SOC, and the job is to work through the headers and the attachment to prove, or disprove, that it's a phishing attempt.

## Walkthrough

### 1. Opening the Email

The lab machine has a file called `challenge.eml` sitting on the desktop. Opened it in Thunderbird.

<img width="1266" height="831" alt="Zrzut ekranu 2026-08-14 135018" src="https://github.com/user-attachments/assets/74cb187c-1abc-4169-8124-01d61974e93f" />


A few things jumped out straight away:

```
Reply-To: "Mr. James Jackson" <info.mutawamarine@mail.com>
From: "Mr. James Jackson" <info@mutawamarine.com>
To: webmaster@redacted.org
Subject: webmaster@redacted.org your: Transfer Reference Number:(09674321)
Date: 09 Jun 2020 22:58:27 -0700
Attachment: SWT_#09674321___PDF__.CAB
```

The From and Reply-To addresses don't match - `info@mutawamarine.com` vs `info.mutawamarine@mail.com`. Already screams phishing, but worth digging further before calling it.

### 2. Checking the Message Source

Before touching the room's questions, checked the raw message source first. Two lines stood out:

```
Received-SPF: fail (domain of mutawamarine.com does not designate x.x.x.x as permitted sender)
Authentication-Results: atlas125.free.mail.bf1.yahoo.com; spf=fail smtp.mailfrom=mutawamarine.com; dmarc=unknown
```

This is bad because it means the email wasn't sent from a server that's actually authorized to send mail for that domain, and there's no DMARC policy actively enforcing a reject or quarantine on that failure - so a spoofed message like this one can still land in the inbox instead of getting blocked automatically.

### 3. Working Through the Room's Questions

From here it's a case of pulling artifacts out of the headers and the attachment, one question at a time.

---

**Question 1:** What is the 'Transfer Reference Number' listed in the email's Subject line?

**Answer:** `09674321`

---

**Question 2:** What is the display name of the sender?

**Answer:** Mr. James Jackson

---

**Question 3:** What is the sender's email address?

**Answer:** info@mutawamarine.com

---

**Question 4:** What email address will receive a reply to this email?

**Answer:** info.mutawamarine@mail.com

---

**Question 5:** Begin analyzing the message source. What is the originating IP address of this email?
<img width="1045" height="509" alt="Zrzut ekranu 2026-08-14 140026" src="https://github.com/user-attachments/assets/dff86103-59a8-47b3-8180-c7c8bc9744f0" />

**Answer:** `192.119.71.157`

---

**Question 6:** Investigate the IP address from the previous question. Who is the owner of the originating IP?

To check this, I needed to open up an external tool - this time it's a website called [ipinfo.io](https://ipinfo.io/192.119.71.157?lookup_source=search-bar). After running the IP from the previous question through that tool, the answer is shown in the ASN field.
<img width="641" height="703" alt="Zrzut ekranu 2026-08-14 140344" src="https://github.com/user-attachments/assets/d3ff4032-60a6-4bce-b3d5-aeb978affac1" />

**Answer:** HostPapa

---

**Question 7:** Run an SPF record check on the Return-Path domain identified in the email headers. What is the full SPF record for this domain?

To do that, I needed to once again use an external tool - this time the [SPF Checker and Validator by dmarcian](https://dmarcian.com/spf-survey/). After putting in `mutawamarine.com` and clicking the Survey Domain button, the tool gave the answer.

**Answer:** `v=spf1 include:spf.protection.outlook.com -all`

---

**Question 8:** Perform a DMARC lookup for the Return-Path domain found in the email headers. What is the complete DMARC record for this domain?

To do that, I needed to use a different tool on the same site - this time the [DMARC Inspector](https://dmarcian.com/dmarc-inspector/).

**Answer:** `v=DMARC1; p=quarantine; fo=1`

---

**Question 9:** What is the file name of the attachment found in the email?

**Answer:** `SWT_#09674321____PDF__.CAB`

---

**Question 10:** Download the attachment to your virtual environment. Using the sha256sum command, what is the SHA256 hash of the file?

This one's simple - gotta download the file (in my case I saved it on the lab machine desktop), and input:

```
sha256sum '/home/ubuntu/Desktop/SWT_#09674321____PDF__.CAB'
```

The output is the SHA256 hash.

**Answer:** `2e91c533615a9bb8929ac4bb76707b2444597ce063d84a4b33525e25074fff3f`

---

**Question 11:** Investigate the file hash from the previous question using VirusTotal. What is the attachment's file size in KB (e.g., 122.31 KB)?

Gotta copy the SHA256 hash into VirusTotal, and it gives the answer at the very top.

<img width="2502" height="280" alt="image" src="https://github.com/user-attachments/assets/2a1ce7cf-cba6-4fa5-8a19-bd0a3423836b" />


**Answer:** `400.26 KB`

---

**Question 12:** Continue your research on the file. What is the actual file type of the attachment?

<img width="2502" height="280" alt="image" src="https://github.com/user-attachments/assets/fb8b7eb7-c0c1-4f37-997a-6a22eb59ccd0" />


**Answer:** RAR

Despite the `.CAB` extension and the "PDF" sitting in the filename, this is actually a RAR archive - one more sign the sender was trying to disguise what the attachment really is.

## Lessons Learned

Pay attention to the small details in an email - mismatched From/Reply-To addresses, SPF/DMARC failures, a file extension that doesn't match the real file type. If something looks off, the move is: don't click anything, don't download or open the attachment, and flag it for the security team to investigate properly.
