# Phishing Books - TryHackMe Writeup

- **Room:** [Phishing Books](https://tryhackme.com/room/phishingbooks)
- **Category:** Phishing Analysis
- **Difficulty:** Easy
- **Module:** First Shift

## Summary

ProbablyFine Ltd's SOC gets a message from a university teacher, Dr. Isabella, saying she's getting repeated MFA approval requests she didn't trigger. No alerts fired in the SIEM, so the raw `.eml` file she reported had to be pulled apart manually to figure out how the phishing email slipped through and where it actually led.

## Walkthrough

### Q1 - Which specific header check explains why Isabella received the email without being rejected by the email platform?

The email had SPF, DKIM, and DMARC all set to `none`. DMARC is the one that matters because it's the enforcement layer, not just another check.

- **SPF** checks if the sending server is allowed to send for that domain.
- **DKIM** checks if the message has a valid signature.
- **DMARC** looks at the SPF and DKIM results and tells the receiving mail server what to actually *do* about it (reject, quarantine, or nothing), based on the domain owner's published policy.

So SPF=none and DKIM=none just mean those individual checks failed or weren't present. On their own, that doesn't block anything. `DMARC=none` means there was no DMARC policy telling the mail platform to take action on those failures, so it just delivered the email normally. DMARC is the control point, SPF/DKIM are just inputs to it.

*[insert image]*

**Answer:** `DMARC=none`

### Q2 - What technique did the attacker use to make the message seem legitimate?

Hint given: "Intentional". I stared at the email for a while trying to figure out what that word was pointing to, then went back and compared the From and To addresses side by side:

```
From: Kingford University Library <library@kinglord.ac.uk>
To:   isabella@kingford.ac.uk
```

That's when it clicked - `kinglord.ac.uk` vs `kingford.ac.uk`. The letters are swapped (`kinglord` instead of `kingford`), close enough that it slips past a quick read but different once you actually compare the two side by side. Registering a domain that's deliberately one or two letters off from the real one, betting the victim won't notice, is exactly what `typosquatting` is - so that's how I landed on the answer.

*[insert image]*

**Answer:** `Typosquatting`

### Q3 - Which MITRE technique and sub-technique ID best fit this sender address trick?

Searched "typosquatting mitre att&ck" - first result was Acquire Infrastructure: Domains.

*[image placeholder]*

**Answer:** `T1583.001`

### Q4 - What is the file extension of the attached file?

The attachment was named `library-invoice.pdf.html` - made to look like a PDF at a glance, but it's actually HTML.

*[insert image placeholder]*

**Answer:** `.html`

### Q5 - What is the MD5 hash of the .HTML file?

Found via the MD5 Scan (VirusTotal) link inside `EML-Analysis-Report.html`.

*[insert image]*

**Answer:** `442f2965cb6e9147da7908bb4eb73a72`

### Q6 - What is the landing page of the phishing attack?

Honestly, I'm a bit embarrassed about how easy this one was and how complicated I made it out to be. All I had to do was download the attachment to the desktop and open it - the browser lands you straight on the answer.

*[insert image]*

**Answer:** `http://lib-service.com:8083/`

### Q7 - Which MITRE technique ID was used inside the attached file?

Opened the attachment in nano - the HTML contained two JS arrays plus some logic to decode them. That's obfuscation of content within a file, which is exactly what `T1027` covers. Worth noting this is distinct from `T1566.001` (Phishing: Spearphishing Attachment, the delivery method) or `T1204.002` (User Execution: Malicious File, Isabella opening it) - T1027 is specifically about the obfuscation trick baked into the file itself.

**Answer:** `T1027`

### Q8 - What is the hidden message the attacker left in the file?

Opened the attachment in nano (same file as Q7) and saw two arrays full of `\uXXXX` sequences. I didn't recognise the format straight away, so I googled it and found out these were Unicode escape codes for individual characters - basically letters written as their character codes instead of plain text.

*[insert image]*

Once I knew what I was looking at, I figured the arrays just needed joining together and then reversing to become readable (the reversing part I worked out by noticing the strings looked backwards even after converting the escapes). CyberChef was the perfect tool for that - pasted the array contents in, joined them, reversed them, and one of the two arrays clearly spelled out a message.

*[insert image]*

**Answer:** `I love to phish books from libraries ^^`

### Q9 - Which line in the attached file is responsible for decoding the URL redirect?

Looking at the code from Q8, I could see there was a line assembling the final `src` value, but I wasn't actually sure what it was doing under the hood - `.split("").reverse().join("")` isn't something I immediately understood. Rather than guess, I put the line into AI and asked it to explain what it does step by step.

*[insert image]*

That's how I learned the URL had been stored backwards on purpose, like writing "library" as "yrarbil", so it wouldn't be obvious to anyone reading the raw code. This line takes that backwards text, splits it into single characters, flips their order, and glues them back together - spelling it correctly again. `src` ends up holding the real, readable URL the attacker wanted the browser to jump to.

**Answer:** `var src = reversed.split("").reverse().join("");`

### Q10 - What is the first URL in the redirect chain?

This took some digging. Hint was "Consider Browser Visualisation", which pointed toward inspecting the landing page's Network tab for other links. Problem: reloading the page only ever returned `304`/`404` status codes (cached), never the fresh `200` that would show the real referer chain.

**Workaround that worked:**
1. Right-click the downloaded attachment on the VM desktop -> "Make Link"
2. Right-click the new link file -> Properties -> copy the file path
3. Open a clean browser with the Network tab already open
4. Paste the copied path into the address bar and hit enter

That gave a proper first-load request, and the answer was sitting in the Request Headers -> Referer field.

**A couple of extra notes from digging into this one:**
- The `xn--...` string is **punycode** - the ASCII-safe way browsers encode domains that contain non-ASCII characters. Decoding `xn--librarytlu-13cwe32432-kwr` gives back `lіibrarytlu-13cwe32432` - same domain we already decoded from the JS, just in a different notation. One character in there isn't a real Latin "i" - it's a Cyrillic "і" (`\u0456`), the same homoglyph trick as the typosquatting in Q2/Q3, just applied to the actual landing domain this time.
- Browsers deliberately render mixed-script domains (Latin + Cyrillic here) as raw punycode instead of pretty Unicode text in the address bar, specifically so a homoglyph swap like this doesn't render as an innocent-looking domain name. That's why the `xn--` form is what you actually see on the wire and in headers, even though the JS source decodes to the "readable" Unicode version.

*[image placeholder]*

**Answer:** `http://xn--librarytlu-13cwe32432-kwr.com:8082/`

### Q11 - What is the Threat Actor associated with this malicious file and/or URL?

Submitted the URL to TryDetectThis (this room's VirusTotal equivalent) first with no luck. Stripping it down to just the bare domain (`lib-service.com`, no `http://`) got a hit straight away.

*[insert image]*

**Answer:** `Cobalt Dickens / Silent Librarian`

### Q12 - What is the main target of this Threat Actor according to MITRE?

Looked up Silent Librarian on MITRE ATT&CK - first sentence answers it directly: the group has targeted research and proprietary data at universities, government agencies, and private sector companies worldwide since at least 2013.

**Answer:** `Research and proprietary data`

## Lessons Learned

- **DMARC is the enforcement layer, SPF/DKIM are just inputs.** A domain can fail every individual check and still land in an inbox if there's no DMARC policy telling the receiving server what to do about it.
- **Typosquatting shows up more than once in the same attack.** Here it was used both in the sender's domain (`kinglord` vs `kingford`) and in the final landing page domain (a Cyrillic homoglyph swapped into "library"). Worth checking every domain in a chain, not just the first one.
- **Don't overthink the obvious step.** Q6 (landing page) had a five-second answer - just open the attachment - and I was worried it'd be far more complicated than that. Try the simple path first, then go deeper if you need to.
- **Cached browser requests hide the real redirect chain.** A reload gives you a `304` pointing at itself, not the original referer. Needed a fresh, uncached first load (via a freshly created link file into a clean browser session) to actually see it.
- **Punycode is worth recognising on sight.** Any domain starting with `xn--` is the browser's ASCII-safe encoding of a Unicode domain - almost always worth decoding when you see it in a phishing investigation, since it often means a homoglyph attack.
- **Threat intel lookups can be picky about input format.** Submitting the full URL to TryDetectThis returned nothing; stripping it to the bare domain got an immediate match. Worth trying both forms before assuming there's no data.