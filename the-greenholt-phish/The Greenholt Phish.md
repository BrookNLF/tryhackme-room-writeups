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

*(screenshot: email opened in Thunderbird)*

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

**Transfer Reference Number (from the Subject line):** `09674321`

**Sender display name:** Mr. James Jackson

**Sender's email address:** info@mutawamarine.com

**Reply-to address:** info.mutawamarine@mail.com

**Originating IP address**, from the message source:

*(screenshot: originating IP in headers)*

`192.119.71.157`

**Owner of that IP** - ran it through [ipinfo.io](https://ipinfo.io). The ASN field gives the answer:

*(screenshot: ipinfo.io result)*

**HostPapa**

**SPF record for the Return-Path domain** - used the [SPF Checker and Validator by dmarcian](https://dmarcian.com/spf-survey/), put in `mutawamarine.com`, clicked Survey Domain:

`v=spf1 include:spf.protection.outlook.com -all`

**DMARC record for the same domain** - same site, [DMARC Inspector](https://dmarcian.com/dmarc-inspector/) tool this time:

`v=DMARC1; p=quarantine; fo=1`

**Attachment file name:** `SWT_#09674321____PDF__.CAB`

**SHA256 hash of the attachment** - downloaded it to the lab machine desktop and ran:

```
sha256sum '/home/ubuntu/Desktop/SWT_#09674321____PDF__.CAB'
```

`2e91c533615a9bb8929ac4bb76707b2444597ce063d84a4b33525e25074fff3f`

**Attachment size** - dropped the hash into VirusTotal, size shown at the top:

*(screenshot: VirusTotal result)*

**400.26 KB**

**Actual file type of the attachment** - also from VirusTotal's analysis:

*(screenshot: VirusTotal file type)*

**RAR**

So despite the `.CAB` extension and the "PDF" sitting in the filename, this is actually a RAR archive - one more sign the sender was trying to disguise what the attachment really is.

## Lessons Learned

Pay attention to the small details in an email - mismatched From/Reply-To addresses, SPF/DMARC failures, a file extension that doesn't match the real file type. If something looks off, the move is: don't click anything, don't download or open the attachment, and flag it for the security team to investigate properly.